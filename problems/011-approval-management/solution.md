## Solution: Approval Management Service

### TL;DR

Approval is a workflow engine. You read a versioned definition (YAML or JSON), walk a state machine, and assign work to humans. The runtime itself is small. What makes the problem real is everything *around* it: turning role names like "the requester's manager" into actual people while handling out-of-office and delegation, surviving an approver who leaves the company, keeping an audit trail a regulator can trust five years later, and letting one engine run leave requests, purchase orders, code reviews, and twenty other shapes without becoming a hardcoded mess.

The data model fits on a napkin. Workflow definitions. Requests. Tasks (one per pending human action). Decisions (immutable record of what each human did). Audit log (every state transition, append-only). The interesting bit is how those tables relate at runtime, which we will draw in a moment.

Scale here is not a queries-per-second story. At 100k employees you see roughly one request per second. The pain comes from *organizational* complexity: 5000 different workflow variants, multi-region data residency, decades-long audit retention. The architecture starts as a single Postgres and grows in a few well-named stages, never preemptively.

### 1. Clarifying questions, recap

The most important question is the first one: *one workflow type or many?* A service for only leave requests is a CRUD app. A workflow engine that runs many workflow types defined by config is the actual system design. The second most important is *who authors workflows?* If it is HR admins through a UI, you need a definition store, versioning, validation, and a lint pass. If it is engineers shipping YAML in the repo, you can skip half of that but you also give up most of the value of building this as a service.

Everything else (delegation, escalation, audit retention) follows from those two answers.

### 2. Capacity, with the math

A 50-person startup generates around 36 requests per day. About one every 40 minutes. A single Postgres yawns. You could literally build this on a Google Sheet and a cron job.

A 100k-person enterprise generates around 71k requests per day. That is one per second sustained, three per second at peak. Dashboard reads dominate: 100k employees opening their inbox ten times a day works out to roughly 25 reads per second, 75 at peak.

Two numbers matter more than the throughput:

- **200k active requests in flight at any moment.** Every "show me my pending approvals" query has to find the right ones fast.
- **5000 workflow definitions** at a mature company. Hot ones (leave, expense) get hit thousands of times a day. Cold ones (annual security review) once a quarter.

Reads beat writes 25-to-1. Cache the read path or it gets ugly long before the writes ever do.

### 3. API

Two endpoints carry the whole product. *Create a request* and *record a decision*. Everything else is a read.

```
POST /api/v1/requests
Idempotency-Key: <uuid>

{
  "workflow_id": "leave_request",
  "workflow_version": null,           # null = latest published
  "inputs": { "start_date": "...", "end_date": "...", "days": 5 }
}
```

The response gives you back the request id and the tasks that just got created. If two parallel approvers need to act, you get two tasks back. If the workflow auto-approved this one (under-3-day leave), you get back a finalized request immediately.

```
POST /api/v1/tasks/{task_id}/decisions

{
  "decision": "approve" | "reject" | "request_changes" | "delegate",
  "comment": "...",
  "on_behalf_of": "user_bob"          # set if a delegate is acting
}
```

A few small but load-bearing choices:

- **Idempotency-Key is required on create.** Mobile retries the submit on timeout; without the key you get duplicate requests, double-charged budget reservations, and angry users.
- **`workflow_version: null` defaults to the latest published.** You can pin to a specific version for testing.
- **Tasks are the unit of work, not requests.** A request can have several tasks in flight at once (parallel approval). Approvers act on tasks. This matters for the database design too.

Status codes worth knowing: 409 on the decision endpoint means the task was already decided (someone else, or a delegate, beat you to it). 410 means the task was cancelled because the request was withdrawn or returned to the requester. Real UIs handle both.

### 4. Data model

Five tables. Two large, three small.

