---
id: 9
title: Design a Distributed Cache (Redis / Memcached)
category: Distributed Systems
topics: [consistent hashing, replication, eviction, ttl, hot keys]
difficulty: Medium
solution: solution.md
---

## Scene

It is the second round. Your interviewer pulls up a whiteboard and says:

> *You've used Redis. I want you to design it. Not the data structures inside one node, the cluster. How does the cluster work? How does it find the key? What happens when a node dies? How do you add a node without downtime?*

They cross their arms. This is the question that filters candidates who treat Redis as a magic black box from those who understand it as a distributed system. If you start with "Redis stores key-value pairs in memory" you have already lost ground. The interviewer knows that. They want to hear about the cluster: hash slots, gossip, replication, failover, resharding.

The trap: every other problem in this repo uses a distributed cache as a primitive. You will be asked this question or one shaped like it.

## Step 1: clarify before you design

Take 5 minutes. Do not start drawing. What do you ask the interviewer? Aim for at least six questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Consistency model.** "Strong consistency on writes, or is eventual OK? Can a read after a write return stale data?" Most cache use cases tolerate eventual consistency. If the answer is "strong," you are now designing a distributed database, not a cache, and the design changes significantly (synchronous replication, quorum reads, leader election on every write).
2. **Persistence.** "Is this a pure in-memory cache, or do we need durability across restarts?" Memcached has no persistence. Redis offers RDB snapshots and AOF logs. The answer determines whether you can lose data on a process restart.
3. **Value size and access patterns.** "Are values small (under 1 KB) or can they be large (multi-MB)? Mostly GET/SET, or do we need sorted sets, lists, pub/sub?" Memcached is a flat KV store. Redis is a data-structure server. The answer scopes the API surface dramatically.
4. **Traffic shape.** "Operations per second? Working set size? Read/write ratio?" Without this you cannot size the cluster. A typical answer: 1M ops/sec, 1 TB working set, 10:1 read:write.
5. **Eviction policy.** "When memory is full, what should happen? Evict LRU? Reject writes? TTL-only?" This is a product question. The wrong eviction policy can silently corrupt application behavior.
6. **Multi-region.** "One region or many? Active-active or active-passive replication across regions?" Cross-region cache replication is unusual; most teams keep caches regional and let the source-of-truth database handle multi-region. Confirm this assumption.
7. **Client topology.** "Smart client that knows the cluster topology, or a proxy in front?" Redis Cluster uses smart clients (MOVED redirects); Memcached typically uses a proxy (mcrouter, twemproxy). Different operational properties.
8. **Failure tolerance.** "How many node failures must we survive without data loss? Without availability loss?" Determines replication factor and quorum settings.

If you only asked "how many ops per second" and "how much data," you have left the consistency, persistence, and eviction questions unaddressed. Those three are the questions an interviewer most wants to hear.

</details>

## Step 2: capacity estimates

Assume the interviewer gave you these numbers:

- 1M operations per second sustained, 3M peak
- 1 TB working set (the data the application actually uses)
- 10:1 read:write ratio
- Average value size: 1 KB
- Average key size: 50 bytes
- P99 latency target: under 5 ms in-region
- Durability: tolerate one node failure per shard with zero data loss

Compute (do this on paper before revealing):

1. Reads per second and writes per second at peak
2. Network bandwidth at peak
3. Memory per node if you use a 6-node cluster, with replication factor 2
4. Memory per node if you use a 12-node cluster
5. Number of TCP connections from a fleet of 1000 application servers, each running a connection pool of 50

<details>
<summary><b>Reveal: the math</b></summary>

**Reads and writes.**
At 3M ops/sec peak with 10:1 read:write:
- Reads: ~2.7M/sec
- Writes: ~300K/sec

A single Redis node tops out around 100K-200K ops/sec with simple GET/SET, depending on hardware and pipelining. Higher with pipelining (commands batched on the wire), lower if the workload uses complex commands. You will need at least 15-30 nodes to handle 3M ops/sec, before accounting for headroom.

