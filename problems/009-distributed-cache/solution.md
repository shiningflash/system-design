## Solution: Design a Distributed Cache (Redis / Memcached)

### TL;DR

A distributed cache is a sharded, replicated, in-memory key-value store with a few hard problems hidden inside: how to find a key when nodes come and go, how to keep replicas close enough to primaries to not lose much on failover, how to evict gracefully under pressure, and how to survive the few keys that get all the traffic.

The standard design is consistent hashing (or fixed hash slots) over a cluster of shards, each shard a primary plus one or more async replicas, with a gossip protocol exchanging topology and liveness. Clients are smart: they cache the slot map locally and adjust when nodes redirect them. Eviction is approximated LRU or LFU because true LRU is too expensive in memory. Persistence is optional and almost always trades latency for durability.

The interesting engineering is at the edges: the hot key that breaks a shard (replicas plus in-process cache), the big key that stalls the event loop (cap value size at write time, scan instead of bulk read), and the resharding that has to happen without dropping traffic (incremental slot migration with `ASK` redirects).

### 1. Clarifying questions

Covered in `question.md`. The eight that matter: consistency model, persistence, value size and data structures, traffic shape, eviction policy, multi-region, client topology, failure tolerance. Skipping the persistence and consistency questions is the classic miss; they change the design fundamentally.

### 2. Capacity estimates

Reiterated:

- 1M ops/sec sustained, 3M peak. 10:1 read:write.
- 1 TB working set. 1 KB average value, 50 byte average key.
- Single-node Redis ceiling: ~150K ops/sec for mixed GET/SET, higher with pipelining.
- Conclusion: 6-shard cluster is undersized for peak; 12-shard is comfortable. With RF=2 that is 24 nodes total.
- Per node: ~170 GB primary data, plus replication buffers and connection state, plus 20% encoding overhead. 256 GB instances with headroom.
- Network: ~110 MB/sec per node, well under a 25 Gbps NIC.
- Connections: 50K per node from 1000 app servers. Bookkeeping cost ~1 GB per node for buffers.

The bottleneck is per-shard CPU (single event-loop), not memory or network. Sizing is driven by ops/sec per shard and blast radius, not raw storage.

### 3. API design

Two distinct things to design: the wire protocol and the client-facing semantics.

**Core commands (Redis-style).**

| Command | Behavior |
|---------|----------|
| `GET key` | Return value or nil. O(1). |
| `SET key value [EX seconds] [NX|XX]` | Set with optional TTL. NX = only if not exists; XX = only if exists. O(1). |
| `MGET key1 key2 ...` | Batch get. Cluster note: keys must be on the same shard, or the client does the fan-out. |
| `MSET k1 v1 k2 v2 ...` | Batch set. Same shard constraint. |
| `DEL key` | Delete. Returns 1 if existed, 0 otherwise. |
| `EXPIRE key seconds` | Add TTL to an existing key. |
| `TTL key` | Return remaining TTL, -1 if no TTL, -2 if key does not exist. |
| `INCR key` / `INCRBY key n` | Atomic increment of integer value. Foundational for counters. |
| `EXISTS key` | Cheap existence check. |
| `SCAN cursor [MATCH pattern]` | Cursored iteration. Use this, never `KEYS *`. |

Plus the data-structure commands: `HSET`/`HGET` (hash), `LPUSH`/`LRANGE` (list), `SADD`/`SMEMBERS` (set), `ZADD`/`ZRANGE` (sorted set), `XADD`/`XREAD` (streams). Each is O(1) or O(log N) per element, but operations on whole structures (`SMEMBERS` on a set with 1M elements) are O(N) and block the event loop.

Memcached is a strict subset: `get`, `set`, `add`, `replace`, `delete`, `incr`, `decr`, `cas` (check-and-set), and that is essentially it. No data structures, no scripting, no pub/sub.

**Cluster-aware semantics.**

- **Hash tags.** `SET {user:42}:profile ...` and `SET {user:42}:settings ...` hash to the same slot because `{...}` is the only thing hashed. This lets multi-key operations like `MGET` or transactions stay on one shard.
- **Multi-key constraints.** `MGET k1 k2` and `MULTI`/`EXEC` only work if all keys are on the same shard. The client either constrains keys with hash tags, or fans out and merges client-side.
- **Pipelining.** Send N commands without waiting for responses, then read N responses. Cuts round-trip overhead from N RTTs to 1 RTT. Critical for throughput. A single client can push 500K ops/sec with aggressive pipelining where without it gets 50K.

**Errors clients must handle.**

