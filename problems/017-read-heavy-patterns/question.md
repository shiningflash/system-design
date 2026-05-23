---
id: 17
title: Design a Read-Heavy System (Patterns Walkthrough)
category: Patterns
topics: [caching, read replicas, cdn, materialized views, denormalization]
difficulty: Easy
solution: solution.md
---

## Scene

The interviewer slides a sketch across the desk.

> *"I have a service that serves a product catalog. Today: 100 users, a single Postgres, everything works. P95 on a product page is 30ms. My PM just told me we're onboarding 100,000 users from a partner deal next month. Reads are 100x writes. What do you do, in order?"*

She pauses. "Walk me through the read path as it scales. Name what breaks first, name the fix, and tell me when *not* to apply each fix."

This is the most common shape of a "scale this" interview I've sat through. The interviewer isn't looking for a final architecture. She's looking for whether you reach for caching before you understand the read pattern, whether you add Redis before checking if a CDN would do the same job for less money, and whether you know the difference between a replica that helps read latency and a replica that just moves the problem somewhere else.

The point of the question is the toolkit: browser cache, CDN, edge cache, in-process cache, distributed cache, read replicas, materialized views, denormalization. And the *order* in which you apply them.

## Step 1: clarify before scaling anything

Take five minutes. The scenario sounds well-defined, but it isn't. You need numbers before you reach for tools.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

The latency target matters first. P95 is 30ms today, but what's acceptable at 100k users? 50ms? 100ms? If the target is 200ms you can solve a lot with one Redis. If it's 20ms globally, you need a CDN.

Freshness tolerance is next. When a price changes, how stale can a viewer's price be? One second? Sixty seconds? Five minutes? That single answer decides TTL, invalidation strategy, and whether read replicas are even acceptable.

Then read pattern. Is every request a lookup by product ID? Or are there list queries, full-text search, filter-by-price-range? Lookup by ID caches well. Full-text search wants its own index. Filters need a different denormalization.

Read distribution is the one juniors skip. Is traffic uniform across products, or does 1% of products get 90% of views? Zipf makes caching trivial. Uniform makes caching almost useless.

Geographic spread tells you whether a CDN earns its keep. Same-region: skip the CDN, use Redis. Spread across continents: a CDN is the cheapest latency win money can buy.

Personalized vs shared. Does every user see the same product page, or do you show user-specific pricing, recommendations, stock-near-you? Shared caches at the CDN. Personalized cannot.

Write pattern. Is the 100x ratio uniform, or are writes bursty (a nightly batch importing a million products)? Bursty writes break cache invalidation strategies that assume a steady trickle.

Consistency per endpoint. The product page might tolerate staleness, but the checkout price has to be exact. Different endpoints get different cache TTLs (or no cache at all).

The junior trap here is starting with "add Redis." You don't know yet whether Redis is the right tool. The CDN might do 90% of the work. The in-process cache might do the other 9%. Redis might never be needed.

</details>

## Step 2: capacity

The interviewer tells you:

- 100,000 users active per day.
- Each user makes about 50 reads per session (browse + view + filter).
- Reads are 100x writes.
- Catalog: 1M products. Each product about 5KB JSON.
- Read latency target: P95 < 50ms.
- Freshness: 60 seconds is fine for price; stock can be 5 minutes stale; description is immutable.
- Traffic is roughly Zipf. Top 10k products serve about 80% of reads.

Compute, on paper, before revealing: reads per second sustained and peak, writes per second, working-set size for the hot 80%, cache hit rate required to keep DB load under 200 QPS, and bandwidth at 5KB per read.

<details>
<summary><b>Reveal: the math</b></summary>

100k users × 50 reads = 5M reads/day. 5M / 86400 ≈ 58 reads/sec sustained. Peak is typically 5x for consumer traffic, so about 300 reads/sec at peak.

Writes drop out of 58 / 100, so 0.58 writes/sec sustained, 3/sec at peak. Trivially small. The catalog is updated by a few admins, not by users.

Hot working set: top 10k products × 5KB = 50MB. Fits in a single Redis node, an in-process LRU, or even the browser cache for repeat visitors.

