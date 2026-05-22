---
id: 3
title: Design a Chat System (WhatsApp / Slack)
category: Real-Time Communication
topics: [websockets, message delivery, presence, ordering, end-to-end]
difficulty: Hard
solution: solution.md
---

## Scene

You are on your third loop. The interviewer has been at WhatsApp and then a stint at Slack. They open with a single line:

> *Design a chat system. Both 1-to-1 and group. Real-time. Mobile clients are the primary surface. Go.*

They are not asking you to design email. The word "real-time" is doing a lot of work in that sentence. If you start by listing REST endpoints, you have already missed what makes this problem hard. If you draw a single WebSocket box and call it done, you have missed the other thing that makes it hard.

The two things that make chat genuinely difficult are **persistent connections at scale** and **per-conversation ordering across mobile clients that go offline constantly**. Everything else is a variation of patterns you have seen.

## Step 1: clarify before you design

Take 5 minutes. Do not draw anything. What do you ask the interviewer? Aim for at least six questions that would change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Scale.** "Daily active users? Connections held open at peak? Messages per day?" A WhatsApp-style answer: 1B DAU, 500M concurrent connections at peak, 100B messages per day. The concurrent-connection number is the single biggest design driver. 500M open TCP sockets is not a back-of-the-envelope detail; it dictates how many edge servers you need and how you load balance them.
2. **1-to-1 vs group, and group size.** "Are groups capped? At what?" WhatsApp caps groups at 1024. Slack channels go to tens of thousands. Discord servers go higher. The group-size answer changes whether per-message fan-out fits in a single transaction or needs a worker pool.
3. **Ordering guarantees.** "Strict FIFO per sender? Total order in a conversation? Causal order?" Most chat products promise FIFO per sender and a "consistent view" per conversation, not strict total order. That difference removes the need for a distributed consensus on every message.
4. **Delivery receipts.** "Do we show sent / delivered / read? For groups too?" Receipts triple the message volume in groups (every recipient sends a delivered, and again a read). If groups are 1000 strong, one message produces 1000 delivered events. This is bigger than the message traffic itself.
5. **Presence.** "Online/offline indicator? Typing indicators? Last-seen?" Presence is a different beast: high update rate, low durability requirements, fan-out to a separate audience (your contacts, not your message threads).
6. **History and retention.** "Server-side history? How far back? Searchable?" WhatsApp historically kept little server-side; Slack keeps everything indexed forever. This decision changes the storage tier entirely.
7. **Encryption.** "End-to-end? Server-side? Key handling on multi-device?" E2E (WhatsApp, Signal) means the server cannot read messages, which constrains search, moderation, and notification content. We mention it but do not deep-dive in this problem.
8. **Push notifications when offline.** "If a recipient is not connected, how do they find out?" Apple/Google push services have their own delivery semantics and rate limits. They are part of the design.

If you only ask "how many users" you have missed the questions that actually shape the architecture. Concurrent connections, group size, and delivery receipts together determine 80% of the system.

</details>

## Step 2: capacity estimates

Inputs from the interviewer:

- 1B DAU
- 500M concurrent WebSocket connections at peak
- 100B messages per day total
- 50% of messages are in groups; average group size 50; max group size 1000
- Each message generates a "delivered" event from each online recipient and a "read" event from each who opens it (assume 80% read rate)
- Average message size 200 bytes
- History retention: 1 year, searchable

Compute (on paper, then reveal):

1. Messages per second sustained and peak
2. Including delivery and read receipts, total events per second through the system
3. Edge server count if one server holds 100K connections
4. Storage for one year of messages
5. Outbound bandwidth at peak (for delivery to recipients)

<details>
<summary><b>Reveal: the math</b></summary>

**Messages per second.**
100B / 86400 ≈ 1.16M messages/sec sustained. Peak 3x → **~3.5M messages/sec**.

