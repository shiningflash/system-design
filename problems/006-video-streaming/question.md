---
id: 6
title: Design YouTube / Netflix (Video Streaming)
category: Streaming
topics: [cdn, adaptive bitrate, transcoding, storage tiers, hls dash]
difficulty: Hard
solution: solution.md
---

## Scene

It is 2:30 PM. The interviewer just got off a call about a CDN cost overrun and they are still thinking about egress. They write on the whiteboard:

> *Design YouTube. Or Netflix, pick one. I want to see the end-to-end story.*

You are about to draw a box labeled "video service". Stop. The first thing to clarify is not how big, it is *which half of the system*. Video streaming is two systems pretending to be one. The upload and transcoding pipeline is write-heavy, batch, compute-bound. The playback path is read-heavy, latency-bound, CDN-bound. They share almost nothing except a metadata database and a storage bucket. A candidate who tries to design both at once with one architecture gets lost in ten minutes.

So my first question is: which side are we going deep on, or do you want both at moderate depth? For practice, assume both, but understand you would scope it on a real interview.

## Step 1: clarify before you design

Take five minutes. Don't draw anything. What would you ask the interviewer? Aim for seven or eight questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. YouTube or Netflix. These look the same but they are not. YouTube has user-generated uploads (any quality, any container, billions of long-tail videos). Netflix has a curated catalog (studio-mastered files, premium codecs, every title is hot). The pipeline, storage tiering, and DRM all differ. For this design, assume YouTube-style UGC with a long tail.
2. Live vs VOD. Live streaming and VOD share almost no infrastructure. Live needs sub-3-second glass-to-glass latency, edge ingest, low-latency HLS or WebRTC, and a real-time transcoding model. VOD is batch transcoded once and served forever. Scope to VOD; mention live as an extension.
3. Geographic scope. Global means CDN tiers, regional origin shields, and cross-region replication for source files. The CDN cost story dominates the architecture.
4. Quality levels and codecs. How many bitrates? Which codecs? The standard ladder is 240p / 360p / 480p / 720p / 1080p / 1440p / 4K. Codecs: H.264 for compatibility, H.265 (HEVC) for mobile bandwidth savings, AV1 for high-traffic videos where the compute cost amortizes. This single answer shifts transcoding compute by 5x.
5. Upload constraints. Max video length? Max file size? Resumable uploads required? YouTube allows 12-hour videos and 256GB files. Resumable upload is mandatory at that size; a flaky home connection cannot restart a 50GB upload from zero.
6. DRM and paid content. Free, ad-supported, paywalled? YouTube is free plus ads. Netflix needs Widevine / FairPlay / PlayReady DRM, which adds a license server, encrypted segments, and per-device key exchange. Different design.
7. Recommendations and search. Are search and recs in scope? These are huge separate systems. The right answer is "out of scope, but I'll show the integration point."
8. Analytics. View counts, watch time, retention curves: real-time or batch? View counts on the watch page can be eventually consistent (the 30-minute YouTube delay is famous). Real-time analytics for creators is a separate pipeline.

If you only asked "how many users" you missed the most important questions. The codec ladder and the live-vs-VOD scope each move the compute budget by an order of magnitude.

</details>

## Step 2: capacity estimates

Assume the interviewer gave you these numbers (YouTube-scale):

- 500 hours of video uploaded *per minute*
- 1 billion hours watched *per day*
- Average view: 10 minutes
- Average upload: 4 minutes
- Bitrate ladder: 240p (400 kbps), 360p (700 kbps), 480p (1.2 Mbps), 720p (2.5 Mbps), 1080p (5 Mbps), 1440p (9 Mbps), 4K (20 Mbps)
- Codecs: H.264 (always), H.265 (for 720p and above), AV1 (for the top 1% of hot videos)
- Source file retention: forever
- Target playback start latency: under 2 seconds

Compute (do this on paper before revealing):

1. Upload bytes per second (source files, before transcoding)
2. Number of new videos per day
3. Concurrent viewers
4. Egress bandwidth at peak
5. Storage for source files over one year
6. Storage for all transcoded variants
7. Transcoding compute (rough order of magnitude)

