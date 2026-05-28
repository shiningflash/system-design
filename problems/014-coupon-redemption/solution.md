## Solution: Coupon Code Redemption System

### What this system is

A coupon redemption system is a small write-light service with one hard problem: surviving a launch burst where 10,000 users hit the same code in the same second and exactly 1,000 of them must win. Everything else is plumbing.

The design has two layers. Postgres is the source of truth; a unique index on `(campaign_id, user_id)` makes double-redemption impossible at the storage level. Redis with a Lua script handles the hot-burst claim so the database is not asked to serialize 10,000 transactions on one row.

Three code patterns share one schema: generic shared codes (`SAVE20`), unique per-user codes (`UID-XXXX`), and pre-generated pools (`BLACKFRI-XXXX`). All three use the same redeem API. A Bloom filter in front of the read path stops brute-force traffic from reaching the database. Rate limits stop the same user from grinding the namespace.

The interesting engineering lives at three edges: making the launch burst correct without a database stampede, defining what "release the code on refund" means without creating a new race, and keeping an audit trail finance can use years later.

---

### 1. The two questions that matter most

**Single-use or reusable, and which code pattern?** Single-use is one row, one redeem, done. Reusable means a counter and a per-user dedup check. If you assume one pattern and the interviewer wanted all three, you redesign mid-interview.

**What is the expected burst size?** Without this, you cannot decide whether the Postgres-only path is enough or whether you need Redis Lua. The architecture changes at around 100 claims per second on one campaign.

Everything else (stacking, expiration, refund, abuse) follows from those two.

---

### 2. The math

| Scale | Redeem QPS | Validate QPS | Storage |
|-------|-----------|--------------|---------|
| Steady state | ~0.6 (peak ~5) | ~3 (peak ~25) | 50 GB total |
| Launch burst | **10,000 in 1 second** | 50,000 in 1 second | ~100 KB hot |

The launch burst is the design pressure. The 9,000 losers all need a fast "sold out" response, not a timeout. Storage barely registers: 200 million codes x 200 bytes is about 40 GB. Add 9 GB of redemption log over 5 years. One Postgres instance handles it.

The system is small. The architecture exists for burst correctness and abuse resistance, not throughput or storage.

---

### 3. The API

Validate and redeem are separate endpoints. Validate is read-only and cheap. Redeem is the atomic act.

```
POST /api/v1/coupons/validate
Authorization: Bearer <token>

{
  "code": "BLACKFRI100",
  "cart_id": "cart_abc",
  "subtotal": 120000
}
```

| Status | Meaning |
|--------|---------|
| **200** | `{"valid": true, "discount": {...}, "expires_at": "..."}` |
| **404** | Unknown code (also returned for unauthenticated audience-mismatch) |
| **410** | Expired or fully exhausted |
| **403** | Authenticated user not in the code's audience |
| **429** | Rate limited |

```
POST /api/v1/coupons/redeem
Authorization: Bearer <token>
Idempotency-Key: <uuid>

{
  "code": "BLACKFRI100",
  "order_id": "ord_xyz",
  "user_id": "usr_42"
}
```

| Status | Meaning |
|--------|---------|
| **201** | Redemption succeeded |
| **200** | Idempotent retry (same Idempotency-Key seen before) |
| **409** | User already redeemed this code |
| **410** | Code exhausted or expired |
| **403** | Audience mismatch |

Load-bearing choices:

| Choice | Reason |
|--------|--------|
| `Idempotency-Key` required on redeem | Network retries at burst are guaranteed. Without the key, the same user's retry can consume two codes. |
| `order_id` ties redemption to order | Needed for refund/release and finance reconciliation. |
| Validate returns 404 for unauthenticated audience mismatches | Prevents scraping the namespace to discover "which codes exist." |

---

### 4. The data model

```mermaid
erDiagram
    campaigns ||--o{ codes : "contains"
    campaigns ||--o{ redemptions : "tracks"
    codes ||--o| redemptions : "consumed by"

    campaigns {
        uuid campaign_id PK
        text type
        jsonb discount
        int total_limit
        int per_user_limit
        jsonb audience_filter
        jsonb stacking_rules
        timestamptz starts_at
        timestamptz ends_at
        text status
    }
    codes {
        uuid code_id PK
        uuid campaign_id FK
        text code
        text intended_user
        text state
        timestamptz expires_at
    }
    redemptions {
        uuid redemption_id PK
        uuid campaign_id FK
        uuid code_id FK
        text user_id
        text order_id
        jsonb discount_applied
        text idempotency_key
        timestamptz redeemed_at
        timestamptz released_at
    }
```

