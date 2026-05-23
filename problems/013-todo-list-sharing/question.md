---
id: 13
title: Design a Todo List with Sharing and Collaboration
category: Collaboration
topics: [real-time, websockets, sharing, permissions, eventual consistency]
difficulty: Easy
solution: solution.md
---

## Scene

You sit down. The interviewer leans back and says:

> *"Everyone has used Todoist or Trello. I want you to build a todo list app. Users create lists, add items, check them off. The interesting part: any list can be shared with friends. When Alice adds an item, Bob, who is also on that list, should see it on his phone within a second or two. Build that."*

They wait.

On the surface it looks like CRUD over items in a list. That answer dies fast. Three problems are hiding inside the question: how do you push changes to other devices in near real-time without melting your servers, what happens when Alice and Bob edit the same item title at the same instant, and what does the permission model look like once a list can be re-shared.

Most candidates jump straight to "I will use WebSockets." That is the right tool but the wrong place to start. The right place is to ask what kind of collaboration the interviewer actually wants. A list that updates every 10 seconds is a totally different system from one where Alice's keystrokes show up on Bob's screen as she types.

## Step 1: clarify before you design

Take 5 minutes. Do not draw anything yet. Ask at least 8 questions. The answers move the design from "polling every minute" all the way to "WebSocket fan-out with CRDTs," and you need to know which direction you are being pushed.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. How real-time is "real-time"? Is a 1-second lag acceptable, or are we aiming for sub-100ms like Google Docs character-by-character? This single answer drives 60% of the design. 1 to 5 seconds means polling or long-poll works. Sub-second means WebSocket. Sub-100ms with multiple editors lands you in CRDT or operational transform territory.
2. What can be shared, and at what granularity? Individual lists, whole workspaces, single items? Can an invitee re-share? Per-list sharing is the common answer. Re-sharing changes the permission model from a flat grant into a transitive graph.
3. Roles and permissions. Viewer, editor, admin? Per list or global? Can a viewer see who else is on the list? Two roles cover 90% of cases; add admin if you want a designated owner who can revoke.
4. Scale. How many users? How many lists per user? How many items per list? 10k users with 50 lists each and 100 items per list is a totally different sizing exercise than 10M users with the same shape. Both are reasonable.
5. Offline support. Can a user add or check off items with no network? What happens on reconnect? Offline-first is a major design choice. It pushes you toward operation logs and conflict resolution rather than naive last-write-wins.
6. Notifications. When Alice adds an item to Bob's shared list, does Bob get a push notification? In-app only? Email digest? Notifications usually live in a separate service consuming events, but you need to know whether they are in scope.
7. Item structure. Plain text checkboxes, or due dates, assignees, attachments, sub-tasks? Affects the data model. Start simple (title + done flag) and ask whether richer fields are needed.
8. Deletion semantics. When Alice deletes an item, does it vanish from Bob's view? Soft delete with undo? Trash bin? Soft delete with tombstones is the right answer for any collaborative system; explain why.
9. Mobile vs web. Both? Native app or PWA? Native gives you durable background sync; web limits you. Affects how you design offline sync.
10. Invite flow. Email invite, share link, or direct in-app share by username? Invite-link is simplest, direct grants are safest, both are commonly supported.

A strong candidate also names what is out of scope. Rich-text editing inside items, file attachments, calendar integration, AI features. Those would multiply complexity if attempted in 45 minutes.

</details>

## Step 2: capacity estimates

The interviewer gives you the small-scale numbers first:

- 10k daily active users
- Average 20 lists per user, but most lists are personal (only ~3 shared lists per user)
- Average shared list has 4 collaborators
- Each user adds or checks ~30 items per day
- Each user opens the app ~15 times per day

Compute on paper before revealing:

1. Writes per second (item add, check off, edit, delete)
2. Reads per second (open the app, refresh a list)
3. Concurrent WebSocket connections (assume each active user holds one)
4. Total storage after 2 years
5. Fan-out: when Alice writes to a 5-person list, how many delivery events fire?

Then redo the same math for 1M daily actives.

<details>
<summary><b>Reveal: the math at both scales</b></summary>

**10k DAU (small scale):**

