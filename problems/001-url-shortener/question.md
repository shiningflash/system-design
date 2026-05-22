---
id: 1
title: Design a URL Shortener
category: Basics
topics: [encoding, kv store, caching, sharding, read-heavy]
difficulty: Medium
solution: solution.md
---

## Scene

It is 10:08 AM. Your interviewer has just opened a blank doc. They paste a single sentence:

> *Design a URL shortener like bit.ly.*

They stop talking and look at you. The room is quiet. They are not going to give you more until you ask.

This is the most common opening in system design interviews. It looks simple. It is not. The interviewer is watching for whether you start drawing boxes immediately, which is the most common failure mode, or whether you pause and ask questions.

## Step 1: clarify before you design

Take 5 minutes. Do not draw anything yet. What would you ask the interviewer? Aim for at least five questions that meaningfully change the design if answered differently.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Traffic shape.** "How many new URLs per day? How many redirects?" Without this you cannot size storage, choose a cache, or decide between hash-based and counter-based shortcodes. A bit.ly-scale answer would be around 100M new URLs per month and 10x more reads than writes.
2. **Shortcode length and character set.** "Is there a target length? Custom aliases allowed?" 7 characters in base62 covers 3.5 trillion URLs which is plenty. Custom aliases change everything because now you cannot just hash, and you must reserve and check for conflicts.
3. **Retention.** "Do links expire? Can users delete them?" Permanent storage versus TTL changes the data store choice. Expiration enables sharding by time which simplifies sharding strategy.
4. **Read latency target.** "What is the redirect P99 latency we are targeting?" Under 100ms globally means you need a CDN or geo-distributed cache. Under 500ms in one region is much easier.
5. **Analytics.** "Do we need click counts? Geo? Referrer? Real-time or batch?" Analytics changes the architecture: real-time counts need a separate write path with a counter store. Batch analytics can lag and is much cheaper.
6. **Authentication and abuse.** "Anonymous shortening allowed? Account required? What do we do about phishing links?" This adds a rate limiter, a safe-browsing check, and a moderation pipeline.
7. **Consistency on conflicts.** "If two requests try to shorten the same long URL, do they get the same shortcode?" This is a deduplication decision that affects how you write to the database.

If you walked into the interview asking only "how many URLs" and "how long are the shortcodes" you have left the interviewer guessing. The five-to-seven question set above gives you enough to scope the design properly.

</details>

## Step 2: capacity estimates

Assume the interviewer gave you these numbers:

- 100M new short URLs per month
- Reads are 10x writes
- Average long URL length: 100 bytes
- Storage retention: 5 years
- Target redirect P99: 100ms globally

Compute (do this on paper before revealing):

1. Writes per second (sustained, not peak)
2. Reads per second (sustained)
3. Total URLs stored after 5 years
4. Storage required for the URL records alone (no logs, no analytics)
5. Cache size required to hit a 90% cache hit rate, assuming hot URLs follow roughly a Zipf distribution where the top 20% serve 80% of traffic
6. Bandwidth at peak (3x sustained)

<details>
<summary><b>Reveal: the math</b></summary>

**Writes per second.**
100M / month = 100M / (30 × 86400) ≈ 38 writes/sec sustained. Peak roughly 3x to 5x sustained, so call it ~150 writes/sec peak.

**Reads per second.**
10x writes = 380 reads/sec sustained, peak ~1500 reads/sec. This is small.

**Total URLs in 5 years.**
100M × 12 × 5 = 6 billion URLs.

**Storage.**
Per record: 7 byte shortcode + 100 byte long URL + 8 byte created_at + 8 byte user_id + ~30 byte overhead ≈ 150 bytes.
6B × 150 bytes = 900GB. Round up to 1TB. Fits on a single big machine, but you will shard for reasons other than capacity (latency, blast radius).

**Cache.**
Total redirects per day ≈ 380 × 86400 ≈ 33M reads/day. 90% hit rate means at most 10% of unique URLs are hot, but in practice with Zipf the hot set is much smaller. The top 1M URLs typically serve ~80% of traffic. 1M URLs × 150 bytes ≈ 150MB. Easily fits in a single Redis node, but you will replicate for availability.

**Bandwidth.**
Peak write: 150 × (100 byte URL + ~50 byte response) ≈ 22 KB/s. Negligible.
Peak read: 1500 × (small request + 302 redirect) ≈ negligible at the API layer. The redirect itself sends the user away.

**Key insight from the math.** This is a heavily read-skewed system with a tiny working set. Cache effectiveness dominates everything. The database is almost an afterthought once the cache is warm.