For DB hit rate, if the goal is to keep the DB under 200 QPS at the 300 reads/sec peak, the DB has to serve at most 200/300 = 67% of traffic. So the cache needs to absorb at least 33%. In practice you target 80-95% so the DB has headroom. With Zipf 80/20 traffic and a cache covering the top 10k products, 80%+ hit rate is the default outcome with a TTL of a few minutes.

Bandwidth: 300 reads/sec × 5KB = 1.5 MB/sec = 12 Mbps. Negligible at the origin. At the CDN it scales out per POP. No bandwidth pressure anywhere.

The takeaway: this is a textbook read-heavy system with a small hot set and low absolute QPS. Almost any caching scheme works. The interview isn't about *whether* to cache; it's about *which layer* to cache at, in what order, and why.

</details>

## Step 3: the caching pyramid

Before drawing any architecture, draw the layers. A read request, on its way from the browser to the database, passes through up to six tiers of cache. Each one is roughly 10x faster than the one below it. Each one also needs its own invalidation and capacity planning.

Sketch the six layers from fastest to slowest. For each, name where it lives, what it stores, and how it gets invalidated.

<details>
<summary><b>Reveal: the six tiers</b></summary>

```
Layer                    Where it lives          Round-trip      Storage
─────                    ──────────────          ──────────      ───────
1. Browser cache         User's browser          0ms (no net)    HTTP cache, service worker
2. CDN edge cache        100s of POPs globally   5-30ms          Cacheable HTTP responses
3. App in-process cache  RAM on the app server   <1ms (local)    LRU map, no network hop
4. Distributed cache     Redis/Memcached cluster 1-5ms           Shared key/value
5. DB shared buffer      RAM on the DB host      1-10ms          Hot pages in shared_buffers
6. Database disk         Disk on DB host         10-50ms         Source of truth
```

Browser cache is instant but only helps a repeat visitor on the same device. CDN edge sits at 5-30ms because the POP is geographically near the user. In-process is sub-millisecond because there's no network hop, just RAM. Redis is 1-5ms inside the same datacenter. DB shared buffer is 1-10ms when the page is hot. DB disk is 10-50ms for SSDs, more when the index overflows the buffer.

The pattern is try the fastest layer first, on miss fall through to the next.

```
Request
  │
  ▼
Browser cache hit? ──yes──► serve, done (0ms)
  │ no
  ▼
CDN edge hit?     ──yes──► serve, done (5-30ms)
  │ no
  ▼
In-process hit?   ──yes──► serve, populate CDN (<1ms)
  │ no
  ▼
Redis hit?        ──yes──► serve, populate in-process (1-5ms)
  │ no
  ▼
Read replica?     ──yes──► serve, populate Redis (10-30ms)
  │ no (replica down or row not replicated)
  ▼
Primary DB ───────────────► serve (30-50ms)
```

Each layer has the same three concerns. What goes in it (not all data is cacheable; personalized responses can't sit in a CDN; mutable counters don't belong in long-TTL caches). How long it lives (TTL is the cheapest invalidation, pub/sub is the most accurate, write-through is the most consistent). How big it is (browser is bounded by the user's disk quota, CDN is paid by GB-month, Redis is paid by RAM, in-process is bounded by your pod's RAM).

Which layers you skip depends on the workload. Browser cache is always on for static assets, skipped for dynamic API responses unless you set explicit cache headers. CDN gets skipped if every response is personalized; otherwise you want it for anything that doesn't vary by user. In-process is a bad idea once your fleet hits 1000+ pods because cache locality is poor and invalidation gets messy; it works well for fleets of 10-50. Distributed cache is the workhorse, almost always present at scale, skipped only at toy scale. DB query cache you don't really control directly; tune `shared_buffers` to fit your hot working set. The database itself is never skipped because it's the source of truth.

</details>

## Step 4: sketch the read-path architecture

Here's an intentionally incomplete diagram. Fill in the five `[ ? ]` placeholders. Each marks a caching tier or a routing decision. Hint: think about where the user's request first touches a server, what sits between the app and the database, and what handles personalized content differently from cacheable content.

