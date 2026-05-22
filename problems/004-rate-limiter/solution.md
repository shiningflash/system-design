## Solution: Design a Rate Limiter

### TL;DR

A rate limiter is a small piece of code with a big blast radius. The algorithm is rarely the interesting part; what matters is where the check runs (in-process middleware on the gateway), how state is shared across nodes (Redis with an atomic Lua script), what happens when state is unreachable (configurable fail-open vs fail-closed per route), and how you keep the limiter itself from becoming the bottleneck (per-node LRU to suppress check traffic for hot keys).

The right algorithm for almost every production case is **sliding window counter**, implemented as a Lua script that atomically increments a per-window counter and computes a weighted blend of current and previous window. It is O(1) memory per client, has no fixed-window boundary doubling, and is implementable in 15 lines of Lua. Token bucket is the strong second choice and is what most public clouds actually run; it is what you use when you want explicit burst allowance.

The interesting engineering is at the seams: layered keys (IP + API key + user_id), per-route cost weighting, customer-specific override storage with a sane staleness policy, and the operational stuff (429 headers, abuse forensics, fail-open vs fail-closed for the route's risk profile).

### 1. Clarifying questions and why each matters

Covered in `question.md`. The two that matter most: **what do you limit on?** (the identification key choice shapes the data model) and **what happens when Redis is down?** (the failure-mode choice shapes the operational story). Candidates who answer "use a token bucket in Redis" without addressing either are answering a different, simpler question.

### 2. Capacity estimates (full working)

- 300K peak QPS → **300K counter writes/sec** baseline, ~1M Redis ops/sec when you count per-minute + per-hour + per-route counters per client.
- 100K clients × 3 counters × 100 bytes = **30MB of state**. Tiny. We shard for availability, not capacity.
- 1500 limit checks per second per gateway node. Sub-millisecond in-process, ~1-2ms with a Redis hop.
- Wire cost is negligible; this system's bottleneck is round-trip count, not bytes.

The math says: state is small, requests are many, every request must touch the limiter. Everything else is consequence.

### 3. API and headers

The limiter is internal middleware, not a public API. But it produces public-facing response headers on every API request that passes through it.

**On allowed requests:**

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1716381660
...response body...
```

**On rejected requests:**

```
HTTP/1.1 429 Too Many Requests
Retry-After: 28
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716381660
Content-Type: application/json

{
  "error": "rate_limited",
  "message": "Request exceeds rate limit of 1000 requests per minute.",
  "retry_after_seconds": 28,
  "doc_url": "https://api.example.com/docs/rate-limits"
}
```

Notes:

- **429 not 503.** 503 means the server is overloaded and the request might succeed later anywhere. 429 means *this client* is over its limit; other clients are fine.
- **`Retry-After` is mandatory.** Some clients respect it (well-written SDKs, browser fetch retries); others ignore it and hammer. Either way, server-computed retry-after costs you nothing.
- **Headers on success too.** Lets good clients self-throttle. The 30 bytes per response are worth it; one ticket-avoided pays for billions of headers.
- **Status header values must be honest.** `X-RateLimit-Remaining` after a rejected request is 0, not -1 or the raw counter value. Do not leak implementation details.

For internal services (not the public API): use a stricter format with `X-Internal-RateLimit-*` prefix so internal callers can distinguish their own limit from a downstream public-API limit they may also have hit.

### 4. Data model

The limiter has two distinct storage layers: hot counters in Redis and config in a separate config store.

**Hot counters (Redis):**

Keys are composite. The shape is:

```
rl:{scope}:{identifier}:{route_class}:{window_start_ts}
```

For example:

```
rl:apikey:sk_live_xyz789:search:1716381600    → integer count, TTL 120s
rl:ip:198.51.100.42:default:1716381600        → integer count, TTL 120s
rl:user:user_42:write:1716381600              → integer count, TTL 120s
```

`window_start_ts` is the start of the current fixed window (e.g. minute-aligned). The sliding window counter (Section 6) reads both the current window's counter and the previous window's counter, so we keep two keys per scope alive at a time.

TTL is set to `2 × window_length` so old keys clean themselves up. We never write a counter without an EXPIRE in the same Lua script.

**Config store (Postgres or DynamoDB):**

```sql
CREATE TABLE rate_limit_tiers (
    tier_name      VARCHAR(32) PRIMARY KEY,    -- 'free', 'pro', 'enterprise'
    default_limit  INTEGER NOT NULL,           -- requests per window
    window_seconds INTEGER NOT NULL,           -- typical 60
    burst_multiplier NUMERIC(3,2) NOT NULL DEFAULT 1.00
);

CREATE TABLE rate_limit_overrides (
    api_key        VARCHAR(64) NOT NULL,
    route_class    VARCHAR(32) NOT NULL,       -- 'default', 'search', 'write', etc.
    limit_value    INTEGER NOT NULL,
    window_seconds INTEGER NOT NULL,
    valid_until    TIMESTAMPTZ,                -- NULL = permanent
    reason         TEXT,                       -- audit trail, never NULL in practice
    PRIMARY KEY (api_key, route_class)
);

CREATE TABLE rate_limit_route_costs (
    route_pattern  VARCHAR(128) NOT NULL,      -- '/api/v1/search'
    route_class    VARCHAR(32) NOT NULL,       -- which counter to charge
    cost_units     INTEGER NOT NULL DEFAULT 1, -- search = 5, lookup = 1
    PRIMARY KEY (route_pattern)
);
```

Design notes:

- **Tiers in a tiny static table** loaded into every gateway at startup and refreshed on a 60-second timer. A tier change applies cluster-wide within a minute, not instantly. This is intentional; rate limits are not security boundaries.
- **Overrides indexed by `(api_key, route_class)`** so an enterprise customer with raised search limits but default write limits is two rows, not one. Avoids embedding policy in JSON.
- **Route costs are a separate table** so the cost model is decoupled from per-customer overrides. The pricing team owns the cost table; sales owns the overrides table.

### 5. The five algorithms in detail

#### a. Token bucket

```
state = (tokens, last_refill_ts)

on_request:
    now = current_time()
    elapsed = now - last_refill_ts
    tokens = min(bucket_size, tokens + elapsed * refill_rate)
    last_refill_ts = now
    if tokens >= 1:
        tokens -= 1
        return allowed
    else:
        return rejected
```

Two parameters: `bucket_size` (max burst) and `refill_rate` (sustained rate). A bucket of size 200 refilled at 1 token/sec allows a burst of 200, then 1/sec sustained.

Pros: explicit burst control, conceptually simple, what most clouds use.

Cons: the refill calculation is stateful in a way that is awkward under contention. Two requests reading `(tokens, last_refill_ts)` simultaneously can both add the refill amount and both consume a token, double-spending the refill window. Atomic Lua handles this but the script is longer than the sliding window equivalent.

#### b. Leaky bucket

Same shape, but instead of refilling tokens, you "drain" the bucket at a fixed rate. Requests enter the bucket; if the bucket is full, reject. The bucket drains continuously.

Equivalent to token bucket in steady state, but burst behavior is different: leaky bucket smooths bursts by queuing them (output is constant rate). Token bucket allows bursts at the input.

Almost no one uses leaky bucket for API rate limiting because the queuing adds latency. Used in network traffic shaping where smoothing matters.

#### c. Fixed window counter

```
key = "rl:apikey:xyz:1716381600"   # minute-aligned timestamp
count = INCR(key)
EXPIRE(key, 120)
if count > limit: reject
```

Three lines of Redis. Beautiful. And broken at boundaries: a client can send `limit` requests at 11:59:59 and `limit` more at 12:00:00, giving them `2 × limit` in two seconds.

For loose advisory limits (block obvious abusers) this is fine. For limits you advertise to customers, it is embarrassing.

#### d. Sliding window log

```
key = "rl:apikey:xyz"
ZREMRANGEBYSCORE(key, 0, now - window)        # drop expired
count = ZCARD(key)
if count < limit:
    ZADD(key, now, request_id)
    EXPIRE(key, window)
    return allowed
else:
    return rejected
```

Stores every request timestamp. Exactly correct. Memory: 8 bytes per request × N requests in window × M clients. At 1000 limit × 100K clients = **80GB**. Unworkable above small N.

#### e. Sliding window counter

The right answer for most production systems. We deep-dive in Section 6.

### 6. Core algorithm: sliding window counter

#### The idea

The fixed window counter has a boundary problem because the count resets instantly. A true sliding window solves this exactly but uses O(N) memory. The sliding window counter is the cheap approximation: keep two fixed-window counters (current and previous), and compute an estimated count as a weighted blend.

If we are 30 seconds into the current minute and the previous minute had 80 requests, the current minute so far has 40, the estimate is:

```
estimated_count = current_count + previous_count × (1 - elapsed_in_current_window / window)
                = 40 + 80 × (1 - 30/60)
                = 40 + 40
                = 80
```

This is an average that smooths the boundary. At the exact boundary (elapsed = 0), the estimate is `0 + previous × 1.0 = previous`, so there is no doubling. As time passes, the previous window contribution decays linearly, and by the end of the current minute the estimate is `current + 0 = current`.

#### The Lua script

The whole limiter logic in one atomic Lua script:

```lua
-- KEYS[1] = current window key, e.g. "rl:apikey:xyz:1716381660"
-- KEYS[2] = previous window key, e.g. "rl:apikey:xyz:1716381600"
-- ARGV[1] = limit (e.g. 1000)
-- ARGV[2] = window_seconds (e.g. 60)
-- ARGV[3] = now_ms (current time in milliseconds)
-- ARGV[4] = cost (units to charge, default 1)
-- Returns: { allowed (0|1), remaining, retry_after_ms }

local limit          = tonumber(ARGV[1])
local window_seconds = tonumber(ARGV[2])
local now_ms         = tonumber(ARGV[3])
local cost           = tonumber(ARGV[4])

local current = tonumber(redis.call('GET', KEYS[1])) or 0
local previous = tonumber(redis.call('GET', KEYS[2])) or 0

-- How far into the current window are we, as a fraction [0,1)?
local window_ms      = window_seconds * 1000
local elapsed_ms     = now_ms % window_ms
local prev_weight    = (window_ms - elapsed_ms) / window_ms

local estimated      = current + math.floor(previous * prev_weight)

if estimated + cost > limit then
    -- Time until enough of previous window has rolled off.
    local need_to_shed = (estimated + cost) - limit
    local retry_after_ms = math.ceil((need_to_shed / math.max(previous, 1)) * window_ms)
    return { 0, 0, retry_after_ms }
end

-- Allowed. Increment the current window counter.
local new_current = redis.call('INCRBY', KEYS[1], cost)
-- Set TTL to 2x window so previous-window keys live long enough to be read.
redis.call('EXPIRE', KEYS[1], window_seconds * 2)

local remaining = math.max(0, limit - (estimated + cost))
return { 1, remaining, 0 }
```

A few things to highlight:

- **One Redis round trip.** Two GETs, one conditional INCRBY, one EXPIRE, all inside one Lua execution. The gateway makes one call.
- **Atomic.** Two concurrent requests cannot both pass the check at count=limit-1. Lua scripts execute serially within a Redis shard.
- **No clock from Redis.** The caller passes `now_ms`. This is intentional: if you let Redis decide "now" (via TIME), clock skew between Redis nodes during failover can cause the same window key to be interpreted as different windows. The caller's clock is the authority.
- **`cost` parameter.** Supports cost-based limits (an expensive query charges 5 units; a cheap one charges 1) with no extra Redis calls.
- **`retry_after_ms` is computed, not guessed.** Even though it is approximate (assumes the rejected window's previous count is the rate going forward), it gives the client a much better hint than a fixed value.

#### Gateway-side code (Python-ish pseudocode)

```python
SCRIPT_HASH = redis.script_load(LUA_SCRIPT)   # done once at startup

class Limiter:
    def __init__(self, redis_client, local_cache_size=10000):
        self.redis = redis_client
        self.lru = LRUCache(local_cache_size)  # key → (count_estimate, exp_ms)

    def check(self, api_key, route_class, cost=1):
        tier = config.get_tier_for(api_key)
        limit, window_s = tier.limit, tier.window_seconds

        now_ms = time.time_ns() // 1_000_000
        window_start = (now_ms // 1000 // window_s) * window_s
        prev_start = window_start - window_s

        cur_key  = f"rl:apikey:{api_key}:{route_class}:{window_start}"
        prev_key = f"rl:apikey:{api_key}:{route_class}:{prev_start}"

        # Fast-path: if local LRU says we are already over, reject without Redis.
        cached = self.lru.get(cur_key)
        if cached and cached.count_estimate >= limit and cached.exp_ms > now_ms:
            return RateLimitResult(allowed=False, remaining=0,
                                   retry_after_ms=cached.exp_ms - now_ms)

        try:
            allowed, remaining, retry_ms = self.redis.evalsha(
                SCRIPT_HASH, 2, cur_key, prev_key,
                limit, window_s, now_ms, cost)
        except RedisError:
            return self._fail_mode(api_key, route_class, cost)

        # Update LRU so the next request can fast-fail without Redis.
        self.lru.set(cur_key, CachedCount(
            count_estimate=limit - remaining,
            exp_ms=now_ms + 100  # 100ms TTL on the cache entry
        ))

        return RateLimitResult(allowed=bool(allowed),
                               remaining=remaining,
                               retry_after_ms=retry_ms)

    def _fail_mode(self, api_key, route_class, cost):
        if config.fail_open_for(route_class):
            metrics.increment("limiter.degraded.fail_open")
            return RateLimitResult(allowed=True, remaining=-1, retry_after_ms=0)
        else:
            metrics.increment("limiter.degraded.fail_closed")
            return RateLimitResult(allowed=False, remaining=0, retry_after_ms=5000)
```

The LRU has a 100ms TTL on entries. This is short enough that genuine recovery (client backs off and returns later) is honored, but long enough to absorb the next 5-10 requests from an abusive client without hitting Redis.

### 7. Architecture (detailed)

```
                              Client (browsers, SDKs, batch jobs)
                                          │
                                          ▼
                                ┌─────────────────────┐
                                │   Global LB / WAF   │  TLS termination, basic
                                │                     │  L4/L7 distribution.
                                └──────────┬──────────┘
                                           │
                          ┌────────────────┼────────────────┐
                          │                │                │
                          ▼                ▼                ▼
                  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                  │  Gateway 1   │ │  Gateway 2   │ │  Gateway N   │  200 stateless
                  │              │ │              │ │              │  nodes.
                  │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
                  │ │ Limiter  │ │ │ │ Limiter  │ │ │ │ Limiter  │ │  In-process
                  │ │  + LRU   │ │ │ │  + LRU   │ │ │ │  + LRU   │ │  middleware,
                  │ └─────┬────┘ │ │ └─────┬────┘ │ │ └─────┬────┘ │  per-node LRU.
                  └───────┼──────┘ └───────┼──────┘ └───────┼──────┘
                          │                │                │
                          └────────────────┼────────────────┘
                                           │  EVALSHA Lua script
                                           ▼
                                ┌─────────────────────┐
                                │   Redis Cluster     │  6 primary shards,
                                │   sharded by         │  one replica each.
                                │   {client_key}      │  Hash tags route
                                │                     │  same-client keys
                                └──────────┬──────────┘  to the same shard.
                                           │
                                           │ AOF persistence + RDB snapshots
                                           ▼
                                ┌─────────────────────┐
                                │  Redis Disk Backup  │  For forensic
                                │  + S3 archive       │  reconstruction.
                                └─────────────────────┘

                          ┌──────────────────────────────┐
                          │  Config Service              │  Tiers, overrides,
                          │  (Postgres + 60s refresh on  │  route costs.
                          │   gateway side)              │  Read-mostly.
                          └──────────────────────────────┘

                          ┌──────────────────────────────┐
                          │  Decision log → Kafka →      │  Every reject + every
                          │  ClickHouse                  │  rare-event allow goes
                          │                              │  to a stream for
                          │                              │  forensics and analytics.
                          └──────────────────────────────┘
```

Component responsibilities:

- **Gateway with in-process limiter.** The limiter is a library, not a separate service. Avoids a network hop per request.
- **Local LRU.** Cuts Redis QPS for hot keys (abusive clients, popular endpoints). Critical at scale.
- **Redis Cluster sharded by client key.** All counters for one client land on the same shard, so the Lua script reads both current and previous window keys without cross-shard fan-out. Hash tag syntax `{api_key}` forces this in Redis Cluster.
- **Config service.** Tiers, overrides, costs. Read-mostly. Refresh on a 60-second timer on each gateway. If it is down, gateways use their last-known config.
- **Decision log.** Every rejected request goes to Kafka with `{client_key, route, limit, count, ts}`. Streamed to ClickHouse for "why was customer X rate-limited yesterday?" investigations. Sample of allowed requests goes too for baseline traffic profile.

### 8. Read and write paths

For a rate limiter, "read" and "write" both happen on every request. The check itself is an atomic read-and-conditional-write.

**Per-request path:**

1. Request arrives at gateway. TLS already terminated by LB.
2. Auth middleware runs first; extracts `(api_key, user_id, ip)` from the request.
3. Limiter middleware computes the rate limit keys. Typically 3-4 keys to check: per-API-key, per-IP, per-user, per-API-key-per-route.
4. For each key, fast-path check against local LRU. If any key is already known-over-limit, **reject immediately**. No Redis call.
5. If no fast-path hit, run all checks against Redis in parallel using a pipelined batch. One EVALSHA per key. Latency: ~1-2ms total (parallel).
6. Take the most restrictive result. If any key rejected, the request is rejected.
7. Update local LRU with each key's new count.
8. If allowed: forward to API service, attach `X-RateLimit-*` headers to the response.
9. If rejected: return 429 with headers and JSON body. Emit decision event to Kafka.

**Cost-based variation:**

- The Lua script takes a `cost` parameter (Section 6).
- The route table tells the limiter how many units to charge: `cost = route_cost_table[matched_route]`.
- A search request costs 5 units; a health check costs 0 (free pass).

**Background config refresh:**

- Every 60 seconds, each gateway pulls the config snapshot from the config service.
- Config is loaded into a versioned in-memory map. Lookups are O(1).
- On parse error or fetch failure, gateway keeps the last-known-good config; logs a warning.

### 9. Scaling

#### a. Redis sharding

- Shard by client key hash. `rl:apikey:{xyz}:search:...` routes all of client xyz's counters to the same shard, because the `{xyz}` hash tag forces it.
- Why same-shard for one client: the Lua script needs both current and previous window keys atomically. If they were on different shards, you'd need a multi-shard transaction, which Redis Cluster does not support.
- 6 shards at the start. ~50K ops/sec per shard, well under Redis limits. Scaling is by adding shards and migrating keys via Redis Cluster reshard.
- One replica per shard for failover.

#### b. Read replicas

- Counter reads (just GET, no INCR) can hit replicas. We do not, because the Lua script needs both reads and writes atomically and Redis replicas are eventually consistent. Single-primary per shard is the model.

#### c. Per-node LRU

- 10K-entry LRU per gateway node. At 100 bytes per entry = 1MB per node. Trivial.
- Entry TTL: 100ms. Long enough to absorb a burst from one client; short enough that a recovering client is not penalized.
- Hit rate: highest for abusers (every request to a known-over-limit client fast-fails) and for popular API keys with sustained traffic.

#### d. Pipelining

- For requests with multiple keys to check, all EVALSHAs go in one Redis pipeline call. One network round trip for N checks.
- For high-volume endpoints, the gateway can batch checks across multiple incoming requests within a 1ms window, but this adds latency that often is not worth it.

#### e. Geo-distribution

- Per-region Redis cluster. Counters are not synced across regions.
- A client routed to us-east has a separate counter from the same client routed to eu-west. This is intentional: cross-region sync adds 100ms+ to every check.
- The practical effect: a client using two regions concurrently could get up to 2x their limit. Most clients use one region; the 2x is academic.
- For "global" enterprise limits (rare): use a separate central counter store, accept the latency penalty, document the trade-off.

### 10. Reliability

The interesting failure modes:

**Redis primary failure.** Sentinel or Cluster promotes the replica. ~10-30s window of failover. During this window, Lua scripts fail; gateway falls into degraded mode. Choose fail-open or fail-closed per route.

**Network partition between gateway and Redis.** Treat as Redis-down from that gateway's perspective. Other gateways still have Redis. The partitioned gateway should also fail its health check so the LB stops routing to it. This is the single most effective defense: do not serve traffic from a node that cannot enforce limits.

**Slow Redis.** P99 latency on the Lua script climbs to 50ms. The whole API is slower. Mitigation: a per-call timeout (e.g. 5ms) with a circuit breaker. After N consecutive timeouts, the gateway flips to degraded mode for that key class.

**Full cluster outage.** Configurable per route:

| Route class | Fail mode | Rationale |
|-------------|-----------|-----------|
| `/api/search`, `/api/list` (read) | fail-open | API uptime matters more than precise limit enforcement |
| `/api/transfer`, `/api/pay` (money) | fail-closed | Abuse risk outweighs uptime |
| `/api/login` (auth) | fail-closed | Brute force protection is the whole point |
| `/api/webhook` (callback) | fail-open with stricter local cap | Webhooks must keep flowing; cap to prevent runaway |

The fail-mode for each route is in the config service, defaulting to fail-open. Engineering manager and security team both sign off on any fail-closed designation; it is a deliberate choice.

**Memory pressure on Redis.** Keys have a 2× window TTL, so old keys self-evict. We do not use `maxmemory-policy allkeys-lru` for the limiter cluster; we want explicit TTLs. If memory pressure is real, we have too many active clients, not a configuration problem.

**Lua script bug.** A bad script update could lock up Redis. Mitigation: scripts are versioned; gateway can roll back the SCRIPT_HASH it sends. Canary the new script to one gateway, watch metrics for 10 minutes, then roll out.

**Counter drift after failover.** If a Redis primary fails before AOF sync, a few counters are lost on failover. The lost counters reset to 0; affected clients get a tiny extra allowance. Acceptable.

### 11. Observability

| Metric | Why |
|--------|-----|
| `limiter.check.latency.p99` per gateway | If this climbs over 5ms, something is wrong (slow Redis, bad LUA, etc.) |
| `limiter.rejections_per_sec` by tier | Spikes indicate abuse or a legitimate customer hitting their cap |
| `limiter.allowed_per_sec` by route | Baseline traffic per route |
| `limiter.local_cache_hit_rate` | Should be > 50% for hot keys. Drop suggests churn or attack with varied keys. |
| `limiter.redis_op_rate` | Tracks Redis cluster load; drives sharding decisions |
| `limiter.degraded_mode.fail_open_count` / `fail_closed_count` | Critical alert; means Redis is unreachable from at least one node |
| `limiter.script_eval_errors` | Lua failures (e.g. due to script update bugs) |
| `config.refresh.lag_seconds` | Stale config = wrong limits |
| `redis.cluster.shard_lag` | Replica lag (matters if we ever start reading from replicas) |

**Alerts:**
- Page on: `limiter.check.latency.p99 > 5ms for 5min`; `limiter.degraded_mode.* > 0 for any 1min sample`; `script_eval_errors > 0`.
- Ticket on: rejection rate spike on any tier (could be legitimate growth or an attack); local cache hit rate sustained drop (key churn = attack pattern).

**Dashboards by client:**

A per-client dashboard showing their request rate, rejection rate, headroom against limit, and their tier. This is what the customer success team uses when "the customer says they are being throttled but they should not be." 90% of those tickets resolve with "your client is sending more than your tier allows; here are the headers you ignored."

### 12. Follow-up answers

**1. Multiple keys per request.**

Three keys typically: per-IP, per-API-key, per-user. Check all three in parallel via a single Redis pipeline (one network round trip, three EVALSHAs). Take the **most restrictive** result: if any key rejected, the request is rejected.

The reason for layered keys: each protects against a different abuse vector. Per-API-key protects against a single key hammering. Per-IP protects against a single IP rotating keys. Per-user protects against a single user creating multiple keys.

If you only check one key, you have a gap. If you check all three sequentially, you add 3x the latency. Parallel pipelining is the answer.

**2. Limits per route.**

The route table maps URL patterns to a `route_class`:

```
/api/v1/search         → "search"
/api/v1/users/:id      → "default"
/api/v1/transfer       → "write"
/health                → "free" (always allowed)
```

The rate limit key includes `route_class`:

```
rl:apikey:xyz:search:1716381600   → counter for search calls only
rl:apikey:xyz:default:1716381600  → counter for default calls only
```

Per-tier configs specify limits per route_class:

```yaml
tier: pro
limits:
  search:  100/min
  write:   50/min
  default: 1000/min
```

A single API key has multiple parallel buckets. Search consumption does not eat into default consumption.

The cost of separate buckets: more Redis keys per client. With 4 route classes and 100K clients, that is 400K keys, ~40MB. Still tiny.

**3. Cost-based limits.**

The limit becomes "units per minute" instead of "requests per minute." A search query costs 5 units; a lookup costs 1.

Implementation: the Lua script's `cost` parameter is passed by the gateway. The gateway looks up the cost for the matched route in the cost table:

```python
cost = route_cost_table.get(matched_route_pattern, 1)
result = limiter.check(api_key, route_class, cost=cost)
```

The Lua script does `INCRBY cost` instead of `INCR`, and the "would this push me over?" check is `estimated + cost > limit`.

Cost weights are not arbitrary. They should reflect actual backend cost: search hits Elasticsearch which is 50x more expensive than a Postgres lookup, so search costs 50. The hard part is keeping the cost table aligned with reality; teams change service costs and forget to update the table.

**4. Burst allowance.**

The sliding window counter allows a burst implicitly: at the start of a window when the previous window had zero, the full window's limit is immediately available. For an explicit burst above the window limit, switch the route to **token bucket**.

Configure: `bucket_size = 200, refill_rate = 1000/60 = 16.67 per sec`. The bucket holds 200 tokens; refills at the sustained rate. The first 200 requests in 0 seconds are all allowed. Then the client is throttled to 16.67/sec.

Express in config:

```yaml
tier: pro
limits:
  default:
    algorithm: token_bucket
    bucket_size: 200       # max burst
    refill_rate: 16.67     # per second (= 1000 / 60)
```

Implementation switch is one branch in the limiter: route lookup tells you which algorithm; both have Lua scripts.

**5. IP rotation.**

Per-IP limits fail against botnets. The defense is layered:

- **Account/API-key-level limits.** A botnet rotating IPs but using one stolen API key gets caught at the key level.
- **Behavioral signals.** Across IPs, look at user-agent stability, request path patterns, timing. If 10K IPs all hit `/api/login` with the same UA and pause exactly 1.0s between requests, that is one attacker. Feed this signal into a separate fraud-detection pipeline; have it write to a `blocked_keys` set in Redis that the limiter consults.
- **IP reputation.** Subscribe to a botnet/proxy IP list (commercial feed, e.g. MaxMind). Pre-classify residential proxies and known malicious ASNs. Apply stricter limits to those buckets.
- **Proof-of-work or CAPTCHA challenges.** For unauthenticated endpoints, ask suspicious traffic to solve a challenge before being counted as a normal request. The rate limiter does not implement this; it just hands off to a challenge service when the threshold is hit.

The rate limiter is one defense, not the only one. Treat it as a speed bump, not a wall.

**6. Customer override.**

Stored in `rate_limit_overrides` (Postgres). Gateway pulls a snapshot every 60 seconds.

Staleness: 60 seconds is fine. A customer who buys an upgrade sees their new limit within a minute, not instantly. This is acceptable because (a) limits are continuous, not transactional, and (b) customer success can manually flush the gateway config cache on upgrade if needed.

If the config service is down: gateways use last-known config. The override is "remembered" indefinitely. The risk: a downgrade (override removed) takes effect when the next config fetch succeeds. We accept this.

For very critical override changes (security incident, "we need to block this customer NOW"), there is a separate `blocked_keys` Redis set that the limiter consults on every request. Writing to that set propagates instantly. It is the emergency channel; not the everyday path.

**7. Distributed accuracy / race conditions.**

Two requests at count=99, both arrive at different gateway nodes within 1ms. Both call EVALSHA on Redis.

Because the Lua script is atomic at the Redis shard level, only one of them will read count=99 and increment to 100. The other will read count=100 and reject (assuming limit=100). **The Lua atomicity is the whole defense.** If you used GET + conditional SET from the gateway side, you would have a race. The script bundles read+write into one atomic operation, which is precisely why we use it.

Edge case: the two requests land on two different Redis shards (because they are checking different keys, e.g. per-IP vs per-API-key). They each succeed independently. This is correct: they are checking different limits.

The only race the design does not prevent is the local LRU's brief inconsistency window (~100ms). During that window, a client might get 1-2 extra requests through per gateway node. With 200 nodes, theoretical max overshoot is 400 requests. Practical overshoot in load tests is under 10. We document this; nobody has ever complained.

**8. Pre-warming a known burst.**

Customer says: "at 2 AM tomorrow, we will run a batch job. 10K requests in 60 seconds. Our normal limit is 1K/min."

Three options:

- **Permanent raise.** Set their limit higher. Bad: they could abuse it the rest of the time.
- **Temporary override with `valid_until`.** Write a row to `rate_limit_overrides` with `valid_until = 2026-05-22 02:01:00`. The override is active for that 1-minute window only. Cleaner. **Recommended.**
- **Out-of-band batch endpoint.** Provide a separate `/batch` endpoint with its own (higher) limits, only accessible to customers with a special API key. The customer points their batch job at the batch endpoint. **Best for recurring batch use cases.**

The temporary override is the cleanest mechanism for one-off pre-announced bursts.

A separate concern: pre-warming the limiter itself. If we know a burst is coming, we can pre-populate the local LRU on every gateway so the burst does not stampede Redis. In practice this is overkill for 10K requests over 60 seconds.

**9. "User spam" vs "user got popular".**

A blog post hits Hacker News. The blog's API endpoint sees 10x normal traffic. The limiter blocks them.

Was the block correct? It depends on whether the rate limit was protecting the API or the customer's budget.

- If protecting the API: the limit was set with peak in mind. 10x is over peak. Block is correct.
- If protecting the customer from their own runaway costs: blocking is what they would want, but maybe not at this specific moment of viral attention.

Smarter signals:

- **Trend detection.** Compare the current minute's rate against the customer's 7-day P95. If it is 10x and sustained, that is a step-change, not a normal spike. Send an automated email: "Your API saw a 10x traffic spike; you have been temporarily auto-throttled. Click here to raise your limit." Self-service relief.
- **Distinguish bot from human.** If the 10x is human traffic (mixed user-agents, geo-distributed, varied request paths), it is probably legitimate viral attention. If it is bot traffic (one UA, one geo, repeated path), it is scraping.
- **Cost signal.** If their account balance can pay for the bursting, raise the limit automatically. If they are free-tier, throttle them and tell them.

The smart-signal layer sits next to the limiter, feeds in via config overrides, and is opt-in per customer. It is not the limiter's job to be clever; the limiter just enforces.

**10. Limiter is the bottleneck.**

Observability says the limiter middleware adds 8ms P99. The latency budget was 2ms. Where to look:

1. **Redis latency.** Is the Lua script slow? Run a Redis latency profile (`redis-cli --latency`). If Redis is the bottleneck, check shard CPU, network path, AOF settings (sync writes can stall everything).
2. **Pipelining.** Are checks for multiple keys being done sequentially instead of pipelined? One round trip per key × 3 keys = 3x latency. Should be one pipelined call.
3. **Local LRU misses.** If the local cache is missing on every request, every check hits Redis. Either the cache is too small (raise size) or every request has a unique key (signal of an attack with varied identifiers; investigate).
4. **Lua script complexity.** Has someone added expensive logic to the script? Profile with `SLOWLOG`.
5. **Gateway CPU.** If the gateway pod is CPU-pegged, even fast Redis responses queue. Scale out.
6. **Connection pool exhaustion.** If the Redis connection pool on the gateway is small, requests wait for a free connection. Raise pool size or add nodes.

The first three explain ~90% of latency regressions in practice. The senior answer touches all six.

### 13. Trade-offs and what a senior would mention

- **Per-region vs global limits.** Per-region is the default because cross-region sync is too slow. Global limits exist only for enterprise contracts that require them; implemented via a central counter store with explicit added latency. Document the trade-off; do not pretend global is free.
- **Tier complexity grows.** "Just 3 tiers" becomes 3 tiers × N routes × custom overrides × time-limited bursts. The data model in Section 4 handles this; do not collapse it into one JSON blob per customer because debugging will be miserable.
- **Abuse detection is a separate system, not part of the limiter.** The limiter enforces; the abuse system decides who to enforce against. Keeping them separate lets each scale independently and lets the abuse system iterate on signals without touching the hot path.
- **Why not put the limiter in front of the load balancer.** Tempting: catch abuse before it costs us a load balancer connection. But the limiter needs application identity (API key, user_id) which is inside the HTTP request. The LB does not parse HTTP. You could put a thin L7 limiter at the edge for IP-only checks and the full limiter at the gateway; this is what big clouds do.
- **Why not use the cloud provider's managed limiter.** AWS WAF, Cloudflare, etc., all offer rate limiting. They are good for IP-level and bot-mitigation use cases. They are not flexible enough for tiered customer pricing or cost-based limits. Most production APIs use both: cloud limiter as outer ring, custom limiter at the gateway as inner ring.
- **What I would revisit at 10x scale.**
  - Move counter state out of Redis to a purpose-built counter store (e.g. ScyllaDB or a custom in-memory grid) once Redis Cluster cost becomes meaningful.
  - Push limit checks to the edge (PoPs) for global APIs. Each PoP has a partial counter; gossip merges them periodically. Accept the inaccuracy.
  - Probabilistic counting (HyperLogLog) for the "distinct clients" signals that feed abuse detection.
  - Move tier resolution into a sidecar proxy (Envoy) so the gateway code does not have to know about tiers at all.

### 14. Common interview mistakes

- **"Just use a token bucket in Redis" with no further structure.** Skipped clarification, did not address sharing across nodes, did not address failure mode. Loses the candidate.
- **Implementing the algorithm at the application layer, not the gateway.** Now every service has to know about rate limits. Centralize at the gateway.
- **Per-IP only.** Fails against NAT (corporate offices, mobile carriers) and against IP rotation. Must mention layered keys.
- **No mention of fail-open vs fail-closed.** The interviewer will ask. Have an opinion.
- **Forgetting `Retry-After`.** Polite clients respect it. You signal you have written real public APIs by including it.
- **"Just put it behind a load balancer with rate limiting built in."** Fine for IP-level. Not fine for tier-based or API-key-based. The interviewer will ask how you handle the tier; if your only answer is "the LB doesn't do that," you have left the bulk of the problem unsolved.
- **No mention of the local LRU / per-node cache.** At 300K QPS, the round-trip count is the bottleneck. Local caching is the cheapest, biggest win and a senior candidate names it without prompting.
- **Confusing fixed window with sliding window.** A fixed window resets at clock boundaries (100 requests at 11:59:59 + 100 at 12:00:00). A sliding window counts the last N seconds rolling. The boundary doubling problem is the reason sliding exists.
- **Ignoring abuse detection.** Rate limiting and abuse detection are different systems. Acknowledge both even if asked only about the limiter.
- **Drawing one Redis box with no replication, no sharding, no failure plan.** Half the design score is in operational reality.

If you can hit 8 of these, you are interviewing at staff level. Most candidates miss the local LRU, the layered keys, and the fail-mode-per-route designs.
