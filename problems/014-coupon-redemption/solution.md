## Solution: Coupon Code Redemption System

### TL;DR

A coupon system is a small, write-light service with one nasty bit: surviving a launch burst where 10,000 users hit the same code in the same second and exactly 1000 of them must win. Everything else is plumbing.

The design has two layers. Postgres holds the source of truth, with a unique index on `(code_id, user_id)` that makes double-redemption impossible at the storage level. Redis with a Lua script handles the hot-burst claim path so the database is not asked to serialize 10,000 transactions on one row. Async writes from Redis to Postgres keep the two in sync; the unique index is the safety net when they disagree.

Three code patterns coexist under one schema: generic shared codes (`SAVE10`), unique per-user codes (`UID-XXXX`), and pre-generated pools (`BLACKFRI-XXXX`). All three use the same redeem API; the difference is in how codes are minted and how the claim is structured. A Bloom filter in front of the read path stops brute-force traffic from reaching the DB. Rate limits stop the same user from grinding the namespace.

The interesting engineering is at three edges: making the launch burst correct without a database stampede, defining what "release the code back to the pool on refund" means without creating a parallel race, and laying down an audit trail that finance can use for accounting years later.

### 1. Clarifying questions and why each matters

Covered in `question.md`. The single most important question: single-use or reusable. They have completely different data models and contention shapes. Almost as important: which of the three code patterns (shared, unique, pool) the system supports. If you assume "just one pattern" and the interviewer wanted all three, you redesign mid-interview.

### 2. Capacity estimates (full working)

- **Launch burst:** 10,000 redeem requests in 1 second, of which 1000 must succeed and 9000 must be told "sold out". One hot key, one hot campaign. The 9000 losers are the design pressure, not the 1000 winners. They all need a fast response, not a timeout.
- **Steady redemption:** 50k/day = ~0.6/sec sustained, ~5/sec peak business hours. Trivial.
- **Validate calls:** 5x redeems = ~3/sec sustained, ~25/sec peak. Plus a burst of 50k at 9am launch when users paste the code into the cart before clicking redeem.
- **Storage:** 200M per-user codes × ~200 bytes = ~40GB. Add 9GB of redemption log over 5 years. ~50GB total. One Postgres instance.
- **Hot working set:** one campaign at a time. The campaign row, the counter, and (for pool campaigns) the unclaimed code list. ~100KB to a few MB. Fits in any cache.

The system is small. The architecture exists for **burst correctness and abuse resistance**, not throughput or storage.

### 3. API design

**Validate (does not consume):**

```
POST /api/v1/coupons/validate
Authorization: Bearer <token>     # optional; required for audience-restricted codes
Content-Type: application/json

{
  "code": "BLACKFRI100",
  "cart_id": "cart_abc",            # for stacking checks and cart-context discounts
  "subtotal": 12000                  # cents
}
```

Responses:

| Status | Meaning | Body |
|--------|---------|------|
| 200 OK | Code valid for this user/cart | `{"valid": true, "discount": {"type": "percent", "value": 100, "max_off": 50000}, "stackable_with": ["FREESHIP"], "expires_at": "..."}` |
| 200 OK | Code valid in isolation but fails stacking | `{"valid": false, "reason": "not_stackable_with_SAVE10"}` |
| 404 Not Found | Code does not exist | `{"valid": false, "reason": "unknown_code"}` |
| 410 Gone | Code expired or fully redeemed | `{"valid": false, "reason": "expired" / "exhausted"}` |
| 403 Forbidden | User not in audience | `{"valid": false, "reason": "audience_mismatch"}` |
| 429 Too Many Requests | Too many validate attempts | `{"reason": "rate_limited", "retry_after": 60}` |

**Redeem (consumes):**

```
POST /api/v1/coupons/redeem
Authorization: Bearer <token>     # required
Idempotency-Key: <uuid>            # required
Content-Type: application/json

{
  "code": "BLACKFRI100",
  "order_id": "ord_xyz",
  "user_id": "usr_42"
}
```

Responses:

| Status | Meaning | Body |
|--------|---------|------|
| 201 Created | Redemption succeeded | `{"redemption_id": "rdm_...", "code": "BLACKFRI100", "discount_applied": {...}}` |
| 200 OK | Idempotent: same Idempotency-Key seen before | same shape as 201 |
| 409 Conflict | User already redeemed this code | `{"error": "already_redeemed", "redemption_id": "rdm_..."}` |
| 410 Gone | Code is exhausted or expired (lost the race) | `{"error": "exhausted"}` |
| 403 Forbidden | Audience mismatch | `{"error": "audience_mismatch"}` |

**List user's redemptions:**

```
GET /api/v1/users/{user_id}/redemptions?status=active&limit=50
```

**Admin: create campaign:**

```
POST /api/v1/admin/campaigns
{
  "name": "blackfri",
  "type": "pool",                   # "shared" | "unique" | "pool"
  "discount": {"type": "percent", "value": 100},
  "total_codes": 1000,
  "per_user_limit": 1,
  "audience_filter": {"min_subscription_days": 30},
  "starts_at": "2026-11-27T09:00:00-08:00",
  "ends_at":   "2026-11-30T23:59:59-08:00",
  "code_format": {"prefix": "BLACKFRI-", "suffix_length": 8, "alphabet": "base32"}
}
```

**Notes on the API:**

- **Validate and redeem are separate.** Validate is read-only and cheap. Redeem is the atomic act. Treating them as one endpoint (POST that always consumes) breaks the cart UX where users want to see the discount before clicking checkout.
- **Idempotency-Key on redeem is required, not optional.** Network retries are guaranteed at launch burst. Without the key, the same user's retry can consume two codes.
- **`order_id` ties the redemption to the order.** Needed for refund/release flow and for finance reconciliation.
- **403 with `audience_mismatch` does not reveal whether the code itself is valid.** Returning 404 to outsiders prevents harvesting; returning 403 only to authenticated users in the wrong audience is the right balance.