**With receipts.**
- 50% of messages are 1-to-1: 1 delivered + 0.8 read = 1.8 receipts per message.
- 50% of messages are in groups of avg size 50: per message there are 49 recipients. ~80% are online at the moment (assume), so ~39 delivered + ~31 read = 70 receipts per group message.
- Weighted average: 0.5 × 1.8 + 0.5 × 70 = ~36 receipts per message.
- Total events: 1.16M messages × (1 + 36) ≈ **43M events/sec sustained, ~130M peak**.

Notice that **receipts dominate the load**, not messages. This is the first non-obvious insight.

**Edge servers.**
500M connections / 100K per server = **5000 edge servers** at peak. In practice you over-provision to 7000-8000 to absorb traffic shifts. Each holds one open TCP socket per connected client; kernel and memory tuning matter.

**Storage.**
100B msgs/day × 365 = 36.5T messages/year. At 200 bytes (content) + ~150 bytes (metadata, sender, conversation_id, timestamp, message_id, indexing overhead) = 350 bytes/msg → **~12 PB/year**. Sharded across hundreds of nodes. Compressed and tiered: hot (last 30 days) on SSD, warm (30 days to 1 year) on cheaper disks, cold (older) archived to object storage.

**Outbound bandwidth.**
3.5M peak messages × 36 fanout × 200 bytes (content) + ~100 bytes (envelope) ≈ 3.5M × 36 × 300 bytes ≈ **38 GB/sec outbound** at peak. Spread across edge servers, that is ~5 MB/sec per server. Trivially within line rate of a single NIC.

**Key insight.** The receipts are the real workload, not the messages. The fan-out for one group message of 1000 members is 1000 deliveries + 800 read receipts = 1800 events for one original event. A naive design treats every receipt as a real message: that gives you 36 PB/year, not 12.

</details>

## Step 3: connection layer: WebSocket, long-poll, or short-poll?

Mobile clients need to receive messages without polling. You have three serious options. Take 8 minutes to write the pros and cons of each.

<details>
<summary><b>Reveal: comparison and recommendation</b></summary>

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Short-poll** | Client GETs `/messages?since=T` every N seconds | Trivial. Stateless servers. Works through any proxy. | High latency (N seconds). Massive request overhead at 500M users polling every 5 seconds = 100M req/sec just for empty polls. |
| **Long-poll** | Client GETs `/messages?since=T`. Server holds the request until a message arrives or 30s timeout. | Lower latency. Works over plain HTTPS. | Still one request per message round-trip. Servers must hold connections open anyway; you have all the WebSocket cost without the WebSocket benefits. |
| **WebSocket** | One persistent bidirectional TCP socket per client | Lowest latency. Bidirectional (server can push, client can ack on same socket). No per-message HTTP overhead. | Persistent connection state. Load balancers and proxies must support upgrade. Mobile networks drop sockets; reconnect logic is nontrivial. Edge server count grows with connection count, not request rate. |

**Recommendation: WebSocket as the primary transport, with HTTPS polling as a fallback** for clients on networks that block WebSocket upgrades (some corporate proxies still do).

Why WebSocket wins:

- 500M concurrent users at one HTTPS request every 5 seconds = 100M req/sec of mostly empty responses. The cost is far higher than holding 500M TCP sockets idle (each idle socket on Linux is a few KB of kernel memory).
- The server-to-client direction is the whole product. Push notifications via APNs / FCM are also part of the stack, but they are for *offline* users; the online experience runs over WebSocket.
- Bidirectional is critical for delivery receipts: when the client receives a message and shows it, the same socket can send the "delivered" ack back without opening a new connection.

For mobile in particular: WebSocket over TLS, with **aggressive keepalive ping every 30 seconds**, and a documented reconnect-with-resume protocol. We come back to the reconnect part in the solution.

</details>

## Step 4: sketch the high-level architecture

Fill in the five `[ ? ]` placeholders. Hint: the gateway holding WebSocket connections is one piece; messages get persisted somewhere; group fan-out is its own subsystem; offline users need a different delivery path; and you need to know which gateway holds which user's connection.

