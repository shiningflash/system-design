---
id: 11
title: Design an Approval Management Service
category: Workflow
topics: [state machine, role routing, delegation, escalation, audit trail]
difficulty: Easy
solution: solution.md
---

## The scene

You sit down. The interviewer slides a sheet of paper across the table.

> *"Every company has approval flows. A purchase order needs your manager and finance to sign off. A leave request goes to your boss. An expense report goes to your boss, then to finance, then back to your boss if finance pushes back. Design one service that handles all of these."*

Then they smile and say: *"Start small. Build it for a 50-person startup first. Then we'll scale it up."*

It looks easy. It is not. The trap is the word "approval." It sounds like a single checkbox. The real problems are different: How do new teams add new workflow types without redeploying? What happens when an approver leaves the company while their task is sitting in the queue? How do you stop someone from approving their own request? And how do you keep a record an auditor can trust five years from now?

We will walk this from a 50-person startup to a 100,000-person enterprise. At every step we will name what breaks first, then add the smallest fix that solves it.

---

## Step 1: Ask the right questions

Before you draw anything, sit for five minutes. Write down questions you would ask the interviewer.

A good answer here is not "10 questions about every edge case." It is the small handful of questions that change the design if answered differently.

<details markdown="1">
<summary><b>Show: 8 questions that matter</b></summary>

