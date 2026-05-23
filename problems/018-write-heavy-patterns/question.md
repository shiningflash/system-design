---
id: 18
title: Design a Write-Heavy System (Patterns Walkthrough)
category: Patterns
topics: [batching, async, partitioning, append-only, queues]
difficulty: Easy
solution: solution.md
---

## Scene

You walk in. The interviewer skips the small talk:

> *"I have an audit logging service. Today it does 100 events per second. Single Postgres. Everything works. My PM just told me we are going to log every user interaction across the company. About 1 million events per second. Walk me through what you do, in order."*

This is not a question about a finished architecture. It is a question about how you *grow* a system from "one Postgres handles it" to "a million events per second." The interviewer wants to see the toolkit come out one piece at a time: buffering, batching, async pipelines, partitioning, append-only logs, sharding by key, and the moments where you give up on strong consistency.

A write-heavy problem is the mirror image of a read-heavy one. The cost is dominated by ingestion. Each event is small, queries are simple or batched, and the storage engine's job is to absorb writes faster than a normal OLTP DB ever could. Recognizing the shape early changes every later decision.

Candidates who blurt "use Kafka and Cassandra" miss the point. The interviewer wants you to reach for each tool *at the moment its predecessor breaks*, not all at once.

## Step 1: Clarify before you design

Take five minutes. Do not draw anything yet. The naive instinct is "just send it to Kafka." That is wrong until you have answers to the questions below. Aim for about eight questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

The first one is durability. "If we drop 0.1% of events during a node failure, is that acceptable?" Audit logging for SOX has zero tolerance. Product telemetry happily drops 1% for cheaper infrastructure. That single answer decides whether you need `acks=all` on Kafka, a transactional outbox, or fire-and-forget UDP.

Then lag. "When an event is ingested, when does it have to be visible to queries? 100ms? 10s? 5 minutes?" Real-time visibility forces synchronous indexing. Five-minute lag lets you batch into Parquet and pay a fraction of the cost.

Ordering matters more than people expect. "Do events need to be globally ordered, ordered per user, or unordered?" Global ordering is a single-writer constraint. Per-key ordering is partitioned-by-key. Unordered is the cheapest and most parallel.

Read pattern shapes the storage engine. "Do consumers query individual events by ID, or do they aggregate?" Point lookups want an LSM with a primary key. Scans and aggregations want columnar layout like Parquet or ClickHouse.

Retention drives the tiered storage decision. "30 days hot, 7 years cold? Forever?" Time-based partitions are trivially droppable; that fact alone often picks the partition scheme.

Event size and shape: a 200-byte event is a completely different system from a 50KB event. Fixed schema lets you use columnar formats. Arbitrary JSON forces store-raw or pay-for-schema-on-read.

Burstiness: "Is 1M/sec sustained or peak? Black Friday multiplier? Log storm risk?" A 5x burst is normal. Some systems see 50x bursts from log loops. Buffer sizing follows from this.

Finally, delivery semantics: at-most-once (lose on failure), at-least-once (may dup), or exactly-once. Exactly-once is expensive and rarely worth the cost. At-least-once with idempotent consumers is the common answer. You should know why.

One more question that separates strong candidates: "What does the *consumer* do with these events?" If it is a dashboard, you optimize for aggregation. If it is fraud detection, you optimize for low-latency streaming. If it is compliance archive, you optimize for cheap durable storage. The downstream changes everything upstream.

</details>

## Step 2: Capacity

The interviewer hands you the numbers. 1M events per second, sustained, 3x peak. Events about 500 bytes each. Retention: 90 days hot, 7 years cold. No global ordering required; per-`user_id` ordering is nice. Acceptable lag from write to readable: 10 seconds.

Compute on paper before peeking:

1. Sustained ingest bandwidth (bytes/sec)
2. Peak ingest bandwidth
3. Daily event volume
4. 90-day hot storage requirement
5. 7-year cold storage requirement (assume 5x compression for cold)
6. Queue depth if downstream stalls for 5 minutes
7. Network connections at the ingest tier if each producer holds an open connection

<details>
<summary><b>Reveal: the math</b></summary>

Sustained bandwidth: 1M events/sec * 500 bytes = 500 MB/sec, which is **4 Gbps sustained**. A single 10G NIC saturates around 80% utilization. You need multiple ingest nodes from the start, not because of CPU, but because of NIC throughput.

