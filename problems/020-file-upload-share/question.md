---
id: 20
title: Design a File Upload & Share Service (Dropbox-lite)
category: Storage
topics: [chunked upload, share links, permissions, storage tiers, virus scan]
difficulty: Easy
solution: solution.md
---

## What we are building

A file upload and share service lets users store large files in the cloud and hand out links to other people. Think of it as Dropbox or Google Drive in miniature.

Concrete example: Alice opens the web app and uploads a 4 GB video she shot on her phone. The connection drops twice over hotel WiFi, but the upload picks up from where it left off. When it finishes, the service runs a background virus scan and marks the file ready. Alice clicks Share and gets a link. She sends it to Bob. Bob opens the link in his browser and downloads the file. Meanwhile, a hundred other users have already uploaded the exact same video file (the same software installer, perhaps). The service stores only one copy of the bytes and lets all hundred users point at it.

The problems hiding in that story:

1. **Resumable upload.** A 4 GB upload over a flaky connection cannot be one big HTTP request. If it dies at 80%, the user cannot restart from scratch.
2. **Chunking and parallelism.** Large files need to split into chunks that upload independently and in parallel.
3. **Content deduplication.** Fifty users uploading the same installer should not store it fifty times. Hash the bytes. Share one copy.
4. **Share-link permissions.** Alice can revoke one link without breaking 999 others. Bob gets view access, not download access.
5. **Virus and abuse scanning.** An infected file uploaded to a public product is a security incident. The scan should not block the upload response.

---

## The lifecycle of one file

Before drawing boxes, picture the states a file moves through.

```mermaid
stateDiagram-v2
    direction LR
    [*] --> Uploading: client starts upload
    Uploading --> Scanning: finalize received
    Scanning --> Ready: scan clean
    Scanning --> Quarantined: scan flagged
    Ready --> Shared: Alice creates share link
    Shared --> Downloaded: Bob opens link
    Ready --> Deleted: Alice deletes
    Quarantined --> Deleted: admin removes
    Deleted --> [*]
```

Everything we add later (chunked upload, dedup, cold tier, revocation) is a complication on top of this state machine.

> **Take this with you.** A file service is a state machine around bytes. The state lives in your database. The bytes live in object storage. They are two separate things.

---

## How big this gets

Two scales shape very different designs. Do the math before drawing anything.

| Input | 10k users | 100M users |
|-------|-----------|------------|
| Uploads per second (sustained) | ~0.08 | ~3,300 |
| Downloads per second (sustained) | ~0.8 | ~33,000 |
| Storage per year (raw) | ~13 TB | ~840 PB |
| Egress at peak | ~100 Mbps | ~6.4 Tbps |

<details markdown="1">
<summary><b>Show: how the numbers come out</b></summary>

**10k users:**
- 10,000 users, 5 uploads per week, 5 MB average.
- 50,000/week = ~7,000/day = **~0.08/sec sustained, ~0.25/sec peak.** Tiny.
- Downloads at 10x: ~0.8/sec.
- Storage: 7,000/day x 5 MB = ~13 TB/year.

One server. One Postgres. One S3 bucket. The throughput is not the challenge. The interesting part is the upload protocol for a 5 GB file and the share-link permission model.

**100M users:**
- 100M users, 20 uploads per week, 8 MB average.
- 2B/week = **~3,300/sec sustained, ~10,000/sec peak.**
- Downloads at 10x: ~33k/sec sustained, ~100k/sec peak.
- Storage: 286M/day x 8 MB = ~2.3 PB/day = ~840 PB/year raw. With ~30% dedup savings: ~**580 PB/year.**
- Egress at peak: 100k x 8 MB = 800 GB/s = **~6.4 Tbps.** CDN is not optional.

**The two numbers that dominate decisions:**

Storage cost is the headline expense. At 580 PB, $0.023/GB/month for S3 Standard is ~$160M/year. Lifecycle tiers and dedup are survival, not optimization.

Bandwidth through your servers is the scaling killer. One 10 Gbps NIC handles ~1.25 GB/s, which is only ~150 concurrent 8 MB uploads. At 10,000 concurrent uploads you need 70 servers just to forward bytes. Presigned upload URLs let the client go direct to S3. Your servers never touch the bytes.

