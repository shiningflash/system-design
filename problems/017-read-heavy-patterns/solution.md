## Solution: Read-Heavy System Patterns

### TL;DR

A read-heavy system at the scale most products see (100x reads to writes, small hot set, Zipf-distributed traffic) is solved by stacking caches in front of the source of truth. Each layer is roughly 10x faster than the one below, and each handles a different slice of the traffic. The CDN catches the global, shared, cacheable responses. The in-process LRU catches the top-N hottest keys per pod. Redis catches the warm tail across all pods. Read replicas catch the cold misses. The primary serves writes and a tiny fraction of reads.

The non-obvious work lives in three places. First, knowing when *not* to add a layer: an in-process cache hurts when you have 1000 pods; a CDN is useless if every response is personalized. Second, invalidation: TTL is the safety net, event-driven pub/sub is the primary path, write-through is for the few endpoints where reads cannot be stale. Third, the failure modes: cache stampede on cold-miss-of-hot-key, hot-key shard saturation in Redis, read-your-writes when reads are routed to replicas, and graceful fallback when the cache itself dies.

The scaling journey is short. Most read-heavy systems never need more than a CDN, a Redis cluster, two read replicas, and a denormalized view table for the most expensive queries. Past that the answers become regional (per-region Redis, per-region replicas), and the failure modes become the next design problem.

### 1. Clarifying questions, recap

Covered in `question.md`. The four most decision-shaping ones, in order: *freshness tolerance* (sets TTL; without this number every other choice is a guess), *read pattern* (lookup by ID caches trivially, filter-by-attribute does not, full-text search wants its own index), *read distribution* (Zipf means caching wins, uniform means caching barely helps), and *personalization* (shared responses cache at the CDN, personalized ones cannot).

The biggest junior mistake here is jumping to "Redis" without first asking whether the requests are even cacheable. A catalog page is cacheable. A user-specific cart is not. The same Redis instance handles both, but the strategy is completely different.

### 2. Capacity, with the math

From `question.md`: 100k users × 50 reads/session = 5M reads/day, about 58 reads/sec sustained and 300/sec peak. Writes are 1/100th of that, so 0.6/sec sustained and 3/sec peak. Trivial. Catalog is 1M products × 5KB = 5GB total, of which the hot 1% (10k products) covers 80% of traffic and fits in 50MB. To keep the DB under 200 QPS at the 300/sec peak the cache only needs a 33% hit rate, but you actually target 80-95% for headroom.

This is a small system in throughput terms. The architecture isn't about throughput; it's about latency, geographic spread, and graceful degradation.

### 3. API

A read-heavy service typically has three shapes of endpoint, each with its own caching strategy. Getting them mixed up is how teams ship correctness bugs.

The cacheable shared response, the kind you want sitting in the CDN:

```
GET /api/v1/products/42
Cache-Control: public, max-age=60, stale-while-revalidate=300

{
  "product_id": 42,
  "name": "Headphones",
  "price_cents": 4999,
  "currency": "USD",
  "description": "...",
  "image_url": "https://cdn.example.com/p/42.jpg",
  "avg_rating": 4.6,
  "review_count": 1284,
  "category": "Electronics > Audio"
}
```

`Cache-Control: public, max-age=60` lets the CDN and browser cache it for a minute. `stale-while-revalidate=300` lets the CDN keep serving the stale version up to 5 minutes while a background refresh runs, so users never wait on a cache miss. ETag plus `If-None-Match` covers 304 responses on conditional GETs.

The personalized read, where the response depends on the user:

```
GET /api/v1/products/42/personalized
Cache-Control: private, max-age=30
Vary: Authorization

{
  "product_id": 42,
  "your_last_viewed": "2026-05-22T10:11:00Z",
  "personalized_price_cents": 4499,
  "in_your_wishlist": true
}
```

`private` keeps it out of the CDN and only in the user's browser. `Vary: Authorization` is the explicit signal that the response depends on who's asking. Short TTL because users expect this to feel live.

The strong-read endpoint, where caching is just wrong:

```
GET /api/v1/cart/total
Cache-Control: no-store
X-Read-Consistency: strong

{
  "subtotal_cents": 9998,
  "tax_cents": 800,
  "total_cents": 10798,
  "computed_at": "2026-05-22T10:15:34Z"
}
```

`no-store` means never cache, anywhere. The custom header tells the routing layer to skip the replica and hit the primary. The caller can trust the price is exact.

These three endpoints share a code base but route through different cache tiers. The junior mistake is slapping a uniform cache on everything; you end up caching the cart total and either overcharging or undercharging customers.

### 4. Data model

A read-heavy system has two halves. The normalized source-of-truth tables on the write side (small, strict schema). And the denormalized read-side views on the read side (wider, optimized for what the API actually returns).

