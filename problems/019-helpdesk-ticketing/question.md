---
id: 19
title: Design a Help Desk Ticketing System (Zendesk basic)
category: Workflow
topics: [state machine, sla, assignment, multi-channel intake, escalation]
difficulty: Easy
solution: solution.md
---

## Scene

The interviewer drops a single line into the doc.

> *"Design a help desk ticketing system. Customers submit tickets via email, chat, web form, and Slack. Agents respond, escalate, and close them. Build it like a basic Zendesk."*

This problem looks like CRUD with a `status` column. It is not. The interesting questions are: how do you turn a stray IMAP email into a structured ticket, how do you thread a customer's follow-up reply into the same ticket instead of opening a new one, how do you assign work fairly across 200 agents on three continents, and how do you detect SLA breaches *while respecting business hours* in the customer's timezone.

The companies who do this well (Zendesk, Freshdesk, Intercom, ServiceNow) all wrestle with the same five problems: intake adapters, ticket state, assignment, SLA timers, and reporting. Get the data model and the state machine right and the rest follows.

## Step 1: clarify before you design

Take 5 minutes. Aim for 8 questions that change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Channels.** "Email only, or also chat, web form, Slack, SMS, social DMs?" Each channel is an intake adapter with its own protocol, latency, and threading semantics. Email is the messiest because of reply threading; chat is the simplest because the session ID gives you the ticket boundary.
2. **Ticket volume and team size.** "How many tickets per day? How many agents? How many teams or queues?" Drives capacity, assignment strategy, and whether you need a search index. 50 tickets/day with 3 agents is a Google Sheet. 50k tickets/day with 2000 agents is a sharded system.
3. **SLA targets.** "Are there SLAs per priority? Per customer tier? Are they business hours or wall clock?" First response in 1 hour and resolution in 24 hours are common. Business hours mean timers pause nights, weekends, and holidays in the customer's timezone, which is the single most error-prone part of the system.
4. **Assignment strategy.** "Round robin, skill-based routing, queue-pull (agents claim next), or load-balanced to least-busy?" Each has different operational properties. The wrong choice causes burnout (load on a few agents) or low throughput (everyone waiting on the queue).
5. **Multi-team routing.** "Does a ticket route to a single team, or can it bounce between billing and engineering? Who decides which team owns it at intake?" Triage rules at intake versus mid-flight reassignment change the API and audit story.
6. **Customer experience.** "Do customers have a portal to view ticket status? Can they reply by email and have it thread back? Are CSAT surveys part of this?" The customer surface is half the product; do not skip it.
7. **Escalation rules.** "What happens when an SLA breaches? Auto-escalate to next tier? Page a manager? Email an executive sponsor?" Escalation is the safety valve; without it the queue grows silently.
8. **Knowledge base and self-service.** "Should the system suggest articles from the KB at ticket creation so the customer self-serves before submitting? Should agents see suggested articles while drafting replies?" Deflection is the cheapest win in any help desk.
9. **Integrations.** "Does it integrate with CRMs (Salesforce), engineering issue trackers (Jira), or status pages (Statuspage)? Real-time sync or batch?" Often deferred to a later phase but shapes the event bus.
10. **Compliance.** "Are we under HIPAA, GDPR, PCI? Do we redact PII from tickets? How long do we retain closed tickets?" Drives encryption, redaction, and retention.

A strong candidate also asks what is *out of scope*. Voice/phone tickets (a different beast, usually a separate Aircall/Twilio integration). Workflow automation (macros, triggers) as a v2 feature. Forecasting and workforce management.

</details>

## Step 2: capacity estimates

The interviewer gives you two scenarios.

**A. Small startup.**
- 50 tickets/day
- 3 agents
- Single channel: email
- Business hours support: 9 AM to 6 PM Pacific, weekdays

**B. Enterprise.**
- 50,000 tickets/day
- 2000 agents across 4 regions
- 5 channels: email, web form, chat, Slack, Twilio SMS
- 24/7 follow-the-sun support
- 200 teams (billing, technical, account management, per product line, per region)

