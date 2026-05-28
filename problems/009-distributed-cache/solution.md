## Solution: Distributed Cache (like Redis or Memcached)

### The short version

A distributed cache is a fast in-memory key-value store split across many servers. Data is sharded (each server holds a slice of the keyspace) and replicated (each shard has a live backup). The service sits in front of a database: a cache hit costs under 1ms, a miss falls through to the database at 10-50ms and writes the result back so the next request is a hit.

At 1M ops/sec with a 1 TB working set, a single Redis server is not enough. You need a cluster: 12-24 shards, each with a replica, coordinated by a gossip protocol that tracks health and slot ownership. Client libraries cache the slot map locally and connect directly to the right shard with no proxy hop.

The hard problems are not scale. They are: the hot key that breaks one shard while others idle, the big key that blocks the single-threaded event loop, the stampede that follows a popular key expiring, and the resharding that must happen without dropping traffic.

---

### 1. The two questions that matter most

**Is this a pure cache or the only copy of the data?** A pure cache can lose data on restart. If Redis holds data that lives nowhere else, you need persistence and `noeviction`. These are two completely different systems that happen to share a name.

**How strict is freshness?** If the app reads from replicas and can tolerate a few milliseconds of stale data, you get 2-3x read throughput per shard at almost no cost. If reads must see the latest write, every read goes to the primary and you lose that benefit.

Everything else (eviction policy, replication mode, persistence tier) follows from these two.

---

### 2. The math, in plain numbers

| Item | Value |
|------|-------|
| Sustained ops/sec | 1M |
| Peak ops/sec | 3M |
| Read-to-write ratio | 10:1 |
| Working set | 1 TB |
| Average value size | 1 KB |
| Single-node Redis ceiling | ~150K ops/sec |
| Shards needed at peak (with headroom) | 24 |
| With replication factor 2: total servers | 48 |
| Data per primary | ~85 GB |
| With Redis encoding overhead (20%) | ~200 GB per server |
| Network per server | ~110 MB/sec |

What this tells us:

- The bottleneck is per-shard CPU (single event-loop thread), not memory or network.
- A 256 GB server fits comfortably with headroom.
- Connection state (50K client connections at 20 KB each) adds ~1 GB per server. Budget this separately.

---

### 3. The API

Two commands carry almost every use case.

| Command | Semantics |
|---------|-----------|
| `GET key` | Return value or nil. O(1). |
| `SET key value [EX seconds] [NX\|XX]` | Write with optional TTL. `NX` = only if missing. `XX` = only if exists. O(1). |
| `MGET k1 k2 ...` | Batch get. All keys must hash to the same slot, or the client fans out. |
| `DEL key` | Delete. Returns 1 if existed. |
| `INCR key` | Atomic integer increment. Foundation for counters and rate limiters. |
| `SCAN cursor [MATCH pattern]` | Cursored iteration. Never use `KEYS *` in production. |

Three cluster-specific responses every client must handle:

| Response | Meaning | Client action |
|----------|---------|---------------|
| `MOVED <slot> <host:port>` | Slot permanently moved to another server. | Update slot map, retry on new server. |
| `ASK <slot> <host:port>` | Slot mid-migration; this key is already on the new server. | Send `ASKING` then the command to the new server. Do not update slot map yet. |
| `READONLY` error | Tried to write to a replica. | Reroute write to the primary. |

---

### 4. The data model and memory layout

The conceptual model is a flat hash table: keys to values. The interesting engineering is how Redis encodes values internally to save memory.

**Object header.** Every Redis value has a small header: type tag, encoding, reference count, and a 24-bit last-access timestamp for LRU/LFU sampling. About 16 bytes per object.

**Compact encodings.** Redis picks an encoding based on size:

| Type | Small encoding | Switches to | Switch point |
|------|---------------|-------------|--------------|
| Short string (<= 44 bytes) | `embstr` (one allocation) | `raw` | 44 bytes |
| Small integer | `int` (no allocation) | `embstr` | Non-integer |
| Hash (small) | `listpack` (flat array) | `hashtable` | 128 entries or any field > 64 bytes |
| Sorted set (small) | `listpack` | `skiplist + hashtable` | 128 entries or any element > 64 bytes |

