## Solution: Help Desk Ticketing System

### The short version

A help desk is a small state machine with a messy front door.

The front door is the hard part. Emails arrive over IMAP with broken subject lines. Chats arrive over websockets and end when the user closes the tab. Web forms post JSON. Slack messages come through a bot. SMS comes through Twilio. Each channel needs an adapter that parses, threads replies into existing tickets, and emits a common intake event.

Once a ticket exists, it follows a small state machine: `new -> assigned -> in_progress -> (waiting_on_customer goes back and forth) -> resolved -> closed`, with reopen and reassignment as common edges. The Ticket Service runs the state machine and writes to Postgres. An Assignment Service picks the team and agent. An SLA worker sweeps every 30 seconds, pauses outside business hours, and fires escalation events on breach.

Scale is not the hard part. Even at 50,000 tickets a day with 2,000 agents, you only write about 15 things per second. The interesting work is correctness: email threading, SLA pause and resume across business hours and timezones, agent assignment races, and reopen after silence.

A few words we'll use:

- **SLA** = service level agreement (a promise like "we respond in 1 hour")
- **Adapter** = a small piece of code that talks to one channel
- **Push** = system assigns ticket to an agent directly
- **Pull** = agent clicks "next ticket" to claim one

---

### 1. The clarifying questions, in one paragraph

Covered in `question.md` Step 1. The two questions that change the design the most:

1. **Which channels?** Email is messy because of threading. Adding chat or Slack later just means adding adapters.
2. **What does the SLA look like?** Wall-clock SLA is easy. Business-hours SLA across timezones is a real engineering project. It is the source of most reporting bugs in production.

Everything else (assignment, escalation, audit, retention) follows from those two.

---

### 2. The math, in plain numbers

| Scale | Tickets/day | Writes/sec | Open tickets at any moment | Reads/sec |
|-------|-------------|------------|----------------------------|-----------|
| Startup (3 agents) | 50 | 0.003 | ~100 | ~3 |
| Enterprise (2,000 agents) | 50,000 | ~5 sustained, ~15 peak | ~150,000 | ~65 |

The interesting numbers:

- **150,000 open tickets at any moment** at enterprise scale. Dashboard queries like "show me my pending tickets" must return in under 50ms.
- **5 messages per ticket on average** at startup. **8 messages** at enterprise (longer conversations, more handoffs).
- **Reads beat writes 10 to 1.** Every agent refreshes their dashboard 2-3 times a minute over an 8-hour shift. Customer portals add more.

The system is small in raw throughput. The architecture exists for correctness, fair assignment under concurrency, SLA accuracy, and operational visibility. Not for QPS.

---

### 3. The API

Three core endpoints carry the product. Create a ticket. Add a message. Change status. Everything else is reading data.

**Create a ticket (from the customer side):**

```
POST /api/v1/tickets
Content-Type: application/json
Idempotency-Key: <uuid>

{
  "subject": "Cannot log in to mobile app",
  "body": "I've been trying since this morning...",
  "channel": "web_form",
  "customer_id": "cust_abc",
  "customer_email": "alice@example.com",
  "metadata": { "browser": "Safari 17", "app_version": "4.2.1" },
  "attachments": [{"filename": "screenshot.png", "upload_id": "att_xyz"}]
}
```

| Status | Meaning |
|--------|---------|
| 201 Created | New ticket made. Response has `ticket_id`, `status`, `public_url`. |
| 200 OK | Same idempotency key seen recently. Return the existing ticket. |
| 400 Bad Request | Validation failed. |
| 429 Too Many Requests | Rate limited. |

**Add a message (agent reply or customer follow-up):**

```
POST /api/v1/tickets/{ticket_id}/messages

{
  "body": "Can you share the error you see?",
  "visibility": "public" | "internal",
  "attachments": [...]
}
```

- A `public` message also pushes out (email reply, chat message, Slack DM).
- An `internal` message is an agent-only note. The customer never sees it.

**Assign or reassign:**

```
POST /api/v1/tickets/{ticket_id}/assign

{
  "team_id": "billing",
  "agent_id": "agt_bob"   # optional; null = let Assignment Service pick
}
```

**Change status:**

```
POST /api/v1/tickets/{ticket_id}/status

{
  "status": "in_progress" | "waiting_on_customer" | "resolved" | "closed" | "spam" | "duplicate",
  "reason": "Customer confirmed fix"
}
```

If the transition is invalid (like trying to jump from `closed` to `in_progress` directly), returns `422 Unprocessable Entity`.

**Search:**

```
GET /api/v1/tickets?status=open&assignee=me&priority=high
GET /api/v1/tickets/search?q=login+mobile&customer_id=cust_abc
```

Filtered lookups hit a read replica. Full-text search hits Elasticsearch.

Three small but load-bearing choices:

- **`Idempotency-Key` on create is required.** SES retries. Mobile clients retry on timeout. Web form double-clicks happen. Without the key, you create duplicate tickets.
- **Messages are the unit of conversation, not tickets.** Tickets group messages. Messages carry content.
- **The `visibility: internal` flag is critical.** The outbound channel adapter checks it before sending. This is what keeps internal "war room" notes from accidentally reaching the customer.

---

### 4. The data model

Five tables. Two big, three small.

```sql
CREATE TABLE tickets (
    ticket_id           UUID PRIMARY KEY,
    public_ref          TEXT NOT NULL UNIQUE,     -- "TKT-1234" shown to customer
    subject             TEXT NOT NULL,
    customer_id         TEXT,
    customer_email      TEXT,
    channel             TEXT NOT NULL,            -- 'email', 'web_form', 'chat', 'slack', 'sms'
    channel_thread_id   TEXT,                     -- channel-native ID for threading
    status              TEXT NOT NULL,
    priority            TEXT NOT NULL DEFAULT 'medium',
    team_id             TEXT,
    assignee_id         TEXT,
    tags                TEXT[] DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    first_response_at   TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    reopen_count        INT NOT NULL DEFAULT 0,
    parent_ticket_id    UUID,                     -- if this is a follow-up to a closed one
    metadata            JSONB
);
CREATE INDEX idx_tk_assignee_status ON tickets (assignee_id, status)
    WHERE status NOT IN ('closed','spam','duplicate');
CREATE INDEX idx_tk_team_status ON tickets (team_id, status, priority);
CREATE INDEX idx_tk_customer ON tickets (customer_id, created_at DESC);
CREATE INDEX idx_tk_channel_thread ON tickets (channel, channel_thread_id);

CREATE TABLE ticket_messages (
    message_id          UUID PRIMARY KEY,
    ticket_id           UUID NOT NULL REFERENCES tickets(ticket_id),
    author_type         TEXT NOT NULL,            -- 'customer', 'agent', 'system'
    author_id           TEXT,
    visibility          TEXT NOT NULL DEFAULT 'public',
    body                TEXT NOT NULL,
    body_format         TEXT NOT NULL DEFAULT 'plain',
    inbound_channel     TEXT,
    inbound_message_id  TEXT,                     -- email Message-ID header, etc.
    in_reply_to         TEXT,                     -- email In-Reply-To header
    attachments         JSONB DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_msg_ticket ON ticket_messages (ticket_id, created_at);
CREATE UNIQUE INDEX idx_msg_inbound ON ticket_messages (inbound_message_id)
    WHERE inbound_message_id IS NOT NULL;

CREATE TABLE assignments (
    assignment_id       UUID PRIMARY KEY,
    ticket_id           UUID NOT NULL REFERENCES tickets(ticket_id),
    team_id             TEXT NOT NULL,
    agent_id            TEXT,                     -- null = sitting in team queue
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    unassigned_at       TIMESTAMPTZ,
    assigned_by         TEXT,
    reason              TEXT                      -- 'initial', 'reassigned', 'escalation', 'vacation_handover'
);
CREATE INDEX idx_assignments_ticket ON assignments (ticket_id, assigned_at DESC);
CREATE INDEX idx_assignments_agent_active ON assignments (agent_id) WHERE unassigned_at IS NULL;

CREATE TABLE sla_targets (
    ticket_id           UUID PRIMARY KEY REFERENCES tickets(ticket_id),
    priority            TEXT NOT NULL,
    first_response_due  TIMESTAMPTZ,
    resolution_due      TIMESTAMPTZ,
    paused_at           TIMESTAMPTZ,
    pause_reason        TEXT,
    business_hours_id   TEXT NOT NULL,
    first_response_at   TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    breach_state        TEXT NOT NULL DEFAULT 'on_track'
);
CREATE INDEX idx_sla_first_due ON sla_targets (first_response_due)
    WHERE first_response_at IS NULL AND paused_at IS NULL;
CREATE INDEX idx_sla_res_due ON sla_targets (resolution_due)
    WHERE resolved_at IS NULL AND paused_at IS NULL;

CREATE TABLE ticket_audit (
    audit_id            UUID PRIMARY KEY,
    ticket_id           UUID NOT NULL,
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    event_type          TEXT NOT NULL,
    actor_type          TEXT NOT NULL,
    actor_id            TEXT,
    payload             JSONB NOT NULL
);
CREATE INDEX idx_audit_ticket ON ticket_audit (ticket_id, occurred_at);
CREATE INDEX idx_audit_event ON ticket_audit (event_type, occurred_at);
```

Five small things doing real work:

**The composite index `(assignee_id, status)` on tickets.** This serves the most common query: "show me my pending tickets." Without it, every agent dashboard scans the whole tickets table.