```
                     User browser
                          │
                          ▼
                  ┌───────────────┐
                  │  [ ? 1 ]      │  geographically distributed; caches
                  │               │  GET responses with Cache-Control
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
                  │  │ [ ? 2 ] │  │  RAM on the server,
                  │  │         │  │  sub-millisecond
                  │  └─────────┘  │
                  └───┬───────┬───┘
                      │       │
            on miss   │       │ on miss of miss
                      ▼       │
              ┌─────────────┐ │
              │  [ ? 3 ]    │ │  cluster, shared across pods
              └─────────────┘ │
                              │
                              ▼
                      ┌───────────────┐
                      │  [ ? 4 ]      │  read traffic only;
                      │               │  async replicated
                      └───────┬───────┘
                              │  writes only
                              ▼
                      ┌───────────────┐
                      │  [ ? 5 ]      │  single source of truth;
                      │               │  accepts writes
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
                  │  │ cache   │  │
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
                      └───────────────┘
                              │
                              │ change events (CDC / pub/sub)
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

The CDN is the cheapest latency improvement on offer. Responses tagged `Cache-Control: public, max-age=60` cache automatically at the edge, and most catalog reads never reach the origin. The load balancer is stateless and just distributes to healthy pods. The app pod's in-process LRU holds the top 1000 hottest keys in RAM; zero network. Redis sits below that, shared across all pods in the region, catching the next 10-100k keys that don't fit in any one pod's local map; it's invalidatable via pub/sub. Read replicas handle reads that miss every cache layer; replication is async and roughly 1s lag is acceptable. The primary accepts writes and acts as source of truth. The invalidator listens to writes and publishes invalidation events; app pods drop local entries, and the CDN gets a purge call for affected URLs.

</details>

## Step 5: read replicas

A read replica is a copy of the database that accepts only reads. The primary takes writes; the replica streams the WAL (write-ahead log) and applies it. Reads route to the replica, freeing the primary.

Three things to know. Replication is async, so a write committed on the primary isn't yet visible on the replica; typical lag is 10-500ms but can spike to seconds under load. That gives you the read-your-writes problem: a user submits a form (write to primary) then refreshes (read from replica) and sees their old data because the write hasn't propagated. And replica routing has to decide which reads go to the replica and which must go to the primary anyway.

Sketch: when do you add a replica? How many? How do you route reads vs writes? How do you handle the read-your-writes case?

<details>
<summary><b>Reveal: replica strategy</b></summary>

Add the first replica when read QPS on the primary exceeds 50% of capacity, or when read latency on the primary degrades because writes are competing for IOPS, or when you need a failover target. Add more replicas when reads on the existing replicas exceed 50% of capacity, or when you need a replica per geography for latency. Typical numbers: 1 primary + 2 replicas for redundancy; 1 primary + N replicas (N per region) for global apps.

Don't add replicas when your DB is bottlenecked on writes; replicas don't help writes, they amplify them (every write is replicated to every replica). Don't add them when your cache hit rate is already 95%+ because the DB barely sees traffic and replicas are wasted spend; fix the cache. Don't add them when queries are slow due to bad indexes or bad query plans; replicating bad queries just spreads the pain.

Routing rule of thumb:

```
Read request → did this user write in the last 5 seconds?
  yes → route to primary  (read-your-writes guarantee)
  no  → route to replica  (cheap, async OK)

Write request → always primary.

Strong-read endpoints (checkout total, account balance) → always primary.
Browse endpoints (product list, category page) → always replica.
```

The 5-second pin is a common pattern. After a user writes, their next reads go to the primary for a short window. You set a session cookie like `pin_to_primary_until=<timestamp>` or a request header. After the window passes, replica routing resumes.

Track `replication_lag_seconds` per replica. P99 should stay under 1s. Alert at >5s and page at >30s. If lag exceeds your read-your-writes window, you have a correctness bug; either fix the lag (more IOPS, faster replication) or increase the pin window.

When a replica falls behind, route around it. Most connection pools (pgbouncer, RDS Proxy) support this. When all replicas are gone, fall back to primary, accept the load increase as a temporary degradation, and alert loudly.

</details>

## Step 6: denormalization and materialized views

Sometimes caching isn't enough. The query itself is expensive: a 4-table JOIN that recomputes for every request, an aggregate that scans millions of rows, a search that needs full-text matching.

Two patterns precompute the answer so the read serves a flat lookup. Denormalization folds the result of a JOIN into one wider table, updated on writes via trigger or CDC. Materialized views store the result of a query (like a cached query result), refreshed on a schedule or on demand.

When do you reach for these? How do they interact with the caching pyramid?

<details>
<summary><b>Reveal: when to denormalize vs materialize</b></summary>

Here's the normalized version, the kind that runs on every product page view:

```sql
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