```sql
-- Workflows live here. Versioned, never edited in place.
CREATE TABLE workflow_definitions (
    workflow_id      TEXT NOT NULL,
    version          INT NOT NULL,
    status           TEXT NOT NULL,         -- 'draft' | 'published' | 'archived'
    definition       JSONB NOT NULL,
    published_at     TIMESTAMPTZ,
    published_by     TEXT,
    PRIMARY KEY (workflow_id, version)
);

-- One row per logical approval request.
CREATE TABLE requests (
    request_id          UUID PRIMARY KEY,
    workflow_id         TEXT NOT NULL,
    workflow_version    INT NOT NULL,
    requester_id        TEXT NOT NULL,
    state               TEXT NOT NULL,      -- current step id, or 'approved'/'rejected'/'cancelled'
    inputs              JSONB NOT NULL,
    state_data          JSONB,              -- engine working memory
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finalized_at        TIMESTAMPTZ,
    final_state         TEXT,
    idempotency_key     TEXT,
    FOREIGN KEY (workflow_id, workflow_version)
        REFERENCES workflow_definitions(workflow_id, version)
);
CREATE INDEX idx_req_requester ON requests (requester_id, created_at DESC);
CREATE INDEX idx_req_open ON requests (state) WHERE finalized_at IS NULL;

-- One row per pending human action.
CREATE TABLE tasks (
    task_id        UUID PRIMARY KEY,
    request_id     UUID NOT NULL REFERENCES requests(request_id),
    step_id        TEXT NOT NULL,
    assignee_id    TEXT NOT NULL,
    assigned_via   JSONB,                   -- delegation chain, if any
    state          TEXT NOT NULL,           -- 'pending' | 'decided' | 'cancelled' | 'expired'
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at     TIMESTAMPTZ              -- triggers escalation
);
CREATE INDEX idx_tasks_assignee_pending
    ON tasks (assignee_id) WHERE state = 'pending';

-- Immutable record of every human decision.
CREATE TABLE decisions (
    decision_id    UUID PRIMARY KEY,
    task_id        UUID NOT NULL REFERENCES tasks(task_id),
    request_id     UUID NOT NULL REFERENCES requests(request_id),
    actor_id       TEXT NOT NULL,
    on_behalf_of   TEXT,
    decision       TEXT NOT NULL,
    comment        TEXT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (task_id)                        -- one decision per task, enforced by the DB
);

-- Append-only audit. INSERT privilege only; no UPDATE or DELETE.
CREATE TABLE audit_log (
    event_id           UUID PRIMARY KEY,
    occurred_at        TIMESTAMPTZ NOT NULL,
    request_id         UUID NOT NULL,
    workflow_id        TEXT NOT NULL,
    workflow_version   INT NOT NULL,
    event_type         TEXT NOT NULL,
    actor              JSONB,
    payload            JSONB NOT NULL,
    snapshot           JSONB
);
CREATE INDEX idx_audit_request ON audit_log (request_id, occurred_at);
```

A few choices worth defending out loud:

The `UNIQUE (task_id)` constraint on `decisions` is doing real work. When two managers in a parallel approval both click "approve" at the same instant, the database serializes them. The first insert wins, the second fails with a unique-violation, the API turns that into a friendly 409, and the request still advances exactly once. No locks at the application layer needed.

`workflow_definitions` is *immutable per version*. Publishing v4 of `leave_request` does not touch v3. In-flight requests stay on the version they were created against, forever. This is what lets you ship workflow changes without breaking running requests and what makes audit replay work years later.

The `audit_log` has no foreign key to `requests` on purpose. If you ever delete a request (GDPR, mistaken bulk import), the audit must survive. Audit is the truth-of-record, not the request table.

Postgres, not Cassandra or DynamoDB. The whole system is small. ACID matters here (claiming a task, recording a decision, advancing the request state all happen together). Postgres gives you all of that in one box.

### 5. The engine

The engine is a state machine executor. One function does most of the work, called every time something might let a request progress: a request was just created, a decision came in, a timeout fired, or an external event arrived (CI status for a code review workflow).

