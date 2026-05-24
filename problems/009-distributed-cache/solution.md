## Solution: Distributed Cache (like Redis or Memcached)

### The short version

A distributed cache is a fast in-memory key-value store split across many servers. The data is **sharded** (each server holds a slice) and **replicated** (each shard has a backup copy). A few hard problems are hiding inside:

- How to find a key when servers come and go.
- How to keep replicas close enough to the primary so failover does not lose much.
- How to throw out old data gracefully when memory fills up.
- How to survive the few keys that get all the traffic.

The standard shape is **consistent hashing** (or fixed hash slots) across a cluster of **shards**. Each shard has one **primary** and one or more **replicas**. The replicas copy from the primary in the background. A **gossip protocol** runs across all servers, sharing health and topology info.

Clients are smart. They cache the slot-to-server map in memory and adjust when servers redirect them. Eviction uses **approximated LRU** or **LFU** because true LRU costs too much memory. Persistence is optional and almost always trades latency for durability.

The interesting engineering happens at the edges: the **hot key** that breaks one shard (fixed with in-process caches and replica reads), the **big key** that stalls the event loop (fixed by capping value size at write time), and the **resharding** that has to happen without dropping traffic (incremental slot migration with `ASK` redirects).

---

### 1. The clarifying questions

Covered in `question.md`. The eight that matter: consistency, persistence, value size, traffic shape, eviction, multi-region, client topology, failure tolerance.

The classic miss is skipping persistence and consistency. They change the design fundamentally.

---

### 2. The math, in plain numbers

| Item | Value |
|------|-------|
| Sustained ops/sec | 1 million |
| Peak ops/sec | 3 million |
| Read:write ratio | 10:1 |
| Working set | 1 TB |
| Average value | 1 KB |
| Single-node Redis ceiling | ~150K ops/sec mixed get/set |

What this tells us:

- A 6-shard cluster is too small for peak. A 12-shard cluster is comfortable.
- With 2 copies of the data (RF=2), that is 24 servers total.
- Per server: about 170 GB of primary data, plus replication buffers and connection state, plus 20% encoding overhead. 256 GB machines fit with headroom.
- Network: about 110 MB/sec per server. Well under a 25 Gbps card.
- Connections: about 50K per server from 1000 application servers. Buffer cost about 1 GB per server.

The bottleneck is **per-shard CPU** (single event loop), not memory or network. Cluster size is driven by per-shard ops/sec and blast radius, not raw storage.

---

### 3. The API

Two things to design: the wire protocol and the client-facing semantics.

#### The core commands (Redis style)

| Command | What it does |
|---------|-------------|
| `GET key` | Return value or nil. O(1). |
| `SET key value [EX seconds] [NX\|XX]` | Set with optional TTL. NX = only if missing. XX = only if exists. O(1). |
| `MGET key1 key2 ...` | Batch get. Note: all keys must be on the same shard, or the client has to fan out. |
| `MSET k1 v1 k2 v2 ...` | Batch set. Same shard constraint. |
| `DEL key` | Delete. Returns 1 if existed, 0 otherwise. |
| `EXPIRE key seconds` | Add TTL to an existing key. |
| `TTL key` | Return remaining TTL. -1 means no TTL, -2 means key does not exist. |
| `INCR key` / `INCRBY key n` | Atomic integer increment. Foundation for counters. |
| `EXISTS key` | Cheap existence check. |
| `SCAN cursor [MATCH pattern]` | Cursored iteration. Use this. **Never use `KEYS *`** in production. |

Plus the data-structure commands: `HSET`/`HGET` (hash), `LPUSH`/`LRANGE` (list), `SADD`/`SMEMBERS` (set), `ZADD`/`ZRANGE` (sorted set), `XADD`/`XREAD` (streams). Each is O(1) or O(log N) per element. But operations on entire structures (`SMEMBERS` on a 1M-element set) are O(N) and block the event loop.

**Memcached is a strict subset:** `get`, `set`, `add`, `replace`, `delete`, `incr`, `decr`, `cas` (check-and-set). That is it. No data structures, no scripting, no pub/sub.

#### Cluster-aware semantics

- **Hash tags.** `SET {user:42}:profile ...` and `SET {user:42}:settings ...` hash to the same slot because only the text inside `{...}` is hashed. This lets multi-key operations like `MGET` or `MULTI`/`EXEC` stay on one shard.
- **Multi-key constraint.** `MGET k1 k2` and transactions only work if all keys are on the same shard. The client either uses hash tags or fans out and merges on its side.
- **Pipelining.** Send N commands without waiting for responses, then read N responses. Cuts round-trip cost from N to 1. Critical for throughput. A client can push 500K ops/sec with aggressive pipelining where without it gets 50K.