| Response | Meaning | Client action |
|----------|---------|---------------|
| `MOVED <slot> <host:port>` | Slot now lives on another node | Update slot map, retry on new node |
| `ASK <slot> <host:port>` | Slot is migrating; this specific key is on the new node temporarily | Send `ASKING` then the command to the new node, do not update slot map yet |
| `CLUSTERDOWN` | Cluster is unhealthy (e.g., some slots have no owner) | Back off and retry; alert the operator |
| `LOADING` | Node is loading data from disk (post-restart) | Retry after a delay |
| `READONLY` | Tried to write to a replica | Update slot map, write to primary |
| `OOM` (out of memory) | Eviction disabled and memory is full | Application must handle write rejection |

### 4. Data model and in-memory representation

The conceptual model is a flat hash table: keys to values. The interesting engineering is how Redis encodes values to save memory.

**Object header.** Every Redis value has a header with type tag, encoding, reference count, and a 24-bit "last access time" used for LRU/LFU sampling. ~16 bytes per object.

**Encoding tricks.** Redis picks an encoding based on size and content.

| Type | Small encoding | Large encoding | Switch threshold |
|------|---------------|----------------|------------------|
| String | `embstr` (one allocation, 44 bytes max) | `raw` (separate allocation) | 44 bytes |
| String of small integer | `int` (no allocation, value in the header pointer) | `embstr`/`raw` | Not an integer |
| Hash | `listpack` (flat array of key-value pairs) | `hashtable` | 128 entries or any field > 64 bytes |
| List | `listpack` | `quicklist` (linked list of listpacks) | 128 entries or any element > 64 bytes |
| Set of integers | `intset` (sorted dense array) | `hashtable` | 512 entries or any non-integer |
| Sorted set | `listpack` | `skiplist + hashtable` | 128 entries or any element > 64 bytes |

Why this matters: a 10-field hash in `listpack` form takes ~200 bytes; the same hash in `hashtable` form takes ~600 bytes. At 100M small hashes that is 40 GB of memory saved by staying in `listpack`.

The flip side: operations on `listpack` are O(N) (linear scan). Once you cross the threshold and switch to `hashtable`, operations become O(1) but memory triples. Redis chooses the encoding at write time and will upgrade in place (never downgrades).

For Memcached, the encoding is simpler: a slab allocator with fixed-size slabs (64 B, 128 B, 256 B, ...). A 100-byte value goes into a 128-byte slab. Memory efficient if your value sizes cluster on slab boundaries; wasteful if they do not. The "calcification" problem: once slabs are allocated to size classes, they cannot easily be reassigned. If your traffic shifts from many small values to fewer large values, the small slabs sit empty.

**Expiration metadata.** Keys with TTL have an entry in a secondary "expires" dictionary mapping key to expiration timestamp. Active expiration samples from this dictionary, not from the main keyspace.

### 5. Core algorithm: consistent hashing and replication

#### Consistent hashing with hash slots

Redis Cluster uses 16384 fixed slots. The number is chosen to be large enough that slots can be assigned at fine granularity (each shard owns 16384/N slots, where N is the number of shards), and small enough that the slot map is a manageable size (16384 bits = 2 KB) to gossip around.

```
slot = CRC16(key) & 16383
node = slot_map[slot]
```

The slot map is replicated across all nodes via gossip. Clients also cache it. When the map changes (shard added, removed, or migrated), clients learn through `MOVED` redirects on the next operation that targets the wrong node.

**Why CRC16 instead of MurmurHash, xxHash, or SHA?** CRC16 is fast (a few cycles per byte), has well-understood distribution properties on text keys, and is good enough; cryptographic hash is overkill. xxHash would be faster but the difference is in nanoseconds when the rest of the operation is microseconds. CRC16 is a "good enough" historical choice.

**Hash tags.** The hashing function only hashes the substring inside `{...}` if present:

```
CRC16("{user:42}:profile") = CRC16("user:42")
CRC16("{user:42}:settings") = CRC16("user:42")
```

So both keys land on the same slot. The application uses this to colocate related keys for multi-key operations.

#### Replication protocol

Each shard has one primary and one or more replicas. Replication is asynchronous by default.

**The replication backlog.** The primary maintains a circular buffer (default 1 MB, configurable) of recent writes. When a replica connects, the primary checks if the replica's replication offset is within the backlog. If yes, it streams forward from that offset (partial resync). If no, the replica falls behind too far and the primary issues a full resync: serializes the entire dataset (RDB snapshot) and ships it to the replica, then streams new writes.

Partial resyncs are fast (seconds). Full resyncs are slow (depends on dataset size; ~30 GB takes a few minutes). Size the backlog so that brief network blips do not trigger full resyncs. 1 MB is too small for most production workloads; 256 MB or even 1 GB is common.

**Replica reads.** Replicas can serve reads if the client issues `READONLY` on connect. The application accepts that reads may be slightly stale (typically by milliseconds; can spike under replication lag).

