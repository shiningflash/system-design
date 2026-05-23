## Solution: Todo List with Sharing and Collaboration

### TL;DR

A shared todo list is a small CRUD app wrapped around two interesting problems: pushing changes to other people's devices in near-real-time, and reconciling conflicting edits when those devices come back online after being disconnected.

The core data model is small. A `users` table, a `lists` table, a `list_items` table, a `share_grants` table for permissions, and an append-only `change_log` keyed by `(list_id, seq)` that records every operation. The change_log is the spine: it powers real-time push, reconnect-and-catch-up, undo, and offline sync.

Real-time delivery uses WebSocket with long-poll fallback. Each WebSocket pod holds many open sockets. Writes go to the REST API, which persists to Postgres and publishes a message on a Redis pub/sub channel named after the list_id. Every WS pod that has a subscriber for that list receives the message and forwards it to local sockets.

Conflict resolution is last-write-wins by server-assigned per-list sequence number, with soft-deleted tombstones so deletes propagate cleanly. CRDTs only become worth their cost at a much later stage, when offline-first becomes a first-class requirement.

The scaling story walks four stages. At 10 users, one Postgres with 10-second polling. At 1k, add WebSocket. At 100k, add Redis pub/sub and shard WS pods. At 1M, shard the DB by list_id, regionalize, and consider CRDTs for offline editing. None of it is hard if you build in order; all of it is hard if you build out of order.

### 1. Clarifying questions and why each matters

Covered in `question.md`. The two questions that change the most: "how real-time is real-time" (drives whether you build WebSocket day one or just poll) and "can a shared list be re-shared" (changes the permission model from a flat grant table to a tree with `granted_by` chains).

The third most important question is "offline support." If yes, the client needs a local op queue and the server needs to accept ops with client-side IDs and out-of-order timestamps. If no, every write requires a connection and the design simplifies significantly.

### 2. Capacity estimates (full working)

**10k DAU (small scale):**

- Writes: 10k × 30 ops/day ≈ 3.5/sec sustained, ~10/sec peak. Trivial.
- Reads: 10k × 15 opens × 3 lists each ≈ 5/sec sustained, ~15/sec peak.
- Concurrent WS: 20% active at peak = 2k sockets. One small box.
- Storage: ~40MB current state + ~60GB op log over 2 years if uncompacted.
- Fan-out: ~40 delivery events/sec at peak.

**1M DAU (large scale):**

- Writes: ~350/sec sustained, ~1k/sec peak.
- Reads: ~520/sec sustained, ~1.5k/sec peak.
- Concurrent WS: 200k sockets. Needs 4 to 8 WS pods at 50k sockets each.
- Storage: ~4GB current state + ~6.5TB raw op log over 2 years (compact aggressively).
- Fan-out: ~4k delivery events/sec at peak, distributed across pods via Redis pub/sub.

**Key insights:**

- Even at 1M DAU, write throughput is tiny by modern DB standards. Postgres handles 1k writes/sec on one box.
- The hard scaling problems are connection count (200k open sockets does not fit on one machine) and fan-out (one write reaches N subscribers).
- The op log dominates storage if you keep every op forever. Compact to "last N days of ops + a snapshot of current state."
- Reads of "render the dashboard" dominate everything. Cache the per-list summary aggressively.

### 3. API design

**List CRUD:**

```
POST   /api/v1/lists                       create a list
GET    /api/v1/lists                       list all lists I have access to
GET    /api/v1/lists/{list_id}             get a list with its items
PATCH  /api/v1/lists/{list_id}             rename, change settings
DELETE /api/v1/lists/{list_id}             delete a list (admin only)
```

**Item CRUD:**

```
POST   /api/v1/lists/{list_id}/items                 add an item
PATCH  /api/v1/lists/{list_id}/items/{item_id}       edit title, toggle done, set due date
DELETE /api/v1/lists/{list_id}/items/{item_id}       soft delete (tombstone)
```

Every write request carries a `client_op_id` header (UUIDv7 from the client) for idempotency. If the same `client_op_id` is seen twice within 24 hours, the server returns the original result rather than applying the op twice.

**Sharing:**

```
POST   /api/v1/lists/{list_id}/shares
{
  "grantee": {"type": "user_id", "value": "..."} | {"type": "email", "value": "..."},
  "role": "viewer" | "editor" | "admin"
}

GET    /api/v1/lists/{list_id}/shares             list current grants
DELETE /api/v1/lists/{list_id}/shares/{grant_id}  revoke

POST   /api/v1/lists/{list_id}/invite-link        create / rotate invite link
POST   /api/v1/invite/{token}/accept              accept an invite link
```

**Get changes since:**

```
GET /api/v1/lists/{list_id}/changes?since_seq=412
```

Returns the ordered list of ops with `seq > 412` for that list, up to a cap (e.g., 500 ops). Used by the client on reconnect to catch up. If the gap is larger than the cap, the response says "too far behind, refetch full state" and the client does `GET /api/v1/lists/{list_id}` to resync.

**WebSocket protocol:**

After upgrade, the client sends:

```json
{ "type": "subscribe", "list_ids": ["..."], "since_seq": { "list_id_1": 412, "list_id_2": 0 } }
```

The server sends back, per list:

```json
{ "type": "op", "list_id": "...", "seq": 413, "op": "add_item", "payload": {...} }
{ "type": "op", "list_id": "...", "seq": 414, "op": "edit_item", "payload": {...} }
```

Ping/pong every 30 seconds. On client send: `{"type": "ping"}`; server responds `{"type": "pong"}`. Connection killed after 60 seconds without traffic.

**Response codes for writes:**

| Status | Meaning |
|--------|---------|
| 200 OK | Op accepted and applied |
| 201 Created | New resource created |
| 401 Unauthorized | Bad or missing auth token |
| 403 Forbidden | User does not have the role required for this action |
| 404 Not Found | List or item does not exist (or user has no read access; we return 404 not 403 to avoid leaking existence) |
| 409 Conflict | Idempotency key reused with a different payload |
| 410 Gone | Item was tombstoned; client must refresh |
| 429 Too Many Requests | Rate limited |