**Network bandwidth at peak.**
Each operation moves roughly: 50 byte key + 1 KB value + ~20 byte protocol overhead = ~1.1 KB.
3M × 1.1 KB = **3.3 GB/sec aggregate**. Across 30 nodes, that's ~110 MB/sec per node. Within a 25 Gbps NIC budget (~3 GB/sec per node), comfortable. Be careful: replication doubles write bandwidth for primaries.

**Memory per node, 6-node cluster, RF=2.**
1 TB working set × 2 copies = 2 TB total. Across 6 nodes = ~340 GB per node. Doable with the right instance type (r7g.16xlarge has 512 GB), but the blast radius of a single node failure is huge: ~170 GB of primary keyspace becomes briefly unavailable.

**Memory per node, 12-node cluster, RF=2.**
2 TB total / 12 = ~170 GB per node. Smaller blast radius. Each node owns ~85 GB of primary keyspace. r7g.8xlarge (256 GB) fits with headroom.

Plus overhead: Redis itself uses ~20% more RAM than the dataset due to encoding, expiration metadata, and replication buffers. Budget 200 GB instances for a 170 GB dataset.

**Connection count.**
1000 servers × 50 connections = 50,000 connections from clients. If using Redis Cluster with smart clients, every client opens connections to every shard primary. That is 50,000 × 12 = **600K connections** across the cluster, ~50K per node.

Redis handles this on a single event loop. 50K idle connections is fine. 50K active connections at 200K ops/sec means each connection does 4 ops/sec, which is light. But each connection costs ~20 KB of buffer memory in Redis: 50K × 20 KB = 1 GB just for connection state. Account for it.

**Key insight from the math.** The cluster is bottlenecked by per-node CPU (single-threaded event loop) and by memory, not by network. The shape of the cluster (number of nodes, replication factor) is driven by per-node ops/sec ceiling and blast-radius tolerance, not raw capacity.

</details>

## Step 3: partitioning

You have a key. You have N nodes. How do you decide which node owns the key? This is the central question. Three serious approaches: **modulo hashing**, **consistent hashing**, **rendezvous (HRW) hashing**. Take 10 minutes to write down the pros and cons of each, and what happens to your cluster when you add or remove a node.

<details>
<summary><b>Reveal: comparison of partitioning schemes</b></summary>

| Scheme | How it works | Cost to add/remove a node | Distribution |
|--------|-------------|---------------------------|--------------|
| **Modulo hashing** | `node = hash(key) % N` | Catastrophic. Changing N from 6 to 7 reshuffles ~6/7 of all keys. Cluster becomes unusable during rehash. | Perfectly even. |
| **Consistent hashing** | Map keys and nodes onto a ring; key goes to the next node clockwise. | Only ~1/N of keys move. Adding one node to a 12-node cluster moves ~8% of keys, all to the new node. | Uneven with few nodes. Fixed by virtual nodes. |
| **Rendezvous (HRW) hashing** | For each key, compute `hash(key, node_id)` for every node; key goes to the node with the highest score. | Only ~1/N of keys move. Same as consistent hashing. | Even by construction. No virtual nodes needed. |

**Consistent hashing with virtual nodes is the standard answer.**

The naive ring (one position per node) gives uneven distribution: with 6 nodes on a ring, one node might own 30% of the keyspace and another 5%. The fix: assign each real node 100 to 200 virtual nodes (vnodes) at different ring positions. With 1200 vnodes for 6 nodes, distribution converges close to uniform.

Redis Cluster uses a variant: **16384 hash slots** statically. Each key is hashed to `CRC16(key) mod 16384`. Slots are assigned to nodes. To rebalance, slots migrate from one node to another. This is mathematically equivalent to consistent hashing with 16384 vnodes globally, but with explicit slot ownership rather than ring traversal.

Memcached takes a different path. The protocol has no built-in clustering; clustering is a client concern. Clients use libketama (consistent hashing with vnodes) to pick a server. There is no server-side rebalancing; if you change the client config, keys move and you accept the cache miss storm.