**WAIT for sync writes.** A client can issue `WAIT <N> <timeout>` after a write to block until N replicas have ack'd. Not true synchronous replication; it is a "best effort" check. If the primary crashes during the wait, the client gets timeout, but the write may or may not have made it to replicas. For true sync semantics you need fencing (e.g., refuse writes on the old primary post-partition).

#### Failover

When the cluster detects a primary is down (gossip-based, threshold around 5 seconds), replicas of that primary race to take over.

1. Each replica computes its readiness rank based on replication lag. Most-current goes first.
2. After a small delay, the replica increments its `currentEpoch` and broadcasts a vote request.
3. Other primaries vote yes if (a) they also see the old primary as failed, (b) they have not yet voted in this epoch, and (c) the requesting replica is a legitimate replica of the failed primary.
4. With a majority vote (more than half the primaries), the replica promotes itself.
5. The promoted node claims the slot range and gossips the new ownership.
6. Other replicas of the old primary reattach to the new primary.

This is Raft-like but not exactly Raft; it's a Redis-specific protocol tuned for fast cache failover. Typical failover completes in 5-15 seconds end to end. Production tuning often shortens the timeout for faster failover at the cost of more sensitivity to brief network glitches.

Redis Sentinel (not Redis Cluster) is a separate failover mechanism for non-clustered deployments: a small set of "Sentinel" processes monitors a primary-replica pair and triggers failover when the primary fails. Same idea, different protocol. Used when you want HA but not horizontal sharding.

### 6. Architecture (detailed)

```
                  ┌─────────────────────────────────────────┐
                  │           Application Tier              │
                  │  ┌────────┐ ┌────────┐ ┌────────┐       │
                  │  │ App 1  │ │ App 2  │ │ App N  │       │
                  │  │ ┌────┐ │ │ ┌────┐ │ │ ┌────┐ │       │
                  │  │ │SDK │ │ │ │SDK │ │ │ │SDK │ │       │  Smart client.
                  │  │ │+LRU│ │ │ │+LRU│ │ │ │+LRU│ │       │  Holds slot map.
                  │  │ └─┬──┘ │ │ └─┬──┘ │ │ └─┬──┘ │       │  Optional in-process
                  │  └───┼────┘ └───┼────┘ └───┼────┘       │  LRU for hot keys.
                  └──────┼──────────┼──────────┼────────────┘
                         │          │          │
                         └──────────┼──────────┘
                                    │
              ┌─────────────────────┼─────────────────────────────┐
              │                     │                             │
              ▼                     ▼                             ▼
       ┌────────────┐        ┌────────────┐               ┌────────────┐
       │  Shard 1   │        │  Shard 2   │      ...      │  Shard 12  │
       │  slots     │        │  slots     │               │  slots     │
       │  0-1364    │        │  1365-2729 │               │  15020-    │
       │            │        │            │               │  16383     │
       │ ┌────────┐ │        │ ┌────────┐ │               │ ┌────────┐ │
       │ │Primary │ │        │ │Primary │ │               │ │Primary │ │
       │ │ 256 GB │ │        │ │        │ │               │ │        │ │
       │ └───┬────┘ │        │ └───┬────┘ │               │ └───┬────┘ │
       │     │      │        │     │      │               │     │      │
       │  async repl│        │     │      │               │     │      │
       │     │      │        │     │      │               │     │      │
       │ ┌───▼────┐ │        │ ┌───▼────┐ │               │ ┌───▼────┐ │
       │ │Replica │ │        │ │Replica │ │               │ │Replica │ │
       │ │ same   │ │        │ │        │ │               │ │        │ │
       │ │ AZ     │ │        │ │        │ │               │ │        │ │
       │ └────────┘ │        │ └────────┘ │               │ └────────┘ │
       │            │        │            │               │            │
       │ ┌────────┐ │        │ ┌────────┐ │               │ ┌────────┐ │
       │ │Replica │ │        │ │Replica │ │               │ │Replica │ │
       │ │ cross  │ │        │ │        │ │               │ │        │ │
       │ │ AZ     │ │        │ │        │ │               │ │        │ │
       │ └────────┘ │        │ └────────┘ │               │ └────────┘ │
       └─────┬──────┘        └─────┬──────┘               └─────┬──────┘
             │                     │                            │
             └─────────────────────┼────────────────────────────┘
                                   │
                          ┌────────▼─────────┐
                          │  Cluster Bus      │  Separate TCP port per node.
                          │  (gossip)         │  Every node pings ~5 peers
                          │                   │  every 100ms. Exchanges:
                          │                   │   - PING/PONG
                          │                   │   - node liveness state
                          │                   │   - slot ownership
                          │                   │   - epoch numbers
                          │                   │   - failover vote requests
                          └───────────────────┘

       Out-of-band (per-node):
       ┌──────────────────────────┐
       │  Disk: RDB snapshot file │   Periodic checkpoint of the dataset.
       │        (optional)         │   Used for backup and replica seeding.
       └──────────────────────────┘
       ┌──────────────────────────┐
       │  Disk: AOF append log     │   Every write appended.
       │        (optional)         │   Used for durability.
       └──────────────────────────┘
       ┌──────────────────────────┐
       │  Monitoring agent         │   Scrapes INFO, slowlog, MEMORY USAGE.
       └──────────────────────────┘
```