#### Errors clients must handle

| Response | Meaning | Client action |
|----------|---------|---------------|
| `MOVED <slot> <host:port>` | Slot now lives on another server. | Update slot map. Retry on the new server. |
| `ASK <slot> <host:port>` | Slot is migrating; this specific key is on the new server temporarily. | Send `ASKING` then the command to the new server. Do not update slot map yet. |
| `CLUSTERDOWN` | Cluster is unhealthy (some slots have no owner). | Back off, retry, alert the operator. |
| `LOADING` | Server is loading data from disk after restart. | Retry after a delay. |
| `READONLY` | Tried to write to a replica. | Update slot map. Write to the primary. |
| `OOM` | Eviction disabled and memory is full. | Application must handle the rejection. |

---

### 4. The data model and how memory is laid out

The conceptual model is a flat hash table: keys to values. The interesting engineering is how Redis encodes values to save memory.

**Object header.** Every Redis value has a small header: type tag, encoding, reference count, and a 24-bit "last access time" used for LRU/LFU sampling. About 16 bytes per object.

**Encoding tricks.** Redis picks an encoding based on size and content:

| Type | Small encoding | Large encoding | Switch threshold |
|------|---------------|----------------|------------------|
| String | `embstr` (one allocation, max 44 bytes) | `raw` (separate allocation) | 44 bytes |
| String of small integer | `int` (value sits in the header pointer) | `embstr`/`raw` | Not an integer |
| Hash | `listpack` (flat array of key-value pairs) | `hashtable` | 128 entries or any field > 64 bytes |
| List | `listpack` | `quicklist` (linked list of listpacks) | 128 entries or any element > 64 bytes |
| Set of integers | `intset` (sorted dense array) | `hashtable` | 512 entries or any non-integer |
| Sorted set | `listpack` | `skiplist + hashtable` | 128 entries or any element > 64 bytes |

> **Why this matters.** A 10-field hash stored as `listpack` takes about 200 bytes. The same hash stored as `hashtable` takes about 600 bytes. At 100M small hashes that is 40 GB saved just by staying in `listpack`.

The trade: operations on `listpack` are O(N) (linear scan). Once the threshold trips and the encoding switches to `hashtable`, operations become O(1) but memory triples. Redis picks the encoding at write time and only upgrades (never downgrades).

**For Memcached:** simpler. A **slab allocator** with fixed-size slabs (64 B, 128 B, 256 B, etc.). A 100-byte value goes into a 128-byte slab. Memory-efficient if value sizes cluster nicely. Wasteful if not. The "calcification" problem: once slabs are allocated to a size class, they cannot easily be reassigned. If traffic shifts from many small values to fewer large values, small slabs sit empty.

**Expiration metadata.** Keys with TTL have an entry in a secondary "expires" dictionary, mapping key to expiration timestamp. Active expiration samples from this dictionary, not from the main keyspace.

---

### 5. The core algorithm: consistent hashing + replication

#### Consistent hashing with hash slots

Redis Cluster uses **16384 fixed slots**. Big enough that slots can be assigned at fine granularity (each shard owns 16384/N slots). Small enough that the slot map is a manageable size to gossip (16384 bits = about 2 KB).

```
slot = CRC16(key) & 16383
server = slot_map[slot]
```

The slot map is replicated across all servers via gossip. Clients also cache it. When the map changes (a shard was added, removed, or migrated), clients learn through `MOVED` redirects on the next operation that targets the wrong server.

The hash ring, drawn flat as fixed slots:

```
   slot:    0        4096       8192      12288     16383
            |---------|----------|----------|---------|
   owner:   Shard 1   Shard 2    Shard 3    Shard 4

   key "user:42:profile"
     -> CRC16("user:42:profile") & 16383 = 5798
     -> slot 5798 lives in [4096..8191] -> Shard 2 primary

   key "{user:42}:settings"
     -> CRC16("user:42") & 16383 = 5798     (hash tag forces same slot)
     -> also Shard 2 primary (sits next to profile)
```

#### A GET, end to end

```
  +------------+                                  +--------------+
  |  App + SDK |-- 1. compute slot -------------->|  slot_map    |
  +-----+------+                                  +------+-------+
        |                                                |
        | 2. open TCP to owning primary (cached)         |
        v                                                v
  +-----------------------------------------------------------+
  |  Shard 2 primary                                          |
  |   - looks up key in hash table                            |
  |   - if TTL expired, deletes and returns nil               |
  |   - updates lru / lfu counter                             |
  |   - returns value                                         |
  +-----------------------------------------------------------+
        |
        | on topology change:
        |   responds MOVED 5798 10.0.3.4:6379
        |   SDK updates slot_map[5798] = 10.0.3.4:6379
        |   retries on new server (one extra round-trip, once)
        v
  +------------+
  |  App + SDK |
  +------------+
```