<b>Migration math.</b>

You have a 12-node cluster, each node holds ~85 GB. You add node 13. With consistent hashing, ~1/13 of keys move, distributed across all 12 old nodes (each gives ~1/12 of its keys to node 13). Concretely:

- Each old node sends ~7 GB to node 13.
- Node 13 receives ~85 GB total.
- At 100 MB/sec migration rate (limited so it doesn't saturate the network), that is ~85 GB / 100 MB/sec = 850 seconds, around 14 minutes.

During migration, what happens to lookups for keys being moved? Redis Cluster handles this with `ASK` redirects: if a key is in flight, the client first tries the old node, gets an `ASK` response pointing to the new node, retries on the new node. After migration completes, the old node responds with `MOVED` for that slot.

Removing a node is the reverse: redistribute its slots to the surviving nodes, then take it down. Plan migration to complete before the node is decommissioned.

**The honest trade-off.** Consistent hashing minimizes movement but does not eliminate it. A naive client that ignores `MOVED`/`ASK` and just retries on the old node will see cache misses (and possibly fallthrough load on the source-of-truth database) during the migration window. The migration is not free; it is just contained.

</details>

## Step 4: sketch the high-level architecture

Here is an intentionally incomplete diagram. Fill in the five `[ ? ]` boxes. Hint: think about how the client finds a node, how nodes find each other, what sits next to each primary for durability, and what runs across all nodes to keep the cluster healthy.

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

- **Smart Client Library.** Holds the slot map locally. Sends each command directly to the right primary. If the topology changed and the client targets the wrong node, the node responds `MOVED <slot> <new_address>` and the client updates its map. `ASK` is similar but only for the duration of a slot migration.
- **Hash Slot Ring.** 16384 fixed slots. Each slot is owned by exactly one primary at any moment. Slot map is replicated across all nodes via gossip.
- **Primary per shard.** Single writer for its slot range. Holds the canonical data in memory.
- **Replica per shard.** Async-replicated copy. Can serve reads (with the trade-off of slightly stale data). Promoted to primary if the primary dies.
- **Gossip protocol.** The cluster's nervous system. Every node sends a `PING` to a small random subset of peers every ~100ms. Includes the node's view of the topology. After enough rounds, every node converges on the same view. Detects failures within a few seconds. Coordinates failover votes.

For Memcached the picture is simpler: there is no built-in cluster, no gossip, no replication. The "cluster" exists only in the client library, which consistent-hashes across a static list of servers. If a server dies, clients detect the connection failure, mark the server dead, and re-hash to the remaining ones. No automatic failover. This is intentional: Memcached prizes simplicity and assumes the application can survive any single key disappearing.

</details>

## Step 5: eviction and expiration

The cache fills up. Now what? There are two distinct mechanisms: **expiration** (keys with a TTL that passes) and **eviction** (memory pressure forcing removal of keys regardless of TTL). They are often confused.

Take 10 minutes. List the policies Redis offers, and explain when you would choose each. What is "approximated LRU" and why does Redis use it instead of true LRU?

<details>
<summary><b>Reveal: eviction policies and the LRU approximation</b></summary>

**Expiration (TTL).**

When you `SET key value EX 60`, the key has a 60-second TTL. Two ways to remove an expired key:

- **Passive.** When a client tries to read the key, Redis notices it has expired and deletes it before responding. Cheap. But an expired key the application never reads sits in memory forever.
- **Active.** A background job samples 20 random keys with TTLs every 100ms. If more than 25% are expired, sample again immediately. This bounds the maximum staleness in memory without scanning the whole keyspace.

Both run together. Active sampling is enough to keep memory usage close to actual live keys.

**Eviction policies (when `maxmemory` is hit).**

Redis offers several:

| Policy | Behavior |
|--------|----------|
| `noeviction` | Reject writes with an error. Reads still work. The application has to handle the rejection. |
| `allkeys-lru` | Evict the approximate least-recently-used key from anywhere in the keyspace. |
| `volatile-lru` | Evict the approximate LRU key, but only from keys that have a TTL set. |
| `allkeys-lfu` | Evict the approximate least-frequently-used key (counts access frequency over a decay window). |
| `volatile-lfu` | Same as `allkeys-lfu` but restricted to keys with a TTL. |
| `allkeys-random` | Evict a random key. |
| `volatile-random` | Evict a random key from those with TTL. |
| `volatile-ttl` | Evict the key with the soonest TTL expiration. |

The choice depends on workload:

- **Pure cache.** `allkeys-lru` or `allkeys-lfu`. You do not care about specific keys; you want to keep the hot set.
- **Mixed cache + queue/state.** `volatile-lru`. Put TTLs on truly-cacheable keys; leave critical keys (counters, locks) without TTL and protect them from eviction.
- **No eviction allowed.** `noeviction` with monitoring. Used when the cache is the source of truth (you should not be doing this, but sometimes you are).

**Approximated LRU.**

True LRU needs a doubly-linked list of every key. On every access you move the touched key to the head. With 100M keys and millions of ops/sec, the pointer overhead is significant: ~16 bytes per key for the prev/next pointers, plus cache-unfriendly random pointer chasing.

Redis approximates LRU. It samples 5 keys at random (configurable, default 5, can raise to 10 for closer-to-true LRU), and evicts the oldest of the sample. With sample size 10, the result is statistically close to true LRU at a fraction of the overhead.

How "oldest" is tracked: every key has a 24-bit "last access time" field (`lru` field in the object header), updated on read or write. Sampling reads this field. No linked list, no pointer overhead.

The trade-off: occasionally evicts a not-quite-oldest key. For cache use cases this is invisible. The memory savings (no list pointers across 100M keys = 1.6 GB saved per node) is significant.

LFU is implemented similarly: sampled, with a counter that increments on access and decays over time so old-frequent keys do not stay sticky forever.

</details>

## Step 6: replication and failover

Each primary has one or more replicas. When the primary dies, a replica takes over. The question is **how**: synchronously or asynchronously? Quorum or unanimous? How does the cluster pick which replica to promote?

Spend 10 minutes thinking through:

- Default async replication. What is the data loss window?
- Synchronous replication option. What is the latency cost?
- How does the cluster detect a primary is down?
- Who decides on the new primary?
- What if the "dead" primary comes back?

<details>
<summary><b>Reveal: replication and failover</b></summary>

**Async replication (default).**

The primary handles the write, replies to the client, and queues the write for replication. Replicas pull the write from the primary's replication buffer and apply it.

- Latency: write returns as soon as the primary commits to memory. Replicas catch up in milliseconds under normal conditions.
- Data loss window: the writes in the replication buffer that have not yet reached the replica. Under steady-state load, this is a handful of operations (a few KB). Under load spike, it can be larger.
- If the primary dies before replication catches up, those writes are lost.

This is the right default for cache workloads. You are caching, not storing the only copy.

**Synchronous replication (the `WAIT` command).**

After a write, the client can issue `WAIT <numreplicas> <timeout_ms>`. This blocks until the specified number of replicas have acknowledged the previous writes, or the timeout fires.

- Latency: now bounded by the slowest of the N replicas. For one replica in the same AZ, adds ~1ms. For cross-AZ, adds ~5-10ms.
- Data loss window: zero if you wait for all replicas. But `WAIT` does not provide true synchronous semantics if the primary crashes after the write but before the WAIT returns; you still need a fencing mechanism to prevent a stale primary from accepting writes after a partition.

In practice, sync replication is rare for caches. Use it only for the small subset of writes that are intolerant to loss (a financial counter, a deduplication marker).

**Failure detection.**

Redis Cluster uses gossip-based detection. Each node pings a random subset of peers every 100ms. If a node fails to respond to pings for ~5 seconds, the pinging node marks it as PFAIL (possibly failed) and gossips this. Once a majority of primaries report PFAIL for the same node, it becomes FAIL (cluster-wide consensus that it is down).

The threshold is configurable. Lower = faster failover but more false positives during network blips. Higher = more conservative but slower recovery.

**Leader election (failover).**

When a primary is declared FAIL, its replicas race to take over.

1. Each replica waits a small delay proportional to its replication lag (least-lagging replica goes first).
2. The replica requests votes from all other primaries: "should I take over for failed primary X?"
3. Other primaries vote yes if they agree primary X is FAIL and the requesting replica is the only one asking (within an epoch).
4. With a majority vote, the replica promotes itself, claims the slot range, and gossips the new topology.
5. Clients discover the new primary via `MOVED` redirects on the next request.

Total failover time: typically 5-15 seconds in well-tuned clusters. Most of that is the failure detection window; the actual promotion takes under a second.

Memcached has no failover. If a server dies, the client library notices and starts hashing to remaining servers. Keys on the dead server are simply gone until it returns.

**The "old primary returns" problem.**

The old primary was network-partitioned, not actually dead. Now it can talk again. The cluster has already promoted a replica. What happens?

The returning primary, on first gossip exchange, learns it has a stale epoch. It demotes itself to replica of the new primary, discards its in-memory state, and syncs from the new primary. Any writes accepted during the partition window are lost. This is the canonical CAP-theorem trade-off: the cluster chose availability (allowed the new primary to accept writes during the partition) and accepted that the partitioned writes on the old primary cannot be reconciled.

To minimize this: use `min-replicas-to-write` so a primary refuses writes if it cannot reach at least one replica. Trades availability for consistency. Some teams set this; most accept the small loss window for caches.

</details>

## Step 7: read the full solution

You have covered the heart of the question: partitioning, replication, failover, eviction. The solution covers persistence options (RDB vs AOF), the hot key and big key problems, encoding tricks for memory efficiency, and the day-2 problems that determine whether your cluster survives production.

## Follow-up questions

Try answering each in 2 to 3 sentences before reading the solution.

1. **Hot key.** One key gets 500K req/s. It lives on one shard. That shard's CPU pegs at 100%. What do you do?
2. **Big key.** One key holds a sorted set with 2M entries. `ZRANGE 0 -1` on it stalls the event loop for 800ms. Every other operation on that node queues behind it. How do you prevent and how do you recover?
3. **You set a TTL of 60 seconds on a million keys, all at once (bulk import).** 60 seconds later, your application latency P99 doubles. Why? How do you fix it without changing the TTL?
4. **Persistence.** When would you choose RDB snapshots, when AOF, when both, when neither? What does each cost in terms of latency and data loss window?
5. **Resharding.** You have 6 nodes and need to grow to 12. How do you do this without dropping ops/sec or losing data? How long does it take?
6. **Network partition.** Two nodes in one DC can talk to each other but not to the rest of the cluster. What happens? Will the partition try to elect its own primaries?
7. **Memory fragmentation.** Your Redis reports `used_memory_rss / used_memory = 1.8`. Translate that for the interviewer and say what you do about it.
8. **Cache stampede.** A popular cache key expires. 10,000 concurrent requests all miss, all go to the database. The database melts. How do you prevent this in the cache layer?
9. **Inconsistent reads.** A user writes `SET balance 100`, immediately reads back, and sees the old value. Why? When can this happen in a cluster that uses replicas for reads?
10. **You discover your cache hit rate dropped from 95% to 60% overnight.** No deploys. What's your investigation path?

## Related problems

- **[URL Shortener (001)](../001-url-shortener/question.md)**, uses this cache heavily on the redirect path. The hot key problem in that design is the same one analyzed here.
- **[News Feed (002)](../002-news-feed/question.md)**, the timeline store is a Redis cluster. Sorted-set encoding, hot-key replicas, and cold-user eviction all come from this design.
- **[Typeahead Autocomplete (005)](../005-typeahead-autocomplete/question.md)**, the prefix index sits in the same kind of distributed cache. The big-key problem is acute there (top prefixes hold large suggestion lists).
