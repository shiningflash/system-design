## Solution: File Upload & Share Service

### The short version

This looks like a thin HTTP wrapper around S3. It is not. The interesting design is everything around the bytes:

- **How does a 5 GB upload survive a bad network?** Chunked upload with presigned URLs. Client splits the file into 8 MB pieces. Each piece uploads on its own. Failed chunks retry without restarting the whole file.
- **How do you store the same file once when 50 people upload it?** Content-addressed dedup. Hash the file content (SHA-256). Two files with the same bytes share one set of bytes in S3.
- **How do you revoke one share link without breaking 999 others?** One row per link in `share_links` with a `revoked_at` column. Revoke is one UPDATE.
- **How does a file nobody has touched in two years stop costing money?** S3 lifecycle policy moves it to cheaper tiers (S3 IA after 90 days, Glacier after 365 days).

The data model fits on a napkin. Seven tables: `files`, `file_versions`, `blobs`, `shares`, `share_links`, `upload_sessions`, `audit`. Bytes live in S3, addressed by their content hash. Metadata is sharded by owner because almost every query is "show me my stuff."

Uploads go client-to-S3 directly via presigned URLs. Your app servers never touch the bytes. That one decision saves you an order of magnitude on bandwidth cost.

---

### 1. The two questions that matter most

**What is the biggest file size?** Anything above ~100 MB forces chunked or presigned uploads. 5 GB forces S3 multipart.

**Sync or share-only?** Sync (Dropbox desktop) is a different problem: delta sync, conflict resolution, file watchers. Share-only (Google Drive web) is what this design covers.

Everything else (versioning, virus scan, GDPR, quotas) follows from those two answers.

---

### 2. The math, in plain numbers

| Scale | Uploads/sec | Downloads/sec | Storage/year | Egress peak |
|-------|------------|--------------|-------------|------------|
| 10k users | ~0.08 sustained, ~0.25 peak | ~0.8 | ~13 TB | ~100 Mbps |
| 100M users | ~3,300 sustained, ~10k peak | ~33k sustained, ~100k peak | ~580 PB (after 30% dedup) | ~6.4 Tbps |

Three numbers that matter more than throughput:

- **Storage cost is the headline expense.** At PB scale, $0.023/GB/month for S3 Standard is hundreds of millions per year. Lifecycle tiers and dedup are survival, not optimization.
- **The system is read-heavy by request count, write-heavy by bytes.** CDN absorbs most downloads. S3 ingests all upload bytes.
- **Metadata DB is tiny.** 100B file rows at ~500 bytes each is ~50 TB. Sharded Postgres handles it. The bottleneck is bytes, not rows.

---

### 3. The API

Five endpoints carry the whole product.

```
POST /api/v1/uploads/init
Authorization: Bearer <token>

{
  "file_name": "vacation.mp4",
  "size": 1572864000,
  "mime_type": "video/mp4",
  "content_hash": "sha256:abc123...",
  "parent_folder_id": "fld_xyz",
  "client_idempotency_key": "uuid"
}
```

| Status | Meaning | Body |
|--------|---------|------|
| 201 Created | New upload session | `{upload_id, chunk_size, presigned_urls[]}` |
| 200 OK | Dedup hit. File already exists. No upload needed. | `{file_id, deduped: true}` |
| 400 | File too big | `{error: "file_too_large"}` |
| 402 | Out of quota | `{error: "quota_exceeded", available_bytes: ...}` |

Client then `PUT`s directly to the presigned S3 URLs. Your server is not in the path.

```
POST /api/v1/uploads/{upload_id}/finalize
{ "parts": [{"part": 1, "etag": "abc"}, ...] }

POST /api/v1/files/{file_id}/share_links
{
  "permission": "download",
  "expires_at": "2026-08-01T00:00:00Z",
  "password": "optional",
  "max_redemptions": null
}

GET /api/v1/share/{token}          -- redeem a share link
GET /api/v1/files/{file_id}/download  -- direct download (307 to signed URL)
```

Three small but load-bearing choices:

- **Idempotency on upload init is required.** Mobile clients retry on timeout. Without it, retries create new sessions and you get orphaned half-uploads everywhere.
- **The content hash at init is what makes "re-uploading my photo library" instant.** Client computes SHA-256. Server checks for a matching blob. If found, return the existing file_id and skip the upload entirely.
- **Finalize takes ETags because S3 multipart finalize needs them.** S3 stitches chunks together based on the part list with ETags.

