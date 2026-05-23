---
id: 5
title: Design Typeahead / Autocomplete Search
category: Search
topics: [trie, prefix tree, ranking, caching, low-latency]
difficulty: Medium
solution: solution.md
---

## Scene

The interviewer this round used to work on the Google search box team. They open a blank doc and type one line:

> *Design the autocomplete that shows up under the Google search box when a user is typing. Walk me through it end-to-end.*

Then they lean back. The problem looks small ("just suggest some queries") but has three brutal constraints stacked together: every keystroke needs a response in under 100ms globally, suggestions must be ranked by something smarter than alphabetical, and the index has to rebuild from query logs without taking the service down. Miss any one of those and the box feels broken. Candidates who jump straight to "use a trie" miss that the trie is the easy part.

## Step 1: clarify before you design

Take 5 minutes. Do not draw anything yet. Write down at least six questions that would meaningfully change the design if answered differently.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. Latency budget. "What is the per-keystroke P99 we are targeting?" Google's bar is roughly 100ms end-to-end including network. If the answer is 300ms you have a very different design (you can afford a round-trip to a regional service); if it is 50ms you need the result at the edge.
2. Scale of queries. "How many search queries per day across the corpus? How many keystrokes per second at peak?" A typical answer: 5B queries/day, 10 keystrokes per typed query, so 50B autocomplete requests/day. That works out to roughly 580K req/sec sustained, 2M peak.
3. Language and locale scope. "English only, or all locales? Do we share a single index or one per locale?" Per-locale changes ranking inputs and storage by a factor of 50 or so. Also matters for tokenization (CJK languages have no spaces).
4. Personalization. "Should suggestions be personalized to the user, or globally ranked?" Personalization adds a per-user data path, a feature store, and roughly doubles the architecture. The usual answer is "globally ranked, with light personalization for logged-in users."
5. Suggestion freshness. "If a new query becomes trending in the last hour, must it appear in suggestions today?" Hourly is easy. Sub-minute is hard and adds a streaming pipeline.
6. Typo tolerance. "If the user types 'gogle' do we suggest 'google'?" Adds either edit-distance lookup (BK-tree) or a separate spell-correction layer.
7. Profanity, safety, legal. "What about hateful, sexual, or libelous suggestions? Per-country blocklists?" Real engineering work, not a checkbox.
8. Read versus write ratio. "How often do we update the index?" Almost always read-dominant. Index rebuilds are batch jobs that run nightly or hourly.

If you only asked about QPS and not about latency budget, you missed the constraint that drives the whole architecture. The latency budget is what forces an in-memory trie and a tiered cache; without it you could just hit a database.

</details>

## Step 2: capacity estimates

Inputs from the interviewer:

- 5B search queries per day
- Average 10 keystrokes per typed query, so 50B autocomplete requests/day
- 100M unique queries in the corpus (the long tail; most are very rare)
- Suggestions per response: 10
- Per-keystroke P99 latency: 100ms globally
- The top 10M queries account for ~95% of the volume (heavy Zipf)

Work it out on paper before revealing:

1. Autocomplete requests per second (sustained and peak)
2. Bytes per request and response
3. Storage required for the trie if you keep the top 10 suggestions cached at every prefix node, for a corpus of 100M queries
4. Cache size required to absorb 95% of traffic
5. Where the bottleneck is: CPU, memory, network, or disk

<details>
<summary><b>Reveal: the math</b></summary>

Requests per second. 50B / 86400 is about 580K req/sec sustained. Peak around 3x, so ~2M req/sec peak. That is an order of magnitude larger than the URL shortener problem and tells you immediately that caching is not optional.

Bytes per request and response. Request is HTTP headers plus the prefix string, call it 200 bytes per keystroke. Response is 10 suggestions at ~30 bytes each plus framing, ~400 bytes. Per request total ~600 bytes. Bandwidth: 2M × 600 = 1.2 GB/sec peak. Not trivial; CDN absorbs most of it.

Trie storage. Average query length 20 characters. Trie nodes total roughly the number of distinct prefixes. For 100M queries averaging 20 chars, with heavy prefix overlap, distinct nodes land around 200M to 400M. Per node: 10 suggestions × (16-byte string ref + 4-byte score) = 200 bytes minimum, plus a character map (typically a small hash, say 100 bytes more). Round to 300 bytes per node.

300 bytes × 300M nodes is roughly 90GB for the trie. Does not fit in a single machine's RAM comfortably, but it shards beautifully by first 1 or 2 characters.

Cache size. Top 10M queries × ~25 prefixes per query (every prefix that resolves to them) is around 250M unique cache keys. In practice the working set is much smaller because users type the same popular prefixes ("face", "you", "amaz"). At 95% hit rate the hot working set is a few million prefixes. At ~500 bytes per cache entry (10 serialized suggestions): ~5GB. Fits in one Redis node, replicated.

