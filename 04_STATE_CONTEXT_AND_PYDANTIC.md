# State, Context, and Pydantic Contracts

## 1. Why Pydantic Is Core

Pydantic does not replace LangGraph and is not an AI framework.

Its role is to make internal communication predictable.

Without typed contracts:

```text
Agent A -> free text -> Agent B -> guessed parsing -> Agent C
```

With Pydantic:

```text
Agent A -> RequirementModel
Agent B -> TestSuite
Agent C -> ValidationResult
```

This is essential for a system with many specialists, retries, persistence, and resumable state.

---

# 2. State Design

Do not create one giant unstructured state dictionary.

Use a typed global workflow state with references to domain models.

Conceptual model:

```python
class QAWorkflowState(BaseModel):
    workflow_id: str
    session_id: str

    requirement_ref: str | None = None
    test_suite_ref: str | None = None
    current_test_id: str | None = None

    capability_resolution_ref: str | None = None
    executable_test_ref: str | None = None

    environment_requirement_ref: str | None = None
    selected_environment_id: str | None = None

    execution_id: str | None = None
    validation_ref: str | None = None
    failure_analysis_ref: str | None = None

    release_decision_ref: str | None = None
    pending_approval_id: str | None = None

    stage: str
    status: str
```

Store large artifacts in MongoDB or artifact storage and pass references in graph state.

---

# 3. Context Types

## Session Context

Tracks what the user is currently working on.

```python
class SessionContext(BaseModel):
    session_id: str
    user_id: str
    current_project: str | None
    current_prd_ref: str | None
    current_test_id: str | None
    current_environment_id: str | None
    current_execution_id: str | None
```

This enables natural references:

```text
"this test"
"the PRD we are working on"
"run it on QA-4"
"why did it fail?"
```

---

## Requirement Context

```python
class RequirementItem(BaseModel):
    id: str
    text: str
    acceptance_criteria: list[str] = []
    dependencies: list[str] = []
    source_ref: str

class StructuredRequirementModel(BaseModel):
    requirement_set_id: str
    title: str
    items: list[RequirementItem]
    ambiguities: list[str] = []
    contradictions: list[str] = []
```

---

## Test Model

```python
class TestStep(BaseModel):
    order: int
    action: str
    expected: str | None = None

class TestCase(BaseModel):
    test_id: str
    title: str
    objective: str
    requirement_ids: list[str]
    preconditions: list[str]
    steps: list[TestStep]
    expected_result: str
    priority: str | None = None
    tags: list[str] = []
```

---

## Coverage Validation

```python
class CoverageGap(BaseModel):
    requirement_id: str
    reason: str
    suggested_fix: str

class CoverageValidationResult(BaseModel):
    status: Literal["PASS", "REVISE", "NEED_HUMAN_REVIEW"]
    covered_requirement_ids: list[str]
    gaps: list[CoverageGap]
    unsupported_assumptions: list[str] = []
    feedback: str
```

---

# 4. Capability Contracts

```python
class CapabilityParameter(BaseModel):
    name: str
    type: str
    required: bool
    description: str
    allowed_values: list[str] | None = None

class CapabilityContract(BaseModel):
    capability_id: str
    name: str
    purpose: str
    parameters: list[CapabilityParameter]
    preconditions: list[str]
    returns: dict
    related_capabilities: list[str]
    source_node_ref: str
```

---

# 5. Structured Test Intent

```python
class TestIntentAction(BaseModel):
    action_id: str
    intent: str
    depends_on: list[str] = []
    validation_required: bool = False

class StructuredTestIntent(BaseModel):
    test_id: str
    actions: list[TestIntentAction]
    required_data: dict = {}
    constraints: list[str] = []
```

---

# 6. Capability Resolution

```python
class CapabilityMatch(BaseModel):
    action_id: str
    capability_id: str
    confidence: float
    evidence: list[str]

class CapabilityResolutionMap(BaseModel):
    test_id: str
    matches: list[CapabilityMatch]
    missing_actions: list[str]
    ambiguous_actions: list[str]
```

Do not use confidence alone to authorize risky actions.

---

# 7. Environment Contracts

```python
class EnvironmentRequirementProfile(BaseModel):
    test_id: str
    required_services: dict[str, str | None]
    required_feature_flags: dict[str, bool]
    required_devices: list[str]
    required_test_data: list[str]
    required_accounts: list[str]
    constraints: list[str]

class EnvironmentSnapshot(BaseModel):
    environment_id: str
    captured_at: datetime
    services: dict[str, str]
    feature_flags: dict[str, bool]
    healthy: bool
    devices: list[str]
    metadata: dict
```

