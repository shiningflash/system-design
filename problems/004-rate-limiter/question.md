---
id: 4
title: Design a Rate Limiter
category: Reliability
topics: [token bucket, sliding window, distributed counters, redis, fail-open]
difficulty: Medium
solution: solution.md
---

## Scene

Second round of an onsite. The interviewer runs the API gateway team at a SaaS company. They put one sentence in the doc:

> *Design a rate limiter for our public API.*

Almost as an afterthought: "We have free, pro, and enterprise tiers. Each tier has different limits. The API is behind a fleet of gateway servers. Make it work."

This problem looks small and isn't. Candidates jump to "token bucket in Redis" inside 30 seconds and then run out of things to say. The algorithm is rarely the interesting part. What matters is where the limiter sits, what happens when Redis dies, how you share counters across N gateway nodes, and how you avoid making every API call slower because of the check itself.

## Step 1: clarify before you design

Take five minutes. Write down at least six questions you'd ask before drawing anything. The interviewer is watching whether you treat "rate limiter" as a small problem or as the cross-cutting concern it actually is.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Traffic shape.** What is total API QPS? P99 per-client QPS? How many distinct clients? Without this you can't size Redis, decide between in-memory and shared state, or know whether one shard is enough. Typical answer: 100K total QPS, 100K distinct clients, P99 client at 100 req/sec.
2. **What you limit on.** Per IP? Per API key? Per user_id? Per route? Some combination? This is the single most important question. A limiter on IP fails the moment a corporate NAT puts 10,000 employees behind one address. A limiter on API key is invisible to abusers who rotate keys. Real systems layer several keys.
3. **Limit semantics.** Is it 100 requests per minute, or 100 in a rolling 60-second window? Bursts allowed? Fixed windows are easier; sliding windows are more accurate. Bursts are usually allowed so the API feels responsive on short spikes.
4. **Tiers.** Static per tier, or per customer with overrides? Most real systems have both: a default per tier, plus per-customer overrides for enterprise accounts that negotiated higher limits.
5. **Failure mode.** If the limiter's backing store is down, do we fail-open (allow everything) or fail-closed (reject everything)? Business question disguised as a technical one. Most public APIs fail-open; payment APIs fail-closed.
6. **Response format.** On a throttle, do we return 429 with `Retry-After`? Do we expose `X-RateLimit-Remaining` for well-behaved clients? Standardized headers are table stakes.
7. **Granularity.** Limit on request count, or on cost? An expensive search query should weigh more than a cheap lookup. Cost-based limits are common at scale and add complexity.
8. **Latency budget.** How many milliseconds can the limiter add per request? Under 1ms means in-process. Under 5ms means a local Redis hop. Over 10ms is a problem.

If you walked in asking only "what is the limit?" you missed the architecture-defining questions. Granularity (what you limit on) and failure mode shape everything that follows.

</details>

## Step 2: capacity estimates

Say the interviewer answers:

- 100K total API QPS, peak 300K
- 100K distinct API keys
- Limits: 100 req/min free, 1000 req/min pro, 10000 req/min enterprise
- Most clients are at 1-10% of their limit; a small fraction sit at 80%+
- 200 gateway nodes in the fleet
- Limit checks add no more than 2ms P99 per request

Compute these before peeking:

1. Counter updates per second
2. State size to track 100K clients
3. Per-node QPS each limiter has to handle
4. Redis read+write rate if every check is a round trip
5. Wire cost of one check (request + response bytes)

<details>
<summary><b>Reveal: the math</b></summary>

**Counter updates per second.**
Every request increments a counter. 300K QPS peak = **300K counter writes/sec**. Keep a few counters per client (per-minute, per-hour, per-route) and multiply by 3 to 5. Call it ~1M Redis ops/sec at peak across the cluster.

**State size.**
100K clients x ~3 counters each x ~100 bytes per counter record (key + integer + TTL metadata) = **30MB**. Negligible. Fits on one Redis node. We'll still shard for availability, not capacity.