---

### 4. The data model

Seven tables. Two big, five supporting.

```mermaid
erDiagram
    blobs ||--o{ files : "content_hash"
    files ||--o{ file_versions : has
    files ||--o{ shares : "direct invite"
    files ||--o{ share_links : "link share"
    files ||--o{ audit : events

    blobs {
        bytea content_hash PK
        bigint size_bytes
        text storage_key
        int refcount
        smallint storage_tier
    }
    files {
        uuid file_id PK
        bigint owner_id
        uuid parent_folder_id
        text name
        bigint size_bytes
        bytea content_hash
        smallint status
        smallint storage_tier
    }
    share_links {
        varchar token PK
        uuid file_id
        bigint created_by
        smallint permission
        timestamptz expires_at
        bytea password_hash
        int redemptions
        timestamptz revoked_at
    }
```

<details markdown="1">
<summary><b>Show: the full SQL for the two core tables</b></summary>

```sql
CREATE TABLE blobs (
    content_hash     BYTEA PRIMARY KEY,
    size_bytes       BIGINT NOT NULL,
    storage_key      TEXT NOT NULL,          -- S3 key /raw/<hash>
    refcount         INT NOT NULL DEFAULT 1,
    first_uploaded   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    storage_tier     SMALLINT NOT NULL DEFAULT 1  -- 1=hot, 2=warm, 3=cold
);

CREATE TABLE files (
    file_id          UUID PRIMARY KEY,
    owner_id         BIGINT NOT NULL,
    parent_folder_id UUID,
    name             TEXT NOT NULL,
    size_bytes       BIGINT NOT NULL,
    content_hash     BYTEA NOT NULL REFERENCES blobs(content_hash),
    current_version  INT NOT NULL DEFAULT 1,
    status           SMALLINT NOT NULL DEFAULT 1, -- 1=uploading, 2=ready, 3=quarantined, 4=deleted
    storage_tier     SMALLINT NOT NULL DEFAULT 1,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at       TIMESTAMPTZ
);
CREATE INDEX idx_files_owner  ON files (owner_id, parent_folder_id);
CREATE INDEX idx_files_hash   ON files (content_hash);   -- dedup lookups

CREATE TABLE upload_sessions (
    upload_id         UUID PRIMARY KEY,
    user_id           BIGINT NOT NULL,
    expected_size     BIGINT NOT NULL,
    expected_hash     BYTEA,
    total_chunks      INT NOT NULL,
    s3_upload_id      TEXT NOT NULL,
    status            SMALLINT NOT NULL DEFAULT 1,  -- 1=active, 2=finalized, 3=abandoned
    idempotency_key   TEXT,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at        TIMESTAMPTZ NOT NULL
);
CREATE UNIQUE INDEX idx_session_idem ON upload_sessions (user_id, idempotency_key);

CREATE TABLE share_links (
    token              VARCHAR(32) PRIMARY KEY,
    file_id            UUID NOT NULL REFERENCES files(file_id),
    created_by         BIGINT NOT NULL,
    permission         SMALLINT NOT NULL,
    expires_at         TIMESTAMPTZ,
    password_hash      BYTEA,
    require_account    BOOLEAN DEFAULT FALSE,
    max_redemptions    INT,
    redemptions        INT NOT NULL DEFAULT 0,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at         TIMESTAMPTZ
);
CREATE INDEX idx_links_file ON share_links (file_id);
```

</details>

Four choices worth defending out loud:

**`blobs` is separate from `files`.** A blob is the actual bytes, addressed by hash. A file is a user-named pointer to a blob. Many files can point at one blob. `refcount` tracks how many files reference each blob. When it hits zero, the blob is eligible for deletion with a 24-hour grace period.

**Sharded by `owner_id`.** Almost every query is "list my files in folder X." Co-locating one user's files on one shard makes those queries single-shard. Cross-shard share lookups use a separate global index on `shares.granted_to`.

**`upload_sessions` lives in Postgres, not Redis.** Sessions can live for hours and must survive cache eviction. Volume is low (one row per in-flight upload).

**`share_links.token` is opaque.** No relationship to the file, the owner, or the creation time. Leaking the generation algorithm leaks nothing about existing tokens.

---

### 5. Core algorithms

**Upload init and finalize:**

