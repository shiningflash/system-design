## Solution: Design Typeahead / Autocomplete Search

### TL;DR

Autocomplete is a read-dominant, latency-bound, ranked-prefix-lookup problem. Every keystroke is a request. At Google scale that is roughly 2M req/sec at peak with a P99 budget of 100ms including network. There is no room for a database hop on the hot path.

The design is a sharded in-memory trie, with the top-10 ranked suggestions precomputed at every node, fronted by a CDN and a regional cache. Most requests never reach the trie service. The trie is rebuilt offline from query logs (Spark or MapReduce), with a delta layer patched in every few minutes for trending terms. Atomic snapshot swap keeps the service online during refreshes.

The interesting engineering is in the cache layering (95%+ of traffic must be absorbed before the trie), the ranking pipeline (frequency, recency, trending, CTR, personalization, safety), the hot-shard problem for popular first characters, and keeping the system useful for misspellings, locale variants, and personalized re-ranking without exploding into a per-user index.

### 1. Clarifying questions, recap

Covered in `question.md`. The single most important question is the latency budget per keystroke. Anything looser than 200ms admits a database-based design (Elasticsearch completion suggester, Postgres prefix index). Anything tighter than 150ms forces in-memory tries, sharding, and aggressive caching. Without that number you cannot decide the structure.

The second most important is personalization scope, because per-user data on the hot path roughly doubles the architecture.

### 2. Capacity, with the math

- 5B queries/day × 10 keystrokes/query = 50B autocomplete req/day = ~580K req/sec sustained, ~2M peak.
- 100M unique queries in the corpus. Average query length 20 chars. Trie has ~300M nodes after prefix sharing. At ~300 bytes/node (top-10 cache + character map) that is ~90GB for the trie.
- 95% of traffic comes from the top 10M queries (heavy Zipf). Hot prefix working set is a few million entries × ~500 bytes each, so ~5GB cache at the regional layer.
- Per-request payload ~600 bytes. Peak bandwidth ~1.2 GB/sec, mostly absorbed by the CDN.
- 5 years of query logs at ~50 bytes/query × 5B/day × 1825 days is roughly 450TB cold storage. S3 / HDFS; only the rolling 30 to 90 days are used by the trie builder.

The math forces three decisions:

- The trie cannot live on one machine. Shard by prefix.
- The database is off the hot path. Everything in the request loop is in memory.
- Cache hit rate dominates capacity planning. If CDN drops from 80% to 50% hit rate, back-end load doubles.

### 3. API

The hot endpoint:

```
GET /api/v1/suggest?q=<prefix>&locale=<lang>&limit=10
Accept: application/json
Cookie: user_session=...   # optional, drives personalization

Response (200):
{
  "prefix": "face",
  "suggestions": [
    { "query": "facebook",          "type": "history", "score": 0.93 },
    { "query": "facebook login",    "type": "global",  "score": 0.91 },
    { "query": "facebook marketplace", "type": "global", "score": 0.87 },
    ...
  ],
  "request_id": "req-abc-123",
  "ts": "2025-05-22T14:22:10.111Z"
}
```

A few load-bearing choices:

- GET, not POST. The CDN can cache GETs. POSTs defeat the entire edge layer.
- `limit` capped at 10 on the server. Larger values waste bandwidth; the UI rarely shows more.
- `locale` is required. Different tries per locale. Defaults from `Accept-Language` if not given.
- `type` field on each suggestion tells the client whether this came from history, global ranking, or a trending injection. Useful for UI badges ("Recent searches") and for client-side analytics.
- No personalization for anonymous users. With no session cookie, the response is identical for any user with the same prefix and locale. That is what makes the CDN useful: identical responses, cache hits.
- For personalized requests, set `Cache-Control: private, max-age=10`. The CDN must not share this response across users.

A separate admin endpoint for blacklists:

```
POST /admin/blacklist
Authorization: Bearer <admin_token>
{
  "locale": "en-US",
  "queries": ["bad query 1", "bad query 2"],
  "reason": "DMCA notice 2025-05-22",
  "ttl": "PT720H"   # 30 days
}
```

Effective within seconds. The Suggest API consults a small blacklist set on every response, filtering candidates before returning.

### 4. Data model

#### Trie node (in memory, on the Trie Service)

```
struct TrieNode {
    children: HashMap<char, TrieNode*>   // child characters to next node
    top_suggestions: [Suggestion; 10]    // precomputed top-10 for this prefix
    is_terminal: bool                    // is this prefix itself a full query?
}

struct Suggestion {
    query_id: u32          // 4 bytes; resolves to display string via dictionary
    score: f32             // 4 bytes
    flags: u8              // bitfield: safety, locale, type
}
```