- Writes: 10k × 30 = 300k operations/day = ~3.5 writes/sec sustained, ~10/sec peak. Trivial.
- Reads: 10k × 15 list opens × ~3 lists per session ≈ 450k reads/day = ~5 reads/sec sustained, ~15/sec peak. Also trivial.
- Concurrent WS connections: if 20% of DAU are active at peak hour, that is 2k open sockets. A single small box (4GB RAM) handles that comfortably.
- Storage: 10k users × 20 lists × 100 items × ~200 bytes/item = 40MB. Plus op-log retention at ~3 ops/sec × 86400 × 365 × 2 × ~300 bytes = ~60GB over 2 years. Easy on one disk.
- Fan-out: each write to a 5-person list fires 4 delivery events (the writer already knows). At 10 writes/sec peak that is 40 deliveries/sec. Nothing.

You could literally build this with one Postgres, one app server, and polling every 5 seconds. The only reason we will not is that we want a design that grows.

**1M DAU (large scale):**

- Writes: 1M × 30 = 30M/day = ~350 writes/sec sustained, ~1k/sec peak. Still small for any modern database.
- Reads: 1M × 15 × 3 ≈ 45M/day = ~520 reads/sec sustained, ~1.5k/sec peak.
- Concurrent WS: 20% active at peak = 200k open sockets. Now you need multiple WS servers; one box does not hold 200k sockets without careful tuning. A typical Node or Go server handles ~50k per box, so 4 to 8 boxes minimum.
- Storage: 1M users × 20 lists × 100 items × ~200 bytes = 4GB for current state. Op log at ~350 ops/sec × 86400 × 365 × 2 × ~300 bytes = ~6.5TB over 2 years if you keep every op forever. You will not. Compact old ops; keep the last 30 days for replay.
- Fan-out: ~1k writes/sec peak × 4 collaborators average = ~4k delivery events/sec. Still small, but now distributed across WS servers, so you need pub/sub between them.

The math tells you something important. Throughput is small even at 1M DAU. The hard part is not the write rate. The hard parts are connection count (200k open sockets needs multiple servers and a way to route a write to the right server), fan-out (one write must reach all active collaborators), and the offline sync queue for users who reconnect after hours away. Storage of the live state is tiny. The op log, if kept forever, dominates storage. Compaction matters. Reads vastly outnumber writes by raw count once you include dashboard rendering. Cache the "list summary" view aggressively.

</details>

## Step 3: real-time updates, three options

Alice adds an item. Bob needs to see it. You have three serious options to push the update to Bob's phone:

1. Long-poll. Bob's client opens an HTTP request that the server holds open until a change happens or a timeout fires.
2. WebSocket. Bob's client opens a persistent bidirectional connection; the server pushes whenever it has something.
3. Server-Sent Events (SSE). Like WebSocket but one-way (server to client), runs over normal HTTP.

Before peeking at the comparison, write down the pros and cons of each. Think about connection cost per user, complexity, mobile background behavior, what happens when 100 things change at once, and what happens behind a corporate proxy that blocks WebSocket.

<details>
<summary><b>Reveal: comparison and recommendation</b></summary>

| Aspect | Long-poll | WebSocket | SSE |
|--------|-----------|-----------|-----|
| Protocol | HTTP request + response (repeated) | Single persistent TCP connection upgraded from HTTP | Single persistent HTTP response, chunked |
| Direction | Client to server only (response carries push data) | Bidirectional | Server to client only |
| Latency to deliver an update | Up to the next reconnect (sub-100ms after a response, more on a fresh connect) | Sub-100ms; immediate push | Sub-100ms; immediate push |
| Connection cost | High (constant reconnects, each is a full HTTP handshake) | Low (one connection, kept open) | Low (one connection, kept open) |
| Server memory per user | Low when not connected, high during the long poll | ~10 to 50 KB per open socket | ~10 to 50 KB per open response |
| Proxy / firewall friendliness | Excellent (just HTTP) | Often blocked or rewritten; needs WSS over 443 with origin headers | Excellent (looks like a slow HTTP response) |
| Mobile background | Naturally pauses when the app suspends; client polls again on wake | Connection dies when app backgrounds; reconnect on foreground | Same as WebSocket, dies on background |
| Complexity | Lowest (just HTTP retries) | Highest (frame handling, ping/pong, reconnect, auth on upgrade) | Medium (HTTP, but need to handle resumption and event IDs) |
| Sending client to server | Use a separate POST | Same socket | Use a separate POST |
| Multiplexing many lists per user | Trivial; each poll returns a delta | Easy; route messages over the same socket with a list_id field | Easy; same as WebSocket |