<details markdown="1">
<summary><b>Show: init_upload and finalize_upload</b></summary>

```python
def init_upload(user_id, file_name, size, content_hash, idempotency_key):
    existing = db.find_session_by_idempotency(user_id, idempotency_key)
    if existing:
        return existing

    if content_hash:
        blob = db.find_blob(content_hash)
        if blob and blob.size_bytes == size:
            file_id = create_file_pointing_at_blob(user_id, file_name, blob)
            return {"deduped": True, "file_id": file_id}

    if not reserve_quota(user_id, size):
        raise QuotaExceeded

    s3_upload_id = s3.create_multipart_upload(bucket="user-data", key=f"raw/pending/{uuid4()}")
    chunk_size = 8 * 1024 * 1024
    total_chunks = ceil(size / chunk_size)

    presigned_urls = [
        s3.presign_upload_part(s3_upload_id, part_number=i+1, expires_in=3600)
        for i in range(total_chunks)
    ]

    session = db.insert_session(
        user_id=user_id, expected_size=size, expected_hash=content_hash,
        total_chunks=total_chunks, s3_upload_id=s3_upload_id,
        expires_at=now() + 24h
    )
    return {"upload_id": session.id, "chunk_size": chunk_size, "presigned_urls": presigned_urls}


def finalize_upload(upload_id, parts):
    session = db.lock_session(upload_id)
    if session.status != ACTIVE:
        raise AlreadyFinalized
    if len(parts) != session.total_chunks:
        raise MissingChunks

    result = s3.complete_multipart_upload(session.s3_upload_id, parts)

    with db.transaction():
        blob = db.upsert_blob(session.expected_hash, session.expected_size, result.key)
        file_id = db.insert_file(session.user_id, session.file_name, blob)
        db.insert_audit(file_id, "file.created")
        db.mark_session_finalized(upload_id)

    publish_event("file.finalized", file_id)
    return file_id
```

</details>

**Share link resolution** (the hot path):

```python
def resolve_share_link(token, password=None):
    link = db.find_share_link(token)
    if not link or link.revoked_at:
        return 404
    if link.expires_at and link.expires_at < now():
        return 410
    if link.max_redemptions and link.redemptions >= link.max_redemptions:
        return 410
    if link.password_hash and not bcrypt.verify(password, link.password_hash):
        return 401

    signed_url = cloudfront.sign(
        key=link.file.blob.storage_key,
        expires_in=900  # 15 minutes
    )
    db.increment_redemptions(token)
    return 302, signed_url
```

**Dedup upsert at finalize:**

```sql
INSERT INTO blobs (content_hash, size_bytes, storage_key, refcount)
VALUES (?, ?, ?, 1)
ON CONFLICT (content_hash) DO UPDATE SET refcount = blobs.refcount + 1;
```

If the blob already exists, no new S3 object is written. The `files` row points at the existing blob.

---

### 6. The architecture

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

Five things to notice:

- The Upload Service mints presigned URLs and never touches the bytes. That single decision is what lets one small pod handle thousands of uploads per second.
- CloudFront sits in front of S3 for downloads. Short TTL (60 seconds) so revocations propagate fast. Without it, a viral file melts your S3 egress bill.
- The Permission Resolver is a separate concern because it is the hottest read path. "Can user X do Y on file Z?" combines owner check, direct share, and folder share inheritance. Cache 30 seconds per `(user, file)` pair. Most lookups never touch the DB.
- Virus scan and lifecycle GC are downstream of SQS/Kafka, not on the write path. If the scan worker falls behind, uploads still succeed.
- The metadata DB is sharded by `owner_id`. Cross-shard share lookups go through a global index.

---

### 7. A request, end to end

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

    Alice->>GW: POST /uploads/init {size, content_hash}
    GW->>US: forward (auth ok)
    US->>DB: check blob by content_hash
    DB-->>US: no match

    rect rgb(241, 245, 249)
        Note over US,DB: atomic quota reservation
        US->>DB: UPDATE users SET reserved_bytes += size WHERE has_room
        DB-->>US: ok
    end

    US->>S3: create_multipart_upload
    S3-->>US: s3_upload_id
    US->>DB: INSERT upload_session
    US-->>GW: 201 {upload_id, presigned_urls[]}
    GW-->>Alice: 201

    Note over Alice,S3: chunks upload directly to S3 in parallel
    Alice->>S3: PUT chunks 1..N (presigned)
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
    US-->>GW: 200 {file_id}
    GW-->>Alice: 200

    Q->>VS: file.finalized
    VS->>VS: scan bytes
    VS->>DB: UPDATE files SET status = ready