Peak: 3x sustained = 3M events/sec = 1.5 GB/sec = **12 Gbps peak**. Now you are looking at multiple machines per region just to terminate TLS and accept the bytes.

Daily volume: 1M * 86400 = **86.4 billion events/day**. At 500 bytes, that is **43 TB/day raw**.

90-day hot storage: 43 TB * 90 = **~3.9 PB hot**. Not a single machine. This is a 10-to-30 node Cassandra cluster, or a heavy-RF Kafka cluster with long retention, or a sharded Postgres farm if you are a masochist.

7-year cold storage: 43 TB * 365 * 7 / 5 (compression) = **~22 PB cold** in S3 / GCS / Azure Blob, stored as Parquet. At $0.023/GB/month for standard storage, that is about $500k/month just for the bits. Glacier Deep Archive cuts it to 1/20 the price if access is rare.

Queue depth on a 5-minute stall: 1M events/sec * 300 sec = **300M events queued**. At 500 bytes, that is **150 GB of buffered data**. A Kafka cluster with 7-day retention handles this trivially. An in-memory queue does not.

Network connections: if each producer holds an open TLS connection (say 100k producer apps across the company) and you have 10 ingest nodes per region, each ingest node is terminating ~10k concurrent TLS connections. Workable, but the ingest tier must be tuned for many idle connections (file descriptor limits, kernel `net.core.somaxconn`, and so on).

What the math is telling you: the system is small in event count (86B/day is large but not crazy) and huge in aggregate bytes. Bandwidth and storage dominate everything. Synchronous per-event writes are physically impossible at this scale. A single Postgres tops out around 10k-50k writes/sec depending on hardware and write shape. We are 20-100x past that on a single shard. Most of the architecture exists to *spread* the write load across many nodes and to *batch* small writes into bigger ones that storage engines can handle efficiently.

</details>

## Step 3: The write pipeline

Here is the conceptual evolution of a write path. Each stage handles roughly 10x the throughput of the previous one. Fill in the limits.

```
Stage A:  producer  --INSERT-->  Postgres                    limit: [ ? ] writes/sec
                                                              fails because: [ ? ]

Stage B:  producer  --INSERT-->  app  --buffer-->  Postgres   limit: [ ? ] writes/sec
                                  (in-memory                  fails because: [ ? ]
                                   flush every
                                   100ms)

Stage C:  producer  --HTTP-->   ingest  -->  [ ? ]   -->  batch writer  --COPY-->  storage
                                                                          (Postgres /
                                                                           Cassandra)
                                  limit: [ ? ] writes/sec
                                  fails because: [ ? ]

Stage D:  producer  -->  regional ingest  -->  regional Kafka  -->  stream processor  -->  [ ? ]
                                                  (partitioned by key)         (Flink /
                                                                                Spark)
                                  limit: 1M+ writes/sec
                                  fails because: [ ? ]
```

<details>
<summary><b>Reveal: the four stages with limits</b></summary>

```
Stage A:  producer  --INSERT-->  Postgres                    limit: ~1k writes/sec
                                                              fails because: WAL flush + index
                                                                            updates per row;
                                                                            disk fsync per commit

Stage B:  producer  --INSERT-->  app  --buffer-->  Postgres   limit: ~10k writes/sec
                                  (in-memory                  fails because: single Postgres
                                   flush every                                still bottlenecks
                                   100ms,                                     on writes; buffer
                                   COPY in                                    is lost on crash
                                   batches of
                                   1000)

Stage C:  producer  --HTTP-->   ingest  -->  Kafka  -->  batch writer  --COPY-->  Cassandra
                                                                                  (or sharded
                                                                                   Postgres)
                                  limit: ~100k writes/sec
                                  fails because: single Cassandra cluster
                                                or single Kafka topic
                                                hits hot partition limits;
                                                consumer lag grows

Stage D:  producer  -->  regional ingest  -->  regional Kafka  -->  Flink  -->  Cassandra (hot)
                                                  (partitioned by                + S3 Parquet
                                                   event_type +                  (cold archive)
                                                   tenant_id)
                                  limit: 1M+ writes/sec
                                  fails because: cross-region replication
                                                lag; cold-tier query
                                                latency; bursts past
                                                Kafka cluster capacity
```

What each transition buys you:

A to B (buffer + batch) stops you from paying fsync cost per event. One fsync now amortizes 1000 events. 10x throughput, same hardware. Risk: in-memory buffer is lost on crash.

B to C (durable queue) decouples producer rate from consumer rate. Producers cannot overwhelm storage; they fill the queue. Storage drains it at its own pace. Cassandra's LSM-tree absorbs sequential writes far better than Postgres's B-tree. Another 10x.

C to D (partitioning + stream processing) scales horizontally with no single bottleneck. Each Kafka partition has its own consumer. Each Cassandra node owns a slice of the key space. A stream processor like Flink does enrichment and routing. Cold tier on S3 Parquet is bottomless and cheap. 10x more, and you can keep going by adding partitions.

What each transition costs:

A to B: lose up to one buffer's worth of events on crash. About 100ms * 10k events/sec = 1000 events lost in the worst case.

B to C: end-to-end lag jumps from milliseconds to seconds. Events are not queryable until the batch writer has flushed them.

C to D: strong consistency is gone. Cross-region replication is eventually consistent. Hot/cold tier splits queries that span both into a join.

The exercise: name the limit before you reach for the next tool.

</details>

## Step 4: Incomplete diagram

Here is the full pipeline for the 1M events/sec design with five placeholders. Fill them in.

```
                                +------------------+
   producer apps  --HTTPS-->   |   [ ? ]          |  (terminates TLS, load balances,
   (web, mobile,                |                  |   anti-abuse, rate limit per tenant)
   server-side)                 +--------+---------+
                                         |
                                         v
                                +------------------+
                                |  Ingest Service  |  (validates schema, assigns event_id,
                                |  (stateless)     |   adds server timestamp, picks
                                |                  |   partition key)
                                +--------+---------+
                                         |
                                         v
                                +------------------+
                                |   [ ? ]          |  (durable, partitioned, replicated;
                                |                  |   7-day retention; absorbs bursts)
                                +--------+---------+
                                         |
                       +-----------------+-----------------+
                       |                 |                 |
                       v                 v                 v
                +-------------+   +-------------+   +-------------+
                |  [ ? ]      |   |  archiver   |   |  realtime   |
                |  (hot       |   |  (writes    |   |  consumer   |
                |   storage   |   |   Parquet   |   |  (fraud,    |
                |   for       |   |   to S3     |   |   alerting) |
                |   point     |   |   in 5-min  |   |             |
                |   reads)    |   |   batches)  |   |             |
                +-------------+   +-----+-------+   +-------------+
                                        |
                                        v
                                +------------------+
                                |   [ ? ]          |  (cold tier, queried by Athena /
                                |                  |   Presto / BigQuery for old events)
                                +------------------+

                                +------------------+
                                |  [ ? ]           |  (consumed by every stage; tracks
                                |                  |   throughput, queue depth, batch
                                |                  |   size, ack lag)
                                +------------------+
```

<details>
<summary><b>Reveal: the full pipeline</b></summary>

```
                                +----------------------+
   producer apps  --HTTPS-->   |  Edge LB + API       |  TLS, anycast,
   (web, mobile,                |  Gateway + WAF       |  per-tenant rate limit,
   server-side)                 |                      |  back-pressure (429 when
                                |                      |  downstream queue full)
                                +----------+-----------+
                                           |
                                           v
                                +----------------------+
                                |  Ingest Service      |  validate schema, assign
                                |  (stateless, N pods) |  event_id (snowflake),
                                |                      |  add server timestamp,
                                |                      |  pick partition key
                                |                      |  (event_type + tenant_id)
                                +----------+-----------+
                                           |
                                           v
                                +----------------------+
                                |  Kafka cluster       |  partitioned by
                                |  (or Kinesis,        |  hash(event_type, tenant_id);
                                |   Pulsar, NATS JS)   |  RF=3; 7-day retention;
                                |                      |  acks=all
                                +----------+-----------+
                                           |
                       +-------------------+---------------------+
                       |                   |                     |
                       v                   v                     v
                +--------------+   +--------------+     +--------------+
                |  Cassandra   |   |  Archiver    |     |  Flink /     |
                |  (hot tier,  |   |  (consumes,  |     |  stream      |
                |   90-day     |   |   buffers    |     |  processor   |
                |   point      |   |   5 min,     |     |  (fraud,     |
                |   reads,     |   |   writes     |     |   alerts,    |
                |   sharded    |   |   Parquet)   |     |   rollups)   |
                |   by key)    |   |              |     |              |
                +--------------+   +------+-------+     +--------------+
                                          |
                                          v
                                +----------------------+
                                |  S3 / GCS (Parquet)  |  partitioned by
                                |  + Athena / Presto / |  date=YYYY-MM-DD/
                                |  BigQuery on top     |  event_type=XYZ/
                                |                      |  tenant_id=ABC/
                                +----------------------+

                                +----------------------+
                                |  Metrics + Logs      |  Prometheus / OTel;
                                |  (Prometheus,        |  every stage reports
                                |   Grafana,           |  throughput, queue depth,
                                |   OpenTelemetry)     |  batch size, ack lag,
                                |                      |  partition skew
                                +----------------------+
```