Compute for each:

1. Tickets per second sustained and peak
2. Messages per ticket on average (initial + replies)
3. Total messages per second
4. Active (not-closed) tickets at any moment
5. Agent dashboard read load (each agent refreshes every ~30 seconds)
6. Storage for 5 years of ticket data

<details>
<summary><b>Reveal: the math at both scales</b></summary>

**A. Small startup:**

- Tickets: 50/day = ~0.0006/sec. About 1 ticket every 30 minutes during business hours. *Trivial.*
- Messages per ticket: typically 4 to 6 (customer ask, agent reply, customer follow-up, agent resolve). Call it 5.
- Total messages: 250/day = ~10/business hour.
- Active tickets: 50/day with ~2-day median resolution = ~100 open at any moment.
- Dashboard reads: 3 agents × 2 per minute × 8 hours = ~3000/day, peak ~3/min.
- Storage: 50 × 365 × 5 = ~91k tickets in 5 years. Each ticket plus messages is ~10KB. ~1GB total. Negligible.

You could build this with a Google Sheet, a Gmail label, and a cron job. We do not, because the goal is to design the same shape that scales to enterprise.

**B. Enterprise:**

- Tickets: 50k/day = ~0.6/sec sustained. Business-day skew puts peak around 2 to 3 per second. With 24/7 ops, peaks soften.
- Messages per ticket: 6 to 10 for enterprise tickets (longer threads, more back-and-forth). Call it 8.
- Total messages: 400k/day = ~5/sec sustained, ~15/sec peak.
- Active tickets: 50k/day × ~3-day median resolution = ~150k open. Long-running ones (engineering escalations) can sit open for weeks.
- Dashboard reads: 2000 agents × 2/minute × 8 hours (per shift, three shifts) = ~3M reads/day = ~35/sec sustained, ~100/sec peak. Adds a customer portal: another ~10/sec.
- Storage: 50k × 365 × 5 = ~91M tickets, ~730M messages. At ~2KB per message + attachments = ~3 to 5 TB raw text, plus another ~50 TB for attachments stored in S3.

**Key insights from the math:**

- This is not a high-throughput system. Even at enterprise scale, the write rate is ~15/sec. The hard part is not raw QPS; it is **state correctness, fair assignment under concurrency, and SLA accuracy across timezones**.
- "Active tickets" is the meaningful metric. 150k open tickets means every agent dashboard query must hit a covering index in under 50ms.
- Reads dominate writes by 10x or more once you count agent dashboards plus customer portal plus reporting. A read cache and search index do most of the heavy lifting.
- Storage is modest in raw bytes, but attachments dominate volume. Stream attachments to object storage at intake; do not put binary blobs in your ticket DB.

</details>

## Step 3: the ticket lifecycle state machine

A ticket is fundamentally a small state machine. Sketch the states and transitions. Fill in the missing arrows and conditions.

```
                     ┌──────────┐
       intake ────► │   new    │
                     └────┬─────┘
                          │ triage rules pick owning team
                          ▼
                     ┌──────────┐
                     │ assigned │  (a specific agent owns it)
                     └────┬─────┘
                          │ agent opens / starts work
                          ▼
                     ┌──────────────┐
                     │ in_progress  │
                     └──┬─────┬─────┘
                        │     │
        agent replies   │     │ agent reassigns
        & needs info    │     │
                        ▼     ▼
        ┌──────────────────────┐    [ ? ]   (back to assigned with new owner)
        │ waiting_on_customer  │
        └──────────┬───────────┘
                   │ customer replies
                   ▼
              [ ? ]   (re-enters which state?)

        From in_progress:
                   │
                   ▼ agent marks fixed
              ┌──────────┐
              │ resolved │
              └────┬─────┘
                   │ customer accepts (or auto-close timer)
                   ▼
              ┌──────────┐
              │  closed  │
              └──────────┘

        Any state can also transition to:
              [ ? ]   (spam, duplicate, withdrawn)
```

