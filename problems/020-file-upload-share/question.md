---
id: 20
title: Design a File Upload & Share Service (Dropbox-lite)
category: Storage
topics: [chunked upload, share links, permissions, storage tiers, virus scan]
difficulty: Easy
solution: solution.md
---

## Scene

The interviewer drops a one-line brief into the doc.

> *Build a Dropbox-lite. Users upload files up to 5GB each, share them with other users by direct invite or by link, and set permissions (view, download, edit). Storage and bandwidth cost money, so don't waste either.*

They lean back. "Take it from a weekend project to something that could survive a million users. I want to hear what you'd build at each stage and what you'd defer."

This problem looks like a CRUD app sitting on top of S3. It is not. The interesting parts are: how the upload survives a flaky hotel WiFi for a 4GB video, how a share link can be revoked without invalidating every other link, how the same file uploaded by 50 people is stored once, and how a file that nobody has touched for two years stops costing you anything.

The candidate who jumps to "we POST the file to S3 and store a row in Postgres" loses 60% of the depth of the problem. We are going to walk through every layer.

## Step 1: clarify before you design

Take 6 minutes. Aim for at least 8 questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Max file size.** "5GB was mentioned. Are there hard cutoffs? What about a 50GB video?" A 5GB cap rules out single-PUT uploads and forces chunked or multipart. A 50GB cap forces presigned-direct-to-S3 because tunneling through your app servers is bandwidth suicide.
2. **Sync versus share-only.** "Is this Dropbox-the-sync-client, or Google-Drive-the-web-share? Sync means delta-sync, conflict resolution, and local file watchers. Share-only is a much smaller problem." Almost always the answer is share-only for an interview; sync is its own design.
3. **Virus scanning.** "Do we scan uploads? Synchronously block on the result, or async-quarantine after the fact?" Sync scan is slow and breaks the upload UX. Async means a file can be downloaded for a few minutes before being quarantined.
4. **Versioning.** "Does editing a file create a new version, or overwrite? How many versions kept? Forever or 30 days?" Versions multiply storage cost. A safe default is 10 versions or 30 days, whichever comes first.
5. **Bandwidth limits.** "Per-user upload and download caps? Per-account total storage quota?" Quotas are not optional once you have a free tier. The check has a sneaky race condition we will come back to.
6. **Sharing model.** "Link share (anyone with the link), direct invite (must have an account), or both? What permission scopes: view-only preview, download, edit?" Each scope has security implications and changes the download URL flow.
7. **File preview.** "Do we render PDFs and images in the browser, or just offer download?" Preview means a thumbnailing pipeline, which is a sub-service in itself.
8. **Compliance and retention.** "Are we GDPR-required to fully delete on request? Are there industries (healthcare, finance) that require encryption-at-rest with customer-managed keys?" The answer changes the storage layer.
9. **Geographic reach.** "Single region or global? If global, do we replicate files between regions or only metadata?" File replication is expensive; metadata replication is cheap.
10. **Authentication.** "Existing auth system, or do we build it? Single-sign-on?" Out of scope for most interviews, but worth confirming.

A strong candidate also names what is *out* of scope: real-time collaborative editing (Google Docs), full-text search inside documents, integrations with third-party apps. These are downstream products built on top of the storage primitive.

</details>

## Step 2: capacity estimates

The interviewer gives you two regimes.

**Small startup:**
- 10,000 users
- Average 5 file uploads per user per week, average size 5MB
- Reads (downloads + previews) 10x writes

**Dropbox-scale:**
- 100M active users
- Average 20 uploads per user per week, average size 8MB (mix of photos, docs, occasional video)
- Reads 10x writes
- Storage retention: indefinite by default

Compute, for both:

1. Uploads per second (sustained and peak)
2. Downloads per second
3. Storage growth per year (raw, before dedup)
4. Egress bandwidth at peak download
5. How much storage gets stale (untouched for 90 days) in year 2

<details>
<summary><b>Reveal: the math</b></summary>

**Small startup (10k users):**

