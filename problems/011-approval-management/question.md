---
id: 11
title: Design an Approval Management Service
category: Workflow
topics: [state machine, role routing, delegation, escalation, audit trail]
difficulty: Easy
solution: solution.md
---

## Scene

You sit down. The interviewer pushes a doc across the table.

> *"Every company has approval flows. Purchase orders need a manager and finance to sign off. Leave requests go to your boss. Expense reports go to your boss, then to finance, then to your boss again if finance pushes back. Code reviews go to two engineers. We want one service that handles all of these. Design it."*

They pause. "Start small. How would you build it for a 50-person company first. Then we'll grow it."

This question looks deceptively simple. Candidates who jump straight to "we'll have an `approvals` table with a `status` column" miss every interesting thing about it. The interesting bits are: how do you let teams *define* their own workflows without redeploying, how do you survive an approver leaving the company mid-request, how do you handle the moment when the same person is the requester and the approver, and how do you keep an audit trail that satisfies a compliance auditor years later.

We are going to walk this from a 50-person startup to a 100,000-person enterprise. At every step, something breaks. Naming what breaks before adding the fix is the actual exercise.

## Step 1: clarify before you design

Take 7 minutes. This problem has more dimensions than most. Aim for 8 to 10 questions.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Single workflow type or many?** "Is this just leave requests, or does the same service handle leave + expense + PO + code review?" The answer is almost always "many, please design for that." This is the single decision that drives 80% of the architecture. One workflow per service is easy; an *engine* that runs many workflow types defined by config is the actual problem.
2. **Who defines the workflows?** "Do engineers ship YAML for each workflow, or do admin users build them in a UI?" Determines whether you need a workflow definition store + UI + versioning, or whether workflows are baked into deploys.
3. **Step types.** "What can a step *do*? Just 'wait for human approval'? What about 'run a script', 'call an external API', 'wait 24 hours', 'auto-approve if amount < $100'?" Tells you whether your engine is a state machine or a full workflow runtime.
4. **Parallel vs serial.** "Can step 2 wait on three approvers in parallel and require all of them? Any of them? Two-of-three?" Parallel approval (with quorum) is a common request that breaks a naive serial-only model.
5. **Delegation and out-of-office.** "If Alice is on vacation, does her approval auto-route to Bob? Does Alice configure this herself, or is it organizational policy?" Delegation chains are a real production source of bugs.
6. **Escalation.** "If an approver does not act in 48 hours, what happens? Auto-approve? Escalate to their manager? Page them?" This is the SLA layer on top of the engine.
7. **Cancellation and edit.** "Can the requester cancel a request mid-flight? Can they edit the amount of a PO that has been partially approved?" Editing usually means resetting the workflow to the start. Cancellation needs to release any holds (e.g., budget reservation).
8. **Audit trail.** "How long do we retain decisions? Who saw the request? Is the audit log immutable? Are we under SOX, HIPAA, GDPR?" Compliance changes the storage and retention model dramatically.
9. **Notification scope.** "When a request is created, who gets notified? Email, Slack, in-app? Is the notification system part of this design?" Usually notification is a separate service; this design *emits events* and the notification service consumes them.
10. **Org structure.** "Does the service know about org hierarchy (manager-of, team-of, department-of)? Or does it call out to an HR system?" The router needs this info; whether it stores it or queries an HRIS changes the latency budget.

A strong candidate also names what is *not* in scope: payment execution, budget enforcement, contract signing. These are downstream of approval; the approval engine emits "approved" and someone else acts on it.

</details>

## Step 2: capacity estimates

The interviewer gives you the small-company answer:

- 50 employees today
- Average 5 approval requests per employee per week
- Most workflows have 1-3 approvers
- Each approver typically responds within 1 day

Compute:

1. Requests per day, per second
2. Average lifecycle of a request (created → fully approved)
3. Decisions per day (one decision = one approver acting)
4. Active requests at any given time (request created but not yet finalized)

Now do the same for a 100k-employee enterprise.