**Recommendation: WebSocket with long-poll as fallback.**

WebSocket is the primary path. Lowest latency, lowest server cost per connection at scale, native browser support. Long-poll is the fallback. Some corporate networks and old mobile networks break WebSocket. The client tries WSS first; on failure (handshake fails, no message within 30 seconds), it falls back to long-poll on a separate endpoint. Same delivery semantics, just slower.

SSE is also reasonable and some teams pick it for its simplicity. The reason most pick WebSocket is the bidirectional channel: you can send presence updates ("Alice is typing") and acks on the same socket. SSE forces a separate POST for every client-to-server message.

WebSocket is not free. An open socket costs memory on the server (10 to 50 KB for buffers, plus app-level state). You pay TCP and TLS keepalive overhead. The client must send a ping every 30 seconds or so to detect dead connections behind NATs. Mobile clients drop the socket constantly (screen off, app backgrounded, weak signal), so you need fast reconnect with a "give me changes since X" delta sync. A single WS server in 2026 handles 50k to 100k connections. Beyond that, you shard across multiple WS servers and need pub/sub to fan a single write out to all of them.

At small scale (Stage 1 in the scaling section), you can skip WebSocket entirely and just poll every 5 to 10 seconds. The user sees changes within 10 seconds, which is fine for a todo list. WebSocket is worth building when you have enough active users that polling becomes a measurable cost.

</details>

## Step 4: sketch the high-level architecture

Here is an intentionally incomplete diagram. Fill in the five `[ ? ]` placeholders.

```
                Client (web, iOS, Android)
                        │
                        ▼
                ┌──────────────┐
                │   [ ? ]      │  (auth, rate limit, sticky routing for WS)
                └──────┬───────┘
                       │
        write          │           subscribe (WS)
                       │
            ┌──────────┼──────────┐
            │                     │
            ▼                     ▼
      ┌──────────┐         ┌──────────────┐
      │  [ ? ]   │         │   [ ? ]      │  (holds open sockets, routes
      │ (CRUD,   │         │   (push      │   messages to subscribers
      │  perms,  │         │    server)   │   on this server)
      │  writes  │         └──────┬───────┘
      │  ops)    │                │
      └────┬─────┘                │
           │                      │
           ▼                      │
      ┌──────────┐                │
      │   [ ? ]  │ ───────────────┘  (source of truth: lists, items,
      │          │                    op log, share grants)
      └────┬─────┘
           │
           ▼
      ┌──────────┐
      │   [ ? ]  │  (fans a write out to every WS server
      │          │   that has a subscriber for that list)
      └──────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                Client (web, iOS, Android)
                        │
                        ▼
                ┌──────────────────┐
                │   API Gateway    │  (auth, rate limit, sticky routing for WS
                │   + Load         │   so a user's reconnects land on the same
                │   Balancer       │   server when possible)
                └──────┬───────────┘
                       │
        write          │           subscribe (WS)
                       │
            ┌──────────┼──────────┐
            │                     │
            ▼                     ▼
      ┌─────────────┐       ┌──────────────────┐
      │  API        │       │  WebSocket       │
      │  Service    │       │  Service         │
      │  (REST):    │       │  (holds 50k      │
      │   CRUD on   │       │   sockets per    │
      │   lists,    │       │   pod; routes    │
      │   items,    │       │   messages to    │
      │   share,    │       │   the right      │
      │   perms;    │       │   subscriber)    │
      │   appends   │       └────────┬─────────┘
      │   to op log)│                │
      └────┬────────┘                │
           │                         │
           ▼                         │
      ┌──────────────────────┐       │
      │  Postgres            │       │
      │  Tables:             │       │
      │   - users            │       │
      │   - lists            │       │
      │   - list_items       │       │
      │   - share_grants     │       │
      │   - change_log       │ ◄─────┘  (WS service reads from change_log
      │     (op log)         │           for replay on reconnect)
      └────────┬─────────────┘
               │
               │  every write also publishes
               ▼
      ┌──────────────────────┐
      │  Redis Pub/Sub       │  (channel per list_id;
      │  (fan-out bus)       │   WS servers subscribe
      │                      │   to channels for the
      │                      │   lists their users
      │                      │   are watching)
      └──────────────────────┘
```

