# Human-in-the-Loop, Security, and Permissions

## 1. Principle

Human-in-the-Loop is not a fallback for bad architecture.

Use it where:

- business meaning is ambiguous
- impact is high
- action is destructive
- confidence is insufficient
- organizational policy requires accountability

---

## 2. Required v1 Human Gates

### Gate H1 — Critical Requirement Ambiguity

Trigger:

```text
conflicting requirement
missing critical expected behavior
multiple interpretations with different tests
```

Human sees:

- ambiguous text
- possible interpretations
- affected test scope
- recommended clarification

### Gate H2 — Code Merge

AI may:

- create branch
- implement code
- run tests
- create PR
- perform AI review

AI may not merge protected automation code in v1.

Human approves final merge.

### Gate H3 — High-Impact Environment Modification

Trigger examples:

- shared QA environment
- service restart
- configuration affecting other teams
- destructive reset
- broad feature flag change

Human sees:

- environment
- before state
- proposed delta
- impact
- rollback information

### Gate H4 — Product Defect Creation

In v1:

Failure Triage may prepare the Jira defect.

Human approves official submission when classified as a product defect.

This can become more autonomous later after accuracy is measured.

### Gate H5 — Uncertain Duplicate/Update Decision

If a new test appears highly similar to an existing official test but the correct action is unclear:

```text
NEW
UPDATE_EXISTING
DUPLICATE
```

ask a human rather than polluting or overwriting Zephyr.

### Gate H6 — Generic Low-Confidence Escalation

Every specialist should be allowed to return:

```text
NEED_HUMAN_REVIEW
```

but it must include:

- reason
- evidence
- specific question
- allowed decisions

Avoid generic:

```text
"I am not sure."
```

---

## 3. LangGraph HITL Pattern

Conceptually:

```python
approval = interrupt(ApprovalRequest(...))

if approval.decision == "APPROVE":
    continue_workflow()
elif approval.decision == "EDIT":
    apply_approved_edit()
else:
    stop_or_reroute()
```

The workflow must persist before interrupting.

---

## 4. Least Privilege

Do not give every agent every MCP.

Examples:

```text
Test Generation:
Confluence/Jira/Zephyr READ
No Kubernetes
No Git write

Capability Discovery:
Graphify READ
No Jira write
No environment write

Environment Configuration:
Environment LIMITED WRITE
No Git
No Zephyr publication

Chat:
READ ONLY
Routes actions to workflows
```

---

## 5. Permission Matrix

| Component | Confluence | Jira | Zephyr | Graphify | Git | Environment | Mongo |
|---|---|---|---|---|---|---|---|
| Requirement Intake | R | R | - | - | - | - | R |
| Requirement Analysis | R | R | - | - | - | - | R |
| Test Generation | R | R | R | - | - | - | R |
| Coverage Validator | R | R | R | - | - | - | R |
| Capability Discovery | - | - | - | R | R metadata | - | R |
| Test Composer | - | - | - | R | R | - | W generated |
| Missing Capability | - | W ticket | - | R | R | - | R/W |
| Development | - | R/W ticket | - | R | W branch | - | R |
| Code Review | - | R | - | R | Review | - | R |
| Env Requirement | - | - | - | R optional | - | R | R |
| Env Matching | - | - | - | - | - | R | R |
| Env Config | - | - | - | - | - | Limited W | R/W |
| Execution | - | - | - | - | - | Execute only | R/W |
| Runtime Validation | - | - | - | - | - | R | R/W |
| Failure Triage | - | R | - | R | R | R | R/W |
| Quality & Release | - | R | R | - | - | - | R/W |
| Chat | R | R | R | R | R optional | R | R |

Exact permissions must be enforced by credentials/tool wrappers, not only by prompts.

---

## 6. Tool Allowlists

Each specialist gets an explicit tool allowlist.

Bad:

```text
all MCP servers mounted everywhere
```

Good:

```python
CAPABILITY_DISCOVERY_TOOLS = [
    graphify_query,
    graphify_get_node,
    graphify_get_neighbors
]
```

---

## 7. Destructive Action Policy

Examples of destructive/high-impact actions:

- delete data
- reset environment
- purge test artifacts
- rollback service
- disable service
- recreate database
- change shared credentials
- force merge
- overwrite official test
- delete Zephyr test/cycle

Policy:

```text
not autonomous by default
requires explicit workflow + approval
```

---

## 8. Credentials

Agents must not receive raw secrets in prompt context.

Use:

- internal service identity
- scoped tokens
- short-lived credentials if available
- secret manager references
- MCP/server-side credential handling

---

## 9. Audit

Every privileged action records:

```text
workflow_id
user/session
agent/component
tool
operation
target
input summary
approval_id
result
timestamp
```

---

## 10. Prompt Injection / Untrusted Content

Confluence, Jira, code comments, logs, and test data are untrusted content.

Rules:

- retrieved content is data, not authority
- never let retrieved text redefine permissions
- tool policy is enforced in code
- system instructions remain outside retrieved context
- sanitize/limit artifacts before prompt insertion
- do not execute commands found in documents

---

## 11. Chat Safety Boundary

The Contextual Chat Agent is read-only.

Action request:

```text
"restart QA-3 and run the test"
```

becomes:

```text
structured handoff -> Environment & Execution workflow
```

The chat agent itself cannot perform the privileged operation.