### 4. Data model

```sql
CREATE TABLE users (
    user_id       UUID PRIMARY KEY,
    email         CITEXT UNIQUE NOT NULL,
    display_name  TEXT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE lists (
    list_id        UUID PRIMARY KEY,
    owner_id       UUID NOT NULL REFERENCES users(user_id),
    title          TEXT NOT NULL,
    settings       JSONB NOT NULL DEFAULT '{}',     -- {"allow_editor_share": true, ...}
    next_seq       BIGINT NOT NULL DEFAULT 1,        -- next change_log seq for this list
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at     TIMESTAMPTZ
);
CREATE INDEX idx_lists_owner ON lists (owner_id) WHERE deleted_at IS NULL;

CREATE TABLE list_items (
    item_id        UUID PRIMARY KEY,
    list_id        UUID NOT NULL REFERENCES lists(list_id),
    title          TEXT NOT NULL,
    done           BOOLEAN NOT NULL DEFAULT FALSE,
    due_date       DATE,
    order_key      TEXT NOT NULL,                    -- fractional index for stable ordering
    created_by     UUID NOT NULL REFERENCES users(user_id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seq       BIGINT NOT NULL,                  -- seq of the last op that touched this item
    deleted_at     TIMESTAMPTZ                       -- tombstone
);
CREATE INDEX idx_items_list ON list_items (list_id, order_key) WHERE deleted_at IS NULL;
CREATE INDEX idx_items_list_seq ON list_items (list_id, last_seq);

CREATE TABLE share_grants (
    grant_id      UUID PRIMARY KEY,
    list_id       UUID NOT NULL REFERENCES lists(list_id),
    grantee_id    UUID NOT NULL REFERENCES users(user_id),
    role          TEXT NOT NULL,                     -- 'viewer' | 'editor' | 'admin'
    granted_by    UUID NOT NULL REFERENCES users(user_id),
    granted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at    TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_grants_active
    ON share_grants (list_id, grantee_id) WHERE revoked_at IS NULL;
CREATE INDEX idx_grants_grantee ON share_grants (grantee_id) WHERE revoked_at IS NULL;

CREATE TABLE change_log (
    list_id        UUID NOT NULL,
    seq            BIGINT NOT NULL,                  -- per-list monotonic
    op             TEXT NOT NULL,                    -- 'add_item' | 'edit_item' | 'delete_item' | 'reorder' | 'rename_list' | 'share' | 'revoke'
    actor_id       UUID NOT NULL,
    payload        JSONB NOT NULL,                   -- op-specific
    client_op_id   UUID,                             -- for idempotency
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (list_id, seq)
);
CREATE INDEX idx_change_log_actor ON change_log (actor_id, occurred_at DESC);
CREATE UNIQUE INDEX idx_change_log_idem
    ON change_log (list_id, client_op_id) WHERE client_op_id IS NOT NULL;
```

**Notes:**

- **`next_seq` on the lists table.** When the server records an op, it bumps `lists.next_seq` and uses the prior value as the op's seq. The bump and the change_log insert happen in the same transaction so seqs are gap-free.
- **`order_key` as TEXT, not INTEGER.** Fractional ordering: between items with keys "a" and "c", insert "b". Between "a" and "b", insert "am". Lets you reorder without renumbering. Search "fractional indexing" or LexoRank.
- **`last_seq` on list_items.** Lets a client doing a partial sync know which op last touched this item without joining to change_log.
- **`deleted_at` as tombstone.** Never DELETE rows. Setting `deleted_at` makes the row invisible in the index (the partial index `WHERE deleted_at IS NULL`) but keeps the row for replay.
- **`(list_id, client_op_id)` unique partial index.** Catches retries from the same client. If the network blipped and the client retried, the server applies the op only once.
- **CITEXT for email.** Case-insensitive comparison without a function call.

**Choice: Postgres, not Cassandra/DynamoDB.** This system needs transactions: appending to change_log and updating the item row must be atomic. Per-list seq generation needs strong ordering. Postgres gives you both for free. Cassandra would force you to invent your own seq generator and accept eventual consistency between change_log and items.

### 5. Core algorithms

#### 5.1 The change feed

Every write that mutates list state follows the same skeleton:

```python
def apply_op(list_id, actor_id, op_type, payload, client_op_id):
    with db.transaction():
        # 1. Idempotency check
        if client_op_id:
            existing = db.query_one("""
                SELECT seq FROM change_log
                WHERE list_id = %s AND client_op_id = %s
            """, list_id, client_op_id)
            if existing:
                return existing  # already applied; return original result
        
        # 2. Permission check (cached per (user, list))
        if not can(actor_id, list_id, action_for(op_type)):
            raise Forbidden()
        
        # 3. Get next seq for this list (SELECT FOR UPDATE on lists row)
        lst = db.query_one("""
            SELECT next_seq FROM lists WHERE list_id = %s FOR UPDATE
        """, list_id)
        seq = lst.next_seq
        db.execute("UPDATE lists SET next_seq = %s WHERE list_id = %s", seq + 1, list_id)
        
        # 4. Apply the op to list_items (or lists, or share_grants)
        apply_op_to_state(op_type, list_id, payload, seq)
        
        # 5. Append to change_log
        db.insert("change_log",
                  list_id=list_id, seq=seq, op=op_type, actor_id=actor_id,
                  payload=payload, client_op_id=client_op_id)
    
    # 6. After commit, publish to Redis
    redis.publish(f"list:{list_id}",
                  json.dumps({"seq": seq, "op": op_type, "payload": payload}))
    
    return {"seq": seq}
```

**Key properties:**

- The seq is assigned by the server, in a transaction, with a row-level lock on the `lists` row. This serializes ops on the same list. Ops on different lists are independent and parallelize.
- Idempotency check is inside the transaction so concurrent retries with the same client_op_id collapse cleanly.
- The Redis publish happens *after* commit. If we published inside the transaction and then rolled back, subscribers would see ops that never persisted.
- Subscribers re-read state from Postgres if they need to. The Redis message carries the op so they usually do not need to, but it is the change_log in Postgres that is the source of truth.

