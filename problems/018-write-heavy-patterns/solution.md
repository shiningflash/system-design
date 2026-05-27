## Solution: Write-Heavy System Patterns

### The short version

A write-heavy system is one where the cost comes from taking in data, not from serving it. Each event is small. Queries are simple. The storage engine has one job: absorb writes faster than a normal database can.

The interesting work is at the seams. When do you add a buffer? When do you add a queue? When do you give up linearizability? When do you stop adding pieces because what you have is already enough?

The toolkit, in the order you reach for it as scale grows:

1. **One Postgres** at ~1k writes/sec.
2. **In-memory buffer + batched commits** at ~10k writes/sec.
3. **Durable queue + async LSM writer** at ~100k writes/sec.
4. **Partitioned queue + stream processing + tiered storage** at 1M+ writes/sec.

Each step trades latency, consistency, or operational complexity for throughput. The senior move is to name the limit of stage N *before* reaching for stage N+1. Not to preemptively build stage 4.

Three big decisions shape the final design:

- **Partition key.** Time, hash, or tenant. Usually a composite like `(event_type, tenant_id, day)`.
- **Storage engine.** LSM at the hot tier. Parquet on object storage at the cold tier.
- **Delivery guarantee.** At-least-once with idempotent storage is right for 95% of cases.

---

### 1. The two questions that matter most

**How much loss is OK?** This decides your whole durability story. Zero loss means `acks=all` on Kafka, transactional ingest, and idempotent consumers. 0.1% loss acceptable means fire-and-forget, no retries, no dedup. Those are 10x apart in cost and complexity.

**How fresh must reads be?** 100ms freshness forces synchronous indexing. 5-minute lag lets you batch into Parquet files and pay 1% of the cost. Most audit systems accept 5-10 seconds. Most fraud detectors want under 1 second. The lag SLA decides where you put the queue and how big your batches can be.

Everything else (partition strategy, storage engine, delivery guarantee) follows from those two answers.

---

### 2. The math, in plain numbers

| Number | Value | Why it matters |
|--------|-------|----------------|
| Steady bandwidth | 500 MB/sec (4 Gbps) | Multiple ingest nodes needed just for NIC, not CPU |
| Peak bandwidth | 1.5 GB/sec (12 Gbps) | Several machines per region just to accept bytes |
| Daily volume | 86 billion events, 43 TB raw | Hot tier is petabytes |
| Hot storage (90 days) | ~3.9 PB | 10-30 node Cassandra cluster, RF=3 |
| Cold storage (7 years, 5x compression) | ~22 PB on S3 | ~$500k/month standard, ~$25k/month Glacier Deep |
| Queue size on 5-min stall | 300M events, 150 GB | Kafka handles easily; in-memory cannot |

The headline: bottleneck is bandwidth and storage layout, not CPU. A single Postgres maxes out at 10k-50k writes/sec. At 1M/sec we are 100x past that on a single shard. We need to spread writes across nodes and batch them so per-write overhead amortizes.

Always do bytes alongside event counts. A pipeline doing 86 billion events of 50 bytes each is a completely different system from one doing 86 billion events of 5 KB each.

---

### 3. The API

Reads are out of scope. Focus on ingest.

**Single-event ingest** for low-volume producers:

```
POST /api/v1/events
Content-Type: application/json
X-Tenant-Id: tenant_42
Authorization: Bearer <token>
Idempotency-Key: <uuid>         # optional; server dedupes on this key if present

{
  "event_type": "user.login",
  "user_id": "u_8201",
  "timestamp": "2026-05-24T10:14:02.331Z",
  "attributes": { ... }
}
```

| Status | Meaning |
|--------|---------|
| **202 Accepted** | Event in durable queue, queryable within ~10s |
| **400 Bad Request** | Schema invalid |
| **413 Payload Too Large** | Single event > 1 MB |
| **429 Too Many Requests** | Rate limited or back-pressure |
| **503 Service Unavailable** | Ingest degraded, producer should retry |

**Batched ingest** for everyone else:

```
POST /api/v1/events/batch
Content-Encoding: gzip
Content-Type: application/json

{
  "events": [
    { ...event 1... },
    { ...event 2... }
  ]
}
```

Four load-bearing choices:

- **202, not 200.** The event is in durable queue, not yet in storage. 202 communicates "we accepted responsibility; you can stop retrying."
- **Batch endpoint is recommended.** Per-event HTTP at 1M/sec means 1M TLS handshakes/sec, which is impossible. Batches of 1000 cut that to 1k req/sec.
- **Server stamps server_ts** alongside producer_ts. Both are kept. Producer time is "when did the event happen." Server time is "when did we accept it." Clock-skew analysis uses the gap.
- **Rate limits are visible.** 429 includes `Retry-After` and `X-RateLimit-Remaining` so well-behaved producers throttle themselves.

---

### 4. The data model

**The canonical event:**

```json
{
  "event_id":      "01H8K2X...",
  "tenant_id":     "tenant_42",
  "event_type":    "user.login",
  "user_id":       "u_8201",
  "producer_ts":   "2026-05-24T10:14:02.331Z",
  "server_ts":     "2026-05-24T10:14:02.412Z",
  "attributes":    { ... },
  "schema_version": 3
}
```

**Hot tier in Cassandra:**

```cql
CREATE TABLE events_by_tenant_day (
    tenant_id      text,
    day            date,
    event_id       timeuuid,
    event_type     text,
    user_id        text,
    producer_ts    timestamp,
    server_ts      timestamp,
    attributes     text,
    schema_version int,
    PRIMARY KEY ((tenant_id, day), event_id)
) WITH CLUSTERING ORDER BY (event_id DESC)
  AND default_time_to_live = 7776000;
```

Three things doing real work:

**`PRIMARY KEY ((tenant_id, day), event_id)`.** The partition key `(tenant_id, day)` bounds partition size. Even the biggest tenant only writes one day at a time into one partition. The clustering column `event_id` keeps rows sorted inside the partition, so "latest events" is a prefix scan.

**`default_time_to_live = 7776000`.** 90 days. Cassandra auto-deletes old rows during compaction. No nightly delete job needed.

**No foreign keys.** Audit-shaped data is reference-free. The event captures who/what/when. If a user account is later deleted, the audit must survive.

**Cold tier on S3** sits at path `s3://events-cold/date=YYYY-MM-DD/event_type=X/tenant_id=Y/fileN.parquet`. Columnar layout inside each file. Column-pruned reads are free. Partition pruning at the prefix level means Athena/Presto skip whole directories that do not match the query.

---

### 5. The six patterns, fast

#### Pattern 1: Batching

Accumulate events in memory. Flush together. One `COPY` statement covers what used to be 1,000 individual `INSERT` calls. Amortizes fsync, network round-trips, and TLS overhead.

**Limit.** ~10k writes/sec on one Postgres, and events in the buffer are lost on crash.

```mermaid
flowchart LR
    P([Producer]):::user --> BF["Buffer<br/>(100ms or 1000 events)"]:::app
    BF --> CP["COPY batch"]:::app --> DB[("Postgres")]:::db

    classDef user fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef app  fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db   fill:#fed7aa,stroke:#c2410c,color:#7c2d12
```

> **Take this with you.** Batching is the first 10x. It costs nothing but a timer and an array. Use it before reaching for Kafka.

---

#### Pattern 2: Append-only log

Only ever append. Never update in place. All writes are sequential. Sequential disk I/O is 10x-100x faster than random I/O.

The audit trail becomes a stream of immutable facts. Current state is derived by replaying events. This is how Kafka, Cassandra's commit log, and every good audit system work.

```mermaid
flowchart LR
    E1["event: login, ts:10:01"]:::db -.-> E2["event: logout, ts:10:05"]:::db -.-> E3["event: login, ts:10:09"]:::db

    classDef db fill:#fed7aa,stroke:#c2410c,color:#7c2d12
```

**Limit.** Storage grows unboundedly. Reads that want current state must aggregate across all events. Fix with tiered storage and materialized views.

