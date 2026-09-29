# Architecture and Technology Stack

## 1. Architectural Goal

The system is not a chatbot with tools attached.

It is a controlled QA automation platform whose user-facing interface is natural language, while the internal implementation remains structured, typed, permissioned, observable, and deterministic wherever possible.

The platform must support:

- PRD-to-test generation
- independent test coverage validation
- reusable automation capability discovery
- automated test composition
- missing capability detection
- automated engineering workflow for missing capabilities
- environment selection and configuration
- test execution
- runtime result validation
- failure classification
- duplicate detection
- staged test lifecycle
- asynchronous Zephyr publishing
- contextual QA chat
- Human-in-the-Loop gates
- parallel work
- resumable long-running workflows

---

## 2. Recommended v1 Stack

| Layer | Recommended Technology | Responsibility |
|---|---|---|
| UI | Existing/new web UI | Natural-language requests, test views, approvals, status |
| Orchestration | LangGraph | State, routing, subgraphs, loops, parallelism, HITL |
| Schema/Contracts | Pydantic | Typed inputs, outputs, state, settings, persistence models |
| LLM Runtime | Internal model adapter | GLM/Kimi/Gemma/local approved models |
| Code Knowledge | Graphify | Queryable graph of automation code and capability wrappers |
| Tool Protocol | MCP + direct internal tools | Confluence, Jira, Zephyr, Graphify, environment, etc. |
| Operational Store | MongoDB | Sessions, generated tests, lifecycle, runs, approvals, sync |
| Async Jobs | Internal queue + workers | Zephyr sync, Graphify refresh, deterministic background work |
| Automation | Existing QA automation framework | Actual reusable test capabilities and execution |
| Source Control | Git / Bitbucket | Capability development and code review |
| Environment | Kubernetes/internal environment tooling | State inspection, configuration, future provisioning |
| Observability | Internal logs/metrics/traces | Debugging, audit, latency, quality, reliability |

---

## 3. Why LangGraph

The core problem is orchestration, not simply agent creation.

The workflow contains:

- explicit steps
- branching
- loops
- retries
- parallel branches
- long-running state
- user approvals
- asynchronous external operations
- specialist teams
- subgraphs
- recoverable failures

This is a graph/state-machine problem.

LangGraph should own:

```text
workflow state
routing
conditional edges
team subgraphs
parallel fan-out/fan-in
interrupt/resume
retry policy
checkpoint boundaries
handoffs
```

The LLM should **not** be allowed to own all routing decisions by default.

### Preferred rule

If a routing decision can be represented deterministically:

```python
if validation.status == "PASS":
    route = "next_stage"
else:
    route = "revision"
```

use code.

Use an LLM router only when the decision itself requires semantic interpretation.

---

## 4. LangGraph vs LangChain

Do not treat "LangChain" as a requirement.

The platform can use LangGraph as the orchestration runtime without building the architecture around high-level LangChain chains.

Recommended v1 approach:

```text
LangGraph
+
Pydantic
+
internal model adapter
+
MCP/direct tools
```

Use individual LangChain integration packages only if they provide a useful adapter that reduces implementation effort.

---

## 5. LangGraph vs OpenAI Agents SDK

Both are open-source frameworks, but they solve overlapping problems.

### LangGraph strength

- explicit workflow graph
- long-running stateful flows
- branches and loops
- subgraphs
- deterministic routing
- parallel execution
- Human-in-the-Loop
- durable workflow-style architecture

### OpenAI Agents SDK strength

- concise agent definition
- tools
- handoffs
- sessions
- guardrails
- agent runtime abstractions

### v1 decision

Use **LangGraph only as the orchestration framework**.

Do not add OpenAI Agents SDK unless a concrete missing capability appears.

Reasons:

1. Avoid two orchestration abstractions.
2. Keep local-model integration simple.
3. Preserve full workflow control.
4. Reduce hidden state and debugging complexity.
5. The system already requires graph-level state and workflow control.

