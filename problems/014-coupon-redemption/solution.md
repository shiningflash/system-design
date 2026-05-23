## Solution: Coupon Code Redemption System

### TL;DR

A coupon system is a small, write-light service with one nasty bit: surviving a launch burst where 10,000 users hit the same code in the same second and exactly 1000 of them must win. Everything else is plumbing.

The design has two layers. Postgres holds the source of truth, with a unique index on `(campaign_id, user_id)` that makes double-redemption impossible at the storage level. Redis with a Lua script handles the hot-burst claim path so the database is not asked to serialize 10,000 transactions on one row. Synchronous writes from Redis to Postgres keep the two in sync; the unique index is the safety net when they disagree.

Three code patterns coexist under one schema: generic shared codes (`SAVE10`), unique per-user codes (`UID-XXXX`), and pre-generated pools (`BLACKFRI-XXXX`). All three use the same redeem API; the difference is in how codes are minted and how the claim is structured. A Bloom filter in front of the read path stops brute-force traffic from reaching the DB. Rate limits stop the same user from grinding the namespace.

The interesting engineering sits at three edges: making the launch burst correct without a database stampede, defining what "release the code back to the pool on refund" means without creating a parallel race, and laying down an audit trail finance can use for accounting years later.

### 1. Clarifying questions, recap

The most important question is the first one: *single-use or reusable?* They have completely different data models and contention shapes. Almost as important: which of the three code patterns (shared, unique, pool) the system supports. If you assume "just one pattern" and the interviewer wanted all three, you redesign mid-interview.

Everything else (stacking, expiration, refund, abuse) follows from those two answers.

### 2. Capacity, with the math

The launch burst is the design pressure point. 10,000 redeem requests in 1 second, of which 1000 must succeed and 9000 must be told "sold out". One hot key, one hot campaign. The 9000 losers are the design pressure, not the 1000 winners. They all need a fast response, not a timeout.

Steady redemption is trivial. 50k per day works out to about 0.6 per second sustained, 5 per second at peak business hours. Validate calls run 5x that: 3 per second sustained, 25 per second at peak. Plus a burst of 50k validate calls at 9am launch when users paste the code into the cart before clicking redeem.

Storage barely registers. 200M per-user codes at ~200 bytes each is ~40GB. Add 9GB of redemption log over 5 years. ~50GB total. One Postgres instance.

Hot working set during launch is one campaign at a time. The campaign row, the counter, and (for pool campaigns) the unclaimed code list. ~100KB to a few MB. Fits in any cache.

The system is small. The architecture exists for *burst correctness and abuse resistance*, not throughput or storage.

### 3. API

Validate and redeem are separate endpoints. Validate is read-only and cheap. Redeem is the atomic act. Treating them as one endpoint (POST that always consumes) breaks the cart UX where users want to see the discount before clicking checkout.

```
POST /api/v1/coupons/validate
Authorization: Bearer <token>     # optional; required for audience-restricted codes
Content-Type: application/json

{
  "code": "BLACKFRI100",
  "cart_id": "cart_abc",
  "subtotal": 12000                  # cents
}
```

| Status | Meaning | Body |
|--------|---------|------|
| 200 OK | Code valid for this user/cart | `{"valid": true, "discount": {"type": "percent", "value": 100, "max_off": 50000}, "stackable_with": ["FREESHIP"], "expires_at": "..."}` |
| 200 OK | Valid in isolation but fails stacking | `{"valid": false, "reason": "not_stackable_with_SAVE10"}` |
| 404 Not Found | Code does not exist | `{"valid": false, "reason": "unknown_code"}` |
| 410 Gone | Code expired or fully redeemed | `{"valid": false, "reason": "expired" / "exhausted"}` |
| 403 Forbidden | User not in audience | `{"valid": false, "reason": "audience_mismatch"}` |
| 429 Too Many Requests | Too many validate attempts | `{"reason": "rate_limited", "retry_after": 60}` |

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

| Status | Meaning | Body |
|--------|---------|------|
| 201 Created | Redemption succeeded | `{"redemption_id": "rdm_...", "code": "BLACKFRI100", "discount_applied": {...}}` |
| 200 OK | Idempotent: same Idempotency-Key seen before | same shape as 201 |
| 409 Conflict | User already redeemed this code | `{"error": "already_redeemed", "redemption_id": "rdm_..."}` |
| 410 Gone | Code is exhausted or expired (lost the race) | `{"error": "exhausted"}` |
| 403 Forbidden | Audience mismatch | `{"error": "audience_mismatch"}` |

Two reads round it out. `GET /api/v1/users/{user_id}/redemptions?status=active&limit=50` for the user's history, and an admin `POST /api/v1/admin/campaigns` to create a campaign:

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

A few small but load-bearing choices. Idempotency-Key on redeem is required, not optional; network retries are guaranteed at launch burst and without the key the same user's retry can consume two codes. `order_id` ties the redemption to the order, which you need for the refund/release flow and for finance reconciliation. And the 403 with `audience_mismatch` does not reveal whether the code itself is valid; returning 404 to outsiders prevents harvesting while returning 403 only to authenticated users in the wrong audience is the right balance.

