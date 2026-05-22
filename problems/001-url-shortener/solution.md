## Solution: Design a URL Shortener

### TL;DR

A URL shortener is a heavily read-skewed key-value lookup with three things that make it interesting: minting unique short codes without a global coordinator on the hot path, surviving a 100k req/s viral link without a hot-key meltdown, and handling phishing without slowing the write path.

The design is a stateless service in front of a sharded database with an aggressive cache. Shortcodes are minted from a sharded counter (each service grabs a range), XOR-scrambled to look random, and base62 encoded. Reads almost always hit cache; the database is a fallback. Click analytics are asynchronous so the redirect stays under 50ms.

The interesting engineering is at the edges: avoiding collisions on a single popular code (replication and request coalescing), keeping the create path fast despite a safe-browsing check (async with a pre-flight allowlist), and recovering when the counter coordinator misbehaves (range ledger and idempotency).

### 1. Clarifying questions and why each matters

Already covered in `question.md`. The seven questions you must ask: traffic shape, shortcode length, retention, latency target, analytics, auth/abuse, dedup on conflicts. If you walked in without asking traffic shape you have no way to size anything; this is the single most common mistake.

### 2. Capacity estimates (full working)

- 100M new URLs / month → 100M / (30 × 86400) = **~38 writes/sec sustained, ~150/sec peak** (3-5x multiplier).
- 10x reads → **~380 reads/sec sustained, ~1500/sec peak**.
- 6B URLs after 5 years at ~150 bytes/record = **~900GB**. Fits on one machine; you shard for blast radius and tail latency, not capacity.
- Top 1M URLs serve ~80% of redirects (Zipf). 1M × 150B = **150MB hot set** → trivially cacheable.
- Bandwidth is negligible. The redirect is a 302 with a Location header and zero body.

The math tells you the system is small. Most of the architecture exists not for capacity but for **availability, latency, and abuse-resistance**.

### 3. API design

Already in `question.md`. The non-obvious calls:

- **302 over 301.** A 301 is browser-cached permanently; you lose the ability to change the target, kill phishing, or track clicks.
- **`Cache-Control: private, max-age=0`** on the redirect response, so intermediate proxies do not cache the redirect.
- **Idempotency on POST.** When the same `long_url` is submitted by the same authenticated user, return 200 with the existing shortcode rather than 201. Anonymous users always get a fresh shortcode to preserve privacy (their two devices submitting the same URL would otherwise leak that they are the same person).

### 4. Data model

```sql
CREATE TABLE short_links (
    shortcode    VARCHAR(16) PRIMARY KEY,        -- base62, typically 7 chars
    long_url     TEXT NOT NULL,
    creator_id   BIGINT,                         -- NULL for anonymous
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at   TIMESTAMPTZ,                    -- NULL = no expiry
    status       SMALLINT NOT NULL DEFAULT 1,    -- 1=active, 2=expired, 3=blocked
    custom_alias BOOLEAN NOT NULL DEFAULT FALSE,
    long_url_hash BYTEA                          -- SHA-256 of long_url for dedup lookups
);

CREATE INDEX idx_creator_created ON short_links (creator_id, created_at DESC);
CREATE INDEX idx_long_url_hash ON short_links (creator_id, long_url_hash);  -- for dedup
CREATE INDEX idx_expires ON short_links (expires_at) WHERE expires_at IS NOT NULL;
```

Choices and why:

- **`shortcode` as PK, not a synthetic ID.** Reads are by shortcode. Having the PK match the lookup key avoids an index hop on every redirect.
- **`long_url` as `TEXT`, not `VARCHAR(2048)`.** URLs do exceed 2048 chars in practice (Google Maps share links, deep OAuth callbacks). Trust the use case.
- **`status` as smallint not enum.** Easier to add new states (4=pending_review, 5=quarantined) without schema migrations.
- **`long_url_hash`** lets you dedup without a full-text comparison or a giant secondary index on TEXT.

### 5. Core algorithm: shortcode generation

#### The three approaches in detail

**a. Hash and truncate.**

```
shortcode = base62(truncate(SHA256(long_url), 42 bits))
```

42 bits in base62 ≈ 7 characters. Collision math by birthday paradox: probability of any collision exceeds 50% at √(2^42) ≈ 2M URLs. At 6B URLs you have many collisions. You handle them by appending a counter or salt and rehashing until unique. Works, but the "until unique" loop has unpredictable latency at the long tail.

**b. Counter, base62 encoded.**