<details>
<summary><b>Reveal: the math</b></summary>

Upload rate. 500 hours per minute = 500 × 60 = 30,000 seconds of video per minute = 500 seconds of video per second. At an average source bitrate of 10 Mbps for a typical 1080p phone upload, ingest bandwidth is 500 × 10 Mbps = 5 Gbps sustained. Peak roughly 3x = 15 Gbps.

Videos per day. 500 hours × 60 min × 24 hours / 4 min average length = 450,000 new videos per day. About 5 new videos per second.

Concurrent viewers. 1B hours / day = 1B × 3600 / 86400 = 42 million viewer-seconds per second, i.e. 42M concurrent viewers on average. Peak 3x = 125M concurrent. At average bitrate 2 Mbps (most viewing is on mobile at 480p / 720p), egress is 125M × 2 Mbps = 250 Tbps peak. This number is the whole point of having a CDN.

Egress. 250 Tbps is roughly 25 PB/hour, 600 PB/day. At commodity CDN pricing this is the dominant cost of the system. Cache hit rate matters more than almost anything else.

Source storage per year. 500 sec/sec × 86400 × 365 × (10 Mbps / 8 bits) = 500 × 86400 × 365 × 1.25 MB = ~20 PB/year just for the source file. Sources are kept forever for re-transcoding (new codecs, new resolutions). This number compounds.

Transcoded variants. A typical ladder produces 7 resolutions × 2 codecs (avg) = ~14 variants. Each variant is smaller than source (compressed at lower bitrate), but in aggregate the variants are roughly 2-3x the source size. So ~50 PB/year of transcoded output. Combined with source: ~70 PB/year of new storage, growing.

Transcoding compute. Software H.264 encoding runs roughly at 1x to 5x real-time on a modern CPU core (depends on preset). 500 sec/sec input × 14 variants = 7000 sec/sec of output to encode. At 2x real-time average that is ~3500 CPU cores running continuously just for routine transcoding. AV1 encoding is 10-50x slower, so even encoding the top 1% of videos in AV1 adds another several thousand cores. Transcoding is the second-biggest cost line after CDN egress.

The takeaway. This system has two huge cost lines (CDN egress, transcoding compute) and one huge storage problem. Everything else is small. Most of the architecture exists to reduce one of those three numbers.

</details>

## Step 3: the upload pipeline

Before drawing the full architecture, think through the upload pipeline by itself. A creator uploads a 5GB MP4 from their phone. What happens? Take ten minutes to write the steps. Consider: how does the upload survive a dropped connection? Where does the file land? Who decides when to start transcoding? How does the creator know the upload succeeded vs the video is ready to publish?

<details>
<summary><b>Reveal: upload pipeline steps</b></summary>

The upload pipeline has six stages. Each must be independently resumable; an upload that fails at stage 5 should not redo stages 1 through 4.

1. Initiate. Client calls `POST /uploads` with metadata (filename, total size, sha256 of the source if known, intended title and description). Server returns an upload ID and a signed URL to the chunk endpoint. Server creates a row in `videos` table with `status = uploading`.

2. Chunked resumable upload (TUS protocol or S3 multipart). Client splits the file into 5-25 MB chunks. Each chunk is `PATCH /uploads/<id>` with a `Content-Range` header. Server writes the chunk to object storage (S3) at a per-upload prefix. The server tracks `bytes_received` per upload. On any failure, client calls `HEAD /uploads/<id>` to find out where to resume.
    - Why chunks, not single PUT. A 5GB single PUT cannot resume. With chunks, only the in-flight chunk has to retry. TUS adds a resumable spec on top of HTTP; alternatives are S3 multipart upload (let the client upload directly to S3 with presigned URLs) or a tus-style protocol on your own gateway.
    - Why direct-to-S3 is tempting and risky. Presigned URLs let the client upload directly to S3 and remove your gateway from the byte path. Saves you 15 Gbps of ingest. The trade-off: you no longer control the upload, so abuse detection, malware scan, and rate limiting are harder. Compromise: presigned for trusted creators, gateway for new accounts.

