---
id: 17
title: Design a Read-Heavy System (Patterns Walkthrough)
category: Patterns
topics: [caching, read replicas, cdn, materialized views, denormalization]
difficulty: Easy
solution: solution.md
---

## Scene

You sit down. The interviewer slides a sketch across the desk.

> *"I have a service that serves a product catalog. Today: 100 users, a single Postgres, everything works. The P95 for a product page is 30ms. Our PM just told me we are about to onboard 100,000 users from a partner deal. Reads are 100 times writes. What do you do, in order?"*

They pause. "Walk me through the read path as it scales. Name what breaks first, name the fix, and tell me when *not* to apply each fix."

This is the most common shape of a "scale this" interview. The interviewer is not looking for a final architecture. They are looking for whether you reach for caching before you understand the read pattern, whether you add Redis before you know if a CDN would do the same job for less money, and whether you know the difference between a replica that helps read latency and a replica that just moves the problem.

The point of the question is the toolkit: browser cache, CDN, edge cache, in-process cache, distributed cache, read replicas, materialized views, denormalization. And the *order* in which you apply them.

## Step 1: clarify before you scale anything

Take 5 minutes. The scenario sounds well-defined, but it is not. You need numbers before you reach for tools.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Read latency target.** "What is the P95 the user actually feels? 30ms today, but what is acceptable at 100k users? 50ms? 100ms?" If the target is 200ms you can solve a lot with one Redis. If it is 20ms globally, you need a CDN.
2. **Freshness tolerance.** "When a product price changes, how stale can a viewer's price be? 1 second? 60 seconds? 5 minutes?" This decides cache TTL, invalidation strategy, and whether read replicas are acceptable.
3. **Read pattern.** "Is every request a lookup by product ID? Or are there list queries (category browse), search queries (full-text), filter queries (price range)?" Lookup by ID caches well. Full-text search needs a separate index. Filters need a different denormalization.
4. **Read distribution.** "Is the traffic uniform across products, or does 1% of products get 90% of views?" Zipf distribution makes caching trivial. Uniform distribution makes caching almost useless.
5. **Geographic distribution.** "Are users in one country or spread across continents?" Same-region: skip CDN, use Redis. Global: CDN is the cheapest latency win you can buy.
6. **Personalized vs shared content.** "Does every user see the same product page? Or do we show user-specific pricing, recommendations, stock-near-you?" Shared content caches at the CDN. Personalized content cannot.
7. **Write pattern.** "Is the 100x read-to-write ratio uniform, or are writes bursty (nightly batch import of 1M products)?" Bursty writes break cache invalidation strategies that assume a steady trickle.
8. **Consistency requirement per endpoint.** "Is the product page allowed to be stale, but the checkout price must be exact?" Different endpoints get different cache TTLs (or no cache at all).

A common junior mistake: starting with "add Redis." You don't know yet whether Redis is the right tool. The CDN might do 90% of the work. The in-process cache might do the other 9%. Redis might never be needed.

</details>

## Step 2: capacity

The interviewer tells you:

- 100,000 users active per day.
- Each user makes ~50 reads per session (browse + view + filter).
- Reads are 100x writes.
- Catalog: 1M products. Each product ~5KB JSON.
- Read latency target: P95 < 50ms.
- Freshness: 60 seconds is fine for price; stock count can be 5 minutes stale; product description is essentially immutable.
- Traffic distribution: roughly Zipf. Top 10k products serve ~80% of reads.

Compute, on paper, before revealing:

1. Reads per second sustained and peak.
2. Writes per second sustained and peak.
3. Working-set size for the hot 80%.
4. Cache hit rate required to keep DB load under 200 QPS.
5. Bandwidth if every read returns 5KB.

<details>
<summary><b>Reveal: the math</b></summary>

**Reads per second.**
100k users × 50 reads = 5M reads/day. 5M / 86400 ≈ 58 reads/sec sustained. Peak typically 5x sustained for consumer traffic, call it ~300 reads/sec peak.

**Writes per second.**
58 / 100 = 0.58 writes/sec sustained. ~3 writes/sec peak. Trivially small. The catalog is updated by a few admins, not by users.

**Hot working set.**
Top 10k products × 5KB = 50MB. Easily fits in a single Redis node, in-process LRU, or even browser cache for repeat visitors.