### 4. Data model

```sql
-- Campaigns: one row per promotional campaign.
CREATE TABLE campaigns (
    campaign_id      UUID PRIMARY KEY,
    name             TEXT UNIQUE NOT NULL,             -- "blackfri", "newsletter-q4"
    type             TEXT NOT NULL,                    -- 'shared' | 'unique' | 'pool'
    discount         JSONB NOT NULL,                   -- {"type": "percent", "value": 100, "max_off": 50000}
    total_codes      INT,                              -- NULL for unlimited (rare)
    issued_codes     INT NOT NULL DEFAULT 0,           -- monotonic counter of codes minted
    redeemed_count   INT NOT NULL DEFAULT 0,           -- updated async from redemptions
    per_user_limit   INT NOT NULL DEFAULT 1,
    audience_filter  JSONB,                            -- {"min_subscription_days": 30, ...}
    stacking_rules   JSONB,                            -- {"stackable_with": [...], "excludes": [...]}
    starts_at        TIMESTAMPTZ NOT NULL,
    ends_at          TIMESTAMPTZ NOT NULL,
    status           TEXT NOT NULL DEFAULT 'active',   -- 'active' | 'paused' | 'ended'
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_campaigns_status_window ON campaigns (status, starts_at, ends_at);

-- Codes: one row per individual code string.
--   - shared:  one row total; counter tracked on the campaign
--   - unique:  one row per (campaign, intended_user)
--   - pool:    one row per code, intended_user is NULL until claimed
CREATE TABLE codes (
    code_id          UUID PRIMARY KEY,
    campaign_id      UUID NOT NULL REFERENCES campaigns(campaign_id),
    code             TEXT NOT NULL,                    -- "BLACKFRI100", "BLACKFRI-AB7K2X9P"
    intended_user    TEXT,                             -- non-null for 'unique' type
    state            TEXT NOT NULL DEFAULT 'unused',   -- 'unused' | 'claimed' | 'used' | 'expired'
    claimed_by       TEXT,                             -- user_id who claimed (pool type)
    claimed_at       TIMESTAMPTZ,
    expires_at       TIMESTAMPTZ,                      -- inherits from campaign unless overridden
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_codes_code ON codes (code);    -- global uniqueness of code string
CREATE INDEX idx_codes_campaign_state ON codes (campaign_id, state) WHERE state = 'unused';
CREATE INDEX idx_codes_intended_user ON codes (intended_user) WHERE intended_user IS NOT NULL;

-- Redemptions: immutable record of every successful claim.
CREATE TABLE redemptions (
    redemption_id    UUID PRIMARY KEY,
    code_id          UUID NOT NULL REFERENCES codes(code_id),
    campaign_id      UUID NOT NULL REFERENCES campaigns(campaign_id),
    user_id          TEXT NOT NULL,
    order_id         TEXT NOT NULL,
    discount_applied JSONB NOT NULL,                   -- snapshot of the discount at redeem time
    idempotency_key  TEXT NOT NULL,
    redeemed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    released_at      TIMESTAMPTZ,                      -- non-null if released back due to refund
    released_reason  TEXT
);
CREATE UNIQUE INDEX idx_redemption_user_campaign ON redemptions (campaign_id, user_id)
    WHERE released_at IS NULL;                         -- enforces per-user limit at the DB level
CREATE UNIQUE INDEX idx_redemption_idempotency ON redemptions (idempotency_key);
CREATE INDEX idx_redemption_order ON redemptions (order_id);

-- Attempts: every redeem attempt (success or failure) for audit and fraud signals.
CREATE TABLE redemption_attempts (
    attempt_id       BIGSERIAL PRIMARY KEY,
    code             TEXT NOT NULL,
    user_id          TEXT,
    ip               INET,
    user_agent_hash  BYTEA,
    result           TEXT NOT NULL,                    -- 'success' | 'unknown_code' | 'exhausted' |
                                                      -- 'already_redeemed' | 'expired' |
                                                      -- 'audience_mismatch' | 'rate_limited'
    attempted_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_attempts_user_time ON redemption_attempts (user_id, attempted_at DESC);
CREATE INDEX idx_attempts_ip_time ON redemption_attempts (ip, attempted_at DESC);
CREATE INDEX idx_attempts_result_time ON redemption_attempts (result, attempted_at DESC);
```

**Why each table:**

- **`campaigns`** holds the policy. Discount, audience, stacking, total cap. The `redeemed_count` is updated asynchronously from the redemptions log; it is a denormalized counter, not the source of truth for "is the campaign exhausted." The source of truth for that is `COUNT(*) FROM redemptions WHERE campaign_id = ?`, but we never query that on the hot path; we trust the counter and reconcile periodically.
- **`codes`** holds the issued strings. For shared campaigns, one row. For unique campaigns, one row per recipient. For pool campaigns, N rows pre-generated. The `unused` filtered index makes "find me an unclaimed code" fast.
- **`redemptions`** is the audit-grade record. The unique index on `(campaign_id, user_id)` is the database-level enforcement of per-user single-use. This is the safety net behind whatever Redis says.
- **`redemption_attempts`** logs everything, including failures. Required for fraud signals (50 attempts/sec from one IP) and for the abuse-prevention dashboards. Partitioned by month in production.

**Choice: Postgres.** Strong consistency for the unique index, transactional semantics for the claim, JSONB for the flexible discount/audience shapes. Cassandra or DynamoDB would force you to invent the unique-claim logic. The data volume is small (50GB total at 5-year retention), so Postgres has no scaling problem.

### 5. Core algorithm: atomic claim

#### Approach A: Postgres-only, low traffic

For campaigns expected to see fewer than a few hundred QPS, skip Redis entirely.

