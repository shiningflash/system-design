## Solution: Help Desk Ticketing System

### TL;DR

A help desk system is a state machine with a messy front door. The front door is the hard part: emails arrive over IMAP with broken subject lines, chats arrive over websockets and end when the user closes the tab, web forms post JSON, Slack messages come through a bot token, SMS comes through Twilio. Each channel needs an adapter that parses, threads (matches replies to existing tickets), and emits a unified intake event.

Once a ticket exists, it is a small state machine: `new → assigned → in_progress → (waiting_on_customer ↔ in_progress)* → resolved → closed`, with reopen and reassignment as common edges. The Ticket Service enforces transitions and persists everything to Postgres. An Assignment Service decides who owns each ticket (skill-based plus load-balanced, with queue-pull fallback). An SLA timer worker sweeps the table every 30 seconds, pauses outside business hours, and fires escalation events on breach.

Scale is not the headline. Even at 50k tickets per day with 2000 agents you only push ~15 writes per second. The interesting engineering is correctness: email threading, SLA pause/resume across business hours and timezones, agent assignment race conditions, and reopen after silence.

### 1. Clarifying questions and why each matters

Covered in `question.md`. The two questions that change the design most:

- **Channels.** Email is the messiest because of reply threading. Adding chat or Slack adds intake adapters but does not change the rest of the system.
- **SLA semantics.** Wall-clock SLA is trivial. Business-hours SLA across customer timezones is a real engineering project and is the source of most reporting bugs.

### 2. Capacity estimates (full working)

**Small startup (50/day, 3 agents):**

- ~50 writes/day across tickets + messages = 0.003 writes/sec. Single Postgres on a t3.small. No cache, no search index needed.
- ~100 active tickets at any moment. Indexes are essentially decorative at this scale.

**Enterprise (50k/day, 2000 agents):**

- ~0.6 ticket creates/sec sustained, ~3/sec peak.
- ~5 messages/sec sustained, ~15/sec peak (each ticket averages 8 messages over its life).
- ~150k active tickets at any moment.
- ~3M agent-dashboard reads/day plus ~500k customer-portal reads/day = ~40/sec sustained, ~120/sec peak.
- ~5 TB ticket data over 5 years (text), ~50 TB attachments in S3.

The system is small in raw QPS terms. The architecture exists for **state correctness, fair assignment under concurrency, SLA accuracy, and operational visibility**, not for throughput.

### 3. API design

**Customer-facing intake:**

```
POST /api/v1/tickets
Content-Type: application/json
Idempotency-Key: <uuid>             # for retried submissions

{
  "subject": "Cannot log in to mobile app",
  "body": "I've been trying since this morning...",
  "channel": "web_form",
  "customer_id": "cust_abc",        # or null for anonymous email-based
  "customer_email": "alice@example.com",
  "metadata": {
    "browser": "Safari 17",
    "app_version": "4.2.1"
  },
  "attachments": [
    {"filename": "screenshot.png", "upload_id": "att_xyz"}
  ]
}
```

| Status | Meaning | Body |
|--------|---------|------|
| 201 Created | New ticket | `{"ticket_id": "tkt_1234", "status": "new", "public_url": "..."}` |
| 200 OK | Idempotency key matched a recent create; return existing | same shape |
| 400 Bad Request | Validation failure (no body, invalid channel) | error detail |
| 429 Too Many Requests | Rate limited (per IP, per customer) | with `retry_after` |

**Agent reply / message append:**

```
POST /api/v1/tickets/{ticket_id}/messages
{
  "body": "Can you share the error you see?",
  "visibility": "public" | "internal",   # internal = not visible to customer
  "attachments": [...]
}
```

A `public` message also pushes outbound (email reply, chat push, Slack DM) via the channel the ticket lives on.

**Assign / reassign:**

```
POST /api/v1/tickets/{ticket_id}/assign
{
  "team_id": "billing",
  "agent_id": "agt_bob"        # optional; null lets Assignment Service pick
}
```

**Status change:**

```
POST /api/v1/tickets/{ticket_id}/status
{
  "status": "in_progress" | "waiting_on_customer" | "resolved" | "closed" | "spam" | "duplicate",
  "reason": "Customer confirmed fix"   # required for some transitions
}
```

The Ticket Service validates the transition against the state machine. Invalid transition returns `422 Unprocessable Entity`.

**Search:**

```
GET /api/v1/tickets?status=open&assignee=me&priority=high
GET /api/v1/tickets/search?q=login+mobile&customer_id=cust_abc
```

Search hits Elasticsearch. Filtered lookup by indexed columns hits the read replica.

**Notes on the API:**

- **Idempotency-Key on create.** Email retries from SES, mobile clients that retry on timeout, web form double-clicks. Without an idempotency key you create duplicate tickets. Hash the key, store for 24 hours.
- **Messages are the unit of conversation, not tickets.** Tickets group messages; messages carry content.
- **`visibility: internal` is critical.** Agent-to-agent comments live on the same ticket but are never sent to the customer. The outbound channel adapter checks `visibility` before sending.

### 4. Data model