Why each piece is here:

The edge LB and API gateway exist because multiple ingest pods are required just to terminate TLS at 12 Gbps. Per-tenant rate limit prevents one noisy tenant from drowning out the rest. Back-pressure (returning 429 to producers when Kafka is unhealthy) protects the cluster from cascading failure.

The ingest service is stateless. Its only job: validate, stamp, route. It does not own state; it puts the event in Kafka and returns 202 Accepted.

Kafka is the shock absorber. Producers fill it; consumers drain it. If a downstream consumer stalls for 5 minutes, Kafka buffers the backlog without affecting producers.

Cassandra hot tier handles recent events queryable by key (event_id, or (tenant_id, ts) range). LSM-tree is the right shape: writes are sequential, no read-modify-write penalty per insert.

The archiver reads from Kafka, buffers 5 minutes of events or N MB, writes one big Parquet file to S3. Trades latency for cost; you get cold-storage prices and columnar query performance.

The stream processor (Flink) handles derived data: per-minute rollups, anomaly scores, alerts. Reads the same Kafka topic.

S3 + query layer is bottomless storage. Athena, Presto, or BigQuery query Parquet without a dedicated cluster.

Observability sits on the side because without metrics on queue depth, batch size, and ack lag, you find out about ingestion problems via customer complaints.

</details>

## Step 5: Append-only log structures (why LSM beats B-tree here)

Postgres uses a B-tree for its primary index. Cassandra uses an LSM-tree. RocksDB, ScyllaDB, HBase, and Kafka's log itself are all LSM-flavored. For write-heavy workloads, LSM wins. Why?

Sketch the difference. In particular, what is a B-tree doing on a random INSERT that an LSM is not?

<details>
<summary><b>Reveal: B-tree vs LSM for writes</b></summary>

A B-tree insert (Postgres, MySQL InnoDB) does five things. Find the leaf page for this key, often requiring a random read from disk if the page is not in the buffer pool. Insert the key into the page; if the page is full, split it (write 2 pages instead of 1). Update parent pages if the split propagates up. Write to WAL (sequential), then update the data page (often random, often later). On commit, fsync the WAL.

Random-key writes are the worst case. Each insert touches a different leaf page. The buffer pool churns. You pay a random read plus a random write per insert. A single Postgres tops out around 1k-10k writes/sec for keyed inserts that do not fit in memory.

An LSM insert (Cassandra, RocksDB, Scylla) is simpler. Append to a write-ahead commit log (sequential disk write). Insert into an in-memory memtable (sorted skip list or balanced tree, in RAM). When the memtable is full, flush it to disk as a new immutable SSTable (one big sequential write). Background compaction merges old SSTables to reclaim space and bound read amplification.

Why this is faster: all disk writes are sequential. The commit log appends. SSTable flushes are big sequential writes. Sequential disk throughput is ~10-100x random disk throughput on spinning disks, and even on NVMe sequential beats random IOPS for sustained workloads. There is no read-before-write (B-tree inserts read the leaf page first; LSM just appends). And no in-place updates: SSTables are immutable, updates are new writes with a tombstone for deletes, concurrency is trivial with no page locks.

What you pay: read amplification (a read might check the memtable plus several SSTables; bloom filters help but reads are slower than B-tree reads for the same data), write amplification (compaction rewrites SSTables; the same byte may be written 5-10 times over its lifetime), and space amplification (until compaction runs, you have multiple copies of data on disk).

