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

This looks like CRUD with a `status` column. It isn't. The hard parts are turning a stray IMAP email into a structured ticket, threading a customer's follow-up reply into the same ticket instead of spawning a new one, fairly distributing work across 200 agents on three continents, and detecting SLA breaches *while respecting business hours in the customer's timezone*.

Zendesk, Freshdesk, Intercom, ServiceNow all wrestle with the same five problems: intake adapters, ticket state, assignment, SLA timers, and reporting. Nail the data model and the state machine first; the rest falls out.

## Step 1: clarify before you design

Take five minutes. Aim for eight questions that actually change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

The two most load-bearing questions are about *channels* and *SLA semantics*. Everything else flows from those.

**Channels.** Email only, or also chat, web form, Slack, SMS, social DMs? Each channel is a separate intake adapter with its own protocol, latency, and threading rules. Email is the messiest because of reply threading. Chat is the simplest because the session ID already gives you the ticket boundary.

**Ticket volume and team size.** How many tickets per day, how many agents, how many teams or queues? 50 tickets a day with three agents is a Google Sheet. 50k tickets a day with 2000 agents is a sharded system.

**SLA targets.** Per priority? Per customer tier? Business hours or wall clock? "First response in one hour, resolution in 24 hours" is common. Business hours means timers pause nights, weekends, and holidays in the customer's timezone, which is the single most error-prone part of the system.

**Assignment strategy.** Round robin, skill-based, queue-pull (agents claim the next one), or load-balanced to the least busy? The wrong choice causes burnout on a few agents or low throughput because everyone waits on the queue.

**Multi-team routing.** Does a ticket route to a single team, or can it bounce between billing and engineering? Who decides ownership at intake? Triage at intake versus mid-flight reassignment changes the API and the audit story.

**Customer experience.** Portal for ticket status? Email replies threading back? CSAT surveys? The customer surface is half the product. Don't skip it.

**Escalation rules.** What happens when an SLA breaches? Auto-escalate to the next tier, page a manager, email an executive sponsor? Without escalation the queue grows silently.

**Knowledge base.** Should the system suggest articles at ticket creation so the customer self-serves? Should agents see suggestions while drafting? Deflection is the cheapest win in any help desk.

**Integrations.** Salesforce, Jira, Statuspage? Real-time or batch? Often deferred but shapes the event bus.

**Compliance.** HIPAA, GDPR, PCI? Redact PII? Retention windows for closed tickets? Drives encryption, redaction, retention.

Also ask what's *out of scope*. Voice/phone tickets are a separate beast (usually a Twilio/Aircall integration). Macros and triggers belong in a v2. Workforce management and forecasting are their own product.

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

Compute for each: tickets/sec sustained and peak, average messages per ticket, total messages/sec, active (not-closed) tickets at any moment, agent dashboard read load (each agent refreshes every ~30 seconds), and storage for five years.

<details>
<summary><b>Reveal: the math at both scales</b></summary>

**Small startup.** 50 tickets a day is one every 30 minutes during business hours. Trivial. About 5 messages per ticket on average (customer ask, agent reply, customer follow-up, agent resolve, close). That's 250 messages a day, around 10 per business hour. With a 2-day median resolution time, ~100 tickets are open at any moment. Agents refreshing twice a minute over an 8-hour shift gets you ~3000 reads/day, peaking around 3/min. Five years of data lands at ~91k tickets, around 1 GB. You could build this on a Google Sheet, a Gmail label, and a cron job. We don't, because the goal is a design that scales to the next box.

**Enterprise.** 50k/day works out to 0.6/sec sustained, 2-3/sec at peak. With 24/7 ops the peaks soften. Enterprise tickets run longer (6-10 messages), call it 8, so messages land at ~5/sec sustained and 15/sec at peak. Active ticket count is the meaningful number: 50k/day with a 3-day median resolution puts ~150k open at any moment, with long-running engineering escalations sitting for weeks. Dashboards add ~35/sec sustained from agents and another ~10/sec from the customer portal. Storage runs ~3-5 TB of text plus ~50 TB of attachments in S3 over five years.

Two things to notice from the math. First, this is not a high-throughput system. Even at enterprise scale the write rate is ~15/sec. The hard part isn't QPS; it's state correctness, fair assignment under concurrency, and SLA accuracy. Second, reads beat writes by 10x or more once you count dashboards plus portal plus reporting. A read cache and a search index do most of the heavy lifting.

