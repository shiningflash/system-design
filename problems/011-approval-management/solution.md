## Solution: Approval Management Service

### The short version

An approval service is a small state machine, run many times, on many shapes of request. You read a recipe that says *"first the manager approves, then finance approves, then we are done,"* and you walk through it step by step. Each step is assigned to a real person. You wait for them to click Approve or Reject. You move on.

The runtime is tiny. What makes the problem interesting is everything around it:

- Turning *"the requester's manager"* into a real person when that person is on vacation.
- Surviving the approver leaving the company while the task is still pending.
- Stopping someone from approving their own request.
- Keeping an audit trail that holds up five years later, in court.

The data model fits on a napkin: **workflow definitions, requests, tasks, decisions, audit log.** Scale is not the hard part. At 100,000 employees, the whole company only generates about 1 request per second. The hard part is **organizational complexity**, not throughput.

---

### 1. The two questions that matter most

If you only get to ask two clarifying questions, ask these.

**One workflow, or many?** If it is just leave requests, you have a CRUD app. If it is a workflow engine that runs many types defined by config, you have a real system design.

**Who writes workflows?** If HR admins through a UI, you need a definition store, versioning, and a lint pass. If engineers in YAML in the repo, you skip half of that.

Everything else (vacation, escalation, audit, retention) follows from those two answers.

---

### 2. The math, in plain numbers

| Scale | Requests/day | Per second | Active in flight |
|-------|--------------|------------|------------------|
| 50-person startup | ~36 | ~0.0004 | 50 to 70 |
| 100k enterprise | ~71,000 | ~1 sustained, ~3 peak | ~200,000 |

The throughput is small. A single Postgres handles it. The numbers that matter:

- **200,000 active requests at any moment.** The "show me my pending approvals" query has to find the right ones in under 50ms.
- **5,000 different workflow types** at a mature company. Most are cold. A few (leave, expense) are hot.
- **Reads beat writes 25 to 1.** Every employee checks their dashboard ten times a day. Caching the read path matters more than write throughput ever will.

---

### 3. The API

Two endpoints carry the whole product. *Create a request* and *record a decision*. Everything else is reading data back.

```
POST /api/v1/requests
Idempotency-Key: <uuid>

{
  "workflow_id": "leave_request",
  "workflow_version": null,            # null = use the latest published
  "inputs": { "start_date": "...", "end_date": "...", "days": 5 }
}
```

You get back the request id and the tasks just created. Two parallel approvers? Two tasks. Auto-approved (under-3-day leave)? A finalized request immediately.

```
POST /api/v1/tasks/{task_id}/decisions

{
  "decision": "approve" | "reject" | "request_changes" | "delegate",
  "comment": "...",
  "on_behalf_of": "user_bob"           # set if a delegate is acting
}
```

Small but load-bearing choices:

- **Idempotency-Key is required on create.** Mobile retries on timeout. Without the key, you get duplicate requests and double-charged budget reservations.
- **`workflow_version: null` defaults to latest published.** Pin to a specific version for testing.
- **Tasks are the unit of work, not requests.** A request can have multiple tasks in flight (parallel approval). Approvers act on tasks.

Status codes worth knowing: **409** means the task was already decided (someone else, or a delegate, beat you). **410** means the task was cancelled because the request was withdrawn or returned to the requester.

---

### 4. The data model

Five tables. Two big, three small.

```mermaid
erDiagram
    workflow_definitions ||--o{ requests : "version pinned"
    requests ||--o{ tasks : has
    tasks ||--|| decisions : "exactly one"
    requests ||--o{ audit_log : "many events"

    workflow_definitions {
        text workflow_id
        int version
        text status
        jsonb definition
    }
    requests {
        uuid request_id
        text workflow_id
        int workflow_version
        text requester_id
        text state
        jsonb inputs
    }
    tasks {
        uuid task_id
        uuid request_id
        text assignee_id
        jsonb assigned_via
        text state
        timestamptz expires_at
    }
    decisions {
        uuid decision_id
        uuid task_id
        text actor_id
        text on_behalf_of
        text decision
    }
    audit_log {
        uuid event_id
        uuid request_id
        text event_type
        jsonb actor
        jsonb payload
        jsonb snapshot
    }
```