</details>

## Step 3: sketch the high-level architecture

Here is an intentionally incomplete diagram. Fill in the four `[ ? ]` boxes. Hint: think about what sits in front of the service, what sits behind it, and what speeds up reads.

```
                  ┌─────────────┐
   Client ──────► │   [ ? ]     │  (sits at the edge, terminates TLS, distributes traffic)
                  └─────┬───────┘
                        │
                        ▼
                  ┌─────────────┐
                  │  URL Service │  (stateless, horizontally scalable)
                  └──┬──────┬───┘
                     │      │
              read   │      │ write
                     │      │
                     ▼      ▼
              ┌──────────┐ ┌──────────┐
              │  [ ? ]   │ │   [ ? ]  │  (fast lookups vs durable storage)
              └──────────┘ └──────────┘

                                ▲
                                │ async writes
                            ┌───┴────┐
                            │ [ ? ]  │  (analytics, click counts)
                            └────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                  ┌──────────────────┐
   Client ──────► │  Load Balancer   │  (terminates TLS, geo-routes,
                  │  + WAF + Rate    │   distributes to nearest region)
                  │     Limiter      │
                  └─────────┬────────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │   URL Service    │  (stateless, horizontally scalable,
                  │ (read & write)   │   deployed per region)
                  └──┬───────────┬───┘
                     │           │
              read   │           │ write
                     │           │
                     ▼           ▼
              ┌──────────┐  ┌──────────────┐
              │  Cache   │  │  Database    │  (sharded by shortcode hash,
              │  (Redis) │  │  (Postgres/   │   replicated for failover)
              │  cluster │  │   Cassandra)  │
              └──────────┘  └──────┬───────┘
                                   │
                                   │ async (CDC or Kafka)
                                   ▼
                            ┌──────────────┐
                            │  Click Stream │  (Kafka → ClickHouse or
                            │  Aggregator   │   BigQuery for analytics)
                            └──────────────┘
```

Why each piece is here:

- **Load Balancer with WAF + rate limiter.** First because the service is internet-facing. The WAF blocks the obvious phishing and bot patterns; the rate limiter caps a single client from creating millions of links.
- **Stateless URL Service.** Lets you scale horizontally and deploy per region. No session affinity needed.
- **Cache in front of the database for reads.** With 90%+ hit rate, the database serves only the cold tail.
- **Database is the source of truth.** Choice between Postgres and Cassandra depends on access patterns; we cover that in the solution.
- **Async click pipeline.** Click counts are not on the redirect's critical path. The redirect must be fast; you fire-and-forget the click event.

</details>

## Step 4: API design

Define the API. Two endpoints minimum. What method, path, request body, response shape, status codes? What about errors?

<details>
<summary><b>Reveal: reference API</b></summary>

**Create short URL**

```
POST /api/v1/links
Content-Type: application/json
Authorization: Bearer <token>     # optional for anonymous; required if rate-limit-bypass desired

{
  "long_url": "https://example.com/very/long/path?with=query",
  "custom_alias": "my-link",       # optional, must match [a-zA-Z0-9_-]{4,32}
  "expires_at": "2026-12-31T00:00:00Z"  # optional, ISO 8601
}
```

Responses:

| Status | Meaning | Body |
|--------|---------|------|
| 201 Created | New short URL minted | `{ "short_url": "https://shrt.ly/abc1234", "shortcode": "abc1234", "long_url": "...", "expires_at": "..." }` |
| 200 OK | Same long_url submitted before by this user → return existing | same shape as 201 |
| 400 Bad Request | Invalid long_url, alias format, or expires_at in past | `{ "error": "invalid_alias", "message": "..." }` |
| 409 Conflict | Custom alias already taken | `{ "error": "alias_taken" }` |
| 429 Too Many Requests | Rate limit hit | `{ "error": "rate_limited", "retry_after": 60 }` |
| 451 Unavailable for Legal Reasons | URL flagged as phishing | `{ "error": "blocked_url" }` |

**Redirect (the hot path)**

```
GET /<shortcode>
```

Responses:

| Status | Meaning | Headers |
|--------|---------|---------|
| 302 Found | Redirect (use 302 not 301 so analytics work and we can change the target later) | `Location: <long_url>`, `Cache-Control: private, max-age=0` |
| 404 Not Found | Shortcode does not exist | |
| 410 Gone | Shortcode existed but expired or was deleted | |
| 451 Unavailable for Legal Reasons | URL was flagged after creation | |

Notes on the redirect:

- **302 not 301.** A 301 is cached by browsers permanently. If you ever need to change the target (you will, for moderation), browsers will skip your service entirely. Use 302 with explicit `Cache-Control: private, max-age=0`.
- **No body needed.** A 302 with `Location` header is enough. Some implementations send a tiny HTML body as fallback for ancient clients; it is unnecessary in 2025.

</details>

## Step 5: encoding scheme

You have to generate the 7-character shortcode somehow. Three serious approaches: **hash the long URL**, **encode an auto-incrementing counter**, **generate random and check for collision**. Take 10 minutes to write the pros and cons of each.

<details>
<summary><b>Reveal: comparison and recommended choice</b></summary>

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Hash long URL (MD5/SHA truncated to 7 chars)** | Take first 42 bits of hash, base62 encode | Deterministic (same URL → same code) which is great for dedup. Stateless. | Birthday collision at ~16M URLs. Must handle collisions explicitly. Hash is one-way so cannot recover long URL from shortcode (which you do not need to). |
| **Counter encoded in base62** | Increment a 64-bit counter, base62 encode it | Zero collisions ever. Compact. Predictable size. | Counter is global state, must be coordinated. Sequential codes are guessable (a security or scraping risk). |
| **Random + check** | Generate random 7-char string, check DB for collision, retry on conflict | Unguessable. Simple. | Latency spike when collision rate gets high (it stays low for years at our scale). Requires extra DB lookup. |

**The recommendation: counter-based with sharded counters.**

- Use a **bank of counters**, each pre-allocating a range. The URL Service grabs a range of (say) 10000 IDs at a time from a coordinator (Redis or ZooKeeper) and assigns from its local range. When it exhausts the range, it grabs another. This removes the per-write coordination overhead.
- To prevent guessability, **shuffle the bits** before base62 encoding. Take the 64-bit counter, XOR with a secret, then base62. Same uniqueness guarantee, but consecutive IDs do not produce consecutive shortcodes.
- For **deduplication** ("same long URL submitted twice"), separately maintain a `long_url_hash → shortcode` map (only for the same user, otherwise privacy). On submit, hash the long URL, look it up; if found, return existing; if not, mint new and store both.

Hash-based is a perfectly valid answer too; what matters is that you talk through the collision math and explain when you would switch approaches. Saying "hash" without acknowledging the birthday problem is what a weaker candidate does.

</details>

## Step 6: read your way through the rest in solution.md

You have done the hardest 60%. The rest is mostly mechanics: sharding strategy, cache invalidation, regional distribution, analytics pipeline, abuse handling. Read `solution.md` once you have made your own attempt at the follow-ups below.

## Follow-up questions

Try answering each in 2 to 3 sentences before reading the solution.

1. **Two users submit the same long URL within milliseconds of each other.** Do they get the same shortcode or different ones? What does your system do? What if they want different ones?
2. **Custom aliases.** How do you reserve them atomically? What happens if user A reserves "summer-sale" at the same moment user B does?
3. **A single shortcode goes viral and gets 100K req/s.** Is your single Redis enough? What if it is one specific cache key seeing all of that traffic? Look up "hot key problem".
4. **Phishing detection.** How do you check submitted URLs against Google Safe Browsing without slowing down the create path? What about URLs that were safe at creation but become malicious later?
5. **Analytics on click counts.** A short URL needs to display its total click count. Real-time? Eventually consistent? How do you avoid every redirect doing a database write?
6. **Custom domains.** Customers want `shrt.acme.com/abc1234` instead of `shrt.ly/abc1234`. What changes in the routing, certificate, and database?
7. **Link expiration with retention.** Links expire after 1 year by default. Do you delete the row, mark it expired, or just set TTL on the cache? What about for analytics that still need the historical link?
8. **What if a popular URL exists in the database but the cache evicts it during a traffic spike?** Describe the thundering herd, and prevent it.
9. **GDPR.** A user wants all their data deleted. How do you find every short URL they created across 8 sharded databases?
10. **You discover at 3am that the counter coordinator (Redis) lost its state due to a misconfigured failover and gave out the same range to two URL Service instances.** What is your blast radius? How do you detect and recover?

## Related problems

- **[Distributed Cache (009)](../009-distributed-cache/question.md)**, the same caching layer this problem leans on; understand its eviction and replication first.
- **[Rate Limiter (004)](../004-rate-limiter/question.md)**, the rate limiter on `POST /links` is one of the standard rate-limiter algorithms.
- **[Notification System (010)](../010-notification-system/question.md)**, the click stream pipeline at the bottom uses the same fan-out and durability patterns.