```sql
-- Source of truth: normalized.
CREATE TABLE products (
    product_id      BIGINT PRIMARY KEY,
    name            TEXT NOT NULL,
    description     TEXT,
    category_id     BIGINT REFERENCES categories(category_id),
    price_cents     INT NOT NULL,
    currency        CHAR(3) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version         INT NOT NULL DEFAULT 1            -- bumped on every write
);
CREATE INDEX idx_products_category ON products (category_id);
CREATE INDEX idx_products_updated ON products (updated_at);

CREATE TABLE categories (
    category_id     BIGINT PRIMARY KEY,
    category_name   TEXT NOT NULL,
    parent_id       BIGINT REFERENCES categories(category_id)
);

CREATE TABLE reviews (
    review_id       BIGINT PRIMARY KEY,
    product_id      BIGINT REFERENCES products(product_id),
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_reviews_product ON reviews (product_id);

-- Read-side: denormalized for the product page.
CREATE TABLE product_view (
    product_id      BIGINT PRIMARY KEY,
    name            TEXT NOT NULL,
    description     TEXT,
    category_name   TEXT NOT NULL,           -- denormalized
    category_path   TEXT NOT NULL,           -- "Electronics > Audio > Headphones"
    price_cents     INT NOT NULL,
    currency        CHAR(3) NOT NULL,
    avg_rating      NUMERIC(2,1),            -- precomputed
    review_count    INT NOT NULL DEFAULT 0,  -- precomputed
    version         INT NOT NULL,            -- from products.version
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Materialized view: top products per category.
CREATE MATERIALIZED VIEW top_products_per_category AS
SELECT
    p.category_id,
    p.product_id,
    p.name,
    COUNT(DISTINCT v.user_id) AS distinct_viewers_30d
FROM products p
JOIN product_views v ON v.product_id = p.product_id
WHERE v.viewed_at > NOW() - INTERVAL '30 days'
GROUP BY p.category_id, p.product_id, p.name
ORDER BY p.category_id, distinct_viewers_30d DESC;

CREATE UNIQUE INDEX idx_top_products ON top_products_per_category (category_id, product_id);
```

The cache keys mirror the same split:

| Key | Value | TTL | Invalidated by |
|-----|-------|-----|----------------|
| `product:{id}:v{ver}` | full product JSON | 1h | URL is versioned; new version replaces |
| `product:{id}` | full product JSON | 60s | TTL + pub/sub event |
| `category:{id}:top:v{ver}` | top product IDs list | 5min | TTL + nightly refresh |
| `search:{normalized_query}:p{page}` | result page | 5min | TTL only |
| `user:{uid}:cart` | cart contents | 5min | write-through on cart update |
| `user:{uid}:pinned_until` | timestamp | 5s | TTL only |

Two choices worth defending. Versioning the key (`product:{id}:v{ver}`) means when a price changes you bump `products.version`. Cached reads use the old key, new reads use the new key, and the old key expires naturally with no explicit purge. The CDN URL gets `?v={ver}` for the same reason. Including the version *inside* the value too lets the app detect stale cache writes: "I have v3 in cache, the DB says v5, evict and refresh."

### 5. Order of operations as the system grows

This is the lesson of the problem. The order in which you reach for each tool, and the conditions under which each one is the wrong choice.

The order of application:

1. **HTTP cache headers and a CDN.** Cheapest. Often free up to a tier. Catches the largest fraction of cacheable traffic.
2. **Distributed cache (Redis).** Next-cheapest. Catches per-key lookups the CDN can't (because the CDN caches at URL granularity, not key granularity, and personalized URLs blow up the cache).
3. **Read replicas.** When the primary's read load is causing write latency to suffer. Not before.
4. **In-process LRU.** When the top-N hot keys are saturating Redis, or when you want sub-millisecond reads for the hottest items.
5. **Denormalization / materialized views.** When the *cache-miss path* is too slow because the underlying query is expensive (multi-table JOIN, aggregate over millions of rows).
6. **Per-region everything.** When latency targets can't be met from a single region.

The anti-order, what *not* to do, in order of how badly it fails: adding Redis before knowing if responses are cacheable; adding read replicas before checking cache hit rate (if hit rate is 5%, fixing it is better than adding replicas); denormalizing aggressively before measuring query time (premature write amplification); building per-region from day one (most apps never need it); building a custom cache when Redis or memcached would do.

The decision tree at each stage looks roughly like this:

```
Question: which layer do I add next?

Is read latency too high?
  ├── Geographic (users far from origin)?               → CDN
  ├── Every request hits the DB?                        → Redis
  ├── Cache misses are slow?                            → Denormalize
  └── Redis overloaded on one key?                      → In-process LRU
       (or Redis itself the bottleneck?)                → Read replica + Redis shard

Is the DB CPU too high?
  ├── Primary handling reads it shouldn't?              → Read replica
  ├── Queries slow due to bad indexes?                  → Fix indexes (not a caching problem)
  └── Writes too frequent for the cache to help?        → Write-behind for non-critical

Is the cache hit rate low?
  ├── TTLs too short?                                   → Increase TTL within freshness budget
  ├── Hot set bigger than cache memory?                 → Bigger cache, or different key shape
  └── Workload uniform (no Zipf)?                       → Caching won't help; rethink
```

### 6. Architecture: the full read path

Here's the whole picture. Multi-region, all six cache tiers, with the invalidation path running off the primary via CDC.