```
id = next_counter_value()           # globally unique 64-bit integer
shortcode = base62(id ^ secret)     # XOR before encoding so codes look random
```

The XOR step is critical. Without it, shortcodes are sequential: `abc1230`, `abc1231`, `abc1232`. An attacker can iterate the namespace and harvest every link. With XOR (or any bijective scrambling like Feistel network), consecutive IDs map to scattered shortcodes. The XOR secret is just a 64-bit constant baked into the service.

**c. Random + check.**

```
while True:
    code = random_base62(7)
    if not db.exists(code):
        return code
```

At 7 chars (62^7 = 3.5T codes) collision rate stays below 0.1% for the first few hundred million URLs. Beyond that, retry latency climbs. Add a "code length" knob and bump to 8 chars at that scale.

#### Recommendation

**Counter with sharded range allocation, then XOR-scrambled.**

Why:
- No collisions, no retry latency.
- Sharded range allocation removes the per-write coordination round-trip. Each URL Service instance grabs a range of (say) 10,000 IDs from a coordinator (Redis with a Lua script, or a small dedicated service backed by Postgres). When the local range is exhausted, it fetches another.
- Counter state must be durable. The coordinator is the single point of contention but it is contacted only every 10,000 writes per instance, so the load is trivial.

The risk: if the coordinator hands out the same range twice (failover bug, misconfigured replica), you get two URL Service instances minting the same shortcode. The mitigation: log every range allocation with a unique ledger ID; on coordinator startup, replay the ledger to refuse already-allocated ranges. We come back to this in follow-up 10.

### 6. High-level architecture (detailed)

```
                                    ┌────────────────┐
   Browser ───────────────────────► │   CDN / Edge   │  cache 302s for popular shortcodes
                                    │   (CloudFront, │  TTL: short, e.g. 60s
                                    │    Fastly)     │
                                    └───────┬────────┘
                                            │
                                    ┌───────▼────────┐
                                    │ Global Load    │  Anycast IP
                                    │ Balancer       │  Routes to nearest healthy region
                                    └───────┬────────┘
                                            │
                  ┌─────────────────────────┼─────────────────────────┐
                  │                         │                         │
            ┌─────▼──────┐            ┌─────▼──────┐           ┌─────▼──────┐
            │  Region    │            │  Region    │           │  Region    │
            │  us-east   │            │  eu-west   │           │  ap-south  │
            │            │            │            │           │            │
            │ ┌────────┐ │            │ ┌────────┐ │           │ ┌────────┐ │
            │ │  WAF + │ │            │ │  WAF + │ │           │ │  WAF + │ │
            │ │  Rate  │ │            │ │  Rate  │ │           │ │  Rate  │ │
            │ │ Limiter│ │            │ │ Limiter│ │           │ │ Limiter│ │
            │ └───┬────┘ │            │ └───┬────┘ │           │ └───┬────┘ │
            │     │      │            │     │      │           │     │      │
            │ ┌───▼────┐ │            │ ┌───▼────┐ │           │ ┌───▼────┐ │
            │ │  URL   │ │            │ │  URL   │ │           │ │  URL   │ │
            │ │Service │ │            │ │Service │ │           │ │Service │ │
            │ │(N pods)│ │            │ │(N pods)│ │           │ │(N pods)│ │
            │ └─┬───┬──┘ │            │ └─┬───┬──┘ │           │ └─┬───┬──┘ │
            │   │   │    │            │   │   │    │           │   │   │    │
            │ ┌─▼─┐ │    │            │ ┌─▼─┐ │    │           │ ┌─▼─┐ │    │
            │ │Red│ │    │            │ │Red│ │    │           │ │Red│ │    │
            │ │is │ │    │            │ │is │ │    │           │ │is │ │    │
            │ │ cl│ │    │            │ │ cl│ │    │           │ │ cl│ │    │
            │ └───┘ │    │            │ └───┘ │    │           │ └───┘ │    │
            │       │    │            │       │    │           │       │    │
            └───────┼────┘            └───────┼────┘           └───────┼────┘
                    │                         │                         │
                    │     ┌───────────────────┴─────────────────────────┘
                    │     │
                    ▼     ▼
            ┌──────────────────────────┐
            │  Primary Database (DB)   │  Sharded by shortcode hash.
            │  (Postgres or DynamoDB)  │  Multi-region: regional writes
            │                          │  go to a primary in one region,
            │                          │  read replicas in every region.
            └────────────┬─────────────┘
                         │  CDC / outbox
                         ▼
            ┌──────────────────────────┐
            │  Click Stream Pipeline   │  Kafka → ClickHouse / BigQuery
            │  (async, off hot path)   │  Aggregated click counts per
            │                          │  shortcode, 1-minute rollup.
            └──────────────────────────┘
```

