## Solution: Help Desk Ticketing System

### TL;DR

A help desk is a state machine with a messy front door. The front door is the hard part. Emails arrive over IMAP with broken subject lines. Chats arrive over websockets and end when the user closes the tab. Web forms post JSON. Slack messages come through a bot token. SMS comes through Twilio. Each channel needs an adapter that parses, threads replies into existing tickets, and emits a unified intake event.

Once a ticket exists, it's a small state machine: `new → assigned → in_progress → (waiting_on_customer ↔ in_progress)* → resolved → closed`, with reopen and reassignment as common edges. The Ticket Service enforces transitions and writes to Postgres. An Assignment Service decides who owns each ticket (skill-based plus load-balanced, with queue-pull as fallback). An SLA worker sweeps the table every 30 seconds, pauses outside business hours, and fires escalation events on breach.

Scale isn't the headline. Even at 50k tickets per day with 2000 agents you only push ~15 writes per second. The interesting engineering is correctness: email threading, SLA pause/resume across business hours and timezones, agent assignment race conditions, and reopen after silence.

### 1. Clarifying questions, recap

Covered in `question.md`. The two questions that change the design most are *channels* (email is messy because of threading; adding chat or Slack just adds adapters) and *SLA semantics* (wall-clock is trivial, business-hours across customer timezones is a real engineering project and the source of most reporting bugs).

### 2. Capacity, with the math

**Small startup (50/day, 3 agents).** ~50 writes/day across tickets and messages = 0.003 writes/sec. Single Postgres on a t3.small. No cache, no search index needed. ~100 active tickets at any moment, indexes are decorative at this scale.

**Enterprise (50k/day, 2000 agents).** Ticket creates land at ~0.6/sec sustained, ~3/sec at peak. Messages run ~5/sec sustained and 15/sec at peak (8 messages per ticket on average). Active tickets sit around 150k at any moment. Agent dashboards plus customer portal generate ~40/sec sustained and ~120/sec at peak. Five years of data is ~5 TB of text and ~50 TB of attachments in S3.

The system is small in raw QPS. The architecture exists for state correctness, fair assignment under concurrency, SLA accuracy, and operational visibility, not for throughput.

### 3. API

Customer-facing intake:

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

| Status | Meaning | Body |
|--------|---------|------|
| 201 Created | New ticket | `{"ticket_id": "tkt_1234", "status": "new", "public_url": "..."}` |
| 200 OK | Idempotency key matched a recent create; return existing | same shape |
| 400 Bad Request | Validation failure | error detail |
| 429 Too Many Requests | Rate limited (per IP, per customer) | with `retry_after` |

Agent reply or message append:

```
POST /api/v1/tickets/{ticket_id}/messages
{
  "body": "Can you share the error you see?",
  "visibility": "public" | "internal",
  "attachments": [...]
}
```

A `public` message also pushes outbound (email reply, chat push, Slack DM) via the channel the ticket lives on. An `internal` message is an agent-to-agent note that the customer never sees.

Assign or reassign:

```
POST /api/v1/tickets/{ticket_id}/assign
{
  "team_id": "billing",
  "agent_id": "agt_bob"        # optional; null lets Assignment Service pick
}
```

Status change:

```
POST /api/v1/tickets/{ticket_id}/status
{
  "status": "in_progress" | "waiting_on_customer" | "resolved" | "closed" | "spam" | "duplicate",
  "reason": "Customer confirmed fix"
}
```

The Ticket Service validates the transition against the state machine. Invalid transition returns `422 Unprocessable Entity`.

Search:

```
GET /api/v1/tickets?status=open&assignee=me&priority=high
GET /api/v1/tickets/search?q=login+mobile&customer_id=cust_abc
```

Search hits Elasticsearch. Filtered lookup by indexed columns hits the read replica.

A few small but load-bearing choices. `Idempotency-Key` is required on create because SES retries, mobile clients retry on timeout, and web form double-clicks all happen; without the key you create duplicate tickets. Messages are the unit of conversation, not tickets; tickets group messages, messages carry content. The `visibility: internal` flag is critical because the outbound channel adapter checks it before sending, which is what keeps internal-only war-room notes from accidentally reaching the customer.

### 4. Data model

Five tables. Two large, three small.

