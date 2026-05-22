---
id: 10
title: Design a Notification System
category: Messaging
topics: [fan-out, retry, deduplication, multi-channel, queue]
difficulty: Medium
solution: solution.md
---

## Scene

The interviewer has spent the last fifteen minutes interrogating you on a feed design. They drop that thread and write a new prompt on the whiteboard:

> *Design the notification system for our product. Push, email, SMS. One event in, the right notifications out.*

They lean back. This question looks easy: read from a queue, call APNs, done. It is not. Notifications are where two ugly problems meet: fan-out (one event becomes many notifications) and external delivery (each channel has its own SLA, failure modes, and rate limits). Get the fan-out wrong and you spam users at 3am. Get the retries wrong and you either drop notifications silently or send the same SMS seven times. Both are user-visible.

The interviewer is watching whether you ask about channels, user preferences, and dedup before you start drawing.

## Step 1: clarify before you design

Take 5 minutes. Write down at least six questions. Do not draw anything yet.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Which channels.** "Push (iOS and Android), email, SMS, in-app? Web push?" Each channel is a different external provider with its own SLA, rate limits, and failure modes. The architecture has to accommodate channel adapters as plug-ins, not bake one in.
2. **What triggers a notification.** "Are these user-generated events (someone liked your post), system events (your invoice is ready), or marketing campaigns? All three?" Marketing campaigns have a completely different shape (one operator action triggers 10M notifications in a few minutes) versus transactional (one event triggers one or a few notifications).
3. **User preferences.** "Can users opt out per channel and per event type? Quiet hours? Locale?" Preferences are the single biggest source of "why did I get this" complaints. They must be honored before fan-out, not after.
4. **Latency targets.** "What is the SLA per channel? Push delivered in under a minute? Email under five? SMS under one?" Each channel has different user expectations. SMS for 2FA is on a strict deadline; a weekly digest email can wait an hour.
5. **Aggregation rules.** "If a user gets 100 likes on a post in an hour, is that 100 notifications or one?" Almost certainly one ("John and 99 others liked your post"). Aggregation lives in the design as a first-class step, not a hack.
6. **Volume and fan-out shape.** "How many notifications per day? Largest single event in terms of fan-out?" The shape matters more than the total. 10B/day with even distribution is very different from 10B/day with 90% of it triggered by ten viral events per week.
7. **Idempotency.** "If the same event arrives twice (because the upstream producer retried), do we send the notification twice or once?" The answer is once. The system needs a dedup key.
8. **Compliance.** "GDPR, TCPA for SMS, CAN-SPAM for email? Unsubscribe links mandatory?" Each channel has regulatory requirements that change the API and the data model.

If you only asked "how many notifications per day" you missed the question that decides the entire architecture: user preferences. Preferences turn this from a simple queue into a real fan-out system.

</details>

## Step 2: capacity estimates

Inputs from the interviewer:

- 1B users
- 10B notifications delivered per day across all channels
- Channel mix: 60% push, 30% in-app, 7% email, 3% SMS
- Single event fan-out: ranges from 0 (preferences blocked all channels) to 1M (a marketer's broadcast)
- Per-channel SLAs: push <1min P99, in-app <5s P99, email <5min P99, SMS <1min P99
- Worst-case event burst: a marketing campaign fires 10M notifications in 5 minutes

Compute (do this on paper before revealing):

1. Notifications per second sustained and peak
2. Per-channel send rate sustained
3. Storage for delivery records with 30-day retention
4. Burst rate for the marketing campaign and what that means for queue sizing
5. Fan-out worker pool size assuming each worker handles 500 channel-API-calls/sec

<details>
<summary><b>Reveal: the math</b></summary>

**Notifications per second.**
10B / 86400 ≈ 116K notifications/sec sustained. Peak (3x): ~350K/sec. This is total across channels.

**Per-channel sustained.**
- Push: 60% × 116K = ~70K/sec
- In-app: 30% × 116K = ~35K/sec
- Email: 7% × 116K = ~8K/sec
- SMS: 3% × 116K = ~3.5K/sec

These all sit comfortably under what APNs, FCM, SendGrid, and Twilio quote as their commercial throughput ceilings, but only if you stay within their per-account rate limits. SMS in particular has carrier-level throughput caps (typically 100 messages/sec per long code, 100s of messages/sec per short code). You will have many sender identities, not one.

**Storage for delivery records.**
Record per delivery: `{notification_id (16B), user_id (8B), channel (1B), event_id (16B), template_id (8B), status (1B), sent_at (8B), provider_msg_id (32B), retries (1B), ~30B overhead} ≈ 120 bytes`.
10B × 120B = 1.2TB/day. 30 days = ~36TB. Sharded by notification_id hash, this fits across ~32-64 shards.

**Marketing campaign burst.**
10M notifications in 5 minutes = 33K/sec sustained for those 5 minutes. Layered on top of the steady-state 116K/sec, that is a 30% spike. The queue must absorb at least 10M messages within 5 minutes without backpressure to the producer. Kafka can handle this trivially as long as partitions are sized appropriately.

**Fan-out worker pool.**
At 500 channel calls/sec per worker, 116K/sec sustained needs ~230 workers. Peak: ~700 workers. You set the auto-scaler with consumer lag as the signal.

**Key insight.** The total throughput is not the hard part. The hard parts are:
- Fan-out per event ranges from 0 to 1M and the system must scale across that range without operators tuning anything.
- External providers (APNs, FCM, SendGrid, Twilio) are slow and lossy compared to in-house services. You must retry without duplicating.
- Preferences and quiet hours must be evaluated cheaply and consistently for every single notification.

</details>

## Step 3: pipeline architecture

Notifications follow a producer-to-consumer pipeline with several stages between the triggering event and the delivered message. Sketch the pipeline. There are roughly four stages: ingest the event, expand to per-recipient notifications, route per channel, deliver via external provider.

<details>
<summary><b>Reveal: full pipeline</b></summary>

```
   Producer Service (the thing that fires the event)
   e.g. PostService, BillingService, MarketingTool
                 │
                 │   1. POST /events  { event_id, type, payload, recipients[] }
                 ▼
   ┌─────────────────────────────────────┐
   │  Notification Ingest API            │  Auth, validation, idempotency
   │  (stateless)                        │  on event_id
   └────────────┬────────────────────────┘
                │
                │   2. Append to events.created Kafka topic
                ▼
   ┌─────────────────────────────────────┐
   │  Fan-out Service                    │  For each recipient:
   │  (Kafka consumer)                   │   - load user preferences
   │                                     │   - load template
   │                                     │   - decide channels (per opt-in)
   │                                     │   - check quiet hours
   │                                     │   - check aggregation window
   │                                     │  Emit per-channel tasks.
   └────────────┬────────────────────────┘
                │
                │   3. Append to per-channel topics:
                │      notifications.push, .email, .sms, .in_app
                ▼
   ┌─────────────────────────────────────┐
   │  Channel Workers                    │  One pool per channel.
   │  (push / email / sms / in-app)      │  Render template, call provider.
   └────────────┬────────────────────────┘
                │
                │   4. Provider call (APNs, FCM, SendGrid, Twilio, ...)
                ▼
              User device / inbox / phone
```

Why each stage:

- **Ingest API.** Front door. Validates and dedups on `event_id`. If the same `event_id` arrives twice, the second one is dropped before it costs anything downstream.
- **Kafka in the middle.** Decouples producers from consumers. Producers post and get a fast 202. Consumers run at their own pace. If a downstream provider goes down, messages queue up and we drain when it returns.
- **Fan-out Service.** This is where the event-to-notifications expansion happens. One event may produce 0 (user opted out), 1 (single channel), or many (multi-channel per recipient × many recipients). All preference and aggregation logic lives here.
- **Per-channel topics.** Each channel has different throughput, different rate limits, different retry semantics. Separating them means a SendGrid outage does not back up push delivery.
- **Channel workers.** Stateless. Render the localized template, call the provider, record the outcome.

</details>

## Step 4: incomplete diagram

Fill in the six `[ ? ]` boxes. Hint: think about who owns user preferences, who renders the message body, where the dedup state lives, and what the external providers look like.

```
   Producer Services ──► ┌────────────────┐
                         │   Ingest API   │
                         └───────┬────────┘
                                 │
                                 ▼
                         ┌────────────────┐
                         │  events Kafka  │
                         └───────┬────────┘
                                 │
                                 ▼
                         ┌────────────────┐         ┌──────────────────┐
                         │   Fan-out      │◄────────│   [ ? ]          │  (per-user opt-in,
                         │   Service      │         │                  │   quiet hours, locale)
                         └──┬─────────────┘         └──────────────────┘
                            │
                            │             ┌──────────────────┐
                            │◄────────────│   [ ? ]          │  (subject, body,
                            │             │                  │   localized, versioned)
                            │             └──────────────────┘
                            │
                            │             ┌──────────────────┐
                            │◄────────────│   [ ? ]          │  (event_id+recipient+
                            │             │                  │   channel → already sent?)
                            │             └──────────────────┘
                            │
                            ▼
                ┌────────────────────────────┐
                │  per-channel Kafka topics  │
                │ push / email / sms / inapp │
                └─────┬────┬────┬────┬───────┘
                      │    │    │    │
                      ▼    ▼    ▼    ▼
                ┌───┐ ┌───┐ ┌───┐ ┌───┐
                │[?]│ │[?]│ │[?]│ │[?]│  (each calls its external provider)
                └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘
                  │     │     │     │
                  ▼     ▼     ▼     ▼
                APNs/  Send  Twilio/ WebSocket
                FCM    Grid   SNS    push to
                              SMS    open app
```

<details>
<summary><b>Reveal: complete diagram</b></summary>

```
   Producer Services ──► ┌────────────────┐
                         │   Ingest API   │  Auth, idempotency on event_id
                         └───────┬────────┘
                                 │
                                 ▼
                         ┌────────────────┐
                         │  events Kafka  │  topic: events.created
                         └───────┬────────┘  partitioned by recipient_id hash
                                 │
                                 ▼
                         ┌────────────────┐         ┌─────────────────────┐
                         │   Fan-out      │◄────────│  Preferences Service │
                         │   Service      │         │  (per-channel opt-in,│
                         └──┬─────────────┘         │   per-event opt-out, │
                            │                       │   quiet hours, tz,   │
                            │                       │   locale)            │
                            │                       └─────────────────────┘
                            │
                            │             ┌─────────────────────┐
                            │◄────────────│  Template Service   │
                            │             │  (localized subject │
                            │             │   + body, versioned │
                            │             │   per channel,      │
                            │             │   A/B variants)     │
                            │             └─────────────────────┘
                            │
                            │             ┌─────────────────────┐
                            │◄────────────│  Dedup Store        │
                            │             │  (Redis with TTL,   │
                            │             │   key = event_id +  │
                            │             │   recipient +       │
                            │             │   channel)          │
                            │             └─────────────────────┘
                            │
                            ▼
                ┌────────────────────────────┐
                │  per-channel Kafka topics  │
                │ push / email / sms / inapp │
                └─────┬────┬────┬────┬───────┘
                      │    │    │    │
                      ▼    ▼    ▼    ▼
                ┌────┐┌────┐┌────┐┌────┐
                │Push││Mail││ SMS││InApp│
                │ Wkr││ Wkr││ Wkr││ Wkr │
                └─┬──┘└─┬──┘└─┬──┘└─┬──┘
                  │     │     │     │
                  ▼     ▼     ▼     ▼
                APNs/  Send  Twilio/ WebSocket
                FCM    Grid   SNS    push to
                              SMS    open app
                  │     │     │     │
                  ▼     ▼     ▼     ▼
              Device Inbox  Phone  In-app
                                   feed
```

What each new piece does:

- **Preferences Service.** Cheap, hot, read-mostly. The Fan-out Service hits it once per recipient. Cached aggressively because preferences change rarely.
- **Template Service.** Stores message templates with variables (`Hi {{name}}, you have {{count}} new likes`), versioning, and localization. The renderer is a small library called inside the channel workers; the Template Service just serves template metadata and bodies.
- **Dedup Store.** Redis with TTL. Key is `event_id + recipient + channel`. If present, the notification was already sent (or is in flight) and we skip. TTL is long enough to cover all retry windows (say 24 hours).
- **Per-channel workers.** Each pool is sized independently. Push workers call APNs and FCM in batches; email workers call SendGrid; SMS workers call Twilio with per-number rate limiting; in-app workers write to a WebSocket gateway or store-and-poll.

</details>

## Step 5: retry, dedup, and deadletter

External providers fail. APNs returns 429 when you push too fast. SendGrid returns 5xx during their incidents. Twilio rejects messages to invalid numbers permanently. Your design must distinguish "transient failure, try again" from "permanent failure, give up and clean up" without sending the same notification twice.

Take 10 minutes to design:

1. The retry policy per channel (how many retries, with what backoff)
2. The idempotency key (so retries do not duplicate)
3. What goes to the deadletter queue and who looks at it
4. How invalid push tokens get cleaned up

<details>
<summary><b>Reveal: retry strategy</b></summary>

**Retry classification.** Every provider response maps into one of three buckets:

| Bucket | What it means | Action |
|--------|---------------|--------|
| Transient | 429, 5xx, timeout, network reset | Retry with exponential backoff |
| Permanent invalid recipient | APNs `Unregistered`, FCM `NotRegistered`, Twilio invalid number, email hard bounce | Do not retry. Mark recipient address as invalid. Trigger cleanup. |
| Permanent rejection | Template rejected, content flagged, blacklisted sender | Do not retry. Send to deadletter for human review. |

**Per-channel retry policy.**

| Channel | Max retries | Backoff | Max retry window |
|---------|-------------|---------|------------------|
| Push (APNs/FCM) | 5 | exp(2,4,8,16,32 sec) + jitter | 60 sec (push must be fresh) |
| In-app | 3 | exp(1,2,4 sec) | 7 sec |
| Email | 8 | exp(30s,1m,5m,15m,1h,4h,12h,24h) | 24 hours |
| SMS | 3 | exp(5s,30s,2m) + jitter | 5 min |

Push retries have a tight window because a 2-hour-old "your driver has arrived" push is useless. Email retries are generous because email is store-and-forward by nature, and provider outages can last hours.

**Idempotency key.**

```
dedup_key = sha256(event_id + recipient_user_id + channel)
```

Stored in Redis with a 24-hour TTL. Two checks:

1. **Before send.** Channel worker checks `EXISTS dedup_key`. If present, skip (this is a retry of an already-delivered message).
2. **After send.** Channel worker `SET dedup_key 1 EX 86400`.

Race condition: two workers process the same message simultaneously (because Kafka rebalanced and replayed). Both see the dedup_key missing, both send. Mitigation: `SET dedup_key 1 NX EX 86400` (set only if not exists) returns whether the worker is the "winner". The loser skips the provider call. This narrows the race window to the inflight provider call itself; we accept the tiny duplicate risk during rebalances.

For mission-critical channels (SMS for 2FA), use a stronger check: write to a Postgres row with `INSERT ... ON CONFLICT DO NOTHING RETURNING` and only proceed if you got the insert. Adds latency but guarantees no duplicates.

**Deadletter queue.**

Messages that exhausted retries go to `notifications.deadletter.{channel}`. A small dashboard surfaces them grouped by error reason. Permanent invalid recipients are auto-cleaned (push token deregistered, email marked bounced). The rest land in front of a human, usually within an hour.

**Push token cleanup.**

APNs and FCM both return "this token is dead" responses. The channel worker reads this and:
- Writes a row to a `push_token_invalidations` topic.
- A small consumer marks the token as `revoked` in the user-devices table.
- Future notifications skip that token. If the user has no other devices, the push channel returns "no recipient" for that user, which the Fan-out Service treats like an opt-out.

</details>

## Step 6: rate limiting, aggregation, and quiet hours

Notifications are the easiest way to make users uninstall your app. You need three guardrails, each operating at a different scope.

Take 10 minutes:

1. **Per-user cap.** A user should not receive more than N notifications per hour across all channels (let alone per channel).
2. **Aggregation.** 100 likes on one post should be one notification ("John and 99 others"), not 100.
3. **Quiet hours.** A user in Tokyo at 2am should not receive a marketing push, even if the system is in California at 10am.

How do you implement each? Where in the pipeline do these live?

<details>
<summary><b>Reveal: guardrails</b></summary>

**Per-user cap.**

Where: Fan-out Service, after channel selection, before emitting per-channel tasks.

Mechanism: a Redis sliding-window counter per `(user_id, channel)`. `INCR notifications:hourly:{user_id}:{channel}` with a 1-hour TTL. If the counter exceeds the cap (say 20/hour for push, 5/hour for SMS), drop the notification or defer it to a daily digest.

The cap is per-channel because the costs differ. SMS at $0.01/each is much costlier than push at near zero, so the SMS cap is much tighter.

Transactional notifications (your code, your invoice, your driver arrived) bypass the cap. They are tagged with a `category=transactional` flag and the cap check skips them. Only `marketing` and `social` categories count toward the cap.

**Aggregation.**

Where: Fan-out Service. There is a separate path for aggregatable events.

Mechanism: events that can be batched (likes, follows, comments on the same post) carry an `aggregation_key` like `post:{post_id}:likes`. The Fan-out Service does:

1. On event arrival, check Redis for an existing aggregation window: `EXISTS agg:{aggregation_key}`.
2. If no window: start a window with `SETEX agg:{aggregation_key} 3600 1` (1-hour window) and emit a "delayed" notification scheduled for window end.
3. If window exists: `INCR agg:{aggregation_key}`. Do not emit a new notification.
4. When the window expires, a scheduled job (or a delayed Kafka message) reads the count and dispatches one notification with the aggregated content.

The notification body is templated: `"{{first_actor.name}} and {{count - 1}} others liked your post"`. The Fan-out Service materializes the actor list at window-close time.

There is a subtle ordering problem: the aggregation window starts on the first event, so a user who gets 1 like at T=0 and 99 likes at T=3599 gets one notification at T=3600 (good). But a user who gets 99 likes at T=0 and 1 like at T=3601 gets two notifications (the first 99 and then the lonely 1). Tunable; some products use rolling windows, some use fixed.

**Quiet hours.**

Where: Fan-out Service, before emitting any task to a channel topic.

Mechanism: each user's preferences include their timezone and a do-not-disturb window (e.g., 22:00 to 07:00 local). The Fan-out Service converts "now" to the user's local time and checks the window.

If now is inside the quiet window:
- **Transactional:** still send. Quiet hours do not apply.
- **Marketing:** drop. Marketing during quiet hours is not delayed; it is dropped. Users dislike receiving a morning rush of overnight notifications.
- **Social (likes, follows, comments):** defer. Hold in a "deferred" topic with a delayed delivery time set to the end of quiet hours. When the user wakes up, they get a digest.

The deferred topic is a Kafka topic with delayed delivery support (or a separate scheduling service that releases messages at the right time). At end of quiet hours, the messages flow into the per-channel topics as normal.

The timezone is critical: a marketing campaign that fires at "Tuesday 10am" must fire at 10am in every recipient's local time, not at 10am UTC. The Fan-out Service shards work by recipient timezone for campaigns.

</details>

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. **A producer service retries an event due to a network blip and sends `event_id=42` twice within 100ms.** Walk through how the system avoids sending duplicate notifications. What if the second retry comes 25 hours later, after the dedup TTL expires?

2. **A marketing campaign is meant to send to 10M users but the operator accidentally targets 100M.** How do you stop it mid-flight? What state has to be torn down?

3. **APNs is down for 30 minutes.** What happens to push notifications during the outage? What happens when it comes back? Do users see a flood at recovery?

4. **A user updates their notification preferences to opt out of marketing, but a marketing campaign was already queued.** Are messages already in the queue still sent, or honored against the latest preferences?

5. **A user has 5 devices. They post something that triggers a notification to themselves (e.g., "your scheduled post just went live").** How many push notifications? On which devices? What if one device is signed out?

6. **You discover that one notification template has a bug: it sends "{{name}}" literally instead of the user's name.** How do you roll back? What about the messages already sent?

7. **The notifications database is showing one shard much hotter than others.** Diagnose.

8. **A user complains they got an SMS at 4am.** Trace the path: who is responsible? How do you reproduce?

9. **Web push (browser-based push notifications) needs to be added as a new channel.** What changes in the architecture? What stays the same?

10. **Compliance asks: prove that user 12345 received exactly the notifications we claim, and not others.** What is your audit trail? How long do you keep it?

## Related problems

- **[News Feed (002)](../002-news-feed/question.md)**, the fan-out worker pattern is the same. The celebrity problem (one author with millions of followers) maps onto the marketing campaign problem here (one event targeting millions of recipients).
- **[Rate Limiter (004)](../004-rate-limiter/question.md)**, the per-user notification cap is exactly a rate limiter scoped to a user. The sliding-window counter and token-bucket variants apply directly.
- **[Chat System (003)](../003-chat-system/question.md)**, push notification delivery to mobile devices is the same problem as chat message delivery. APNs and FCM are the same tools, and the device-token lifecycle is shared.
- **[Distributed Cache (009)](../009-distributed-cache/question.md)**, preferences and dedup state both live in Redis with TTL. Hot-key and eviction behavior matters here too.