Component responsibilities:

- **CDN / edge cache.** Caches the 302 response for popular shortcodes. TTL is short (60s) so we can revoke phishing links. Most redirects never touch the origin.
- **Global LB + Anycast.** One IP, routed to the nearest healthy region. Removes a DNS round-trip and gives failover for free.
- **WAF + Rate Limiter.** WAF for obvious abuse (SQL injection in custom alias, etc.). Rate limiter caps `POST /links` per IP and per account.
- **URL Service.** Stateless. Owns counter range, talks to local cache, falls back to DB on miss. Replicates click events to Kafka asynchronously.
- **Regional Redis.** Caches reads. Replicated within the region for HA. Inter-region cache is not shared; if you have a miss in eu-west you go to the database, not to us-east's Redis.
- **Sharded DB.** One primary region for writes (we keep this simple; multi-master sharded DBs are a lot more complex than the problem requires). Read replicas everywhere.
- **Click pipeline.** Kafka with a 7-day retention; ClickHouse or BigQuery for aggregation. The aggregation is a streaming job that emits per-shortcode click counts every minute. The result is written back to a counter store (separate from the primary DB) that the UI reads.

### 7. Read path vs write path

**Write path (`POST /links`)**

1. Client → CDN → LB → Regional API Gateway → URL Service.
2. WAF and rate limiter checks. Rate limit: 100 links / hour for anonymous, 10,000 / hour for authenticated, by IP and account respectively.
3. Pre-flight URL validity check (DNS resolvable, not on local blocklist).
4. If `custom_alias` given: atomic `INSERT ... ON CONFLICT DO NOTHING` on the shortcode. If it returns 0 rows, return 409.
5. Otherwise: pop next ID from local counter range. If range empty, fetch new range from coordinator (this is the only call to a global service in the write path).
6. Scramble ID (XOR with secret), base62 encode.
7. `INSERT` into DB sharded by hash(shortcode). If insert fails due to duplicate shortcode (range allocation bug), bump ID and retry.
8. Write to local Redis with a short TTL (60s); rely on read path to populate longer-lived cache entries.
9. Fire-and-forget event to Kafka `link_created` topic.
10. Asynchronously call the safe-browsing API; if it flags the URL, update status to `blocked` (3) and invalidate cache.
11. Return 201 with the shortcode.

P99 target: 200ms. Bottleneck is the synchronous DB insert; counter range fetches are too rare to matter.

**Read path (`GET /<shortcode>`)**

1. Browser → CDN. If cached and not expired → return 302 directly. **80%+ of hot URL traffic should never reach origin.**
2. CDN miss → LB → Regional URL Service.
3. URL Service checks Redis. If hit and `status=active`, return 302. **Total time: ~5ms.**
4. Redis miss → DB read (regional read replica). If found and active → write to Redis with TTL of, say, 1 hour for hot keys (use random jitter to avoid synchronized expiry) → return 302. **Total time: ~30-50ms.**
5. DB miss → return 404.
6. Status is `expired` (2) → return 410.
7. Status is `blocked` (3) → return 451.

In parallel with the response, fire a click event to Kafka. **Never block the redirect on analytics.**

P99 target: 50ms regionally, 100ms cross-region. CDN cache means most users see <20ms.

### 8. Scaling

#### a. Caching strategy

- **Cache key:** the shortcode.
- **Cache value:** the long URL + status + expires_at (so we can return 410 without going to DB).
- **TTL:** 1 hour with ±10% jitter. Jitter prevents synchronized expiry causing a thundering herd against the DB.
- **Eviction:** LRU. The hot set is small enough that the working set fits.
- **Write-through, not write-back.** On `POST`, write to DB first, then populate cache. Write-back risks losing newly-created links if the cache node dies.
- **Cache invalidation.** Two reasons to invalidate: link is blocked (phishing) or link is deleted. On state change, publish to a Redis pub/sub channel that all URL Service instances subscribe to; each instance evicts the affected key from its local cache.

#### b. Database sharding

- **Shard key: hash(shortcode).** Spreads writes and reads evenly. Avoids hotspots that would happen with a sequential ID shard key.
- **Number of shards: 64 to start.** Easy to add more with consistent hashing; harder to remove. Start over-provisioned.
- **Indexes per shard.** PK on shortcode; secondary on `(creator_id, created_at)` for the "my links" page; secondary on `(creator_id, long_url_hash)` for dedup.