**Why CRC16 and not a fancier hash?** CRC16 is fast (a few cycles per byte) and has well-understood distribution on text keys. xxHash would be slightly faster, but the difference is nanoseconds when the rest of the op is microseconds. A historical "good enough" choice.

#### Hash tags

The hash function only hashes the text inside `{...}` if present:

```
CRC16("{user:42}:profile")  = CRC16("user:42")
CRC16("{user:42}:settings") = CRC16("user:42")
```

Both keys land on the same slot. The application uses this to colocate related keys for multi-key operations.

#### Replication protocol

Each shard has one primary and one or more replicas. Replication is async by default.

**The replication backlog.** The primary keeps a circular buffer (default 1 MB, configurable) of recent writes. When a replica connects, the primary checks if the replica's replication offset is within the backlog.

- If yes, stream forward from that offset (**partial resync**, fast, seconds).
- If no, the replica fell too far behind. The primary does a **full resync**: serializes the entire dataset (RDB snapshot), ships it, then streams new writes. Slow (depends on dataset size; ~30 GB takes a few minutes).

> Size the backlog so brief network blips do not trigger a full resync. 1 MB is too small for most production workloads. 256 MB or even 1 GB is common.

**Replica reads.** Replicas serve reads if the client issues `READONLY` on connect. Application accepts that reads may be slightly stale (usually milliseconds; can spike under replication lag).

**WAIT for sync writes.** A client can issue `WAIT <N> <timeout>` after a write to block until N replicas have acked. This is best-effort, not true synchronous replication. If the primary crashes during the wait, the client gets a timeout but the write may or may not have made it.

#### Failover

When gossip detects a primary down (threshold around 5 seconds), replicas of that primary race to take over:

1. Each replica computes its readiness based on replication lag. Most-current goes first.
2. After a small delay, the replica increments its `currentEpoch` and broadcasts a vote request.
3. Other primaries vote yes if (a) they also see the old primary as failed, (b) they have not voted yet in this epoch, and (c) the requester is a real replica of the failed primary.
4. With a majority vote, the replica promotes itself.
5. The promoted server claims the slot range and gossips the new ownership.
6. Other replicas of the old primary reattach to the new primary.

Raft-like but not exactly Raft. A Redis-specific protocol tuned for fast cache failover. Typical failover completes in 5-15 seconds end to end.

**Redis Sentinel** (not Redis Cluster) is a separate failover mechanism for non-clustered deployments. A small set of Sentinel processes watches a primary-replica pair and triggers failover when the primary fails. Same idea, different protocol. Used when you want high availability but not horizontal sharding.

---

### 6. The architecture, drawn out

```
                +-----------------------------------------+
                |           Application Tier              |
                |  +--------+ +--------+ +--------+       |
                |  | App 1  | | App 2  | | App N  |       |
                |  | +----+ | | +----+ | | +----+ |       |
                |  | |SDK | | | |SDK | | | |SDK | |       |  Smart client.
                |  | |+LRU| | | |+LRU| | | |+LRU| |       |  Holds slot map.
                |  | +-+--+ | | +-+--+ | | +-+--+ |       |  Optional in-process
                |  +---+----+ +---+----+ +---+----+       |  LRU for hot keys.
                +------+----------+----------+------------+
                       |          |          |
                       +----------+----------+
                                  |
            +---------------------+--------------------------+
            |                     |                          |
            v                     v                          v
       +----------+         +----------+               +----------+
       | Shard 1  |         | Shard 2  |   .  .  .     | Shard 12 |
       | slots    |         | slots    |               | slots    |
       | 0-1364   |         | 1365-    |               | 15020-   |
       |          |         | 2729     |               | 16383    |
       | +------+ |         | +------+ |               | +------+ |
       | |Prim. | |         | |Prim. | |               | |Prim. | |
       | |256GB | |         | |      | |               | |      | |
       | +--+---+ |         | +--+---+ |               | +--+---+ |
       |    | async         |    |     |               |    |     |
       |    | repl          |    |     |               |    |     |
       | +--v---+ |         | +--v---+ |               | +--v---+ |
       | |Repl. | |         | |Repl. | |               | |Repl. | |
       | |same  | |         | |      | |               | |      | |
       | |AZ    | |         | |      | |               | |      | |
       | +------+ |         | +------+ |               | +------+ |
       |          |         |          |               |          |
       | +------+ |         | +------+ |               | +------+ |
       | |Repl. | |         | |Repl. | |               | |Repl. | |
       | |cross | |         | |      | |               | |      | |
       | |AZ    | |         | |      | |               | |      | |
       | +------+ |         | +------+ |               | +------+ |
       +----+-----+         +----+-----+               +----+-----+
            |                    |                          |
            +--------------------+--------------------------+
                                 |
                       +---------v----------+
                       |  Cluster Bus        |  Separate TCP port per server.
                       |  (gossip)           |  Every server pings ~5 peers
                       |                     |  every 100ms. Shares:
                       |                     |   - PING/PONG
                       |                     |   - liveness state
                       |                     |   - slot ownership
                       |                     |   - epoch numbers
                       |                     |   - failover vote requests
                       +---------------------+

       Per server, out of band:
       +---------------------------+
       |  Disk: RDB snapshot file  |   Periodic checkpoint of the dataset.
       |  (optional)               |   Used for backup and replica seeding.
       +---------------------------+
       +---------------------------+
       |  Disk: AOF append log     |   Every write appended.
       |  (optional)               |   Used for durability.
       +---------------------------+
       +---------------------------+
       |  Monitoring agent         |   Scrapes INFO, slowlog, MEMORY USAGE.
       +---------------------------+
```