---

#### Pattern 3: Queue-based ingestion

Decouple producer speed from storage speed. Producers write to a durable queue. Storage drains at its own pace. Multiple consumers read the same events independently.

```mermaid
flowchart LR
    P([Producer]):::user --> K{{"Kafka"}}:::queue
    K --> HW["Hot writer"]:::app --> CS[("Cassandra")]:::db
    K --> AR["Archiver"]:::app --> S3[("S3")]:::db
    K --> FW["Fraud alerts"]:::app

    classDef user  fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

**Limit.** End-to-end lag jumps from ms to seconds. If consumer lag grows past Kafka's retention window, events are lost. Fix with Kafka disk retention sized for your worst-case stall (7 days is standard).

> **Take this with you.** The queue is the shock absorber. If the notification service dies at 3am, ingest still works. Events just queue up.

---

#### Pattern 4: LSM trees

Postgres uses a B-tree: every insert finds a page, writes in-place, pays a random I/O cost. Cassandra uses an LSM tree: every insert appends to a commit log (sequential), then flushes to an SSTable (sequential). Compaction happens in the background, off-peak.

```mermaid
flowchart LR
    subgraph BTree["B-tree insert (Postgres)"]
        direction TB
        BW["Random read + random write<br/>per insert<br/>fsync on every commit"]:::bad
    end
    subgraph LSM["LSM insert (Cassandra)"]
        direction TB
        LW["Sequential append to commit log<br/>flush to SSTable when memtable fills<br/>compaction in background"]:::ok
    end

    classDef ok  fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef bad fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
```

**When LSM wins.** Write-heavy work, time-series, logs, telemetry. Above ~50k writes/sec on a single node, Cassandra/ScyllaDB/RocksDB are the right choice.

**When B-tree wins.** OLTP with frequent updates to the same rows. Read-heavy. Many secondary indexes. Postgres, MySQL.

---

#### Pattern 5: Sharding and partition keys

Spread writes across many nodes. The partition key decides which node gets which writes. A bad key creates a hot node that gets 100x the traffic of others.

| Strategy | Good for | Fails when |
|----------|----------|------------|
| **Time-based** (`day`) | Time-range reads, cheap data expiry | Hot write partition (current hour/day) |
| **Hash-based** (`hash(key) % N`) | Even write distribution, no hotspots | Range queries scatter-gather |
| **Tenant-based** (`tenant_id`) | Multi-tenant isolation, per-tenant deletion | One big tenant melts one node |
| **Composite** (`(tenant_id, day)`) | Bounds partition size, cheap expiry, good reads | Slightly more complex routing |

For 1M events/sec: Kafka key = `hash(event_type, tenant_id)`. Cassandra primary key = `((tenant_id, day), event_id)`. S3 path = `date=Y/event_type=X/tenant_id=Z/`.

> **Take this with you.** The partition key is the most important design decision in write-heavy storage. A wrong key gives you a hot node. A hot node negates all the scaling work.

---

#### Pattern 6: Tiered storage

Not all data is accessed equally. Last hour's events: read often, must be fast. Three-year-old events: read rarely, must be cheap. Match storage cost to access frequency.

```mermaid
flowchart LR
    K{{"Kafka<br/>(ingest)"}}:::queue --> CS[("Cassandra<br/>hot, 90-day TTL")]:::db
    CS -.nightly.-> S3[("S3 Parquet<br/>cold, 1-7 years")]:::db
    S3 -.after 1 year.-> GL[("S3 Glacier<br/>deep archive")]:::db
    CS --> Q1["Point reads<br/>10ms"]:::app
    S3 --> AT["Athena<br/>seconds-minutes"]:::app
    GL --> RC["Bulk retrieval<br/>12-48h"]:::app

    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

**Cost comparison (1M events/sec at 500 bytes):**
- Cassandra hot tier (90 days): ~3.9 PB, ~$50k/month on managed clusters.
- S3 Parquet standard (7 years, 5x compression): ~22 PB, ~$500k/month.
- S3 Glacier Deep Archive: ~$25k/month for same 22 PB.