Bottleneck. Not disk (everything is in memory). Not network on a per-request basis (600 bytes is small). The bottleneck is CPU for trie traversal at the unsharded scale, and memory for the trie itself. Both scream "shard the trie across machines, cache aggressively, terminate at the edge."

The decisive insight: at 2M req/sec peak, every millisecond of per-request work matters. You cannot afford a database hop on the hot path. Everything must be in memory and most of it must be in a CDN.

</details>

## Step 3: data structure

You need a structure that maps a prefix string to a ranked list of completions. Three serious options. Take 10 minutes to write the pros, cons, and when each wins.

<details>
<summary><b>Reveal: comparison and recommended choice</b></summary>

| Structure | How it works | Lookup cost | Memory | Best for |
|-----------|--------------|-------------|--------|----------|
| Trie (prefix tree) with top-N cached at each node | Walk the trie character by character. At the matching node, return its precomputed top-10 list. | O(K) where K is prefix length. Constant after that. | High: every prefix node holds 10 suggestions. ~90GB for 100M queries. | Heavy read, willing to spend memory, want sub-millisecond lookups. |
| Hash table mapping prefix to top-10 list | One hash lookup per request. | O(1) average. | Even higher than the trie because there is no prefix sharing; "facebo" and "faceboo" each store their own list. ~150GB for 100M queries. | Want simplicity, willing to spend more memory. |
| Prefix-indexed database (Elasticsearch completion suggester, Postgres `LIKE 'prefix%'` with index) | Database lookup per keystroke. | 5-50ms depending on the engine. | Modest: ~10GB for the same corpus, since you store queries once and search them. | When latency budget is loose (>=200ms) and corpus is huge. |

I go with a trie with top-N suggestions cached at every node.

Why:

- The trie wins on read latency. Walking 20 characters and returning a cached list is ~10 microseconds in memory. A hash table is ~1 microsecond but loses prefix iteration. A database is at least 5ms, which blows the 100ms budget before you start.
- Top-N at every node is the trick. A naive trie would, on every request, traverse the subtree below the matching node, score every leaf, and return the top 10. For "f" that is millions of nodes. Precomputing the top-10 at every node turns this into a constant-time read.
- Memory is the cost. ~90GB total. Shard across ~30 machines, ~3GB each. Each machine fits in commodity RAM.
- The trie is rebuilt offline. Live updates are tricky and rare; we rebuild from query logs daily (or hourly) and do an atomic swap.

A hash table is a fine answer for smaller corpora (millions of queries, not 100M). Saying "trie" without acknowledging the memory cost is the weak version of this answer.

One more subtlety. "Top-N at every node" trades read latency for build time. Rebuilding the trie from 100M queries with top-N propagated at every node is a real MapReduce/Spark job, on the order of 30 minutes to an hour. Fine if the refresh cadence is hourly or slower.

</details>

## Step 4: sketch the high-level architecture

Here is an intentionally incomplete diagram. Fill in the five `[ ? ]` boxes. Think about what sits at the edge to absorb cache hits, what serves the actual prefix lookup, what stores the trie, and what produces it from logs.

```
        User typing in the search box
                    │
                    │ one request per keystroke
                    ▼
            ┌───────────────┐
            │   [ ? ]       │  (caches popular prefix responses
            │               │   close to the user)
            └───────┬───────┘
                    │ on miss
                    ▼
            ┌───────────────┐
            │  Suggest      │  (stateless, regional)
            │  API          │
            └──────┬────────┘
                   │
                   ▼
            ┌───────────────┐
            │   [ ? ]       │  (returns top-10 for a given prefix
            │               │   in microseconds)
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │   [ ? ]       │  (the in-memory trie, sharded)
            └───────────────┘

                                  Built offline:
                                  ┌──────────────────┐
                                  │   [ ? ]          │  (query logs from
                                  │                  │   search service)
                                  └────────┬─────────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │   [ ? ]          │  (batch job: aggregate,
                                  │                  │   rank, propagate top-N,
                                  │                  │   produce a trie snapshot)
                                  └──────────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
        User typing in the search box
                    │
                    │ one request per keystroke
                    ▼
            ┌───────────────────────┐
            │    CDN (edge cache)   │  Caches `GET /suggest?q=<prefix>`
            │  (CloudFront / Fastly │  for the most popular prefixes.
            │   / Cloudflare)       │  TTL: 1-10 min. Absorbs most traffic.
            └──────────┬────────────┘
                       │ on miss
                       ▼
            ┌───────────────────────┐
            │  Suggest API Service  │  Stateless, regional.
            │  (HTTP/gRPC handler)  │  Auth, locale routing,
            │                       │  personalization injection.
            └──────────┬────────────┘
                       │
                       ▼
            ┌───────────────────────┐
            │  Regional cache       │  Redis or in-process LRU.
            │  (Redis cluster)      │  Caches prefix to top-10 list
            │                       │  for warm-but-not-hot prefixes.
            └──────────┬────────────┘
                       │ on miss
                       ▼
            ┌───────────────────────────────────┐
            │  Trie Service (sharded)           │  In-memory trie shards.
            │  Sharded by first 1-2 characters  │  Each shard holds part of
            │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ │  the trie. Replicated
            │  │a-c  │ │d-f  │ │g-i  │ │...  │ │  ~3x for HA and read
            │  └─────┘ └─────┘ └─────┘ └─────┘ │  throughput.
            └───────────────────────────────────┘
                       ▲
                       │ atomic snapshot swap
                       │
            ┌───────────────────────┐
            │  Trie Builder         │  Spark/MapReduce/Flink batch job.
            │  (offline pipeline)   │  Reads logs, aggregates, ranks,
            │                       │  propagates top-N, emits trie
            │                       │  snapshot to object storage.
            └──────────┬────────────┘
                       │ reads
                       ▼
            ┌───────────────────────┐
            │  Query log store      │  All historical search queries
            │  (S3 / HDFS / Kafka)  │  with timestamp, user_id (if any),
            │                       │  locale, result-click info.
            └───────────────────────┘
```

