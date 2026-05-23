---
id: 1
title: Design a URL Shortener
category: Basics
topics: [encoding, kv store, caching, sharding, read-heavy]
difficulty: Medium
solution: solution.md
---

## Scene

10:08 AM. Your interviewer opens a blank doc and pastes one sentence:

> *Design a URL shortener like bit.ly.*

Then they go quiet and look at you. They are not going to feed you more until you ask.

This is the most common opening in system design interviews. It looks easy. It is not. The interviewer is watching whether you start drawing boxes right away (the usual failure mode) or whether you pause and ask questions.

## Step 1: clarify before you design

Take 5 minutes. Don't draw anything yet. What would you ask? Aim for at least five questions that would meaningfully change the design depending on the answer.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Traffic shape.** "How many new URLs per day? How many redirects?" Without this you can't size storage, pick a cache, or choose between hash-based and counter-based shortcodes. A bit.ly-scale answer is around 100M new URLs per month with 10x more reads than writes.
2. **Shortcode length and character set.** "Target length? Custom aliases allowed?" 7 characters in base62 covers 3.5 trillion URLs, which is plenty. Custom aliases change the picture because you can't just hash; you have to reserve and check for conflicts.
3. **Retention.** "Do links expire? Can users delete them?" Permanent storage vs TTL changes the data store. Expiration also opens the door to sharding by time, which simplifies things.
4. **Read latency target.** "What's the redirect P99 we're shooting for?" Under 100ms globally means a CDN or geo-distributed cache. Under 500ms in one region is far easier.
5. **Analytics.** "Click counts? Geo? Referrer? Real-time or batch?" Real-time counts need a separate write path with a counter store. Batch can lag and is much cheaper.
6. **Auth and abuse.** "Anonymous shortening? Account required? What do we do about phishing?" This pulls in a rate limiter, a safe-browsing check, and a moderation pipeline.
7. **Dedup on conflicts.** "If two requests shorten the same long URL, do they get the same shortcode?" This is a deduplication decision that drives the write path.

If you walked in asking only "how many URLs" and "how long are the shortcodes," you've left the interviewer guessing. The seven above give you enough to scope the design.

</details>

## Step 2: capacity estimates

Assume the interviewer hands you these numbers:

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
100M / month = 100M / (30 × 86400) ≈ 38 writes/sec sustained. Peak is 3x to 5x sustained, so call it ~150 writes/sec peak.

**Reads per second.**
10x writes = 380 reads/sec sustained, ~1500/sec peak. Small.

**Total URLs in 5 years.**
100M × 12 × 5 = 6 billion URLs.

**Storage.**
Per record: 7 byte shortcode + 100 byte long URL + 8 byte created_at + 8 byte user_id + ~30 byte overhead ≈ 150 bytes.
6B × 150 bytes = 900GB. Round up to 1TB. Fits on a single big box, but you shard for reasons other than capacity (latency, blast radius).

**Cache.**
Total redirects per day ≈ 380 × 86400 ≈ 33M reads/day. 90% hit rate means at most 10% of unique URLs are hot, but with Zipf the hot set is much smaller. The top 1M URLs typically serve ~80% of traffic. 1M × 150 bytes ≈ 150MB. Fits in a single Redis node, but replicate for availability.

**Bandwidth.**
Peak write: 150 × (100 byte URL + ~50 byte response) ≈ 22 KB/s. Negligible.
Peak read: 1500 × (small request + 302 redirect), also negligible at the API layer. The redirect itself sends the user away.

**The takeaway.** Heavily read-skewed system, tiny working set. Cache effectiveness dominates everything. Once the cache is warm, the database is almost an afterthought.

</details>

## Step 3: sketch the high-level architecture

Here's an intentionally incomplete diagram. Fill in the four `[ ? ]` boxes. Hint: think about what sits in front of the service, what sits behind it, and what speeds up reads.

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

- LB with WAF and rate limiter goes first because the service is internet-facing. WAF blocks the obvious phishing and bot patterns; the rate limiter caps a single client from minting millions of links.
- Stateless URL Service scales horizontally and deploys per region. No session affinity.
- Cache in front of the database for reads. With 90%+ hit rate, the database serves only the cold tail.
- Database is the source of truth. Postgres vs Cassandra depends on access patterns; covered in the solution.
- Async click pipeline. Click counts are not on the redirect's critical path. The redirect has to be fast, so you fire-and-forget the click event.

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