</details>

## Step 3: the ticket lifecycle state machine

A ticket is a small state machine. Sketch the states and transitions. Fill in the missing arrows.

```
                  ┌──────────┐
    intake ────► │   new    │
                  └────┬─────┘
                       │ triage rules pick owning team
                       ▼
                  ┌──────────┐
                  │ assigned │
                  └────┬─────┘
                       │ agent starts work
                       ▼
                  ┌──────────────┐
                  │ in_progress  │
                  └──┬─────┬─────┘
                     │     │
     agent needs     │     │ agent reassigns
     info from cust  │     │
                     ▼     ▼
     ┌──────────────────────┐    [ ? ]
     │ waiting_on_customer  │
     └──────────┬───────────┘
                │ customer replies
                ▼
           [ ? ]

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
    │waiting_on_cust   │ │resolved│ │ assigned │
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
                    │  reopened    │
                    └──────────────┘

     Terminal exits from any non-terminal state:
           ┌──────────┐  ┌────────────┐  ┌────────────┐
           │   spam   │  │ duplicate  │  │ withdrawn  │
           └──────────┘  └────────────┘  └────────────┘
```

A few transitions are worth saying out loud. `new → assigned` happens when triage rules pick the owning team and (optionally) a specific agent. `in_progress → waiting_on_customer` pauses the SLA timers; you can't blame the agent for waiting on the customer. `waiting_on_customer → in_progress` resumes the timers on customer reply. `resolved → closed` happens when the customer accepts or the 72-hour auto-close fires. `closed → reopened` happens when the customer replies within the 7-day grace window; after that, a reply creates a new ticket linked to the closed one via `parent_ticket_id`.

The reopen window is the trap. A naive design lets the customer reply weeks later and silently spawns a new ticket, fragmenting history. The 7-day window plus the parent-link pattern is the standard fix.

</details>

## Step 4: sketch the high-level architecture

Fill in the five `[ ? ]` placeholders. Each box is a major role: ingestion across channels, ticket persistence, assignment, SLA timing, and the knowledge base.

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
                          │     │                │   [ ? ]      │  (assign to team/agent)
                          │     │                └──────────────┘
                          │     │
                          │     └─────────────► ┌──────────────┐
                          │                     │   [ ? ]      │  (per-ticket SLA timer)
                          │                     └──────────────┘
                          │
                          ▼
                  ┌──────────────┐
                  │  Tickets DB  │  (Postgres)
                  └──────────────┘

         Side channel from intake:
                  ┌──────────────┐
                  │   [ ? ]      │  (KB search; suggests articles at
                  │              │   intake and for agent compose)
                  └──────────────┘
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
                       │  Intake Adapters     │  (one per channel; parses,
                       │  (email, form, chat, │   threads, normalizes;
                       │   slack, sms)        │   emits TicketIntakeEvent)
                       └─────────┬────────────┘
                                 │
                                 ▼
                       ┌──────────────────────┐
                       │  Ticket Service      │  (state machine, append
                       │                      │   messages, enforce
                       │                      │   transitions)
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
                          │                     │  (sweeps, fires  │
                          │                     │   escalations,   │
                          │                     │   pauses outside │
                          │                     │   business hours)│
                          │                     └──────────────────┘
                          │
                          ▼
                  ┌──────────────────┐
                  │  Tickets DB      │
                  │  (Postgres)      │
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

Each component carries one job. Intake adapters speak one channel each. The email adapter polls IMAP or accepts SES inbound, parses the message, walks `In-Reply-To`/`References` to thread into an existing ticket, and emits a unified intake event. Web form posts JSON. Chat keeps a websocket and ships each session. Each adapter is independently deployable and crash-recoverable.

The Ticket Service is the state machine executor. It owns the API for create, reply, status change, assign, and search. It refuses bad transitions (no `closed → in_progress` without going through `reopened`) and writes to Postgres in transactions.

The Assignment Service decides team and agent. It looks up team queues, agent skills, current load, and returns an assignment that the Ticket Service persists.

The SLA Timer Worker sweeps every ~30 seconds, finds tickets approaching or past their deadlines, pauses during `waiting_on_customer` and outside business hours, and fires `sla.breached` events that an escalation consumer turns into reassignments or notifications.

