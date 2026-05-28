## Solution: Notification System

### What this system is

A notification system is a policy-enforcement pipeline. One event comes in. The system decides, per recipient and per channel, whether to send at all. If yes, it delivers the message through a third-party provider (APNs, FCM, SendGrid, Twilio) and retries on transient failure without sending duplicates.

The architecture is a stateless Fan-out Service in front of per-channel Kafka queues in front of channel worker pools. Fan-out is the brains: it checks user preferences, quiet hours, per-user caps, and aggregation windows. Workers are the hands: they call providers, retry with backoff, and record results.

What trips candidates: one worker pool for all channels (one bad provider stalls all channels), synchronous Postgres preference reads on the hot path (cannot scale past ~1,000/sec), and retries without dedup keys (users get the same SMS twice).

---

### 1. The two questions that matter most

**Fan-out shape: transactional or marketing?** A transactional event (Alice liked your post) goes to one user. A marketing campaign targets ten million. These need separate code paths. The marketing path needs a Campaign Scheduler, a rate limiter, and per-user local-time delivery windows. The transactional path needs low latency and high dedup correctness. Conflating them builds the wrong system.

**What are the delivery freshness requirements per channel?** A push notification about a nearby driver arriving must land within 60 seconds or it is useless. An invoice email can arrive 4 hours late and still be useful. These different deadlines drive completely different retry policies. Without asking this question, you cannot size the retry windows.

---

### 2. The math, in plain numbers

| Scale | Notifications/day | Sustained QPS | Peak QPS | Storage (30 days) |
|-------|-------------------|---------------|----------|-------------------|
| Startup (100k users) | ~1 million | ~12 | ~50 | 3.6 GB |
| Mid-size (10M users) | ~100 million | ~1,200 | ~5,000 | 360 GB |
| Billion-user product | ~10 billion | ~116,000 | ~350,000 | 36 TB |

Per-channel split at billion-user scale:

| Channel | Share | QPS |
|---------|-------|-----|
| Push (iOS + Android) | 60% | ~70,000 |
| In-app | 30% | ~35,000 |
| Email | 7% | ~8,000 |
| SMS | 3% | ~3,500 |

One marketing campaign targeting 10 million users in 5 minutes: 10M / 300 sec = ~33,000/sec for that burst, roughly 30% on top of the baseline.

Worker pool sizing: at 500 provider calls per second per worker, sustained 116,000/sec needs ~230 workers; peak needs ~700. Scale on Kafka consumer lag.

The real ceiling is provider quotas, not compute. APNs throttles per HTTP/2 connection per certificate. Twilio throttles per sender number. At scale, hold many provider credentials and round-robin across them.

---

### 3. The API

Two endpoints carry the core product.

**Event submission (from producer services):**
```
POST /api/v1/events
Idempotency-Key: <event_id>

{
  "event_id": "evt_8f3a91...",
  "event_type": "post.liked",
  "category": "social",
  "actor": { "user_id": 123 },
  "recipients": [{ "user_id": 456, "context": { "post_id": 789 } }],
  "aggregation_key": "post:789:likes",
  "template_id": "tpl_like_v3",
  "template_vars": { "actor_name": "Alice", "post_title": "My trip" },
  "ttl_seconds": 3600
}
```

| Status | Meaning |
|--------|---------|
| **202** | Event queued |
| **200** | Same event_id already accepted (idempotent replay) |
| **400** | Invalid payload or unknown template |
| **413** | Recipient list exceeds 10k; use the campaign endpoint |
| **429** | Producer rate limit hit |

Three load-bearing choices in the payload:

- `Idempotency-Key` is required. Mobile apps and microservices retry on timeout. Without it, a retry creates a duplicate notification.
- `category` controls policy. `transactional` bypasses quiet hours and per-user caps. `marketing` does not.
- `ttl_seconds` is a hard deadline. A push that sits in a backed-up queue past its TTL is dropped, not delivered stale. This prevents the flood-at-recovery problem after a provider outage.

**Campaign endpoint (for marketing blasts):**
```
POST /api/v1/campaigns
{
  "campaign_id": "cmp_winter_2026",
  "segment_query": { "country": "US", "tier": "active" },
  "template_id": "tpl_winter_v2",
  "channels": ["push", "email"],
  "send_window": {
    "start_local_time": "10:00",
    "end_local_time": "20:00",
    "timezone_strategy": "per_user_local"
  },
  "rate_limit": { "per_second": 5000 }
}
```

