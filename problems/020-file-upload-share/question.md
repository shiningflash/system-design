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

The trap is that this looks like a CRUD app on top of S3. It isn't. The interesting parts are how a 4GB upload survives hotel WiFi, how one share link gets revoked without breaking every other link, how the same file uploaded by fifty people sits on disk once, and how a file nobody has touched in two years stops costing you anything. A candidate who jumps to "POST the file to S3, write a row in Postgres" misses about 60% of the depth.

## Step 1: clarify before you design

Spend six minutes here. Eight or so questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

The first one I always ask is the max file size. 5GB was in the brief, but is that a hard cutoff or a soft one? Anything above a single-PUT limit forces chunked or multipart. A 50GB cap is bandwidth suicide if it tunnels through your app servers, so it forces presigned-direct-to-S3.

Second: sync or share-only. Is this Dropbox-the-sync-client, or Google-Drive-the-web-share? Sync means delta-sync, conflict resolution, and local file watchers; that's its own design problem. Share-only is much smaller. In an interview, the answer is almost always share-only.

Then virus scanning. Do we scan uploads at all, and if so do we block on the result or quarantine after the fact? Sync scanning kills the upload UX. Async means a file is downloadable for a few minutes before potentially being quarantined.

Versioning matters next. Does an edit create a new version or overwrite? How many do we keep? Versions multiply storage cost. A reasonable default is ten versions or thirty days, whichever comes first.

Bandwidth limits and quotas. Per-user upload caps, account-wide storage quotas. The check has a sneaky race we'll come back to.

Sharing model. Link share, direct invite, or both? What permission scopes: view-only preview, download, edit? Each scope has security implications and changes the download flow.

File preview. Do we render PDFs and images in the browser, or just offer download? Preview means a thumbnailing pipeline, which is a sub-service in itself.

Compliance and retention. GDPR delete? Customer-managed encryption keys for healthcare or finance? The answer changes the storage layer.

Geographic reach. One region or global? If global, do we replicate the files or only the metadata? File replication is expensive; metadata replication is cheap.

Auth is usually out of scope, but worth confirming.

A strong candidate also names what's out of scope: real-time collaborative editing, full-text search inside documents, third-party integrations. Those are downstream products built on top of the storage primitive, not part of it.

</details>

## Step 2: capacity estimates

Two regimes.

**Small startup:**
- 10,000 users
- 5 uploads per user per week, ~5MB average
- Reads about 10x writes

**Dropbox-scale:**
- 100M active users
- 20 uploads per user per week, ~8MB average (mix of photos, docs, occasional video)
- Reads 10x writes
- Indefinite retention by default

For both, work out uploads per second (sustained and peak), downloads per second, storage growth per year before dedup, egress at peak download, and how much storage goes stale (untouched 90+ days) by year two.

<details>
<summary><b>Reveal: the math</b></summary>

**Small startup, 10k users.**

Uploads come out to 10,000 times 5 per week, so 50,000 a week, around 7,000 a day, which is 0.08 per second sustained and maybe 0.25 at peak. Trivial. Downloads at 10x are still under 3 per second. Storage growth is 50,000 a week at 5MB, so roughly 13TB a year. One S3 bucket, no thought. Egress at peak is 2.5 downloads per second times 5MB, about 100 Mbps. Modest. By year two probably 60-70% of those files are stale (most things get looked at once and forgotten).

A single app server plus S3 plus Postgres handles all of this. The system isn't interesting from a throughput angle. What's interesting is the upload UX for a 5GB file and the share-link permission model.

**Dropbox-scale, 100M users.**

Uploads: 100M times 20 per week is 2B per week, about 286M per day, which is ~3,300 per second sustained and ~10k at peak. Downloads at 10x are ~33k per second sustained, ~100k peak. Storage growth is 286M per day times 8MB, so 2.3 PB per day, around 840 PB per year before dedup. With ~30% dedup savings (people share popular images, identical PDFs) call it ~580 PB per year. Egress at peak is 100k times 8MB, which is 800 GB/s or ~6.4 Tbps. That's CDN territory; you can't serve it from a single region without one. By year two, ~70-80% of stored bytes are untouched 90+ days. On 1.2EB stored, that's ~900PB cold. Lifecycle to S3 IA plus Glacier saves ~70% of the cold-tier cost.

The math tells you a few things.

The system is read-heavy by request count but write-heavy by bytes. Uploads dominate ingress; downloads (with CDN) dominate egress.

Storage cost is the headline expense. 840 PB per year at $0.023/GB/month for S3 Standard is roughly $230M a year for raw storage alone. Lifecycle policies and dedup aren't optimizations, they're survival.

