## Solution: URL Shortener

### What this system is

A URL shortener is a lookup table with three hard requirements layered on top: short codes must be uniquely minted across many servers, the redirect must be very fast everywhere on the planet, and a bad link must be killable in under a minute even when copies are cached on six continents.

The architecture is a stateless service in front of a multi-layer cache in front of a sharded database. Reads beat writes 10 to 1. The hot working set is small enough (~150 MB) that a single Redis box serves about 80% of all traffic. The database is mostly a backstop for cold lookups.

The data model fits on a napkin: one main table keyed by shortcode, plus a tiny ledger for ID range allocations. Scale is not the hard part. At bit.ly numbers the system handles ~38 writes per second steady-state. The interesting work is at the edges: surviving a viral link, revoking a phishing URL fast, recovering from a counter coordinator misfire.

---

### 1. The two questions that matter most

If only two clarifying questions are allowed, ask these.

**How much traffic, and what is the latency target?** Without traffic numbers we cannot size anything. Without a latency target we cannot decide whether a CDN is mandatory or optional.

**Custom aliases, yes or no?** Aliases change the write path. With aliases, every create has to atomically check whether the specific string is taken. Without, every create just mints a fresh code from local memory.

Everything else (expiration policy, analytics shape, abuse handling) flows from those two.

---

### 2. The math

| Metric | Value |
|--------|-------|
| Writes per second, steady | ~38 |
| Writes per second, peak | ~150 |
| Reads per second, steady | ~380 |
| Reads per second, peak | ~1,500 |
| Total URLs after 5 years | 6 billion |
| Total storage | ~900 GB |
| Hot working set (top 1M URLs) | ~150 MB |

What this tells us:

- The system is small. A single Postgres handles the writes. Sharding is for failure isolation and latency, not capacity.
- Cache fits in memory. The whole hot set is 150 MB. One Redis node serves about 80% of all traffic.
- A 302 has no body. We serve nothing back. The user goes straight to the target.

---

### 3. The API

Two endpoints carry the entire product.

**Create:**
```
POST /api/v1/links
Idempotency-Key: <uuid>

{
  "long_url": "https://example.com/very/long/path?with=query",
  "custom_alias": "summer-sale",            // optional
  "expires_at": "2027-12-31T00:00:00Z"      // optional
}
```

**Redirect:**
```
GET /:shortcode
HTTP/1.1 302 Found
Location: <long_url>
Cache-Control: private, max-age=0
```

A few small but load-bearing choices:

| Choice | Reason |
|--------|--------|
| 302, not 301 | A 301 tells the browser to cache forever locally. We lose the ability to revoke. |
| `Idempotency-Key` required on create | Mobile drops connections. Without the key, retries mint duplicate codes. |
| `Cache-Control: private` on the redirect | Stops middleboxes from caching. The CDN can still cache because we control it explicitly. |

Status codes worth knowing:

| Code | Meaning |
|------|---------|
| `201` | Link created |
| `302` | Redirect (the common case) |
| `404` | Code does not exist |
| `409` | Custom alias already taken |
| `410` | Link was deleted or expired |
| `451` | Blocked for legal or safety reasons |

---

### 4. The data model

```mermaid
erDiagram
    short_links {
        varchar shortcode PK
        text long_url
        bigint creator_id
        timestamptz created_at
        timestamptz expires_at
        smallint status
        boolean custom_alias
        bytea long_url_hash
    }
    range_allocations {
        bigint range_start PK
        bigint range_end
        text instance_id
        timestamptz allocated_at
    }
```

<details markdown="1">
<summary><b>Show: the full SQL</b></summary>

