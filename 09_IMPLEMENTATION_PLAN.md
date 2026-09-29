# Implementation Plan

## Objective

Build the platform incrementally so each phase delivers a testable vertical slice.

Do not implement all 20 specialists before the first working flow.

---

## Phase 0 — Foundation

### Deliverables

- Python project structure
- LangGraph installed and minimal graph running
- Pydantic domain package
- Pydantic Settings
- internal model adapter
- model profiles
- structured-output helper
- MongoDB connection/repository abstraction
- workflow/session IDs
- logging/tracing foundation
- MCP client wrappers
- permissions/tool registry

### Exit Criteria

```text
one LangGraph workflow can:
- accept typed input
- call a local model
- return validated Pydantic output
- persist/resume state
- call one MCP read tool
```

---

## Phase 1 — PRD to Validated Tests MVP

Implement only:

```text
Requirement Intake
Requirement Analysis
Test Generation
Coverage Validation
Mongo staging
```

Integrations:

```text
Confluence MCP
Jira read if needed
MongoDB
local model
```

Workflow:

```text
PRD -> analyze -> generate -> validate
                         ^        |
                         +--revise+
```

Add HITL for critical ambiguity.

### Exit Criteria

- PRD retrieved by URL/reference
- structured requirements stored
- test suite generated
- independent validator maps requirements to tests
- revision loop works
- validated suite stored in Mongo
- no Zephyr dependency in request latency

---

## Phase 2 — Zephyr Async Publication

Implement:

```text
Test Quality Gate v0
Zephyr Sync Worker
sync_jobs
idempotency
retry/dead-letter
UI sync status
```

At this stage duplicate checking can be basic.

### Exit Criteria

- main test-generation workflow finishes without waiting for Zephyr
- worker publishes staged tests
- retries do not create duplicates
- Mongo stores Zephyr IDs
- failed sync is visible and recoverable

---

## Phase 3 — Graphify Capability Layer

Implement:

```text
Graphify server
repository indexing
capability docstring standard
Capability Discovery Specialist
Test Intent Specialist
CapabilityResolutionMap
```

Select a small set of existing automation functions and wrap them as reference capabilities.

Example:

```text
create_new_case
send_message
verify_status
```

### Exit Criteria

- Graphify query finds approved capability by natural-language intent
- capability metadata is returned as Pydantic contract
- missing capability is detected reliably
- agent does not need full repository context

---

## Phase 4 — Test Composition

Implement:

```text
Test Composition Specialist
Composition Validation Specialist
generated test workspace
static validation
```

Do not yet automate missing-capability code development if it slows the MVP.

### Exit Criteria

- natural-language request can be composed from existing capabilities
- generated test passes syntax/static validation
- missing capability routes cleanly to a gap object

---

## Phase 5 — Missing Capability Engineering

Implement:

```text
Missing Capability Specialist
Jira ticket generation
Automation Development Specialist
Capability Contract Validator
Automation Code Review Specialist
Human PR Gate
Graphify Update Worker
resume original workflow
```

### Exit Criteria

- a missing capability creates a precise ticket
- coding specialist creates PR
- contract validator verifies agent-facing wrapper contract
- human approves merge
- Graphify updates
- original test workflow resumes and discovers the new capability

---

## Phase 6 — Environment and Execution

Implement:

```text
Environment Requirement Specialist
Environment Matching Specialist
Environment Snapshot
Test Execution Specialist
Runtime Validation Specialist
```

Start environment management as read-only.

Do not begin with autonomous configuration.

### Exit Criteria

- test declares environment requirements
- platform selects a valid environment
- test executes
- evidence stored
- deterministic validation used when possible
- multimodal validation works for one defined UI scenario

---

## Phase 7 — Controlled Environment Changes

Add:

```text
Environment Configuration Specialist
allowlisted low-risk operations
before/after snapshots
HITL for shared/high-risk changes
rollback metadata
```

### Exit Criteria

- configurable mismatch can be fixed safely
- workflow verifies post-change state
- risky changes interrupt for approval
- audit trail is complete

---

## Phase 8 — Failure Triage and Release Quality

Implement:

```text
Failure Triage Specialist
duplicate/similarity pipeline
Test Quality & Release Specialist
publication policies
```

### Exit Criteria

- failed execution is classified
- automation defect routes to engineering
- environment issue routes to environment flow
- product defect requires human approval
- duplicate candidates prevent blind Zephyr creation

---

## Phase 9 — Contextual QA Chat

Implement chat after core state exists.

The chat should leverage:

```text
session context
current test
workflow state
environment state
execution history
Graphify
Confluence/Jira/Zephyr read tools
```

### Exit Criteria

- resolves references such as "this test"
- explains current state
- answers why environment is unsuitable
- routes action requests rather than executing privileged actions directly

---

## Phase 10 — Parallelism and Performance

Add parallelism only after correctness.

Priority opportunities:

1. process independent test cases in parallel
2. capability discovery + duplicate pre-search
3. PR validation checks
4. runtime evidence analysis
5. candidate environment state reads

Add concurrency limits.

---

## Phase 11 — Production Hardening

Implement:

- role-based authorization
- tool credential isolation
- audit logs
- workflow replay/debugging
- prompt-injection defenses
- queue/dead-letter operational dashboards
- schema migration
- backup/recovery
- load tests
- security review
- HA strategy
- disaster recovery
- version compatibility matrix

---

## Suggested Repository Structure

```text
qa-agent-platform/
|
+-- app/
|   +-- api/
|   +-- ui_backend/
|
+-- orchestration/
|   +-- main_graph.py
|   +-- teams/
|       +-- requirements/
|       +-- composition/
|       +-- engineering/
|       +-- environment/
|       +-- quality/
|
+-- agents/
|   +-- requirements/
|   +-- composition/
|   +-- engineering/
|   +-- environment/
|   +-- quality/
|   +-- chat/
|
+-- contracts/
|   +-- requirements.py
|   +-- tests.py
|   +-- capabilities.py
|   +-- environment.py
|   +-- execution.py
|   +-- approvals.py
|
+-- tools/
|   +-- confluence/
|   +-- jira/
|   +-- zephyr/
|   +-- graphify/
|   +-- environment/
|   +-- execution/
|
+-- services/
|   +-- mongo/
|   +-- sessions/
|   +-- permissions/
|   +-- model_gateway/
|
+-- workers/
|   +-- zephyr_sync/
|   +-- graphify_update/
|
+-- policies/
|   +-- permissions.py
|   +-- hitl.py
|   +-- retries.py
|   +-- environment.py
|
+-- prompts/
|   +-- versioned/
|
+-- tests/
|   +-- unit/
|   +-- integration/
|   +-- agent_evals/
|   +-- workflow/
|
+-- config/
```

---

## Implementation Priorities

The first engineering milestone should prove this:

```text
PRD
 -> generated tests
 -> independent coverage validation
 -> Mongo
 -> asynchronous Zephyr publish
```

The second should prove this:

```text
Natural-language test
 -> Graphify capability discovery
 -> test composition
 -> static validation
```

The third should prove this:

```text
test
 -> environment match
 -> execution
 -> runtime validation
```

Everything else builds on these three vertical slices.