**The unique index on `ticket_messages.inbound_message_id`.** When an adapter crashes and re-processes the same email, the duplicate insert fails with a unique-violation. The second copy is silently dropped. No duplicate conversation on the ticket.

**`assignments` is history, not current.** The current assignee lives on the ticket row for fast reads. The history sits here for audit and analytics ("how many times was this reassigned?").

**`sla_targets` is a separate narrow table.** The SLA worker queries it constantly. Keeping SLA data out of the wide `tickets` table makes the index small and the sweep fast.

**`ticket_audit` is append-only.** Every state change, every reply, every assignment shows up here. Retention is 5-7 years. Used for investigations and compliance.

Why Postgres and not Mongo or DynamoDB? Because state-machine transitions need transactional guarantees. When an agent replies, three things must happen together: insert the message, update `first_response_at`, and update `sla_targets`. If any one fails, all roll back. Postgres gives you this for free.

---

### 5. The intake adapter, in detail

Email is the messy one. Here is what the adapter does for each inbound message.

```python
def handle_inbound_email(raw_email):
    msg = parse_email(raw_email)

    # Filter spam BEFORE creating tickets
    if is_spam(msg):
        record_dropped(msg, reason='spam')
        return

    # Layer 1: In-Reply-To and References headers (RFC 5322)
    parent_msg_id = msg.headers.get('In-Reply-To')
    refs = msg.headers.get('References', '').split()
    candidate_msg_ids = [parent_msg_id] + refs if parent_msg_id else refs

    existing_ticket = None
    for mid in candidate_msg_ids:
        m = db.query(
            "SELECT ticket_id FROM ticket_messages WHERE inbound_message_id = ?", mid)
        if m:
            existing_ticket = m.ticket_id
            break

    # Layer 2: ticket reference in subject like "[TKT-1234]"
    if not existing_ticket:
        ref = extract_ticket_ref(msg.subject)
        if ref:
            existing_ticket = db.query(
                "SELECT ticket_id FROM tickets WHERE public_ref = ?", ref)

    # Layer 3: heuristic match (same sender + similar subject + recent). Off by default.

    if existing_ticket:
        emit_event("ticket.reply_received", existing_ticket, msg)
    else:
        emit_event("ticket.intake", channel='email', payload=normalize(msg))
```

The three layers in priority order:

1. **`In-Reply-To` / `References` headers.** The RFC standard. About 85% of replies hit this path.
2. **Subject line tag** like `[TKT-1234]`. Catches another 10% when corporate mail proxies strip headers.
3. **Heuristic match** (same sender + similar subject + recent). Off by default because it cross-contaminates tickets.

The remaining 5% open new tickets. An agent merges them manually with a "merge into" button.

> Why not just open new tickets and let agents merge? Because customers send 5-10 follow-up emails per ticket. If 5% of all tickets are duplicate fragments, your agents spend their day merging instead of answering. Threading at the front door is worth the complexity.

---

### 6. Assignment, in code

Push assignment (system picks the agent):

```python
def assign_ticket(ticket):
    team = pick_team(ticket)              # triage rules: tag, keyword, channel
    candidates = list_agents_in_team(team)

    # Filter by skill if the ticket has required skills
    if ticket.required_skills:
        candidates = [a for a in candidates
                      if set(ticket.required_skills).issubset(a.skills)]

    if not candidates:
        # Nobody available with the right skills. Drop into team queue.
        return Assignment(team_id=team, agent_id=None)

    candidates = [a for a in candidates if a.is_available(now())]

    with db.transaction():
        # Critical: lock at the team level so two parallel assigners
        # don't both pick the same agent
        db.lock_for_update("teams", team)

        loads = db.query("""
          SELECT agent_id, COUNT(*) AS open_count, MAX(assigned_at) AS last_assigned
          FROM assignments
          WHERE agent_id = ANY(?) AND unassigned_at IS NULL
          GROUP BY agent_id
        """, [a.id for a in candidates])

        # Pick the least busy, tie-break by least recently assigned
        chosen = min(candidates, key=lambda a: (
            loads.get(a.id).open_count,
            loads.get(a.id).last_assigned
        ))

        db.insert("assignments",
                  ticket_id=ticket.id, team_id=team,
                  agent_id=chosen.id, reason='initial')
        db.update("tickets", ticket.id,
                  team_id=team, assignee_id=chosen.id)
        return Assignment(team_id=team, agent_id=chosen.id)
```

> Why the team-level lock? Without it, two parallel `assign_ticket` calls for the same team can both query the load, both see "Carol has the fewest tickets," and both assign to Carol. Now Carol has 2 extra tickets while Dave (who really had the fewest) gets none. The lock makes assignments inside a team strictly sequential. Across teams, it stays parallel.

Pull assignment (agent claims the next ticket):

