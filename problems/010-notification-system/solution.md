## Solution: Notification System

### The short version

A notification system is a pipeline. One event comes in. The right messages go out. In between, four stages: ingest the event, decide what to send (fan-out), route per channel, deliver through an external provider.

The pipeline runs on Kafka. Each stage is its own service so each one can scale on its own bottleneck. Push, email, SMS, and in-app each get their own queue and their own worker pool. That way, if SendGrid goes down, push keeps flowing.

The interesting work is at the edges. **Idempotency keys** (`event_id + recipient + channel`) stop duplicate sends across retries and Kafka rebalances. **Aggregation windows** turn 100 likes into one "John and 99 others" notification. **Quiet hours** and **per-user caps** stop the system from waking people up or training them to disable notifications. **Push token cleanup** feeds back into the device registry so dead tokens get pruned within minutes.

What trips candidates: one worker pool for all channels (one bad provider drags down everyone), preferences as a sync database call on the hot path (cannot scale to 100,000 per second), and retries without dedup keys producing duplicate sends.

---

### 1. The clarifying questions, ranked

Covered in `question.md`. Ranked by how much they change the architecture:

1. **User preferences.** Biggest impact. Without preferences, fan-out is "for each recipient, send." With preferences, it becomes "for each recipient, decide channels, check time of day, check the hourly cap, check if we already sent this." Different system.
2. **Channels.** Decides how many provider adapters you build and how independent they need to be.
3. **Fan-out shape.** A 10 billion per day system with even spread is easier than one with viral spikes. The marketing-campaign worst case decides queue sizing.
4. **Idempotency.** A "no" answer means you must handle producer retries safely, which means dedup keys.

If you walked in asking only about throughput, you wrote the right architecture for the wrong problem.

---

### 2. The math, in plain numbers

| Scale | Notifications per day | Per second sustained | Peak | Storage (30 days) |
|-------|-----------------------|----------------------|------|-------------------|
| Small startup | 1 million | ~12 | ~50 | 3.6 GB |
| Mid-size | 100 million | ~1,200 | ~5,000 | 360 GB |
| Billion-user product | 10 billion | ~116,000 | ~350,000 | 36 TB |

Per-channel split at the billion-user scale:

| Channel | Share | Sustained QPS |
|---------|-------|---------------|
| Push    | 60%   | ~70,000/sec  |
| In-app  | 30%   | ~35,000/sec  |
| Email   | 7%    | ~8,000/sec   |
| SMS     | 3%    | ~3,500/sec   |

Worst burst: a 10-million-recipient marketing campaign in 5 minutes adds 33,000 per second for the duration. A ~30% spike on top of the steady 116,000.

Worker pool sizing at 500 calls per second per worker: ~230 workers sustained, ~700 at peak.

The real limit is not your code. It is the **per-account quotas** at the providers. SendGrid throttles per sub-account. Twilio throttles per phone number. APNs throttles per HTTP/2 stream per certificate. To run at scale, you hold many provider credentials and load-balance across them.

---

### 3. The API

Two endpoints carry the whole product. **Submit an event** and **read/write preferences**. Everything else is reading data back.

#### Producer API (event submission)

```
POST /api/v1/events
Content-Type: application/json
Authorization: Bearer <producer-service-token>
Idempotency-Key: <event_id>

{
  "event_id": "evt_8f3a91...",
  "event_type": "post.liked",
  "category": "social",                  // transactional | social | marketing
  "actor": { "user_id": 123 },
  "recipients": [
    { "user_id": 456, "context": { "post_id": 789 } }
  ],
  "aggregation_key": "post:789:likes",  // optional
  "template_id": "tpl_like_v3",
  "template_vars": { "actor_name": "Alice", "post_title": "..." },
  "ttl_seconds": 3600                   // drop if not delivered in this window
}
```

Responses:

| Status | Meaning |
|--------|---------|
| 202 Accepted | Event queued |
| 200 OK | Same event_id already accepted (idempotent replay) |
| 400 Bad Request | Invalid payload, unknown template, etc. |
| 401 Unauthorized | Bad producer token |
| 413 Payload Too Large | Recipient list too long for one request (>10k) |
| 429 Too Many Requests | Producer rate limit |

A few small but load-bearing choices:

- **`Idempotency-Key` is required.** It is what makes producer retries safe. The Ingest API uses it to dedup before the message ever hits Kafka.
- **`recipients` is a list.** For 1-to-1 notifications it has one entry. For fan-out events it can have up to 10,000. Above that, use a bulk endpoint that takes a recipient query (like "all users in segment X") and expands on the server.
- **`category` controls policy.** It decides whether quiet hours apply, whether per-user caps apply, and how aggressively the system retries.
- **`ttl_seconds` is a hard deadline.** Push notifications with a 1-hour TTL that miss the window are dropped, not delivered late. This is what stops the "flood at recovery" problem when a provider comes back.

> **Why does category change retry behavior?** Because a 2-hour-old marketing push is annoying but harmless. A 2-hour-old "your driver arrived" push is wrong. Different categories deserve different deadlines.

#### Bulk campaign API

