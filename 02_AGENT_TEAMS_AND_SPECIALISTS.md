# Agent Teams and Specialist Definitions

## Standard Agent Contract

Every specialist must be defined with:

```text
Name
Team
Role
Goal
Inputs
Outputs
Tools
Read Permissions
Write Permissions
Allowed Actions
Forbidden Actions
Receives Handoff From
Hands Off To
Parallel Compatibility
Human-in-the-Loop
Context Scope
Failure Behavior
```

Agents must use structured Pydantic outputs.

---

# Team 1 — Requirements & Test Design

## Agent 01 — Requirement Intake Specialist

**Role:** Acquire and normalize the source requirements.

**Goal:** Produce the correct, versioned requirement package for downstream analysis.

**Inputs**
- Confluence URL, Jira reference, or user request
- session context
- optional explicit requirement version

**Outputs**
- `RequirementPackage`
- source identifiers
- source version/revision metadata
- retrieval status

**Tools**
- Confluence MCP
- Jira MCP read-only

**Permissions**
- Read: Confluence, Jira
- Write: none

**Allowed**
- retrieve PRD
- retrieve linked acceptance criteria
- resolve linked requirement pages
- collect metadata
- identify version conflicts

**Forbidden**
- modify Confluence
- modify Jira
- invent missing requirement content

**Receives From**
- UI
- Contextual Chat handoff
- scheduled/request-driven workflow

**Hands Off To**
- Requirement Analysis Specialist

**Parallel**
- normally no; it is an entry dependency

**HITL**
- when multiple conflicting source versions exist
- when required source access is missing
- when the requested requirement cannot be uniquely identified

**Context Scope**
- source request
- retrieved requirement metadata
- minimal session reference

**Failure**
- `SOURCE_NOT_FOUND`
- `ACCESS_DENIED`
- `VERSION_CONFLICT`
- `NEED_HUMAN_REVIEW`

---

## Agent 02 — Requirement Analysis Specialist

**Role:** Convert raw requirements into a structured testable model.

**Goal:** Understand what the product is expected to do before test generation starts.

**Inputs**
- `RequirementPackage`

**Outputs**
- `StructuredRequirementModel`
- ambiguity list
- contradiction list
- requirement IDs
- acceptance criteria
- dependencies
- assumptions explicitly marked

**Tools**
- Confluence MCP read-only
- Jira MCP read-only

**Permissions**
- Read only

**Allowed**
- extract functional requirements
- extract acceptance criteria
- identify business rules
- identify dependencies
- identify preconditions
- identify ambiguity
- identify contradiction

**Forbidden**
- resolve business ambiguity by silently choosing an interpretation
- edit requirements

**Receives From**
- Requirement Intake Specialist

**Hands Off To**
- Test Generation Specialist
- Human Gate on critical ambiguity

**Parallel**
- no direct parallel dependency in the primary flow

**HITL**
- critical ambiguity affecting expected behavior
- contradictory requirements
- missing acceptance criteria when required for safe generation

**Context Scope**
- requirement package
- only directly relevant linked requirements

**Failure**
- `READY`
- `AMBIGUOUS`
- `CONTRADICTORY`
- `INCOMPLETE`
- `NEED_HUMAN_REVIEW`

---

## Agent 03 — Test Generation Specialist

**Role:** Generate logical QA test cases.

**Goal:** Create a high-quality test suite independent of automation implementation details.

**Inputs**
- `StructuredRequirementModel`
- optional prior validation feedback
- optional existing related tests for reference

**Outputs**
- `GeneratedTestSuite`

**Tools**
- Jira read-only
- Zephyr read-only/search when useful
- internal test-store search read-only

**Permissions**
- Read only external systems
- no publication rights

**Allowed**
- positive scenarios
- negative scenarios
- edge cases
- boundary cases
- preconditions
- test steps
- expected results
- requirement traceability
- tags/metadata

**Forbidden**
- publish directly to Zephyr
- execute tests
- change requirements
- create automation code

**Receives From**
- Requirement Analysis Specialist
- Coverage Validation Specialist on revision loop

**Hands Off To**
- Coverage Validation Specialist

**Parallel**
- test generation may be parallelized by independent requirement group only if traceability remains deterministic

**HITL**
- not normally required
- escalate if validation feedback requires business interpretation

**Context Scope**
- structured requirement model
- validation feedback for the current revision
- selected related tests only