```sql
CREATE TABLE short_links (
    shortcode      VARCHAR(16) PRIMARY KEY,
    long_url       TEXT NOT NULL,
    creator_id     BIGINT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at     TIMESTAMPTZ,
    status         SMALLINT NOT NULL DEFAULT 1,
    custom_alias   BOOLEAN NOT NULL DEFAULT FALSE,
    long_url_hash  BYTEA
);

CREATE INDEX idx_creator_created
    ON short_links (creator_id, created_at DESC);
CREATE UNIQUE INDEX idx_dedup
    ON short_links (creator_id, long_url_hash)
    WHERE creator_id IS NOT NULL;
CREATE INDEX idx_expires
    ON short_links (expires_at)
    WHERE expires_at IS NOT NULL;

CREATE TABLE range_allocations (
    range_start    BIGINT PRIMARY KEY,
    range_end      BIGINT NOT NULL,
    instance_id    TEXT NOT NULL,
    allocated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

</details>

Four small things doing real work:

**`shortcode` is the primary key.** Reads lookup by shortcode. Matching the primary key to the lookup key removes one index hop per redirect. Small per-request, but at 1,500 reads per second across 6 billion rows, it compounds.

**`long_url` is `TEXT`, not `VARCHAR(2048)`.** Google Maps share links, OAuth callbacks, and tracked marketing URLs routinely exceed 2,048 characters. Set no upper bound.

**`status` is `SMALLINT`.** Adding new states later (`pending_review`, `quarantined`) is just a new integer value. No schema migration.

**Unique index on `(creator_id, long_url_hash)`.** This is what makes dedup safe under concurrent writes. Two simultaneous POSTs of the same URL by the same user: one wins, the other gets a conflict and returns the existing shortcode. The database does the work, not the application.

Why Postgres and not Cassandra? Two reasons. We need ACID for the counter range allocation and the dedup unique index, which Postgres gives us for free. And at 38 writes per second steady, we are nowhere near needing a NoSQL store.

---

### 5. How short codes are minted

The mint path has three steps.

```mermaid
flowchart LR
    Coord[("Counter Coordinator<br/>Postgres sequence")]:::db
    Coord -->|"range 10000-19999"| Pod1["URL Service Pod 1<br/>local range"]:::app
    Coord -->|"range 20000-29999"| Pod2["URL Service Pod 2<br/>local range"]:::app
    Pod1 -->|"next ID: 10042"| XOR["XOR with 64-bit secret"]:::app
    XOR -->|"47829104"| B62["base62 encode"]:::app
    B62 -->|"Xk2pQz3"| Out[/shortcode/]:::ok

    classDef db fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef app fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef ok fill:#dcfce7,stroke:#15803d,color:#14532d
```

<details markdown="1">
<summary><b>Show: the mint function</b></summary>

```python
def mint_shortcode():
    if local_range.exhausted():
        local_range = coordinator.allocate_range(size=10_000)

    counter_id = local_range.next()
    scrambled = counter_id ^ XOR_SECRET   # bijection: no collisions, looks random
    return base62_encode(scrambled)
```

</details>

Three properties make this work in production:

| Property | Why it matters |
|----------|----------------|
| Range allocation | One coordinator call per 10,000 writes per pod. At 38 writes/sec per pod, one call every ~4 minutes. The coordinator is rarely on the critical path. |
| XOR bijection | Same uniqueness as the raw counter, but consecutive counters give scattered codes. Scrapers cannot guess the next one. |
| Postgres sequence as backing store | Sequences are durable and survive restarts. Plain Redis is the wrong choice because its failover can replay recently-issued IDs. |

Why not hash the URL? Birthday collisions. At 2 million URLs, 50% chance of one collision. At 6 billion, many. A retry loop handles them but adds unpredictable tail latency on writes.

---

### 6. The architecture

```mermaid
flowchart TB
    subgraph Edge["Client edge"]
        C([Web / Mobile]):::user
        CDN["CDN<br/>302 cache, 60s TTL"]:::edge
        GW["API Gateway<br/>TLS · rate limit · WAF"]:::edge
    end

    subgraph App["URL Service (stateless pods)"]
        S["URL Service<br/>+ in-process LRU"]:::app
        Coord["Counter Coordinator<br/>Postgres sequence"]:::db
    end

    Cache[("Redis<br/>hot set, 150 MB")]:::cache
    DB[("Postgres<br/>short_links<br/>range_allocations")]:::db

    K{{"Kafka<br/>link.created · click.event · link.blocked"}}:::queue

    subgraph Async["Async consumers"]
        CL[("ClickHouse<br/>analytics")]:::db
        SB["Safe Browse Worker"]:::app
    end

    C --> CDN
    CDN --> GW
    GW --> S
    S --> Coord
    S -->|reads| Cache
    S -.miss.-> DB
    S -->|writes| DB
    DB -->|CDC| K
    K --> CL
    K --> SB
    SB -.flag.-> DB

    classDef user fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef edge fill:#e2e8f0,stroke:#475569,color:#1e293b
    classDef app fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