<details>
<summary><b>Reveal: the complete state machine</b></summary>

```
                       ┌──────────┐
       intake ──────► │   new    │
                       └────┬─────┘
                            │ triage rules pick owning team
                            ▼
                       ┌──────────┐  ◄────────────────┐
                       │ assigned │                   │
                       └────┬─────┘                   │
                            │ agent opens             │ reassign
                            ▼                         │
                  ┌────────────────────────┐          │
                  │     in_progress        │──────────┘
                  └─┬─────────┬─────────┬──┘
                    │         │         │
        wait info   │         │ fixed   │ reassign team
                    ▼         ▼         ▼
       ┌──────────────────┐ ┌──────┐  ┌──────────┐
       │waiting_on_cust   │ │resolved│ │ assigned │ (new team)
       └────┬─────────┬───┘ └───┬───┘ └──────────┘
            │         │         │
   customer │         │ 7-day   │ customer accepts
   replies  │         │ silence │ or 72h auto-accept
            │         │         │
            ▼         ▼         ▼
      ┌─────────────┐   ┌──────────┐
      │ in_progress │   │  closed  │
      └─────────────┘   └─────┬────┘
                              │ customer replies within 7 days
                              ▼
                       ┌──────────────┐
                       │  reopened    │ (back to assigned with same agent
                       └──────────────┘  if still active, else reassigned)

        Terminal exits from any non-terminal state:
              ┌──────────┐  ┌────────────┐  ┌────────────┐
              │   spam   │  │ duplicate  │  │ withdrawn  │
              └──────────┘  └────────────┘  └────────────┘
```

**Why each transition matters:**

- **new → assigned.** Triage rules at intake assign the owning team and (optionally) the specific agent. Routing is data-driven (channel, keywords, customer tier).
- **assigned → in_progress.** The agent actively starts work. This is when the "agent response time" SLA stops.
- **in_progress → waiting_on_customer.** The agent needs information. Crucially, the SLA timers *pause* here; you cannot blame the agent for waiting on the customer.
- **waiting_on_customer → in_progress.** Customer reply auto-resumes the timers. The agent gets a notification.
- **in_progress → resolved.** Agent believes the issue is fixed. Customer has a window (often 72 hours) to confirm or push back.
- **resolved → closed.** Customer accepts, or the auto-close timer expires.
- **closed → reopened.** Customer replies to a closed ticket within a grace period (usually 7 days). Reopens with SLA reset.
- **assigned/in_progress → assigned (different team).** Reassignment. Common: customer service triages, realizes it is a billing issue, hands off.
- **Any → spam/duplicate/withdrawn.** Terminal exits without a "resolution." Important for reporting (these do not count against your SLA stats).

**Reopen is the trap.** A naive design lets a closed ticket sit forever; a customer's reply weeks later just creates a new ticket. That fragments the history. The right design has a reopen window (7 days) where a reply reactivates the original; after that, a new ticket is created but *linked* to the closed one for context.

</details>

## Step 4: sketch the high-level architecture

Fill in the five `[ ? ]` placeholders. Each box represents a major role: ingestion across channels, ticket persistence, assignment logic, SLA timing, and the knowledge base.