**Why each piece:**

- **Smart client with optional in-process LRU.** First line of defense against hot keys. A 1000-entry process-local LRU with a 5-second TTL turns 500K req/s on one key into 200 req/s reaching the cluster.
- **12 shards.** Sized so per-shard ops/sec stays comfortably under 200K and per-shard memory under 200 GB primary.
- **Replicas: one same-AZ, one cross-AZ.** Same-AZ catches up in under 1ms (low lag, used for failover). Cross-AZ protects against AZ failure but lags more.
- **Gossip on a separate port.** Cluster bus traffic (port 16379 by default, data port + 10000) does not compete with client traffic. Bus messages are small (about 100 bytes per ping); aggregate gossip is a few MB/sec across the cluster.
- **Disk persistence is optional.** Most caches do not persist. Some do, to seed new replicas faster or to survive a full cluster restart without losing the working set.

---

### 7. Read and write paths

#### Write path: `SET key value EX 60`

1. Application calls `client.set("user:42:profile", v, ex=60)`.
2. Client computes `slot = CRC16("user:42:profile") & 16383`. Looks up `slot_map[slot]` to find the primary.
3. Client sends SET over its persistent TCP connection to that primary.
4. Primary parses the command, validates key length, applies to its in-memory hash table, sets TTL in the expires dictionary. Updates the `lru` field on the object.
5. Primary appends the command to its replication backlog (and to the AOF file, if AOF is enabled).
6. Primary returns `+OK`. Latency so far: about 0.3 ms in the same data center.
7. In the background, replicas pull from the backlog and apply. Replication lag at steady state: under 1 ms intra-AZ, about 5 ms cross-AZ.
8. If `WAIT 1 100` follows, the primary blocks until at least one replica acks (or 100ms timeout). Most workloads do not use WAIT.

P99 target: 5 ms. Under load, the bottleneck is event-loop contention on the primary (single-threaded). The mitigation is keep keys spread evenly and never let big keys hog the event loop.

#### Read path: `GET key`

1. Application calls `client.get("user:42:profile")`.
2. Client computes the slot, looks up the primary (or a replica if `READONLY` is set on the connection).
3. Client sends GET.
4. Server looks up the key in its hash table. If the TTL has expired (passive expiration), deletes the key and returns nil.
5. Updates the `lru` field (so the key stays "young" for LRU eviction).
6. Returns the value. Latency: about 0.2 ms in the same DC.

**If the slot has moved:**

- Server returns `MOVED <slot> <new_server>`.
- Client updates its slot map and retries on the new server. One extra round-trip on the first redirected request after a topology change.

**If the slot is mid-migration:**

- Server returns `ASK <slot> <new_server>`.
- Client sends `ASKING` followed by the command to the new server. Does not update the slot map yet (migration is not complete).

**In-process LRU.** Before any of the above, the client checks its local LRU. Hit returns in microseconds. Miss falls through to the cluster.

---

### 8. Scaling: add and remove servers without downtime

#### Adding a shard (12 to 13)

1. Bring up the new primary and its replica. Empty.
2. Add them to the cluster (`CLUSTER MEET`). They appear in gossip but own no slots.
3. Plan slot migrations. To even things out, each existing shard gives about 1/13 of its slots to the new shard. 16384/13 ≈ 1260 slots to the new shard.
4. Migrate slot by slot. For each slot:
   - Mark the slot as MIGRATING on the source and IMPORTING on the destination.
   - Iterate the keys in that slot. For each, call `MIGRATE`, which copies it to the destination atomically (network ack required) and deletes it from the source.
   - During this window, the source still serves reads/writes for keys not yet moved, and responds with `ASK` redirects for keys that have moved.
   - After all keys in the slot are migrated, send `CLUSTER SETSLOT <slot> NODE <new_node>` to all primaries. They gossip the new ownership.