#### 5.2 Real-time delivery

When a client opens the app:

1. Initial fetch over REST: `GET /api/v1/lists/{list_id}`. Returns current items and the highest seq seen.
2. Open WebSocket connection.
3. Send `subscribe` message with `since_seq` per list.
4. Server checks if any ops happened between `since_seq` and now (between the initial REST fetch and the subscribe). If yes, sends those ops.
5. Server subscribes the connection to the Redis channel for each list_id.
6. From now on, any new op on those lists is pushed over the socket.

When a write happens on the server:

1. Op is persisted and seq is assigned (as in 5.1).
2. Server publishes to `list:{list_id}` Redis channel.
3. Every WS pod with a local subscriber for that list receives the message.
4. The pod forwards the message to each local socket subscribed to that list, except the originating socket (which already has the optimistic update applied; it gets an `ack` instead).

**Why publish after commit:** if the publish happened inside the transaction and the transaction rolled back, subscribers would see phantom ops. After commit, no phantoms; you may get duplicate delivery if the publish retries (Redis pub/sub is fire-and-forget; not a problem because clients dedupe by seq).

**Why dedupe by seq:** every op has a unique `(list_id, seq)`. The client tracks the highest seq it has applied per list. If it receives an op with seq <= highest, it ignores. This handles both duplicate delivery and out-of-order delivery (which should not happen on a single connection but can happen across a reconnect).

#### 5.3 Conflict resolution

Last-write-wins by per-list seq.

```python
def apply_remote_op(local_state, op):
    if op.op == "edit_item":
        item = local_state.items.get(op.item_id)
        if item is None:
            return  # item was deleted locally; ignore (or refetch)
        if item.last_seq >= op.seq:
            return  # we have a newer op already; ignore
        item.title = op.payload.title
        item.last_seq = op.seq
    elif op.op == "delete_item":
        item = local_state.items.get(op.item_id)
        if item is None:
            return
        if item.last_seq >= op.seq:
            return
        item.deleted = True
        item.last_seq = op.seq
    elif op.op == "add_item":
        if op.item_id in local_state.items:
            return  # duplicate
        local_state.items[op.item_id] = Item(
            title=op.payload.title,
            order_key=op.payload.order_key,
            last_seq=op.seq
        )
    # ...
```

**The full conflict story for "Alice edits, Bob edits same item":**

1. Alice's client optimistically updates locally to "Buy oat milk."
2. Alice's request reaches server. Server assigns seq=512, persists, publishes.
3. Bob's client receives the op (seq=512), sees item.last_seq (currently from when the item was created at seq=400), updates locally to "Buy oat milk."
4. Bob, who was concurrently editing, submits his change "Buy almond milk."
5. Bob's client optimistically updates locally to "Buy almond milk."
6. Bob's request reaches server. Server assigns seq=513, persists, publishes.
7. Alice's client receives seq=513, sees item.last_seq=512, updates to "Buy almond milk."
8. Bob's client receives seq=513 (its own); item.last_seq is now 513 (or already 513 from local optimistic update); idempotent.

Final state on all clients: "Buy almond milk", last_seq=513. Bob's screen never had a flash because his optimistic update was already the winner.