<details>
<summary><b>Reveal: the math at both scales</b></summary>

**50-person startup:**

- 50 × 5 = 250 requests per week = ~36 per day. About 1 every 40 minutes during business hours. *Trivial.*
- Avg lifecycle ~ 1 to 2 days.
- Decisions per day: 36 requests × ~2 approvers = 72 decisions per day. ~5 per business hour.
- Active requests at any moment: ~50 to 70 in flight.

You could literally build this with a Google Sheet and a cron job. Instead we build the right shape, because we are scaling it.

**100k-person enterprise:**

- 100k × 5 = 500k requests per week ≈ 71k per day ≈ ~1 per second sustained. Peak hour roughly 3x: ~3 per second.
- Lifecycle: 1 to 5 days depending on workflow type. Compliance workflows can last weeks.
- Decisions per day: 71k × ~3 approvers (enterprise workflows are longer) = ~210k decisions per day ≈ ~3 per second sustained.
- Active requests at any moment: 71k/day × ~3 day avg lifecycle = ~200k in flight at any moment.

**Key insights from the math:**

- The volume is *small* compared to consumer-scale problems. A single Postgres handles all writes fine. The interesting scale problem is not write throughput; it is **organizational complexity**: 5000 different workflow definitions, 50k different approver roles, hundreds of integrations.
- "Active requests in flight" is the meaningful metric. At 200k in flight you need indexes that let "show me my pending approvals" return in 50ms for every employee.
- Reads massively dominate writes. Every employee opens their dashboard ~10x per day to check status. That is 1M dashboard loads per day even at the enterprise scale. Caching the read path matters more than optimizing writes.

</details>

## Step 3: the workflow definition language

Before you draw any architecture, you have to decide *how a workflow is described*. This is the data structure your engine reads to know "what step comes next." If you get this wrong, every later feature is a hack.

Here is an intentionally incomplete workflow definition for a leave request. Fill in the `[ ? ]` placeholders. The placeholders mark concepts the language must support: parallel approval, conditional branching, escalation timer, delegation, and the auto-approve shortcut.

```yaml
workflow: leave_request
version: 3

inputs:
  - employee_id: string
  - start_date: date
  - end_date: date
  - days: int     # derived from start/end

steps:

  - id: auto_approve_short
    when: [ ? ]                          # condition: < 3 days
    action: approve
    note: "Short leaves auto-approved by policy."

  - id: manager_approval
    after: auto_approve_short
    type: approval
    approver: "{{ employee.manager }}"
    timeout: [ ? ]                        # escalate after N hours
    on_timeout: [ ? ]                     # what to do (escalate? auto-reject?)
    on_delegation: [ ? ]                  # if approver has set OOO, route to whom?

  - id: hr_and_grandboss
    after: manager_approval
    when: days > 14                       # only for long leaves
    type: [ ? ]                           # parallel: both must approve
    branches:
      - approver: "{{ employee.manager.manager }}"
      - approver: "hr-leave-admin"
    quorum: [ ? ]                         # all? any? 2-of-N?

  - id: finalize
    after: hr_and_grandboss
    type: terminal
    state: approved
```

<details>
<summary><b>Reveal: the completed workflow language</b></summary>

```yaml
workflow: leave_request
version: 3

inputs:
  - employee_id: string
  - start_date: date
  - end_date: date
  - days: int

steps:

  - id: auto_approve_short
    when: days < 3
    action: approve
    note: "Short leaves auto-approved by policy."

  - id: manager_approval
    after: auto_approve_short
    type: approval
    approver: "{{ employee.manager }}"
    timeout: 48h
    on_timeout: escalate                  # send to grandparent (manager's manager)
    on_delegation: follow                 # if Alice has OOO set to Bob, route to Bob

  - id: hr_and_grandboss
    after: manager_approval
    when: days > 14
    type: parallel
    branches:
      - approver: "{{ employee.manager.manager }}"
      - approver: "hr-leave-admin"        # role, not a user
    quorum: all                           # both must approve

  - id: finalize
    after: hr_and_grandboss
    type: terminal
    state: approved
```