#### c. Read replicas

- One primary per shard in the home region. Read replicas in every other region. Replication lag SLO: 1 second P99.
- Reads use the closest replica; writes always go to the primary. Eventually consistent is fine here; a redirect lagging by 1 second is invisible to users.

#### d. CDN

- Cache 302 responses by shortcode. TTL 60 seconds.
- **The trade-off.** Longer TTL = more cache hits but slower phishing-link revocation. 60 seconds is a reasonable balance. If a link is flagged, it can still serve from CDN for up to 60s.
- For known-suspicious links: set `Cache-Control: no-store` so the CDN never caches them. They are now hot for the origin, but they should be a tiny fraction of traffic.

#### e. Geographic distribution

- Three regions (us, eu, ap). Each has its own URL Service, Redis, and DB read replicas.
- Writes route to the home region; if that region is offline, write availability is lost but reads continue. For most URL shorteners this trade-off is acceptable (writes are rare; reads must work).
- For higher write availability you would adopt a leaderless multi-region DB (DynamoDB Global Tables, Spanner). That is a more complex answer and only justified at YouTube/TikTok scale.

### 9. Reliability

- **Cache failure.** If Redis is down, reads fall through to the DB. The DB cannot serve 1500 req/s × 10 (assuming all traffic now falls through) = 15k req/s for an extended period. Mitigation: shed traffic (return 503 with `Retry-After`), and have the URL Service detect Redis health and start a self-protective slow path with stricter rate limiting.
- **DB primary failure.** Promote a replica. Writes are unavailable for 30 to 60 seconds (typical Postgres failover). Reads keep working from other replicas. The counter coordinator must not lose state during this; back it with Postgres or use ZooKeeper.
- **Counter coordinator failure.** This is the scariest one. If the coordinator gives the same range to two instances, you get duplicate shortcodes. Mitigation: range allocations are logged to a durable ledger; on coordinator startup it reads the ledger and refuses to re-allocate any range it has already given out. Each URL Service also writes its own "I am using range X" record so on misbehavior the ranges can be reconciled.
- **Multi-region failure.** Detected by the global LB; traffic shifts to healthy regions. Cache in the failed region is cold elsewhere; expect a 5-minute degraded latency window after a region swap.

### 10. Observability

What must be instrumented from day one:

| Metric | Why |
|--------|-----|
| `redirect.latency` by region, p50/p95/p99 | The headline SLO |
| `redirect.cache_hit_rate` | If this drops below 80% something is wrong |
| `redirect.404_rate` | Spikes indicate scrapers; can become a signal for new rate limit rules |
| `create.latency` p50/p99 | Slow creates indicate counter coordinator or DB problems |
| `create.dedup_rate` | If 30%+ of POSTs are dedups, your clients are buggy |
| `counter.range_fetches_per_sec` | Should be roughly QPS / 10000 |
| `counter.range_age_max` | Alerts on coordinators stuck |
| `safe_browsing.flagged_rate` | Sudden spike = abuse campaign |
| `kafka.click_event_lag` | If clicks lag by >5 min, analytics is stale |
| `db.replication_lag_p99` | Should be under 1s |

Alerting:
- Page on: redirect P99 > 200ms for 5min, cache hit rate < 70% for 5min, create error rate > 5%.
- Ticket on: dedup rate spike, 404 rate spike (could be scraping or could be that someone deleted a popular link).

### 11. Follow-up answers

**1. Two users submit the same long URL within milliseconds.**

If both are anonymous: they get different shortcodes. This is privacy-preserving and avoids accidentally linking two devices.

If both are the same authenticated user: dedup. The flow is:
- Compute `sha256(long_url)`.
- `SELECT shortcode FROM short_links WHERE creator_id = ? AND long_url_hash = ? LIMIT 1`.
- If hit, return existing.
- If miss, INSERT with the hash.

The race condition: both requests miss the SELECT and both proceed to INSERT. Solve with a unique index on `(creator_id, long_url_hash)` and `INSERT ... ON CONFLICT (creator_id, long_url_hash) DO UPDATE SET ... RETURNING shortcode`. The second INSERT gets the first's shortcode back. Idempotent under concurrency.

If they want different shortcodes despite the same URL: add a `force_new=true` query parameter. Skips the dedup lookup.

**2. Custom aliases (atomic reservation).**

