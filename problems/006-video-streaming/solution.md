## Solution: Design YouTube / Netflix (Video Streaming)

### TL;DR

A video streaming platform is two systems that share a database. The **upload + transcoding pipeline** is a write-heavy, compute-bound batch system. The **playback path** is a read-heavy, latency-bound, CDN-dominated system. They share the metadata catalog and the transcoded object store; everything else is separate.

The interesting engineering is in three places: (1) keeping the upload byte path off your origin servers using TUS or S3 presigned multipart, (2) sizing the transcoding farm against bursty ingest while keeping codec costs sane (AV1 is 30x slower than H.264), and (3) building a three-tier CDN (edge, regional shield, origin) that gets above 95% hit rate so you only pay S3 egress on the long tail. Storage tiering across hot SSD, warm HDD, and Glacier reduces the storage bill by 5x.

Numbers worth remembering: 500 hours uploaded per minute, 1B hours watched per day, 125M concurrent viewers at peak, 250 Tbps egress, 70 PB/year of new storage, thousands of CPU cores for steady-state transcoding.

### 1. Clarifying questions and why each matters

Already covered in `question.md`. The eight questions you must ask:

- **YouTube vs Netflix.** UGC long-tail vs curated catalog. Storage tiering is opposite (UGC has cold-heavy distribution; catalog has nearly all hot).
- **Live vs VOD.** Live has its own architecture; VOD is the default.
- **Geographic scope.** Determines whether the CDN story is global or single-region.
- **Codec ladder.** H.264 only is one design; H.264 + H.265 + AV1 is a 5x compute story.
- **Upload constraints.** Resumable upload becomes mandatory above ~1GB.
- **DRM.** Adds license server, per-device key flow, encrypted segments.
- **Recommendations / search.** Mention the integration point; do not design them.
- **Analytics freshness.** Real-time vs batch determines whether you have one or two view-count pipelines.

If you walked in and started drawing without asking "live or VOD" you are about to design the wrong system.

### 2. Capacity estimates (full working)

- **Ingest:** 500 hours/min = 500 video-seconds/sec. At 10 Mbps avg source bitrate, **5 Gbps sustained, 15 Gbps peak**. This is what your upload gateway (or S3, if you use presigned URLs) handles.
- **New videos/day:** 500 hours × 60 × 24 / 4-min average = **~450K new videos/day**, ~5/sec sustained, ~15/sec peak.
- **Concurrent viewers:** 1B hours/day × 3600 / 86400 = **42M concurrent average, ~125M peak**.
- **Egress:** 125M × 2 Mbps avg playback bitrate = **250 Tbps peak**. The CDN sees this; your origin sees a fraction (4-5%).
- **Source storage:** 500 sec/sec × 86400 × 365 × (10 Mbps / 8) = **~20 PB/year of source files**.
- **Transcoded storage:** ~14 variants per video, aggregating to ~2.5x source = **~50 PB/year of variants**.
- **Combined yearly growth:** **~70 PB/year**. Compounds.
- **Transcoding compute:** 500 sec/sec input × 14 variants / 2x real-time = **~3500 CPU cores** continuously for H.264. Add several thousand more for AV1 on the top 1% of videos. GPU acceleration (NVENC, NVDEC) helps for H.264 but AV1 is still mostly CPU.

The two cost lines that dominate everything: **CDN egress** and **transcoding compute**. Storage is third. Everything else is rounding error.

### 3. API design

**Upload (creator-side)**

```
POST /api/v1/uploads
Content-Type: application/json
Authorization: Bearer <token>

{
  "filename": "vacation.mp4",
  "size_bytes": 5368709120,
  "sha256": "abc123...",            // optional, client-computed
  "title": "Beach trip 2026",
  "description": "...",
  "visibility": "public"             // public | unlisted | private
}
```

Response 201:
```
{
  "video_id": "v_8h3jK2p",
  "upload_url": "https://upload.example.com/u/8h3jK2p",
  "chunk_size": 16777216,            // 16 MB
  "expires_at": "2026-05-22T18:00:00Z"
}
```

**Chunk upload (TUS-style)**

```
PATCH /u/<upload_id>
Upload-Offset: 33554432
Content-Length: 16777216
Content-Type: application/offset+octet-stream
<binary chunk>
```

Response 204 with `Upload-Offset: 50331648` (new total).

```
HEAD /u/<upload_id>     -> returns Upload-Offset for resume
```

**Finalize**

```
POST /api/v1/uploads/<upload_id>/finalize

Response: 200
{
  "video_id": "v_8h3jK2p",
  "status": "transcoding"
}
```

**Get video metadata (creator + viewer)**

```
GET /api/v1/videos/<video_id>

Response:
{
  "video_id": "v_8h3jK2p",
  "title": "Beach trip 2026",
  "description": "...",
  "owner": { "user_id": "u_42", "name": "Amir" },
  "status": "ready",                  // uploading | transcoding | ready | blocked
  "duration_seconds": 248,
  "thumbnails": ["https://i.example.com/v_8h3jK2p/t0.jpg", ...],
  "manifests": {
    "hls": "https://cdn.example.com/v_8h3jK2p/master.m3u8",
    "dash": "https://cdn.example.com/v_8h3jK2p/manifest.mpd"
  },
  "available_variants": ["240p", "360p", "480p", "720p", "1080p"],
  "view_count": 14271
}
```

**Manifest** (served by CDN, not your API)

```
GET https://cdn.example.com/v_8h3jK2p/master.m3u8     -> ~1 KB text
GET https://cdn.example.com/v_8h3jK2p/720p/seg_42.ts  -> ~2 MB binary
```

These are static files. Your API is not on the playback hot path.