What each piece does:

- CDN: first line of defense. For a prefix like `y` or `face` the response is identical for every user (unless personalized) and changes slowly. Edge-cache it. Hit rate target ~80%.
- Suggest API Service: stateless front door. Auth, locale routing, rate limiting, optional personalization mixin. Stores nothing.
- Regional cache: catches prefixes too rare for the CDN but popular enough to be hot. Lives close to the Trie Service. Hit rate target ~15% of total traffic, so cumulative ~95% cache hit before the trie sees anything.
- Trie Service: holds the actual trie in memory, sharded by prefix bucket. Each shard replicated for read throughput and HA. The remaining ~5% of traffic lands here.
- Trie Builder: offline. Reads days or weeks of query logs, aggregates counts, applies ranking, propagates top-N to every node, writes a serialized trie snapshot to object storage. Trie Service shards pull the new snapshot and atomically swap.
- Query log store: append-only. Every search query recorded here (with appropriate privacy filtering). Input to the builder.

</details>

## Step 5: ranking

You have 10 suggestions to show for any given prefix. There are probably hundreds of candidate queries that match. How do you pick which 10? What signals matter, and how do you combine them?

<details>
<summary><b>Reveal: ranking signals and how they combine</b></summary>

The ranking question is "given that thousands of queries start with `face`, which 10 should we surface?"

Signals, in rough order of importance:

1. Historical frequency. How many times has this query been issued, total. The baseline.
2. Recency-weighted frequency. Recent occurrences count more. Use exponential decay: a query issued today counts 10x a query issued 90 days ago. This is what makes "facebook layoffs" rank above "facebook ipo" today even though the latter has more lifetime volume.
3. Click-through rate from autocomplete. When this query was suggested, how often did the user pick it? High CTR means the suggestion is good. Low CTR means it is technically frequent but not what people want.
4. Trending velocity. Sudden spike in volume in the last hour. Detects breaking news and viral queries. Implemented as `(current_hour_count - moving_average) / moving_average`, capped to prevent dominance.
5. Personalization. For logged-in users: queries this user has issued before, queries from their geographic cohort, queries in their language.
6. Locale and language match. A French speaker on a French keyboard should not see English-only queries near the top.
7. Safety penalty. Profanity, hate, sexual content, defamation: drop entirely or demote sharply. Per-country rules.

Combine them with a weighted linear blend for the baseline rank, then re-rank the top 50 candidates with an ML model that takes the same signals plus user-state features. Two-stage retrieval, like a feed.

```
score(query, prefix, user, time) = 
      w1 * log(historical_frequency(query))
    + w2 * recency_weighted_frequency(query, time)
    + w3 * autocomplete_ctr(query, prefix)
    + w4 * trending_velocity(query, time)
    + w5 * personalization_match(query, user)
    - w6 * safety_penalty(query)
```

Weights are tuned by holdout evaluation on click-through. Personalization is added at the API layer as a re-rank, not baked into the trie (the trie is global; personalization differs per user).

The trie stores the top-N by global ranking. The Suggest API may re-order this list by personalization before returning, but only re-orders; it does not introduce new candidates. Introducing new candidates would need a per-user trie, which is impractical.

</details>

## Step 6: building the index

The trie has to be built somehow. From what? On what cadence? What happens when you swap a new one in?