```python
def advance_request(request_id):
    with db.transaction():
        request = db.lock_for_update("requests", request_id)
        if request.finalized_at is not None:
            return                                      # nothing to do

        workflow = workflow_store.get(request.workflow_id, request.workflow_version)
        current_step = workflow.find_step(request.state)

        # Is the current step done?
        if current_step.type == "approval":
            tasks = db.query_tasks(request_id, current_step.id)
            if not quorum_met(tasks, current_step.quorum):
                return                                  # still waiting on humans

        next_step = workflow.next_step(current_step, request.inputs, request.state_data)

        if next_step.type == "terminal":
            db.finalize(request_id, state=next_step.state)
            emit("request.finalized", request_id)
            return

        if next_step.type == "approval":
            assignees = resolve_assignees(next_step, request.requester)
            for who in assignees:
                db.insert_task(request_id, next_step.id, who, expires_at=...)
                emit("task.assigned", task_id)

        elif next_step.type == "auto":
            outcome = execute_auto_step(next_step, request)
            db.update_state(request_id, outcome.state)
            return advance_request(request_id)          # recurse: auto-steps chain

        db.update_state(request_id, next_step.id)
        emit("request.advanced", request_id, new_state=next_step.id)
```

Three things make this safe:

The whole advance happens in one DB transaction. Crashes mid-way roll back. After restart, the engine sees the same state and tries again. No partial-advance state ever lands on disk.

The `SELECT... FOR UPDATE` lock on the request row serializes concurrent attempts to advance the same request. Without it, two simultaneous decisions could each try to push the request forward and corrupt the state machine.

The function is idempotent at step boundaries. Calling it twice on the same request either sees the same state and proceeds the same way, or sees that someone else already advanced past this step.

### 6. Resolving an approver (the part that goes wrong in production)

The workflow says `approver: "{{ employee.manager }}"`. The engine has to turn that into a real person before it can create a task. That sounds simple. Production says otherwise.

```python
def resolve_approver(spec, requester, when):
    target = render_template(spec, {"employee": requester})

    if is_role(target):                                 # e.g. "finance-admin"
        members = org.role_members(target, at=when)
        if not members:
            raise NoApproverFound(target)
        target = pick_round_robin(members)

    return follow_delegation(target, when, depth=0, visited=set())

def follow_delegation(user, when, depth, visited):
    if depth > 5:
        raise DelegationTooDeep(user)
    if user.id in visited:
        raise DelegationCycle(visited)
    visited.add(user.id)

    if not user.exists:
        return fallback_for_departed(user)              # try last manager, then workflow fallback

    ooo = org.get_active_ooo(user, at=when)
    if ooo is None or ooo.delegate is None:
        return user                                     # OOO with no delegate falls through

    return follow_delegation(ooo.delegate, when, depth + 1, visited)
```

The shape comes from four real situations:

Bob is in. Fine. Resolves to Bob immediately.

Bob is OOO, delegated to Carol. Carol is in. Resolves to Carol after one hop.

Bob is OOO to Carol. Carol is also OOO to Dave. Two hops. The visited-set guard makes sure we never loop forever if Bob → Carol → Bob.

Bob left the company. The user lookup fails, we fall back to his last known manager, then to the workflow's `fallback_approver` if even that fails.

The `when` parameter looks redundant but matters for audit replay. To reconstruct *who would have been the approver* on the day this request was created, you need a point-in-time view of the org. People change jobs. Delegations expire. Roles get re-membered.

Cache role membership for five minutes and OOO state for one minute. That alone drops HRIS load by 100x. Invalidate on org-system webhooks for the cases where freshness really matters.

When you do follow a delegation chain, write the full chain to the task as `assigned_via`. Dave's dashboard then shows "you are approving on behalf of Bob, via Carol." The audit records the same chain, which is what an auditor will want to see five years later.

### 7. Architecture

Here is the whole picture. Drawn small enough to fit one screen.