```
   Email (IMAP/SES)  Web form  Chat (websocket)  Slack  Twilio SMS
        │              │            │              │         │
        └──────────────┴────────────┴──────────────┴─────────┘
                                │
                                ▼
                       ┌──────────────────┐
                       │   [ ? ]          │  (per-channel intake adapters;
                       │                  │   normalize to Ticket event)
                       └─────────┬────────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │   [ ? ]          │  (CRUD on tickets, messages;
                       │                  │   state machine executor)
                       └──┬─────┬──────┬──┘
                          │     │      │
                          │     │      └───────► ┌──────────────┐
                          │     │                │   [ ? ]      │  (assign to team/agent;
                          │     │                │              │   round-robin / skill /
                          │     │                │              │   queue / load-balanced)
                          │     │                └──────────────┘
                          │     │
                          │     └─────────────► ┌──────────────┐
                          │                     │   [ ? ]      │  (per-ticket SLA timer,
                          │                     │              │   fires escalations on breach)
                          │                     └──────────────┘
                          │
                          ▼
                  ┌──────────────┐
                  │  Tickets DB  │  (Postgres: tickets, ticket_messages,
                  │              │   assignments, sla_targets, audit)
                  └──────────────┘

         Side channel from intake:
                  ┌──────────────┐
                  │   [ ? ]      │  (semantic / keyword search over articles;
                  │              │   surfaces suggestions at intake and for agents)
                  └──────────────┘

         Agent and customer reads:
                  ┌──────────────┐         ┌──────────────┐
                  │  Read Cache  │ ◄────── │  Search Index│  (Elasticsearch / OpenSearch
                  │  (Redis)     │         │              │   over ticket content + meta)
                  └──────────────┘         └──────────────┘
```

<details>
<summary><b>Reveal: completed architecture</b></summary>

```
   Email (IMAP/SES)  Web form  Chat (websocket)  Slack  Twilio SMS
        │              │            │              │         │
        └──────────────┴────────────┴──────────────┴─────────┘
                                │
                                ▼
                       ┌──────────────────────┐
                       │  Intake Adapters     │  (one per channel; each
                       │  (email parser,      │   parses, threads, normalizes;
                       │   form receiver,     │   emits TicketIntakeEvent)
                       │   chat ingest, etc.) │
                       └─────────┬────────────┘
                                 │
                                 ▼
                       ┌──────────────────────┐
                       │  Ticket Service      │  (create/update tickets,
                       │  (state machine,     │   append messages,
                       │   API for agents     │   enforce transitions)
                       │   and customers)     │
                       └──┬─────┬──────┬──────┘
                          │     │      │
                          │     │      └───────► ┌──────────────────┐
                          │     │                │  Assignment      │
                          │     │                │  Service         │
                          │     │                │  (skill matching,│
                          │     │                │   load tracking, │
                          │     │                │   queues)        │
                          │     │                └──────────────────┘
                          │     │
                          │     └─────────────► ┌──────────────────┐
                          │                     │  SLA Timer       │
                          │                     │  Worker          │
                          │                     │  (sweeps tickets,│
                          │                     │   fires escalate │
                          │                     │   events, pauses │
                          │                     │   outside hours) │
                          │                     └──────────────────┘
                          │
                          ▼
                  ┌──────────────────┐
                  │  Tickets DB      │
                  │  (Postgres)      │
                  │                  │
                  │  tickets         │
                  │  ticket_messages │
                  │  assignments     │
                  │  sla_targets     │
                  │  ticket_audit    │
                  └──┬───────────────┘
                     │  CDC / outbox
                     ▼
                  ┌──────────────────┐
                  │  Kafka           │
                  │  ticket.created  │
                  │  ticket.replied  │
                  │  ticket.assigned │
                  │  sla.breached    │
                  └──┬────────┬──────┘
                     │        │
                     ▼        ▼
              ┌──────────┐ ┌──────────────────┐
              │ Indexer  │ │ Notifications    │  (email, Slack DM,
              │ → ES     │ │ Service          │   push, in-app)
              └────┬─────┘ └──────────────────┘
                   │
                   ▼
              ┌──────────────┐
              │  Search Index│  ◄── KB Lookup queries
              │  (ES/OS)     │      (intake + agent compose)
              └──────────────┘
```

What each component owns:

- **Intake adapters.** One process per channel. Email adapter polls IMAP or receives SES inbound; parses the message; checks `In-Reply-To`/`References` headers to thread into an existing ticket; emits a unified `TicketIntakeEvent`. Web form posts JSON. Chat keeps a websocket and ships each session as a ticket. Each adapter is independently deployable and crash-recoverable.
- **Ticket Service.** The state machine executor. Owns the API for create, reply, change status, assign, search. Validates transitions (you cannot go from `closed` to `in_progress` without going through `reopened`). Writes to the Tickets DB transactionally.
- **Assignment Service.** Given a ticket, decides which team and (optionally) which agent. Looks up team queues, agent skills, current load. Returns an assignment that the Ticket Service writes to the DB.
- **SLA Timer Worker.** Background sweep every ~30 seconds. Finds tickets approaching or past their SLA target. Pauses during waiting-on-customer and outside business hours. Fires `sla.breached` events; an escalation consumer reassigns the ticket up the chain.
- **Knowledge Base Search.** Elasticsearch (or OpenSearch) index over KB articles. Called at intake to suggest self-service articles to the customer; called by agents when composing a reply.
- **Read Cache + Search Index.** Agent dashboards hit cache; ticket search hits Elasticsearch. The primary Postgres handles writes and the long-tail "show me this exact ticket" lookups.

</details>

## Step 5: assignment strategies

You have a fresh ticket. Which agent gets it? There are four serious strategies. Each is simple. Picking the wrong one for your team shape causes a real, painful operational problem.

Take 10 minutes. Write down how each strategy works and what its failure mode is, before peeking.

<details>
<summary><b>Reveal: comparison of the four strategies</b></summary>

| Strategy | How it works | Pros | Cons | Best for |
|----------|--------------|------|------|----------|
| **Round robin** | Maintain a pointer per team. Next ticket goes to `agents[i % N]`, increment. | Trivially simple. Perfectly fair *if all tickets are equal*. | Ignores current load. Ignores skill. An agent on a long ticket still receives new ones. | Small teams (under 10) with similar skills and similar ticket weight. |
| **Skill-based** | Tag the ticket at intake (billing, technical, mobile, enterprise). Match to agents who hold the required skill tags. Within match, fall back to round robin or load. | Routes hard tickets to people who can solve them. Reduces handoffs. | Requires accurate skill tagging on both tickets and agents. Stale skill lists cause silent misrouting. Cold-start problem for new agents. | Specialized teams (technical support with 5+ skill domains). |
| **Queue-based (pull)** | Tickets land in a team queue. Agents click "Next Ticket" when free. The system gives them the oldest unassigned in their queue. | Self-paced. Agents never overloaded. Naturally load-balanced. | Some tickets sit longer if all agents avoid them ("nobody wants the angry customer"). Requires discipline. Hard to enforce SLA without push fallback. | Mature teams that prefer autonomy (think ServiceNow, common in IT support). |
| **Load-balanced (push to least-busy)** | System tracks `active_ticket_count` per agent. New ticket goes to the agent with the fewest open tickets, breaking ties by least-recent-assignment. | Even load by construction. No agent gets crushed. | Misses skill. Misses ticket complexity (one ticket can be 10x harder than another). | Mid-size teams with homogenous skills, where load fairness matters more than skill specialization. |

**The real answer in production:** hybrid. Most help desks use **skill-based to pick the candidate pool, then load-balanced within that pool, with a queue-pull fallback for overflow**. Round robin is rare in production beyond very small teams.

**A subtle point on push vs pull:** push assignment (round-robin, skill, load) means every ticket has an owner the instant it arrives. Pull (queue) means tickets without an owner accumulate until an agent grabs them. Push is better for SLA accountability. Pull is better for agent autonomy and prevents bottlenecks when a particular agent goes offline mid-ticket.

**Race condition to watch:** two assigners running in parallel can both pick the same agent for two different tickets, blowing past the load target. Solve with a row-level lock on the agent's load counter, or a single-writer per team.

</details>

## Step 6: SLA timers and escalation

A common SLA: **first response in 1 hour, full resolution in 24 hours**, for priority "high" tickets. On breach, auto-escalate: notify the team lead at 30 minutes (warning), reassign to a senior agent at 60 minutes (first breach), notify the manager at 90 minutes.

Two things make this harder than it looks:

1. **Business hours.** A ticket created Friday 5 PM with a 1-hour first-response SLA does not breach at 6 PM Friday. It pauses at end of business and resumes Monday 9 AM. If the customer is in Tokyo and the team is in San Francisco, *whose* business hours?
2. **Pauses.** When the agent moves the ticket to `waiting_on_customer`, the SLA clock pauses. When the customer replies, it resumes.

Sketch the SLA timer logic. How would you store it? How does the worker decide what to fire?

<details>
<summary><b>Reveal: SLA timer model</b></summary>

```sql
CREATE TABLE sla_targets (
    ticket_id           UUID PRIMARY KEY,
    priority            TEXT NOT NULL,            -- 'low' | 'medium' | 'high' | 'urgent'
    first_response_due  TIMESTAMPTZ,              -- computed deadline (business-hours aware)
    resolution_due      TIMESTAMPTZ,
    elapsed_business_ms BIGINT NOT NULL DEFAULT 0,-- accumulated active time
    paused_at           TIMESTAMPTZ,              -- non-null when timer is paused
    pause_reason        TEXT,                     -- 'waiting_on_customer' | 'out_of_hours'
    business_hours_id   TEXT NOT NULL,            -- which schedule applies (team or customer)
    first_response_at   TIMESTAMPTZ,              -- when first agent reply landed
    resolved_at         TIMESTAMPTZ,
    breach_state        TEXT NOT NULL DEFAULT 'on_track' -- 'on_track' | 'warning' | 'breached'
);
CREATE INDEX idx_sla_due ON sla_targets (first_response_due)
    WHERE first_response_at IS NULL AND paused_at IS NULL;
CREATE INDEX idx_sla_resolution_due ON sla_targets (resolution_due)
    WHERE resolved_at IS NULL AND paused_at IS NULL;
```

**Computing the deadline.**

`first_response_due` is not `created_at + 1 hour`. It is `created_at + 1 hour of business time`, projected forward through the business hours schedule.

```python
def add_business_time(start, duration, schedule):
    """Add `duration` (timedelta) of business time to `start` (timestamp),
       skipping nights, weekends, and holidays per `schedule`."""
    cursor = start
    remaining = duration
    while remaining > 0:
        if not schedule.is_business_time(cursor):
            cursor = schedule.next_business_window_start(cursor)
            continue
        window_end = schedule.current_window_end(cursor)
        chunk = min(remaining, window_end - cursor)
        cursor += chunk
        remaining -= chunk
    return cursor
```

This is recomputed only when the SLA is set (at create) or when the schedule changes. The deadline is stored as a wall-clock timestamp so the worker can do a simple `WHERE first_response_due < NOW()` query.

**The SLA timer worker (simplified):**

```python
def sla_sweep():
    # First-response breaches
    candidates = db.query("""
      SELECT ticket_id, first_response_due, priority
      FROM sla_targets
      WHERE first_response_at IS NULL
        AND paused_at IS NULL
        AND first_response_due < NOW() + interval '30 minutes'
        AND breach_state != 'breached'
    """)
    for c in candidates:
        if c.first_response_due < now() - interval('0'):
            emit("sla.first_response_breached", c.ticket_id)
            db.update("sla_targets", c.ticket_id, breach_state='breached')
        elif c.first_response_due < now() + interval('30 minutes'):
            emit("sla.first_response_warning", c.ticket_id)
            db.update("sla_targets", c.ticket_id, breach_state='warning')

    # Resolution breaches: same pattern with resolution_due
    # ...
```

The worker runs every ~30 seconds. Emit-then-update is idempotent enough; if the worker crashes between emit and update, the next run re-emits and the consumer dedupes.

**Pause and resume.**

When a ticket moves to `waiting_on_customer`:

```python
def pause_sla(ticket_id, reason):
    target = db.get("sla_targets", ticket_id)
    if target.paused_at is not None:
        return  # already paused, idempotent
    # No SLA clock change here; we just mark it paused so the worker skips it.
    # The deadline is *extended* on resume.
    db.update("sla_targets", ticket_id, paused_at=NOW(), pause_reason=reason)

def resume_sla(ticket_id):
    target = db.get("sla_targets", ticket_id)
    if target.paused_at is None:
        return
    paused_duration = NOW() - target.paused_at
    db.update("sla_targets", ticket_id,
              paused_at=None,
              first_response_due=target.first_response_due + paused_duration,
              resolution_due=target.resolution_due + paused_duration)
```

Note: pause/resume extends the deadline by the wall-clock paused duration. If business hours overlap with the paused window, the math is more involved (you only want to extend by the *business* duration that was paused). The implementation uses the same `add_business_time` helper to walk forward.

**Escalation on breach.**

```yaml
escalation_policy: high_priority_default
rules:
  - on: sla.first_response_warning      # 30 min before
    action: notify
    recipient: team_lead
  - on: sla.first_response_breached     # 0 min
    action: reassign
    target: senior_agent_pool
    keep_original_assignee_cc: true
  - on: sla.first_response_breached + 30min
    action: notify
    recipient: support_manager
    via: [email, slack, pagerduty]
```

The escalation consumer reads `sla.*` events and applies the matching policy. Reassignment is recorded as a normal ticket assignment event (audit picks it up).

**Why a worker, not per-ticket timers.**

A per-ticket scheduled callback (cron job per ticket, or a delayed-queue entry per ticket) sounds elegant but at 150k open tickets you have 150k pending callbacks. Cancellation on resolve/pause/reassign becomes a coordination nightmare. The sweep-the-table pattern is boring, durable, and easy to reason about. The cost is a few seconds of breach-detection latency.

</details>

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. **Email threading.** A customer replies to "Ticket #4521: payment failed" from their phone, which strips the subject prefix. How do you still thread their reply into the existing ticket and not open a new one?

2. **The intake adapter crashes mid-batch.** The email adapter has pulled 50 messages from IMAP and crashed after processing 30. On restart, how do you avoid processing the first 30 again or losing the last 20?

3. **Two agents claim the same queued ticket.** Both click "Next Ticket" within the same second. The query returns the same ticket to both. How do you ensure only one gets it?

4. **SLA business hours across timezones.** Customer in Tokyo, team in San Francisco. Whose business hours? What if the contract says "follow the sun support"?

5. **Ticket re-open after long silence.** A customer replies to a ticket closed 6 months ago. What does the system do? Where does the conversation live?

6. **Conversation threading vs new tickets.** A customer opens a ticket about billing. While that is open, they email again about a totally different issue (broken login). Do you append to the existing ticket or open a new one? How does the system decide?

7. **Agent goes on vacation mid-ticket.** Bob has 47 open tickets and starts a 2-week vacation. How do you hand them off? Manually one at a time? Bulk reassign?

8. **Spam at the front door.** Your inbound email address receives 10,000 spam messages per day. How do you avoid filling the ticket DB with garbage?

9. **Knowledge base suggestions at intake.** When a customer fills the web form, before submitting, you want to suggest 3 KB articles that might solve their issue. How do you do this without slowing the form?

10. **Reporting on "average time to resolve" is wrong.** Management complains the dashboard says 2 hours but tickets take days. What is going wrong, and how do you compute it correctly?

## Related problems

- **[Approval Management (011)](../011-approval-management/question.md)**. Shares the workflow engine + state machine + role routing + escalation patterns. If you have built one, the other is mostly a relabeling.
- **[Notification System (010)](../010-notification-system/question.md)**. Every state change (assigned, replied, breached, resolved) fans out to email, Slack, push. The fan-out, retry, and quiet-hours machinery there is what consumes ticket events.
- **[Comment System (015)](../015-comment-system/question.md)**. A ticket's `ticket_messages` are the same shape as comments on a post: threaded, paginated, with attachments. Reuse the same storage and indexing patterns.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md)**. Agent dashboards and customer portals load tickets thousands of times per day. Cache tiering, read replicas, and denormalized views from that problem apply directly here.