1. **One workflow type, or many?** Just leave requests? Or also expenses, POs, code reviews? *(Almost always "many." This is the biggest decision in the design.)*
2. **Who writes the workflows?** Engineers in code? Or HR admins in a UI? *(Determines if you need a workflow definition store + UI + versioning.)*
3. **Can a step do anything besides "wait for human"?** Auto-approve under $100? Call an external API? Wait 24 hours? *(Tells you if you're building a state machine or a real workflow engine.)*
4. **Parallel or serial only?** Can a step wait for three managers in parallel and need all three? Or any two of three? *(Hard to retrofit if you build serial-only.)*
5. **What about vacation?** If Alice is out, does her approval auto-route to Bob? *(Delegation is the single biggest source of production bugs.)*
6. **What if nobody acts?** If the approver is silent for 48 hours, do we auto-approve? Escalate to their boss? Page someone? *(This is the SLA layer.)*
7. **Audit retention?** How long do we keep the records? SOX needs 7 years. Healthcare needs longer.
8. **What is NOT in scope?** Payment execution? Budget reservation? Contract signing? *(All downstream. The engine just emits "approved" and someone else acts.)*

A strong candidate also asks the meta question: *"Is the notification system part of this design, or separate?"* The right answer is separate. The engine emits events. The notification service consumes them.

</details>

---

## Step 2: How big is this thing?

Same problem, two scales. Do the math.

**At a 50-person startup:**

- 50 employees
- ~5 approval requests per person per week
- 1 to 3 approvers per request

**At a 100,000-person enterprise:**

- Same per-person rate

Compute four numbers for each scale: requests per day, requests per second, decisions per day, active requests in flight at any moment.

<details markdown="1">
<summary><b>Show: the math</b></summary>

**Startup (50 people):**

- 50 × 5 = 250 requests/week ≈ **36 requests/day**. One every 40 minutes during business hours. Tiny.
- Decisions: 36 × 2 approvers ≈ 72/day.
- Active at any moment: ~50 to 70 in flight.

You could literally run this on a Google Sheet and a cron job. But we're not going to, because we're scaling it up.

**Enterprise (100,000 people):**

- 100k × 5 = 500k/week ≈ **71,000 requests/day** ≈ **1 per second steady**, **3 per second at peak**.
- Decisions: 71k × 3 approvers (enterprise flows are longer) ≈ 210k/day ≈ **3 per second**.
- Active at any moment: 71k/day × ~3 day average lifecycle ≈ **200,000 in flight**.

**What the math is telling you:**

The numbers are small. Tiny, even, compared to Twitter or YouTube. A single Postgres can handle the writes.

The real problem at enterprise scale is not throughput. It is **organizational complexity**: 5,000 different workflow types, 50,000 different approver roles, hundreds of downstream integrations. The architecture exists to handle that, not the QPS.

Also: reads beat writes by 25 to 1. Every employee opens their dashboard ~10 times per day to check status. The read path matters more than the write path.

</details>

---

## Step 3: How do you describe a workflow?

This is the central decision. Before you draw any boxes, decide how a workflow is described.

Why does this matter? If you hardcode the leave-request workflow in Python, then expense reports need a new service. So does PO approval. So does code review. You end up with 20 services, each one a near-copy of the others.

The fix: workflows are **data**, not code. The engine reads the data and runs it.

Here is a workflow for leave requests, written in YAML. A few pieces are missing. See if you can guess what goes in the `[ ? ]` spots.

```yaml
workflow: leave_request
version: 3

inputs:
  - employee_id: string
  - start_date: date
  - end_date: date
  - days: int

steps:

  # Short leaves don't need anyone's approval.
  - id: auto_approve_short
    when: [ ? ]                       # condition: leaves under 3 days
    action: approve

  # Manager approves anything 3 days or longer.
  - id: manager_approval
    after: auto_approve_short
    type: approval
    approver: "{{ employee.manager }}"
    timeout: [ ? ]                    # how long before we escalate?
    on_timeout: [ ? ]                 # what happens then?
    on_delegation: [ ? ]              # if manager is out, who acts?

  # Long leaves also need HR + grandboss approval, in parallel.
  - id: hr_and_grandboss
    after: manager_approval
    when: days > 14
    type: [ ? ]                       # parallel or serial?
    branches:
      - approver: "{{ employee.manager.manager }}"
      - approver: "hr-leave-admin"
    quorum: [ ? ]                     # all? any one? two of three?

  - id: finalize
    after: hr_and_grandboss
    type: terminal
    state: approved
```

<details markdown="1">
<summary><b>Show: the filled-in workflow</b></summary>

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

  - id: manager_approval
    after: auto_approve_short
    type: approval
    approver: "{{ employee.manager }}"
    timeout: 48h
    on_timeout: escalate              # send to manager's manager
    on_delegation: follow             # if Alice is OOO with delegate Bob, route to Bob

  - id: hr_and_grandboss
    after: manager_approval
    when: days > 14
    type: parallel
    branches:
      - approver: "{{ employee.manager.manager }}"
      - approver: "hr-leave-admin"
    quorum: all                       # both must approve

  - id: finalize
    after: hr_and_grandboss
    type: terminal
    state: approved
```

The language has to support five things, and each one maps to a real-world problem:

1. **Conditional steps (`when:`).** Auto-approve short leaves. Skip finance for tiny POs. Every real workflow needs this.
2. **Timeouts (`timeout:` + `on_timeout:`).** Humans miss things. Without timeouts, your queue grows forever.
3. **Delegation (`on_delegation:`).** Vacation happens. The engine has to follow the chain (Alice → Bob → Carol if Bob is also out) without looping forever.
4. **Parallel with quorum.** Code review needs 2 of 3 senior approvals. Some signoffs need *all* department heads. Cannot be retrofitted onto a serial-only engine.
5. **Roles, not just users.** If you write `approver: "hr-leave-admin"` and that role's current member leaves the company, the workflow still works. The engine resolves the role to a real person at runtime.

**Why YAML and not Python.** Most workflow authors are HR admins, finance leads, ops people. Not engineers. They edit through a UI that produces YAML. The engine reads YAML and runs it. Adding a new workflow takes 5 minutes and zero deploys.

**Why the `version` field is sacred.** When this request was created, the workflow was on v3. Even if v4 ships tomorrow, this request keeps running on v3. Forever. Otherwise running requests change shape mid-flight, and audit becomes impossible.

</details>

---

## Step 4: Draw the system

You know how a workflow is described. Now draw the boxes that run it.

Try to fill in the missing pieces below. Eight boxes are missing. Think about: where do requests come in, where do workflow YAMLs live, what runs the workflow, where do decisions go, who knows the org chart, who feeds notifications, where does audit live, and what does the dashboard read from.

```
            Client (web, mobile, Slack bot)
                       │
                       ▼
              ┌─────────────────┐
              │   [ ? ]         │  auth, rate limit, idempotency
              └────┬───────┬────┘
                   │       │
      create       │       │      read dashboard
                   │       │
                   ▼       ▼
       ┌─────────────┐  ┌─────────────┐
       │   [ ? ]     │  │   [ ? ]     │  fast reads for "my pending"
       │  runs the   │  │             │
       │  workflow   │  └─────────────┘
       └─┬─┬─┬───────┘
         │ │ │
         │ │ └────► [ ? ]   resolves roles, handles vacation
         │ │
         │ └─────► [ ? ]    stores workflow YAML, versioned
         │
         ▼
       ┌─────────────┐
       │   [ ? ]     │  source of truth: requests, tasks, decisions
       └──────┬──────┘
              │
              ▼
       ┌─────────────┐
       │   [ ? ]     │  append-only, queryable for years
       └─────────────┘

       Events stream out:
       ┌─────────────┐
       │   [ ? ]     │  feeds notifications, integrations, analytics
       └─────────────┘
```

<details markdown="1">
<summary><b>Show: the full architecture</b></summary>

```
            Client (web, mobile, Slack bot)
                       │
                       ▼
              ┌─────────────────┐
              │  API Gateway    │   auth, rate limit, idempotency
              └────┬───────┬────┘
                   │       │
      create       │       │      read dashboard
                   │       │
                   ▼       ▼
       ┌─────────────┐  ┌──────────────────┐
       │  Workflow   │  │  Read Service    │   denormalized cache,
       │  Engine     │  │  ("my pending"   │   serves dashboards
       │             │  │   in <50ms)      │   in <50ms
       └─┬─┬─┬───────┘  └────────┬─────────┘
         │ │ │                    │
         │ │ └─────► Org Service  │  manager-of, role members,
         │ │        (HRIS proxy)  │  OOO + delegation
         │ │
         │ └────► Workflow Defs   │  YAML, versioned, immutable
         │
         ▼                        │
       ┌──────────────────────────┘
       │  Postgres
       │   requests
       │   tasks (one per pending action)
       │   decisions (immutable)
       │   audit_log (recent 90 days)
       └──────────┬───────────────
                  │
                  │  CDC / outbox
                  ▼
       ┌──────────────────┐
       │  Kafka topics    │   approval.request.created
       │  approval.*      │   approval.task.assigned
       │                  │   approval.decision.recorded
       │                  │   approval.request.finalized
       └──┬──────────┬────┘
          │          │
          ▼          ▼
     Notification   Integrations    Audit cold tier
     Service        (NetSuite,      (S3 + Athena,
     (email, Slack) Workday)        7-year retention)
```

What each piece does, in one line:

- **API Gateway.** Auth (who is this), rate limit (no bots), idempotency (mobile retried submit).
- **Workflow Engine.** The brain. Reads the current state, decides what's next, assigns the next task.
- **Workflow Definition Store.** Where YAML lives. Versioned. New versions never overwrite old ones.
- **Org Service.** Knows Alice's manager is Bob, who is on vacation, with Carol as delegate. Usually a thin layer over Workday or BambooHR.
- **Postgres.** Source of truth. Three small tables for the live state, one append-only table for audit.
- **Read Service.** Optimized for "show me my dashboard." Reads from Redis cache populated by engine events. Avoids hammering the primary DB.
- **Audit Log.** Append-only. Recent 90 days in Postgres, older in S3. 7-year retention.
- **Kafka.** Decouples the engine from the side-effect world (notifications, downstream syncs, analytics).

</details>

---

## Step 5: The vacation problem (delegation)

The workflow YAML says `approver: "{{ employee.manager }}"`. That's a template. The engine has to turn it into a real person before it can assign a task.

That sounds simple. Let's see why it isn't.

**Alice submits a leave request. Her manager is Bob. But:**

- Bob is on vacation. He set Carol as his delegate.
- Carol is on the same vacation. She set Dave as her delegate.
- Dave is in the office.

Who gets the task?

<details markdown="1">
<summary><b>Show: how to resolve the approver safely</b></summary>

The answer is Dave. But getting there safely takes a small algorithm.

```python
def resolve_approver(spec, requester, when):
    # Step 1: turn the template into a target.
    target = render_template(spec, {"employee": requester})

    # Step 2: if it's a role like "hr-admin", pick an actual person.
    if is_role(target):
        members = org.role_members(target, at=when)
        if not members:
            raise NoApproverFound(target)
        target = pick_round_robin(members)

    # Step 3: follow the vacation chain, with safety guards.
    return follow_delegation(target, when, depth=0, visited=set())


def follow_delegation(user, when, depth, visited):
    if depth > 5:                           # don't follow more than 5 hops
        raise DelegationTooDeep(user)
    if user.id in visited:                  # Alice → Bob → Alice = cycle
        raise DelegationCycle(visited)
    visited.add(user.id)

    if not user.exists:                     # user left the company
        return fallback_for_departed(user)

    ooo = org.get_active_ooo(user, at=when)
    if ooo is None or ooo.delegate is None:
        return user                         # found a live approver
    return follow_delegation(ooo.delegate, when, depth + 1, visited)
```

Four real-life cases this handles cleanly:

| Case | Result |
|------|--------|
| Bob is in | Returns Bob |
| Bob OOO → Carol (Carol in) | Returns Carol |
| Bob OOO → Carol OOO → Dave (Dave in) | Returns Dave |
| Bob OOO → Carol OOO → Bob (cycle) | Raises error, assigns to Bob anyway with a warning |

The `when` parameter looks redundant but it is the key to audit replay. To rebuild who *would have been* the approver back when this request was created, you need a point-in-time view of the org chart. People change jobs. Delegations expire. Roles get reassigned.

One last detail: when the engine assigns to Dave (after a Bob → Carol → Dave chain), it stores the chain on the task as `assigned_via`. Dave's dashboard then shows *"You are approving on behalf of Bob, via Carol."* The audit log records the full chain. Five years later, an auditor can see exactly how the approval reached Dave.

</details>

---

## Step 6: The audit trail (why it must be unbreakable)

Five years from now, an auditor will ask you: *"Show me every approval decision on purchase orders over $50,000 in Q3 2024."*

By then:
- The people who made those decisions may have left
- The workflow definitions have changed many times
- The approvers' roles have been reorganized

How does your system answer the question?

<details markdown="1">
<summary><b>Show: how to design the audit log</b></summary>

```sql
CREATE TABLE audit_log (
    event_id          UUID PRIMARY KEY,
    occurred_at       TIMESTAMPTZ NOT NULL,
    request_id        UUID NOT NULL,           -- referenced, not foreign key
    workflow_id       TEXT NOT NULL,
    workflow_version  INT NOT NULL,            -- pinned forever
    event_type        TEXT NOT NULL,           -- request.created, decision.recorded, etc.
    actor             JSONB,                   -- {user, role, delegated_from}
    payload           JSONB NOT NULL,
    snapshot          JSONB                    -- request state at this moment
);

CREATE INDEX idx_audit_request   ON audit_log (request_id, occurred_at);
CREATE INDEX idx_audit_workflow  ON audit_log (workflow_id, occurred_at);
CREATE INDEX idx_audit_actor     ON audit_log USING gin (actor);
```

Five rules you cannot break:

1. **Append-only.** No UPDATE. No DELETE. Ever. The DB user that writes audit has INSERT privilege only.
2. **Snapshot in every row.** The request's state at that moment, frozen. Lets you replay the request's life by walking events in order.
3. **Workflow version pinned.** If `leave_request` is on v5 today and this request ran against v3, the audit shows v3.
4. **Who and on whose behalf.** If Carol approved as Bob's delegate, both names are recorded.
5. **Hash chain (for high-compliance industries).** Each event has `prev_hash` and `hash`. Tampering with one event invalidates every event after it. Healthcare and finance often require this.

**How long?** Default 7 years (SOX for US public companies). Two tiers of storage:

- Last 90 days in Postgres for fast queries.
- Older in S3 (Parquet format), queried via Athena when an auditor calls.
- A nightly job moves rows from hot to cold.

**Why a separate audit table at all?** The `requests` table holds *current* state, which gets overwritten on every transition. The audit holds *history*. Auditors need history, not current state. The two are different products in the same database.

</details>

---

## Step 7: Four workflows, one engine

Here are four real workflows. The same engine, the same data model, the same audit log runs all of them. Each one stresses a different feature.

For each, guess which engine feature it depends on most. Then check.

**A. Purchase Order.** Engineer submits PO for $12,000 for servers. Workflow: manager approves. If amount > $5k, finance also approves. If amount > $25k, CFO approves too.

**B. Leave Request.** The one we already built. Short leaves auto-approve. Medium leaves need manager. Long leaves need HR + grandboss in parallel.

**C. Expense Report.** Submit receipts. Manager reviews. Finance reviews. Finance often says "receipt is missing, please resubmit." Request goes back to the requester, then forward again.

**D. Code Review.** Pull request needs 2 approvals from senior engineers. Author cannot approve their own. CI must pass. If anyone requests changes, prior approvals are wiped when new commits land.

<details markdown="1">
<summary><b>Show: what each one teaches</b></summary>

**A. Purchase Order: conditional branching.**

The workflow has three potential approval steps. Only the ones that match the amount run. The engine evaluates `when:` at each step. A $12k PO hits manager + finance, skips CFO. A $30k PO hits all three.

> *Common bug:* the workflow author writes `when: amount > 5000` in CFO too (copy-paste). A $6k PO needlessly goes to the CFO. Lint your workflow definitions.

**B. Leave Request: parallel approval with quorum.**

A 21-day leave goes to manager (one step), then in parallel to HR-admin AND grandboss. Both must approve. The engine creates two tasks at once and waits for both. If grandboss rejects, the whole request rejects, even if HR-admin already approved.

> *Common bug:* default quorum is `any`. Now one approver rubber-stamps before the other has even seen it. Default should be `all`, require explicit `any` in YAML.

**C. Expense Report: going backward.**

Finance rejects. The request goes back to the requester (state: `returned_to_requester`). Requester edits, resubmits. Now what?

Two options. Reset to start (manager re-approves from scratch). Or resume from manager (manager sees "Carol resubmitted with new receipts"). Most real systems do the second. The engine needs explicit `on_action: return_to_step(<step_id>)` support.

> *Common bug:* when the request rewinds, finance's pending task is left orphan. The engine must close that task with reason `request_returned` so it disappears from finance's dashboard.

**D. Code Review: invalidating on input change.**

Two reviewers approve. Author pushes new commits. Do the approvals stick? GitHub says no (commits invalidate). The engine needs `on_input_change: invalidate_approvals`.

> *Common bug:* "you cannot approve your own request" lives outside the workflow definition. The role-resolver must exclude the requester. Filter `approver != requester`.

**The big idea.** One engine. One data model. One audit log. Four totally different workflows. If you instead built a separate `purchase_orders` service, you would also need a separate `leave_requests` service, then `expense_reports`, then `code_reviews`, then twenty more. That is the trap.

</details>

---

## Follow-up questions

Try answering each in 2 or 3 sentences before opening the solution.

1. **Self-approval.** A user submits a PO and is also in the finance approver group. How do you stop them from approving their own request?

2. **Approver leaves the company.** Their dashboard still shows a pending task forever, but they cannot log in. What happens to that task?

3. **Delegation cycle.** Alice delegates to Bob. Bob delegates to Alice. The engine tries to resolve Alice's request and loops forever. How do you stop it?

4. **Workflow version migration.** You ship `leave_request` v4. There are 800 requests still in flight on v3. What happens to them?

5. **Two approvers click at the same moment.** They are both listed in parallel with `quorum: any`. Both hit Approve in the same millisecond. Does the request advance twice?

6. **Auto-approval rule was broken.** Last night, finance's `when: amount < 100` rule auto-approved 50,000 fraudulent micro-purchases. How do you detect this and recover?

7. **Bulk import.** HR wants to load 5,000 historical leave requests with their original timestamps and approvers. How do you preserve the audit trail's accuracy?

8. **Slow dashboard.** Carol has 120 pending tasks. Her dashboard takes 4 seconds to load. Why? How do you fix it?

9. **Search across all approvals.** Auditor needs *"all POs mentioning vendor Acme Corp approved in Q2."* Your `requests` table has a JSON `inputs` column. Naive search is slow. What do you do?

10. **NetSuite integration.** Every approved PO must create a record in NetSuite. NetSuite returns 5xx errors 1% of the time. How do you guarantee the record is created exactly once?

11. **Notification storm.** A request transitions through 8 states in 10 minutes. 12 watchers get 8 emails each. They unsubscribe. How do you fix it?

12. **The "approve all" button.** Carol has 80 pending leave requests for school holiday week. She wants to approve them all at once. What does the backend API look like, and what can go wrong?

13. **Privacy.** Salary-affecting decisions (raise requests) should not be visible to non-HR users, even in audit logs. How do you enforce this?

14. **Infinite-loop workflow.** A workflow author writes step A → step B → step A. You publish it. First request through it loops forever. How do you catch this before publication?

15. **Multi-region.** EU operations open. EU employee data must stay in EU. How does the engine handle a request where the requester is in EU but the approver is in US?

---

## Related problems

- **[Notification System (010)](../010-notification-system/question.md).** Every approval event fires off notifications. The fan-out, retry, and quiet-hours machinery there consumes the approval engine's events.
- **[Help Desk Ticketing (019)](../019-helpdesk-ticketing/question.md).** Same state-machine + role-routing + SLA-timer patterns. A ticket's lifecycle is structurally identical to an approval's.
- **[Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md).** The audit log here is exactly a write-heavy append-only system.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md).** The "my pending approvals" dashboard is the read-heavy half of this design.