```

Target latencies:

| Operation | P99 |
|-----------|-----|
| Upload init | ~200 ms (dedup lookup when cache cold) |
| Finalize | ~500 ms (S3 multipart completion) |
| Permission resolution | ~50 ms (cached) |
| Download (CDN hit) | ~30 ms (edge latency) |

---

### 8. Storage tiers

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

| Tier | Cost/GB/month | Retrieval | Retrieval cost |
|------|--------------|---------|---------------|
| **S3 Standard** | $0.023 | < 100 ms | free |
| **S3 IA** | $0.0125 | < 100 ms | $0.01/GB |
| **Glacier** | $0.0036 | 1-5 min fast, 3-5 hr standard | $0.03/GB |

On 580 PB cold: Glacier is ~$25M/year. Standard is ~$160M/year. The lifecycle policy is the business model.

Three gotchas: Glacier retrieval surprises users (show progress UI), small files under 128 KB cost more to tier to IA than to leave hot, and cold-tier deletes incur a 90-day minimum storage penalty (soft-delete first, hard-delete later).

---

### 9. Scaling journey: 10 to 1M users

```mermaid
flowchart LR
    S1["Stage 1<br/>10-1k users<br/>1 VM + local disk<br/>~$20/mo"]:::s1
    S2["Stage 2<br/>~10k users<br/>+ S3 presigned<br/>+ share links<br/>~$100/mo"]:::s2
    S3["Stage 3<br/>100k users<br/>+ chunked upload<br/>+ CDN + scan<br/>+ lifecycle<br/>~$10k/mo"]:::s3
    S4["Stage 4<br/>1M users<br/>+ regional buckets<br/>+ sharded DB<br/>+ per-tenant KMS<br/>~$200k/mo"]:::s4

    S1 --> S2 --> S3 --> S4

    classDef s1 fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef s2 fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef s3 fill:#fef3c7,stroke:#a16207,color:#713f12
    classDef s4 fill:#fce7f3,stroke:#be185d,color:#831843