**Failure**
- `GENERATION_FAILED`
- `REQUIRES_CLARIFICATION`

---

## Agent 04 — Coverage Validation Specialist

**Role:** Independently validate generated tests against requirements.

**Goal:** Prevent incomplete, weak, or unsupported test suites from moving forward.

**Inputs**
- `StructuredRequirementModel`
- `GeneratedTestSuite`

**Outputs**
- `CoverageValidationResult`
- requirement-to-test mapping
- missing coverage
- duplicate/redundant logical scenarios
- invalid assumptions
- revision feedback

**Tools**
- Confluence read-only
- Jira read-only
- Zephyr read-only if existing coverage is relevant

**Permissions**
- Read only

**Allowed**
- challenge generated tests
- identify missing scenarios
- identify unsupported assumptions
- identify weak expected results
- request a revision

**Forbidden**
- change requirements
- approve uncertain coverage without escalation
- publish tests

**Receives From**
- Test Generation Specialist

**Hands Off To**
- Test Generation Specialist on `REVISE`
- Mongo staging on `PASS`
- Human Gate on unresolved ambiguity

**Parallel**
- can validate independent test groups in parallel, followed by aggregate coverage validation

**HITL**
- when a coverage decision depends on unresolved requirement meaning

**Context Scope**
- requirement model
- generated suite
- previous validation history if needed

**Failure**
- `PASS`
- `REVISE`
- `NEED_HUMAN_REVIEW`

---

# Team 2 — Test Composition & Capabilities

## Agent 05 — Test Intent Specialist

**Role:** Convert natural-language or logical test cases into automation-oriented intent.

**Goal:** Express what the test must do without binding it to implementation details too early.

**Inputs**
- user test request or logical `TestCase`
- session context
- optional current project/module context

**Outputs**
- `StructuredTestIntent`

**Tools**
- internal test store read-only
- session context service

**Permissions**
- Read only

**Allowed**
- identify actions
- identify validations/assertions
- identify ordering constraints
- identify required data
- identify dependencies
- identify environment-related needs at a high level

**Forbidden**
- invent code capabilities
- write automation implementation

**Receives From**
- UI/chat
- staged logical test flow

**Hands Off To**
- Capability Discovery Specialist

**Parallel**
- separate test cases can be processed in parallel

**HITL**
- only if user intent is materially ambiguous

**Context Scope**
- current test/request
- project/session metadata

**Failure**
- `READY`
- `REQUEST_CLARIFICATION`

---

## Agent 06 — Capability Discovery Specialist

**Role:** Discover reusable automation capabilities in the existing codebase.

**Goal:** Match test intent to trusted existing automation wrappers before any new code is created.

**Inputs**
- `StructuredTestIntent`

**Outputs**
- `CapabilityResolutionMap`
- missing capability list
- candidate matches with confidence/evidence

**Tools**
- Graphify MCP:
  - `query_graph`
  - `get_node`
  - `get_neighbors`
  - `shortest_path` when needed
- repository metadata read-only

**Permissions**
- Read only

**Allowed**
- search Graphify
- inspect capability contracts/docstrings
- inspect call relationships
- identify dependencies
- identify reusable wrappers
- identify missing capability

**Forbidden**
- modify repository
- fabricate a capability
- call arbitrary low-level functions as a replacement for an approved capability unless policy explicitly permits it

**Receives From**
- Test Intent Specialist

**Hands Off To**
- Test Composition Specialist if complete
- Missing Capability Specialist if gaps exist

**Parallel**
- can run in parallel with duplicate pre-check
- separate test cases can run independently

**HITL**
- if multiple semantically different capabilities are equally plausible and choosing incorrectly changes test meaning

**Context Scope**
- test intent
- returned Graphify subgraph only
- capability catalog metadata

**Failure**
- `COMPLETE`
- `MISSING_CAPABILITY`
- `AMBIGUOUS_MATCH`

---

## Agent 07 — Test Composition Specialist

**Role:** Build an executable automation test from approved capabilities.

**Goal:** Compose existing wrappers into a valid test workflow.

**Inputs**
- `StructuredTestIntent`
- `CapabilityResolutionMap`

**Outputs**
- `ExecutableTestDefinition`

**Tools**
- Graphify MCP read-only
- capability catalog
- generated-test workspace

**Permissions**
- Read automation infrastructure metadata
- Write only generated test artifacts/workspace

