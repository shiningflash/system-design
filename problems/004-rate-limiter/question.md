---
id: 4
title: Design a Rate Limiter
category: Reliability
topics: [token bucket, sliding window, distributed counters, redis, fail-open]
difficulty: Medium
solution: solution.md
---

## Scene

It is the second round of an onsite. The interviewer manages the API gateway team at a SaaS company. They put one sentence in the doc:

> *Design a rate limiter for our public API.*

They add, almost as an afterthought: "We have free, pro, and enterprise tiers. Each tier has different limits. The API is behind a fleet of gateway servers. Make it work."

This is a deceptively small problem. Candidates often dive into "use a token bucket in Redis" within 30 seconds and then run out of things to say. The interesting parts are not the algorithm. They are: where the limiter sits, what happens when Redis dies, how you share counters across N gateway nodes, and how you avoid making the API slower by checking rate limits on every request.

## Step 1: clarify before you design

Take 5 minutes. Write down at least six questions you would ask before drawing anything. The interviewer is watching whether you treat "rate limiter" as a small problem or as the cross-cutting concern it really is.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Traffic shape.** "What is total API QPS? What is the P99 per-client QPS? How many distinct clients?" Without this you cannot size Redis, decide between in-memory and shared state, or know whether a single shard is enough. A typical answer: 100K total QPS, 100K distinct clients, P99 client at 100 req/sec.
2. **What to limit on.** "Per IP? Per API key? Per user_id? Per route? Some combination?" This is the most important question. A limiter on IP fails the moment a corporate NAT puts 10,000 employees behind one address. A limiter on API key is invisible to abusers who rotate keys. Real systems layer several keys.
3. **Limit semantics.** "Is the limit 100 requests per minute, or 100 in a rolling 60-second window? Bursts allowed?" Fixed windows are easier; sliding windows are more accurate. Bursts are usually allowed (the API needs to feel responsive on short spikes).
4. **Tiers.** "Are limits per-tier static, or per-customer with overrides?" Most real systems have both: a default per tier, plus per-customer overrides for enterprise accounts that negotiated higher limits.
5. **Failure mode.** "If the limiter's backing store is down, do we fail-open (allow everything) or fail-closed (reject everything)?" This is a business question disguised as a technical one. Most public APIs fail-open; payment APIs fail-closed.
6. **Response format.** "When a request is throttled, do we return 429 with `Retry-After`? Are we expected to expose `X-RateLimit-Remaining` to good citizens?" Standardized headers are table stakes for any public API.
7. **Granularity.** "Is the limit on requests, or also on cost? An expensive search query should count more than a cheap lookup." Cost-based limits are common at scale but add complexity.
8. **Latency budget.** "How many milliseconds can the limiter add to each request?" Under 1ms means in-process. Under 5ms means a local Redis hop. Over 10ms is a problem.

If you walked in asking only "what is the limit?" you missed the architecture-defining questions. The granularity (what to limit on) and the failure mode (fail-open vs closed) shape everything that follows.

</details>

## Step 2: capacity estimates

Suppose the interviewer answers:

- 100K total API QPS, peak 300K
- 100K distinct API keys
- Limits: 100 req/min free, 1000 req/min pro, 10000 req/min enterprise
- Most clients are at 1-10% of their limit; a small fraction sit at 80%+
- The gateway fleet has 200 nodes
- Limit checks must add no more than 2ms P99 to each request

Compute these before revealing:

1. Counter updates per second
2. State size: how much memory to track 100K clients
3. Per-node QPS the limiters must handle locally
4. Redis read+write rate if every check is a round trip
5. The wire cost of one check (request + response bytes)

<details>
<summary><b>Reveal: the math</b></summary>

**Counter updates per second.**
Every request increments a counter. 300K QPS peak = **300K counter writes/sec**. If we keep a few counters per client (per-minute, per-hour, per-route), multiply by 3 to 5. Call it ~1M Redis ops/sec at peak across the cluster.

**State size.**
100K clients × ~3 counters each × ~100 bytes per counter record (key + integer value + TTL metadata) = **30MB**. Negligible. Fits in one Redis node. We will still shard for availability, not capacity.

**Per-node QPS.**
300K / 200 gateway nodes = **1500 QPS per gateway node**. Each node must perform 1500 rate limit checks per second. Trivial in-process, but each one talking to Redis is 1500 network round trips per node, per second.

**Redis read+write rate.**
If every check is a Redis call: 300K req/sec × ~2 ops (read counter + conditional increment) = **600K ops/sec**. A single Redis instance does ~100K ops/sec comfortably, so we need to shard, or batch, or cache locally.