**The language has to support five things, and each maps to a real-world requirement:**

1. **Conditional steps (`when:`).** Auto-approve short leaves. Skip finance review for purchases under $100. Required for any non-trivial workflow.
2. **Timeouts (`timeout:` + `on_timeout:`).** Real humans miss things. Without timeouts your approval queue grows forever.
3. **Delegation (`on_delegation:`).** Alice goes on vacation. Bob is her delegate. The engine must follow the delegation chain (Alice → Bob → Carol if Bob is also out) without infinite-looping.
4. **Parallel steps with quorum (`type: parallel`, `quorum: all|any|N-of-M`).** Code reviews need 2 of 3 senior approvals. Document signoffs need *all* department heads. Hard to retrofit if your model is serial-only.
5. **Roles, not just users (`approver: "hr-leave-admin"`).** When Alice the HR admin leaves the company, the workflow should not break. Roles get resolved to users at runtime by the org service.

**Why YAML/JSON not code:** Workflow authors are often non-engineers (HR admins, finance leads). They edit through a UI that produces this YAML. The engine reads the YAML and executes. Storing workflows as data, not code, means you can deploy new workflows without releasing the service.

**Version field is non-optional.** Workflow `leave_request` version 3 was the spec when this request was created. The request executes against version 3 *forever*, even if version 4 ships tomorrow. Otherwise running requests would change shape mid-flight, which breaks audit and surprises users.

</details>

## Step 4: sketch the system architecture

You now know the data structure. Build the service around it. Fill in the eight `[ ? ]` boxes. The boxes mark the major roles: where requests come in, where workflow definitions live, the engine that executes them, where decisions are persisted, who knows the org chart, where notifications go, where the audit trail lives, and what the dashboard reads from.

```
                Client (web, mobile, Slack bot)
                              │
                              ▼
                    ┌──────────────────┐
                    │   [ ? ]          │  (auth, rate limit, request shaping)
                    └────────┬─────────┘
                             │
        create request       │       read dashboard
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
        ┌─────────────┐            ┌─────────────┐
        │  [ ? ]      │            │  [ ? ]      │  (fast reads, "my pending")
        │ (executes)  │            │             │
        └──┬───┬──┬───┘            └─────────────┘
           │   │  │
           │   │  └─────────► [ ? ]   (resolves roles → users,
           │   │                       handles OOO / delegation)
           │   │
           │   └─────────► [ ? ]      (where workflow YAML lives,
           │                          versioned)
           │
           ▼
        ┌─────────────┐
        │   [ ? ]     │   (single source of truth for
        │             │    requests, decisions, state)
        └──────┬──────┘
               │
               ▼
        ┌─────────────┐
        │   [ ? ]     │   (immutable, append-only,
        │             │    queryable years later)
        └─────────────┘

        Events out:
            ┌─────────────┐
            │   [ ? ]     │  (downstream: notifications,
            │             │   integrations, analytics)
            └─────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                Client (web, mobile, Slack bot)
                              │
                              ▼
                    ┌──────────────────┐
                    │   API Gateway    │  (auth, rate limit, idempotency keys)
                    └────────┬─────────┘
                             │
        create request       │       read dashboard
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
        ┌─────────────────┐         ┌──────────────────┐
        │  Workflow       │         │  Read Service    │  (cache + read replica;
        │  Engine         │         │  "my pending     │   serves dashboards)
        │  (executes      │         │   approvals"     │
        │   state         │         │   in <50ms)      │
        │   transitions) │         └────────┬─────────┘
        └─┬──┬──┬────────┘                  │
          │  │  │                            │
          │  │  └──────► ┌──────────────────┐│
          │  │           │  Org Service /   ││
          │  │           │  Role Resolver   ││
          │  │           │  (manager-of,    ││
          │  │           │   role members,  ││
          │  │           │   OOO delegation)││
          │  │           └──────────────────┘│
          │  │                                │
          │  └──────► ┌──────────────────┐    │
          │           │  Workflow        │    │
          │           │  Definition      │    │
          │           │  Store           │    │
          │           │  (YAML, versioned│    │
          │           │   per workflow)  │    │
          │           └──────────────────┘    │
          │                                   │
          ▼                                   │
        ┌──────────────────┐                  │
        │  Request DB      │◄─────────────────┘
        │  (Postgres)      │
        │  Tables:         │
        │   - requests     │
        │   - tasks (per   │
        │     pending      │
        │     approval)    │
        │   - decisions    │
        └────────┬─────────┘
                 │
                 │ CDC / outbox
                 ▼
        ┌──────────────────┐
        │  Audit Log       │  (append-only; S3 + Athena
        │  (immutable)     │   or ClickHouse; 7-year
        │                  │   retention default)
        └──────────────────┘

        Engine also publishes events:
            ┌──────────────────┐
            │  Kafka topic     │  approval.request.created
            │  approval.*      │  approval.task.assigned
            │                  │  approval.decision.recorded
            │                  │  approval.request.finalized
            └──────────────────┘
                Consumers:
                  - Notification Service (email, Slack)
                  - Integration adapters (Workday, NetSuite, Jira)
                  - Analytics (which workflows are slowest)
```