```sql
BEGIN;

-- For 'pool' type: claim one unused code atomically.
WITH next_code AS (
  SELECT code_id FROM codes
  WHERE campaign_id = $campaign_id AND state = 'unused'
  ORDER BY code_id
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE codes c
SET state = 'used', claimed_by = $user_id, claimed_at = NOW()
FROM next_code nc
WHERE c.code_id = nc.code_id
RETURNING c.code_id, c.code;

-- If the above returned 0 rows, return 'exhausted' and ROLLBACK.

-- For 'shared' or 'unique' type: insert directly into redemptions.
INSERT INTO redemptions (redemption_id, code_id, campaign_id, user_id, order_id,
                         discount_applied, idempotency_key)
VALUES ($redemption_id, $code_id, $campaign_id, $user_id, $order_id,
        $discount, $idem_key)
ON CONFLICT (campaign_id, user_id) WHERE released_at IS NULL
DO NOTHING
RETURNING redemption_id;

COMMIT;
```

`FOR UPDATE SKIP LOCKED` is the key. Without `SKIP LOCKED`, concurrent transactions queue up on the first unused row; with it, each picks a different unused row.

The `ON CONFLICT ... DO NOTHING RETURNING` is the per-user single-use enforcement. If the user already has a non-released redemption for this campaign, the insert returns zero rows and we return 409.

Throughput: a few hundred claims per second on a single Postgres on this exact query, limited by row-level locking on the campaign's row set.

#### Approach B: Redis Lua for the burst, Postgres as backstop

For BLACKFRI100-style 10k QPS bursts, the hot path goes to Redis.

```lua
-- KEYS[1] = "campaign:{cid}:remaining"
-- KEYS[2] = "campaign:{cid}:users"
-- KEYS[3] = "campaign:{cid}:pool"  (Redis list of pre-generated codes, only for pool type)
-- ARGV[1] = user_id
-- ARGV[2] = campaign_type ('shared' | 'pool')

local remaining = tonumber(redis.call('GET', KEYS[1]))
if remaining == nil then
  return {'err', 'unknown_campaign'}
end
if remaining <= 0 then
  return {'err', 'exhausted'}
end

local already = redis.call('SISMEMBER', KEYS[2], ARGV[1])
if already == 1 then
  return {'err', 'already_redeemed'}
end

local claimed_code = nil
if ARGV[2] == 'pool' then
  claimed_code = redis.call('LPOP', KEYS[3])
  if claimed_code == false then
    return {'err', 'exhausted'}
  end
end

redis.call('DECR', KEYS[1])
redis.call('SADD', KEYS[2], ARGV[1])

return {'ok', claimed_code}
```

The Lua block runs atomically. Redis is single-threaded so concurrent scripts serialize on one CPU. Latency under 1ms; throughput well above 50k ops/sec on a single Redis node, far above the 10k burst target.

After Redis returns OK, the service writes the redemption to Postgres synchronously (~5ms). Total redeem latency: ~6ms.

The Postgres write uses the same `ON CONFLICT (campaign_id, user_id)` clause as Approach A. If Redis hiccupped and somehow let two requests through for the same user, the DB unique index catches the second one. The service handles this case by rolling back the Redis state (re-adding the user to the set, incrementing the counter) and returning 409 to the second user.

#### Hybrid: when to use which

| Campaign expected QPS | Approach |
|----------------------|----------|
| < 100 | Approach A (Postgres only) |
| 100 to 10,000 | Approach B (Redis + Postgres) |
| > 10,000 | Approach B with sharded Redis (counter and user-set partitioned by hash of user_id) |

The hybrid in production: every campaign uses Redis. For low-traffic ones the Redis cost is negligible and the consistent code path is worth it. For high-traffic ones the Redis layer is what keeps the DB alive.

#### Bloom filter prefilter

Before validate even queries Redis or the DB, check a Bloom filter of all issued codes:

```python
def validate(code):
    if not bloom_filter.maybe_contains(code):
        return 404  # definitely not a real code
    # falls through to Redis/DB
```

Bloom filters never have false negatives (if it says "not present", the code definitely was never issued). They have a tunable false-positive rate (~0.1% is typical). At 0.1%, a brute-force scraper sees 99.9% of attempts cut off at the filter, never touching the DB.

The filter is rebuilt when a campaign is created (add all new codes) and reloaded on service start. Memory footprint: ~12 bits per code at 0.1% FPR. 200M codes = ~300MB. Fits in process memory.

### 6. Architecture (already in question.md)

Reviewing the components:

| Component | Stateful? | Scaling story |
|-----------|-----------|--------------|
| API Gateway + Rate Limiter | Limited (token buckets) | Horizontal; rate-limit state in Redis |
| Coupon Service (read) | No | Horizontal pods; bounded by cache fill |
| Coupon Service (write) | No (state in DB/Redis) | Horizontal pods; partitioned by campaign for hot campaigns |
| Read Cache (Redis) | Yes | Replicated; warmable on demand |
| Redis (hot-burst) | Yes | Per-campaign keyspace; can shard one campaign across nodes via hash |
| Postgres | Yes | Primary + read replica; partition redemptions table by month |
| Kafka | Yes | Standard cluster scaling |

### 7. Read and write paths

**Validate path (read-only, ~3-25 QPS sustained, burst at launch):**

1. Client → API Gateway. Per-user rate limit (10/min validate, separate bucket from redeem).
2. Coupon Service (read) normalizes the code (uppercase, strip whitespace).
3. Bloom filter check. If miss, return 404 immediately. ~99% of brute-force load stops here.
4. Read cache lookup by code. Cache holds: campaign_id, discount, expires_at, audience_filter, status, stackable_with. TTL 60s with jitter.
5. Cache miss: query Postgres `SELECT ... FROM codes JOIN campaigns ON ... WHERE code = ?`. Populate cache.
6. Check `expires_at`: return 410 if past.
7. Check `audience_filter` against the authenticated user's attributes (fetched from user service, also cached). Return 403 if mismatch.
8. Check stacking rules against the cart's existing coupons. Return 200 with `{"valid": false, "reason": "not_stackable_with_..."}` if conflict.
9. Return 200 with discount shape.