Match result:

```python
class EnvironmentMatchDecision(BaseModel):
    status: Literal["READY", "CONFIGURABLE", "NO_ENVIRONMENT", "NEED_HUMAN_REVIEW"]
    environment_id: str | None
    missing_requirements: list[str]
    required_changes: list[str]
    rationale: str
```

---

# 8. Execution Contracts

```python
class ExecutionPlan(BaseModel):
    execution_id: str
    test_id: str
    environment_id: str
    timeout_seconds: int
    required_artifacts: list[str]

class ExecutionPackage(BaseModel):
    execution_id: str
    status: str
    started_at: datetime
    finished_at: datetime | None
    assertion_results: list[dict]
    log_refs: list[str]
    screenshot_refs: list[str]
    video_refs: list[str]
    api_evidence_refs: list[str]
    telemetry_refs: list[str]
```

---

# 9. Runtime Validation

```python
class RuntimeValidationResult(BaseModel):
    execution_id: str
    status: Literal["PASS", "FAIL", "INCONCLUSIVE"]
    evidence_refs: list[str]
    failed_expectations: list[str]
    confidence: float | None
    rationale: str
```

---

# 10. Failure Analysis

```python
class FailureAnalysis(BaseModel):
    execution_id: str
    category: Literal[
        "PRODUCT_DEFECT",
        "AUTOMATION_DEFECT",
        "ENVIRONMENT_ISSUE",
        "TEST_DATA_ISSUE",
        "REQUIREMENT_ISSUE",
        "FLAKY_TEST",
        "UNKNOWN"
    ]
    evidence_refs: list[str]
    confidence: float
    recommended_route: str
    proposed_ticket: dict | None = None
```

---

# 11. Release Decision

```python
class ReleaseDecision(BaseModel):
    test_id: str
    status: Literal[
        "READY_TO_PUBLISH",
        "DUPLICATE",
        "UPDATE_EXISTING",
        "HOLD",
        "REJECT",
        "NEED_HUMAN_REVIEW"
    ]
    duplicate_candidate_ids: list[str] = []
    rationale: str
```

---

# 12. Human Approval Contract

```python
class ApprovalRequest(BaseModel):
    approval_id: str
    workflow_id: str
    action_type: str
    summary: str
    impact: str
    proposed_changes: dict
    evidence_refs: list[str]
    allowed_decisions: list[str]
```

Response:

```python
class ApprovalDecision(BaseModel):
    approval_id: str
    decision: Literal["APPROVE", "REJECT", "EDIT", "CANCEL"]
    edited_payload: dict | None = None
    comment: str | None = None
```

---

# 13. Context Isolation

The global workflow may know many references.

The specialist should not.

Example:

```text
Global state contains:
PRD
tests
Graphify refs
environment refs
run history
approval history
chat session

Capability Discovery receives:
- current test intent
- relevant project/module scope
- Graphify tools

Nothing else.
```

Every node should have an explicit context builder.

```python
def build_capability_context(state: QAWorkflowState) -> CapabilityDiscoveryInput:
    ...
```

---

# 14. Sub-agent Context Windows

Each specialist call can use the full available context window of its assigned model.

That does **not** mean passing the full parent history.

The desired pattern:

```text
Parent state
   |
context projection
   |
Specialist context window
   |
structured output
   |
merge only the output into parent state
```

This provides:

- independent reasoning space
- less context pollution
- predictable input
- easier replay/evaluation
- lower token use
- stronger permission isolation

---

# 15. Pydantic Settings

Use `pydantic-settings` for validated application configuration.

Example:

```python
class Settings(BaseSettings):
    mongo_uri: str
    graphify_mcp_url: str
    jira_mcp_url: str
    zephyr_mcp_url: str
    confluence_mcp_url: str

    default_reasoning_model: str
    default_multimodal_model: str

    max_agent_retries: int = 1
    max_parallel_tests: int = 5
```

Secrets must come from approved internal secret management, not source code.

---

# 16. Schema Versioning

Persist a schema version with long-lived objects.

Example:

```json
{
  "schema_version": "1.0",
  "test_id": "..."
}
```

When changing Pydantic models:

- add migration strategy
- preserve old workflow resumability
- version external payload adapters
- do not silently reinterpret persisted records