```sql
CREATE TABLE tickets (
    ticket_id           UUID PRIMARY KEY,
    public_ref          TEXT NOT NULL UNIQUE,      -- "TKT-1234" shown to customer
    subject             TEXT NOT NULL,
    customer_id         TEXT,
    customer_email      TEXT,
    channel             TEXT NOT NULL,              -- 'email' | 'web_form' | 'chat' | 'slack' | 'sms'
    channel_thread_id   TEXT,                       -- channel-native ID for threading
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
    parent_ticket_id    UUID,
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
    author_type         TEXT NOT NULL,              -- 'customer' | 'agent' | 'system'
    author_id           TEXT,
    visibility          TEXT NOT NULL DEFAULT 'public',
    body                TEXT NOT NULL,
    body_format         TEXT NOT NULL DEFAULT 'plain',
    inbound_channel     TEXT,
    inbound_message_id  TEXT,                       -- e.g., email Message-ID header
    in_reply_to         TEXT,                       -- email In-Reply-To header
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
    agent_id            TEXT,                       -- null = team queue, not yet picked
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    unassigned_at       TIMESTAMPTZ,
    assigned_by         TEXT,
    reason              TEXT                        -- 'initial' | 'reassigned' | 'escalation' | 'vacation_handover'
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

A few choices worth defending out loud.

`tickets` is the source of truth for current state. The composite indexes serve the two dominant queries: "show me my pending tickets" (`assignee_id, status`) and "show me my team's queue" (`team_id, status, priority`).

The unique index on `ticket_messages.inbound_message_id` is doing real work. When the same email gets reprocessed after an adapter crash, the second insert fails with a unique-violation, the message is silently deduped, and no duplicate conversation appears on the ticket.

`assignments` is history, not current. The current assignee lives on the ticket row for fast reads; the history sits here for audit and analytics (mean time to first assignment, handoff counts).

`sla_targets` is separated from `tickets` so the SLA worker queries a narrow table without scanning ticket bodies.

`ticket_audit` is append-only. Captures every state transition, every reply, every assignment change. Retention is 5 to 7 years.

Postgres, not Mongo or DynamoDB. State-machine transitions need transactional guarantees (status change plus audit insert in one transaction). Postgres handles JSON metadata fine via JSONB. You only outgrow Postgres at this workload if you sell to thousands of multi-tenant customers, and even then sharding by `team_id` keeps you on Postgres per shard.

### 5. The intake adapter and email threading

Email is the messy channel. Here's what the email adapter does on a new message:

```python
def handle_inbound_email(raw_email):
    msg = parse_email(raw_email)

    if is_spam(msg):
        record_dropped(msg, reason='spam')
        return

    # Layer 1: In-Reply-To / References headers (RFC 5322).
    parent_msg_id = msg.headers.get('In-Reply-To')
    refs = msg.headers.get('References', '').split()
    candidate_msg_ids = [parent_msg_id] + refs if parent_msg_id else refs

    existing_ticket = None
    for mid in candidate_msg_ids:
        m = db.query("SELECT ticket_id FROM ticket_messages WHERE inbound_message_id = ?", mid)
        if m:
            existing_ticket = m.ticket_id
            break

    # Layer 2: ticket reference in subject (e.g., "[TKT-1234]").
    if not existing_ticket:
        ref = extract_ticket_ref(msg.subject)
        if ref:
            existing_ticket = db.query(
                "SELECT ticket_id FROM tickets WHERE public_ref = ?", ref)

    # Layer 3: heuristic (same sender + similar subject + recent). Off by default.

    if existing_ticket:
        emit_event("ticket.reply_received", existing_ticket, msg)
    else:
        emit_event("ticket.intake", channel='email', payload=normalize(msg))
```

The three layers, in priority order. `In-Reply-To` and `References` headers are the RFC 5322 standard; most clients populate them and this catches ~85% of replies. The `[TKT-1234]` subject-line tag survives even when aggressive mail proxies strip headers; that's another ~10%. The heuristic (same sender plus similar subject plus recent window) is dangerous because it cross-contaminates tickets, so it's off by default. The remaining ~5% open new tickets and an agent merges duplicates manually.

### 6. Assignment

```python
def assign_ticket(ticket):
    team = pick_team(ticket)        # triage rules: tag/keyword/channel
    candidates = list_agents_in_team(team)

    if ticket.required_skills:
        candidates = [a for a in candidates
                      if set(ticket.required_skills).issubset(a.skills)]

    if not candidates:
        return Assignment(team_id=team, agent_id=None)  # falls into team queue

    candidates = [a for a in candidates if a.is_available(now())]

    with db.transaction():
        db.lock_for_update("teams", team)
        loads = db.query("""
          SELECT agent_id, COUNT(*) AS open_count, MAX(assigned_at) AS last_assigned
          FROM assignments
          WHERE agent_id = ANY(?) AND unassigned_at IS NULL
          GROUP BY agent_id
        """, [a.id for a in candidates])

        chosen = min(candidates, key=lambda a: (
            loads.get(a.id).open_count,
            loads.get(a.id).last_assigned
        ))

        db.insert("assignments",
                  ticket_id=ticket.id, team_id=team,
                  agent_id=chosen.id, reason='initial')
        db.update("tickets", ticket.id, team_id=team, assignee_id=chosen.id)
        return Assignment(team_id=team, agent_id=chosen.id)