```
                 ┌────────────────────────────────────────────────────┐
                 │ Clients: web, mobile, Slack/Teams bot              │
                 └─────────────────────┬──────────────────────────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │   API Gateway    │  auth, idempotency,
                              │                  │  rate limit
                              └────┬────────┬────┘
                       create      │        │      read dashboard
                                   │        │
                                   ▼        ▼
              ┌─────────────────────────┐  ┌──────────────────────┐
              │   Workflow Engine        │  │   Read Service        │
              │   (stateless pods)       │  │   (denormalized       │
              │                          │  │    dashboard cache)   │
              │   ┌─ resolves approvers ─┼─►│                       │
              │   ┌─ reads workflow defs ┼─►├──────────────────────┘
              │   └─ writes request state│
              └─────────┬──────┬─────────┘
                        │      │
                        │      ▼
                        │   ┌──────────────────┐
                        │   │  Org Service      │  manager-of, role members,
                        │   │  (HRIS adapter +  │  OOO + delegation
                        │   │   short TTL cache)│
                        │   └──────────────────┘
                        ▼
              ┌──────────────────────────────────────┐
              │  Postgres                              │
              │   workflow_definitions (small, hot)   │
              │   requests                            │
              │   tasks                               │
              │   decisions                           │
              │   audit_log (recent 90 days)          │
              └────────────────┬──────────────────────┘
                               │
                  CDC / outbox │
                               ▼
              ┌──────────────────────────────────────┐
              │  Kafka                                 │
              │   approval.request.created            │
              │   approval.task.assigned              │
              │   approval.decision.recorded          │
              │   approval.request.finalized          │
              └──┬──────────────┬──────────────┬──────┘
                 │              │              │
                 ▼              ▼              ▼
        ┌─────────────┐  ┌──────────────┐  ┌────────────────┐
        │ Notification │  │  Downstream   │  │ Analytics +     │
        │ Service      │  │  integrations │  │ Audit cold      │
        │ (email,Slack)│  │  (NetSuite,   │  │ tier (S3 +      │
        │              │  │  Workday)     │  │ Athena, 7yr)    │
        └─────────────┘  └──────────────┘  └────────────────┘
```

Five things to notice while reading this:

The engine reads workflow definitions but does not call out to other services on the write path. The Org Service is the only external call (cached aggressively). Everything else is a Postgres transaction.

Notifications, integrations, and audit archival are downstream of Kafka. They are not in the write path of the engine. If the notification service is down, requests still get created and approved; the emails just queue up.

The Read Service is denormalized. It listens to Kafka events and maintains a "what's pending for each user" cache. Dashboard reads never touch the primary Postgres in the common case.

Audit lives in two tiers. Recent 90 days in Postgres for fast investigations. Older events in S3 Parquet for compliance queries via Athena. A nightly job moves rows between tiers.

The engine pods are stateless and horizontally scalable. State is in Postgres. You can roll them in the middle of the day with no impact.

### 8. What a request looks like, end to end

Here is the create-request flow drawn as a sequence:

```
   Client       API GW       Engine       Org Svc      Postgres      Kafka
     │            │            │             │            │            │
     │ POST /req  │            │             │            │            │
     ├───────────►│            │             │            │            │
     │            │ validate   │             │            │            │
     │            ├───────────►│             │            │            │
     │            │            │ resolve     │            │            │
     │            │            │ first       │            │            │
     │            │            │ approver    │            │            │
     │            │            ├────────────►│            │            │
     │            │            │◄────────────┤            │            │
     │            │            │             │            │            │
     │            │            │ BEGIN TX                 │            │
     │            │            ├──────────────────────────►            │
     │            │            │ INSERT requests          │            │
     │            │            │ INSERT tasks             │            │
     │            │            │ INSERT audit_log         │            │
     │            │            │ COMMIT                   │            │
     │            │            │◄──────────────────────────┤            │
     │            │            │             │            │            │
     │            │            │ emit request.created │   │            │
     │            │            │ emit task.assigned   │   │            │
     │            │            ├──────────────────────────────────────►│
     │            │ 201 OK     │             │            │            │
     │            │◄───────────┤             │            │            │
     │ 201 OK     │            │             │            │            │
     │◄───────────┤            │             │            │            │
     │            │            │             │            │            │
     │            │            │             │            │  consumer  │
     │            │            │             │            │            ├──► email Bob
     │            │            │             │            │            ├──► slack message
     │            │            │             │            │            └──► analytics
```

Recording a decision looks almost the same. The engine fetches the task, validates the caller, inserts a row in `decisions` (unique constraint catches double-decide races), updates the task, writes an audit event, and calls `advance_request` to see if the workflow can move forward. All in one transaction.