**Allowed**
- order capability calls
- pass outputs into later inputs
- define assertions
- bind parameters
- add required preconditions
- build generated test code/configuration

**Forbidden**
- change shared automation framework
- create unapproved infrastructure capabilities
- bypass missing-capability flow

**Receives From**
- Capability Discovery Specialist

**Hands Off To**
- Composition Validation Specialist

**Parallel**
- separate tests can be composed in parallel

**HITL**
- not normally required

**Context Scope**
- intent
- selected capability contracts
- minimal required code examples

**Failure**
- `COMPOSED`
- `INVALID_DEPENDENCY`
- `MISSING_REQUIRED_INPUT`

---

## Agent 08 — Composition Validation Specialist

**Role:** Validate generated automation before runtime.

**Goal:** Reject structurally invalid or incompatible test compositions before environment allocation/execution.

**Inputs**
- `ExecutableTestDefinition`
- referenced capability contracts

**Outputs**
- `CompositionValidationResult`

**Tools**
- Graphify read-only
- parser/compiler
- linter/static validator
- non-destructive dry-run validator when available

**Permissions**
- Read only
- temporary validation execution only

**Allowed**
- schema validation
- parameter validation
- dependency validation
- capability compatibility validation
- static/dry validation

**Forbidden**
- modify shared infrastructure
- silently repair missing infrastructure code

**Receives From**
- Test Composition Specialist

**Hands Off To**
- Environment Requirement Specialist on valid
- Test Composition Specialist on revision
- Missing Capability Specialist on infrastructure gap

**Parallel**
- static checks may internally run in parallel

**HITL**
- rare; only unresolved semantic mismatch

**Failure**
- `VALID`
- `REVISE`
- `MISSING_CAPABILITY`
- `NEED_HUMAN_REVIEW`

---

# Team 3 — Automation Engineering

## Agent 09 — Missing Capability Specialist

**Role:** Convert an automation gap into a precise engineering requirement.

**Goal:** Define what must be added to the automation framework.

**Inputs**
- missing capability description
- structured test intent
- Graphify search evidence
- related capability context

**Outputs**
- `CapabilityDevelopmentSpecification`
- Jira automation-development ticket reference

**Tools**
- Graphify MCP
- Jira MCP
- repository search read-only

**Permissions**
- Read repository/Graphify/Jira
- Write Jira engineering ticket

**Allowed**
- locate related code
- identify reusable functions
- define wrapper interface
- define inputs/outputs
- define preconditions
- define acceptance criteria
- define capability contract/docstring requirements
- create ticket

**Forbidden**
- implement production code
- merge code

**Receives From**
- Capability Discovery Specialist
- Composition Validation Specialist
- Failure Triage Specialist for automation defects

**Hands Off To**
- Automation Development Specialist

**Parallel**
- separate missing capabilities can be analyzed independently

**HITL**
- optional ticket approval in early rollout
- required if requested capability implies destructive/high-risk functionality

**Context Scope**
- missing action
- relevant Graphify neighborhood
- coding/capability standards

**Failure**
- `SPEC_CREATED`
- `INSUFFICIENT_CONTEXT`
- `NEED_HUMAN_REVIEW`

---

## Agent 10 — Automation Development Specialist

**Role:** Implement new reusable automation capability.

**Goal:** Add the smallest safe reusable wrapper required to close the capability gap.

**Inputs**
- `CapabilityDevelopmentSpecification`
- Jira ticket
- relevant source context

**Outputs**
- branch/changeset
- tests
- capability docstring/contract
- pull request

**Tools**
- Git/Bitbucket
- Graphify MCP
- Jira MCP
- test runner
- linter
- static analysis
- build tools

**Permissions**
- Read repository
- Write development branch
- Create pull request
- No merge
- No protected branch write

**Allowed**
- reuse existing infrastructure
- implement wrapper
- add tests
- add documentation
- update generated capability metadata
- create PR

**Forbidden**
- merge own PR
- bypass validation
- modify production
- expose secrets

**Receives From**
- Missing Capability Specialist

**Hands Off To**
- Capability Contract Validator
- test/static-analysis workers

**Parallel**
- unit tests/static analysis/documentation checks can run in parallel after implementation

**HITL**
- not during implementation by default
- human merge approval later

**Context Scope**
- ticket
- related code
- Graphify neighborhood
- coding standards