5. After all 1260 slots migrate, the cluster is balanced.

**Throughput cost.** Migration uses bandwidth and CPU on both source and destination. Default migration rate is a few thousand keys/sec per slot, configurable. For 100 GB per source shard moving to a new shard, expect a few hours of migration. Migration is incremental; the cluster keeps serving traffic throughout.

**Client behavior during migration.** Smart clients handle `MOVED`/`ASK` transparently. The application sees slightly higher tail latency (one extra round-trip per redirected request) but no errors. Dumb clients that ignore redirects see "cache misses" and may overload the database. Client choice matters.

#### Removing a shard (12 to 11)

1. Pick the shard. Move its slots to the remaining 11 (same mechanism).
2. After migration completes, the shard owns zero slots. Shut it down.
3. Tell the cluster (`CLUSTER FORGET`). Other servers remove it from their topology.

#### Rebalancing without changing shard count

If load is uneven (some shards hotter than others), move slots between shards without adding or removing servers. Same mechanism. Useful when traffic patterns drift.

---

### 9. Reliability

#### Persistence options

| Mechanism | What it saves | Latency cost | Data loss window |
|-----------|--------------|--------------|------------------|
| **None** (default for caches) | Nothing | Zero | Everything on process restart |
| **RDB snapshot** | Point-in-time binary dump of the dataset | Brief CPU + I/O spike during snapshot. Fork-based, so 2x memory transient. | Up to the snapshot interval (typical 5 min) |
| **AOF (append-only file)** | Every write command, appended | Small per-write (depends on fsync policy) | `appendfsync always`: 0 (per-write fsync, kills throughput). `everysec`: 1 second. `no`: OS decides (seconds to minutes). |
| **RDB + AOF** | Both | AOF latency cost + RDB CPU spike | AOF window. RDB is just for faster boot. |

**When to use each:**

- **Pure cache, ephemeral data.** No persistence. The database is the source of truth. If cache restarts, repopulate on miss.
- **Cache that takes hours to warm.** RDB snapshots every 5 minutes. After restart, load the snapshot and start serving warm immediately. Acceptable loss: the last 5 minutes of cache updates, which the application will repopulate naturally.
- **Stateful Redis (Redis as primary store).** AOF with `everysec`. Up to 1 second of writes lost on crash; fine for many workloads. Combined with replicas, real-world loss is rare.
- **Strict durability.** AOF with `always`. Throughput drops about 10x. Use only for the few keys that demand it.

**For Memcached:** no persistence. Process restart = empty cache. By design. Memcached assumes a warmer cache layer or the database absorbs the warm-up traffic.

#### Failure scenarios

- **One server dies.** Replica is promoted in 5-15 seconds. Slot range unavailable for that window. Clients see errors; well-built apps treat cache errors as cache misses and fall through to the database. Database briefly sees a fraction of cache traffic for the affected slots.

- **Entire shard (primary + all replicas) dies.** Slot range unavailable until at least one replica returns or the shard is rebuilt. RF=2 (primary + 1 replica) tolerates one failure per shard. RF=3 tolerates two.

- **Network partition.** The smaller side cannot get a majority of primaries to vote on failover and refuses to promote replicas. Avoids split-brain. The larger side proceeds normally. When the partition heals, the smaller side reattaches with the larger as authoritative.

- **Cluster-wide reboot.** Without persistence, the cluster comes back empty. The application's database tier absorbs the cache-miss storm. Sizing the database to handle this is a separate decision; you do not want a 100% cache miss to take down production.

- **Slow consumer (replica falling behind).** Replication backlog overflows. Primary promotes the replica from "online" to "needs full resync." Replica requests an RDB dump and reloads. During this window the replica cannot serve reads. Mitigation: make the backlog generous, monitor replication lag, alert on saturation.

---

### 10. Observability

| Metric | What it tells you | Alert threshold |
|--------|------------------|-----------------|
| `ops_per_sec` per shard | Hot-shard detection | Shard at >2x cluster median |
| `latency.p99` per shard | Slow shard detection | P99 > 10 ms |
| `cache_hit_rate` | App using cache effectively | Below 80% (depends on workload) |
| `used_memory` vs `maxmemory` | Eviction headroom | Above 80% |
| `mem_fragmentation_ratio` (rss/used) | Memory waste from the allocator | Above 1.5 |
| `evicted_keys_per_sec` | Eviction pressure | Spike from baseline |
| `expired_keys_per_sec` | Active expiration cost | Spike (could mean TTL stampede) |
| `replication_lag_sec` | Replica freshness | Above 1 second sustained |
| `connected_clients` | Connection leak detection | Spike beyond expected |
| `slowlog_count` | Commands past threshold (default 10ms) | Any sustained appearance |
| `cluster_state` | Cluster health (ok / fail) | Not "ok" |
| `keyspace_hits` / `keyspace_misses` | Cache efficiency | Miss rate climbing |