Keeping 7 years in Cassandra is not just expensive. It is structurally wrong: Cassandra is optimized for fast point reads, not for Athena-style columnar scans across years of data.

---

### 6. The architecture

```mermaid
flowchart TB
    subgraph Edge["Client edge"]
        P([Web / Mobile / Server]):::user
        GW["API Gateway<br/>(TLS, rate limit, back-pressure)"]:::edge
    end

    subgraph WritePath["Synchronous write path"]
        IS["Ingest Service<br/>(stateless, N pods)"]:::app
    end

    K{{"Kafka<br/>200 partitions, RF=3<br/>hash(event_type, tenant_id)<br/>7-day retention"}}:::queue

    subgraph Consumers["Async consumers (independent consumer groups)"]
        HW["Hot writer<br/>(UNLOGGED BATCH<br/>per partition)"]:::app
        AR["Archiver<br/>(buffer 5 min<br/>write Parquet)"]:::app
        FL["Flink<br/>(rollups, alerts)"]:::app
    end

    CS[("Cassandra<br/>hot tier, RF=3<br/>90-day TTL")]:::db
    S3[("S3 Parquet<br/>cold tier")]:::db
    AT["Athena / Presto"]:::app
    OB["Prometheus + Grafana<br/>(lag, skew, batch size)"]:::cache

    P --> GW --> IS --> K
    K --> HW --> CS
    K --> AR --> S3 --> AT
    K --> FL
    IS & K & HW & AR -.metrics.-> OB

    classDef user  fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef edge  fill:#e2e8f0,stroke:#475569,color:#1e293b
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

Five things to notice:

- Ingest pods are stateless. Their only durability hop is Kafka. A pod crash causes the producer's connection to drop. The producer retries. Idempotency key (if present) deduplicates.
- Three consumer groups read the same Kafka topic independently. Archiver falls behind without affecting Cassandra. Flink crashes without affecting the archiver.
- Cassandra and S3 are not redundant. They serve different access patterns. Cassandra answers "give me events for tenant X on day D" in 30ms. S3 answers "give me everything for tenant X over 2 years" in minutes.
- Observability is on every box. Without consumer lag and partition-skew metrics, you learn about failures from customer complaints.
- Concrete choices: Edge LB via AWS ALB or Cloudflare. Ingest pods in Go or Rust (low GC overhead). Kafka via Confluent, MSK, or Aiven. Hot store: Cassandra or ScyllaDB. Flink for stateful stream processing.

---

### 7. An event, end to end

```mermaid
sequenceDiagram
    autonumber
    participant P as Producer
    participant E as Edge LB
    participant I as Ingest
    participant K as Kafka
    participant HW as Hot Writer
    participant Cs as Cassandra
    participant A as Archiver
    participant S as S3

    P->>E: POST /events/batch
    E->>I: TLS, rate limit check
    I->>I: validate, stamp event_id and server_ts
    I->>K: produce (acks=all)
    K->>K: replicate to ISR
    K-->>I: ack
    I-->>E: 202 Accepted
    E-->>P: 202 Accepted
    Note over P,K: sync slice ends (P99 ~50ms)

    K-->>HW: poll
    HW->>Cs: UNLOGGED BATCH per partition
    Cs-->>HW: ack
    HW->>K: commit offset

    K-->>A: poll (separate consumer group)
    Note over A: buffer 5 min or 100k events
    A->>S: PUT Parquet file
    S-->>A: ack
    A->>K: commit offset
