---
id: 9
title: Design a Distributed Cache (Redis / Memcached)
category: Distributed Systems
topics: [consistent hashing, replication, eviction, ttl, hot keys]
difficulty: Medium
solution: solution.md
---

## Scene

Second round. The interviewer pulls up a whiteboard:

> *You've used Redis. Design it. Not the data structures inside one node, the cluster. How does the cluster work? How does it find the key? What happens when a node dies? How do you add a node without downtime?*

They cross their arms. This is the question that separates candidates who treat Redis as a magic black box from candidates who treat it as a distributed system. If I open with "Redis stores key-value pairs in memory" I have already lost ground. The interviewer wants the cluster story: hash slots, gossip, replication, failover, resharding.

The trap is that every other problem in this repo uses a distributed cache as a primitive. I will be asked this question, or something shaped like it, sooner or later.

## Step 1: clarify before designing

Five minutes. No drawing yet. The goal is at least six questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. Consistency model. "Strong consistency on writes, or eventual? Can a read after a write return stale data?" Most caches tolerate eventual. If the answer is strong, I am now designing a distributed database, not a cache, and the whole shape changes (synchronous replication, quorum reads, leader election on every write).
2. Persistence. "Pure in-memory, or do we need durability across restarts?" Memcached has none. Redis has RDB snapshots and AOF logs. The answer decides whether process restart loses data.
3. Value size and access patterns. "Values small (under 1 KB) or multi-MB? Just GET/SET, or do I need sorted sets, lists, pub/sub?" Memcached is flat KV. Redis is a data-structure server. The answer scopes the API surface.
4. Traffic shape. "Ops per second? Working set size? Read/write ratio?" Without these I cannot size the cluster. A typical answer: 1M ops/sec, 1 TB working set, 10:1 read:write.
5. Eviction policy. "When memory is full, what happens? LRU? Reject writes? TTL-only?" Product question. The wrong eviction policy silently corrupts application behavior.
6. Multi-region. "One region or many? Active-active or active-passive?" Cross-region cache replication is unusual; most teams keep caches regional and let the source-of-truth database deal with multi-region. Confirm.
7. Client topology. "Smart client that knows topology, or a proxy in front?" Redis Cluster uses smart clients (MOVED redirects); Memcached typically uses a proxy (mcrouter, twemproxy). Different operational properties.
8. Failure tolerance. "How many node failures must we survive without data loss? Without availability loss?" Drives replication factor and quorum.

A candidate who only asked "how many ops per second" and "how much data" missed consistency, persistence, and eviction. Those three are what the interviewer most wants to hear.

</details>

## Step 2: capacity estimates

Assume the interviewer gave me these numbers:

- 1M operations per second sustained, 3M peak
- 1 TB working set
- 10:1 read:write ratio
- Average value size: 1 KB
- Average key size: 50 bytes
- P99 latency target: under 5 ms in-region
- Durability: tolerate one node failure per shard with zero data loss

Compute on paper before revealing:

1. Reads per second and writes per second at peak
2. Network bandwidth at peak
3. Memory per node with a 6-node cluster, replication factor 2
4. Memory per node with a 12-node cluster
5. TCP connection count from 1000 application servers each pooling 50

<details>
<summary><b>Reveal: the math</b></summary>

Reads and writes. At 3M ops/sec peak with 10:1 read:write:

- Reads: ~2.7M/sec
- Writes: ~300K/sec

A single Redis node tops out around 100K-200K ops/sec on simple GET/SET, depending on hardware and pipelining. Higher with pipelining (commands batched on the wire), lower with complex commands. Need at least 15-30 nodes to handle 3M ops/sec before headroom.

Network bandwidth at peak. Each op moves roughly 50 byte key + 1 KB value + ~20 byte protocol overhead = ~1.1 KB. 3M times 1.1 KB = 3.3 GB/sec aggregate. Across 30 nodes, ~110 MB/sec per node. Comfortable within a 25 Gbps NIC budget (~3 GB/sec per node). Watch out: replication doubles write bandwidth for primaries.

Memory per node, 6-node cluster, RF=2. 1 TB working set times 2 copies = 2 TB total. Across 6 nodes, ~340 GB per node. Doable on the right instance (r7g.16xlarge has 512 GB), but blast radius of a single failure is huge: ~170 GB of primary keyspace briefly unavailable.

Memory per node, 12-node cluster, RF=2. 2 TB total / 12 = ~170 GB per node. Smaller blast radius. Each node owns ~85 GB primary. r7g.8xlarge (256 GB) fits with headroom.