```

The serial lock per team is the critical detail. Without it, two parallel `assign_ticket` calls for the same team can pick the same agent for two tickets, both believing the agent has the lowest load. The team lock makes assignments strictly sequential within a team while staying parallel across teams.

For the queue-pull strategy (agents click "Next Ticket"), the logic inverts:

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
        db.update("tickets", ticket.id, assignee_id=agent.id, status='assigned')
        db.insert("assignments", ticket_id=ticket.id,
                  agent_id=agent.id, reason='pulled')
        return ticket
```

`FOR UPDATE SKIP LOCKED` is the right primitive. Two agents calling at the same instant: each gets a different ticket because the lock on the first ticket makes the second query skip it.

### 7. SLA monitoring, in summary

Covered in detail in `question.md` Step 6. The implementation in one paragraph: one row per ticket in `sla_targets`, with the deadline stored as a wall-clock timestamp precomputed against the right business-hours schedule. A sweep worker runs every ~30 seconds with two narrow queries, one for `first_response_due` and one for `resolution_due`. On breach it emits `sla.first_response_breached` or `sla.resolution_breached`, and an escalation consumer reads those and applies the policy (notify, reassign, page). Pause and resume adjust the deadline by the paused business-time duration, not the wall-clock duration.

### 8. Architecture

Here's the whole picture. Drawn small enough to fit one screen.

```
   Email (IMAP/SES)  Web form  Chat (websocket)  Slack  Twilio SMS
        │              │            │              │         │
        └──────────────┴────────────┴──────────────┴─────────┘
                                │
                                ▼
                       ┌──────────────────────┐
                       │  Intake Adapters     │  parse, thread, normalize
                       │  (one per channel)   │  emit TicketIntakeEvent
                       └─────────┬────────────┘
                                 │
                                 ▼
                       ┌──────────────────────┐
                       │  Ticket Service      │  state machine, validates
                       │  (stateless pods)    │  transitions, writes to DB
                       └──┬─────┬──────┬──────┘
                          │     │      │
                          │     │      └───────► ┌──────────────────┐
                          │     │                │  Assignment      │
                          │     │                │  Service         │
                          │     │                │  (skill + load,  │
                          │     │                │   queue fallback)│
                          │     │                └──────────────────┘
                          │     │
                          │     └─────────────► ┌──────────────────┐
                          │                     │  SLA Timer       │
                          │                     │  Worker          │
                          │                     │  (sweeps every   │
                          │                     │   30s, pauses    │
                          │                     │   on OOO + OOH)  │
                          │                     └──────────────────┘
                          ▼
                  ┌──────────────────┐
                  │  Postgres        │
                  │   tickets        │
                  │   ticket_messages│
                  │   assignments    │
                  │   sla_targets    │
                  │   ticket_audit   │
                  └──┬───────────────┘
                     │  CDC / outbox
                     ▼
                  ┌──────────────────┐
                  │  Kafka           │
                  │   ticket.created │
                  │   ticket.replied │
                  │   ticket.assigned│
                  │   sla.breached   │
                  └─┬──────┬──────┬──┘
                    │      │      │
                    ▼      ▼      ▼
            ┌──────────┐ ┌──────────┐ ┌──────────────┐
            │ Indexer  │ │Notifier  │ │  Read Cache  │
            │ → ES     │ │(email,   │ │  Invalidator │
            │          │ │ Slack,   │ │  (Redis)     │
            │          │ │ push)    │ │              │
            └────┬─────┘ └──────────┘ └──────┬───────┘
                 │                            │
                 ▼                            ▼
            ┌──────────────┐           ┌──────────────┐
            │  Search Index│  ◄── KB   │ Agent + Cust │
            │  (ES/OS)     │   queries │  Dashboards  │
            └──────────────┘           └──────────────┘
```

A few things to notice while reading this. The Ticket Service is the only writer to Postgres. Everything else is downstream of Kafka: notifications, search indexing, cache invalidation. If the notifier is down, tickets still get created and assigned; emails just queue up.

Intake adapters are stateful in one narrow sense: each tracks per-channel offsets (last IMAP UID, last chat session pointer) so it can resume after a crash. The Ticket Service itself is stateless; you can roll it mid-day.

The SLA worker is a separate process so a slow sweep can't stall the API. At small scale it's a singleton; at large scale it shards by `ticket_id` hash across N workers, each holding a per-shard advisory lock.