```

**End-to-end lag:**
- Producer to Cassandra-queryable: 1-3 seconds, P99 ~10 seconds.
- Producer to S3 Parquet: up to 5-6 minutes (archiver flush window dominates).

**Read latencies:**
- Point read by event_id: P99 ~10ms.
- Per-tenant range scan: P99 ~30ms.
- Analytics over 1 year via Athena: seconds to minutes.

---

### 8. The scaling journey: 100 events/sec to 1M

```mermaid
flowchart LR
    S1["Stage 1<br/>100 events/sec<br/>1 Postgres + 1 pod<br/>~$80/mo"]:::s1
    S2["Stage 2<br/>10k events/sec<br/>buffer + batch COPY<br/>~$500/mo"]:::s2
    S3["Stage 3<br/>100k events/sec<br/>Kafka + Cassandra<br/>~$10-20k/mo"]:::s3
    S4["Stage 4<br/>1M+ events/sec<br/>regional ingest<br/>tiered storage<br/>~$200-500k/mo"]:::s4

    S1 --> S2 --> S3 --> S4

    classDef s1 fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef s2 fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef s3 fill:#fef3c7,stroke:#a16207,color:#713f12
    classDef s4 fill:#fce7f3,stroke:#be185d,color:#831843
```

#### Stage 1: 100 events/sec

One Postgres (db.t3.medium, 4 GB RAM). One app instance. One table, indexed by `event_id` and `(user_id, created_at)`. No queue. No cache. No replicas. ~$80/month.

Enough because 100 events/sec is 8.6M events/day. Postgres handles this with room to spare. At 5ms per insert, using maybe 50% of one core. Anything more is over-engineering.

#### Stage 2: 10k events/sec

**What breaks.** Postgres CPU at 80%. WAL fsync is the bottleneck: every commit flushes to disk. Inserts land in random pages, so the buffer pool churns.

**Fixes:**

- In-app buffer: accumulate events, flush every 100ms or 1000 events via Postgres `COPY` instead of `INSERT`. 10-20x faster.
- Partition the table by day (`PARTITION OF ... FOR VALUES FROM ...`). Old partitions drop cheaply.
- Second app instance behind a load balancer.
- Read replica for analytics.

**What you accept.** Buffered events lost on app crash, worst case 100ms x 10k/sec = ~1,000 events. Tolerable for most use cases. End-to-end lag goes from immediate to ~100ms.

**What you do NOT build.** No Kafka, no Cassandra, no multi-region. Buffer + batch buys 10x runway with minimal complexity.

#### Stage 3: 100k events/sec

**What breaks:**

- Postgres cannot keep up even with batching. Single instance at I/O ceiling.
- App process holding the buffer is a single point of failure. At 100k/sec, a crash loses 10k events.
- Read replica lags 30 seconds because primary is saturated.

**Fixes:**

- Kafka in front of storage. Producers get 202 immediately. Kafka buffers if storage falls behind.
- Switch to Cassandra for the hot tier. LSM tree absorbs sequential writes far better than a B-tree.
- Separate Kafka consumer process writes to Cassandra in UNLOGGED BATCHes per partition. Stateless. Run N of them. Crashes do not lose data (Kafka holds the events).
- 50 Kafka partitions, hash by `(event_type, tenant_id)`. Cassandra primary key `((tenant_id, day), event_id)`.

**What you accept.** End-to-end lag jumped from 100ms to ~5 seconds. Strong consistency is gone. One Kafka consumer crash = a few seconds of replay, handled by idempotent storage.

#### Stage 4: 1M events/sec

**What breaks:**

- Single-region Kafka hits NIC and disk limits.
- Hot tenant (one team's mobile app misbehaving) saturates one Kafka partition and one Cassandra node, hurting others.
- Cold-tier costs explode if everything stays in Cassandra.
- Compliance asks for 7-year retention.

**Fixes:**

- Regional ingest + regional Kafka clusters. Each region (us-east, eu-west, ap-south) has its own stack. Events written to producer's home region.
- Sub-shard hot partitions with a random suffix: ingest tier appends `random(0..15)` to the partition key for a hot tenant. Trades per-key ordering for load distribution.
- Flink for stream processing: enrichment, routing to multiple sinks, rollups, fraud alerts.
- Tiered storage: hot in Cassandra (90-day TTL), cold in S3 Parquet (7-year retention). Archiver consumer writes 5-min Parquet batches.
- Per-tenant rate limits at ingest. Hard cap. 429 above the limit.

---

### 9. Reliability

**Queue overflow.** Kafka at 90% disk. Back-pressure at ingest: when Kafka send latency P99 exceeds 1 second, ingest returns 429. Producers back off. Premium tenants get reserved Kafka capacity. Free-tier tenants get throttled first.

**Consumer fell behind.** Scale consumer group up temporarily. Configure throttled catch-up (2x normal rate). Catch-up is a separate workload. Do not pretend it is normal traffic; it will melt storage.

**Partial batch loss.** A Kafka consumer crashes after writing 800 of 1000 events to Cassandra, before committing the offset. On restart, it reprocesses the full batch. Cassandra primary-key UPSERT semantics make the 800 already-written events a no-op. The 200 missing events get written. No data loss.

**Ingest pod crash.** Pod crashes between accepting a request and sending to Kafka. Producer's HTTP connection drops with no response. Producer retries. With idempotency key, second submission is deduped. Mitigation: ingest pod sends to Kafka before returning 202.

**Region failure.** Producers in failed region cannot publish. SDK should fail over to next-nearest region. Data loss window: events buffered in the producer's local SDK that were never sent. Typically under 1 second. DR play: cross-region Kafka replication via MirrorMaker.

---

### 10. Observability

| Metric | Why it matters | Alert threshold |
|--------|----------------|-----------------|
| `ingest.requests.rate` | Top-line throughput | Drop >50% in 5 min |
| `ingest.latency.p99` | Producer-facing SLO | >100ms for 5 min |
| `ingest.5xx.rate` | Our problem | >0.1% sustained |
| `kafka.producer.ack.latency.p99` | Kafka health | >1s for 5 min |
| `kafka.partition.rate.max / median` | Hot partition skew | >10x ratio |
| `kafka.consumer.lag` per group | Are consumers keeping up? | >30s lag for any group |
| `kafka.broker.disk.used.pct` | Storage capacity | >80% |
| `cassandra.write.latency.p99` | Storage health | >50ms |
| `cassandra.batch.size.p99` | Batching effectively? | <10 events = too small |
| `archiver.lag` | Cold tier behind real-time | >10 min |
| `archiver.parquet_file.size.p50` | Small file problem? | <10 MB = too many files |
| `dedup.collision.rate` | Duplicate event_ids? | Any non-zero is interesting |

**Four golden signals:**

1. Ingest throughput (events/sec at the API).
2. End-to-end lag (ingest timestamp to Cassandra-readable timestamp).
3. Consumer lag per Kafka consumer group.
4. Error rates at each stage.

Page on: ingest 5xx > 0.5%, ingest P99 > 200ms for 10 min, consumer lag > 5 min for any group, Kafka broker disk > 90%.

---

### 11. Follow-up answers

**1. Back-pressure.**

When Kafka send latency P99 exceeds 1 second or any broker is unhealthy, ingest returns 429 with `Retry-After`. The producer SDK backs off with exponential backoff and caches events locally (in-memory or disk-backed) until health returns. Without producer-side backoff, the 429 storm becomes a retry storm. If Kafka is so unhealthy that even 429s are slow, the ingest tier can close connections (TCP RST), forcing producers to back off at a lower level.

**2. Hot tenant.**

Find them: the per-partition rate metric points at one Kafka partition. Cross-reference with the partition key. The next 5 minutes: hard rate limit at the ingest tier, capping the tenant at 50k events/sec. Excess gets 429. Sub-shard them by appending `random(0..15)` to the partition key, spreading across 16 partitions. They lose per-key ordering during the spike, but the cluster survives. The next 5 days: contact the tenant, diagnose the bug (often a retry loop, log misconfiguration, or a runaway script). Establish per-tenant quota policy and dedicated capacity for premium tenants.

**3. Clock skew.**

The fix is structural. Always stamp server_ts at ingest; that is the timestamp used for partitioning, bucketing, and ordering downstream. Producer_ts is kept for user display only. NTP-sync producers to the second (reduces skew enough that producer_ts is a useful sanity check). Detect outliers: any event with `producer_ts` more than 5 minutes off from `server_ts` is flagged. Stream processor tolerates late events via Flink watermarks. Never use producer_ts for anything that requires precise ordering.

**4. Duplicate events.**

Detection: Cassandra primary-key conflict on insert is the direct signal. Consumer counts `cassandra.dedup.collisions` per minute. A nightly audit: `SELECT event_id, count(*) FROM events GROUP BY event_id HAVING count(*) > 1` should always be zero. Dedup: `event_id` is the dedup key. Prefer producer-generated event_ids (ULID is good: time-ordered, unique, no coordination). If the producer cannot generate one, the server generates at ingest, but then producer retries get different event_ids and dedup is impossible. Cassandra UPSERT semantics make duplicate inserts a no-op for the same primary key.

**5. Consumer fell behind.**

Two failure modes to avoid on restart: resource exhaustion at storage (consumer wakes up, polls Kafka at full speed, sends 1.8B events to Cassandra as fast as Cassandra accepts, melts the cluster); and out-of-order derived data (rolling metrics get skewed by 1.8B events arriving in one hour). Strategies: throttled catch-up at 2x normal throughput (1.8B events at 200k/sec takes ~2.5 hours to drain). Scale the consumer group up temporarily, but watch storage. If derived consumers care about "now-time" processing, flag them to skip events older than X and replay historical events as a separate batch job.

**6. Schema evolution.**

Old producers send 5 fields, new producers send 6. Options: keep raw event as JSON (flexible but no columnar benefits). Use Avro/Protobuf with backward/forward compatibility rules (old consumers ignore unknown fields; Schema Registry enforces compatibility at registration). Versioned event types (`user.login.v1`, `user.login.v2`) let old consumers stick to v1. Never make a backward-incompatible schema change without versioning. Adding fields is fine. Removing or renaming is not without a migration plan.

**7. Recent-event reads.**

Three approaches: (1) Document the lag ("data has ~5s lag"). Most use cases accept this. Cheapest. (2) Dual-write to a real-time store. Second consumer reads Kafka and writes to Redis (per-user list, capped at 1000 recent events). Queries that need <1s freshness hit Redis; older data falls through to Cassandra. Costs 2x writes at the consumer tier. (3) Query Kafka directly via kSQL. Operationally awkward; Kafka is not a key-value store. For most audit pipelines, option 1 is correct. For fraud or live-dashboard use cases, option 2 is worth the cost.

**8. Region goes down.**

If the producer SDK has failover: it detects local Kafka unreachable, buffers events locally, fails over to the next-nearest region. When local cluster recovers, it drains the buffer. Data loss window: events buffered in the producer process at the moment of failure, typically under 1 second. If the SDK is naive: producer gets timeouts. All events generated during the outage are lost. DR play: cross-region Kafka replication via MirrorMaker so events that landed in the failed region's Kafka eventually replicate elsewhere. For zero-data-loss DR: write events to two Kafka clusters synchronously. Doubles producer cost.

**9. Cold-tier query cost.**

"Every event for tenant X for 3 years" scans naively across 3 years x 365 date partitions x all event_type subdirectories. Optimizations: partition pruning requires `tenant_id` in the S3 prefix path, not buried inside Parquet (critical). Parquet's per-row-group statistics let Athena skip row groups that cannot match the filter. For hot queries, pre-aggregate nightly into a small summary table (events per tenant per day). Events older than 1 year move to S3 Glacier Deep Archive (1/20 the cost; 12-48h retrieval for compliance queries). For interactive sub-second analytics: build an OLAP layer (ClickHouse, BigQuery, or Snowflake) and pre-load data there. S3 Parquet via Athena is for ad-hoc, not interactive dashboards.

**10. Exactly-once for a side effect.**

SMS is non-idempotent. Use a transactional outbox at the fraud-alert consumer specifically: consumer reads Kafka events and writes "send SMS" records to a local DB table `pending_sms(event_id, status)` with `status = 'pending'`. A separate sender process picks pending rows, calls the SMS API with `idempotency_key = event_id` (Twilio supports this), then marks the row `status = 'sent'`. If the sender crashes between API call and status update, on restart it retries. The SMS provider's idempotency key prevents a second SMS from sending. This pays one extra DB write per event. Apply the expensive guarantee only at the consumer that needs it, not on the whole pipeline.

---

### 12. Trade-offs worth saying out loud

**Sync vs async.** Sync writes give immediate consistency and simpler reasoning. They cap throughput at the storage layer's speed. Async (via queue) decouples producer rate from storage rate. Higher throughput at the cost of end-to-end lag and weaker consistency. The right call depends on the lag SLA. Under 100ms means sync. Over 1 second acceptable means async.

**Batch vs stream.** Batching amortizes per-write overhead. Larger batches mean higher throughput but higher latency per event. The sweet spot is "batch enough to amortize, small enough to land within the lag SLA." Typical: 100ms or 1000 events, whichever comes first.

**Exactly-once cost.** Kafka EOS drops throughput 20-40% and adds operational complexity. The right move is usually "at-least-once with idempotent storage." Reserve true exactly-once for non-idempotent side effects.

**Why Cassandra over Postgres at the hot tier.** Postgres B-tree is bad at write-heavy random-key inserts. Cassandra LSM is built for it. Under 50k writes/sec, Postgres is fine and operationally simpler. Above that, Cassandra wins on throughput per node and horizontal scaling (add a node, rebalance automatically).

**Why S3 Parquet for cold tier.** Cheap, durable, columnar, queryable with serverless tools. Keeping 7 years of events in Cassandra costs 100x more for events accessed once a year. The access pattern decides the storage tier.

**What I would revisit at 10M events/sec.** Specialized vendor (Datadog, Splunk, Honeycomb) or specialized infrastructure. Or split into purpose-specific pipelines: one for security events (strict durability, long retention), one for product analytics (loose durability, short retention, aggregated only), one for traces (sampled, very short retention). The "one pipeline for everything" pattern stops scaling cleanly past ~1M events/sec.

---

### 13. Common mistakes

**Reaching for Kafka and Cassandra immediately.** The interviewer started you at 100 events/sec. If you propose a Kafka cluster in minute one, you have not earned the architecture. Walk the stages. Justify each addition with a specific failure of the previous one.

**Confusing buffering with batching.** Buffering means holding events in memory before writing. Batching means writing many events in one storage call. You usually do both. Articulate the difference.

**Not addressing partitioning.** "Use Cassandra" without specifying the partition key is a hand-wave. The partition key determines whether your cluster has hot nodes. It is the most important decision in the storage layer.

**Claiming exactly-once without paying the cost.** If you say "exactly-once," you must explain how (Kafka EOS, transactional outbox) and what it costs. If you say "at-least-once with idempotent storage," you score the same correctness with a tenth the operational pain.

**Forgetting back-pressure.** A write-heavy system without back-pressure cascades on the first downstream slowness. The 429 and retry-with-backoff loop is mandatory.

**Not designing for the hot partition.** "It will distribute evenly" is a hope, not a design. Real systems have one tenant 1000x bigger than the median. Have an answer: sub-sharding, rate limiting, dedicated capacity.

**No tiered storage.** Keeping 7 years of events in Cassandra is wasted money. Cold tier (S3 Parquet) is the standard play.

**Treating lag as a bug.** End-to-end lag is the price you pay for async ingestion. Some consumers want lag (cheap batch processing). Some do not (fraud alerts). Acknowledge the spectrum.

**Ignoring observability.** Queue depth, consumer lag, batch size, and partition skew are the metrics that diagnose every common failure. They are part of the design, not an afterthought.

**Building the 1M/sec architecture at 100/sec scale.** The system collapses under its own operational weight before it ever sees the traffic it was built for. The discipline: design for one stage past current, not three.

If you hit 7 of these 10 cleanly, you are interviewing well. The two that separate staff-level candidates: addressing the hot partition problem unprompted, and explaining the exact moment you would *not* yet introduce Kafka (the stage 1 to 2 transition). Both show that you understand cost as well as throughput.