**Required cache hit rate.**
Goal: DB stays under 200 QPS even at peak (300 reads/sec). DB serves the misses. Required hit rate = 1 - (200/300) = 33% at minimum to stay under DB capacity. In practice you target 80-95% hit rate so the DB has headroom. With Zipf 80/20 traffic and a cache covering the top 10k products, 80%+ hit rate is the default outcome with TTL of a few minutes.

**Bandwidth.**
300 reads/sec × 5KB = 1.5 MB/sec = 12 Mbps. Negligible at the origin. At the CDN, it scales out per POP. No bandwidth pressure anywhere.

**Key insight.** This is a textbook read-heavy system with a small hot set and a low absolute QPS. Almost any caching scheme works. The interview is not about whether to cache; it is about *which layer* to cache at, in what order, and why.

</details>

## Step 3: the caching pyramid

Before you draw any architecture, draw the layers. A read request, on its way from the user's browser to the database, passes through up to six tiers of cache. Each one is roughly 10x faster than the one below it. Each one also requires invalidation and capacity planning.

Sketch the six layers from fastest to slowest. For each, name where it lives, what it stores, and how it gets invalidated.

<details>
<summary><b>Reveal: the six tiers</b></summary>

```
Layer                    Where it lives         Round-trip      Storage
─────                    ──────────────         ──────────      ───────
1. Browser cache         User's browser         0ms (no net)    HTTP cache, IndexedDB, service worker
2. CDN edge cache        100s of POPs globally  5-30ms          Static files, cacheable HTTP responses
3. App in-process cache  RAM on the app server  <1ms (local)    LRU map, no network hop
4. Distributed cache     Redis/Memcached cluster 1-5ms          Shared key/value
5. DB shared buffer      RAM on the DB host     1-10ms          Hot pages in Postgres shared_buffers
6. Database disk         Disk on DB host        10-50ms         The source of truth
```

**Speed differences (rough numbers):**

- Browser cache: instant, but only helps a repeat visitor on the same device.
- CDN edge: 5-30ms because the POP is geographically near the user.
- In-process: sub-millisecond because no network hop, just RAM access.
- Distributed (Redis): 1-5ms within the same datacenter.
- DB query cache (shared buffer): 1-10ms when the page is hot.
- DB disk: 10-50ms for SSDs, more for indexes that overflow buffer.

**The pattern: try the fastest layer first; on miss, fall through.**

```
Request
  │
  ▼
Browser cache hit? ──yes──► serve, done (0ms)
  │ no
  ▼
CDN edge hit? ──yes──► serve, done (5-30ms)
  │ no
  ▼
App in-process hit? ──yes──► serve, populate CDN (<1ms)
  │ no
  ▼
Redis hit? ──yes──► serve, populate in-process (1-5ms)
  │ no
  ▼
Read replica? ──yes──► serve, populate Redis (10-30ms)
  │ no (replica lag, replica down)
  ▼
Primary DB ──► serve, populate replica naturally (30-50ms)
```

**Each layer has the same three concerns:**

1. **What goes in it.** Not all data is cacheable. Personalized responses cannot go in a CDN. Mutable counters do not belong in long-TTL caches.
2. **How long it lives.** TTL is the cheapest invalidation. Pub/sub invalidation is the most accurate. Write-through is the most consistent.
3. **How big it is.** Browser cache is bounded by the user's disk quota. CDN is paid by GB-month. Redis is paid by RAM. In-process is bounded by your app server's RAM.

**Which layers you skip:**

- **Browser cache:** always on for static assets. Skipped for dynamic API responses unless you set explicit cache headers.
- **CDN:** skip if every response is personalized. Use for catalog pages, product images, search results that do not vary by user.
- **In-process:** skip if your fleet has 1000+ pods (cache locality is poor; same key may be cached in 1000 places, hard to invalidate). Use for fleets of 10-50 pods.
- **Distributed cache:** the workhorse. Almost always present at scale. Skip only at toy scale (<100 QPS).
- **DB query cache:** you do not control this directly. Tune `shared_buffers` to fit your hot working set.
- **Database:** never skipped. It is the source of truth.

</details>

## Step 4: sketch the read-path architecture

Here is an intentionally incomplete diagram. Fill in the five `[ ? ]` placeholders. Each marks a caching tier or a routing decision. Hint: think about where the user's request first touches a server, what sits between the app and the database, and what handles personalized content differently from cacheable content.

