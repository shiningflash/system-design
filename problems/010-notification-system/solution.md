## Solution: Design a Notification System

### TL;DR

A notification system is a fan-out pipeline where one event becomes zero, one, or many delivered messages depending on preferences, channels, and aggregation rules. The architecture is a Kafka-backed pipeline with four stages: ingest, fan-out, per-channel routing, and provider delivery. Each stage scales independently, and each external provider (APNs, FCM, SendGrid, Twilio) is wrapped in its own worker pool with channel-specific retry semantics.

The interesting engineering is at the edges. Idempotency keys (`event_id + recipient + channel`) prevent duplicate sends across retries and consumer rebalances. Aggregation windows turn 100 likes into one "John and 99 others" notification. Quiet hours plus per-user caps stop the system from waking people up or training them to disable notifications. Push token cleanup feeds back into a device registry so invalid tokens are pruned within minutes.

The mistakes that break candidates are: building a single worker pool for all channels (which means one bad provider drags down the rest), treating preferences as a sync RPC on the hot path (which can not scale to 100K/sec), and forgetting that retries without dedup keys produce duplicate sends.

### 1. Clarifying questions and why each matters

Covered in `question.md`. The most important questions, ranked:

1. **User preferences.** This is the question that changes the architecture. Without preferences, fan-out is "for each recipient, send"; with preferences, it is "for each recipient, decide channels, decide if now is OK, decide if this is the 21st notification this hour."
2. **Channels.** Determines how many provider adapters you build and how independent they need to be.
3. **Fan-out shape.** A 10B/day system with even distribution is much easier than one with viral spikes. The marketing-campaign worst case sets the queue sizing.
4. **Idempotency.** A "no" here means you must handle producer retries gracefully, which means dedup keys.

If you walked in asking only about throughput, you wrote the right architecture for the wrong problem.

### 2. Capacity estimates (full working)

From `question.md`:

- 10B notifications/day across 1B users → ~116K/sec sustained, ~350K/sec peak.
- Per channel: push 70K/sec, in-app 35K/sec, email 8K/sec, SMS 3.5K/sec.
- Worst-case burst: 10M-recipient campaign in 5 minutes = +33K/sec for the duration.
- Storage for delivery records at ~120B each, 30-day retention: ~36TB. Sharded by notification_id.
- Worker pool sizing at 500 calls/sec/worker: ~230 workers sustained, ~700 peak.

The total throughput is not the limiting factor. The limits come from external providers' per-account quotas: SendGrid sub-account throughput, Twilio per-number rate, APNs HTTP/2 stream caps per certificate. You operate at scale by holding many provider credentials and load-balancing across them.

### 3. API design

#### Producer API (the event ingestion side)

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

| Status | Meaning | Body |
|--------|---------|------|
| 202 Accepted | Event queued | `{ "event_id": "...", "queued_at": "..." }` |
| 200 OK | Same event_id already accepted (idempotent replay) | same shape as 202 |
| 400 Bad Request | Invalid payload, unknown template, etc. | `{ "error": "invalid_template" }` |
| 401 Unauthorized | Bad producer token | |
| 413 Payload Too Large | Recipient list too long for one request (>10k) | `{ "error": "use_bulk_endpoint" }` |
| 429 Too Many Requests | Producer rate limit | `{ "error": "rate_limited", "retry_after": 30 }` |

Notes:

- **`Idempotency-Key`** is required. It is what makes producer retries safe. The Ingest API uses it to dedup before the message hits Kafka.
- **`recipients`** is a list. For one-to-one notifications it has one entry; for fan-out events it can have up to 10k. Above that, use a bulk endpoint that takes a recipient query (e.g., "all users in segment X") and expands on the server.
- **`category`** is the single most important field for downstream policy. It controls whether quiet hours apply, whether per-user caps apply, and how aggressively the system retries.
- **`ttl_seconds`** is a hard deadline. Push notifications with a 1-hour TTL that miss the window are dropped, not delivered late. This is what prevents the "flood at recovery" problem when a provider comes back from an outage.

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
    "push": { "enabled": true },
    "email": { "enabled": true },
    "sms": { "enabled": false },
    "in_app": { "enabled": true }
  },
  "categories": {
    "marketing": { "enabled": false },
    "social": { "enabled": true, "channels": ["push", "in_app"] },
    "transactional": { "enabled": true }    // always true, cannot disable
  },
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "07:00",
    "timezone": "America/Los_Angeles"
  }
}
```

Writes go to the Preferences Service primary; reads are served from a regional cache with eventual consistency. The cache is invalidated on every preference write via pub/sub.

### 4. Data model

#### Notifications (delivery log)

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

Sharded by `notification_id` hash. The `idx_event_recipient_channel` index is what makes "did we already send this?" lookups O(log N) on the row store. The Redis dedup store is the fast path; this index is the durable backup.

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

Sharded by `user_id`. Cached in Redis as JSON, keyed by user_id, with a 5-minute TTL and pub/sub invalidation on write. Cache hit rate runs north of 99% because preferences change rarely.

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

Templates are versioned and immutable per version. To "edit" a template you publish version N+1. The Fan-out Service resolves `template_id` at fan-out time to whichever version is current; this means you can roll forward by flipping the current version pointer. Roll back is just flipping the pointer to the previous version.

Localization: the resolution order is `(template_id, version, channel, user.locale)` with fallback to `en_US`. A/B variants: the Fan-out Service hashes `user_id mod 100` to pick variant.

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

Sharded by `user_id`. The Fan-out Service hits this to expand "send to user 456" into "send to device tokens T1, T2, T3 for user 456".

### 5. Core algorithm: event to delivered notification

Here is the path of one event from producer to delivered push notification on a user's phone.

```
T+0ms     Producer calls POST /events with event_id=evt_X, recipient=user_456.
T+5ms     Ingest API validates schema, checks idempotency on event_id via Redis.
          Not seen before. Inserts event_id into idempotency cache (TTL 24h).
          Appends to events.created Kafka topic, partition = hash(recipient_user_id).
          Returns 202.

T+10ms    Fan-out Service consumer reads the message.
T+11ms    Loads user_456's preferences from Redis (cache hit, ~1ms).
T+12ms    Loads template tpl_like_v3 metadata from Redis (cache hit).
T+13ms    Evaluates:
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

T+30ms    In-app worker writes to the WebSocket gateway. If user is online, message
          appears in their feed within 100ms. If offline, stored in inbox; appears on
          next app open.
```

Total elapsed from event submission to delivered push: ~150ms P50, dominated by the APNs round-trip.

What changes for the aggregated case: at T+3600s the agg-close message fires. The Fan-out Service reads the count from `agg:post:789:likes` (say 47), reads the actor list materialized during the window, and emits a single notification with template `tpl_like_aggregated_v1` and `template_vars={actor_name: "Alice", others_count: 46}`. The path from there is the same as a regular notification.

### 6. Architecture (detailed)

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  Producer Services                                                 │
   │  PostService, BillingService, MarketingTool, OnboardingService, ...│
   └────────────────────────────────┬───────────────────────────────────┘
                                    │
                                    ▼  POST /events
                          ┌──────────────────────┐
                          │   Ingest API         │  Stateless, N pods.
                          │   (REST + gRPC)      │  Auth, schema validation,
                          │                      │  idempotency on event_id.
                          └──────────┬───────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │   events.created     │  Kafka topic.
                          │   topic              │  Partitioned by recipient_user_id.
                          │   ~64 partitions     │  7-day retention.
                          └──────────┬───────────┘
                                     │
                                     ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │  Fan-out Service (consumer pool)                                    │
   │                                                                     │
   │  For each event message:                                            │
   │   1. Load preferences (Redis cache).                                │
   │   2. Load template metadata.                                        │
   │   3. Evaluate category, quiet hours, per-user cap.                  │
   │   4. Aggregation window check.                                      │
   │   5. Expand recipient to devices (push) and contact (email/sms).    │
   │   6. Dedup_key set (SETNX) per (event_id, recipient, channel).      │
   │   7. INSERT notification rows (status=queued).                      │
   │   8. Emit per-channel tasks.                                        │
   └────┬───────────────┬───────────────┬───────────────┬────────────────┘
        │               │               │               │
        ▼               ▼               ▼               ▼
   ┌────────┐      ┌────────┐      ┌────────┐      ┌──────────┐
   │ push   │      │ email  │      │  sms   │      │ in_app   │  Per-channel
   │ topic  │      │ topic  │      │ topic  │      │ topic    │  Kafka topics.
   └───┬────┘      └───┬────┘      └───┬────┘      └────┬─────┘
       │               │               │                │
       ▼               ▼               ▼                ▼
   ┌────────┐      ┌────────┐      ┌────────┐      ┌──────────┐
   │ Push   │      │ Email  │      │  SMS   │      │ In-App   │  Per-channel
   │ Wkrs   │      │ Wkrs   │      │  Wkrs  │      │  Wkrs    │  worker pools.
   │ ~150   │      │ ~50    │      │  ~30   │      │  ~80     │  Each stateless.
   └───┬────┘      └───┬────┘      └───┬────┘      └────┬─────┘
       │               │               │                │
       │               │               │                ▼
       │               │               │         ┌──────────────┐
       │               │               │         │  WebSocket   │  Persistent
       │               │               │         │  Gateway     │  connections to
       │               │               │         │              │  online clients.
       │               │               │         └──────────────┘
       │               │               │
       │               │               ▼
       │               │         ┌──────────────┐
       │               │         │  Twilio /    │
       │               │         │  SNS SMS     │
       │               │         └──────────────┘
       │               │
       │               ▼
       │         ┌──────────────┐
       │         │  SendGrid /  │
       │         │  SES         │
       │         └──────────────┘
       │
       ▼
   ┌──────────────────────┐
   │  APNs (iOS) +        │
   │  FCM (Android, web)  │
   └──────────────────────┘

   Supporting services:

   ┌──────────────────────┐    ┌──────────────────────┐
   │  Preferences Service │    │  Template Service    │
   │  Postgres + Redis    │    │  Postgres + S3 +     │
   │                      │    │  Redis               │
   └──────────────────────┘    └──────────────────────┘

   ┌──────────────────────┐    ┌──────────────────────┐
   │  Device Registry     │    │  Dedup Store         │
   │  Postgres sharded    │    │  Redis cluster       │
   │  by user_id          │    │  with 24h TTL        │
   └──────────────────────┘    └──────────────────────┘

   ┌──────────────────────┐    ┌──────────────────────┐
   │  Notifications DB    │    │  Deadletter Topics   │
   │  Postgres sharded    │    │  notifications.dlq.* │
   │  by notification_id  │    │  Manual review       │
   └──────────────────────┘    └──────────────────────┘

   ┌──────────────────────┐
   │  Campaign Scheduler  │  For bulk/marketing.
   │  (segments → events) │  Reads segment queries,
   │                      │  paces emission into the
   │                      │  events topic.
   └──────────────────────┘
```