```
                              User browser
                                   │
                                   │  (1) browser HTTP cache check
                                   ▼
                          ┌─────────────────┐
                          │  Browser cache  │  hit? → done
                          └────────┬────────┘
                                   │ miss
                                   ▼
                          ┌─────────────────┐
                          │     CDN POP     │  CloudFront / Fastly / Cloudflare.
                          │   (regional)    │  Cache-Control: public, max-age=60.
                          │                 │  hit? → done (~10ms)
                          └────────┬────────┘
                                   │ miss (or no-cache header)
                                   ▼
                          ┌─────────────────┐
                          │ Origin LB       │  Routes to nearest healthy region.
                          └────────┬────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
        ┌──────────┐         ┌──────────┐         ┌──────────┐
        │  Region  │         │  Region  │         │  Region  │
        │  us-east │         │  eu-west │         │  ap-south│
        │          │         │          │         │          │
        │ ┌──────┐ │         │ ┌──────┐ │         │ ┌──────┐ │
        │ │  LB  │ │         │ │  LB  │ │         │ │  LB  │ │
        │ └──┬───┘ │         │ └──┬───┘ │         │ └──┬───┘ │
        │ ┌──▼───┐ │         │ ┌──▼───┐ │         │ ┌──▼───┐ │
        │ │ App  │ │         │ │ App  │ │         │ │ App  │ │
        │ │ +LRU │ │         │ │ +LRU │ │         │ │ +LRU │ │
        │ └──┬───┘ │         │ └──┬───┘ │         │ └──┬───┘ │
        │ ┌──▼───┐ │         │ ┌──▼───┐ │         │ ┌──▼───┐ │
        │ │Redis │ │         │ │Redis │ │         │ │Redis │ │
        │ │clstr │ │         │ │clstr │ │         │ │clstr │ │
        │ └──┬───┘ │         │ └──┬───┘ │         │ └──┬───┘ │
        │ ┌──▼───┐ │         │ ┌──▼───┐ │         │ ┌──▼───┐ │
        │ │ Read │ │         │ │ Read │ │         │ │ Read │ │
        │ │replic│ │         │ │replic│ │         │ │replic│ │
        │ └──┬───┘ │         │ └──┬───┘ │         │ └──┬───┘ │
        └────┼─────┘         └────┼─────┘         └────┼─────┘
             │                    │                    │
             └────────────────────┼────────────────────┘
                                  │ writes only
                                  ▼
                          ┌─────────────────┐
                          │   Primary DB    │  Postgres single primary.
                          │   (Postgres)    │  Source of truth.
                          └────────┬────────┘
                                   │
                                   │ CDC (Debezium / logical replication)
                                   ▼
                          ┌─────────────────┐
                          │   Kafka topic   │  product.changed
                          └────────┬────────┘
                                   │
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼
         ┌─────────────┐    ┌─────────────┐   ┌─────────────┐
         │ View-table  │    │ Cache       │   │ CDN purge   │
         │ updater     │    │ invalidator │   │ worker      │
         │ (recomputes │    │ (Redis pub/ │   │ (calls CDN  │
         │ denormalized│    │ sub; per-   │   │ purge API   │
         │ rows)       │    │ pod evicts) │   │ for hot     │
         │             │    │             │   │ URLs)       │
         └─────────────┘    └─────────────┘   └─────────────┘
```

A few things worth pointing out while reading this. The browser cache is free; it's just `Cache-Control` headers. The CDN POPs are the first server-side cache and serve roughly 80% of cacheable traffic with sub-30ms latency. The in-process LRU per pod (10-50MB) catches the absolute hottest keys with sub-millisecond latency and zero network. Per-region Redis catches the warm tail (5-10GB), shared across all pods in the region. Per-region read replicas serve cache misses; lag is under a second P99. The primary is single-region for writes and emits change events via logical replication. The CDC pipeline reads the WAL into Kafka, decoupling writes from downstream consumers. From there the view-table updater recomputes denormalized rows, the cache invalidator publishes to Redis pub/sub so app pods drop entries, and the CDN purge worker calls the CDN purge API for URLs that have to drop immediately.

### 7. A cache-miss read, drawn as a sequence

Here's a single read working its way down through all the cache tiers, missing each one, hitting the replica, and re-warming everything on the way back up.

