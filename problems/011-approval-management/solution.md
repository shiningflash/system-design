## Solution: Approval Management Service

### TL;DR

An approval service is a workflow engine that reads a versioned YAML/JSON definition, walks a state machine, and assigns human tasks. The interesting design work is not the runtime, it is everything around it: how roles get resolved to actual users (handling out-of-office, delegation, departures), how concurrent decisions are reconciled, how the audit trail stays trustworthy for seven years, and how a single engine handles ten different workflow shapes without becoming a hardcoded mess.

The core data model is small. A `requests` row per logical approval request. A `tasks` row per pending human action. A `decisions` row per recorded human action. An immutable `audit_log` capturing every state transition with a snapshot. A separate `workflow_definitions` store holding the YAML, versioned and never edited in place.

The scaling story is not about queries per second. At 100k employees you only see ~1 request per second. The scale problem is *organizational*: 5000 workflow variants, thousands of roles, multiple regions with data residency, audit retention measured in years. The right architecture starts as a single Postgres and adds, in order: a read-side cache for dashboards, role-resolver caching, an event stream for downstream consumers, then audit-log tiering to cold storage. Each step is driven by a specific pain point, never preemptive.

### 1. Clarifying questions and why each matters

Covered in `question.md`. The most important question is "single workflow type or many." A service for *only* leave requests is a CRUD app. A *workflow engine* handling many workflow types defined by config is the actual system design.

The second most important is "do non-engineers author workflows." If yes, you need a UI, a YAML store, versioning, validation, and lint checks. If workflows are baked into code, you skip all of that but you also lose 90% of the value of building this as a service.

### 2. Capacity estimates (full working)

**50-person startup:**

- Requests: 50 × 5/week = ~36/day. Trivial.
- Active requests (in flight): ~50 to 70 at any moment.
- Decisions: ~72/day.
- Dashboard loads: 50 × 10 × business-day = ~500/day.

A single t3.medium Postgres handles all of this with room to spare. A single application instance is fine. No queue needed; engine state transitions are synchronous database writes.

**100k-person enterprise:**

- Requests: 500k/week = ~71k/day = ~1/sec sustained, ~3/sec peak.
- Active requests in flight: ~200k.
- Decisions: ~210k/day = ~3/sec sustained, ~10/sec peak.
- Dashboard loads: 100k × 10 = 1M/day = ~25/sec sustained, ~75/sec peak.

Writes are tiny (10/sec peak). Reads dominate (75/sec dashboard + ~10/sec engine internal lookups). The interesting numbers:

- **Active requests = 200k.** Indexes must support "WHERE assignee = ? AND status = 'pending'" returning in <50ms for any user.
- **5000 workflow definitions.** Hot ones (leave, expense) accessed thousands of times per day. Cold ones (annual security review) accessed once a quarter.
- **Audit log retention: 7 years.** At 200k events/day × 365 × 7 = ~510M audit events. At ~500 bytes each = ~250GB. Tier-able to S3.

Even at enterprise scale, this is a *small* workload by modern standards. The architecture exists not for throughput but for **correctness, traceability, and organizational flexibility**.

### 3. API design

**Create request:**

```
POST /api/v1/requests
Content-Type: application/json
Idempotency-Key: <uuid>
Authorization: Bearer <token>

{
  "workflow_id": "leave_request",
  "workflow_version": null,           # null = use latest published; or pin to a version
  "inputs": {
    "employee_id": "emp_42",
    "start_date": "2026-06-01",
    "end_date": "2026-06-05",
    "days": 5
  },
  "metadata": {
    "source": "web",
    "client_request_id": "..."
  }
}
```

Responses:

| Status | Meaning | Body |
|--------|---------|------|
| 201 Created | Request created, engine started executing | `{"request_id": "req_abc123", "state": "pending_manager_approval", "current_tasks": [{"task_id": "tsk_xyz", "assignee": "user_bob"}]}` |
| 200 OK | Same Idempotency-Key seen before; return existing | same shape |
| 400 Bad Request | Inputs failed workflow schema validation | `{"error": "invalid_input", "details": {"days": "must be >= 1"}}` |
| 404 Not Found | Workflow id does not exist or no published version | `{"error": "workflow_not_found"}` |
| 409 Conflict | Workflow auto-rejected (e.g., requester is on a frozen list) | `{"error": "auto_rejected", "reason": "..."}` |

**Record a decision (the most-called endpoint):**

```
POST /api/v1/tasks/{task_id}/decisions
{
  "decision": "approve" | "reject" | "request_changes" | "delegate",
  "comment": "Looks fine.",
  "delegate_to": "user_carol",        # only if decision == delegate
  "on_behalf_of": "user_bob"          # if Carol approves as Bob's delegate
}
```

Responses:

| Status | Meaning |
|--------|---------|
| 200 OK | Decision recorded, engine advanced state |
| 403 Forbidden | Caller is not the assigned approver |
| 409 Conflict | Task already decided (race condition) |
| 410 Gone | Task was cancelled (request was withdrawn or returned to requester) |

**Read endpoints:**

```
GET /api/v1/requests/{request_id}                  # full request detail with history
GET /api/v1/requests?status=in_flight&for_me=true  # my pending approvals
GET /api/v1/requests?requester=me                  # my requests
GET /api/v1/requests/{request_id}/audit            # full audit trail (auth required)
```

**Workflow management (admin):**

```
GET /api/v1/workflows                              # list all workflow definitions
GET /api/v1/workflows/{id}/versions                # version history
POST /api/v1/workflows/{id}/versions               # draft a new version
POST /api/v1/workflows/{id}/versions/{v}/publish   # publish (immutable after)
```

**Notes on the API:**

