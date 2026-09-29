QA Agentic Platform Architecture v1
Purpose
This document set defines an implementation-ready architecture for an on-premise, open-source QA Agentic Platform.
The platform hides the complexity of the existing QA automation infrastructure behind a natural-language user experience. A QA user should be able to ask for outcomes such as:
- "Generate test cases for this PRD."
- "Build an automated test that creates a case, sends a message, and verifies the result."
- "Why did this test fail?"
- "Can this test run on QA-3?"
- "Run this test on a suitable environment."
The user should not need to know the automation framework internals, repository structure, environment configuration, Jira/Zephyr APIs, or low-level helper functions.
The platform converts natural-language intent into controlled, traceable workflows executed by specialist agent teams and deterministic services.
Core Architecture Decisions
1. LangGraph is the orchestration backbone.
   - The main workflow is a graph, not an autonomous mega-agent.
   - It controls state, routing, loops, parallel execution, retries, checkpoints, and Human-in-the-Loop.
   - Teams are implemented as LangGraph subgraphs.
2. Agents are specialists, not general-purpose super-agents.
   - Each specialist has a narrow role.
   - Each specialist receives only the tools and context required for that role.
   - Least privilege is enforced.
3. Pydantic is the contract layer.
   - Agent inputs and outputs are structured Pydantic models.
   - Workflow state, environment state, validation results, release decisions, approval requests, and persistence models are typed.
   - Free-form text is not the primary interface between internal components.
4. Graphify is the automation capability knowledge layer.
   - Existing automation code is mapped into a queryable code graph.
   - Agent-facing wrapper functions expose reusable automation capabilities.
   - Capability docstrings act as semantic contracts.
   - Agents discover capabilities instead of inventing automation logic.
5. MongoDB is the operational/staging store.
   - Generated tests are not required to be written immediately to Zephyr.
   - Tests can be generated, validated, composed, executed, triaged, duplicate-checked, and quality-gated before publication.
   - Zephyr synchronization is asynchronous.
6. Zephyr synchronization is a worker, not an LLM agent.
   - Publishing is deterministic integration work.
   - It must be idempotent, retryable, observable, and non-blocking.
7. Environment management is a first-class workflow.
   - A test declares environment requirements.
   - The platform compares them with current environment state.
   - It can select another environment or request/configure changes.
   - Future versions may provision environments automatically.
8. Human-in-the-Loop is explicit and selective.
   - Human approval is used for high-risk or ambiguous decisions.
   - It is not added to every step.
9. The system is on-premise and model-agnostic.
   - Current local model candidates include GLM 5.3, GLM 5.3 Flash for multimodal tasks, Kimi, Gemma, and other internally approved models.
   - The architecture must not depend on external cloud-model APIs.
10. OpenAI Agents SDK is not required in v1.
    - It is open source and may be evaluated later.
    - Using it together with LangGraph in v1 would duplicate orchestration responsibilities without a demonstrated need.
    - v1 should remain simpler: LangGraph + local model adapter + MCP/tools.
Architecture at a Glance
                               QA Web UI
                                  |
                    +-------------+-------------+
                    |                           |
             Workflow Requests            Contextual Chat
                    |                           |
                    +-------------+-------------+
                                  |
                         Main LangGraph
                           Orchestrator
                                  |
        +-------------------------+-------------------------+
        |                         |                         |
 Requirements &             Test Composition          Automation
 Test Design Team           & Capability Team         Engineering Team
    Subgraph                   Subgraph                  Subgraph
        |                         |                         |
        +-------------------------+-------------------------+
                                  |
                         Environment &
                         Execution Team
                            Subgraph
                                  |
                         Quality & Lifecycle
                              Subgraph
                                  |
                             MongoDB
                                  |
                         Async Publish Queue
                                  |
                         Zephyr Sync Worker
                                  |
                            Jira / Zephyr
Shared systems:
Confluence MCP
Jira MCP
Zephyr MCP
Graphify MCP
Git / Bitbucket integration
Kubernetes / environment tools
Automation test runner
Logs / monitoring
MongoDB
Local LLM endpoints
Teams and Specialists
Team 1 — Requirements & Test Design
1. Requirement Intake Specialist
2. Requirement Analysis Specialist
3. Test Generation Specialist
4. Coverage Validation Specialist
Team 2 — Test Composition & Capabilities
5. Test Intent Specialist
6. Capability Discovery Specialist
7. Test Composition Specialist
8. Composition Validation Specialist
Team 3 — Automation Engineering
9. Missing Capability Specialist
10. Automation Development Specialist
11. Capability Contract Validator
12. Automation Code Review Specialist
Team 4 — Environment & Execution
13. Environment Requirement Specialist
14. Environment Matching Specialist
15. Environment Configuration Specialist
16. Test Execution Specialist
17. Runtime Validation Specialist
Team 5 — Quality & Lifecycle
18. Failure Triage Specialist
19. Test Quality & Release Specialist
Cross-System
20. Contextual QA Chat Agent
Non-Agent Components
The following should not be implemented as LLM agents unless a future requirement proves otherwise:
- MongoDB repository/persistence service
- Zephyr Sync Worker
- Graphify update/re-index worker
- Job queue
- Scheduler
- Retry manager
- Session store
- Authentication/authorization service
- Git merge operation
- Basic schema validation
- Deterministic environment health checks
- Metrics and audit logging
Document Set
- 01_ARCHITECTURE_AND_STACK.md — technology stack and architectural decisions
- 02_AGENT_TEAMS_AND_SPECIALISTS.md — full definition of all 20 agents
- 03_WORKFLOWS_AND_ORCHESTRATION.md — team subgraphs, routing, loops, parallelism, and handoffs
- 04_STATE_CONTEXT_AND_PYDANTIC.md — state model, context isolation, and structured contracts
- 05_GRAPHIFY_CAPABILITY_LAYER.md — capability wrappers, docstring contracts, discovery, and missing-capability flow
- 06_MONGODB_ZEPHYR_LIFECYCLE.md — operational test lifecycle and asynchronous publication
- 07_ENVIRONMENT_EXECUTION_VALIDATION.md — environment matching, configuration, execution, and validation
- 08_HITL_SECURITY_PERMISSIONS.md — Human-in-the-Loop, least privilege, and approval policy
- 09_IMPLEMENTATION_PLAN.md — recommended phased delivery plan
- 10_OBSERVABILITY_RELIABILITY_TESTING.md — telemetry, retries, evals, quality, and operations
- 11_DECISIONS_AND_OPEN_QUESTIONS.md — architecture decisions and unresolved design questions
- 12_LANGGRAPH_IMPLEMENTATION_SKELETON.md — implementation-oriented LangGraph structure and examples
- 99_REFERENCES_AND_LICENSE_NOTES.md — OSS/license notes
Guiding Principle
Use agents for reasoning and judgment.
Use tools for actions.
Use workers for deterministic asynchronous work.
Use services for stable platform capabilities.
Use LangGraph to control who runs, when, with what state and permissions.