```
 Browser    CDN POP    App+LRU   Redis     Replica   Primary   Invalidator
    │          │          │         │          │         │           │
    │ GET /42  │          │         │          │         │           │
    ├─────────►│          │         │          │         │           │
    │          │ miss     │         │          │         │           │
    │          ├─────────►│         │          │         │           │
    │          │          │ LRU miss│          │         │           │
    │          │          ├────────►│          │         │           │
    │          │          │         │ Redis    │         │           │
    │          │          │         │  miss    │         │           │
    │          │          │         ├─────────►│         │           │
    │          │          │         │          │ SELECT  │           │
    │          │          │         │          │ product │           │
    │          │          │         │          │  _view  │           │
    │          │          │         │          │  =42    │           │
    │          │          │         │          │◄────────┤           │
    │          │          │         │          │ row     │           │
    │          │          │         │◄─────────┤         │           │
    │          │          │◄────────┤ populate │         │           │
    │          │          │ populate│ TTL 60s  │         │           │
    │          │◄─────────┤ LRU     │  +jitter │         │           │
    │          │ cache    │         │          │         │           │
    │◄─────────┤ at edge  │         │          │         │           │
    │ 200 OK   │ (max-age=│         │          │         │           │
    │          │   60)    │         │          │         │           │
    │          │          │         │          │         │           │

(Later: an admin updates the price.)

  Admin     App+LRU   Primary    CDC/Kafka   Invalidator    Redis,LRUs,CDN
    │          │          │           │             │              │
    │ PUT /42  │          │           │             │              │
    ├─────────►│          │           │             │              │
    │          │ UPDATE   │           │             │              │
    │          ├─────────►│           │             │              │
    │          │          │ WAL emit  │             │              │
    │          │          ├──────────►│             │              │
    │          │          │           │ product.    │              │
    │          │          │           │  changed    │              │
    │          │          │           ├────────────►│              │
    │          │          │           │             │ evict        │
    │          │          │           │             │ everywhere   │
    │          │          │           │             ├─────────────►│
    │◄─────────┤ 200 OK   │           │             │              │
```

Each layer populates the one above it on the way back up, so the system self-warms. The invalidation path is decoupled from the write path: the write commits, the response returns, and the cache eviction happens asynchronously via CDC and Kafka. Bounded staleness in the worst case is the longer of the TTL or the propagation lag through Kafka and pub/sub (typically under 2 seconds).

P95 budget: cache hit at any layer is 30-50ms including the user-to-CDN network; a full miss to the DB is 80-150ms.

### 8. Read path and write path in words

The read path for `GET /products/42`: the browser checks its HTTP cache first; non-expired entry, serve from disk, zero network, done. Otherwise the browser hits the nearest CDN POP; if cached and not expired, serve, ~10ms. On CDN miss, the POP forwards to the origin LB which routes to the nearest healthy region. The regional LB forwards to an app pod. The pod checks its in-process LRU by `product:42`; LRU hit serves in under a millisecond. LRU miss, the pod queries Redis; Redis hit copies into LRU and serves in 1-5ms. Redis miss, the pod queries the regional read replica with `SELECT * FROM product_view WHERE product_id = 42`, populates Redis with TTL of 60s plus jitter, serves in ~30ms. Replica miss (replica down or row not yet replicated), fall back to the primary. Primary miss, the row genuinely doesn't exist, return 404.

The write path for `PUT /products/42`: client to CDN to LB to regional LB to app pod (CDN never caches writes; POST/PUT/DELETE aren't cached by default). The pod validates the request and performs auth, then writes to the primary DB (writes always go to primary). DB write succeeds, the pod publishes a `product.changed` event to Kafka with `{product_id: 42, version: 5}`, and responds 200 to the client. Asynchronously, downstream consumers act: the view-table updater recomputes `product_view` for product 42; the cache invalidator publishes `evict: product:42` on Redis pub/sub so all app pods (in all regions) drop the key from their LRU and from Redis; the CDN purge worker calls the CDN purge API for `/products/42`, taking 5-30s to fully propagate. Replicas pick up the write within ~1s via standard streaming replication.

The write path is slower than the read path because it has more side effects. But writes are 1/100th of reads, so total system load is dominated by reads.

Read-your-writes works through a session cookie. When the app pod processes a write, it sets `pin_to_primary_until=<now + 5s>` on the response. On subsequent reads from the same client, the pod inspects the cookie; if the timestamp is in the future, the read routes to the primary instead of the replica. After 5 seconds (longer than worst-case replication lag) the cookie expires and reads resume from the replica. Costs almost nothing (5 seconds of slightly heavier load on the primary per write) and eliminates the most common consistency bug users notice.

### 9. Scaling journey: 10 to 1M users

Each stage has one big architectural change driven by a concrete pain point. Build nothing preemptively.

**Stage 1: 10 to 100 users.** Single Postgres, single app server, no cache. About 10 reads/min, 0.1 writes/min. P95 = 30ms. Cost: ~$50/month. There's nothing to optimize. Postgres comfortably handles hundreds of QPS for indexed lookups. The app server sits at 5% CPU. The interviewer should stop you if you try to add Redis here. What you don't have: no CDN, no cache, no read replicas. What breaks first in the future is probably geographic latency once you take on users far from your single region, and that's solved by a CDN alone.

**Stage 2: 10k users.** Add Redis in front of the hottest read endpoints. ~6 reads/sec sustained, ~20/sec peak; writes still ~0.2/sec. What just broke: P95 is now 80ms because the catalog page does a 3-table JOIN that takes 60ms even on warm data, and every read hits Postgres. Cost: ~$150/month.

The fix is Redis in front of `GET /products/{id}`. TTL 60s, key `product:{id}`, value the full denormalized JSON. Cache-aside: pod checks Redis first, on miss queries Postgres, populates Redis, returns. TTL gets ±10% jitter to prevent synchronized expiry. Cache hit rate lands at ~80% from the Zipf shape, DB QPS drops from ~6/sec to ~1.2/sec, P95 on hit is 5ms, on miss 30ms, weighted ~8ms.

What you don't add yet: no CDN (most users are in one region; setup overhead isn't worth it); no read replicas (DB is barely loaded); no in-process LRU (with 2-3 app pods the marginal benefit is small); no denormalization (the JOIN is fast enough on cache miss).