```
   Mobile / Web client
          │
          │ WebSocket (TLS)
          ▼
   ┌─────────────────┐
   │   [ ? ]         │  Holds 100K WebSocket connections per node.
   │  (stateful)     │  Routes inbound messages, pushes outbound.
   └────┬─────────┬──┘
        │         │
   send │         │ recv (via pub/sub from other gateways)
        ▼         ▲
   ┌─────────────────┐
   │   Message       │  Validates, assigns IDs, persists.
   │   Service       │
   └────┬────────────┘
        │
        ▼
   ┌─────────────┐         ┌──────────────────────┐
   │  [ ? ]      │────────►│   [ ? ]              │
   │  (durable   │ change  │  (decides who needs  │
   │   storage)  │ stream  │   to receive this)   │
   └─────────────┘         └──────┬───────────────┘
                                  │
                  ┌───────────────┼────────────────┐
                  │               │                │
                  ▼               ▼                ▼
            ┌──────────┐    ┌──────────┐    ┌──────────┐
            │   [ ? ]  │    │   [ ? ]  │    │  APNs /  │
            │ (knows   │    │ (online  │    │   FCM    │
            │  which   │    │  gateway │    │ (offline │
            │ gateway  │    │  fanout) │    │   users) │
            │ each     │    │          │    │          │
            │ user is  │    │          │    │          │
            │ on)      │    │          │    │          │
            └──────────┘    └──────────┘    └──────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
   Mobile / Web client
          │
          │ WebSocket (TLS)
          ▼
   ┌─────────────────────────┐
   │  Connection Gateway     │   Stateful. Holds ~100K WS connections.
   │  (a.k.a. edge / chat    │   Sticky routing: a user's socket
   │   server)               │   is pinned to one gateway instance.
   └────┬──────────────┬─────┘
        │              │
   send │              │ recv (from other gateways via pub/sub)
        ▼              ▲
   ┌─────────────────────────┐
   │  Message Service        │   Stateless. Validates, assigns
   │  (REST + RPC)           │   message_id (Snowflake), persists,
   └────┬────────────────────┘   acks back to sender.
        │
        ▼
   ┌─────────────────┐      ┌─────────────────────────┐
   │  Message Store  │─────►│  Fan-out Dispatcher      │
   │  (Cassandra,    │ CDC  │  Reads new messages from │
   │   sharded by    │      │  CDC. Resolves who needs │
   │   conversation_ │      │  to receive (members of  │
   │   id)           │      │  conversation).          │
   └─────────────────┘      └──────┬───────────────────┘
                                   │
              ┌────────────────────┼──────────────────────┐
              │                    │                      │
              ▼                    ▼                      ▼
        ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
        │  Presence /  │     │  Delivery    │     │  Push        │
        │  Session     │     │  Workers     │     │  Service     │
        │  Registry    │     │              │     │              │
        │              │     │  For each    │     │  For users   │
        │  Maps        │     │  online      │     │  offline:    │
        │  user_id →   │     │  recipient:  │     │  send via    │
        │  gateway_id  │     │  publish to  │     │  APNs / FCM. │
        │              │     │  gateway's   │     │              │
        │  (Redis,     │     │  pub/sub     │     │              │
        │   replicated)│     │  channel.    │     │              │
        └──────────────┘     └──────────────┘     └──────────────┘
```

Component responsibilities:

- **Connection Gateway.** Holds the TCP/TLS socket. Stateful in the sense that it knows which user_id is connected and on which socket. Auto-scales horizontally; sticky DNS or LB-level steering keeps a user pinned to one gateway across reconnects so receipts and inbound messages route consistently.
- **Message Service.** Validates the message (length, conversation membership, blocked-user check), assigns a globally unique `message_id` (Snowflake, sortable by time), persists, and acks. The ack returns to the sender via the gateway over the same WebSocket.
- **Message Store.** Cassandra or a similar wide-column store, sharded by `conversation_id`. All messages for one conversation live on the same shard; reads of recent history are a single range query.
- **Fan-out Dispatcher.** Consumes the CDC stream from the Message Store. Looks up conversation membership. For each member, decides: are they online (Presence Registry says yes) or offline? Emits delivery tasks to either Delivery Workers or Push Service accordingly.
- **Presence / Session Registry.** Lightweight Redis cluster. Map `user_id → gateway_id, connection_id`. Updated on connect/disconnect. Also stores derived presence state (online/offline/idle).
- **Delivery Workers.** Consume the fan-out task. Use the Presence Registry to find the right gateway, then publish the message to that gateway's pub/sub channel. The gateway sees the message on its channel and writes it to the WebSocket.
- **Push Service.** Talks to APNs (iOS) and FCM (Android, web). Batches and rate-limits to comply with provider quotas. Stores a per-device push token registry.

</details>

## Step 5: ordering guarantees

A user sends three messages quickly: "hi", "are you there", "ok bye". On the recipient's screen, they must appear in that order even if the underlying transport reorders or retries. The same conversation can have two senders typing at once; what order do those interleave in?

Take 10 minutes. Sketch the ordering guarantees you want, and how the message_id design supports them.

<details>
<summary><b>Reveal: ordering model and how to enforce it</b></summary>

The guarantee chat products actually provide:

1. **FIFO per sender within a conversation.** Messages from sender A appear in A's send order. This is the only guarantee users actually notice.
2. **Consistent total order per conversation, but the specific interleaving across senders is server-decided.** If A and B send at nearly the same time, every recipient sees the same A/B interleaving (the server picks one), but no client tries to enforce a particular cross-sender order.
3. **No total order across conversations.** Messages in conversation X and Y are independent.

Implementation:

- **Snowflake-style `message_id`.** 64-bit: 41 bits timestamp (ms since epoch) + 10 bits shard/datacenter + 12 bits sequence. Globally unique without coordination, sortable by time within ~1ms resolution.
- **Within a conversation, message_id is the canonical order.** Recipients sort by message_id when rendering.
- **FIFO per sender: enforced by the Message Service.** Each sender's connection has a monotonic client-side sequence counter. The Message Service rejects out-of-order arrivals from the same sender (returns an error, client retries). This catches the rare case where the same sender's two messages arrive via different paths (e.g., reconnect with retry of an in-flight message).
- **The interleaving:** when A and B both send to the same conversation, the Message Service assigns message_ids in the order their requests reach it. Two messages with timestamps within 1ms get distinct sequence values inside the same millisecond. Every recipient sorts by message_id and gets the same interleaving.

What you do *not* need:

- Distributed consensus on the order. Each message goes through one Message Service instance; that instance picks the order. Different conversations can be on different instances.
- Vector clocks. Overkill for the guarantee chat users actually care about.

Common interview mistake here: candidates try to design Lamport timestamps or vector clocks. The interviewer will let you, then ask "what does the user see if those guarantees are violated?" and the answer is "nothing, because the user is on a phone screen looking at 50 messages and they will not notice a 1ms reordering." Match the engineering to the user-visible problem.

</details>

## Step 6: delivery semantics and receipts