Plus overhead: Redis itself uses ~20% more RAM than the dataset due to encoding, expiration metadata, and replication buffers. Budget 200 GB instances for a 170 GB dataset.

Connection count. 1000 servers times 50 connections = 50K client connections. With Redis Cluster smart clients, every client opens connections to every shard primary. 50K times 12 = 600K connections across the cluster, ~50K per node.

Redis handles this on a single event loop. 50K idle connections is fine. 50K active connections at 200K ops/sec means each connection does 4 ops/sec, light. But each connection costs ~20 KB of buffer memory in Redis: 50K times 20 KB = 1 GB per node just for connection state. Account for it.

The shape that matters. The cluster is bottlenecked by per-node CPU (single-threaded event loop) and memory, not network. Number of nodes and replication factor are driven by per-node ops/sec ceiling and blast-radius tolerance, not raw capacity.

</details>

## Step 3: partitioning

I have a key. I have N nodes. How do I decide which node owns the key? Three serious approaches: modulo hashing, consistent hashing, rendezvous (HRW) hashing. Ten minutes. Write down the pros and cons of each, and what happens when nodes come and go.

<details>
<summary><b>Reveal: comparison of partitioning schemes</b></summary>

| Scheme | How it works | Cost to add/remove a node | Distribution |
|--------|-------------|---------------------------|--------------|
| Modulo hashing | `node = hash(key) % N` | Catastrophic. Changing N from 6 to 7 reshuffles ~6/7 of keys. Cluster is unusable during rehash. | Perfectly even. |
| Consistent hashing | Map keys and nodes onto a ring; key goes to the next node clockwise. | Only ~1/N of keys move. Adding one node to a 12-node cluster moves ~8% of keys, all to the new node. | Uneven with few nodes. Fixed by virtual nodes. |
| Rendezvous (HRW) hashing | For each key, compute `hash(key, node_id)` for every node; key goes to the highest score. | Only ~1/N of keys move. Same as consistent hashing. | Even by construction. No virtual nodes needed. |

Consistent hashing with virtual nodes is the standard answer.

The naive ring (one position per node) gives uneven distribution: with 6 nodes, one node might own 30% of the keyspace and another 5%. The fix: 100 to 200 virtual nodes (vnodes) per real node at different ring positions. With 1200 vnodes for 6 nodes, distribution converges close to uniform.

Redis Cluster uses a variant: 16384 hash slots, statically. Each key hashes to `CRC16(key) mod 16384`. Slots are assigned to nodes. Rebalancing moves slots. Mathematically equivalent to consistent hashing with 16384 vnodes globally, but with explicit slot ownership rather than ring traversal.

Memcached takes a different path. The protocol has no built-in clustering; clustering is a client concern. Clients use libketama (consistent hashing with vnodes) to pick a server. No server-side rebalancing; if I change the client config, keys move and I accept the cache miss storm.

Migration math.

12-node cluster, ~85 GB per node. I add node 13. With consistent hashing, ~1/13 of keys move, distributed across the 12 old nodes (each gives ~1/12 of its keys to node 13):

- Each old node sends ~7 GB to node 13.
- Node 13 receives ~85 GB total.
- At 100 MB/sec migration rate (throttled to not saturate the network), ~85 GB / 100 MB/sec = 850 seconds, about 14 minutes.

During migration, what happens to lookups for keys being moved? Redis Cluster uses `ASK` redirects: if a key is in flight, the client first tries the old node, gets an `ASK` pointing to the new node, retries on the new node. After migration completes, the old node responds with `MOVED` for that slot.

Removing a node is the reverse: redistribute its slots to survivors, then take it down. Plan migration to complete before decommission.

Honest trade-off. Consistent hashing minimizes movement but does not eliminate it. A naive client that ignores `MOVED`/`ASK` and just retries on the old node will see cache misses (and possibly fall through to the source-of-truth database) during the migration window. Migration is not free; it is just contained.

</details>

## Step 4: sketch the architecture

Intentionally incomplete diagram. Fill in the five `[ ? ]` boxes. Hint: how does the client find a node, how do nodes find each other, what sits next to each primary for durability, what runs across all nodes to keep the cluster healthy?