**Playback telemetry (player → backend)**

```
POST /api/v1/telemetry/playback
{
  "video_id": "v_8h3jK2p",
  "session_id": "...",
  "events": [
    { "ts": 1716364800000, "type": "start", "variant": "720p" },
    { "ts": 1716364812000, "type": "rebuffer", "duration_ms": 850 },
    { "ts": 1716364900000, "type": "variant_switch", "from": "720p", "to": "1080p" },
    { "ts": 1716365100000, "type": "complete", "watched_seconds": 300 }
  ]
}
```

Batched; not on critical path; fire-and-forget.

**Key API design notes.**

- **Manifests and segments are served by CDN, not by your service.** They are static signed URLs. Your service produces the URL pattern when you assemble the manifest; the CDN serves the bytes.
- **Idempotency.** `POST /uploads` accepts a client-provided idempotency key. Re-sent finalize is safe (no-op if already finalized).
- **Signed URLs.** Manifests and segments have signed URLs with a short expiry (typically 4-8 hours) so they cannot be embedded on third-party sites for free. Important for cost control and for DRM gating.

### 4. Data model

```sql
-- The catalog row. One per video.
CREATE TABLE videos (
    video_id        VARCHAR(16) PRIMARY KEY,    -- short ID, base62
    owner_id        BIGINT NOT NULL,
    title           VARCHAR(200),
    description     TEXT,
    visibility      SMALLINT NOT NULL DEFAULT 1, -- 1=public, 2=unlisted, 3=private
    status          SMALLINT NOT NULL DEFAULT 0, -- 0=uploading, 1=transcoding, 2=ready, 3=blocked, 4=deleted
    duration_ms     BIGINT,                      -- known after transcode
    source_path     TEXT,                        -- s3://video-source/...
    created_at      TIMESTAMPTZ NOT NULL,
    published_at    TIMESTAMPTZ,
    sha256          BYTEA
);

CREATE INDEX idx_owner ON videos (owner_id, created_at DESC);

-- One row per (video, variant). Tracks transcoding progress.
CREATE TABLE variants (
    video_id        VARCHAR(16),
    variant         VARCHAR(20),                 -- "240p_h264", "1080p_h265", "1080p_av1"
    status          SMALLINT,                    -- 0=pending, 1=encoding, 2=ready, 3=failed
    bitrate_kbps    INT,
    codec           VARCHAR(16),                 -- "h264", "h265", "av1"
    container       VARCHAR(8),                  -- "ts", "m4s"
    segment_count   INT,
    output_prefix   TEXT,                        -- s3://video-transcoded/{id}/{variant}/
    worker_id       VARCHAR(64),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    PRIMARY KEY (video_id, variant)
);

-- One row per video summarizing what's playable.
CREATE TABLE manifests (
    video_id        VARCHAR(16) PRIMARY KEY,
    hls_master_url  TEXT,
    dash_mpd_url    TEXT,
    available_variants TEXT[],                   -- json or array
    updated_at      TIMESTAMPTZ
);

-- Aggregated view counts. Eventually consistent.
CREATE TABLE view_counts (
    video_id        VARCHAR(16) PRIMARY KEY,
    total_views     BIGINT,
    total_watch_seconds BIGINT,
    last_updated    TIMESTAMPTZ
);
```

**Why these choices.**

- **Wide-column store (Cassandra or DynamoDB) is the right pick at scale.** The access pattern is: read by `video_id` (point lookup), write by `video_id`. No relational joins. Cassandra's partition key on `video_id` gives you constant-time reads, and you can have variants as clustering columns inside the partition for free.
- **Postgres works fine until you hit ~10 billion videos.** YouTube has ~10B+ videos uploaded over its lifetime; Postgres at that size requires aggressive sharding and the operational burden is high. Cassandra was literally invented for this access pattern.
- **`variants` as separate rows, not a JSON column on `videos`.** Each variant is updated independently as its transcode worker finishes. Atomic per-variant updates are cleaner than CAS on a JSON blob.
- **`view_counts` is separate.** It is updated by a streaming aggregator on a different cadence than the video metadata. Mixing them would force every view-count update to go through the same write path as catalog edits.

### 5. Core algorithm: transcoding pipeline + manifest generation

#### 5a. Transcoding job graph

For one source video, the system spawns one job per `(variant, codec)` pair. Concretely, for a 1080p source:

| Job | Codec | Bitrate | Resolution |
|-----|-------|---------|------------|
| 1 | H.264 | 400 kbps | 426x240 |
| 2 | H.264 | 700 kbps | 640x360 |
| 3 | H.264 | 1200 kbps | 854x480 |
| 4 | H.264 | 2500 kbps | 1280x720 |
| 5 | H.264 | 5000 kbps | 1920x1080 |
| 6 | H.265 | 1500 kbps | 1280x720 |
| 7 | H.265 | 3000 kbps | 1920x1080 |
| 8 | AV1 (if video is "popular candidate") | 1000 kbps | 1280x720 |
| 9 | AV1 | 2000 kbps | 1920x1080 |

That is 7-9 jobs per video. Each runs independently. Workers pick from a Kafka topic partitioned by `video_id` (so all variants of one video tend to run on adjacent workers, simplifying log correlation, but this is not strictly required).

#### 5b. One worker's loop