**Wire cost per check.**
Redis INCR + EXPIRE pipelined: ~80 bytes request, ~40 bytes response. At 300K QPS that is **~36MB/sec of internal traffic**. Not a problem, but worth noting that the limiter generates real load on its own dependencies.

**Key insight from the math.** State is tiny. The cost is round-trip count, not data volume. The whole design pressure is around *how many times we touch the backing store per request*, not how much we store.

</details>

## Step 3: choose the algorithm

There are five common rate-limiting algorithms. Each makes a different trade between memory, accuracy, and burst handling. Before reading on, list them and the trade-off each makes. Then write a one-line scenario where each one is the wrong choice.

The five:

- **Token bucket**
- **Leaky bucket**
- **Fixed window counter**
- **Sliding window log**
- **Sliding window counter**

<details>
<summary><b>Reveal: comparison table</b></summary>

| Algorithm | How it works | Memory per client | Burst behavior | Boundary accuracy | When wrong |
|-----------|-------------|------------------|----------------|-------------------|-----------|
| **Token bucket** | A bucket holds up to N tokens. Refilled at rate R tokens/sec. Each request consumes one token; if empty, reject. | ~16 bytes (token count + last refill ts) | Allows bursts up to bucket size, then throttles to refill rate. | N/A (continuous) | When you want strict no-bursts (a leaky bucket is better). |
| **Leaky bucket** | Requests queue into a bucket. Bucket drains at fixed rate R. If bucket is full, reject. | ~16 bytes + the queue itself | Smooths spikes by queuing, not rejecting. Output is strictly constant. | N/A (continuous) | When latency matters more than smoothing. Queues add delay. |
| **Fixed window counter** | Count requests in the current N-second window. Window resets at fixed clock boundaries. | ~16 bytes | Allows 2x burst at boundary. | Bad: a client can send N requests at 11:59:59 and N more at 12:00:00. | When you advertise "100/min" and clients notice the boundary doubling. |
| **Sliding window log** | Store the timestamp of every request in the last window. Count >= N → reject. | O(N) per client (can be MB for high limits) | Exact. | Best possible. | When N is large. 10,000 timestamps × 8 bytes = 80KB per client × 100K clients = 8GB. |
| **Sliding window counter** | Weighted average of current and previous fixed window. Approximates true sliding without storing every request. | ~24 bytes (two counters + ts) | Close to ideal, no boundary doubling. | ~99% accurate vs true sliding window. | When you need exact correctness (rare; the approximation is fine for billing-grade APIs). |

**The recommendation: sliding window counter.**

Reasons:

- Memory is O(1) per client, like fixed window. No per-request storage.
- Boundary doubling is gone. The weighted average smooths across the boundary.
- Implementable in a Redis Lua script in ~10 lines, atomic, P99 under 1ms.
- The 1% inaccuracy is invisible to clients and acceptable for almost every use case.

Token bucket is a strong second choice and is what AWS, Stripe, and most production APIs actually use. It is harder to implement atomically in a distributed setting (the refill calculation is stateful) but maps cleanly onto a Lua script too.

The interviewer is testing whether you know *why* a sliding window counter exists, not which algorithm has the prettiest name. The "why" is: fixed window has a boundary problem, sliding window log uses too much memory, and the counter is the cheap approximation that fixes both.

</details>

## Step 4: incomplete architecture diagram

Fill in the `[ ? ]` placeholders. Decide:

- Where the rate limiter sits (gateway, sidecar, in-process library?)
- Where state lives (in-process map, local Redis, central Redis, both?)
- What happens when the state store is unreachable
- How counters are kept consistent across N gateway nodes

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
                  │  (200 nodes)   │  as an in-process middleware.
                  │                │  Each node has a small local LRU
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
   tier-based ceiling (e.g. burst × 1.5). This is fail-open with a safety
   net: we keep serving, but each node enforces independently with no
   coordination. Reports a critical alert immediately.