**Failure**
- `IMPLEMENTED`
- `BUILD_FAILED`
- `TEST_FAILED`
- `BLOCKED`

---

## Agent 11 — Capability Contract Validator

**Role:** Validate that a new capability is understandable and discoverable by future agents.

**Goal:** Ensure working code is also a valid agent-facing capability.

**Inputs**
- implementation PR
- capability specification

**Outputs**
- `CapabilityContractValidationResult`

**Tools**
- Graphify MCP
- Git/Bitbucket read-only
- static parser/validator

**Permissions**
- Read only

**Allowed**
- validate capability name
- validate docstring format
- validate supported values
- validate inputs/outputs
- validate preconditions
- validate related-capability metadata
- validate Graphify discoverability after test indexing if available

**Forbidden**
- change implementation directly
- approve missing semantic contract

**Receives From**
- Automation Development Specialist

**Hands Off To**
- Automation Code Review Specialist on pass
- Development Specialist on revision

**Parallel**
- can run alongside unit tests and static analysis

**HITL**
- not normally required

**Context Scope**
- capability spec
- changed code only
- relevant related capabilities

**Failure**
- `CONTRACT_VALID`
- `CONTRACT_INVALID`

---

## Agent 12 — Automation Code Review Specialist

**Role:** Perform independent AI-assisted code review.

**Goal:** Validate correctness, reuse, safety, regression risk, and architecture fit before human review.

**Inputs**
- implementation PR
- capability specification
- test results
- contract validation

**Outputs**
- `CodeReviewResult`

**Tools**
- Git/Bitbucket read-only/review
- Graphify MCP
- test results
- static analysis results

**Permissions**
- Read
- PR comment/review
- No merge

**Allowed**
- review architecture
- detect duplication
- detect unsafe changes
- check backward compatibility
- review tests
- request changes

**Forbidden**
- merge
- self-approve final production change

**Receives From**
- contract validator
- automated checks

**Hands Off To**
- Development Specialist on changes
- Human PR Gate on approval

**Parallel**
- final review occurs after required automated checks join

**HITL**
- mandatory human approval before merge in v1

**Context Scope**
- PR diff
- related code graph
- ticket
- validation results

**Failure**
- `APPROVE`
- `REQUEST_CHANGES`
- `NEED_HUMAN_REVIEW`

---

# Team 4 — Environment & Execution

## Agent 13 — Environment Requirement Specialist

**Role:** Derive the environment state required by an executable test.

**Goal:** Convert the test into an explicit environment requirement profile.

**Inputs**
- `ExecutableTestDefinition`
- capability metadata

**Outputs**
- `EnvironmentRequirementProfile`

**Tools**
- capability metadata
- environment policy catalog read-only
- Graphify read-only if capability implementation details matter

**Permissions**
- Read only

**Allowed**
- identify required services
- versions
- feature flags
- configuration
- devices
- test data
- accounts
- dependencies

**Forbidden**
- choose/change environment before profile exists

**Receives From**
- Composition Validation Specialist

**Hands Off To**
- Environment Matching Specialist

**Parallel**
- can run in parallel with other non-dependent quality checks after composition

**HITL**
- not normally required

**Context Scope**
- executable test
- capability requirements

**Failure**
- `PROFILE_READY`
- `UNKNOWN_REQUIREMENT`

---

## Agent 14 — Environment Matching Specialist

**Role:** Select the best available environment for the test.

**Goal:** Compare required state with actual state and return a safe execution target.

**Inputs**
- `EnvironmentRequirementProfile`
- environment inventory/snapshots
- optional user-selected environment

**Outputs**
- `EnvironmentMatchDecision`

**Tools**
- Kubernetes MCP read-only
- environment APIs read-only
- deployment metadata
- configuration-state tool
- device inventory where relevant

**Permissions**
- Read only

**Allowed**
- inspect candidates
- compare versions/configuration
- rank suitable environments
- calculate configuration delta

**Forbidden**
- change environment
- choose an environment that violates hard requirements

**Receives From**
- Environment Requirement Specialist
- UI if user requested a specific environment

**Hands Off To**
- Test Execution Specialist when ready
- Environment Configuration Specialist when configurable
- Human/future Provisioning flow if none exists

**Parallel**
- candidate environment inspections may run in parallel

**HITL**
- when policy requires user choice among constrained environments

**Context Scope**
- required profile
- candidate snapshots only