Five things to notice:

- The CDN is in front of everything. For a viral link, ~95% of traffic never reaches origin.
- The synchronous write path has one external dependency: the counter coordinator. Called once per 10,000 writes per pod.
- ClickHouse is downstream of Kafka. If it falls over, redirects keep working. Click counts just lag.
- The Safe Browse Worker is also downstream of Kafka. A phishing URL may be live for up to a minute before being blocked. That trade-off buys ~400ms off the create latency.
- URL Service pods are stateless. State lives in Postgres and Redis. Roll the pods any time.

---

### 7. A redirect, traced

```mermaid
sequenceDiagram
    autonumber
    participant Bob
    participant CDN
    participant GW as API Gateway
    participant S as URL Service
    participant R as Redis
    participant DB as Postgres
    participant K as Kafka

    Bob->>CDN: GET /Xk2pQz3
    alt CDN cache hit (~95%)
        CDN-->>Bob: 302 Location (~10ms)
    else CDN miss
        CDN->>GW: forward
        GW->>S: route to nearest region

        rect rgb(241, 245, 249)
            Note over S,DB: in-service read path
            S->>S: check in-process LRU
            alt LRU hit
                Note over S: serve from pod memory
            else LRU miss
                S->>R: GET Xk2pQz3
                alt Redis hit (~90%)
                    R-->>S: long_url, status=active
                else Redis miss
                    S->>DB: SELECT WHERE shortcode=Xk2pQz3
                    DB-->>S: row
                    S->>R: SET TTL=1h ± jitter
                end
            end
        end

        S-->>Bob: 302 Location: <long_url>
        S->>K: click.event (fire and forget)
    end
```

Target latencies for the common paths:

| Path | P99 |
|------|-----|
| CDN hit | ~10ms (edge node distance) |
| In-process LRU hit | ~5ms |
| Redis hit | ~30ms regional, ~80ms cross-region |
| Postgres hit (cache miss) | ~50ms regional |
| Create (new link) | ~50ms (DB insert + Kafka) |

---

### 8. The scaling journey: 1,000 users to 1 billion links

At each stage, name what just broke and what fixes it. Build nothing preemptively.

```mermaid
flowchart LR
    S1["Stage 1<br/>~1,000 users<br/>1 server, 1 Postgres<br/>~$50/mo"]:::s1
    S2["Stage 2<br/>~100K users<br/>+ Redis<br/>+ Kafka clicks<br/>~$400/mo"]:::s2
    S3["Stage 3<br/>10M users, multi-region<br/>+ CDN + shards<br/>+ phishing pipeline<br/>~$5–10K/mo"]:::s3
    S4["Stage 4<br/>bit.ly scale<br/>+ cold storage<br/>+ Snowflake IDs<br/>1B+ links"]:::s4
    S1 --> S2 --> S3 --> S4

    classDef s1 fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef s2 fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef s3 fill:#fef3c7,stroke:#a16207,color:#713f12
    classDef s4 fill:#fce7f3,stroke:#be185d,color:#831843
```

#### Stage 1: ~1,000 users

One server. Single Postgres. Counter is a Postgres sequence (no separate coordinator yet). No CDN, no Kafka, no sharding. ~$50/month. Ships in a weekend.

This is enough because we see ten links a day. Anything more is overbuilt.

#### Stage 2: ~100,000 users

What breaks: read latency rises during the day, and Postgres CPU sits at 60%.

What we add:
- Redis in front of Postgres. ~85% cache hit rate. Postgres CPU drops to 10%.
- Move click counting to a Kafka pipeline so the redirect stops bumping a row on every hit.
- One read replica for durability.

Still single-region, still unsharded. ~$400/month.

#### Stage 3: 10 million users, multi-region