```sql
-- Tickets: one row per logical ticket.
CREATE TABLE tickets (
    ticket_id           UUID PRIMARY KEY,
    public_ref          TEXT NOT NULL UNIQUE,      -- "TKT-1234" shown to customer
    subject             TEXT NOT NULL,
    customer_id         TEXT,                       -- null for anonymous email-only
    customer_email      TEXT,
    channel             TEXT NOT NULL,              -- 'email' | 'web_form' | 'chat' | 'slack' | 'sms'
    channel_thread_id   TEXT,                       -- channel-native ID for threading
    status              TEXT NOT NULL,              -- state machine state
    priority            TEXT NOT NULL DEFAULT 'medium',
    team_id             TEXT,                       -- owning team
    assignee_id         TEXT,                       -- current agent
    tags                TEXT[] DEFAULT '{}',        -- ["billing", "mobile"]
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    first_response_at   TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    reopen_count        INT NOT NULL DEFAULT 0,
    parent_ticket_id    UUID,                       -- if linked to a prior closed ticket
    metadata            JSONB                       -- channel-specific (browser, app version, etc.)
);
CREATE INDEX idx_tk_assignee_status ON tickets (assignee_id, status) WHERE status NOT IN ('closed','spam','duplicate');
CREATE INDEX idx_tk_team_status ON tickets (team_id, status, priority);
CREATE INDEX idx_tk_customer ON tickets (customer_id, created_at DESC);
CREATE INDEX idx_tk_channel_thread ON tickets (channel, channel_thread_id);   -- for reply threading

-- Messages: every customer or agent message on a ticket.
CREATE TABLE ticket_messages (
    message_id          UUID PRIMARY KEY,
    ticket_id           UUID NOT NULL REFERENCES tickets(ticket_id),
    author_type         TEXT NOT NULL,              -- 'customer' | 'agent' | 'system'
    author_id           TEXT,
    visibility          TEXT NOT NULL DEFAULT 'public',  -- 'public' | 'internal'
    body                TEXT NOT NULL,
    body_format         TEXT NOT NULL DEFAULT 'plain',   -- 'plain' | 'html' | 'markdown'
    inbound_channel     TEXT,                       -- which channel delivered this
    inbound_message_id  TEXT,                       -- e.g., email Message-ID header
    in_reply_to         TEXT,                       -- email In-Reply-To header
    attachments         JSONB DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_msg_ticket ON ticket_messages (ticket_id, created_at);
CREATE INDEX idx_msg_inbound ON ticket_messages (inbound_message_id) WHERE inbound_message_id IS NOT NULL;

-- Assignments: history of who owned the ticket and when.
CREATE TABLE assignments (
    assignment_id       UUID PRIMARY KEY,
    ticket_id           UUID NOT NULL REFERENCES tickets(ticket_id),
    team_id             TEXT NOT NULL,
    agent_id            TEXT,                       -- null = team queue, not yet picked
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    unassigned_at       TIMESTAMPTZ,                -- non-null when superseded
    assigned_by         TEXT,                       -- agent_id or 'system'
    reason              TEXT                        -- 'initial' | 'reassigned' | 'escalation' | 'vacation_handover'
);
CREATE INDEX idx_assignments_ticket ON assignments (ticket_id, assigned_at DESC);
CREATE INDEX idx_assignments_agent_active ON assignments (agent_id) WHERE unassigned_at IS NULL;

-- SLA targets: see Step 6 in question.md for the full schema.
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

-- Audit: append-only log of every state change.
CREATE TABLE ticket_audit (
    audit_id            UUID PRIMARY KEY,
    ticket_id           UUID NOT NULL,
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    event_type          TEXT NOT NULL,              -- 'created' | 'status_changed' | 'assigned' | 'replied' | 'escalated' | etc.
    actor_type          TEXT NOT NULL,              -- 'customer' | 'agent' | 'system'
    actor_id            TEXT,
    payload             JSONB NOT NULL
);
CREATE INDEX idx_audit_ticket ON ticket_audit (ticket_id, occurred_at);
CREATE INDEX idx_audit_event ON ticket_audit (event_type, occurred_at);
```

**Why each table:**

- **`tickets`** is the source of truth for current state. The composite indexes serve the two dominant queries: "show me my pending tickets" (`assignee_id, status`) and "show me my team's queue" (`team_id, status, priority`).
- **`ticket_messages`** is conversation. Indexed by `inbound_message_id` for fast threading lookups from intake.
- **`assignments`** is history, not current. The current assignee is on the ticket row for fast reads; the history lives here for audit and analytics (mean time to first assignment, handoff counts).
- **`sla_targets`** is one row per ticket. Separating SLA from the ticket row lets the SLA worker query a narrow table without scanning ticket bodies.
- **`ticket_audit`** is append-only. Captures every state transition, every reply, every assignment change. Retention 5 to 7 years.

**Choice: Postgres, not Mongo or DynamoDB.** The state-machine transitions need transactional guarantees (status change + audit insert in one transaction). Postgres also handles the JSON metadata fine via JSONB. You only outgrow Postgres at this workload if you sell to thousands of multi-tenant customers; even then, sharding by `team_id` keeps you on Postgres per shard.

### 5. Core algorithms

#### 5a. Intake adapter and email threading

Email is the messy channel. Walk through what the email adapter does on a new message:

```python
def handle_inbound_email(raw_email):
    msg = parse_email(raw_email)

    # 1. Spam / abuse filter (SpamAssassin, custom rules).
    if is_spam(msg):
        record_dropped(msg, reason='spam')
        return

    # 2. Look for a thread reference.
    #    Strategy 1: In-Reply-To and References headers (the RFC-compliant way).
    parent_msg_id = msg.headers.get('In-Reply-To')
    refs = msg.headers.get('References', '').split()
    candidate_msg_ids = [parent_msg_id] + refs if parent_msg_id else refs

    existing_ticket = None
    for mid in candidate_msg_ids:
        m = db.query("SELECT ticket_id FROM ticket_messages WHERE inbound_message_id = ?", mid)
        if m:
            existing_ticket = m.ticket_id
            break

    # 3. Strategy 2 fallback: ticket reference in subject (e.g., "[TKT-1234]").
    if not existing_ticket:
        ref = extract_ticket_ref(msg.subject)   # regex on "[TKT-1234]"
        if ref:
            existing_ticket = db.query("SELECT ticket_id FROM tickets WHERE public_ref = ?", ref)

    # 4. Strategy 3 fallback: same sender + similar subject + recent ticket.
    #    Last-resort; high false-positive risk. Off by default.

    if existing_ticket:
        # Append message; reopen if needed.
        emit_event("ticket.reply_received", existing_ticket, msg)
    else:
        # New ticket.
        emit_event("ticket.intake", channel='email', payload=normalize(msg))
```

**The threading layers, in priority order:**

1. **`In-Reply-To` / `References` headers.** RFC 5322 standard. Most email clients populate these correctly. This catches ~85% of replies.
2. **Subject-line tag.** Outbound agent replies include `[TKT-1234]` in the subject. Even when headers are stripped by aggressive mail proxies, the subject tag usually survives. Catches another ~10%.
3. **Heuristic (same sender + similar subject + recent window).** Catches the tail. False-positive-prone; off by default for most teams.

The remaining ~5% open new tickets. An agent merges duplicates manually.

#### 5b. Assignment

```python
def assign_ticket(ticket):
    team = pick_team(ticket)        # via triage rules; tag/keyword/channel match
    candidates = list_agents_in_team(team)

    # Filter by required skills (if the ticket is tagged with skills).
    if ticket.required_skills:
        candidates = [a for a in candidates if set(ticket.required_skills).issubset(a.skills)]

    if not candidates:
        # No skilled agent available; fall back to team queue (no specific assignee).
        return Assignment(team_id=team, agent_id=None)

    # Filter out unavailable (vacation, off-shift).
    candidates = [a for a in candidates if a.is_available(now())]

    # Load-balanced pick: least-busy agent, tie-break by least-recently-assigned.
    with db.transaction():
        # Lock team row to serialize assignment writes within this team.
        db.lock_for_update("teams", team)
        loads = db.query("""
          SELECT agent_id, COUNT(*) AS open_count, MAX(assigned_at) AS last_assigned
          FROM assignments
          WHERE agent_id = ANY(?) AND unassigned_at IS NULL
          GROUP BY agent_id
        """, [a.id for a in candidates])

        chosen = min(candidates, key=lambda a: (loads.get(a.id).open_count, loads.get(a.id).last_assigned))

        db.insert("assignments",
                  ticket_id=ticket.id,
                  team_id=team,
                  agent_id=chosen.id,
                  reason='initial')
        db.update("tickets", ticket.id, team_id=team, assignee_id=chosen.id)
        return Assignment(team_id=team, agent_id=chosen.id)
```

**The serial lock per team** is the critical detail. Without it, two parallel `assign_ticket` calls for the same team can pick the same agent for two tickets, both believing the agent has the lowest load. The team lock makes assignments within a team strictly sequential. Across teams they remain parallel.

For the queue-pull strategy (agents click "Next Ticket"), the logic inverts:

```python
def claim_next_ticket(agent):
    with db.transaction():
        ticket = db.query("""
          SELECT ticket_id FROM tickets
          WHERE team_id = ? AND assignee_id IS NULL AND status IN ('new', 'assigned')
          ORDER BY priority DESC, created_at ASC
          FOR UPDATE SKIP LOCKED
          LIMIT 1
        """, agent.team_id)
        if not ticket:
            return None
        db.update("tickets", ticket.id, assignee_id=agent.id, status='assigned')
        db.insert("assignments", ticket_id=ticket.id, agent_id=agent.id, reason='pulled')
        return ticket
```

**`FOR UPDATE SKIP LOCKED`** is the right primitive here. Two agents calling at the same instant: each gets a different ticket because the lock on the first ticket makes the second query skip it.

#### 5c. SLA monitoring

Covered in question.md Step 6. The implementation summary:

- One row per ticket in `sla_targets`. Deadline is stored as a wall-clock timestamp, precomputed to account for business hours.
- A sweep worker runs every ~30 seconds with two queries: tickets approaching `first_response_due` and tickets approaching `resolution_due`.
- On breach, emit `sla.first_response_breached` or `sla.resolution_breached`. An escalation consumer reads these and applies the policy (notify, reassign, page).
- Pause and resume adjust the deadline by the paused business-time duration.

### 6. Architecture

Already in question.md Step 4. Component responsibility recap:

| Component | Stateful? | Scaling story |
|-----------|-----------|---------------|
| Intake Adapters | Yes (channel offsets) | One process per channel; shard by mailbox / chat partition |
| Ticket Service | No (state in DB) | Horizontal pods; partition by `team_id` for lock locality |
| Assignment Service | Stateless | Horizontal; per-team lock serializes writes |
| SLA Timer Worker | No (state in DB) | Singleton or partition by ticket_id range; idempotent |
| Tickets DB (Postgres) | Yes | Primary + read replicas; shard by `team_id` at very large scale |
| Read Cache (Redis) | Yes | Cluster; populated from Kafka events |
| Search Index (Elasticsearch) | Yes | Standard ES cluster; CDC from Postgres |
| KB Search | Yes | Separate ES index over articles |
| Kafka | Yes | Standard cluster |
| Notifications Service | Stateless | Consumes Kafka; covered in problem 010 |

### 7. Read and write paths

**Write path (new ticket from email):**

1. IMAP poller (or SES inbound webhook) → Intake Adapter.
2. Adapter parses, runs spam filter, checks threading headers.
3. If a parent ticket is found, this is a `reply_received` event, not a `create`.
4. Adapter writes to a staging table (or publishes to a Kafka topic) with `at-least-once` semantics.
5. Ticket Service consumes the staging event in a transaction:
   - Insert `tickets` row.
   - Insert first `ticket_messages` row.
   - Compute SLA targets (call business-hours service); insert `sla_targets`.
   - Insert `ticket_audit` events.
6. Call Assignment Service to pick team and agent; update `tickets` and insert `assignments`.
7. Emit `ticket.created` and `ticket.assigned` events to Kafka.
8. Indexer consumer adds the ticket to Elasticsearch.
9. Notifications consumer sends the agent assignment email/Slack DM.

P99 target: 500ms intake to assigned. Email intake is async (customer does not wait); web form is sync and must hit this target.

**Write path (agent reply):**

1. Agent's UI → API Gateway → Ticket Service.
2. Validate: ticket exists, agent is assignee or has team permission.
3. In one DB transaction: insert `ticket_messages`, update `tickets.first_response_at` if null, update `sla_targets.first_response_at`, insert `ticket_audit`.
4. If public visibility, hand off to the outbound channel adapter (send email reply, post to chat, etc.).
5. Emit `ticket.replied` event.

P99: 200ms.

**Read path (agent dashboard):**

1. Agent UI → API Gateway → Ticket Service read endpoint.
2. Check Redis cache `agent:{id}:open_tickets`. Hit: return immediately.
3. Cache miss: query read replica `SELECT ... FROM tickets WHERE assignee_id = ? AND status IN ('new','assigned','in_progress','waiting_on_customer')` with covering index. Populate cache, 60-second TTL.
4. Cache invalidation via Kafka events: when any of those tickets transitions, the invalidator deletes the affected agent's cache keys.

P99: 50ms.

**Read path (ticket search):**

1. Agent UI → Search Service → Elasticsearch.
2. ES returns ticket IDs in relevance order.
3. Hydrate from Postgres or read cache for display.

P99: 200ms.

### 8. Scaling journey: 10 to 1M users

The system is small in raw QPS even at the top. Each stage adds something driven by a real operational pain, not anticipated throughput.

#### Stage 1: 10 to 100 users, 3 agents, 50 tickets/day

**What you build:**

- Single Postgres (t3.medium, 4GB).
- Single Python/Go/Node Ticket Service process.
- One intake adapter: email, polling Gmail via IMAP.
- Assignment is round-robin within the single team. Stored as `next_agent_index` in Postgres, incremented on each assignment.
- No cache. No search. No queue. No replicas.
- SLA timer is a cron job hitting `SELECT ...` every minute.
- Notifications are inline: send email via SendGrid HTTP API at the moment of state change.

**Why this is enough:**

- 50 writes/day. Postgres is bored.
- 3 agents × 100 active tickets total. Even a `SELECT *` returns in 5ms.
- One person reads reports. They run `psql` queries directly.

**What you do not build:**

- No multi-channel. Email only.
- No business hours. Wall-clock SLA. "First response in 4 hours" is the only policy.
- No customer portal. Customers email; agents reply by email; the customer sees the conversation in their inbox.
- No knowledge base lookup.
- No Elasticsearch.

**Cost:** ~$50/month for DB, ~$30/month for the app, ~$20/month for SendGrid. Ships in two weeks.

#### Stage 2: 500 tickets/day, 20 agents

**What just broke:**

- "We added a web form on the marketing site. How do tickets from there get in?" One adapter is not enough.
- "Customer Success agents and Technical agents are different teams now. Round robin sends billing questions to technical agents." Need skill-based routing.
- "We promised paying customers a 1-hour response SLA, but we have no way to know when we are about to breach." Need an SLA worker, not just a cron.
- "Marketing wants reports on which channel generates the most tickets." Need analytics-friendly storage.

**What you add:**

- **Multi-channel intake.** Three adapters: email (existing), web form (POST endpoint), chat (websocket terminator). Each adapter normalizes to a common `TicketIntakeEvent` schema.
- **Skill-based assignment.** Tag tickets at intake (use regex/keyword rules; e.g., "invoice", "refund" → billing skill). Tag agents with skill arrays. Assignment Service filters candidates by skill, then load-balances within the matched set.
- **SLA worker.** Dedicated background process that sweeps the `sla_targets` table every 30 seconds. Emits warning at 75% of deadline, breach at 100%. Escalation policy is a simple YAML config; for now, "notify the team channel in Slack on breach."
- **Audit log.** Append-only `ticket_audit` table. Every state change recorded. Replaces ad-hoc `application.log` scraping.
- **Internal vs public messages.** Agents can leave internal notes that customers do not see. Add `visibility` column.