A small sketch of the trie itself, for the prefix `fa`:

```
        (root)
          │
          f
        / │ \
       a  o  i
      /│  │   \
     c  …  …   …
     │
     e  ── top_suggestions: [facebook, facebook login, face id, ...]
     │
     b
     │
     o
     │
     o
     │
     k  (terminal: "facebook")
```

Every node carries its own precomputed top-10. A request for `fa` walks `f` then `a` and returns the list cached at the `a` node. No subtree traversal at read time.

Notes on the layout:

- `query_id`, not the full string. Every distinct query gets a 32-bit ID at build time; the trie node references IDs, not strings. A side table maps `query_id` to display string. Nodes stay uniform in size (~150 bytes raw versus ~300 bytes with string overhead) and that saves serious memory at 300M nodes.
- HashMap, not array. UTF-8 means the alphabet is huge. A fixed-size array per node (say 256 entries) wastes memory; most nodes have very few children.
- Top suggestions are precomputed. This is the central optimization. Without it, "f" would mean traversing millions of subtree nodes per request.
- No mutation at runtime. The trie is built offline; the live trie is read-only. New tries replace old via snapshot swap.

#### Suggestion store (query_id to display)

Just a flat array, one entry per `query_id`:

```
struct QueryRecord {
    display_string: String   // the original query text, properly cased
    locale: u8
    total_count: u64
    last_seen_ts: u32
    safety_flags: u8
}
```

For 100M queries × ~50 bytes/record = ~5GB. Loaded once per Trie Service instance.

#### Query log (input to the builder)

```
{
  "query": "facebook login",
  "ts": 1716383530,
  "user_id_hash": "abc123",     // hashed, never the raw user_id in cold storage
  "locale": "en-US",
  "session_id": "sess-789",
  "shown_suggestions": ["facebook", "facebook login"],
  "clicked_position": 1,         // 0-indexed; -1 if no autocomplete click
  "from_keystroke": true,        // whether the search came from typing or autocomplete pick
  "country": "US"
}
```

Stored in S3 in Parquet, partitioned by date and locale. 30 days hot (used by builder), 5 years cold (used for analytics).

### 5. The trie with cached top-N

#### Lookup (the hot path)

```python
def suggest(prefix: str, locale: str, k: int = 10) -> list[Suggestion]:
    root = trie_for(locale)
    node = root
    for ch in normalize(prefix):
        node = node.children.get(ch)
        if node is None:
            return []                        # no match for this prefix
    return node.top_suggestions[:k]          # constant-time
```

Latency: walk K characters (typical 5-15), each step is a hash lookup. Total 1-10 microseconds in a hot cache. The expensive part is not the walk; it is the network and serialization overhead.

What you do *not* do at request time: traverse the subtree below the matching node, score candidates, or query a database. All the work is at build time. Runtime is constant per request.

#### Build (offline, the cold path)

```python
def build_trie(queries: list[ScoredQuery]) -> TrieNode:
    root = TrieNode()
    # Step 1: insert every query, marking terminals.
    for q in queries:
        node = root
        for ch in q.normalized:
            node = node.children.setdefault(ch, TrieNode())
        node.is_terminal = True
        node.terminal_score = q.score
        node.query_id = q.id

    # Step 2: propagate top-N upward. Recursive post-order.
    def propagate(node):
        candidates = []
        if node.is_terminal:
            candidates.append((node.query_id, node.terminal_score))
        for child in node.children.values():
            propagate(child)
            candidates.extend(child.top_suggestions)
        # take top 10 by score
        node.top_suggestions = heapq.nlargest(10, candidates, key=lambda s: s.score)
    propagate(root)

    return root
```

A few practical notes:

- Propagation is O(N × K) where N is nodes and K is the top-N size. Heap-merging child lists is the inner loop.
- For a 90GB trie this runs in roughly 30 minutes on a 100-node Spark cluster.
- The build never happens on the Trie Service. It runs in a separate batch cluster. The Trie Service only loads finished snapshots.

#### Why "top-N at every node" is the right trade

The alternative is "store one trie, traverse subtree at read time." Trie is smaller (~10GB), but read latency is O(subtree size), which for "y" is hundreds of millions of nodes. Unacceptable.

The reverse: "store a hash map from every prefix to top-10." O(1) lookup, but no prefix sharing, so memory grows to ~150GB and updates are harder because a new query affects all of its prefixes independently.

Top-N at every node hits the sweet spot. Reads in microseconds, memory at 90GB, builds in 30 minutes.

### 6. Architecture

Here is the whole picture. Three regions shown for shape; the same idea repeats per region.