LSM wins for write-heavy, read-by-key-or-range, time-series, logs, telemetry. Cassandra, ScyllaDB, RocksDB, LevelDB, HBase, BigTable, DynamoDB under the hood, InfluxDB.

B-tree wins for OLTP with frequent updates to the same rows, read-dominated workloads, secondary-index-heavy queries. Postgres, MySQL.

The append-only log extreme is even simpler than LSM: Kafka's storage is pure append to a segment file, no compaction unless you enable log compaction explicitly. Reads are by offset (O(1) seek to position). This is why Kafka can ingest millions of messages per second per partition on commodity hardware. The trade-off: you cannot query by arbitrary key, only by offset or time.

For our 1M events/sec audit pipeline, the storage layers stack up: Kafka log (pure append, infinite write throughput per partition until disk fills), Cassandra hot tier (LSM, query by key, 90-day retention), S3 Parquet cold tier (immutable columnar files, queried by partition). Each is optimized for write throughput at its scale.

</details>

## Step 6: Partitioning strategy

You have to pick a partition key for Kafka and a shard key for Cassandra. The choice determines whether load distributes evenly or whether one partition gets 10x the traffic of the rest. Three families of strategies. For each, name when it works and when it does not.

| Strategy | How it works | Good for | Fails when |
|----------|-------------|----------|-----------|
| Time-based | partition = day or hour | [ ? ] | [ ? ] |
| Hash-based | partition = hash(key) % N | [ ? ] | [ ? ] |
| Tenant-based | partition = tenant_id | [ ? ] | [ ? ] |

<details>
<summary><b>Reveal: the three partitioning families</b></summary>