**Preferences:**
```
PUT /api/v1/users/me/notification-preferences
{
  "channels": { "push": { "enabled": true }, "sms": { "enabled": false } },
  "categories": { "marketing": { "enabled": false }, "transactional": { "enabled": true } },
  "quiet_hours": { "enabled": true, "start": "22:00", "end": "07:00", "timezone": "America/Los_Angeles" }
}
```

---

### 4. The data model

```mermaid
erDiagram
    user_preferences ||--o{ notifications : "guides fan-out"
    templates ||--o{ notifications : "versioned"
    user_devices ||--o{ notifications : "token lookup"
    notifications }o--|| events : "created from"

    user_preferences {
        bigint user_id PK
        jsonb channels
        jsonb categories
        time quiet_hours_start
        time quiet_hours_end
        text timezone
        int version
    }
    notifications {
        uuid notification_id PK
        uuid event_id
        bigint recipient_user_id
        smallint channel
        smallint status
        text provider
        text provider_msg_id
        timestamptz queued_at
        timestamptz sent_at
    }
    templates {
        text template_id PK
        int version PK
        smallint channel PK
        text locale PK
        text body
    }
    user_devices {
        uuid device_id PK
        bigint user_id
        smallint platform
        text push_token
        smallint status
    }
```

<details markdown="1">
<summary><b>Show: the full SQL</b></summary>

```sql
CREATE TABLE notifications (
    notification_id     UUID PRIMARY KEY,
    event_id            UUID NOT NULL,
    recipient_user_id   BIGINT NOT NULL,
    channel             SMALLINT NOT NULL,       -- 1=push, 2=email, 3=sms, 4=inapp
    template_id         VARCHAR(64) NOT NULL,
    template_version    INT NOT NULL,
    status              SMALLINT NOT NULL,       -- 1=queued, 2=sent, 3=failed, 4=dropped
    category            SMALLINT NOT NULL,       -- 1=transactional, 2=social, 3=marketing
    provider            VARCHAR(32),
    provider_msg_id     VARCHAR(128),
    retry_count         SMALLINT NOT NULL DEFAULT 0,
    queued_at           TIMESTAMPTZ NOT NULL,
    sent_at             TIMESTAMPTZ,
    error_code          VARCHAR(64),
    error_message       TEXT
);

CREATE INDEX idx_event_recipient_channel
    ON notifications (event_id, recipient_user_id, channel);
CREATE INDEX idx_recipient_recent
    ON notifications (recipient_user_id, queued_at DESC);

CREATE TABLE user_preferences (
    user_id              BIGINT PRIMARY KEY,
    channels             JSONB NOT NULL,
    categories           JSONB NOT NULL,
    quiet_hours_start    TIME,
    quiet_hours_end      TIME,
    timezone             VARCHAR(64),
    locale               VARCHAR(16),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version              INT NOT NULL DEFAULT 1
);

CREATE TABLE templates (
    template_id     VARCHAR(64) NOT NULL,
    version         INT NOT NULL,
    channel         SMALLINT NOT NULL,
    locale          VARCHAR(16) NOT NULL,
    subject         TEXT,
    body            TEXT NOT NULL,
    variables       JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    deprecated_at   TIMESTAMPTZ,
    PRIMARY KEY (template_id, version, channel, locale)
);

CREATE TABLE user_devices (
    device_id       UUID PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    platform        SMALLINT NOT NULL,    -- 1=ios, 2=android, 3=web
    push_token      VARCHAR(512) NOT NULL,
    status          SMALLINT NOT NULL,    -- 1=active, 2=revoked, 3=invalid
    last_seen_at    TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_user_active ON user_devices (user_id) WHERE status = 1;
```

**Dedup store (Redis, not SQL):**
```
KEY:   dedup:{event_id}:{recipient_user_id}:{channel}
VALUE: notification_id
TTL:   24 hours
```

</details>

Four choices doing real work:

**`notifications` sharded by `notification_id` hash.** Writes spread evenly. The `idx_event_recipient_channel` index makes "did we already send this?" lookups fast within the shard. Redis SETNX is the fast path; this index is the durable backup for reconciliation.

**`templates` are immutable per version.** To fix a template, publish version N+1 and flip the current-version pointer. Roll back by pointing back to N. Messages already sent are stuck (you cannot recall a push), but new sends pick up the fix within minutes.