</details>

> **Take this with you.** Reads beat writes by request count, but writes beat reads by bytes. CDN absorbs downloads. Presigned URLs remove your servers from the upload byte path. Storage lifecycle tiers are the cost model, not a nice-to-have.

---

## The smallest version that works

For 10 users, three boxes are enough.

```mermaid
flowchart LR
    A([Alice]):::user --> API[/"Upload Service<br/>(init + finalize)"/]:::app
    API --> S3[("S3<br/>bytes")]:::db
    API --> DB[("Postgres<br/>files table")]:::db
    B([Bob]):::user --> SLR[/"Share Link Resolver"/]:::app
    SLR --> DB
    SLR -.signed URL.-> S3

    classDef user  fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
```

Two phases: upload, then share-link redeem.

```mermaid
sequenceDiagram
    autonumber
    participant Alice
    participant App
    participant S3
    participant DB
    participant Bob

    Alice->>App: POST /uploads/init
    App->>S3: create presigned PUT URL
    App->>DB: INSERT upload_session
    App-->>Alice: presigned URL

    Alice->>S3: PUT file bytes
    S3-->>Alice: ETag

    Alice->>App: POST /uploads/finalize (ETag)
    App->>DB: INSERT files row (status=ready)
    App-->>Alice: file_id

    Alice->>App: POST /files/{id}/share_links
    App->>DB: INSERT share_links (token=random)
    App-->>Alice: share URL

    Bob->>App: GET /share/{token}
    App->>DB: look up token
    App->>S3: sign download URL (15 min)
    App-->>Bob: 302 redirect
    Bob->>S3: GET file bytes
```

<details markdown="1">
<summary><b>Show: the two core tables</b></summary>

```sql
CREATE TABLE files (
    file_id      UUID PRIMARY KEY,
    owner_id     BIGINT NOT NULL,
    name         TEXT NOT NULL,
    size_bytes   BIGINT NOT NULL,
    content_hash BYTEA NOT NULL,
    status       SMALLINT NOT NULL DEFAULT 1,  -- 1=uploading, 2=ready, 3=quarantined, 4=deleted
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE share_links (
    token           VARCHAR(32) PRIMARY KEY,  -- 192-bit random
    file_id         UUID NOT NULL,
    created_by      BIGINT NOT NULL,
    permission      SMALLINT NOT NULL,        -- 1=view, 2=download, 3=edit
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ
);
```

</details>

---

## Decision 1: how do we make a large upload survive a bad connection?

A 4 GB upload over hotel WiFi is not a single HTTP request. Any dropped packet restarts the whole thing. The protocol has to be chunked.

Three options:

```mermaid
flowchart TB
    subgraph A["Option A: single POST to app server"]
        A1["Client streams<br/>full file to your server"] --> A2["Server writes to S3"]
        A3["Problem: one dropped TCP packet<br/>restarts the full 4 GB"]:::bad
        A4["Problem: 10k concurrent uploads<br/>need 70 servers just for forwarding bytes"]:::bad
    end
    subgraph B["Option B: chunked to app server"]
        B1["Client splits into<br/>8 MB chunks"] --> B2["Chunks POST to your server"]
        B3["Problem: your servers still<br/>touch every byte. Bandwidth cost."]:::bad
    end
    subgraph C["Option C: presigned URL per chunk, direct to S3"]
        C1["Server mints one<br/>presigned URL per chunk"] --> C2["Client uploads chunks<br/>in parallel, direct to S3"]
        C3["Retry: only the failed<br/>chunk, not the whole file"]:::ok
        C4["Servers never touch bytes.<br/>Cost scales with files, not servers."]:::ok
    end

    classDef bad fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef ok  fill:#dcfce7,stroke:#15803d,color:#14532d
```

The answer is C, combined with S3 multipart upload. Each chunk gets its own presigned URL, uploading directly and in parallel. A failed chunk retries on its own. When all chunks land, the client sends a finalize call with the list of ETags and S3 stitches them together.