<details>
<summary><b>Reveal: build pipeline</b></summary>

Input. All search queries from the last N days (typically 30 to 90). Stored in S3 or HDFS, one row per query: `(query_string, ts, user_id_hash, locale, was_clicked, click_position)`.

Pipeline (Spark or MapReduce):

1. Filter and clean. Drop queries with PII patterns. Drop queries shorter than 2 chars or longer than 100. Drop bot traffic identified by user-agent or volume signature. Lowercase, normalize whitespace, strip diacritics for the matching key (keep the display form separately).
2. Aggregate. Group by `(normalized_query, locale)`, compute total count, recency-weighted count, autocomplete CTR, trending velocity. One big group-by; standard Spark job.
3. Score. Apply the ranking formula. Each (query, locale) gets a global score.
4. Apply safety filters. Run each query through the profanity/safety classifier. Drop or demote.
5. Build trie. For each locale, build a trie. For every node, walk the subtree below it, collect all queries, take the top 10 by score, store at the node. This is the expensive step. Done with a recursive bottom-up propagation: each node's top-10 is the merge of its children's top-10 lists, picking the best 10 by score. O(N) in total nodes.
6. Serialize. One binary trie file per locale per shard, written to object storage with a version tag.
7. Snapshot swap. Trie Service shards poll object storage for new versions. Each shard downloads the new snapshot in the background, validates it, and atomically swaps the live pointer. A read in flight either sees the old trie or the new one, never a partial state.

Cadence:

- Daily full rebuild is the baseline. Catches new queries, updates weights, applies safety classifier updates.
- For trending: a faster path. Every 5-15 minutes, a lightweight job processes the last hour of logs, identifies queries with high trending velocity that are not in the current top-10 at their prefix, and patches them into a delta layer. The Trie Service consults the delta layer in addition to the static trie. Avoids a full rebuild for every minor change.
- For emergency (legal takedown of a suggestion): direct API to blacklist a (locale, query) pair. Suggest API filters this list before returning. Effective in seconds.

Validation before swap:

- Compare top-10 for ~1000 sample prefixes against the previous snapshot. If more than 50% of suggestions changed, refuse the swap and alert. Catches builder bugs.
- Compare size. A 90GB trie suddenly becoming 30GB is a sign of pipeline failure.
- Compare presence of known-popular queries. "weather", "facebook", "youtube" should always rank in the top 10 of their prefixes. Spot check.

</details>

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. Typos. A user types "facbook" instead of "facebook". Should you suggest "facebook"? How do you find candidates that are *close* to a misspelled prefix without scanning the whole trie?
2. Trending queries. A celebrity dies at 2:00 PM. By 2:05 PM many users are searching their name. How quickly can your suggestions reflect this, and what is the minimum-cost way to make it faster?
3. Hot prefix. Everyone types "y" first. A single Trie Service shard (the one owning the `y` bucket) handles disproportionate traffic. What do you do?
4. Personalization without a per-user trie. A logged-in user previously searched "deep learning" three times. When they now type "d", "deep learning" should rank near the top. How do you do this without storing a per-user trie?
5. Profanity and offensive suggestions. Your trie suggests something hateful for the prefix "j". You did not catch it in the safety pass. How do you remove it within 5 minutes globally, and how do you prevent it from coming back on the next rebuild?
6. Multilingual queries. A French user types `b`. Should they see English queries that happen to be popular globally, or only French queries? What if they sometimes search in English?
7. A new query becomes popular but is not yet in the trie. Until the next rebuild, it will not appear as a suggestion. Is there a way to inject it sooner without rebuilding the whole trie?
8. Cold start for a new locale. You launch in Vietnamese. There are no query logs yet. How do you bootstrap the suggestions?
9. Privacy. Personalization requires using a user's past queries. How do you do this without leaking one user's queries into another user's suggestions, and how do you handle a "delete my history" request?
10. The Trie Service shard for `f` runs hot and one replica falls over. What is the blast radius and how does the system degrade?

## Related problems

- [URL Shortener (001)](../001-url-shortener/question.md): same heavy reliance on tiered caching (CDN + Redis + in-memory) for a read-heavy workload. Understand the cache layering before tackling typeahead.
- [Distributed Cache (009)](../009-distributed-cache/question.md): the regional cache and the in-process LRU rely on the same eviction, replication, and hot-key mitigations.
- [Web Crawler (008)](../008-web-crawler/question.md): both have an offline batch pipeline (MapReduce/Spark) that produces a serving index on a regular cadence. The build-and-swap pattern is the same.
- [News Feed (002)](../002-news-feed/question.md): both use two-stage retrieval (cheap candidate generation followed by a slower re-rank). The ranking placement decision is structurally identical.