3. Finalize. When all chunks are received, client calls `POST /uploads/<id>/finalize`. Server assembles the chunks into a single source file (or marks the multipart upload complete in S3). Verifies sha256. Moves the source to `s3://video-source/{video_id}/source.mp4`. Updates `videos.status = transcoding`.

4. Enqueue transcoding jobs. Server publishes a message to Kafka topic `transcode.requested` with `{video_id, source_url, target_ladder}`. The target ladder is determined by source quality: a 480p source does not need a 4K variant.

5. Transcoding workers consume. A pool of GPU/CPU workers picks up jobs. For each variant in the ladder, a worker:
    - Downloads the source (or streams it).
    - Runs `ffmpeg` with the codec, bitrate, and segmentation parameters.
    - Outputs HLS segments (`.ts` or `.m4s`) and a per-variant playlist (`.m3u8`).
    - Uploads the segments to `s3://video-transcoded/{video_id}/{variant}/`.
    - On success, posts to Kafka `transcode.completed` with `{video_id, variant, segment_count}`.

6. Manifest assembly. A separate service watches `transcode.completed` events. When all variants in the ladder are done, it generates the master playlist (`master.m3u8` for HLS, `manifest.mpd` for DASH) that references each variant playlist. It writes the master to `s3://video-transcoded/{video_id}/master.m3u8`. Then it updates `videos.status = ready` and fires `video.published` to a Kafka topic that downstream systems (search indexer, notification service, recommendation service) consume.

The creator polls `GET /videos/{id}` to see status transition `uploading → transcoding → ready`. Some platforms (YouTube) show the video as available at 360p first, while higher resolutions are still encoding; that requires the manifest assembler to publish a partial master playlist after each variant completes.

</details>

## Step 4: sketch the high-level architecture

Here is an intentionally incomplete diagram. Fill in the six `[ ? ]` boxes. Hint: think about what sits between the client and storage on upload, what fans out transcoding work, where the manifest is built, what tiers the playback CDN has, and where metadata lives.

```
   ┌──────────┐
   │ Creator  │
   └────┬─────┘
        │  upload chunks
        ▼
   ┌─────────────────┐
   │   [ ? A ]       │  (handles chunked upload, validates, writes to source bucket)
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │  Source object  │
   │  storage (S3)   │
   └────────┬────────┘
            │  enqueue transcode jobs
            ▼
   ┌─────────────────┐
   │   [ ? B ]       │  (job queue / message bus)
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │   [ ? C ]       │  (transcoding worker pool, GPU + CPU)
   └────────┬────────┘
            │  write transcoded segments
            ▼
   ┌─────────────────┐
   │  Transcoded     │
   │  object storage │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │   [ ? D ]       │  (assembles master.m3u8 / manifest.mpd, publishes ready event)
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │   [ ? E ]       │  (catalog: title, description, owner, status, available variants)
   └─────────────────┘

   ─── playback path ───

   Viewer ──► [ ? F ] ──► origin shield ──► transcoded object storage
              (edge CDN with cached segments and manifests)
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
   ┌──────────┐
   │ Creator  │
   └────┬─────┘
        │  upload chunks (TUS / S3 multipart)
        ▼
   ┌─────────────────┐
   │ Upload Gateway  │  Resumable upload, sha256 verify, rate limit,
   │ (stateless API) │  presigned-URL issuance, quota check.
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │  s3://video-    │  Source bucket. Lifecycle: move to cold tier
   │  source/        │  after 30 days if no re-transcode in flight.
   └────────┬────────┘
            │  Kafka: transcode.requested
            ▼
   ┌─────────────────┐
   │   Kafka topic   │  Partitioned by video_id. Durable, replayable.
   │ transcode.req   │  Backpressure when transcode farm saturated.
   └────────┬────────┘
            │
            ▼
   ┌──────────────────────┐
   │ Transcoding Workers  │  Kubernetes pool. GPU nodes for AV1 / HEVC,
   │ (FFmpeg + GPU/CPU)   │  CPU nodes for H.264. Autoscales on queue depth.
   │ Each worker: 1 job   │  One job = one (video_id, variant) pair.
   └────────┬─────────────┘
            │  write segments
            ▼
   ┌─────────────────┐
   │ s3://video-     │  Transcoded variants. Per-video prefix.
   │ transcoded/     │  Hot videos replicated to regional buckets.
   └────────┬────────┘
            │  Kafka: transcode.completed
            ▼
   ┌─────────────────┐
   │ Manifest        │  Consumes transcode.completed. When all variants
   │ Assembler       │  done, writes master.m3u8 / manifest.mpd.
   │                 │  Updates Metadata DB status -> ready.
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ Metadata DB     │  Cassandra or DynamoDB. videos table:
   │ (videos, vari-  │  video_id PK; title, desc, owner_id, status,
   │ ants, segments) │  available_variants, created_at, view_count.
   └─────────────────┘

   ─── playback path ───

   ┌────────┐   ┌──────────┐    ┌────────────┐    ┌──────────────┐
   │ Viewer │──►│ Edge CDN │───►│  Regional  │───►│ Origin       │──► transcoded
   │ player │   │ (PoP, in │    │  origin    │    │ (S3 + small  │    object store
   │ ABR    │   │ city)    │    │  shield    │    │  signing svc)│
   └────────┘   └──────────┘    └────────────┘    └──────────────┘
       │
       │ first: GET /watch/{video_id} -> Watch Page
       │ then:  GET master.m3u8       -> CDN (cached)
       │ then:  GET segment_N.ts      -> CDN (cached) for each segment
       │
       └──► reports playback events to:
            ┌─────────────────┐
            │ Telemetry → Kafka│ → ClickHouse for analytics,
            │ (QoE, view, ad)  │   counter store for view counts.
            └─────────────────┘
```

