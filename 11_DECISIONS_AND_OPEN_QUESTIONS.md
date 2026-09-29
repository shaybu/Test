# Architecture Decisions and Open Questions

## Confirmed Decisions

### ADR-001 — LangGraph as v1 Orchestrator
Use LangGraph for the main workflow and team subgraphs.

### ADR-002 — Pydantic for Internal Contracts
All important handoffs use typed schemas.

### ADR-003 — Team/Subgraph Architecture
Five functional teams plus a cross-system chat agent.

### ADR-004 — 20 Specialist Agents
The initial logical specialist catalog contains 20 roles.

### ADR-005 — Main Orchestrator Is Not an Agent
Routing is code-first and deterministic where possible.

### ADR-006 — Graphify as Code Capability Knowledge Layer
Agents discover and understand automation capabilities through Graphify.

### ADR-007 — Capability Wrappers Are the Agent-Facing Automation API
Automation teams expose reusable high-level wrappers with structured docstring contracts.

### ADR-008 — MongoDB Is the Operational Test Store
Tests can mature internally before official Zephyr publication.

### ADR-009 — Zephyr Sync Is Asynchronous
Publication is performed by a deterministic worker.

### ADR-010 — Human Merge in v1
AI can create and review PRs, but final merge requires a human.

### ADR-011 — Environment State Is Explicit
Environment selection/configuration is based on typed snapshots and requirements.

### ADR-012 — Context Is Scoped Per Specialist
Sub-agents/specialists receive only relevant projected context.

### ADR-013 — Local Models Only
No architecture dependency on external model APIs.

### ADR-014 — Least Privilege Tool Access
Tool permissions are enforced by code/credentials, not prompts alone.

### ADR-015 — OpenAI Agents SDK Deferred
Do not mix agent frameworks until a measured need exists.

### ADR-016 — JEV-like Small Decision Model Deferred
Revisit only if repetitive semantic decisions become a measurable bottleneck.

---

## Open Question 1 — When Is a Test Official?

Choose publication policy:

```text
A. after design/coverage validation
B. after automation composition
C. after first successful execution
D. after stability threshold
```

The system can support all as policy modes.

A product decision is still required.

---

## Open Question 2 — Test Cycle Policy

When should the platform create/add to a Zephyr Test Cycle?

Options:

```text
user explicitly requests
after suite publication
after automation ready
before execution
release-specific workflow
```

---

## Open Question 3 — Duplicate Update Policy

If a similar Zephyr test exists:

```text
always ask human
automatically update if same requirement link
create new version
create separate test
```

Need organizational rule.

---

## Open Question 4 — Test Storage Format

Determine canonical internal test representation.

It should support:

- manual readable test
- automation binding
- Zephyr mapping
- requirement traceability
- version history

Avoid making Zephyr's schema the internal domain model.

Use an adapter.

---

## Open Question 5 — Generated Automation Artifact

Decide whether composed tests are:

```text
Python code
declarative YAML/JSON DSL
framework-native object
combination
```

A declarative intermediate representation may make validation and generation safer.

---

## Open Question 6 — Environment Ownership

Need metadata:

```text
shared vs dedicated
team owner
allowed mutation level
capacity
maintenance windows
```

Without this, automatic configuration policy is unsafe.

---

## Open Question 7 — Provisioning

Future decision:

- reuse existing internal provisioning
- Kubernetes namespace/environment templates
- infrastructure-as-code
- external team approval

Do not implement before actual provisioning workflow is understood.

---

## Open Question 8 — User Roles

Define permissions for:

```text
manual QA
automation QA
QA lead
developer
admin
environment owner
```

The UI and workflow policy should respect user role.

---

## Open Question 9 — Jira Ticket Types

Need exact mapping for:

```text
automation capability gap
product defect
requirement clarification
environment issue
test maintenance
```

---

## Open Question 10 — Graphify Capability Scope

Decide how wrappers are identified.

Possible convention:

```text
specific package/folder
decorator
docstring Capability field
naming convention
registry file
```

A machine-verifiable convention is preferable.

---

## Open Question 11 — Approval UX

The UI needs explicit approval cards, not generic chat messages.

Example:

```text
Action:
Restart case-service in QA-3

Reason:
Feature configuration requires restart

Impact:
Shared environment, estimated 2 minutes

[Approve] [Reject] [Edit]
```

---

## Open Question 12 — Model Assignment

Benchmark local models per role before final assignment.

Do not assume one model is best for every specialist.

---

## Open Question 13 — Persistence Backend for LangGraph Checkpoints

MongoDB is the operational domain store.

LangGraph checkpoint storage can be separate if technically beneficial.

Do not overload one database solely for conceptual simplicity.

---

## Open Question 14 — Source of Existing Test Similarity

Potential indexes:

- Zephyr search
- local mirrored test metadata
- embeddings/search index
- structured fingerprints

Choose based on scale and performance after measuring Zephyr search latency.

---

## Open Question 15 — Human Gate Reduction

After collecting accuracy metrics, some v1 gates may be relaxed.

Examples:

- product defect creation
- low-risk environment configuration
- capability ticket creation

Use measured precision and incident history before removing gates.