**Per-node QPS.**
300K / 200 gateway nodes = **1500 QPS per gateway node**. Each node performs 1500 limit checks per second. Trivial in-process. Each one talking to Redis is 1500 network round trips per node per second.

**Redis read+write rate.**
If every check is a Redis call: 300K req/sec x ~2 ops (read counter + conditional increment) = **600K ops/sec**. A single Redis instance handles ~100K ops/sec comfortably, so we shard, batch, or cache locally.

**Wire cost per check.**
Redis INCR + EXPIRE pipelined: ~80 bytes request, ~40 bytes response. At 300K QPS that's **~36MB/sec of internal traffic**. Not a problem, but worth noting the limiter generates real load on its own dependencies.

**Key insight from the math.** State is tiny. The cost is round-trip count, not data volume. The whole design pressure is around *how many times we touch the backing store per request*, not how much we store.

</details>

## Step 3: choose the algorithm

Five common algorithms. Each makes a different trade between memory, accuracy, and burst handling. Before reading on, list them and the trade each makes. Then write one scenario where each is the wrong choice.

The five:

- Token bucket
- Leaky bucket
- Fixed window counter
- Sliding window log
- Sliding window counter

<details>
<summary><b>Reveal: comparison table</b></summary>

| Algorithm | How it works | Memory per client | Burst behavior | Boundary accuracy | When wrong |
|-----------|-------------|------------------|----------------|-------------------|-----------|
| Token bucket | A bucket holds up to N tokens. Refilled at rate R tokens/sec. Each request consumes one; empty bucket means reject. | ~16 bytes (token count + last refill ts) | Allows bursts up to bucket size, then throttles to refill rate. | N/A (continuous) | When you want strict no-bursts. Use leaky bucket. |
| Leaky bucket | Requests queue into a bucket. Bucket drains at fixed rate R. Full bucket means reject. | ~16 bytes + the queue | Smooths spikes by queuing rather than rejecting. Output is strictly constant. | N/A (continuous) | When latency matters more than smoothing. Queues add delay. |
| Fixed window counter | Count requests in the current N-second window. Window resets at fixed clock boundaries. | ~16 bytes | Allows 2x burst at boundary. | Bad: a client can send N at 11:59:59 and N more at 12:00:00. | When you advertise "100/min" and clients notice the doubling. |
| Sliding window log | Store the timestamp of every request in the last window. Count >= N means reject. | O(N) per client (can be MB for high limits) | Exact. | Best possible. | When N is large. 10,000 timestamps x 8 bytes = 80KB per client x 100K clients = 8GB. |
| Sliding window counter | Weighted average of current and previous fixed window. Approximates true sliding without storing every request. | ~24 bytes (two counters + ts) | Close to ideal. No boundary doubling. | ~99% accurate vs true sliding. | When you need exact correctness (rare; the approximation is fine even for billing-grade APIs). |

**My recommendation: sliding window counter.**

- Memory is O(1) per client, like fixed window. No per-request storage.
- Boundary doubling is gone. The weighted average smooths across the boundary.
- Implementable in a Redis Lua script in ~10 lines, atomic, P99 under 1ms.
- The 1% inaccuracy is invisible to clients and acceptable for almost every use case.

Token bucket is a strong second choice and is what AWS, Stripe, and most production APIs actually use. It's harder to implement atomically in a distributed setting (the refill calculation is stateful) but maps cleanly onto a Lua script too.

The interviewer is testing whether you know *why* a sliding window counter exists, not which algorithm has the prettiest name. The why: fixed window has a boundary problem, sliding window log uses too much memory, the counter is the cheap approximation that fixes both.

</details>

## Step 4: incomplete architecture diagram

Fill in the `[ ? ]` placeholders. Decide:

- Where the rate limiter sits (gateway, sidecar, in-process library?)
- Where state lives (in-process map, local Redis, central Redis, both?)
- What happens when the state store is unreachable
- How counters stay consistent across N gateway nodes

```
   Client ────►   ┌────────────┐
                  │  [ ? ]     │  (terminates TLS, routes by host)
                  └──────┬─────┘
                         │
                         ▼
                  ┌────────────┐
                  │   [ ? ]    │  (where does the limiter run?)
                  │            │  ┌───────────────┐
                  │            ├─►│  [ ? ]        │  (shared state? local? both?)
                  │            │  └───────────────┘
                  └──────┬─────┘
                         │  allowed? → forward
                         │  rejected? → 429
                         ▼
                  ┌────────────┐
                  │ API Service│
                  └────────────┘

   Failure path: if [ ? ] is unreachable, the limiter must decide:
                 fail-open (allow) or fail-closed (reject)?
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
   Client ────►   ┌────────────────┐
                  │  Load Balancer │  Anycast IP, TLS termination
                  │  (L7)          │  Health-checks gateway nodes
                  └────────┬───────┘
                           │
                           ▼
                  ┌────────────────┐
                  │  API Gateway   │  Stateless. Runs the rate limiter
                  │  (200 nodes)   │  as in-process middleware.
                  │                │  Each node keeps a small local LRU
                  │  ┌──────────┐  │  cache of recent counter values.
                  │  │ Limiter  │  │
                  │  │ middlewar│──┼─────────┐
                  │  └──────────┘  │         │ Lua script:
                  └────────┬───────┘         │ atomic INCR + EXPIRE +
                           │                 │ window math, returns
                           │ allowed         │ (allowed, remaining, reset_at)
                           ▼                 ▼
                  ┌────────────────┐  ┌────────────────────┐
                  │  API Service   │  │   Redis Cluster    │
                  │                │  │  Sharded by key    │
                  └────────────────┘  │  hash. Replicated. │
                                      │  6 primary shards, │
                                      │  one replica each. │
                                      └─────────┬──────────┘
                                                │
                                                │ async dump
                                                ▼
                                      ┌────────────────────┐
                                      │  Cold counter      │  For abuse forensics:
                                      │  archive (S3 + CH) │  reconstruct a bad
                                      │                    │  actor's rate history.
                                      └────────────────────┘

   Failure path: if Redis cluster is unreachable from a gateway node,
   that node falls back to per-node in-process counters with a stricter
   tier-based ceiling (e.g. burst x 1.5). Fail-open with a safety net:
   we keep serving, each node enforces independently with no
   coordination. Critical alert immediately.
```

Why each piece is here:

- L7 load balancer isn't the place for rate limiting. LBs operate at connection level; rate limiting needs application-level identity (API key, user_id). Limits at the LB only cover per-IP, which is the weakest defense.
- Gateway with in-process middleware. The limiter is a library, not a service. Calling out to a separate "rate limiter service" adds a network hop per request and doubles the latency budget. In-process middleware plus remote Redis is the standard.
- Per-node LRU. A tiny local cache of "this key was just at N requests, don't ask Redis for 100ms" cuts Redis QPS by 80%+ for hot clients. More on this in Step 5.
- Redis cluster. Shared state across nodes. Sharded by client key hash, replicated for failover. Lua scripts evaluate atomically, which is what makes "check counter and decrement" safe under concurrent requests.
- Cold archive. When you investigate why client X got blocked yesterday, in-Redis counters with TTL have already expired. Stream rejected events to S3 + ClickHouse for forensics.

</details>

## Step 5: distributed state

200 gateway nodes. Each one is processing rate limit checks. How do you share counters across them so a client can't get N requests through each node and effectively get 200N total?

Three approaches. Write the pros and cons of each, then read on.

- Central Redis. Every check is a remote call.
- In-process counters with periodic gossip. Each node keeps its own counters; nodes broadcast their counts every few seconds.
- Local-first with remote reconciliation. Each node has a local counter, drains to Redis on a short interval, reads back the global value periodically.