Why each component:

- **Smart client with optional in-process LRU.** First line of defense against hot keys. A 1000-entry process-local LRU with 5-second TTL turns 500K req/s on one key into 200 req/s reaching the cluster.
- **12 shards.** Sized so per-shard ops/sec stays comfortably under 200K and per-shard memory stays under 200 GB primary.
- **Replicas: one same-AZ, one cross-AZ.** Same-AZ replica catches up in <1ms (low replication lag, used for failover). Cross-AZ replica protects against AZ failure but lags more.
- **Gossip on a separate port.** Cluster bus traffic (port 16379 by default, +10000 from the data port) does not contend with client traffic. Bus messages are small (~100 bytes per ping); aggregate gossip traffic is a few MB/sec across the cluster.
- **Disk persistence optional.** Most caches do not persist. Some do, to seed new replicas faster or to survive a full cluster restart without losing the working set.

### 7. Read and write paths

**Write path (`SET key value EX 60`)**

1. Application calls `client.set("user:42:profile", v, ex=60)`.
2. Client computes `slot = CRC16("user:42:profile") & 16383`. Looks up `slot_map[slot]` to find primary node.
3. Client sends the SET command over its persistent TCP connection to that primary.
4. Primary parses the command, validates the key length, applies it to its in-memory hash table, sets the TTL in the expires dictionary. Updates the `lru` field on the object.
5. Primary appends the command to its replication backlog (and, if AOF is enabled, to the AOF file).
6. Primary returns `+OK` to the client. **Latency so far: ~0.3 ms in same DC.**
7. Asynchronously, replicas pull from the replication backlog and apply the same command. Replication lag at steady state: <1 ms intra-AZ, ~5 ms cross-AZ.
8. If a `WAIT 1 100` follows, the primary blocks until at least one replica acks (or 100ms timeout). Most workloads do not use WAIT.

P99 latency target: 5 ms. Under load, the bottleneck is event-loop contention on the primary (single-threaded). Mitigation: spread keys evenly via hash tags only when needed, do not let big keys hog the event loop.

**Read path (`GET key`)**

1. Application calls `client.get("user:42:profile")`.
2. Client computes the slot and looks up the primary (or a replica if `READONLY` is set on the connection).
3. Client sends GET.
4. Node looks up the key in its hash table. If TTL has expired (passive expiration), deletes the key and returns nil.
5. Updates the `lru` field on the object (so it stays "young" for LRU eviction purposes).
6. Returns the value. **Latency: ~0.2 ms in same DC.**

If the slot has moved:
- Returns `MOVED <slot> <new_node:port>`.
- Client updates its slot map and retries on the new node. **One extra round trip on the first redirected request after a topology change.**

If the slot is mid-migration:
- Returns `ASK <slot> <new_node:port>`.
- Client sends `ASKING` followed by the command to the new node. Does not update the slot map yet (because the migration is not complete).

In-process LRU: before any of the above, the client checks its local LRU cache. Hit returns in microseconds. Miss falls through to the cluster.

### 8. Scaling: add and remove nodes without downtime

**Adding a shard (12 → 13).**

1. Bring up the new primary and its replica. They are empty.
2. Add them to the cluster (`CLUSTER MEET`). They appear in gossip but own no slots.
3. Plan slot migrations. To equalize, each existing shard gives ~1/13 of its slots to the new shard. 16384/13 ≈ 1260 slots to the new shard.
4. Migrate slot by slot. For each slot:
   a. Mark the slot as MIGRATING on the source node and IMPORTING on the destination.
   b. Iterate the keys in that slot. For each key, call `MIGRATE` which copies it to the destination atomically (network ack required) and deletes it from the source.
   c. During this window, the source still serves reads/writes for keys not yet moved, and responds with `ASK` redirects for keys that have moved.
   d. After all keys in the slot are migrated, send `CLUSTER SETSLOT <slot> NODE <new_node>` to all primaries. They gossip the new ownership.
5. After all 1260 slots migrate, the cluster is balanced.