**Failure**
- `READY`
- `CONFIGURABLE`
- `NO_ENVIRONMENT`
- `NEED_HUMAN_REVIEW`

---

## Agent 15 — Environment Configuration Specialist

**Role:** Prepare a selected environment for execution.

**Goal:** Apply the minimum allowed configuration delta and verify readiness.

**Inputs**
- selected environment
- required delta
- environment policy
- approval if required

**Outputs**
- `EnvironmentConfigurationResult`
- before/after snapshot
- applied changes

**Tools**
- Kubernetes MCP
- configuration APIs
- environment-management tools
- monitoring/health checks

**Permissions**
- Limited write
- Environment-specific policy restrictions

**Allowed**
- approved feature flag changes
- approved settings changes
- approved service operations
- health verification

**Forbidden**
- production changes unless explicitly supported by future policy
- unrelated changes
- destructive changes without approval
- broad cluster-admin behavior

**Receives From**
- Environment Matching Specialist
- Human Gate where required

**Hands Off To**
- Test Execution Specialist on ready
- alternative environment search/human on failure

**Parallel**
- independent safe configuration tasks may run in parallel only if non-conflicting

**HITL**
- shared environment impact
- service restart with cross-team effect
- destructive operation
- high-risk settings
- future environment provisioning

**Context Scope**
- selected environment only
- requested delta
- current snapshot

**Failure**
- `READY`
- `CONFIGURATION_FAILED`
- `ROLLBACK_REQUIRED`
- `NEED_HUMAN_REVIEW`

---

## Agent 16 — Test Execution Specialist

**Role:** Execute an approved test on an approved environment.

**Goal:** Perform the test and capture complete execution evidence.

**Inputs**
- executable test definition
- environment snapshot
- runtime data
- execution policy

**Outputs**
- `ExecutionPackage`

**Tools**
- automation test runner
- device infrastructure
- Jenkins/execution backend if used
- logs
- artifact capture
- screenshots/video capture

**Permissions**
- Execute approved test only
- No arbitrary infrastructure changes

**Allowed**
- launch execution
- monitor execution
- capture logs
- capture screenshots/video
- collect API responses
- store artifacts

**Forbidden**
- rewrite test
- change expected results
- silently switch environment
- modify shared automation framework

**Receives From**
- Environment Matching/Configuration flow

**Hands Off To**
- Runtime Validation Specialist

**Parallel**
- independent tests may execute in parallel subject to environment/device capacity

**HITL**
- only for scarce/expensive/destructive execution policies

**Context Scope**
- current execution only

**Failure**
- `COMPLETED`
- `EXECUTION_ERROR`
- `TIMEOUT`
- `INFRASTRUCTURE_ERROR`

---

## Agent 17 — Runtime Validation Specialist

**Role:** Determine whether observed runtime behavior satisfies expected results.

**Goal:** Produce an evidence-backed PASS/FAIL/INCONCLUSIVE decision.

**Inputs**
- `ExecutionPackage`
- expected results
- validation rules

**Outputs**
- `RuntimeValidationResult`

**Tools**
- structured runner results
- logs
- APIs
- monitoring
- screenshot/image analysis
- multimodal local model

**Permissions**
- Read only

**Validation Order**
1. deterministic assertions
2. structured API/runner outputs
3. logs/telemetry
4. multimodal/visual reasoning
5. human review if still inconclusive

**Allowed**
- compare expected vs actual
- combine evidence
- identify missing evidence
- return confidence and rationale

**Forbidden**
- change expected results to create a pass
- modify environment/test during validation

**Receives From**
- Test Execution Specialist

**Hands Off To**
- Test Quality & Release Specialist on pass
- Failure Triage Specialist on fail
- Human/Triage on inconclusive

**Parallel**
- log analysis, API validation, screenshot validation, and telemetry checks can run in parallel and join

**HITL**
- inconclusive evidence
- low-confidence visual-only result

**Context Scope**
- current execution evidence only

**Failure**
- `PASS`
- `FAIL`
- `INCONCLUSIVE`

---

# Team 5 — Quality & Lifecycle

## Agent 18 — Failure Triage Specialist

**Role:** Classify why a test failed.

**Goal:** Route failures to the correct remediation workflow instead of treating every failure as a product bug.

**Inputs**
- failed execution package
- runtime validation
- environment snapshot
- test definition
- relevant recent code/environment changes when available

**Outputs**
- `FailureAnalysis`