```
                                          End user (keystroke)
                                                 │
                                                 ▼
                                       ┌──────────────────────┐
                                       │   CDN / Edge Cache   │  CloudFront, Fastly, Cloudflare.
                                       │                      │  TTL 60-600s. Hits ~80%
                                       │                      │  of unauthenticated traffic.
                                       └──────────┬───────────┘
                                                  │ on miss
                                                  ▼
                                       ┌──────────────────────┐
                                       │  Global Load         │  Anycast. Routes to nearest
                                       │  Balancer            │  healthy region.
                                       └──────────┬───────────┘
                                                  │
                ┌─────────────────────────────────┼─────────────────────────────────┐
                │                                 │                                 │
                ▼                                 ▼                                 ▼
        ┌───────────────┐                  ┌───────────────┐                ┌───────────────┐
        │  Region:      │                  │  Region:      │                │  Region:      │
        │  us-east-1    │                  │  eu-west-1    │                │  ap-south-1   │
        │               │                  │               │                │               │
        │ ┌───────────┐ │                  │ ┌───────────┐ │                │ ┌───────────┐ │
        │ │ Suggest   │ │                  │ │ Suggest   │ │                │ │ Suggest   │ │
        │ │ API       │ │                  │ │ API       │ │                │ │ API       │ │
        │ │ (N pods)  │ │                  │ │ (N pods)  │ │                │ │ (N pods)  │ │
        │ └─┬────┬────┘ │                  │ └─┬────┬────┘ │                │ └─┬────┬────┘ │
        │   │    │      │                  │   │    │      │                │   │    │      │
        │   ▼    ▼      │                  │   ▼    ▼      │                │   ▼    ▼      │
        │ ┌────┐┌────┐  │                  │ ┌────┐┌────┐  │                │ ┌────┐┌────┐  │
        │ │Regn││Pers│  │                  │ │Regn││Pers│  │                │ │Regn││Pers│  │
        │ │ache││nzns│  │                  │ │ache││nzns│  │                │ │ache││nzns│  │
        │ └─┬──┘└────┘  │                  │ └─┬──┘└────┘  │                │ └─┬──┘└────┘  │
        │   │           │                  │   │           │                │   │           │
        │   ▼           │                  │   ▼           │                │   ▼           │
        │ ┌──────────┐  │                  │ ┌──────────┐  │                │ ┌──────────┐  │
        │ │ Trie     │  │                  │ │ Trie     │  │                │ │ Trie     │  │
        │ │ Service  │  │                  │ │ Service  │  │                │ │ Service  │  │
        │ │ (shards) │  │                  │ │ (shards) │  │                │ │ (shards) │  │
        │ └──────────┘  │                  │ └──────────┘  │                │ └──────────┘  │
        └───────┬───────┘                  └───────┬───────┘                └───────┬───────┘
                │                                  │                                │
                │ all regions pull trie snapshots from object storage                │
                └──────────────────────────────────┴────────────────────────────────┘
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │  Object Storage      │  S3 / GCS.
                                       │  (trie snapshots,    │  Versioned trie files
                                       │   delta files)       │  per locale per shard.
                                       └──────────┬───────────┘
                                                  ▲
                                                  │
                                       ┌──────────────────────┐
                                       │  Trie Builder        │  Spark/MapReduce.
                                       │  (batch pipeline)    │  Daily full rebuild.
                                       │                      │  15-min delta updates.
                                       └──────────┬───────────┘
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │  Query Logs          │  S3 / HDFS / Kafka.
                                       │  (Parquet,           │  All historical queries
                                       │   partitioned)       │  with click info.
                                       └──────────────────────┘
```

What each piece does:

- CDN: caches the GET response keyed on `(prefix, locale)`. Personalized responses bypass the CDN via `Cache-Control: private`. The CDN is the cheapest tier; every byte served here is a byte not served by the origin.
- Suggest API: stateless. Calls regional cache; on miss calls Trie Service. After getting the global top-N, optionally calls the Personalization service to re-rank. Adds telemetry, returns.
- Regional cache (Redis cluster): catches popular prefixes that miss the CDN (because CDN entries just expired, or the prefix is hot in one region but not globally). Replaces ~15% of trie load.
- Personalization store: per-user list of most-issued queries with their counts. Used by Suggest API to mix into the response. Keyed by `user_id_hash`.
- Trie Service: the actual prefix lookup. Sharded by first character (or first two characters for finer granularity). Each shard holds a slice of the trie for one or more characters. Replicated 3x for HA and read throughput.
- Trie Builder: offline. Daily full rebuild. Reads logs, computes scores, builds tries, writes snapshots to object storage with version tags.
- Delta Builder: a faster pipeline (every 5-15 minutes) for trending queries that need to appear sooner than the next full rebuild.
- Object Storage: holds snapshots. Trie Service instances poll for new versions and swap atomically.