P99 target: 50ms. Most calls are cache hits at ~5ms.

**Redeem path (atomic claim):**

1. Client → API Gateway. Per-user rate limit (5/min redeem). Stricter than validate because each successful redeem has real cost.
2. Coupon Service (write) checks Idempotency-Key in Redis (5min TTL). If present, return the cached response. Required to handle network retries during the launch burst.
3. Normalize code. Bloom filter check. Cache lookup for campaign metadata.
4. Verify audience filter, expiry, status.
5. **Atomic claim (hot path):**
   - Call Redis Lua script. Returns OK with optional claimed_code (pool type), or err with reason.
   - On `exhausted` or `already_redeemed`, log to redemption_attempts and return 410 or 409.
6. **Persist (synchronous):**
   - `INSERT INTO redemptions ...` with `ON CONFLICT (campaign_id, user_id) DO NOTHING RETURNING`.
   - If conflict (DB says user already redeemed but Redis said no), this is a state drift. Roll back Redis: `SREM users {user_id}, INCR remaining`. Return 409. Alert.
7. **For pool type**, also update the `codes` row: `UPDATE codes SET state = 'used', claimed_by = ?, claimed_at = NOW() WHERE code = ?`.
8. Cache the response under the Idempotency-Key (5min TTL).
9. Emit `coupon.redeemed` event to Kafka.
10. Return 201 with redemption_id and discount.

P99 target: 50ms at burst. Breakdown: rate limit ~1ms, Redis Lua ~1ms, Postgres insert ~5-10ms, Kafka publish (async, fire-and-forget) ~0ms blocking. Most time is in network round-trips.

### 8. Scaling journey: 10 to 1M users

This is the section the interviewer cares about most. Each stage names what just broke and the cheapest fix.

#### Stage 1: 10 to 100 users (a small startup)

**What you build:**
- One Postgres (db.t3.small).
- One application instance.
- Campaigns table, codes table, redemptions table with the unique index on `(campaign_id, user_id)`.
- Validate and redeem are both straight DB calls in a transaction.
- No Redis, no cache, no Kafka. The cart service polls the redemption table or you call it inline.

**Why this is enough:**
- ~10 redemptions per day. ~50 validate calls per day. Postgres yawns.
- The unique index on redemptions enforces per-user single-use atomically. That alone is the entire correctness story.
- No campaign sees more than 1 QPS. No race conditions of concern.

**What you do *not* build:**
- No Bloom filter. The DB is fine handling 1-2 QPS of bogus codes.
- No rate limiting. You will add it the first time someone runs a script against you, not before.
- No async event stream. The cart service calls the coupon service directly during checkout.

**Total cost:** ~$50/month for the DB, ~$30/month for the app.

#### Stage 2: 1,000 users (small business)

**What just broke:**

- A user shared a code on a deal forum. 200 anonymous users tried it in 5 minutes. Most failed validation. Each failure was a DB query. The DB had a brief spike to 50% CPU.
- Marketing started running per-user codes. A campaign with 2000 codes mailed individually means 2000 rows in `codes`. Lookups by code string get slower without an index. You add `CREATE UNIQUE INDEX idx_codes_code ON codes (code)` and the issue disappears.
- A bug in the cart UI caused some users to submit redeem twice in quick succession. The unique index caught the double-claim correctly, but the second request got a confusing 500 (unhandled unique violation). You add proper `ON CONFLICT DO NOTHING RETURNING` handling and return 409.

**What you add:**

- **Per-user rate limiting** in the API Gateway. Token bucket in Redis. 10 validate/min, 5 redeem/min per authenticated user. Catches the casual script kid.
- **Per-IP rate limiting** for unauthenticated traffic. 30 validate/min per IP.
- **In-process LRU cache** for campaign and code lookups. 1000 entries, 60s TTL. Most validate calls hit the cache. DB load drops ~5x.
- **Redemption_attempts table** to log every attempt for fraud signals. Partitioned monthly. You start a dashboard showing "top 10 IPs by failed-validate rate in the last hour."
- **Idempotency-Key on redeem.** Required, not optional. Stored in Redis with 5min TTL. The cart UI bug stops causing problems.

**What you do *not* yet build:**

- No Bloom filter. At 1k users you have at most a few thousand distinct code attempts per day. The cache absorbs most of it.
- No Redis for the atomic claim. Postgres handles all redeems directly with `FOR UPDATE SKIP LOCKED`. Max contention is on the hottest campaign which sees maybe 10 QPS.
- No Kafka. The cart calls the coupon service inline. Latency is fine.

**New cost:** ~$200/month (Redis for rate limits and idempotency keys, slightly bigger DB).

#### Stage 3: 100,000 users (mid-size e-commerce)

**What just broke:**

- The first real flash sale. 2000 codes for FLASH50 dropped at noon. 5000 users tried at once. Postgres CPU pegged at 100% for 30 seconds. The `FOR UPDATE SKIP LOCKED` query serialized on the `(campaign_id, state = 'unused')` index. Most requests timed out at the gateway. Marketing was furious.
- Someone scripted a brute-force across the SAVE-prefix namespace. 5000 attempts/sec from rotating IPs. Even with the LRU cache, the misses (which are guaranteed-misses on bogus codes) hit the DB at ~1000 QPS. The DB was healthy but the dashboard was screaming.
- The cart service started experiencing race conditions: a user redeemed a code, the cart applied the discount, then the user removed an item from the cart that made the discount no longer applicable. The discount stayed applied. Edge case but real.

**What you add:**