What each piece does:

The API Gateway plus Load Balancer is the first hop. It terminates TLS, runs auth (JWT or session cookie), enforces rate limits. For WebSocket upgrade requests it picks a WS pod and prefers sticky routing (same client back to the same pod across reconnects) to keep cache warm.

The API Service handles all writes: create list, add item, toggle done, edit title, share, revoke. Every write appends to the `change_log` table within the same transaction as the state update. It also handles non-WS reads (initial page load).

The WebSocket Service holds open sockets. When a client connects, it authenticates, then subscribes the connection to the Redis pub/sub channels for each list the user has access to. When a message arrives on a subscribed channel, it forwards to the connection. It also serves the "changes since seq X" replay endpoint over the socket.

Postgres is the source of truth. It stores the current state of every list and item, plus an append-only change_log. The change_log is the spine of the system. It powers reconnect-and-catch-up, undo, and audit.

Redis Pub/Sub is the fan-out bus. Every write publishes a message to a channel named after the list_id. Every WS server holding at least one connection subscribed to that list receives the message and forwards to local sockets. That is what scales fan-out across many WS pods.

Why two services (API and WS) instead of one? They scale differently. API is request/response, low memory, scales with QPS. WS is connection-heavy, high memory per pod, scales with concurrent users. Keeping them separate lets each scale independently and lets you deploy WS server updates without disrupting REST traffic.

</details>

## Step 5: conflict resolution

Alice and Bob both have the same shared list open. At 2:31:05 PM:

- Alice edits the title of item #42 from "Buy milk" to "Buy oat milk."
- At the same millisecond, Bob edits the title of item #42 from "Buy milk" to "Buy almond milk."

Both writes hit your servers within 50ms of each other. What does the system do? Whose title wins? Does Bob see Alice's title flash on his screen and then change back? What if Alice was offline and queued the change locally, then synced an hour later?

Three serious options:

1. Last-write-wins (LWW) by wall-clock timestamp or by an op sequence number.
2. Operational transform (OT) like Google Docs uses for character-level merges.
3. CRDT (conflict-free replicated data type) that mathematically guarantees convergence regardless of order.

<details>
<summary><b>Reveal: which to pick and why</b></summary>

For a todo list with item-level edits (not character-level co-editing inside one field), LWW with a logical clock is the right answer. Here is why and how.

Why not OT. Operational transform shines when two users are typing in the same text field at the same time and you want both keystrokes preserved. That is overkill for "edit item title." If Alice and Bob edit at the same time, one of their edits has to lose; the user-visible behavior is "whichever was second wins." That is fine for a todo list. OT adds large amounts of complexity (transformation functions per op type, central server arbitration in most designs) that you do not need.

Why not CRDT yet. CRDTs are great for offline-first collaborative apps where two users may be disconnected for hours and you want their edits to merge without a server. They have real costs: bigger payload per op (because the CRDT metadata travels with the data), trickier debugging, and a steeper learning curve for the team. For a todo list at small to medium scale, LWW is simpler and the user experience is acceptable. CRDT becomes a strong choice once offline-first is a first-class requirement (see the scaling section, stage 4).

Why LWW with a logical clock, not wall-clock. Wall-clock timestamps are unreliable across devices. Alice's phone might be 30 seconds ahead of Bob's. A correct LWW scheme uses a logical clock so that "newer" wins by a definition that does not depend on clock skew.

Two practical options:

Per-list sequence number. Every op on a list gets a server-assigned monotonically increasing `seq`. The server is the single source of truth for ordering on that list. When Alice and Bob both submit edits, the server stamps each with the next seq. Higher seq wins. Simple, correct, requires the server to be the arbiter.