### 7. The read path (the keystroke flow)

For a user typing "f", "fa", "fac", "face":

1. Keystroke 1, "f". Client issues `GET /suggest?q=f&locale=en-US`.
2. CDN. Almost certainly cached; "f" is one of the most popular prefixes. Returns the top-10 in ~5ms (mostly network from edge to user).
3. Keystroke 2, "fa". Same flow; also very likely cached.
4. Keystroke 3, "fac". May or may not be in CDN. On miss, continues to origin.
5. CDN miss. Request hits the regional Suggest API.
6. Suggest API checks the regional cache. Lookup `prefix:fac:en-US` in Redis. Common prefixes are hot in Redis even if the CDN cooled. ~2ms.
7. Regional cache miss. Suggest API calls the appropriate Trie Service shard. For prefix "fac" the shard owns the `f` bucket. ~3-5ms cross-AZ.
8. Trie Service lookup. Walk 3 nodes, return the cached top-10. ~50 microseconds local CPU.
9. Back at the Suggest API. If the user is authenticated, fetch their personalization data (a small list of frequent queries). Compute a re-ranking: any of their history that starts with "fac" gets boosted, the rest stays in order. ~5ms.
10. Cache the result. Write to regional Redis with a short TTL (30-60s). For anonymous responses, also send back to the CDN with `Cache-Control: public, max-age=300`.
11. Return to client. Total origin-side ~15-25ms. Client-perceived including network ~50-80ms.

P99 latency target 100ms end-to-end. P50 ~30ms.

Personalized request path. Same flow up to step 7. Step 9 does more work:

- Fetch user's frequent queries (`user:history:{user_id_hash}` in Redis or the Personalization service): ~5ms.
- Compute boost score per candidate: trivial.
- Re-order top 20, return top 10.
- Mark response `Cache-Control: private, max-age=10` so the CDN does not share it.

For users who issue 5-10 queries per session, this hits a per-user cache and stays fast.

### 8. Index building pipeline

#### Daily full rebuild

```
00:00 UTC: snapshot of query_logs for the trailing 90 days.
00:10 UTC: Spark job starts. Inputs: ~450B query records.
            
            Stage 1: filter + normalize. Drop bots (UA filtering, volume signature),
                    PII patterns (credit card numbers, SSN-like strings),
                    queries < 2 chars or > 100 chars. Normalize.
                    Output: ~400B cleaned rows.
            
            Stage 2: aggregate. groupBy(normalized_query, locale).
                    Compute total count, recency-weighted count (exp decay tau=30d),
                    autocomplete CTR, trending velocity (vs 30-day moving average).
                    Output: ~100M unique (query, locale) rows.
            
            Stage 3: score. Apply ranking formula. Tag with safety flags
                    (run each query through the safety classifier).
                    Output: ~100M scored rows.
            
            Stage 4: build trie per locale. Group by locale, then for each
                    locale group, build trie + propagate top-N.
                    Output: ~30 locale tries, ~3GB each average.
            
            Stage 5: shard. For each locale trie, split by first character.
                    Each shard becomes a separate file.
                    Output: ~30 locales × 30 shards = ~900 snapshot files.
            
            Stage 6: write to object storage with version tag
                    `trie/v=2025-05-22/locale=en-US/shard=f.trie`.
            
00:45 UTC: build done. Total wall time ~35 min on a 200-executor cluster.

00:45 UTC: Trie Service instances begin polling. Each instance
            downloads its assigned shard for the new version.
00:50 UTC: download complete (each shard ~3GB; <5 min on internal network).
00:51 UTC: validate snapshot. Check size sanity, spot-check
            ~1000 sample prefixes against the previous version.
            If validation fails, alert and do not swap.
00:52 UTC: atomic pointer swap. Old trie freed after a grace period.
            Memory usage briefly doubles during swap.
```

#### Delta updates for trending

A celebrity dies at 2:00 PM. By 2:15 PM their name is the dominant query but the trie was built at 00:45 UTC. Without a delta path, suggestions are stale for up to a day.

The delta pipeline:

```
Every 5 min:
  - Read the last hour of query logs from Kafka (not from S3).
  - Aggregate, score with trending velocity weighted heavily.
  - For each (locale, prefix) that has a candidate not in the current top-10,
    emit a delta entry: (prefix, locale, query_id, score_override).
  - Write delta file to object storage.
  - Trie Service instances pull the delta and merge it at lookup time:
        top_10 = merge(static_trie_node.top_suggestions,
                       delta_layer[prefix][locale])[:10]
```

The delta layer is small (a few thousand entries at any moment). Lives in memory alongside the trie. Lookup cost adds ~1 microsecond.