A 10-field hash as `listpack` takes ~200 bytes. The same hash as `hashtable` takes ~600 bytes. At 100M small hashes, staying in `listpack` saves 40 GB.

**Expiration metadata.** Keys with TTL have an entry in a separate dictionary mapping key to expiration timestamp. Active expiration samples from this dictionary, not the whole keyspace.

<details markdown="1">
<summary><b>Show: Memcached slab allocator and Redis replication buffer sizing</b></summary>

**Memcached's slab allocator.** Memcached uses fixed-size slabs (64 B, 128 B, 256 B, ...). A 100-byte value goes into a 128-byte slab. Memory-efficient when value sizes cluster tightly. Wasteful when they vary widely. Once slabs are allocated to a size class, they cannot easily be reassigned, so traffic shifting from small to large values leaves small slabs idle.

**Replication buffer sizing.** The primary keeps a circular ring buffer of recent writes (default 1 MB in Redis, set to 256 MB or more in production). When a replica reconnects after a brief outage, it requests a partial resync from its last known offset. If that offset is still in the buffer, only the missed writes are sent (fast). If the buffer has wrapped, a full RDB snapshot is sent (slow, can take minutes for a large dataset). Size the buffer so brief network blips do not trigger full resyncs.

</details>

---

### 5. The engine: hash slots and replication

**Slot routing.**

```
slot = CRC16(key) & 16383
server = slot_map[slot]
```

The slot map is replicated across all servers via gossip and cached locally by clients. Clients compute the slot before sending the request and connect directly to the right server.

**Hash tags.** `{user:42}:profile` and `{user:42}:settings` both hash on `user:42` (the text inside `{}`). Both land on the same slot. Multi-key operations stay on one shard.

**Replication protocol.** Async by default. The primary writes to memory, replies `+OK`, and queues the write for replication. Replicas pull and apply. Lag at steady state: under 1ms intra-AZ. For the few writes that must not be lost, call `WAIT <N> <timeout_ms>` to block until N replicas ack.

**Failover.** Gossip detects a dead primary in ~5-8 seconds (majority of primaries must agree). The replica with the smallest replication lag broadcasts a vote request. Primaries vote yes if they also see the old primary as dead and have not voted in this epoch. A majority vote promotes the winner. Total failover: 5-15 seconds.

---

### 6. The architecture

```mermaid
flowchart TB
    subgraph AppTier["Application tier"]
        A1["App 1<br/>smart client + local LRU"]:::user
        A2["App 2<br/>smart client + local LRU"]:::user
        AN["App N<br/>smart client + local LRU"]:::user
    end

    subgraph Cluster["Cache cluster (3 of 12 shards shown)"]
        S1P["Shard 1<br/>Primary<br/>slots 0-1364"]:::cache
        S1R["Shard 1<br/>Replica"]:::cache
        S2P["Shard 2<br/>Primary<br/>slots 1365-2729"]:::cache
        S2R["Shard 2<br/>Replica"]:::cache
        S3P["Shard 3<br/>Primary<br/>slots 2730-4094"]:::cache
        S3R["Shard 3<br/>Replica"]:::cache
        Bus{{"Cluster Bus<br/>gossip port"}}:::queue
        S1P -.async repl.-> S1R
        S2P -.async repl.-> S2R
        S3P -.async repl.-> S3R
        S1P --- Bus
        S2P --- Bus
        S3P --- Bus
        S1R --- Bus
        S2R --- Bus
        S3R --- Bus
    end

    DB[("Postgres<br/>source of truth")]:::db

    A1 -->|"slot 0-1364"| S1P
    A1 -->|"slot 1365-2729"| S2P
    A2 -->|"slot 2730-4094"| S3P
    AN --> S1P
    S1P -.miss.-> DB
    S2P -.miss.-> DB
    S3P -.miss.-> DB

    classDef user fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef db fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

Five things to notice:

- Each app server holds a complete slot map and connects directly to the right shard. No proxy, no extra hop.
- Replicas are async. Failover loses the writes in the replication buffer at the moment the primary died, typically a handful of operations.
- The cluster bus runs on a separate TCP port (data port + 10000 in Redis). Gossip traffic does not compete with client traffic.
- Persistence is optional. Most caches do not persist. Some use RDB snapshots to seed a warm replica quickly after restart.
- The database is the source of truth. If the whole cache cluster dies, the app falls through to the database. The database must be sized to handle cold-cache load.

---

### 7. A request, end to end

```mermaid
sequenceDiagram
    autonumber
    participant App
    participant LRU as Local LRU
    participant SDK as Smart Client
    participant S2P as Shard 2 Primary
    participant DB as Postgres

    App->>LRU: GET user:42:profile
    alt local hit (microseconds)
        LRU-->>App: value
    else local miss
        LRU-->>App: miss
        App->>SDK: GET user:42:profile
        SDK->>SDK: slot = CRC16(key) & 16383 = 5798

        rect rgb(241, 245, 249)
            Note over SDK,S2P: direct TCP to shard, no proxy hop
            SDK->>S2P: GET user:42:profile
            alt cache hit (~90%)
                S2P-->>SDK: value (~0.5ms)
                SDK-->>App: value
                App->>LRU: populate local LRU
            else cache miss (~10%)
                S2P-->>SDK: nil
                SDK-->>App: nil
                App->>DB: SELECT * FROM users WHERE id = 42
                DB-->>App: row (~20ms)
                App->>S2P: SET user:42:profile <value> EX 300
                S2P-->>App: OK
                App->>LRU: populate local LRU
            end
        end
    end