**Throughput cost.** Migration uses bandwidth and CPU on both source and destination. Default migration rate is a few thousand keys/sec per slot, throttleable. For 100 GB of data per source shard moving to a new shard, expect a few hours of migration time. Migration is incremental; cluster keeps serving traffic throughout.

**Client behavior during migration.** Smart clients handle `MOVED`/`ASK` transparently. The application sees slightly elevated tail latency (one extra round-trip per redirected request) but no errors. Dumb clients that ignore redirects see "cache misses" and may overload the database; this is why client choice matters.

**Removing a shard (12 → 11).**

1. Pick the shard to remove. Distribute its slots to the remaining 11 shards (using the same migration mechanism).
2. After migration completes, the shard owns zero slots. It can be shut down.
3. Inform the cluster (`CLUSTER FORGET`). Other nodes remove it from their topology.

**Rebalancing without changing the shard count.** If load is uneven (some shards hotter than others), you can move slots between shards without adding or removing nodes. Same migration mechanism. Useful when traffic patterns drift.

### 9. Reliability

**Persistence options.**

| Mechanism | What it stores | Latency cost | Data loss window |
|-----------|---------------|--------------|------------------|
| None (default for caches) | Nothing | Zero | Everything on process restart |
| RDB snapshot | Point-in-time binary dump of the dataset | Brief CPU + I/O spike during snapshot. Fork-based, so 2x memory transient | Up to the snapshot interval (typical 5 minutes) |
| AOF (append-only file) | Every write command, appended | Small per-write (fsync policy dependent) | `appendfsync always`: 0 (per-write fsync, kills throughput). `everysec`: 1 second. `no`: OS decides (seconds-minutes) |
| RDB + AOF | Both | AOF latency cost + RDB CPU spike | AOF window; RDB is just for faster boot |

**When to use each.**

- **Pure cache, ephemeral data.** No persistence. The database is the source of truth; if cache restarts, repopulate on miss.
- **Cache that takes hours to warm.** RDB snapshots every 5 minutes. After restart, the cluster loads the snapshot and starts serving warm data immediately. Tolerable loss: the last 5 minutes of cache updates, which the application will repopulate.
- **Stateful Redis (Redis as primary store).** AOF with `everysec`. Up to 1 second of writes lost on crash; acceptable for many workloads. Combined with replicas, real-world loss is rare.
- **Strict durability.** AOF with `always`. Throughput drops ~10x. Use only for the few keys that demand it.

For Memcached: no persistence at all. Process restart = empty cache. This is by design; Memcached prizes simplicity and assumes a warmer cache layer or database can absorb the warm-up traffic.

**Failure scenarios.**

- **One node dies.** Replica is promoted (5-15 seconds). Slot range unavailable for that window. Clients see errors; well-built apps treat cache errors as cache misses and fall through to the database. Database briefly sees a fraction of cache traffic for the affected slots.
- **Entire shard (primary + replicas) dies.** Slot range is unavailable until at least one replica returns or the shard is rebuilt. This is the case for replication factor planning: RF=2 (primary + 1 replica) tolerates one failure per shard. RF=3 tolerates two.
- **Network partition.** The smaller side of the partition cannot get a majority of primaries to vote on failover and will refuse to promote replicas. Avoids split-brain. The larger side proceeds normally. When the partition heals, the smaller side reattaches with the larger side as authoritative.
- **Cluster-wide reboot.** Without persistence, the cluster comes back empty. The application's database tier absorbs the cache-miss storm. Sizing the database to handle this is a separate engineering decision; you do not want a 100% cache miss to take down production.
- **Slow consumer (replica falling behind).** Replication backlog overflows. Primary promotes the replica from "online" to "needs full resync." Replica requests an RDB dump from the primary and reloads. During this window, the replica cannot serve reads. Mitigation: make the backlog generous, monitor replication lag, alert on saturation.

### 10. Observability

Metrics that matter, with thresholds:

| Metric | What it tells you | Alert threshold |
|--------|------------------|-----------------|
| `ops_per_sec` per shard | Hot-shard detection | Shard at >2x cluster median |
| `latency.p99` per shard | Slow shard detection | P99 > 10 ms |
| `cache_hit_rate` | Application is using cache effectively | Below 80% (depends on workload) |
| `used_memory` vs `maxmemory` | Eviction headroom | Above 80% |
| `mem_fragmentation_ratio` (rss/used) | Memory waste due to allocator | Above 1.5 |
| `evicted_keys_per_sec` | Eviction pressure | Spike from baseline |
| `expired_keys_per_sec` | Active expiration cost | Spike (could mean TTL stampede) |
| `replication_lag_sec` | Replica freshness | Above 1 second sustained |
| `connected_clients` | Connection leak detection | Spike beyond expected |
| `slowlog_count` | Commands exceeding threshold (default 10ms) | Any sustained appearance |
| `cluster_state` | Cluster health (ok / fail) | Not "ok" |
| `keyspace_hits` / `keyspace_misses` | Cache efficiency | Miss rate climbing |