```

Why each piece is here:

- **L7 load balancer.** Not the place for rate limiting. LBs operate at connection level; rate limiting needs application-level identity (API key, user_id). Putting limits at the LB only covers per-IP, which is the weakest defense.
- **Gateway with in-process middleware.** The limiter is a library, not a service. Calling out to a separate "rate limiter service" adds a network hop per request, doubling the latency budget. In-process middleware + remote Redis is the standard.
- **Per-node LRU.** A tiny local cache of "this key was just at N requests, no need to ask Redis again for 100ms" cuts Redis QPS by 80%+ for hot clients. We come back to this in Step 5.
- **Redis cluster.** Shared state across nodes. Sharded by client key hash, replicated for failover. Lua scripts are evaluated atomically, which is what makes "check counter and decrement" safe under concurrent requests.
- **Cold archive.** When you need to investigate why client X got blocked yesterday, in-Redis counters with TTL have already expired. Stream the rejected events to S3 + ClickHouse for forensics.

</details>

## Step 5: distributed state

You have 200 gateway nodes. Each one is processing rate limit checks. How do you share counters across them so a client cannot get N requests through each node and effectively get 200N total?

Three approaches. Write the pros and cons of each, then read on.

- **Central Redis. Every check is a remote call.**
- **In-process counters with periodic gossip.** Each node keeps its own counters; nodes broadcast their counts every few seconds.
- **Local-first with remote reconciliation.** Each node has a local counter, drains to Redis on a short interval, reads back the global value periodically.

<details>
<summary><b>Reveal: comparison and recommended approach</b></summary>

| Approach | Accuracy | Latency overhead | Failure behavior | When to use |
|----------|---------|------------------|------------------|------------|
| **Central Redis every check** | Exact (Lua script makes it atomic) | 1-2ms per request | Redis down = limiter down. Must fail-open or fail-closed. | Default. The right answer at this scale. |
| **Periodic gossip** | Lags. A client can burst N×200 before the gossip catches up. | Sub-millisecond | No coordination needed; nodes survive independently. | When latency budget is brutally tight and limits are soft (advisory, not security). |
| **Local-first with reconciliation** | Approximate. Drift bounded by reconcile interval. | Sub-millisecond on hot path; one Redis hop per N requests in background. | Survives short Redis outages. Drift grows linearly with outage length. | When you want both low latency and central state but can tolerate ~10% drift. |

**The recommendation: central Redis with Lua scripts, plus per-node LRU to suppress hot-key check traffic.**

The flow on every request:

1. Compute the rate limit key, e.g. `rl:{api_key}:{route}:{minute_window}`.
2. Check local LRU. If the key is in the cache and the cached count is already over the limit, reject immediately. **No Redis call.**
3. Else, execute the Lua script on Redis. Script does: `INCR key; if new value > limit, return rejected; else EXPIRE key window_seconds; return allowed + remaining`. Atomic.
4. Update local LRU with the new count. Used for fast-path rejection on the next request.

The local LRU is the trick. For a client hammering at 100x their limit, only the first few requests in each window hit Redis; everything else fast-fails locally. Cuts Redis QPS by 90%+ for abusive clients, which is exactly when you need the savings.

For accuracy: requests within the same millisecond can race past the LRU before its update lands. This means a client might briefly exceed their limit by ~1-2 requests per node before being throttled. With 200 nodes and a 100/min limit, the theoretical overshoot is 400 requests in a window. In practice, well under 10. We accept this; the user-visible difference is invisible.

When Redis fails: see Step 5.5 below.

</details>

## Step 5.5: what happens when Redis dies

This is the question that separates senior from mid-level candidates. Walk through the system behavior in three scenarios:

1. One Redis primary shard goes down. Its replica takes over.
2. The whole Redis cluster is unreachable from a specific gateway node (network partition).
3. The whole Redis cluster is unreachable from every gateway node.

<details>
<summary><b>Reveal: failure handling</b></summary>

**Scenario 1: single shard failover.** The replica is promoted. There is a ~5 to 30 second window where some keys may be served stale. Acceptable. Mitigation: use Redis Sentinel or Redis Cluster's automatic failover; the gateway library retries on connection errors.

**Scenario 2: one node loses Redis connectivity but others have it.** That node alone has no shared state. Its limiter must fall back to per-node counters. The other nodes are unaffected. The single broken node now under-counts (it does not see traffic from the other 199 nodes), so a client routed to it can effectively get N×(some multiplier) requests through. Mitigation: the node's limiter sets a stricter local ceiling (e.g. burst × 1.5 of the tier limit) to bound the damage. Also: the load balancer marks the node unhealthy if it loses Redis, and traffic shifts to healthy nodes.

**Scenario 3: Redis cluster fully down.** This is the disaster scenario. Two choices:

- **Fail-open.** Every gateway node continues using its local LRU as the source of truth. Limits become per-node, not global. A user might get 200x their stated limit. The API stays up. **This is the right default for public read APIs.**
- **Fail-closed.** Every check returns "rejected." The API stops serving traffic. **This is the right default for write-heavy payment-grade APIs** where the limits are protecting against abuse that could cost real money.

The choice should be configurable per route. `/api/search` fails open; `/api/transfer-money` fails closed.

Whichever choice you make, the failure must be loud:
- Page on Redis cluster availability.
- Emit a `limiter.degraded_mode` metric per gateway node. If it spikes, you know.
- A separate canary client tests the limiter actually rejects when over limit; alerts if the limiter is silently broken.

The bad answer: "we just use Redis." Cannot answer "what happens when Redis goes down" and the interview is over.

</details>

## Step 6: response and headers

The limiter has decided to reject the request. What do you return? What do well-behaved clients need from you so they can back off correctly?

<details>
<summary><b>Reveal: response format</b></summary>

**HTTP status:**

- `429 Too Many Requests`. The standard. Do not use 503; that signals server overload, which is different.

**Response headers (required):**

```
HTTP/1.1 429 Too Many Requests
Retry-After: 28                          # seconds until they can retry
X-RateLimit-Limit: 100                   # the limit for this key+route
X-RateLimit-Remaining: 0                 # always 0 on a 429
X-RateLimit-Reset: 1716381660            # unix ts when the window resets
Content-Type: application/json
```

**Response body:**

```json
{
  "error": "rate_limited",
  "message": "Request exceeds rate limit of 100 requests per minute.",
  "retry_after_seconds": 28,
  "doc_url": "https://api.example.com/docs/rate-limits"
}
```

**Headers on allowed requests too:**

Send `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` on every response, not just 429s. Good clients use these to self-throttle before they ever get rejected. The cost is ~30 extra bytes per response, which is nothing.

**Retry-After semantics:**

- For a sliding window: `Retry-After` is the time until enough of the current window has rolled off that the next request would succeed. Not the time until the entire window resets.
- For a token bucket: `Retry-After` is the time until the next token is refilled.
- Compute it server-side; do not make the client guess.

**Doc URL:**

Always include a link to the rate-limit documentation. This is the single most effective thing you can do to reduce support tickets. When developers see the URL in the error response, half of them go read the docs instead of opening a ticket.

</details>

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. **Multiple keys per request.** A request has an IP, an API key, and a user_id. Do you check all three rate limits in parallel or in sequence? What if the API key limit allows it but the IP limit doesn't?
2. **Limits per route.** `/api/search` is expensive; `/api/health` is cheap. Same client. How do you express this in your data model and check logic?
3. **Cost-based limits.** Instead of "100 requests per minute," the limit is "100 units per minute" and a search query costs 5 units while a lookup costs 1. How does the limiter change?
4. **Burst allowance.** A client should be able to send a burst of 200 in the first 10 seconds of a minute but still total 1000 in that minute. Which algorithm supports this and how do you configure it?
5. **IP rotation.** An abuser is rotating through 10,000 residential IPs (botnet). Per-IP limits are useless. What do you do? Hint: layered keys, behavioral signals.
6. **Customer override.** Enterprise customer X negotiated a 100x limit. Where is that override stored? How fresh must it be? What if the override service is down?
7. **Distributed accuracy.** Two requests from the same client land on two gateway nodes within 1ms. Both check Redis, both get count=99 (under 100 limit), both INCR. Now count=101 and both were allowed. What is your defense?
8. **Pre-warming a known burst.** A customer says they will run a batch job at 2:00 AM that needs 10,000 requests in 60 seconds, way over their normal limit. How do you accommodate it without raising their permanent limit?
9. **Telling the difference between "user spam" and "user got popular".** A blog post gets posted on HN and suddenly receives 10x traffic to a user's API endpoint. The limiter blocks them. Is that correct? How would you build a smarter signal?
10. **Limiter is the bottleneck.** Observability shows the limiter middleware itself is adding 8ms to P99 latency. Where do you look first?

## Related problems

- **[URL Shortener (001)](../001-url-shortener/question.md)**. The `POST /links` endpoint sits behind exactly this kind of rate limiter; the design integrates directly with that system.
- **[Distributed Cache (009)](../009-distributed-cache/question.md)**. The Redis cluster backing the limiter is itself a distributed cache; the eviction, sharding, and replication choices there determine the limiter's failure modes.
- **[News Feed (002)](../002-news-feed/question.md)**. The `POST /posts` endpoint and the timeline read path both benefit from rate limiting; the cost-based limit pattern in follow-up 3 is what feeds use to throttle expensive ranking queries.