<details markdown="1">
<summary><b>Show: the full SQL</b></summary>

```sql
CREATE TABLE campaigns (
    campaign_id      UUID PRIMARY KEY,
    name             TEXT UNIQUE NOT NULL,
    type             TEXT NOT NULL,
    discount         JSONB NOT NULL,
    total_limit      INT,
    per_user_limit   INT NOT NULL DEFAULT 1,
    audience_filter  JSONB,
    stacking_rules   JSONB,
    starts_at        TIMESTAMPTZ NOT NULL,
    ends_at          TIMESTAMPTZ NOT NULL,
    status           TEXT NOT NULL DEFAULT 'active'
);

CREATE TABLE codes (
    code_id          UUID PRIMARY KEY,
    campaign_id      UUID NOT NULL REFERENCES campaigns,
    code             TEXT NOT NULL,
    intended_user    TEXT,
    state            TEXT NOT NULL DEFAULT 'unused',
    claimed_by       TEXT,
    claimed_at       TIMESTAMPTZ,
    expires_at       TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_codes_code ON codes (code);
CREATE INDEX idx_codes_pool ON codes (campaign_id, state) WHERE state = 'unused';

CREATE TABLE redemptions (
    redemption_id    UUID PRIMARY KEY,
    campaign_id      UUID NOT NULL REFERENCES campaigns,
    code_id          UUID REFERENCES codes,
    user_id          TEXT NOT NULL,
    order_id         TEXT NOT NULL,
    discount_applied JSONB NOT NULL,
    idempotency_key  TEXT NOT NULL,
    redeemed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    released_at      TIMESTAMPTZ,
    released_reason  TEXT
);
CREATE UNIQUE INDEX idx_redemption_once
    ON redemptions (campaign_id, user_id) WHERE released_at IS NULL;
CREATE UNIQUE INDEX idx_redemption_idempotency ON redemptions (idempotency_key);
CREATE INDEX idx_redemption_order ON redemptions (order_id);

CREATE TABLE redemption_attempts (
    attempt_id    BIGSERIAL PRIMARY KEY,
    code          TEXT NOT NULL,
    user_id       TEXT,
    ip            INET,
    result        TEXT NOT NULL,
    attempted_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_attempts_user ON redemption_attempts (user_id, attempted_at DESC);
CREATE INDEX idx_attempts_ip ON redemption_attempts (ip, attempted_at DESC);
```

</details>

Three things doing real work:

**`UNIQUE(campaign_id, user_id) WHERE released_at IS NULL`.** Two browser tabs racing each other. The database serializes them. First insert wins. Second fails with a unique-violation. The API returns 409. This is the safety net behind whatever Redis says.

**`discount_applied` is a snapshot.** The discount as it was at redeem time, frozen. Marketing can change the campaign's discount later. Already-redeemed orders are unaffected. Finance audit is accurate.

**`redemption_attempts` logs everything, including failures.** Required for fraud signals. At production scale, partition it monthly.

---

### 5. The core algorithm: the atomic claim

Two paths, picked by campaign expected QPS.

**Low-traffic path: Postgres only.**

```sql
BEGIN;

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

INSERT INTO redemptions (redemption_id, campaign_id, code_id, user_id, order_id,
                         discount_applied, idempotency_key)
VALUES (...)
ON CONFLICT (campaign_id, user_id) WHERE released_at IS NULL DO NOTHING
RETURNING redemption_id;

COMMIT;
```

`FOR UPDATE SKIP LOCKED` lets concurrent transactions each pick a different unused row instead of queuing on one. Throughput: a few hundred claims per second on one Postgres, which is fine for any campaign below about 100 QPS.

**High-traffic path: Redis Lua + Postgres.**

```lua
-- KEYS[1] = "campaign:{cid}:remaining"
-- KEYS[2] = "campaign:{cid}:users"
-- KEYS[3] = "campaign:{cid}:pool"  (Redis list, pool type only)
-- ARGV[1] = user_id
-- ARGV[2] = campaign_type

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

Latency under 1 ms. Redis is single-threaded, so 10,000 concurrent requests serialize on one CPU core. After Redis returns OK, the service writes to Postgres synchronously (about 5 ms). Total redeem latency: about 6 ms. The Postgres `ON CONFLICT` clause is the backstop if Redis hiccups.

Which path to use:

| Campaign expected QPS | Approach |
|----------------------|----------|
| < 100 | Postgres only |
| 100 to 10,000 | Redis Lua + Postgres backstop |
| > 10,000 | Redis Lua + sharded Redis (counter and user-set by `hash(user_id)`) |

In practice, run all campaigns through Redis. The cost for a low-traffic campaign is negligible. The code path is uniform.

**The Bloom filter prefilter.**

```python
def validate(code):
    if not bloom_filter.maybe_contains(code):
        return 404
    # continues to Redis / DB