Alice did see a flash: "Buy oat milk" (her local) → "Buy almond milk" (server's seq=513 arrived). For a todo list this is acceptable. For document editing it would not be.

**Tombstones for deletes:**

When Alice deletes item I at seq=600, the row stays in the DB with `deleted_at = now()` and `last_seq=600`. The op is added to change_log. The op fans out. Clients with the item locally set their local copy to deleted.

If Bob, who was offline, edits item I (the deleted one) at his local time and syncs later, the server stamps his edit as seq=900. The server applies the edit: title is updated, but `deleted_at` is not touched. Bob's edit lives in change_log forever; on the next render the item is still hidden because `deleted_at IS NOT NULL`. If Alice undoes the delete (sets `deleted_at = NULL`), Bob's edit becomes visible.

**Compacting the change_log:**

Keep the last 30 days of ops per list (configurable). Older ops are summarized into a "snapshot" record. Clients catching up from a `since_seq` older than 30 days are told "too far behind, fetch full state." This caps storage; the snapshot is just the current state of `list_items` at that point.

### 6. Architecture (recap from question.md)

| Component | Stateful? | Scaling story |
|-----------|-----------|---------------|
| API Gateway / LB | No | Horizontal. Sticky routing for WS based on user_id hash, best-effort. |
| API Service (REST) | No | Horizontal pods; scales with QPS. |
| WebSocket Service | Yes (in-memory connections) | Horizontal; each pod holds 50k sockets. Pub/sub fans messages across pods. |
| Postgres | Yes (source of truth) | Primary + read replicas; shard by list_id at very large scale. |
| Redis Pub/Sub | Yes (ephemeral) | One Redis for small scale; cluster mode for large; partition channels by hash(list_id). |
| Notification Service | No (consumes events) | Separate; consumes change_log via outbox or CDC. |

### 7. Read and write paths

**Write path (add an item):**

1. Client → API Gateway. Auth. Rate limit check (default: 100 writes/min per user).
2. API Service. Permission check (cached). Compute item_id (UUIDv7 from client) and order_key (from `before_id`/`after_id` hints).
3. Transaction: `SELECT next_seq FROM lists WHERE list_id = ? FOR UPDATE`. Bump seq. INSERT into `list_items`. INSERT into `change_log`.
4. Commit.
5. `redis.publish("list:{list_id}", op)`.
6. Return 201 with `{item_id, seq}`.

P99 target: 100ms. The bottleneck is the `FOR UPDATE` lock on the lists row, which serializes writes per list. At 10 writes/sec on the same list (rare), this is fine; at 1000 writes/sec on the same list (very rare for a todo list), you would need to revisit.

**Write path fan-out:**

7. Every WS pod that has a subscriber for `list_id` receives the message from Redis.
8. The pod iterates its local connections subscribed to this list. For each connection (except the originator), it sends the op as a WS frame.
9. Originating connection receives an `ack` with the assigned seq, lets the client confirm its optimistic update.

P99 end-to-end (Alice presses Enter → Bob's screen updates): ~150 to 300ms in the same region.

**Read path (open a list):**

1. Client → API Gateway. Auth.
2. API Service: permission check. Cache lookup for the list summary (Redis hash: `list:{list_id}:summary` → {title, item_count, last_seq}).
3. Cache miss: `SELECT * FROM lists WHERE list_id = ?`, `SELECT * FROM list_items WHERE list_id = ? AND deleted_at IS NULL ORDER BY order_key`. Populate cache with 5-min TTL.
4. Return JSON: list metadata, items, latest seq.
5. Client opens WS, subscribes with `since_seq = <latest seq from step 4>`. Server checks change_log; usually 0 ops to deliver (because the read just happened).

P99 target: 50ms warm, 200ms cold.

**Read path (dashboard, list of all my lists):**

1. Client → API Gateway. Auth.
2. API Service: query share_grants for this user, join lists.
3. Cache lookup for each list's summary; batch the misses.
4. Return JSON array.

Cache: per-user dashboard cache with 30-second TTL, invalidated when any of the user's lists changes.

### 8. Scaling journey: 10 to 1M users

This is the section that demonstrates judgment. At each stage, name what broke and what fixes it. Resist the urge to build stage-3 features at stage-1 scale.

#### Stage 1: 10 to 100 users

**What you build:**

- One Postgres (db.t3.small, 2GB RAM).
- One application server (one container, ~512MB RAM).
- No WebSocket. Client polls `GET /api/v1/lists/{list_id}?since_seq=X` every 10 seconds when the list is open.
- No Redis. Permission cache lives in the app process memory.
- No fan-out infrastructure. The poll model is pull-based; the server has zero responsibility for pushing anything.

**Why this is enough:**

- 100 users × 30 ops/day = 3000 ops/day = ~0.04 ops/sec. Postgres yawns.
- 100 users × 15 opens × 3 lists = 4500 reads/day. Also nothing.
- Polling: at peak 20 users active, each polling every 10 sec = 2 reads/sec. Tiny.
- Latency: changes appear within 10 seconds. Users do not complain because they are not in active back-and-forth editing.

**What you do not build:**

- No WebSocket.
- No Redis.
- No replicas.
- No notification service. Notifications are inline (call SendGrid from the API server).
- No offline support. Network required; show "offline" banner if disconnected.

**Cost:** ~$30/month for the DB, ~$20/month for the app. You ship in a week.

#### Stage 2: 1k to 10k users

**What just broke:**

- "Why does it take 10 seconds for my friend's edit to show up?" Power users in actively-edited lists notice the polling lag.
- Polling cost is now 200 active users × one poll every 10 sec = 20 polls/sec. Still small in absolute terms, but each poll is now a real database query (because the changes might exist), not just a 304.
- Mobile users keep the app open and burn battery polling.

**What you add:**

- **WebSocket service.** A separate process (or a separate port on the same process) that holds open sockets. On client connect, authenticate, then load the user's lists and subscribe the connection to in-memory delivery.
- **In-process fan-out.** Because there is still only one WS server process, the API service publishes to a local channel (e.g., a Go channel, a Node EventEmitter, a Python asyncio.Queue) that the WS service listens to. When a write happens, the API service writes to the channel and the WS service forwards to subscribers.
- **Long-poll fallback endpoint.** For clients that cannot use WebSocket (corporate proxies, ancient browsers). `GET /api/v1/lists/{list_id}/changes?since_seq=X&wait=30` blocks for up to 30 seconds waiting for new ops, then returns them.
- **Per-user permission cache** in Redis (added now because the app server is starting to need shared state). 60-second TTL on `(user, list) → role` lookups.
- **One read replica.** Reads of "list of my lists" and "list contents on initial load" go to the replica; writes go to primary.

**What you do not yet build:**

- No multi-WS-pod yet. One WS pod is enough; 10k users × 30% active = 3k sockets, fits on one box easily.
- No Redis pub/sub. The in-process channel still works.
- No sharding of Postgres.

**Why this is enough:**

- 10k DAU at 30 ops/day = ~3.5 ops/sec, peak 10/sec. Postgres handles it.
- 3k WS connections on one server: trivial.
- Cache hit rate on perms: 95%+ after warmup.

**Cost:** ~$150/month (bigger DB, replica, Redis for cache, WS server).

#### Stage 3: 100k users

**What just broke:**

- 30k concurrent WS connections at peak. One server cannot hold this many comfortably (memory pressure, file descriptor limits, GC pauses).
- A single popular list with 50 active editors: every write fans out to 50 sockets, and those sockets are split across the multiple WS pods you now have. Your in-process channel does not cross pods.
- The "list of my lists" dashboard query for power users (with 200+ lists) is taking 800ms because every list does a row lookup.
- Permission revocation: when Alice revokes Bob, Bob's WS connection keeps receiving updates for up to 60 seconds (cache TTL).

**What you add:**

- **Multiple WS pods.** Run 4 to 8 pods at 50k sockets each. The load balancer routes new WS upgrades by user_id hash (sticky-ish; not strict, because pods come and go).
- **Redis pub/sub for fan-out.** Replace the in-process channel. Every API write publishes to `list:{list_id}`. Every WS pod subscribes to channels for the lists its connections are watching. A message published on `list:L` is received by every pod with at least one subscriber to L, and each pod forwards to its local connections.
- **Dashboard denormalization.** Maintain a `user_list_summaries` Redis hash per user: list_id → {title, item_count, unread_changes, last_modified}. Updated by a worker that consumes change_log events. Dashboard load = one Redis HGETALL. Cold path falls back to the SQL query.
- **Permission invalidation channel.** When Alice revokes Bob, the API service publishes to a dedicated Redis channel `perm:{user_id}` ("you lost access to list X"). All WS pods subscribe to this. The pod holding Bob's connection unsubscribes him from the list channel and sends a `{"type": "access_revoked", "list_id": "X"}` message. Bob's client closes the list view.
- **Read replicas (multiple).** Primary + 3 read replicas. Reads spread across replicas; writes to primary. Replication lag SLO: 1 sec P99.
- **Change_log compaction.** Nightly job archives ops older than 30 days to S3 (Parquet) and deletes from the hot table. Clients catching up from too far back are told to refetch state.
- **Notification service.** Consumes change_log via outbox or CDC (Debezium). Batches per recipient per list over a 60-second window. Sends one push notification with "Alice added 3 items" rather than 3 separate pushes.

**Why this is enough:**

- 100k DAU peak ~30k WS = 4-6 pods at 50k sockets.
- 350 writes/sec peak fans out to ~4 subscribers each = ~1500 deliveries/sec across pods. Redis pub/sub handles this trivially.
- Dashboard load from Redis is sub-10ms regardless of how many lists.

**Cost:** ~$2-4k/month (multiple WS pods, Redis cluster, more DB replicas, S3, Kafka for notification consumer).

#### Stage 4: 1M users

**What just broke:**

- 200k concurrent WS connections at peak. Even with sharded pods, one Redis pub/sub instance is now publishing 1k+ messages/sec to thousands of subscribers; its CPU is getting hot.
- Single Postgres primary writing 1000 ops/sec is fine but you are now nervous about write availability if it fails.
- Users in Europe complain about 200ms latency to US-east API.
- Power users with the app open across 3 devices want their offline edits to merge cleanly when they reconnect after a 12-hour flight. LWW is starting to show its limits ("I made 30 changes on the plane and 5 of them got overwritten by my husband on the ground").
- A celebrity todo list ("Daily affirmations") has 50k viewers subscribed; every edit fans out to 50k clients, hitting the same channel on every WS pod.

**What you add:**

- **Shard Postgres by list_id hash.** 16 shards to start. Each shard has its own primary and replicas. Writes route to the shard owning the list_id; cross-shard queries are rare (a user's dashboard lists shards by user_id, but those lists are scattered across all list shards; this requires a separate `user_list_membership` table sharded by user_id, denormalizing the share_grants for fast lookup).
- **Regional WS clusters.** Each region (us-east, eu-west, ap-south) has its own WS pods, its own Redis pub/sub. A user connects to their nearest region. Writes on a list in us-east are published to us-east's Redis and also forwarded to eu-west's Redis (and ap-south's) via a cross-region replication bridge. Each region's WS pods subscribe locally.
- **Partition Redis pub/sub by hash(list_id).** Run a Redis cluster; each list's channel lives on one Redis node. WS pods determine the right node via consistent hash. Spreads pub/sub load.
- **CRDT for offline-first.** For the offline-editing scenarios that LWW handles poorly, introduce a CRDT-backed item editor. The item title becomes a YATA or Yjs document. Edits are encoded as CRDT ops, not "set title to X." Two users editing offline can both contribute and their edits merge. This is a complexity increase; only adopt where the user pain justifies it. The rest of the system (add/delete/reorder/check) stays LWW.
- **Hot-list cache.** For a list with 50k viewers, the cost is not the writes; it is the fan-out. Add a per-list edge cache layer: 50k clients all receive an op, but the op is sent through a CDN-like fan-out tree, not 50k unicast WS sends from one pod. In practice this means routing the heaviest lists through a dedicated "broadcast" pod tier with optimized I/O.
- **Outbox pattern for cross-region.** Every write to `change_log` also writes to an `outbox` table. A worker reads outbox and forwards to other regions; on success, deletes the outbox row. Exactly-once cross-region delivery, with retries.

**Why this is enough:**

- 1M DAU peak ~200k WS = 4 to 8 pods per region × 3 regions.
- Sharded DB handles 1k writes/sec per shard easily.
- Regional architecture cuts latency to <50ms for most users.
- CRDT handles the offline-editing pain for the users who care.

**Cost:** ~$20-50k/month depending on region count.

#### What you would do at 10M+ DAU

Honest answer: at 10M DAU your todo list app is in the top 50 mobile apps globally. The work shifts from "scale the system" to "operate the system." You would:

- Maintain a dedicated team per shard.
- Adopt a managed real-time delivery product (Ably, Pusher, AWS IoT) instead of running your own WS pods, unless the cost analysis favors in-house.
- Move CRDT to the default model for item-level edits.
- Run an active-active multi-region DB (Spanner, CockroachDB) so a regional outage does not block writes.
- Build a dedicated abuse / spam pipeline (lists used for phishing; bots creating spam lists).

The point of the journey: each stage's complexity is driven by a real bottleneck, not by anticipated load. Build WebSocket at stage 2 because polling burns battery; do not build it at stage 1 just because "real-time is cool."

### 9. Reliability

**Postgres primary fails.** Promote a replica. Writes are unavailable for 30 to 60 seconds. During the outage, the API service returns 503 with `Retry-After: 30`. The WS service keeps connections open and serves cached read state; clients see "saving..." banners.

**Redis pub/sub node fails.** Subscribers reconnect to the new node. During the gap (5 to 10 seconds), no messages are delivered. On reconnect, clients use the WS reconnect protocol: send `since_seq` from their last known state; server replays missed ops from change_log. **No ops are lost** because the source of truth is Postgres; Redis is just the delivery channel.

**WS pod crashes.** All connections on that pod die. Clients reconnect with exponential backoff; the LB routes them to a healthy pod. On reconnect, the catch-up flow delivers any missed ops. Worst case for a user: a 5-second visible disconnect.

**Network partition splits a region.** If the affected region's Postgres replicas cannot reach the primary, reads continue from local replicas (slightly stale). Writes fail. Cross-region delivery to that region pauses; on recovery, the outbox worker drains the backlog.

**Bad release of the WS service drops connections.** Use connection draining: on shutdown, the WS pod sends a "reconnect" message to each socket and waits 10 seconds. Clients reconnect to a sibling pod. Rolling deploys with 10-second drain windows cause no user-visible disruption.

**A buggy client sends 10k ops/sec.** Rate limiter per user per minute (default 100). Sustained abuse triggers a per-user circuit breaker that returns 429 for 5 minutes. Server-side log indicates which user; an alert fires.

**Pub/sub message loss.** Redis pub/sub is fire-and-forget; if a subscriber misses a message, it is gone. Mitigation: clients dedupe by seq, and any gap detection (seq jumps from 412 to 415) triggers a catch-up REST call. The change_log in Postgres is the recovery path.

### 10. Observability

| Metric | Why it matters |
|--------|----------------|
| `ops.applied.rate` | Steady-state write throughput. Sudden spike = abuse or buggy client. |
| `ws.connections.current` | Per-pod count. Pods nearing capacity (>50k) need a scale-out. |
| `ws.message.fanout.p99` | From op committed to last subscriber receiving. SLO: <500ms regionally. |
| `ws.disconnect.rate` | Spike: a deploy went bad, or a network event, or NAT timeouts. |
| `pubsub.subscriber.count` per channel | Tail: which lists have many subscribers. Hot-list detection. |
| `lock.wait.p99` on `lists.next_seq` | If high, a list is being concurrently written by many users. |
| `dashboard.cache.hit_rate` | Should be >90%. Drops mean the cache or invalidation is broken. |
| `change_log.catchup.bytes` p99 | If clients are downloading >10k of catchup, they were offline a long time. |
| `perm.revocation.propagation` p99 | From revoke API call to all WS pods removing access. SLO: <2 sec. |
| `db.replication_lag` p99 | If >2 sec, reads see stale state. |

Alerts:

- Page on: WS message fanout p99 >2 sec for 5 min; write error rate >2%; pubsub subscriber pile-up on any channel.
- Ticket on: dashboard cache hit rate <70%; perm revocation latency >5 sec; one list with >5k subscribers (unusual for a todo list).

### 11. Common gotchas

These are the things that separate a thoughtful design from a generic CRUD answer.

**1. Offline edits sync conflicts.**

Alice edits 30 items on a plane. She lands and reconnects. Her client replays 30 ops to the server. Each gets a new seq. If Bob edited 5 of those items while Alice was offline, Bob's edits had seqs lower than Alice's late-arriving seqs, and Alice's wins by LWW. This is the user-visible "my husband's edits got overwritten" complaint.

Mitigations:
- For high-conflict items, use CRDT for the title field.
- Show a conflict resolution UI: "5 items were edited by Bob while you were offline. Your version: X. Bob's version: Y. Pick one."
- Default to "last writer wins" but make the alternate version recoverable from change_log (a hidden "version history" view).

**2. Deleted items returning from the dead.**

Alice deletes item I. Bob, offline, edited the title of I. Bob syncs and his edit is applied. If you store the edit with `deleted_at IS NOT NULL`, the item stays hidden. Good. But if Alice un-deletes (clears `deleted_at`), Bob's edited title appears. This is usually fine, but if the user never explicitly un-deletes and the item resurfaces due to some other bug (race in setting `deleted_at`), it is a confusing experience.

Mitigation: explicitly version the deletion state. A delete op has its own seq. An un-delete op has its own seq. The visible state at any moment is determined by walking the ops in seq order: latest wins.

**3. Permission revocation while user is connected.**

Bob has the list open. Alice revokes Bob. Bob keeps seeing updates for up to 60 sec (the perm cache TTL).

Fix: dedicated `perm:{user_id}` Redis channel. On revoke, the API service publishes; every WS pod subscribes. The pod holding Bob's connection immediately closes his subscription to the list's channel and sends an `{"type": "access_revoked"}` message. Bob's client closes the list view and shows "Alice removed you from this list."

If Bob has the list cached locally on his device, he can still see the data offline. There is no general fix for that; once data has been delivered to a device, it cannot be recalled. This is a limitation of every shared system and worth saying out loud in the interview.

**4. Item ordering with concurrent reorders.**

Two users drag the same item to different positions. If you store order as integer indexes (item.position = 5), every reorder is a renumbering of all subsequent items, and concurrent reorders race horribly.

Fix: fractional indexing (LexoRank). Each item has an `order_key` like "a3", "aB", "b1". To insert between "a3" and "aB", pick a key lexicographically between them ("a7"). Concurrent reorders produce different keys; the order may not be exactly what either user intended, but it is deterministic and stable.

**5. Seq number gaps after rollback.**

A transaction starts, gets seq=512 from `next_seq`, then fails before commit. The transaction rolls back; `next_seq` is back to 512. But if the transaction was reading uncommitted state from another in-flight transaction, you might end up with gap or duplicate seqs.

Fix: `SELECT next_seq FROM lists WHERE list_id = ? FOR UPDATE`. The row lock serializes seq generation per list. Postgres default isolation (READ COMMITTED) with row locking is sufficient. SERIALIZABLE is overkill and slower.

**6. WS connection storms after a deploy.**

You deploy a new WS pod version. All connections drop, all clients reconnect within 30 seconds, overwhelming your remaining pods.

Fix: rolling deploys with connection draining. Drain one pod at a time: send each socket a "reconnect please" message, wait 10 sec, kill the pod. Clients reconnect smoothly to other pods. Total deploy time is longer but no thundering herd.

**7. Idempotency without client cooperation.**

If the client does not send `client_op_id`, the server cannot dedupe retries. A flaky network where the client retries on timeout causes duplicate items.

Fix: require `client_op_id` on writes. Reject requests without it (after a deprecation window). Modern mobile SDKs handle this transparently.

**8. List of lists query: avoiding N+1.**

`SELECT * FROM share_grants WHERE grantee_id = ?` returns 50 grants. Then 50 `SELECT * FROM lists WHERE list_id IN (...)`. Easy mistake: 50 separate `SELECT` calls if you naively loop. Use a single `IN` query or a join.

**9. The "shared with me" privacy leak.**

If Bob is on Alice's list, Bob can see other members' names and emails by default. Some users want to be private. Add a "show as anonymous" toggle that displays "Anonymous editor" instead of email.

### 12. Follow-up answers

**1. Reconnect after long disconnect.**

Bob's client knows `since_seq = 412` for list L. On reconnect:

```
GET /api/v1/lists/L/changes?since_seq=412
→ {"ops": [{"seq": 413, ...}, ...], "has_more": false}
```

If `412` is older than the compaction window (e.g., older than 30 days), the change_log entries are gone:

```
→ {"too_far_behind": true, "current_seq": 9001}
```

The client responds by doing `GET /api/v1/lists/L` to refetch full state and subscribes from `since_seq = 9001`.

Cap on a single response: 500 ops. If there are more, the response has `has_more: true` and the client paginates with `since_seq = last_seq_received`. This prevents a single user with 100k missed ops from blocking the WS server.

**2. Presence.**

Avoid writing to the database for presence. Presence is ephemeral state.

Implementation: each WS pod maintains in-memory `presence[list_id] = set(user_ids currently subscribed)`. When a connection subscribes to a list, add user_id to the set on that pod; publish `{"type": "presence_join", "list_id": L, "user_id": U}` to a `presence:{list_id}` Redis channel. When a connection unsubscribes or dies, publish `presence_leave`.

Every WS pod subscribed to `presence:{list_id}` maintains a merged set in memory. Clients receive `presence_update` events. Each client renders the avatar list from the merged set.

For cross-pod aggregation: each pod aggregates the global presence set by listening to all `presence_join`/`presence_leave` messages and applying them to a local view. Stale entries are pruned by a heartbeat: each pod re-emits its local presence every 30 sec; entries not heard from in 90 sec are dropped.

Database: not touched. Presence is ephemeral; if all WS pods crash simultaneously, presence resets correctly when clients reconnect.

**3. Permission revoked while connected.**

Sequence (covered in gotcha #3):

1. Alice calls `DELETE /api/v1/lists/L/shares/{bob_grant_id}`.
2. API service updates `share_grants.revoked_at = NOW()` and invalidates perm cache for (bob, L).
3. API service publishes to `perm:{bob_user_id}` Redis channel: `{"revoked_lists": [L]}`.
4. Every WS pod subscribed to `perm:{bob_user_id}` checks if it has any of Bob's connections. The one that does:
   - Unsubscribes Bob's connection from `list:{L}` channel.
   - Sends `{"type": "access_revoked", "list_id": L}` over the WS.
5. Bob's client receives the message. UI: redirect to dashboard with a banner.

End-to-end latency: <2 seconds, dominated by Redis publish/subscribe propagation.

If Bob's device has the list cached locally, the client should clear local state when it receives `access_revoked`. This is a client-side defense; the server has no way to enforce it.

**4. Item ordering.**

Fractional indexing (LexoRank):

- Initial items: keys "n", "v" (spread the namespace).
- To insert between "n" and "v": pick "r".
- To insert between "n" and "r": pick "o".
- After many inserts in the same gap, keys grow longer: "noooo7". After very many, rebalance: a background job renumbers a list's items with evenly-spread keys.

Concurrent reorders: Alice moves item X between A and B; Bob moves item X between C and D simultaneously. Each generates an op with a different `order_key`. The seq number determines who wins: the later seq's `order_key` is what sticks. Item X ends up in one of the two positions; not split across both. Acceptable for a todo list.

Edge case: two ops generate the same key (very rare). Server uses `(order_key, item_id)` as the sort tuple to break ties.

**5. Notifications.**

Architecture: a separate Notification Service consumes change_log events via CDC (Debezium → Kafka, or an outbox pattern: an `outbox` table written in the same transaction as change_log).

The notification service:
1. Receives events from Kafka.
2. For each event, looks up subscribers of the list (from `share_grants`).
3. For each subscriber other than the actor, adds to a per-recipient bucket keyed by `(recipient, list_id)`.
4. Buckets are flushed on a timer: 60 seconds after the first event in the bucket, send a digest notification.

Digest text: "Alice added 3 items and checked off 1 in 'Grocery list'."

Why batched: a user editing actively triggers many ops in a short window. Without batching, the recipient gets 30 push notifications in 5 minutes.

User preferences: per user, per list. Defaults: in-app and push for direct lists; daily digest for high-traffic lists. Stored in a `notification_prefs` table; the notification service reads on every send (cached for 5 min).

**6. Search.**

For small scale (<1M items per user): `SELECT * FROM list_items WHERE list_id IN (lists_user_can_access) AND title ILIKE '%milk%' AND deleted_at IS NULL LIMIT 50`. With a trigram index (`CREATE INDEX ... USING gin (title gin_trgm_ops)`), this is fast enough.

For large scale: pipe items into Elasticsearch. Index per user (or per tenant in a multi-tenant world). The search query hits ES with permission scoping (`list_id IN [...]`). ES returns matches; the API service rehydrates from Postgres.

Permission scoping is the hard part: a user's accessible lists change in real-time. Option 1: pre-compute the user's accessible list_ids and pass them as a filter to ES. Works for users with <1000 accessible lists. Option 2: ES stores `(item, [accessible_user_ids])`; queries filter by user_id. Heavy storage, simple query.

**7. Undo.**

Bob deletes item I. Op recorded at seq=600. Bob hits Cmd-Z within 5 seconds. The client sends a `revert` op referencing the original op:

```
POST /api/v1/lists/{list_id}/ops/revert
{"target_seq": 600, "client_op_id": "..."}
```

Server checks: the target op exists, the requester is the original actor (cannot undo someone else's op), the target op is recent (within 30 seconds). If all true, generate a compensating op (seq=601, op=`undelete_item`, payload references item I).

If other clients already saw the delete and updated locally: they receive the undelete op (seq=601) and locally restore the item. Brief flash; acceptable.

If the time window passed (>30 sec): no undo available; Bob has to recreate the item manually.

**8. Sticky routing fails.**

Without sticky routing, Bob's reconnect lands on a different WS pod. Without state, the new pod does not know which lists Bob was subscribed to.

Fix: Bob's client re-sends the `subscribe` message on reconnect with the full list of list_ids and per-list `since_seq`. The new pod authenticates, subscribes to the relevant Redis channels, and replays catchup ops. No state needs to transfer between pods.

In other words: sticky routing is an optimization (warm cache, lower setup cost), not a correctness requirement. The system works correctly without it; performance is slightly worse on reconnect (one extra round trip for re-subscribe).

**9. A list with 50k subscribers.**

50k subscribers means every edit fans out to 50k WS sockets. The cost: 50k message sends per write. Across N WS pods (say 8), each pod has ~6k subscribers and sends 6k messages per write.

What breaks first: outbound bandwidth on the WS pod. A 500-byte message × 6k subscribers = 3MB per write per pod. At 1 write/sec on the popular list, that is 3MB/sec per pod, manageable. At 10 writes/sec, 30MB/sec per pod, getting tight.

Mitigations:
- **Compact payloads.** Op deltas, not full item objects. 100 bytes per op instead of 500.
- **Coalesce.** If two writes happen within 100ms, send a single combined op.
- **Dedicated "broadcast" pod tier.** The popular list's subscribers are routed to specialized pods optimized for fan-out (higher network bandwidth, tuned kernel). Normal pods do not handle this list.
- **Edge delivery.** For read-mostly popular lists, deliver via CDN-style push (think real-time-CDN products) rather than per-user unicast. Each write goes to a few hundred edge nodes; each edge node serves nearby clients.

For a todo list this scenario is rare; you might cap subscribers per list (max 100) and route popular content to a separate broadcast product.

**10. Privacy: hiding identity.**

Add a per-list `display_preferences` field per share_grant: `{"display_as": "anonymous" | "display_name" | "email"}`. The dashboard and member list respect this. When Alice opens the list, she sees "Anonymous editor" for users who chose to be anonymous; her own choice does not apply to her own view of herself.

For audit purposes: the server's change_log always stores the real `actor_id`. The client API filters the visible name based on display preferences. This means a determined attacker with API access still sees the real actor; but a normal collaborator does not. For stricter privacy, the server can refuse to return actor_id and only return display name, but this breaks features like "Alice's most-edited list."

The deeper question is whether email visibility is a problem. Most products show the email of collaborators (Google Docs, Notion). Some (Figma, Slack) show only display names. Pick a default and let users opt-in to more visibility if needed.

### 13. Trade-offs and what a senior would mention

**LWW vs OT vs CRDT.**

- LWW: simple, works for item-level edits, has the offline-conflict problem.
- OT: optimal for character-level co-editing; complex implementation; overkill for todo lists.
- CRDT: handles offline-first cleanly; metadata overhead per op; harder to debug.

For a todo list: LWW until offline editing becomes a major feature request. Then CRDT for the title field (and only the title field). The rest stays LWW.

**WebSocket vs polling at small scale.**

At 100 users, polling every 10 seconds is cheaper to build and operate than WebSocket. The latency is worse but acceptable. Build WebSocket when polling cost (battery, server) outweighs the build cost. Most teams build WebSocket too early.

**Postgres vs DynamoDB vs Cassandra.**

Postgres wins until you cannot fit the data on one box (~1TB-ish). Then shard by list_id, still on Postgres. DynamoDB is attractive at scale (managed, scales horizontally) but the data model fits Postgres naturally: relational join between lists and share_grants, transactional seq generation, partial indexes for soft delete. DynamoDB forces you to denormalize all of these.

**Redis pub/sub vs Kafka for fan-out.**

Redis pub/sub: fast, simple, fire-and-forget. Best for transient delivery where loss is acceptable (because the source of truth is in Postgres and clients catch up on reconnect).

Kafka: durable, replayable, slower (lower latency than you would think but slower than Redis). Best when you need a guaranteed-delivery log (notifications, analytics).

In our design: Redis pub/sub for live WS fan-out, Kafka (via outbox or CDC) for notifications and analytics. Two systems, each picked for its strengths.

**Per-list seq vs global seq.**

Per-list seq: simpler, no global coordinator, ops on different lists parallelize. Loses cross-list ordering (cannot say "this op on list A happened before that op on list B").

Global seq: requires a global counter, becomes the bottleneck under high write load, gives total ordering across all lists.

For a todo list, per-list seq is the right answer. Cross-list ordering is rarely needed.

**What I would revisit at 10x scale.**

- Move WS to a managed product (Ably, Pusher) or a Yjs-based collaborative editing infrastructure. Building it in-house is fun but operationally heavy.
- Adopt CRDT as the default for item-level state. Yjs is mature enough in 2026.
- Multi-region active-active DB. The outbox pattern works but is complex.
- Dedicated abuse / safety pipeline; large user bases attract spam list creation, phishing in item titles, etc.

### 14. Common interview mistakes

1. **Diving into "use WebSocket and a database."** No clarifying questions, no math, no permission model. Loses you immediately.

2. **Treating real-time as binary.** "We need real-time so WebSocket." No. Define what real-time means. 10-second polling is real-time for a todo list at small scale.

3. **No change log.** A junior design has `list_items` with `updated_at`. A senior design has an append-only log of ops. The log powers real-time delivery, reconnect, undo, audit, and conflict resolution. Without it, every one of those features is a separate hack.

4. **Hard delete instead of tombstone.** Hard delete means a deleted item that an offline client tries to edit later returns 404 silently. Tombstone (`deleted_at`) makes the deletion an event that propagates and can be undone.

5. **Ignoring offline.** Most candidates assume the client is always connected. The interviewer almost always asks "what if Alice was offline." Have an answer.

6. **Permission check on the client only.** "The client knows the user is a viewer so it greys out the edit button." Server must enforce. Always. The client is hostile.

7. **No idempotency on writes.** Mobile networks retry. Without `client_op_id`, retries create duplicate items. This is a famous mobile-app failure mode.

8. **Designing fan-out as "broadcast to all clients."** Without pub/sub, every WS pod has to know about every write. Pub/sub channels per list_id is the standard pattern; mention it explicitly.

9. **Putting the conflict resolution decision in the database.** "We will use Postgres advisory locks to serialize edits." That works for one list but does not scale and does not handle offline edits. The right answer is op-based: the op order (by seq) is the truth.

10. **Treating presence as a database problem.** Writing "user X is online" to Postgres every 30 seconds for 200k users melts the DB. Presence is ephemeral and belongs in memory or Redis, not durable storage.

If you can hit 7 of these 10, you are interviewing well. The change_log + tombstones + idempotency trio is what separates a thoughtful design from a generic CRUD answer.
