## Solution: URL Shortener

### The short version

A URL shortener is a key-value lookup with one twist: reads beat writes by 10 to 1, and one specific key can suddenly take 100,000 requests per second when it goes viral.

The runtime is small. A stateless service in front of a cache in front of a sharded database. Short codes come from a counter that hands out ranges (no coordination on the write path), then get XOR-scrambled so they look random. Reads almost always hit cache. The database is a backstop.

The data model fits on a napkin: one table with shortcode as the primary key. Scale is not the hard part. At bit.ly numbers the whole system handles 38 writes per second steady. The hard part is the edges: surviving a viral link, handling phishing, recovering when the counter coordinator misbehaves.

---

### 1. The clarifying questions, in one paragraph

The single most important question is *how much traffic?* Without numbers you cannot pick a cache size, a sharding scheme, or even a database. Second is *custom aliases yes or no?* Because aliases change the write path from "mint a fresh code" to "atomically reserve this exact string." Third is *who can shorten?* Because anonymous shortening forces rate limits, abuse handling, and a phishing pipeline that logged-in services often skip.

Everything else (latency target, analytics shape, retention) follows from those three.

---

### 2. The math, in plain numbers

| Number | Value |
|--------|-------|
| New URLs per month | 100M |
| Writes per second, steady | ~38 |
| Writes per second, peak | ~150 |
| Reads per second, steady | ~380 |
| Reads per second, peak | ~1,500 |
| Total URLs after 5 years | 6 billion |
| Total storage | ~900GB |
| Hot working set (top 1M URLs) | ~150MB |

What the numbers tell you:

- **The system is small.** A single Postgres could handle the writes. You shard for failure blast radius and regional latency, not for capacity.
- **Cache wins everything.** 150MB hot set on one Redis node serves ~80% of traffic. Get the cache right and the database is almost an afterthought.
- **The redirect bandwidth is zero.** A 302 with a `Location` header has no body. The user goes away to the target. You serve nothing.

---

### 3. The API

Two endpoints. Create a link. Follow a link. Full schemas are in `question.md`. The non-obvious choices:

- **302, not 301.** A 301 says "permanent." Browsers cache it for life. If you ever need to block the link (phishing, copyright, etc.), the user's browser will not even ask you. With 302, every click goes through you, so you keep control.

- **`Idempotency-Key` is required on create.** Mobile drops connections. The client retries. Without the key, one tap becomes two short codes.

- **`Cache-Control: private, max-age=0` on the redirect response.** Tells the browser and middlebox proxies not to cache. Your CDN can still cache it because you control the CDN explicitly.

Status codes worth knowing: **410 Gone** for expired or deleted links (different from 404 because it tells the browser not to retry), **451 Unavailable for Legal Reasons** for blocked URLs.

---

### 4. The data model

One main table. That's it.

```sql
CREATE TABLE short_links (
    shortcode      VARCHAR(16) PRIMARY KEY,         -- base62, usually 7 chars
    long_url       TEXT NOT NULL,
    creator_id     BIGINT,                          -- NULL for anonymous
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at     TIMESTAMPTZ,                     -- NULL = no expiry
    status         SMALLINT NOT NULL DEFAULT 1,     -- 1=active, 2=expired, 3=blocked
    custom_alias   BOOLEAN NOT NULL DEFAULT FALSE,
    long_url_hash  BYTEA                            -- SHA-256 for dedup lookups
);

CREATE INDEX idx_creator_created ON short_links (creator_id, created_at DESC);
CREATE UNIQUE INDEX idx_dedup ON short_links (creator_id, long_url_hash)
    WHERE creator_id IS NOT NULL;
CREATE INDEX idx_expires ON short_links (expires_at) WHERE expires_at IS NOT NULL;
```

Four small choices doing real work:

**`shortcode` as the primary key.** Reads are by shortcode. Matching the PK to the lookup key skips an index hop on every single redirect. Sounds tiny. At 1,500 reads per second across 6 billion rows, it matters.

**`long_url` is `TEXT`, not `VARCHAR(2048)`.** Real URLs exceed 2048 characters. Google Maps share links. Deep OAuth callbacks. Trust the use case.

**`status` is `SMALLINT`, not an enum.** When you want to add `4 = pending_review` or `5 = quarantined`, no schema migration. Just write the new number.

**Unique index on `(creator_id, long_url_hash)`.** This is what makes dedup safe under concurrent writes. Two simultaneous INSERTs of the same URL by the same user: one wins, the other gets a conflict, returns the existing shortcode. No app-level locks.