After the next daily rebuild incorporates the trending queries naturally, delta entries expire.

#### Emergency blacklist

For legal takedowns or moderation:

```
POST /admin/blacklist
```

Writes a row to a `blacklists` table (small, ~10k rows total). Suggest API instances reload the blacklist every 30 seconds. Filters candidates before returning. Effective globally within ~60 seconds.

The blacklist is per-locale because legal requirements differ by country. The Suggest API selects the union of "global blacklist" plus "locale-specific blacklist" for filtering.

### 9. Scaling

#### a. Trie sharding

- Shard by first character. 26 letters + digits + a handful of common UTF-8 prefixes is around 50 shards for English. For multilingual, shard by `(locale, first character)`.
- Each shard is a separate process on a dedicated machine.
- Replicate each shard 3x for HA and to multiply read throughput.
- Total: ~150 Trie Service instances for English-only; ~4500 for 30 locales (in practice locales share machines because they are tiny next to English).

Sizing per instance: ~3GB shard + ~5GB query dictionary + ~2GB OS/JVM overhead = ~10GB RAM. Use 16GB machines.

Routing: the Suggest API knows the shard map (a tiny config file: `{locale: {first_char: shard_id}}`). On request, hash the prefix to find the shard.

#### b. Cache tiering

The cumulative cache hit math:

```
CDN hit rate:        80%   (absorbs 1.6M req/s of 2M peak)
Regional Redis hit:  15%   (absorbs 0.3M req/s)
Trie Service load:    5%   (~100K req/s across ~150 shards = 700 req/s/shard)
```

700 req/s per shard is trivial. A 16GB machine running a hot in-memory trie handles tens of thousands of req/s.

The CDN is the single biggest investment. If CDN hit rate drops 10 percentage points (to 70%), origin load doubles. Watch CDN hit rate aggressively; it is the leading indicator of capacity stress.

#### c. Hot prefix problem

The shard owning "y" handles maybe 20% of all suggest traffic ("y" leads to youtube, yahoo, and so on, which are massively popular first keystrokes). At 2M peak × 20% = 400K req/s on that one shard.

Mitigations:

1. Aggressive CDN caching for single-character prefixes. "y" alone never changes much; cache it for 10 minutes at the edge. Biggest win; 400K req/s at origin becomes maybe 5K.
2. More replicas for hot shards. Instead of 3 replicas of the "y" shard, run 10.
3. In-process caching on the Suggest API. A 1000-entry LRU per pod, keyed on `(prefix, locale)`, 10-second TTL. Absorbs flash spikes.
4. Finer-grained sharding for hot first characters. Instead of "y" being one shard, shard it by second character: "ya", "ye", "yi", "yo", "yu". Ten shards share the load.

Most production systems use all four, in that order.

#### d. Memory pressure during snapshot swap

When a new trie loads, both old and new are in memory briefly. 3GB × 2 = 6GB per shard during swap. Mitigations:

- Stagger swaps across replicas. One replica at a time goes into "swap mode" and is removed from the load balancer briefly.
- Memory-map the trie file so old pages can be reclaimed naturally rather than held in RSS.

#### e. Multi-region

Each region runs its own Trie Service, Suggest API, regional cache. All regions pull from the same object storage. Snapshot swap is coordinated loosely: each region swaps when it has finished downloading; they may be a few minutes apart.

Cross-region replication of query logs: queries from eu-west are written locally to S3, then replicated to a central bucket for the global builder. The builder is one job, not per-region, because we want one consistent global ranking.

For data sovereignty (GDPR, Schrems II): EU query logs may stay in EU regions for the builder. Run a separate EU-only build pipeline, or anonymize before cross-region transfer.

### 10. Reliability

- Trie Service shard failure. Three replicas; load balancer drops the failed one, traffic shifts. Hot shards have 10 replicas for extra margin. Worst case the prefix served by that shard returns 503 for a few seconds until rerouting; clients display no suggestions and the search box still works.
- Regional cache (Redis) failure. Reads fall through to Trie Service. Load spikes 5x on the trie shards (cache used to absorb 15% of post-CDN traffic, now 0%). Trie shards have headroom and should survive. Alert at 80% shard CPU.
- CDN failure (rare but real). Origin load multiplies 5x. Will exceed capacity in most setups. Mitigation: capacity planning assumes CDN at 50% hit rate, not 80%. Also serve a degraded response (cached top-100 most popular queries) at the API layer as a last resort.
- Snapshot swap failure (bad build). Validation step refuses the swap. Service continues serving the previous trie. Page the on-call.
- Object storage outage. Existing tries continue serving from RAM. New snapshots cannot be pulled, so the trie ages. After ~24 hours suggestions become stale. Acceptable for short outages.
- Trie Builder failure. The most recent valid trie keeps serving. After a few days of stale data, trending suggestions get noticeably stale. Alert if the builder has not produced a snapshot in 26 hours.
- Personalization service failure. Suggest API skips the personalization step and returns the global top-10. Quality drops slightly for logged-in users; everyone else is unaffected. Degradation is invisible.