**`user_devices` filtered by `status = 1`.** Fan-out expands "send to user_456" into active device tokens only. Revoked and invalid tokens never reach the provider.

**`status` is a `SMALLINT`.** Adding new states (`deferred`, `gdpr_deleted`) later is a new integer value, not a schema migration.

---

### 5. The engine: event to delivered push

The timing of one event through the full pipeline:

```
T+0ms      Post Service calls POST /events (event_id=evt_X, recipient=user_456).
T+5ms      Ingest API checks payload. SETNX event_id=evt_X in Redis (24h TTL).
           Appends to events.created Kafka topic.
           Returns 202.

T+10ms     Fan-out consumer reads the event.
T+11ms     Loads user_456's prefs from Redis (~1ms, cache hit).
T+12ms     Checks in order:
            - category=social, prefs.social=on. OK.
            - quiet_hours: user is in PST, time is 14:00. Not in window.
            - per-user cap: counter=3, cap=20. OK.
            - aggregation: no open window for post:789:likes.
              SETEX agg:post:789:likes 3600 1. Schedule close at T+3600s.
            - dedup: SETNX dedup:evt_X:user_456:push. Won.
           Decision: emit to push (iOS + Android) and in-app.
T+14ms     Expand user_456 to active devices: device_A (iOS), device_B (Android).
T+15ms     Emit 2 push tasks and 1 in-app task to channel topics.
T+17ms     INSERT 3 rows into notifications (status=queued).

T+25ms     Push worker pulls task for device_A.
T+26ms     Render template body (~1ms).
T+27ms     POST /3/device/{token_A} via persistent HTTP/2 connection.
T+150ms    APNs returns 200. Worker updates: status=sent, provider_msg_id=apns_xyz.

T+3600s    Aggregation window closes.
           Fan-out reads count: 47 likes.
           Emits one notification: "Alice and 46 others liked your post."
```

P50 end-to-end for a single push: ~150ms. The APNs round trip is the biggest chunk.

---

### 6. The architecture

```mermaid
flowchart TB
    subgraph Edge["Client edge"]
        P([Producer Services]):::user
        GW["API Gateway<br/>auth · rate limit · idempotency"]:::edge
    end

    subgraph Ingest["Ingest path"]
        I["Ingest API<br/>(dedup on event_id)"]:::app
        K{{"Kafka<br/>events.created · 64 partitions"}}:::queue
    end

    subgraph FanOut["Fan-out"]
        F["Fan-out Service<br/>(stateless pods)"]:::app
        Prefs[("Preferences<br/>Redis + Postgres")]:::cache
        Tmpl[("Template Service")]:::db
        DD[("Dedup Store<br/>Redis 24h TTL")]:::cache
        DR[("Device Registry<br/>Postgres by user_id")]:::db
    end

    subgraph Channels["Per-channel queues + workers"]
        KP{{"push"}}:::queue
        KE{{"email"}}:::queue
        KS{{"sms"}}:::queue
        KI{{"in-app"}}:::queue
        PW["Push Workers (~150)"]:::app
        EW["Email Workers (~50)"]:::app
        SW["SMS Workers (~30)"]:::app
        IW["In-app Workers (~80)"]:::app
    end

    subgraph Providers["External providers"]
        APNs["APNs (iOS)"]:::ext
        FCM["FCM (Android)"]:::ext
        SG["SendGrid / SES"]:::ext
        TW["Twilio / SNS"]:::ext
        WS["WebSocket Gateway"]:::app
    end

    DB[("Notifications DB<br/>Postgres · sharded")]:::db
    DLQ{{"Dead-letter topics<br/>dlq.push / dlq.email / ..."}}:::queue
    Camp["Campaign Scheduler"]:::app

    P --> GW --> I
    I --> K
    K --> F
    F --> Prefs
    F --> Tmpl
    F --> DD
    F --> DR
    F --> KP --> PW
    F --> KE --> EW
    F --> KS --> SW
    F --> KI --> IW
    PW --> APNs
    PW --> FCM
    EW --> SG
    SW --> TW
    IW --> WS
    PW --> DB
    EW --> DB
    SW --> DB
    IW --> DB
    PW -.failed.-> DLQ
    EW -.failed.-> DLQ
    SW -.failed.-> DLQ
    Camp --> I

    classDef user fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef edge fill:#e2e8f0,stroke:#475569,color:#1e293b
    classDef app fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
    classDef ext fill:#e9d5ff,stroke:#7e22ce,color:#581c87
```