| Strategy | How it works | Good for | Fails when |
|----------|-------------|----------|-----------|
| **Time-based** | Each hour or day gets its own partition. Files named `events/2026-05-23/...` | Time-range queries ("events from last week"). Cheap to drop old data (delete the partition directory). Cold-storage archiving (move yesterday's partition to S3 Glacier). Natural fit for Parquet on S3. | **Hot partition on writes.** All writes go to the current hour's partition. The newest partition is hot; the rest are cold. Compute is wasted on cold partitions; the hot one melts. Mitigation: combine with a secondary key (hour + hash). |
| **Hash-based** | `partition_id = hash(event_id) % N` or `hash(user_id) % N` | Even write distribution. No hotspots if the key has high cardinality. Trivial horizontal scaling: add nodes, rebalance. | **Range queries become scatter-gather.** "All events for user X in last hour" hits every partition unless you hash by user_id (then "all events between time A and B" is the scatter case). Cannot drop old data without scanning. Adding nodes triggers a rebalance that moves data across the network. |
| **Tenant-based** | `partition_id = tenant_id` or hash(tenant_id) | Multi-tenant isolation. Noisy-tenant containment (one tenant's spike does not affect others). Easy per-tenant deletion (GDPR, customer offboarding). Per-tenant SLAs. | **Uneven partition sizes.** Your largest tenant might be 1000x bigger than the median. That tenant's partition is hot; smaller ones are cold. The classic "one big customer" problem. Mitigation: sub-shard the largest tenants by appending a random suffix (`tenant_42_0`, `tenant_42_1`, ..., `tenant_42_15`). |

The right answer is usually a composite.

For 1M events/sec audit logging, the Kafka partition key is `hash(event_type, tenant_id)`. That spreads load across partitions while preserving per-key ordering within a single (event_type, tenant_id) stream. Start at about 100 partitions, scale to 1000+ as needed. Each partition handles ~10k events/sec at the high end.

The Cassandra primary key is `((tenant_id, day), event_id)`. Partition key `(tenant_id, day)` puts all events for a tenant on a given day in one place (cheap range scan). Clustering column `event_id` gives uniqueness and ordering within the partition. The day component bounds partition size: even a huge tenant only writes one day at a time per partition.

The S3 Parquet prefix is `date=YYYY-MM-DD/event_type=X/tenant_id=Y/`. Time-based at the top level (cheap to drop old data), hash-friendly subdirs for query pruning. Athena uses partition pruning to skip entire prefixes when the query filters on date or tenant.

The hot partition problem in practice: suppose one tenant's mobile app has a bug that emits 100k events/sec instead of the normal 100. Their (event_type, tenant_id) partition gets 100k/sec while every other partition gets 100. That partition's Kafka broker pegs CPU, lag grows, consumers cannot keep up, eventually the broker disk fills.

Mitigations, in order: detect early via per-partition rate monitoring with alerting on `max(partition_rate) / median(partition_rate) > 10x`. Sub-shard the hot key by adding a random suffix to the partition key for that one tenant: `(event_type, tenant_id, random(0..15))`. Now their load spreads across 16 partitions. Cost: you cannot rely on per-key ordering for that tenant. Apply a per-tenant rate limit at the ingest tier; cap any single tenant at, say, 50k events/sec. Excess events get 429. The tenant fixes their bug. Long term, quota and isolation: premium tenants get reserved capacity; long-tail tenants share a pool.

For Cassandra, hot partition = hot row = hot node. Same mitigations apply: sub-shard the key, monitor for skew, throttle at ingest.

</details>

## Step 7: When to give up consistency

At small scale, every event is in one Postgres. One ACID transaction. Either it is durably written or it is not. Linearizable. Easy.

At 1M events/sec, that promise is gone. You cannot have linearizable writes across a distributed Kafka cluster + Cassandra + S3 and 10 Gbps of bandwidth without paying enormous latency. So you trade.

Three delivery semantics, and every write-heavy system has to pick:

- **At-most-once.** Send and forget. If anything fails between producer and storage, the event is lost. Cheapest, fastest, lowest latency.
- **At-least-once.** Acknowledge end-to-end. On any failure, retry. May produce duplicates. Most common in practice.
- **Exactly-once.** Each event appears in storage exactly one time, no duplicates, no losses. Expensive, complicated, slow.

For each, when would you use it?

<details>
<summary><b>Reveal: when each semantics is the right choice</b></summary>

**At-most-once.** Use when the data is statistical, not transactional. Page-view metrics, click-tracking, performance traces, ad impressions. Losing 0.1% does not change the analysis. Use when latency matters more than completeness. UDP-style fire-and-forget. The producer does not wait for an ack.

Implementation: producer sends, returns immediately, does not retry. If the broker is down or the network drops the packet, the event is gone forever. Cost: zero retry overhead, no idempotency machinery, no transactional logs. Cheapest pipeline you can build.

Do not use for anything where missing one event has user-facing consequences (audit, billing, security alerts, order events).

**At-least-once.** The default for almost every modern event pipeline. Use when loss is unacceptable but duplicates are tolerable, often because the consumer is idempotent. Kafka producers with `acks=all` and consumer-side commit-after-process gives you at-least-once.

Implementation: producer retries on any failure to ack. Network blip? Producer did not see the ack but the broker may have written it. Producer retries; broker writes a second copy. Event appears twice. Consumer commits offsets after processing. If consumer crashes between processing and committing, it reprocesses on restart. Event processed twice.

How you handle duplicates: idempotent consumer (use the event's `event_id` as the primary key in storage; second write is a no-op via INSERT ON CONFLICT DO NOTHING, Cassandra's `IF NOT EXISTS` clause, or dedupe at S3 Parquet compaction time). Or a deduplication window: keep recent event_ids in a Bloom filter or Redis set; reject duplicates seen in the last N minutes. Cheap, probabilistic. Or an idempotent producer: Kafka's idempotent producer (transactional ID + sequence numbers) prevents duplicates within a producer session. Not full exactly-once but kills the most common dup source (retries on transient broker errors).

Cost: the storage layer carries the dedup burden. Indexing on event_id eats space.

This is what 80% of production pipelines do. Acknowledge the duplication risk; design consumers to be idempotent; move on.

**Exactly-once.** Use when the downstream effect is non-idempotent and money is involved. Payment processing. Order fulfillment. Inventory deduction. Audit logging *sometimes* claims to need this. In practice, at-least-once with idempotent storage (event_id as primary key) is equivalent to exactly-once and 10x cheaper.

Implementation in the Kafka world: Kafka EOS (exactly-once semantics) via transactional producers + read-committed consumers. The producer wraps writes to multiple partitions in a transaction; the consumer only sees committed messages. Combined with idempotent producer IDs.

Cost: throughput drops 20-40% vs at-least-once. Latency increases (you wait for commits). Operational complexity goes up (transactional IDs must survive producer restarts, consumer offset commits become two-phase, partition rebalances are more expensive). All downstream consumers must opt into read-committed mode.

The transactional outbox pattern is the non-Kafka version: producer writes the event to its local DB in the same transaction as the business state change, into an outbox table; a separate process reads the outbox and ships to Kafka with retries. The "exactly once" is "the event will appear in Kafka at least once, but only if the business write succeeded." Same idempotent-consumer requirement on the read side.

The honest take: most "we need exactly-once" requirements are actually "we need at-least-once with idempotent processing." The latter is much easier and cheaper. The former is for cases where you literally cannot make the consumer idempotent (a side effect that cannot be deduped, like sending an SMS).

**What we pick for the 1M events/sec audit pipeline.** At-least-once. Audit needs durability, so at-most-once is out. Storage is Cassandra keyed by `event_id`, so idempotency is free; second write is a no-op. Exactly-once via Kafka EOS would cost ~30% throughput and would not help (we already dedupe at storage).

Producer: Kafka producer with `acks=all`, `enable.idempotence=true`, `retries=Integer.MAX_VALUE`. Within a producer session, no duplicates due to retries. Across producer crashes, duplicates are possible (handled by `event_id` dedup).

Consumer: commit Kafka offset after the Cassandra write completes. If consumer crashes mid-batch, it reprocesses the batch on restart; Cassandra's `IF NOT EXISTS` (or just the primary-key constraint) catches the duplicates.

End-to-end guarantee: every event that the producer believed it sent successfully is durably stored at least once. Reads see each event exactly once (deduplicated by `event_id`).

</details>

## Follow-up questions (10)

Try answering each in 2 to 4 sentences before reading the solution.

1. **Back-pressure.** Producers are sending 2x the rate Kafka can absorb. The Kafka brokers are healthy but the per-partition log is filling and disk is at 80%. What does the ingest service do? What does the producer SDK do?

2. **Hot tenant.** One tenant's mobile app has a bug and is sending 200k events/sec when the average is 100. They are saturating one Kafka partition and that partition's broker is at 100% CPU. How do you find them, and what do you do in the next 5 minutes versus the next 5 days?

3. **Clock skew.** Two producers on different machines disagree about "now" by 3 minutes. They both send events tagged with their local timestamps. Your downstream reports show events arriving "in the future." How do you fix it without forcing every producer to NTP-sync to nanosecond precision?

4. **Duplicate events.** A producer retried a batch after a broker timeout. The broker had actually committed the first attempt. Now Cassandra has two copies of every event in that batch. Walk me through the detection and dedup strategy.

5. **Consumer fell behind.** A Cassandra writer consumer has been stuck for 30 minutes due to a bad deploy. Kafka has buffered 1.8 billion events for that consumer group. What happens when you fix the bug and restart? How do you keep it from melting the storage layer when it catches up?

6. **Schema evolution.** A team added a new field to the event schema. Old producers send 5 fields, new producers send 6. Cassandra has a fixed table schema. How do you handle the migration without dropping events or breaking old consumers?

7. **Recent-event reads.** A consumer wants to query "all events for user U in the last 30 seconds." Your storage path is Kafka, then batched 5 min into Cassandra. The event might still be in Kafka. How do you make the read see both?

8. **A region goes down.** US-East Kafka cluster is offline. Producers in that region cannot publish. Producers in other regions are fine. What is your DR play, and how much data could be lost?

9. **Cold-tier query cost.** A user wants "every event for tenant X for the past 3 years." Naive Athena query scans 3 PB of Parquet. How do you make this both fast and cheap?

10. **Exactly-once for a side effect.** One downstream consumer sends an SMS for every fraud alert. SMS is non-idempotent (the user gets two texts if you send twice). How do you guarantee exactly one SMS without paying Kafka EOS cost on the whole pipeline?

## Related problems

- **[Approval Management (011)](../011-approval-management/question.md)**. The audit log in that design is a write-heavy append-only system. The patterns here (tiered storage, append-only log, batched writes) apply directly to its audit pipeline.
- **[Todo List Sharing (013)](../013-todo-list-sharing/question.md)**. The change-log / sync pipeline for collaborative todo edits is the same shape at smaller scale: every edit is a small append-only event, partitioned by list_id, eventually projected to read-side state.
- **[News Feed (002)](../002-news-feed/question.md)**. Timeline write fan-out is the canonical write-heavy problem at consumer scale: each post produces N writes (one per follower's timeline). Same partitioning and batching toolkit, different shape because fan-out multiplies the load.