### 11. Observability

| Metric | Why |
|--------|-----|
| `suggest.latency_p99` per region | Headline SLO. Target <100ms. |
| `suggest.cdn_hit_rate` | Leading indicator of capacity. Alert below 70%. |
| `suggest.regional_cache_hit_rate` | If this drops the trie shards take a hit. |
| `trie.shard_qps` per shard | Spot hot shards before they melt. |
| `trie.shard_cpu` per shard | Cardinal CPU signal. |
| `trie.snapshot_age` per shard | Should be <26 hours. |
| `trie.snapshot_swap_duration` | Spikes indicate IO problems on the shard host. |
| `delta.lag` | Time from query trending in logs to appearing in delta. Target <15 min. |
| `builder.run_duration` | Daily build should finish under 60 min. |
| `builder.input_row_count` | Sudden drop indicates broken log pipeline. |
| `safety.blacklisted_query_serve_count` | Should always be 0. Non-zero means a blacklist leak. |
| `personalization.fetch_p99` | If this slows, the suggest path slows. |
| `cdn.response_size_p99` | Sudden growth indicates a payload bug. |

Alerts:

- Page on: `suggest.latency_p99 > 200ms` for 5 min in any region; `trie.snapshot_age > 26h`; `safety.blacklisted_query_serve_count > 0`.
- Ticket on: CDN hit rate drop, delta lag spike, build duration drift.

A specific dashboard I always want: "hot shard heatmap." A grid of all shards colored by current QPS. Lets the on-call see at a glance whether load is balanced or one shard is on fire.

### 12. Follow-up answers

**1. Typos.**

The user types "facbook" instead of "facebook". Plain trie traversal returns no match because "facb" has no children.

Approaches:

- BK-tree (Burkhard-Keller tree). A separate structure indexed by Levenshtein distance. Given a misspelled prefix, find candidate corrections within edit distance 1 or 2, then look up suggestions for each candidate.
- Symspell or fuzzy hash. Precompute all single-deletion variants of every common query at index time. Lookup runs against this expanded dictionary. Fast at query time but bloats the index 4-5x.
- N-gram index. Index 3-grams of every query. To find candidates for "facbook" find queries that share many 3-grams, score by overlap, pick best matches.
- Two-pass. Try a clean trie lookup. If 0 results, run the spell-correction service to suggest a corrected prefix, then re-lookup. Adds latency on misses only.

I would put a small inline BK-tree at the Suggest API, fired only when the trie returns 0 results (or fewer than 3). The BK-tree is keyed only on the top ~1M most popular queries, so it stays small. Edit distance 2 is the cap.

Cost: ~5ms per fallback lookup. Frequency: ~3% of requests. Net latency impact: negligible.

**2. Trending queries (5-minute freshness).**

The delta pipeline (Section 8) is the answer. Read the last hour of query logs from Kafka every 5 minutes, score by trending velocity, emit a delta file consumed by Trie Service instances.

A faster path is possible: skip the file and push deltas directly via a pub/sub channel. Each Trie Service shard subscribes and updates its in-memory delta layer. Latency from log to suggestion drops to ~2 minutes. Cost is more complexity (every shard maintains a small mutable structure alongside the static trie).

The hard part is not the pipeline. It is deciding what counts as trending. The minimum threshold has to be high enough that you do not promote spam (a single bot sending 1000 queries should not move ranking) but low enough that real news events get picked up quickly. Typical formula: trending velocity > 10x baseline AND total volume > 100 in the last 5 minutes AND originated from > 10 distinct user_id_hashes.

**3. Hot prefix.**

Covered in Section 9c. The stack:

1. CDN caches "y", "a", "f" and friends for ~10 minutes. Absorbs the bulk.
2. More replicas for hot shards (10 instead of 3).
3. In-process LRU on Suggest API for the top ~1000 prefixes.
4. Finer-grained sharding of hot first characters (`y` to `ya`, `ye`, `yi`, `yo`, `yu`).

The senior answer also mentions: monitor the heatmap, autoscale the hot shards on QPS, and pre-warm the CDN for known-popular prefixes after a snapshot swap (otherwise the swap briefly invalidates edge caches and load spikes at origin).

**4. Personalization without a per-user trie.**

A per-user trie would be ~3GB × number of users. Impossible.

Instead, store per-user a small list of their most-issued queries (say their top 100, with counts). Keyed by `user_id_hash`, stored in Redis (or a per-user feature store).