Component responsibilities:

- Upload Gateway. Handles TUS or S3 multipart. Stateless. Verifies content-length, sha256. Enforces per-creator quotas. The gateway can offload the actual byte stream to S3 via presigned URLs for trusted creators.
- Source bucket (S3). Cheap, durable, eventual move to a colder tier (S3 Standard → S3 Infrequent Access → Glacier) by lifecycle policy.
- Kafka transcode topic. Durable buffer between upload and transcode. Lets the transcode farm be sized for steady-state, not peak.
- Transcoding Workers. The compute-bound part. One worker, one job. Idempotent: re-running a job overwrites the same output path.
- Transcoded bucket. Holds all variants. Hot videos get replicated to regional buckets near the CDN; cold videos stay in one region.
- Manifest Assembler. Stateless. Watches transcode events; emits master playlist when ready.
- Metadata DB. Catalog. Wide-column store (Cassandra) is a fine pick: one row per video keyed by video_id, columns for each variant's status.
- Edge CDN / origin shield / origin. Three-tier cache. The edge serves segments to viewers from local PoPs. On miss it pulls from the regional shield. Shield miss goes to origin (S3 + URL signer). The shield exists specifically to reduce S3 fan-out from N edges to 1.

</details>

## Step 5: playback path

Now zoom in on what happens when a viewer hits play. The viewer's player issues HTTP requests. What does it ask for, in what order, and how does the player decide which bitrate to use? Take ten minutes to think through, then reveal.

<details>
<summary><b>Reveal: playback sequence and ABR</b></summary>

The playback sequence for an HLS video:

1. Player loads the watch page. It calls `GET /api/videos/{video_id}` which returns metadata: title, owner, thumbnail URLs, the master playlist URL, available languages, ad markers.
2. Player fetches the master playlist. `GET https://video-cdn.example.com/{video_id}/master.m3u8`. This returns ~1 KB:
    ```
    #EXTM3U
    #EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=426x240,CODECS="avc1.42E01E"
    240p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=700000,RESOLUTION=640x360,CODECS="avc1.42E01E"
    360p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720,CODECS="avc1.4D401F"
    720p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028"
    1080p/playlist.m3u8
    ```
    Each `BANDWIDTH` tells the player the bitrate of that variant. Each line points to a variant-level playlist.
3. Player picks a starting variant. Usually starts with 360p or 480p (don't start at 1080p; you don't know the network yet). Fetches that variant's playlist:
    ```
    #EXTM3U
    #EXT-X-TARGETDURATION:6
    #EXTINF:6.0,
    segment_0.ts
    #EXTINF:6.0,
    segment_1.ts
    #EXTINF:6.0,
    segment_2.ts
    ...
    ```
    Each segment is a self-contained ~6-second chunk of video, encoded at that variant's bitrate, addressable by HTTP.