Reads have their own short flow. Client hits the API gateway, the request goes to the Read Service, the Read Service checks its Redis cache (`user:{uid}:pending_tasks`), and returns. Cache misses fall through to a read replica of Postgres; whichever hits, the result is fast.

Target latencies, for a sense of scale: create request P99 around 200ms (bottleneck is the role-resolver round-trip when cache is cold). Decision P99 around 150ms (role is usually warm because it was looked up when the task was created). Dashboard read P99 around 50ms.

### 9. Scaling journey: 10 to 1M users

This is the part interviewers care about most. At every stage, name what just broke and what fixes it. Build nothing preemptively.

**Stage 1: 10 to 100 users.** One Postgres, one app instance. Workflow YAML lives in the repo. Manager mappings hardcoded. No cache, no queue. Notifications go out as inline HTTP calls to SendGrid. About $80/month total. You ship in two weeks.

This is enough because you see ten requests a day. Postgres is loafing. The audit query "who approved request X" runs in 5ms. Building anything more is over-engineering.

**Stage 2: 1k users.** Something breaks: somebody asks for a new workflow type without a deploy. Now workflow YAML belongs in a database table with a minimal admin UI. While you are in there, you add versioning so published workflows are immutable. You also stop hardcoding manager mappings and integrate with the HRIS (Workday or BambooHR), with a 5-minute cache. Notifications stop being inline calls and start consuming a small `request_events` table that a notifier process polls. About $250/month.

Still no Kafka, no read replicas, no Redis. You see one request every couple of minutes. One Postgres is still fine.

**Stage 3: 10k to 100k users.** Now several things break at once. The dashboard for power approvers takes four seconds because hot users have 100+ pending tasks. Audit queries on 80M rows time out. Engine pods compete for the same `SELECT FOR UPDATE` lock under bursty load. The Workday API rate-limits you during peak hours.

The fixes, in order: add two Postgres read replicas (dashboards read from them, engine writes go to primary, ~1s replication lag is fine). Add a Redis cache for "my pending tasks" populated by listening to engine events; dashboards read Redis directly. Move audit older than 90 days to S3 Parquet, queryable via Athena. Partition the engine pods by `request_id` hash so they stop racing for the same locks across pods. Tighten the Org Service cache to pre-fetch on dashboard load. Bring in Kafka properly: notifications, audit tiering, and external integrations all become consumers.

Cost jumps to $2-5k/month for the bigger DB, replicas, Redis, Kafka, S3.

**Stage 4: 1M users.** New problems: EU operations open, EU data must stay in EU. One customer is a hospital and needs HIPAA-grade audit hash chaining. 5000+ workflow definitions, different teams want different RBAC. Some workflows have 30 approval steps and live for weeks.

Multi-region everything. Engine, DB, Redis, Kafka all per-region. Requester's home region (from HRIS) determines where the request lives and where audit is written. Cross-region approvals (US person approves an EU request) routed via authenticated cross-region API. Workflow definitions get their own RBAC by "policy domain." Audit events get a hash chain so tampering breaks the chain. Long-running workflows survive engine deploys because state lives entirely in DB and pods are interchangeable.

The architecture has not fundamentally changed since stage 3. You added regions, RBAC, and tenant isolation, but the core data model is the same one you wrote in stage 1.

What you would do at 10M users: by then you are bigger than any single company, which means you are selling this as SaaS (Workday, ServiceNow scale). At that point you might swap your homegrown engine for Temporal as the runtime and keep your approval-specific logic on top.

### 10. The four variants, fast

Same engine, four real workflows. Each stresses a different feature.

- **Purchase order** uses conditional branching. `when: amount > 5000` sends it to finance. `when: amount > 25000` adds the CFO. Lint your conditions or you get bugs like a $25,001 PO going to both finance and the CFO unnecessarily.
- **Leave request** uses parallel with quorum. Long leaves go to HR and the grandboss at the same time, both must approve. If one rejects, the other's task is cancelled and the request rejects.
- **Expense report** needs backward transitions. Finance says "missing receipt." The request goes back to the requester, who edits and resubmits. The engine supports `on_action: return_to(<step>)` for this. Subtle: pending tasks downstream get cancelled when the request rewinds.
- **Code review** needs external-event integration. The workflow waits for CI status, not a human. When a new commit lands, prior approvals get invalidated via `on_input_change: invalidate_approvals`.