**Slowlog deserves attention.** Every command past a threshold (default 10ms) gets logged. For a single-threaded event-loop server, a 100ms command blocks every other operation for 100ms. The slowlog names the offenders. Almost always: `KEYS *`, `SMEMBERS` on a giant set, `HGETALL` on a giant hash. Cap value sizes and avoid these commands.

**Alerting:**

- **Page on:** `cluster_state != ok`, `replication_lag > 5s`, P99 latency > 20ms, ops/sec drop > 50%.
- **Ticket on:** hit rate drop > 10 percentage points, evicted_keys spike, fragmentation > 1.8.

---

### 11. Follow-up answers

**1. Hot key (500K req/s on one key).**

Symptoms: that shard's CPU pegs. P99 for everyone on that shard climbs. Other shards fine.

Mitigations in order of cheapness:

1. **In-process LRU on application servers.** A 100-entry LRU with a 5-second TTL absorbs over 99% of the load. The hottest 1% of keys never reach the cluster. The single biggest win.
2. **Replica reads.** Direct reads for that key to replicas (using `READONLY` connections). With 2 replicas, triple the read throughput for that key. Trade: slightly stale.
3. **Key splitting (sharding the hot key).** Store the same value under N keys (`hot_key:0`, `hot_key:1`, ..., `hot_key:7`), each on a different slot. Clients pick a random suffix per request. Spreads load across 8 shards. Adds complexity: every write must update all 8 copies. Worth it only for extreme hot keys.
4. **Client-side request coalescing.** If many concurrent requests on one app server want the same expired key, only one fetches from cache+DB; the rest wait. Reduces stampede when the local LRU entry expires.

**2. Big key (sorted set with 2M entries, ZRANGE 0 -1 stalls for 800ms).**

The event loop is single-threaded. An 800ms command blocks every other operation on that server for 800ms. P99 spikes across all keys on that server.

*Prevention:*

- Cap value size at the application layer. Reject writes that would push a set past 10K entries, or split into multiple smaller sets.
- Use `SCAN`-family commands (`SSCAN`, `HSCAN`, `ZSCAN`) which return small batches per call. Many small commands instead of one giant one.
- Track key size via `MEMORY USAGE key`. Alert on outliers.

*Recovery (key already exists, currently causing pain):*

- Do not delete with `DEL`. `DEL` on a 2M-entry sorted set is itself a multi-second blocking call.
- Use `UNLINK` (Redis 4+) which moves the deletion to a background thread. Returns immediately; deallocation happens off the event loop.
- For breaking up the data: iterate with `ZSCAN` in a script, move chunks to new keys, then `UNLINK` the original.

**3. TTL stampede (1M keys all expire at 60s).**

At t=60s, the active expiration job has to delete 1M keys at once. Active expiration samples 20 keys per 100ms; if more than 25% are expired, samples again immediately. With 1M keys all expired, the job runs at full speed for many seconds. CPU dedicated to expiration; client latency spikes.

Also: at expiration, dependent application code (a "regenerate" path) fires for many keys at once, hammering the source-of-truth database.

*Fix without changing the TTL value:* add **jitter** at write time. Instead of `EX 60`, use `EX (60 + random(0, 30))`. Spreads expirations across a window. 1M keys over 30 seconds = 33K expirations/sec, which active expiration handles without spiking.

Same idea as cache stampede prevention: never let many things expire simultaneously when they were created simultaneously.

**4. RDB vs AOF.**

| Use case | Recommendation | Why |
|----------|---------------|-----|
| Pure cache, fast warm time | RDB only, snapshot every 5-15 min | Recovers a warm cache after restart in seconds. Some data loss between snapshots is acceptable. |
| Pure cache, slow warm time (hours to fill) | RDB + small AOF (or just RDB) | Same as above; RDB is enough. |
| Cache that doubles as a queue or counter store | AOF with `everysec` | Up to 1 second loss on crash. Tolerable for most use cases. |
| Source-of-truth storage | AOF with `everysec`, plus replicas | Durability without paying for `appendfsync always`. |
| Tightest durability | AOF with `always` | Each write fsyncs. Throughput drops to thousands of ops/sec per server. Use sparingly. |

**Both:** snapshot for fast recovery; AOF for the writes since the last snapshot. AOF rewrite compacts the log periodically (replays it into a new file with all redundancies removed).