```
                ┌────────────────────────┐
                │   Application Server    │
                │   ┌────────────────┐   │
                │   │  [ ? ]         │   │  (knows the cluster topology,
                │   └───────┬────────┘   │   picks the right node for a key)
                └───────────┼────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────────┐
        │            [ ? ]                              │  (decides which node owns the key:
        │                                               │   slot/hash → node mapping)
        └────────┬──────────────┬──────────────┬────────┘
                 │              │              │
                 ▼              ▼              ▼
         ┌────────────┐  ┌────────────┐  ┌────────────┐
         │ Shard 1    │  │ Shard 2    │  │ Shard 3    │
         │            │  │            │  │            │
         │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────────┐ │
         │ │Primary │ │  │ │Primary │ │  │ │Primary │ │
         │ └───┬────┘ │  │ └───┬────┘ │  │ └───┬────┘ │
         │     │ repl │  │     │ repl │  │     │ repl │
         │ ┌───▼────┐ │  │ ┌───▼────┐ │  │ ┌───▼────┐ │
         │ │ [ ? ]  │ │  │ │ [ ? ]  │ │  │ │ [ ? ]  │ │  (takes over on primary failure)
         │ └────────┘ │  │ └────────┘ │  │ └────────┘ │
         └────────────┘  └────────────┘  └────────────┘
                 ▲              ▲              ▲
                 └──────────────┼──────────────┘
                                │
                       ┌────────▼─────────┐
                       │    [ ? ]         │  (every node talks to every other node,
                       │                  │   exchanges topology and health)
                       └──────────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                ┌────────────────────────────────┐
                │   Application Server            │
                │   ┌────────────────────────┐   │
                │   │   Smart Client Library  │   │  Caches slot→node map.
                │   │   (Lettuce, Jedis,      │   │  Picks node by CRC16(key) % 16384.
                │   │    redis-py-cluster)    │   │  Reacts to MOVED/ASK redirects.
                │   └───────────┬────────────┘   │
                └───────────────┼────────────────┘
                                │
                                ▼
        ┌───────────────────────────────────────────────────┐
        │   Hash Slot Ring (16384 slots)                     │
        │   slot = CRC16(key) % 16384                        │
        │   slot → owning node, kept in sync by gossip       │
        └────────┬──────────────┬──────────────┬─────────────┘
                 │              │              │
                 ▼              ▼              ▼
         ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
         │  Shard 1     │ │  Shard 2     │ │  Shard 3     │
         │  slots 0-5460│ │  5461-10922  │ │  10923-16383 │
         │              │ │              │ │              │
         │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
         │ │ Primary  │ │ │ │ Primary  │ │ │ │ Primary  │ │
         │ │ accepts  │ │ │ │          │ │ │ │          │ │
         │ │ writes   │ │ │ │          │ │ │ │          │ │
         │ └────┬─────┘ │ │ └────┬─────┘ │ │ └────┬─────┘ │
         │      │ async │ │      │       │ │      │       │
         │      │ repl  │ │      │       │ │      │       │
         │ ┌────▼─────┐ │ │ ┌────▼─────┐ │ │ ┌────▼─────┐ │
         │ │ Replica  │ │ │ │ Replica  │ │ │ │ Replica  │ │
         │ │ serves   │ │ │ │          │ │ │ │          │ │
         │ │ reads    │ │ │ │          │ │ │ │          │ │
         │ │ promotes │ │ │ │          │ │ │ │          │ │
         │ │ on fail  │ │ │ │          │ │ │ │          │ │
         │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
         └──────────────┘ └──────────────┘ └──────────────┘
                 ▲              ▲              ▲
                 └──────────────┼──────────────┘
                                │
                       ┌────────▼─────────────┐
                       │  Gossip Protocol      │  Every node pings a few peers
                       │  (cluster bus on a    │  per second over a separate
                       │  separate port)       │  TCP port. Exchanges:
                       │                       │   - node liveness
                       │                       │   - slot ownership map
                       │                       │   - epoch/version numbers
                       │                       │   - failover votes
                       └───────────────────────┘
```

Component responsibilities:

- Smart Client Library. Holds the slot map locally. Sends each command directly to the right primary. If topology changed and the client targets the wrong node, the node responds `MOVED <slot> <new_address>` and the client updates its map. `ASK` is the same idea but only for the duration of a slot migration.
- Hash Slot Ring. 16384 fixed slots. Each slot owned by exactly one primary at any moment. Slot map replicated across all nodes via gossip.
- Primary per shard. Single writer for its slot range. Holds canonical data in memory.
- Replica per shard. Async-replicated copy. Can serve reads (slightly stale). Promoted to primary if the primary dies.
- Gossip protocol. The cluster's nervous system. Every node sends a `PING` to a small random subset of peers every ~100ms, including its view of the topology. After enough rounds, every node converges. Detects failures within a few seconds. Coordinates failover votes.

