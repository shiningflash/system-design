---
id: 8
title: Design a Web Crawler (Googlebot)
category: Distributed Systems
topics: [bfs, frontier, deduplication, politeness, distributed coordination]
difficulty: Hard
solution: solution.md
---

## Scene

Second slot of the loop. The interviewer worked on Google's indexing pipeline a long time ago and wants a war-game. They pull up a whiteboard tab and write one line:

> *Design a web crawler at Google's scale. Walk me through it.*

Then they wait. Most candidates start by drawing a queue and a worker pool. At this scale that gets you the wrong answer, because the system breaks for reasons that have nothing to do with the queue. Slow down. Figure out what kind of crawler this is before you commit to a shape.

## Step 1: clarify before you design

Take 5 minutes. The shape of the system changes a lot depending on what is being crawled and why. Write down at least seven questions before you peek.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. Scope. "HTML only, or also images, video, PDFs, JavaScript-rendered pages?" The default Googlebot answer is HTML first, a separate render path for JS-heavy pages, and a separate pipeline for media. Assume "everything in one pass" and you will design a much larger and slower system than the interviewer wanted.
2. Seed set. "What are we starting from? A seed list, sitemaps, or the existing web graph?" Cold-starting a crawler is a different problem from running an established one. Real crawlers always have an existing frontier; they never start from zero in production.
3. Freshness target. "How fresh must the index be? Hourly for news, daily for the rest, monthly for dormant pages?" This drives the recrawl scheduler, which is often a bigger system than the discovery crawler.
4. Politeness rules. "Do we respect robots.txt strictly? What is the default per-host rate? What is the SLA back to webmasters?" One stranger's site should never be melted by your crawler. Non-negotiable.
5. Output shape. "What does the crawler produce? Raw HTML to a blob store, parsed text to an indexer, structured records?" Most candidates skip this and end up with no plan for what happens after the fetch.
6. Scale and budget. "Pages per day? Hosts? Bandwidth budget? Storage budget?" 5 billion fetches per day at 100KB per page is 500TB per day. If the interviewer says 1 billion pages per day, the storage problem is a tenth the size.
7. JavaScript rendering. "Do we render with a headless browser, or only static HTML?" Rendering is 10x to 100x more expensive than a plain fetch. Treat it as a separate, smaller pipeline.
8. Trust and abuse. "How do we deal with crawler traps, infinite calendar URLs, spam farms, soft 404s?" The interviewer wants to see whether you understand the adversarial parts of the open web.

If you only asked "how many pages per day", you skipped politeness, scope, and freshness, which together account for most of the design's complexity. Politeness in particular is a load-bearing constraint, not an afterthought.

</details>

## Step 2: capacity estimates

Suppose the interviewer hands you these numbers:

- Crawl target: 5 billion pages per day (the public web is roughly 50 to 100 billion useful pages; we recrawl the most useful subset frequently and the long tail rarely)
- Average HTML page size: 100KB compressed, ~500KB uncompressed
- Average outlinks per page: 20
- Total URLs known to the system (frontier plus already-seen): 50 billion
- Distinct hosts in the frontier: 500 million
- Default politeness: 1 request per host per second
- Time horizon for storage: 30 days hot, then archived

Do the math yourself before revealing:

1. Fetches per second (sustained, peak)
2. Bandwidth in and out at peak
3. Storage per day for raw HTML, and total for 30 days
4. Storage for the URL frontier and the seen-URL dedup index
5. Concurrent host slots needed, given the per-host rate limit and the daily target
6. Fetcher node count, given one fetcher sustains around 500 simultaneous TCP connections

<details>
<summary><b>Reveal: the math</b></summary>

Fetches per second. 5B / 86400 = ~58K fetches/sec sustained. Peak 2x to 3x, call it 150K fetches/sec peak.

Bandwidth. At 100KB per page compressed, ingress = 58K × 100KB = 5.8 GB/sec sustained, ~17 GB/sec peak. Roughly 50 to 150 Gbps of inbound bandwidth across the fleet. Real Googlebot pulls more than this; we are sizing one well-fed crawler.