What breaks:
- European users see 200ms latency (server is in us-east).
- One viral link starts taking 30K req/s and pegs one Redis CPU.
- Phishing complaints arrive weekly.

What we add (in this order):
1. **CDN.** Edge caches the 302. Drops origin traffic by ~80% for popular links.
2. **Regional read paths.** URL Service + Redis + read replica per region. Writes still go to one primary region.
3. **Database sharding.** 64 shards by hash of shortcode. Spreads load.
4. **Dedicated counter coordinator.** Range allocation backed by Postgres or ZooKeeper. Never Redis.
5. **Phishing pipeline.** Safe Browsing API consumed via Kafka. Cache invalidation via pub/sub when a link is flagged.

Cost ~$5,000-10,000/month.

#### Stage 4: bit.ly scale (1B+ links)

What changes:
- CDN becomes critical, not optional. ~95% of viral traffic served from edge.
- Hot-key defenses (in-process LRU, Redis replicas, coalescing) are routine, not emergency.
- Storage goes multi-tier. Cold tier in S3 for links older than 1 year.
- Multi-region writes become tempting. Usually still not worth it because writes are only ~150/sec at peak.

The architecture has not fundamentally changed since Stage 3. We added more shards, more replicas, more edge coverage, more careful abuse handling. The data model is still the table from Stage 1.

At ~10x bit.ly, the counter coordinator becomes a chokepoint. Move to Snowflake-style IDs (machine ID + timestamp + sequence). No coordinator at all. Each pod mints codes forever. Trade-off: codes get one character longer.

---

### 9. Reliability

**Cache failure.** Redis goes down. All reads fall through to Postgres. The DB can handle 1,500 req/s for a while, not forever. The URL Service detects the failure and switches to protective mode: stricter rate limits, shed non-essential traffic with `503 Retry-After`.

**DB primary failure.** Promote a read replica. Writes are unavailable for 30 to 60 seconds during failover. Reads keep working from replicas everywhere else.

**Counter coordinator failure.** The worst case. If the coordinator hands the same range to two pods, both will mint colliding codes. The unique constraint on `shortcode` catches the collision at INSERT time (one wins, the other returns a 500 that the client retries). The `range_allocations` ledger detects the overlap within a minute and pages. Recovery: scan the conflicting range, find any duplicates, reissue affected codes.

**Region failure.** Global LB shifts traffic to healthy regions. The new region's cache is cold initially. Expect ~5 minutes of higher latency as it warms.

**CDN failure.** Rare but high-impact. Traffic falls to origin. Origin sees ~5x normal load. Auto-scaling buys time. If load is too high, shed with 503s and tighten rate limits.

---

### 10. Observability

| Metric | Why it matters |
|--------|----------------|
| `redirect.latency` p50/p95/p99 by region | The headline SLO. The whole product is "the redirect is fast." |
| `redirect.cache_hit_rate` per layer | Drop below 80% on Redis means something is wrong: TTL too short, cache node died, or the Zipf assumption broke. |
| `redirect.404_rate` | A spike usually means someone is scraping the namespace. Time to tighten rate limits. |
| `create.latency` p50/p99 | Slow creates point at the counter coordinator or the DB write path. |
| `create.dedup_rate` | If 30% of POSTs hit the dedup unique index, clients are buggy or not respecting Idempotency-Key. |
| `counter.range_overlap_detected` | Must be 0. Any non-zero value pages immediately. |
| `safe_browsing.flagged_rate` | A sudden spike means an active abuse campaign. |
| `kafka.click_event_lag` | If clicks lag more than 5 minutes, analytics is stale. |
| `db.replication_lag_p99` | Must stay under 1 second. |

**Page on:** redirect P99 > 200ms for 5 minutes. Cache hit rate < 70% for 5 minutes. Counter range overlap detected (any).

**Ticket on:** dedup rate spike. 404 rate spike (could be scraping, could be someone deleted a popular link).

---

### 11. Follow-up answers

**1. Two users submit the same long URL within milliseconds.**

If both are anonymous, give them different shortcodes. Two devices submitting the same URL should not be linkable as the same person.