Common mistake at this stage: caching the wrong endpoints. Caching the cart total is a correctness bug. Caching the search-results page works but doesn't help much (search has too many distinct queries; hit rate stays low).

**Stage 3: 100k users.** Add CDN, replicas, event-driven invalidation, and in-process LRU. ~58 reads/sec sustained, ~300/sec peak; writes ~3/sec peak. What just broke: users on three continents complain about 250ms+ load from APAC; the 60s TTL is no longer fast enough because a flash sale changes prices and 60s of stale prices generates complaints; primary DB CPU climbs because reads compete with the nightly catalog import. Cost: ~$2k/month.

The fixes: a CDN in front of all GET endpoints with `Cache-Control: public, max-age=60`, catching ~70% of reads at the edge and dropping global P95 to 30ms. Two read replicas, with reads routed to replicas and writes to primary, cutting primary load in half. Event-driven cache invalidation via Kafka: write to primary, publish `product.changed`, an invalidator consumer evicts Redis keys and calls the CDN purge API, while app pods subscribe to a Redis pub/sub channel for fast in-process eviction. And an in-process LRU on app pods (10MB, top 1000 keys) for sub-millisecond reads on the hottest products.

Numbers: CDN hit rate ~70% globally, Redis hit rate on CDN misses ~85%, DB QPS on replicas ~5/sec, writes-only ~3/sec on primary. User-perceived P95: 30ms (CDN hit), 50ms (CDN miss but Redis hit), 80ms (full miss). Price update propagation: under 2s end to end (Kafka + Redis pub/sub + CDN purge).

What you don't add: no per-region Redis yet (one central Redis is fine because Redis-to-app latency is acceptable from any region on the miss path); no denormalized view table yet (cache-miss path is fast enough on a replica with good indexes); no regional databases (primary stays single-region; replicas can be regional but writes route back).

Common mistake at this stage: trusting TTL alone. The flash-sale scenario is exactly why event-driven invalidation matters. TTL is the safety net; event-driven is the SLA.

**Stage 4: 1M users.** Per-region full stack (CDN POPs, regional Redis, regional read replicas, regional app servers), single primary in one region for writes, denormalized `product_view` table updated by CDC, ML-driven cache warming for predicted-hot products. ~600 reads/sec sustained, ~3000/sec peak; writes ~30/sec peak with nightly bulk imports pushing this to ~500/sec for a short window. What just broke: cross-region Redis writes for invalidation have 100ms+ latency so invalidation takes 500ms+ end to end and the flash sale starts before the cache is purged; the 3-table JOIN on cache miss takes 200ms and at 5% miss rate on 3000/sec that's 150 slow queries per second on the replicas; a few products go viral (one product gets 5000 req/s for an hour) and the Redis shard owning that key saturates at 100% CPU while surrounding shards sit idle. Cost: ~$30k/month.

The fixes: per-region Redis cluster, with cross-region invalidation routed through a global Kafka topic so each region's invalidator consumer evicts locally (no more cross-region Redis writes). Per-region read replicas, with inter-region async replication at ~2s P99 lag (acceptable). Denormalized `product_view` table updated by CDC, dropping cache-miss latency from 200ms to 5ms. Hot-key mitigation in Redis: app pods sample keys for hotness and promote anything above 100 req/s on a single pod to a long-TTL in-process cache; Redis read replicas (multiple read-only nodes per master) round-robin reads for hot keys. CDN cache warming for predicted-hot products before launch, either via a script that geo-distributes GETs across all POPs or via the CDN's pre-fetch API. Stale-while-revalidate everywhere, so the CDN serves stale up to 5min while refreshing in the background and the user never sees a slow page.

Numbers: CDN hit rate ~85% globally, Redis hit rate on CDN misses ~95%, DB QPS on replicas per region ~10/sec, writes-only ~30/sec on primary. User-perceived P95: 25ms globally. Price-update propagation: under 3s globally.

What you don't add: no multi-master DB (one primary; if it fails, promote a replica with ~60s downtime, writes are small enough that this is acceptable); no custom cache implementation (Redis + LRU is enough); no ML-based cache eviction (LRU is fine).

What you'd do at 10M+ users: honestly, you'd re-evaluate whether you're a catalog provider or a marketplace. At 10M users on a catalog, the read path is solved. The interesting problems become inventory consistency, payments, fraud, and search relevance, all of which are different system designs. You might also move to a multi-master DB (DynamoDB Global Tables, Spanner) if you have global write traffic, adopt edge compute (CloudFlare Workers, Fastly Compute@Edge) to run logic at the POP and eliminate origin hits for personalized-but-cacheable responses, or use a search-specific store (Elasticsearch) for the search path while the catalog read path stays as designed.

### 10. Reliability