The Agents SDK can be reconsidered later as an internal agent runtime if a measurable benefit exists.

---

## 6. Open-Source / On-Premise Requirement

The platform must be deployable without external model APIs.

Verified project licenses at the time of this architecture:

- LangGraph: MIT
- Pydantic: MIT
- Pydantic Settings: MIT
- Graphify current repository: Apache-2.0, with older MIT-licensed portions retained
- OpenAI Agents SDK: MIT, but not required in v1

Licenses must be re-verified by the company before production adoption.

---

## 7. Local Model Layer

The architecture must not hardcode agent logic to one model.

Create a model abstraction:

```python
class ModelProfile:
    name: str
    endpoint: str
    multimodal: bool
    context_window: int
    structured_output: bool
    tool_calling: bool
```

Example internal profiles:

```text
reasoning_default -> GLM 5.3
multimodal_default -> GLM 5.3 Flash
alternative_reasoning -> Kimi
fallback -> internally approved Gemma model
```

Exact assignments should be benchmarked rather than assumed.

### Model selection should be per specialist

Examples:

- Requirement Analysis: strong text reasoning model
- Test Generation: strong reasoning model
- Runtime visual validation: multimodal model
- Classification/routing: smaller model may be sufficient
- Code Development: strongest internal code-capable model

---

## 8. Optional Small Decision Model

A JEV-like small decision model is **not a v1 dependency**.

It may become useful later for:

- cheap classification
- repetitive routing
- confidence scoring
- selecting known options
- simple policy decisions

Only introduce it after telemetry proves that:

1. the same semantic decisions occur frequently,
2. they are too complex for deterministic code,
3. a smaller model performs reliably,
4. using it reduces latency or compute cost.

Do not complicate the initial architecture for a hypothetical optimization.

---

## 9. Agent vs Tool vs Worker vs Service

This distinction is mandatory.

### Agent
Use when reasoning or judgment is required.

Examples:
- determine requirement ambiguity
- generate tests
- classify a failure
- find the best capability match

### Tool
Use for a bounded action invoked by an agent.

Examples:
- `get_confluence_page`
- `query_graphify`
- `get_environment_snapshot`
- `run_test`
- `search_zephyr_tests`

### Worker
Use for asynchronous deterministic processing.

Examples:
- Zephyr Sync Worker
- Graphify Update Worker
- batch status updater

### Service
Use for stable platform logic.

Examples:
- Mongo repository
- permission service
- session service
- environment inventory API
- idempotency service

---

## 10. Team/Subgraph Model

Each team is a LangGraph subgraph.

```text
Main QA Graph
|
+-- Requirements & Test Design Subgraph
+-- Test Composition & Capability Subgraph
+-- Automation Engineering Subgraph
+-- Environment & Execution Subgraph
+-- Quality & Lifecycle Subgraph
|
+-- Contextual Chat entry path
```

Specialists inside a team behave like sub-agents from a product perspective, but implementation should remain explicit.

A Team Supervisor should not automatically be an LLM.

Prefer:

```text
deterministic subgraph routing
```

over:

```text
LLM supervisor deciding every next step
```

unless semantic routing is actually needed.

---

## 11. Context Isolation

Every specialist invocation has access to the model's context window, but it should receive a **scoped context package**, not the entire global history.

Example:

```text
Automation Development Specialist receives:
- missing capability specification
- Jira ticket
- relevant Graphify neighborhood
- selected source files
- coding standards

It does NOT receive:
- full PRD history
- unrelated Zephyr tests
- all environment states
- complete user chat transcript
```

This prevents:

- context pollution
- token waste
- irrelevant reasoning
- accidental cross-domain actions
- harder debugging

---

## 12. Core Architectural Rule

The workflow controls the agents.

Agents do not control the platform.

```text
LangGraph decides:
- what state exists
- who receives it
- who can write
- what runs next
- what can run in parallel
- where approval is required

Agent decides:
- the semantic result of its narrowly defined task
```