Storage for raw HTML. 58K/sec × 86400 = 5B pages/day × 100KB = 500TB/day of compressed HTML. 30 days hot = 15PB. After dedup by content hash, expect 20% to 40% savings because near-identical templates show up everywhere. Round to ~10PB hot.

URL frontier storage. 50 billion URLs. A canonicalized URL averages ~80 bytes. Add metadata (host_id 4 bytes, priority float 4 bytes, scheduled_at 8 bytes, attempt_count 1 byte, parent ref 8 bytes) and you get ~110 bytes per row. 50B × 110 = ~5.5TB of frontier metadata. Sharded, fits comfortably.

Seen-URL dedup index. 50B URLs, each a 128-bit hash in a Bloom filter. A Bloom filter with 50B elements at 1% false positive rate needs ~10 bits per element = 500 Gbit = ~60GB total. Tiny. The whole filter sits in memory across a small cluster.

Concurrent host slots. Per-host rate limit = 1 req/sec, so one host delivers 1 page/sec. We need 58K pages/sec, so at least 58K hosts being crawled in parallel at any moment. With 500M hosts in the frontier, hosts are not scarce. The constraint is: do not stack two simultaneous fetches against the same host.

Fetcher node count. One fetcher node sustains 500 concurrent connections, each connection averages a 1 second fetch (DNS + TCP + TLS + GET + body), so one node = 500 fetches/sec. 150K/sec peak means ~300 fetcher nodes. Add 50% headroom: ~450 nodes.

The point the math leaves you with: storage is large but tractable, bandwidth is large but linear. The real complexity is coordination between 450 nodes that must collectively respect 1 req/sec per host across 500M hosts, with dedup across 50 billion URLs.

</details>

## Step 3: BFS-like exploration

A crawler is a graph traversal of the web. URLs lead to other URLs through outlinks. Classical BFS visits one level at a time. In a real crawler you cannot do strict BFS because:

- The graph is infinite (calendars, parameter combinations, search result pages).
- Most pages are not equally important; you want to spend disk on the useful ones.
- Politeness forces you to interleave hosts, not chase a single host depth-first.

Before reading on, decide: what governs the order of URL visits in your crawler? What data structure holds pending URLs? How do you ensure you do not crawl the same URL twice?

<details>
<summary><b>Reveal: priority frontier, not pure BFS</b></summary>

The frontier is a priority queue, not a FIFO. Each URL gets a score based on:

- Estimated importance (PageRank, inlink count, or a learned signal).
- Freshness need (news every hour, static pages monthly).
- Discovery freshness (newly found URLs go in with a default priority).
- Penalties for traps (URLs from known-trap patterns get lowered or dropped).

URLs are popped in priority order, subject to politeness. The "BFS-like" description is loose: yes, we explore outlinks of fetched pages, but ordering is by priority, not by depth.

Dedup has two layers:

1. URL dedup. Canonicalize the URL (lowercase host, sort query parameters alphabetically, strip session tokens like `?sid=`, drop `#frag`, normalize trailing slashes). Hash the canonical form. Check the Bloom filter; if probably-new, do a confirming lookup in a sharded KV store. If new, add to the frontier and mark seen.
2. Content dedup. After fetching, hash the meaningful content (strip ads, headers, footers, then hash). If the content hash matches one already stored, we still record the URL but link it to the canonical content. This catches mirror sites and duplicate templates.

Depth vs breadth control. Each URL carries a depth-from-seed counter. Past depth ~20 you almost always stop, because past depth 20 you are inside a calendar widget or a session-token-paginated search result. We also rate-limit "how many URLs we accept from one host per day" so a single trap site cannot flood the frontier.

</details>

## Step 4: sketch the high-level architecture

Fill in the five `[ ? ]` placeholders. Think about where the URLs to crawl live, what fetches them, what extracts links, where raw HTML is stored, and what prevents duplicates.