<details markdown="1">
<summary><b>Show: the full SQL</b></summary>

```sql
CREATE TABLE workflow_definitions (
    workflow_id      TEXT NOT NULL,
    version          INT NOT NULL,
    status           TEXT NOT NULL,        -- 'draft' | 'published' | 'archived'
    definition       JSONB NOT NULL,
    published_at     TIMESTAMPTZ,
    published_by     TEXT,
    PRIMARY KEY (workflow_id, version)
);

CREATE TABLE requests (
    request_id          UUID PRIMARY KEY,
    workflow_id         TEXT NOT NULL,
    workflow_version    INT NOT NULL,
    requester_id        TEXT NOT NULL,
    state               TEXT NOT NULL,
    inputs              JSONB NOT NULL,
    state_data          JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finalized_at        TIMESTAMPTZ,
    final_state         TEXT,
    idempotency_key     TEXT,
    FOREIGN KEY (workflow_id, workflow_version)
        REFERENCES workflow_definitions(workflow_id, version)
);
CREATE INDEX idx_req_requester ON requests (requester_id, created_at DESC);
CREATE INDEX idx_req_open      ON requests (state) WHERE finalized_at IS NULL;

CREATE TABLE tasks (
    task_id        UUID PRIMARY KEY,
    request_id     UUID NOT NULL REFERENCES requests(request_id),
    step_id        TEXT NOT NULL,
    assignee_id    TEXT NOT NULL,
    assigned_via   JSONB,                  -- delegation chain
    state          TEXT NOT NULL,          -- 'pending' | 'decided' | 'cancelled' | 'expired'
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at     TIMESTAMPTZ
);
CREATE INDEX idx_tasks_assignee_pending
    ON tasks (assignee_id) WHERE state = 'pending';

CREATE TABLE decisions (
    decision_id    UUID PRIMARY KEY,
    task_id        UUID NOT NULL REFERENCES tasks(task_id),
    request_id     UUID NOT NULL REFERENCES requests(request_id),
    actor_id       TEXT NOT NULL,
    on_behalf_of   TEXT,
    decision       TEXT NOT NULL,
    comment        TEXT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (task_id)                       -- one decision per task
);

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

</details>

Three small things doing real work:

**`UNIQUE (task_id)` on decisions.** Two managers click Approve at the same instant. The database serializes them. First INSERT wins. Second fails with a unique-violation. The API turns that into a friendly 409. The request still advances exactly once. No application-level locks needed.

**`workflow_definitions` is immutable per version.** Publishing v4 of `leave_request` does not touch v3. In-flight requests stay on v3 forever. This is what lets you ship workflow changes without breaking running requests, and what makes audit replay work years later.

**`audit_log` has no foreign key to `requests`.** On purpose. If a request is ever deleted (GDPR, mistaken bulk import), the audit must survive. Audit is the truth-of-record, not the requests table.

Why Postgres, not Cassandra? The system is small and ACID matters here (claim a task, record a decision, advance the request, all in one go). Postgres gives you that in one box.

---

### 5. The engine

The engine is a state machine executor. One function does most of the work. It is called every time something might let a request progress: a request was just created, a decision came in, a timeout fired, or an external event arrived.

```mermaid
stateDiagram-v2
    direction LR
    [*] --> Created
    Created --> Resolving: pick approver
    Resolving --> Pending: task assigned
    Pending --> Pending: waiting (quorum not met)
    Pending --> Advancing: decision recorded
    Advancing --> Resolving: next step exists
    Advancing --> Finalized: terminal step
    Pending --> Rewound: returned to requester
    Rewound --> Resolving: resubmit
    Finalized --> [*]