4. Player fetches segment 0, decodes, starts playback. While segment 0 is decoding, it starts fetching segment 1. Buffer fills to ~30 seconds ahead.
5. Player measures download speed per segment. If segment 0 (a ~1MB chunk for 360p) downloaded in 200ms, that is 40 Mbps available bandwidth. Player can step up to 1080p (5 Mbps) safely with margin.
6. Player switches variants between segments. Variants are encoded with aligned segment boundaries specifically so the player can switch mid-stream. Next request goes to `1080p/segment_3.ts` instead of `360p/segment_3.ts`. The viewer sees the resolution jump.

Adaptive bitrate (ABR) algorithm. The player runs a simple state machine:

- Maintain a buffer of ~30 seconds of video ahead.
- Measure bandwidth as (segment size / segment download time), smoothed over the last few segments.
- Pick the highest variant whose bitrate is below (smoothed bandwidth × safety factor, e.g., 0.8).
- If buffer drops below 5 seconds, drop one variant immediately to refill faster.
- If buffer is full and bandwidth is high, try the next variant up.

This is why every variant must have identical segment boundaries. If 360p has segment boundaries at 6.0s, 12.0s, 18.0s, then 1080p must also break exactly at 6.0s, 12.0s, 18.0s. Otherwise switching mid-stream produces visible glitches.

HLS vs DASH. Both do the same thing:

- HLS: Apple's protocol. Manifests are `.m3u8` text files. Segments are `.ts` (MPEG-2 Transport Stream) or `.m4s` (fragmented MP4). Universal browser support. Default on iOS.
- DASH: ISO standard. Manifests are `.mpd` XML files. Segments are always `.m4s`. Default on Android.
- Modern players support both. You typically encode segments in fragmented MP4 (`.m4s`) and serve both an HLS-flavored `.m3u8` manifest and a DASH-flavored `.mpd` manifest pointing at the same byte files. This is called CMAF (Common Media Application Format).

Where the CDN comes in. Every one of these requests (master, variant playlist, every segment) is a plain HTTP GET against a URL that includes the video_id and segment number. The CDN treats them as static files. Hot videos have hit rates above 95%. The origin serves only the long tail and the first viewer of every new video.

</details>

## Step 6: storage tiering

You will accumulate exabytes of video over time. Some videos get a billion views (Gangnam Style). Most get fewer than 100 views ever. Storing them all on the same medium is wasteful. Think through the tiers, then reveal.

<details>
<summary><b>Reveal: storage tiers</b></summary>

A video has two storage components: the source file (the originally uploaded master) and the transcoded variants (what viewers actually watch). Each tiers independently.

| Tier | What lives here | Medium | Cost (rough, per GB-month) | Retrieval latency |
|------|-----------------|--------|---------------------------|--------------------|
| CDN edge cache | Hot segments (videos getting views right now) | Edge SSD/RAM at PoPs | n/a (cache, not storage) | sub-50ms |
| Origin shield cache | Warm segments (popular videos this region) | Regional SSD | $0.10 | sub-100ms |
| Hot object store | Transcoded variants for popular videos | S3 Standard (SSD-backed) | $0.023 | ~30ms |
| Warm object store | Transcoded variants for cold videos | S3 Infrequent Access (HDD-backed) | $0.0125 | ~30ms (but retrieval fee per GB) |
| Cold object store | Source files after 30 days, transcoded variants of dead videos | S3 Glacier Instant | $0.004 | seconds |
| Archive | Source files of videos with no views in a year | S3 Glacier Deep Archive (tape) | $0.00099 | 12 hours |

How content moves between tiers.