```
                      seed URLs                  outlinks from fetched pages
                          │                                    │
                          ▼                                    │
                  ┌──────────────────┐                         │
                  │   [ ? ]          │ ◄───────────────────────┘
                  │ (holds URLs to   │
                  │  crawl, ordered) │
                  └────────┬─────────┘
                           │ pop next batch (respecting politeness)
                           ▼
                  ┌──────────────────┐
                  │   [ ? ]          │
                  │ (talks HTTP to   │
                  │  the open web)   │
                  └────────┬─────────┘
                           │ raw HTML
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
        ┌──────────────┐       ┌──────────────┐
        │   [ ? ]      │       │   [ ? ]      │
        │ (durable     │       │ (parses HTML │
        │  storage of  │       │  and emits   │
        │  the page)   │       │  new links)  │
        └──────────────┘       └──────┬───────┘
                                      │
                                      ▼
                              ┌──────────────┐
                              │   [ ? ]      │
                              │ (was this    │
                              │  URL seen?)  │
                              └──────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                      seed URLs                  outlinks from fetched pages
                          │                                    │
                          ▼                                    │
                  ┌──────────────────┐                         │
                  │   URL Frontier   │ ◄───────────────────────┘
                  │  (priority queue │
                  │   sharded by     │
                  │   host hash)     │
                  └────────┬─────────┘
                           │ pop next batch (per-host token bucket gates release)
                           ▼
                  ┌──────────────────┐
                  │  Fetcher Pool    │   ~450 nodes.
                  │  (HTTP/HTTPS,    │   Each node owns a slice of hosts.
                  │   DNS cache,     │   500 concurrent fetches per node.
                  │   robots.txt)    │   Honors crawl-delay header.
                  └────────┬─────────┘
                           │ raw HTML + headers + status
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
        ┌──────────────┐       ┌──────────────┐
        │ Content      │       │ Link         │
        │ Store        │       │ Extractor    │
        │ (S3/GCS by   │       │ (parse HTML, │
        │  content     │       │  canonicalize│
        │  hash; meta  │       │  outlinks,   │
        │  in KV)      │       │  filter)     │
        └──────────────┘       └──────┬───────┘
                                      │ candidate new URLs
                                      ▼
                              ┌──────────────┐
                              │ Dedup Index  │   Bloom filter + sharded KV.
                              │ (seen URLs   │   "Probably new?" then confirm
                              │  hash set)   │   "definitely new" then frontier.
                              └──────────────┘

  Side flows (not shown in detail):
    - robots.txt cache, per-host, refreshed every 24h.
    - DNS cache, per-host, refreshed on TTL.
    - Recrawl scheduler: a separate job that re-injects already-seen URLs
      back into the frontier on their refresh cadence.
    - Trap detector: scans patterns in the frontier; downgrades or drops
      suspicious branches.
```

Component responsibilities:

- URL Frontier. The brain. Priority queue per host. Sharded so all URLs for a given host land on the same shard, which makes politeness a single-node decision.
- Fetcher Pool. The dumb hands. Stateless workers that take a batch of URLs from the frontier, fetch them with appropriate headers, and emit results. DNS cache and robots.txt cache live near the fetcher to avoid hammering DNS and webmasters.
- Content Store. Raw HTML in object storage, addressed by content hash. Metadata (URL, fetched_at, status code, content hash, parent URL) in a sharded KV store keyed by URL hash.
- Link Extractor. Parses HTML, finds `<a href>` and equivalent link sources (sitemap-driven, canonical link rel), canonicalizes each URL, filters obvious junk (mailto:, javascript:, fragment-only), and emits candidate URLs to the dedup index.
- Dedup Index. Bloom filter first ("probably seen"), then a sharded KV confirms. Only confirmed-new URLs go back to the frontier.

</details>

## Step 5: politeness

The single feature most candidates skip and most interviewers grade on. The open web is run by people you have never met. If your crawler hammers their site, three things happen: their site goes down, your IP gets blocked, and they file a complaint with your abuse desk. Politeness is non-negotiable, and it is the constraint that shapes coordination.

Before reading: what are the politeness rules and where in the architecture do they live?

<details>
<summary><b>Reveal: politeness mechanisms</b></summary>

robots.txt.