<details>
<summary><b>Reveal: comparison and recommendation</b></summary>

| Approach | Accuracy | Latency overhead | Failure behavior | When to use |
|----------|---------|------------------|------------------|------------|
| Central Redis every check | Exact (Lua script makes it atomic) | 1-2ms per request | Redis down = limiter down. Must fail-open or fail-closed. | Default. The right answer at this scale. |
| Periodic gossip | Lags. A client can burst Nx200 before gossip catches up. | Sub-millisecond | No coordination; nodes survive independently. | When latency budget is brutally tight and limits are soft (advisory, not security). |
| Local-first with reconciliation | Approximate. Drift bounded by reconcile interval. | Sub-millisecond on hot path; one Redis hop per N requests in background. | Survives short Redis outages. Drift grows linearly with outage length. | When you want both low latency and central state and can tolerate ~10% drift. |

**My recommendation: central Redis with Lua scripts, plus per-node LRU to suppress hot-key check traffic.**

Per-request flow:

1. Compute the key, e.g. `rl:{api_key}:{route}:{minute_window}`.
2. Check local LRU. If the cached count is already over the limit, reject immediately. **No Redis call.**
3. Otherwise execute the Lua script on Redis. Script does: `INCR key; if new value > limit, return rejected; else EXPIRE key window_seconds; return allowed + remaining`. Atomic.
4. Update local LRU with the new count. Used for fast-path rejection next time.

The local LRU is the trick. For a client hammering at 100x their limit, only the first few requests per window hit Redis; everything else fast-fails locally. Cuts Redis QPS by 90%+ for abusive clients, which is exactly when you need the savings.

Accuracy: requests within the same millisecond can race past the LRU before its update lands. A client might briefly exceed their limit by ~1-2 requests per node before being throttled. With 200 nodes and a 100/min limit, the theoretical overshoot is 400 requests in a window. In practice, well under 10. We accept this; the user-visible difference is invisible.

When Redis fails: see Step 5.5.

</details>

## Step 5.5: what happens when Redis dies

This is the question that separates senior from mid-level. Walk through three scenarios:

1. One Redis primary shard goes down. Its replica takes over.
2. The whole Redis cluster is unreachable from a specific gateway node (network partition).
3. The whole Redis cluster is unreachable from every gateway node.

<details>
<summary><b>Reveal: failure handling</b></summary>

**Scenario 1: single shard failover.** Replica promoted. ~5 to 30 second window where some keys may be served stale. Acceptable. Use Redis Sentinel or Cluster's automatic failover; the gateway library retries on connection errors.

