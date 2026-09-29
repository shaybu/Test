# Observability, Reliability, and Testing

## 1. Every Workflow Needs a Trace

Record:

```text
workflow_id
session_id
test_id
agent/node
model
prompt/version
tool calls
latency
structured output status
retry count
handoff
human interrupt
result
error
```

Do not store secrets in traces.

---

## 2. Metrics

### Agent Quality

```text
schema validation failure rate
revision-loop count
human escalation rate
tool failure rate
average latency
model usage by specialist
```

### Test Design

```text
coverage validator pass rate
average revisions per suite
human ambiguity escalations
requirements with no tests
```

### Graphify

```text
capability match rate
missing capability rate
ambiguous match rate
query latency
graph freshness lag
```

### Execution

```text
environment match rate
configuration-required rate
execution pass/fail/inconclusive
flaky rate
environment-correlated failures
```

### Publishing

```text
ready-to-publish count
duplicate rate
Zephyr sync latency
sync retry rate
sync failure rate
```

---

## 3. Agent Evals

Each specialist should have an evaluation dataset.

Example — Coverage Validator:

```text
input:
- PRD
- generated tests

expected:
- missing requirement IDs
- unsupported assumption
- PASS/REVISE
```

Example — Failure Triage:

```text
input:
- logs
- environment state
- test result

expected category:
ENVIRONMENT_ISSUE
```

Measure specialists independently before measuring the full workflow.

---

## 4. Golden Cases

Maintain a curated "golden" suite of known scenarios.

Include:

- clear PRD
- ambiguous PRD
- missing acceptance criteria
- complete test suite
- incomplete test suite
- capability found
- capability missing
- duplicate test
- environment ready
- environment configurable
- no environment
- product failure
- automation failure
- flaky execution
- visual validation

Run them on every prompt/model/framework change.

---

## 5. Deterministic Workflow Tests

LangGraph routing should have normal unit tests.

Example:

```text
CoverageValidationResult(PASS)
=> next node must be StageTest

CoverageValidationResult(REVISE)
=> next node must be TestGeneration

EnvironmentMatch(CONFIGURABLE)
=> next node must be EnvironmentConfiguration
```

Do not test deterministic routing using an LLM.

---

## 6. Contract Tests

Validate every tool boundary:

- Jira payload
- Zephyr payload
- Graphify response adapter
- Mongo model
- environment snapshot
- execution service
- local model structured output

Pydantic validation must fail loudly on incompatible schemas.

---

## 7. Model Regression

Changing the local model can change behavior.

Before promoting a model:

```text
run specialist evals
run golden workflows
compare:
- quality
- latency
- structured-output reliability
- tool-selection behavior
- context-window usage
```

Model assignment can be different per specialist.

---

## 8. Retry Strategy

Centralize retry policy.

Example defaults:

```text
LLM invalid schema:
1 retry with validation feedback

MCP read transient failure:
2-3 retries with backoff

MCP write:
retry only if idempotent

environment mutation:
no blind retry

test execution:
policy dependent

Zephyr sync:
worker retry with idempotency
```

---

## 9. Timeouts

Every node/tool must have a timeout.

Long-running work should not keep one HTTP request open.

Use:

```text
workflow state
job status
UI polling/websocket/event update
```

---

## 10. Dead-Letter Handling

Async workers require dead-letter behavior.

Example:

```text
Zephyr publish failed permanently
 -> sync_job DEAD_LETTER
 -> operator notification
 -> UI visible
 -> manual retry after fix
```

---

## 11. Graph Freshness

Track:

```text
repository_commit
graph_commit
graph_generated_at
```

If Graphify is behind the source branch used for test generation, warn or block capability decisions according to policy.

---

## 12. Workflow Replay

For a failed workflow, operators should be able to inspect:

```text
state at checkpoint
specialist input
specialist structured output
tool response refs
routing decision
approval decision
```

Avoid relying only on textual logs.

---

## 13. Prompt Versioning

Prompts are production code.

Store:

```text
agent name
prompt version
schema version
model profile
date
change reason
evaluation result
```

Persist prompt version with each run.

---

## 14. Cost / Compute

Even on-prem, compute is not free.

Track:

```text
model inference latency
GPU usage if available
tokens/context size
parallel agent count
multimodal invocation rate
```

Use smaller models only where measured quality supports it.