- Before fetching any URL on a host, GET `https://host/robots.txt`. Cache the result for 24 hours per host.
- Parse the directives: `User-agent`, `Disallow`, `Allow`, `Crawl-delay`, `Sitemap`. Match against your crawler's user agent string.
- If robots.txt cannot be fetched (5xx, timeout) treat the host as "crawl with caution". Some crawlers default to allowed, some default to disallowed. The safe default is to skip until robots.txt is readable, which is what major crawlers do.
- A 404 on robots.txt means "no rules, fetch freely" by RFC 9309.
- The robots.txt cache lives next to the fetcher, not in the frontier. The frontier knows nothing about robots.txt; the fetcher refuses to fetch URLs that violate it and feeds back a "skipped: disallowed" event.

Per-host rate limit.

- Default: 1 request per second per host. Honor `Crawl-delay` if specified.
- Implemented with a token bucket per host. The bucket holds N tokens with refill rate R. To fetch a URL on host H, the fetcher consumes a token from H's bucket. If empty, the URL is requeued with a delay.
- Aggressive hosts (large news sites with explicit agreements) may be opted into higher rates. Configurable per host.
- The bucket lives on the frontier shard that owns the host, not on every fetcher. That is why we shard the frontier by host hash: it makes the bucket a single-node decision.

HTTP status backoff.

- `200`, `301`, `302`: success, follow normally.
- `404`: record, do not retry.
- `403`: record as disallowed, do not retry for 7 days.
- `429 Too Many Requests`: honor `Retry-After`, double the per-host crawl-delay, exponential backoff on repeat.
- `503 Service Unavailable`: same as 429.
- `5xx` repeated: exponential backoff with jitter; after N failures (say 5 over 24h) treat the host as down and recrawl much less frequently.
- Connection timeouts: penalize the host's "health score". If health drops below threshold, the host is parked in a slow-lane queue.

User agent.

- Identify yourself: `Mozilla/5.0 (compatible; MyCrawler/2.0; +https://example.com/crawler-info)`.
- The URL in the user agent leads to a page explaining your crawler and how to block it. This is a baseline expectation by webmasters; missing it is rude.

Bandwidth caps.

- Per-host: do not pull more than X MB per day from one host without explicit allowance. Stops you from accidentally pulling a multi-GB site mirror.
- Per-network: respect bandwidth limits across your own datacenter egress.

Why this matters for design. Politeness, taken seriously, means each host has state (rate-limit tokens, last-fetched timestamp, health score, robots.txt cache, crawl-delay). That state must be consistent across the crawler. If two fetchers think they can each fetch from `example.com` at the same instant, you have failed politeness. This is why the frontier shards by host: all decisions about a host happen on one node.

</details>

## Step 6: distributed coordination

You have ~450 fetcher nodes. They share a 50-billion-URL frontier. They must collectively crawl 5B pages/day without duplicating work and without violating per-host rates. How is the work divided?

Three serious options. For each, ask: what happens when a node dies? What happens when a host has an outsized fraction of URLs? How is per-host politeness enforced?

<details>
<summary><b>Reveal: hash-partition by host</b></summary>

Option A: every fetcher pulls from a single shared queue.

Pros: simple. Cons: every fetcher coordinates with every other fetcher about politeness ("are you about to fetch from example.com? then I won't"). At 450 nodes, this is N² gossip and it does not scale.

Option B: hash by URL.

Pros: even load. Cons: URLs from the same host land on different shards. Politeness now requires cross-shard coordination on every fetch. Same N² problem.

Option C: hash by host.

```
shard_id = hash(host) mod num_shards
```

Each frontier shard owns a slice of hosts. All URLs for `example.com` live on one shard. That shard:

- Holds the priority queue for those hosts' URLs.
- Holds the per-host token bucket and rate-limit state.
- Holds the robots.txt cache for those hosts.
- Holds the per-host health score.

When a fetcher needs work, it asks a specific shard for a batch of URLs that are safe to fetch right now (politeness satisfied). The shard picks N URLs across N different hosts that have available tokens, and hands them to the fetcher. The fetcher does the IO, returns the results, and the shard updates state.

This is the standard pattern. Almost every large crawler is structured this way.

Failure handling.