Component responsibilities:

- **API Gateway.** First line. Auth (which user is this), rate limit (no employee creates 10,000 requests in a minute), idempotency key handling (the mobile client retried submit).
- **Workflow Engine.** The brain. Reads the request's current state, decides what step is next, resolves the approver (calls Org Service), creates a *task* in the DB, emits an event. When a decision comes in, validates it, advances state, repeats.
- **Workflow Definition Store.** Where YAML lives. Versioned. New versions are draft → review → published. Published versions are immutable.
- **Org Service / Role Resolver.** Knows "Alice's manager is Bob, Bob's manager is Carol. The role 'finance-approver' currently has these 7 users. Alice has set OOO from May 5 to May 10 delegating to Dave." Usually a thin wrapper over Workday or your HRIS.
- **Request DB (Postgres).** Source of truth. Three core tables: `requests`, `tasks` (one row per pending approver action), `decisions` (immutable record of what each approver decided).
- **Read Service.** Optimized for "show me my dashboard." Reads from a denormalized cache populated by listening to engine events. Avoids hammering the primary DB for every dashboard load.
- **Audit Log.** Immutable. Every state transition, every decision, every escalation, every delegation lookup. Long retention (often 7 years for SOX, longer for healthcare). Cheap storage (S3) with a query layer (Athena or ClickHouse).
- **Event stream (Kafka).** The engine is the producer. Consumers handle the side-effect world: send notifications, sync to downstream systems, feed analytics.

</details>

## Step 5: role resolution and delegation

The system says "approver: `{{ employee.manager }}`". The engine has to turn that into a real person before assigning the task. And the answer might be different depending on who is out of office today. Walk through how role resolution works.

Sketch the lookup logic. What if Alice's manager Bob has set OOO and delegated to Carol, but Carol is on the same vacation and her delegate is Dave?

<details>
<summary><b>Reveal: role resolution algorithm</b></summary>