**Cache stampede.** A popular key (`product:42`) is in cache. Its TTL of 60s expires at the moment 1000 users hit refresh. All 1000 miss, all 1000 hit the DB, DB CPU spikes.

Fixes in order: TTL jitter (each entry has TTL ± 10%, synchronized expiry becomes unlikely). Single-flight (per-key in-process lock; the first miss queries the DB and populates the cache, subsequent concurrent misses wait on the lock and read the populated value; 1000 misses become 1 DB query). Stale-while-revalidate (serve the just-expired value while one request asynchronously refreshes; users never wait). Probabilistic early refresh (each cache read has a small probability proportional to age/TTL of triggering a background refresh, so the cache is refreshed before it expires).

**Cache fallback to DB.** Redis goes down for 10 minutes. Every read becomes a full miss. The DB can't handle 3000 req/sec.

In-process LRU is the first line of defense; even with Redis dead, the top 1000 keys are served from app pod RAM. Read replicas absorb the rest (5 replicas at 100 QPS each = 500 QPS sustained). A circuit breaker on the DB: if DB latency exceeds 200ms P99, the app starts shedding cache-miss reads (return cached-stale data or 503 with `Retry-After: 30`). Throttle the cache-miss path with a per-pod concurrency cap. And when Redis comes back, don't let traffic flow to it immediately; the first 60 seconds would be 100% miss and the DB would melt. Pre-warm Redis with a one-shot scan or use Redis AOF persistence to recover the hot set on restart.

**Replica fail.** A read replica is unreachable. The connection pool detects and routes around it. If all replicas are down, fall back to primary. Alert immediately.

**Multi-region failure.** Global LB routes traffic to surviving regions. Those regions' caches were warm for *their* traffic; they're cold for the failed region's traffic. Expect a 5-15 minute degraded latency window as caches warm up. ML-driven warming can preload predicted-popular keys.

**Cold start dogpile.** Whole region's app fleet restarts at once. All in-process caches are cold. All requests fall through to Redis. Redis is fine; LRUs warm up within a minute. Fix: rolling restarts not all-at-once; pre-warm in-process cache on pod start with a small set of hottest keys fetched from Redis.

### 11. Observability

What must be instrumented from day one:

| Metric | Why it matters |
|--------|----------------|
| `cache.hit_rate.{layer}` (cdn, lru, redis) | The headline. Drops here precede latency regressions. |
| `cache.miss_path.latency.p99` | If this regresses, your DB or denormalized view path is slow. |
| `replica.lag.seconds.p99` | Should be under 1s. Spikes during bulk imports. |
| `read.latency.p50/p95/p99 by route` | Per-endpoint SLO tracking. |
| `db.qps.{primary, replica}` | Primary should be ~writes-only; replicas match expected miss rate. |
| `redis.hot_key.requests_per_sec` | Detects hot-key shards before they melt. |
| `invalidation.lag.p99` | Time from DB write to cache eviction across all layers. Under 2s. |
| `cdn.purge.latency.p99` | Time from purge API call to full propagation. |
| `stampede.coalesced.requests_per_sec` | Single-flight is firing; concurrent misses on the same key. |
| `circuit_breaker.{db, redis}.tripped` | Counts of times protection fired. |

Page on: cache hit rate < 70% for 5 min; replica lag > 30s; DB QPS spike > 10x baseline. Ticket on: any new hot-key signal; CDN miss rate trending up; invalidation lag P99 > 5s.

Two dashboards earn their keep. A "read path waterfall" per endpoint shows how many requests are served by which layer; this is how you find endpoints whose cache strategy is misconfigured. A "hot key heat map" ranks Redis keys by QPS; watch for one key dominating.

### 12. Gotchas the senior interviewer is listening for

Most of these only surface when the interviewer asks "what happens if..." The senior candidate brings them up unprompted.

Caching the wrong endpoints. Cart total, account balance, inventory count: these must not be cached, ever. Every team needs a clear policy and a code-review checklist.

Personalized content at the CDN. A page that varies by user with no `Vary: Authorization` header serves user A's view to user B. The bug is silent until someone notices.

`Cache-Control: public` on private data. Same problem as above. Default to `private`, opt in to `public`.