If both are logged in as the same user, dedup. Compute `sha256(long_url)`. Use `INSERT ... ON CONFLICT (creator_id, long_url_hash) DO UPDATE ... RETURNING shortcode`. The unique index makes the race safe. The second INSERT sees the conflict and returns the first one's shortcode.

If a logged-in user wants two separate codes for the same URL (A/B testing two marketing campaigns), accept a `force_new=true` parameter to skip dedup.

**2. Custom aliases (atomic reservation).**

```sql
INSERT INTO short_links (shortcode, long_url, creator_id, custom_alias)
VALUES ($alias, $url, $user, TRUE)
ON CONFLICT (shortcode) DO NOTHING
RETURNING shortcode;
```

If `RETURNING` is empty, the alias was taken. Return 409. Atomic at the DB level. No application-level locks.

One extra concern: a custom alias like `abc1234` looks identical to a generated code. Reserve disjoint namespaces. Custom aliases must be at least 4 characters and contain a non-base62 character (hyphen, underscore), or at least 8 characters. Generated codes are exactly 7 base62 characters. No overlap.

**3. Hot key (viral link).**

Layered defense, cheapest first:

- **CDN.** ~95% of load served from the edge. Origin sees 5% of the storm.
- **In-process LRU on each pod.** Top 1,000 keys, 60s TTL. Zero network cost per hit.
- **Redis read replicas.** Round-robin reads across N replicas. Multiplies hot-key throughput by N.
- **Request coalescing on cache miss.** Only one goroutine fetches from DB. The rest wait on a per-key lock.

For predictable virality, pre-warm every region's cache before the announcement.

**4. Phishing detection.**

Google Safe Browsing takes 200 to 500ms. We cannot block create on that.

At create time: synchronous check against a small bloom filter of known-bad domains. Catches the worst offenders instantly. Asynchronously via Kafka: a worker calls Safe Browsing. If flagged, flip `status = blocked` and evict the cache everywhere via pub/sub. Continuously: a nightly job rescans links created in the last 30 days, because phishing campaigns sometimes weaponize URLs that were clean at creation.

The trade-off: a phishing URL can be live for 1 to 2 minutes before being blocked. To shrink the window further, fire the Safe Browsing call before returning from create but only wait for the response if it lands within 50ms.

**5. Click counts.**

`UPDATE short_links SET clicks = clicks + 1` on every redirect: 1,500 writes per second serialized on hot rows. Lock contention on viral links. It melts the DB.

Pipeline instead: redirect emits `{shortcode, ts, ip_hash, ua_hash}` to Kafka. A streaming job (Flink or ksqlDB) aggregates per shortcode in 1-minute windows. Aggregates go to Redis for rolling totals and ClickHouse for time series. The UI reads from those, not from the primary DB.

Consistency: eventually consistent, 1 to 2 minutes behind real time. Fine for almost every use case.

**6. Custom domains (shrt.acme.com/abc1234).**

- **TLS.** Per-customer cert via Let's Encrypt automation, or a wildcard cert if customers are subdomains.
- **Routing.** Load balancer SNI-routes the TLS connection to the URL Service. The service reads the `Host` header and looks up which tenant owns it.
- **Data model.** Add `tenant_id` to `short_links`, and a `custom_domains` table mapping domain to tenant. Shortcodes are now scoped per tenant. The same `abc1234` can exist for two tenants.
- **DNS.** Customer adds a CNAME from their domain to your load balancer.

This is how every URL shortener supports paid plans. The base shortener is the loss leader. Custom domains are the upsell.

**7. Expiration with retention.**

Three states: `active`, `expired`, `deleted`.

Expired links return 410. The row stays for analytics. Deleted links are gone. For GDPR, null out `long_url` while keeping the shortcode reservation so the code is not reissued.

A nightly job sets `status = expired WHERE expires_at < now()`. Cache entries naturally expire and the next fetch picks up the new status. Move rows older than 6 months to a cold storage table, then to S3 after a year.

**8. Thundering herd on cache miss.**

Without protection: a popular cache entry expires. 10,000 concurrent requests all hit the DB for the same row. CPU spikes. The DB potentially falls over.

Three fixes, all needed together:

- **Jittered TTL.** Add ±10% random jitter to every TTL. Prevents the top 1% of keys from expiring in the same second.
- **Request coalescing.** First request fetches from DB and populates cache. Others wait on a per-key lock and read the populated value. 10,000 DB reads become one.
- **Stale-while-revalidate.** Serve the stale value while one background goroutine refreshes. Same idea HTTP's `stale-while-revalidate` uses. Every major CDN does this internally.

**9. GDPR delete.**

Scatter-gather across all 64 shards: `WHERE creator_id = ?` runs in parallel, union the results. Either DELETE the rows or null out `long_url` and `creator_id` and set `status = gdpr_deleted`. Keep the shortcode reservation so the code is not reissued. Anonymize click history: hash or drop the user identifier and IP. Publish a pub/sub event to evict all of this user's keys from every cache. Confirm to the user within the 30-day GDPR window.

If many GDPR requests are expected, maintain a `creator_id -> shortcode[]` secondary index in a dedicated table. Avoids the scatter-gather for the common case.

**10. Counter coordinator handed the same range twice.**

**Detection.** Every range allocation writes to `range_allocations`. A periodic job (every minute) scans for overlaps:

```sql
SELECT a.*, b.*
  FROM range_allocations a, range_allocations b
 WHERE a.range_start < b.range_end
   AND b.range_start < a.range_end
   AND a.instance_id != b.instance_id;
```

Non-empty result pages immediately.

**Recovery.** Scan the conflicting range against `short_links`. Find IDs minted by both pods. For each duplicate with a different `long_url`, keep the earlier one and reissue the later one with a new shortcode. Notify the affected user.

**Prevention.** Never run the coordinator on plain Redis. Its failover can lose recent writes, replaying the same N seconds of IDs. Use Postgres sequences or ZooKeeper. The cost is a small bit of extra infrastructure. The benefit is correctness.

A mid-level answer covers only prevention. A senior answer covers detection, recovery, and prevention in one breath.

---

### 12. Trade-offs worth stating out loud

**Why not a multi-master database.** DynamoDB Global Tables or Spanner would let writes happen in every region. The operational complexity and write-latency tax are not worth it at URL-shortener traffic. Single-primary with read replicas is the right answer here.

**Why not content-addressed (hash-based) shortcodes.** If the code is a hash of the URL, the target cannot be changed. Mutability is a feature here: moderation, phishing takedowns, and legal compliance all need it.

**Why 302 and not 301 is not trivia.** It is the difference between having control over users' browser caches and not. The phishing use case makes 302 mandatory, not just preferable.

**What to revisit at 10x scale.** Move to Snowflake-style IDs to eliminate the counter coordinator entirely. Make the tiered cache (in-process LRU, regional Redis, DB) explicit in the design rather than emergent. At 64 shards, reconsider Postgres. Cassandra or DynamoDB start winning on operational cost at that point.

---

### 13. Common mistakes

**Diving straight into "use Redis and a database."** No clarification, no math, no API. Loses the interviewer in the first two minutes.

**Hashing the URL with MD5 and assuming no collisions.** Birthday paradox. At 6 billion URLs we will have many. Either acknowledge the collision-handling cost or use a counter.

**Using 301 instead of 302.** Browsers cache 301s forever locally. We lose the ability to block phishing, change targets, or count clicks at the service level. 302 is correct.

**Ignoring click analytics.** Every URL shortener has them. A senior candidate mentions the async pipeline even if not asked.

**Hand-waving cache invalidation.** "We'll use TTLs" handles routine eviction but not blocked links that need immediate revocation. Mention the pub/sub eviction path.

**No mention of phishing.** It is a real operational issue. Any URL shortener that accepts anonymous submissions needs to address it.

**Over-engineering from the start.** Do not propose a globally distributed multi-master CRDT-backed KV store unless the numbers demand it. They do not.

**Ignoring the hot key problem.** It will be asked. Layered caching is the answer.

**No story for counter coordinator failure.** This is where senior candidates separate from mid-level ones. Detection, recovery, and prevention in one breath.

If seven of these nine show up cleanly in the answer, the interview is at staff level. The three that matter most: 302 vs 301, the layered hot-key defense, and the counter coordinator failure story.