Why each piece sits where it does:

- **Ingest API in front of Kafka.** The producer must not be slowed by downstream. A producer call returns 202 in under 10ms regardless of fan-out load.
- **Kafka partitioned by `recipient_user_id`.** Means all events for one user land on one partition. Useful for per-user ordering and for the per-user cap counter (single consumer maintains the counter for users in its partition, no contention).
- **Fan-out Service separate from channel workers.** Fan-out is CPU-light, IO-heavy (preferences, template, dedup lookups). Channel workers are IO-heavy and provider-blocked. Separating them lets each scale on its own bottleneck.
- **Per-channel topics.** If SendGrid is down, the email topic backs up; push and SMS keep flowing. One channel's outage cannot become an everything outage.
- **Redis for hot reads.** Preferences, dedup, aggregation counters, per-user caps. All small, all hot, all need <2ms reads. Postgres is the source of truth and the rebuild path.
- **Separate Campaign Scheduler.** Marketing campaigns are paced (don't fire 10M push notifications in 10 seconds). The scheduler reads a segment query, expands it into events, and feeds them into the ingest API at a controlled rate.

### 7. Channel adapters

Each channel is wrapped in an adapter that hides the provider's specifics. The Fan-out Service does not know that "push to iOS" means "APNs HTTP/2 with a JWT". It just emits to the `push` topic. The push worker is the only thing that talks to APNs and FCM.

#### Push (APNs and FCM)

```python
class PushAdapter:
    def send(self, task):
        if task.platform == 'ios':
            return self._send_apns(task)
        elif task.platform in ('android', 'web'):
            return self._send_fcm(task)
    
    def _send_apns(self, task):
        # APNs uses HTTP/2 with a long-lived connection per certificate.
        # One connection multiplexes many requests.
        response = self.apns_conn.post(
            path=f"/3/device/{task.token}",
            headers={
                'apns-topic': self.bundle_id,
                'apns-push-type': 'alert',
                'apns-priority': '10' if task.urgent else '5',
                'apns-expiration': str(task.deadline_epoch),
            },
            json={
                'aps': {
                    'alert': {'title': task.title, 'body': task.body},
                    'badge': task.badge,
                    'sound': task.sound,
                }
            }
        )
        return self._classify_apns_response(response)
    
    def _classify_apns_response(self, response):
        if response.status == 200:
            return Result.SUCCESS
        if response.status == 410:
            return Result.TOKEN_INVALID  # device unregistered
        if response.status == 429:
            return Result.TRANSIENT  # back off
        if response.status >= 500:
            return Result.TRANSIENT
        return Result.PERMANENT
```

Key points:

- **HTTP/2 connection reuse.** APNs is sensitive to connection setup overhead. One worker holds many concurrent streams on one connection. Tear-and-rebuild on errors is expensive.
- **Per-token TTL via `apns-expiration`.** If the message is older than the TTL, APNs drops it server-side. This is how we avoid the "flood at recovery" problem.
- **Token invalidation feedback.** A 410 from APNs means the token is dead. The worker writes to `push_token_invalidations` topic; a small consumer marks the row in `user_devices` as `status=invalid`.

FCM is similar but uses HTTP/1.1 or HTTP/2 with different auth (OAuth bearer token) and a different invalid-token signal (`NotRegistered`). Same pattern though.

#### Email (SendGrid / SES)

```python
class EmailAdapter:
    def send(self, task):
        # SendGrid v3 API. Batches up to 1000 personalizations per request.
        response = self.sendgrid.post('/v3/mail/send', json={
            'from': {'email': task.from_email, 'name': task.from_name},
            'personalizations': [{
                'to': [{'email': task.to_email}],
                'subject': task.subject,
                'dynamic_template_data': task.template_vars,
            }],
            'template_id': task.provider_template_id,
            'tracking_settings': {
                'click_tracking': {'enable': True},
                'open_tracking': {'enable': True},
            },
            'headers': {'X-Notification-ID': task.notification_id},
            'asm': {'group_id': task.unsubscribe_group_id},
        })
        return self._classify_sendgrid_response(response)
```

Key points:

- **Provider templates vs in-house templates.** SendGrid lets you store templates on their side. For simple emails this saves bandwidth and gets you their drag-drop editor. For complex emails it is easier to render in-house and send the full HTML body.
- **Unsubscribe group ID.** SendGrid's compliance feature. Required for marketing emails. Each category (newsletter, promotional, etc.) has its own group; users can unsubscribe per group instead of from all email.
- **Hard bounces.** SendGrid notifies via webhook when an email hard-bounces. A small consumer reads these webhooks and marks the email address as `status=invalid` in the user profile. The Fan-out Service skips invalid addresses.

#### SMS (Twilio / SNS)

```python
class SMSAdapter:
    def send(self, task):
        # Twilio Messaging Services API. Routes through a sender pool
        # (long codes + short codes + toll-free) with per-carrier optimization.
        response = self.twilio.post('/Messages.json', data={
            'MessagingServiceSid': self.messaging_service_sid,
            'To': task.to_phone,
            'Body': task.body,
            'StatusCallback': self.callback_url,
        })
        return self._classify_twilio_response(response)
```

Key points:

- **Messaging Services, not single From numbers.** Twilio's Messaging Services route across a pool of sender numbers, respecting per-number throughput. Without this you would manually shard recipients across numbers.
- **Async status callbacks.** Twilio responds 201 when it accepts the message, but actual carrier delivery takes seconds to minutes. They post status updates via webhook (`queued` → `sent` → `delivered` or `failed`). A small webhook consumer updates the notifications row.
- **Invalid number handling.** Twilio error 21211 means invalid number. Mark the phone in user profile, never retry.
- **Per-carrier throughput.** US carriers throttle messaging. Going over the limit causes `Filtering` errors. The Messaging Service handles much of this; for high-volume senders you also need 10DLC registration with the carriers.

#### In-app

```python
class InAppAdapter:
    def send(self, task):
        # Two-part: persist to inbox, push to live WebSocket if connected.
        self.inbox_store.insert(InboxItem(
            user_id=task.user_id,
            notification_id=task.notification_id,
            payload=task.payload,
            created_at=now(),
        ))
        # WebSocket fan-out (fire-and-forget; failure to push is OK because
        # the next app open reads from inbox_store).
        self.ws_gateway.try_push(task.user_id, task.payload)
        return Result.SUCCESS
```

In-app is the simplest channel because there is no external provider with its own SLA. The inbox is your own database; the WebSocket gateway is your own service. The only failure mode is the WebSocket gateway being down, and we tolerate that because the inbox is the source of truth.

### 8. Scaling

#### a. Kafka partitions

- `events.created`: 64 partitions, partitioned by `recipient_user_id`. Why by recipient: it co-locates all events for one user on one consumer, which makes the per-user cap counter easy (no cross-partition contention).
- Per-channel topics: 32 partitions each, partitioned by `notification_id` (which is random). Why random: we don't need any ordering at this stage, and random partitioning gives the best load distribution across workers.

#### b. Worker pools

- Push: ~150 workers sustained. Each holds APNs and FCM connections. Auto-scale on consumer lag.
- Email: ~50 workers. Batch sends (up to 1000 per SendGrid request) so per-worker throughput is high.
- SMS: ~30 workers. Throughput is limited by Twilio's per-Messaging-Service limits, not worker CPU. Adding workers does not help once you hit the provider cap.
- In-app: ~80 workers. CPU-bound on serialization; the WebSocket gateway is a separate scaling concern.

Scaling triggers: consumer lag > 30 seconds for push, > 5 minutes for email. The auto-scaler reacts to these by spinning new pods.

#### c. Hot recipient (campaign-style fan-out)

A campaign targeting 10M users emits 10M events. Each event has one recipient. They land on partitions by `recipient_user_id`, so they distribute well. The Fan-out Service workers process them in parallel; no single consumer is overloaded.

The problem is downstream: 10M push tokens lookups in a 5-minute window stress the device registry. Solution: the campaign expander batches lookups (`SELECT * FROM user_devices WHERE user_id = ANY(...)` with 1000 IDs at a time) and prewarms a cache.

The other problem is provider rate limits. 10M emails in 5 minutes = 33K emails/sec. SendGrid will throttle a single sub-account at that rate. Solution: hold multiple SendGrid sub-accounts, round-robin across them. Or use the Campaign Scheduler's `rate_limit` config to pace the campaign to fit in the provider's quota.

#### d. Hot recipient (popular user case)

If user 456 follows 1000 high-volume accounts, they could receive 1000s of notifications/hour. The per-user cap stops the spam, but the cap counter itself becomes a hot Redis key: every event for that user does `INCR notifications:hourly:user_456:push`. At 100s of writes/sec on one key, Redis handles it, but it is worth knowing the limit.

Mitigation if it ever becomes a problem: shard the counter across N sub-counters and sum at read time. For 99% of users (who get < 100 notifications/hour) the single counter is fine.

#### e. Database growth

Notifications table grows by 1.2TB/day. With 30-day retention that is 36TB sharded across 64 shards = ~600GB per shard. Manageable, but partition by month or week within each shard so you can drop old data with `DROP PARTITION` instead of `DELETE` (which would bloat the table).

For long-term audit (some compliance regimes require 7 years), archive to S3 as compressed Parquet after 30 days. The audit-query path reads from S3 via Athena, not from the live database.

### 9. Reliability

#### Retries (already covered in question.md Step 5)

The summary: transient failures retry with exponential backoff; permanent failures go to deadletter or auto-cleanup. Idempotency keys prevent duplicate sends across retries and consumer rebalances.

#### Provider outage

APNs goes down. The push channel topic backs up. Push workers retry with backoff; after exhausting retries they emit to deadletter. When APNs returns, the backlog drains over the next 5-30 minutes.

What we do not do: dump the entire backlog at APNs the instant it returns. Two safeguards:

1. **Per-notification TTL.** Notifications older than their TTL are dropped, not sent. A 30-minute outage means notifications queued in the first 30 minutes are dropped if their TTL was 30 minutes.
2. **Worker rate-limiting.** Workers respect APNs's published throughput and our own backpressure. They do not flood on recovery.

#### Late-arriving events

A producer sits on an event for 10 minutes due to its own outage, then sends it. Should we deliver?

Two cases:

- **Has TTL.** Honor the TTL. If TTL is 10 minutes and the event is 10 minutes old, drop.
- **No TTL.** Send. Some events (your invoice for last month) genuinely should be delivered even if they are slightly stale.

Producers are expected to set TTL based on event semantics.

#### Consumer rebalance duplicates

Kafka consumer groups rebalance when a consumer dies or joins. During rebalance, a partition can be processed by two consumers briefly. Without dedup keys, this causes duplicate sends.

The dedup key in Redis (`SETNX dedup:event_id:recipient:channel`) handles this. The losing consumer sees the key already set and skips the provider call.

The remaining window: between `SETNX success` and `provider call complete`, if the worker dies, the consumer that replays sees the key set and skips. The notifications row is in `status=queued`, but the actual provider call may or may not have happened. This is a real edge case.

Mitigation for mission-critical channels:
- Set dedup_key with worker_id and timestamp. On replay, check if the original worker is still alive (heartbeat in Redis with TTL). If dead, the replay can proceed.
- Or use a two-phase commit: mark the row as `status=sending` before the provider call, `status=sent` after. On replay, only retry if `status=queued` or `status=sending` and elapsed time > N seconds.

For social and marketing categories, the at-most-once guarantee with a tiny duplicate risk is fine. For 2FA SMS, you want exactly-once with stronger guarantees, which costs latency. Pick per category.

#### Deadletter triage

Messages in deadletter are categorized:

- **Invalid recipient (90% of deadletters).** Auto-handled: token invalidated, email marked bounced, phone marked invalid. No human action.
- **Template rendering failure (rare).** A bug in the template caused the renderer to fail. Page the on-call.
- **Provider permanent rejection (5-10%).** Content flagged, sender blacklisted. Surface to product owner; may indicate abuse or compliance issue.

Deadletter messages older than 7 days are archived to S3 and dropped from the active queue.

### 10. Observability

The metrics you must have from day one:

| Metric | Why |
|--------|-----|
| `notifications.queued_rate` (by channel, category) | Headline throughput |
| `notifications.delivered_rate` (by channel) | Headline delivery |
| `notifications.delivery_rate_pct` (delivered / queued) | Lower bound on health |
| `fanout.latency_p99` (event arrival → first channel emit) | Fan-out service health |
| `channel.latency_p99` (channel emit → provider response) | Per-channel provider health |
| `provider.error_rate` (by provider, error code) | Spot APNs / SendGrid / Twilio issues early |
| `dedup.hit_rate` | Should be ~0% normally; spike means producer is retrying often |
| `aggregation.windows_open` | Sanity on aggregation correctness |
| `quiet_hours.deferred_rate` | If too high, the system might be holding too much |
| `preferences.cache_hit_rate` | Should be >99% |
| `kafka.consumer_lag_p99` (per topic) | Leading indicator |
| `deadletter.rate` (by channel, reason) | Manual triage trigger |
| `token_invalidation.rate` | If suddenly spikes, may be a token-revocation bug |
| `unsubscribe.rate` (by category) | Product signal; correlates with notification fatigue |

Alerts:

- Page on: any channel's delivery_rate_pct drops below 95% for 5 minutes; fan-out latency P99 > 30s for 5 min; deadletter rate spike > 10x baseline.
- Ticket on: unsubscribe rate spike (likely a bad campaign); token invalidation rate spike (auth bug?); preferences cache hit rate dropping (cache eviction tuning).

Per-user audit log: every notification has a row in the `notifications` table with `event_id`, decisions taken (which channels selected and why others were skipped), and provider message ID. Compliance can answer "what did user 12345 receive on 2026-03-15?" with one query.

### 11. Follow-up answers

**1. Producer retries `event_id=42` twice within 100ms. Later, after dedup TTL expires.**

Within 100ms: the Ingest API hashes `event_id` and checks Redis. The first call SETs the key with a 24-hour TTL and gets a `1`. The second call sees the key already set and returns 200 with the original `queued_at`. No downstream impact; the event is processed once.

After 25 hours: the dedup key has expired. The second call is treated as a new event, and a second notification fires. This is intentional: 24 hours is long enough that a legitimate retry has already happened. After 24 hours, "retry" is more likely "operator manually re-sent" or "system bug producing the same event_id twice for different intents."

If you need protection beyond 24 hours: write the event_id into a Postgres `processed_events` table at ingest time with a unique constraint. INSERT-OR-IGNORE pattern. Costs more (one DB write per event) but is durable across Redis evictions.

**2. Marketing campaign accidentally targets 100M instead of 10M.**

Detection: operator notices, or the volume alerts fire (notifications.queued_rate spike for the marketing category).

Stopping it: the Campaign Scheduler exposes `POST /campaigns/{id}/pause`. This writes a `paused=true` flag to the campaign state. The scheduler stops emitting new events for that campaign within 1-2 seconds.

But the events already in `events.created` Kafka are not stopped. To kill those:

- Fan-out Service checks a "campaign blocklist" Redis set on every event. If the event's `campaign_id` is in the blocklist, drop the event without fan-out.
- Operator adds the campaign_id to the blocklist via an admin endpoint. Takes effect within seconds.

Per-channel topics still have queued messages from the bad campaign. Same trick: channel workers check the blocklist before calling the provider. Drop if blocked.

Cleanup state: campaign_id stays in blocklist for 24 hours, then expires. Notifications already sent are in the notifications table for audit; nothing to roll back.

The deeper lesson: a circuit breaker per campaign is a cheap insurance policy. Build it on day one.

**3. APNs is down for 30 minutes.**

During: push channel topic backs up. Push workers retry; after max retries, messages go to deadletter. We do not actively drain APNs; we let backoff naturally throttle our retries.

Recovery: APNs returns. Workers resume. Backlog of (say) 500K messages drains over the next ~5 minutes at our throughput.

Flood prevention: the per-message TTL drops anything older than its expiration. A "your driver has arrived" push with a 5-minute TTL is dropped if it sat in the queue for 30 minutes. Marketing pushes typically have 1-hour TTLs and may also be dropped.

What users see: they get the notifications that are still valid, and they miss the ones that aren't. They do not get a flood; we deliberately drop the stale ones.

**4. User opts out of marketing while a campaign is in flight.**

The Fan-out Service evaluates preferences at the moment it processes the event, not at the moment the event was created. The Preferences cache has a 5-minute TTL with pub/sub invalidation on write.

So: user opts out at T=0. Pub/sub invalidates the cache. Fan-out Service refetches on next event. Within ~1 second of the opt-out, the new preference is honored.

Events already past the Fan-out stage (sitting on per-channel topics) do not re-check preferences. Those are sent. Window of inconsistency: ~10 seconds typically.

For strict compliance (legally must not send marketing after opt-out), you could re-check preferences in the channel worker too. This adds a Preferences cache lookup per send. Worth the extra ~1ms for marketing; not necessary for transactional.

**5. User has 5 devices; notification triggered for themselves.**

The Fan-out Service expands "send to user_456" to all active devices for that user: device_A (ios, active), device_B (android, active), device_C (ios, signed out, status=revoked, skip), device_D (web push, active), device_E (ios, old token, status=invalid, skip).

Result: 3 push notifications, one per active device. Signed-out and invalid-token devices are filtered at expansion time.

The user gets 3 push notifications. This is correct: they have 3 active devices they might be looking at. Whether to also dedup within a user (one push per user, picking the most recently active device) is a product decision; most products send to all active devices and let the user mute on devices they don't want.

In-app: one entry in the inbox, regardless of devices. Inbox is per-user, not per-device.

**6. Template has a bug; rendering literal `{{name}}` instead of user's name.**

Detection: support tickets, or an automated template QA check that catches unrendered variables.

Roll back: templates are versioned and immutable. To roll back, flip the "current version" pointer for `tpl_X` from v3 to v2. The Fan-out Service picks up the new pointer on next template metadata read (cache TTL is short for the pointer, longer for the body).

Messages already sent: they are sent. Cannot un-send a push or email. For very severe issues (PII leaked in the body, say), you can send a follow-up notification apologizing. Mostly you accept the damage and move on.

Prevention: every template change goes through:
- Lint: check that template_vars provided match the variables referenced in the body.
- Preview: render a sample with test data and have a human review.
- Canary: send to 0.1% of recipients first; if error rate spikes (rendering failures, unsubscribes), pause.
- Full rollout.

A senior candidate mentions all four.

**7. One shard hotter than others.**

Diagnosis steps:

1. **Check the shard key.** `notifications` is sharded by `notification_id` hash, which is random. So an imbalanced shard suggests something other than shard key. Maybe one shard has more long-running queries pending.
2. **Check the query pattern.** If you have a query like `SELECT FROM notifications WHERE recipient_user_id = ?` and recipient_user_id is not the shard key, the query scatter-gathers across all shards. One shard slow = whole query slow, and the slow shard looks "hot" from outside.
3. **Check partition skew within the shard.** Notifications grow over time; if you partition by month within shard, the latest month is hottest. This is expected and fine.
4. **Check noisy neighbor.** Shared infrastructure? The shard may be hot because another tenant is hogging the host.
5. **Resolve.** If shard key is bad (e.g., you accidentally sharded by `user_id` instead of `notification_id`), rebalance via consistent hashing. If it is query pattern, add a secondary index or a denormalized table. If it is noisy neighbor, move the shard.

Most often it is a query-pattern issue. A senior candidate asks for the slow-query log before reaching for re-sharding.

**8. User got an SMS at 4am.**

Trace path:

1. Find the notification row in DB: `SELECT FROM notifications WHERE recipient_user_id = ? AND sent_at BETWEEN ? AND ? AND channel = 3`. Returns `notification_id`, `event_id`, `template_id`, `category`, `sent_at`.
2. From `event_id`, find the originating event. Who emitted it? What category?
3. Check user's preferences: are quiet hours set? What timezone?
4. Check the Fan-out Service log for this event: did it evaluate quiet hours? What did it decide?

Likely causes:

- **Transactional category.** Quiet hours do not apply to transactional. If the SMS was a 2FA code, that is by design.
- **Timezone wrong.** User's timezone is UTC in our records but they are actually in PST. We sent at "10am UTC" which is 3am PST. Fix: refresh user timezone on app open.
- **Bug in quiet hours evaluation.** Off-by-one or DST issue. Reproduce by replaying the same event with the same user state.

The audit log (which Fan-out Service decisions were made for this notification) is what makes diagnosis possible. Without that audit log, you cannot answer this question.

**9. Add web push as a new channel.**

What changes:

- New channel adapter for Web Push protocol (uses VAPID keys, talks to FCM for Chrome, Mozilla autopush for Firefox, etc.). New entry in the `channel` enum.
- New template variant per template (web push bodies are short, no rich content).
- Subscription endpoint: browser subscribes to push and gets a subscription object (endpoint URL + keys). We store this in `user_devices` with `platform=3 (web)`.
- Preferences UI adds the web push toggle.

What stays the same:

- Ingest API: no change. Just emit events; the Fan-out Service decides whether to add web push as a channel based on the user having an active web subscription.
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

This is the source-of-truth answer. To prove the negative ("user did not receive notification X"), the absence of a row is sufficient because every emitted notification is logged at queue time, before any provider call.

Retention: 30 days hot in Postgres, archived to S3 as Parquet for 7 years. Compliance queries hit Postgres for recent data and Athena for archived.

For deeper audit (which Fan-out decisions were made, e.g., "we considered SMS but skipped because user opted out"), structured logs from the Fan-out Service are shipped to a log warehouse. Cheaper than putting every decision into Postgres; sufficient for the rare compliance audit.

### 12. Trade-offs and what a senior would mention

- **Build vs buy channel adapters.** You could use a single service like OneSignal or Braze to handle all channels. The pro: less code, faster to launch. The con: you pay per notification, you cannot tune behavior per channel, and you have less observability into provider issues. For a product sending billions, build. For a product sending millions, buy.

- **Aggregation aggressiveness.** Aggregate too aggressively and users miss real-time events. Aggregate too little and they get spammed. The right answer is product-specific and depends on user segmentation. Some products tier: power users see real-time, casual users get hourly digests.

- **Where to put preferences evaluation.** Earlier (in Fan-out) means cheaper but less responsive to preference changes. Later (in channel worker) means more responsive but more expensive. We chose Fan-out for the steady-state case and accepted a 10-second window where stale preferences may still send.

- **Strict vs eventual dedup.** SETNX in Redis is fast but has a tiny window where two workers can both win. Two-phase commit with Postgres is durable but adds 5-10ms. We use SETNX for social and marketing, two-phase for transactional and 2FA.

- **Per-user cap vs per-recipient cap.** A user can have many recipients (multiple devices, multiple emails). Do we cap per user (one limit across all their addresses) or per recipient (each device has its own limit)? Per user. Otherwise a user with 5 devices effectively gets 5x the cap.

- **What I would revisit at 10x scale.**
  - Move the Fan-out Service to a streaming framework (Flink or Kafka Streams) instead of bare Kafka consumers. Lets us express aggregation windows declaratively and get state management for free.
  - Federate the dedup store regionally. At 100K/sec writes, one global Redis becomes the bottleneck. Shard by recipient_user_id with strong consistency within shard, eventually consistent across shards.
  - Build a "shadow send" mode where new channels or new templates can be evaluated against real traffic without actually delivering. The shadow path writes to a log instead of calling the provider. Lets you validate behavior before rolling out.

### 13. Common interview mistakes

- **One worker pool for all channels.** "We have a worker that handles push, email, and SMS." Wrong. One bad channel drags down the others. Separate pools, separate Kafka topics.

- **No idempotency key on producer calls.** "Producers just call our API." Then a producer retry sends two notifications. Bad.

- **Synchronous preference lookup from a database.** "We look up preferences for each event from Postgres." At 100K/sec, that is 100K Postgres reads/sec, which melts the database. Redis cache with pub/sub invalidation.

- **No mention of aggregation.** A user with 100 likes on a post gets 100 notifications. Real product complaint within a day of launch.

- **No quiet hours, no per-user cap.** Both are required for any product that has more than a handful of users.

- **Treating push token revocation as a manual cleanup.** APNs and FCM tell you when a token is dead. Build the feedback loop on day one.

- **No deadletter strategy.** "Failed messages just retry forever." Either you flood the provider or messages pile up forever. Deadletter + categorization + auto-cleanup is required.

- **Ignoring channel-specific SLAs.** Treating push and email with the same retry policy means you either over-retry push (delivers a "your driver arrived" 2 hours late) or under-retry email (drops messages that could have been delivered after the provider's 1-hour outage).

- **Forgetting that marketing is different.** Transactional and marketing have different latency, different scale, different compliance. A single uniform pipeline blurs them and creates problems.

- **No mention of compliance.** GDPR for EU, TCPA for SMS in US, CAN-SPAM for email. Each has hard rules (unsubscribe links mandatory in marketing email, opt-in proof for SMS). The architecture must accommodate them, not bolt them on later.

If you hit 8 of these 10, you are interviewing well. Most candidates miss aggregation, quiet hours, and push token cleanup.