- **Redis with Lua for atomic claims on hot campaigns.** When a campaign is marked `hot` at creation time (admin flag, or automatic if expected_qps > 100), the system maintains a Redis counter (`campaign:{id}:remaining`) and a Redis set of users (`campaign:{id}:users`). The Lua script in section 5 does the atomic claim. Postgres backstop with the unique index still catches drift.
- **For pool campaigns**, pre-load all unclaimed codes into a Redis list. `LPOP` is sub-millisecond.
- **Bloom filter prefilter** loaded at service start, updated when campaigns are created. Bogus codes return 404 in microseconds without touching Redis or DB. The brute-force scraper is reduced to noise.
- **Async write to Postgres for redemptions on hot campaigns.** Redis says yes immediately; the Postgres write happens within ~50ms via a small per-pod outbox. The unique index catches any double-write. Acceptable risk: if Redis loses state in that 50ms window, you have ~5 unaudited redemptions to reconcile.
- **Kafka topic `coupon.redeemed`.** The cart service consumes the event and applies the discount idempotently (uses `redemption_id` as the dedup key). The coupon service no longer makes a synchronous call to the cart.
- **Read replica** for the validate path. The write path stays on the primary.
- **Velocity-based campaign pause.** If redemption_attempts for a campaign spikes 100x in 1 minute over the trailing-hour baseline, auto-pause and page the on-call. Catches both leakage and bugs.

**Why this is enough:**

- Flash sales now serve 10k QPS on the hot campaign without breaking a sweat. Redis runs the whole burst on one core.
- Bloom filter cuts brute-force DB load by ~99.9%.
- The Postgres unique index is still the safety net. Any drift between Redis and Postgres is caught and reconciled.

**New cost:** ~$1-2k/month (Redis cluster, Kafka cluster, read replica, bigger primary DB).

#### Stage 4: 1,000,000 users (large e-commerce / multi-region)

**What just broke:**

- A single Redis node was the bottleneck for the hottest campaign during Black Friday. 50k QPS on one Redis core saturated it. Latency went from 1ms to 50ms.
- EU operations launched. EU users redeem codes on the EU site; the codes need to be valid across both regions. Cross-region replication of Redis state introduces ambiguity about who got the code.
- A bug in pre-generated code minting caused 10 million pool codes to be inserted with the same campaign_id but wrong discount value. Bulk fixup was needed without invalidating the 50,000 already-redeemed ones.
- Marketing wants per-customer-segment campaigns: "the same code SAVE10 means 20% off for VIPs and 10% off for everyone else." The campaign model needs a layer of indirection.

**What you add:**

- **Sharded Redis for ultra-hot campaigns.** The counter and user-set for the hottest campaign are partitioned by hash(user_id) across N Redis nodes. Each node holds 1/N of the cap. A user's redeem call routes to the node owning their hash. Lua script runs per-node. Total throughput scales linearly with N.
  - Caveat: the "remaining" count is per-shard. The total is the sum. If one shard is exhausted but others have slots, late users to the wrong shard see "exhausted" while slots exist elsewhere. Mitigation: route losers to a fallback shard once. Or accept that the absolute cap is approximate (1000 +/- 10) and document it.
- **Pre-loaded code pool partitioned by hash.** Pool codes are split across the same Redis shards as the counter. Each shard holds N/shard_count codes. `LPOP` from the shard mapped to the user's hash.
- **Multi-region with per-region authority.** A code is "owned" by one region. Redemption of a code on the wrong region routes through to the owning region. The cross-region call is rare (most users redeem in their home region) and adds ~100ms which is acceptable. Cross-region traffic is logged for fraud signals.
- **Audit hash chain** on the redemptions table. Each row stores `prev_hash = SHA256(prev_redemption.canonical)` and `hash = SHA256(this.canonical + prev_hash)`. Finance auditors can verify the chain has not been tampered with. Required for SOX compliance at this scale.
- **Bulk campaign update tool.** Allows admin to update campaign discount metadata without touching the `redeemed` snapshot in each redemption (the snapshot is what the cart actually applied; it must never change). For codes still unredeemed, the new discount takes effect on next redeem.
- **Per-segment discount via campaign rules.** Campaign defines `discount_rules: [{audience: "vip", discount: 20%}, {audience: "default", discount: 10%}]`. The validate path evaluates the user's segment and returns the matching discount. The redemption snapshot records which rule fired.
- **CDC pipeline from Postgres to analytics.** Redemption events stream to ClickHouse for marketing dashboards. Finance reconciliation runs nightly: sum of redemptions per campaign vs Redis counter, alert on drift.

**Why this is enough:**

Even at 1M users, the volume is small by absolute terms (~10 QPS sustained, 100k QPS peak burst). The architecture has not fundamentally changed since stage 3: regions, sharding, audit chain, RBAC. The core insight (Postgres unique index = safety net, Redis Lua = burst path, Bloom filter = brute-force shield) carries through unchanged.

**Cost:** ~$10-30k/month depending on regions and audit volume.

#### What you would not build at 10M+ users

At 10M users you are a top-10 e-commerce platform. At that point coupon redemption is a real product (think Shopify Discounts, or the discount engine inside Amazon). The conversation shifts from "build a coupon system" to "build a discount engine that also handles bundles, tiered pricing, A/B testing of promotions, machine-learned personalized offers." Different problem.

### 9. Reliability

**Redis dies mid-redemption.**

Scenario: Lua script returned OK, decremented the counter, added user to the set. Service crashes before the Postgres write. Redis later loses state (failover with stale replica).

Outcome: ~5 redemptions exist in Redis but not in Postgres. Counter shows N-5; reality is N.

Mitigations:
1. **Synchronous Postgres write before returning success.** Adds 5-10ms. Worth it for correctness.
2. **Postgres unique index is the ground truth.** If Redis is rebuilt from scratch (warm-start), the rebuild process queries Postgres for the current redeemed count and reseeds the Redis counter. The user-set in Redis is similarly seeded from Postgres on warm-start.
3. **Periodic reconciliation.** A nightly job compares Redis counter vs Postgres count. Drift > 10 alerts the on-call.