```python
def claim_next_ticket(agent):
    with db.transaction():
        ticket = db.query("""
          SELECT ticket_id FROM tickets
          WHERE team_id = ? AND assignee_id IS NULL
            AND status IN ('new', 'assigned')
          ORDER BY priority DESC, created_at ASC
          FOR UPDATE SKIP LOCKED
          LIMIT 1
        """, agent.team_id)
        if not ticket:
            return None
        db.update("tickets", ticket.id,
                  assignee_id=agent.id, status='assigned')
        db.insert("assignments", ticket_id=ticket.id,
                  agent_id=agent.id, reason='pulled')
        return ticket
```

`FOR UPDATE SKIP LOCKED` is the magic. Two agents calling at the same instant: each gets a different ticket, because the lock on the first ticket makes the second query skip it. This is textbook Postgres.

---

### 7. SLA monitoring, in summary

The details are in `question.md` Step 7. The short version:

- One row per ticket in `sla_targets`. The deadline stored as a wall-clock timestamp, precomputed with `add_business_time` against the right schedule.
- A sweep worker runs every 30 seconds. Two narrow queries: one for `first_response_due`, one for `resolution_due`.
- On breach, emit `sla.first_response_breached` or `sla.resolution_breached`.
- An escalation consumer reads the events and applies the policy (notify, reassign, page).
- Pause and resume adjust the deadline by the paused duration.

> Why a worker and not per-ticket timers? With 150k open tickets, per-ticket scheduled callbacks become a nightmare. You'd have 150k cancellable timers, and every pause/resume/resolve must cancel cleanly. Sweep is boring but reliable. The cost is up to 30 seconds of breach-detection lag, which nobody notices.

---

### 8. Architecture, all together

```
   Email   Web form   Chat   Slack   SMS
     |       |         |       |      |
     +-------+---------+-------+------+
                       |
                       v
              +------------------------+
              |  Intake Adapters       |  parse, thread, normalize
              |  (one per channel)     |  emit TicketIntakeEvent
              +-----------+------------+
                          |
                          v
              +------------------------+
              |  Ticket Service        |  state machine,
              |  (stateless pods)      |  validates transitions,
              |                        |  writes to Postgres
              +-+----------+---------+-+
                |          |         |
                |          |         +-----> +-------------------+
                |          |                 |  Assignment Svc   |
                |          |                 |  (skill + load,   |
                |          |                 |   pull fallback)  |
                |          |                 +-------------------+
                |          |
                |          +-----------------> +-------------------+
                |                              |  SLA Worker       |
                |                              |  (sweeps every    |
                |                              |   30s, pauses     |
                |                              |   on waiting +    |
                |                              |   off-hours)      |
                |                              +-------------------+
                v
        +---------------------+
        |  Postgres            |
        |   tickets            |
        |   ticket_messages    |
        |   assignments        |
        |   sla_targets        |
        |   ticket_audit       |
        +----------+-----------+
                   |  CDC / outbox
                   v
        +---------------------+
        |  Kafka               |
        |   ticket.created     |
        |   ticket.replied     |
        |   ticket.assigned    |
        |   sla.breached       |
        +-+-------+--------+---+
          |       |        |
          v       v        v
    +--------+ +--------+ +--------------+
    | Index  | | Notify | | Cache        |
    | to ES  | | (email,| | invalidator  |
    |        | | Slack, | | (Redis)      |
    |        | | push)  | |              |
    +---+----+ +--------+ +------+-------+
        |                        |
        v                        v
    +-----------+         +--------------+
    | Search    |         | Agent +      |
    | (ES/OS)   | <-- KB  | customer     |
    |           | queries | dashboards   |
    +-----------+         +--------------+
```

Five things worth noticing while reading this:

- **The Ticket Service is the only writer to Postgres.** Everything else (notifications, indexing, cache invalidation) is downstream of Kafka. If the notifier is down, tickets still get created and assigned. Emails just queue up.

- **Intake adapters track their own progress.** Each one stores its "last processed offset" (last IMAP UID, last chat session pointer) so it can resume after a crash without losing or duplicating messages.

- **The Ticket Service is stateless.** State lives in Postgres. You can roll it during the day with zero impact.

- **The SLA worker is separate.** A slow sweep can't stall the API. At small scale it's one pod. At large scale it shards by `ticket_id` hash across N pods.

- **Audit lives in two tiers.** Last 90 days in Postgres for fast queries. Older events in S3 Parquet for compliance queries via Athena.

---

### 9. A ticket from email to close, end to end

Here is what happens when an email lands and runs through resolution.