```
POST /api/v1/campaigns
{
  "campaign_id": "cmp_winter_sale_2026",
  "segment_query": { "country": "US", "engagement_tier": "active" },
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

Returns 202 with a `campaign_id`. The Campaign Service expands the segment query into a recipient stream and feeds it into the Fan-out path at the configured rate.

#### Preferences API (user-facing)

```
GET /api/v1/users/me/notification-preferences
PUT /api/v1/users/me/notification-preferences
{
  "channels": {
    "push":  { "enabled": true },
    "email": { "enabled": true },
    "sms":   { "enabled": false },
    "in_app":{ "enabled": true }
  },
  "categories": {
    "marketing":     { "enabled": false },
    "social":        { "enabled": true, "channels": ["push", "in_app"] },
    "transactional": { "enabled": true }    // always true, cannot disable
  },
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end":   "07:00",
    "timezone": "America/Los_Angeles"
  }
}
```

Writes go to the Preferences Service primary. Reads come from a regional cache. The cache is invalidated on every write via pub/sub.

---

### 4. The data model

Five tables. Each one does one job.

#### Notifications (the delivery log)

```sql
CREATE TABLE notifications (
    notification_id     UUID PRIMARY KEY,
    event_id            UUID NOT NULL,
    recipient_user_id   BIGINT NOT NULL,
    channel             SMALLINT NOT NULL,        -- 1=push, 2=email, 3=sms, 4=inapp
    template_id         VARCHAR(64) NOT NULL,
    template_version    INT NOT NULL,
    status              SMALLINT NOT NULL,        -- 1=queued, 2=sent, 3=failed, 4=dropped
    category            SMALLINT NOT NULL,        -- 1=transactional, 2=social, 3=marketing
    provider            VARCHAR(32),              -- 'apns', 'fcm', 'sendgrid', 'twilio'
    provider_msg_id     VARCHAR(128),             -- their message ID for tracing
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
CREATE INDEX idx_status_queued
    ON notifications (status, queued_at) WHERE status IN (1, 3);
```

Sharded by `notification_id` hash. The `idx_event_recipient_channel` index makes "did we already send this?" lookups fast on the row store. The Redis dedup store is the fast path. This index is the durable backup.

#### Preferences

```sql
CREATE TABLE user_preferences (
    user_id              BIGINT PRIMARY KEY,
    channels             JSONB NOT NULL,         -- {"push": true, "email": true, ...}
    categories           JSONB NOT NULL,         -- per-category settings
    quiet_hours_start    TIME,
    quiet_hours_end      TIME,
    timezone             VARCHAR(64),            -- IANA tz name
    locale               VARCHAR(16),            -- en_US, ja_JP
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version              INT NOT NULL DEFAULT 1  -- optimistic concurrency
);
```

Sharded by `user_id`. Cached in Redis as JSON, keyed by user_id, with a 5-minute TTL and pub/sub invalidation on write. Cache hit rate is north of 99% because preferences change rarely.

#### Templates

```sql
CREATE TABLE templates (
    template_id          VARCHAR(64) NOT NULL,
    version              INT NOT NULL,
    channel              SMALLINT NOT NULL,
    locale               VARCHAR(16) NOT NULL,
    subject              TEXT,                   -- email only
    body                 TEXT NOT NULL,          -- mustache-style
    variables            JSONB NOT NULL,         -- expected variable schema
    ab_variant           VARCHAR(16),            -- 'A', 'B', null=no test
    created_at           TIMESTAMPTZ NOT NULL,
    created_by           BIGINT NOT NULL,
    deprecated_at        TIMESTAMPTZ,
    PRIMARY KEY (template_id, version, channel, locale, ab_variant)
);
```

Templates are versioned and never edited in place. To "edit" a template, publish version N+1. The Fan-out Service resolves `template_id` at fan-out time to the current version, so you roll forward by flipping the current-version pointer. Roll back is the same operation in reverse.

> **Why versioning?** Because a bug in a template that gets edited in place affects every notification using it instantly. A versioned template means the bug is in v3, you ship v4 with the fix, and you flip the pointer. Old v3 messages already sent are stuck (you cannot un-send a push). But v4 takes effect within a minute.

#### Dedup store (Redis)

```
KEY:    dedup:{event_id}:{recipient_user_id}:{channel}
VALUE:  notification_id (so you can find the original)
TTL:    24 hours
```

#### Device tokens

```sql
CREATE TABLE user_devices (
    device_id            UUID PRIMARY KEY,
    user_id              BIGINT NOT NULL,
    platform             SMALLINT NOT NULL,      -- 1=ios, 2=android, 3=web
    push_token           VARCHAR(512) NOT NULL,
    status               SMALLINT NOT NULL,      -- 1=active, 2=revoked, 3=invalid
    last_seen_at         TIMESTAMPTZ NOT NULL,
    app_version          VARCHAR(32)
);
CREATE INDEX idx_user_active ON user_devices (user_id) WHERE status = 1;
```

Sharded by `user_id`. The Fan-out Service uses this to turn "send to user 456" into "send to device tokens T1, T2, T3 for user 456."

---

### 5. Core algorithm: event to delivered notification

Here is the path of one event from producer to delivered push on a user's phone, with timing.

```
T+0ms     Producer calls POST /events with event_id=evt_X, recipient=user_456.
T+5ms     Ingest API checks payload, checks idempotency on event_id in Redis.
          Not seen before. Inserts event_id into idempotency cache (TTL 24h).
          Appends to events.created Kafka topic, partition = hash(recipient_user_id).
          Returns 202.

T+10ms    Fan-out Service consumer reads the message.
T+11ms    Loads user_456's preferences from Redis (cache hit, ~1ms).
T+12ms    Loads template tpl_like_v3 metadata from Redis (cache hit).
T+13ms    Checks:
          - category=social, preferences.categories.social.enabled=true. OK.
          - quiet_hours: user is in PST, now is 14:00 PST. Not in quiet window.
          - per-user-cap: notifications.hourly.user_456.push counter = 3. Cap is 20. OK.
          - aggregation: aggregation_key=post:789:likes. Check agg:post:789:likes.
            Window does not exist. Start window: SETEX 3600 1.
            Schedule a delayed "agg-close" message for T+3600s.
          - dedup: not relevant yet (only one event so far).

          Decision: emit to push and in-app channels (per user_456's channel prefs).

T+15ms    Look up user_456's active devices: [device_A (ios), device_B (android)].
T+16ms    Emit two messages to notifications.push topic:
          - {notification_id: n1, device: device_A, token: t1, ...}
          - {notification_id: n2, device: device_B, token: t2, ...}
          Emit one message to notifications.in_app.

T+18ms    INSERT 3 rows into notifications table with status=queued.

T+25ms    Push worker (one of the 230 in the pool) pulls n1.
T+26ms    Check dedup_key in Redis: not present. SETNX dedup_key with value=n1.
T+27ms    Render template body with template_vars. ~1ms.
T+28ms    Call APNs HTTP/2 stream: POST /3/device/{token}.
T+150ms   APNs returns 200. Worker updates notifications row: status=sent,
          provider_msg_id=apns_xyz, sent_at=now.
T+155ms   Worker fires a "delivered" event to Kafka (for analytics).

T+150ms   Push worker for n2 similarly completes.

T+30ms    In-app worker writes to the WebSocket gateway. If user is online,
          message appears in their feed within 100ms. If offline, stored in
          inbox; appears on next app open.
```

Total time from event submission to delivered push: ~150ms P50. The biggest chunk is the round trip to APNs.

What changes for the aggregated case: at T+3600s the close message fires. Fan-out reads the count from `agg:post:789:likes` (say 47), reads the actor list materialized during the window, and emits a single notification with template `tpl_like_aggregated_v1` and `template_vars={actor_name: "Alice", others_count: 46}`. From there, the path is the same as a regular notification.

---

### 6. The full architecture

```
   +----------------------------------------------------------------------+
   |  Producer Services                                                   |
   |  PostService, BillingService, MarketingTool, OnboardingService, ... |
   +----------------------------------+-----------------------------------+
                                      |
                                      v  POST /events
                            +----------------------+
                            |   Ingest API         |  Stateless, N pods.
                            |   (REST + gRPC)      |  Auth, schema check,
                            |                      |  dedup on event_id.
                            +----------+-----------+
                                       |
                                       v
                            +----------------------+
                            |   events.created     |  Kafka topic.
                            |   topic              |  Partitioned by
                            |   ~64 partitions     |  recipient_user_id.
                            +----------+-----------+
                                       |
                                       v
   +----------------------------------------------------------------------+
   |  Fan-out Service (consumer pool)                                     |
   |                                                                      |
   |  For each event message:                                             |
   |   1. Load preferences (Redis cache).                                 |
   |   2. Load template metadata.                                         |
   |   3. Check category, quiet hours, per-user cap.                      |
   |   4. Aggregation window check.                                       |
   |   5. Expand recipient to devices (push) and contact (email/sms).     |
   |   6. Dedup_key set (SETNX) per (event_id, recipient, channel).       |
   |   7. INSERT notification rows (status=queued).                       |
   |   8. Emit per-channel tasks.                                         |
   +----+-----------------+-----------------+-----------------+-----------+
        |                 |                 |                 |
        v                 v                 v                 v
   +--------+        +--------+        +--------+        +----------+
   | push   |        | email  |        |  sms   |        | in_app   |  Per-channel
   | topic  |        | topic  |        | topic  |        | topic    |  Kafka topics.
   +---+----+        +---+----+        +---+----+        +----+-----+
       |                 |                 |                  |
       v                 v                 v                  v
   +--------+        +--------+        +--------+        +----------+
   | Push   |        | Email  |        |  SMS   |        | In-App   |  Per-channel
   | Wkrs   |        | Wkrs   |        |  Wkrs  |        |  Wkrs    |  worker pools.
   | ~150   |        | ~50    |        |  ~30   |        |  ~80     |  Each stateless.
   +---+----+        +---+----+        +---+----+        +----+-----+
       |                 |                 |                  |
       |                 |                 |                  v
       |                 |                 |           +--------------+
       |                 |                 |           |  WebSocket   |
       |                 |                 |           |  Gateway     |
       |                 |                 |           +--------------+
       |                 |                 v
       |                 |           +--------------+
       |                 |           |  Twilio /    |
       |                 |           |  SNS SMS     |
       |                 |           +--------------+
       |                 v
       |           +--------------+
       |           |  SendGrid /  |
       |           |  SES         |
       |           +--------------+
       v
   +----------------------+
   |  APNs (iOS) +        |
   |  FCM (Android, web)  |
   +----------------------+

   Supporting services:

   +----------------------+    +----------------------+
   |  Preferences Service |    |  Template Service    |
   |  Postgres + Redis    |    |  Postgres + S3 +     |
   |                      |    |  Redis               |
   +----------------------+    +----------------------+

   +----------------------+    +----------------------+
   |  Device Registry     |    |  Dedup Store         |
   |  Postgres sharded    |    |  Redis cluster       |
   |  by user_id          |    |  with 24h TTL        |
   +----------------------+    +----------------------+

   +----------------------+    +----------------------+
   |  Notifications DB    |    |  Deadletter Topics   |
   |  Postgres sharded    |    |  notifications.dlq.* |
   |  by notification_id  |    |  Manual review       |
   +----------------------+    +----------------------+

   +----------------------+
   |  Campaign Scheduler  |  For bulk/marketing.
   |  (segments -> events)|  Reads segment queries,
   |                      |  paces emission into the
   |                      |  events topic.
   +----------------------+
```

A few things worth pointing at:

- **The Ingest API sits in front of Kafka** so the producer is never slowed by downstream. A producer call returns 202 in under 10ms regardless of fan-out load.

- **Kafka is partitioned by `recipient_user_id`.** All events for one user land on one partition. Useful for per-user ordering and for the per-user cap counter (a single consumer maintains the counter for users in its partition, no cross-pod fighting).

- **Fan-out and channel workers are split** because they have different bottlenecks. Fan-out is light on CPU, heavy on IO (preferences, template, dedup lookups). Channel workers are heavy on IO and blocked by the provider. Splitting them lets each scale on its own constraint.

- **Per-channel topics matter for blast radius.** If SendGrid is down, the email topic backs up. Push and SMS keep flowing. One channel's outage cannot become an everything outage.

- **Redis carries the hot reads.** Preferences, dedup, aggregation counters, per-user caps. All small, all hot, all need under 2ms reads. Postgres is the source of truth and the rebuild path.

- **The Campaign Scheduler is separate** because marketing campaigns need pacing. You do not fire 10 million push notifications in 10 seconds. The scheduler reads a segment query, expands it into events, and feeds them into the ingest API at a controlled rate.

> **Why so many separate services?** Could you put it all in one big service? Yes. Until one bug brings down notifications across the company. Or one bursty workload starves the others. The separation costs a little operational complexity, but it gives you blast-radius limits for free.

---

### 7. Channel adapters

Each channel is wrapped in an adapter that hides the provider's specifics. The Fan-out Service does not know that "push to iOS" means "APNs HTTP/2 with a JWT." It just emits to the `push` topic. The push worker is the only thing that talks to APNs and FCM.

#### Push (APNs and FCM)

The Push adapter holds a long-lived HTTP/2 connection per certificate to APNs (for iOS) and to FCM (for Android and web). It POSTs to `/3/device/{token}` with headers including `apns-topic`, `apns-push-type`, `apns-priority` (10 for urgent, 5 otherwise), and `apns-expiration` (the TTL deadline). The body holds the alert (title, body), badge, and sound. The response is classified as SUCCESS (200), TOKEN_INVALID (410, device unregistered), TRANSIENT (429 or 5xx), or PERMANENT (other 4xx).

Three things to know:

- **APNs cares about connection setup overhead.** One worker holds many open streams on one HTTP/2 connection. Tearing down and rebuilding on errors is expensive, so the adapter is careful about when to recycle.

- **The `apns-expiration` header is how we avoid the "flood at recovery" problem.** If the message is older than the TTL, APNs drops it server-side instead of delivering a stale push.

- **Token invalidation feeds back through the system.** A 410 from APNs means the token is dead. The worker writes to `push_token_invalidations`. A small consumer marks the row in `user_devices` as `status=invalid`. Future notifications skip that token.

FCM is similar but uses HTTP/1.1 or HTTP/2 with OAuth bearer tokens and a different invalid-token signal (`NotRegistered`). Same pattern.

#### Email (SendGrid / SES)

The Email adapter calls SendGrid's v3 API, batching up to 1,000 personalizations per request. Each call sets `template_id`, `dynamic_template_data` (the user-specific variables), and `asm.group_id` (the unsubscribe group). A few things to know:

- There is a choice between provider templates (SendGrid stores them) and in-house templates (you render and send full HTML). For simple emails the provider templates save bandwidth and give you a drag-drop editor. For complex emails it is easier to render in-house.
- `unsubscribe_group_id` is SendGrid's compliance feature and is required for marketing emails. Each category (newsletter, promotional, etc.) gets its own group. Users can unsubscribe per group instead of from all email.
- Hard bounces flow back via webhook. SendGrid notifies you when an email hard-bounces. A small consumer marks the email address as `status=invalid`. Fan-out then skips invalid addresses.

#### SMS (Twilio / SNS)

The SMS adapter calls Twilio's `/Messages.json` with a `MessagingServiceSid` (not a single `From` number), a `To` phone, the `Body`, and a `StatusCallback` URL. A few SMS-specific things:

- **Always use Messaging Services, not single From numbers.** Twilio routes across a pool of sender numbers, respecting per-number throughput. Without this you would manually shard recipients across numbers.
- **Twilio responds 201 when it accepts the message, but actual carrier delivery takes seconds to minutes.** They post status updates via webhook (`queued` -> `sent` -> `delivered` or `failed`). A small webhook consumer updates the notifications row.
- **Twilio error 21211 means invalid number.** Mark the phone in the user profile. Never retry.
- **US carriers throttle hard.** Going over the limit causes `Filtering` errors. For high-volume senders, you also need 10DLC registration with the carriers.

#### In-app

In-app is two parts: save the message to the user's inbox table, then try to push it to a live WebSocket if the user is online. WebSocket push is fire-and-forget. If it fails, the next app open reads from the inbox store, so nothing is lost. It is the simplest channel because there is no external provider with its own SLA. The only failure mode is the WebSocket gateway being down, and you tolerate that because the inbox is the source of truth.

---

### 8. Scaling

#### Kafka partitions

- **`events.created`:** 64 partitions, partitioned by `recipient_user_id`. Why by recipient? It co-locates all events for one user on one consumer, which makes the per-user cap counter easy (no cross-partition contention).
- **Per-channel topics:** 32 partitions each, partitioned by `notification_id` (random). Why random? You do not need ordering at this stage, and random partitioning gives the best load spread across workers.

#### Worker pools

- **Push:** ~150 workers sustained. Each holds APNs and FCM connections. Auto-scale on consumer lag.
- **Email:** ~50 workers. Batch sends (up to 1,000 per SendGrid request), so per-worker throughput is high.
- **SMS:** ~30 workers. Throughput is limited by Twilio's per-Messaging-Service limits, not worker CPU. Adding workers does not help once you hit the provider cap.
- **In-app:** ~80 workers. CPU-bound on serialization. The WebSocket gateway is a separate scaling concern.

Scaling triggers: consumer lag > 30 seconds for push, > 5 minutes for email. The auto-scaler reacts to these by spinning new pods.

#### Hot recipient: campaign-style fan-out

A campaign targeting 10 million users emits 10 million events. Each event has one recipient. They land on partitions by `recipient_user_id`, so they spread well. Fan-out workers process them in parallel. No single consumer is overloaded.

The problem is downstream: 10 million push-token lookups in a 5-minute window stress the device registry. Fix: the campaign expander batches lookups (`SELECT * FROM user_devices WHERE user_id = ANY(...)` with 1,000 IDs at a time) and prewarms a cache.

The other problem is provider rate limits. 10 million emails in 5 minutes is 33,000 emails per second. SendGrid throttles a single sub-account at that rate. Fix: hold multiple SendGrid sub-accounts and round-robin across them. Or use the Campaign Scheduler's `rate_limit` config to pace the campaign within the provider's quota.

#### Hot recipient: popular user case

If user 456 follows 1,000 high-volume accounts, they could get thousands of notifications per hour. The per-user cap stops the spam, but the cap counter itself becomes a hot Redis key. Every event for that user does `INCR notifications:hourly:user_456:push`. At hundreds of writes per second on one key, Redis handles it, but it is worth knowing the limit.

Mitigation if it becomes a problem: shard the counter across N sub-counters and sum at read time. For 99% of users (who get fewer than 100 notifications per hour) the single counter is fine.

#### Database growth

Notifications table grows by 1.2 TB per day. With 30-day retention that is 36 TB across 64 shards, around 600 GB per shard. Manageable, but partition by month or week within each shard so you can drop old data with `DROP PARTITION` instead of `DELETE` (which would bloat the table).

For long-term audit (some rules require 7 years), archive to S3 as compressed Parquet after 30 days. The audit-query path reads from S3 via Athena, not from the live database.

---

### 9. Reliability

#### Retries (covered in question.md Step 6)

Short version: transient failures retry with exponential backoff. Permanent failures go to deadletter or auto-cleanup. Idempotency keys stop duplicate sends across retries and consumer rebalances.

#### Provider outage

APNs goes down. The push channel topic backs up. Push workers retry with backoff. After exhausting retries, they emit to deadletter. When APNs returns, the backlog drains over the next 5 to 30 minutes.

What we do **not** do: dump the entire backlog at APNs the instant it returns. Two safeguards:

1. **Per-notification TTL.** Notifications older than their TTL are dropped, not sent. A 30-minute outage means notifications queued in the first 30 minutes are dropped if their TTL was 30 minutes.
2. **Worker rate-limiting.** Workers respect APNs's published throughput and our own backpressure. They do not flood on recovery.

#### Late-arriving events

A producer sits on an event for 10 minutes due to its own outage, then sends it. Do we deliver?

Two cases:

- **Has TTL.** Honor the TTL. If TTL is 10 minutes and the event is 10 minutes old, drop.
- **No TTL.** Send. Some events (your invoice for last month) really should be delivered even if they are slightly stale.

Producers are expected to set TTL based on event meaning.

#### Consumer rebalance duplicates

Kafka consumer groups rebalance when a consumer dies or joins. During rebalance, a partition can briefly be processed by two consumers. Without dedup keys, this causes duplicate sends.

The dedup key in Redis (`SETNX dedup:event_id:recipient:channel`) handles this. The losing consumer sees the key already set and skips the provider call.

There is still a window. Between `SETNX success` and `provider call complete`, if the worker dies, the consumer that replays sees the key set and skips. The notifications row is in `status=queued`, but the actual provider call may or may not have happened. A real edge case.

Mitigation for mission-critical channels:

- Set dedup_key with worker_id and timestamp. On replay, check if the original worker is still alive (heartbeat in Redis with TTL). If dead, the replay can proceed.
- Or use a two-phase commit. Mark the row as `status=sending` before the provider call, `status=sent` after. On replay, only retry if `status=queued` or `status=sending` and elapsed time > N seconds.

For social and marketing categories, at-most-once with a tiny duplicate risk is fine. For 2FA SMS, you want exactly-once with stronger guarantees, which costs latency. Pick per category.

#### Deadletter triage

Messages in deadletter are categorized:

- **Invalid recipient (~90%).** Auto-handled. Token invalidated, email marked bounced, phone marked invalid. No human action.
- **Template rendering failure (rare).** A bug in the template made the renderer fail. Page the on-call.
- **Provider permanent rejection (5-10%).** Content flagged, sender blacklisted. Surface to product owner. May indicate abuse or compliance issue.

Deadletter messages older than 7 days are archived to S3 and dropped from the active queue.

---

### 10. Observability

| Metric | Why it matters |
|--------|-----|
| `notifications.queued_rate` (by channel, category) | Headline throughput |
| `notifications.delivered_rate` (by channel) | Headline delivery |
| `notifications.delivery_rate_pct` (delivered / queued) | Lower bound on health |
| `fanout.latency_p99` (event arrival -> first channel emit) | Fan-out service health |
| `channel.latency_p99` (channel emit -> provider response) | Per-channel provider health |
| `provider.error_rate` (by provider, error code) | Spot APNs / SendGrid / Twilio issues early |
| `dedup.hit_rate` | Should be ~0% normally. Spike means producer is retrying often |
| `aggregation.windows_open` | Sanity check on aggregation correctness |
| `quiet_hours.deferred_rate` | If too high, the system may be holding too much |
| `preferences.cache_hit_rate` | Should be >99% |
| `kafka.consumer_lag_p99` (per topic) | Leading indicator |
| `deadletter.rate` (by channel, reason) | Manual triage trigger |
| `token_invalidation.rate` | Sudden spike may be a token-revocation bug |
| `unsubscribe.rate` (by category) | Product signal. Correlates with notification fatigue |

Alerts:

- **Page on:** any channel's delivery_rate_pct drops below 95% for 5 minutes. Fan-out latency P99 > 30s for 5 min. Deadletter rate spike > 10x baseline.
- **Ticket on:** unsubscribe rate spike (likely a bad campaign). Token invalidation rate spike (auth bug?). Preferences cache hit rate dropping (cache eviction tuning).

Per-user audit: every notification has a row in the `notifications` table with `event_id`, decisions taken (which channels selected and why others were skipped), and provider message ID. Compliance can answer "what did user 12345 receive on 2026-03-15?" with one query.

---

### 11. Follow-up answers

**1. Producer retries `event_id=42` twice within 100ms. Later, after dedup TTL expires.**

Within 100ms: the Ingest API hashes `event_id` and checks Redis. The first call SETs the key with a 24-hour TTL and gets a `1`. The second call sees the key already set and returns 200 with the original `queued_at`. No downstream impact. The event is processed once.

After 25 hours: the dedup key has expired. The second call is treated as a new event and a second notification fires. This is intentional. 24 hours is long enough that a legitimate retry has already happened. After that, "retry" is more likely an operator manually resending or a system bug producing the same event_id twice for different intents.

If you need protection beyond 24 hours: write the event_id into a Postgres `processed_events` table at ingest time with a unique constraint. INSERT-OR-IGNORE pattern. Costs more (one DB write per event) but is durable across Redis evictions.

**2. Marketing campaign accidentally targets 100M instead of 10M.**

Detection: the operator notices, or the volume alerts fire (notifications.queued_rate spike for the marketing category).

Stopping it: the Campaign Scheduler exposes `POST /campaigns/{id}/pause`. This writes a `paused=true` flag to the campaign state. The scheduler stops emitting new events for that campaign within 1-2 seconds.

But events already in `events.created` Kafka are not stopped. To kill those:

- Fan-out Service checks a "campaign blocklist" Redis set on every event. If the event's `campaign_id` is in the blocklist, drop the event without fan-out.
- Operator adds the campaign_id to the blocklist via an admin endpoint. Takes effect within seconds.

Per-channel topics still have queued messages from the bad campaign. Same trick: channel workers check the blocklist before calling the provider. Drop if blocked.

Cleanup state: campaign_id stays in blocklist for 24 hours, then expires. Notifications already sent are in the notifications table for audit. Nothing to roll back.

The deeper lesson: a circuit breaker per campaign is a cheap insurance policy. Build it on day one.

**3. APNs is down for 30 minutes.**

During: push channel topic backs up. Push workers retry. After max retries, messages go to deadletter. We do not actively drain APNs. Backoff naturally throttles our retries.

Recovery: APNs returns. Workers resume. A backlog of (say) 500K messages drains over the next ~5 minutes at our throughput.

Flood prevention: per-message TTL drops anything older than its expiration. A "your driver has arrived" push with a 5-minute TTL is dropped if it sat in the queue for 30 minutes. Marketing pushes typically have 1-hour TTLs and may also be dropped.

What users see: they get the notifications that are still valid, and they miss the ones that are not. No flood. We deliberately drop stale ones.

**4. User opts out of marketing while a campaign is in flight.**

Fan-out checks preferences at the moment it processes the event, not at the moment the event was created. The Preferences cache has a 5-minute TTL with pub/sub invalidation on write.

So: user opts out at T=0. Pub/sub invalidates the cache. Fan-out refetches on next event. Within ~1 second of the opt-out, the new preference is honored.

Events already past the Fan-out stage (sitting on per-channel topics) do not re-check preferences. Those are sent. Window of inconsistency: about 10 seconds typically.

For strict compliance (legally must not send marketing after opt-out), re-check preferences in the channel worker too. Adds about 1ms per send. Worth it for marketing. Not necessary for transactional.

**5. User has 5 devices; notification triggered for themselves.**

Fan-out expands "send to user_456" to all active devices for that user: device_A (ios, active), device_B (android, active), device_C (ios, signed out, status=revoked, skip), device_D (web push, active), device_E (ios, old token, status=invalid, skip).

Result: 3 push notifications, one per active device. Signed-out and invalid-token devices are filtered at expansion time.

The user gets 3 push notifications. Correct. They have 3 active devices they might be looking at. Whether to also dedup within a user (one push per user, picking the most recently active device) is a product decision. Most products send to all active devices and let the user mute on devices they do not want.

In-app: one entry in the inbox, regardless of devices. Inbox is per-user, not per-device.

**6. Template has a bug. It renders literal `{{name}}` instead of the user's name.**

Detection: support tickets, or an automated template QA check that catches unrendered variables.

Roll back: templates are versioned and immutable. Flip the "current version" pointer for `tpl_X` from v3 to v2. Fan-out picks up the new pointer on next template metadata read (cache TTL is short for the pointer, longer for the body).

Messages already sent: they are sent. You cannot un-send a push or email. For very severe issues (PII leaked in the body, say), send a follow-up notification apologizing. Mostly you accept the damage and move on.

Prevention: every template change goes through:

- **Lint:** check that template_vars provided match the variables referenced in the body.
- **Preview:** render a sample with test data and have a human review.
- **Canary:** send to 0.1% of recipients first. If error rate spikes (rendering failures, unsubscribes), pause.
- **Full rollout.**

A senior candidate mentions all four.

**7. One shard hotter than others.**

Diagnosis steps:

1. **Check the shard key.** `notifications` is sharded by `notification_id` hash, which is random. An imbalanced shard suggests something other than shard key. Maybe one shard has more long-running queries pending.
2. **Check the query pattern.** If a query like `SELECT FROM notifications WHERE recipient_user_id = ?` runs and recipient_user_id is not the shard key, the query scatter-gathers across all shards. One slow shard means the whole query is slow, and the slow shard looks "hot" from outside.
3. **Check partition skew within the shard.** Notifications grow over time. If you partition by month within shard, the latest month is hottest. Expected and fine.
4. **Check noisy neighbor.** Shared infrastructure? The shard may be hot because another tenant is hogging the host.
5. **Resolve.** If shard key is bad (e.g., accidentally sharded by `user_id` instead of `notification_id`), rebalance via consistent hashing. If it is query pattern, add a secondary index or a denormalized table. If it is noisy neighbor, move the shard.

Most often it is a query-pattern issue. A senior candidate asks for the slow-query log before reaching for re-sharding.

**8. User got an SMS at 4am.**

Trace path:

1. Find the notification row in DB: `SELECT FROM notifications WHERE recipient_user_id = ? AND sent_at BETWEEN ? AND ? AND channel = 3`. Returns `notification_id`, `event_id`, `template_id`, `category`, `sent_at`.
2. From `event_id`, find the originating event. Who emitted it? What category?
3. Check user's preferences: are quiet hours set? What timezone?
4. Check the Fan-out Service log for this event. Did it check quiet hours? What did it decide?

Likely causes:

- **Transactional category.** Quiet hours do not apply to transactional. If the SMS was a 2FA code, by design.
- **Timezone wrong.** User's timezone is UTC in your records but they are actually in PST. You sent at "10am UTC" which is 3am PST. Fix: refresh user timezone on app open.
- **Bug in quiet hours evaluation.** Off-by-one or DST issue. Reproduce by replaying the same event with the same user state.

The audit log (which Fan-out decisions were made for this notification) is what makes diagnosis possible. Without it, you cannot answer this question.

**9. Add web push as a new channel.**

What changes:

- New channel adapter for Web Push protocol (uses VAPID keys, talks to FCM for Chrome, Mozilla autopush for Firefox, etc.). New entry in the `channel` enum.
- New template variant per template (web push bodies are short, no rich content).
- Subscription endpoint: browser subscribes to push and gets a subscription object (endpoint URL + keys). We store this in `user_devices` with `platform=3 (web)`.
- Preferences UI adds the web push toggle.

What stays the same:

- Ingest API: no change. Producers emit events. Fan-out decides whether to add web push as a channel based on the user having an active web subscription.
- Kafka topology: add `notifications.web_push` topic, but the pattern is identical.
- Templates, dedup, preferences, retries, deadletter: all reused.

Adding a channel is a 2-3 week project once. The architecture pays back this design choice every time.

**10. Audit trail for compliance.**

Every notification has a row in `notifications` with:

- `notification_id`, `event_id`, `recipient_user_id`, `channel`, `template_id`, `template_version`, `category`.
- `status` (queued / sent / failed / dropped).
- `queued_at`, `sent_at`.
- `provider`, `provider_msg_id` (the provider's own ID, traceable on their dashboard).
- `error_code`, `error_message` if it failed.

For "prove user 12345 received exactly these notifications":

```sql
SELECT * FROM notifications
WHERE recipient_user_id = 12345
  AND sent_at BETWEEN ? AND ?
  AND status = 2  -- sent
ORDER BY sent_at;
```

That is the source-of-truth answer. To prove the negative ("user did not receive notification X"), the absence of a row is sufficient because every emitted notification is logged at queue time, before any provider call.

Retention: 30 days hot in Postgres, archived to S3 as Parquet for 7 years. Compliance queries hit Postgres for recent data and Athena for archived.

For deeper audit (which Fan-out decisions were made, like "we considered SMS but skipped because user opted out"), structured logs from Fan-out are shipped to a log warehouse. Cheaper than putting every decision into Postgres. Sufficient for the rare compliance audit.

---

### 12. Trade-offs worth saying out loud

**Build vs buy channel adapters.** You could use a single service like OneSignal or Braze to handle all channels. Pro: less code, faster to launch. Con: you pay per notification, you cannot tune behavior per channel, and you have less visibility into provider issues. For a product sending billions, build. For a product sending millions, buy.

**Aggregation aggressiveness.** Aggregate too much and users miss real-time events. Aggregate too little and they get spammed. The right answer is product-specific and depends on user segmentation. Some products tier: power users see real-time, casual users get hourly digests.

**Where to put preferences evaluation.** Earlier (in Fan-out) is cheaper but less responsive to preference changes. Later (in channel worker) is more responsive but more expensive. Pick Fan-out for the steady state and accept the ~10-second window where stale preferences may still send.

**Strict vs eventual dedup.** SETNX in Redis is fast but has a tiny window where two workers can both win. Two-phase commit with Postgres is durable but adds 5-10ms. SETNX for social and marketing. Two-phase for transactional and 2FA.

**Per-user cap vs per-recipient cap.** A user can have many recipients (multiple devices, multiple emails). Do we cap per user (one limit across all their addresses) or per recipient (each device its own limit)? Per user. Otherwise a user with 5 devices effectively gets 5x the cap.

**What you would revisit at 10x scale:**

- Move the Fan-out Service to a streaming framework (Flink or Kafka Streams) instead of bare Kafka consumers. Lets you express aggregation windows declaratively and get state management for free.
- Federate the dedup store regionally. At 100K/sec writes, one global Redis becomes the bottleneck. Shard by recipient_user_id with strong consistency within shard, eventually consistent across shards.
- Build a "shadow send" mode where new channels or new templates can be tested against real traffic without actually delivering. The shadow path writes to a log instead of calling the provider. Lets you validate behavior before rolling out.

---

### 13. Common mistakes

Most weak answers fall into one of these:

- **One worker pool for all channels.** "We have a worker that handles push, email, and SMS." Wrong. One bad channel drags down the others. Separate pools, separate Kafka topics.

- **No idempotency key on producer calls.** "Producers just call our API." Then a producer retry sends two notifications. Bad.

- **Synchronous preference lookup from a database.** "We look up preferences for each event from Postgres." At 100K/sec, that is 100K Postgres reads per second, which melts the database. Use a Redis cache with pub/sub invalidation.

- **No mention of aggregation.** A user with 100 likes on a post gets 100 notifications. Real product complaint within a day of launch.

- **No quiet hours, no per-user cap.** Both are required for any product that has more than a handful of users.

- **Treating push token revocation as a manual cleanup.** APNs and FCM tell you when a token is dead. Build the feedback loop on day one.

- **No deadletter strategy.** "Failed messages just retry forever." Either you flood the provider or messages pile up forever. Deadletter + categorization + auto-cleanup is required.

- **Ignoring channel-specific SLAs.** Treating push and email with the same retry policy means you either over-retry push (delivers a "your driver arrived" 2 hours late) or under-retry email (drops messages that could have been delivered after the provider's 1-hour outage).

- **Forgetting that marketing is different.** Transactional and marketing have different latency, different scale, different compliance. A single uniform pipeline blurs them and creates problems.

- **No mention of compliance.** GDPR for EU, TCPA for SMS in US, CAN-SPAM for email. Each has hard rules (unsubscribe links required in marketing email, opt-in proof for SMS). The architecture must accommodate them, not bolt them on later.

If you hit 8 of these 10, you are interviewing well. Most candidates miss aggregation, quiet hours, and push token cleanup. The three that separate strong from average answers: separate per-channel queues, dedup keys on the producer path, and preferences as a cached read.