Five things to notice:

- The Ingest API returns 202 before any fan-out work starts. Producers are never blocked on delivery latency.
- Kafka is partitioned by `recipient_user_id` on `events.created`. All events for one user land on one partition. The per-user cap counter lives on the consumer that owns that partition, so no cross-pod synchronization is needed.
- Fan-out and channel workers split because they have different bottlenecks. Fan-out is IO-bound on Redis lookups. Channel workers are blocked by provider latency.
- Per-channel topics contain blast radius. SendGrid outage backs up only the email topic.
- Campaign Scheduler is separate because marketing needs pacing. Without it, one careless campaign floods Kafka and delays 2FA codes.

---

### 7. A request, end to end

```mermaid
sequenceDiagram
    autonumber
    participant P as Post Service
    participant GW as API Gateway
    participant I as Ingest API
    participant K as Kafka
    participant F as Fan-out
    participant DD as Dedup (Redis)
    participant KP as push topic
    participant W as Push Worker
    participant A as APNs

    P->>GW: POST /events (event_id=evt_8f3a)
    GW->>I: forward (auth ok)
    I->>DD: SETNX event_id=evt_8f3a (24h TTL)
    DD-->>I: set ok, first time seen
    I->>K: append to events.created
    I-->>P: 202 Accepted (~5ms total)

    K->>F: read event (partition by recipient_id)

    rect rgb(241, 245, 249)
        Note over F,DD: fan-out checks, all Redis, ~3ms
        F->>F: load prefs, check category, quiet hours, cap
        F->>DD: SETNX dedup:evt_8f3a:user_456:push
        DD-->>F: set ok
    end

    F->>KP: emit push task (~15ms from event)
    KP->>W: read task
    W->>W: render template (~1ms)
    W->>A: POST /3/device/{token} (HTTP/2)
    A-->>W: 200 OK (~120ms round trip)
    W->>W: UPDATE notifications status=sent
```

Target latencies for the common paths:

| Path | P99 |
|------|-----|
| Ingest to Kafka | ~5ms |
| Fan-out checks (all Redis) | ~3ms |
| Push worker to APNs and back | ~120ms |
| Full end-to-end (happy path push) | ~150ms |
| Email delivery (SendGrid queued) | ~5 seconds |

---

### 8. The scaling journey: 10 users to 1 billion

```mermaid
flowchart LR
    S1["Stage 1<br/>~10k users<br/>1 app + 1 Postgres<br/>~$50/mo"]:::s1
    S2["Stage 2<br/>~1M users<br/>+ Kafka + Prefs svc<br/>+ push worker pool<br/>~$500/mo"]:::s2
    S3["Stage 3<br/>10M-100M users<br/>+ Redis prefs cache<br/>+ per-channel workers<br/>+ Campaign Scheduler<br/>~$3-8k/mo"]:::s3
    S4["Stage 4<br/>1B users<br/>+ multi-region<br/>+ sharded dedup store<br/>+ Flink aggregation"]:::s4

    S1 --> S2 --> S3 --> S4

    classDef s1 fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef s2 fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef s3 fill:#fef3c7,stroke:#a16207,color:#713f12
    classDef s4 fill:#fce7f3,stroke:#be185d,color:#831843
```

#### Stage 1: ~10,000 users

One Postgres, one app instance. Preferences in Postgres. Push only (APNs + FCM). Notifications delivered inline via HTTP call. No Kafka, no retry logic, no dedup.

Enough because you see a few hundred notifications per hour. Building more is over-engineering.

#### Stage 2: ~1 million users

What breaks: the inline APNs call takes 50-200ms per notification. At a few hundred concurrent requests, the Ingest API starts timing out.

Add Kafka to decouple ingest from delivery. Add a push worker pool. Add basic retry logic. ~$500/month. Still synchronous Postgres preference reads; at 1M users this is manageable.

#### Stage 3: 10M to 100M users

Several things break at once:

- Preferences Postgres at 10,000 events/sec: 10,000 synchronous reads per second. Database falls over.
- One marketing campaign targeting 5M users overwhelms the single worker pool. Email backs up behind push.
- Kafka consumer rebalance causes workers to re-process 500 messages. Users get duplicate push notifications.