```
 Customer   Mail      Intake    Ticket    Assign    Postgres   Kafka   Notifier
   email    server    Adapter   Service   Service                      + Outbound
     |        |         |         |         |          |        |        |
     | send   |         |         |         |          |        |        |
     +------->|         |         |         |          |        |        |
     |        | poll    |         |         |          |        |        |
     |        +-------->|         |         |          |        |        |
     |        |         | parse + |         |          |        |        |
     |        |         | thread  |         |          |        |        |
     |        |         | check   |         |          |        |        |
     |        |         | (no     |         |          |        |        |
     |        |         |  match) |         |          |        |        |
     |        |         +-------->|         |          |        |        |
     |        |         |         | BEGIN TX|          |        |        |
     |        |         |         +--------------------+        |        |
     |        |         |         | INSERT tickets              |        |
     |        |         |         | INSERT ticket_messages      |        |
     |        |         |         | INSERT sla_targets          |        |
     |        |         |         | INSERT ticket_audit         |        |
     |        |         |         | COMMIT                      |        |
     |        |         |         |<---------------------+      |        |
     |        |         |         | assign? |          |        |        |
     |        |         |         +-------->|          |        |        |
     |        |         |         |         | lock     |        |        |
     |        |         |         |         | team,    |        |        |
     |        |         |         |         | pick     |        |        |
     |        |         |         |         | agent    |        |        |
     |        |         |         |<--------+          |        |        |
     |        |         |         | emit ticket.created          |        |
     |        |         |         | emit ticket.assigned         |        |
     |        |         |         +----------------------------->|        |
     |        |         |         |                              +------->| Slack DM
     |        |         |         |                              +------->| email ack
     |                                                                     |
     | ----- agent reads dashboard, replies -----                          |
     |                                                                     |
     |        |         |         | POST /messages (public)               |
     |        |         |         |<----- agent UI                         |
     |        |         |         | BEGIN TX                              |
     |        |         |         | INSERT ticket_messages                |
     |        |         |         | UPDATE first_response_at              |
     |        |         |         | UPDATE sla_targets                    |
     |        |         |         | INSERT ticket_audit                   |
     |        |         |         | COMMIT                                |
     |        |         |         | emit ticket.replied                   |
     |        |         |         +-------------------------------------->|
     |        |         |         |                              +------->| outbound
     |<----------------------------------------------------------------+ | email to
     |                                                                   | customer
     |                                                                   |
     | --- customer accepts, agent marks resolved -> closed ---          |
     |                                                                   |
```

Recording a state change follows the same shape. The Ticket Service validates the transition, opens a transaction, updates the ticket row, writes audit, and (if relevant) updates SLA. The escalation flow looks the same: the SLA worker emits `sla.breached`, an escalation consumer reads it and calls back into the Ticket Service to reassign.

Reads have their own short flow:

1. Agent UI hits the API.
2. API checks Redis (`agent:{id}:open_tickets`).
3. Cache hit: return immediately.
4. Cache miss: query a Postgres read replica, populate cache with 60-second TTL, return.

Cache invalidation happens via Kafka events. When a ticket transitions, the invalidator deletes the affected agent's cache key.

Target latencies:

- Web form intake: P99 ~500ms (email is async, customer doesn't wait)
- Agent reply: P99 ~200ms
- Dashboard read: P99 ~50ms
- Search: P99 ~200ms

---

### 10. Scaling: 10 users to 1 million

Each stage adds something driven by a real pain, not anticipated throughput. Build nothing before it hurts.

#### Stage 1: 10 to 100 users, 3 agents, 50 tickets/day

One Postgres on a t3.medium. One Ticket Service process. One IMAP poller for Gmail. Assignment is round-robin: a `next_agent_index` integer in Postgres. No cache, no search, no queue, no replicas.

SLA timer is a cron job running `SELECT ...` every minute. Notifications go out as inline SendGrid HTTP calls when state changes. About $100/month.

Postgres is bored at 50 writes/day. The one person reading reports runs `psql` directly.

#### Stage 2: 500 tickets/day, 20 agents

Something breaks. Marketing adds a web form (need a second adapter). Billing and technical agents diverge (need skill-based routing instead of round robin). Paying customers got a 1-hour SLA, but you have no way to know when you're about to breach.

Fixes:

- Add a web form intake adapter and a chat adapter.
- Each adapter normalizes to a common `TicketIntakeEvent`.
- Add skill-based assignment. Tag tickets at intake with regex (`"invoice"` -> billing). Tag agents with skill arrays. Filter candidates by skill, load-balance within.
- Add a dedicated SLA worker that sweeps every 30 seconds.
- Add the `ticket_audit` table for history.
- Add `visibility: internal` for agent-only notes.

Still no read replicas, no Kafka, no Elasticsearch. About $300/month.

#### Stage 3: 5,000 tickets/day, 200 agents, multiple teams

Several things break at once:

- Agent dashboards take 3 seconds. The combined query for "my tickets + my team's queue + urgent count" is slow.
- Postgres full-text search returns in 4 seconds when queries match thousands of tickets.
- The SLA worker takes 45 seconds for one sweep over 30k open tickets. Some breaches get detected late.
- Each team wants its own routing rules and SLA policy.
- Agents want real-time updates when a teammate takes a ticket from the queue.

Fixes:

- Move team and SLA policies into a `team_configs` table editable through an admin UI.
- Bring in Elasticsearch via CDC from Postgres (Debezium -> Kafka -> ES indexer). Search drops from 4s to 100ms.
- Add two Postgres read replicas. Dashboards read from replicas. Writes go to primary. About 1s replication lag is fine.
- Add Redis cache (`agent:{id}:open_tickets`, `team:{id}:queue_summary`) invalidated by Kafka events.
- Bring in Kafka with topics: `ticket.created`, `ticket.replied`, `ticket.assigned`, `ticket.status_changed`, `sla.breached`. Consumers are the indexer, notifier, cache invalidator, analytics.
- Add a WebSocket push service so dashboards get live updates instead of polling.
- Shard the SLA worker by `ticket_id` hash across N pods.
- Make business hours a first-class concept. Each team gets a `business_hours_id`. Each customer can have a timezone. SLA computation runs `add_business_time` against the right schedule.

Cost: $3-5k/month.

#### Stage 4: 50,000 tickets/day, 2,000 agents, 4 regions

New problems:

- One Postgres holds all tickets across all teams. A long analytical query stalls the Ticket Service.
- EU mailbox intake from US-East adds 150ms latency on every event.
- Email parsing for 50k inbound messages a day starves the single email adapter's CPU during outage bursts.
- Skill-based router's hand-maintained keyword rules misroute about 10% of tickets.
- Enterprise customers complain their tickets can't live in US-East (data residency).

Fixes:

- Shard the tickets DB by `team_id`. Each shard is its own Postgres cluster. Cross-team queries go through a coordinator that scatter-gathers (rare).
- Deploy per region (US, EU, APAC, AU). Each region has its own intake adapters, Ticket Service, DB shards, Redis, Kafka. Customers route to nearest region. Enterprise customers with residency requirements pin to a specific region.
- Split intake into a "receiver" tier that buffers raw messages to Kafka, and a "parser" tier that consumes from Kafka and emits structured events. Receivers stay light. CPU-bound parsers scale independently.
- Replace keyword rules with a classifier (BERT or even logistic regression on TF-IDF) trained on historical ticket-to-team assignments. Humans can override. Corrections become training data.
- Add a cross-region read-only API so a US agent can see an EU ticket with explicit residency checks.
- Surface KB suggestions at intake (deflect) and at agent compose (insert articles into replies).
- Tier audit storage: hot 90 days in Postgres, older in S3 Parquet via Athena.

The architecture hasn't fundamentally changed since Stage 3. Same shape per region. $30-80k/month depending on region count, agents, and attachment storage.

#### Beyond 1M users

You're selling this as SaaS. Zendesk, Freshdesk, Intercom scale. Concerns shift from per-customer correctness to multi-tenant isolation, per-tenant rate limits, billing, and an integrations marketplace. The base architecture above stays. You wrap it in tenant isolation and a customer-facing admin product.

---

### 11. Reliability

**Intake adapters use at-least-once.** For IMAP, track `last_processed_uid` per mailbox. Commit only after the message is safely staged. On restart, resume from the last UID. Reprocessed messages get deduped by the unique index on `ticket_messages.inbound_message_id`. For SES webhooks, return 200 only after the message is staged so SES retries on 5xx.

**The SLA worker is stateless and idempotent.** State lives in `sla_targets`. On restart, the next sweep picks up where the previous one left off. Worst case is 30 seconds of delay in detecting a breach. Run two workers active-active behind per-shard advisory locks.

**Assignment races.** Two `assign_ticket` calls for the same team at the same time: serialized by the team-level row lock. Two pull-queue calls for the same ticket: serialized by `SELECT ... FOR UPDATE SKIP LOCKED`. Two agents acting on the same ticket: serialized by `SELECT ... FOR UPDATE` on the ticket row.

**Notification failures.** Consumers run at-least-once with exponential backoff. Dead-letter topic after max retries, watched by an on-call dashboard.

**Outbound email bounces.** SES delivers a bounce notification to a Kafka topic. The bounce consumer marks the ticket `outbound_failed`, switches status to "needs alternative contact," and alerts the agent.

**Search index drift.** Elasticsearch is downstream of Postgres via CDC. If indexing lags, search returns stale results. A nightly reconciliation job compares row counts and triggers re-indexing for any mismatch.

**Postgres failover.** Primary dies. Replica promotes. In-flight transactions roll back. Clients retry with idempotency keys. Workers reconnect within seconds.

---

### 12. Observability

| Metric | Why it matters |
|--------|----------------|
| `tickets.created.rate` by channel | Sudden drop = an adapter is broken. Spike = outage or marketing blast. |
| `tickets.in_flight.count` by team and status | Slow rise = team is drowning. |
| `time_to_first_response` p50/p95 | The headline customer-facing SLO. |
| `time_to_resolution` p50/p95 by priority | The other headline SLO. |
| `sla.breach.rate` by priority and team | Per-team operational health. |
| `agent.utilization` (active / max concurrent) | Identifies overloaded and underused agents. |
| `assignment.latency` p99 | If high, Assignment Service is contended. Check team lock wait. |
| `intake.parse.latency` per channel | If email parsing latency rises, adapter or dependency is slow. |
| `intake.threading.success_rate` | Fraction of emails that thread into an existing ticket. A drop means headers are being stripped. |
| `kb.suggestion.deflection.rate` | Customers who saw KB suggestions and closed the form without submitting. Self-service ROI. |
| `reopen.rate` | High = agents closing too aggressively. |
| `audit.write.lag` | Audit must not lag. Alert at >5s. |

Page on: any adapter offline >2 min, SLA worker not running, Postgres primary unreachable.

Ticket on: `time_to_first_response` p95 regression >30%, `sla.breach.rate` doubles, reopen rate above 10%.

---

### 13. Follow-up answers

These are the questions a senior interviewer is listening for. The depth is in the *why*.

**1. Email threading when subject is stripped.**

`In-Reply-To` and `References` headers are the primary signal. About 85% of replies hit this path. Index on `ticket_messages.inbound_message_id` makes lookup a single index hit. Subject prefix `[TKT-1234]` is the fallback for when corporate mail proxies strip headers. As a last resort (off by default), match same-sender + similar-subject + within-N-days. If all three fail, a new ticket opens and the agent merges via a "merge into" action.

**2. Intake adapter crashes mid-batch.**

The adapter never deletes messages from IMAP until they're safely staged. Track `last_processed_uid` per mailbox, updated only after successful staging. On restart, resume from `last_processed_uid + 1`. Reprocessed messages get deduped by the unique index on `inbound_message_id`. Worst case is a few seconds of duplicate parsing work. No message loss. No duplicate tickets.

**3. Two agents claim the same queued ticket.**

The pull query is `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` inside a transaction. Agent A's transaction locks TKT-100 and updates `assignee_id`. Agent B's identical query at the same instant skips TKT-100 (already locked) and gets TKT-101. Both commit. No conflict. This is the canonical use case for `SKIP LOCKED`.

**4. SLA business hours across timezones.**

Default: the team's business hours, stored as `business_hours_id` on the team config (like "US-West 9-6 weekdays + US holidays"). Override per contract: enterprise customers with a 24/7 contract get `business_hours_id = always`. The contract policy lives on the customer record. At ticket creation, SLA computation picks the customer's policy if set, otherwise the team's. For "follow the sun," set the effective schedule to "always" and let the assignment service route to the on-shift region.

**5. Reopen after long silence.**

Within the 7-day reopen window from `closed_at`, a customer reply reactivates the original: status goes to `reopened`, then immediately to `in_progress`. `reopen_count` increases. SLA clock resets, optionally to a tighter "reopen SLA" (4 hours instead of 24).

Outside the window, a new ticket opens with `parent_ticket_id` pointing to the closed one. The agent UI surfaces the parent: "This customer had a similar issue last year; view prior ticket."

**6. New issue from a customer with an open ticket.**

Default is open a new ticket. The unrelated-issue case is more common than the related-follow-up case. Detection uses the threading logic: `In-Reply-To` pointing to an agent message in the existing ticket -> append. Subject mentions the ticket ref -> append. Otherwise -> new ticket. If the agent realizes two tickets are about the same thing, they merge.

**7. Agent vacation handover.**

The agent sets `out_of_office: {start, end, delegate_id}` in their profile. On the start date, a job runs:

```sql
SELECT ticket_id FROM tickets
WHERE assignee_id = ? AND status NOT IN ('closed', 'resolved')
```

It reassigns each ticket to the delegate (or to the team queue if no delegate), records `reason: vacation_handover` in `assignments`, and notifies the delegate via Slack and email. On return, tickets stay where they were assigned during vacation. The returning agent gets a digest of what happened.

For new incoming tickets where the OOO agent would have been the round-robin pick: skip them in the rotation while OOO.

**8. Spam at the front door.**

Layered defense:

- SPF/DKIM/DMARC at SMTP rejects obviously spoofed mail.
- SpamAssassin or AWS SES spam check on accepted mail sends high-score mail to a quarantine mailbox, not the ticket DB.
- Rate limit per sender domain/IP drops to quarantine above N messages per minute.
- Allowlist of known customer domains bypasses aggressive filtering for paying customers.
- Periodic human review of quarantine catches false positives.

The 1% that gets through is closed as `spam` by agents with a one-click action.

**9. KB suggestions at intake.**

Pre-submit: as the customer types, debounce 500ms then call `POST /api/v1/kb/suggest` with the current subject + body. The endpoint queries Elasticsearch (top-3 articles by relevance) and returns titles + snippets within 50ms. Display inline: "Maybe one of these solves your issue?" Track if the customer clicks (`kb.suggestion.click`), abandons the form (`kb.suggestion.deflection`), or submits anyway. Deflection is the win metric. Mature setups deflect 20-40% of would-be tickets.

**10. "Average time to resolve" is wrong.**

Three common bugs:

1. **It includes `waiting_on_customer` time**, but the agent isn't working then. Subtract time spent in that status.
2. **It's wall-clock and ignores business hours.** A ticket created Friday 5 PM and resolved Monday 10 AM isn't 65 hours. It's about 3 business hours.
3. **It counts spam and duplicate tickets**, which close instantly and skew the average down. Exclude `final_state IN ('spam', 'duplicate')`.

The correct computation: `sum of business-hours-active time per ticket / count of resolved tickets, excluding spam and duplicate`. Show p50 and p95 alongside the average. The average alone hides long-tail outliers.

---

### 14. Trade-offs worth saying out loud

**Push vs pull assignment.** Push (round robin / skill / load) gives every ticket an owner instantly. Better for SLA accountability and "you have been assigned" emails. Pull is better for agent autonomy and avoids "assigned to an agent who just logged off." Most production setups push by default with pull as fallback when the assigned agent is unavailable.

**Canned responses vs ML-suggested replies.** Canned responses are agent-curated templates with placeholders. Predictable and controllable. Agents stay in the driver's seat. ML-suggested replies save typing but risk wrong suggestions that the agent rubber-stamps. Best practice: keep canned responses for the top 50 known scenarios, layer suggestions on top for the long tail, never auto-send.

**Postgres FTS vs Elasticsearch.** Postgres full-text search gets you to about 1M tickets cheaply. Elasticsearch is a separate cluster to operate but gives faceted search, fuzzy matching, and relevance tuning. Switch when FTS latency exceeds 500ms or when product asks for facets.

**Custom build vs Zendesk / Freshdesk.** If your support volume is small, buy. Build-vs-buy tips toward build when you have unusual integrations, a workflow that doesn't fit the SaaS model (regulated industries, embedded support inside another product, deep cross-system automation), or volume large enough that the SaaS bill exceeds engineering cost. Below that line, don't build this.

**Per-ticket scheduled timers vs sweep worker.** Per-ticket timers sound clean ("real-time!") but cancellation on resolve/pause becomes brittle. Sweep worker is boring but reliable. Sweep wins at scale.

---

### 15. Common interview mistakes

Most weak answers fall into one of these:

**Tickets as CRUD with a `status` column and hardcoded transitions.** If `status` is a free-form string and any agent can flip to any value, you have a Trello clone, not a help desk. The state machine *is* the system.

**No intake adapter layer.** Skipping straight to "the API takes JSON" misses the entire email-parsing-and-threading problem. That's the single hardest part of the system.

**Round-robin assignment as the default at any scale.** Works for the smallest team. Breaks at 20+ agents where skill matters and at 100+ where load matters. Acknowledge the trade-off and offer skill-based as the next step up.

**Wall-clock SLA with no business hours.** Most candidates skip this. Real customers won't tolerate a 24-hour SLA that expires Sunday at 3 AM.

**No SLA pause for `waiting_on_customer`.** Without pause, every ticket that needs customer info eventually breaches through no fault of the agent. The SLA system loses credibility and agents game it ("close and reopen later").

**Per-ticket scheduled timer.** Tempting because it's "real-time." A nightmare at scale: 150k pending callbacks, cancellation races, fragile failure modes. Sweep is the right answer.

**No reopen handling.** Customer replies to a closed ticket, a new ticket opens, conversation history is lost. Build the 7-day reopen window from day one.

**Email threading via subject only.** Subjects get rewritten, prefixed, translated. Use `In-Reply-To` as the primary signal, subject as fallback.

**No assignment race control.** Two agents claim the same ticket. Without `SKIP LOCKED` (pull) or row locks (push), you get split-brain assignments.

**Notifications glued to the ticket service.** Tickets emit events. The notification service consumes them. Coupling them means you can't swap Slack for Teams without touching the core.

If you can name 7 of these 10 unprompted, you're interviewing at senior or staff level. The three that separate strong from average: state-machine over CRUD, threading by headers not subject, and business-hour SLA pausing. Those are the answers a senior architect listens for.