```
loop:
    job = consume_from_kafka("transcode.requested")     // (video_id, variant_spec)
    if job is None: sleep(0.5); continue

    mark_started(job)                                    // UPDATE variants SET status=1
    source = download_or_stream("s3://video-source/{video_id}/source.mp4")

    ffmpeg_args = build_args(
        input=source,
        codec=job.codec,
        bitrate=job.bitrate,
        resolution=job.resolution,
        segment_duration=6,                              // seconds
        output_format=job.container,                     // "m4s" or "ts"
    )
    run_ffmpeg(ffmpeg_args)                              // produces seg_0.m4s, seg_1.m4s, ...

    upload_segments_to_s3(job.output_prefix)
    write_variant_playlist("playlist.m3u8")              // per-variant HLS playlist
    upload(playlist.m3u8, job.output_prefix)

    mark_completed(job, segment_count)                   // UPDATE variants SET status=2
    publish_to_kafka("transcode.completed", job)
```

**Important details.**

- **Segments must be aligned across variants.** Every variant cuts segments at the same wall-clock offsets (e.g., 0s, 6s, 12s, 18s). This is what lets the player switch variants mid-stream. Use ffmpeg's `-force_key_frames` to put a keyframe exactly at each segment boundary.
- **Segment duration trade-off.** 6 seconds is a typical compromise. Shorter (2s) = lower start latency for live, but more files and more CDN requests. Longer (10s) = better compression but slower seek and slower variant switching. VOD usually picks 6s; live LL-HLS picks 2s or less.
- **Idempotency.** Re-running the same job overwrites the same S3 keys. This is important because Kafka may redeliver after a worker crash.
- **Failure handling.** A worker that crashes mid-job leaves a stale `status=encoding` row. A janitor process scans for rows with `started_at` older than 2x expected job time and re-publishes them to Kafka. The "at-least-once" delivery combined with idempotent output makes this safe.

#### 5c. Manifest generation

When `transcode.completed` is published, a separate Manifest Assembler service consumes it:

```
on event ("transcode.completed", video_id, variant):
    all_variants = SELECT * FROM variants WHERE video_id = ?
    if any variant has status != 2 (ready) and is required:
        return                          // wait for more events
    
    master = build_master_m3u8(all_variants)
    mpd    = build_dash_mpd(all_variants)
    
    write_to_s3(f"s3://video-transcoded/{video_id}/master.m3u8", master)
    write_to_s3(f"s3://video-transcoded/{video_id}/manifest.mpd", mpd)
    
    UPDATE videos SET status = 'ready', duration_ms = ... WHERE video_id = ?
    publish_to_kafka("video.published", {video_id})
```

The master m3u8 is a small text file that references each per-variant playlist. The per-variant playlists were already written by the workers; the assembler does not touch the segment files.

**Progressive availability.** A platform like YouTube shows the video as playable in 360p before 1080p is ready. The Manifest Assembler can publish a partial master playlist after the first variant completes, then re-publish as more variants come in. Each re-publish invalidates the CDN cache for `master.m3u8` (TTL is short, e.g., 60 seconds, exactly to support this).

### 6. High-level architecture (detailed)

```
   ┌──────────┐
   │ Creator  │
   └────┬─────┘
        │ TUS / S3 multipart (chunked, resumable)
        ▼
   ┌──────────────────────┐
   │  Upload Gateway      │   Stateless. Auth, quota, chunk routing,
   │  (API pods)          │   sha256 verify, presigned-URL issuance.
   └─────────┬────────────┘
             │
             ▼
   ┌──────────────────────┐
   │ s3://video-source/   │   Durable source store. Lifecycle policy
   │ (S3 Standard)        │   moves to S3 IA at 30 days, Glacier at 90.
   └─────────┬────────────┘
             │ on finalize → Kafka
             ▼
   ┌──────────────────────┐
   │ Kafka                │   Topics: transcode.requested,
   │ (durable queue)      │           transcode.completed,
   │                      │           video.published.
   └─────────┬────────────┘
             │
             ▼
   ┌──────────────────────────────────────────┐
   │  Transcoding Farm (Kubernetes pool)      │
   │  ─────────────────────────────────────   │
   │   CPU nodes (H.264, H.265): N=~3000      │
   │   GPU nodes (NVENC for H.264):  M=~200   │
   │   CPU nodes (AV1, slow):       K=~2000   │
   │  Autoscaler watches Kafka lag.           │
   └─────────┬────────────────────────────────┘
             │
             ▼
   ┌──────────────────────┐
   │ s3://video-          │   Transcoded variants. Per-video prefix.
   │ transcoded/          │   Hot videos cross-region replicated.
   │ (S3 Standard, IA,    │   Tiering by view recency.
   │  Glacier)            │
   └─────────┬────────────┘
             │
             ▼
   ┌──────────────────────┐
   │ Manifest Assembler   │   Watches transcode.completed.
   │ (consumer service)   │   Builds master.m3u8 / manifest.mpd.
   │                      │   Updates Metadata DB on ready.
   └─────────┬────────────┘
             │
             ▼
   ┌──────────────────────┐
   │ Metadata DB          │   Cassandra (videos, variants, manifests).
   │ (sharded by          │   Read-replicated globally.
   │  video_id)           │
   └──────────────────────┘
                                ▲
                                │ reads from Watch Page API
                                │
   ─────────── Playback Path ────────────────────────────────────────
                                │
   ┌──────────┐                 │
   │  Viewer  │                 │
   │  player  │                 │
   └────┬─────┘                 │
        │ GET /watch/{id}        │
        ▼                       │
   ┌──────────────────────┐     │
   │ Watch Page API       │─────┘
   │ (returns metadata +  │
   │  signed manifest URL)│
   └──────────────────────┘

        │ GET master.m3u8         │ GET seg_N.ts/m4s
        ▼                         ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  CDN: 3-tier cache                                           │
   │  ─────────────────────────────────────────────────────────   │
   │   Edge PoPs (in-city) ──► Regional shields ──► Origin (S3)   │
   │   ~200+ PoPs               ~10-20 shields       1 per region │
   │   TTL 24h-7d for segs      TTL 7d               authoritative │
   │   Hit rate target: 95%+    Hit rate: 80%+                    │
   └──────────────────────────────────────────────────────────────┘

        │ playback events (batched)
        ▼
   ┌──────────────────────┐
   │ Telemetry → Kafka    │   Per-session events: start, rebuffer,
   │                      │   variant switch, complete.
   └─────────┬────────────┘
             ▼
   ┌──────────────────────┐
   │ Stream processor     │   Flink. Aggregates view counts, watch
   │ (Flink / KSQL)       │   time, QoE metrics. Windows: 1min, 1h.
   └─────────┬────────────┘
             ▼
   ┌──────────────────────┐
   │ View counter store   │   Redis (real-time counters) +
   │ + Analytics DB       │   ClickHouse (time-series).
   │ (Redis + ClickHouse) │
   └──────────────────────┘
```