Per-server bandwidth is what dies first if you tunnel uploads. A 10Gbps NIC at full pelt handles ~1.25 GB/s, which is ~150 concurrent 8MB uploads per second. You'll run out of NICs long before you run out of CPU. That's why presigned URLs to S3 are the answer.

The metadata DB is small. 100M users times 1000 files each is 100B rows at ~500 bytes, so ~50TB. Sharded Postgres or a wide-column store handles it. The bytes live in S3; the metadata is what you actually run a database for.

</details>

## Step 3: upload protocols

The biggest single design decision is how the bytes get from the client to your storage. Three serious options. Take ten minutes and write down the pros and cons of each before peeking.

- **A. Direct upload (single HTTP POST).** Client streams the whole file in one request to your API server, which forwards to S3.
- **B. Chunked resumable upload (TUS protocol, or S3 multipart).** Client splits the file into 5-100MB chunks and uploads them with retry-per-chunk semantics.
- **C. Presigned S3 URL.** Your server hands the client a short-lived signed URL; the client uploads directly to S3, skipping your app servers entirely.

<details>
<summary><b>Reveal: comparison</b></summary>

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Direct (single POST)** | `POST /upload` with multipart body. Server buffers or streams to S3. | Simple. One request, one response. Works in any HTTP client. | Whole upload restarts on any network blip. 4GB upload on hotel WiFi never finishes. Saturates your app server bandwidth. Hard to show real progress. |
| **Chunked resumable (TUS, S3 multipart)** | Initiate session, upload chunk 1 of N, then chunk 2, and so on, then finalize. Each chunk is its own HTTP request. Failed chunks retry independently. | Survives flaky networks. Real progress bars. Parallel chunks speed large files. Server can pause and resume. | More complex protocol. Server tracks per-session state (which chunks landed). Garbage collection of abandoned uploads. |
| **Presigned S3 URL** | `POST /upload/init` returns a presigned URL valid for ~1h. Client `PUT`s directly to S3. On success, client calls `POST /upload/finalize` so server knows. | Zero bandwidth through your servers. Cheapest at scale. S3 handles the heavy lifting. Combines with multipart for huge files. | Client must speak S3 directly. Quota enforcement is harder (server didn't see the bytes). Virus scan is post-hoc. Signed URL leaks mean anyone can upload to that key. |

My recommendation is hybrid: chunked plus presigned, depending on file size.

Files under 5MB: single POST through the app server is fine. Simpler client.

Files 5MB to 5GB: presigned S3 multipart. Client splits into 8MB chunks, gets a presigned URL per chunk (or one initiation token), uploads chunks in parallel directly to S3, calls finalize when done.

Files over 5GB: same approach with 64MB chunks and explicit server-side caps.

The hybrid keeps the API simple for the common case and offloads the painful case to S3 entirely.

Why TUS at all? TUS (`tus.io`) is an open protocol for resumable uploads with a Go/Node server library that the client speaks to. Useful if your storage isn't S3 (some shops use MinIO, NetApp, or their own object store) and you want a standard client protocol. If you're on AWS, S3 multipart is more native; TUS is for the "S3 alternative" case.

</details>

## Step 4: incomplete architecture diagram

Sketch the high-level architecture. Fill in the five `[ ? ]` boxes. Each is one of: upload service, object store, metadata DB, share link resolver, virus scanner.

```
                Client (browser, mobile, desktop sync agent)
                              |
                              v
                    +------------------+
                    |   API Gateway    |  (auth, rate limit, request shaping)
                    +--------+---------+
                             |
        upload init,         |       download,
        finalize, list       |       share link redeem
                             |
                +------------+------------+
                |                         |
                v                         v
        +-------------+            +-------------+
        |  [ ? ]      |            |  [ ? ]      |   (resolves /share/<token>
        |             |            |             |    to a file + permission)
        +-----+-------+            +------+------+
              |                            |
              |   presigned URLs           |
              |                            |
              v                            v
        +--------------------------------------+
        |  [ ? ]                               |  (the bytes live here;
        |                                      |   cheap, durable, huge)
        +--------------------------------------+
              ^
              |
              |   reads/writes
              |
        +-----+--------+
        |  [ ? ]       |   (file_id, owner, name, size,
        |              |    hash, status, version)
        +--------------+

        Async pipeline:
            +------------------+
            |  [ ? ]           |  (consumes "file finalized" events,
            |                  |   scans, marks quarantined if positive)
            +------------------+
```

<details>
<summary><b>Reveal: completed architecture</b></summary>

```
                Client (browser, mobile, desktop sync agent)
                              |
                              v
                    +------------------+
                    |   API Gateway    |  auth, rate limit, request shaping,
                    |   + WAF          |  blocks obvious abuse
                    +--------+---------+
                             |
        upload init,         |       download,
        finalize, list       |       share link redeem
                             |
                +------------+------------+
                |                         |
                v                         v
        +-----------------+         +------------------+
        |  Upload Service |         |  Share Link      |
        |  mints          |         |  Resolver        |
        |  presigned      |         |  token ->        |
        |  URLs, tracks   |         |  file + perms,   |
        |  session state  |         |  checks expiry,  |
        |                 |         |  password        |
        +-----+-----------+         +------+-----------+
              |                            |
              |   presigned PUT URLs       |  presigned GET URLs
              |                            |  (CloudFront-signed)
              v                            v
        +--------------------------------------+
        |  Object Store (S3)                   |  Standard for hot,
        |  Bucket layout:                      |  IA after 90 days,
        |   /raw/<file_id>/<version>           |  Glacier after 365.
        |   /chunks/<upload_id>/<n>            |  Lifecycle auto-migrates.
        +--------------------------------------+
              ^
              |
              |   metadata reads/writes
              |
        +-----+----------------+
        |  Metadata DB         |   files, file_versions,
        |  (Postgres, sharded  |   shares, share_links, audit.
        |  by owner_id)        |   Cross-shard share lookups
        |                      |   via separate index.
        +----------------------+

        Async pipeline:

                                       (file finalized event)
                                  +-------------------------+
                                  |  SQS / Kafka            |
                                  +----------+--------------+
                                             |
                                             v
                                  +----------------------+
                                  |  Virus Scan Worker   |  ClamAV or
                                  |  (Lambda or          |  3rd-party API.
                                  |  long-running pod)   |  Flips status
                                  |                      |  to quarantined
                                  |                      |  if positive.
                                  +----------------------+

        CDN for downloads:
            +------------------+
            |  CloudFront      |  Signed URLs respect share link
            |  in front of S3  |  permissions. Edge cache hot files.
            +------------------+
```

A few notes on why each piece is there.

API Gateway plus WAF is the first defense. Auth, rate limit (no client uploads 10k files in a minute), block obvious malicious requests.

Upload Service owns the upload session lifecycle. Mints presigned URLs. Tracks "upload_id, chunks uploaded, finalize." Enforces per-user quota at init. Doesn't touch bytes.

Share Link Resolver is standalone so it scales independently. Hot path for any shared file. Lightweight: look up token, check expiry and password, return permission grant.

Object Store is the source of truth for bytes. One bucket per environment, key layout that makes lifecycle and dedup easy.

Metadata DB is the source of truth for everything that isn't bytes. Sharded by owner because most queries are "show me my files."

Async virus scan pipeline is decoupled from upload latency. Files are downloadable as soon as finalized; if the scan flags them, status flips to quarantined and downloads start returning 451.

CDN is what keeps egress affordable. Without it you pay full S3 egress plus your origin bandwidth. With it, the second download of a shared file comes from edge cache at a fraction of the cost.

</details>

## Step 5: share links and permissions

Sharing has two flavors and three permission scopes.

The two flavors are direct invite (owner enters someone's email, that user must have an account, file shows up in their "shared with me" tab) and share link (owner generates a URL, anyone with the URL gets in, account may or may not be required).

The three permission scopes are view (preview-only, no download button), download (can fetch the file), and edit (can upload a new version).

What does the data model look like? How do you expire a link? How do you password-protect one? What stops an attacker from enumerating share tokens?

<details>
<summary><b>Reveal: share link design</b></summary>

Two tables.

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

-- Anonymous or semi-anonymous share links.
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

Token generation is straightforward:

```python
token = base62(secrets.token_bytes(24))   # 24 bytes = 192 bits of entropy
```

192 bits is enormous. Birthday collision probability is negligible until you have about 2^96 tokens. You'll never collide. Crucially, that keyspace is too large to brute-force, so attackers can't enumerate.

Why opaque random instead of a hash of (file_id, salt)? Because hashed tokens are deterministic; if someone learns the salt they can compute all your tokens. Pure-random tokens have no relationship to the file they unlock, so leaking the algorithm leaks nothing.

The resolution flow for `GET /share/<token>`:

1. Look up token in `share_links`.
2. Not found, 404.
3. `revoked_at` set, 410 Gone.
4. `expires_at < now`, 410.
5. `max_redemptions` reached, 410.
6. Password required, return 401 with WWW-Authenticate; on submission, verify against the stored hash.
7. Account required, redirect to login; on return, log redemption against the user.
8. Return file_id, file_name, permission, owner_name.
9. Increment redemptions counter atomically.
10. Issue a short-lived (15-minute) download cookie or signed CloudFront URL.

The 15-minute signed URL matters. If the link itself is the only credential, then a view-only share leaks the file the moment someone right-clicks "save target as." Force every download through a signed URL that the share resolver mints fresh each time, with short expiry. View-only shares mint URLs scoped to the preview renderer only, not the raw file.

`expires_at` drives a periodic sweeper that hard-deletes very old rows (90 days past expiry) to keep the table small. Until the sweep, expired rows return 410, which is the correct response.

Password protection: bcrypt or Argon2 at create time. Verify at redeem. HTTP 401 on first request, then accept the password on a follow-up POST. Never put the password in the URL.

Rate limit per token. A single token getting 1000 redemption attempts a minute is brute-force traffic. Cap at ~10 attempts per IP per token per minute; lock the token for 5 minutes if it trips.

Direct invite permissions are simpler because the recipient is authenticated. The `shares` table is checked at file-access time: "is there a non-revoked row granting this user permission on this file?" Index on `granted_to` makes "show me shared-with-me" fast.

Permission scope semantics. View serves through a preview renderer (PDF viewer, image viewer) with `Content-Disposition: inline` and no download button shown. Determined attackers can still pull bytes; treat view-only as a soft control, not a security boundary. Download serves the original file with `Content-Disposition: attachment`. Edit is download plus the API endpoint to upload a new version.

Folder-level shares cascade to all files inside. Implementation: `shares.target_id` with `target_type = 'folder'` or `'file'`. At access-check time, traverse the parent chain: "is this file in a folder I have access to?" Cache the traversal aggressively because it's read-heavy.

</details>

## Step 6: storage tiers

A file uploaded today might be downloaded 50 times this week. A file uploaded two years ago is statistically never touched again. Paying S3 Standard rates for both is wasteful. What's the tiering strategy?

<details>
<summary><b>Reveal: lifecycle policy and tier choice</b></summary>

Three tiers.

| Tier | Use case | Cost / GB / month | Retrieval latency | Retrieval cost |
|------|----------|-------------------|--------------------|----------------|
| **Hot (S3 Standard)** | Files accessed in the last 30 days. New uploads. | $0.023 | <100ms | $0 |
| **Warm (S3 Infrequent Access)** | Files untouched 30-365 days. | $0.0125 | <100ms | $0.01 / GB retrieved |
| **Cold (S3 Glacier Flexible Retrieval)** | Files untouched > 365 days. | $0.0036 | 1-5 min (expedited) or 3-5 hr (standard) | $0.03 / GB retrieved + per-request fee |

An S3-native lifecycle policy looks like this:

```
Rule: tier-down-by-age
  Apply to: bucket prefix /raw/
  Transition to S3 IA: 90 days after upload
  Transition to S3 Glacier: 365 days after upload
  Expiration: never (we keep files until user deletes)
```

But that's crude. A file uploaded 91 days ago that's being actively downloaded shouldn't tier down. Two options to handle access patterns.

Option A is S3 Intelligent-Tiering. S3 monitors access per object and moves them between tiers automatically. Costs $0.0025 per 1000 objects per month for monitoring. Worth it once you have millions of objects with unpredictable access.

Option B is custom tiering driven by your own access stats. You log every download to a `file_access` table or stream. A daily job computes "files not accessed in N days" and explicitly transitions them via S3 SDK. More control, more code.

Pick A unless you have a specific reason to need B (compliance often requires explicit control of where data lives).

The retrieval-latency UX problem is real. A file in Glacier takes 3-5 hours to retrieve via standard retrieval. The user clicks Download and waits forever. A few ways out:

Show the wait explicitly. "This file is in cold storage. Restoring will take ~5 minutes. We'll email you when ready." Set the expectation up front.

Use Glacier Instant Retrieval instead. Costs ~3x Glacier Flexible but reads are sub-second. Worth it for files users might re-access.

Pre-warm on first share. When a user shares a Glacier-tier file, kick off restoration immediately so by the time the recipient clicks, the file is hot.

Don't tier metadata. The `files` row stays in the hot DB regardless of where the bytes live. The row has a `storage_tier` column so the API knows whether to expect latency.

Don't tier files smaller than ~128KB. S3 IA and Glacier charge minimum object sizes (128KB for IA). Tiering a 10KB file actually costs more than leaving it in Standard. Exclude small files from the lifecycle rule.

Deletion in cold tiers has its own gotcha. A user deletes a file in Glacier, the object goes away but you still pay the 90-day minimum storage charge. Factor this into your cost model and into your trash-bin retention. Soft-delete first (`status = deleted_pending`), hard-delete after the user is past the regret window.

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