```

<details markdown="1">
<summary><b>Show: the advance function</b></summary>

```python
def advance_request(request_id):
    with db.transaction():
        request = db.lock_for_update("requests", request_id)
        if request.finalized_at is not None:
            return                                  # already done

        workflow = workflow_store.get(request.workflow_id, request.workflow_version)
        current_step = workflow.find_step(request.state)

        if current_step.type == "approval":
            tasks = db.query_tasks(request_id, current_step.id)
            if not quorum_met(tasks, current_step.quorum):
                return                              # still waiting on humans

        next_step = workflow.next_step(current_step, request.inputs, request.state_data)

        if next_step.type == "terminal":
            db.finalize(request_id, state=next_step.state)
            emit("request.finalized", request_id)
            return

        if next_step.type == "approval":
            for who in resolve_assignees(next_step, request.requester):
                db.insert_task(request_id, next_step.id, who, expires_at=...)
                emit("task.assigned", task_id)

        elif next_step.type == "auto":
            outcome = execute_auto_step(next_step, request)
            db.update_state(request_id, outcome.state)
            return advance_request(request_id)      # auto-steps chain

        db.update_state(request_id, next_step.id)
        emit("request.advanced", request_id, new_state=next_step.id)
```

</details>

Three things make this safe:

- The whole advance happens in one DB transaction. Crashes mid-way roll back. After restart, the engine sees the same state and tries again. No partial advance ever lands on disk.
- The `SELECT ... FOR UPDATE` lock on the request row serializes concurrent attempts to advance the same request. Without it, two simultaneous decisions could each push the request forward and corrupt the state machine.
- The function is idempotent at step boundaries. Calling it twice on the same request either sees the same state and proceeds, or sees that someone else already advanced and does nothing.

---

### 6. Resolving an approver

The workflow says `approver: "{{ employee.manager }}"`. The engine must turn that into a real person before assigning a task. That sounds simple. Production says otherwise. The full algorithm is in `question.md` Step 8. The key ideas:

- Cache role membership for 5 minutes. Cache OOO state for 1 minute. Cuts HRIS load by 100x.
- When you follow a delegation chain (Bob OOO → Carol OOO → Dave), store the full chain on the task as `assigned_via`. Dave's UI then shows *"approving on behalf of Bob, via Carol."*
- The `when` parameter to `resolve_approver` is what lets audit replay work years later. You need a point-in-time view of the org chart.

---

### 7. The architecture

```mermaid
flowchart TB
    subgraph Edge["Client edge"]
        C([Web / Mobile / Slack]):::user
        GW["API Gateway<br/>(auth · idempotency · rate limit)"]:::edge
    end

    subgraph WritePath["Synchronous write path"]
        E["Workflow Engine<br/>(stateless pods)"]:::app
        O["Org Service<br/>HRIS adapter + cache"]:::ext
        Defs[("Workflow<br/>Definitions")]:::db
    end

    DB[("Postgres<br/>requests · tasks ·<br/>decisions · audit (90d hot)")]:::db

    subgraph ReadPath["Dashboard read path"]
        R["Read Service"]:::app
        Cache[("Redis")]:::cache
    end

    K{{"Kafka<br/>approval.request.created<br/>approval.task.assigned<br/>approval.decision.recorded<br/>approval.request.finalized"}}:::queue

    subgraph Consumers["Async consumers"]
        N["Notification Service"]:::app
        I["Downstream<br/>NetSuite, Workday"]:::ext
        Cold[("Audit cold tier<br/>S3 + Athena · 7yr")]:::db
    end

    C --> GW
    GW -->|create| E
    GW -->|read| R
    E --> Defs
    E --> O
    E --> DB
    R --> Cache
    R -.miss.-> DB
    DB -->|CDC / outbox| K
    K --> N
    K --> I
    K --> Cold

    classDef user fill:#dbeafe,stroke:#1e40af,color:#1e3a8a
    classDef edge fill:#e2e8f0,stroke:#475569,color:#1e293b
    classDef app fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef db fill:#fed7aa,stroke:#c2410c,color:#7c2d12
    classDef cache fill:#fecaca,stroke:#b91c1c,color:#7f1d1d
    classDef queue fill:#ddd6fe,stroke:#6d28d9,color:#4c1d95
    classDef ext fill:#e9d5ff,stroke:#7e22ce,color:#581c87