- New upload. Source goes to hot; transcoded variants go to hot. CDN warms naturally on first views.
- Daily view counts feed a tiering job. Variant gets demoted (hot → warm) when no views in 7 days. Demoted (warm → cold) when no views in 90 days.
- Promotion on read. If a cold variant gets a sudden spike (an old video goes viral), the first viewer pays a one-time retrieval cost, and the job promotes it back to hot.
- Source files. Kept forever in case you need to re-transcode (new codec like AV1 ships, you want to re-encode the catalog). After 30 days, source moves from hot to cold. Re-transcoding from cold takes seconds to minutes; acceptable for a background job.

The economics.

- 70 PB/year of new content, growing.
- If all of it lived in S3 Standard: $0.023/GB/mo × 70 PB × 12 = $20M/year just for the first year's content storage. After 10 years compounded: hundreds of millions.
- Tiered: roughly 5% stays hot, 15% warm, 80% cold. Blended cost: ~$0.005/GB/mo. Same data costs ~$4M/year. This is real money.

The trade-off you must articulate.

- More aggressive tiering means lower storage cost but more retrieval cost when a long-tail video gets a sudden view.
- The breakeven: tier moves should pay back within the demote-to-promote period. If most demoted videos stay demoted, demotion pays. If 30% get promoted back within a month, demotion was premature and net-negative.
- Real systems instrument both demote count and within-30-day promotion count; tune the demote threshold based on the ratio.

Where transcoding interacts with tiering.

- When a new codec ships (you decide to add AV1 for the top 1% of videos), you launch a backfill that reads source files and produces new variants. If sources are in cold storage, the backfill scheduler has to either (a) accept the per-GB Glacier retrieval cost or (b) wait for a periodic warm-up window where sources are bulk-restored for cheap. Mature systems do the latter.

</details>

## Follow-up questions

Try answering each in three or four sentences before reading the solution.

1. A creator deletes a video that has 100M views and is in every CDN edge cache. How do you make sure it stops playing for every viewer within 60 seconds? CDN cache invalidation at this scale is not free.
2. A new codec (AV1 successor, say "AV2") ships. You want to re-encode the top 10,000 most-watched videos to take advantage of better compression and lower bandwidth bills. Design the backfill, including how you decide which videos are worth re-encoding given that re-encoding consumes compute.
3. Live streaming. A creator wants to go live to 5M concurrent viewers with sub-3-second glass-to-glass latency. What changes in the architecture? Hint: transcoding becomes real-time, ingest is now WebRTC or RTMP, segments are LL-HLS or LL-DASH.
4. Thumbnails. Every video has 1-3 thumbnails plus auto-generated frame thumbnails for the seek-bar preview. How do you generate them, store them, and serve them at scale? They are tiny but plentiful (millions of thumbnails per day).
5. A copyright takedown notice arrives for a specific video. How does the system block playback globally within 5 minutes while also keeping the file available for legal review (you cannot just delete it)?
6. Watch time analytics. A creator wants to see "viewers dropped off at the 3:47 mark." Where does that data come from, and how do you compute it across billions of viewing sessions?
7. DRM. Switch to Netflix mode. Now every segment must be encrypted, every device must request a per-session decryption key, and the key must be unique per (user, session). Add the DRM components to your diagram and explain the key flow.
8. Subtitles and audio tracks. A video has English, Spanish, and Hindi audio plus 12 subtitle languages. How are these encoded into HLS / DASH? What does the manifest look like?
9. Origin shield went down. All your edges now hit S3 directly. What is the blast radius, and how do you survive until shield comes back?
10. The recommendation team wants real-time view-count signals (per video, per minute) for ranking the front page. They want it at less than 5-second freshness. Your current view-count pipeline is 30 minutes behind. Where in the architecture do they plug in, and what do you give up?

## Related problems

- [URL Shortener (001)](../001-url-shortener/question.md). The CDN concepts (edge cache, TTL trade-offs, hot key behavior) used here are introduced there in a simpler context.
- [Notification System (010)](../010-notification-system/question.md). The upload-complete event ("your video is ready") and the "new video from a creator you follow" notification both ride on that system. Same fan-out patterns.
- [News Feed (002)](../002-news-feed/question.md). The watch page is one row in a feed; the feed and video systems share the metadata store and the "creator follows" graph.
- [Distributed Cache (009)](../009-distributed-cache/question.md). The manifest cache and metadata cache lean on the same primitives.