- **Idempotency-Key on create.** Mobile clients retry on timeout. Without an idempotency key, double-submit creates duplicate requests. The key is hashed and stored for 24 hours.
- **`workflow_version: null` defaults to latest.** Pinning to a version is useful for testing.
- **Tasks are the unit of work, not requests.** A request can have multiple in-flight tasks (parallel approval). Approvers act on tasks, not requests.

### 4. Data model

```sql
-- Workflow definitions (the YAML/JSON the engine reads).
CREATE TABLE workflow_definitions (
    workflow_id      TEXT NOT NULL,                -- "leave_request"
    version          INT NOT NULL,                 -- 3
    status           TEXT NOT NULL,                -- 'draft' | 'published' | 'archived'
    definition       JSONB NOT NULL,               -- the workflow spec
    published_at     TIMESTAMPTZ,
    published_by     TEXT,
    PRIMARY KEY (workflow_id, version)
);
CREATE INDEX idx_wd_status ON workflow_definitions (workflow_id, status) WHERE status = 'published';

-- Requests (one row per logical approval request).
CREATE TABLE requests (
    request_id          UUID PRIMARY KEY,
    workflow_id         TEXT NOT NULL,
    workflow_version    INT NOT NULL,
    requester_id        TEXT NOT NULL,
    state               TEXT NOT NULL,             -- e.g., 'pending_manager_approval', 'approved', 'rejected'
    inputs              JSONB NOT NULL,             -- the input payload
    state_data          JSONB,                      -- engine's working memory (visited steps, derived values)
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finalized_at        TIMESTAMPTZ,                -- non-null when terminal
    final_state         TEXT,                       -- 'approved' | 'rejected' | 'cancelled'
    idempotency_key     TEXT,
    FOREIGN KEY (workflow_id, workflow_version) REFERENCES workflow_definitions(workflow_id, version)
);
CREATE INDEX idx_req_requester ON requests (requester_id, created_at DESC);
CREATE INDEX idx_req_state ON requests (state) WHERE finalized_at IS NULL;
CREATE UNIQUE INDEX idx_req_idempotency ON requests (idempotency_key) WHERE idempotency_key IS NOT NULL;

-- Tasks (one row per pending human action).
CREATE TABLE tasks (
    task_id             UUID PRIMARY KEY,
    request_id          UUID NOT NULL REFERENCES requests(request_id),
    step_id             TEXT NOT NULL,             -- step within the workflow
    assignee_id         TEXT NOT NULL,             -- resolved user
    assignee_role       TEXT,                       -- the role spec (for display)
    assigned_via        JSONB,                      -- delegation chain if applicable
    state               TEXT NOT NULL,              -- 'pending' | 'decided' | 'cancelled' | 'expired'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ,                -- when escalation/timeout kicks in
    decided_at          TIMESTAMPTZ
);
CREATE INDEX idx_tasks_assignee ON tasks (assignee_id, state) WHERE state = 'pending';
CREATE INDEX idx_tasks_request ON tasks (request_id);
CREATE INDEX idx_tasks_expires ON tasks (expires_at) WHERE state = 'pending';

-- Decisions (immutable record of every human action).
CREATE TABLE decisions (
    decision_id         UUID PRIMARY KEY,
    task_id             UUID NOT NULL REFERENCES tasks(task_id),
    request_id          UUID NOT NULL REFERENCES requests(request_id),
    actor_id            TEXT NOT NULL,             -- who clicked the button
    on_behalf_of        TEXT,                       -- if delegate, original assignee
    decision            TEXT NOT NULL,              -- 'approve' | 'reject' | 'request_changes' | 'delegate'
    comment             TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (task_id)                                -- one decision per task; enforces no double-decide
);
CREATE INDEX idx_decisions_actor ON decisions (actor_id, created_at DESC);

-- Audit log (append-only, never edited).
CREATE TABLE audit_log (
    event_id            UUID PRIMARY KEY,
    occurred_at         TIMESTAMPTZ NOT NULL,
    request_id          UUID NOT NULL,             -- not FK; audit must survive request deletion
    workflow_id         TEXT NOT NULL,
    workflow_version    INT NOT NULL,
    event_type          TEXT NOT NULL,
    actor               JSONB,
    payload             JSONB NOT NULL,
    snapshot            JSONB
);
CREATE INDEX idx_audit_request ON audit_log (request_id, occurred_at);
CREATE INDEX idx_audit_time ON audit_log (occurred_at DESC);
```

**Why each table:**

- **`workflow_definitions`** is *immutable per version*. Publishing a new version inserts a new row. Old versions are kept forever so in-flight requests can resolve their step definitions.
- **`requests`** holds the current state. The `state_data` JSONB is the engine's working memory (which steps have run, computed values, branch decisions).
- **`tasks`** holds the *pending* human work. The `UNIQUE` constraint on `decisions.task_id` is what prevents double-decide races: two managers click Approve, the second insert fails with a unique-constraint error, the engine handles the conflict gracefully.
- **`decisions`** is technically derivable from audit_log, but having it as a primary table makes "show me all decisions by user X" fast without scanning audit.
- **`audit_log`** is the truth-of-record for compliance. INSERT-only privilege at the DB level.

**Choice: Postgres, not Cassandra or DynamoDB.** This system is small. ACID transactions matter (claiming a task, recording a decision, advancing the request state). Postgres gets you all of this in one DB. Cassandra would force you to invent your own consistency on top.

### 5. The workflow engine (core algorithm)

The engine is a state machine executor. Pseudocode of the central loop:

```python
def advance_request(request_id):
    """Called whenever something might let a request progress:
       - request created
       - a decision was just recorded
       - a timeout fired
       - an external event arrived (e.g., CI status from code review workflow)
    """
    with db.transaction():
        request = db.lock_for_update("requests", request_id)
        if request.finalized_at is not None:
            return

        workflow = workflow_store.get(request.workflow_id, request.workflow_version)
        current_step = workflow.find_step(request.state)
        
        # Determine if the current step is "done"
        if current_step.type == "approval":
            tasks = db.query("SELECT * FROM tasks WHERE request_id = ? AND step_id = ?",
                              request_id, current_step.id)
            if not all_tasks_decided(tasks, current_step.quorum):
                return  # still waiting
        
        # Step is done. Determine next step.
        next_step = workflow.next_step(current_step, request.inputs, request.state_data)
        
        if next_step.type == "terminal":
            db.update("requests", request_id,
                      state=next_step.state,
                      finalized_at=NOW(),
                      final_state=next_step.state)
            emit_event("request.finalized", request_id, snapshot=request)
            return
        
        # Advance to next step. Create new tasks if needed.
        if next_step.type == "approval":
            assignees = []
            if next_step.kind == "single":
                approver = resolve_approver(next_step.approver_spec, request.requester, NOW())
                assignees.append(approver)
            elif next_step.kind == "parallel":
                for branch in next_step.branches:
                    approver = resolve_approver(branch.approver_spec, request.requester, NOW())
                    assignees.append(approver)
            
            for assignee in assignees:
                task_id = db.insert("tasks",
                                     request_id=request_id,
                                     step_id=next_step.id,
                                     assignee_id=assignee.id,
                                     state="pending",
                                     expires_at=NOW() + next_step.timeout)
                emit_event("task.assigned", task_id)
        
        elif next_step.type == "auto":
            # Steps that don't need human input: auto-approve, call API, etc.
            outcome = execute_auto_step(next_step, request)
            db.update("requests", request_id, state=outcome.state)
            # Recurse to advance further
            return advance_request(request_id)
        
        db.update("requests", request_id, state=next_step.id, updated_at=NOW())
        emit_event("request.advanced", request_id, new_state=next_step.id)
```

**Key properties:**

- **Transactional.** Every state advance happens in a DB transaction. Crashes mid-advance leave the request in its prior state, not a half-advanced state.
- **Idempotent at the step boundary.** Calling `advance_request` twice on the same request is safe; the second call sees the same state and either does nothing or proceeds the same way.
- **Recursive for auto-steps.** Auto-approve and other no-human-needed steps chain without waiting.
- **Locks the request row.** `SELECT ... FOR UPDATE` serializes concurrent advances. Without this, two simultaneous "decision recorded" events could try to advance the request in parallel and corrupt state.

### 6. Role resolution and delegation

Already covered in question.md Step 5. The key implementation detail not yet stated:

**Cache aggressively.** Role membership and OOO data changes slowly (minutes to days). Cache `org.role_members(role, at=now)` for 5 minutes. Cache `org.get_active_ooo(user)` for 1 minute. Invalidate on org-system webhooks. Reduces Org Service load by ~100x.

**Snapshot delegation chains in `assigned_via`.** When the engine resolves "approver: Bob → Carol (delegate) → Dave (delegate)" and assigns the task to Dave, store `{"original": "bob", "chain": ["carol", "dave"]}` on the task. Dave's dashboard shows "approving on behalf of Bob via Carol." When Dave later decides, the audit records the full chain.

### 7. Architecture (already in question.md)

Reviewing the components:

| Component | Stateful? | Scaling story |
|-----------|-----------|--------------|
| API Gateway | No | Horizontal pods, sticky session not required |
| Workflow Engine | No (state in DB) | Horizontal pods, partitioned by request_id hash to reduce lock contention |
| Workflow Definition Store | Yes, but tiny | One Postgres table, all definitions fit in memory |
| Org Service / Role Resolver | Stateful (cache) | Read-through cache against HRIS, scale by adding replicas |
| Request DB (Postgres) | Yes | Primary + read replicas; partition only when truly needed (rare) |
| Read Service | No (denormalized cache) | Horizontal; backed by a separate "dashboard read" DB |
| Audit Log | Yes, large | Hot (Postgres) → cold (S3 + Athena/ClickHouse) tier transition |
| Kafka | Yes | Standard cluster scaling |

### 8. Read and write paths

**Write path (create request):**

1. Client → API Gateway. Auth, idempotency-key lookup.
2. Idempotent? Return cached response.
3. Engine validates `inputs` against workflow schema. Returns 400 on schema mismatch.
4. Engine resolves the first step. If approval, resolves approver.
5. In one DB transaction: insert `requests` row, insert `tasks` row(s), insert `audit_log` event.
6. Emit `request.created` and `task.assigned` events to Kafka.
7. Return 201 with request_id and current tasks.

P99 target: 200ms. Bottleneck: role resolution (one call to Org Service per task, cached).

**Write path (record decision):**

1. Client → API Gateway. Auth.
2. Engine fetches task. Validates: task is pending, caller is the assignee (or is decided to be acting on behalf of with permission), decision shape is valid.
3. In one DB transaction:
   - `INSERT INTO decisions ...` (UNIQUE constraint on task_id catches double-decide races; second insert fails with 409)
   - `UPDATE tasks SET state = 'decided', decided_at = NOW()`
   - `INSERT INTO audit_log ...`
4. Call `advance_request(request_id)` to see if the workflow can progress.
5. Emit events.
6. Return 200.

P99: 150ms. Often faster because role resolution is cached from when the task was created.

**Read path (my pending approvals dashboard):**