**What you do not yet add:**

- No read replicas. One Postgres handles all reads.
- No Kafka. The `ticket_audit` table doubles as the event stream; consumers poll it (every 5 seconds is fine at this volume).
- No Elasticsearch. Postgres full-text search on `ticket_messages.body` (with a `tsvector` index) is enough for ~50k tickets total.

**Why this is enough:**

- 500 tickets/day = ~0.006/sec. Still trivial.
- 20 agents × 100 dashboard views/day = 2000 reads/day. Postgres handles this.
- 50k tickets in the DB. Postgres FTS returns search results in <500ms.

**Cost:** ~$300/month (slightly bigger DB, two channel-adapter processes, SLA worker pod).

#### Stage 3: 5000 tickets/day, 200 agents, multi-team

**What just broke:**

- Agent dashboard load takes 3 seconds. The combined query "my tickets + my team's queue + count of urgent tickets" is now slow.
- Search returns in 4 seconds for queries that match thousands of tickets. Postgres FTS is at its limit.
- The SLA worker takes 45 seconds to do one sweep when there are 30k open tickets. Some breaches are detected late.
- Each team wants its own queue, its own routing rules, its own SLA policy. Hardcoded YAML is unmanageable.
- Real-time updates: when a teammate takes a ticket from the queue, your dashboard should reflect it within a second. Polling every 30 seconds is laggy.

**What you add:**

- **Per-team queues with configurable routing.** Move team and SLA policies into a `team_configs` table editable through an admin UI. Each team has: assignment strategy (round robin / skill / pull), business hours schedule, default priority, escalation policy.
- **Search index (Elasticsearch).** CDC from Postgres (Debezium → Kafka → ES indexer). All tickets and messages indexed. Search latency drops from 4s to 100ms.
- **Read replicas (2).** Dashboard reads go to replicas; writes go to primary. ~1-second replication lag is acceptable for dashboards.
- **Redis cache for hot reads.** `agent:{id}:open_tickets`, `team:{id}:queue_summary`. Invalidated by Kafka events.
- **Kafka.** Move from polling `ticket_audit` to publishing events to topics: `ticket.created`, `ticket.replied`, `ticket.assigned`, `ticket.status_changed`, `sla.breached`. Consumers: indexer, notifier, cache invalidator, analytics.
- **WebSocket push to dashboards.** Agents connect to a WebSocket service. Cache invalidator pushes updates ("ticket TKT-456 was claimed by Bob") to subscribed clients. No more polling.
- **SLA worker partitioning.** Shard the sweep by `ticket_id` hash across N worker pods. Each pod sweeps its slice. Total sweep latency stays bounded.
- **Business hours and timezone-aware SLA.** Each team has a `business_hours_id`; each customer can have a `timezone`. SLA computation uses `add_business_time` against the right schedule.

**Why this is enough:**

- 5000 tickets/day = ~0.06/sec. Writes are nowhere near the limit.
- ~30k open tickets. Indexes still serve dashboard queries in <50ms.
- Search latency under 100ms via ES. Agents are happy.

**Cost:** ~$3-5k/month (Postgres + replicas, Redis cluster, Kafka cluster, ES cluster, WebSocket pods).

#### Stage 4: 50k tickets/day, 2000 agents across 4 regions

**What just broke:**

- One Postgres holds all tickets across all teams. A long-running analytical query on the primary stalls the Ticket Service.
- Intake from the EU mailbox lives in US-East; ~150ms latency on each intake event. Acceptable but customers expect snappier acknowledgement.
- Email parsing for 50k inbound messages per day is starving the single email adapter's CPU. Bursty volume during outages overwhelms it.
- The skill-based router uses hand-maintained keyword rules. Adding new keywords requires a deploy, and ambiguous tickets get misrouted ~10% of the time.
- A few enterprise customers have started complaining about EU data residency. Their tickets cannot live in US-East.

**What you add:**

- **Shard the Tickets DB by `team_id`.** Each shard is a Postgres cluster. Cross-team queries (rare) go through a query coordinator that scatter-gathers.
- **Regional deployment.** Intake adapters per region (US, EU, APAC, AU). Each region has its own Ticket Service, DB shards, Redis, Kafka. Customers route to their nearest region. Enterprise customers with residency requirements get pinned to a specific region.
- **Async intake pipeline.** Each intake adapter is now multi-process: a "receiver" tier that just buffers raw messages to Kafka, and a "parser" tier that consumes from Kafka and emits structured `TicketIntakeEvent`s. Receivers stay light (CPU-bound parsers can scale independently).
- **ML-based skill suggestion.** Replace keyword rules with a classifier (a fine-tuned BERT or even logistic regression on TF-IDF). Trained on historical "ticket text → team assigned" pairs. Suggestions feed into the Assignment Service; humans can override; corrections become new training data.
- **Cross-region agent dashboards.** A US agent sometimes needs to see an EU ticket. A read-only API replica in each region serves cross-region lookups with explicit data residency checks (does this agent have permission to see EU customer data?).
- **KB suggestions both at intake and at agent compose.** Same ES index serves both. At intake, surface top-3 articles to the customer ("Maybe one of these solves your issue?"). At compose, suggest to the agent ("Insert this article into the reply.").
- **Tiered audit storage.** Hot 90 days in Postgres. Older in S3 (Parquet) queried via Athena. Audit volume at this scale is ~5M events/day; cheap to keep but expensive to query without tiering.