Even cached, the cache-miss path is slow (200-500ms for popular products with many reviews). The denormalized version is a single-row lookup:

```sql
CREATE TABLE product_view (
    product_id     BIGINT PRIMARY KEY,
    name           TEXT NOT NULL,
    description    TEXT NOT NULL,
    category_name  TEXT NOT NULL,      -- denormalized from categories
    avg_rating     NUMERIC(2,1),       -- precomputed
    review_count   INTEGER,            -- precomputed
    last_updated   TIMESTAMPTZ
);
```

Sub-millisecond even uncached.

Three ways to keep it fresh. Triggers on `products`, `categories`, or `reviews` recompute `product_view` for the affected `product_id`; simple but couples writes, so writes get slower. A CDC stream (Debezium or similar) reads the WAL and publishes change events to Kafka; a consumer recomputes the row asynchronously, writes stay fast. A scheduled refresh recomputes nightly; coarse, accepts up to 24h staleness, fine for cold data but not live catalog.

Postgres' materialized views are a separate tool:

```sql
CREATE MATERIALIZED VIEW top_products_by_category AS
SELECT category_id, product_id, view_count
FROM product_views
WHERE view_count > 1000
ORDER BY view_count DESC;

REFRESH MATERIALIZED VIEW CONCURRENTLY top_products_by_category;
```

Stored physically like a table, indexes work normally, REFRESH re-runs the underlying query (which can be expensive if the view is large), CONCURRENTLY lets reads continue during refresh. Best for analytics-shaped queries that are read frequently and updated rarely.

Quick decision guide:

| Scenario | Choice |
|----------|--------|
| Single-row lookup that joins 3 tables, frequently updated | Denormalize with CDC |
| Aggregate across millions of rows, updated nightly | Materialized view with scheduled REFRESH |
| Search across all products | Separate search index (Elasticsearch / Postgres FTS) |
| Per-user feed | Neither; that's a different problem (Problem 002) |
| Top-N leaderboard | Materialized view refreshed every minute |

The trade-off is write amplification. Denormalization makes reads faster and writes slower because every write to `products` triggers a recompute (or invalidation) of `product_view`. At 100x read-to-write ratio this is a clear win. At 10x or less, reconsider.

Where this fits in the pyramid: denormalized tables sit *below* the cache. The cache still caches the result. The denormalization makes the cache-miss path fast. Without it, a cache miss costs 200ms; with it, 5ms. Both layers compound.

</details>

## Step 7: when cache invalidation becomes the hardest problem

Cache invalidation is the famous one. Phil Karlton's quote: "There are only two hard things in computer science: cache invalidation and naming things."

Four patterns. TTL expires each entry after N seconds; simple, tolerates up to N seconds of staleness. Write-through updates the cache on every write so it's always consistent with the DB, but writes get slower. Write-behind updates the cache and queues the DB write asynchronously; fast writes but queue loss equals data loss. Event-driven invalidation publishes events on writes; caches subscribe and evict affected keys.

Sketch when to use each.

<details>
<summary><b>Reveal: invalidation patterns and when each fits</b></summary>

**TTL.**

```
SET product:42 "{json}" EX 60     # Redis: expires in 60 seconds.
```

Use when staleness is acceptable up to the TTL. Simplest, most operationally predictable, and the TTL becomes the staleness contract with the user. Add jitter (±10% of TTL) to prevent synchronized expiry causing thundering herd. Good fit: product catalog with 60s TTL. Price changes are visible within a minute. Acceptable for browse, not for checkout.

**Write-through.**

```
def update_product(p):
    db.update(p)        # write to DB
    cache.set(p)        # then update cache
```

Use when read consistency matters and writes are infrequent. Cache stays consistent (modulo race conditions), but writes are slower because there are now two operations.

The race: two writers. Writer A writes 5 to DB, writer B writes 7 to DB, then A writes 5 to cache, then B writes 7 to cache. Cache ends with 7, which matches the DB. But reorder it: A writes DB(5), B writes DB(7), B writes cache(7), A writes cache(5). Now cache has 5; DB has 7. Inconsistent.