Hybrid logical clock (HLC). Combines wall-clock and a counter so concurrent ops can be totally ordered without a central server. Useful for true offline-first systems where the client writes ops while disconnected and assigns ordering metadata that survives sync.

The resolution flow for Alice and Bob's clash:

1. Alice submits her edit. Server stamps it `seq=512`, persists to `list_items` and appends to `change_log`. Server publishes `{list_id=L, seq=512, op="edit", item=42, title="Buy oat milk"}` to the Redis channel for list L.
2. Bob's edit arrives 50ms later. Server stamps it `seq=513`, persists, appends, publishes.
3. Both ops fan out via Redis to all WS pods. Alice's WS pod sees seq=512 (her own, already applied locally; she may see a brief flash of "your change is confirmed") and seq=513 (apply: title becomes "Buy almond milk").
4. Bob's WS pod sees seq=512 (apply: his screen briefly shows Alice's "oat milk" if his client hadn't optimistically applied his own change yet) and seq=513 (his own, confirmed).
5. Final state across all clients: title is "Buy almond milk" (the seq=513 winner).

The brief flash is the cost of LWW. Bob saw his own edit applied optimistically, then briefly saw Alice's edit, then his own edit came back as the winner. In practice it happens fast enough (under 200ms) that users do not notice for a todo list. For a document editor that flash would be unacceptable, which is why Google Docs uses OT.

Offline edits queue at the client. If Alice is offline and edits item #42, her client stores the op locally with a tentative client-side ID and a timestamp. When she reconnects, the client replays the op against the server. The server assigns the next seq (which may be much higher than when Alice did the edit). If Bob edited the same item while Alice was offline, Bob's edit was seq=513 and Alice's late-arriving edit is seq=900. Alice's edit wins (higher seq, even though it was authored earlier in wall time). This is the well-known "LWW does not respect intent during long offline periods" caveat. Acceptable for a todo list; not acceptable for document editing.

Special cases:

Delete + edit race. Alice deletes item #42 at seq=512. Bob edits item #42 at seq=513. The delete is recorded as a tombstone with seq=512. Bob's edit at seq=513 is applied to the tombstoned item; on next render, item #42 is hidden because tombstone wins for visibility. Bob's edit lives in the change_log forever; if Alice un-deletes (undo), the edit is still there. Simpler alternative: edits on tombstoned items are rejected with 410 Gone, forcing Bob's client to refresh.

Two adds with the same client-generated ID. Should never happen if client IDs are UUIDv7. Server enforces uniqueness on `(list_id, client_op_id)` to catch retries.

</details>

## Step 6: permissions model

Alice owns a list. She shares it with Bob (editor) and Carol (viewer). Bob then wants to share the list with Dave. Does Bob have that authority? If yes, what role does Dave get? What if Alice later revokes Bob? Does Dave lose access too?

Sketch your share_grant data model and your authorization check pseudocode.

<details>
<summary><b>Reveal: permissions model</b></summary>

Three roles:

| Role | Read items | Write items | Share list | Manage members | Delete list |
|------|------------|-------------|------------|----------------|-------------|
| viewer | yes | no | no | no | no |
| editor | yes | yes | optional (configurable per list) | no | no |
| admin | yes | yes | yes | yes | yes |

The list creator is implicitly admin. Each list has at most one admin in the simple model; advanced models allow multiple admins.

Direct grants, not transitive by default.

```sql
CREATE TABLE share_grants (
    grant_id      UUID PRIMARY KEY,
    list_id       UUID NOT NULL REFERENCES lists(list_id),
    grantee_id    UUID NOT NULL REFERENCES users(user_id),
    role          TEXT NOT NULL,             -- 'viewer' | 'editor' | 'admin'
    granted_by    UUID NOT NULL REFERENCES users(user_id),
    granted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at    TIMESTAMPTZ,                -- soft delete
    UNIQUE (list_id, grantee_id) WHERE revoked_at IS NULL
);
CREATE INDEX idx_share_grantee ON share_grants (grantee_id) WHERE revoked_at IS NULL;
CREATE INDEX idx_share_list ON share_grants (list_id) WHERE revoked_at IS NULL;
```