```
                     User browser
                          │
                          ▼
                  ┌───────────────┐
                  │  [ ? 1 ]      │  (geographically distributed; caches
                  │               │   GET responses with Cache-Control)
                  └───────┬───────┘
                          │  cache miss
                          ▼
                  ┌───────────────┐
                  │  Load Balancer│
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  App Server   │
                  │  ┌─────────┐  │
                  │  │ [ ? 2 ] │  │  (RAM on the server,
                  │  │         │  │   sub-millisecond)
                  │  └─────────┘  │
                  └───┬───────┬───┘
                      │       │
            on miss   │       │ on miss of miss
                      ▼       │
              ┌─────────────┐ │
              │  [ ? 3 ]    │ │  (cluster, shared across pods)
              └─────────────┘ │
                              │
                              ▼
                      ┌───────────────┐
                      │  [ ? 4 ]      │  (read traffic only;
                      │               │   async replicated)
                      └───────┬───────┘
                              │  writes only
                              ▼
                      ┌───────────────┐
                      │  [ ? 5 ]      │  (single source of truth;
                      │               │   accepts writes)
                      └───────────────┘
```

<details>
<summary><b>Reveal: complete read-path architecture</b></summary>

```
                     User browser
                          │
                          │ (browser HTTP cache hits here first;
                          │  not shown but always present)
                          ▼
                  ┌───────────────┐
                  │      CDN      │  CloudFront / Fastly / Cloudflare.
                  │  (edge POPs)  │  Caches GET responses with
                  │               │  Cache-Control: public, max-age=N.
                  └───────┬───────┘  ~80% of reads end here.
                          │  cache miss
                          ▼
                  ┌───────────────┐
                  │  Load Balancer│  ALB / Envoy / nginx. Stateless.
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  App Server   │  Stateless. Horizontal.
                  │  ┌─────────┐  │
                  │  │ LRU in- │  │  Per-pod RAM map. 10-50 MB.
                  │  │ process │  │  Hit: sub-ms. No network.
                  │  │ cache   │  │  Eviction: LRU.
                  │  └─────────┘  │
                  └───┬───────┬───┘
                      │       │
            on miss   │       │ on miss
                      ▼       │
              ┌─────────────┐ │
              │   Redis     │ │  Cluster, 1-5ms. Shared across
              │  (cluster)  │ │  all app pods. ~10 GB hot set.
              └──────┬──────┘ │
                     │ on miss│
                     ▼        │
                      ┌───────────────┐
                      │ Read Replicas │  Postgres async replicas.
                      │  (Postgres)   │  Replication lag ~1s P99.
                      │               │  Routed by region.
                      └───────┬───────┘
                              │  writes only
                              ▼
                      ┌───────────────┐
                      │   Primary DB  │  Postgres single primary.
                      │  (Postgres)   │  Accepts all writes.
                      │               │  Source of truth.
                      └───────────────┘
                              │
                              │ change events (CDC, Debezium, or pub/sub)
                              ▼
                      ┌───────────────┐
                      │  Invalidator  │  Listens to writes, publishes
                      │   service     │  cache-purge events to Kafka.
                      └───────────────┘
                              │
                              ▼
                      Redis pub/sub channel; app pods subscribe
                      and evict their in-process entries. CDN
                      receives purge API calls for hot keys.
```

Why each piece is here:

- **CDN.** Cheapest read latency improvement available. Responses with `Cache-Control: public, max-age=60` cache automatically at the edge. Most catalog reads never reach the origin.
- **Load balancer.** Stateless. Distributes to healthy app pods.
- **App with in-process LRU.** Top 1000 hottest keys live in app RAM. Zero network. The win is enormous for the top-N skew.
- **Redis cluster.** Shared across all app pods. Catches the next 10-100k keys that don't fit in any one pod's local RAM. Invalidatable via pub/sub.
- **Read replicas.** Reads that miss all caches go to a regional read replica, not the primary. Replication is async; ~1s lag is typical and acceptable.
- **Primary DB.** Accepts writes. Source of truth.
- **Invalidator.** When the catalog changes (price update), it publishes to a pub/sub topic. App pods drop their local entries. The CDN gets a purge API call for affected URLs.

</details>

## Step 5: read replicas