```mermaid
flowchart LR
    subgraph Phase1["Init"]
        C1([Alice]) --> A1["POST /uploads/init<br/>(size, hash)"]:::app
        A1 --> B1["check dedup<br/>reserve quota<br/>create S3 multipart"]:::app
        B1 --> C2["presigned URL<br/>per 8 MB chunk"]:::app
    end

    subgraph Phase2["Upload chunks in parallel"]
        C3([Alice]) --> S3A[("S3 chunk 1")]:::db
        C3 --> S3B[("S3 chunk 2")]:::db
        C3 --> S3C[("S3 chunk N")]:::db
    end

    subgraph Phase3["Finalize"]
        C4([Alice]) --> A2["POST /uploads/finalize<br/>(ETags)"]:::app
        A2 --> S3D["S3 complete multipart"]:::db
        A2 --> DB2[("Postgres files row")]:::db
    end

    classDef user  fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
```

A 4 GB upload at 8 MB per chunk uses ~500 chunks. If chunk 312 fails, only chunk 312 retries.

<details markdown="1">
<summary><b>Show: the chunked upload sequence in full</b></summary>

```mermaid
sequenceDiagram
    autonumber
    participant Alice
    participant US as Upload Service
    participant S3
    participant DB as Postgres

    Alice->>US: POST /uploads/init {size, content_hash}
    US->>DB: check blob by content_hash
    DB-->>US: no match

    rect rgb(241, 245, 249)
        Note over US,DB: reserve quota atomically
        US->>DB: UPDATE users SET reserved_bytes += size WHERE has_room
        DB-->>US: ok
    end

    US->>S3: create_multipart_upload
    S3-->>US: s3_upload_id
    US->>US: presign one URL per chunk (8 MB each)
    US->>DB: INSERT upload_session
    US-->>Alice: {upload_id, chunk_size: 8MB, presigned_urls[]}

    par chunk 1
        Alice->>S3: PUT chunk 1 (presigned URL)
        S3-->>Alice: ETag 1
    and chunk 2
        Alice->>S3: PUT chunk 2 (presigned URL)
        S3-->>Alice: ETag 2
    and chunk N
        Alice->>S3: PUT chunk N (presigned URL)
        S3-->>Alice: ETag N
    end

    Alice->>US: POST /uploads/finalize {parts: [ETags]}
    US->>S3: complete_multipart_upload
    S3-->>US: final S3 key

    rect rgb(241, 245, 249)
        Note over US,DB: one transaction
        US->>DB: upsert blob (refcount++)
        US->>DB: insert files row (status=uploading)
        US->>DB: insert audit
        US->>DB: COMMIT
    end

    US-->>Alice: 200 {file_id, status: "scanning"}
```

</details>

> **Take this with you.** Chunked upload with presigned URLs solves two problems at once: the client retries individual chunks (resilience), and the bytes never pass through your servers (cost).

---

## Decision 2: how do we avoid storing the same file 50 times?

Fifty users upload the same 200 MB software installer. Storing 10 GB for what is effectively one file wastes storage and money.

The fix: content-addressed dedup. Hash the bytes (SHA-256). Two files with the same bytes produce the same hash. Store the bytes once. Let many user-owned file records point at the same blob.

```mermaid
flowchart TB
    subgraph Blobs["blobs table (one per unique byte sequence)"]
        B1["hash: abc...<br/>size: 200 MB<br/>refcount: 3"]:::db
        B2["hash: def...<br/>size: 4 GB<br/>refcount: 1"]:::db
    end

    subgraph Files["files table (one per user pointer)"]
        F1["installer-2024.exe (Alice)"]:::app
        F2["setup.exe (Bob)"]:::app
        F3["v1.0-install.exe (Carol)"]:::app
        F4["vacation-video.mp4 (Alice)"]:::app
    end

    F1 --> B1
    F2 --> B1
    F3 --> B1
    F4 --> B2

    classDef db  fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef app fill:#dcfce7,stroke:#15803d,color:#14532d
```

The dedup check happens at upload init. The client sends the SHA-256 hash before uploading. If a blob with that hash already exists, the server skips the upload entirely and returns the existing file ID. The client never sends a byte.

