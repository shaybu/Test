# Environment, Execution, and Validation

## 1. Environment as State

An environment is not just a name such as `QA-3`.

The platform must model its state.

Example:

```text
environment_id
application version
service versions
feature flags
service health
available devices
connected dependencies
test data
accounts
configuration
capacity
ownership/shared status
```

The Environment Matching Specialist compares this state to the requirements of a test.

---

## 2. Environment Requirement Extraction

A test may require:

```text
Feature X enabled
Case service >= 2.7
Android device available
SMS simulator enabled
Authenticated test account
Specific dataset loaded
External dependency reachable
```

Requirements should come from:

- capability contracts
- test definition
- explicit user constraints
- platform policies

Do not infer hidden environmental assumptions from memory.

---

## 3. Match Algorithm

Conceptual sequence:

```text
hard requirements
      |
filter environments
      |
eligible candidates
      |
calculate configuration delta
      |
rank:
1. already ready
2. minimal safe changes
3. user preference
4. resource availability
      |
decision
```

Use deterministic filtering for hard constraints.

Use an agent only where semantic interpretation/ranking is useful.

---

## 4. Configuration Policy

Environment changes should have a risk classification.

```text
READ_ONLY
LOW_RISK
SHARED_IMPACT
DESTRUCTIVE
PRODUCTION
```

Example policy:

```text
READ_ONLY       -> autonomous
LOW_RISK        -> autonomous if allowlisted
SHARED_IMPACT   -> Human-in-the-Loop
DESTRUCTIVE     -> Human-in-the-Loop
PRODUCTION      -> forbidden in v1 unless explicitly designed later
```

---

## 5. Before/After Snapshots

Every write must record:

```text
before snapshot
requested delta
approval reference if needed
applied changes
after snapshot
verification result
```

This allows:

- audit
- troubleshooting
- rollback planning
- failure classification

---

## 6. Future Environment Provisioning

Do not mix provisioning into v1 unless required.

Future flow:

```text
NO_ENVIRONMENT
    |
Provisioning Plan
    |
cost/capacity/policy check
    |
HITL
    |
Infrastructure Provisioning Service
    |
health validation
    |
Environment Snapshot
    |
Execution
```

Provisioning should likely be a deterministic infrastructure workflow with an agent generating/validating the plan, not an LLM freely operating infrastructure.

---

## 7. Execution Boundary

The Test Execution Specialist should call an execution service/tool.

It should not itself implement the runner.

```text
Agent
 |
run_test(ExecutionPlan)
 |
Execution Service
 |
Automation Framework
```

Benefits:

- stable API
- easier retries
- isolation
- observability
- capacity management

---

## 8. Execution Evidence

Minimum evidence should be configurable by test type.

Possible evidence:

```text
runner assertions
structured return values
API responses
logs
screenshots
video
device logs
service telemetry
environment snapshot
timestamps
```

Evidence references should be stored, not embedded as huge blobs in graph state.

---

## 9. Validation Hierarchy

Prefer the most deterministic evidence first.

```text
1. explicit automation assertion
2. structured API result
3. structured log/telemetry condition
4. image/screenshot comparison
5. multimodal reasoning
6. human review
```

Do not use an LLM to validate something a deterministic assertion can prove.

---

## 10. Multimodal Validation

Use GLM 5.3 Flash or another approved multimodal model for cases such as:

- UI state verification
- visual confirmation
- screenshot anomaly
- text rendered in image-only UI
- cross-check of expected screen state

The multimodal model should receive:

```text
expected condition
relevant screenshots
small structured execution context
```

not the entire workflow history.

---

## 11. Flakiness

A failed run is not always a failed product.

Store stability history.

Possible policy:

```text
single deterministic assertion failure -> FAIL
infrastructure timeout -> RETRYABLE
visual low-confidence mismatch -> INCONCLUSIVE
historically flaky test -> TRIAGE
```

Track:

```text
pass rate
retry count
failure categories
environment correlation
recent code changes
```

---

## 12. Execution Concurrency

Parallel execution must respect:

- environment capacity
- device capacity
- data collision
- account collision
- service rate limits
- test isolation
- destructive side effects

Introduce resource locks/leases where required.

Example:

```text
device_id lease
environment mutation lock
shared test-account lock
unique case/data namespace
```

---

## 13. Environment-Aware Chat

The Contextual Chat Agent should be able to answer:

```text
"Why can't this test run on QA-3?"
"What is missing on QA-4?"
"Which environment can run it now?"
"What did you change before the last run?"
```

It reads structured environment state.

It does not mutate the environment directly.