**Tools**
- Graphify MCP
- logs
- monitoring
- Git/Bitbucket read-only
- Jira read-only
- environment read tools

**Permissions**
- Read only by default

**Classification**
- `PRODUCT_DEFECT`
- `AUTOMATION_DEFECT`
- `ENVIRONMENT_ISSUE`
- `TEST_DATA_ISSUE`
- `REQUIREMENT_ISSUE`
- `FLAKY_TEST`
- `UNKNOWN`

**Allowed**
- collect evidence
- classify
- recommend next route
- propose defect/ticket content

**Forbidden**
- repair all categories itself
- create official product defect without required approval policy
- modify code/environment directly

**Receives From**
- Runtime Validation Specialist

**Hands Off To**
- Automation Engineering for automation defect
- Environment Team for environment issue
- Requirements flow for requirement issue
- Human/Jira defect flow for product defect
- retry policy for flaky/infrastructure cases

**Parallel**
- evidence collection tasks can run in parallel

**HITL**
- product defect publication in v1
- unknown/low-confidence classification

**Context Scope**
- failed test/run
- relevant evidence only

**Failure**
- classification + confidence
- `NEED_HUMAN_REVIEW`

---

## Agent 19 — Test Quality & Release Specialist

**Role:** Decide whether a test is ready to become an official managed Zephyr test.

**Goal:** Prevent low-quality, broken, duplicate, unstable, or unresolved tests from polluting the official test repository.

**Inputs**
- staged test record
- requirement validation
- composition validation
- execution history
- runtime validation history
- duplicate/similarity results
- unresolved failures

**Outputs**
- `ReleaseDecision`

**Tools**
- MongoDB/internal store read
- Zephyr search/read
- Jira read
- similarity search service

**Permissions**
- Read external systems
- Write lifecycle status to internal store only

**Allowed**
- duplicate check
- near-duplicate assessment
- stability review
- quality gate
- decide whether to create new test or update existing candidate
- request human review

**Forbidden**
- write directly to Zephyr
- delete existing Zephyr tests
- publish unresolved test

**Receives From**
- validated runtime flow
- staging lifecycle
- duplicate pre-check results

**Hands Off To**
- Zephyr Sync Worker when `READY_TO_PUBLISH`
- Human Gate on uncertain duplicate/update decision
- hold/revision flow otherwise

**Parallel**
- duplicate candidate search can begin earlier
- final release decision waits for required evidence

**HITL**
- uncertain duplicate/update decision
- policy-sensitive replacement of existing test
- low-confidence quality result

**Context Scope**
- test lifecycle summary, not full raw history unless requested

**Failure**
- `READY_TO_PUBLISH`
- `DUPLICATE`
- `UPDATE_EXISTING`
- `HOLD`
- `REJECT`
- `NEED_HUMAN_REVIEW`

---

# Cross-System Agent

## Agent 20 — Contextual QA Chat Agent

**Role:** Natural-language interface over the QA platform.

**Goal:** Allow the user to ask questions and request actions using normal language while preserving workflow safety.

**Inputs**
- user message
- session context
- current PRD/test/workflow/environment/run references

**Outputs**
- answer
- clarification request
- structured workflow handoff request

**Tools — Read Only by Default**
- Confluence MCP
- Jira MCP
- Zephyr MCP
- Graphify MCP
- internal test store
- environment state
- execution history
- workflow/session status

**Permissions**
- Read only by default
- no privileged direct action

**Allowed**
- answer contextual questions
- explain current workflow
- explain failures
- explain environment mismatch
- explain available capabilities
- search related tests/requirements
- route requested actions to the correct subgraph

**Forbidden**
- directly modify environment
- directly run privileged tests
- directly merge code
- directly publish to Zephyr
- bypass approval policy

**Example**

```text
User: "Why can't this test run on QA-3?"
Chat Agent:
- resolves "this test" from session context
- reads its EnvironmentRequirementProfile
- reads QA-3 snapshot
- explains mismatch

User: "Configure QA-3 and run it."
Chat Agent:
- creates structured handoff
- Environment & Execution subgraph takes control
```

**HITL**
- chat itself: no
- downstream workflows apply normal approval policy

**Context Scope**
- session summary
- explicit current references
- dynamically retrieved context
- never load all platform data into one prompt

**Failure**
- `ANSWER`
- `HANDOFF`
- `CLARIFY`
- `NOT_AUTHORIZED`
