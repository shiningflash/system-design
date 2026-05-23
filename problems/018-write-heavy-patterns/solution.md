## Solution: Design a Write-Heavy System (Patterns Walkthrough)

### TL;DR

A write-heavy system is one whose cost is dominated by ingestion. Each event is small, the queries are simple or batched, and the database's job is to absorb writes faster than a normal OLTP DB can.

The architectural toolkit, in the order you reach for it as scale grows:

1. **Sync DB write** (~1k/sec on Postgres). Fine until it isn't.
2. **In-memory buffer + batched commit** (~10k/sec). Amortize fsync over many events.
3. **Durable queue + async writer** (~100k/sec). Decouple producer from storage; switch storage to an LSM-tree (Cassandra, Scylla, RocksDB) for sequential writes.
4. **Partitioned queue + stream processing + tiered storage** (1M+/sec). Hash the load across partitions; route hot data to Cassandra, cold to S3 Parquet; let Flink/Spark do enrichment.

Each step trades latency, consistency, or operational complexity for throughput. The senior move is naming the limit of stage N before you reach for stage N+1, not pre-emptively building stage 4.

Three cross-cutting decisions decide the final shape:

- **Partition key.** Time, hash, or tenant. Composite keys (`(event_type, tenant_id, day)`) are the usual answer.
- **Append-only storage.** LSM-tree at the hot tier, Parquet on object storage at the cold tier. Both are write-optimized; both make random updates expensive (which you don't do).
- **Delivery semantics.** At-least-once with idempotent storage (event_id as primary key) is the right answer for 95% of cases. At-most-once is for stats. Exactly-once is for non-idempotent side effects only.

### 1. Clarifying questions

Covered in `question.md`. The four that meaningfully change the design:

- **Durability.** Zero-loss vs lossy changes everything from producer acks to whether you need a transactional outbox.
- **Acceptable lag.** Sub-second forces synchronous writes; 10s lets you batch into Parquet at 1% of the cost.
- **Ordering.** Per-key vs global vs none drives partition strategy.
- **Read pattern.** Point lookups vs scans vs aggregations selects the storage engine.

If you walked in without asking durability and lag, you have no basis to choose between Kafka EOS (expensive, exactly-once) and UDP fire-and-forget (cheap, lossy). They are 1000x apart in cost.

### 2. Capacity (full working)

Given: 1M events/sec sustained, 3M peak, 500 bytes/event, 90 days hot, 7 years cold, no global ordering, 10s acceptable lag.

- **Bandwidth.** 500 MB/sec sustained = 4 Gbps. Peak 12 Gbps. You need multiple ingest nodes per region just for NIC throughput.
- **Daily volume.** 86.4B events/day = 43 TB/day raw.
- **Hot storage.** 90 × 43 = ~3.9 PB. 10-30 node Cassandra cluster with RF=3.
- **Cold storage.** 365 × 7 × 43 / 5 (compression) = ~22 PB in S3 Parquet.
- **Buffer for 5-min downstream stall.** 300M events = 150 GB. Kafka with 7-day retention handles this trivially.
- **Ingest connections.** Multiple producers per company app × thousands of apps = ~100k concurrent TLS connections, spread across ~10 ingest nodes per region.

**The key insight:** the bottleneck is bandwidth and storage layout, not CPU. A single Postgres maxes out around 10k writes/sec, which is 100x too slow. We need to spread writes across nodes AND batch them so that the per-write overhead amortizes.

### 3. API design (write API)

The read API is out of scope for this problem; reads come later as a separate concern.

**Single-event ingest (low-volume producers):**

```
POST /api/v1/events
Content-Type: application/json
X-Tenant-Id: tenant_42
Authorization: Bearer <token>
Idempotency-Key: <uuid>           # optional; if set, server dedupes on this key

{
  "event_type": "user.login",
  "user_id": "u_8201",
  "timestamp": "2026-05-23T10:14:02.331Z",   # producer clock
  "attributes": { ... }                       # arbitrary JSON
}
```

Responses:

| Status | Meaning |
|--------|---------|
| 202 Accepted | Event written to durable queue; will be queryable within 10s |
| 400 Bad Request | Schema invalid (missing required field, wrong type) |
| 401 Unauthorized | Missing or bad token |
| 413 Payload Too Large | Single event > 1 MB |
| 429 Too Many Requests | Per-tenant rate limit exceeded, or downstream queue overloaded (back-pressure) |
| 503 Service Unavailable | Ingest cluster degraded; producer should retry with backoff |

**Batched ingest (high-volume producers, recommended path):**

```
POST /api/v1/events/batch
Content-Encoding: gzip
Content-Type: application/json

{
  "events": [
    { ...event 1... },
    { ...event 2... },
    ...up to 1000 events or 1 MB compressed...
  ]
}
```

Responses are per-batch (whole-batch accept) or per-event (partial accept):

| Status | Meaning |
|--------|---------|
| 202 Accepted | Whole batch durably queued. Response body: `{"batch_id": "...", "accepted": 1000}` |
| 207 Multi-Status | Partial success. `{"accepted": 998, "failed": [{"index": 17, "error": "schema_invalid"}]}` |
| 429 / 503 | Same as single-event |

**Why the API looks this way:**

- **202 not 200.** The event is not durably stored at the time of response. It is durably queued. Returning 200 implies "we have it" with full guarantees; 202 communicates "we have accepted responsibility and you can stop retrying."
- **Idempotency-Key optional.** Most producers don't need it (event_id deduplication at storage handles dups). For producers that can't retry safely (one-shot lambdas), an Idempotency-Key gives end-to-end dedup.
- **Batch endpoint is the recommended path.** Per-event HTTP at 1M/sec is 1M TLS handshakes/sec (impossible), or 1M HTTP requests on persistent connections (very expensive). Batching of 1000 cuts to 1k req/sec. Same bytes, 1000x cheaper.
- **Server-side timestamp added in addition to producer timestamp.** Both are kept. Producer time is "when did the event happen." Server time is "when did we accept it." Clock-skew analysis uses the gap.
- **Per-tenant rate limit visible to the producer.** The 429 response includes `Retry-After` and `X-RateLimit-Remaining` so well-behaved producers throttle themselves.

### 4. Data model

**Event schema (canonical):**

```json
{
  "event_id": "01H8K2X...",            // ULID or snowflake, generated by server if absent
  "tenant_id": "tenant_42",
  "event_type": "user.login",
  "user_id": "u_8201",
  "producer_ts": "2026-05-23T10:14:02.331Z",
  "server_ts":   "2026-05-23T10:14:02.412Z",
  "attributes": { ... },
  "schema_version": 3
}
```

**Hot tier (Cassandra):**

```cql
CREATE TABLE events_by_tenant_day (
    tenant_id     text,
    day           date,                   -- derived from server_ts
    event_id      timeuuid,                -- sortable, unique
    event_type    text,
    user_id       text,
    producer_ts   timestamp,
    server_ts     timestamp,
    attributes    text,                    -- JSON serialized
    schema_version int,
    PRIMARY KEY ((tenant_id, day), event_id)
) WITH CLUSTERING ORDER BY (event_id DESC)
  AND default_time_to_live = 7776000;     -- 90 days
```

Partition key `(tenant_id, day)`:
- Bounds partition size. Even the largest tenant only writes one day at a time into one partition. A "large" tenant at 100k events/sec × 86400s × 500B = 4 TB/day, which is too big for one Cassandra partition (target: <100 MB per partition). For those tenants, subshard by appending an hour or a random suffix: `(tenant_id, day, bucket)`.
- Makes "events for tenant X on day D" a single-partition read. Common query.

Clustering column `event_id` (timeuuid, monotonic):
- Sorted within the partition. "Latest events" is a prefix scan.
- Provides natural dedup: same `event_id` inserted twice is the same row.

**Secondary tables (denormalized, separate write path):**

```cql
CREATE TABLE events_by_user_day (
    user_id       text,
    day           date,
    event_id      timeuuid,
    event_type    text,
    tenant_id     text,
    attributes    text,
    PRIMARY KEY ((user_id, day), event_id)
);
```

Same data, different partition key, supports "events for user U." Written by the stream processor (Flink) reading the Kafka topic and writing two tables. The cost: 2x write amplification at the storage layer. The benefit: O(1) lookups for both access patterns.

**Cold tier (S3 Parquet):**

Partition path: `s3://events-cold/date=YYYY-MM-DD/event_type=X/tenant_id=Y/file_N.parquet`

Inside each Parquet file: columnar layout. Column-pruned reads (only read `user_id` and `event_type` from a 50-column event) are essentially free. Partition pruning at the file-prefix level means Athena/Presto skip entire directories that don't match the query filters.

### 5. Core algorithm: the write pipeline

The pipeline has four stages. Each stage has a clear contract with the next.

**Stage 1: Ingest service (synchronous part of producer's request).**

```python
def handle_batch(request):
    tenant = authenticate(request)
    rate_limiter.check(tenant)                     # 429 if over quota
    queue_health.check()                            # 503 if Kafka is unhealthy

    events = parse_and_validate(request.body)       # 400 on schema errors
    for e in events:
        e.event_id = e.event_id or new_ulid()
        e.server_ts = now()
        e.tenant_id = tenant.id

    # Group by partition key for fewer Kafka send calls
    by_partition = group_by(events, lambda e: partition_for(e))

    futures = []
    for partition, group in by_partition.items():
        # Async send; producer batches internally per partition
        futures.append(kafka_producer.send_async(
            topic="events",
            partition=partition,
            value=group,
            acks="all",                              # wait for ISR
            timeout_ms=5000
        ))

    wait_all(futures, timeout_ms=10000)             # ack-wait
    return 202_accepted(batch_id=request.id)
```

Key properties:
- **Producer waits for `acks=all`.** The HTTP response is sent only after Kafka confirms durable replication. If the producer sees 202, the event is on disk on multiple brokers.
- **No DB write at ingest.** Ingest's only durability is Kafka. Storage writes happen downstream.
- **Bounded latency.** P99 ingest target: 50 ms (excludes producer-to-edge network). If Kafka send exceeds 5s timeout, return 503 (let producer retry).

**Stage 2: Kafka (durable queue).**

Topic configuration:
- Partitions: 200 to start, expandable. Each partition handles ~5k events/sec comfortably.
- Replication factor: 3.
- `min.insync.replicas=2` with producer `acks=all` gives "no loss as long as 2 brokers survive."
- Retention: 7 days. Long enough to replay a full week if consumers fall behind.
- Compression: zstd. Trades a bit of CPU for ~60% smaller on-disk footprint.

Per-partition ordering is preserved. Cross-partition ordering is not. That is the trade we made when picking a partition strategy in question.md Step 6.

**Stage 3: Hot-tier writer (Cassandra).**

```python
def consume_loop(consumer, cass_session):
    while True:
        records = consumer.poll(timeout_ms=100, max_records=1000)
        if not records:
            continue

        # Build a Cassandra UNLOGGED BATCH per partition
        batches_by_partition = defaultdict(list)
        for r in records:
            event = json.loads(r.value)
            pk = (event["tenant_id"], date(event["server_ts"]))
            batches_by_partition[pk].append(event)

        for pk, events in batches_by_partition.items():
            batch = BatchStatement(batch_type=BatchType.UNLOGGED)
            for e in events:
                batch.add(insert_stmt, params_for(e))
            cass_session.execute(batch)              # one network roundtrip

        consumer.commit()                            # only after Cassandra ack
```

Key properties:
- **UNLOGGED BATCH per Cassandra partition.** Cassandra's UNLOGGED batches that hit a single partition are essentially a single coordinated write. 10-100x throughput vs single-event INSERTs.
- **Commit Kafka offset after Cassandra ack.** At-least-once: if the consumer crashes between Cassandra write and offset commit, it reprocesses the records. Primary-key dedup in Cassandra (or `IF NOT EXISTS`) catches the duplicates.
- **Bounded batch size.** Don't let a single Cassandra BATCH exceed ~5 KB; larger triggers Cassandra warnings and tombstone-related issues. Limit to ~50 events per batch as a safety bound.

**Stage 4: Cold-tier archiver (S3 Parquet).**

```python
def archive_loop(consumer):
    buffer = []
    last_flush = now()
    while True:
        records = consumer.poll(timeout_ms=1000, max_records=10000)
        buffer.extend(records)

        if len(buffer) >= 100_000 or now() - last_flush > 300:    # 100k or 5 min
            flush(buffer)
            buffer = []
            last_flush = now()

def flush(records):
    # Group by partition path: date / event_type / tenant_id
    grouped = group_by(records, lambda r: parquet_path(r))
    for path, group in grouped.items():
        write_parquet(s3_client, path, group)
    consumer.commit()
```

Key properties:
- **Time-or-size flush.** Flush every 5 minutes or every 100k events, whichever comes first. Small batches waste S3 PUT operations ($0.005 per 1000); 5-min batches keep cost manageable.
- **Parquet over S3.** Columnar. Compressed. Athena-queryable. Cheap.
- **Group by partition path.** Each S3 key is one Parquet file with one (date, event_type, tenant_id). Lets queries prune entire prefixes.

### 6. Architecture (full pipeline)

```
                                        ┌───────────────────────┐
   producer apps  ──HTTPS──►            │  Edge LB + WAF        │  anycast, TLS,
   (web, mobile,                        │  + API Gateway        │  per-tenant rate
   server-side)                         │                       │  limit, anti-abuse
                                        └────────────┬──────────┘
                                                     │
                              ┌──────────────────────┼──────────────────────┐
                              │                      │                      │
                              ▼                      ▼                      ▼
                       ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
                       │  Ingest     │        │  Ingest     │        │  Ingest     │
                       │  (N pods)   │        │  (N pods)   │        │  (N pods)   │
                       │  validate,  │        │             │        │             │
                       │  stamp,     │        │             │        │             │
                       │  route      │        │             │        │             │
                       └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
                              │                      │                      │
                              └──────────────┬───────┴──────────────────────┘
                                             │
                                             ▼
                                ┌────────────────────────┐
                                │  Kafka cluster         │  200 partitions,
                                │  topic: events         │  RF=3, 7-day retention,
                                │  partitioned by        │  acks=all, zstd
                                │  hash(event_type,      │
                                │       tenant_id)       │
                                └──┬──────────┬──────────┘
                                   │          │          │
                ┌──────────────────┘          │          └─────────────────────┐
                │                             │                                │
                ▼                             ▼                                ▼
        ┌──────────────────┐         ┌──────────────────┐           ┌──────────────────┐
        │  Hot writers     │         │  Archiver        │           │  Stream proc     │
        │  (Cassandra      │         │  (S3 Parquet     │           │  (Flink)         │
        │   consumer       │         │   consumer       │           │                  │
        │   group)         │         │   group)         │           │  - per-min       │
        │                  │         │                  │           │    rollups       │
        │  UNLOGGED        │         │  5-min batches   │           │  - fraud         │
        │  BATCH per       │         │  100k events     │           │    alerts        │
        │  partition       │         │                  │           │  - dedup         │
        └────────┬─────────┘         └────────┬─────────┘           └────────┬─────────┘
                 │                            │                              │
                 ▼                            ▼                              ▼
        ┌──────────────────┐         ┌──────────────────┐           ┌──────────────────┐
        │  Cassandra       │         │  S3 / GCS        │           │  Rollup DB       │
        │  (hot tier)      │         │  (Parquet,       │           │  (ClickHouse or  │
        │  90-day TTL      │         │   7-year         │           │   Druid for      │
        │  RF=3            │         │   retention)     │           │   dashboards)    │
        │  sharded by      │         │  date / event /  │           │                  │
        │  (tenant, day)   │         │  tenant prefix   │           │                  │
        └──────────────────┘         └────────┬─────────┘           └──────────────────┘
                                              │
                                              ▼
                                     ┌──────────────────┐
                                     │  Athena / Presto │  ad-hoc queries
                                     │  / BigQuery      │  over cold data
                                     └──────────────────┘

         ┌─────────────────────────────────────────────────────────────────┐
         │  Observability: Prometheus + Grafana + OpenTelemetry tracing    │
         │  every stage publishes: rate, queue depth, batch size, ack lag, │
         │  partition skew, error rate                                     │
         └─────────────────────────────────────────────────────────────────┘
```

Component responsibilities (recap with concrete tech):

- **Edge LB + API Gateway.** AWS ALB or Cloudflare + Envoy. Terminates TLS, enforces per-tenant rate limits via Redis-backed token bucket.
- **Ingest pods.** Stateless. Go or Rust for low GC overhead at high request rates. Horizontally scalable; 10-30 pods per region depending on traffic.
- **Kafka.** Confluent Kafka, AWS MSK, or Aiven Kafka. Alternatives: AWS Kinesis (managed, simpler ops, harder to migrate off), Apache Pulsar (better multi-tenant story), Redpanda (Kafka-API compatible, lower latency).
- **Hot writers.** A Kafka consumer group. Each consumer instance owns N partitions. Writes batched per-Cassandra-partition for throughput.
- **Cassandra.** ScyllaDB if you want better performance per node and lower ops cost. DynamoDB if you want fully managed and can stomach the cost.
- **Archiver.** A Kafka consumer that writes Parquet to S3. Often a Flink job for the buffering + columnar transform, or a simpler Go/Java service.
- **Stream processor.** Flink for sophisticated stateful processing. KSQL or Spark Structured Streaming for SQL-flavored. Materialize if you want incremental views over the stream.
- **Cold tier.** S3 + Athena. Glacier Deep Archive for events older than 1 year.

### 7. Read and write paths

The problem is write-focused but the read path informs the storage choices.

**Write path (single-event API call, end-to-end):**

1. Producer sends POST /events/batch.
2. Edge LB terminates TLS, routes to ingest pod.
3. Ingest pod: auth, rate limit, schema validate, stamp event_id and server_ts.
4. Ingest pod: kafka_producer.send(events) with acks=all.
5. Kafka brokers: write to leader, replicate to followers, ack when min.insync.replicas reached.
6. Ingest pod: return 202 to producer.

P99 target: 50 ms. P50: 5 ms. Latency budget consumed almost entirely by Kafka send + ack-wait.

**Async write path (Kafka to storage):**

7. Hot writer consumer polls Kafka, gets a batch of records.
8. Groups by Cassandra partition key.
9. Executes UNLOGGED BATCH per partition.
10. Commits Kafka offset.

End-to-end lag (ingest to Cassandra-queryable): typically 1-3 seconds, P99 ~10 seconds (configurable trade-off vs throughput).

**Cold-tier write path:**

11. Archiver consumer polls Kafka, buffers in memory.
12. Every 5 min or 100k events, writes one Parquet file per (date, event_type, tenant_id) to S3.
13. Commits Kafka offset.

End-to-end lag (ingest to S3): up to 5-6 minutes.

**Read path (recent event by ID):**

1. Client → API → Cassandra query: `SELECT * FROM events_by_tenant_day WHERE tenant_id = ? AND day = ? AND event_id = ?`.
2. Cassandra returns row. P99: 10 ms.

**Read path (events for user in last hour):**

1. Client → API → Cassandra query against `events_by_user_day` (the denormalized table).
2. Single-partition prefix scan. P99: 30 ms.

**Read path (analytics, last year):**

1. Client → API → Athena query against S3 Parquet.
2. Partition pruning skips irrelevant date/event_type/tenant prefixes.
3. Columnar read pulls only the requested columns.
4. P99: seconds to minutes depending on scope.

**Read path that crosses tiers (last 30 seconds, user X):**

The event might still be in Kafka, not yet in Cassandra. Three options:
- Accept the lag. Tell the user "data has ~5s lag."
- Query Kafka directly for very recent data, fall back to Cassandra for older. Operationally hard; Kafka is not designed for key lookups.
- Push events to a real-time read store (Redis hash per user) in parallel with Kafka. Pay 2x writes for sub-second visibility.

Pick based on the user-facing SLA. For most audit workloads, "5-10s lag" is fine.

### 8. Scaling journey: 10 to 1M users (write path evolution)

This is the section the interviewer cares most about. At every stage, what just broke, what fixes it, and what you do *not* yet build.

#### Stage 1: 100 events/sec (a single product, a few thousand users)

**What you build:**

- Single Postgres (db.t3.medium, 4 GB RAM).
- One Go/Python app instance with a `/events` endpoint that does `INSERT INTO events_table (...) VALUES (...)`.
- One table, indexed by `event_id` and `(user_id, created_at)`.
- No queue, no cache, no replicas.

**Why this is enough:**

- 100 events/sec is 8.6M events/day. Postgres handles this with room. ~5 ms per insert; even at 100/sec you are using 50% of a single core.
- One table. ~30 GB/year at 500 B/event. Fits on disk.
- Reads are easy because writes are easy.

**Cost:** ~$50/month for the DB, ~$30/month for app hosting.

**What you do *not* build:** Kafka, Cassandra, S3 archiving, multi-region, sharding, batching. All premature.

The lesson: at 100 events/sec, the simplest possible system *is* the right system. Anything else is a portfolio piece, not engineering.

#### Stage 2: 10k events/sec (the company is logging more, or the product grew 100x)

**What just broke:**

- Postgres CPU is at 80% steady state. Spikes during deploy or analytics queries.
- WAL fsync is the bottleneck. Each commit fsyncs to disk.
- Inserts are landing in random pages (because the `(user_id, created_at)` index has high cardinality), so the buffer pool churns.
- App is doing one network round-trip per event. Even at 1 ms each, 10k of them is 10 cores of round-trip waiting.

**What you add:**

- **In-app buffering.** App accumulates events in memory; flushes every 100 ms or every 1000 events, whichever first. Flush is a Postgres `COPY` instead of `INSERT`, which is 10-20x faster than per-row INSERTs.
- **Partition the table by day.** `CREATE TABLE events_2026_05_23 PARTITION OF events_master FOR VALUES FROM ('2026-05-23') TO ('2026-05-24')`. Drops old partitions cheaply. Newer indexes stay small.
- **A second app instance, behind a load balancer.** Now the buffering happens on two nodes; the writes still go to one Postgres but each batch is bigger.
- **A read replica** for the analytics queries that were spiking the primary.

**What just broke that you accept for now:**

- **Buffered events are lost on app crash.** Worst case: 100 ms of events (~1000 events at 10k/sec). Tolerable for most use cases. Document it.
- **Slightly higher end-to-end lag.** Events are queryable within ~100 ms instead of immediately. Usually fine.

**What you do *not* yet build:**

- No Kafka. The buffer + batch trick gets you to 10k/sec on one Postgres.
- No Cassandra. Postgres still wins on operational simplicity at this scale.
- No multi-region. Single-region deployment is fine.

**Cost:** ~$500/month. The bigger Postgres instance dominates.

**Lesson:** stage 2 is where most teams overengineer. They jump to Kafka here because "we'll need it eventually." That is true at stage 3, not stage 2. The buffer-and-batch pattern buys you a 10x runway with minimal complexity.

#### Stage 3: 100k events/sec (you're now ingesting all server logs, or several products)

**What just broke:**

- Postgres can't keep up with 100k inserts/sec even with batching. The single instance is at IO ceiling. Vertical scaling has hit its limit (largest RDS Postgres instance does maybe 30k writes/sec).
- The app process holding the buffer is now a single point of failure with 1000 events in flight at any moment. Acceptable at 10k/sec; scary at 100k/sec because a restart loses 100k events worth of buffer.
- Read replica is lagging by 30 seconds because the primary is saturated with writes.
- A bad consumer query takes down the primary (which is also serving writes).

**What you add:**

- **Kafka in front of storage.** Events go to Kafka first; storage writes are async. Decouples the producer rate from the storage rate. Kafka buffers a backlog if storage falls behind. Producers get 202 immediately.
- **Switch storage to Cassandra.** LSM-tree is the right shape. Sequential writes. Scales horizontally by adding nodes; consistent hashing handles the partitioning. RF=3 across three AZs.
- **A separate Kafka consumer process** that reads events and writes to Cassandra in batched UNLOGGED BATCH per partition. This consumer is stateless; you run N of them for redundancy. Crashes don't lose data because Kafka holds the events.
- **First pass at partitioning.** Kafka topic with 50 partitions, hash by `(event_type, tenant_id)`. Cassandra primary key `((tenant_id, day), event_id)`.
- **First pass at observability.** Track Kafka consumer lag, partition rate, batch size, Cassandra write latency. You will use this constantly from here on.

**What just broke that you accept for now:**

- **End-to-end lag jumped from 100 ms to ~5 seconds.** Events go ingest → Kafka → consumer poll → Cassandra. Each step adds latency. Real-time consumers (fraud, alerting) read Kafka directly to avoid the storage lag.
- **Strong consistency is gone.** If two producers write events with the same event_id and the second is the "real" one, Cassandra's last-write-wins might give you the wrong one. Use a unique event_id (ULID or snowflake) to avoid the conflict.
- **One Kafka consumer crash = a few seconds of replay.** Idempotent storage handles the duplicates.

**What you do *not* yet build:**

- No multi-region (yet). All Kafka, all Cassandra, in one region.
- No cold tier (yet). Cassandra TTL deletes events after 90 days. Older events are gone unless someone needs them.
- No Flink. The Kafka consumer is doing all the processing.

**Cost:** ~$10-20k/month. Kafka cluster, Cassandra cluster (5-10 nodes), Kafka MSK + Cassandra DataStax or self-hosted.

**Lesson:** stage 3 is the inflection point. You introduced asynchrony, you accepted eventual consistency, you have an actual distributed system with multiple moving parts. The operational complexity went 10x. Worth it because the alternative is sharded Postgres which is its own nightmare.

#### Stage 4: 1M events/sec (you're logging every interaction across the company)

**What just broke:**

- Kafka has one cluster per region. The single-region setup hits NIC and disk limits on the cluster. Cross-region producers add latency.
- Cassandra cluster is 30 nodes. One node failure no longer self-heals fast enough; repairs take days.
- Hot tenant (one team's mobile app is misbehaving) is saturating one Kafka partition and one Cassandra node. Other tenants' writes are degraded.
- Cold-tier costs explode if you keep everything in Cassandra. 3.9 PB hot is expensive.
- Compliance asks for 7-year retention. Cassandra is wrong tier; it's not affordable at petabyte scale.

**What you add:**

- **Regional ingest + regional Kafka clusters.** Each region (us-east, eu-west, ap-south) has its own ingest tier and Kafka cluster. Events are written to the producer's home region. Cross-region replication for events that need to be globally visible (a separate "global" topic populated by an in-region MirrorMaker job).
- **Sharded Kafka topics by `event_type + tenant_id`.** Bigger tenants get their own topic (or their own partitions with high partition count). Long-tail tenants share a multi-tenant topic. Prevents one big tenant's burst from affecting unrelated tenants.
- **Sub-shard hot partitions with a random suffix.** When a partition's rate exceeds a threshold, the ingest tier starts appending `random(0..15)` to the partition key for that tenant. Trades per-key ordering for load distribution. Reversed when the spike ends.
- **Flink stream processing.** Reads from Kafka, does enrichment (geoip, user attribute join), routes outputs to multiple sinks (Cassandra for hot, S3 for cold, ClickHouse for rollups, Kafka for fraud alerts).
- **Tiered storage:** hot in Cassandra (90-day TTL), cold in S3 Parquet (7-year retention). A dedicated archiver consumer writes 5-minute Parquet batches. The archiver is a separate consumer group from the Cassandra writer; they each have their own Kafka offset.
- **Dedicated archiver moves cold partitions.** A nightly job lifts Cassandra partitions older than 90 days to Parquet (if not already there via the live archiver), then drops the Cassandra partition. Belt and suspenders: the live archiver should already have written them.
- **Per-tenant rate limits.** Hard cap at the ingest tier. Tenant who blows past their quota gets 429.

**What just broke that you accept for now:**

- **Cross-region replication lag is seconds to minutes.** A query that needs all-region data sees a slightly stale view.
- **Cold-tier queries are slow.** "Give me everything for tenant X for 2 years" runs in Athena and takes minutes. Acceptable for compliance and audit; not for interactive use.
- **Operational complexity is now significant.** You have a Kafka team, a Cassandra team, a Flink team. Or a small platform team that knows all of these.

**What you would do at 10x scale (10M events/sec):**

Honest answer: at 10M events/sec you are at the scale of a major observability vendor (Datadog, Splunk, Honeycomb). You stop building this in-house and either:

- Move to a fully managed platform (Kinesis Data Firehose + Iceberg on S3, BigQuery Storage Write API).
- Build very specialized infrastructure (custom Kafka forks, custom storage engines like Honeycomb's Retriever).

Or you split the workload by purpose: one pipeline for security events (strict durability, long retention), one for product analytics (loose durability, short retention, aggregated only), one for performance traces (sampled, very short retention). Different pipelines have different requirements; trying to make one pipeline serve all of them at 10M/sec is what kills the system.

**Cost at 1M events/sec:** roughly $200k-500k/month all-in, depending on managed vs self-hosted and storage retention choices. Bandwidth and storage dominate compute.

The lesson of the four stages: each stage solved a specific pain. None were preemptive. The most common failure mode in interviews is to build the stage-4 architecture for a stage-1 problem. The second most common is to refuse to evolve past stage 2 because "Postgres handles everything."

### 9. Reliability

**Queue overflow.**

Kafka cluster is at 90% disk. Producers continue sending. Three things to do:

1. **Backpressure at the ingest tier.** When Kafka send latency P99 exceeds 1 second or any broker is unhealthy, the ingest service starts returning 429 to producers. Producers back off. Total system protection.
2. **Shed less-important traffic.** Premium tenants get reserved Kafka partition capacity. Free-tier tenants get throttled first.
3. **Emergency disk add.** Kafka brokers can have storage expanded. Painful but possible. Time-bounded fix: 30-60 min.

**Downstream slow / consumer lag growing.**

Cassandra writer lag is at 5 minutes and climbing. Kafka is fine; Cassandra is the bottleneck.

1. **Scale Cassandra writer consumers horizontally.** Adding consumers in the same group spreads partitions across them. Quick win if the bottleneck is consumer CPU.
2. **Check Cassandra.** If Cassandra is at IO ceiling, you need more Cassandra nodes. Adding takes hours (data redistribution).
3. **Temporarily downshift.** Pause the archiver (it can catch up later from Kafka). Free up Kafka broker IO for the hot writer.
4. **Worst case: drop or sample.** If lag becomes unbounded, sample the events going to storage (write 1 in 10) until backlog clears. Audit-grade systems cannot do this; product-telemetry systems can.

**Partial batch loss.**

A Kafka consumer crashes after writing 800 of 1000 events to Cassandra but before committing the offset. On restart, it reprocesses the entire batch.

- Idempotent storage saves you. The 800 already-written events are no-op on re-insert (Cassandra UPSERT semantics with same primary key).
- The 200 not-yet-written events get written for the first time.
- No data loss, no corruption.

If the consumer crashed mid-Cassandra-batch (some events of an UNLOGGED BATCH applied, others not), Cassandra's behavior is "the batch is not atomic, individual events may or may not be applied." Same recovery: replay the batch on restart, idempotent storage handles the partial state.

**Ingest pod crash.**

Pod crashes between accepting a request and sending to Kafka. The producer's HTTP connection drops with no response. Producer must retry. With idempotency key, second submission is dedup'd; without idempotency key, you get a duplicate.

Mitigation: ingest pod should send to Kafka *before* returning 202. If Kafka send fails, return 503 (let producer retry).

**Region failure.**

Producers in failed region cannot publish. Their SDK should fail over to the next-nearest region. Events get stamped with their original `tenant_id` but written to a different region's Kafka. Cross-region replication eventually brings them back to the original region's storage if needed for locality.

Data loss window: events buffered in the producer's local SDK that were never sent. Typically <1 second of data.

**Cassandra node failure.**

RF=3 means surviving 1 node failure with no data loss; surviving 2 with degraded consistency (CL=ONE reads still work). Failed node is auto-replaced; data rebuilds from replicas over hours to days. Writes continue; reads might see slight latency increase on the affected partitions until repair completes.

**Total Kafka cluster loss.**

Catastrophic. Producers can't send. Data in flight is lost. Mitigations:

- **Multi-region Kafka.** Different cluster per region; failover is automatic.
- **MirrorMaker** for cross-region replication. The destination cluster gets data within seconds of the source.
- **Producer-side buffering.** Producers buffer events locally (disk-backed queue) when the broker is unreachable. Replay when broker recovers. Worth it for high-value events; expensive for high-volume telemetry.

### 10. Observability

| Metric | Why it matters | Alert threshold |
|--------|----------------|------------------|
| `ingest.requests.rate` | Top-line throughput | Drop >50% in 5 min = upstream broken |
| `ingest.latency.p99` | Producer-facing SLO | >100 ms for 5 min |
| `ingest.4xx.rate` | Producer misbehavior | >1% sustained |
| `ingest.5xx.rate` | Our problem | >0.1% sustained |
| `kafka.producer.ack.latency.p99` | Kafka health | >1s for 5 min |
| `kafka.partition.rate.max / median` | Partition skew (hot partition) | >10x ratio |
| `kafka.consumer.lag` per group | Are consumers keeping up? | >30s lag for any group |
| `kafka.broker.disk.used.pct` | Storage capacity | >80% |
| `cassandra.write.latency.p99` | Storage health | >50 ms |
| `cassandra.batch.size.p99` | Are we batching effectively? | <10 events = too small |
| `cassandra.partition.size.max` | Hot partition at storage layer | >100 MB |
| `archiver.lag` | Cold tier behind real-time | >10 min |
| `archiver.parquet_file.size.p50` | Small file problem? | <10 MB = too many small files |
| `flink.checkpoint.duration.p99` | Stream processing health | >30 sec |
| `dedup.collision.rate` | Are we seeing duplicate event_ids? | Any non-zero is interesting |

**The four golden signals for the pipeline:**

1. **Ingest throughput** (events/sec at the API).
2. **End-to-end lag** (event ingest timestamp to Cassandra-readable timestamp).
3. **Consumer lag** per consumer group on Kafka.
4. **Error rates** at each stage.

**Distributed tracing.**

Each event gets a trace ID stamped at ingest. The trace follows it through Kafka → consumer → Cassandra. Lets you debug "why did this specific event take 2 minutes to land" without grepping through gigabytes of logs.

Sampling: trace 0.1% of events end-to-end. The rest you just have metrics for.

**Alerting cadence:**

- Page: ingest 5xx > 0.5%, ingest p99 > 200 ms for 10 min, consumer lag > 5 min for any group, Kafka broker disk > 90%.
- Ticket: partition skew > 10x for 1 hour, parquet file size < 10 MB sustained, dedup collisions > 100/hour.
- Dashboard only: throughput, latency, error rates by tenant, batch sizes.

### 11. Common gotchas

**Back-pressure done wrong.**

If you don't have back-pressure at the ingest tier, a downstream slowness becomes a cascading failure. Producers keep retrying; the retry storm doubles or triples the load; the cluster collapses. The correct response is to start refusing new traffic (429) when downstream is unhealthy, not to absorb infinite buffer until you fall over.

**Hot partition.**

One key (a tenant, a user, a topic) gets 100x the traffic of others. The partition owning that key is saturated; the rest are idle. You have a 10-node cluster doing the work of 1 node.

Detection: per-partition rate metric. Alert on `max / median > 10x`.

Mitigation:
- Sub-shard the hot key with a random suffix (`(event_type, tenant_id, random(0..15))`). Spreads across 16 partitions. Cost: per-key ordering is broken for that tenant.
- Per-tenant rate limit at the ingest tier. The offending tenant gets 429 above a threshold; their problem to fix.
- Dedicated capacity for premium tenants; long-tail in a shared pool. Naturally isolates.

**Clock skew.**

Producer A's clock is 3 minutes ahead. Producer B's clock is 30 seconds behind. Events from A look like they happened in the future; events from B look out of order.

Mitigations:
- **Always stamp server time.** Producer time is kept for the user's reference; server time is the source of truth for ordering and bucketing.
- **NTP everywhere.** Reduces skew to milliseconds; not zero but workable.
- **Late-event tolerance in stream processing.** Flink allows a "late arrival" window per partition. Events arriving 1 minute "late" relative to wall clock still get processed into the correct time bucket.
- **Detect outliers.** Events with producer_ts > server_ts + 5 min get flagged. Usually indicates a misconfigured client.

**Duplicate events.**

Producer retried after a broker timeout that actually succeeded. Now Kafka has two copies of every event in the retried batch.

Detection: monitor `event_id` collision rate at the storage layer. Cassandra primary-key conflict on insert is your "duplicate" signal.

Dedup:
- **Same event_id = same event.** Use a deterministic event_id (ULID with the producer's source info). Second insert is a no-op.
- **Kafka idempotent producer.** Prevents duplicates within a producer session. Doesn't help across producer restarts.
- **Bloom filter at the consumer.** Keep recent event_ids in a Bloom filter; reject duplicates. Cheap, probabilistic.

**Tombstone proliferation.**

Cassandra deletes don't free space immediately; they write tombstones that get cleaned up at compaction. If you do many deletes (or use a high TTL), tombstones can dominate the read path and slow queries dramatically.

Mitigation: avoid deletes in the write-heavy path. Use TTL on the table (declarative expiration). Run regular compaction.

**Small file problem on S3.**

Archiver flushes too often → many small Parquet files. Each Athena query has to open thousands of files. Query becomes slow and expensive.

Mitigation:
- Bigger archiver batches (5+ minutes, 100k+ events).
- Periodic compaction job: merge files smaller than 64 MB into larger files. Standard Iceberg/Delta Lake feature.

**Producer-side buffering bugs.**

Producer SDK buffers events to coalesce network calls. Producer process crashes. Buffer lost. User thought their event was sent; it wasn't.

Mitigations:
- Disk-backed producer buffer for critical events. Slow but durable.
- Sync mode for critical events (wait for ack before returning).
- For telemetry-grade events, accept the loss and document it.

**Re-partition rebalance storm.**

You add a Kafka consumer to a consumer group. Kafka rebalances; every consumer briefly pauses, rejoins, picks up new partitions. During the rebalance (5-30 seconds), no consumption happens. Lag spikes.

Mitigation:
- Use cooperative rebalancing (Kafka 2.4+) instead of eager rebalancing. Only the moving partitions pause.
- Scale consumers during low-traffic windows when possible.
- Static membership (`group.instance.id`) for long-running consumers; reduces rebalance frequency on transient restarts.

### 12. Follow-up answers

**1. Back-pressure.**

The ingest service checks Kafka health (broker reachability, producer ack latency, consumer lag for any group falling behind). When any of these crosses thresholds, the ingest service starts returning 429 with `Retry-After`. Producers back off and retry with exponential backoff.

The producer SDK respects 429 and 503; it caches events locally (in-memory or disk-backed) and replays when health returns. Without producer-side backoff, the 429 storm just becomes a retry storm; the SDK has to actively reduce send rate.

If Kafka is so unhealthy that even 429s are slow, the ingest tier can shed load by simply closing connections (TCP RST), forcing producers to back off at a lower level. Brutal but protects the cluster.

**2. Hot tenant.**

Find them: the per-partition rate metric points at one Kafka partition. Cross-reference with the partition key (tenant + event_type). The dashboard shows tenant_42 at 200k events/sec when average is 100/sec.

Next 5 minutes:
- Apply a hard rate limit at the ingest tier: tenant_42 capped at 50k events/sec. Excess gets 429. Their misbehaving client sees errors; their well-behaved clients continue working.
- If Kafka partition is still saturated, sub-shard them: start hashing `(tenant_42, random(0..15))` for their events. Spreads across 16 partitions. They lose per-key ordering during the spike but the cluster survives.

Next 5 days:
- Contact tenant_42. Diagnose the bug (often a retry loop, a log misconfiguration, or a runaway script).
- Establish a per-tenant quota policy with documented limits.
- Build per-tenant isolation: premium tenants get reserved Kafka partition capacity; long-tail in a shared pool. Hot tenants get auto-detected and isolated to their own partitions.

**3. Clock skew.**

The fix is structural, not "make every producer perfect."

- **Always stamp server time at the ingest tier.** That is the timestamp used for partitioning, bucketing, and ordering downstream. Producer time is kept for user-facing display.
- **NTP-sync producers to the second.** Achievable everywhere; reduces skew enough that producer_ts is useful as a sanity check.
- **Detect outliers.** Any event with producer_ts more than 5 minutes off from server_ts is logged as suspicious. Usually identifies a misconfigured client (factory-default time, wrong timezone).
- **Stream processor tolerates late events.** Flink watermarks define a "lateness budget" (typically 1 minute); events arriving within the budget still land in their correct time bucket.

The trap: do not use producer time for anything that requires precise ordering. Use server time always.

**4. Duplicate events.**

Detection:
- Cassandra primary-key conflict on insert is the most direct signal. The consumer counts `cassandra.dedup.collisions` per minute. Spikes mean producers are retrying frequently.
- A separate audit: nightly job runs `SELECT event_id, count(*) FROM events GROUP BY event_id HAVING count(*) > 1`. Should be zero. Actually it should never run because Cassandra's primary key prevents duplicates at insert time, but the audit catches schema-level dedup bugs.

Dedup:
- The `event_id` is the dedup key. Generated by the producer if possible (ULID is good: time-ordered, unique, no coordination needed). If absent, server generates one at ingest, but then producer retries get *different* event_ids and dedup is impossible. Always prefer producer-generated event_ids.
- Cassandra UPSERT semantics (default for INSERT) make duplicate inserts a no-op for the same primary key. No explicit "if not exists" needed.
- For at-most-once semantics at the consumer (don't process a dup), a Bloom filter of recent event_ids in the consumer. Memory bounded, probabilistic.

**5. Consumer fell behind.**

Kafka has 1.8 billion events buffered. Consumer restarts. Two failure modes to avoid:

- **Resource exhaustion at the storage layer.** Consumer wakes up, polls Kafka at full speed, sends 1.8B events to Cassandra as fast as Cassandra accepts them, melts the cluster.
- **Out-of-order processing of derived data.** If the consumer was computing rolling metrics, 1.8B events arriving in the next hour skews every metric for that hour.

Strategies:
- **Throttled catch-up.** Configure the consumer to ingest at 2x normal throughput, not unbounded. Catch-up takes longer but storage stays healthy. At 100k/sec normal, 200k/sec catch-up, 1.8B events take 2.5 hours to drain.
- **Parallel consumers.** Scale the consumer group up temporarily. More consumers consume more partitions in parallel. Watch storage; don't scale past what storage can absorb.
- **Watermark-aware downstream.** If derived consumers care about "now-time" processing, give them a flag to skip events older than X. Bulk-replay the historical events into a separate batch job (Spark on the Kafka backlog) that writes directly to Cassandra without going through the live processor.

The general principle: catch-up is a separate workload. Don't pretend it's normal traffic.

**6. Schema evolution.**

Old producers send 5 fields. New producers send 6. Cassandra has 5 columns plus a JSON blob column for the rest.

Strategies for the schema migration:

- **Schema-flexible storage.** Keep raw event as JSON (or in a JSON column). Use Avro / Protobuf with backward/forward compatibility rules. Old consumers ignore unknown fields; new consumers know the field is optional for old records.
- **Schema registry** (Confluent Schema Registry or similar). Producers register their schema version; consumers fetch the schema and parse correctly. Compatibility checks enforced at registration.
- **Versioned event type.** `user.login.v1`, `user.login.v2`. Old consumers stick to v1; new ones handle v2. Trade-off: data fragmentation across versions.
- **Cassandra schema add.** ALTER TABLE adds a column. Old data has NULL for that column; new data has the value. Generally safe but coordinate with rolling consumer deploys.

The discipline: never make a backward-incompatible schema change without versioning. Adding fields is fine. Removing or renaming is not without a migration plan.

**7. Recent-event reads.**

Three real approaches:

- **Document the lag.** "Data has ~5s lag." Most use cases accept this. Cheapest option.
- **Dual write to a real-time store.** A second consumer reads Kafka and writes to Redis (per-user list, capped at 1000 recent events). Queries that need <1s freshness hit Redis; older data falls through to Cassandra. Costs 2x writes at the consumer tier.
- **Query Kafka directly.** Use kSQL or a Pulsar query function to filter Kafka topics by `user_id`. Operationally awkward; Kafka is not a key-value store. Works only for narrow windows.

For most audit pipelines, option 1 is correct. For fraud or live-dashboard use cases, option 2 is worth the cost.

**8. Region down.**

If the producer SDK is well-designed:
- Producer detects local Kafka cluster unreachable.
- Buffers events locally (in-memory or disk).
- Fails over to next-nearest region's Kafka.
- When local cluster recovers, drains buffer to local Kafka.

Data loss window: events in producer buffer at the moment the producer process dies. Typically <1 second.

If the producer SDK is naive (no failover):
- Producer gets timeouts on Kafka send.
- Producer either accepts the loss or holds the events in memory until OOM.
- All events generated during the outage are lost.

DR play: cross-region Kafka replication (MirrorMaker) ensures the events that did land in the failed region's Kafka eventually replicate elsewhere. When the failed region comes back, consumers replay from Kafka; downstream storage catches up.

The hard part: if writes were already in flight to Cassandra in the failed region, those Cassandra writes are stuck. Cross-region Cassandra replication (RF spread across regions) handles this for the events that were already in storage. For events still in Kafka when the region failed, they sit in Kafka until the region recovers.

For zero-data-loss DR: write events to two Kafka clusters synchronously. Doubles producer cost but guarantees survival of one region's loss.

**9. Cold-tier query cost.**

"Every event for tenant X for 3 years" scans naively across 3 years × 365 partitions × all event_type subdirectories. Even with partition pruning by tenant, that is a lot of files.

Optimizations:

- **Partition pruning by tenant_id at the prefix level.** Path is `date=Y/event_type=X/tenant_id=Z/`. Athena prunes paths that don't match. Critical: tenant_id is in the path, not buried inside the Parquet file.
- **File-level statistics.** Parquet's per-row-group statistics let Athena skip row groups that can't match the filter. Add a min/max on event_id, user_id, etc.
- **Pre-aggregated cube for hot queries.** If "events per tenant per day" is a common query, materialize it nightly into a small summary table. Original Parquet stays for ad-hoc; the summary serves the dashboard.
- **Cold-cold storage.** Events older than 1 year move to S3 Glacier or Glacier Deep Archive. Queries against deep archive take 12-48 hours to retrieve files, but cost 1/20 of standard S3. Acceptable for "compliance asks for an event from 2 years ago."
- **Tenant-specific extracts.** When a tenant requests their data (GDPR or audit), pre-build a single Parquet extract for them. Subsequent queries hit the extract, not the full lake.

For interactive queries on 3 years of data: build an OLAP layer (ClickHouse, BigQuery, or Snowflake) and pre-load the data there. S3 Parquet via Athena is for ad-hoc, not for sub-second analytics.

**10. Exactly-once for a side effect.**

SMS is non-idempotent. You can't dedup at the receiver. So you have to ensure the side effect runs exactly once.

Pattern:

- **Outbox table.** Consumer reads Kafka events, writes "send SMS" records to a local DB table `pending_sms` keyed by event_id, in the same transaction as the Kafka offset commit. Wait: Kafka offsets and DB transactions don't share a transaction by default. So:
- **Transactional outbox + idempotent sender.** Consumer writes to `pending_sms` keyed by event_id; the row has `status = 'pending'`. A separate sender process picks pending rows, calls the SMS API with `idempotency_key = event_id` (the SMS provider must support idempotency keys; Twilio does, for instance), then marks the row `status = 'sent'`.
- **Crash safety.** If the sender crashes between API call and status update, on restart it sees the row as still pending and retries. The SMS provider's idempotency key prevents a second SMS.

Cost: one extra DB write per event. Adds maybe 5 ms per event at the consumer. Cheap insurance.

The general lesson: when you need exactly-once at a specific downstream, use a transactional outbox at that consumer rather than paying Kafka EOS cost on the whole pipeline. The expensive guarantee is local to where you need it.

### 13. Trade-offs and what a senior would mention

- **Sync vs async.** Sync writes (producer waits for storage) give you immediate consistency and simpler reasoning. They cap your throughput at the storage layer's speed. Async (via queue) decouples producer rate from storage rate; you get higher throughput at the cost of end-to-end lag and weaker consistency. The right call depends on the lag SLA. <100 ms = sync. >1 s acceptable = async.
- **Batch vs stream.** Batching amortizes per-write overhead (TLS, fsync, network round-trip). Larger batches = higher throughput but higher latency per event. The sweet spot is "batch enough to amortize, small enough to land within the lag SLA." Typical: 100 ms or 1000 events.
- **Exactly-once cost.** Kafka EOS drops throughput 20-40%. Add operational complexity. The right move is usually "at-least-once with idempotent storage" which is functionally equivalent at the storage layer for 10x less complexity. Reserve true exactly-once for non-idempotent side effects (SMS, payment processing).
- **Why Cassandra over Postgres at the hot tier.** Postgres B-tree is bad at write-heavy random-key inserts; Cassandra LSM is built for it. At <50k writes/sec, Postgres is fine and operationally simpler. Above that, Cassandra wins on throughput per node. At scale, Cassandra also gives you easier horizontal scaling (add a node, rebalance automatically) vs Postgres which requires manual sharding.
- **Why Kafka and not just Kinesis / Pulsar / NATS.** Kafka has the largest ecosystem, the deepest tooling, the best operational documentation. Kinesis is simpler to operate (fully managed) but harder to migrate off. Pulsar has a better multi-tenant story (per-tenant namespaces). NATS JetStream is lighter weight. At 1M events/sec, all of them work; pick what your team can operate.
- **Why S3 Parquet for cold tier.** Cheap, durable, columnar, queryable with serverless tools (Athena/BigQuery). The alternative (keep everything in Cassandra) costs 100x more for events that are accessed once a year.
- **What I would revisit at 10M events/sec.** Specialized vendor (Datadog, Splunk, Honeycomb) or specialized infrastructure (Honeycomb's Retriever, ClickHouse for everything). Or split into purpose-specific pipelines (security, analytics, traces). The "one pipeline for everything" pattern stops scaling cleanly past ~1M events/sec.

### 14. Common interview mistakes

1. **Reaching for Kafka and Cassandra immediately.** The interviewer started you at 100 events/sec. If you immediately propose a Kafka cluster, you have not earned the architecture. Walk the stages. Justify each addition with a specific failure mode of the previous stage.

2. **Confusing buffering with batching.** Buffering means holding events in memory before writing. Batching means writing many events in one storage call. You usually do both. Articulate the difference.

3. **Not addressing partitioning.** "Use Cassandra" without specifying the partition key is a hand-wave. The partition key determines whether your cluster has hot nodes; it is the most important decision in the storage layer.

4. **Claiming exactly-once without paying the cost.** Senior interviewers will probe. If you say "exactly-once," you must explain how (Kafka EOS, transactional outbox) and what it costs. If you say "at-least-once with idempotent storage," you score the same correctness with a tenth the operational pain.

5. **Forgetting back-pressure.** A write-heavy system without back-pressure cascades on the first downstream slowness. The 429 / retry-with-backoff loop is mandatory.

6. **Not designing for the hot partition.** "It will distribute evenly" is a hope, not a design. Real systems have one tenant 1000x bigger than the median. Have an answer (sub-sharding, rate limiting, dedicated capacity).

7. **No tiered storage.** Keeping 7 years of events in Cassandra is wasted money. Cold tier (S3 Parquet) is the standard play. Mention it.

8. **Treating lag as a bug.** End-to-end lag is the *price* you pay for async ingestion. Some consumers want lag (cheap batch processing). Some don't (fraud alerts). Acknowledge the spectrum.

9. **Ignoring observability.** "We'll add monitoring later" loses senior-level credit. Queue depth, consumer lag, batch size, and partition skew are the metrics that diagnose every common failure. They are part of the design, not an afterthought.

10. **Building the 1M/sec architecture at 100/sec scale.** Junior engineers love this mistake. The system collapses under its own operational weight before it ever sees the traffic it was built for. The discipline is "design for one stage past current, not three."

If you hit 7 of these 10 cleanly, you are interviewing well. The two that separate staff-level candidates from senior: addressing the hot partition problem unprompted, and explaining the exact moment you would *not* yet introduce Kafka (the stage 1 → 2 transition). Both demonstrate that you understand cost as well as throughput.
