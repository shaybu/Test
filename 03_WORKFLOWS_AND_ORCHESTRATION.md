# Workflows and Orchestration

## 1. Main Graph

```text
Main QA LangGraph
|
+-- Entry Router
|   +-- PRD/Test Design Request
|   +-- Build Automation Request
|   +-- Execute Test Request
|   +-- Chat/Question
|
+-- Requirements & Test Design Subgraph
+-- Test Composition & Capability Subgraph
+-- Automation Engineering Subgraph
+-- Environment & Execution Subgraph
+-- Quality & Lifecycle Subgraph
|
+-- MongoDB checkpoints
+-- HITL interrupts
+-- Async worker handoffs
```

The Main Graph is **not an LLM agent**.

Its routing should be deterministic wherever possible.

---

# 2. Workflow A — PRD to Validated Test Cases

```text
User provides PRD URL/reference
        |
Requirement Intake
        |
Requirement Analysis
        |
   Ambiguous?
    /     \
  yes      no
  |        |
HITL       |
  |        |
  +--------+
        |
Test Generation
        |
Coverage Validation
   /          \
REVISE        PASS
  |            |
  +--> Test    |
       Gen     |
              v
          MongoDB
       status=STAGED
```

Important:

- Do not block on Zephyr.
- Do not publish immediately.
- Store generated/validated test data internally first.
- Preserve requirement traceability.

---

# 3. Workflow B — Natural Language Test to Executable Automation

Example:

```text
"Create a criminal case, send a message, and verify that
 the message appears in the case timeline."
```

Flow:

```text
Test Intent Specialist
        |
Capability Discovery
        |
  +-----+------+
  |            |
complete     missing
  |            |
Test          Missing Capability
Composer      Specialist
  |            |
Validation     Jira + Engineering
  |            |
  |<-----------+
  |
VALID
```

A capability gap is not treated as a generation failure.

It becomes an engineering workflow.

---

# 4. Workflow C — Missing Capability Engineering Loop

```text
Missing Capability
       |
Missing Capability Specialist
       |
CapabilityDevelopmentSpecification
       |
Jira ticket
       |
Automation Development Specialist
       |
 +-----+------------------+
 |                        |
unit tests            static checks
 |                        |
 +-----------+------------+
             |
Capability Contract Validator
             |
Code Review Specialist
      /              \
REQUEST_CHANGES      APPROVE
      |                |
      +--> Developer   |
                       v
                  HUMAN PR GATE
                       |
                     MERGE
                       |
             Graphify Update Worker
                       |
             Capability discoverable
                       |
             resume Test Composition
```

The workflow must persist correlation identifiers so the original test request can resume after the engineering cycle.

---

# 5. Workflow D — Environment and Execution

```text
Executable Test
      |
Environment Requirement Specialist
      |
Environment Matching Specialist
      |
 +----+----------+-------------+
 |               |             |
READY        CONFIGURABLE    NONE
 |               |             |
 |      Environment Config     |
 |               |             |
 |       HITL if required      |
 |               |             |
 +---------------+             |
 |                             |
Test Execution          Human / future
 |                      Provisioning
Runtime Validation
```

### Environment decision policy

```text
1. Honor a user-selected environment if it satisfies hard requirements.
2. Otherwise search available environment inventory.
3. Prefer an already-ready environment.
4. If none is ready, consider a safely configurable environment.
5. If configuration is high impact, interrupt for approval.
6. If no suitable environment exists, return NO_ENVIRONMENT.
7. Future capability: provision a new environment.
```

---

# 6. Workflow E — Runtime Validation and Failure Triage

```text
ExecutionPackage
      |
Runtime Validation
  /       |         \
PASS     FAIL     INCONCLUSIVE
 |        |           |
 |     Failure        |
 |      Triage        |
 |        |           |
 |   classification   |
 |        |           |
 |   route to team    |
 |                    |
 +--------------------+
          |
 Quality/Lifecycle
```

Failure routes:

```text
AUTOMATION_DEFECT -> Automation Engineering
ENVIRONMENT_ISSUE -> Environment Team
TEST_DATA_ISSUE   -> Test-data remediation flow/service
REQUIREMENT_ISSUE -> Requirement flow / human
PRODUCT_DEFECT    -> Human gate -> Jira defect
FLAKY_TEST        -> retry/stability flow
UNKNOWN           -> human review
```