KB Search is an Elasticsearch index over articles. At intake it suggests self-service articles to the customer; agents call the same index when composing.

The read cache and search index do the heavy read lifting. Dashboards hit cache. Search hits ES. Primary Postgres handles writes and "show me this exact ticket" lookups.

</details>

## Step 5: assignment strategies

You have a fresh ticket. Which agent gets it? Four serious strategies exist. Each is simple. Pick the wrong one for your team shape and you'll cause a painful operational problem.

Take ten minutes. Write down how each works and what its failure mode is, before peeking.

<details>
<summary><b>Reveal: comparison of the four strategies</b></summary>

| Strategy | How it works | Pros | Cons | Best for |
|----------|--------------|------|------|----------|
| Round robin | Pointer per team; next ticket goes to `agents[i % N]`, increment | Trivially simple. Fair if all tickets are equal weight | Ignores current load and skill. An agent on a long ticket still gets new ones | Small teams (under 10) with similar skills |
| Skill-based | Tag ticket at intake (billing, technical, mobile). Match to agents with the right skill tags. Within the match, round-robin or load-balance | Routes hard tickets to people who can solve them | Needs accurate skill tagging on both sides. Stale skill lists silently misroute | Specialized teams (technical support, 5+ skill domains) |
| Queue-pull | Tickets land in a team queue. Agents click "Next Ticket" when free. They get the oldest unassigned in their queue | Self-paced. Agents never overloaded. Naturally load-balanced | Some tickets sit longer ("nobody wants the angry customer"). Hard to enforce SLA without a push fallback | Mature teams that want autonomy (IT support, ServiceNow shops) |
| Load-balanced push | Track `active_ticket_count` per agent. New ticket goes to least busy, tie-break by least recently assigned | Even load by construction. No agent gets crushed | Ignores skill and ticket complexity (one ticket can be 10x harder) | Mid-size teams with homogenous skills |

In production, most help desks run a hybrid: skill-based to pick the candidate pool, then load-balanced inside that pool, with queue-pull as overflow. Round robin is rare beyond very small teams.

A subtle point on push vs pull. Push assignment gives every ticket an owner the instant it arrives. Pull means tickets without an owner pile up until an agent grabs them. Push is better for SLA accountability. Pull is better for autonomy and avoids the "ticket assigned to an agent who just went offline" trap.

The race condition to watch: two assigners running in parallel can both pick the same agent for two different tickets, blowing past the load target. Solve with a row-level lock on the agent's load counter, or a single-writer per team.

</details>

## Step 6: SLA timers and escalation

A common policy: first response in 1 hour, full resolution in 24 hours, for priority "high" tickets. On breach, auto-escalate: warn the team lead at 30 minutes, reassign to a senior agent at 60 minutes (first breach), notify the manager at 90 minutes.

Two things make this harder than it looks.

First, business hours. A ticket created Friday 5 PM with a 1-hour first-response SLA does not breach at 6 PM Friday. It pauses at end of business and resumes Monday 9 AM. And if the customer is in Tokyo and the team is in San Francisco, *whose* hours apply?

Second, pauses. When the ticket moves to `waiting_on_customer`, the SLA clock pauses. When the customer replies, it resumes.

Sketch the SLA timer logic. How would you store it? How does the worker decide what to fire?

<details>
<summary><b>Reveal: SLA timer model</b></summary>

```sql
CREATE TABLE sla_targets (
    ticket_id           UUID PRIMARY KEY,
    priority            TEXT NOT NULL,
    first_response_due  TIMESTAMPTZ,
    resolution_due      TIMESTAMPTZ,
    elapsed_business_ms BIGINT NOT NULL DEFAULT 0,
    paused_at           TIMESTAMPTZ,
    pause_reason        TEXT,                     -- 'waiting_on_customer' | 'out_of_hours'
    business_hours_id   TEXT NOT NULL,
    first_response_at   TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    breach_state        TEXT NOT NULL DEFAULT 'on_track'
);
CREATE INDEX idx_sla_due ON sla_targets (first_response_due)
    WHERE first_response_at IS NULL AND paused_at IS NULL;
CREATE INDEX idx_sla_resolution_due ON sla_targets (resolution_due)
    WHERE resolved_at IS NULL AND paused_at IS NULL;
```