Component-by-component responsibilities:

- **Upload Gateway.** Stateless. Public HTTPS endpoint. Authenticates creator. Issues per-upload IDs. Optionally proxies chunks to S3, or issues presigned URLs so the client uploads directly. The choice depends on whether you trust the creator and whether you want to MITM the bytes for malware scanning.
- **Source bucket.** S3 Standard initially. Lifecycle rules demote to IA at 30 days, Glacier Instant at 90 days. Sources are never deleted; you might need them to re-encode in a new codec.
- **Kafka.** Three topics: `transcode.requested` (one message per `(video_id, variant)`), `transcode.completed` (worker fans out completion events), `video.published` (downstream consumers: search indexer, notification, recs). Each topic is partitioned by `video_id`.
- **Transcoding Farm.** A Kubernetes pool. The H.264 jobs run on CPU or NVENC GPU nodes; H.265 mostly CPU; AV1 always CPU (and slow). Autoscaler watches Kafka consumer lag and scales worker replicas. The pool has separate node groups per codec because the optimal hardware differs.
- **Transcoded bucket.** Where every segment lands. Same lifecycle tiering as source but driven by view recency, not age.
- **Manifest Assembler.** Stateless consumer. One per Kafka consumer group. Builds and overwrites master playlists.
- **Metadata DB (Cassandra).** Wide-column. `video_id` is the partition key. Read-heavy (every watch page hits it once). Read replicas in every region.
- **Watch Page API.** Stateless. Reads from Metadata DB, builds signed CDN URLs, returns JSON. The only piece of your code on the playback critical path. Should be < 50ms P99.
- **CDN.** The most expensive line item. Three tiers explained below.
- **Telemetry pipeline.** Player POSTs batches of events. Goes to Kafka. Flink aggregates. View counts land in Redis (for the watch page) and ClickHouse (for creator analytics).

### 7. Upload path (detailed walkthrough)

A creator uploads a 5 GB MP4. Full sequence:

1. Creator's client calls `POST /uploads`. The Upload Gateway:
   - Authenticates with bearer token.
   - Checks the creator's quota.
   - Creates a row in `videos` with `status = uploading`, mints `video_id = v_8h3jK2p`.
   - Returns the upload URL.
2. Client splits the file into 16 MB chunks (≈ 320 chunks).
3. Client sends `PATCH /u/v_8h3jK2p` repeatedly, each with `Upload-Offset` and one chunk in the body.
   - If a chunk fails: client retries the same chunk. The gateway is idempotent on chunk write (same offset = same bytes).
   - If the connection drops for minutes: client calls `HEAD /u/v_8h3jK2p` to get the current offset, then resumes from there.
4. After the last chunk, the client calls `POST /uploads/v_8h3jK2p/finalize`.
5. Upload Gateway verifies the sha256 if provided. Marks `videos.status = transcoding`. Publishes a `transcode.requested` message per `(video_id, variant)` pair to Kafka. There are typically 7-9 of these per video.
6. Transcoding workers consume the messages. Each worker:
   - Updates `variants.status = encoding` for its variant.
   - Downloads (or streams) the source.
   - Runs ffmpeg.
   - Uploads segments to `s3://video-transcoded/v_8h3jK2p/{variant}/`.
   - Writes per-variant playlist.
   - Updates `variants.status = ready`.
   - Publishes `transcode.completed`.
7. Manifest Assembler consumes `transcode.completed`. When all required variants are ready (or after the first variant if progressive availability is enabled):
   - Builds master playlist.
   - Updates `videos.status = ready`.
   - Publishes `video.published`.
8. Downstream consumers of `video.published`:
   - Search indexer updates the search index.
   - Notification service sends "new video from a creator you follow" pushes.
   - Recommendation service updates candidate sets.
9. The creator's client polls `GET /videos/v_8h3jK2p` and sees `status = ready`. They can publish.

**Where the path can break.**

- **Chunk upload fails partway.** Resume mechanism handles this. No data lost.
- **Finalize succeeds but Kafka publish fails.** The finalize endpoint is idempotent; client can retry. To handle partial publish (some `(variant)` messages sent, some not), use a transactional outbox: write Kafka messages to a `pending_kafka_messages` table in the same transaction as the status update, then a separate publisher drains the table.
- **Transcoder crashes mid-job.** Janitor process scans for stuck rows and republishes. Workers are idempotent on output S3 keys.
- **Source corrupted.** sha256 verification at finalize catches this. Bad uploads are rejected.

### 8. Playback path (detailed walkthrough)

A viewer opens youtube-like.com/watch?v=v_8h3jK2p. Full sequence:

1. Browser loads HTML page.
2. JS calls `GET /api/v1/videos/v_8h3jK2p`. Watch Page API:
   - Looks up `videos` row in Cassandra. Cache hit in Redis or local in-process LRU for popular videos.
   - Looks up `manifests` row to get the master playlist URL.
   - Signs the master URL with a short-lived token (e.g., HMAC, expires in 4 hours).
   - Returns metadata + signed manifest URL.
3. Player (HLS.js, Shaka, ExoPlayer) fetches `master.m3u8` from the signed CDN URL.
   - **CDN edge cache.** Almost always hit for popular videos. ~50ms.
   - **On miss, edge fetches from regional shield.** Shield aggregates requests from many edges; the same master playlist might be requested by 50 edges. Shield does one fetch from S3 and serves the next 49 requests from its cache.
4. Player picks a starting variant (typically the second-from-bottom, e.g., 360p). Fetches that variant's `playlist.m3u8`.
5. Player fetches segment 0, decodes, starts playback. Continues fetching ahead.
6. Player measures bandwidth from segment download times. Switches variants up/down between segments.
7. Telemetry: player batches events (start, rebuffer, switch, pause, complete) and POSTs to `/telemetry/playback` every few seconds. This is fire-and-forget; failure does not affect playback.

**Latency budget for first frame.**

| Step | Time | Notes |
|------|------|-------|
| TCP + TLS to watch page | 100-200 ms | HTTP/3 + 0-RTT helps |
| `GET /videos/<id>` API call | 50 ms | Mostly cached |
| `GET master.m3u8` (CDN hit) | 50 ms | Edge in same city |
| `GET playlist.m3u8` (CDN hit) | 50 ms | Same |
| `GET seg_0.ts` (CDN hit) | 200-500 ms | Depends on segment size |
| Decode first frame | 50 ms | |
| **Total** | **~500-900 ms to first frame** | CDN hits assumed |

A 2-second start latency target is achievable with cache hits. Cache miss on segment 0 (a brand new video, no one watched yet) adds 100-300 ms for the shield-to-S3 fetch.

### 9. Scaling

#### 9a. The CDN: three tiers

This is where the egress bill lives. Three tiers:

| Tier | Where | What it caches | TTL | Target hit rate |
|------|-------|----------------|-----|-----------------|
| **Edge** | PoPs (one per major city, 200+ globally) | Segments + manifests of videos popular in this city | 24h-7d for segments; 60s for master.m3u8 | 95%+ |
| **Regional shield** | One per cloud region (10-20 globally) | Anything an edge fetched from this region | 7d | 80%+ on edge miss |
| **Origin** | S3 + signing service | Everything | n/a (authoritative) | n/a |

**Why the shield matters.** Without it, every edge miss goes to S3. 200 edges in a region, each missing the same segment, means 200 S3 GETs for one piece of content. The shield aggregates: edges miss the segment, all 200 ask the shield, the shield does 1 S3 GET and serves the 199 others from its cache. Without the shield, your S3 request rate scales with the number of edges; with it, S3 sees mostly cold-tail traffic.

**Tuning for hit rate.**

- **Segment TTL.** 7 days for segments. They never change once written (segments are content-addressed by video_id + segment number).
- **Manifest TTL.** 60 seconds. Master playlists do change (when a new variant becomes ready). Short TTL ensures viewers see the new variants quickly.
- **Purge on takedown.** Cache invalidation by URL pattern (`/v_8h3jK2p/*`). All major CDNs (Cloudflare, Fastly, Akamai, CloudFront) support pattern purges in seconds.

**Cost.** CDN egress costs $0.005-0.02 per GB depending on volume tier. At 600 PB/day this is $3M-12M/day if you paid for every byte. With 95% edge hit rate, only 5% of bytes hit the shield, and only 20% of that hits S3. So actual S3 egress is ~1% of total = ~6 PB/day from S3 alone. The CDN itself charges per byte served at the edge but at much lower rates than S3 egress.

#### 9b. Transcoding farm scaling

- **Autoscaler signal:** Kafka consumer lag on `transcode.requested`. If lag exceeds X messages, scale up. If lag is consistently 0, scale down.
- **Separate pools per codec.** H.264 jobs go to nodes with NVENC GPUs (massively faster). H.265 jobs go to either NVENC or CPU. AV1 jobs go to CPU pools with high core counts (AV1 GPU encoders exist but quality is currently worse than CPU at the same bitrate).
- **Priority lanes.** Premium creators or live-event archives get a high-priority Kafka topic. The high-priority worker pool drains it first. Default uploads use the standard pool.
- **Spot / preemptible nodes.** Transcoding is restartable. Run 60-80% of the pool on preemptible nodes. Workers checkpoint progress: if preempted halfway through encoding segment 47, the resumed worker restarts at segment 47, not from scratch.

#### 9c. Storage tiering (cost reduction)

Already covered in `question.md` Step 6. The summary:

- **Hot (S3 Standard):** anything watched in the last 7 days. ~5% of catalog.
- **Warm (S3 IA):** watched in the last 90 days. ~15% of catalog.
- **Cold (Glacier Instant):** the long tail. ~80% of catalog.
- **Archive (Glacier Deep Archive):** source files of videos with no view in a year.

Blended cost goes from $0.023/GB-mo (all-hot) to ~$0.005/GB-mo (tiered). 5x savings.

The job that moves content between tiers reads `view_counts.last_viewed_at` from the analytics DB. Runs daily. Emits S3 lifecycle transitions in bulk.

#### 9d. Metadata DB scaling