When Alice deletes her copy: decrement refcount from 3 to 2. Bob and Carol still point at the blob. Blob stays. When refcount hits zero, schedule the bytes for deletion after a 24-hour grace period.

Consumer file-sharing services see ~30% storage savings from dedup. On 580 PB that is 170 PB saved, which at $0.023/GB/month works out to roughly $50M/year.

| Operation | What happens |
|-----------|--------------|
| User uploads new file | Check hash at init. No match: proceed with S3 multipart. |
| User uploads duplicate | Match found at init: return existing file_id. No S3 call. Dedup hit rate ~30%. |
| User deletes their copy | Decrement refcount. If 0: schedule S3 delete after 24h grace. |
| Two users delete at once | `UPDATE blobs SET refcount = refcount - 1 ... RETURNING refcount`. Atomic. |

> **Take this with you.** Blob is the bytes. File is the user-named pointer. Keep them in separate tables. The rest follows from refcount.

---

## Decision 3: how do share-link permissions work?

Alice has 1,000 share links on the same file. She wants to revoke one of them. The other 999 should keep working. And she wants one link to be view-only while another is download-only.

The wrong design: make the file ID the share credential. If the URL is `/files/abc123`, every link gives the same access and you cannot revoke one without revoking all.

The right design: one row per share link, with an opaque high-entropy token.

```mermaid
sequenceDiagram
    autonumber
    participant Bob
    participant SLR as Share Link Resolver
    participant DB as Postgres
    participant CF as CloudFront

    Bob->>SLR: GET /share/<token>
    SLR->>DB: look up token

    alt token not found or revoked
        SLR-->>Bob: 404 Not Found
    else expired
        SLR-->>Bob: 410 Gone
    else password required, none given
        SLR-->>Bob: 401 Unauthorized
    else all checks pass
        SLR->>SLR: sign CloudFront URL (15 min TTL)
        SLR->>DB: increment redemptions counter
        SLR-->>Bob: 302 redirect to signed URL
        Bob->>CF: GET signed URL
        CF-->>Bob: file bytes (~30ms CDN hit)
    end
```

The signed CloudFront URL expires in 15 minutes. A view-only link gets a URL scoped to that permission. A download link gets a wider URL. The permission is enforced at link creation, not at download time.

Revoke one link: `UPDATE share_links SET revoked_at = NOW() WHERE token = ?`. One row update. The other 999 rows are untouched.

Token generation: 192 bits of randomness. No relationship to the file ID, owner, or creation time. Brute force is out.

> **Take this with you.** One row per share link. Revoke by setting `revoked_at` on that row. Never make the file ID the download credential.

---

## Decision 4: how does the virus scan work without blocking the upload?

Scanning a 4 GB file takes minutes. Blocking the upload response until the scan finishes is a bad user experience.

Two options:

```mermaid
flowchart LR
    subgraph Sync["Synchronous scan"]
        SY1["Finalize received"] --> SY2["Call scan API<br/>(2-4 min for 4 GB)"]:::bad
        SY2 --> SY3["Return 200 to Alice<br/>(4 minutes later)"]
        SY4["Problem: bad UX<br/>Alice thinks upload died"]:::bad
    end

    subgraph Async["Asynchronous scan"]
        AS1["Finalize received"] --> AS2["Return 200 to Alice<br/>(status=scanning, ~500ms)"]:::ok
        AS2 --> AS3["Emit file.finalized event"]
        AS3 --> AS4["Scan worker picks up event<br/>scans in background"]
        AS4 --> AS5["Flip status to ready or quarantined"]:::ok
        AS6["Trade-off: file visible as 'scanning'<br/>for 1-3 min before flagging"]
    end

    classDef bad fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef ok  fill:#dcfce7,stroke:#15803d,color:#14532d
```

The async approach wins on UX. The trade-off: a malicious file is live for 1 to 3 minutes before the scan completes. Downloads of unscanned files return `425 Too Early` so Bob cannot download while scanning is in progress.

The scan pipeline:

```mermaid
flowchart LR
    DB[("Postgres<br/>status=uploading")]:::db -->|CDC outbox| Q{{"SQS<br/>file.finalized"}}:::queue
    Q --> VS["Virus Scan Worker<br/>(ClamAV + hash blocklist)"]:::app
    VS -->|clean| DB2[("Postgres<br/>status=ready")]:::db
    VS -->|infected| DB3[("Postgres<br/>status=quarantined")]:::db
    DB3 --> RL["revoke all share links<br/>for this file"]:::app
    DB3 --> CF["CDN invalidation"]:::edge

    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef edge  fill:#e2e8f0,stroke:#475569,color:#1e293b
```

If the scan worker dies and the message goes back to the queue, another worker picks it up. The scan is idempotent. If the scan queue falls behind, uploads still succeed. Scans just lag.

> **Take this with you.** Anything reactive to an upload goes after the queue, not before the 200 response. If the worker dies at 3 a.m., uploads still work.

---

## Decision 5: how do we control storage cost as the system grows?

A file uploaded today might get downloaded 50 times this week. A file from two years ago is probably never touched again. Paying the same rate for both wastes money.

S3 has three tiers:

| Tier | Cost/GB/month | Retrieval time | Retrieval cost |
|------|---------------|----------------|----------------|
| S3 Standard (hot) | $0.023 | < 100 ms | free |
| S3 Infrequent Access (warm) | $0.0125 | < 100 ms | $0.01/GB |
| Glacier (cold) | $0.0036 | 1-5 min fast, 3-5 hr standard | $0.03/GB |

```mermaid
flowchart TD
    A([New file uploaded]) --> B["S3 Standard"]:::app
    B --> C{Accessed in last 90 days?}
    C -->|Yes| B
    C -->|No| D["S3 Infrequent Access"]:::cache
    D --> E{Accessed in last 365 days?}
    E -->|Yes| D
    E -->|No| F["S3 Glacier"]:::db
    F --> G{User downloads?}
    G -->|No| F
    G -->|Yes| H["Restore: 1-5 min expedited<br/>or 3-5 hr standard"]:::queue
    H --> B

    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

On 580 PB: Glacier is ~$25M/year. Standard is ~$160M/year. The lifecycle policy is the difference between a profitable product and a burning one.

Three gotchas to mention:

- **Glacier retrieval surprises users.** Show a "Restoring, we will email you when ready" state. Never silently make a user wait 5 hours.
- **Do not tier small files.** S3 IA charges a 128 KB minimum object size. Tiering a 10 KB file costs more than leaving it hot.
- **Cold-tier deletes carry penalties.** A file deleted from Glacier still incurs the 90-day minimum storage charge. Soft-delete first, hard-delete later.

> **Take this with you.** S3 lifecycle rules are three lines of config. At PB scale they save tens of millions of dollars per year.

---

## The full architecture

Putting all five decisions together:

```mermaid
flowchart TB
    subgraph Edge["Client edge"]
        C([Web / Mobile / Desktop]):::user
        GW["API Gateway<br/>(auth · rate limit · WAF)"]:::edge
    end

    subgraph WritePath["Upload path"]
        US["Upload Service<br/>(presigned URLs · dedup · quota)"]:::app
    end

    subgraph ReadPath["Download / share path"]
        FA["File + Share API"]:::app
        PR["Permission Resolver<br/>(cached 30s)"]:::app
        CF["CloudFront<br/>(CDN · 60s TTL)"]:::edge
    end

    DB[("Postgres<br/>files · blobs · shares<br/>share_links · audit")]:::db
    S3[("S3<br/>/raw/<hash><br/>90d→IA, 365d→Glacier")]:::db
    Cache[("Redis<br/>perm cache")]:::cache

    Q{{"SQS / Kafka<br/>file.finalized · file.deleted"}}:::queue

    subgraph Async["Async workers"]
        VS["Virus Scan Worker<br/>(ClamAV)"]:::app
        LM["Lifecycle Manager<br/>(refcount GC · orphan cleanup)"]:::app
    end

    C --> GW
    GW -->|init / finalize| US
    GW -->|download / share| FA
    US --> DB
    US --> S3
    FA --> PR
    PR --> Cache
    PR -.miss.-> DB
    FA --> CF
    CF -.miss.-> S3
    DB -->|CDC outbox| Q
    Q --> VS
    Q --> LM
    VS --> DB

    classDef user  fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef edge  fill:#e2e8f0,stroke:#475569,color:#1e293b
    classDef app   fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db    fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