```python
def resolve_approver(spec, requester, when):
    """
    spec: a workflow approver string like "{{ employee.manager }}" or "finance-approver"
    requester: the employee who created the request
    when: timestamp of when we are evaluating (always = "now" for live requests,
          but stamped for audit replay)
    """
    # Step 1: resolve the spec to a target (user, manager-chain, or role)
    target = render_template(spec, {"employee": requester})

    # Step 2: if target is a role, pick the first available role member
    if is_role(target):
        members = org.role_members(target, at=when)
        if not members:
            raise NoApproverFound(f"role {target} has no members")
        candidate = pick_round_robin(members)
    else:
        candidate = target

    # Step 3: follow OOO/delegation chain with cycle and depth guard
    return follow_delegation(candidate, when, depth=0, visited=set())

def follow_delegation(user, when, depth, visited):
    if depth > 5:                       # never follow more than 5 hops
        raise DelegationTooDeep(user)
    if user.id in visited:              # break cycles (Alice delegates to Bob;
        raise DelegationCycle(visited)  # Bob delegates back to Alice)
    visited.add(user.id)

    if not user.exists:                 # left the company
        # Fall back to user's last manager, or workflow's "fallback" if defined.
        return fallback_for_departed(user)

    ooo = org.get_active_ooo(user, at=when)
    if ooo is None:
        return user                     # found a live approver

    if ooo.delegate is None:
        return user                     # OOO but no delegate; assign anyway
                                        # (will likely time out and escalate)

    return follow_delegation(ooo.delegate, when, depth + 1, visited)
```

Four real-world cases this handles:

1. **Alice's manager is Bob. Bob is in.** Resolves to Bob immediately.
2. **Bob is OOO, delegated to Carol. Carol is in.** Resolves to Carol. Follow once.
3. **Bob → Carol → Dave (Carol also OOO).** Resolves to Dave. Follow twice.
4. **Bob → Carol → Bob (Carol set her delegate to Bob by mistake).** Cycle detected. Raise. Fallback to assigning Bob anyway with a warning.

**Why the `when` parameter.** During audit replay, you must reconstruct who *would have been* the approver at the time the request was created, not who would be approver now. People change jobs, OOO settings shift, delegations expire.