```

Five things to notice:

- The engine does not call out to other services on the write path, except the Org Service (cached aggressively). Everything else is a Postgres transaction.
- Notifications, integrations, and audit archival are downstream of Kafka. Not on the write path. If the notification service dies, requests still get created and approved. Emails just queue up.
- The Read Service is denormalized. It listens to Kafka and maintains a "what is pending for each user" Redis cache. Dashboards never touch the primary DB in the common case.
- Audit lives in two tiers. Last 90 days in Postgres. Older in S3 Parquet, queried via Athena. A nightly job moves rows between tiers.
- Engine pods are stateless. State is in Postgres. Roll them in the middle of the day with zero impact.

---

### 8. A request, end to end

```mermaid
sequenceDiagram
    autonumber
    participant Alice
    participant GW as API Gateway
    participant E as Engine
    participant O as Org Service
    participant DB as Postgres
    participant K as Kafka
    participant N as Notification Svc

    Alice->>GW: POST /requests (5 days off)
    GW->>E: forward (auth ok, idempotency checked)
    E->>O: who is Alice's manager?
    O-->>E: Bob (in office)

    rect rgb(241, 245, 249)
        Note over E,DB: one database transaction
        E->>DB: INSERT request
        E->>DB: INSERT task (assignee=Bob)
        E->>DB: INSERT audit_log
        E->>DB: COMMIT
    end

    E->>K: emit request.created
    E->>K: emit task.assigned
    E-->>GW: 201 Created
    GW-->>Alice: 201 Created
    K->>N: task.assigned event
    N-->>Bob: email + Slack
```

Recording a decision looks almost the same. The engine fetches the task, checks the caller, inserts a row in `decisions` (the unique constraint catches double-decide races), updates the task, writes an audit event, and calls `advance_request` to see if the workflow can move forward. All in one transaction.

Reads have their own short flow. Client → API gateway → Read Service → Redis check (`user:{uid}:pending_tasks`) → return. Cache misses fall through to a Postgres read replica.

Target latencies, roughly:

| Operation | P99 |
|-----------|-----|
| Create request | ~200ms (bottleneck: role resolver when cache is cold) |
| Record decision | ~150ms (role usually warm) |
| Dashboard read | ~50ms |

---

### 9. The scaling journey: 10 users to 1 million

At every stage, name what just broke and what fixes it. Build nothing preemptively.

```mermaid
flowchart LR
    S1["Stage 1<br/>10–100 users<br/>1 Postgres + 1 pod<br/>~$80/mo"]:::s1
    S2["Stage 2<br/>~1,000 users<br/>+ workflow store<br/>+ HRIS<br/>~$250/mo"]:::s2
    S3["Stage 3<br/>10k–100k users<br/>+ read replicas<br/>+ Redis · Kafka<br/>+ S3 audit<br/>~$2–5k/mo"]:::s3
    S4["Stage 4<br/>1M users<br/>+ multi-region<br/>+ hash chain audit<br/>+ workflow RBAC"]:::s4

    S1 --> S2 --> S3 --> S4

    classDef s1 fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef s2 fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef s3 fill:#fef3c7,stroke:#a16207,color:#713f12
    classDef s4 fill:#fce7f3,stroke:#be185d,color:#831843