- Uploads: 10,000 × 5 / week = 50,000/week = ~7,000/day = **~0.08 uploads/sec sustained**, ~0.25 peak. Trivial.
- Downloads: 10x = ~0.8/sec sustained, ~2.5 peak. Still trivial.
- Storage growth: 50,000/week × 5MB × 52 = ~13TB/year. Fits in one S3 bucket with no thought.
- Egress at peak: 2.5 downloads/sec × 5MB = 12.5 MB/s = ~100 Mbps. Modest.
- Stale at year 2: probably 60-70% of files (most uploads are looked at once, never again).

A single app server plus S3 plus Postgres handles all of this. The system is not interesting from a throughput angle. It is interesting because of the upload UX for a 5GB file and the share-link permission model.

**Dropbox-scale (100M users):**

- Uploads: 100M × 20 / week = 2B/week ≈ 286M/day ≈ **~3,300 uploads/sec sustained**, ~10k peak.
- Downloads: 10x = ~33,000/sec sustained, ~100k peak.
- Storage growth: 286M/day × 8MB ≈ 2.3 PB/day ≈ **~840 PB/year before dedup**. With ~30% dedup savings (people share popular images, identical PDFs, etc.) call it ~580 PB/year.
- Egress at peak: 100k × 8MB = 800 GB/s = ~6.4 Tbps. This is CDN territory; you cannot serve this from a single region without it.
- Stale at year 2: roughly 70-80% of stored bytes are untouched for 90 days. At 1.2EB stored after 2 years, ~900PB is cold. Lifecycle policy to S3 IA + Glacier saves ~70% of storage cost on that tier.

**Key insights from the math:**

- **The system is read-heavy by request count but write-heavy by bytes.** Uploads dominate ingress bandwidth; downloads (with CDN) dominate egress to end users.
- **Storage cost is the headline expense.** At 840 PB/year × ~$0.023/GB/month (S3 standard), you are paying ~$230M/year for raw storage. Lifecycle policies and dedup are not optimizations, they are survival.
- **Per-server bandwidth is the bottleneck if you tunnel uploads through app servers.** A 10Gbps NIC at 100% sustained handles ~1.25 GB/s, which is ~150 concurrent uploads of an 8MB file per second. You will run out of NICs long before you run out of CPU. Hence presigned URLs.
- **The metadata DB is small.** 100M users × 1000 files/user = 100B files × ~500 bytes/row = ~50TB. Sharded Postgres or a wide-column store handles it. The bytes are in S3; the metadata is what you actually run a database for.

</details>

## Step 3: upload protocols

The single biggest design decision is *how the bytes get from the client to your storage*. Three serious options. Take 10 minutes to write down the pros and cons of each before peeking.

- **A. Direct upload (single HTTP POST).** Client streams the whole file in one request to your API server, which forwards to S3.
- **B. Chunked resumable upload (TUS protocol, or S3 multipart).** Client splits the file into 5-100MB chunks and uploads them with retry-per-chunk semantics.
- **C. Presigned S3 URL.** Your server hands the client a short-lived signed URL; the client uploads directly to S3, skipping your app servers entirely.