Slowlog deserves attention. It logs every command that exceeds a threshold (default 10ms). For a single-threaded event-loop server, a 100ms command blocks every other operation for 100ms. The slowlog tells you which keys and commands are the offenders. Almost always: `KEYS *`, `SMEMBERS` on a giant set, `HGETALL` on a giant hash. Cap value sizes and avoid these commands.

Alerting:
- Page on: cluster_state != ok, replication_lag > 5s, P99 latency > 20ms, ops_per_sec drop > 50%.
- Ticket on: hit rate drop > 10pp, evicted_keys spike, fragmentation > 1.8.

### 11. Follow-up answers

**1. Hot key (500K req/s on one key).**

Symptoms: that shard's CPU pegs. P99 for everyone on that shard climbs. Other shards are fine.

Mitigations in order of cheapness:

1. **In-process LRU on application servers.** A 100-entry LRU with 5-second TTL absorbs >99% of the load. The hottest 1% of keys never reach the cluster. This is the single biggest win.
2. **Replica reads.** Direct reads for that key to replicas (using `READONLY` connections). With 2 replicas, you triple the read throughput for that key. Trade: replica reads are slightly stale.
3. **Key splitting (sharding the hot key).** Store the same value under N keys (`hot_key:0`, `hot_key:1`, ..., `hot_key:7`), each on a different slot. Clients pick a random suffix per request. Spreads load across 8 shards. Adds complexity: every write must update all 8 copies. Worth it only for extreme hot keys.
4. **Client-side request coalescing.** If many concurrent requests on one app server want the same expired key, only one fetches from cache+DB; the rest wait. Reduces stampede when the local LRU entry expires.

**2. Big key (sorted set with 2M entries, ZRANGE 0 -1 stalls for 800ms).**

The event loop is single-threaded. An 800ms command blocks every other operation on that node for 800ms. P99 spikes across all keys on that node.

Prevention:
- Cap value size at the application layer. Reject writes that would push a set past 10K entries, or split into multiple smaller sets.
- Use `SCAN`-family commands (`SSCAN`, `HSCAN`, `ZSCAN`) which return small batches per call. Many small commands instead of one giant one.
- Track key size via `MEMORY USAGE key`. Alert on outliers.

Recovery (key already exists, currently causing pain):
- Do not delete it with `DEL`. `DEL` on a 2M-entry sorted set is itself a multi-second blocking call.
- Use `UNLINK` (Redis 4+) which moves the deletion to a background thread. Returns immediately; deallocation happens off the event loop.
- For breaking up the data: iterate with `ZSCAN` in a script, move chunks to new keys, then `UNLINK` the original.

**3. TTL stampede (1M keys all expire at 60s).**

At t=60s, Redis's active expiration job has to delete 1M keys at once. Active expiration samples 20 keys per 100ms; if more than 25% are expired, samples again immediately. With 1M keys all expired, the job runs at full speed for many seconds. CPU dedicated to expiration; client latency spikes.

Also: at expiration, dependent application code (e.g., a "regenerate" path) fires for many keys simultaneously, hammering the source-of-truth database.

Fix without changing the TTL value: add jitter at write time. Instead of `EX 60`, use `EX (60 + random(0, 30))`. Spreads expirations across a window. 1M keys over 30 seconds = 33K expirations/sec, which active expiration handles without spiking.

This is the same insight as cache stampede prevention: never let many things expire simultaneously when they were created simultaneously.

**4. RDB vs AOF.**

| Use case | Recommendation | Why |
|----------|---------------|-----|
| Pure cache, fast warm time | RDB only, snapshot every 5-15 min | Recovers a warm cache after restart in seconds. Some data loss between snapshots is acceptable. |
| Pure cache, slow warm time (hours to fill) | RDB + small AOF (or just RDB) | Same as above; RDB is enough. |
| Cache that doubles as a queue or counter store | AOF with `everysec` | Up to 1 second loss on crash. Tolerable for most use cases. |
| Source-of-truth storage | AOF with `everysec`, plus replicas | Combination gives durability without paying for `appendfsync always`. |
| Tightest durability | AOF with `always` | Each write fsyncs. Throughput drops to thousands of ops/sec per node. Use sparingly. |

Both: snapshot for fast recovery; AOF for the writes since the last snapshot. AOF rewrite compacts the log periodically (replays it into a new file with all redundancies removed).

Neither (Memcached-style): when restart latency is acceptable and the database can handle the warm-up storm. The most common production choice for true cache workloads.