```

#### Stage 1: 10 to 100 users

One Postgres, one app instance. Workflow YAML lives in the repo. Manager mappings hardcoded. No cache, no queue. Notifications are inline HTTP calls to SendGrid. ~$80/month. Two weeks to ship.

Enough because you see ten requests a day. Building more is over-engineering.

#### Stage 2: 1,000 users

Something breaks: someone asks for a new workflow type without a deploy.

Move workflow YAML out of code into a `workflow_definitions` Postgres table with a minimal admin UI. Add versioning while you are in there. Integrate with the HRIS (Workday or BambooHR) instead of hardcoded manager mappings. 5-minute cache on HRIS responses. Notifications stop being inline and start consuming a small `request_events` table. ~$250/month.

Still no Kafka, no read replicas, no Redis. One request every couple of minutes. One Postgres is still fine.

#### Stage 3: 10,000 to 100,000 users

Several things break at once:

- The dashboard for power approvers takes 4 seconds. Carol has 120 pending tasks.
- Audit queries on 80M rows time out.
- Engine pods compete for the same `SELECT FOR UPDATE` lock under bursty load.
- The Workday API rate-limits you during peak hours.

Fixes, in order:

- Two Postgres read replicas. Dashboards read from them, engine writes go to primary. ~1s replication lag is fine.
- A Redis cache for "my pending tasks" populated by listening to engine events.
- Move audit older than 90 days to S3 Parquet. Athena for ad-hoc queries.
- Partition engine pods by `request_id` hash. Stops them racing for the same locks.
- Tighten Org Service cache. Pre-fetch on dashboard load.
- Bring in Kafka properly. Notifications, audit tiering, and external integrations all become consumers.

Cost jumps to $2-5k/month.

#### Stage 4: 1 million users

New problems:

- EU operations open. EU data must stay in EU.
- One customer is a hospital. HIPAA needs hash-chained audit.
- 5,000+ workflow definitions. Different teams want different RBAC.
- Some workflows have 30 steps and live for weeks.

Multi-region everything. Engine, DB, Redis, Kafka all per-region. The requester's home region (from HRIS) decides where the request lives and where audit is written. Cross-region approvals (US person approves EU request) routed via authenticated cross-region API.

Workflow definitions get RBAC by "policy domain." Audit events get a hash chain. Long-running workflows survive engine deploys because state lives entirely in DB.

The architecture has not fundamentally changed since Stage 3. You added regions, RBAC, and tenant isolation. The core data model is still the same one you wrote in Stage 1.

#### What you would do at 10M users

You wouldn't. By then you are bigger than any one company, which means you are selling this as SaaS (Workday, ServiceNow scale). At that point you might swap your homegrown engine for Temporal as the runtime and keep your approval-specific logic on top.

---

### 10. The four variants, fast

Same engine. Four real workflows. Each one stresses a different feature.

- **Purchase order** uses conditional branching. `when: amount > 5000` sends to finance. `when: amount > 25000` adds the CFO. Lint your conditions or a $25,001 PO needlessly goes to both finance and the CFO.
- **Leave request** uses parallel with quorum. Long leaves go to HR and grandboss at the same time. Both must approve. If one rejects, the other's task is cancelled and the request rejects.
- **Expense report** needs backward transitions. Finance says "missing receipt." The request rewinds to the requester. The engine has `on_action: return_to(<step>)` for this. Subtle: pending downstream tasks must be cancelled when the request rewinds.
- **Code review** needs external-event integration. The workflow waits for CI status, not just a human. When a new commit lands, prior approvals get invalidated.

One engine, four workflows, no special cases. That is the design victory.

---

### 11. Reliability

Engine crashes mid-advance roll back cleanly. State is in the DB. The transaction is atomic. The lock prevents concurrent advance attempts. After restart, the engine sees the request's current state and continues.

The escalation worker is separate from the engine. It scans for `expires_at < now()` and records timeout events. If it dies, escalations get delayed but never lost.

Audit writes happen in the same transaction as the state change. If the audit insert fails, the state change rolls back. You never advance state without an audit record.

If HRIS goes down, the Org Service serves stale data from cache. The engine keeps working with whoever was cached. After an hour, it logs warnings. After 24 hours, it refuses to assign tasks that need role lookups.

If a region goes down, that region's requests pause. Cross-region approvals routed through the down region queue in Kafka. When the region comes back, queued operations replay.

---

### 12. Observability

| Metric | Why it matters |
|--------|----------------|
| `requests.created.rate` | Spike means a bot. Drop means auth or HRIS broken. |
| `requests.in_flight` | Growing slowly means approvals are stalling on someone. |
| `request.cycle_time` p50/p95 per workflow | The headline SLO. Often audit-required. |
| `task.assignment_latency` p99 | High means the role resolver is sluggish. |
| `escalations.fired.rate` | Spike means lots of stalled tasks. Could be a workflow bug. Could be flu season. |
| `delegation.depth.max` | Should never exceed 3-4. Higher means OOO chains that need fixing. |
| `delegation.cycles.detected` | Should be 0. Page if non-zero. |
| `engine.lock_wait` p99 | High means you need partitioning across pods. |
| `audit.write.lag` | Audit must never lag. Alert at >5s. |

Page on: engine error rate >1% over 5min. Audit write failure (any). Delegation cycle detected.

Ticket on: cycle time regression >30%. HRIS cache age >30 min.

---

### 13. Follow-up answers

These are the questions a senior interviewer is listening for. Each answer is short on purpose. The depth is in the *why*.

**1. Self-approval.**

Filter the requester out of the eligible approver set inside `resolve_approver`. For role-based steps, pick the next available role member. For named-user steps where the user happens to be the requester, raise a workflow definition error. The check lives in the engine, not just in the API, because someone could otherwise reassign a task to the requester through delegation.

**2. Approver leaves the company.**

A nightly job:

```sql
SELECT t.task_id, t.assignee_id, r.requester_id
  FROM tasks t JOIN requests r ON t.request_id = r.request_id
 WHERE t.state = 'pending'
   AND t.assignee_id NOT IN (SELECT id FROM users WHERE active);