One engine, four workflows, no special cases. That is the design victory.

### 11. Reliability

Engine crashes mid-advance roll back cleanly. State is in the DB, the transaction is atomic, the lock prevents concurrent advance attempts. After restart, the engine sees the request's current state and continues from there.

The escalation worker is separate from the engine. It scans for tasks where `expires_at < now()` and records timeout events. If it dies, escalations are delayed by however long it takes to restart. Idempotent: re-running the worker over the same expired tasks does nothing harmful.

Audit writes happen in the same transaction as the state change. If the audit insert fails, the state change rolls back. You never advance state without an audit record.

If the HRIS goes down, the Org Service serves stale data from cache. The engine keeps working with whoever was cached at last refresh. After an hour, the engine logs warnings. After 24 hours, it refuses to assign tasks that require role lookups (named-user assignments still work).

If a region goes down, that region's requests pause. Cross-region approvals routed through the down region queue in Kafka. When the region comes back, the queued operations replay.

### 12. Observability

| Metric | Why it matters |
|--------|----------------|
| `requests.created.rate` | Spike = bot. Drop = auth or HRIS broken. |
| `requests.in_flight` | Growing slowly = approvals are stalling on someone. |
| `request.cycle_time` p50, p95 per workflow | The headline SLO. Often audit-required. |
| `task.assignment_latency` p99 | If > 1s, role resolver is sluggish. |
| `escalations.fired.rate` | Spike = lots of stalled tasks. Could be workflow bug, could be flu season. |
| `delegation.depth.max` | Should never exceed 3-4. Higher = OOO chains that need fixing. |
| `delegation.cycles.detected` | Should be 0. Page if non-zero. |
| `engine.lock_wait` p99 | High = you need partitioning across pods. |
| `audit.write.lag` | Audit must never lag. Alert at > 5s. |

Page on: engine error rate > 1% over 5min, audit write failure (any), delegation cycle detected. File a ticket on: cycle time regression > 30%, HRIS cache age > 30 min.

### 13. Gotchas the senior interviewer is listening for

Some of these only come out when the interviewer asks "what happens if...". The senior candidate brings them up unprompted.

**Delegation cycles.** Alice delegates to Bob, Bob delegates to Alice. The recursion sees this and raises. Mitigation: assign to the original target with a warning. The UI tells them to fix their OOO settings.

**Departed approvers.** A nightly job finds tasks whose `assignee_id` no longer exists in HRIS and reassigns to the assignee's last manager. Mass layoffs need a one-shot rebalance tool that reassigns to a triage queue.

**Conditional clauses with overlapping ranges.** $5000 matches `> 1000`. $5000 also matches `> 5000` if you wrote it as `>=`. Boundary cases bite. Lint the workflow definition at publish time.

**Self-approval.** Someone in the finance role submits a finance-approval request. Filter `approver != requester` in `resolve_approver`. Critical. The interviewer will ask.

**Notification storms.** A request transitions through five states in two minutes. Naive design sends five emails. Aggregate per-recipient per-request on a 60-second window and send one summary.

**Send-back leaves orphan tasks.** The expense report goes back to the requester, but the finance-pending task is still showing in finance's dashboard. The engine has to close downstream tasks with reason `request_returned` when rewinding.

### 14. Follow-up answers

**1. Self-approval prevention.** Filter the requester out of the eligible approver set in `resolve_approver`. For role-based steps, pick the next available role member. For named-user steps where the user happens to be the requester, raise a workflow definition error. The check lives in the engine, not just the API, because someone could reassign a task to the requester via delegation otherwise.

**2. Approver leaves the company.** A nightly job:

```sql
SELECT t.task_id, t.assignee_id, r.requester_id
  FROM tasks t JOIN requests r ON t.request_id = r.request_id
 WHERE t.state = 'pending'
   AND t.assignee_id NOT IN (SELECT id FROM users WHERE active);
```