**Postgres primary fails.**

Failover to read replica takes 30-60 seconds. During failover, redeem is unavailable. Validate still works against the replica (which is now primary).

The Idempotency-Key cache in Redis means users who retry after failover get a clean response (either their previous successful redemption, or a fresh attempt against the new primary).

**Payment fails after redemption.**

User redeemed BLACKFRI100, the cart applied 100% off, then payment failed at checkout.

Two policies, choose at campaign-creation time:
- **`release_on_failure = false` (default).** The redemption stands. The user lost their chance. They are told "your discount has been consumed; please contact support."
- **`release_on_failure = true`.** The redemption is released. Mechanism: cart service emits `payment.failed` on the order; coupon service consumes it; if `order_id` matches a redemption, mark `released_at = NOW()` and run the release procedure.

Release procedure: set `released_at` on the redemption row (the partial unique index `WHERE released_at IS NULL` makes the user free to redeem again). For pool codes, set the `codes` row state back to `unused`. For shared counters, increment the Redis counter. Each step is idempotent.

The senior answer: "Default is `release_on_failure = false` because release introduces a parallel race (someone else can claim the released code while the original user is debugging payment). For high-value campaigns where customer goodwill matters, opt in to release."

**Payment succeeds but user requests refund (next day).**

Same release procedure, gated by an admin or automated rule. Marketing decides per-campaign whether refund releases the code. Most do not.

**Kafka is down.**

The redemption succeeds in Postgres. The cart does not receive the event. The cart polls the coupon service periodically for redemptions tied to active carts, or the user re-applies the code in the cart UI (the validate endpoint returns "valid: true, already_redeemed_for_this_order: true" with the discount).

The fallback path is essential. A Kafka outage should not block checkout.

### 10. Observability

| Metric | Why it matters |
|--------|----------------|
| `redeem.latency` p50/p95/p99 | Headline SLO; alert if p99 > 200ms at burst |
| `redeem.success.rate` per campaign | Drops indicate exhaustion or backend errors |
| `redeem.attempts.rate` per campaign | Sudden 100x spike triggers velocity pause |
| `redeem.result.distribution` | `success`, `exhausted`, `already_redeemed`, `audience_mismatch`, etc. Drift tells you about UX bugs |
| `validate.cache_hit_rate` | Should be > 90%; below = cache too small or invalidation too aggressive |
| `bloom_filter.hit_rate` | Tracks the brute-force shield; spikes indicate scraper traffic |
| `redis.counter.drift_vs_db` | Computed nightly; non-zero = a bug or a Redis incident |
| `rate_limit.triggered.count` per user/IP | Top offenders dashboard for the security team |
| `kafka.lag.coupon_redeemed` | Cart applies discount with this lag; alert at > 30s |
| `audit_chain.verification.failed` | If non-zero ever, someone tampered with the redemption log; page everyone |

Alerts:
- Page: redeem error rate > 5% for 5min; redis-postgres drift > 100; audit chain failure.
- Ticket: validate cache hit rate < 80%; bloom filter rebuild stuck; rate limit triggers > 10x daily baseline.

### 11. Common gotchas

1. **Race condition on the hot key.** Two requests arrive in the same millisecond for the last available slot. Without atomic claim (Lua or `FOR UPDATE SKIP LOCKED`), both succeed and the campaign goes over cap. The unique index on `(campaign_id, user_id)` only protects against the same user racing themselves, not against two different users racing for the last slot. The atomic claim is non-optional.
2. **Codes leaked publicly.** Shared codes on a deal forum is a feature of how the internet works. Defenses are audience filter, unique per-user codes, velocity-based pause, and a sensible cap. There is no perfect prevention.
3. **Double redemption from retry.** Network failed after the redemption succeeded; client retried. Without Idempotency-Key, the retry consumes a second slot. With it, the retry returns the original response. Required.
4. **User-agent fingerprinting for abuse.** Same browser fingerprint creating ten accounts and each trying ten codes is a signal. Combine fingerprint with IP and user account for the abuse detector. Real protection is layered; fingerprinting alone is bypassable.
5. **Time zone in expiry.** "Midnight Dec 31" is ambiguous. Store `expires_at` as a UTC timestamp. Display in the user's local timezone. Always reject in the API based on UTC comparison.
6. **The `released_at` race.** When you release a code due to refund, a third user can immediately re-claim it. If the original refund is itself reversed (rare but possible), you have a code claimed twice. Mitigation: never release after T hours from original redemption; after T hours, treat the redemption as permanent regardless of order status.
7. **Cart applies the wrong discount snapshot.** The user redeems at 9am with a 100% discount. At 10am, marketing edits the campaign to 50%. The cart re-validates the redemption and now sees 50%. Bug: the cart should use the `discount_applied` snapshot from the redemption row, not re-query the campaign. The snapshot is the contract; the campaign is the rule for *future* redemptions.
8. **Bloom filter staleness.** Campaigns created after the filter was built are not in the filter. Mitigation: when a campaign is created, broadcast a "rebuild filter" event to all service pods, or include new codes in an additional small in-memory set checked alongside the main filter.
9. **Per-user limit > 1.** If `per_user_limit = 5`, the unique index `(campaign_id, user_id)` is wrong; it allows only 1. Solution: change to `(campaign_id, user_id, redemption_seq)` where redemption_seq is an integer that increments per user per campaign. Enforce the limit in the service code with a count check before insert.
10. **POS / offline redemption.** A retail terminal goes offline, accepts a code, then syncs at 11pm. By then the same code may have been used online. Resolution requires per-channel policy: either "online wins" (POS redemption gets reversed) or "first-sync wins" (whoever syncs first to the central service wins; the other gets a chargeback). Pick one before you ship offline support; do not let it be ambiguous.

### 12. Follow-up answers

**1. Redis decremented but Postgres did not record; user retries.**