**Neither (Memcached-style):** when restart latency is acceptable and the database can handle the warm-up storm. The most common production choice for true cache workloads.

**5. Resharding from 6 to 12 servers.**

1. Bring up 6 new shards (each with primary + replica). Empty and gossip in.
2. Plan slot migrations: each of the original 6 gives half its slots to one of the new shards. About 1365 slots per migration.
3. Migrate slot by slot using `MIGRATE`. Each slot's keys move atomically; clients get `ASK` redirects during migration and `MOVED` after.
4. Throttle migration so source and destination are not saturated. Target: about 5% CPU overhead.
5. Migration time depends on data per shard. For 200 GB per source shard, half (about 100 GB) moves at about 50 MB/sec sustained: about 30 minutes per source shard. In parallel across the 6 sources = about 30 minutes total.

Traffic continues throughout. Cache hit rate may dip slightly (a few percentage points) from redirect overhead on migrating slots. Application sees no errors if the client handles `ASK`/`MOVED`.

**6. Network partition (two servers isolated).**

Two servers can talk to each other but not to the other 22.

If those two are not primaries (or only one is): primaries on the other side promote replicas. The isolated servers, when they get only minority votes in any election attempt, refuse to act. Eventually they time out and shut down (the `cluster-require-full-coverage` flag affects this; defaults vary).

If both are primaries of different shards: the larger side declares them dead and promotes their replicas. The isolated primaries, unable to gather a majority for anything, refuse new writes (they cannot confirm they still own their slots safely). They accept reads of their cached data until they realize they are partitioned and stop.

When the partition heals, the isolated servers gossip in, see their epochs are stale, demote themselves to replicas of the new primaries.

**No split-brain.** A primary on the minority side that kept accepting writes would corrupt the system. The cluster prevents this by requiring majority for failovers and by old primaries deferring to higher-epoch servers on reunion.

**7. Memory fragmentation ratio 1.8.**

`used_memory_rss / used_memory = 1.8` means the OS sees Redis using 1.8x the memory Redis thinks it is using. The 80% gap is **allocator fragmentation**: jemalloc has free chunks scattered through arenas that it cannot return to the OS because of allocation patterns.

*Causes:*

- Workloads with widely varying value sizes (small and large mixed). The allocator cannot cleanly defragment.
- High churn: many small allocations and frees over time fragment the heap.
- Keys with TTLs that expire in random order leave gaps.

*Mitigations:*

- Enable active defragmentation (`activedefrag yes`). Redis runs a background defragmenter that moves objects to consolidate free space. CPU cost: about 5%.
- Restart Redis. Aggressive but effective: new process starts with a clean memory layout. Plan failover; restart the replica first, fail over, then restart the demoted primary.
- Audit value size distribution. If many sizes, consider normalizing (pad to power-of-two sizes) or using Memcached's slab approach for that specific workload.

Above 1.5 warrants investigation. Above 2.0 is action-required.

**8. Cache stampede on key expiry.**

A popular key expires. 10K concurrent requests miss and hit the database. The database melts.

*Mitigations:*

- **Probabilistic early expiration.** A small percentage of reads (well before the TTL) trigger a refresh. One client refreshes; others continue serving the old value. Spreads the refresh across the TTL window. Implementations: XFetch algorithm.
- **Request coalescing.** On miss, the client uses a per-key in-process lock. The first request fetches and populates the cache; concurrent requests for the same key wait on the lock and read the populated value. Reduces 10K DB calls to 1.
- **Stale-while-revalidate.** Serve the expired value for a short grace window while one goroutine fetches the new value asynchronously. Application tolerates a few seconds of staleness in exchange for no stampede.
- **Jittered TTLs (prevention).** As above, prevents synchronized expiration.

All four together is the standard defense. None alone is sufficient at high request rates.

**9. Inconsistent read after write.**

User issues `SET balance 100`, then `GET balance`, gets the old value.

Most common cause: the GET went to a replica that has not yet received the write. Replication lag is usually under 1 ms intra-AZ but can spike under load.

Other causes:

- The SET went to a shard that just failed over; the read goes to the new primary which has not yet received that write through replication. Failover with `min-replicas-to-write` would prevent this but at the cost of some availability.
- The client has a stale slot map and sent SET to one server and GET to another (briefly possible during resharding).

*Fixes:*

- Route both SET and GET to the primary when read-after-write consistency is required for that code path.
- Use a transaction or pipeline that keeps both commands on the same server.
- For cluster-wide guarantees, use `WAIT 1 timeout_ms` after the SET so it is acked by at least one replica before GET goes to a replica.
- Accept eventual consistency for caches (the common choice).