```

Each component, in one sentence:

| Component | Purpose |
|-----------|---------|
| API Gateway | Auth, rate limiting, WAF. Entry point for all traffic. |
| Upload Service | Mints presigned URLs, checks dedup and quota. Never touches bytes. |
| File + Share API | Generates signed CloudFront URLs. Resolves share tokens. |
| Permission Resolver | "Can user X do Y on file Z?" Combines owner, invite, and folder checks. Cached 30s. |
| CloudFront | Edge cache. Makes the first 1% of downloads pay for the other 99%. |
| Postgres | Source of truth for metadata. Sharded by owner_id at scale. |
| S3 | Source of truth for bytes. Keyed by content hash. Lifecycle rules tier cold objects. |
| Redis | Permission cache. Most access checks never reach the DB. |
| SQS / Kafka | Decouples virus scan and GC from the write path. |
| Virus Scan Worker | Runs ClamAV async. Flips status on the file row. |
| Lifecycle Manager | Decrements refcounts on delete. Aborts abandoned uploads. |

Notice what is not on the synchronous path: virus scanning, analytics, and lifecycle GC. If any of those workers die at 3 a.m., uploads and downloads keep working.

---

## Walk: one upload, end to end

Alice uploads a 1.5 GB video (~190 chunks at 8 MB each).

```mermaid
sequenceDiagram
    autonumber
    participant Alice
    participant GW as API Gateway
    participant US as Upload Service
    participant DB as Postgres
    participant S3
    participant Q as SQS
    participant VS as Virus Scan

    Alice->>GW: POST /uploads/init {size: 1.5GB, content_hash}
    GW->>US: forward (auth ok)
    US->>DB: check blob by content_hash
    DB-->>US: no match

    rect rgb(241, 245, 249)
        Note over US,DB: reserve quota atomically
        US->>DB: UPDATE users SET reserved_bytes += 1.5GB WHERE has_room
        DB-->>US: ok
    end

    US->>S3: create_multipart_upload
    S3-->>US: s3_upload_id
    US->>DB: INSERT upload_session
    US-->>GW: 201 {upload_id, chunk_size: 8MB, presigned_urls[190]}
    GW-->>Alice: 201 (~200ms)

    Note over Alice,S3: Alice uploads ~190 chunks directly to S3 in parallel
    Alice->>S3: PUT chunks 1..190 (presigned)
    S3-->>Alice: ETags

    Alice->>GW: POST /uploads/finalize {parts: [ETags]}
    GW->>US: forward
    US->>S3: complete_multipart_upload
    S3-->>US: final S3 key

    rect rgb(241, 245, 249)
        Note over US,DB: one transaction
        US->>DB: upsert blob (refcount=1)
        US->>DB: insert files row (status=uploading)
        US->>DB: insert audit
        US->>DB: COMMIT
    end

    US->>Q: emit file.finalized
    US-->>GW: 200 {file_id, status: "scanning"} (~500ms)
    GW-->>Alice: 200

    Q->>VS: file.finalized
    VS->>VS: scan bytes (1-3 min async)
    VS->>DB: UPDATE files SET status = ready