**5. Resharding from 6 to 12 nodes.**

1. Bring up 6 new shards (each with primary + replica). They are empty and gossip in.
2. Plan slot migrations: each of the original 6 shards gives half its slots to one of the new shards. ~1365 slots per migration.
3. Migrate slot by slot using `MIGRATE`. Each slot's keys move atomically; the client gets `ASK` redirects during migration and `MOVED` after.
4. Throttle migration rate so the source and destination are not saturated. Target: ~5% CPU overhead.
5. Migration time depends on data volume per shard. For 200 GB per source shard, half of that (~100 GB) moves at ~50 MB/sec sustained: ~30 minutes per source shard. Done in parallel across the 6 source shards = ~30 minutes total.

During the migration, traffic continues. Cache hit rate may dip slightly (a few percentage points) due to the redirect overhead on the migrating slots. Application sees no errors if the client handles `ASK`/`MOVED`.

**6. Network partition (two nodes isolated).**

Two nodes can talk to each other but not to the other 22 nodes.

If those two nodes are not primaries (or only one is): their primaries on the other side promote replicas. The isolated nodes, when they get only minority votes in any election attempt, refuse to act. Eventually they timeout and shut down (the `cluster-require-full-coverage` flag affects this; defaults vary).

If those two nodes are both primaries of different shards: the larger side declares them dead and promotes their replicas. The isolated primaries, unable to gather a majority for anything, refuse new writes (because they cannot confirm they still own their slots safely). They accept reads of their cached data until they realize they are partitioned and stop.

When the partition heals, the isolated nodes gossip in, see they have stale epochs, and demote themselves to replicas of the new primaries.

No split-brain: a primary on the minority side that kept accepting writes would corrupt the system. The cluster prevents this by requiring majority for failovers and by old primaries deferring to higher-epoch nodes on reunion.

**7. Memory fragmentation ratio 1.8.**

`used_memory_rss / used_memory = 1.8` means the OS sees Redis using 1.8x the memory Redis thinks it is using. The 80% gap is allocator fragmentation: jemalloc has free chunks scattered through arenas, unable to return to the OS because of allocation patterns.

Causes:
- Workloads with widely varying value sizes (small and large mixed). The allocator can't cleanly defrag.
- High churn: many small allocations and frees over time fragment the heap.
- Specifically: keys with TTLs that expire in random order leave gaps.

Mitigations:
- Enable active defragmentation (`activedefrag yes`). Redis runs a background defragmenter that moves objects to consolidate free space. CPU cost: ~5%.
- Restart Redis. Aggressive but effective: new process starts with clean memory layout. Plan failover; restart replica first, fail over, then restart the demoted primary.
- Audit value size distribution. If many sizes, consider normalizing (pad to power-of-two sizes) or using Memcached's slab approach for that specific workload.

Above 1.5 warrants investigation; above 2.0 is action-required.

**8. Cache stampede on key expiry.**

A popular key expires. 10K concurrent requests miss, hit the database, melt it.

Mitigations:
- **Probabilistic early expiration.** A small percentage of reads (well before the TTL) trigger a refresh. One client refreshes; others continue serving the old value. Spreads the refresh across the TTL window. Implementations: XFetch algorithm.
- **Request coalescing.** On miss, the client uses a per-key in-process lock. First request fetches and populates the cache; concurrent requests for the same key wait on the lock and read the populated value. Reduces 10K DB calls to 1.
- **Stale-while-revalidate.** Serve the expired value for a short grace window while one goroutine fetches the new value asynchronously. Application tolerates a few seconds of staleness in exchange for no stampede.
- **Jittered TTLs (prevention).** As above, prevents synchronized expiration.

All four together is the standard cache stampede defense. None alone is sufficient at high request rates.

**9. Inconsistent read after write.**

User issues `SET balance 100`, then `GET balance`, gets the old value.

Most common cause: the GET went to a replica that has not yet received the write. Replication lag is typically <1 ms intra-AZ but can spike during load.

Other causes:
- The SET went to a shard that has just failed over; the read is going to the new primary which has not yet received that write through replication. Failover with `min-replicas-to-write` would prevent this but at the cost of some availability.
- The client has stale slot map and sent SET to one node and GET to another (briefly possible during resharding).

Fixes:
- Route both SET and GET to the primary when read-after-write consistency is required for that specific code path.
- Use a transaction or pipeline that guarantees both commands hit the same node.
- For cluster-wide guarantees, use `WAIT 1 timeout_ms` after the SET so it is acknowledged by at least one replica before GET goes to a replica.
- Accept eventual consistency for caches (the common choice).

**10. Cache hit rate dropped from 95% to 60% overnight.**