Fixes in priority order:

1. Redis cache for preferences with pub/sub invalidation on write. Cache hit rate hits 99%.
2. Separate Kafka topics and worker pools per channel. Email and push fail independently.
3. Redis dedup store with SETNX. Consumer rebalance duplicates handled.
4. Campaign Scheduler to pace large campaigns.
5. Aggregation windows, quiet hours, and per-user caps in fan-out.

Cost jumps to $3-8k/month.

#### Stage 4: 1 billion users

New problems at this scale:

- GDPR requires EU user data to stay in EU. The architecture goes multi-region.
- Redis dedup store at 100k writes/sec on one cluster becomes a bottleneck. Shard by `recipient_user_id`.
- Aggregation window state is too large for stateless fan-out workers. Move to Flink or Kafka Streams for declarative window aggregation with built-in state management.
- Marketing campaigns can target 100M+ users. Campaign Scheduler needs segment pre-computation and multi-region fan-out.

Each region has its own Kafka, workers, Postgres, and Redis. Cross-region delivery (a US user notifying an EU user) routed via authenticated cross-region API. The core data model has not changed since Stage 1.

---

### 9. The variants

| Variant | What changes |
|---------|--------------|
| **In-app only (no external providers)** | No retry complexity, no token cleanup, no provider quotas. Fan-out writes directly to an inbox table. WebSocket gateway for online users. |
| **2FA / security SMS** | Exactly-once guarantee required. Use Postgres `INSERT ON CONFLICT` (two-phase commit) instead of Redis SETNX. Accept ~5ms extra latency. |
| **Weekly digest** | Fan-out defers all social notifications to a "digest" bucket. A cron job at Sunday midnight aggregates per user and emits one email. |
| **WhatsApp / LINE channel** | Same channel adapter pattern. New Kafka topic, new worker pool, new provider credentials. ~2 weeks per channel. Nothing else changes. |

---

### 10. Reliability

**Provider outage.** APNs is down for 30 minutes. Push topic backs up. Workers retry with exponential backoff. After 5 attempts (~60 seconds), messages go to dead-letter. When APNs returns, the backlog drains. Two safeguards stop a flood: per-notification TTL drops anything past its expiration, and workers respect APNs's throughput limits on recovery.

**Consumer rebalance.** Kafka consumer groups rebalance when a pod dies or joins. A partition briefly gets processed by two consumers. Redis SETNX serializes them: the loser sees the key set and skips the provider call. For 2FA SMS, use Postgres two-phase commit instead.

**Late-arriving events.** A producer held an event for 10 minutes due to its own outage, then sent it. If `ttl_seconds` was 5 minutes, drop it. If no TTL was set, deliver it. Producers set TTL based on the event's business meaning.

**Preferences write race.** A user rapidly toggles marketing opt-out. The pub/sub invalidation fires multiple times. The preferences cache refetches from Postgres on each invalidation and picks up the latest value. Last write wins. Correct behavior.

---

### 11. Observability

| Metric | Why it matters |
|--------|----------------|
| `notifications.queued_rate` (by channel, category) | Headline throughput |
| `notifications.delivery_rate_pct` (delivered / queued) | Below 95% is a page |
| `fanout.latency_p99` | Fan-out service health |
| `channel.latency_p99` (per provider) | Spots APNs or SendGrid slowdowns early |
| `provider.error_rate` (by code) | Distinguishes transient from permanent failures |
| `dedup.hit_rate` | Near 0% in steady state; a spike means producers are retrying |
| `quiet_hours.deferred_rate` | Too high may mean the system is holding too much |
| `preferences.cache_hit_rate` | Should be above 99% |
| `kafka.consumer_lag_p99` (per topic) | Leading indicator of backup |
| `deadletter.rate` (by channel, reason) | Triggers manual triage |
| `token_invalidation.rate` | A sudden spike may indicate an auth bug |
| `unsubscribe.rate` (by category) | Product signal for notification fatigue |

Page on: any channel's `delivery_rate_pct` below 95% for 5 minutes. Fan-out `latency_p99` above 30 seconds for 5 minutes. Dead-letter rate spike more than 10x baseline.

Ticket on: unsubscribe rate spike (likely a bad campaign). Token invalidation spike (auth bug?). Preferences cache hit rate dropping (eviction tuning needed).

---

### 12. Follow-up answers