**Why this is enough:**

- 50k tickets/day = ~0.6/sec. Sharding by team gives plenty of headroom per shard.
- Regional intake brings inbound latency to <50ms.
- The system is now organizationally complex (4 regions, 200 teams, ML-based routing) but each region's data plane is the same shape as stage 3.

**Cost:** ~$30-80k/month depending on region count, agent count, attachment storage.

#### What you would do beyond 1M users

At this point you are selling this as a SaaS product (Zendesk, Freshdesk, Intercom scale). The architectural concerns shift from per-customer correctness to multi-tenant isolation, per-tenant rate limits, billing, and a marketplace of integrations. The base architecture above is unchanged; you wrap it in tenant isolation and a customer-facing admin product.

### 9. Reliability

**Intake adapter dies mid-batch.** Each adapter uses at-least-once semantics. For IMAP: track the last-processed UID per mailbox in a `mailbox_state` row, committed only after the message is successfully handed off to the Ticket Service (or staged in Kafka). On restart, resume from the last committed UID. Messages reprocessed will be deduplicated by `inbound_message_id` (unique index on `ticket_messages.inbound_message_id`). For SES inbound webhooks: SES retries on 5xx; the receiver returns 200 only after the message is staged.

**SLA timer worker dies.** The worker is stateless and idempotent. State lives in `sla_targets`. On restart, the next sweep picks up where the previous left off. Worst case: a 30-second delay in detecting a breach. Run two workers in active-active behind a per-shard advisory lock (each worker owns one shard at a time; if it dies, the other takes over within the lock TTL).

**Assignment race.** Two `assign_ticket` calls for the same team simultaneously: serialized by the team-level row lock. Two queue-pull calls for the same ticket: serialized by `SELECT ... FOR UPDATE SKIP LOCKED`. Two agents trying to take an action on the same ticket: serialized by `SELECT ... FOR UPDATE` on the ticket row inside the transition transaction.

**Notification fails.** Notifications consume Kafka events with at-least-once delivery. On failure, the consumer retries with exponential backoff. After max retries, the event lands in a dead-letter topic monitored by an on-call dashboard.

**Outbound email bounces.** Customer's email is dead or full. SES delivers a bounce notification to a Kafka topic. The bounce consumer marks the ticket with `outbound_failed`, switches the ticket to "needs alternative contact" status, alerts the assigned agent.

**Search index drift.** Elasticsearch is downstream from Postgres via CDC. If indexing lags or fails, search returns stale results. A nightly reconciliation job compares row counts and checksums per index; mismatches trigger a re-indexing run for the affected ticket IDs.

**Postgres failover.** Primary fails → replica promoted. In-flight transactions roll back; clients retry with idempotency keys. Engine and worker pods reconnect within seconds. SLA worker's next sweep handles any tickets that breached during the failover gap.

### 10. Observability

| Metric | Why it matters |
|--------|----------------|
| `tickets.created.rate` by channel | Sudden drop = an intake adapter is broken. Sudden spike = outage or marketing campaign. |
| `tickets.in_flight.count` by team and status | Slow accumulation = team is underwater. |
| `time_to_first_response` p50/p95 | The headline customer-facing SLO. |
| `time_to_resolution` p50/p95 by priority | The other headline SLO. |
| `sla.breach.rate` by priority and team | Per-team operational health. |
| `agent.utilization` (active tickets / max concurrent) | Identifies overloaded and underused agents. |
| `assignment.latency` p99 | If high, the Assignment Service is contended; check team lock wait times. |
| `intake.parse.latency` per channel | If email parsing latency rises, the adapter or its dependencies (spam filter, threading lookup) are slow. |
| `intake.threading.success_rate` | What fraction of incoming emails thread into an existing ticket vs open a new one. A drop means email headers are being stripped, or the matcher logic is wrong. |
| `kb.suggestion.deflection.rate` | Of customers who saw KB suggestions, how many closed the form without submitting. This is your self-service ROI. |
| `reopen.rate` | High reopen rate = agents are closing too aggressively. |
| `audit.write.lag` | Audit must not lag; alert at >5 seconds. |

Alerts:

- Page: any channel's intake adapter offline for >2 minutes; SLA worker not running; Postgres primary unreachable.
- Ticket: `time_to_first_response` p95 regression >30%; `sla.breach.rate` doubles; reopen rate above 10%.

### 11. Common gotchas