```sql
INSERT INTO short_links (shortcode, long_url, creator_id, custom_alias)
VALUES ($alias, $url, $user, TRUE)
ON CONFLICT (shortcode) DO NOTHING
RETURNING shortcode;
```

If `RETURNING` returns 0 rows, the alias was taken. Return 409. This is atomic at the DB level. No additional locking needed.

The annoying case: aliases that "look like" generated shortcodes (`abc1234`). To prevent confusion, reserve a namespace: custom aliases must be at least 4 chars and contain at least one non-base62 character (a hyphen or underscore), or be at least 8 characters. This eliminates conflict between the two pools without runtime checks.

**3. Hot key problem (one shortcode getting 100K req/s).**

Symptoms: that one Redis key sees all the load. CPU on the Redis shard owning that key pegs at 100%, P99 latency for everyone else on that shard degrades, and you start losing traffic.

Mitigations in order of complexity:

1. **CDN.** The first defense. If 99% of those 100K req/s hit the CDN, your Redis sees 1000 req/s for that key, which it handles fine.
2. **Read-only replicas of the hot key.** Redis has read replicas. The URL Service round-robins reads across them. Multiplies the hot key throughput by N.
3. **In-process LRU cache on the URL Service.** A 1000-entry in-process cache with 60s TTL turns 100K Redis reads into a few dozen. This is the cheapest, biggest win.
4. **Probabilistic refresh** (request coalescing). If the cache entry is about to expire, only one goroutine refreshes it; the rest get the stale value. Prevents thundering herd on expiry.

Tier all four. For YouTube-trailer-scale virality you also pre-warm: when a marketing team is about to announce something, push the entry to every region's cache in advance.

**4. Phishing detection.**

The write path cannot wait for Google Safe Browsing (200-500ms call). The approach:

- **At create time:** Synchronous check against a small in-memory bloom filter of known-bad domains. Misses rarely; catches the worst.
- **Asynchronously:** Queue the new URL for safe-browsing verification via Kafka. A separate worker calls the external API. If flagged, it updates `status=3 (blocked)` in the DB and broadcasts cache invalidation.
- **Continuously:** A nightly job re-scans URLs created in the last 30 days. New phishing campaigns sometimes weaponize previously-clean URLs.

The trade-off: a phishing URL is live for up to a few minutes before being blocked. To reduce this window, send the URL to safe-browsing **before** returning the response from create, but only block on the response if it returns within 50ms. After that, return success and let the async pipeline handle late detection. This is a tail-latency vs accuracy trade.

**5. Click count analytics.**

Each redirect cannot do a DB write. 1500 req/s of `UPDATE short_links SET clicks = clicks + 1` would melt the DB and serialize writes on hot rows.

Pipeline:
- Redirect emits an event to Kafka (`{shortcode, ts, ip_hash, ua_hash}`).
- A streaming job (Flink or KSQL) aggregates per shortcode in 1-minute windows.
- Aggregate writes to a counter store (Redis with INCR is fine for the rolling count; ClickHouse for the time series).
- UI queries the counter store, not the primary DB.

Consistency: eventually consistent, up to 1-2 minutes behind. For 99% of use cases this is fine. If someone wants real-time-exact, point them at the raw Kafka stream and explain the cost.

**6. Custom domains.**

`shrt.acme.com/abc1234` instead of `shrt.ly/abc1234`. Changes:

- **TLS.** Need a wildcard or per-customer cert. Either provision via Let's Encrypt automation per domain, or use a wildcard cert across a domain you own.
- **Routing.** Load balancer SNI-routes incoming TLS to the URL Service. The service reads the Host header and looks up which tenant owns it.
- **Database.** Add a `tenant_id` column to `short_links`, and a `custom_domains` table mapping domain → tenant. Shortcodes are now scoped per tenant; the same `abc1234` could exist for two tenants.
- **Migration.** Existing shortcodes have an implicit `tenant_id = 0` (the default shrt.ly tenant). All new schema queries scope by tenant.
- **DNS.** Customer points their domain at your LB via CNAME. You handle the rest.

**7. Link expiration with retention.**

Three states: `active`, `expired`, `deleted`. Distinct because:
- **Expired** links return 410, but the row exists for analytics ("how many clicks before it expired?").
- **Deleted** links are gone; you may even hash-blank the long_url for GDPR.

For TTL: the cache TTL is independent of the link's `expires_at`. A nightly batch job sets `status=2 (expired)` on rows where `expires_at < now()`. The cache entries expire naturally and re-fetch finds the new status.

For storage: keep expired rows in cold storage tables. Move to S3 after 6 months. Analytics retains references.