**Pick strategy for role members.** Round-robin is the default for fairness. For high-priority requests you might use "least-loaded" (look at each member's current task count). Avoid "random" because it makes the audit story confusing.

**Departed users.** Someone gets fired or leaves. Their pending tasks are orphaned. A nightly sweep reassigns to fallback (their last manager). High-volume departures (layoffs) trigger a special bulk-reassign tool.

</details>

## Step 6: the audit trail (and why it must be immutable)

Five years from now, an auditor asks: "Show me every approval decision on purchase orders over $50,000 in Q3 2024." How does your system answer that, given that the people who made those decisions may have left, the workflow definitions have changed, and the approvers' roles may have been reorganized?

Sketch the audit log schema. What fields are non-negotiable, what is optional, and what must *never* be overwritten?

<details>
<summary><b>Reveal: audit log design</b></summary>

```sql
CREATE TABLE audit_log (
    event_id          UUID PRIMARY KEY,
    occurred_at       TIMESTAMPTZ NOT NULL,        -- wall clock
    request_id        UUID NOT NULL,                -- foreign reference, not FK
    workflow_id       TEXT NOT NULL,                -- "leave_request"
    workflow_version  INT NOT NULL,                 -- 3
    event_type        TEXT NOT NULL,                -- request.created, decision.recorded, etc.
    actor             JSONB,                        -- {"user_id": "...", "role": "...", "delegated_from": "..."}
    payload           JSONB NOT NULL,               -- event-specific
    snapshot          JSONB                         -- the request state at this moment
);

CREATE INDEX idx_audit_request ON audit_log (request_id, occurred_at);
CREATE INDEX idx_audit_workflow_time ON audit_log (workflow_id, occurred_at);
CREATE INDEX idx_audit_actor ON audit_log USING gin (actor);
```

**Non-negotiable rules:**

1. **Append-only.** No UPDATE, no DELETE, ever. The DB user that writes audit has INSERT-only privileges.
2. **Includes a snapshot.** Each event carries the request state at that moment. Lets you replay the request's life by walking events.
3. **Includes the workflow version.** If `leave_request` is on v5 today and this request ran against v3, the audit must show v3.
4. **Captures who and on whose behalf.** If Carol approved as Bob's delegate, both names are recorded.
5. **Hashed chain (optional, for high-compliance).** Each event has a `prev_hash` and `hash`. Tampering with one event invalidates the chain. Healthcare and finance industries sometimes require this.

**Retention:**

- Default 7 years (SOX retention horizon for US public companies).
- Stored in two tiers: last 90 days in Postgres for fast queries, older in S3 (Parquet) with Athena for ad-hoc auditor queries.
- Background job moves rows from hot to cold tier nightly.

**What audit answers:**

- "Who approved request X?" → query `request_id`.
- "Show all decisions by user Alice in 2024." → query `actor`.
- "All purchase orders over $50k approved by anyone other than the requester's direct manager." → join `audit` to `requests` snapshot data.
- "Replay request X to see exactly what the engine did at each step." → walk `event_id`s for that request in time order.

**Why not just rely on the primary DB tables.** The `requests` and `tasks` tables get updated when state changes. The current state of a request five years from now reflects the latest version, not the audit trail. Auditors need the *history*, not the current state.

</details>

## Step 7: the four real-world variants

The same engine runs all of these. Each one stresses a different part of the design. For each variant below, identify which feature of the engine is critical. Then peek at the answer.

**A. Purchase Order**

> Engineer submits PO for $12,000 for new servers. Workflow: manager approves → if amount > $5k, finance approves → if amount > $25k, CFO approves → procurement team executes.

**B. Leave Request**

> Same as the YAML we built. 1-2 days auto-approve, 3-14 days manager, 14+ adds HR + grandboss in parallel.

**C. Expense Report**

> Submit receipts. Manager reviews. If approved, goes to finance for reimbursement. Finance may push back ("Receipt is missing, please re-upload") which sends it *back* to the requester, then back to manager, then back to finance.

**D. Code Review**

> Pull request needs 2 approvals from senior engineers. Author cannot approve their own. CI must pass. If any approver requests changes, the request resets (loses approvals) when new commits are pushed.

<details>
<summary><b>Reveal: what each variant stresses</b></summary>

**A. Purchase Order: conditional branching.**

The workflow has three potential approval steps but only the ones that match the amount are executed. The engine evaluates the `when:` clause at each step. The PO at $12k goes through manager + finance but skips CFO. A $30k PO hits all three.

> Common bug: the workflow author writes `when: amount > 5000` in finance and `when: amount > 25000` in CFO. A $25,001 PO goes to *both* finance and CFO. Probably fine. A $5,001 PO goes to finance, then to CFO because $5001 > 25000 is false, so CFO is skipped. Logic correct. But if the workflow author writes `when: amount > 5000` in CFO too (copy-paste error), a $6000 PO needlessly goes to the CFO. Lint your workflow definitions.

**B. Leave Request: parallel approval with quorum.**

A 21-day leave goes to manager (serial), then *parallel* to HR-admin (role) AND grandboss (template-resolved). Both must approve before finalize. The engine creates two tasks at once, waits for both, then advances. If HR-admin approves but grandboss rejects, the whole request rejects.

> Common bug: someone writes `quorum: any` instead of `all`. Now one approver can rubber-stamp the request before the other has even seen it. Default `quorum` should be `all`; require explicit `any` in the YAML.

**C. Expense Report: backward transitions ("send back").**

Finance pushes back. The request goes *back* to the requester (state: returned_to_requester). The requester edits and resubmits. Now what? Two options:

- **Reset to start:** entire workflow restarts. Manager has to re-approve from scratch. Safe; loses prior context.
- **Resume from manager:** request re-enters at the manager step. Manager sees "Carol resubmitted with new receipts." Faster; requires the engine to support backward transitions explicitly.

Most real systems do **resume from manager** with a banner showing what changed. The engine has to support `on_action: return_to_step(<step_id>)` transitions.

> Common bug: when the request goes back to the requester, finance's pending task is left orphan. The engine must close that task with reason `request_returned` so it stops showing in finance's dashboard.

**D. Code Review: invalidation on input change.**

If two reviewers approve, then the author pushes new commits, do the approvals carry over? GitHub says no by default (commit invalidates approvals). The engine needs an `on_input_change: invalidate_approvals` hook.

> Common bug: "self-approval is not allowed" is a separate constraint from the workflow definition. The role-resolver for approvers must exclude the requester. Filter `approver != requester` in `resolve_approver`.

**The point of these four variants:** the same engine, same data model, same audit log, four totally different workflows. *That is the value of the design.* If you implemented a hardcoded "purchase_orders" service, you would have to build a separate service for every other workflow type.

</details>

## Follow-up questions (15)

Try answering each in 3 to 4 sentences before reading the solution.

1. **Self-approval prevention.** A user submits a PO on their own behalf and is in the finance approver group. How do you ensure they cannot approve their own request? Where in the system does this check live?

2. **Approver leaves the company while a request is in flight.** Their dashboard shows a pending task forever, but they cannot log in. What happens? How do you find these stale tasks?

3. **Delegation cycle.** Alice delegates to Bob. Bob delegates to Alice. The engine resolves an approver for Alice's request and infinite-loops. How do you prevent this?

4. **Workflow version migration.** You ship `leave_request` v4 which adds a new step. There are 800 in-flight requests on v3. What happens to them?

5. **Concurrent approval race.** Two managers are listed in `parallel` with `quorum: any`. Both click Approve at the same moment. Does the request advance twice? Get into a bad state? Detail the locking strategy.

6. **Auto-approval is broken.** Last night, finance's auto-approval rule (`when: amount < 100`) auto-approved 50,000 fraudulent micro-purchases. How would you detect this? How would you recover (reverse all those approvals)?

7. **Bulk import.** HR wants to import 5000 historical leave requests with their original timestamps and approver decisions. How do you load them through the engine while preserving the audit trail's accuracy?

8. **Performance: "my pending approvals" dashboard.** Carol is on 47 teams. She has ~120 pending tasks at any time. Her dashboard query takes 4 seconds. How do you fix it?

9. **Search across all approvals.** A finance auditor needs to find "all purchase orders mentioning vendor 'Acme Corp' approved in Q2." Your `requests` table has a JSON `inputs` column. Naive search is slow. What do you do?

10. **External integration: NetSuite sync.** When a PO is approved, your system must create a record in NetSuite. NetSuite returns 5xx errors 1% of the time. How do you guarantee the record is eventually created without ever duplicating?

11. **Notifications fan-out for a watcher list.** Alice's PO has 12 watchers (procurement, legal, IT, security, etc.). Every state change must notify all of them. How do you avoid notification storms when a request transitions through 8 states in 10 minutes?

12. **The "approve all" button.** Carol has 80 pending leave requests, all for the same week (school holiday). She wants to approve them all at once. Your UI builds the "approve all" button. What backend API does it call? What can go wrong?

13. **Visibility / privacy.** Salary-affecting decisions (raise requests) should not be visible to non-HR users, even in audit logs. How do you enforce this across the read service, the audit query interface, and the event stream?

14. **Workflow author makes an infinite loop.** Step A transitions to Step B which transitions back to Step A. You publish the workflow. The first request through it loops forever. How do you catch this *before* publication?

15. **Multi-region deployment.** Your company opens EU operations. EU employees' data must stay in EU. How does the engine handle a request where the requester is in EU but the approver is in US?

## Related problems

- **[Notification System (010)](../010-notification-system/question.md)**. Every approval event fires off notifications. The fan-out, retry, and quiet-hours machinery there is what consumes the approval engine's events. Designing notifications and approvals together is a common follow-up.
- **[Help Desk Ticketing (019)](../019-helpdesk-ticketing/question.md)**. A ticket has a similar lifecycle: created → assigned → in-progress → resolved → closed. The state machine, role-routing, and SLA timer patterns are the same. Help Desk adds intake-channel-routing on top.
- **[Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md)**. The audit log here is exactly a write-heavy append-only system. The patterns from that problem apply directly.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md)**. The "my pending approvals" dashboard is the read-heavy half of this design. Cache tiering and read-replica strategies from that problem apply.