For Memcached, the picture is simpler: no built-in cluster, no gossip, no replication. The "cluster" lives only in the client library, which consistent-hashes across a static list of servers. If a server dies, clients detect the connection failure, mark the server dead, and re-hash to the survivors. No automatic failover. Intentional: Memcached prizes simplicity and assumes the application survives any single key disappearing.

</details>

## Step 5: eviction and expiration

The cache fills up. Now what? Two distinct mechanisms, often confused: expiration (TTL passes) and eviction (memory pressure forces removal regardless of TTL).

Ten minutes. List the policies Redis offers and when to choose each. What is "approximated LRU" and why does Redis use it instead of true LRU?

<details>
<summary><b>Reveal: eviction policies and the LRU approximation</b></summary>

Expiration (TTL).

`SET key value EX 60` gives the key a 60-second TTL. Two ways to remove an expired key:

- Passive. When a client tries to read the key, Redis notices it has expired and deletes it before responding. Cheap. But an expired key the application never reads sits in memory forever.
- Active. A background job samples 20 random keys with TTLs every 100ms. If more than 25% are expired, sample again immediately. Bounds maximum staleness in memory without scanning the whole keyspace.

Both run together. Active sampling keeps memory usage close to actual live keys.

Eviction policies (when `maxmemory` is hit).

| Policy | Behavior |
|--------|----------|
| `noeviction` | Reject writes with an error. Reads still work. Application has to handle the rejection. |
| `allkeys-lru` | Evict the approximate least-recently-used key from anywhere in the keyspace. |
| `volatile-lru` | Evict the approximate LRU key, but only from keys with a TTL. |
| `allkeys-lfu` | Evict the approximate least-frequently-used key (counts access frequency over a decay window). |
| `volatile-lfu` | Same as `allkeys-lfu` but restricted to keys with a TTL. |
| `allkeys-random` | Evict a random key. |
| `volatile-random` | Evict a random key from those with TTL. |
| `volatile-ttl` | Evict the key with the soonest TTL expiration. |

Choice depends on workload:

- Pure cache: `allkeys-lru` or `allkeys-lfu`. I do not care about specific keys; keep the hot set.
- Mixed cache + queue/state: `volatile-lru`. Put TTLs on truly cacheable keys; leave critical keys (counters, locks) without TTL so eviction skips them.
- No eviction allowed: `noeviction` with monitoring. Used when the cache is the source of truth (which I should not be doing, but sometimes I am).

Approximated LRU.

True LRU needs a doubly-linked list of every key. Every access moves the touched key to the head. With 100M keys and millions of ops/sec, the pointer overhead hurts: ~16 bytes per key for prev/next pointers, plus cache-unfriendly random pointer chasing.

Redis approximates. Sample 5 keys at random (configurable, default 5, can raise to 10 for closer-to-true LRU) and evict the oldest of the sample. With sample size 10, the result is statistically close to true LRU at a fraction of the overhead.

How "oldest" is tracked: every key has a 24-bit "last access time" field (`lru` field in the object header), updated on read or write. Sampling reads this field. No linked list, no pointer overhead.

Trade-off: occasionally evicts a not-quite-oldest key. For cache use cases this is invisible. Memory savings (no list pointers across 100M keys = 1.6 GB saved per node) is significant.

LFU is implemented similarly: sampled, with a counter that increments on access and decays over time so old-frequent keys do not stay sticky forever.

</details>

## Step 6: replication and failover

Each primary has one or more replicas. When the primary dies, a replica takes over. How: synchronously or asynchronously? Quorum or unanimous? Who picks the new primary?

Ten minutes:

- Default async replication. What is the data loss window?
- Synchronous option. What is the latency cost?
- How does the cluster detect a primary is down?
- Who decides on the new primary?
- What if the "dead" primary comes back?

<details>
<summary><b>Reveal: replication and failover</b></summary>

Async replication (default).

Primary handles the write, replies to the client, queues the write for replication. Replicas pull from the primary's replication buffer and apply.

- Latency: write returns as soon as the primary commits to memory. Replicas catch up in milliseconds under normal conditions.
- Data loss window: the writes in the buffer that have not yet reached the replica. Under steady-state load, a handful of operations (a few KB). Under load spike, larger.
- If the primary dies before replication catches up, those writes are lost.

This is the right default for cache workloads. I am caching, not storing the only copy.

Synchronous replication (the `WAIT` command).

After a write, the client can issue `WAIT <numreplicas> <timeout_ms>`. Blocks until the specified number of replicas have acked the previous writes, or the timeout fires.