Audit lives in two tiers. Recent 90 days in Postgres for fast investigations. Older events in S3 Parquet for compliance queries via Athena.

### 9. The ticket lifecycle, end to end

Here's what happens when an email lands and runs through resolution.

```
 Customer    Email      Intake    Ticket    Assignment   Postgres   Kafka   Notifier
   email     channel    Adapter   Service     Service                       + Outbound
     │          │          │         │           │           │        │        │
     │ send     │          │         │           │           │        │        │
     ├─────────►│          │         │           │           │        │        │
     │          │ IMAP     │         │           │           │        │        │
     │          │ poll/SES │         │           │           │        │        │
     │          ├─────────►│         │           │           │        │        │
     │          │          │ parse + │           │           │        │        │
     │          │          │ thread  │           │           │        │        │
     │          │          │ check   │           │           │        │        │
     │          │          │ (no     │           │           │        │        │
     │          │          │  match) │           │           │        │        │
     │          │          ├────────►│           │           │        │        │
     │          │          │         │ BEGIN TX  │           │        │        │
     │          │          │         ├─────────────────────► │        │        │
     │          │          │         │ INSERT tickets        │        │        │
     │          │          │         │ INSERT ticket_messages│        │        │
     │          │          │         │ INSERT sla_targets    │        │        │
     │          │          │         │ INSERT ticket_audit   │        │        │
     │          │          │         │ COMMIT                │        │        │
     │          │          │         │◄─────────────────────┤         │        │
     │          │          │         │ assign? │           │          │        │
     │          │          │         ├────────►│           │          │        │
     │          │          │         │         │ pick team │          │        │
     │          │          │         │         │ + agent   │          │        │
     │          │          │         │         │ (locked)  │          │        │
     │          │          │         │◄────────┤           │          │        │
     │          │          │         │ emit ticket.created │          │        │
     │          │          │         │ emit ticket.assigned│          │        │
     │          │          │         ├──────────────────────────────► │        │
     │          │          │         │                                ├──────► │ Slack DM
     │          │          │         │                                ├──────► │ email ack
     │          │          │         │                                │        │
     │ ─ ─ ─ ─ ─ ─ ─  agent reads dashboard, replies  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
     │          │          │         │           │           │        │        │
     │          │          │         │ POST /messages (public)        │        │
     │          │          │         │◄────── agent UI                │        │
     │          │          │         │ BEGIN TX  │           │        │        │
     │          │          │         │ INSERT ticket_messages         │        │
     │          │          │         │ UPDATE first_response_at       │        │
     │          │          │         │ UPDATE sla_targets             │        │
     │          │          │         │ INSERT ticket_audit            │        │
     │          │          │         │ COMMIT                         │        │
     │          │          │         │ emit ticket.replied            │        │
     │          │          │         ├──────────────────────────────► │        │
     │          │          │         │                                ├──────► │ outbound email
     │◄─────────────────────────────────────────────────────────────────────── │ to customer
     │          │          │         │           │           │        │        │
     │ ─ ─ ─ ─ ─  customer accepts, agent marks resolved → closed  ─ ─ ─ ─     │
     │          │          │         │           │           │        │        │
```

Recording a state change looks almost the same. The Ticket Service validates the transition, opens a transaction, updates the ticket row, writes an audit event, and (if relevant) updates `sla_targets`. The escalation flow uses the same shape: the SLA worker emits `sla.breached`, an escalation consumer reads it and calls back into the Ticket Service to reassign.

Reads have their own short flow. Agent UI hits the API, which checks Redis (`agent:{id}:open_tickets`). On a hit, return immediately. On a miss, query a Postgres read replica with the covering index, populate the cache with a 60-second TTL, return. Cache invalidation happens via Kafka events: when any of those tickets transitions, the invalidator deletes the affected agent's cache keys.