---

# 7. Workflow F — Quality Gate and Zephyr Publication

```text
Mongo staged test
      |
collect:
- coverage status
- composition validation
- runtime history
- failure state
- duplicate candidates
      |
Test Quality & Release Specialist
      |
 +----+----------+------------+-------------+
 |               |            |             |
READY          HOLD       DUPLICATE      HUMAN
 |                                          |
Mongo: READY_TO_PUBLISH                     |
 |                                          |
Publish Queue <-----------------------------+
 |
Zephyr Sync Worker
 |
Zephyr/Jira
 |
Mongo:
PUBLISHED
zephyr_test_id
zephyr_cycle_id (if applicable)
```

Publishing is asynchronous and idempotent.

---

# 8. Parallelism Rules

A branch may run in parallel only when all four conditions are true:

1. It does not require another branch's output.
2. It does not perform conflicting writes.
3. It has a defined join condition.
4. A branch failure has a defined policy.

Do not use parallelism merely because LangGraph supports it.

---

## Parallel Pattern P1 — Test-Level Fan-Out

After a test suite is validated:

```text
Validated Test Suite
      |
fan out each TestCase
  /     |      |      \
T1     T2     T3      Tn
```

Each test can independently begin:

- intent normalization
- capability discovery
- duplicate candidate pre-search

This is likely one of the largest performance gains.

---

## Parallel Pattern P2 — Discovery + Duplicate Pre-check

For one validated test:

```text
             Test
          /        \
Capability       Duplicate
Discovery        Pre-check
          \        /
             JOIN
```

The duplicate pre-check does not need to block capability discovery.

The final release decision occurs later after runtime evidence exists.

---

## Parallel Pattern P3 — Code Validation

After new capability implementation:

```text
                PR
       /---------+---------\
      /          |          \
Unit Tests   Static Scan   Contract Validation
      \          |          /
       \---------+---------/
                 |
              JOIN
                 |
           Code Review
```

---

## Parallel Pattern P4 — Runtime Evidence Analysis

```text
ExecutionPackage
  /       |        |        \
Logs     API     Visual    Telemetry
  \       |        |        /
   \------+--------+-------/
          |
       Evidence Join
          |
   Runtime Validation
```

---

# 9. Subgraphs and "Sub-agents"

Each team is a subgraph.

A specialist can be treated as a sub-agent conceptually, but do not rely on opaque parent-agent delegation.

Preferred implementation:

```text
Team Subgraph
|
+-- explicit specialist node
+-- explicit specialist node
+-- explicit conditional edge
+-- explicit interrupt
+-- explicit exit contract
```

Benefits:

- debuggable
- testable
- permissionable
- typed
- observable
- resumable

---

# 10. Handoffs

A handoff is a structured state transition, not a free-form prompt.

Example:

```python
class HandoffRequest(BaseModel):
    source_team: TeamName
    target_team: TeamName
    reason: str
    entity_id: str
    required_input_refs: list[str]
    priority: str
```

Never hand off by embedding an entire previous conversation as text.

---

# 11. Join Semantics

Every parallel fan-out must define how results join.

Example:

```python
class EvidenceBundle(BaseModel):
    log_result: LogValidation | None
    api_result: ApiValidation | None
    visual_result: VisualValidation | None
    telemetry_result: TelemetryValidation | None
```

Join policy can be:

- all required
- quorum
- first-success
- best-evidence
- timeout + partial result

The policy must be explicit per workflow.

---

# 12. Retry Policy

Retries belong to workflow/tool policy, not agent improvisation.

Examples:

```text
Network read failure:
automatic retry with backoff

LLM schema validation failure:
1 structured retry, then error

Environment configuration failure:
no blind repeated mutation

Test execution timeout:
retry only if policy permits

Zephyr sync failure:
worker retry with idempotency key

Human rejection:
do not auto-retry
```

---

# 13. Checkpoints

Recommended checkpoint boundaries:

```text
requirements retrieved
requirements analyzed
test suite generated
coverage passed
test staged in Mongo
capability resolution completed
test composition validated
environment selected
environment configured
execution completed
runtime validation completed
release decision created
Zephyr publication completed
human approval/rejection
```

Long-running workflows must be resumable from these points.