- Latency: bounded by the slowest of the N replicas. One replica same-AZ adds ~1ms. Cross-AZ adds ~5-10ms.
- Data loss window: zero if I wait for all replicas. But `WAIT` does not give true synchronous semantics if the primary crashes after the write but before the WAIT returns; I still need a fencing mechanism to prevent a stale primary from accepting writes after a partition.

Sync replication is rare for caches. Use it only for the small subset of writes intolerant to loss (a financial counter, a deduplication marker).

Failure detection.

Redis Cluster uses gossip-based detection. Each node pings a random subset of peers every 100ms. If a node fails to respond for ~5 seconds, the pinging node marks it PFAIL (possibly failed) and gossips this. Once a majority of primaries report PFAIL for the same node, it becomes FAIL (cluster-wide consensus that it is down).

Threshold is configurable. Lower = faster failover but more false positives during network blips. Higher = more conservative but slower recovery.

Leader election (failover).

When a primary is declared FAIL, its replicas race to take over.

1. Each replica waits a small delay proportional to its replication lag (least-lagging goes first).
2. The replica requests votes from all other primaries: "should I take over for failed primary X?"
3. Other primaries vote yes if they agree X is FAIL and the requesting replica is the only one asking within an epoch.
4. With a majority vote, the replica promotes itself, claims the slot range, and gossips the new topology.
5. Clients discover the new primary via `MOVED` redirects on the next request.

Total failover time: typically 5-15 seconds in well-tuned clusters. Most of that is the detection window; the actual promotion takes under a second.

Memcached has no failover. Server dies, the client library notices, hashes to remaining servers. Keys on the dead server are simply gone until it returns.

The "old primary returns" problem.

The old primary was network-partitioned, not dead. Now it can talk again. The cluster has already promoted a replica.

On first gossip exchange, the returning primary learns its epoch is stale. It demotes itself to replica of the new primary, discards in-memory state, syncs. Any writes accepted during the partition window are lost. Canonical CAP trade-off: the cluster chose availability (let the new primary take writes during the partition) and accepts that partitioned writes on the old primary cannot be reconciled.

To minimize this: use `min-replicas-to-write` so a primary refuses writes if it cannot reach at least one replica. Trades availability for consistency. Some teams set this; most accept the small loss window for caches.

</details>

## Step 7: read the full solution

That is the heart: partitioning, replication, failover, eviction. The solution adds persistence options (RDB vs AOF), the hot key and big key problems, encoding tricks for memory efficiency, and the day-2 problems that determine whether the cluster survives production.

## Follow-up questions

Answer each in 2 to 3 sentences before reading the solution.

1. Hot key. One key gets 500K req/s. It lives on one shard. That shard's CPU pegs. What do I do?
2. Big key. One key holds a sorted set with 2M entries. `ZRANGE 0 -1` stalls the event loop for 800ms. Every other op queues behind it. How do I prevent and how do I recover?
3. I set a TTL of 60 seconds on a million keys at once (bulk import). 60 seconds later, application latency P99 doubles. Why? Fix it without changing the TTL.
4. Persistence. When RDB? When AOF? When both? When neither? What does each cost in latency and data loss window?
5. Resharding. 6 nodes growing to 12. How do I do it without dropping ops/sec or losing data? How long does it take?
6. Network partition. Two nodes in one DC can talk to each other but not to the rest of the cluster. What happens? Will the partition try to elect its own primaries?
7. Memory fragmentation. Redis reports `used_memory_rss / used_memory = 1.8`. Translate that for the interviewer and say what I do about it.
8. Cache stampede. A popular cache key expires. 10,000 concurrent requests miss, hit the database. Database melts. How do I prevent this in the cache layer?
9. Inconsistent reads. A user writes `SET balance 100`, immediately reads back, sees the old value. Why? When can this happen in a cluster that uses replicas for reads?
10. Cache hit rate dropped from 95% to 60% overnight. No deploys. What is my investigation path?

## Related problems

- [URL Shortener (001)](../001-url-shortener/question.md), uses this cache heavily on the redirect path. The hot key problem there is the same one analyzed here.
- [News Feed (002)](../002-news-feed/question.md), timeline store is a Redis cluster. Sorted-set encoding, hot-key replicas, cold-user eviction all come from this design.
- [Typeahead Autocomplete (005)](../005-typeahead-autocomplete/question.md), the prefix index sits in the same kind of distributed cache. The big-key problem is acute there (top prefixes hold large suggestion lists).