```

Bloom filters never have false negatives. If the filter says "not present," the code was definitely never issued. They have a tunable false-positive rate. At 0.1%, 99.9% of brute-force attempts on bogus codes are cut off before touching the database. Memory footprint at 200 million codes: about 300 MB in process memory. Fits easily in each pod.

---

### 6. The architecture

```mermaid
flowchart TB
    subgraph Edge["Client edge"]
        C([Web / Mobile / POS]):::user
        GW["API Gateway<br/>(auth · rate limit · WAF)"]:::edge
    end

    subgraph WritePath["Synchronous write path"]
        S["Coupon Service (write)<br/>(stateless pods)"]:::app
        Redis{{"Redis<br/>Lua claim · counter · user-set"}}:::cache
    end

    DB[("Postgres<br/>campaigns · codes<br/>redemptions · attempts")]:::db

    subgraph ReadPath["Validate read path"]
        R["Coupon Service (read)"]:::app
        Bloom["Bloom filter (in-process)"]:::app
        RCache[("Redis<br/>campaign meta cache")]:::cache
    end

    K{{"Kafka<br/>coupon.redeemed<br/>coupon.released"}}:::queue

    subgraph Consumers["Async consumers"]
        Cart["Cart Service"]:::app
        Fraud["Fraud Service"]:::app
        Analytics[("ClickHouse")]:::db
    end

    C --> GW
    GW -->|redeem| S
    GW -->|validate| R
    S --> Redis
    Redis --> DB
    R --> Bloom
    Bloom -.miss.-> RCache
    RCache -.miss.-> DB
    DB -->|CDC / outbox| K
    K --> Cart
    K --> Fraud
    K --> Analytics

    classDef user  fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef edge  fill:#e2e8f0,stroke:#475569,color:#1e293b
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

Five things to notice:

- The write path touches Redis and Postgres synchronously. Nothing else. Kafka, cart, fraud, and analytics are downstream of CDC. Cart down does not block checkout.
- The Bloom filter lives in process memory, not Redis. Process memory is faster and cheaper for this use case.
- The Postgres unique index is the ground truth. Every other layer can lose data and the system recovers from the database. If the unique index breaks, you have a correctness bug.
- Coupon Service pods are stateless. The Bloom filter is rebuilt on startup from an S3 snapshot plus a tail of `campaign_created` events.
- Reads (validate) and writes (redeem) scale independently. The read service can be replicated freely since it is read-only.

---

### 7. A redeem, traced

```mermaid
sequenceDiagram
    autonumber
    participant Alice
    participant GW as API Gateway
    participant S as Coupon Service
    participant R as Redis
    participant DB as Postgres
    participant K as Kafka
    participant Cart

    Alice->>GW: POST /redeem (BLACKFRI100, Idempotency-Key: abc123)
    GW->>S: forward (auth ok, rate limit ok)
    S->>R: GET idempotency:abc123
    R-->>S: nil (not seen)
    S->>R: EVAL Lua (check counter, user-set, decr, sadd)
    R-->>S: ok, claimed

    rect rgb(241, 245, 249)
        Note over S,DB: one database transaction
        S->>DB: INSERT redemptions ON CONFLICT DO NOTHING
        S->>DB: INSERT redemption_attempts (result=success)
        S->>DB: COMMIT
    end

    S->>R: SET idempotency:abc123 = {redemption_id} EX 300
    S-->>GW: 201 Created
    GW-->>Alice: 201 Created (~6ms total)
    DB->>K: CDC: coupon.redeemed
    K->>Cart: apply discount to order
```

Target latencies:

| Operation | P99 |
|-----------|-----|
| Validate (cache hit) | ~5 ms |
| Validate (cache miss) | ~50 ms |
| Redeem (burst) | ~50 ms (Redis 1 ms + Postgres 5-10 ms + network) |

---

### 8. The scaling journey: 10 users to 1 million