The Idempotency-Key in Redis (5min TTL) catches it. The retry's key matches the original; the cached response is returned. The user does not see the failure; the system effectively recovered.

If the Idempotency-Key cache also lost the entry (Redis failover), the retry hits Redis Lua again. The user is already in the user-set, so the script returns `already_redeemed` with the original redemption_id (looked up from Postgres). Idempotent.

The bad case: Idempotency-Key cache lost, *and* Postgres lost the row (which should never happen with a healthy DB). Then the user successfully redeems a second time. Mitigation: synchronous Postgres write before returning success. This is why we accept the 5-10ms cost of the synchronous insert.

**2. 1000 codes in the campaign; 1003 in Postgres after launch.**

Possible causes:
- A bug in the release-on-refund flow: 3 codes were released and re-claimed; the original 3 plus the 3 re-claims = 1003 rows in redemptions, but only 1000 with `released_at IS NULL`. Not actually a problem if you count active redemptions correctly.
- A real bug: the unique index was misconfigured (`WHERE released_at IS NULL` was missing), allowing two active rows per user.
- A real bug: Redis and Postgres drifted because synchronous Postgres writes failed silently and Redis kept going. The campaign exhausted in Redis after 1000, but those 3 extra were ones that succeeded in Postgres after a Redis stutter.

Detection: nightly reconciliation job comparing `SELECT COUNT(*) FROM redemptions WHERE campaign_id = ? AND released_at IS NULL` against the Redis counter. Alert on any drift.

Prevention: synchronous Postgres write before returning success; the unique index is the backstop. The campaign should also have a Postgres-side hard cap: `INSERT INTO redemptions ... WHERE (SELECT COUNT(*) FROM redemptions WHERE campaign_id = ?) < total_codes`. This is expensive at burst (the count query is the bottleneck), so use it only as a periodic safety check, not in the hot path.

**3. Stacking SAVE10 + FREESHIP + BLACKFRI100.**

The cart holds the list of applied coupons. Validate is called per coupon and is told what other coupons are in the cart.

Coupon-side logic:
- Each campaign has `stacking_rules`: `{"stackable_with": ["FREESHIP"], "excludes": ["BLACKFRI*"]}` (allowlist + denylist).
- Validate computes: do the existing cart coupons satisfy this coupon's rules?
- If BLACKFRI100 is in the cart and you try to validate SAVE10, the stacking check fails: "not_stackable_with_blackfri".

Cart-side logic:
- The cart enforces a max number of coupons per cart (e.g., 3) regardless of stacking rules.
- The cart computes the final discount: if both are percentages, decide compound (1 - 0.1)(1 - 0.5) vs sum 10% + 50% = 60% off (clamped at 100%). This is a business decision per campaign; default is sum, capped at 100%.

The coupon service is the source of truth for stacking *rules*. The cart applies the rules. The split is important because the cart already has the context (other coupons, totals).

**4. Refund releases the code, or not?**

Per-campaign policy. Default: no release.

If release is on:
- `payment.refunded` event triggers the release procedure.
- Set `released_at` on the redemption.
- For pool: set the code row's state back to `unused` and push the code back to the Redis list.
- For shared: `INCR` the Redis counter.
- For unique per-user: the user can now re-redeem the same code.

The hidden cost: a released code creates a parallel race. Someone else can claim it within milliseconds. Customer service hates this when the refunding customer asks "can you give me that code back so I can use it tomorrow."

The cleaner answer: do not release. Refund the money. The discount is gone. If the customer is high-value, support manually issues a courtesy code from a separate budget. This avoids the race entirely.

**5. Expiration in the wrong timezone.**

Store `expires_at` as a UTC timestamp. Compare against UTC `NOW()`. Done.

For UI: render in the user's local timezone. "Expires in 3 hours" is better than "Expires at 2026-12-31 23:59:59 UTC" because it removes the timezone math from the user's brain.

For marketing: when they say "midnight Dec 31", ask which timezone. The campaign creation form requires a timezone selector. Default to the company's headquarters timezone but never assume.

The Twitter-yelling avoided: a user in Tokyo who sees "expires in 3 hours" at 11pm Tokyo time gets the same UTC-anchored expiry as a user in LA. The math is unambiguous. The wording is local. Everyone happy.

**6. Mass-update 10M codes without invalidating already-redeemed ones.**

The redemption snapshot is immutable; that is the contract. Updating the campaign's `discount` field does not retroactively change what past redemptions applied; the cart uses the snapshot in each redemption row.

For unredeemed codes, the campaign update takes effect on next redeem. The `codes` table does not store the discount per code; it joins to the campaign. So updating the campaign updates every future redemption from any of its codes.

If the discount is wrong on already-redeemed orders (e.g., wrong percentage applied), that is an accounting/refund problem, not a coupon-system problem. The coupon-system change just stops the bleeding for new redemptions.

Code path: `UPDATE campaigns SET discount = ? WHERE campaign_id = ?`. One row. Fast. Cache invalidation event broadcast to all service pods to drop the cached campaign metadata.

**7. Multi-region single-use guarantee.**

Each code is "owned" by exactly one region at creation time (typically the region whose campaign created it). The owning region's Postgres holds the authoritative redemption row.

If a code is redeemed in a non-owning region:
- The non-owning region's service detects the mismatch (campaign metadata says "owned by us-east").
- It forwards the redeem call via authenticated cross-region API to us-east.
- us-east performs the atomic claim against its Postgres and Redis.
- The response is returned across regions to the user.

Latency cost: ~100ms cross-region. Acceptable for the rare cross-region case.

Per-user uniqueness across regions: the unique index on `(campaign_id, user_id)` lives only in the owning region. Cross-region calls all funnel into it. The unique index does the work.

Alternative: replicate the unique index globally using a strongly-consistent multi-region database (Spanner, DynamoDB Global Tables with conditional writes). Higher cost, simpler model. Worth it if cross-region redemptions are >10% of traffic; not worth it for the typical 1%.