```

#### Stage 1: 10 to 1,000 users

One VM. Files saved to local disk under `/var/data/<user_id>/<file_id>`. Postgres on the same machine. Direct POST upload, files capped at 100 MB. Direct invite sharing only. No virus scan, no versioning, no quota. ~$20/month. Ships in a weekend.

The throughput is tiny (well under 1 upload per minute). Nothing is big enough to fail mid-upload. No presigned URLs needed yet.

#### Stage 2: 10,000 users

Something breaks: local disk fills up. A user's 2 GB upload drops at 1.8 GB and they have to start over.

Fixes:
- Move bytes to S3 via presigned PUT URLs. App server leaves the byte path.
- Add anonymous share links. `share_links` table with 192-bit tokens.
- Add view/download/edit permission scopes.
- Add per-user quota with soft-delete and 30-day trash.

Still no chunked upload. Still no CDN. One DB. ~$100-300/month.

#### Stage 3: 100,000 users

Several things break at once:
- A user uploads a 4 GB video on hotel WiFi. Fails at 80%. Whole thing restarts. 1-star review.
- A malware PDF gets shared and downloaded 500 times before anyone notices.
- Storage cost at $3-5k/month growing 30% per quarter. 70% of bytes untouched after 30 days.
- S3 egress at $2k/month because every download streams full-price from S3.

Fixes, in order:
- Chunked upload via S3 multipart. A flaky 4 GB upload now survives.
- Virus scan pipeline via SQS. Async, does not block uploads.
- CloudFront in front of S3 with 60-second TTL on signed URLs. Egress drops ~90%.
- S3 lifecycle policy: 90 days to IA, 365 to Glacier. Saves ~70% on cold bytes.
- Two Postgres read replicas. Dashboards read from replicas.
- Atomic quota reservation with `UPDATE WHERE has_room`.

Still single-region. DB does not need sharding yet (100M rows at 500 bytes is 50 GB). ~$10-20k/month.

#### Stage 4: 1 million users

New pains:
- EU users complain about latency uploading to us-east-1.
- A healthcare customer demands data residency.
- A viral share link gets 1M downloads in 24 hours. CloudFront absorbs 99% but the 1% concentrates on one S3 prefix and triggers S3 throttling.
- Metadata DB primary at 60% CPU at peak.

Fixes:
- Regional S3 buckets per region. Files land in the user's home region.
- Shard metadata DB by `owner_id`. Each region is primary for its own users.
- Share link tokens encode the file's home region in the first few characters so any region routes correctly.
- CloudFront with multiple origins and hash-prefix-sharded S3 keys so viral files spread across prefixes.
- Per-user quota in Redis with `INCRBY` reservations for the race-free fast path.
- Per-tenant KMS keys for enterprise (customer-controlled kill switch).

The architecture is the same shape as Stage 3, multiplied across regions. ~$200-500k/month, dominated by S3 storage and egress.

---

### 10. Reliability

**Interrupted upload resume.** Upload sessions live 24 hours in `upload_sessions`. Client calls `GET /uploads/{upload_id}` to ask which chunks have landed. Server queries S3 `ListParts` and returns uploaded part numbers. Client re-uploads only missing parts. After 24 hours, the Lifecycle Manager calls `s3.abort_multipart_upload` and marks the session abandoned.

**Orphan chunks.** Lifecycle Manager runs every 6 hours. For each session with `status=active AND expires_at < now()`, abort the S3 multipart. Safety net: a global S3 lifecycle rule aborts any multipart older than 7 days.

**Virus scan failures.** Three flavors:
- Worker dies mid-scan. SQS visibility timeout expires. Another worker picks up the message. Idempotent.
- Scan API is down. Worker retries with backoff. If down over an hour, files remain `status=uploading` and downloads return 425 Too Early.
- Scan finds a virus after downloads have already happened. Flip `status=quarantined`. Set `revoked_at` on all share links for the file. Notify authenticated downloaders by email.

**Metadata DB primary failure.** Promote a replica. Reads continue from other replicas during the 30-60 second promotion. In-flight upload sessions see a few errors and retry via the idempotency key.

**S3 outage.** Presigned PUTs return 5xx. Signed GETs return 5xx. CloudFront serves whatever it has cached for downloads. Metadata operations still work. Honest answer: wait for S3 to recover. Multi-region replication helps for reads but is expensive and only justified for enterprise HA tiers.

---

### 11. Observability

| Metric | Why it matters |
|--------|---------------|
| `upload.init.rate` | Drop signals auth or API issues. |
| `upload.success_rate` (finalize/init) | Headline UX SLO. If 60% of inits never finalize, something is broken. |
| `upload.duration` by file size bucket | Tells you "all uploads are slow" from "1 GB+ uploads are slow." |
| `download.cache_hit_rate` (CDN) | Should be > 95% in steady state. |
| `share_link.resolution.p99` | Hot path latency. |
| `share_link.brute_force_alerts` | Tokens with > 50 failed redemption attempts in an hour. |
| `quota.exceeded.rate` | Spikes signal a rogue user or integration. |
| `virus_scan.queue_depth` | If growing, scanner cannot keep up. Recent uploads are in a scan gap. |
| `virus_scan.positive_rate` | Sudden spike means a malware campaign. |
| `blob.refcount.zero.count` | Eligible-for-GC blobs. Should drain over time. |
| `storage.by_tier.bytes` | Hot/warm/cold breakdown. Input for cost forecasting. |
| `egress.bytes_per_region` | Where the money goes. |

Page on: upload success rate under 90% for 5 min. Virus scan queue depth over 10k. CDN origin error rate over 1%.

Ticket on: per-user bandwidth anomaly. Quota race detected. Storage growth more than 2 sigma above forecast.

---

### 12. Follow-up answers

**1. Resumable upload across days.**

Client persists `upload_id` locally. On reopen, calls `GET /uploads/{upload_id}`. Server queries S3 `ListParts`, returns uploaded part numbers and original chunk size. Client uploads only missing parts and finalizes. If the session is past its 24-hour TTL, client gets 410 Gone and must start fresh. Abandoned sessions get GC'd every 6 hours by the Lifecycle Manager calling `AbortMultipartUpload`.

**2. Quota race.**

Fix with an explicit reservation:

```sql
UPDATE users
   SET reserved_bytes = reserved_bytes + ?
 WHERE user_id = ?
   AND used_bytes + reserved_bytes + ? <= quota_bytes