```mermaid
flowchart LR
    S1["Stage 1<br/>10 to 100 users<br/>1 Postgres, no Redis<br/>~$80/mo"]:::s1
    S2["Stage 2<br/>~1,000 users<br/>+ rate limits + LRU cache<br/>~$250/mo"]:::s2
    S3["Stage 3<br/>10k to 100k users<br/>+ Redis Lua + Bloom filter<br/>+ Kafka<br/>~$1-2k/mo"]:::s3
    S4["Stage 4<br/>1M users<br/>+ sharded Redis<br/>+ multi-region<br/>+ hash-chain audit"]:::s4

    S1 --> S2 --> S3 --> S4

    classDef s1 fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef s2 fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef s3 fill:#fef3c7,stroke:#a16207,color:#713f12
    classDef s4 fill:#fce7f3,stroke:#be185d,color:#831843
```

**Stage 1: 10 to 100 users.** One Postgres, one app pod. Three tables with the unique index. Validate and redeem are straight database calls. No Redis, no Kafka. About $80/month. The unique index alone is the correctness story.

**Stage 2: 1,000 users.** A user shares a code on a forum. 200 anonymous attempts hit in 5 minutes. Database CPU spikes. Add per-user rate limiting (token bucket in Redis) and per-IP rate limiting for unauthenticated traffic. A cart bug caused some users to submit redeem twice. Add `ON CONFLICT DO NOTHING` handling and return 409. Require `Idempotency-Key` on redeem. About $250/month.

**Stage 3: 10,000 to 100,000 users.** A flash sale: 2,000 codes, 5,000 users at noon. Postgres CPU pegged for 30 seconds under hot row contention. Most requests timed out. A brute-force script fired 5,000 bogus attempts per second from rotating IPs. Fix: Redis Lua for hot campaigns, Bloom filter in process (bogus codes return 404 in microseconds), pool codes pre-loaded into a Redis list, Kafka for async consumers. About $1-2k/month.

**Stage 4: 1 million users.** The hottest Black Friday campaign saturated one Redis core at 50k QPS. EU operations launched and codes need single-use guarantees across regions. Fix: sharded Redis for ultra-hot campaigns (counter and user-set partitioned by `hash(user_id)` across N nodes), per-region code ownership with cross-region routing via authenticated API, audit hash chain on the redemptions table for SOX compliance. About $10-30k/month.

The core insight carries through all four stages: Postgres unique index as safety net, Redis Lua as burst path, Bloom filter as brute-force shield.

---

### 9. The three code patterns

| Pattern | Claim operation | Leak blast radius | Best for |
|---------|----------------|-------------------|---------|
| **Generic shared** | Lua DECR + unique index | High (one code = all slots) | Public promos with loose audience |
| **Unique per-user** | UPDATE WHERE state='unused' | Low (one code = one slot) | Newsletter rewards, referrals |
| **Pre-generated pool** | LPOP from Redis list | Medium (one code = one slot) | Flash sales with meaningful code artifacts |

One engine, three patterns, no special API routes. The campaign `type` field switches the internal claim logic.

---

### 10. Reliability

**Redis dies mid-redemption.** Lua returned OK and decremented the counter. The service crashed before the Postgres write. Redis later lost state in a failover.

Three things keep this safe:

1. Synchronous Postgres write before returning success. This is why we accept the 5-10 ms cost.
2. The unique index as ground truth. If Redis is rebuilt, the rebuild queries Postgres for the current redeemed count and reseeds the counter and user-set.
3. A nightly reconciliation job: `SELECT COUNT(*) FROM redemptions WHERE campaign_id=? AND released_at IS NULL` vs Redis counter. Alert on any drift.

**Payment fails after redemption.** Two policies, picked at campaign creation:

- Default (`release_on_failure = false`): the redemption stands. The user lost their slot. Simpler and avoids the release race.
- Opt-in (`release_on_failure = true`): set `released_at` on the redemption, push the code back to the Redis list, increment the Redis counter. Each step is idempotent.

The senior position: default to no release. Release introduces a parallel race where someone else claims the slot while the original user is debugging payment. For high-value campaigns where customer goodwill matters, opt in knowingly.

**Kafka is down.** Redemptions succeed in Postgres but the cart does not receive the event. The cart polls the coupon service for redemptions tied to active carts as a fallback. Or the user re-applies the code in the cart UI (validate returns `{"valid": true, "already_redeemed_for_this_order": true}` with the discount). A Kafka outage must not block checkout.

---

### 11. Observability