**1. Producer retries `event_id=42` twice within 100ms. Later, after TTL expires.**

Within 100ms: the Ingest API checks Redis for the event_id. The first call SETs the key (24h TTL) and proceeds. The second call sees the key and returns 200 immediately. Nothing reaches Kafka twice.

After 25 hours: the dedup key has expired. The second call is treated as a new event and a second notification fires. This is intentional. Twenty-four hours is long enough that a legitimate retry has already landed. After that, the same event_id is more likely an operator manually resending or a bug. For strict deduplication beyond 24 hours, write the event_id to a Postgres `processed_events` table with a unique constraint at ingest time.

**2. Marketing campaign accidentally targets 100M instead of 10M.**

Detection: the volume alert fires when `notifications.queued_rate` for the marketing category spikes beyond the expected band.

Stop at the Campaign Scheduler: `POST /campaigns/{id}/pause`. The scheduler stops emitting events within 1-2 seconds.

Events already on Kafka are not stopped by pausing the scheduler. The fan-out Service checks a campaign blocklist Redis set on every event. If the campaign_id is in the blocklist, drop without fan-out. The operator adds the campaign_id via an admin endpoint, effective within seconds. Channel workers also check the blocklist before calling the provider.

**3. APNs is down for 30 minutes.**

During: push topic backs up. Workers retry with exponential backoff. After 5 attempts (~60 seconds), messages go to dead-letter.

Recovery: APNs returns. Workers resume. A backlog of ~500k messages drains over the next few minutes. Flood prevention: per-notification TTL drops anything stale (a "your driver arrived" push with a 5-minute TTL is discarded if it sat in the queue for 30 minutes). Workers also respect APNs's throughput limits on resumption.

**4. User opts out of marketing while a campaign is mid-fan-out.**

Fan-out checks preferences at the moment it processes the event, not when the event was created. The preferences cache has a short TTL with pub/sub invalidation on write. Opt-out at T=0 invalidates the cache. Fan-out refetches on next event, typically within 1 second.

Events already past fan-out and sitting on per-channel topics are not re-checked. Those are sent. Window of inconsistency: about 10 seconds. For strict legal compliance, also re-check preferences in the channel worker before calling the provider.

**5. User has 5 devices; notification to themselves.**

Fan-out expands "send to user_456" against the device registry filtered by `status = 1` (active). If device_C is revoked and device_E is invalid, they are filtered at expansion time before any provider call. Result: 3 push notifications, one per active device. For in-app: one row in the inbox per user, regardless of device count.

**6. Template bug: renders `{{name}}` literally.**

Detection: support tickets, or a lint pass that checks rendered output against a sample.

Roll back: templates are versioned and immutable. Flip the current-version pointer from v3 to v2. Fan-out picks up the new pointer when the template metadata cache expires (short TTL on the pointer, longer on the body).

Messages already sent cannot be recalled. For severe issues, send a follow-up apology notification. Prevention: every template change goes through lint, a preview render with sample data, a canary to 0.1% of recipients, then full rollout.

**7. One database shard hotter than others.**

Diagnosis steps:

1. Check the shard key. `notifications` sharded by `notification_id` hash. An imbalance on hash-sharded data usually means queries are not using the shard key and are scatter-gathering.
2. Check slow-query logs. `SELECT WHERE recipient_user_id = ?` scatter-gathers all shards when `recipient_user_id` is not the shard key. A secondary index on `recipient_user_id` on each shard handles the common case without re-sharding.
3. Check partition age. If the table is range-partitioned by week, the current week's partition takes all writes.
4. Check for a noisy neighbor on the same physical host.

Most often it is a query-pattern issue, not a sharding problem.

**8. User got an SMS at 4 a.m.**

Trace:

1. Query `notifications` for `recipient_user_id = ?`, `channel = 3` (SMS), and `sent_at` around 4 a.m. Get `notification_id`, `event_id`, `template_id`, `category`.
2. From `event_id`, find the originating event. Who emitted it? What `category`?
3. Check the user's preferences: are quiet hours set? What timezone is stored?
4. Check fan-out logs for this event: did it run the quiet-hours check? What did it decide?

Likely causes: the category was `transactional` and quiet hours were skipped by design; the user's timezone is wrong in the preferences table; or a DST edge case in the quiet-hours evaluation. Reproduce by replaying the event with the same user state against the fan-out logic. The structured fan-out log is what makes diagnosis possible.