**8. Bloom filter has no false negatives.**

Correct. A Bloom filter never says "definitely not present" for something that was actually added; if a code was inserted into the filter, it will always be reported as "maybe present."

What Bloom filters *do* have is **false positives**: occasionally saying "maybe present" for a code that was never inserted. This is fine for our use case: a false positive sends the request through to the cache/DB, which correctly returns 404. No harm done; just a tiny bit of leaked load.

In this design, the filter is sized for 0.1% false-positive rate at 200M items. That means out of every 1000 brute-force attempts on bogus codes, ~999 are rejected at the filter and ~1 falls through to the DB. At a brute-force rate of 5000 attempts/sec, the DB sees ~5 lookups/sec from this source. Trivial.

What you must not do: rely on the filter to *prove* a code exists. You must still validate against the DB on a maybe-present hit, because the filter is approximate.

**9. SAVE10 has 9999 of 10000 uses; 20 users hit simultaneously.**

The Redis Lua script handles this. Each of the 20 requests enters the script serially (Redis is single-threaded). The first sees `remaining = 1`, decrements to 0, returns OK. The other 19 see `remaining = 0` and return `exhausted`.

The 1 winner's Postgres write proceeds normally with the unique index check (they have not redeemed before). The 19 losers see 410.

If the per-user-limit is 1 and any of the 20 users had already redeemed SAVE10 previously, the Lua script also rejects them with `already_redeemed` regardless of the counter, returning 409.

**10. Unredeemed code after expiry: reuse the string?**

No. Codes are append-only forever, even after expiry.

Reasons:
- The codes table has a unique index on the code string. Reuse would require deleting the expired row, which loses audit history.
- A user might bookmark the code or have it in an email. Reusing the string for a new campaign creates confusion ("why does my SAVE10 from last year suddenly apply 25% off?").
- Fraud signal: trying an expired code is a useful pattern. Reusing the string erases that signal.

What you can do: design the code namespace to be effectively infinite. 10 characters in base32 = ~10^15 codes. You will never run out. Just mint a new string for each new campaign.

For pool campaigns where unclaimed codes pile up, the cost is the storage of those expired rows. At ~200 bytes each, 1M expired unredeemed codes is 200MB. Negligible. Optionally, archive expired codes to cold storage after 1 year.

### 13. Trade-offs and what a senior would mention

- **Why Postgres unique index *and* Redis Lua.** Either alone is a single point of failure. The unique index alone melts under 10k QPS on one row. Redis Lua alone loses correctness if Redis drops state. Together they form a fast hot path with a strong safety net.
- **Why not just a sequence-backed counter in Postgres.** A `SERIAL` column gives unique IDs but does not enforce the cap (the cap is a count check, not a uniqueness check). And the row-level contention is still there. The unique index on `(campaign_id, user_id)` is the right primitive.
- **Why Bloom filter, not a Redis SET membership check.** Bloom filter is in-process, sub-microsecond, fixed memory. A Redis SISMEMBER is a network round-trip (1ms) and grows linearly with the number of issued codes. At 200M codes, the SET would be ~4GB in Redis; the Bloom filter is ~300MB in each pod's process memory. Bloom wins on cost and latency.
- **Why an immutable redemptions log, not just "current state."** Finance auditors, fraud investigators, and refund workflows all need the history. Mutating in place destroys traceability.
- **What I would revisit at 10x scale.** Move to a discount engine (rules-based, not just code-based). Pre-personalize coupons per user. Add A/B testing of discount values. The simple coupon model breaks once marketing wants to ask "what if we showed user X a 15% code and user Y a 10% code based on their cart value."

### 14. Common interview mistakes

1. **Diving straight into a `coupons` table without thinking about concurrency.** The whole question is about the 10k-QPS burst. If your design's first sentence is "we'll have a coupons table with a `times_used` column," you have missed the point. The interviewer will steer you to the race condition; you should arrive there yourself.
2. **Using Redis without a Postgres backstop.** Redis can lose state. If you describe Redis as the source of truth and shrug at the failure modes, you fail the senior bar.
3. **Forgetting the per-user uniqueness check.** Some candidates correctly atomic-claim the campaign counter and forget that the same user can race themselves with two browser tabs. The unique index on `(campaign_id, user_id)` is non-optional.
4. **No idempotency on redeem.** Without Idempotency-Key, every network retry burns a code. At burst with flaky networks, this is catastrophic.
5. **Conflating validate and redeem.** A POST that always consumes is a UX disaster (no preview of the discount). A GET that consumes violates HTTP semantics and breaks browser pre-fetch. Two endpoints, clear separation.
6. **Ignoring brute-force.** Coupon namespaces get scraped. If you do not mention rate limiting and the Bloom filter (or some equivalent), you have not thought about the abuse model.
7. **Hand-waving release-on-refund.** "Yeah, we'll just put it back" creates a parallel race the candidate has not thought through. The senior answer either says "we don't release" with reasons, or describes the release race explicitly.
8. **No mention of the discount snapshot.** The cart applies the discount as it existed at redeem time, not as it exists now. Without the snapshot in the redemption row, marketing can retroactively change discounts on past orders, which breaks accounting.
9. **Designing for write throughput.** Even at 1M users, sustained QPS is single digits. The design pressure is the launch burst and the abuse traffic, not steady state. Candidates who size for "millions of writes per second" have misread the problem.
10. **Skipping the audit log.** Finance and fraud need it. The redemption table is your audit log; an immutable schema with a hash chain (at scale) is the senior addition.

If you can hit 7 of these 10, you are interviewing at the senior level. The launch-burst correctness story is what separates strong candidates from generic "design a CRUD app" answers. Most candidates miss the per-user-race issue and the importance of the Postgres backstop behind the Redis hot path. Those two are the most common drop-points in interviews on this problem.