```

Three things to notice:

1. Quota is reserved at init, not finalize. If Alice's phone and laptop both start uploading an 80 MB file when she has only 100 MB left, the `UPDATE WHERE has_room` serializes them. Only one wins.
2. The blob upsert, file row, and audit write happen in one transaction. A crash mid-write rolls back cleanly. State is never partial.
3. Virus scan runs after Alice gets her 200. Scan results arrive asynchronously, 1-3 minutes later.

---

## The dedup race

Two users upload the same file within milliseconds. Who wins? Both should.

```mermaid
sequenceDiagram
    autonumber
    participant Alice
    participant Bob
    participant US as Upload Service
    participant S3
    participant DB as Postgres

    Note over Alice,Bob: both compute SHA-256 = abc123... client-side
    Alice->>US: POST /uploads/init {hash: abc123}
    Bob->>US: POST /uploads/init {hash: abc123}

    US->>DB: check blob abc123
    DB-->>US: no match (not yet)
    US->>S3: create_multipart_upload (for Alice)
    US-->>Alice: presigned_urls[]

    US->>DB: check blob abc123
    DB-->>US: no match (not yet)
    US->>S3: create_multipart_upload (for Bob)
    US-->>Bob: presigned_urls[]

    Note over Alice,S3: both upload in parallel
    Alice->>US: finalize
    Bob->>US: finalize

    rect rgb(241, 245, 249)
        Note over US,DB: transaction A (Alice wins the upsert)
        US->>DB: INSERT INTO blobs (hash=abc123, refcount=1)
        DB-->>US: ok. refcount=1
        US->>DB: INSERT files row for Alice
    end

    rect rgb(241, 245, 249)
        Note over US,DB: transaction B (Bob hits ON CONFLICT)
        US->>DB: INSERT INTO blobs (hash=abc123) ON CONFLICT DO UPDATE SET refcount=refcount+1
        DB-->>US: ok. refcount=2. S3 key already set.
        US->>DB: INSERT files row for Bob
    end

    Note over US,S3: abort Bob's redundant S3 multipart (bytes already exist)
    US->>S3: abort_multipart_upload (Bob's session)
```

The `ON CONFLICT DO UPDATE` is the atomic guard. No matter how many concurrent finalizes race in, the blob is created once and the refcount increments correctly. The second uploader's S3 object gets aborted because the blob already has a valid storage key.

> **Take this with you.** The database unique constraint on the blob hash is what makes concurrent dedup correct. The application does not need a lock.

---

## Follow-up questions

Try answering each in 2-4 sentences before reading the solution.

1. **Resume the next day.** Alice uploads 3 GB of a 5 GB file, then closes her laptop. The next morning she reopens the app. What happens? How does the client know which chunks already landed? How long do you keep half-finished uploads around?

2. **Quota race.** Alice has 100 MB of quota left. Her phone and laptop both start uploading 80 MB files at the same instant. Both pass the quota check at init. Both upload. Now she is 60 MB over quota. How do you prevent this?

3. **Dedup details.** Three users upload the same 200 MB installer. How do you store it once? What does "delete" mean when one user deletes their copy? What about privacy across tenants?

4. **Token guessing.** Your tokens are 192 bits, so brute force is out. But a researcher finds your `created_at` timestamps in the response. Is this a real attack? What other side channels leak?

5. **Big delete.** A user with a 50 TB account deletes 10 TB in one click. Your metadata DB does 200,000 row updates and S3 issues 200,000 delete requests. What goes wrong? How do you smooth it out?

6. **Late-positive virus scan.** A scan flags a file as malware after 500 people have already downloaded it. What is your response? Can you tell who downloaded it? What about the share links?

7. **Edit conflict.** Two users with Edit permission upload a new version of the same file within 10 seconds. Whose version wins? How does the loser find out?

8. **Viral file.** A YouTuber's public share link gets 1 million downloads in 24 hours for a 200 MB tutorial video. CDN cache hits 99%, but the 1% miss rate still hammers one S3 prefix. What do you do?

9. **GDPR delete.** A user wants their data fully erased. They have 12,000 files, some deduped with other users. They also created share links and were granted shares on other users' files. How do you erase them?

10. **Per-tenant billing.** You sell this to enterprises. One customer wants a monthly bill: storage GB by tier, egress GB, virus-scan calls, API requests. How do you attribute every byte and every call to the right tenant?

---

## Related problems

- **[Video Streaming (006)](../006-video-streaming/question.md).** Same shape: bytes in S3, metadata in Postgres, CDN in front. Video adds adaptive bitrate transcoding. The storage and CDN layers overlap heavily.
- **[Distributed Cache (009)](../009-distributed-cache/question.md).** The permission resolver cache and the CDN edge cache both follow the same eviction and warming patterns.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md).** The "show me my files" dashboard and share-link resolution are textbook read-heavy paths.
- **[Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md).** The audit log here is exactly a write-heavy append-only system.