Target latencies: web-form intake P99 ~500ms (email intake is async, the customer doesn't wait). Agent reply P99 ~200ms. Dashboard read P99 ~50ms. Search P99 ~200ms.

### 10. Scaling journey, 10 to 1M users

The system is small in raw QPS even at the top. Each stage adds something driven by a real operational pain, not anticipated throughput.

**Stage 1: 10 to 100 users, 3 agents, 50 tickets/day.** Single Postgres on a t3.medium. One Ticket Service process. One intake adapter polling Gmail IMAP. Assignment is round-robin within the single team, stored as `next_agent_index` in Postgres. No cache, no search, no queue, no replicas. SLA timer is a cron job hitting `SELECT ...` every minute. Notifications go out as inline SendGrid HTTP calls at the moment of state change. About $100/month. Ships in two weeks. Postgres is bored at 50 writes/day; the one person who reads reports runs `psql` directly.

**Stage 2: 500 tickets/day, 20 agents.** Something breaks. The marketing site sprouted a web form (need a second adapter). Billing and technical agents diverge into separate teams (need skill-based routing instead of round robin). Paying customers got a 1-hour SLA but you have no way to know when you're about to breach. Reports want per-channel breakdowns.

Add three intake adapters: email (existing), web form (POST endpoint), chat (websocket terminator). Each normalizes to a common `TicketIntakeEvent`. Add skill-based assignment: tag tickets at intake (regex on "invoice", "refund" → billing), tag agents with skill arrays, filter candidates by skill then load-balance within. Add a dedicated SLA worker that sweeps every 30 seconds and emits a warning at 75% of deadline. Add the `ticket_audit` table for state-change history. Add the `visibility` column for internal-only agent notes.

Still no read replicas, no Kafka, no Elasticsearch. Consumers poll `ticket_audit` every 5 seconds. Postgres full-text search on `ticket_messages.body` handles ~50k tickets fine. ~$300/month.

**Stage 3: 5000 tickets/day, 200 agents, multi-team.** Now several things break at once. Agent dashboard load takes 3 seconds because the combined query for "my tickets plus my team's queue plus count of urgent tickets" is now slow. Postgres FTS returns search in 4 seconds when queries match thousands of tickets. The SLA worker takes 45 seconds for one sweep over 30k open tickets, so some breaches are detected late. Each team wants its own queue, routing rules, and SLA policy, which makes hardcoded YAML unmanageable. Agents want real-time updates when a teammate takes a ticket from the queue.

Move team and SLA policies into a `team_configs` table editable through an admin UI. Bring in Elasticsearch with CDC from Postgres (Debezium → Kafka → ES indexer); search latency drops from 4s to 100ms. Add two Postgres read replicas; dashboards read from replicas, writes go to primary, ~1s replication lag is fine. Add a Redis cache (`agent:{id}:open_tickets`, `team:{id}:queue_summary`) invalidated by Kafka events. Bring in Kafka properly with topics for `ticket.created`, `ticket.replied`, `ticket.assigned`, `ticket.status_changed`, `sla.breached`; consumers are the indexer, notifier, cache invalidator, and analytics pipeline. Add a WebSocket push service so agent dashboards get live updates instead of polling. Shard the SLA worker by `ticket_id` hash across N pods, each owning one slice. Make business hours a first-class concept: each team gets a `business_hours_id`, each customer can have a `timezone`, and SLA computation runs `add_business_time` against the right schedule.

Cost jumps to $3-5k/month for Postgres plus replicas, Redis cluster, Kafka cluster, ES cluster, WebSocket pods.

**Stage 4: 50k tickets/day, 2000 agents across 4 regions.** New problems. One Postgres holds all tickets across all teams, so a long analytical query on the primary stalls the Ticket Service. EU mailbox intake from US-East adds ~150ms latency on every intake event. Email parsing for 50k inbound messages per day starves the single email adapter's CPU during outage bursts. The skill-based router's hand-maintained keyword rules misroute ambiguous tickets ~10% of the time. A few enterprise customers complain that their tickets can't live in US-East for data-residency reasons.

Shard the Tickets DB by `team_id`; each shard is its own Postgres cluster, and rare cross-team queries go through a coordinator that scatter-gathers. Deploy per region (US, EU, APAC, AU), each with its own intake adapters, Ticket Service, DB shards, Redis, Kafka; customers route to the nearest region, enterprise customers with residency requirements pin to a specific region. Split intake into a "receiver" tier that buffers raw messages to Kafka and a "parser" tier that consumes from Kafka and emits structured events; receivers stay light, CPU-bound parsers scale independently. Replace keyword rules with a classifier (fine-tuned BERT or even logistic regression on TF-IDF) trained on historical ticket-text-to-team-assigned pairs; humans can override, corrections become training data. Add a cross-region read-only API so a US agent can see an EU ticket with explicit residency checks. Surface KB suggestions at both intake (to deflect) and agent compose (to insert articles into replies). Tier audit storage: hot 90 days in Postgres, older in S3 Parquet via Athena.

The architecture hasn't fundamentally changed since stage 3. You added regions, sharding, and ML-based routing, but each region's data plane has the same shape. $30-80k/month depending on region count, agent count, attachment storage.

Beyond 1M users: you're selling this as SaaS (Zendesk, Freshdesk, Intercom scale). The concerns shift from per-customer correctness to multi-tenant isolation, per-tenant rate limits, billing, and an integrations marketplace. The base architecture above stays, wrapped in tenant isolation and a customer-facing admin product.

### 11. Reliability

Intake adapters use at-least-once semantics. For IMAP, track the last-processed UID per mailbox in a `mailbox_state` row, committed only after the message is safely staged. On restart, resume from the last committed UID; reprocessed messages get deduped by the unique index on `ticket_messages.inbound_message_id`. For SES inbound webhooks, return 200 only after the message is staged so SES retries on 5xx.

The SLA worker is stateless and idempotent. State lives in `sla_targets`. On restart, the next sweep picks up where the previous one left off. Worst case is a 30-second delay in detecting a breach. Run two workers active-active behind per-shard advisory locks so a death is covered within the lock TTL.

Assignment races. Two `assign_ticket` calls for the same team simultaneously: serialized by the team-level row lock. Two queue-pull calls for the same ticket: serialized by `SELECT ... FOR UPDATE SKIP LOCKED`. Two agents acting on the same ticket: serialized by `SELECT ... FOR UPDATE` on the ticket row inside the transition transaction.

Notification failures. Consumers run with at-least-once delivery, exponential backoff on retry, dead-letter topic after max retries monitored by an on-call dashboard.

Outbound email bounces. SES delivers a bounce notification to a Kafka topic. The bounce consumer marks the ticket `outbound_failed`, switches status to "needs alternative contact", and alerts the assigned agent.

Search index drift. ES is downstream from Postgres via CDC. If indexing lags, search returns stale results. A nightly reconciliation job compares row counts and checksums per index and triggers re-indexing for the affected ticket IDs on mismatch.

Postgres failover. Primary dies, replica gets promoted. In-flight transactions roll back; clients retry with idempotency keys. Worker pods reconnect within seconds. The SLA worker's next sweep handles any tickets that breached during the failover gap.

### 12. Observability

| Metric | Why it matters |
|--------|----------------|
| `tickets.created.rate` by channel | Sudden drop = an intake adapter is broken. Spike = outage or marketing campaign |
| `tickets.in_flight.count` by team and status | Slow accumulation = team is underwater |
| `time_to_first_response` p50/p95 | The headline customer-facing SLO |
| `time_to_resolution` p50/p95 by priority | The other headline SLO |
| `sla.breach.rate` by priority and team | Per-team operational health |
| `agent.utilization` (active tickets / max concurrent) | Identifies overloaded and underused agents |
| `assignment.latency` p99 | If high, Assignment Service is contended; check team lock wait |
| `intake.parse.latency` per channel | If email parsing latency rises, adapter or its dependencies are slow |
| `intake.threading.success_rate` | Fraction of incoming emails that thread into an existing ticket. A drop means headers are being stripped |
| `kb.suggestion.deflection.rate` | Customers who saw KB suggestions and closed the form without submitting. Self-service ROI |
| `reopen.rate` | High = agents closing too aggressively |
| `audit.write.lag` | Audit must not lag. Alert at > 5s |

Page on: any channel's intake adapter offline for >2 min, SLA worker not running, Postgres primary unreachable. File a ticket on: `time_to_first_response` p95 regression >30%, `sla.breach.rate` doubles, reopen rate above 10%.

### 13. Gotchas a senior interviewer is listening for

Some of these only come out when the interviewer asks "what happens if...". The senior candidate brings them up unprompted.

**Timezones for business hours.** US team supporting an EU customer: which clock does the SLA run on? Industry convention is the team's clock by default, the customer's clock if the contract specifies. Mixing them silently is the source of half the SLA disputes in production. Always show both clocks in the agent UI.

**Reopen.** A customer replies to a "resolved" ticket. Naive design opens a new ticket. Right design: within 7 days, reply reopens the original; after 7 days, opens a new ticket linked via `parent_ticket_id` so the agent has context. Reopen also resets the SLA, sometimes to a tighter reopen-specific target.

**Threading via subject alone.** Subjects get rewritten, prefixed, translated. `In-Reply-To` is the primary signal; subject is fallback. Heuristic threading (same sender, similar subject) is dangerous and off by default.

**Agent vacation handover.** Bob has 47 open tickets and starts a 2-week vacation. Manual reassignment is painful. Vacation mode: Bob sets OOO start, end, and delegate; on the start date a job reassigns all his open tickets to the delegate (or to the team queue), records `reason: vacation_handover` in `assignments`, notifies the delegate. On return, tickets stay where they are.

**Spam.** A public support email gets spam. Layered defense: SPF/DKIM/DMARC at SMTP, SpamAssassin scoring with a quarantine mailbox, rate limiter per sender domain/IP, allowlist of known customer domains. The 1% that gets through is closed as `spam` by agents with a one-click action.

**Self-replies and auto-responders.** Your outbound "Ticket received" autoresponder can loop with the customer's autoresponder, generating dozens of messages per minute on the same ticket. Detect via `Auto-Submitted: auto-replied`, `X-Auto-Response-Suppress`, or sender == your own support address. Drop silently.

**Attachment storage.** Putting attachments in the ticket DB blows up storage and slows backups. Stream to S3 at intake, store a reference in `ticket_messages.attachments`. Generate signed URLs for download.

**PII redaction.** Customers paste credit card numbers and passwords into ticket bodies. Run a redaction pass at intake (regex for known formats). Store encrypted original, serve redacted by default, "show original" requires a permission grant and is audited.

**Status transition shortcuts.** Agents want a "resolved" button that skips `in_progress`. Allow shortcuts at the API level but enforce the state machine in the engine; record the transition with `via: shortcut` so audit can see what happened.

**Bulk operations.** "Close all tickets from this customer for the holiday week." Bulk endpoints exist but serialize the operations to keep audit ordering correct. Show per-ticket results so the user knows which failed.

### 14. Follow-up answers

**1. Email threading when subject is stripped.** The reliable path is `In-Reply-To` and `References` (RFC 5322). Most clients populate them; the index on `ticket_messages.inbound_message_id` makes lookup a single index hit. Subject prefix `[TKT-1234]` is the fallback when headers are stripped. As a last resort (off by default), match same-sender plus similar-subject plus within-N-days. If all three fail, a new ticket opens and an agent merges via a "merge into" action.

**2. Intake adapter crashes mid-batch.** The adapter never deletes messages from IMAP until they're safely staged. Track `last_processed_uid` per mailbox, updated only after successful staging to Kafka or a staging table with at-least-once semantics. On restart, resume from `last_processed_uid + 1`. Reprocessed messages get deduped by the unique index on `inbound_message_id`. Worst case is a few seconds of duplicate work, no message loss, no duplicate tickets.

**3. Two agents claim the same queued ticket.** The pull query is `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` inside a transaction. Agent A's transaction locks TKT-100 and updates `assignee_id`. Agent B's identical query at the same instant skips TKT-100 (already locked) and gets TKT-101. Both commit. No conflict. This is the canonical use case for `SKIP LOCKED`.

**4. SLA business hours across timezones.** Default is the team's business hours, stored as `business_hours_id` on the team config ("US-West 9-6 weekdays + US holidays"). Override per contract: enterprise customers with a 24/7 contract get `business_hours_id = always`. The contract policy lives on the customer record; at ticket creation, SLA computation picks the customer's policy if set, otherwise the team's. For "follow the sun", set the team's effective schedule to "always" and let the assignment service route to the on-shift region.

**5. Reopen after long silence.** Within the 7-day reopen window from `closed_at`, a customer reply reactivates the original: status goes to `reopened` then immediately to `in_progress`, `reopen_count++`. SLA clock resets, optionally to a tighter "reopen SLA" (4 hours instead of 24). Outside the window, a new ticket opens with `parent_ticket_id` pointing to the closed one. The agent UI surfaces the parent context: "This customer had a similar issue last year; view prior ticket."

**6. New issue from a customer with an open ticket.** Default is open a new ticket. The unrelated-issue case is more common than the related-follow-up case. Detection uses the same threading logic: `In-Reply-To` pointing to an agent message in the existing ticket → append. Subject mentions the ticket ref → append. Otherwise → new ticket. If the agent realizes two tickets are about the same thing, they merge.

**7. Agent vacation handover.** Agent sets `out_of_office: {start, end, delegate_id}` in their profile. On the start date, a job runs `SELECT ticket_id FROM tickets WHERE assignee_id = ? AND status NOT IN ('closed', 'resolved')`, reassigns each to the delegate (or to the team queue if no delegate), records `reason: vacation_handover` in `assignments`, notifies the delegate via Slack and email. On return, tickets stay where they are; the returning agent gets a digest of what happened. New incoming tickets for which the OOO agent would have been the round-robin pick: skip them in the rotation while OOO.

**8. Spam.** Layered defense at the email channel. SPF/DKIM/DMARC at SMTP rejects obviously spoofed mail. SpamAssassin or AWS SES spam check on accepted mail sends high-score mail to a quarantine mailbox, not the ticket DB. Rate limit per sender domain/IP drops to quarantine above N messages per minute. Allowlist of known customer domains bypasses aggressive filtering for paying customers. Periodic human review of quarantine catches false positives. A 99% filter at the front door drops most volume; the remaining ~100 spammy tickets per day get closed as `spam` by agents with a one-click action.

**9. KB suggestions at intake.** Pre-submit: as the customer types in the web form, debounce 500ms then call `POST /api/v1/kb/suggest` with the current subject+body. The endpoint hits Elasticsearch (top-3 articles by relevance score) and returns titles plus snippets within 50ms. Display inline: "Maybe one of these solves your issue?" Track if the customer clicks (`kb.suggestion.click`), abandons the form (`kb.suggestion.deflection`), or submits anyway. Deflection is the win metric; mature setups deflect 20-40% of would-be tickets.

**10. "Average time to resolve" is wrong.** Three common bugs. First, it includes `waiting_on_customer` time, but the agent isn't working then. Subtract the time spent in that status. Second, it's wall-clock and ignores business hours; a ticket created Friday 5 PM and resolved Monday 10 AM isn't 65 hours, it's ~3 business hours. Third, it counts spam and duplicate tickets, which close instantly and skew the average down. Exclude `final_state IN ('spam', 'duplicate')`. The correct computation is `sum of business-hours-active time per ticket / count of resolved tickets, excluding spam and duplicate`. Show p50 and p95 alongside the average; the average alone hides long-tail outliers.

### 15. Trade-offs worth saying out loud

**Push assignment vs pull from queue.** Push (round robin / skill / load) gives every ticket an owner instantly and is better for SLA accountability and "you have been assigned to Bob" emails. Pull is better for agent autonomy and avoids "assigned to an agent who just logged off". Most production setups push by default with pull as fallback when the assigned agent is unavailable.

**Canned responses vs ML-suggested replies.** Canned responses are agent-curated templates with placeholders. Predictable, controllable, agents stay in the driver's seat. ML-suggested replies save typing but risk wrong suggestions that the agent rubber-stamps. Best practice: keep canned responses for the top 50 known scenarios, layer suggestions on top for the long tail, never auto-send.

**Embedded chat session vs ticket per session.** Per-session is cleaner for SLA tracking; rolling gives agents context. Pick one and stick to it; mixing causes reporting confusion.

**Postgres FTS vs Elasticsearch for search.** FTS gets you to ~1M tickets cheaply. ES is a separate cluster to operate but gives faceted search, fuzzy matching, and relevance tuning. Switch when FTS search latency exceeds 500ms or when product asks for facets.

**Custom build vs Zendesk/Freshdesk.** If your support volume is small, buy. Build-vs-buy tips toward build when you have unusual integrations, a workflow that doesn't fit the SaaS model (regulated industries, embedded support inside another product, deep cross-system automation), or volume large enough that the SaaS bill exceeds engineering cost. Below that line, don't build this.

**Per-ticket scheduled timers vs sweep worker.** Per-ticket timers (one cron entry per ticket, or one delayed message per ticket in a queue) sound clean but cancellation on resolve/pause becomes brittle. Sweep worker is boring but reliable. Sweep wins at scale.

### 16. Common interview mistakes

Most weak answers fall into one of these.

**Tickets as CRUD with a `status` column and hardcoded transitions.** If `status` is a free-form string and any agent can flip to any value, you have a Trello clone, not a help desk. The state machine *is* the system.

**No intake adapter layer.** Skipping straight to "the API takes JSON" misses the entire email-parsing-and-threading problem, which is the single hardest part.

**Round-robin assignment as the default.** Works for the smallest team. Breaks at 20+ agents where skill matters and at 100+ where load matters. Acknowledge the trade-off and offer skill-based as the next step up.

**Wall-clock SLA, no business hours.** Most candidates skip this. Real customers won't tolerate a 24-hour SLA that expires Sunday at 3 AM.

**No SLA pause for `waiting_on_customer`.** Without pause, every ticket that needs customer info eventually breaches through no fault of the agent. The SLA system loses credibility and agents game it ("close and reopen later").

**Per-ticket scheduled timer.** Tempting because it's "real-time". A nightmare at scale: 150k pending callbacks, cancellation races, fragile failure modes. Sweep is the right answer.

**No reopen handling.** Customer replies to a closed ticket, a new ticket opens, conversation history is lost. Build the 7-day reopen window from day one.

**Email threading via subject only.** Subjects get rewritten, prefixed, translated. Use `In-Reply-To` headers as the primary signal, subject as fallback.

**No assignment race control.** Two agents claim the same ticket. Without `SKIP LOCKED` (pull) or row locks (push), you get split-brain assignments.

**Notifications glued to the ticket service.** Tickets emit events; the notification service consumes. Coupling them means you can't swap Slack for Teams without touching the core.

If you hit 7 of these 10, you're interviewing at senior or staff level. The most common shallow answer is "tickets table with a status column and a queue"; the state-machine plus intake-adapter plus SLA-business-hours triad is what separates a thoughtful design from a generic CRUD answer.