- Cassandra cluster sharded by `video_id`. ~100 nodes for ~10 billion videos.
- Replication factor 3 within a region; one full replica per region.
- All reads are point reads or partition reads. No scatter-gather (except for "videos by owner_id" which uses a secondary table with `owner_id` as partition key).
- Hot videos: their row is cached in Redis with a TTL of a few hours, plus in-process LRU in the Watch Page API for the very top videos.

#### 9e. Geographic distribution

- Source bucket: one region (e.g., us-east) for write; cross-region replicated for read.
- Transcoded bucket: region of the worker that produced it; hot videos replicated to all regions.
- Metadata DB: multi-region replicated. Writes to nearest region; eventually consistent globally with ~1-2s replication lag.
- CDN: PoPs everywhere; shields per region; origin is S3 in the region nearest each shield.

### 10. Reliability

- **CDN PoP outage.** Traffic shifts to neighboring PoPs via CDN's own routing. Latency increases for viewers in that city; nothing else breaks.
- **Origin shield outage.** Edges fall back to direct-to-S3. S3 request volume spikes by 10-50x. Mitigation: S3 is provisioned for this burst (request-per-second limits are generous), and the edges have stale-while-revalidate set so most viewers see cached content from the edge even while the shield refresh is broken. Worst case: cold-tail videos cannot play until shield recovers.
- **S3 outage in one region.** Switch to cross-region replicas via signed URLs that point at the surviving region. Replication lag becomes visible: a video uploaded in the last few minutes might 404.
- **Metadata DB partial failure.** Cassandra survives node failures by definition (RF=3). A whole-region failure means writes from that region's creators stall; reads continue from other regions.
- **Transcoding farm queue overflow.** New videos sit in Kafka waiting their turn. Creators see "transcoding" status for longer. Eventually, the autoscaler catches up.
- **Kafka broker loss.** Replicate Kafka with RF=3, leader election handles broker failure. Producer retries with idempotence enabled prevent duplicates.
- **Janitor for stuck jobs.** A periodic scan finds `variants` rows in `status=encoding` for longer than 2x expected duration. Republishes the job to Kafka. Combined with idempotent ffmpeg output, this is safe.

### 11. Observability

What must be instrumented from day one:

| Metric | Why |
|--------|-----|
| `upload.bytes_per_sec`, `upload.chunks_failed_rate` | Detect upload regressions |
| `upload.finalize.p99_latency` | Slow finalize = sha256 mismatch storm or backend overload |
| `kafka.transcode.requested.lag` | Transcoding pipeline depth; autoscaler input |
| `transcoder.job_duration` by variant and codec | Detect ffmpeg regressions |
| `transcoder.failure_rate` | Bad sources, OOMs, codec bugs |
| `manifest.assembly_lag` (seconds from last variant complete to master published) | Pipeline freshness |
| `cdn.edge_hit_rate` per PoP | Headline cost driver. Below 90% = investigate. |
| `cdn.shield_hit_rate` | Below 70% means shield is undersized or TTLs wrong |
| `cdn.s3_origin_request_rate` | Should be tiny fraction of total |
| `playback.first_frame_p99_ms` | The user SLO |
| `playback.rebuffer_ratio` (rebuffer seconds / playback seconds) | QoE metric. Below 0.5% target. |
| `playback.variant_distribution` | Are viewers stuck on low quality? Means bandwidth problems or ABR bugs. |
| `db.cassandra.read_p99` | Watch page latency depends on this |
| `view_counter.lag_seconds` | If creators see "0 views" for a viral video, this is broken |

**Alerts.**

- Page on: cdn.edge_hit_rate < 85% for 10 min, playback.first_frame_p99 > 3000 ms for 5 min, transcoder.failure_rate > 5%, s3_origin_request_rate > 10x baseline.
- Ticket on: manifest.assembly_lag > 5 min, view_counter.lag > 30 min.

### 12. Follow-up answers

**1. Delete a video that is in every edge cache.**

The video has 100M views, so its segments are in hundreds of edge PoPs. The flow:

- Update `videos.status = deleted` in Cassandra immediately. The Watch Page API now refuses to issue signed URLs for it; new viewers see "video unavailable".
- Issue a CDN cache purge by URL pattern for `/v_8h3jK2p/*`. Major CDNs propagate purges in 5-30 seconds globally.
- For in-flight playback sessions that already have segments cached in the player: they continue playing until the cached buffer drains, then the next segment fetch fails (CDN purged). The viewer sees an error mid-video. This is the price of long edge TTLs.
- For paranoid takedowns (CSAM, court order), revoke the signed-URL signing key for that video. Even if a segment is somehow served, the signature is invalid. Belt and suspenders.

The 60-second target is achievable with CDN pattern purge + signed-URL revocation. The 60-second window is documented; legal teams understand it.

**2. Backfill AV1 for the top 10,000 videos.**

- **Why backfill.** AV1 is ~30% smaller than H.264 at the same quality. For a video with 100M views at 5 Mbps, switching to AV1 saves 30% of bandwidth = real money.
- **Selection.** Sort videos by `total_watch_seconds` (more meaningful than view count). Take the top 10K. Estimate per-video savings: `(h264_bitrate - av1_bitrate) × 8 × watch_seconds_per_month`. Filter to videos where savings exceed 10x the one-time encoding cost.
- **Compute estimate.** Average video length 10 min. AV1 encoding at 0.5x real-time = 20 min compute per video × 10K = 200K compute-minutes = ~3300 hours per machine = ~140 machine-days for a single core. With 1000 cores running in parallel: ~3 hours wall-clock for the whole batch. Fits in a maintenance window.
- **Execution.** Add `av1` variant to the existing transcoding pipeline. Publish 10K `transcode.requested` messages with priority="backfill" (lower than user-uploaded). Manifest Assembler picks up `transcode.completed` events and adds AV1 variants to the existing master playlist. Players that support AV1 (newer Chrome, Firefox, mobile) will pick it; older players stick with H.264.
- **Source files.** Some sources may be in Glacier. The backfill job restores them in batches (bulk restore is much cheaper than per-object). Schedule the restore the day before the encode job.