### 4. Data model

Four tables. Two large, two small.

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
    audience_filter  JSONB,
    stacking_rules   JSONB,                            -- {"stackable_with": [...], "excludes": [...]}
    starts_at        TIMESTAMPTZ NOT NULL,
    ends_at          TIMESTAMPTZ NOT NULL,
    status           TEXT NOT NULL DEFAULT 'active',   -- 'active' | 'paused' | 'ended'
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_campaigns_status_window ON campaigns (status, starts_at, ends_at);

-- Codes: one row per individual code string.
--   shared:  one row total; counter tracked on the campaign
--   unique:  one row per (campaign, intended_user)
--   pool:    one row per code, intended_user is NULL until claimed
CREATE TABLE codes (
    code_id          UUID PRIMARY KEY,
    campaign_id      UUID NOT NULL REFERENCES campaigns(campaign_id),
    code             TEXT NOT NULL,                    -- "BLACKFRI100", "BLACKFRI-AB7K2X9P"
    intended_user    TEXT,                             -- non-null for 'unique' type
    state            TEXT NOT NULL DEFAULT 'unused',   -- 'unused' | 'claimed' | 'used' | 'expired'
    claimed_by       TEXT,                             -- user_id who claimed (pool type)
    claimed_at       TIMESTAMPTZ,
    expires_at       TIMESTAMPTZ,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_codes_code ON codes (code);
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

A few choices worth defending out loud.

The `UNIQUE (campaign_id, user_id) WHERE released_at IS NULL` index on redemptions is doing real work. When the same user races themselves with two browser tabs, the database serializes them. The first insert wins, the second fails with a unique-violation, the API turns that into a friendly 409. This is the safety net behind whatever Redis says. If you only remember one thing from this design, remember this index.

`campaigns.redeemed_count` is denormalized and async. It is not the source of truth for "is the campaign exhausted." The source of truth for that is `COUNT(*) FROM redemptions WHERE campaign_id = ?`, but we never query that on the hot path; we trust the counter in Redis and reconcile periodically against the table.

`redemption_attempts` logs everything, including failures. Required for fraud signals (50 attempts/sec from one IP) and for the abuse-prevention dashboards. Partitioned by month in production.

Postgres, not Cassandra or DynamoDB. Strong consistency for the unique index. Transactional semantics for the claim. JSONB for the flexible discount and audience shapes. Cassandra would force you to invent compensating logic for the same guarantees, and the data volume (50GB at 5-year retention) gives Postgres no scaling problem to solve.

### 5. Core algorithm: the atomic claim

Two paths, picked per campaign.

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

`FOR UPDATE SKIP LOCKED` is the key. Without `SKIP LOCKED`, concurrent transactions queue up on the first unused row; with it, each picks a different unused row. Throughput is a few hundred claims per second on a single Postgres on this exact query, limited by row-level locking on the campaign's row set.

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

After Redis returns OK, the service writes the redemption to Postgres synchronously (~5ms). Total redeem latency: ~6ms. The Postgres write uses the same `ON CONFLICT (campaign_id, user_id)` clause as the Postgres-only path. If Redis hiccupped and somehow let two requests through for the same user, the DB unique index catches the second one. The service handles this case by rolling back the Redis state (re-adding the user to the set, incrementing the counter) and returning 409 to the second user.

Picking between them:

| Campaign expected QPS | Approach |
|----------------------|----------|
| < 100 | Postgres only |
| 100 to 10,000 | Redis + Postgres backstop |
| > 10,000 | Redis + Postgres with sharded Redis (counter and user-set partitioned by hash of user_id) |

The hybrid in production: every campaign uses Redis. For low-traffic ones the Redis cost is negligible and the consistent code path is worth it. For high-traffic ones the Redis layer is what keeps the DB alive.

Before validate even queries Redis or the DB, a Bloom filter check.

```python
def validate(code):
    if not bloom_filter.maybe_contains(code):
        return 404  # definitely not a real code
    # falls through to Redis/DB
```

Bloom filters never have false negatives. If the filter says "not present", the code definitely was never issued. They have a tunable false-positive rate (~0.1% is typical). At 0.1%, a brute-force scraper sees 99.9% of attempts cut off at the filter, never touching the DB.

The filter is rebuilt when a campaign is created (add all new codes) and reloaded on service start. Memory footprint is ~12 bits per code at 0.1% FPR. 200M codes is ~300MB. Fits in process memory.

### 6. Architecture

Here is the whole picture. Drawn small enough to fit one screen.

```
              ┌────────────────────────────────────────────────┐
              │ Clients: web, mobile, POS terminal             │
              └─────────────────────┬──────────────────────────┘
                                    │
                                    ▼
                         ┌──────────────────┐
                         │  API Gateway     │ auth, per-user + per-IP
                         │  + Rate Limiter  │ rate limit, WAF
                         └────┬────────┬────┘
                  validate    │        │   redeem
                              │        │
                              ▼        ▼
              ┌─────────────────────┐ ┌──────────────────────┐
              │  Coupon Service      │ │  Coupon Service       │
              │  (read, stateless)   │ │  (write, stateless)   │
              │                      │ │                       │
              │  ┌─ Bloom filter ────┘ │  ┌─ Idempotency key  │
              │  ├─ in-proc LRU       │  ├─ Redis Lua claim  │
              │  └─ cache lookup      │  └─ Postgres write   │
              └─────────┬─────────────┘ └──┬───────┬──────────┘
                        │                  │       │
                        ▼                  ▼       │
              ┌──────────────────┐   ┌──────────────────┐
              │  Read Cache       │   │  Redis (hot      │
              │  (Redis)          │   │  burst)           │
              │  campaign meta,   │   │  remaining counter│
              │  60s TTL + jitter │   │  user-set, pool   │
              └──────────────────┘   └────────┬──────────┘
                                              │
                                              ▼
              ┌──────────────────────────────────────────────┐
              │  Postgres (source of truth)                  │
              │   campaigns, codes,                          │
              │   redemptions (unique on (campaign,user)),   │
              │   redemption_attempts                        │
              └────────────────┬─────────────────────────────┘
                               │
                  CDC / outbox │
                               ▼
              ┌──────────────────────────────────────────────┐
              │  Kafka                                        │
              │   coupon.redeemed                             │
              │   coupon.released                             │
              └──┬──────────────┬──────────────┬─────────────┘
                 │              │              │
                 ▼              ▼              ▼
        ┌─────────────┐  ┌──────────────┐  ┌────────────────┐
        │ Cart Svc    │  │  Fraud Svc    │  │ Analytics +     │
        │ apply       │  │  velocity     │  │ Finance recon   │
        │ discount    │  │  signals      │  │ (ClickHouse)    │
        └─────────────┘  └──────────────┘  └────────────────┘
```

Five things to notice while reading this.

The write path touches Redis and Postgres in one transaction window. Nothing else is in the synchronous hot path. Kafka, the cart service, fraud, analytics are all downstream of a CDC stream from Postgres. If the cart is down, redemptions still succeed; the discount catches up when the cart comes back.

The Bloom filter and in-process LRU live in the read service, not in Redis. Process memory is the cheapest, fastest tier. Redis is the next tier. Postgres is the floor.

Hot-burst Redis and read-cache Redis are different roles, often on different clusters. The hot-burst one runs Lua scripts and serializes campaigns on a single core; the read cache holds campaign metadata for thousands of campaigns and scales horizontally with sharding.

The Postgres unique index is the ground truth. Every other layer can lose data and the system recovers. If the unique index loses data, you have a real correctness bug.

The Coupon Service pods are stateless and horizontally scalable. The Bloom filter is rebuilt on startup from a snapshot in S3 plus a tail of `campaign_created` events from Kafka.

### 7. What a redeem looks like, end to end

Here is the redeem-on-hot-campaign flow drawn as a sequence:

```
   Client      API GW     Coupon Svc      Redis         Postgres        Kafka
     │           │            │              │              │             │
     │ POST      │            │              │              │             │
     │ /redeem   │            │              │              │             │
     ├──────────►│            │              │              │             │
     │           │ rate limit │              │              │             │
     │           │ check      │              │              │             │
     │           ├───────────►│              │              │             │
     │           │            │ idem-key     │              │             │
     │           │            │ lookup       │              │             │
     │           │            ├─────────────►│              │             │
     │           │            │◄─────────────┤              │             │
     │           │            │              │              │             │
     │           │            │ bloom + cache│              │             │
     │           │            │ (in-proc)    │              │             │
     │           │            │              │              │             │
     │           │            │ EVAL Lua     │              │             │
     │           │            │ (check       │              │             │
     │           │            │  counter,    │              │             │
     │           │            │  user-set,   │              │             │
     │           │            │  decr, sadd) │              │             │
     │           │            ├─────────────►│              │             │
     │           │            │◄─────────────┤              │             │
     │           │            │   ok         │              │             │
     │           │            │              │              │             │
     │           │            │ BEGIN TX                    │             │
     │           │            ├─────────────────────────────►             │
     │           │            │ INSERT redemptions          │             │
     │           │            │   ON CONFLICT DO NOTHING    │             │
     │           │            │ UPDATE codes (pool only)    │             │
     │           │            │ INSERT redemption_attempts  │             │
     │           │            │ COMMIT                      │             │
     │           │            │◄─────────────────────────────             │
     │           │            │              │              │             │
     │           │            │ cache idem-key response     │             │
     │           │            ├─────────────►│              │             │
     │           │            │              │              │             │
     │           │            │ emit coupon.redeemed (CDC)  │             │
     │           │            │              │              ├────────────►│
     │           │ 201 OK     │              │              │             │
     │           │◄───────────┤              │              │             │
     │ 201 OK    │            │              │              │             │
     │◄──────────┤            │              │              │             │
     │           │            │              │              │  consumer   │
     │           │            │              │              │             ├──► cart applies
     │           │            │              │              │             ├──► fraud signal
     │           │            │              │              │             └──► analytics
```

Validate has its own short flow. Client hits the gateway, the request goes to the read service, the Bloom filter rejects most bogus codes immediately, the in-process LRU absorbs most real-code reads, cache misses fall through to Redis and then Postgres. Whichever hits, the result is fast.

Target latencies, for a sense of scale: validate P99 around 50ms (most calls are cache hits at ~5ms). Redeem P99 around 50ms even at the launch burst (rate limit ~1ms, Redis Lua ~1ms, Postgres insert ~5-10ms, the rest is network).

### 8. Scaling journey: 10 to 1M users

At every stage, name what just broke and what fixes it. Build nothing preemptively.

**Stage 1: 10 to 100 users.** One Postgres (db.t3.small), one app instance, the three tables with the unique index on `(campaign_id, user_id)`. Validate and redeem are both straight DB calls in a transaction. No Redis, no cache, no Kafka. The cart calls the coupon service inline. About $80/month total.

Ten redemptions per day. Postgres yawns. The unique index alone is the entire correctness story. No campaign sees more than 1 QPS, no race conditions of concern. Building anything more is over-engineering.

**Stage 2: 1k users.** Something breaks: a user shared a code on a deal forum, 200 anonymous attempts hit in 5 minutes, the DB spiked to 50% CPU. Marketing started running per-user codes (2000 rows in `codes`); lookups by code string got slow without an index, so you added `CREATE UNIQUE INDEX idx_codes_code ON codes (code)` and the issue vanished. A cart bug caused some users to submit redeem twice in quick succession; the unique index caught the double-claim correctly but the second request got a confusing 500. You added `ON CONFLICT DO NOTHING RETURNING` handling and started returning 409 instead.

You added per-user rate limiting in the gateway (token bucket in Redis, 10 validate/min and 5 redeem/min per authenticated user), per-IP rate limiting for unauthenticated traffic, and an in-process LRU for campaign and code lookups (1000 entries, 60s TTL, dropped DB load ~5x). The `redemption_attempts` table started filling, partitioned monthly, feeding a "top 10 IPs by failed-validate rate" dashboard. Idempotency-Key on redeem became required, stored in Redis with 5min TTL, and the cart UI bug stopped causing problems. About $250/month.

Still no Bloom filter, still no Redis for the atomic claim. At 1k users you have at most a few thousand distinct code attempts per day; the cache absorbs most. Postgres handles all redeems directly with `FOR UPDATE SKIP LOCKED`. Max contention is on the hottest campaign at maybe 10 QPS. No Kafka. The cart calls the coupon service inline.

**Stage 3: 100k users.** Now several things break at once. The first real flash sale: 2000 codes for FLASH50 at noon, 5000 users at once, Postgres CPU pegged for 30 seconds because `FOR UPDATE SKIP LOCKED` serialized on the `(campaign_id, state = 'unused')` index, most requests timed out at the gateway, marketing was furious. A scripted brute-force across the SAVE-prefix namespace fired 5000 attempts/sec from rotating IPs; the LRU helped on hits, but the guaranteed-misses on bogus codes hit the DB at ~1000 QPS. The cart started showing race conditions when a user redeemed, the cart applied the discount, then the user removed an item that made the discount no longer applicable.

The fixes, in order. Redis with Lua for atomic claims on hot campaigns (campaign marked `hot` at creation time or auto-flagged at expected_qps > 100); the Lua script in section 5 does the work; Postgres backstop with the unique index still catches drift. For pool campaigns, pre-load unclaimed codes into a Redis list; `LPOP` is sub-millisecond. Bloom filter prefilter loaded at service start, updated on campaign create; bogus codes return 404 in microseconds without touching Redis or DB. Synchronous write to Postgres for redemptions on hot campaigns (the unique index catches any double-write). Kafka topic `coupon.redeemed`; the cart consumes the event and applies the discount idempotently (uses `redemption_id` as the dedup key). Read replica for the validate path; primary stays write-only. Velocity-based campaign pause: 100x spike in 1 minute over the trailing-hour baseline auto-pauses and pages the on-call.

Flash sales now serve 10k QPS on the hot campaign on one Redis core. Bloom filter cuts brute-force DB load by ~99.9%. The Postgres unique index is still the safety net. About $1-2k/month.

**Stage 4: 1M users.** New problems. The hottest Black Friday campaign saturated one Redis core at 50k QPS; latency went from 1ms to 50ms. EU operations launched; EU users redeem codes on the EU site, and the codes need to be valid across both regions, but cross-region replication of Redis state introduces ambiguity about who got the code. A pre-generated code minting bug inserted 10 million pool codes with the wrong discount value; bulk fixup was needed without invalidating the 50,000 already-redeemed ones. Marketing wanted per-segment campaigns: same code SAVE10 meaning 20% off for VIPs and 10% off for everyone else.

Sharded Redis for ultra-hot campaigns: counter and user-set partitioned by hash(user_id) across N nodes, each holding 1/N of the cap. A user's redeem routes to the node owning their hash. Caveat: "remaining" is per-shard, so if one shard exhausts while others have slots, late users to the wrong shard see "exhausted" while slots exist elsewhere. Route losers to a fallback shard once, or accept that the cap is approximate (1000 ± 10) and document it.

Pre-loaded code pool partitioned by the same shards. Multi-region with per-region authority: a code is owned by one region, redemption on the wrong region routes through to the owning region. Cross-region call adds ~100ms which is fine because most users redeem in their home region. Audit hash chain on the redemptions table: each row stores `prev_hash = SHA256(prev_redemption.canonical)` and `hash = SHA256(this.canonical + prev_hash)`. Finance auditors can verify the chain. Required for SOX at this scale.

Bulk campaign update tool that updates the campaign's `discount` metadata without touching the snapshot in each redemption (the snapshot is what the cart actually applied; it must never change). Per-segment discount via campaign rules: `discount_rules: [{audience: "vip", discount: 20%}, {audience: "default", discount: 10%}]`. The validate path evaluates the user's segment and returns the matching discount; the redemption snapshot records which rule fired. CDC pipeline from Postgres to ClickHouse for marketing dashboards. Nightly finance reconciliation: sum of redemptions per campaign vs Redis counter, alert on drift.

Even at 1M users, the volume is small (~10 QPS sustained, 100k QPS peak burst). The architecture has not fundamentally changed since stage 3: regions, sharding, audit chain, RBAC. The core insight (Postgres unique index as safety net, Redis Lua as burst path, Bloom filter as brute-force shield) carries through unchanged. About $10-30k/month depending on regions and audit volume.

At 10M users you are a top-10 e-commerce platform. Coupon redemption becomes a real product (Shopify Discounts, the discount engine inside Amazon). The conversation shifts from "build a coupon system" to "build a discount engine that also handles bundles, tiered pricing, A/B testing of promotions, ML-personalized offers." Different problem.

### 9. Reliability

Redis dies mid-redemption: Lua script returned OK, decremented the counter, added the user to the set; the service crashed before the Postgres write; Redis later lost state in a failover with a stale replica. Outcome: a few redemptions exist in Redis but not in Postgres, the counter shows N minus a few, reality is N.

Three things keep this safe. The synchronous Postgres write before returning success (5-10ms cost, worth it for correctness). The Postgres unique index as ground truth: if Redis is rebuilt from scratch on warm-start, the rebuild queries Postgres for the current redeemed count and reseeds the counter; the user-set is similarly seeded from Postgres. And a periodic reconciliation job that compares Redis counter vs Postgres count, alerting on drift greater than 10.

Postgres primary fails: failover to read replica takes 30 to 60 seconds. During failover, redeem is unavailable. Validate still works against the replica (which is now primary). The Idempotency-Key cache in Redis means users who retry after failover get a clean response, either their previous successful redemption or a fresh attempt against the new primary.

Payment fails after redemption: user redeemed BLACKFRI100, the cart applied 100% off, payment failed at checkout. Two policies, picked at campaign-creation time. Default `release_on_failure = false`: the redemption stands, the user lost their chance, they are told "your discount has been consumed; please contact support." Opt-in `release_on_failure = true`: `payment.failed` triggers the release procedure, which sets `released_at` on the redemption (the partial unique index `WHERE released_at IS NULL` makes the user free to redeem again), and for pool codes sets the `codes` row back to `unused`, and for shared counters increments the Redis counter. Each step is idempotent.

The senior position: default to no release. Release introduces a parallel race where someone else can claim the released code while the original user is debugging payment. For high-value campaigns where customer goodwill matters, opt in.

Refund the next day works the same way, gated by an admin or automated rule. Marketing picks per-campaign whether refund releases. Most do not.

Kafka is down: the redemption succeeds in Postgres but the cart does not receive the event. The cart polls the coupon service periodically for redemptions tied to active carts, or the user re-applies the code in the cart UI (the validate endpoint returns "valid: true, already_redeemed_for_this_order: true" with the discount). The fallback path is essential. A Kafka outage should not block checkout.

### 10. Observability

| Metric | Why it matters |
|--------|----------------|
| `redeem.latency` p50/p95/p99 | The headline SLO; alert if p99 > 200ms at burst. |
| `redeem.success.rate` per campaign | Drops indicate exhaustion or backend errors. |
| `redeem.attempts.rate` per campaign | Sudden 100x spike triggers velocity pause. |
| `redeem.result.distribution` | success, exhausted, already_redeemed, audience_mismatch, etc. Drift tells you about UX bugs. |
| `validate.cache_hit_rate` | Should be > 90%. Below means the cache is too small or invalidation too aggressive. |
| `bloom_filter.hit_rate` | Tracks the brute-force shield. Spikes indicate scraper traffic. |
| `redis.counter.drift_vs_db` | Computed nightly. Non-zero is a bug or a Redis incident. |
| `rate_limit.triggered.count` per user/IP | Top offenders dashboard for the security team. |
| `kafka.lag.coupon_redeemed` | Cart applies discount with this lag. Alert at > 30s. |
| `audit_chain.verification.failed` | If non-zero ever, someone tampered with the log. Page everyone. |

Page on: redeem error rate > 5% for 5min, redis-postgres drift > 100, audit chain failure. Ticket on: validate cache hit rate < 80%, Bloom filter rebuild stuck, rate limit triggers > 10x daily baseline.

### 11. Gotchas the senior interviewer is listening for

Most of these only come out when the interviewer asks "what happens if...". The senior candidate brings them up unprompted.

**Race on the hot key.** Two requests in the same millisecond for the last slot. Without atomic claim (Lua or `FOR UPDATE SKIP LOCKED`), both succeed and the campaign goes over cap. The unique index on `(campaign_id, user_id)` only protects against the same user racing themselves, not against two different users racing for the last slot. The atomic claim is non-optional.

**Codes leaked publicly.** Shared codes on a deal forum is a feature of how the internet works. Defenses are audience filter, unique per-user codes, velocity-based pause, and a sensible cap. There is no perfect prevention.

**Double redemption from retry.** Network failed after the redemption succeeded; client retried. Without Idempotency-Key, the retry consumes a second slot. With it, the retry returns the original response. Required, not optional.

**Time zone in expiry.** "Midnight Dec 31" is ambiguous. Store `expires_at` as a UTC timestamp. Display in the user's local timezone. Always reject in the API based on UTC comparison.

**The `released_at` race.** When you release a code due to refund, a third user can immediately re-claim it. If the original refund is itself reversed (rare but possible), you have a code claimed twice. Mitigation: never release after T hours from original redemption; after T hours, treat the redemption as permanent regardless of order status.

**Cart applies the wrong discount snapshot.** The user redeems at 9am with a 100% discount. At 10am, marketing edits the campaign to 50%. The cart re-validates the redemption and now sees 50%. Bug: the cart should use the `discount_applied` snapshot from the redemption row, not re-query the campaign. The snapshot is the contract; the campaign is the rule for *future* redemptions.

**Bloom filter staleness.** Campaigns created after the filter was built are not in the filter. Mitigation: when a campaign is created, broadcast a "rebuild filter" event to all service pods, or include new codes in an additional small in-memory set checked alongside the main filter.

**Per-user limit > 1.** If `per_user_limit = 5`, the unique index `(campaign_id, user_id)` is wrong; it allows only 1. Solution: change to `(campaign_id, user_id, redemption_seq)` where redemption_seq is an integer that increments per user per campaign. Enforce the limit in service code with a count check before insert.

**POS / offline redemption.** A retail terminal goes offline, accepts a code, then syncs at 11pm. By then the same code may have been used online. Resolution requires per-channel policy: either "online wins" (POS redemption gets reversed) or "first-sync wins" (whoever syncs first wins, the other gets a chargeback). Pick one before you ship offline support; do not let it be ambiguous.

### 12. Follow-up answers

**1. Redis decremented but Postgres did not record; user retries.**

The Idempotency-Key in Redis (5min TTL) catches it. The retry's key matches the original; the cached response is returned. The user does not see the failure; the system effectively recovered.

If the Idempotency-Key cache also lost the entry (Redis failover), the retry hits Redis Lua again. The user is already in the user-set, so the script returns `already_redeemed` with the original redemption_id (looked up from Postgres). Idempotent.

The bad case is when the Idempotency-Key cache is lost *and* Postgres lost the row (which should never happen with a healthy DB). Then the user successfully redeems a second time. Mitigation: synchronous Postgres write before returning success. This is why we accept the 5-10ms cost of the synchronous insert.

**2. 1000 codes in the campaign; 1003 in Postgres after launch.**

Three possible causes. A bug in the release-on-refund flow: 3 codes were released and re-claimed, so 1003 rows in redemptions but only 1000 with `released_at IS NULL`. Not actually a problem if you count active redemptions correctly. A real bug: the unique index was misconfigured (the `WHERE released_at IS NULL` clause was missing), allowing two active rows per user. A real bug: Redis and Postgres drifted because synchronous Postgres writes failed silently while Redis kept going; the campaign exhausted in Redis after 1000 but those 3 extra succeeded in Postgres after a Redis stutter.

Detection: nightly reconciliation job comparing `SELECT COUNT(*) FROM redemptions WHERE campaign_id = ? AND released_at IS NULL` against the Redis counter. Alert on any drift.

Prevention: synchronous Postgres write before returning success; the unique index is the backstop. The campaign should also have a Postgres-side hard cap: `INSERT INTO redemptions... WHERE (SELECT COUNT(*) FROM redemptions WHERE campaign_id = ?) < total_codes`. Expensive at burst because the count query is the bottleneck, so use it only as a periodic safety check, not in the hot path.

**3. Stacking SAVE10 + FREESHIP + BLACKFRI100.**

The cart holds the list of applied coupons. Validate is called per coupon and is told what other coupons are in the cart.

On the coupon side, each campaign has `stacking_rules`: `{"stackable_with": ["FREESHIP"], "excludes": ["BLACKFRI*"]}` (allowlist plus denylist). Validate computes: do the existing cart coupons satisfy this coupon's rules? If BLACKFRI100 is in the cart and you try to validate SAVE10, the stacking check fails with `not_stackable_with_blackfri`.

On the cart side, the cart enforces a max number of coupons per cart (e.g., 3) regardless of stacking rules. The cart computes the final discount: if both are percentages, decide compound `(1 - 0.1)(1 - 0.5)` vs sum `10% + 50% = 60%` off (clamped at 100%). This is a business decision per campaign; default is sum, capped at 100%.

The coupon service is the source of truth for stacking *rules*. The cart applies the rules. The split matters because the cart already has the context (other coupons, totals).

**4. Refund releases the code, or not?**

Per-campaign policy. Default: no release.

If release is on: `payment.refunded` triggers the release procedure. Set `released_at` on the redemption. For pool, set the code row's state back to `unused` and push the code back to the Redis list. For shared, `INCR` the Redis counter. For unique per-user, the user can re-redeem the same code.

The hidden cost is the parallel race: someone else can claim it within milliseconds. Customer service hates this when the refunding customer asks "can you give me that code back so I can use it tomorrow."

The cleaner answer is to not release. Refund the money. The discount is gone. If the customer is high-value, support manually issues a courtesy code from a separate budget. This avoids the race entirely.

**5. Expiration in the wrong timezone.**

Store `expires_at` as a UTC timestamp. Compare against UTC `NOW()`. Done.

For UI: render in the user's local timezone. "Expires in 3 hours" is better than "Expires at 2026-12-31 23:59:59 UTC" because it removes the timezone math from the user's brain.

For marketing: when they say "midnight Dec 31", ask which timezone. The campaign creation form requires a timezone selector. Default to the company's headquarters timezone but never assume.

The Twitter-yelling avoided: a user in Tokyo who sees "expires in 3 hours" at 11pm Tokyo time gets the same UTC-anchored expiry as a user in LA. The math is unambiguous. The wording is local. Everyone happy.

**6. Mass-update 10M codes without invalidating already-redeemed ones.**

The redemption snapshot is immutable; that is the contract. Updating the campaign's `discount` field does not retroactively change what past redemptions applied; the cart uses the snapshot in each redemption row.

For unredeemed codes, the campaign update takes effect on next redeem. The `codes` table does not store the discount per code; it joins to the campaign. Updating the campaign updates every future redemption from any of its codes.

If the discount is wrong on already-redeemed orders (e.g., wrong percentage applied), that is an accounting and refund problem, not a coupon-system problem. The coupon-system change just stops the bleeding for new redemptions.

Code path: `UPDATE campaigns SET discount = ? WHERE campaign_id = ?`. One row. Fast. Cache invalidation event broadcast to all service pods to drop the cached campaign metadata.

**7. Multi-region single-use guarantee.**

Each code is owned by exactly one region at creation time (typically the region whose campaign created it). The owning region's Postgres holds the authoritative redemption row.

If a code is redeemed in a non-owning region: the non-owning region's service detects the mismatch (campaign metadata says "owned by us-east"), forwards the redeem call via authenticated cross-region API to us-east, us-east performs the atomic claim against its Postgres and Redis, and the response is returned across regions to the user. Latency cost ~100ms cross-region. Acceptable for the rare cross-region case.

Per-user uniqueness across regions: the unique index on `(campaign_id, user_id)` lives only in the owning region. Cross-region calls all funnel into it. The unique index does the work.

Alternative: replicate the unique index globally using a strongly-consistent multi-region database (Spanner, DynamoDB Global Tables with conditional writes). Higher cost, simpler model. Worth it if cross-region redemptions are >10% of traffic; not worth it for the typical 1%.

**8. Bloom filter has no false negatives.**

Correct. A Bloom filter never says "definitely not present" for something that was actually added; if a code was inserted into the filter, it will always be reported as "maybe present."

What Bloom filters *do* have is **false positives**: occasionally saying "maybe present" for a code that was never inserted. This is fine for our use case: a false positive sends the request through to the cache/DB, which correctly returns 404. No harm done; just a tiny bit of leaked load.

In this design, the filter is sized for 0.1% false-positive rate at 200M items. Out of every 1000 brute-force attempts on bogus codes, ~999 are rejected at the filter and ~1 falls through to the DB. At a brute-force rate of 5000 attempts/sec, the DB sees ~5 lookups/sec from this source. Trivial.

What you must not do: rely on the filter to *prove* a code exists. You must still validate against the DB on a maybe-present hit, because the filter is approximate.

**9. SAVE10 has 9999 of 10000 uses; 20 users hit simultaneously.**

The Redis Lua script handles this. Each of the 20 requests enters the script serially (Redis is single-threaded). The first sees `remaining = 1`, decrements to 0, returns OK. The other 19 see `remaining = 0` and return `exhausted`.

The 1 winner's Postgres write proceeds normally with the unique index check (they have not redeemed before). The 19 losers see 410.

If the per-user-limit is 1 and any of the 20 users had already redeemed SAVE10 previously, the Lua script also rejects them with `already_redeemed` regardless of the counter, returning 409.

**10. Unredeemed code after expiry: reuse the string?**

No. Codes are append-only forever, even after expiry.

The codes table has a unique index on the code string. Reuse would require deleting the expired row, which loses audit history. A user might bookmark the code or have it in an email; reusing the string for a new campaign creates confusion ("why does my SAVE10 from last year suddenly apply 25% off?"). Trying an expired code is itself a useful fraud signal; reusing the string erases that signal.

What you can do: design the code namespace to be effectively infinite. 10 characters in base32 is ~10^15 codes. You will never run out. Mint a new string for each new campaign.

For pool campaigns where unclaimed codes pile up, the cost is the storage of those expired rows. At ~200 bytes each, 1M expired unredeemed codes is 200MB. Negligible. Optionally, archive expired codes to cold storage after 1 year.

### 13. Trade-offs worth saying out loud

**Why Postgres unique index *and* Redis Lua.** Either alone is a single point of failure. The unique index alone melts under 10k QPS on one row. Redis Lua alone loses correctness if Redis drops state. Together they form a fast hot path with a strong safety net.

**Why not just a sequence-backed counter in Postgres.** A `SERIAL` column gives unique IDs but does not enforce the cap (the cap is a count check, not a uniqueness check). And the row-level contention is still there. The unique index on `(campaign_id, user_id)` is the right primitive.

**Why a Bloom filter, not a Redis SET membership check.** Bloom filter is in-process, sub-microsecond, fixed memory. A Redis SISMEMBER is a network round-trip (1ms) and grows linearly with the number of issued codes. At 200M codes, the SET would be ~4GB in Redis; the Bloom filter is ~300MB in each pod's process memory. Bloom wins on cost and latency.

**Why an immutable redemptions log, not just current state.** Finance auditors, fraud investigators, and refund workflows all need the history. Mutating in place destroys traceability.

**What I would revisit at 10x scale.** Move to a discount engine (rules-based, not just code-based). Pre-personalize coupons per user. Add A/B testing of discount values. The simple coupon model breaks once marketing wants to ask "what if we showed user X a 15% code and user Y a 10% code based on their cart value."

### 14. Common interview mistakes

Most weak answers fall into one of these.

**Diving straight into a `coupons` table without thinking about concurrency.** The whole question is about the 10k-QPS burst. If your design's first sentence is "we'll have a coupons table with a `times_used` column," you have missed the point. The interviewer will steer you to the race condition; you should arrive there yourself.

**Using Redis without a Postgres backstop.** Redis can lose state. If you describe Redis as the source of truth and shrug at the failure modes, you fail the senior bar.

**Forgetting the per-user uniqueness check.** Some candidates correctly atomic-claim the campaign counter and forget that the same user can race themselves with two browser tabs. The unique index on `(campaign_id, user_id)` is non-optional.

**No idempotency on redeem.** Without Idempotency-Key, every network retry burns a code. At burst with flaky networks, this is catastrophic.

**Conflating validate and redeem.** A POST that always consumes is a UX disaster (no preview of the discount). A GET that consumes violates HTTP semantics and breaks browser pre-fetch. Two endpoints, clear separation.

**Ignoring brute-force.** Coupon namespaces get scraped. If you do not mention rate limiting and the Bloom filter (or some equivalent), you have not thought about the abuse model.

**Hand-waving release-on-refund.** "Yeah, we'll just put it back" creates a parallel race the candidate has not thought through. The senior answer either says "we don't release" with reasons, or describes the release race explicitly.

**No mention of the discount snapshot.** The cart applies the discount as it existed at redeem time, not as it exists now. Without the snapshot in the redemption row, marketing can retroactively change discounts on past orders, which breaks accounting.

**Designing for write throughput.** Even at 1M users, sustained QPS is single digits. The design pressure is the launch burst and the abuse traffic, not steady state. Candidates who size for "millions of writes per second" have misread the problem.

**Skipping the audit log.** Finance and fraud need it. The redemption table is your audit log; an immutable schema with a hash chain (at scale) is the senior addition.

If you can hit 7 of these 10, you are interviewing at the senior level. The launch-burst correctness story is what separates strong candidates from generic "design a CRUD app" answers. Most candidates miss the per-user-race issue and the importance of the Postgres backstop behind the Redis hot path. Those two are the most common drop-points in interviews on this problem.