At request time:

1. Fetch the user's top-100 list. ~2ms.
2. Filter to those starting with the current prefix. Usually 0-5 hits.
3. Get the global top-10 from the trie.
4. Merge: user-history matches get a strong boost in score; reorder; take top 10.

Net effect: the user's frequent queries float to the top of their suggestions. New queries (the global top-10) still appear. The trie itself is untouched.

For users who issue many queries, this works well. For new users with no history, the result is identical to anonymous. Graceful degradation.

The per-user list size controls the memory: 100 entries × ~50 bytes = 5KB per user. 1B users × 5KB = 5TB. Sharded Redis or a dedicated feature store handles this fine.

**5. Profanity (removing within 5 minutes).**

Hateful content appears for prefix "j". The trie has it in the top-10 because, at build time, the safety classifier missed it.

Immediate action:

- Admin posts to `/admin/blacklist` with the offending query (and the locale).
- Suggest API reloads blacklists every 30s; within a minute it filters the entry from responses.
- CDN entries are still serving the bad suggestion. Issue a purge for the affected prefix on the CDN. ~30-90 seconds to clear globally.

Total time to clean: 2-3 minutes.

Preventing it from coming back:

- The blacklist entry persists across rebuilds. The Trie Builder reads the blacklist file and demotes or drops blacklisted queries during the score step.
- The classifier gets retrained on the new example. Add the missed query to the safety training set. The next classifier deploy catches similar variants.

A residual problem: the blacklist is keyed by exact normalized string. Synonyms and obfuscations (zero-substitution like "h@te") need separate entries. A robust system uses a small set of regex patterns and a classifier on top of the blacklist set, not just a literal match.

**6. Multilingual queries.**

A French user types `b`. They probably want French suggestions ("bonjour", "billet") but might also want English ("breaking news") if they often search in English.

Two layers:

- Primary trie selection. Default to the user's UI locale. French user means `locale=fr-FR` trie. Their `b` returns French suggestions.
- Cross-locale mixin. For users with multilingual history (detected from their past queries), mix in the top 1-2 results from a secondary locale. The mix ratio is data-driven; A/B test it.

For anonymous users use `Accept-Language` header. If two languages tie, pick the more popular one in the user's IP country.

A subtle issue: some queries are global and identical across locales (`facebook`, `youtube`, brand names). They should appear in all locale tries. Handle by tagging certain query_ids as "global" during build; every locale trie includes them.

**7. New popular query not yet in the trie.**

Delta pipeline (Section 8). 5-15 minute latency from a query becoming popular to it appearing as a suggestion. Without the delta, you would wait for the next full rebuild (up to 24 hours).

If even 5 minutes is too slow: streaming Flink job that updates the delta in real time. Sub-minute latency. Costs more compute and more complexity. Reserve it for trending domains where freshness is critical (news search, e-commerce).

**8. Cold start for a new locale.**

You launch in Vietnamese. No query logs.

Bootstrapping options:

- Seed from Wikipedia titles. Vietnamese Wikipedia article titles ranked by pageview. Loaded as initial top-suggestions. Covers ~10M entities; not perfect but reasonable.
- Seed from translation of the English top-N. Translate the top 100K English queries into Vietnamese; use them as starter content. Quality is uneven; brand names should remain in original form.
- Seed from product catalogs. If this is an e-commerce search, use product names as seed queries.
- Aggressive trend detection. As real queries trickle in, the delta pipeline rapidly promotes them. Within a week or two, real query data dominates.

I would combine all of them: bootstrap with Wikipedia plus a curated brand list plus translated top queries, then let real logs take over. Quality monitoring: human reviewers spot-check the top-10 for popular prefixes during the first month.

**9. Privacy and deletion.**

Personalization uses past queries. Risks:

- Leaking one user's queries into another user's suggestions. Per-user data is keyed by hashed user_id and never aggregated across users for personalization. Cross-user signals (global rank, trending) come from the trie, not from individual histories.
- Building the global trie from logs that contain user identifiers. At log ingestion, hash user_ids with a rotating salt. The builder operates on hashed IDs and aggregates volume only; the trie itself has no user-identifying data.
- "Delete my history" request:
 - Delete all rows in `user:history:{user_id_hash}` immediately.
 - Mark the user_id_hash as "deleted" in the query log retention table; scrub their rows in the next log compaction cycle.
 - Their contribution to global trends is small (one user out of billions). We do not retroactively rebuild the trie to exclude their contributions because their identity is already pseudonymous in the logs and excluding their entries would not materially change the trie.
 - Confirm to the user within the regulatory window.

For minors and sensitive accounts: never store personalization data at all. The user setting controls this.