**3. Live streaming.**

Live and VOD share almost no infrastructure. Changes:

- **Ingest.** Creator pushes via RTMP (legacy) or WebRTC (low-latency) to an ingest server (a regional pool of nginx-rtmp or pion-webrtc instances).
- **Transcoding.** Real-time, not batch. The ingest server transcodes on-the-fly into the ladder, producing segments as the stream comes in. Latency budget: 1-2 seconds for transcoding.
- **Distribution.** Low-Latency HLS (LL-HLS) or Low-Latency DASH (LL-DASH). Segment duration is short (1-2 sec) and HTTP chunked transfer is used so the segment is delivered while still being produced.
- **CDN.** Same CDN, but segments have very short TTLs and use shorter cache key lifetimes. The CDN's shield aggregates the "many viewers asking for segment N before it exists" case via HTTP request coalescing.
- **Glass-to-glass latency.** With LL-HLS: 3-5 sec. With WebRTC delivery (no segments, direct connection): under 1 sec but cannot scale to 5M viewers without an SFU mesh. For 5M viewers, LL-HLS at 3-5 sec is the practical choice.
- **DVR / VOD conversion.** As segments are generated, archive them. When the live stream ends, the same segments become the VOD asset. No re-transcoding needed.

**4. Thumbnails.**

- **Generation.** During transcoding, a separate ffmpeg pass extracts frames at fixed intervals (every 2 seconds for seek-bar thumbnails; 1-3 at percentiles 10%/50%/90% for the main thumbnail).
- **Storage.** Thumbnails are small JPGs (~10-50 KB each). Store in S3 alongside the video. A 4-minute video has ~120 seek thumbnails × 30 KB = 3.6 MB. At 450K videos/day = ~1.6 TB/day of thumbnails.
- **Sprite sheets.** For the seek bar, pack thumbnails into a sprite sheet (a single image with all thumbs in a grid) plus a JSON describing the grid. One image fetch instead of 120. This is what YouTube does.
- **Serving.** Same CDN. Thumbnails get long TTLs (1 year; immutable).
- **Custom thumbnails.** Creators upload their own. Upload Gateway handles it as a small file (no chunking needed); store next to the auto-generated ones with a `is_custom=true` flag.

**5. Copyright takedown.**

- **Receive notice.** A trust-and-safety team reviews the notice. If valid, they call an internal API: `POST /admin/videos/<id>/block`.
- **Block.** Update `videos.status = blocked`. Revoke the signed-URL signing key for that video. Trigger CDN purge for `/v_<id>/*`.
- **Keep file accessible to legal.** Do not delete the S3 objects. Move them to a restricted bucket with a separate access policy that only the legal team and a logged-access process can read. This is the "available for review" requirement.
- **Audit trail.** Every block has a record: who blocked, when, which takedown notice, expected unblock date. Required for compliance.
- **Re-enable.** If the takedown is withdrawn, restore the signing key and clear the block. Files are still in S3, just in the restricted bucket; move them back.

**6. Drop-off analytics.**

Players emit periodic playback heartbeats with `(video_id, session_id, position_seconds)`. The Flink job aggregates:

- For each video and each 10-second bucket of position, count the number of distinct sessions that reached that bucket.
- A "retention curve" is the distribution: at position 0:00, 100% of sessions. At 1:00, maybe 78%. At 5:00, maybe 41%. The shape tells the creator where viewers leave.
- Stored in ClickHouse, partitioned by video_id and bucketed by position. Query latency in hundreds of ms even on huge data.
- Creators see the curve in their Studio. Updated nightly (real-time would be expensive and would not change creator decisions).

**7. DRM (Netflix mode).**

- Every transcoded segment is encrypted with AES-128 CTR mode using a per-content encryption key (CEK).
- The CEK is wrapped (encrypted) with the device's public key obtained from the DRM provider (Widevine, FairPlay, PlayReady).
- When a viewer hits play:
  1. Player fetches manifest. Manifest references a license server URL.
  2. Player requests license from license server. License server authenticates the user (logged in, subscription active, geo allowed).
  3. License server returns the wrapped CEK, valid for this device + this session, with a TTL.
  4. Player decrypts segments on-the-fly using CEK.
- Add to architecture: **License Server** (stateless API), **Key Management Service** (KMS storing CEKs by video_id), integration with **Widevine/FairPlay/PlayReady** SDKs.
- Trade-off: per-session license issuance adds 50-200ms to playback start. Manifest can be cached but license cannot.

**8. Subtitles and multiple audio tracks.**

HLS master playlist has separate `#EXT-X-MEDIA` entries for each audio and subtitle track:

```
#EXTM3U
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",NAME="English",LANGUAGE="en",DEFAULT=YES,URI="audio/en/playlist.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",NAME="Spanish",LANGUAGE="es",URI="audio/es/playlist.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",NAME="Hindi",LANGUAGE="hi",URI="audio/hi/playlist.m3u8"
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English",LANGUAGE="en",URI="subs/en.m3u8"
... (12 subtitle tracks)
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720,AUDIO="audio",SUBTITLES="subs",CODECS="avc1.4D401F,mp4a.40.2"
720p/playlist.m3u8
```

Audio is transcoded as a separate variant (one per language). Subtitles are WebVTT files served as their own playlists. Player picks one of each based on user preference. Audio is muxed by the player at playback time, not at encode time, so we don't need 7 video × 3 audio = 21 variants; we need 7 video + 3 audio = 10.