For each message you must support: **sent** (the server stored it), **delivered** (the recipient's device received it), **read** (the recipient opened the conversation). For 1-to-1 these are three states. For groups they are per-recipient state.

Sketch the storage and the event flow for receipts.

<details>
<summary><b>Reveal: receipt model</b></summary>

**Storage shape.** For 1-to-1 chats, a single row per message captures sender, recipient, and the timestamps for each state (sent_at, delivered_at, read_at). For group chats, you store per-(message_id, member_id) state in a separate table because storing per-member columns on the message row would be unbounded.

```
Group receipts (per message × per member):
  PK: (message_id, member_id)
  Columns: state (sent/delivered/read), state_changed_at
```

This table is enormous (one row per group member per message). It is the highest write-volume table in the whole system. Mitigations:

- **Batch receipts.** A client does not send a `delivered` event for each individual message it receives. It batches every 1-2 seconds: "messages M1..M20 are all delivered as of timestamp T." One event from the client, one row update per (message, member) on the server, but the network and gateway work is amortized.
- **TTL the receipts table.** After 90 days, receipts on old messages do not matter. Cassandra TTL handles this for free.
- **Skip storing 'delivered' for group chats above a threshold.** WhatsApp groups above 256 members only show "sent" and "read"; delivered is dropped. The volume saved is substantial.

**Event flow for a 1-to-1 message:**

1. A sends "hi". Message Service persists, returns ack to A. A's client UI shows a single check (sent).
2. Delivery Worker pushes to B's gateway. B's client receives the message, writes "delivered" event back over the WebSocket. Message Service updates the row. A's gateway sees the change via pub/sub and pushes the receipt to A. A's UI shows double check (delivered).
3. B opens the conversation. B's client emits "read M1..Mn" batch. Same flow. A sees double-check-blue.

**Event flow for a group message of 50 members:**

1. A sends to group G. Message persisted with `message_id`.
2. Fan-out Dispatcher resolves 49 other members. 40 are online. Delivery Workers route to 40 gateways.
3. As each recipient's client acks delivery (batched per client), the receipts table accumulates 40 rows for (message_id, *).
4. A's client wants to show "delivered to 40/49". A's client polls or subscribes to a derived counter `(message_id, delivered_count, read_count)` rather than the raw row set. The counter is updated as receipts arrive, cached in Redis, and pushed to A.

The derived counter is the right abstraction: A does not want a list of 49 names with checkmarks, A wants two numbers. Computing counts on read by scanning 49 rows is fine at message scale, but at group-of-1000 scale you maintain the counter incrementally.

</details>

## Step 7: read the full solution

You have done the heart of the design: connections, ordering, delivery, receipts. The solution covers presence in detail, the message store schema and shard strategy, how reconnect-with-resume actually works on the wire, push notifications and their rate limits, and the dozen failure modes you only think about after an outage.

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. **A user goes offline mid-conversation, comes back 30 minutes later.** What is the protocol for catching up on missed messages? How does the client know what it missed without re-downloading the entire history?
2. **A group has 1000 members. One of them is a bot that sends a message every 5 seconds.** What strain does this put on the system, and how do you protect against it?
3. **Presence updates (online/offline indicators).** If every user has 200 contacts and updates presence every time they open the app, what is the fan-out? Is presence stored anywhere durable?
4. **Typing indicators ("Alice is typing...").** How are these delivered? Are they persisted? What is the failure mode if a typing event is sent but the user closes the app before sending the message?
5. **Multi-device sync.** A user has a phone and a laptop both logged in. How do you ensure a message read on the phone disappears from the unread badge on the laptop, and vice versa?
6. **End-to-end encryption.** If messages are E2E-encrypted, the server cannot see content. How does this affect search, push notification previews, and group fan-out?
7. **A gateway holding 100K WebSocket connections crashes.** What happens to those clients? How do they recover? How fast is reconnection at scale?
8. **A new device is added to an existing account.** It has zero local history. How do you bootstrap it efficiently for a user with 10 years of chat history and 500 conversations?
9. **Anti-abuse: a user is mass-DMing strangers (spam).** How do you detect and rate-limit without blocking legitimate high-volume users (e.g., a business account)?
10. **At 3am you see message delivery lag spike to 30 seconds for one region but the metrics for the message store look fine.** Where do you look?

## Related problems

- **[News Feed (002)](../002-news-feed/question.md)**, same fan-out fundamentals; group chat is essentially fan-out-on-write with a small audience.
- **[Notification System (010)](../010-notification-system/question.md)**, the push-notification path in this design is the same machinery; offline delivery is its own subsystem.
- **[Distributed Cache (009)](../009-distributed-cache/question.md)**, the presence registry and receipt counters lean on Redis; understanding its limits is required.