**10. Trie Service shard for `f` runs hot and one replica falls over.**

Blast radius:

- All prefixes starting with `f` are served by this shard. Roughly 5-8% of all suggest traffic.
- One replica down means the other 2-9 replicas (depending on replication factor) absorb its load. CPU jumps proportionally.
- If the shard was already running hot (say 80% CPU), losing a replica may push the rest to 100%. P99 latency degrades. Some requests time out.

Degradation cascade if not contained:

- Suggest API times out, returns empty suggestions for `f` prefixes, user-visible degradation (empty box).
- Regional cache eviction policy notices empty responses, may not cache them; load continues hitting the trie.
- Eventually, the autoscaler spins up new shard replicas. Time to fully recover: 2-5 minutes.

Mitigations in order of preference:

1. Always run hot shards at <50% CPU baseline. Headroom for failures.
2. Autoscale on shard CPU. Add replicas before they melt.
3. Aggressive in-process LRU on Suggest API. During shard distress, serve from local cache even if slightly stale. Returns a usable response.
4. Circuit breaker. Suggest API detects shard timeouts > 1% over 30s, opens the circuit, returns a fallback (top-100 cached "always popular" queries for the locale). User sees something, even if not ideal.
5. Pre-warm replacement replicas after snapshot swap. Reduces the time-to-serve of a newly-spawned replica.

The senior answer addresses all five. The mid-level answer mentions only "add more replicas."

### 13. Trade-offs worth saying out loud

- Why not Elasticsearch's completion suggester. It works, has decent latency (~20ms), and you skip building a custom trie. The downside is per-shard memory (ES's FSTs are smaller than our top-N tries but) worse tail latency due to the JVM and a broader feature set. For startup-scale, ES is the right answer. For Google-scale, you build your own.
- Why not a real-time trie that updates per query. Mutating a tree concurrently with readers is hard. Lock contention kills throughput. The community standard is "atomic snapshot swap" precisely because it sidesteps this problem. The cost is the freshness lag, which the delta layer handles.
- Why not precompute personalized tries. Per-user tries are far too expensive at billions of users. Mixing personalization on top of a global trie at request time costs 5ms and produces good-enough quality.
- Why a CDN at all. Some teams argue against the CDN because "responses are personalized." Split it. Anonymous and short-prefix responses cache at the CDN (60-80% of traffic). Personalized responses bypass it. The CDN is not all-or-nothing.
- Why not vector search for autocomplete. Vector search (embed the prefix, find nearest queries) handles typos and semantic matches better than a trie. The cost is latency (each lookup is ~10-30ms) and recall on exact prefixes (a vector for "fac" may not return "facebook" as top-1). For now, trie plus spell correction plus BK-tree is faster and more predictable. Vector search shows up as a secondary candidate generator at the next architectural level.
- What I would revisit at 10x scale:
 - GPU-served trie shards for batched lookups (each request is tiny; batching at the shard level cuts CPU).
 - Edge tries: ship a tiny LRU trie (top ~10k queries) directly to mobile clients. First two keystrokes resolved on-device with zero network. Server is consulted only for keystroke 3+.
 - Federated personalization: keep per-user history on-device; mix on the client. Eliminates server-side personalization entirely. Major privacy win, somewhat reduced personalization quality.

### 14. Common interview mistakes

- "Just use a trie." Without acknowledging memory cost, the top-N caching trick, the offline rebuild pipeline, or the cache layering, this is a junior answer.
- Forgetting the CDN. At 2M req/sec, every byte that does not get cached at the edge costs you. Mention the CDN early.
- Not computing the trie size. The 90GB number is what justifies sharding. Without it the candidate sounds like they have not thought about scale.
- Storing full strings at every node. Use query_ids and a side dictionary; saves 50%+ of memory.
- Ignoring ranking. Alphabetical suggestions or pure frequency are wrong. Recency, CTR, trending, personalization, safety all matter.
- Live writes to the trie. Snapshot swap is the standard. Mutating a live trie is a bug magnet.
- No mention of typos. Real users misspell things constantly. A BK-tree or spell-correction fallback is expected at staff level.
- No mention of safety or profanity. Real product concern. Mention it unprompted.
- No mention of personalization mixing strategy. The "boost user history on top of global rank" trick is a standard answer; not mentioning it suggests you have not thought about logged-in users.
- Underweighting cache hit rate as the capacity driver. At 95% cache hit rate the back-end load is 100K req/s, easy. At 80% it is 400K req/s, painful. The whole capacity plan rests on the CDN.

If you hit 8 of these 10 head-on, you are interviewing at senior or staff level. Most candidates get the trie and the cache, miss the ranking depth and the build pipeline, and either skip personalization entirely or invent a per-user trie they then cannot defend.