**9. Origin shield outage.**

Blast radius: every CDN edge that gets a cache miss now fans out to S3 directly. S3 request rate spikes 10-50x. Most viewers see cached content from the edge and notice nothing. Cache misses (long-tail videos, first viewers of new videos) see higher latency (S3 fetch from the nearest replica, possibly across regions).

Survival:
- Edges have `stale-while-revalidate` set: serve stale segments while refresh fails. Viewers see no interruption.
- S3 is provisioned for the burst (request-per-second buckets are large). Worst case, you hit the S3 rate limit and start getting 503s. Mitigation: pre-provision higher S3 limits if you know shield is your only buffer.
- Long-tail videos may be unplayable until shield recovers. Acceptable cost for a 15-30 min outage.

**10. Real-time view counts for ranking.**

Current pipeline: Player → Kafka → Flink (1-min windows) → ClickHouse → reads. Lag: ~5-30 min.

For 5-second freshness:
- Add a parallel high-frequency aggregator. Same Flink job but with 5-sec windows and writing to a separate output (Redis with INCR on a `views:<video_id>:current_5s` key, then a rolling sum).
- Output is approximate: dedup is best-effort (no exactly-once across Kafka rebalances). For ranking signals this is acceptable.
- The recommendation team reads from the high-frequency store. The displayed view count on the watch page reads from the slower exactly-once pipeline.

Trade-off: the high-frequency path is approximate. The slow path is exact. Two pipelines is the price of having both.

### 13. Trade-offs and what a senior would mention

- **Build vs buy CDN.** At YouTube scale Google built their own (Google Edge Network). At smaller scale you buy Cloudflare or Fastly. The breakeven is around 10 Tbps sustained egress; below that, buying is cheaper because you avoid PoP capex and operations. Above that, owning the CDN starts to make sense. A senior candidate names the breakeven, not just "buy" or "build".
- **Codec choice trade-off.**
  - H.264: encode fast, decode universal, file size baseline. Always ship.
  - H.265 (HEVC): 30% smaller files than H.264 at equal quality. Encode 2-5x slower than H.264. Decoder support: iOS yes, Android yes, web partial. Trade-off: bandwidth savings vs encode cost vs licensing fees (H.265 patent pool fees are real).
  - AV1: 30% smaller than H.265, but 10-30x slower to encode. Royalty-free. Decoder support: modern Chrome, Firefox, recent Android, modern Apple. Use for the top 1-5% of videos by watch time, where the bandwidth savings amortize the encode cost in days. Not viable for routine encoding yet.
- **Storage tier transitions: when do they pay off?** Demote a video after 7-day no-view. If 30% of demoted videos get re-promoted within 30 days, demotion was premature. Tune the threshold per content category: news ages fast (aggressive demote), evergreen tutorials less so. A senior candidate measures this.
- **Push vs pull manifest updates.** Master playlists change as variants come online. Push (websocket to player) would cut start latency for newly-uploaded videos, but adds complexity. Pull with short TTL is the standard answer; only adopt push at extreme scale.
- **Single bucket vs region-per-bucket.** Cross-region S3 replication for hot videos costs replication bandwidth. The trade-off: faster reads from local region vs bandwidth cost to replicate. Replicate hot videos (top 5% by views) eagerly; replicate the rest on first read miss (origin shield in another region triggers replication).
- **What I would revisit at 10x scale.**
  - Custom hardware encoding (Argos-class video transcoding ASICs that Google deploys). 10-100x more efficient than CPU encode at huge capex.
  - Per-shot encoding (encode each scene at the optimal bitrate, not a global ladder). Saves 20%+ bandwidth. Netflix does this.
  - Move manifests to a CDN-edge-computed format (build the manifest at the edge based on viewer's device + region) instead of pre-built static manifests.

### 14. Common interview mistakes

- **Designing one system for upload and playback.** They are two different systems sharing a database. A candidate who keeps them in one architecture diagram gets lost.
- **Skipping the CDN tiers.** "Use a CDN" is not enough. The shield is the critical piece for keeping S3 cost down. Mention all three tiers.
- **No mention of segment alignment across variants.** Players cannot switch mid-stream without aligned keyframes. This is an essential detail that the average candidate skips.
- **Hand-waving the transcoding compute.** "We'll have transcoders" is not enough. Mention separate node pools per codec, autoscaling on queue depth, idempotent output for restartability, and the AV1 cost story.
- **Forgetting resumable upload.** A 5GB single PUT fails on a phone connection. Resumable upload (TUS or S3 multipart) is mandatory.
- **No view-count pipeline.** Every viewer triggers a view; you cannot UPDATE a DB row per view at 125M concurrent users. Async pipeline via Kafka + Flink is the standard answer.
- **Ignoring storage tiering.** The 5x cost reduction story is important; storage costs in PB are real money.
- **Ignoring DRM when asked about Netflix.** Even if not asked, mention the license server as the difference between YouTube and Netflix designs.
- **Treating live as a small variation on VOD.** Live needs real-time transcoding, RTMP/WebRTC ingest, LL-HLS or LL-DASH delivery, and a different latency budget. Acknowledge it is a different system.
- **Overengineering thumbnails.** They are tiny. A sprite sheet plus CDN handles them. Don't spend 5 minutes on this when the interviewer wants to hear about CDN tiering.

If you can hit 8 of these 10 in your design walkthrough, you are interviewing well. Hit 6 and you are at the bar for a senior role at a media company. The remaining 4 are what separate a senior from a staff engineer who has actually run a transcoding farm.