Investigation order:

1. **Eviction pressure.** Check `evicted_keys_per_sec`. If high, the cache is full and is evicting hot keys. Cause: working set grew (new feature, more users) or memory shrunk (replica added, RAM used by something else).
2. **TTL change.** Did someone shorten TTLs in a deploy? Even a small reduction can collapse the hit rate on long-tail keys.
3. **Key churn.** Are application servers writing new keys at a higher rate? `DEBUG OBJECT` on samples, look at write rate trends. A code change writing a unique key per request (instead of a shared one) is a classic regression.
4. **Cluster topology change.** Did a node fail and is the cluster in a degraded state? `CLUSTER INFO` will tell you. During degraded operation, redirects increase and some keys may be effectively missing.
5. **Resharding.** A migration that did not complete cleanly can leave slots half-owned. Symptoms: high `MOVED` rate in slowlog, application metrics showing redirect chains.
6. **Application bug.** Cache key construction changed (e.g., new prefix, version bump). The application is reading with a new key name and writing too. Existing keys are stranded.

Mid-level investigates eviction first. Senior also considers application changes and topology. The fastest signal is usually: did anything deploy? Has anyone changed Redis config? If both no, look at the cluster state and the application traffic patterns.

### 12. Trade-offs and what a senior would mention

- **Memcached vs Redis.** Memcached is simpler, multi-threaded per-process (scales to all cores on a single box), and has lower memory overhead per key (slab allocator, no per-key encoding metadata). Redis is single-threaded per process, supports data structures, persistence, scripting, pub/sub, and cluster mode. Use Memcached when you want a pure KV cache with maximum density and the simplest operational model. Use Redis when you want any of: TTL, data structures, atomic counters, persistence, replication, cluster.
- **Sharded client (twemproxy) vs cluster mode.** Twemproxy and similar proxies sit between clients and a static set of Redis nodes; clients are dumb (talk to the proxy, which forwards). Pro: simple client. Con: no automatic failover, no live resharding. Cluster mode (smart client + gossip) handles both but requires a more capable client library. Cluster mode is the default modern choice for new deployments; proxies linger in legacy systems.
- **Why single-threaded event loop, not multi-threaded.** Avoids lock contention. Every command is atomic against the keyspace by construction. Trade: a single slow command stalls everything. The fix is operational discipline (cap value sizes, no `KEYS *`, watch slowlog). Redis 6+ added I/O threads (parses requests and writes responses in parallel) but the actual command execution remains serial.
- **Multi-region cache.** Almost always wrong. Cache should be regional; the database (or its replication) handles multi-region durability. Cross-region cache replication adds latency and consistency headaches with little benefit for a layer that is supposed to be lossy by design.
- **What I would revisit at 10x scale.**
  - Move from a single large cluster to many smaller per-service clusters. Reduces blast radius and lets each service tune eviction, persistence, and replication independently.
  - Adopt a tiered cache (in-process LRU → regional Redis → DB) explicitly, with metrics at each tier.
  - For specific high-traffic keys, consider Memcached or a custom in-memory service. Redis's data structures are not needed for everything.
  - Re-evaluate hash slot count. 16384 was fine at 6-12 shards; at 100 shards, slots get smaller and migration is more granular.

### 13. Common interview mistakes

- **"Redis stores things in memory."** Yes, but the question is about the cluster. Skip the intro; jump to partitioning.
- **Using modulo hashing.** Falls apart on every cluster size change. Always say consistent hashing or hash slots.
- **No mention of virtual nodes.** Consistent hashing without vnodes gives uneven distribution. The interviewer waits for "vnodes" or "hash slots."
- **Confusing expiration and eviction.** TTL is per-key; eviction is global memory pressure. Different mechanisms with different policies.
- **Recommending true LRU.** Memory overhead for a doubly-linked list across 100M keys is significant. Approximated LRU is the right answer, and you should know why.
- **Forgetting the hot key problem.** Will be asked. In-process LRU plus replica reads is the standard answer.
- **Forgetting the big key problem.** Less often asked, but a sign of seniority when you bring it up unprompted. The phrase is "single-threaded event loop blocks on O(N) commands."
- **Synchronous replication everywhere.** Doubles latency for writes that almost never need it. Default async; opt into `WAIT` per-call.
- **No persistence story.** Even if the answer is "none," articulate the choice. The interviewer wants to hear you reasoned about it, not skipped it.
- **Cache stampede handwave.** "We'll use TTLs" is not enough. Mention jitter, request coalescing, stale-while-revalidate.

Hit 8 of these 10 and you are clearly above the bar. The combination of consistent hashing, replication-and-failover narrative, eviction policy choice, and hot-key mitigation is the spine of a strong answer.