```

P99 for a local LRU hit: microseconds. P99 for a Redis hit: ~0.5ms. P99 for a cache miss that falls through to Postgres: ~20ms. The bottleneck on the Redis path is the single-threaded event loop, not network.

---

### 8. The scaling journey: 10 users to millions

```mermaid
flowchart LR
    S1["Stage 1<br/>startup<br/>1 Redis, no cluster<br/>~$50/mo"]:::s1
    S2["Stage 2<br/>growth<br/>3 shards + replicas<br/>cluster mode on<br/>~$500/mo"]:::s2
    S3["Stage 3<br/>scale<br/>12 shards, 2 replicas each<br/>local LRU on app servers<br/>~$5K/mo"]:::s3
    S4["Stage 4<br/>enterprise<br/>per-service clusters<br/>tiered L1/L2/L3 cache"]:::s4
    S1 --> S2 --> S3 --> S4

    classDef s1 fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef s2 fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef s3 fill:#fef3c7,stroke:#a16207,color:#713f12
    classDef s4 fill:#fce7f3,stroke:#be185d,color:#831843
```

#### Stage 1: startup (10-100 users)

One Redis server. No cluster mode, no replicas, no persistence (or RDB every 15 minutes). Cache-aside in the application. If Redis restarts, the cache is cold until traffic rewarms it. Ship in a day.

#### Stage 2: growth (~10K users)

What breaks: one Redis server can no longer hold the working set, or it is pegged at CPU.

What to add: enable Redis Cluster, start with 3 shards (6 servers including replicas). Set `maxmemory` and `allkeys-lru`. Alert on `used_memory > 80% of maxmemory`. Traffic stays alive during resharding if the client handles `MOVED`/`ASK`.

#### Stage 3: scale (100K-1M users)

What breaks: hot keys appear. A few keys dominate traffic. One shard shows 2x the ops/sec of others.

What to add, in order:

1. In-process LRU on every app server (1,000 entries, 5-second TTL). The hottest 1% of keys stop reaching the cluster entirely.
2. Replica reads for read-heavy paths. `READONLY` connections to replicas effectively double or triple per-shard read throughput.
3. Grow to 12 shards if per-shard ops/sec still exceeds 150K. Roll shards one at a time with no downtime.

#### Stage 4: enterprise (millions of users)

Split one monolithic cluster into per-service clusters. The news feed cluster, session cluster, and rate-limit cluster each have their own sizing, eviction policy, and on-call rotation. Blast radius shrinks.

For extreme hot keys: key splitting across N slots. For the slowest-warming caches: RDB persistence so restarts do not trigger a long cold-start period.

---

### 9. The five hard problems

#### Hot key (500K req/s on one key)

Symptoms: that shard's CPU at 100%, others idle. P99 for all keys on that shard climbs.

Mitigations in order of cheapness:

1. In-process LRU. A 1,000-entry LRU with a 5-second TTL absorbs over 99% of traffic for the top keys. The single biggest win.
2. Replica reads. With 2 replicas, triple the read throughput for that key. Accept slightly stale values.
3. Key splitting. Store the value under `hot_key:0` through `hot_key:7`, each on a different slot. Client picks a random suffix per read. Every write must update all copies. Use only for extreme cases.
4. Request coalescing on app servers. If many goroutines need the same expired key, only one fetches it; the others wait on a local lock.

#### Big key (sorted set with 2M entries)

`ZRANGE 0 -1` on 2M entries blocks the single-threaded event loop for ~800ms. Every other key on that shard stalls.

Prevention: cap at write time. Reject writes that would push a set past 10K entries. Track key sizes with `MEMORY USAGE key`. Alert on outliers.

Recovery once the key exists: use `UNLINK` (not `DEL`). `DEL` on a 2M-element set is itself a multi-second blocking call. `UNLINK` moves deallocation to a background thread. To break the key apart, iterate with `ZSCAN` in batches of 100, write chunks to new keys, then `UNLINK` the original.

#### TTL stampede

1M keys all given `EX 60` during a bulk import. At t=60s, active expiration fires on all of them simultaneously. CPU spikes. The application's cache-miss regeneration path hammers the database at the same time.

Fix: add jitter at write time. Use `EX (60 + random(0, 30))`. Spreads expirations over 30 seconds: 1M keys / 30s = 33K expirations per second. Active expiration handles this comfortably.

#### Cache stampede

A popular key expires. 10,000 concurrent requests miss and hit the database. The database melts.

Standard four-layer defense:

1. **Request coalescing.** Per-key in-process lock. First request fetches and writes the cache; concurrent requests wait. Reduces 10K DB calls to 1 per app server.
2. **Stale-while-revalidate.** Serve the expired value for a short grace window (2-5 seconds) while one goroutine fetches the new value asynchronously.
3. **Probabilistic early expiration (XFetch).** A small fraction of reads near the TTL boundary trigger an early refresh. One client refreshes while others serve the still-valid value.
4. **Jittered TTLs (prevention).** Stagger writes so keys do not expire simultaneously.

#### Resharding (6 to 12 shards)

1. Bring up 6 new primaries + 6 replicas. Empty. Introduce them to the cluster with `CLUSTER MEET`.
2. Use `redis-cli --cluster rebalance` to plan slot migrations. Each of the 6 old shards gives half its slots to a new shard.
3. Each slot migration is atomic: keys move one by one. Source responds `ASK` for in-flight keys. After migration, source responds `MOVED` and clients update their slot map.
4. Throttle to ~5% CPU overhead on source and destination.

Traffic stays live throughout. Application sees tail latency spike, not errors, if the client handles `ASK`/`MOVED`. For 1 TB total at 50 MB/sec migration rate per pair: ~30 minutes. All 6 migration pairs run in parallel.

---

### 10. Reliability

**Crash mid-write.** The primary writes to memory and replies `+OK` before replication. If it crashes a millisecond later, that write is lost unless AOF is enabled. For a pure cache, this is acceptable. For stateful data, use AOF with `everysec`.

**Replica falls behind.** If the replication backlog overflows, the replica needs a full RDB resync (entire dataset shipped over the network). Set the replication buffer to 256 MB minimum in production, 1 GB for active clusters. Default 1 MB is too small.

**Cluster-wide reboot.** Without persistence, every shard comes back empty. The database absorbs the full working set as cache misses. Size the database to handle this, or enable RDB persistence to restore a warm snapshot.

**Persistence options:**

| Mechanism | Latency cost | Data loss window |
|-----------|-------------|-----------------|
| None | Zero | Everything on restart |
| RDB snapshot (every 5 min) | Brief CPU spike during fork | Up to 5 minutes |
| AOF (`everysec`) | ~1ms per write | Up to 1 second |
| AOF (`always`) | ~5ms per write | Near zero |

For pure cache: no persistence. For slow-to-warm caches: RDB every 5 minutes. For stateful Redis (counters, rate limiters, session tokens): AOF with `everysec` plus replicas.

---

### 11. Observability

| Metric | Why it matters | Alert threshold |
|--------|---------------|-----------------|
| `ops_per_sec` per shard | Detects hot shards | Shard at >2x cluster median |
| `latency_p99` per shard | Slow shard (big key, CPU pressure) | >10ms |
| `cache_hit_rate` | Cache effectiveness | Drop >10 percentage points |
| `used_memory` vs `maxmemory` | Eviction headroom | >80% |
| `evicted_keys_per_sec` | Eviction pressure | Spike from baseline |
| `replication_lag_sec` | Replica freshness | >1 second sustained |
| `mem_fragmentation_ratio` | Allocator waste (rss/used) | >1.5 |
| `slowlog_count` | Commands past 10ms threshold | Any sustained appearance |
| `cluster_state` | Cluster health | Anything other than "ok" |

Page on: `cluster_state != ok`, replication lag > 5s, P99 latency > 20ms, ops/sec drop > 50%.

Ticket on: hit rate drop > 10 points, eviction spike, fragmentation > 1.8.

---

### 12. Follow-up answers

**1. Hot key (500K req/s).**

First, add an in-process LRU on each app server (1,000 entries, 5s TTL). At 1,000 app servers, 500K req/s becomes 500 req/s reaching the cluster. That is the cheapest fix. If that is not enough, enable `READONLY` replica reads for that key. If still not enough, split the key into N variants and have the client pick a random one per request.

**2. Big key.**

Do not delete with `DEL`. On a 2M-entry sorted set, `DEL` itself blocks for seconds. Use `UNLINK`, which moves deallocation to a background thread. To break the key apart, read with `ZSCAN` in batches of 100, write chunks to new keys, then `UNLINK` the original. Prevention: cap at write time. Reject writes that would push a collection past 10K entries.

**3. TTL stampede.**

At t=60s, active expiration tries to delete all 1M expired keys at once, spiking CPU. Simultaneously, application cache-miss handlers all fire and hit the database. Fix: add jitter at write time. Instead of `EX 60`, use `EX (60 + random(0, 30))`. Expirations spread over a 30-second window.

**4. Persistence choice.**

Use nothing for a pure cache where the database can handle cold-start load. Use RDB snapshots (every 5-15 minutes) when the cache takes hours to warm, so a restart does not cause a long recovery period. Use AOF with `everysec` when Redis holds stateful data (counters, session tokens). Use AOF with `always` only for the rare keys that need near-zero loss, and accept ~10x lower write throughput for those.

**5. Resharding (6 to 12).**

Bring up 6 new primaries + 6 replicas. Use `redis-cli --cluster rebalance` to plan and execute slot migrations. Each slot migration is atomic: keys move one by one, `ASK` redirects serve in-flight requests, `MOVED` follows after full migration. Throttle to ~5% CPU overhead. Total time: ~30 minutes for a 1 TB cluster at 50 MB/sec. Zero downtime if the client library handles redirects correctly.

**6. Network partition (two servers isolated).**

The isolated pair cannot reach a majority, so they refuse to promote themselves. If they are primaries, they eventually stop accepting writes (if `min-replicas-to-write` is set) or continue accepting writes that will be discarded on partition heal (if not). When the partition heals, they gossip in, see their epochs are stale, and demote themselves. No split-brain: majority is required for all decisions.

**7. Memory fragmentation ratio 1.8.**

`rss / used = 1.8` means the OS sees Redis using 1.8x what Redis itself tracks. The 80% gap is allocator fragmentation: the allocator holds free chunks it cannot return to the OS because small allocations are scattered between larger ones. Causes: mixed value sizes, high churn of TTL-bearing keys. Fixes: enable `activedefrag yes` (~5% CPU background defragmenter), or plan a rolling restart (let the replica take over, restart the former primary fresh). Investigate above 1.5. Act above 2.0.

**8. Cache stampede.**

Two defenses that work together. Request coalescing: per-key mutex on each app server so only one goroutine fetches the new value while others wait. Stale-while-revalidate: serve the just-expired value for 2-5 seconds while one goroutine refreshes asynchronously. For prevention, add jitter to TTLs. For high-stakes cases, implement probabilistic early expiration (XFetch): occasionally refresh a key a few seconds before it expires so expiry is never a surprise.

**9. Read-after-write.**

The write went to the primary. The read was routed to a replica that had not yet received the replication update. Replication lag is usually under 1ms intra-AZ but can spike under load or during a resync. Fix: route reads to the primary for code paths that require read-after-write consistency. Or use `WAIT 1 100` after the write to block until at least one replica has acked before reading from that replica. Accept eventual consistency everywhere else.

**10. Hit rate crash (95% to 60%).**

Investigate in this order:

1. `evicted_keys_per_sec` spiking? Working set grew or `maxmemory` shrank. Common cause: new feature writing more keys, or a new service sharing the same cluster.
2. Did any TTLs shorten? Cutting from 300s to 60s collapses hit rate on long-tail keys.
3. New key naming convention in a deploy? If the app started writing `v2:user:42:profile` instead of `user:42:profile`, warm keys are stranded and never read again.
4. Cluster degraded? `CLUSTER INFO` shows this. During failover, some slots have no primary for 5-15 seconds.
5. Resharding in progress? `MOVED` redirects add latency and occasionally cause client-side misses if the client does not handle `ASK` correctly.

---

### 13. Trade-offs worth saying out loud

**Redis vs Memcached.** Memcached is multi-threaded per process (scales to all cores on a box) and has lower per-key overhead. Redis has one event loop per thread but supports TTL, data structures, atomic counters, persistence, pub/sub, and cluster mode natively. Use Memcached for a pure key-value cache where you want maximum density and the simplest operational model. Use Redis for almost everything else.

**Smart client vs proxy.** A proxy (like Twemproxy) is simpler for the client but is a single point of failure and does not support live resharding. Smart clients handle `MOVED`/`ASK` transparently and support full cluster mode. Proxies persist in legacy systems. New deployments use smart clients.

**Why single-threaded execution?** Every Redis command is atomic against the keyspace by construction. No locks needed anywhere. A slow command blocks everything, but that is a discipline problem (cap value sizes, ban `KEYS *`, watch slowlog). Redis 6+ added I/O threads for parsing and writing, but command execution stays serial.

**Multi-region cache.** Almost always wrong. The cache should be regional. Cross-region replication adds latency and consistency problems for a layer designed to be lossy. Let the database handle multi-region durability.

**What to revisit at 10x scale.** Move from one monolithic cluster to per-service clusters. Add explicit tiered caching (L1 local LRU, L2 Redis, L3 database) with metrics at each tier. For the highest-traffic keys, consider dedicated Memcached if Redis data structures are not needed.

---

### 14. Common mistakes

**"Redis stores things in memory."** True and irrelevant. The question is the cluster. Skip to partitioning.

**Modulo hashing.** Adding one server re-assigns 85% of keys at once, causing a mass cache-miss storm. Always say consistent hashing or hash slots.

**Consistent hashing without virtual nodes.** Gives wildly uneven distribution. The interviewer waits for "virtual nodes" or "16,384 hash slots."

**Confusing expiration and eviction.** TTL fires per-key when the timer runs out. Eviction fires globally when memory hits `maxmemory`. Different mechanisms, different policies, both matter.

**Recommending true LRU.** A doubly-linked list across 100M keys uses 1.6 GB of pointer overhead. Approximated LRU (sample 5, evict the oldest) is the right answer.

**No hot key answer.** The interviewer will ask. In-process LRU plus replica reads is the standard answer. Name both.

**Forgetting big key.** A sorted set with millions of entries blocks the event loop on any O(N) command. Bringing this up unprompted puts you above the bar.

**Sync replication everywhere.** Doubles write latency for operations that almost never need it. Default to async. Opt into `WAIT` per call for the few writes that matter.

**Cache stampede as "we will use TTLs."** TTLs cause stampedes. The fix is jitter plus request coalescing. Name the mechanism.

**No persistence story.** Even if the answer is "none," say so and explain why. The interviewer wants to hear that you reasoned about it.

The spine of a strong answer: hash slots plus smart client, replication and failover narrative, eviction policy choice, hot key mitigation. Candidates who cover all four and explain the trade-offs are clearly above the bar.