> Why Postgres and not Cassandra? Because you want ACID for the counter-range allocation and the dedup unique index. Postgres gives you both for free. At 38 writes per second steady, you are nowhere near needing a NoSQL store. Save the complexity for a real reason.

---

### 5. How short codes get made

The engine of the write path. Three approaches, the recommendation, and why.

```mermaid
flowchart LR
  A[Counter coordinator] -->|gives range of 10000 IDs| B[URL Service Pod 1]
  A -->|gives different range| C[URL Service Pod 2]
  B -->|next ID| D[XOR with secret]
  D -->|base62 encode| E[shortcode]
```

**The chosen approach: sharded counter, scrambled.**

```python
def mint_shortcode():
    # Local range, no network call most of the time
    if local_range.exhausted():
        local_range = coordinator.allocate_range(size=10000)

    counter_id = local_range.next()                    # e.g. 8472913
    scrambled = counter_id ^ XOR_SECRET                # bijective scramble
    return base62_encode(scrambled)                    # 7 chars
```

Three things make this work:

**1. Range allocation removes the per-write coordination call.** A pod grabs 10,000 IDs from the coordinator. Then it serves 10,000 writes from local memory before talking to the coordinator again. At 38 writes per second per pod, that is one coordinator call every 4 minutes per pod. Trivial.

**2. The XOR scramble kills sequential guessability.** Without it, codes look like `Aaa0001, Aaa0002, Aaa0003`. A scraper iterates the namespace and harvests every link. With XOR, consecutive counters give scattered codes. Same uniqueness (XOR is bijective). No scraping risk.

**3. The coordinator's state has to survive a failover.** This is the trap. If the coordinator is plain Redis and the replica is stale, the same range gets handed out twice and two pods mint colliding codes. The fix is to back the coordinator with a strongly-consistent store (a Postgres sequence, or ZooKeeper).

> Why not just hash the URL? Birthday collisions. At 2 million URLs you have a 50% chance of one collision. At 6 billion, lots. You can handle collisions with a retry loop and a salt, but the loop's latency is unpredictable at the tail. Counters are cleaner.

---

### 6. The architecture, drawn out

```
                 +--------------------------------------+
                 |  Clients: web, mobile, anywhere      |
                 +-------------------+------------------+
                                     |
                                     v
                          +-----------------+
                          |   CDN / Edge    |  caches 302s, 60s TTL,
                          |  (CloudFront,   |  most viral traffic
                          |   Fastly)       |  never reaches origin
                          +--------+--------+
                                   |
                                   v
                          +-----------------+
                          |  Global LB +    |  anycast IP, routes
                          |  WAF + Rate     |  to nearest region,
                          |  Limiter        |  TLS termination
                          +--------+--------+
                                   |
            +----------------------+----------------------+
            |                      |                      |
            v                      v                      v
    +-------------+        +-------------+        +-------------+
    |  Region:    |        |  Region:    |        |  Region:    |
    |  us-east    |        |  eu-west    |        |  ap-south   |
    |             |        |             |        |             |
    | URL Service |        | URL Service |        | URL Service |
    |    + Redis  |        |    + Redis  |        |    + Redis  |
    |    + DB     |        |    + DB     |        |    + DB     |
    |  read repl  |        |  read repl  |        |  read repl  |
    +------+------+        +------+------+        +------+------+
           |                      |                      |
           +----------+-----------+-----------+----------+
                      |                       |
                      v                       v
            +-------------------+    +------------------+
            | Primary DB        |    | Counter          |
            | (one region for   |    | Coordinator      |
            | writes, sharded   |    | (Postgres        |
            | by shortcode hash)|    | sequence or ZK)  |
            +---------+---------+    +------------------+
                      |
                      |  CDC / outbox
                      v
            +-------------------+
            | Kafka topics      |
            | link.created      |
            | click.event       |
            | link.blocked      |
            +---+---------+-----+
                |         |
                v         v
        +------------+ +--------------+
        | ClickHouse | | Safe Browse  |
        | (analytics)| | Worker       |
        |            | | (async check)|
        +------------+ +--------------+
```

Five things to notice while reading this:

- **The CDN is in front of everything.** A viral link's 302 is cached at the edge. Most clicks never reach your origin. This is the single biggest performance lever in the design.
- **Each region is mostly self-sufficient.** Local URL Service, local Redis, local DB read replica. Reads stay regional. Only writes go to the primary region.
- **The write path has exactly one external dependency.** The counter coordinator. Called once per 10,000 writes per pod. Everything else is a local DB transaction.
- **Click analytics is downstream of Kafka.** Never in the redirect path. If ClickHouse falls over, redirects keep working. Click counts just lag.
- **Safe Browsing is downstream of Kafka.** Same idea. The create call doesn't wait for the safe-browsing check. A bad URL might be live for a minute before being blocked. That trade-off is worth the latency win.

---

### 7. A redirect, drawn end to end

```mermaid
sequenceDiagram
  participant U as User
  participant CDN
  participant LB as Load Balancer
  participant SVC as URL Service
  participant R as Redis
  participant DB as Postgres
  participant K as Kafka

  U->>CDN: GET /abc1234
  alt CDN hit (most common)
    CDN-->>U: 302 Location: long_url
  else CDN miss
    CDN->>LB: forward
    LB->>SVC: route to nearest region
    SVC->>R: GET abc1234
    alt Redis hit (~90 percent)
      R-->>SVC: long_url, status=active
    else Redis miss
      SVC->>DB: SELECT * WHERE shortcode=abc1234
      DB-->>SVC: row
      SVC->>R: SET abc1234 with 1hr TTL + jitter
    end
    SVC-->>U: 302 Location: long_url
    SVC->>K: fire click event (async, never blocks)
  end
```

Target latencies in the common path:

- CDN hit: **~10ms** (depends on user's distance to the edge).
- Redis hit, no CDN: **~30ms** regional, ~80ms cross-region.
- DB hit (cache miss): **~50ms** regional.

The create path is similar but writes happen instead of reads. Counter allocation (cached locally), DB INSERT, write-through to Redis, fire `link.created` event to Kafka, return 201. P99 around 200ms. The bottleneck is the synchronous DB insert.

---

### 8. The scaling journey: from one server to billions of links

This is the part interviewers care about most. At every stage, name what just broke and what fixes it. Build nothing preemptively.

#### Stage 1: 1,000 users

One server. Single Postgres. Cache in memory. No CDN, no Kafka, no sharding. Counter is a Postgres sequence. About $50/month. Ships in a weekend.

Enough because you see 10 links a day. Anything more is over-engineering.

#### Stage 2: 100,000 users

Something breaks: read latency spikes during the day, and Postgres CPU hits 60%.

Add Redis in front. The hot working set (a few hundred MB) fits in one Redis node easily. Cache hit rate goes from 0% to ~85%, Postgres CPU drops to 10%. Add one read replica for durability. Move click counting to a Kafka pipeline so the redirect path stops bumping a row on every hit.

Still one region. Still no sharding. Counter is still a Postgres sequence. About $400/month.

#### Stage 3: 10 million users, multi-region

New problems pile up:

- European users see 200ms latency because the server is in us-east.
- One viral link starts getting 30K req/s and pegs one Redis CPU.
- Phishing complaints arrive weekly. You need a process.

Fixes, in order:

- **Add a CDN.** Caches the 302 at the edge. Drops origin traffic by 80% for popular links.
- **Add regional read paths.** URL Service + Redis + read replica in each region. Writes still go to one primary region.
- **Shard the database.** 64 shards by hash of shortcode. Spreads load evenly. Now no single DB node feels the write pressure.
- **Move the counter to a dedicated coordinator.** Range allocation, durable state in Postgres or ZooKeeper.
- **Build a phishing pipeline.** Safe Browsing API consumed via Kafka. Async. Cache invalidation via pub/sub when a link is flagged.

Cost climbs to $5-10k/month. Latency is good worldwide.

#### Stage 4: 1 billion users (bit.ly scale)

What changes:

- The CDN is now critical infrastructure, not a nice-to-have. 95% of viral traffic served from the edge.
- Hot key problem becomes routine. Layered defenses (CDN, in-process cache, Redis replicas, request coalescing) all in place.
- Storage is multi-TB. Cold tier in S3 for links older than 1 year.
- Multi-region writes become tempting. Probably still not worth it. A single primary region for writes is simpler and the write rate is small.

The architecture has not fundamentally changed since Stage 3. You added more shards, more replicas, more CDN coverage, more careful abuse handling. The core data model is the same one you wrote in Stage 1.

#### What you would do at 10x of bit.ly

Move counter allocation client-side with a Snowflake-style ID: machine ID + timestamp + sequence. No coordinator at all. Each pod can mint codes forever without ever talking to a central service. Trade: you lose a tiny bit of compactness (codes get one character longer).

---

### 9. The four hard problems, fast

Same engine, four hard cases. Each one stresses a different feature.

- **Viral link (hot key)** uses layered caching. CDN at the edge, in-process cache on each pod, Redis read replicas, request coalescing on miss. The cheapest layer (CDN) buys the most.
- **Dedup race** uses a unique index on `(creator_id, long_url_hash)`. The DB serializes the writes. The second INSERT gets a conflict, returns the first's shortcode. No app-level locks.
- **Phishing detection** is async via Kafka. The create call returns fast. A worker calls Safe Browsing later. If flagged, status flips to blocked and pub/sub evicts the cache everywhere. The trade-off: the link is live for a minute before being killed.
- **Counter coordinator failure** uses a ledger. Every range allocation writes a row. A periodic job scans for overlapping ranges and alerts. Prevention: never run the coordinator on plain Redis. Use Postgres sequences or ZooKeeper.

One engine, four hard problems, all solved with small targeted patterns. That is the design victory.

---

### 10. Reliability

The engine has only one shared write dependency (the counter coordinator) and one shared read dependency (the cache). Each failure mode has a known fallback.

**Cache failure.** Redis is down. All reads fall through to the DB. The DB can hold 1,500 req/s for a while, but not forever. The URL Service detects Redis health and switches to a self-protective mode: stricter rate limits, smaller payloads, shed any non-essential traffic with 503 + `Retry-After`.

**DB primary failure.** Promote a read replica. Writes are unavailable for 30 to 60 seconds (typical Postgres failover). Reads keep working everywhere else. The counter coordinator must not lose state during this window, which is why it lives on its own store.

**Counter coordinator failure.** The worst case. If the coordinator hands the same range to two pods, two pods mint colliding codes. The DB unique constraint catches the collision (one INSERT wins, the other 500s), so the user-visible effect is some failed creates. The ledger detects the overlap within minutes and alerts. Recovery: scan the conflicting range, find any collisions that happened, reissue the duplicates manually.

**Region failure.** The global LB detects it and shifts traffic to healthy regions. The failed region's cache is cold elsewhere. Expect ~5 minutes of degraded latency as the new region's cache warms up.

**CDN failure.** Rare but happens (provider outage). Traffic falls back to origin. Origin sees 5x normal load. Auto-scaling buys time. If the load is too high, shed with 503s and pray.

---

### 11. Observability

What must be instrumented from day one:

| Metric | Why it matters |
|--------|----------------|
| `redirect.latency` p50/p95/p99 by region | The headline SLO. The whole product is "the redirect is fast." |
| `redirect.cache_hit_rate` | If this drops below 80%, something is wrong (TTL too short, cache box died, Zipf assumption broken). |
| `redirect.404_rate` | Spikes mean someone is scraping the namespace. New rate limit rule needed. |
| `create.latency` p50/p99 | Slow creates point to the counter coordinator or DB. |
| `create.dedup_rate` | If 30% of POSTs are dedups, your clients are buggy or not respecting the idempotency key. |
| `counter.range_fetches_per_sec` | Should be roughly write QPS divided by range size. |
| `counter.range_overlap_detected` | Should be 0. Page on any non-zero value. |
| `safe_browsing.flagged_rate` | Sudden spike means an abuse campaign. |
| `kafka.click_event_lag` | If clicks lag by more than 5 minutes, analytics is stale. |
| `db.replication_lag_p99` | Should stay under 1 second. |

Page on: redirect P99 > 200ms for 5 minutes. Cache hit rate < 70% for 5 minutes. Counter range overlap detected (any).

Ticket on: dedup rate spike. 404 rate spike (could be scraping, could be someone deleted a popular link).

---

### 12. Follow-up answers

These are the questions a senior interviewer is listening for. Each answer is short on purpose. The depth is in the *why*.

**1. Two users submit the same long URL within milliseconds.**

If both are anonymous, give them different shortcodes. Same URL going to two codes is fine, and it preserves privacy. Two devices submitting the same URL should not be linkable as the same person.

If both are logged in as the same user, dedup. Compute `sha256(long_url)`. Do an upsert: `INSERT ... ON CONFLICT (creator_id, long_url_hash) DO UPDATE ... RETURNING shortcode`. The unique index makes the race safe. The second INSERT sees the conflict and returns the first one's shortcode.

If they want different codes for the same URL, add `?force_new=true` to skip dedup. Real-world example: a marketing team running an A/B test wants separate codes so they can track each campaign's clicks separately.

**2. Custom aliases (atomic reservation).**

```sql
INSERT INTO short_links (shortcode, long_url, creator_id, custom_alias)
VALUES ($alias, $url, $user, TRUE)
ON CONFLICT (shortcode) DO NOTHING
RETURNING shortcode;
```

If `RETURNING` is empty, the alias was taken. Return 409. Atomic at the DB level. No locks needed.

One annoying case: a custom alias that looks like a generated code (`abc1234`). Reserve a namespace. Custom aliases must be at least 4 chars and contain at least one non-base62 character (hyphen or underscore), or at least 8 chars. Now generated codes and custom aliases live in disjoint pools. No conflict possible.

**3. Hot key problem (one shortcode getting 100K req/s).**

Layer four defenses, cheapest first:

- **CDN.** 99% of the load served from the edge. Origin sees 1,000 req/s.
- **In-process LRU cache on each pod.** Top 1,000 keys held in pod memory with a 60s TTL. Zero network cost.
- **Redis read replicas.** Read from N replicas, round-robin. Multiplies hot-key throughput by N.
- **Request coalescing on cache miss.** When the entry expires, only one goroutine refreshes. The rest wait on a per-key lock.

The cheapest layer (CDN) does the most work. The expensive layer (replicas) is a backstop.

For pre-known viral events: pre-warm the cache. If marketing announces something at noon, push the entry to every region's cache at 11:55. The storm hits a warm cache.

**4. Phishing detection.**

Google Safe Browsing takes 200 to 500ms. You cannot block create on that.

The approach:

- At create time: synchronous check against a small in-memory bloom filter of known-bad domains. Cheap. Catches the worst offenders.
- Asynchronously via Kafka: a worker calls the Safe Browsing API. If flagged, `status = blocked` and a pub/sub event evicts the cache everywhere.
- Continuously: a nightly job rescans URLs created in the last 30 days. Phishing campaigns sometimes weaponize URLs that were clean at creation.

Trade-off: a phishing URL can be live for 1 to 2 minutes before being blocked. To shrink the window, fire the Safe Browsing call before returning from create, but only block on the response if it lands within 50ms. After that, return success and let the async pipeline catch late detection.

**5. Click count analytics.**

`UPDATE short_links SET clicks = clicks + 1` on every redirect would melt the DB. 1,500 writes per second serialized on hot rows. Lock contention on viral links.

Pipeline:

- Redirect emits an event to Kafka: `{shortcode, ts, ip_hash, ua_hash}`.
- A streaming job (Flink, ksqlDB) aggregates per shortcode in 1-minute windows.
- Aggregates written to a counter store (Redis `INCR` for the rolling total, ClickHouse for the time series).
- UI queries the counter store, not the primary DB.

Consistency: eventually consistent, 1 to 2 minutes behind real time. Fine for almost every use case. If a customer needs real-time-exact counts, point them at the Kafka stream directly and explain the cost.

**6. Custom domains.**

`shrt.acme.com/abc1234` instead of `shrt.ly/abc1234`. What changes:

- **TLS.** Per-customer or wildcard cert. Let's Encrypt automation per domain works for most.
- **Routing.** Load balancer SNI-routes the TLS connection to the URL Service. The service reads the `Host` header and looks up which tenant owns it.
- **Database.** Add `tenant_id` to `short_links`, and a `custom_domains` table mapping domain to tenant. Shortcodes are now scoped per tenant. The same `abc1234` can exist for two tenants.
- **DNS.** Customer adds a CNAME from their domain to your load balancer.

Real-world: this is how every URL shortener supports paid plans. The base shortener is the loss leader; custom domains are the upsell.

**7. Link expiration with retention.**

Three states: `active`, `expired`, `deleted`.

- Expired links return 410. The row stays for analytics ("how many clicks did this campaign get before it expired?").
- Deleted links are gone. For GDPR you may even null out `long_url`.

A nightly job sets `status = expired` on rows where `expires_at < now()`. Cache entries naturally expire and the next fetch picks up the new status. No special invalidation needed.

For old data: keep expired rows in cold storage tables. Move to S3 after 6 months. Click analytics retains shortcode references but not the full row.

**8. Thundering herd on cache miss.**

A popular URL's cache entry expires. 10K concurrent requests all hit the DB for the same row. CPU spikes.

Three fixes, all needed:

- **Jittered TTL.** Don't let all the top 1% of keys expire in the same second. Add ±10% random jitter to every TTL.
- **Request coalescing.** First request fetches from DB and populates the cache. Others wait on a per-key lock and read the populated value. 10K misses become 1 DB read.
- **Stale-while-revalidate.** Serve the stale value while one background request refreshes. Same trick HTTP's `stale-while-revalidate` directive uses.

> Real example: every CDN does this internally. Cloudflare's tiered caching coalesces concurrent origin fetches for the same key.

**9. GDPR delete.**

User asks for all their data deleted.

- **Find.** Scatter-gather across all 64 shards. `WHERE creator_id = ?` runs in parallel. Union the results.
- **Delete or anonymize.** Either DELETE the rows, or NULL out `long_url`, `creator_id` and set `status = gdpr_deleted`. Keep the shortcode reservation so the code is not reissued.
- **Click history.** Anonymize the analytics records. Hash or drop the user identifier and IP.
- **Cache.** Publish a pub/sub event to evict all of this user's keys from every cache.
- **Confirm.** Tell the user within the regulatory window (30 days under GDPR).

If you expect many GDPR requests, maintain a `creator_id -> shortcode[]` secondary index in a dedicated table. Avoids the scatter-gather for the common case.

**10. Counter coordinator handed the same range twice.**

The scariest 3am page. Two pods are minting the same codes. Some creates will collide on the DB unique constraint and 500.

**Detection.** Every range allocation writes a row to a `range_allocations` ledger: `(range_start, range_end, instance_id, allocated_at)`. A periodic job (every minute) scans for overlapping ranges:

```sql
SELECT a.*, b.*
  FROM range_allocations a, range_allocations b
 WHERE a.range_start < b.range_end
   AND b.range_start < a.range_end
   AND a.instance_id != b.instance_id;
```

Non-empty result, page immediately.

**Recovery.** Scan the conflicting range against the actual short_links table. Find which IDs got minted by both pods. For each duplicate that ended up with a different `long_url` (the disaster case): keep the earlier one, contact the second user, reissue with a new shortcode.

**Prevention.** Never run the coordinator on plain Redis. Back it with Postgres (a real sequence) or ZooKeeper (designed for this). The cost of Redis failover during a primary swap is sometimes "the replica is stale by N seconds." For a counter, that means handing out the same N seconds worth of IDs twice.

The mid-level answer covers only prevention. The senior answer covers detection, recovery, and prevention in one breath.

---

### 13. Trade-offs worth saying out loud

**Why not a multi-master database.** DynamoDB Global Tables or Spanner would let writes happen in every region. The write-latency tax and operational complexity are rarely worth it at URL-shortener traffic (38 writes per second). Single-primary with read replicas is the right answer at this scale.

**Why not content-addressed (hash) URLs.** Sometimes asked as a curveball. If shortcodes are hashes of the long URL, you cannot ever change the target. Mutability is a feature here. You need it for moderation, for phishing, for legal takedowns.

**Why YAML/config for nothing.** Unlike workflow engines, a URL shortener doesn't need user-defined logic. The whole product fits in one schema. Don't invent a DSL.

**What you would revisit at 10x scale.**
- Snowflake-style IDs to remove the counter coordinator entirely.
- Tiered cache (in-process LRU, regional Redis, DB) as an explicit design, not emergent.
- Reevaluate Postgres at 64 shards. Cassandra or DynamoDB starts winning on operational cost at that point.

---

### 14. Common mistakes

Most weak answers fall into one of these:

**Diving straight into "use Redis and a database."** No clarification, no math, no API. Loses the interviewer immediately.

**Hashing the long URL with MD5 and assuming no collisions.** Birthday paradox. At 6 billion URLs you will have many. Either acknowledge the collision-handling cost or use a counter.

**Using 301 instead of 302.** Browsers cache 301s forever. You lose the ability to block phishing, change targets, or count clicks at the service. 302 is correct.

**Ignoring click analytics.** Every URL shortener has them. A senior candidate mentions the async pipeline even if not asked.

**Hand-waving cache invalidation.** "We'll use TTLs" is fine for routine eviction but does not cover blocked links that need immediate revocation. Mention pub/sub eviction.

**No mention of phishing.** It is a real operational issue. Acknowledge it.

**Over-engineering.** Don't propose a globally distributed multi-master CRDT-backed KV store unless the numbers demand it. They don't.

**Ignoring the hot key problem.** You will be asked. Layered caching is the answer.

**No story for the counter coordinator failure.** This is where senior candidates separate from mid-level ones.

If you hit 7 of these 9 cleanly, you are interviewing at staff level. The three that matter most: 302 vs 301, the layered hot-key defense, and the counter coordinator failure recovery. Those are the answers a senior architect listens for.