**8. Thundering herd on cache miss.**

A popular URL's cache entry expires. Suddenly 10k req/s hit the DB for that one row. The DB CPU spikes.

Mitigations:
- **Jittered TTLs.** As mentioned. Top 1% of keys should not expire in the same second.
- **Request coalescing.** When the URL Service sees a cache miss for a key, it acquires a per-key in-process lock. The first request fetches from DB and populates cache; the rest wait on the lock and read the populated value. 10k concurrent misses become 1 DB read.
- **Stale-while-revalidate.** Serve the stale value while one goroutine refreshes. This is what HTTP `stale-while-revalidate` directives do; you implement the equivalent at the cache layer.

**9. GDPR delete.**

User requests deletion of all their data.

- Find: scatter-gather across all 64 shards. The query `WHERE creator_id = ?` runs in parallel; results are unioned.
- Delete or anonymize: either DELETE the rows, or NULL out `long_url`, `creator_id`, and set `status=4 (gdpr_deleted)`. Keep the shortcode reservation so the same shortcode is not handed to someone else later.
- Click history: anonymize the analytics records too. Hash the user identifier; drop any PII (IP).
- Cache: publish a pub/sub event to evict all of this user's keys from all caches.
- Confirm completion to the user within the regulatory window (30 days in most jurisdictions).

The scatter-gather is the expensive part. If you anticipate many GDPR requests, maintain a secondary index from `creator_id` to all their shortcodes in a dedicated index table.

**10. Counter coordinator misbehavior.**

Scenario: Redis (the coordinator) failed over, the replica had stale state, and it gave the range [10000, 20000] to instance A while another instance B was also assigned that range earlier.

Detection:
- Each URL Service writes a row to a `range_allocations` table at allocation time: `(range_start, range_end, instance_id, allocated_at)`.
- A periodic job checks for overlapping ranges. Alerts immediately.

Recovery:
- For all overlapping IDs, scan the DB to find which (if any) were actually used. Mark the conflicting ones (you have at most a few hundred duplicates).
- For each duplicate shortcode that ended up with two different `long_url` values: this is the disaster case. You have to decide whose long_url to honor. Practically, you keep the earlier one and contact the second user to apologize and reissue.
- Tighten the coordinator: never run a single coordinator without a strongly-consistent backing store. Use Postgres with a sequence, or ZooKeeper, instead of Redis.

This is a real "what would you do at 3am" scenario. The senior answer talks about detection, recovery, and prevention all in one breath. The mid-level answer only talks about prevention.

### 12. Trade-offs and what a senior would mention

- **Why no multi-master DB.** You could use DynamoDB Global Tables or Spanner. The cost in latency on writes and in operational complexity is rarely justified for URL shortener traffic. Single-primary with read replicas is the right answer at this scale.
- **Why not blockchain or content-addressed storage.** Sometimes asked as a curveball. Answer: deterministic hash-addressed URLs lose the ability to change targets (for moderation). Mutability is a feature here.
- **Why no analytics database optimization upfront.** Premature. Ship the basic Kafka → ClickHouse pipeline. Optimize only after you see real query patterns.
- **What I would revisit at 10x scale.**
  - Move counter allocation entirely to client-side using a Snowflake-style ID (machine ID + timestamp + sequence). Eliminates the coordinator.
  - Adopt a tiered cache (in-process LRU → regional Redis → DB) explicitly, not as an emergent property.
  - Re-evaluate the DB choice; Postgres at 64 shards is workable but Cassandra or DynamoDB starts winning on operations.

### 13. Common interview mistakes

- **Diving straight into "use Redis and a database".** No clarification, no math, no API. Loses the candidate immediately.
- **Hashing the long URL with MD5 and assuming no collisions.** Birthday paradox; at 6B URLs you have many. Either acknowledge the collision handling, or use a counter.
- **Ignoring analytics.** Click counts are part of every URL shortener. A senior candidate mentions the async pipeline even if not asked.
- **Cache invalidation handwave.** "We'll just use TTLs" is fine for routine eviction but does not address blocked links needing immediate revocation. Mention the pub/sub eviction.
- **No mention of phishing.** This is a real operational issue. Acknowledge it.
- **Overengineering.** Don't propose a globally distributed multi-master CRDT-backed K/V store unless asked to. Match the design to the stated scale.
- **Ignoring the hot-key problem.** You will be asked about it. Read replicas and in-process caches are the answers.

If you can hit 8 of those 13, you are interviewing well.