<details>
<summary><b>Reveal: comparison</b></summary>

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Direct (single POST)** | `POST /upload` with multipart body. Server buffers or streams to S3. | Simple. One request, one response. Works in any HTTP client. | Whole upload restarts on any network blip. 4GB upload on hotel WiFi never finishes. Saturates your app server bandwidth. Hard to show real progress. |
| **Chunked resumable (TUS, S3 multipart)** | Initiate session → upload chunk 1 of N → ... → finalize. Each chunk is its own HTTP request. Failed chunks retry independently. | Survives flaky networks. Real progress bars. Parallel chunk upload speeds large files. Server can pause and resume. | More complex protocol. Server tracks per-session state (which chunks landed). Garbage collection of abandoned uploads. |
| **Presigned S3 URL** | `POST /upload/init` returns a presigned URL valid for ~1h. Client `PUT`s directly to S3. On success, client calls `POST /upload/finalize` so server knows. | Zero bandwidth through your servers. Cheapest at scale. S3 handles the heavy lifting. Can combine with multipart for huge files. | Client must speak S3 directly. Quota enforcement is harder (server didn't see the bytes). Virus scan is post-hoc. Signed URL leaks = anyone can upload to that key. |

**The recommendation: chunked + presigned, depending on file size.**

- Files under 5MB: single POST through the app server is fine. Simpler client.
- Files 5MB to 5GB: presigned S3 multipart upload. Client splits into 8MB chunks, gets a presigned URL per chunk (or one initiation token), uploads chunks in parallel directly to S3, calls finalize when done.
- Files over 5GB: same as above but bumped chunk size to 64MB and enforced server-side limits.

The hybrid keeps the API simple for the common case and offloads the painful case (big files) to S3 entirely.

**Why TUS at all?** TUS (`tus.io`) is an open protocol for resumable uploads. It is implemented as a Go/Node server library that the client speaks to. Useful if your storage is not S3 (some companies use MinIO, NetApp, or their own object store) and you want to give clients a standard protocol. If you are on AWS, S3 multipart is more native; TUS is for the "S3 alternative" case.

</details>

## Step 4: incomplete architecture diagram

Sketch the high-level architecture. Fill in the five `[ ? ]` boxes. Each placeholder is one of: upload service, object store, metadata DB, share link resolver, virus scanner.

```
                Client (browser, mobile, desktop sync agent)
                              │
                              ▼
                    ┌──────────────────┐
                    │   API Gateway    │  (auth, rate limit, request shaping)
                    └────────┬─────────┘
                             │
        upload init,         │       download,
        finalize, list       │       share link redeem
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
        ┌─────────────┐            ┌─────────────┐
        │  [ ? ]      │            │  [ ? ]      │   (resolves /share/<token>
        │             │            │             │    to a file + permission)
        └─────┬───────┘            └──────┬──────┘
              │                            │
              │   presigned URLs           │
              │                            │
              ▼                            ▼
        ┌──────────────────────────────────────┐
        │  [ ? ]                               │  (the bytes live here;
        │                                      │   cheap, durable, huge)
        └──────────────────────────────────────┘
              ▲
              │
              │   reads/writes
              │
        ┌─────┴────────┐
        │  [ ? ]       │   (file_id, owner, name, size,
        │              │    hash, status, version)
        └──────────────┘

        Async pipeline:
            ┌──────────────────┐
            │  [ ? ]           │  (consumes "file finalized" events,
            │                  │   scans, marks quarantined if positive)
            └──────────────────┘
```

<details>
<summary><b>Reveal: completed architecture</b></summary>

```
                Client (browser, mobile, desktop sync agent)
                              │
                              ▼
                    ┌──────────────────┐
                    │   API Gateway    │  (auth, rate limit, request shaping,
                    │   + WAF          │   blocks obvious abuse)
                    └────────┬─────────┘
                             │
        upload init,         │       download,
        finalize, list       │       share link redeem
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
        ┌─────────────────┐         ┌──────────────────┐
        │  Upload Service │         │  Share Link      │
        │  (mints         │         │  Resolver        │
        │   presigned     │         │  (token →        │
        │   URLs, tracks  │         │   file + perms,  │
        │   session       │         │   checks expiry, │
        │   state)        │         │   password)      │
        └─────┬───────────┘         └──────┬───────────┘
              │                            │
              │   presigned PUT URLs       │  presigned GET URLs
              │                            │  (with CloudFront signing)
              ▼                            ▼
        ┌──────────────────────────────────────┐
        │  Object Store (S3)                   │  Standard tier for hot,
        │  Bucket layout:                      │  IA after 90 days,
        │   /raw/<file_id>/<version>           │  Glacier after 365.
        │   /chunks/<upload_id>/<n>            │  Lifecycle policy auto-migrates.
        └──────────────────────────────────────┘
              ▲
              │
              │   metadata reads/writes
              │
        ┌─────┴────────────────┐
        │  Metadata DB         │   Tables: files, file_versions,
        │  (Postgres, sharded  │   shares, share_links, audit.
        │   by owner_id)       │   Sharded by owner_id; cross-shard
        │                      │   share lookups via separate index.
        └──────────────────────┘

        Async pipeline:
                                       (file finalized event)
                                  ┌─────────────────────────┐
                                  │  SQS / Kafka            │
                                  └──────────┬──────────────┘
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │  Virus Scan Worker   │  ClamAV or
                                  │  (Lambda or          │  3rd-party API
                                  │   long-running pod)  │  (VirusTotal).
                                  │                      │  Marks file
                                  │                      │  status =
                                  │                      │  quarantined
                                  │                      │  if positive.
                                  └──────────────────────┘

        CDN for downloads:
            ┌──────────────────┐
            │  CloudFront      │  Signed URLs respect share link
            │  in front of S3  │  permissions. Edge cache hot files.
            └──────────────────┘
```

Why each piece is here:

- **API Gateway + WAF.** Auth, rate limit (no client uploads 10k files in a minute), block obvious malicious requests.
- **Upload Service.** Owns the upload session lifecycle. Mints presigned URLs. Tracks "upload_id → chunks uploaded → finalize." Enforces per-user quota on init.
- **Share Link Resolver.** Standalone service so it can scale independently. Hot path for any shared file. Lightweight: look up token in DB, check expiry/password, return permission grant.
- **Object Store (S3).** Source of truth for bytes. Bucket per environment, key layout that makes lifecycle and dedup easy.
- **Metadata DB.** Source of truth for everything that is not bytes. Sharded by owner because most queries are "show me my files."
- **Async virus scan pipeline.** Decoupled from upload latency. Files are downloadable as soon as finalized; if scan flags them, status flips to `quarantined` and downloads start returning 451.
- **CDN.** Without it, you pay full S3 egress + your origin bandwidth. With it, the second download of a shared file comes from edge cache at a fraction of the cost.

</details>

## Step 5: share links and permissions

Sharing has two flavors and three permission scopes. Walk through the design of each.

**Two flavors:**

1. **Direct invite.** Owner enters another user's email. That user must have an account. The invited user sees the file in their "shared with me" tab.
2. **Share link.** Owner generates a URL. Anyone who has the URL can access. No account required (or required, configurable).

**Three permission scopes:**

- **View.** Preview-only (rendered in browser, no download button).
- **Download.** Can fetch the file.
- **Edit.** Can upload a new version.

What does the data model look like? How do you expire a share link? How do you password-protect one? What stops an attacker from enumerating share tokens?

<details>
<summary><b>Reveal: share link design</b></summary>

**Two tables:**

```sql
-- Direct invites (recipient is a known user).
CREATE TABLE shares (
    share_id      UUID PRIMARY KEY,
    file_id       UUID NOT NULL,
    granted_to    BIGINT NOT NULL,    -- user_id
    granted_by    BIGINT NOT NULL,
    permission    SMALLINT NOT NULL,  -- 1=view, 2=download, 3=edit
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at    TIMESTAMPTZ
);
CREATE INDEX idx_shares_recipient ON shares (granted_to) WHERE revoked_at IS NULL;
CREATE INDEX idx_shares_file ON shares (file_id);

-- Anonymous/semi-anonymous share links.
CREATE TABLE share_links (
    token              VARCHAR(32) PRIMARY KEY,  -- opaque, 192-bit random
    file_id            UUID NOT NULL,
    created_by         BIGINT NOT NULL,
    permission         SMALLINT NOT NULL,
    expires_at         TIMESTAMPTZ,             -- NULL = never
    password_hash      BYTEA,                    -- NULL = no password
    require_account    BOOLEAN DEFAULT FALSE,
    max_redemptions    INT,                      -- NULL = unlimited
    redemptions        INT NOT NULL DEFAULT 0,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at         TIMESTAMPTZ
);
CREATE INDEX idx_links_file ON share_links (file_id);
```

**Token generation:**

```python
token = base62(secrets.token_bytes(24))   # 24 bytes = 192 bits of entropy
```

192 bits is enormous. The birthday collision probability is negligible until you have ~2^96 tokens. Practically: you will never collide. Crucially, 192 bits also means an attacker cannot enumerate; the keyspace is too large to brute-force.

Why opaque random, not a hash of (file_id, salt)? Because hashed tokens are deterministic; if someone learns the salt they can compute all your tokens. Pure-random tokens have no relationship to the file they unlock, so leaking the algorithm leaks nothing.

**Resolution flow (`GET /share/<token>`):**

```
1. Look up token in share_links table.
2. If not found → 404.
3. If revoked_at is not null → 410 Gone.
4. If expires_at < now → 410 Gone.
5. If max_redemptions is set and redemptions >= max → 410 Gone.
6. If password_hash is set, demand password (return 401 with WWW-Authenticate);
   on submission, verify against stored hash.
7. If require_account, redirect to login; on return, log redemption against user.
8. Return: {file_id, file_name, permission, owner_name}.
9. Increment redemptions counter (atomic).
10. Issue a short-lived (15-min) download cookie or signed CloudFront URL.
```

**Why a 15-minute signed URL and not a permanent download endpoint:**

If the link itself is the only credential, then a "view-only" share leaks the file the moment someone right-clicks "save target as." Force every download through a signed URL that the share resolver mints fresh each time and that expires quickly. View-only shares mint URLs scoped to the preview renderer only, not the raw file.

**Expiring links.** The `expires_at` column drives a periodic sweeper that hard-deletes very old rows (90 days past expiry) to keep the table small. Until the sweep, expired rows return 410, which is the correct response.

**Password protection.** Bcrypt or Argon2 the password at create time. At redeem, compare. Use HTTP 401 on first request, then accept the password on a follow-up POST. Don't ever put the password in the URL.

**Rate limit per token.** A single token getting 1000 redemption attempts per minute is brute-force traffic. Limit to ~10 redemption attempts per IP per token per minute; lock the token for 5 minutes if the limit trips.

**Direct invite permissions:** simpler because the recipient is authenticated. The `shares` table is checked at file-access time: "is there a non-revoked row granting this user permission on this file?" Index on `(granted_to)` makes "show me shared-with-me" fast.

**Permission scope semantics:**

- View: download URL serves the file through a preview renderer (PDF viewer, image viewer) with `Content-Disposition: inline`. No "download original" button shown. Determined attackers can still pull bytes; treat view-only as a soft control, not a security boundary.
- Download: signed URL serves the original file with `Content-Disposition: attachment`.
- Edit: same as download, plus the API endpoint to upload a new version is permitted.

**Folder-level shares.** If you support folders, sharing a folder cascades to all files inside. Implementation: `shares.target_id` with `target_type = 'folder'` or `'file'`. At access-check time, traverse the parent chain: "is this file in a folder I have access to?" Cache the traversal aggressively because it is read-heavy.

</details>

## Step 6: storage tiers

A file uploaded today might be downloaded 50 times this week. A file uploaded two years ago is statistically never touched again. Paying S3 Standard rates for both is wasteful. Walk through the tiering strategy.

<details>
<summary><b>Reveal: lifecycle policy and tier choice</b></summary>

**Three tiers:**

| Tier | Use case | Cost / GB / month | Retrieval latency | Retrieval cost |
|------|----------|-------------------|--------------------|----------------|
| **Hot (S3 Standard)** | Files accessed in the last 30 days. New uploads. | $0.023 | <100ms | $0 |
| **Warm (S3 Infrequent Access)** | Files untouched 30-365 days. | $0.0125 | <100ms | $0.01 / GB retrieved |
| **Cold (S3 Glacier Flexible Retrieval)** | Files untouched > 365 days. | $0.0036 | 1-5 min (expedited) or 3-5 hr (standard) | $0.03 / GB retrieved + per-request fee |

**Lifecycle policy (S3 native, no code required):**

```
Rule: tier-down-by-age
  Apply to: bucket prefix /raw/
  Transition to S3 IA: 90 days after upload
  Transition to S3 Glacier: 365 days after upload
  Expiration: never (we keep files until user deletes)
```

But this is too crude. A file uploaded 91 days ago that is being actively downloaded should *not* tier down. Two options:

**Option A: Use S3 Intelligent-Tiering.** S3 monitors access patterns per object and moves them between tiers automatically. Costs $0.0025 per 1000 objects per month for monitoring. Worth it once you have millions of objects with unpredictable access.

**Option B: Custom tiering driven by your own access stats.** You log every download to a `file_access` table or stream. A daily job computes "files not accessed in N days" and explicitly transitions them via S3 SDK. More control, more code.

Pick A unless you have a specific reason to need B (compliance often requires explicit control of where data lives).

**The retrieval-latency UX problem.** A file in Glacier takes 3-5 hours to retrieve via standard retrieval. The user clicks Download and waits forever. Solutions:

- **Show the wait explicitly.** "This file is in cold storage. Restoring will take ~5 minutes. We'll email you when ready." Set the user expectation up front.
- **Use Glacier Instant Retrieval tier instead.** Costs ~3x Glacier Flexible but reads are sub-second. Worth it for files users might re-access.
- **Pre-warm on first share.** When a user shares a Glacier-tier file, kick off restoration immediately so by the time the recipient clicks, the file is hot.

**Don't tier metadata.** The `files` row stays in the hot DB regardless of where the bytes live. The row has a `storage_tier` column so the API knows whether to expect latency.

**Don't tier files smaller than ~128KB.** S3 IA and Glacier charge minimum object sizes (128KB for IA). Tiering a 10KB file actually costs more than leaving it in Standard. Exclude small files from the lifecycle rule.

**Deletion in cold tiers.** A user deletes a file that lives in Glacier. The object is deleted from Glacier with a 90-day minimum storage charge (you pay for 90 days even if you delete on day 2). Factor this into your cost model and into your "trash bin" retention. Soft-delete first (status = `deleted_pending`), hard-delete after the user is past the regret window.

</details>

## Follow-up questions

Try answering each in 2 to 4 sentences before reading the solution.

1. **Resumable upload across days.** A user uploads 3GB of a 5GB file, then closes their laptop. The next morning they reopen the app. What happens? How does the client know which chunks already landed? How long do you keep abandoned uploads around?

2. **Quota enforcement race.** A user has 100MB of quota left. Two clients (their phone and laptop) initiate uploads of 80MB each at the same instant. Both pass the quota check at init. Both upload. Now the user is 60MB over quota. How do you prevent this?

3. **Content-addressed dedup.** Three different users upload the same 200MB software installer. How do you store it once? What does "delete" mean when one user deletes their copy? What about privacy (knowing my hash matches yours implies we have the same file)?

4. **Share link enumeration.** Your tokens are 192 bits, so brute force is out. But a security researcher finds your `created_at` timestamps in the response and notes that links from the same minute share a creation pattern. Is this a real attack? What other side channels leak?

5. **Large file delete causing tombstone backlog.** A user with a 50TB account deletes 10TB in one click. Your metadata DB does 200k row updates and S3 issues 200k delete requests. What problems does this cause? How do you smooth it out?

6. **Virus scan returns positive after the file has been downloaded 500 times.** What is your incident response? Can you tell who downloaded it? Do you notify them? What about the share links pointing at the now-quarantined file?

7. **Edit conflict.** Two users with Edit permission upload a new version of the same file within 10 seconds of each other. Whose version wins? Both? The first? The second? How does the loser find out?

8. **A specific shared file goes viral.** A YouTuber's public share link gets 1 million download requests in 24 hours. The file is a 200MB tutorial video. Your CDN cache hits 99% but the 1% miss rate still saturates one S3 prefix. What do you do?

9. **GDPR delete.** A user requests full deletion. Their account has 12,000 files, including some that other users have copies of (via dedup). They also created share links and were granted shares on other users' files. How do you fully erase them?

10. **Cost attribution for an internal customer.** You sell this to enterprises. One enterprise customer wants a monthly invoice breakdown: storage GB by tier, egress GB, virus-scan-API calls, requests-per-million. How do you attribute every byte and every API call to a tenant?

## Related problems

- **[Video Streaming (006)](../006-video-streaming/question.md)**. Same fundamental shape: bytes in S3, metadata in a DB, CDN in front. Video adds adaptive bitrate transcoding; this problem adds share-link permissions. The storage and CDN layer overlaps heavily.
- **[Distributed Cache (009)](../009-distributed-cache/question.md)**. Hot files need an in-memory layer (or at least CDN edge cache) for popular share links. The eviction and warming patterns there apply directly.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md)**. The "show me my files" dashboard and share-link resolution are textbook read-heavy paths. The cache tiering and read-replica strategies from that problem apply directly.
