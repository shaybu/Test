# LangGraph Implementation Skeleton

This file is implementation-oriented but intentionally not a complete application.

Its purpose is to show how the architecture maps to code.

---

## 1. State

```python
from pydantic import BaseModel

class QAWorkflowState(BaseModel):
    workflow_id: str
    session_id: str
    stage: str

    requirement_ref: str | None = None
    test_suite_ref: str | None = None
    current_test_id: str | None = None

    capability_resolution_ref: str | None = None
    executable_test_ref: str | None = None

    environment_requirement_ref: str | None = None
    selected_environment_id: str | None = None

    execution_id: str | None = None
    release_decision_ref: str | None = None
    pending_approval_id: str | None = None
```

For LangGraph, this may be represented by a TypedDict/dataclass internally while Pydantic models validate domain boundaries.

---

## 2. Team Subgraph Example — Requirements

```python
from langgraph.graph import StateGraph, START, END


def build_requirements_graph():
    graph = StateGraph(RequirementsState)

    graph.add_node("intake", requirement_intake_node)
    graph.add_node("analyze", requirement_analysis_node)
    graph.add_node("generate", test_generation_node)
    graph.add_node("validate", coverage_validation_node)
    graph.add_node("stage", stage_tests_node)

    graph.add_edge(START, "intake")
    graph.add_edge("intake", "analyze")
    graph.add_edge("analyze", "generate")
    graph.add_edge("generate", "validate")

    graph.add_conditional_edges(
        "validate",
        route_coverage_result,
        {
            "revise": "generate",
            "pass": "stage",
            "human": "human_requirement_gate",
        },
    )

    graph.add_edge("stage", END)

    return graph.compile()
```

---

## 3. Specialist Node Pattern

```python
async def coverage_validation_node(state: RequirementsState):
    requirement = await requirement_repo.get(state.requirement_ref)
    tests = await test_repo.get_suite(state.test_suite_ref)

    prompt_input = CoverageValidationInput(
        requirements=requirement,
        tests=tests,
    )

    result: CoverageValidationResult = await agent_runner.run_structured(
        agent_name="coverage_validation",
        model_profile="reasoning_default",
        input_model=prompt_input,
        output_model=CoverageValidationResult,
        tools=READ_ONLY_REQUIREMENT_TOOLS,
    )

    await validation_repo.save(result)

    return {
        "coverage_validation_ref": result.validation_id
    }
```

The agent returns data.

The routing function makes the workflow decision.

---

## 4. Deterministic Router

```python
def route_coverage_result(state: RequirementsState) -> str:
    result = validation_repo.get_cached(state.coverage_validation_ref)

    if result.status == "PASS":
        return "pass"

    if result.status == "REVISE":
        return "revise"

    return "human"
```

Do not ask another LLM to decide what `PASS` means.

---

## 5. Parallel Fan-Out

Conceptual graph:

```python
graph.add_edge("test_staged", "capability_discovery")
graph.add_edge("test_staged", "duplicate_precheck")

graph.add_edge("capability_discovery", "post_discovery_join")
graph.add_edge("duplicate_precheck", "post_discovery_join")
```

For dynamic fan-out over many test cases, use LangGraph's dynamic fan-out pattern rather than hardcoding one node per test.

---

## 6. Graphify Tool Wrapper

```python
class GraphifyClient:
    async def query(self, question: str) -> GraphQueryResult:
        ...

    async def get_node(self, node_id: str) -> GraphNode:
        ...

    async def neighbors(self, node_id: str) -> list[GraphNode]:
        ...
```

Agent tool:

```python
async def find_capabilities(intent: StructuredTestIntent):
    result = await graphify.query(
        build_capability_query(intent)
    )
    return adapt_graphify_result(result)
```

Keep the MCP response adaptation outside the prompt.

---

## 7. Capability Discovery Output

```python
class CapabilityResolutionMap(BaseModel):
    test_id: str
    matches: list[CapabilityMatch]
    missing_actions: list[str]
    ambiguous_actions: list[str]
```