**9. Add web push as a new channel.**

What changes: a new channel adapter for the Web Push protocol (VAPID keys, FCM for Chrome, Mozilla autopush for Firefox). A new entry in the `channel` enum. A new template variant per template (web push bodies are short). A subscription endpoint where the browser registers an `{endpoint URL, keys}` object stored in `user_devices` with `platform=3`. A web push toggle in the preferences UI.

What stays the same: Ingest API, Kafka topology, dedup store, preferences service, retry and dead-letter logic, template service. Fan-out emits to a new `notifications.web_push` topic. Adding a channel is a 2-3 week project.

**10. Audit trail for compliance.**

Every notification has a row in `notifications` with `notification_id`, `event_id`, `recipient_user_id`, `channel`, `template_id`, `template_version`, `category`, `status`, `queued_at`, `sent_at`, `provider`, `provider_msg_id`, `error_code`.

For "prove user 12345 received exactly these notifications":

```sql
SELECT *
  FROM notifications
 WHERE recipient_user_id = 12345
   AND sent_at BETWEEN ? AND ?
   AND status = 2
 ORDER BY sent_at;
```

The absence of a row proves the negative: every emitted notification is logged at queue time (status=queued) before any provider call. If status never became `sent`, the row is still there with the reason.

Retention: 30 days hot in Postgres, archived to S3 as Parquet for 7 years. Compliance queries hit Postgres for recent data and Athena for archived. For suppression audit (why a notification was dropped: quiet hours, cap, opt-out?), structured fan-out logs ship to a log warehouse.

---

### 13. Trade-offs worth saying out loud

**Build vs. buy channel adapters.** OneSignal or Braze handle all channels. Faster to launch. But you pay per notification at scale, cannot tune behavior per channel, and have less visibility into provider failures. For a product sending billions, build. For millions, buy and iterate.

**Where to evaluate preferences.** Fan-out is cheaper and runs once per event. Channel worker re-check is more responsive but adds one extra Redis read per send. Fan-out is the right default; add channel-worker re-check only for strict marketing compliance requirements.

**Strict vs. eventual dedup.** Redis SETNX is fast but has a small window where two workers can both win the key. Postgres two-phase commit is durable but adds 5-10ms. Use SETNX for social and marketing notifications; use two-phase commit for 2FA SMS where exactly-once matters.

**Aggregation window shape.** Tumbling windows (fixed duration, no overlap) are simple but can split a burst of likes across two windows and produce two notifications. Rolling windows extend on each event and delay delivery. Most products use tumbling with a 1-hour default and accept the edge case.

**What to revisit at 10x scale.** Move fan-out to Flink or Kafka Streams for declarative window aggregation and built-in state management. Federate the dedup store regionally. Build a shadow-send mode where new channels or templates can be tested against real traffic without calling the provider.

---

### 14. Common mistakes

**One worker pool for all channels.** A single worker handles push, email, and SMS. One bad channel stalls all three. Separate pools, separate topics.

**No idempotency key on producer calls.** A producer retry sends two notifications. Bad.

**Synchronous Postgres preference lookup on the hot path.** At 100k/sec, that is 100k Postgres reads per second. Cache in Redis with pub/sub invalidation.

**No aggregation.** A user with 100 likes on a post gets 100 separate notifications. A real product complaint within a day of launch.

**No quiet hours or per-user cap.** Required for any product with more than a few hundred users.

**Treating push token revocation as manual cleanup.** APNs and FCM tell you when a token is dead. Build the feedback loop on day one, or your account-wide delivery rate degrades silently.

**No dead-letter strategy.** Failed messages pile up or retry forever. Both outcomes are bad.

**Same retry policy across all channels.** Push with a 24-hour retry window delivers a "your driver arrived" push two hours late. Email with a 60-second window drops messages that could have delivered after a 1-hour SendGrid outage.

**Forgetting marketing is different from transactional.** Marketing needs pacing, a Campaign Scheduler, separate quiet-hours behavior, and compliance handling (CAN-SPAM, GDPR, TCPA). A uniform pipeline blurs them.

**No mention of compliance.** GDPR for EU, TCPA for SMS in the US, CAN-SPAM for email. The architecture must accommodate these from the start.

The three that separate strong from average answers: separate per-channel queues, dedup keys at both the ingest and fan-out layers, and preferences as a cached read. Hit those three and you are doing well.