A read replica is a copy of the database that accepts only reads. The primary takes writes; the replica streams the WAL (write-ahead log) and applies it. Reads route to the replica, freeing the primary.

Three things to know:

1. **Replication is async.** A write committed on the primary is *not yet visible* on the replica. Typical lag: 10-500ms; can spike to seconds under load.
2. **Read-your-writes problem.** A user submits a form (write to primary), then refreshes (read from replica). The replica may not have the write yet. The user sees stale data.
3. **Replica routing.** Most apps route reads to replicas by default and writes to the primary. Some queries need stronger consistency and must go to the primary even on reads.

Sketch: when do you add a replica? How many? How do you route reads vs writes? How do you handle the read-your-writes case?

<details>
<summary><b>Reveal: replica strategy</b></summary>

**When to add a replica.**

Add the first replica when:
- Read QPS on the primary exceeds 50% of its capacity, *or*
- Read latency on the primary degrades because writes are competing for IOPS, *or*
- You need a failover target.

Add more replicas when:
- Reads on the existing replicas exceed 50% of capacity, *or*
- You need a replica per geography for latency.

Typical numbers: 1 primary + 2 replicas for redundancy; 1 primary + N replicas (N per region) for global apps.

**Don't add replicas when:**

- Your DB is bottlenecked on writes. Replicas do not help writes; they amplify them (every write is replicated to every replica).
- Your cache hit rate is already 95%+. The DB barely sees traffic; replicas are wasted spend. Fix the cache, not the DB.
- Your queries are slow because of bad indexes or bad query plans. Replicating bad queries just spreads the pain.

**Routing.**

```
Read request → was a write made by this user in the last 5 seconds?
  yes → route to primary  (read-your-writes guarantee)
  no  → route to replica  (cheap, async OK)

Write request → always primary.

Specific endpoints (checkout total, account balance) → always primary.
Browse endpoints (product list, category page) → always replica.
```

The "5 second pin" is a common pattern. After a user writes, their next reads go to the primary for a short window. Implemented via a session cookie (`pin_to_primary_until=<timestamp>`) or a request header. After the window passes, replica routing resumes.

**Replica lag monitoring.**

- Track `replication_lag_seconds` per replica. P99 should be under 1s.
- Alert at >5s; page at >30s.
- If lag exceeds your read-your-writes window, you have a correctness bug. Fix lag (more IOPS, faster replication) or increase the pin window.

**Replica failure.**

- If a replica falls behind, route around it. Most connection pools (pgbouncer, RDS Proxy) support this.
- If all replicas are gone, fall back to primary. Accept the load increase as a temporary degradation; alert loudly.

</details>

## Step 6: denormalization and materialized views

Sometimes caching is not enough. The query itself is expensive: a 4-table JOIN that recomputes for every request, an aggregate that scans millions of rows, a search that needs full-text matching.

Two patterns precompute the answer so the read serves a flat lookup:

- **Denormalization.** Fold the result of a JOIN into a single wider table. Update it on writes (via trigger or CDC).
- **Materialized views.** A query whose result is stored, like a cached query result. Refreshed on a schedule or on demand.

When do you reach for these? How do they interact with the caching pyramid?

<details>
<summary><b>Reveal: when to denormalize vs materialize</b></summary>

**The normalized version (slow):**

```sql
-- Three-table JOIN, runs on every product page view.
SELECT
  p.product_id, p.name, p.description,
  c.category_name,
  AVG(r.rating) AS avg_rating,
  COUNT(r.review_id) AS review_count
FROM products p
JOIN categories c ON c.category_id = p.category_id
LEFT JOIN reviews r ON r.product_id = p.product_id
WHERE p.product_id = ?
GROUP BY p.product_id, c.category_name;
```

This runs on every page view. Even cached, the cache miss path is slow (200-500ms for popular products with many reviews).

**The denormalized version:**

```sql
CREATE TABLE product_view (
    product_id     BIGINT PRIMARY KEY,
    name           TEXT NOT NULL,
    description    TEXT NOT NULL,
    category_name  TEXT NOT NULL,       -- denormalized from categories
    avg_rating     NUMERIC(2,1),         -- precomputed
    review_count   INTEGER,              -- precomputed
    last_updated   TIMESTAMPTZ
);
```

The read is now a single-row lookup by PK. <1ms even uncached.

**How to keep it fresh:**

