# MongoDB and Zephyr Test Lifecycle

## 1. Why MongoDB Is More Than a Buffer

MongoDB is the internal operational source of truth while a test is being created, validated, automated, executed, and quality-gated.

Zephyr is the official managed test repository, but publication should happen only when the test is ready.

This avoids:

- slow synchronous Zephyr writes
- publishing tests that later prove invalid
- duplicate test creation
- blocking the agent workflow
- losing intermediate state
- coupling workflow availability to Zephyr latency

---

## 2. Test Lifecycle

Recommended high-level statuses:

```text
CREATED
REQUIREMENTS_VALIDATED
TEST_GENERATED
COVERAGE_VALIDATED
STAGED
CAPABILITIES_RESOLVED
AUTOMATION_COMPOSED
COMPOSITION_VALIDATED
ENVIRONMENT_READY
EXECUTION_PENDING
RUNNING
RUNTIME_PASSED
RUNTIME_FAILED
INCONCLUSIVE
QUALITY_GATE
READY_TO_PUBLISH
HOLD
REJECTED
PUBLISHING
PUBLISHED
SYNC_FAILED
```

Do not force one linear enum to represent every sub-workflow if it becomes too complex.

A better implementation may use:

```text
design_status
automation_status
execution_status
release_status
sync_status
```

---

## 3. Recommended MongoDB Collections

### `sessions`

```text
session_id
user_id
current_project
current_prd_ref
current_test_id
current_environment_id
current_execution_id
created_at
updated_at
```

### `requirements`

```text
requirement_set_id
source_type
source_ref
source_version
raw_snapshot_ref
structured_model
analysis_status
schema_version
created_at
updated_at
```

### `test_cases`

```text
test_id
requirement_set_id
title
objective
preconditions
steps
expected_result
traceability
design_status
automation_status
release_status
zephyr_test_id
duplicate_candidates
schema_version
created_at
updated_at
```

### `capability_resolutions`

```text
resolution_id
test_id
graph_version
resolved_actions
missing_actions
ambiguous_actions
created_at
```

### `executions`

```text
execution_id
test_id
environment_id
environment_snapshot_ref
started_at
finished_at
runner_status
runtime_validation_status
artifact_refs
failure_analysis_ref
```

### `environment_snapshots`

```text
snapshot_id
environment_id
captured_at
services
feature_flags
devices
health
configuration_hash
```

### `capability_gaps`

```text
gap_id
test_id
action
jira_ticket_id
implementation_pr
status
resolved_capability_id
```

### `approvals`

```text
approval_id
workflow_id
action_type
requested_at
decision
decided_by
decided_at
payload
```

### `sync_jobs`

```text
sync_job_id
test_id
operation
idempotency_key
status
attempt_count
last_error
zephyr_test_id
zephyr_cycle_id
created_at
updated_at
```

---

## 4. Publication Policy

A test should normally not become `READY_TO_PUBLISH` until required gates pass.

Example:

```text
coverage_validated == true
composition_validated == true          # if automated
unresolved_blocking_failures == false
duplicate_decision accepted
release_gate == READY_TO_PUBLISH
```

Whether successful runtime execution is mandatory before Zephyr publication should be a configurable policy.

Possible policies:

```text
DESIGN_ONLY:
publish validated manual tests before automation exists

AUTOMATION_REQUIRED:
publish only after automation composition is valid

EXECUTION_REQUIRED:
publish only after at least one successful runtime execution

STABILITY_REQUIRED:
publish only after N successful runs / no recent instability
```

Do not hardcode this until product owners decide.

---

## 5. Duplicate Detection

Duplicate handling should be hybrid.

Step 1 — deterministic candidate retrieval:

- same requirement
- same feature/module
- similar title
- shared tags
- keyword overlap
- structured step similarity
- existing linked test IDs

Step 2 — semantic specialist review:

```text
NEW_TEST
DUPLICATE
SIMILAR
UPDATE_EXISTING
NEED_HUMAN_REVIEW
```

The LLM should compare only a small candidate set, not the full Zephyr repository.

---

## 6. Zephyr Sync Worker

The worker is deterministic.

Responsibilities:

```text
consume READY_TO_PUBLISH jobs
create/update Zephyr test
create/link folder if required
create/add to test cycle if policy requests it
capture returned IDs
update MongoDB
retry transient errors
dead-letter permanent failures
```

It should not reason about test quality.

That decision has already happened.

---

## 7. Idempotency

Every publish operation requires an idempotency key.

Conceptual key:

```text
project + test_id + release_version + operation
```

Before creating a test:

```text
1. Check Mongo sync state.
2. Check known Zephyr mapping.
3. Use deterministic correlation metadata.
4. Avoid duplicate create on retry.
```

Retries must never generate duplicate official tests.

---

## 8. Asynchronous User Experience

The UI should separate:

```text
Test Ready Internally
```

from:

```text
Zephyr Publication Status
```

Example:

```text
Test generation: Complete
Coverage validation: Passed
Automation: Ready
Execution: Passed
Release gate: Approved
Zephyr sync: 14/20 published
```

The user does not need to wait for all external writes.

---

## 9. Test Cycle Creation

Do not automatically create a Zephyr Test Cycle for every generated suite.

Define a policy:

```text
create_cycle:
  never
  user_requested
  release_gate_requested
  suite_complete
```

The cycle may be created by the Zephyr Sync Worker after required test IDs are known.

---

## 10. Sync Failure Policy

Transient:

```text
timeout
rate limit
temporary Zephyr/Jira outage
network error
```

=> retry with backoff.

Permanent:

```text
invalid project
permission denied
invalid field mapping
schema mismatch
```

=> mark `SYNC_FAILED`, dead-letter, notify operator.

Do not repeatedly hammer a permanent error.

---

## 11. MongoDB Is Not the Final Source of Product Requirements

MongoDB is the operational workflow store.

Authoritative external sources remain:

```text
Confluence -> product requirements
Jira       -> engineering/work items
Zephyr     -> official published tests
Git        -> automation source code
Graphify   -> generated code-knowledge view
```

Persist references and snapshots for reproducibility.