1. **Timezone for business hours.** A US team supporting an EU customer: which clock does the SLA run on? Industry convention: the team's clock by default, the customer's clock if the contract specifies. Mixing them silently is the source of half the SLA disputes in production. Always show both clocks in the agent UI: "Customer time: 14:00 CET. Your time: 8:00 EST. SLA deadline: 17:00 CET (your 11:00 EST)."
2. **Ticket re-opens.** A customer replies to a "resolved" ticket. The naive design opens a new ticket. The right design: within 7 days, the reply reopens the original; after 7 days, opens a new ticket linked via `parent_ticket_id` so the agent has context. Reopening also resets the SLA; some teams use a separate "reopen SLA" with a tighter target.
3. **Conversation threading across email replies.** `In-Reply-To` and `References` headers are the only reliable way to thread. Subject-line tags (`[TKT-1234]`) are the fallback when headers are stripped. Heuristic threading (same sender, similar subject) is dangerous; it cross-contaminates tickets. Default: off.
4. **Agent vacation handover.** Bob has 47 open tickets and starts a 2-week vacation. Manual reassignment is painful. Build a "vacation mode" feature: Bob sets `out_of_office` start and end dates, picks a delegate (Carol). On the start date, a job reassigns all of Bob's open tickets to Carol (or to the team queue if no delegate). On Bob's return, tickets stay where they are; Bob does not get them back automatically.
5. **Spam at the front door.** A public support email address receives spam. Stack: SPF/DKIM check, a spam-scoring filter (SpamAssassin or commercial), a rate limiter per sender IP, an allowlist for known customers (their email gets through even if scored high). Drop spam to a dead-letter mailbox; review periodically.
6. **Self-replies and auto-responders.** Your own outbound "Ticket received" autoresponse can loop with the customer's autoresponder, generating dozens of messages on the same ticket per minute. Detect: `Auto-Submitted: auto-replied` header, or `X-Auto-Response-Suppress`, or sender == your support address. Drop these silently.
7. **Attachment storage.** Putting attachments in the ticket DB blows up storage and slows backups. Stream to S3 at intake, store a reference in `ticket_messages.attachments`. Generate signed URLs for download.
8. **PII redaction.** Customers paste credit card numbers and passwords into ticket bodies. Run a redaction pass at intake (regex for known formats). Store original encrypted; serve redacted by default; "show original" requires a permission grant and is audited.
9. **Status transition shortcuts.** Agents want a "resolved" button that skips `in_progress`. Allow shortcuts at the API level but enforce the state machine in the engine (transitions from `new` straight to `resolved` are valid but recorded with `via: shortcut`).
10. **Bulk operations.** "Close all tickets from this customer for the holiday week." Bulk endpoints exist but serialize the operations to keep audit ordering correct. Show per-ticket results so the user knows which ones failed.

### 12. Follow-up answers

**1. Email threading when subject is stripped.**

The reliable path is `In-Reply-To` and `References` headers (RFC 5322). Most clients populate them. We index `ticket_messages.inbound_message_id` so a header lookup is a single index hit. Subject prefix `[TKT-1234]` is the fallback when headers are stripped. As a last resort (off by default), match same-sender + similar-subject + within-N-days. If all three fail, a new ticket opens and an agent can merge later via a "merge into" action.

**2. Intake adapter crashes mid-batch.**

The adapter never deletes messages from IMAP until they are safely staged. Track the last-processed UID per mailbox in a `mailbox_state` row, updated only after successful staging (to a Kafka topic or staging table) with at-least-once semantics. On restart, resume from `last_processed_uid + 1`. Reprocessed messages are deduplicated by `inbound_message_id` (unique index on `ticket_messages`). Worst case: a few seconds of duplicate work; no message loss, no duplicate tickets.

**3. Two agents claim the same queued ticket.**

The pull query is `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` inside a transaction. Agent A's transaction locks ticket TKT-100 and updates `assignee_id`. Agent B's identical query at the same instant skips TKT-100 (already locked) and gets TKT-101. Both transactions commit. No conflict. This is the canonical use case for `SKIP LOCKED`.

**4. SLA business hours across timezones.**

Default: the team's business hours, stored as `business_hours_id` on the team config (e.g., "US-West 9-6 weekdays + US holidays"). Override per contract: enterprise customers with a 24/7 contract get a `business_hours_id` of "always". The contract policy is on the customer record; at ticket creation, the SLA computation picks the customer's policy if set, otherwise the team's. For "follow the sun", set the team's effective schedule to "always" and let the assignment service route to the on-shift region.

**5. Ticket reopen after long silence.**

Within the reopen window (default 7 days from `closed_at`), a customer reply reactivates the original ticket: status → `reopened`, then immediately → `in_progress`, `reopen_count++`. SLA clock resets with a tighter "reopen SLA" if configured (e.g., 4 hours instead of 24). Outside the window, a new ticket opens with `parent_ticket_id` pointing to the closed one. The agent UI surfaces the parent context: "This customer had a similar issue last year; view prior ticket."

**6. New issue from a customer with an open ticket.**

Default: open a new ticket. The unrelated-issue case is more common than the related-follow-up case. The way you detect "is this a follow-up to the existing ticket" is the same threading logic: `In-Reply-To` pointing to an agent message in the existing ticket → append. Subject mentions the ticket ref → append. Otherwise → new ticket. If the agent realizes after the fact that two tickets are about the same thing, they merge.

**7. Agent vacation handover.**

Agent sets `out_of_office: {start, end, delegate_id}` in their profile. On the start date, a job runs: `SELECT ticket_id FROM tickets WHERE assignee_id = ? AND status NOT IN ('closed', 'resolved')`, reassigns each to the delegate (or to the team queue if no delegate), records `reason: vacation_handover` in `assignments`, notifies the delegate via Slack/email. On the return date, tickets stay where they are; the returned agent gets a digest of what happened in their absence. New incoming tickets for which the OOO agent would have been the round-robin pick: skip them in the rotation while OOO.

**8. Spam.**