1. **Triggers.** On write to `products`, `categories`, or `reviews`, a Postgres trigger recomputes `product_view` for the affected `product_id`. Simple but couples writes; slow writes.
2. **CDC stream.** Debezium or similar reads the WAL, publishes change events to Kafka. A consumer recomputes `product_view`. Async; writes stay fast.
3. **Scheduled refresh.** A nightly job recomputes the whole table. Coarse; accepts up to 24h staleness. Fine for cold data (last month's aggregates) but not for live catalog.

**Materialized views (Postgres feature):**

```sql
CREATE MATERIALIZED VIEW top_products_by_category AS
SELECT category_id, product_id, view_count
FROM product_views
WHERE view_count > 1000
ORDER BY view_count DESC;

REFRESH MATERIALIZED VIEW CONCURRENTLY top_products_by_category;
```

- Stored physically, like a table. Indexes work normally.
- `REFRESH` re-runs the underlying query. Can be expensive if the view is large.
- `CONCURRENTLY` lets reads continue during refresh.
- Best for analytics-shaped queries that are read frequently and updated rarely.

**When to denormalize vs materialize:**

| Scenario | Choice |
|----------|--------|
| Single-row lookup that joins 3 tables, frequently updated | Denormalize with CDC |
| Aggregate across millions of rows, updated nightly | Materialized view with REFRESH on schedule |
| Search across all products | Separate search index (Elasticsearch / Postgres FTS), not a view |
| Per-user feed | Neither; this is a different problem (see Problem 002) |
| Top-N leaderboard | Materialized view refreshed every minute |

**Trade-off: write amplification.**

Denormalization makes reads faster and writes slower. Every write to `products` triggers a recompute (or invalidation) of `product_view`. At 100x read-to-write ratio this is a clear win. At 10x or less, reconsider.

**Where this fits in the pyramid.**

Denormalized tables are a layer *below* the cache. The cache still caches the result. The denormalization makes the cache miss path fast. Without denormalization, a cache miss costs 200ms. With denormalization, a cache miss costs 5ms. Both layers compound.

</details>

## Step 7: when cache invalidation becomes the hardest problem

Cache invalidation is the famous one. Phil Karlton's quote: "There are only two hard things in computer science: cache invalidation and naming things."

Four patterns:

1. **TTL.** Each cache entry expires after N seconds. Simple. Tolerates up to N seconds of staleness.
2. **Write-through.** On every write, update the cache too. Cache is always consistent with the DB. Writes are slower.
3. **Write-behind.** On every write, update the cache; queue the DB write asynchronously. Writes are fast. Risk: queue loss = data loss.
4. **Event-driven invalidation.** Writes publish events to a pub/sub. Caches subscribe and evict affected keys.

Sketch when to use each.

<details>
<summary><b>Reveal: invalidation patterns and when each fits</b></summary>

**1. TTL.**

```
SET product:42 "{json}" EX 60     # Redis: expires in 60 seconds.
```

- Use when: staleness is acceptable up to the TTL.
- Simplest, most operationally predictable.
- The TTL is the staleness contract with the user.
- Add jitter (±10% of TTL) to prevent synchronized expiry causing thundering herd.

Example fit: product catalog with 60s TTL. Price changes are visible within a minute. Acceptable for browse; not for checkout.

**2. Write-through.**

```
def update_product(p):
    db.update(p)        # write to DB
    cache.set(p)        # then update cache
```

- Use when: read consistency matters and writes are infrequent.
- Cache is always consistent (modulo race conditions; see "stale write" gotcha).
- Writes are slower (now two operations).

The race: two writers. Writer A writes value 5 to DB, then writer B writes value 7 to DB, then writer A writes value 5 to cache, then writer B writes value 7 to cache. Cache ends up with 7 (correct). But: A writes DB(5), B writes DB(7), B writes cache(7), A writes cache(5). Cache ends with 5; DB has 7. *Inconsistent.*

Mitigation: use cache aside (write-around), or version the cache writes (only update cache if your version is newer).

Example fit: a config service where writes are rare and reads must reflect the latest config immediately.

**3. Write-behind.**

```
def update_product(p):
    cache.set(p)              # update cache first
    queue.publish(p)          # async DB write
```

- Use when: writes are very frequent and durability is acceptable to lose.
- Risk: queue loss = data lost from the source of truth.
- Often used for counters (page views, like counts) where exact accuracy is less important than throughput.

Example fit: increment a "view count" on every page load. Buffer increments in Redis, flush to DB every minute. Acceptable to lose a few seconds of view counts if Redis dies.

**4. Event-driven invalidation.**

```
def update_product(p):
    db.update(p)
    kafka.publish("product.updated", {"product_id": p.id})

# Subscribers (every app pod):
def on_product_updated(event):
    in_process_cache.evict(event.product_id)
    redis.delete(f"product:{event.product_id}")
    cdn.purge(f"/products/{event.product_id}")
```

- Use when: you have multiple cache layers that all need invalidating.
- More accurate than TTL; staleness is bounded by event-propagation latency (<1s).
- Operationally more complex; requires a pub/sub system.

Example fit: a price change must propagate to in-process caches on 50 app pods, Redis, and the CDN within seconds. TTL would take up to a minute (the full TTL). Event-driven gets it within seconds.

**The right answer is usually a mix:**

- TTL as the safety net (caps staleness even if the event system fails).
- Event-driven invalidation as the primary path (fast and accurate).
- Write-through only for specific endpoints where read consistency is non-negotiable.
- Write-behind only for fire-and-forget counters.

**The CDN twist.**

CDN invalidation is slow (seconds to minutes) and not free. Two approaches:

- **Versioned URLs.** Instead of `/products/42`, serve `/products/42?v=17` where `v` is the version number. On update, bump the version. Old URL stays cached but never requested again. New URL is uncached, fetched fresh. Effectively instant invalidation.
- **Purge API.** Call CloudFront/Fastly purge endpoint with the affected URL. Takes 5-30 seconds to propagate. Free at low volume, paid at high volume.

For a catalog: versioned URLs for product pages; purge API for hot pages where you must drop the old version immediately (legal takedown, mispriced item).

</details>

## Follow-up questions

Try answering each in 2 to 4 sentences before reading the solution.

1. **Cache stampede.** A popular product's cache entry expires at exactly the moment 1000 users hit refresh. They all miss; they all hit the DB. The DB CPU spikes; some requests time out. What patterns prevent this?

2. **Cache fallback when Redis is down.** Your Redis cluster is unreachable for 10 minutes. Every read is now a cache miss. What does the app do? Does it just slam the DB? How do you survive this gracefully?

3. **Personalized pages and the CDN.** Your product page now shows "recommended for you" based on browsing history. You cannot cache the page at the CDN anymore. What changes? Can you still cache *parts* of the page?

4. **Read-your-writes.** A user updates their profile and refreshes immediately. The replica has not yet applied the write; they see their old name. How do you fix this without sending all reads to the primary?

5. **Hot key in Redis.** One specific product key gets 10,000 req/s. Redis is a single-threaded server; that key's shard pegs at 100% CPU while others sit idle. What do you do?

6. **CDN cache miss thundering herd.** A new product launches. 100,000 users hit the launch page at the same second. None of them hit the CDN cache. They all hit the origin. How do you survive this?

7. **Cache key design.** You cache `product:42`. Then you add a feature where staff users see internal pricing. Should you cache `product:42:staff` separately? `product:42:role:<role>`? What's the trade-off?

8. **Replication lag spike during a backfill.** You import 10M products overnight. Replication lag balloons to 5 minutes. Reads-from-replica start serving very stale data. What do you do during the backfill?

9. **Cache size estimation.** You have 1M products at 5KB each = 5GB. Your Redis node has 8GB of RAM. Is that enough? What do you forget when sizing?

10. **The endpoint that should never be cached.** Inventory count must be exact at checkout (we cannot oversell). Everything else (browse, search, recommendation) can be cached. How do you enforce "this endpoint must always hit the source of truth" across a team of 30 engineers?

## Related problems

- **[URL Shortener (001)](../001-url-shortener/question.md)**. The canonical read-heavy system. A single shortcode mapping cached in front of a key-value lookup. The hot-key problem, the cache miss thundering herd, and the CDN strategy all show up there.
- **[Distributed Cache (009)](../009-distributed-cache/question.md)**. The Redis layer of this design, in depth. Eviction policies, replication, hot-key handling, and consistency models.
- **[News Feed (002)](../002-news-feed/question.md)**. The personalized read-heavy case. The CDN-friendly catalog approach in this problem does not work for feeds; that one needs precomputed timelines per user, which is its own architecture.