RETURNING reserved_bytes;
```

If this returns 0 rows, the upload is rejected with 402. Reservation is held until finalize (moves to `used_bytes`) or session expiry (released). At higher scale, run the same logic in Redis with `INCRBY` and compare-and-swap, with periodic reconciliation back to the DB.

**3. Content-addressed dedup.**

A `blobs` table keyed by SHA-256. On finalize:

```sql
INSERT INTO blobs (content_hash, size_bytes, storage_key, refcount)
VALUES (?, ?, ?, 1)
ON CONFLICT (content_hash) DO UPDATE SET refcount = blobs.refcount + 1;
```

If the blob exists, no new S3 object is written. The `files` row points at the existing blob. Delete decrements `refcount` atomically. When it hits zero, schedule the S3 object for deletion after a 24-hour grace period. For sensitive use cases (legal, medical), disable cross-tenant dedup and dedup only within one account.

**4. Share link enumeration.**

192 bits of entropy makes direct brute force impossible. Side channels are the real concern:
- Timing: "token not found" and "token found but expired" should take the same time. Use a unified error path.
- Error leakage: return the same 404 for "no such token" and "expired." Do not include `expired_at` in unauthenticated responses.
- Web logs: tokens appear in access logs. Treat them as secrets. Hash before logging.
- Rate limiting: per-IP limits on `/share/*` thwart high-volume probing.

**5. Large delete tombstone backlog.**

200k file row updates and 200k S3 DELETEs done synchronously cause DB lock contention, S3 throttling, and HTTP timeouts. The user clicks Delete again and doubles the load.

Fix: write a single `deletion_jobs` row `(user_id, target_folder_id, requested_at)` and return 202 Accepted with a `job_id`. A background worker processes in batches of 1,000: soft-delete in DB, decrement blob refcounts, enqueue S3 deletion (S3 bulk-delete handles 1,000 objects per call). After 30 days of trash retention, a sweeper hard-deletes blobs whose refcount reached zero.

**6. Late-positive virus scan.**

In order:
1. Flip `status` to quarantined.
2. `UPDATE share_links SET revoked_at = NOW() WHERE file_id = ?`.
3. Invalidate CloudFront for the file's URL prefix.
4. Query audit for `event_type = 'file.downloaded' AND file_id = ?` to identify authenticated downloaders.
5. Notify them in-app and by email: "A file you downloaded was later found to contain malware."
6. Notify the uploader.

Pre-mitigation: keep the post-finalize, pre-scan window short. Faster scanning narrows the exposure gap.

**7. Edit conflict.**

Two users with Edit upload new versions within 10 seconds. Default behavior: both succeed, both create version rows. `files.current_version` points to the last one to land. The losing user's version exists in `file_versions`.

Surface a conflict notification: "User B also uploaded a new version at the same time. Theirs is now current. Yours is in history at version N."

For stronger guarantees, accept `If-Match: <current_version>` on upload. A mismatch returns 412 Precondition Failed and the client shows a conflict picker.

**8. Viral file.**

1M downloads of a 200 MB file in 24 hours. 99% CDN hit. The 1% miss is ~10k requests to S3 over the day, but if they cluster they can hit S3's 5,500 GET/s per-prefix limit.

Mitigations:
- Pre-warm CDN edges when a high-traffic link is created (push to all CloudFront edges immediately).
- Hash-prefix-shard S3 keys: store blobs at `/raw/<hash[0:2]>/<hash[2:4]>/<hash>` so popular files land on different prefixes.
- CloudFront Origin Shield: a regional cache between edges and S3, collapsing many edge misses into one S3 fetch.

**9. GDPR delete.**

1. Mark account `status = deleting`, log the user out, freeze to read-only.
2. Enqueue a `deletion_jobs` row for background processing.
3. Walk owned files: decrement each blob's refcount, revoke all share links the user created.
4. Walk received shares: set `revoked_at` where the user is `granted_to`. The file stays for the owner.
5. Walk audit log: replace `actor_id` with a hash, drop PII from payloads.
6. Delete the user row after the grace period (typically 30 days).
7. Email a deletion certificate.

The tricky part: files deduped with other users. Decrementing refcount may leave the blob alive. That is correct. The user's pointer is gone even if the bytes persist (referenced by someone else). If the user demands the bytes be deleted regardless, copy the blob to a user-private path for remaining references, then delete. Do this only on explicit request.

**10. Per-tenant cost attribution.**

Every billable action is tagged with `tenant_id`:
- **Storage:** S3 inventory runs daily, listing all objects with size and tier. Keys encode `tenant_id` (`/raw/<tenant>/<hash>`). Daily job aggregates by tenant.
- **Egress:** CloudFront real-time logs include the URL. URLs encode `tenant_id`. Daily job aggregates bytes-out per tenant.
- **API requests:** API Gateway access logs include the JWT's `tenant_id` claim.
- **Virus scan calls:** Scan worker writes each scan to a `scan_invocations` table with `tenant_id`.

A nightly job rolls these into a `billing_lines` table with columns for `storage_hot_gb`, `storage_warm_gb`, `storage_cold_gb`, `egress_gb`, `api_requests`, `scan_calls`. The sum of per-tenant attributions should match your AWS bill within ~1%. Gaps are platform overhead.

---

### 13. Trade-offs worth saying out loud

**Single big PUT vs chunked.** Single PUT is simpler for files under 100 MB. Chunked is necessary above 5 GB (S3's single-PUT limit) and strongly recommended above 100 MB for network resilience. Right answer: hybrid. Single PUT below a threshold, chunked above.

**Client-side encryption.** Optional, important for high-security customers. Encrypt with a key the server never sees. Tradeoff: no dedup possible (ciphertext differs for identical plaintext), no server-side preview, key management is the customer's burden. Implement as opt-in for enterprise tier.

**Dedup by content hash.** Saves ~30% storage at consumer scale. Adds complexity: refcount management, GC, blob lifecycle, privacy. Worth it at scale. Skip for the first 1,000 users.

**Lifecycle to cold tier.** Saves 50-70% of storage cost on the cold tail. Costs: retrieval latency surprises users, retrieval fees on access, minimum-storage-duration penalties on early delete. Tune the transition age to your actual access pattern. 90 days is sometimes too aggressive for bursty files.

**Postgres vs DynamoDB for metadata.** Postgres gives joins, transactions, and a familiar query model. DynamoDB gives infinite horizontal scale and predictable latency. Under 100M files, Postgres is simpler. Above that, sharding Postgres becomes painful and DynamoDB is worth considering. Do not switch preemptively.

**Sync vs async virus scan.** Sync blocks the upload UX for minutes on big files. Async accepts a 1-3 minute exposure window. Almost everyone picks async. The exposure is small; the UX win is large.

---

### 14. Common mistakes

**Tunneling uploads through your app server.** The biggest scaling mistake. Bandwidth cost is per-byte. Doubling traffic doubles your NIC bill. Presigned-direct-to-S3 is non-negotiable above ~100 concurrent uploads.

**Single POST for all uploads.** Works until the first 4 GB upload fails on hotel WiFi. Chunked is mandatory above ~100 MB.

**No quota enforcement at all.** "We will add it later" turns into "one user uploaded their Steam library and ate our budget." Quota at signup is easier to retrofit than quota after the fact.

**Forgetting to GC abandoned uploads.** S3 multipart uploads that never finalize keep costing money. A nightly sweeper plus an S3 lifecycle rule that aborts multiparts older than 7 days is the safety net.

**Share links that are just file IDs in the URL.** `https://app.com/file/abc123` is not a share link. It is a permanent backdoor. Use an opaque high-entropy token with explicit expiry and revocation.

**No virus scan at all.** Fine for an internal tool. Indefensible for a public product. Basic ClamAV catches most known malware.

**Hot path doing folder traversal for permissions.** "Is this file in any folder shared with me?" walked on every download is slow at scale. Cache or materialize the access set per user.

**Skipping the `status` field on files.** Without an explicit state machine (uploading, ready, quarantined, deleted), you get edge cases: downloads of half-finalized files, deletes of in-flight uploads. The `status` smallint pays for itself in week 1.

**Mixing user files and system files in one bucket without prefixing.** Lifecycle rules apply per bucket or prefix. If you mix, you cannot tier user files independently of thumbnails or temp data. Pick a prefix scheme on day one.

**No audit log.** When a security report comes in ("user X claims their file was leaked"), the absence of "who accessed this and when" turns a 1-hour investigation into a week-long one. Audit is cheap to build, painful to retrofit.

If you can hit 7 of these 10 and walk through the upload-protocol trade-off and the scaling journey in the same conversation, you are interviewing well above the bar for this problem.