- 302 not 301. Browsers cache 301s permanently. If you ever need to change the target (you will, for moderation), browsers skip your service entirely. Use 302 with an explicit `Cache-Control: private, max-age=0`.
- No body needed. A 302 with `Location` header is enough. Some implementations send a tiny HTML body for ancient clients; you can skip it.

</details>

## Step 5: encoding scheme

You have to mint a 7-character shortcode. Three serious approaches: hash the long URL, encode an auto-incrementing counter, or generate random and check for collision. Take 10 minutes to write the pros and cons of each.

<details>
<summary><b>Reveal: comparison and recommended choice</b></summary>

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Hash long URL (MD5/SHA truncated to 7 chars)** | Take first 42 bits of hash, base62 encode | Deterministic (same URL → same code) which is great for dedup. Stateless. | Birthday collision at ~16M URLs. Must handle collisions explicitly. Hash is one-way so cannot recover long URL from shortcode (which you do not need to). |
| **Counter encoded in base62** | Increment a 64-bit counter, base62 encode it | Zero collisions ever. Compact. Predictable size. | Counter is global state, must be coordinated. Sequential codes are guessable (a security or scraping risk). |
| **Random + check** | Generate random 7-char string, check DB for collision, retry on conflict | Unguessable. Simple. | Latency spike when collision rate gets high (it stays low for years at our scale). Requires extra DB lookup. |

**Recommendation: counter-based with sharded counters.**

- Use a bank of counters, each pre-allocating a range. The URL Service grabs a range of (say) 10000 IDs at a time from a coordinator (Redis or ZooKeeper) and hands them out from its local range. When it exhausts the range, it grabs another. No per-write coordination round-trip.
- To prevent guessability, shuffle the bits before base62 encoding. Take the 64-bit counter, XOR with a secret, then base62. Same uniqueness guarantee, but consecutive IDs don't produce consecutive shortcodes.
- For dedup ("same long URL submitted twice"), keep a separate `long_url_hash → shortcode` map (scoped per user for privacy). On submit, hash the long URL, look it up; if found, return existing; if not, mint and store both.

Hash-based is a fine answer too. What matters is that you talk through the collision math and say when you'd switch approaches. Saying "hash" without acknowledging the birthday problem is what a weaker candidate does.

</details>

## Step 6: read your way through the rest in solution.md

You've done the hardest 60%. The rest is mechanics: sharding, cache invalidation, regional distribution, analytics pipeline, abuse handling. Read `solution.md` once you've made your own attempt at the follow-ups below.

## Follow-up questions

Try answering each in 2 to 3 sentences before reading the solution.

1. **Two users submit the same long URL within milliseconds.** Do they get the same shortcode or different ones? What does your system do? What if they want different ones?
2. **Custom aliases.** How do you reserve them atomically? What happens if user A reserves "summer-sale" at the same moment user B does?
3. **A single shortcode goes viral and takes 100K req/s.** Is your single Redis enough? What if one specific cache key sees all of that traffic? Look up "hot key problem".
4. **Phishing detection.** How do you check submitted URLs against Google Safe Browsing without slowing the create path? What about URLs that were safe at creation but become malicious later?
5. **Click counts.** A short URL needs to display its total click count. Real-time? Eventually consistent? How do you avoid every redirect doing a database write?
6. **Custom domains.** Customers want `shrt.acme.com/abc1234` instead of `shrt.ly/abc1234`. What changes in the routing, certificate, and database?
7. **Expiration with retention.** Links expire after 1 year by default. Delete the row, mark it expired, or just set TTL on the cache? What about analytics that still need the historical link?
8. **A popular URL exists in the database but the cache evicts it during a traffic spike.** Describe the thundering herd and prevent it.
9. **GDPR.** A user wants all their data deleted. How do you find every short URL they created across 8 sharded databases?
10. **3am: the counter coordinator (Redis) lost its state due to a misconfigured failover and handed the same range to two URL Service instances.** What's the blast radius? How do you detect and recover?

## Related problems

- [Distributed Cache (009)](../009-distributed-cache/question.md), the caching layer this problem leans on. Understand its eviction and replication first.
- [Rate Limiter (004)](../004-rate-limiter/question.md), the rate limiter on `POST /links` is one of the standard algorithms.
- [Notification System (010)](../010-notification-system/question.md), the click stream pipeline at the bottom reuses the same fan-out and durability patterns.