1. Client → API Gateway → Read Service.
2. Read Service queries the denormalized "my tasks" cache (Redis hash: `user:{uid}:pending_tasks` → list of task IDs).
3. Cache miss: fall through to `SELECT task_id FROM tasks WHERE assignee_id = ? AND state = 'pending'` against the read replica. Populate cache with 5min TTL.
4. For each task, hydrate from the cache of request summaries.

P99: 50ms. Cache hit rate >95% for active users.

### 9. Scaling journey: 10 → 1k → 100k → 1M users

This is the section the interviewer cares about most. At every scale we name what *just* broke and what fixes it.

#### Stage 1: 10 to 100 users (a small startup)

**What you build:**
- One Postgres (db.t3.medium, 4GB RAM).
- One Python/Go/Node app instance.
- Workflow YAML files committed to the repo. No UI for authoring.
- No cache. No queue. No replicas.
- Audit log = same Postgres table.

**Why this is enough:**
- ~10 requests/day. ~30 dashboard loads/day. Postgres yawns.
- 5 workflow types max. They live in the codebase. New workflow = new commit.
- One person reads audit. They use `psql` directly.

**What you do *not* build:**
- No Kafka. Notifications are inline HTTP calls to whatever email service you use.
- No Org Service. Hardcode the manager mapping in a `managers.yaml` checked into the repo.
- No multi-region anything.

**Total cost:** ~$50/month for the database, ~$30/month for the app. You ship in two weeks.

#### Stage 2: 1k users (Series A startup)

**What just broke:**

- "I want to add a new approval workflow without a deploy." Workflow YAML in code is now a friction point.
- HR Director asks for "show me who delegated to whom in the last quarter for our audit." The audit log is in the primary DB and runs slow queries.
- Sales team wants approvals for sales contracts. Marketing wants approvals for campaign budgets. The hardcoded `managers.yaml` is wrong half the time.

**What you add:**

- **Workflow Definition Store.** Move workflows out of code into a `workflow_definitions` Postgres table. Build a minimal admin UI for editing.
- **Versioning.** When a workflow is edited, save a new version. Published versions are immutable. In-flight requests stay on their published version.
- **Integrate with HRIS (Workday/BambooHR).** Replace `managers.yaml` with calls to the HRIS for manager lookups. Cache for 5 minutes.
- **Notifications.** Build a thin "approval-notifier" service that consumes a `request_events` table (or a basic queue if you have one) and sends emails. Outsource SMTP to SendGrid.

**What you do *not* yet build:**

- No read replicas. One Postgres still handles all reads.
- No Kafka. The `request_events` table polled by the notifier is enough.
- No partitioning of audit. ~30k audit events/month is nothing.

**Why this is enough:**
- 1k users × 5 req/week = 5000 req/week = ~700/day = ~1 request every 2 minutes. Still trivial for one DB.
- Dashboard loads: 1k × 10 = 10k/day = ~3/min. Postgres handles this easily.

**New cost:** ~$200/month (bigger DB, basic email service).

#### Stage 3: 10k to 100k users (a mid-size company or enterprise customer)

**What just broke:**

- "My pending approvals" dashboard takes 4 seconds. There are 50k pending tasks across all users; the query plan is no longer obvious to the optimizer.
- Audit log is now 80M rows. Some compliance queries against it take minutes.
- Workflow engine pods compete for the same `SELECT ... FOR UPDATE` lock when multiple requests advance simultaneously.
- The Workday API rate-limited you. The Org Service is now your bottleneck during peak hours.

**What you add:**

- **Read replicas.** Primary + 2 read replicas. Reads go to the replicas. Engine writes go to primary. ~1-second replication lag is acceptable for dashboards.
- **Denormalized "my tasks" cache (Redis).** A separate process listens to engine events and maintains `user:{uid}:pending_tasks` Redis lists. Dashboards read Redis directly. Replicas are the fallback.
- **Audit log tiering.** Move events older than 90 days from Postgres to S3 (Parquet). Use Athena for ad-hoc audit queries on cold data. Hot tier stays in Postgres for recent investigations.
- **Workflow engine partitioning.** Engine pods now hash-partition by `request_id`. Each pod owns a slice of requests; advances for that slice queue locally instead of racing for DB locks across pods.
- **Org Service cache tightening.** Cache role members for 5 minutes, OOO for 1 minute. Pre-fetch on dashboard load. Cuts Workday calls from 10/sec to 0.5/sec.
- **Kafka.** Replace the `request_events` table with a proper Kafka topic. Notification, audit-tiering, and external integrations all become Kafka consumers. Decouples engine throughput from consumer throughput.

**Why this is enough:**
- 100k users × 5 req/week = 500k/week = ~71k/day = ~1/sec. Still small.
- Dashboard reads: 100k × 10/day = 1M/day = ~12/sec sustained. Redis handles this trivially.
- The bottleneck shifts from compute to *correctness under concurrent edits to the same request*.

**New cost:** ~$2-5k/month (bigger DB, replicas, Redis cluster, Kafka cluster, S3 storage).

#### Stage 4: 1M users (large enterprise; or you sell to multiple customers)

**What just broke:**