- If a frontier shard dies, all hosts on that shard are temporarily unavailable. Replicate the shard (Raft, or primary + warm standby). Cassandra/DynamoDB-style replication works; the consistency needs are not strict (we already tolerate a few duplicate fetches).
- If a fetcher dies mid-batch, the URLs it had reserved time-out and become available again. Use a lease pattern: fetcher gets URLs with a 60-second lease; if not acknowledged within the lease, they go back into the queue.

Hot host.

- One host has 50M URLs in the frontier. The shard owning that host is hot. Mitigation:
 - Cap the per-host pop rate: even if the bucket is full, never hand out more than K URLs per second for one host, regardless of how starved the queue is for that host.
 - Spill the host's URLs to secondary storage; only keep the top-priority subset hot in the queue.
 - For truly huge hosts (Wikipedia, GitHub), split by sub-host or by URL hash within the host onto multiple shards, but coordinate the token bucket separately. The rare exception.

Fetcher-to-shard assignment.

- Fetchers are not assigned to shards. Any fetcher can pull from any shard. The fetcher chooses which shard to pull from based on consumer-fairness (round-robin) or based on which shards have available work (push from shards).
- Keeps fetchers stateless and lets them auto-scale on CPU/network.

</details>

## Step 7: read the full solution

You have covered the four hard parts: priority frontier, content and URL dedup, politeness, and host-sharded coordination. The solution covers data models, recrawl scheduling, JavaScript rendering, trap detection, multi-region distribution, and the day-2 reliability problems.

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. Crawler trap. A site has a calendar widget that links to `?date=2026-05-23`, `?date=2026-05-24`, and so on for the next 10,000 years. Your crawler dutifully follows every link. How do you detect and stop this without hardcoding a list of trap sites?
2. Soft 404. A site returns HTTP 200 with a body that says "Page not found, sorry". You add it to your index. Later you discover the entire site is identical "not found" pages with different URLs. How do you detect this in the pipeline?
3. JavaScript-rendered content. A modern SPA returns an empty `<div id="root">` and loads everything via JS. The link extractor finds zero outlinks. What is the pipeline change to handle these pages?
4. Recrawl scheduling. A news site posts 100 articles per day. A dormant blog posts once a year. You want news pages refreshed within an hour and the blog refreshed monthly. How do you decide each URL's recrawl interval without hand-tuning per site?
5. Frontier persistence. A frontier shard's machine reboots. You have 5 billion URLs queued in that shard. How is the queue persisted across restarts without making every push a synchronous disk write?
6. Distributed Bloom filter coordination. The Bloom filter for "have we seen this URL" is 60GB. You shard it across 8 nodes. Two fetchers in different regions encounter the same new URL at the same time and both try to add it. How do you avoid double-adding to the frontier?
7. Two URLs serve identical content. `example.com/article/123` and `example.com/article/123?utm=email` are the same page. Both got fetched. How does the storage layer recognize and consolidate them, and what does the index see?
8. A new important domain is launched. It has 1 million internal pages and the world starts linking to it heavily. Your default per-host rate of 1 req/sec means you need ~12 days to crawl it. How do you accelerate ethically?
9. Spam farm. An adversary generates 10 million auto-generated low-value pages on cheap domains that interlink and link to a target site to manipulate ranking. How does the crawler avoid wasting capacity on them?
10. Geo-distributed targets. A French news site is hosted in France. Your fetchers are in us-east. Each fetch takes 200ms of round-trip. How do you cut the latency without spinning up a full crawler in every region?

## Related problems

- [Rate Limiter (004)](../004-rate-limiter/question.md). Per-host politeness is exactly a token-bucket rate limiter at huge cardinality. The data structures and edge cases transfer directly.
- [Distributed Cache (009)](../009-distributed-cache/question.md). The dedup index, robots.txt cache, and DNS cache all share the same sharded-cache patterns. The eviction and consistency trade-offs there are foundational here.
- [Typeahead Autocomplete (005)](../005-typeahead-autocomplete/question.md). Both this and typeahead have a batch pipeline that transforms raw input into a serving-side index. The pipeline shape (Kafka spine, stateless workers, periodic compaction) is the same.