```

For each orphan, reassign to the original assignee's last manager from HRIS history. If not findable, fall back to the workflow's `fallback_approver`. Notify the new assignee: *"this task was originally assigned to X who has since left."* For layoffs, run the job on demand with a bulk-reassign mode.

**3. Delegation cycle.**

The `visited` set in `follow_delegation` raises `DelegationCycle` on a repeat. Assign to the original target anyway with a warning in the UI. Better still: when a user *sets* their delegate, walk the chain and refuse if it includes them. Catch it at the source, not at runtime.

**4. Workflow version migration.**

In-flight requests stay on their original version forever. `requests.workflow_version` is set at create time and never updated. Old versions are kept indefinitely. If you ship v4 with a critical fix and want in-flight requests to benefit, you do not modify v3. You manually apply a "patch" to specific requests (rare; carries audit annotation `workflow_patched`). Workflows can be archived but never deleted while any historical request references them.

**5. Concurrent approval race.**

`UNIQUE (task_id)` on `decisions` is the lock. Two approvers click at once. Two INSERTs race. The first wins. The second gets a unique-violation. The API turns that into 409. The first decision's `advance_request` runs, sees the quorum is satisfied, and cancels the sibling tasks. The losing approver sees "task no longer requires action" (410 Gone). No application-level coordination needed. The database does the work.

**6. Auto-approval bug.**

Detection: `metrics.auto_approve.rate` is normally stable. A 100x spike triggers a page.

Recovery: an `admin_reopen(request_id)` endpoint creates an `audit.admin_reopened` event, resets state to a configurable point, and cancels downstream effects. The downstream is whoever consumes the "approved" event. If NetSuite already wrote a payment record, NetSuite needs its own reversal.

Prevention: cap auto-approve rate per workflow per hour. A circuit breaker switches to manual review on overrun.

**7. Bulk import.**

A separate `bulk_import` endpoint bypasses the engine and writes directly to `requests`, `tasks`, `decisions`, and `audit_log` with the original timestamps. Each audit event tagged with `source: bulk_import_<batch_id>` so future auditors can tell imported from live. The audit hash chain (if enabled) treats imported events as a separate chain. Admin-only. Heavily audited itself.

**8. Slow dashboard.**

Carol's 120 pending tasks make her dashboard slow because the frontend issues 120 sub-queries hydrating each task.

Fix: the backend returns a denormalized "task list view" per assignee from the Read Service cache. Single Redis read. Cache updated on `task.assigned` and `task.decided` events. Paginate if she has more than 50 tasks.

**9. Search across all approvals.**

Postgres JSONB search on `requests.inputs` is slow without help. For most companies, a `tsvector` GIN index on `to_tsvector('english', inputs::text)` is enough. Above ~10M searchable requests, pipe `requests` to Elasticsearch or ClickHouse via CDC.

**10. NetSuite integration.**

A `netsuite-sync-worker` consumes `request.finalized` from Kafka. For each event: check an `external_syncs (request_id, target)` table. If status is `success`, skip. Otherwise call NetSuite with `idempotency_key = "ns-{request_id}"` (NetSuite uses it to dedupe). On 2xx, mark success. On 5xx, exponential backoff (1s, 5s, 25s, 2m, 10m, 30m, deadletter). On 4xx, mark `failed_permanent` and alert a human.

**11. Notification storm.**

60-second aggregation window per recipient per request. Send one digest. Configurable per user (instant, hourly, daily). The notification service owns preferences. The engine just emits raw events.

**12. The "approve all" button.**

API: `POST /api/v1/tasks/bulk_decide` with `{"task_ids": [...], "decision": "approve"}`. Server iterates serially (each decision might advance a request and concurrent advances cause version skew). 80 decisions at 50ms each = 4 seconds. Returns a per-task result array. Some tasks might 409 if a delegate beat the user. The result shows that.

**13. Privacy and visibility scoping.**

Workflow definitions get a `visibility_scope` field: `public | private | department | role:<role>`. Denormalized to `requests.visibility_scope`. All read APIs filter by it. Restricted workflows publish to a separate Kafka topic (`approval.events.restricted`) that only authorized consumers subscribe to. Notifications for restricted workflows omit details: *"you have a pending approval; log in to see."*

**14. Infinite-loop workflow.**

Publish-time check: graph-traverse the workflow definition. If there is a cycle without any `when:` condition that can break it, refuse to publish. Some cycles are intentional (review → request changes → review), allowed only if at least one transition has a guard referencing input data that can change. Runtime safety net: max-transitions counter per request. More than 100 transitions pauses the request for human review.

**15. Multi-region with data residency.**

Each region has its full stack. The requester's home region (from HRIS) decides where the request lives. Cross-region approvals: the EU region creates the task and assigns to the US employee, who sees it via the US region's dashboard. When the US employee decides, the decision is forwarded to EU via an authenticated cross-region API call (queued in a `cross_region_decisions_outbox` table and retried idempotently). The decision is recorded in the EU region's tables and audit. The US region keeps only a *"this US user sent a decision to EU region"* record for its own audit. No PII about the EU request.

The lesson: data residency forces the request to be the unit of locality. The user can be anywhere. The request is somewhere specific.

---

### 14. Trade-offs worth saying out loud

**Why a custom engine and not Temporal or Step Functions.** Off-the-shelf workflow engines are great for general computation. They are awkward for approval-shaped workflows because the role, delegation, and audit story is approval-specific and would have to be built on top anyway. At 10M+ workflows you might pick Temporal as the runtime. The approval logic stays in your service either way.

**Why Postgres and not Cassandra.** Approval needs strong consistency. No double-approve. No lost decisions. Postgres gives ACID for free. Cassandra would force you to invent compensating logic for the same guarantees.

**Why YAML for workflow definitions and not code.** YAML lets non-engineers author workflows through a UI. Code forces every workflow change through a deploy. The cost is limited expressiveness. For the few workflows that need real code (a custom integration step), the engine has an `exec` step that runs sandboxed JavaScript.

**Why an immutable audit log and not just event-sourcing the requests.** Event-sourcing would make audit queries fast but ties the application's data model to the audit format. Decoupling lets you refactor the application without breaking audit. Worth the duplication.

**What you would revisit at 10x scale.** Adopt Temporal as the runtime. Move workflow authoring into its own product with its own DB and RBAC. Move audit to a dedicated immutable-storage product (QLDB, or a hash-chained ledger).

---

### 15. Common mistakes

Most weak answers fall into one of these:

**Modeling requests with a hardcoded status enum.** If your design has `status: pending | approved | rejected` and transitions live in if-statements, you have written CRUD, not a workflow service. The whole point is a data-driven engine.

**No version pinning on workflows.** If in-flight requests follow the latest version, you introduce time-traveling bugs and break audit replay.

**Self-approval allowed by default.** Junior candidates rarely think of it. The interviewer will ask.

**Audit as a side effect.** *"I'll just log it."* Audit has a schema, a retention policy, an immutability contract, and a query interface. It is a feature, not a log line.

**Treating notifications as part of the engine.** They are not. The engine emits events. The notification service consumes them. This separation is what lets you add Slack, Teams, or SMS without touching the engine.

**Designing for write throughput.** Even at enterprise scale you have ten writes per second. The hard numbers are about read latency for dashboards and audit queries.

**Forgetting parallel approval with quorum.** Serial-only works for 80% of workflows. The other 20% (medical, legal, compliance) need parallel. Retrofitting later is expensive.

**Hand-waving the org service.** The role, delegation, and OOO machinery is half the system. Treating it as a single `manager_id` column loses the design.

**No mention of multi-region or data residency.** GDPR alone forces it at enterprise scale.

**Microservices "the right way" before understanding the workflow shape.** Start from the data model. Service boundaries follow from that, not the other way around.

If you can name seven of these ten without prompting, you are interviewing at staff level. The three that separate strong from average answers: engine versus CRUD, audit immutability, and role-resolution depth. Those are the answers a senior architect listens for.