For each orphan, reassign to the original assignee's last manager from HRIS history. If not findable, fall back to the workflow's `fallback_approver`. Notify the new assignee with context: "this task was originally assigned to X who has since left." For layoffs, run the job on demand with a bulk-reassign mode.

**3. Delegation cycle.** The `visited` set in `follow_delegation` raises `DelegationCycle` on a repeat. Assign to the original target anyway, with a warning in the UI: "We detected a circular delegation; please update your OOO settings." Prevention: when a user sets their delegate, walk the chain and refuse if it includes them.

**4. Workflow version migration.** In-flight requests stay on their original version forever. `requests.workflow_version` is set at create time and never updated. Old versions are kept indefinitely. If you ship v4 with a critical fix and want in-flight requests to benefit, you do not modify v3; you manually apply a "patch" to specific requests (rare; carries audit annotation `workflow_patched`). Workflows can be archived but never deleted while any historical request references them.

**5. Concurrent approval race.** `UNIQUE (task_id)` on `decisions` is the lock. Two approvers click at once, two INSERTs race, the first wins, the second gets a unique-violation, the API returns 409 to that user. The first decision's `advance_request` runs, sees parallel-quorum-any is satisfied, and cancels the sibling tasks. The losing approver sees "task no longer requires action" (410 Gone).

**6. Auto-approval bug.** Detection: `metrics.auto_approve.rate` is normally stable, so a 100x spike triggers a page. Recovery: an `admin_reopen(request_id)` endpoint creates an `audit.admin_reopened` event, resets state to a configurable point, and cancels downstream effects. The downstream is whoever consumes the "approved" event; if NetSuite already wrote a payment record, NetSuite needs its own reversal. Going forward: cap auto-approve rate per workflow per hour; a circuit breaker switches to manual review on overrun.

**7. Bulk import.** A separate `bulk_import` endpoint bypasses the engine and writes directly to `requests`, `tasks`, `decisions`, and `audit_log` with the original timestamps. Each audit event is tagged with `source: bulk_import_<batch_id>` so future auditors can tell imported from live. The audit hash chain (if enabled) treats imported events as a separate chain. Admin-only. Heavily audited itself.

**8. Dashboard performance for power approvers.** Carol's 120 pending tasks make her dashboard slow because the frontend issues 120 sub-queries hydrating each task. Fix: the backend returns a denormalized "task list view" per assignee from the Read Service cache. Single Redis read. Cache updated incrementally on `task.assigned` and `task.decided` events. Paginate if she has > 50 tasks.

**9. Search across all approvals.** Postgres JSONB search on `requests.inputs` is slow without help. For most companies, a `tsvector` GIN index on `to_tsvector('english', inputs::text)` is enough. Above ~10M searchable requests, pipe `requests` to Elasticsearch or ClickHouse via CDC.

**10. NetSuite integration with idempotency.** A `netsuite-sync-worker` consumes `request.finalized` from Kafka. For each event: check an `external_syncs (request_id, target)` table; if status is `success`, skip. Otherwise call NetSuite with `idempotency_key = "ns-{request_id}"` (NetSuite uses it to dedupe). On 2xx, mark success. On 5xx, exponential backoff (1s, 5s, 25s, 2m, 10m, 30m, deadletter). On 4xx, mark `failed_permanent` and alert a human.

**11. Notification storm.** Already covered: 60-second aggregation window per recipient per request, sent as one digest. Configurable per user (instant, hourly digest, daily summary). The notification service owns preferences; the engine just emits raw events.

**12. The "approve all" button.** API: `POST /api/v1/tasks/bulk_decide` with `{"task_ids": [...], "decision": "approve"}`. Server iterates serially through the list (serial because each decision might advance a request and concurrent advances cause version skew). 80 decisions at 50ms each = 4 seconds. Returns a per-task result array. Some tasks might 409 if a delegate beat the user to it; the result shows that.