**Scenario 2: one node loses Redis connectivity, others have it.** That node alone has no shared state. Its limiter falls back to per-node counters. Other nodes are unaffected. The broken node now under-counts (it doesn't see traffic from the other 199 nodes), so a client routed there can get Nx(some multiplier) requests through. The node's limiter sets a stricter local ceiling (e.g. burst x 1.5) to bound the damage. The load balancer marks the node unhealthy if it loses Redis, traffic shifts elsewhere.

**Scenario 3: Redis cluster fully down.** Disaster. Two choices:

- **Fail-open.** Every gateway uses its local LRU as the source of truth. Limits become per-node, not global. A user might get 200x their stated limit. The API stays up. **Right default for public read APIs.**
- **Fail-closed.** Every check returns rejected. The API stops serving. **Right default for write-heavy payment-grade APIs** where the limits protect against abuse that costs real money.

Choice should be configurable per route. `/api/search` fails open; `/api/transfer-money` fails closed.

Whichever you pick, the failure must be loud:

- Page on Redis cluster availability.
- Emit a `limiter.degraded_mode` metric per gateway. Spikes mean you know.
- A separate canary client tests that the limiter actually rejects when over limit; alerts if the limiter is silently broken.

The bad answer: "we just use Redis." Can't answer "what happens when Redis goes down" and the interview is over.

</details>

## Step 6: response and headers

The limiter has decided to reject. What do you return? What do well-behaved clients need from you so they can back off correctly?

<details>
<summary><b>Reveal: response format</b></summary>

HTTP status:

- `429 Too Many Requests`. The standard. Don't use 503; that signals server overload, which is different.

Headers (required):

```
HTTP/1.1 429 Too Many Requests
Retry-After: 28                          # seconds until they can retry
X-RateLimit-Limit: 100                   # the limit for this key+route
X-RateLimit-Remaining: 0                 # always 0 on a 429
X-RateLimit-Reset: 1716381660            # unix ts when the window resets
Content-Type: application/json
```

Body:

```json
{
  "error": "rate_limited",
  "message": "Request exceeds rate limit of 100 requests per minute.",
  "retry_after_seconds": 28,
  "doc_url": "https://api.example.com/docs/rate-limits"
}
```

Headers on allowed requests too:

Send `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` on every response, not just 429s. Good clients use these to self-throttle before they ever get rejected. The cost is ~30 extra bytes per response. Nothing.

`Retry-After` semantics:

- For a sliding window: time until enough of the current window has rolled off that the next request would succeed. Not the time until the entire window resets.
- For a token bucket: time until the next token refills.
- Compute it server-side. Don't make the client guess.

Doc URL:

Always include a link to the rate-limit docs. Single most effective thing you can do to reduce support tickets. When developers see the URL in the error, half of them go read the docs instead of opening a ticket.

</details>

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. **Multiple keys per request.** A request has an IP, an API key, and a user_id. Check all three rate limits in parallel or in sequence? What if the API key limit allows it but the IP limit doesn't?
2. **Limits per route.** `/api/search` is expensive; `/api/health` is cheap. Same client. How do you express this in your data model and check logic?
3. **Cost-based limits.** Instead of "100 requests per minute," the limit is "100 units per minute," and a search query costs 5 units while a lookup costs 1. How does the limiter change?
4. **Burst allowance.** A client should be able to send a burst of 200 in the first 10 seconds of a minute but still total 1000 in that minute. Which algorithm supports this and how do you configure it?
5. **IP rotation.** An abuser is rotating through 10,000 residential IPs (botnet). Per-IP limits are useless. What do you do? Hint: layered keys, behavioral signals.
6. **Customer override.** Enterprise customer X negotiated a 100x limit. Where is that override stored? How fresh must it be? What if the override service is down?
7. **Distributed accuracy.** Two requests from the same client land on two gateway nodes within 1ms. Both check Redis, both get count=99 (under 100 limit), both INCR. Now count=101 and both were allowed. What is your defense?
8. **Pre-warming a known burst.** A customer says they'll run a batch job at 2:00 AM that needs 10,000 requests in 60 seconds, way over their normal limit. How do you accommodate it without raising their permanent limit?
9. **"User spam" vs "user got popular".** A blog post hits HN and suddenly receives 10x traffic to a user's API endpoint. The limiter blocks them. Is that correct? How would you build a smarter signal?
10. **Limiter is the bottleneck.** Observability shows the limiter middleware itself is adding 8ms to P99 latency. Where do you look first?

## Related problems

- **[URL Shortener (001)](../001-url-shortener/question.md)**. The `POST /links` endpoint sits behind exactly this kind of rate limiter; the design integrates directly with that system.
- **[Distributed Cache (009)](../009-distributed-cache/question.md)**. The Redis cluster backing the limiter is itself a distributed cache; the eviction, sharding, and replication choices there determine the limiter's failure modes.
- **[News Feed (002)](../002-news-feed/question.md)**. The `POST /posts` endpoint and the timeline read path both benefit from rate limiting; the cost-based limit pattern in follow-up 3 is what feeds use to throttle expensive ranking queries.