The deadline is *not* `created_at + 1 hour`. It's `created_at + 1 hour of business time`, projected forward through the schedule.

```python
def add_business_time(start, duration, schedule):
    """Add `duration` of business time to `start`, skipping nights,
       weekends, and holidays per `schedule`."""
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

Recompute only when the SLA is set (at create) or when the schedule changes. Store the result as a wall-clock timestamp so the worker can run a simple `WHERE first_response_due < NOW()`.

The sweep worker:

```python
def sla_sweep():
    candidates = db.query("""
      SELECT ticket_id, first_response_due, priority
      FROM sla_targets
      WHERE first_response_at IS NULL
        AND paused_at IS NULL
        AND first_response_due < NOW() + interval '30 minutes'
        AND breach_state != 'breached'
    """)
    for c in candidates:
        if c.first_response_due < now():
            emit("sla.first_response_breached", c.ticket_id)
            db.update("sla_targets", c.ticket_id, breach_state='breached')
        elif c.first_response_due < now() + interval('30 minutes'):
            emit("sla.first_response_warning", c.ticket_id)
            db.update("sla_targets", c.ticket_id, breach_state='warning')
```

Runs every ~30 seconds. Emit-then-update is idempotent enough; if the worker dies between emit and update, the next sweep re-emits and the consumer dedupes.

Pause and resume:

```python
def pause_sla(ticket_id, reason):
    target = db.get("sla_targets", ticket_id)
    if target.paused_at is not None:
        return
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

Pause-resume extends the deadline by the wall-clock paused duration. If business hours overlap with the paused window you only want to extend by the *business* duration that was paused; same `add_business_time` helper handles that.

The escalation policy lives in config:

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

Why a worker, not per-ticket timers? A scheduled callback per ticket sounds elegant, but at 150k open tickets you'd have 150k pending callbacks, and cancellation on resolve/pause/reassign becomes a coordination nightmare. Sweep is boring, durable, easy to reason about. The cost is a few seconds of breach-detection latency, which nobody cares about.

</details>

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. **Email threading.** A customer replies to "Ticket #4521: payment failed" from their phone, which strips the subject prefix. How do you still thread the reply into the existing ticket?

2. **The intake adapter crashes mid-batch.** The email adapter pulled 50 messages from IMAP and crashed after processing 30. On restart, how do you avoid reprocessing the first 30 and losing the last 20?

3. **Two agents claim the same queued ticket.** Both click "Next Ticket" within the same second. The query returns the same ticket to both. How do you make sure only one gets it?

4. **SLA business hours across timezones.** Customer in Tokyo, team in San Francisco. Whose business hours? What if the contract says "follow the sun support"?

5. **Reopen after long silence.** A customer replies to a ticket closed 6 months ago. What does the system do? Where does the conversation live?

6. **Threading vs new ticket.** A customer with an open billing ticket emails again about a totally different issue (broken login). Do you append or open a new one? How does the system decide?

7. **Agent goes on vacation mid-ticket.** Bob has 47 open tickets and starts a 2-week vacation. How do you hand them off?

8. **Spam at the front door.** Your inbound email address gets 10,000 spam messages per day. How do you avoid filling the ticket DB with garbage?

9. **KB suggestions at intake.** When a customer fills the web form, before submitting, you want to suggest 3 KB articles that might solve their issue. How do you do this without slowing the form?

10. **"Average time to resolve" is wrong.** Management complains the dashboard says 2 hours but tickets take days. What's going wrong, and how do you compute it correctly?

## Related problems

- **[Approval Management (011)](../011-approval-management/question.md)**. Shares the workflow engine plus state machine plus role routing plus escalation patterns. If you have built one, the other is mostly a relabeling.
- **[Notification System (010)](../010-notification-system/question.md)**. Every state change (assigned, replied, breached, resolved) fans out to email, Slack, push. The fan-out, retry, and quiet-hours machinery from that problem is what consumes ticket events.
- **[Comment System (015)](../015-comment-system/question.md)**. A ticket's `ticket_messages` are the same shape as comments on a post: threaded, paginated, with attachments. Reuse the same storage and indexing patterns.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md)**. Agent dashboards and customer portals load tickets thousands of times per day. Cache tiering, read replicas, and denormalized views from that problem apply directly here.