Mitigation: cache-aside (write-around) instead, or version the cache writes so you only update if your version is newer. Good fit: a config service where writes are rare and reads must reflect the latest config immediately.

**Write-behind.**

```
def update_product(p):
    cache.set(p)              # update cache first
    queue.publish(p)          # async DB write
```

Use when writes are very frequent and some durability loss is acceptable. Risk: queue loss equals data lost from the source of truth. Often used for counters (page views, like counts) where exact accuracy matters less than throughput. Good fit: incrementing a "view count" on every page load. Buffer increments in Redis, flush to DB every minute.

**Event-driven invalidation.**

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

Use when you have multiple cache layers that all need invalidating. More accurate than TTL; staleness is bounded by event-propagation latency (typically under a second). Operationally heavier; needs a pub/sub system. Good fit: a price change has to propagate to in-process caches on 50 app pods, Redis, and the CDN within seconds. TTL would take up to a minute. Event-driven gets it within seconds.

The right answer is usually a mix. TTL as the safety net (caps staleness even if the event system fails). Event-driven as the primary path (fast and accurate). Write-through only for specific endpoints where read consistency is non-negotiable. Write-behind only for fire-and-forget counters.

The CDN twist: CDN invalidation is slow (seconds to minutes) and not free. Two approaches. Versioned URLs: instead of `/products/42`, serve `/products/42?v=17`. On update, bump the version. Old URL stays cached but never requested again; new URL is uncached, fetched fresh. Effectively instant invalidation. Or purge API: call CloudFront/Fastly purge with the affected URL, taking 5-30 seconds to propagate. Free at low volume, paid at high.

For a catalog, versioned URLs for product pages, and the purge API for hot pages you need to drop immediately (legal takedown, mispriced item).

</details>

## Follow-up questions

Try answering each in 2-4 sentences before reading the solution.

1. **Cache stampede.** A popular product's cache entry expires at exactly the moment 1000 users hit refresh. They all miss, they all hit the DB, the DB CPU spikes, some requests time out. What patterns prevent this?

2. **Cache fallback when Redis is down.** Your Redis cluster is unreachable for 10 minutes. Every read is now a cache miss. What does the app do? Does it just slam the DB? How do you survive this gracefully?

3. **Personalized pages and the CDN.** Your product page now shows "recommended for you" based on browsing history. You can't cache the page at the CDN anymore. What changes? Can you still cache *parts* of the page?

4. **Read-your-writes.** A user updates their profile and refreshes immediately. The replica hasn't applied the write; they see their old name. How do you fix this without sending all reads to the primary?

5. **Hot key in Redis.** One specific product key gets 10,000 req/s. Redis is single-threaded; that key's shard pegs at 100% CPU while others sit idle. What do you do?

6. **CDN cache miss thundering herd.** A new product launches. 100,000 users hit the launch page at the same second. None of them hit the CDN cache. They all hit the origin. How do you survive this?

7. **Cache key design.** You cache `product:42`. Then you add a feature where staff users see internal pricing. Should you cache `product:42:staff` separately? `product:42:role:<role>`? What's the trade-off?

8. **Replication lag spike during a backfill.** You import 10M products overnight. Replication lag balloons to 5 minutes. Reads-from-replica start serving very stale data. What do you do during the backfill?

9. **Cache size estimation.** You have 1M products at 5KB each = 5GB. Your Redis node has 8GB of RAM. Is that enough? What do you forget when sizing?

10. **The endpoint that should never be cached.** Inventory count must be exact at checkout (you can't oversell). Everything else (browse, search, recommendation) can be cached. How do you enforce "this endpoint must always hit the source of truth" across a team of 30 engineers?

## Related problems

- **[URL Shortener (001)](../001-url-shortener/question.md).** The canonical read-heavy system. A single shortcode mapping cached in front of a key-value lookup. The hot-key problem, the cache miss thundering herd, and the CDN strategy all show up there.
- **[Distributed Cache (009)](../009-distributed-cache/question.md).** The Redis layer of this design, in depth. Eviction policies, replication, hot-key handling, and consistency models.
- **[News Feed (002)](../002-news-feed/question.md).** The personalized read-heavy case. The CDN-friendly catalog approach in this problem doesn't work for feeds; that one needs precomputed timelines per user, which is its own architecture.