A user can access a list if they have a non-revoked grant. The `UNIQUE WHERE revoked_at IS NULL` partial index prevents duplicate active grants for the same user on the same list.

Authorization check:

```python
def can(user_id, list_id, action):
    grant = db.query_one("""
        SELECT role FROM share_grants
        WHERE list_id = %s AND grantee_id = %s AND revoked_at IS NULL
    """, list_id, user_id)
    if not grant:
        return False
    role = grant.role
    if action == "read":
        return role in ("viewer", "editor", "admin")
    if action == "write":
        return role in ("editor", "admin")
    if action == "share":
        return role == "admin" or (role == "editor" and list_allows_editor_share(list_id))
    if action == "manage_members":
        return role == "admin"
    if action == "delete_list":
        return role == "admin"
    return False
```

Cache the grant lookup per (user_id, list_id) for 60 seconds. A user touches the same lists repeatedly; the hit rate is very high. Invalidate on grant change.

Transitive sharing: Bob shares with Dave. If Bob is an editor and the list owner has set `allow_editor_share = true`, Bob can invite Dave. Bob can grant Dave any role less than or equal to his own (so Bob the editor cannot create an admin Dave). The new grant on `share_grants` has `granted_by = bob`.

Revocation cascade: Alice revokes Bob. The simple model: revoking Bob does not revoke Dave. Dave still has a direct grant on the list with `granted_by = bob`. The owner (Alice) is the only one who can revoke Dave directly. Bob being kicked out does not mean Dave loses access; Dave was given access as an individual, not as Bob's delegate.

The complicated model is cascading revocation. If `granted_by` chains form a tree, revoking Bob also revokes everyone Bob added, recursively. Much harder to reason about and it surprises users. Slack famously avoids it; Notion offers it as an option per workspace.

Recommendation: non-cascading by default. Show Alice a list of "users added by Bob" when she revokes him and ask whether to also revoke them. Explicit, not implicit.

Share by link. Two kinds:

Invite link. A signed token in the URL. Anyone with the link can join the list (gets a `share_grants` row inserted for their user_id when they accept). Owner can disable the link to revoke future invites without affecting current members.

Public view link. Anyone with the link can view (read-only) without signing in. This is its own row in a separate `public_links` table: `list_id, public_token, role='viewer'`. Different access path; the auth check skips the `share_grants` lookup if a valid public token is presented.

</details>

## Step 7: offline editing and the sync protocol

A mobile client is the dominant use case. The user opens the app on the subway, adds five items, marks two as done, and edits the title of another. There is no network. Twenty minutes later they surface and the phone reconnects. What does the protocol between client and server look like?

Sketch the sync flow. Think about: how does the client identify ops it created offline so the server does not duplicate them on retry, what does the request body look like for a batch sync, what does the server return, and what does the client do when the server says "you tried to edit item I but it was deleted by Alice while you were offline"?

<details>
<summary><b>Reveal: the offline sync protocol</b></summary>

Client-side state while offline.

The client keeps two stores. First, a local mirror of every list the user has access to (or has opened recently): items, titles, done flags, the last `seq` seen for each list. Second, a pending-ops queue: a list of ops the user performed locally that have not yet been confirmed by the server. Each op has a `client_op_id` (UUIDv7 generated at op creation time), a `list_id`, an op type, and a payload.

While offline, the client applies its own ops to the local mirror optimistically. The user sees the result of their edits immediately, with a small visual indicator ("Saving..." or a cloud-with-dash icon) that ops are pending.

On reconnect:

1. The client tries to open a WebSocket. If it succeeds, subscribe with `since_seq` per list. The server replays missed ops from change_log.
2. The client then flushes its pending-ops queue. For each pending op, send a regular write request (`POST`, `PATCH`, `DELETE`) with the same `client_op_id` it generated offline.
3. The server's idempotency check (unique index on `(list_id, client_op_id)`) makes retries safe. If the op was somehow already applied (maybe a previous flush partially succeeded), the server returns the original result.

Batch flush. If the queue is large (e.g., 50 ops), the client may use a batch endpoint:

```
POST /api/v1/lists/{list_id}/ops/batch
[
  {"client_op_id": "...", "op": "add_item", "payload": {...}},
  {"client_op_id": "...", "op": "edit_item", "payload": {...}},
  ...
]
```

The server processes them in order, stamping each with a fresh seq. The response is an array of `{client_op_id, seq, status}` so the client can mark each as confirmed.

Conflicts during sync. Three notable cases:

1. Edit on a tombstoned item. Server returns `410 Gone` for that specific op. Client surfaces it to the user: "You edited 'Buy milk' but it was deleted by Alice. Restore?"
2. Add to a list the user no longer has access to. Server returns `403 Forbidden`. Client surfaces: "Your edits to 'Family chores' were not saved; Alice removed you from this list."
3. The user's local seq is older than the server's compaction window. WS subscribe responds with `too_far_behind=true`. The client refetches the full list state, then reapplies pending ops. If any pending op's target item no longer exists, fall through to case 1.

Why `client_op_id` is non-negotiable. Without it, the server cannot tell a duplicate retry from a new op. The mobile flush might partially succeed (network drop mid-batch), and on retry the previously-applied ops would be re-applied, creating duplicate items. The unique index on `(list_id, client_op_id)` is the only thing that makes the system safe under retry.

Why the pending ops queue lives on disk, not just in memory. The user closes the app. The OS evicts the process. They re-open it the next day. The pending ops need to be there. Use SQLite on iOS/Android, IndexedDB on web. Survive process kills, app updates, and reboots.

</details>

## Step 8: read your way through the rest in solution.md

You have done the conceptual core. The rest is mechanics: data model details, the change_log replay protocol, fan-out via Redis, sticky routing for WS, offline sync, presence, and the scaling journey from 10 users to 1M. Read `solution.md` after you have attempted the follow-ups below.

## Follow-up questions

Try answering each in 2 to 4 sentences before reading the solution.

1. Reconnect after a long disconnect. Bob's phone has been offline for 4 hours. He reconnects and his client knows it last saw `seq=412` on list L. How does the server send Bob just the deltas since then, and how do you cap the cost of a very long catch-up?

2. Presence ("Alice is here"). Bob wants to see a little avatar showing Alice is currently viewing the list. How do you implement presence without writing to the database every second?

3. Permission revoked while user is connected. Alice revokes Bob while Bob has the list open on his phone. Bob's WS connection is still subscribed to the list channel. How fast does Bob actually lose access, and what does his client see?

4. Item ordering. Users can drag items to reorder. Two users reorder simultaneously. How do you represent ordering so the conflict resolution does not produce a mess (items in wildly different orders for different users)?

5. Notifications. When Alice adds an item to a shared list, Bob should get a push notification ("Alice added 'Buy milk' to your shared list"). Where does this happen in your architecture, and how do you avoid sending Bob 50 notifications when Alice adds 50 items in 10 seconds?

6. Search. Bob wants to search across all his lists for "milk." How do you support this without scanning every item in every list?

7. Undo. Bob accidentally deletes an item. He hits Cmd-Z. How does this work, and what happens if other collaborators have already seen the deletion?

8. Sticky routing fails. Your load balancer cannot guarantee sticky routing (cloud LB just round-robins). Bob's reconnect lands on a different WS pod than before. What goes wrong, and how do you fix it?

9. A list with 50k subscribers (rare but possible). Some celebrity creates a "Daily affirmations" list and 50k people subscribe to view. Every edit fans out to 50k clients. What breaks, and what do you do?

10. Privacy: Bob is on a shared list and he can see Alice's email/name. Is that a privacy issue? How do you let users hide their identity on shared lists, and what does the client show instead?

## Related problems

- [Approval Management Service (011)](../011-approval-management/question.md) also uses an append-only event log as the spine. Compare the `change_log` here with the `audit_log` there. Same idea, different consumers.
- [Comment System (015)](../015-comment-system/question.md). Comments on shared items use the same real-time fan-out and permissions checks. The thread structure and notification batching there are directly applicable.
- [Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md). The "render Bob's dashboard" path is a heavy read; the caching and denormalization patterns from that problem apply directly to the list summary view.