- 5000+ workflow definitions. Workflow store table is fine but listing all workflows in the admin UI takes 2 seconds.
- Different teams want different workflow authors with different permissions. You need RBAC on the workflow definition store itself.
- EU operations open. EU employee data must stay in EU. Your single-region Postgres no longer suffices.
- One customer (you're now selling this to other companies) is a hospital; HIPAA requires audit hash-chaining.
- 100k active requests/day; some need 30 different approvers (multi-step review for medical device approvals).

**What you add:**

- **Multi-region deployment.** Engine, DB, Redis, Kafka all per-region. Requests routed to the requester's home region. Cross-region approvals (US employee approving an EU contract) handled by a thin proxy that forwards the decision to the request's home region.
- **Workflow definition RBAC.** Workflows belong to "policy domains" (e.g., HR, Finance, Engineering). Domain admins can edit only their domain's workflows. Publishing a workflow requires approval (eat your own dog food: use the approval engine itself to approve workflow changes).
- **Audit hash chain.** Each audit event has `prev_hash = SHA256(prev_event.canonical_form)` and `hash = SHA256(this_event.canonical_form + prev_hash)`. Tampering with one event invalidates the chain forever after. Hash chain is verified on each audit query (or nightly job).
- **Workflow execution sharding by tenant (if multi-tenant).** Each customer's requests live in their own logical schema. Hot tenants get their own DB; long-tail tenants share a multi-tenant DB.
- **Long-workflow support.** A 30-step regulatory approval can take weeks. The engine must survive deploys mid-workflow. State lives entirely in DB; engine pods are interchangeable. Pause/resume support via a `paused_until` column on requests.
- **External event ingestion.** Code review workflows wait on CI status. The engine subscribes to a `external_event` topic; events come in tagged with `request_id`; the engine resumes the matching request.

**Why this is enough:**

Even at 1M users you only see ~10 requests per second peak. The system stays *organizationally* complex, not computationally. The architecture has not fundamentally changed since stage 3; it has only added: regions, RBAC layers, and tenant isolation.

**Cost:** ~$30-100k/month depending on region count and audit volume.

#### What you would do at 10M+ users

Honest answer: you would not. At 10M employees you are bigger than any single company. You are selling this as a SaaS product (Workday, ServiceNow, Coupa scale). At that point:

- Multi-tenant separation becomes the primary architectural concern, not approval throughput.
- The workflow engine becomes a configurable runtime (think AWS Step Functions, Temporal). You may even abandon your homegrown engine and adopt one of these.
- Audit becomes its own product surface (compliance dashboards, e-discovery support).

The point of the scaling journey: each stage is driven by *real* user pain, not anticipated load. Building stage-3 features at stage-1 scale is the most common mistake junior teams make.

### 10. The four real-world variants

Already covered in question.md Step 7. Recap with what the *implementation* looks like for each:

**A. Purchase Order.** The conditional branching uses `when: amount > 5000` evaluated against `request.inputs.amount`. The engine's expression language is a sandboxed subset of JavaScript or a custom mini-language (CEL is popular). Lint check at workflow-publish-time catches typos.

**B. Leave Request.** Parallel-with-quorum uses `type: parallel` plus `quorum: all`. Engine creates N tasks at the parallel step, waits until quorum is satisfied (all decided, or N-of-M approved, or any approved depending on quorum spec). Subtle: if one branch rejects and quorum is "all", request rejects immediately and other branches' tasks are cancelled (state changes to 'cancelled').

**C. Expense Report.** The "send back" feature uses an explicit transition `on_action(request_changes): return_to(requester)`. The requester sees the request in `state: returned_to_requester`. They edit and resubmit. The engine has `resume_from(<step_id>)` to skip already-completed steps. *Audit-critical:* the returned-and-resubmitted version is a new "iteration" of the same request, not a fresh request. The audit log shows the full back-and-forth.

**D. Code Review.** External-event integration: the engine subscribes to a CI status topic. On `ci.passed` for `request_id`, it advances the request. On `code.pushed`, it triggers `on_input_change: invalidate_approvals`, which clears all prior approve decisions for that request. Audit captures the invalidation.

The point: one engine, one data model, four workflows. The complexity is in the workflow definitions, not in the engine. That is the design victory.

### 11. Reliability

**Engine crashes mid-advance.** State is in DB. The `SELECT ... FOR UPDATE` ensures only one engine instance is advancing the same request at a time. If the engine crashes between operations within the transaction, the DB rolls back; no partial advance. After restart, the engine continues with the original `state`.

**Timeout / escalation worker fails.** A separate worker scans `tasks WHERE expires_at < NOW() AND state = 'pending'` every minute. If it dies, escalations are delayed by however long it takes to restart. Idempotent: the worker's only action is to record a `timeout` event, which the engine then handles. Re-running the worker on the same expired tasks does nothing harmful.

**Audit write fails.** The audit insert is in the same DB transaction as the state change. If audit fails, the state change rolls back. Never advance state without an audit record.

**Notification fails.** Notifications consume Kafka events. If the notifier dies, events queue. When it recovers, it resumes from its committed offset. Worst case: notifications are delayed but never lost.

**HRIS (Workday) goes down.** Org Service serves stale data from cache. Engine continues working; new requests assigned to whichever cached user matches the spec. After 1 hour of HRIS downtime, the engine starts logging warnings (cached data may be inaccurate). After 24 hours, the engine refuses to assign tasks to roles that need lookups (single-user assignments still work).

**Cross-region failure.** If one region is down, that region's requests are paused. Cross-region approvals routed via the down region are queued. After recovery, queued operations replay from Kafka. Requests do not need to fail; they just wait.

### 12. Observability

| Metric | Why it matters |
|--------|----------------|
| `requests.created.rate` | Sudden spike: someone wrote a bot. Sudden drop: auth or HRIS broken. |
| `requests.in_flight.count` | Slow accumulation = workflows are stalling on someone. |
| `request.cycle_time` p50/p95 per workflow | The headline SLO. Audit-required for many workflows. |
| `task.assignment_latency` p99 | If >1s, role resolver is sluggish. |
| `task.decision_latency` p99 | How long a human takes to decide. Useful for SLA dashboards. |
| `escalations.fired.rate` | Spikes when many tasks time out: bug in workflow or rampant absenteeism. |
| `delegation.depth.max` | Should never exceed 3-4. Higher = chains of out-of-office that need fixing. |
| `delegation.cycles.detected` | Should be 0. Page if non-zero. |
| `engine.lock_wait.p99` | If high, you need partitioning. |
| `audit.write.lag` | Audit must never lag; alert at >5s. |
| `workflow.publish.rate` | Spikes = workflow author is iterating; offer them a "draft" mode. |

Alerts:
- Page: engine error rate > 1% for 5 min; audit write failure (any); cycle detected.
- Ticket: cycle_time p95 regression >30%; delegation depth > 5; HRIS cache age > 30 min.

### 13. Common gotchas (the section that separates senior from mid-level)

1. **Delegation cycles.** Alice → Bob → Alice. Detected by passing a `visited` set. Fallback: assign to original target with a warning.
2. **Departed approvers.** Run a nightly job: find tasks where `assignee_id` no longer exists in HRIS, reassign to their last manager. Bulk departure (layoff) needs a one-shot rebalance tool.
3. **Conditional `when:` clauses with overlapping ranges.** A $5000 PO matches both `amount > 1000` and `amount > 5000`. Test workflows with boundary values before publishing.
4. **Self-approval.** Filter `approver != requester` in `resolve_approver`. Critical: someone in the finance role submitting a finance-approval request must not be allowed to approve themselves.
5. **Workflow version migration.** New version of a workflow does *not* affect in-flight requests on older versions. Requests are pinned to their version at creation.
6. **Notification storm.** A request that transitions through 5 states in 2 minutes sends 5 notifications. Batch: notification service aggregates events per recipient on a 60-second window.
7. **The "approve all" feature.** Carol's 80 leave requests for school holiday week. The bulk-approve endpoint serializes the decisions to maintain audit ordering. Returns a per-request success/failure status.
8. **OOO with no delegate.** Alice is OOO but has not set a delegate. Engine assigns the task to her anyway; the timeout fires after 48 hours and escalates to her manager. This is the default if the workflow does not specify `on_delegation: skip`.
9. **Concurrent decision on parallel tasks.** Two managers in parallel quorum-any. Both click Approve at the same instant. DB unique constraint on `decisions.task_id` (per task) lets both decisions land; the second `advance_request` call sees the first decision already advanced the state and gracefully does nothing.
10. **Backward transitions (send-back).** When the request goes back to the requester, pending tasks at downstream steps must be cancelled. The engine's "rewind to step X" function cancels all tasks at steps after X.

### 14. Follow-up answers

**1. Self-approval prevention.**

In `resolve_approver`, after computing the candidate, check: `if candidate == request.requester: raise SelfApprovalNotAllowed`. The engine then either falls back to the next eligible role member (for role-based steps) or fails the request with `auto_rejected_self_approval`.

Important: the check lives in the engine, not in the API. If you only check in the API ("is the caller the requester?") you miss the case where someone reassigns the task to the requester. Belt-and-suspenders: also check in the decision endpoint.

For role-based assignment, exclude requester from the eligible set when picking round-robin. For named-user assignment (`approver: "user_42"`) where user_42 is also the requester, the workflow author wrote a bug; surface it loudly with `WorkflowSelfApprovalError`.

**2. Approver leaves the company.**

A nightly job runs:

```sql
SELECT t.task_id, t.assignee_id, r.requester_id
FROM tasks t
JOIN requests r ON t.request_id = r.request_id
WHERE t.state = 'pending'
  AND t.assignee_id NOT IN (SELECT id FROM users WHERE active = true);
```

For each orphaned task, the job calls `reassign_to_fallback(task)` which:
1. Look up the original assignee's *last* manager from HRIS history.
2. If found, reassign and audit.
3. If not found (very long-departed user), assign to the workflow's `fallback_approver` from the workflow definition.
4. Notify the new assignee with context: "This task was originally assigned to X who has since left."

For mass departures (layoffs), run the job on-demand with a different mode: bulk-reassign to a single "layoff queue" approver who then triages.

**3. Delegation cycle.**

Already covered in question.md Step 5: the `follow_delegation` function carries a `visited` set and raises `DelegationCycle` if it sees a repeat. Mitigation: assign to the original target with a warning logged. The UI shows them a banner: "We detected a circular delegation chain; please update your OOO settings." Workflow does not stall.

Prevention: when a user sets their delegate, check that the chosen delegate's chain does not include the user themselves. Block at the OOO-setting endpoint, not just at runtime.

**4. Workflow version migration.**

In-flight requests stay on their original version forever. The `requests.workflow_version` column is set at create time and never updated. The engine fetches the matching version from `workflow_definitions`. Old versions are kept indefinitely (storage is cheap, audit requires this).

If you ship v4 with a bug fix and you *want* in-flight requests to benefit: do not modify v3. Ship a "patch" workflow that the operator manually applies to specific requests (rare; carries audit annotation `workflow_patched`).

Workflows can be `archived` (no new requests allowed) but never `deleted` while any historical request references them.

**5. Concurrent approval race.**

`UNIQUE` constraint on `decisions(task_id)` is the database-level guarantee that one task = one decision. Both `INSERT INTO decisions` attempts race; one wins, the other gets a unique-violation error.

The advance logic runs after each successful decision:
- First decision insert succeeds, advance_request acquires `SELECT ... FOR UPDATE` on requests row, sees state needs advance, advances.
- Second decision insert fails with unique-violation. The API returns 409 to the second user: "Task already decided."

In parallel-quorum-any, only one of N parallel tasks needs to be decided to advance. The first decision succeeds and advances; the other parallel tasks are then marked `cancelled` by the advance logic. If those other approvers were about to click Approve, their UI shows "Task no longer requires action" and the decision endpoint returns 410 Gone.

**6. Auto-approval bug.**

Detection: `metrics.auto_approve.rate` should be stable. A sudden 100x spike triggers an alert. Add `auto_approve.amount.sum` and alert on order-of-magnitude jumps.

Recovery: cannot literally unsign decisions, but you can re-open requests for review. The engine supports an `admin_reopen(request_id, reason)` endpoint that:
1. Creates a new audit event `request.admin_reopened`.
2. Resets state to a configurable point (often the first manual step).
3. Cancels any downstream effects (in this case, the downstream is "trigger payment in NetSuite"; you also need NetSuite to support reversing those transactions).

For the 50k fraudulent approvals: write a one-time script that calls `admin_reopen` on each, then notifies finance to chase the reversals downstream.

Prevention going forward: cap auto-approve rate per workflow per hour. If exceeded, switch to manual review with a circuit breaker. Auto-approve is only safe up to a small fraction of total throughput.

**7. Bulk import.**

Build a separate `bulk_import` endpoint that bypasses the engine's create-and-execute flow. It accepts a JSON file with all the historical data per request:

```json
{
  "workflow_id": "leave_request",
  "workflow_version": 1,
  "requester_id": "...",
  "inputs": {...},
  "events": [
    {"occurred_at": "2024-01-15T09:00:00Z", "type": "created", "actor": "..."},
    {"occurred_at": "2024-01-16T10:30:00Z", "type": "decision", "actor": "...", "decision": "approve"}
  ],
  "final_state": "approved",
  "finalized_at": "2024-01-16T10:30:00Z"
}
```

The import writes directly to `requests`, `tasks`, `decisions`, and `audit_log` with the original timestamps. It bypasses the engine's normal logic but tags every audit event with `source: bulk_import_<batch_id>` so a future auditor can distinguish imported from live data.

The audit chain (if hash-chaining is enabled) treats imported events as a separate chain; the live chain starts at the first non-imported event.

Permission: only admin users can call bulk_import. Heavily audited.

**8. Dashboard performance for power approvers.**

Carol's query `SELECT * FROM tasks WHERE assignee_id = 'carol' AND state = 'pending'` is slow because:

- The composite index `(assignee_id, state)` exists and is the right shape.
- But she has 120 tasks; each requires loading the parent `requests` row to display title, requester, age.
- The frontend issues 120 sub-queries.

Fix:
- Backend returns a denormalized "task list view" per assignee from the Read Service cache. Single Redis read returns all 120 task summaries with the data the dashboard needs.
- Cache is updated incrementally on `task.assigned` and `task.decided` Kafka events.
- Paginate the dashboard if she has >50 tasks; default to showing 20 with infinite scroll.

**9. Search across all approvals.**

Postgres JSONB query against `requests.inputs` for "Acme Corp in vendor field" is slow without a GIN index. Even with GIN, full-text search on JSON values is not great.

Right answer: pipe `requests` to a search index. Options:
- Elasticsearch/OpenSearch: full power, operational complexity. Best for >10M searchable requests.
- Postgres FTS with a `tsvector` column generated from interesting JSON paths: simple, works for <1M requests.
- ClickHouse with `inputs` flattened by a CDC pipeline: good if you also want analytics.

For most companies, Postgres FTS is enough. CREATE INDEX on `to_tsvector('english', inputs::text)`. The audit search ("show me all POs mentioning Acme") returns in <500ms.

**10. NetSuite integration with idempotency.**

The "approval finalized" event triggers a sync to NetSuite. NetSuite returns 5xx 1% of the time.

Pattern: a `netsuite-sync-worker` consumes `request.finalized` from Kafka. For each event:

1. Check `external_syncs` table: `(request_id, target = 'netsuite')`. If row exists with status `'success'`, skip (already synced).
2. Make the NetSuite API call with a deterministic `idempotency_key = "ns-{request_id}"`. NetSuite uses this to dedupe.
3. On 2xx: insert `external_syncs (request_id, target, status='success', external_id=<from response>)`. Update with idempotency key reused on retry.
4. On 5xx: leave the row in `status='pending'` with `last_attempt` and `retry_count`. Worker retries with exponential backoff: 1s, 5s, 25s, 2min, 10min, 30min, then deadletter.
5. On 4xx (validation error): mark `status='failed_permanent'`, alert a human.

NetSuite's `idempotency_key` ensures even if the worker calls twice (network blip after a successful response), only one NetSuite record is created.

**11. Watcher list notifications.**

The naive design fans out: 12 watchers × 8 state changes in 10 min = 96 notifications. A user with a few large purchases gets 200+ emails per week. Unsubscribes.

Aggregation pattern: the notification service buckets events per recipient per request on a 60-second window. At the end of the window, send one summary notification: "Your PO #1234 advanced from manager approval to finance approval (2 steps in 10 minutes)."

Configurable per user: instant (high-priority workflows), digest (1 email per hour), daily summary. Notification service stores user preferences; engine just emits raw events.

**12. The "approve all" button.**

API: `POST /api/v1/tasks/bulk_decide` with body `{"task_ids": [...], "decision": "approve", "comment": "Approved for school holiday."}`.

Implementation: server iterates serially through task_ids. For each:
- Reuses the standard `record_decision` flow.
- Captures success/failure per task.
- Returns a per-task result array.

Why serial not parallel: each decision potentially advances a request; advancing concurrently can cause version-skew. Serial is slower but correct. 80 decisions × 50ms = 4 seconds. Acceptable; user sees a progress bar.

What can go wrong: some tasks might have been decided by someone else (delegate) in the meantime. They fail with 409 Conflict; the per-task result shows this. User sees "78 approved, 2 already decided by Bob."

**13. Privacy / visibility scoping.**

Add a `visibility_scope` field to the workflow definition: `"public" | "private" | "department" | "role:<role>"`.

- `requests` table gets a denormalized `visibility_scope` column.
- All read APIs filter by visibility: a non-HR user querying audit gets no results for `visibility_scope = "role:hr-admin"`.
- Kafka events for restricted workflows go to a separate topic (`approval.events.restricted`) that only authorized consumers subscribe to.
- The notification service respects visibility: it does not include the request details in emails for restricted workflows; just "You have a pending approval. Log in to see details."

Audit access: even auditors must be scoped. A finance auditor cannot read HR-restricted audit events. The audit query API accepts a `scope` parameter and returns 403 for unauthorized scopes.

**14. Infinite-loop workflows.**

A workflow author writes step A → step B → step A. The first request through it transitions A → B → A → B forever.

Prevention at publish time: when a workflow draft is submitted for publication, run a graph traversal. Detect cycles. If a cycle exists *and* there is no `when:` condition that can break the cycle, refuse to publish. Show the author "Steps A → B → A form a cycle. Add a guard condition or change the transitions."

Some cycles are intentional (review → request changes → review). Those are allowed only if at least one transition in the cycle has a condition referencing input data, and the input can change (e.g., expense report goes back to requester who edits the amount).

Runtime safety net: max-transitions counter per request. If a request transitions >100 times, the engine pauses it and flags for human review.

**15. Multi-region with data residency.**

Architecture:
- Each region has full stack: Engine, DB, Redis, Kafka, Audit log.
- Requester's home region (from HRIS) determines which region the request lives in. Request is created there, audit is written there, never crosses borders for storage.
- Cross-region approval (US employee approves an EU PO): the EU region's engine generates the task and assigns it to the US employee. The US employee's dashboard (served from US region's read service) shows the task. When they decide, the decision is forwarded to EU region via an authenticated cross-region API call.
- The decision creates a row in the US region's `cross_region_decisions_outbox` table; a worker forwards it to EU and waits for confirmation. Idempotent retries handle network failures.
- The decision is *recorded* in the EU region's tables and audit. US region only has a "this US user sent a decision to EU region" record for their own audit (no PII about the EU request).

The lesson: data residency forces the design to keep the request as the unit of locality. The user can be anywhere; the request is somewhere specific.

### 15. Trade-offs and what a senior would mention

- **Why a custom engine, not Temporal / Cadence / AWS Step Functions.** Off-the-shelf workflow engines are great for general computation. They are bad for "approval-shaped" workflows because the role/delegation/audit story is approval-specific and would have to be built on top anyway. At very large scale (10M+ workflows) you might pick Temporal as the runtime; the approval logic stays in your service.
- **Why Postgres, not Cassandra/DynamoDB.** Approval needs strong consistency (no double-approve, no lost decisions). Postgres gives you ACID for free. Cassandra would force you to invent compensating logic.
- **Why YAML for workflow definitions, not Java/Python plugins.** YAML lets non-engineers author workflows. Plugins force every workflow change through a deploy. The trade-off: YAML expressiveness is limited. For the 5% of workflows that need real code (custom integration logic), the engine has an "exec step" type that runs sandboxed JavaScript.
- **Why an immutable audit log, not just relying on event-sourcing the requests.** Event-sourcing the requests would make audit queries fast but ties the application's data model to the audit format. Decoupling lets you keep the audit format stable across application refactors.
- **What I would revisit at 10x scale.** Move to Temporal as the underlying runtime; build the approval logic on top. Separate the workflow authoring product into its own service (with its own DB, RBAC, etc.). Move audit to a dedicated immutable-storage product (something like AWS QLDB or a hash-chained ledger).

### 16. Common interview mistakes

1. **Modeling requests with a `status` enum and hardcoded transitions.** The whole question is about the engine; if your `status` field is `pending | approved | rejected` and your transitions are `if status == 'pending' and approver clicks approve then status = 'approved'`, you have written CRUD, not a workflow service. The engine reads a *data-driven* definition.
2. **No version pinning on workflows.** If you let in-flight requests follow the latest workflow version, you introduce time-traveling bugs and break audit.
3. **Self-approval allowed by default.** A junior engineer designing this rarely thinks of it. The interviewer will ask. Have an answer.
4. **No audit log, or audit log is just "I logged it."** Audit is a feature, not a side effect. It has a schema, retention, immutability, queryability.
5. **Treating notifications as part of the engine.** They are not. Engine emits events. Notification service consumes. This separation is what makes it possible to add Slack/SMS/Teams without modifying the engine.
6. **Designing for write throughput.** Even at enterprise scale, you have 10 writes per second. The hard numbers are about read latency for dashboards and audit queries.
7. **Forgetting parallel approval with quorum.** Serial-only design works for 80% of workflows. The other 20% (which include most of the high-stakes ones: medical, legal, compliance) need parallel.
8. **Ignoring the org service.** The role/delegation/OOO machinery is half the system. Hand-waving it ("we'll just have a `manager_id` column") loses the design.
9. **No mention of multi-region or data residency.** At enterprise scale, GDPR alone forces multi-region. Demonstrating awareness lifts you to senior.
10. **Building it as a microservice "the right way" before understanding the workflow shape.** Start with the data model (workflow YAML, requests, tasks, decisions, audit). The service boundaries follow from that, not the other way around.

If you can hit 7 of these 10, you are interviewing at staff level. Most candidates miss the engine vs CRUD distinction, the audit immutability requirement, and the role-resolution depth. Those three are what separate a thoughtful approval design from a generic "design a system that tracks state" answer.