**13. Privacy and visibility scoping.** Workflow definitions get a `visibility_scope` field: `public | private | department | role:<role>`. Denormalized to `requests.visibility_scope`. All read APIs filter by it. Restricted workflows publish to a separate Kafka topic (`approval.events.restricted`) that only authorized consumers subscribe to. Notifications for restricted workflows omit details: "you have a pending approval; log in to see."

**14. Infinite-loop workflows.** Publish-time check: graph-traverse the workflow definition. If there is a cycle without any `when:` condition that can break it, refuse to publish. Some cycles are intentional (review → request changes → review), allowed only if at least one transition has a guard referencing input data that can change. Runtime safety net: max-transitions counter per request; > 100 transitions pauses for human review.

**15. Multi-region with data residency.** Each region has its full stack. Requester's home region (from HRIS) determines where the request lives. Cross-region approvals: the EU region creates the task and assigns to the US employee, who sees it via the US region's dashboard. When the US employee decides, the decision is forwarded to EU via an authenticated cross-region API call (queued in a `cross_region_decisions_outbox` table and retried idempotently). The decision is recorded in the EU region's tables and audit. The US region keeps only a "this US user sent a decision to EU region" record for its own audit, no PII about the EU request.

The lesson: data residency forces the request to be the unit of locality. The user can be anywhere; the request is somewhere specific.

### 15. Trade-offs worth saying out loud

**Why a custom engine and not Temporal or Step Functions.** Off-the-shelf workflow engines are great for general computation. They are awkward for approval-shaped workflows because the role, delegation, and audit story is approval-specific and would have to be built on top anyway. At 10M+ workflows you might pick Temporal as the runtime; the approval logic stays in your service either way.

**Why Postgres and not Cassandra.** Approval needs strong consistency. No double-approve. No lost decisions. Postgres gives ACID for free. Cassandra would force you to invent compensating logic for the same guarantees.

**Why YAML for workflow definitions and not code.** YAML lets non-engineers author workflows through a UI. Plugins force every workflow change through a deploy. The cost is limited expressiveness; for the few workflows that need real code (a custom integration step), the engine has an `exec` step that runs sandboxed JavaScript.

**Why an immutable audit log and not just event-sourcing the requests.** Event-sourcing would make audit queries fast but ties the application's data model to the audit format. Decoupling lets you refactor the application without breaking audit. Worth the duplication.

**What you would revisit at 10x scale.** Adopt Temporal as the runtime. Move workflow authoring into its own product with its own DB and RBAC. Move audit to a dedicated immutable-storage product (QLDB, or a hash-chained ledger).

### 16. Common interview mistakes

Most weak answers fall into one of these:

**Modeling requests with a hardcoded status enum.** If your design has `status pending | approved | rejected` and transitions live in if-statements, you have written CRUD, not a workflow service. The whole point is a data-driven engine.

**No version pinning on workflows.** If in-flight requests follow the latest version, you introduce time-traveling bugs and break audit replay.

**Self-approval allowed by default.** Junior candidates rarely think of it. The interviewer will ask.

**Audit as a side effect.** "I'll just log it." Audit has a schema, a retention policy, an immutability contract, and a query interface. It is a feature, not a log line.

**Treating notifications as part of the engine.** They are not. Engine emits events, notification service consumes. This separation is what lets you add Slack, Teams, or SMS without touching the engine.

**Designing for write throughput.** Even at enterprise scale you have ten writes per second. The hard numbers are about read latency for dashboards and audit queries.

**Forgetting parallel approval with quorum.** Serial-only works for 80% of workflows. The other 20% (medical, legal, compliance) need parallel. Retrofitting it later is expensive.

**Hand-waving the org service.** The role, delegation, and OOO machinery is half the system. Treating it as a single `manager_id` column loses the design.

**No mention of multi-region or data residency.** GDPR alone forces it at enterprise scale.

**Microservices "the right way" before understanding the workflow shape.** Start from the data model. Service boundaries follow from that, not the other way around.

If you can articulate seven of these ten without prompting, you are interviewing at staff level. The three that separate strong from average answers: engine versus CRUD, audit immutability, and role-resolution depth. Those are the answers a senior architect listens for.