**10. Cache hit rate dropped from 95% to 60% overnight.**

Investigation order:

1. **Eviction pressure.** Check `evicted_keys_per_sec`. If high, the cache is full and evicting hot keys. Cause: working set grew (new feature, more users) or memory shrunk (replica added, RAM used by something else).
2. **TTL change.** Did someone shorten TTLs in a deploy? Even a small reduction can collapse hit rate on long-tail keys.
3. **Key churn.** Are app servers writing new keys at a higher rate? `DEBUG OBJECT` on samples, look at write rate trends. A code change writing a unique key per request (instead of a shared one) is a classic regression.
4. **Cluster topology change.** Did a server fail and is the cluster degraded? `CLUSTER INFO` will say. During degraded operation, redirects increase and some keys may be effectively missing.
5. **Resharding.** A migration that did not complete cleanly can leave slots half-owned. Symptoms: high `MOVED` rate in slowlog, application metrics showing redirect chains.
6. **Application bug.** Cache key construction changed (new prefix, version bump). The application is reading with a new key name and writing too. Existing keys are stranded.

Mid-level investigates eviction first. Senior also considers application changes and topology. Fastest signal is usually: did anything deploy? Has anyone changed Redis config? If both no, look at cluster state and application traffic patterns.

---

### 12. Trade-offs worth saying out loud

**Memcached vs Redis.** Memcached is simpler, multi-threaded per process (scales to all cores on a single box), and has lower memory overhead per key (slab allocator, no per-key encoding metadata). Redis is single-threaded per process and supports data structures, persistence, scripting, pub/sub, and cluster mode. Use Memcached when you want a pure KV cache with maximum density and the simplest operational model. Use Redis when you want any of: TTL, data structures, atomic counters, persistence, replication, cluster mode.

**Sharded client (twemproxy) vs cluster mode.** Twemproxy and similar proxies sit between clients and a static set of Redis servers. Clients are dumb (talk to the proxy, which forwards). Pro: simple client. Con: no automatic failover, no live resharding. Cluster mode (smart client + gossip) handles both but requires a more capable client library. Cluster mode is the default modern choice for new deployments; proxies linger in legacy systems.

**Why single-threaded event loop?** Avoids lock contention. Every command is atomic against the keyspace by construction. Trade: a single slow command stalls everything. The fix is operational discipline (cap value sizes, no `KEYS *`, watch slowlog). Redis 6+ added I/O threads (parse requests and write responses in parallel) but command execution stays serial.

**Multi-region cache.** Almost always wrong. Cache should be regional; the database (or its replication) handles multi-region durability. Cross-region cache replication adds latency and consistency headaches with little benefit for a layer that is supposed to be lossy by design.

**What you would revisit at 10x scale:**

- Move from a single large cluster to many smaller per-service clusters. Reduces blast radius and lets each service tune eviction, persistence, and replication independently.
- Adopt a tiered cache (in-process LRU → regional Redis → DB) explicitly, with metrics at each tier.
- For specific high-traffic keys, consider Memcached or a custom in-memory service. Redis's data structures are not needed for everything.
- Re-evaluate hash slot count. 16384 was fine at 6-12 shards. At 100 shards, slots get smaller and migration is more granular.

---

### 13. Common mistakes

- **"Redis stores things in memory."** Yes, but the question is the cluster. Skip the intro; go to partitioning.
- **Using modulo hashing.** Falls apart on every cluster size change. Always say consistent hashing or hash slots.
- **No mention of virtual nodes.** Consistent hashing without vnodes gives uneven distribution. The interviewer waits for "vnodes" or "hash slots."
- **Confusing expiration and eviction.** TTL is per-key; eviction is global memory pressure. Different mechanisms, different policies.
- **Recommending true LRU.** Memory overhead for a doubly-linked list across 100M keys is huge. Approximated LRU is the right answer, and you should know why.
- **Forgetting the hot key problem.** Will be asked. In-process LRU plus replica reads is the standard answer.
- **Forgetting the big key problem.** Less often asked, but a sign of seniority if you bring it up unprompted. The phrase is "single-threaded event loop blocks on O(N) commands."
- **Synchronous replication everywhere.** Doubles latency for writes that almost never need it. Default async; opt into `WAIT` per-call.
- **No persistence story.** Even if the answer is "none," explain the choice. The interviewer wants to hear that you reasoned about it, not that you skipped it.
- **Cache stampede handwave.** "We'll use TTLs" is not enough. Mention jitter, request coalescing, stale-while-revalidate.

If you can name 7 of these 10 without prompting, you are clearly above the bar. The spine of a strong answer is: consistent hashing, replication-and-failover narrative, eviction policy choice, and hot-key mitigation.