Layered defense at the email channel: (1) SPF/DKIM/DMARC at the SMTP layer; reject obviously spoofed mail. (2) SpamAssassin or AWS SES spam check on accepted mail; mail with high spam score goes to a quarantine mailbox, not the ticket DB. (3) Rate limiter per sender domain/IP: more than N messages per minute drops to quarantine. (4) Allowlist of known customer domains: their mail bypasses aggressive filtering. (5) Periodic human review of quarantine to catch false positives. A 99% spam filter at the front door drops most volume; the remaining ~100 spammy tickets per day are closed as `spam` by agents (with a one-click action).

**9. KB suggestions at intake.**

Pre-submit: as the customer types in the web form, debounce 500ms then call `POST /api/v1/kb/suggest` with the current subject+body. The endpoint hits Elasticsearch (top-3 articles by relevance score) and returns titles + snippets within 50ms. Display them inline: "Maybe one of these solves your issue?" Track if the customer clicks (`kb.suggestion.click`), abandons the form (`kb.suggestion.deflection`), or submits anyway. Deflection is the win metric; mature setups deflect 20-40% of would-be tickets.

**10. "Average time to resolve" is wrong.**

Three common bugs in this metric:

- **Includes `waiting_on_customer` time.** The agent is not working; the customer is. Should subtract the time spent in that status.
- **Wall-clock, ignoring business hours.** A ticket created Friday 5 PM and resolved Monday 10 AM is not 65 hours; it is ~3 business hours.
- **Counts spam and duplicate tickets.** These close instantly and skew the average down. Exclude `final_state IN ('spam', 'duplicate')`.

The correct computation: `sum of business-hours-active time per ticket / count of resolved tickets, excluding spam/duplicate`. Show p50 and p95 alongside the average; the average alone hides long-tail outliers.

### 13. Trade-offs

- **Push assignment vs pull from queue.** Push (round robin / skill / load) means every ticket has an owner instantly. Pull means tickets sit in a team queue until an agent claims one. Push is better for SLA accountability and customer "you have been assigned to Bob" emails. Pull is better for agent autonomy and prevents the case where the round-robin assigns to an agent who just logged off. Most production setups push by default, with pull as a fallback when the assigned agent is unavailable.
- **Canned responses vs ML-suggested replies.** Canned responses are agent-curated reply templates with placeholders. Predictable, controllable, agents stay in the driver's seat. ML-suggested replies (look at the ticket, suggest a draft) save agent typing time but risk wrong suggestions that the agent rubber-stamps. Best practice: keep canned responses for the top 50 known scenarios; layer suggestions on top for the long tail; never auto-send.
- **Embedded chat session vs ticket per session.** Some products treat every chat session as a ticket; others batch a customer's chat history into one rolling ticket. Per-session is cleaner for SLA tracking; rolling gives agents context. Pick one and stick to it; mixing causes reporting confusion.
- **Postgres FTS vs Elasticsearch for search.** FTS gets you to ~1M tickets cheaply. ES is a separate cluster to operate but gives faceted search, fuzzy matching, and relevance tuning. Switch when search latency on FTS exceeds 500ms or when product asks for facets.
- **Custom engine vs Zendesk/Freshdesk.** If your support volume is small, buy. The build-vs-buy line is around when you have unusual integrations or a workflow that does not fit the SaaS model (regulated industries, embedded support inside another product, deep cross-system automation). Below that line, do not build this.
- **Per-ticket scheduled timers vs sweep worker.** Per-ticket timers (one cron entry per ticket, or one delayed message per ticket in a queue) sound clean but cancellation on resolve/pause becomes brittle. Sweep worker is boring but reliable. Sweep wins at scale.

### 14. Common interview mistakes

1. **Treating tickets as CRUD with a `status` column and hardcoded transitions.** The state machine is the system. If your `status` is a free-form string and any agent can flip to any value, you have built a Trello clone, not a help desk.
2. **No intake adapter layer.** Skipping straight to "the API takes JSON" misses the whole email-parsing-and-threading problem, which is the single hardest part.
3. **Round-robin assignment as the default.** Works for the smallest team. Breaks at 20+ agents where skill matters or 100+ agents where load matters. Acknowledge the trade-off and offer skill-based as the next step up.
4. **Wall-clock SLA, no business hours.** Most candidates skip this. Real customers will not tolerate a 24-hour SLA that expires Sunday at 3 AM. Show you have thought about business hours and timezones.
5. **No SLA pause for `waiting_on_customer`.** Without pause, every ticket that needs customer info eventually breaches SLA through no fault of the agent. The SLA system loses credibility and agents game it ("close and reopen later").
6. **Per-ticket scheduled timer.** Tempting because it is "real-time". A nightmare at scale: 150k pending callbacks, cancellation races, fragile failure modes. Sweep is the right answer.
7. **No reopen handling.** Customer replies to a closed ticket; new ticket opens; conversation history is lost. Build the 7-day reopen window from day one.
8. **Email threading via subject only.** Subjects get rewritten, prefixed, translated. Use `In-Reply-To` headers as the primary signal; subject as fallback.
9. **No assignment race control.** Two agents claim the same ticket. Without `SKIP LOCKED` (pull) or row locks (push), you get split-brain assignments.
10. **Treating notifications as part of the ticket service.** They are not. Tickets emit events; the notification service consumes. Coupling them means you cannot swap Slack for Teams without touching the core.

If you hit 7 of these 10, you are interviewing at senior or staff level. The most common shallow answer is "tickets table with a status column and a queue"; the state-machine/intake-adapter/SLA-business-hours triad is what separates a thoughtful design from a generic CRUD answer.