Router:

```python
def route_capability_resolution(state):
    result = capability_repo.get(state.capability_resolution_ref)

    if result.ambiguous_actions:
        return "human_or_disambiguate"

    if result.missing_actions:
        return "engineering"

    return "compose"
```

---

## 8. Human-in-the-Loop

Conceptual:

```python
from langgraph.types import interrupt


def environment_approval_node(state):
    request = ApprovalRequest(
        approval_id=new_id(),
        workflow_id=state.workflow_id,
        action_type="ENVIRONMENT_CHANGE",
        summary="Restart case-service in QA-3",
        impact="Shared environment",
        proposed_changes=state.proposed_changes,
        evidence_refs=state.evidence_refs,
        allowed_decisions=["APPROVE", "REJECT", "EDIT"],
    )

    decision = interrupt(request.model_dump())

    return apply_approval_decision(decision)
```

Persist the workflow before interruption.

---

## 9. Zephyr Publication

Do not put this in the main request path.

```python
async def mark_ready_to_publish(test_id: str):
    job = SyncJob(
        test_id=test_id,
        operation="CREATE_OR_UPDATE_TEST",
        idempotency_key=build_idempotency_key(test_id),
        status="PENDING",
    )
    await sync_job_repo.insert(job)
    await queue.publish(job.sync_job_id)
```

Worker:

```python
async def process_sync_job(job_id: str):
    job = await sync_job_repo.lock(job_id)

    if job.status == "COMPLETED":
        return

    test = await test_repo.get(job.test_id)

    result = await zephyr_adapter.publish(test, job.idempotency_key)

    await test_repo.set_zephyr_mapping(
        test.test_id,
        result.zephyr_test_id,
    )

    await sync_job_repo.complete(job_id)
```

---

## 10. Tool Registry

```python
AGENT_TOOL_POLICY = {
    "requirement_intake": {
        "confluence.read",
        "jira.read",
    },
    "capability_discovery": {
        "graphify.query",
        "graphify.get_node",
        "graphify.get_neighbors",
    },
    "environment_config": {
        "environment.read",
        "environment.allowlisted_write",
    },
    "chat": {
        "confluence.read",
        "jira.read",
        "zephyr.read",
        "graphify.read",
        "environment.read",
        "workflow.read",
    },
}
```

The runtime verifies permission before executing a tool.

Prompt instructions are not sufficient authorization.

---

## 11. Model Registry

```python
MODEL_PROFILES = {
    "reasoning_default": ModelProfile(...),
    "multimodal_default": ModelProfile(...),
    "code_default": ModelProfile(...),
    "small_classifier": ModelProfile(...),
}
```

Agent configuration references a role, not a hardcoded vendor model.

---

## 12. Agent Registry

```python
class AgentDefinition(BaseModel):
    name: str
    team: str
    model_profile: str
    prompt_version: str
    output_schema: type[BaseModel]
    tool_policy: str
    max_retries: int
```

Example:

```python
AgentDefinition(
    name="coverage_validation",
    team="requirements",
    model_profile="reasoning_default",
    prompt_version="coverage-v1",
    output_schema=CoverageValidationResult,
    tool_policy="coverage_validator",
    max_retries=1,
)
```

---

## 13. Main Graph Skeleton

```text
START
 |
Entry Router
 |
 +--> Requirements Subgraph
 |
 +--> Composition Subgraph
 |
 +--> Environment/Execution Subgraph
 |
 +--> Contextual Chat
 |
Quality/Lifecycle
 |
Async publish handoff
 |
END
```

The real main graph should route by structured workflow request type.

---

## 14. Development Rule

Every new specialist must include:

```text
1. Pydantic input schema
2. Pydantic output schema
3. prompt version
4. tool allowlist
5. permission policy
6. unit tests for router
7. agent eval examples
8. timeout
9. retry policy
10. telemetry fields
```

An "agent" is not complete until all ten exist.