| Metric | Why it matters |
|--------|----------------|
| `redeem.latency` p50/p95/p99 | The headline SLO. Alert if p99 > 200 ms at burst. |
| `redeem.result.distribution` | success / exhausted / already_redeemed / audience_mismatch. Drift signals UX bugs. |
| `redeem.attempts.rate` per campaign | 100x spike triggers velocity pause. |
| `validate.cache_hit_rate` | Should be > 90%. Below means the cache is too small. |
| `bloom_filter.false_positive_rate` | Spikes indicate scraper traffic bypassing the filter. |
| `redis.counter.drift_vs_db` | Computed nightly. Non-zero is a Redis incident or a bug. |
| `rate_limit.triggered.count` | Top offenders dashboard for the security team. |
| `kafka.lag.coupon_redeemed` | Cart applies discounts with this lag. Alert at > 30 s. |

Page on: redeem error rate > 5% for 5 min. Redis-Postgres drift > 100. Bloom filter rebuild stuck.

Ticket on: validate cache hit rate < 80%. Rate limit triggers > 10x daily baseline.

---

### 12. Follow-up answers

**1. Redis decremented but Postgres did not record. User retries.**

The `Idempotency-Key` in Redis (5-minute TTL) catches it. The retry's key matches. The cached response is returned. The user sees success without a second database write.

If the idempotency cache also lost the entry (Redis failover), the retry hits the Lua script again. The user is already in the user-set, so the script returns `already_redeemed`. The service looks up the original `redemption_id` in Postgres and returns it. Idempotent.

The one bad case: the idempotency cache is lost AND Postgres also lost the row (should not happen with a healthy database). Then the user redeems a second time. Mitigation: write to Postgres synchronously before returning success.

**2. 1,000 codes in the campaign. 1,003 in Postgres after launch.**

Three possible causes. One: the release-on-refund flow released 3 codes and they were re-claimed. Not an overcount: `WHERE released_at IS NULL` gives exactly 1,000 active redemptions. Two: the partial unique index was missing (`WHERE released_at IS NULL` omitted), allowing two active rows per user. Three: Redis and Postgres drifted because Postgres writes failed silently while Redis kept running.

Detection: nightly reconciliation comparing the index-constrained count vs Redis counter. Alert on any drift. Prevention: synchronous Postgres write before success. The unique index as backstop.

**3. Stacking `SAVE10` + `FREESHIP` + `BLACKFRI100`.**

Each campaign has `stacking_rules`: `{"excludes": ["BLACKFRI*"]}`. Validate is told what other coupons are in the cart. It evaluates the stacking rules for the incoming code against the cart's existing codes. If `BLACKFRI100` excludes `SAVE10`, validate returns `{"valid": false, "reason": "not_stackable_with_SAVE10"}`.

The coupon service owns stacking rules. The cart owns the final discount calculation (additive vs multiplicative, capped at 100%). Business decision per campaign.

**4. Refund releases the code, or not?**

Per-campaign policy. Default: no release. Refund the money. The discount is gone. If the customer is high-value, support manually issues a courtesy code from a separate budget. This avoids the release race entirely.

If release is on: set `released_at` on the redemption (the partial unique index makes the user free to redeem again). For pool codes, set the code row back to `unused` and push it to the Redis list. For shared counters, `INCR` the Redis counter. Each step is idempotent but the parallel race is real: someone else can claim within milliseconds of the release.

**5. Expiration in the wrong timezone.**

Store `expires_at` as a UTC timestamp. Compare against UTC `NOW()`.

For UI: render in the user's local timezone as a relative duration ("Expires in 3 hours") rather than an absolute timestamp. When marketing says "midnight Dec 31," require a timezone selector on the campaign creation form. Default to the company's HQ timezone, never assume.

**6. Mass-update 10M codes without invalidating already-redeemed ones.**

The `discount_applied` snapshot in each redemption row is immutable. Updating the campaign's `discount` field does not touch past redemptions. The cart uses the snapshot, not the live campaign discount.

For unredeemed codes, the campaign update takes effect on next redeem. The `codes` table does not store the discount per code. It joins to the campaign. Code path: `UPDATE campaigns SET discount=? WHERE campaign_id=?`. One row. Broadcast a cache-invalidation event to all pods to drop the cached campaign metadata.

**7. Multi-region single-use guarantee.**

Each code is owned by exactly one region at creation. The owning region's Postgres holds the authoritative unique index. A redemption on the wrong region routes to the owning region via authenticated cross-region API. The owning region runs the Lua claim and Postgres write. Latency cost: about 100 ms. Acceptable for the rare cross-region case, which is typically < 1% of traffic.