Stale write on write-through. Two concurrent writes can land in different orders in the DB and the cache. Mitigation: cache-aside (don't write the cache on write, just invalidate; let the next read populate).

Cache invalidation that doesn't actually invalidate. A pub/sub message lost in transit means an evicted entry stays alive. TTL is the backstop; never rely on event-driven alone.

Replication lag during backfills. A nightly import of 1M products generates 1M WAL entries and replicas fall hours behind. Throttle the import, or pause reads-from-replica during the window, or use a separate replica for backfill writes.

In-process cache divergence across pods. Pod A invalidates its LRU, pod B didn't get the message, pod B serves stale. Short TTL on in-process cache (~5min) even if pub/sub is in place. Pub/sub is the fast path; TTL is the correctness floor.

Versioned URLs and SEO. `?v=17` looks like a different URL to Googlebot. Include `<link rel="canonical">` pointing to the unversioned URL.

Read-your-writes broken by the load balancer. The 5-second-pin cookie only works if the LB routes the next request through a path that respects the cookie. If the LB strips cookies or your client is on a different origin, the pin fails silently. Test the path.

Stale-while-revalidate with a broken backend. SWR will keep serving stale forever if the background refresh always fails. Add a max-stale window (never serve content older than 1 hour even if refresh fails). After max-stale, return an error instead.

### 13. Follow-up answers

**1. Cache stampede.** Combine four patterns. TTL jitter so top keys don't expire in lockstep. Single-flight (per-key lock) so 1000 concurrent misses become 1 DB query. Stale-while-revalidate so a just-expired value is served while one goroutine refreshes in the background. Probabilistic early refresh so the cache is refreshed before expiry. Single-flight alone fixes the immediate spike; SWR makes it invisible to users.

**2. Cache fallback when Redis is down.** Every cache layer needs a defined fallback. In-process LRU as first line (top 1000 keys still served from pod RAM). Read replicas absorb the rest. Circuit breaker on the DB: if P99 latency exceeds a threshold, shed load (503 with `Retry-After`, or serve a degraded response). Per-pod query throttle. And critically, don't cold-start Redis under load; warm it via a scan from DB or use AOF persistence to restore the hot set, otherwise the first minute of post-recovery traffic is 100% miss again.

**3. Personalized pages and the CDN.** Personalization doesn't kill caching; it shifts what you cache. Find the seams between shared and personal content, then cache the shared parts. Four options: split into two requests (`GET /products/42` cacheable plus `GET /products/42/recommendations` personalized, browser assembles); edge personalization with CloudFlare Workers or Fastly Compute@Edge stitching in user-specific data at the POP; ESI (older pattern, still used) with the CDN inlining per-user fragments into a cached page; or vary by user-segment rather than user (if recommendations are bucket-based, 10 buckets × 1M products = 10M cache entries, still cacheable).

**4. Read-your-writes.** 5-second pin to primary after a write. On write, set a cookie `pin_to_primary_until=<now + 5s>` on the response. Subsequent reads from the same client inspect the cookie; if in the future, route to primary. After 5s the cookie expires and reads resume from replica. Cost: at write rate 3/sec, at most 15 concurrent pinned users. Negligible. For cross-device read-your-writes (user writes on phone, reads on laptop) the cookie doesn't work; either accept the inconsistency (typical) or use a server-side per-user version vector that the LB reads.

**5. Hot key in Redis.** One product gets 10k req/s; the Redis shard pegs at 100% CPU.

Mitigations in order of cost. In-process LRU on app pods (1000-entry LRU with 60s TTL catches the hot key locally; 10k req/s on the shard becomes maybe 100 req/s, one refresh per pod per minute). Redis read replicas (each master has 1-3 read-only slaves; app reads round-robin; throughput per key scales linearly). Key splitting (replace `product:42` with `product:42:{shard}` where shard is 0-9; reads pick a random shard, writes update all; spreads the load across 10 Redis shards). CDN (if the key corresponds to a cacheable URL, the CDN absorbs most of the traffic before it reaches Redis). For viral content, all four together.

**6. CDN cache miss thundering herd.** 100k users hit a new product launch URL at the same instant. CDN is cold. All requests fall through to origin.

Pre-warm the CDN before launch: hit the URL at every POP via a script that geo-distributes requests; the CDN now has the response at every POP. Origin shield (CloudFlare, Fastly feature): designate one POP as the shield that other POPs ask first on miss; origin sees one request per shield, not one per POP, reducing origin traffic 100x. Stale-while-revalidate if the page existed in any prior version. Single-flight at origin: even if the CDN doesn't coalesce, origin can.

The operational lesson: any large launch should be preceded by a cache warming run.

**7. Cache key design.** Cache `product:42`, then add staff-internal pricing.

Three options. Separate cache by role (`product:42:role:staff` vs `product:42:role:public`); two entries per product; cleanest but doubles cache memory if both roles are common. Wrap response with personalization: cache the shared `product:42` without prices, then at read time overlay the role-specific price from a separate small cache; one entry per product but more compute per read. Don't cache for staff: staff traffic is 0.1% of total, so skip the cache for them; one entry per product and staff reads always hit DB.

The right answer depends on the read ratio. If staff is <1% of traffic, "don't cache for staff" is correct. If staff is 50%, "separate cache by role" is correct. Cache key cardinality is your budget; spend it on the dimensions that matter.

**8. Replication lag during backfill.** Import 10M products overnight, replication lag balloons to 5 minutes.

During the import: pause reads from the most-lagged replica (connection pool detects lag and stops routing); throttle the import to a rate replicas can keep up with (slower import is fine, lagged reads are not); use a separate "backfill" replica that accepts the lag and never serves reads, the others stay caught up; pin reads to primary for the duration as a last resort.

Post-import: verify cache state (some Redis entries may be from before the import; bulk-invalidate the affected key range or wait for TTL); re-warm denormalized tables (if `product_view` is updated via CDC, the CDC pipeline may be lagging too).

The lesson: bulk writes are different from steady-state writes. They need their own write path and their own monitoring.

**9. Cache size estimation.** 1M products × 5KB = 5GB. Redis node has 8GB RAM. Enough?

What you forget: Redis overhead (each key has metadata, TTL, type, encoding; ~100 bytes per key; 1M × 100 = 100MB). Memory fragmentation (jemalloc overhead, real usage often 30-50% higher than naive math). Replication buffer (master keeps a backlog, default 1MB, often configured to 256MB). Client buffers (with 1000 connections, easily hundreds of MB). OS overhead (Redis can't have all 8GB). Eviction headroom (with `maxmemory-policy: allkeys-lru` you need ~10% headroom for clean eviction).

Realistic budget: an 8GB node holds at most 4-5GB of actual user data. If the hot set is 5GB, you need a bigger node or a cluster. Always size for hot set + 50% headroom. Plan for cluster mode from the start so you can add shards without re-architecting.

**10. Endpoint that should never be cached.** Inventory count at checkout must be exact, enforced across 30 engineers.

Code-level guard: a function `get_strong(key)` that explicitly bypasses cache; reviewer checks that strong-read endpoints use it. Framework-set `Cache-Control: no-store` for the endpoint; cannot be overridden by handlers. Linter rule that flags any function in `checkout/` calling the generic `get_cached` helper, fails CI. HTTP response audit: a nightly job samples production responses and flags any `/checkout/*` response with non-`no-store` cache headers. Naming convention: endpoints that must hit primary are `/api/v1/strong/*` or annotated `@StrongRead`, standing out in code review.

Correctness requirements need systematic enforcement, not "everyone knows not to cache that." A team of 30 has at least one engineer who doesn't know.

### 14. Trade-offs worth saying out loud

**Consistency vs latency.** Stronger consistency means more reads to the primary, longer pins, or shorter TTLs, all of which slow the read path. Weaker consistency (longer TTLs, more replica routing) means faster reads but more user-visible staleness. Pick per endpoint, not per system.

**Memory vs replicas.** Adding RAM to Redis is cheaper than adding read replicas. But Redis has a hard ceiling (one node's RAM); past that, you shard. Replicas have no per-node ceiling. At enterprise scale you have both; at startup scale you have neither.

**CDN cost vs origin offload.** CDN is paid by GB-out. At low traffic it's nearly free. At very high traffic it costs more than origin compute. Negotiate volume pricing or use multiple CDNs (Fastly + CloudFlare in front of origin shield).

**In-process cache vs distributed cache.** In-process is faster and free, but invalidation is harder (every pod needs to evict) and capacity is per-pod. Use in-process for top-N hottest keys only; use distributed for everything else.

**Denormalization vs query optimization.** Denormalized tables make reads faster and writes more complex. Sometimes a better index on the normalized schema solves the same problem. Measure before denormalizing.

**TTL vs event-driven invalidation.** TTL is simple and self-healing (if pub/sub fails, TTL eventually expires the entry). Event-driven is fast and accurate but operationally heavier. Use both: event-driven as primary, TTL as floor.

### 15. Common interview mistakes

Most weak answers fall into one of these.

Reaching for Redis without checking if a CDN would do the job. The CDN is cheaper and absorbs more traffic for cacheable responses. Always check what fraction of reads are cacheable shared responses before adding Redis.

Adding read replicas before fixing the cache. Replicas amplify wasted work if the hit rate is bad. Fix the cache first; the replicas may not be needed.

Uniform TTL across all endpoints. Different endpoints have different freshness requirements. A single TTL is either too long for some (correctness bugs) or too short for others (wasted DB load).

Trusting TTL alone for invalidation. TTL means a write can take up to N seconds to be visible. For a flash sale or a takedown, that's unacceptable. Add event-driven invalidation.

No fallback when the cache is down. When Redis dies the DB gets every request, and if you haven't prepared, the DB also dies. Always have a circuit breaker and a degraded mode.

Caching personalized responses without `Vary`. User A's response served to user B. Silent until a customer notices.

In-process cache as the only cache. Works at 1 pod. Fails at 100 pods (one write invalidates 1, the other 99 keep serving stale). Always combine with a distributed cache and pub/sub eviction.

Sizing Redis for the cold data set, not the hot one. You don't need to cache 1M products; you need to cache the top 10k. Size for hot set + headroom.

No observability per layer. "Cache hit rate is good" hides the case where one layer has 99% hit rate and another has 30%. Track each layer separately.

Designing for read-your-writes when it doesn't matter. Most user-facing reads tolerate 1-second staleness. Pinning every user to the primary after every write wastes capacity. Pin only for the few endpoints that need it (profile edits, settings).

Building per-region from day one. Regional infrastructure is 3-10x the operational cost. Most products never need it. Build single-region; add regions when latency complaints come from a specific geography.

Hand-rolling a cache when Redis exists. Custom in-memory caches are tempting and almost always wrong. Redis is battle-tested; your custom cache is not. Use Redis (or memcached) unless you have a specific reason not to.

If you can hit 9 of these 12, you're interviewing well. The pattern that separates senior from mid-level: senior candidates name the order in which to apply the patterns and the conditions under which each one is the *wrong* answer. Mid-level candidates name the patterns but apply them all uniformly.