Alternative: strongly-consistent multi-region database (Spanner, DynamoDB Global Tables). Higher cost, simpler model. Worth it only if cross-region redemptions exceed about 10% of traffic.

**8. Bloom filter has no false negatives.**

Correct. A Bloom filter never says "definitely not present" for something that was inserted. If a code was added to the filter, the filter always reports "maybe present."

What Bloom filters have is false positives: occasionally saying "maybe present" for a code never inserted. This is fine here. A false positive sends the request through to the cache and database, which correctly return 404. At 0.1% false-positive rate and 5,000 brute-force attempts per second, about 5 fall through to the database. Trivial.

**9. `SAVE10` has 9,999 of 10,000 uses. Twenty users hit redeem simultaneously.**

The Lua script handles this. Redis is single-threaded, so the 20 requests serialize. The first sees `remaining = 1`, decrements to 0, returns OK. The other 19 see `remaining = 0` and return `exhausted` (410). The one winner's Postgres insert proceeds normally. The 19 losers get 410 immediately, without waiting for the winner to finish.

**10. Unused expired code. Can you reuse the code string?**

No. Codes are append-only forever, even after expiry. The unique index on the code string is permanent. Reusing the string requires deleting the expired row, which loses audit history. A user may have the code bookmarked. Reusing the string for a new campaign creates confusion ("why does my code from last year apply a different discount?"). An attempt on an expired code is also a useful fraud signal.

Design the code namespace to be effectively infinite. 10 characters of base32 is about 10^15 codes. Mint a new string for each campaign.

---

### 13. Trade-offs worth saying out loud

**Why Postgres unique index and Redis Lua, not just one.** The unique index alone melts under 10k QPS on one row. Redis Lua alone loses correctness if Redis drops state. Together: fast hot path with a strong safety net.

**Why a Bloom filter instead of a Redis SET membership check.** Bloom filter is in-process, sub-microsecond, fixed memory (about 300 MB for 200 million codes). A Redis `SISMEMBER` is a network round-trip (1 ms) and grows linearly. At 200 million codes, the SET is about 4 GB in Redis. Bloom wins on cost and latency.

**Why an immutable redemptions log instead of just current state.** Finance auditors, fraud investigators, and refund workflows all need the history. Mutating in place destroys traceability. The `discount_applied` snapshot is the only way to answer "what discount did this customer actually get on that order in Q3?"

**Why not default to releasing codes on refund.** The release creates a parallel race. Someone else can claim the released slot within milliseconds. The cleaner answer is to not release and issue a courtesy code manually for high-value cases.

---

### 14. Common mistakes

**Diving straight into a `coupons` table without thinking about concurrency.** If your first sentence is "we'll have a `times_used` column," you have missed the point. The interviewer will steer you to the race condition. Arrive there first.

**Using Redis without a Postgres backstop.** Redis can lose state. If you describe Redis as the source of truth and shrug at failure modes, you fail the senior bar.

**Forgetting the per-user uniqueness check.** Some candidates atomic-claim the campaign counter and forget the same user can race themselves with two browser tabs. The `UNIQUE(campaign_id, user_id)` index is non-optional.

**No idempotency on redeem.** Without `Idempotency-Key`, every network retry burns a slot. At burst with flaky networks, this is catastrophic.

**Conflating validate and redeem.** A POST that always consumes is a UX disaster (no discount preview before checkout). A GET that consumes violates HTTP semantics and breaks browser pre-fetch. Two endpoints, clear separation.

**Ignoring brute-force.** Coupon namespaces get scraped. Without rate limiting and the Bloom filter, an attacker enumerates the namespace in hours.

**Hand-waving release-on-refund.** "Yeah, we'll just put it back" without naming the parallel race is a weak answer. Say explicitly whether you release, why, and what the race looks like if you do.

**No discount snapshot in the redemption row.** Marketing can retroactively change campaign discounts. Without the snapshot, past orders get the wrong discount amount in finance reconciliation.

**Designing for steady-state write throughput.** Even at 1 million users, sustained QPS is single digits. The design pressure is the launch burst. Candidates who size for "millions of writes per second" have misread the problem.

The launch-burst correctness story is what separates strong candidates from generic "design a CRUD app" answers. The two most common drop-points: the per-user race issue, and the importance of the Postgres backstop behind the Redis hot path.
