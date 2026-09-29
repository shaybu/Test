# Graphify Capability Layer

## 1. Purpose

Graphify is the code knowledge layer for the QA automation framework.

It should answer questions such as:

- What reusable capability creates a new case?
- What function sends a message?
- Which wrapper verifies the case timeline?
- What does `create_new_case` depend on?
- Is there already a capability similar to the missing requirement?
- Where should a new capability logically be implemented?

Graphify does **not** execute the automation.

It helps agents understand and navigate the automation codebase.

---

## 2. Capability Layer Principle

Automation engineers continue owning the real framework.

They expose stable, reusable wrappers that are easy for agents to discover.

Example:

```python
def create_new_case(...):
    """
    Capability: Create New Case

    Purpose:
        Creates a new case in the application.

    Supported case types:
        criminal
        civil
        financial

    Requires:
        authenticated user

    Inputs:
        case_type
        name
        classification

    Returns:
        Case object containing case_id.

    Related capabilities:
        send_message
        add_evidence
        change_case_status
    """
```

The implementation behind the wrapper can be complex.

The agent should not care.

---

## 3. Agent-Facing vs Internal Functions

Two layers must remain distinct.

```text
Agent-facing capability layer:
create_new_case()
send_message()
verify_timeline()
close_case()

Internal implementation:
click_element()
post_api()
create_payload()
wait_for_event()
parse_response()
set_dropdown()
```

Agents should prefer the capability layer.

They should not compose arbitrary low-level internal functions merely because Graphify can see them.

This boundary protects:

- stability
- readability
- maintainability
- security
- reuse
- test intent

---

## 4. Capability Contract Convention

Adopt one standardized docstring format.

Recommended:

```text
Capability
Purpose
Inputs
Supported Values
Preconditions
Returns
Side Effects
Related Capabilities
Environment Requirements
Failure Modes
Safety Classification
```

Example:

```python
def send_case_message(case_id: str, channel: str, text: str):
    """
    Capability: Send Case Message

    Purpose:
        Sends a message associated with an existing case.

    Inputs:
        case_id: Existing case identifier.
        channel: Delivery channel.
        text: Message content.

    Supported Values:
        channel:
          - internal
          - sms
          - email

    Preconditions:
        - Case exists
        - User is authenticated
        - Selected environment supports requested channel

    Returns:
        message_id
        delivery_status

    Side Effects:
        Creates a message record.

    Related Capabilities:
        create_new_case
        verify_case_message
        verify_timeline_event

    Environment Requirements:
        messaging service enabled

    Failure Modes:
        invalid_case
        unsupported_channel
        delivery_failure

    Safety Classification:
        normal
    """
```

---

## 5. Discovery Flow

```text
StructuredTestIntent
      |
Capability Discovery Specialist
      |
Graphify query
      |
candidate nodes
      |
inspect node + neighbors
      |
map action -> capability
      |
CapabilityResolutionMap
```

Recommended MCP operations:

```text
query_graph
get_node
get_neighbors
shortest_path
```

Use narrow queries.

Do not dump the full graph into model context.

---

## 6. Missing Capability Flow

Example user intent:

```text
create a case
send WhatsApp message
verify timeline
```

Discovery result:

```text
create_new_case       FOUND
verify_timeline       FOUND
send_whatsapp_message MISSING
```

Flow:

```text
MISSING
 |
Missing Capability Specialist
 |
Graphify:
- find related messaging functions
- find existing send_message wrappers
- find correct module
 |
CapabilityDevelopmentSpecification
 |
Jira ticket
 |
Development
 |
Contract validation
 |
Review + human merge
 |
Graphify update
 |
capability becomes discoverable
 |
resume original composition
```

---

## 7. Graph Update Strategy

The code graph must be kept current.

Recommended:

```text
Git/Bitbucket merge
      |
Repository webhook/CI event
      |
Graphify Update Worker
      |
update graph
      |
health/sanity check
      |
mark graph version active
```

Do not rely on an agent remembering to refresh the graph.

---

## 8. Graph Versioning

Store:

```text
graph_version
repository_commit
branch
generated_at
status
```

Capability resolution should record which graph version was used.

This is important for reproducibility.

---

## 9. Graphify Availability Policy

If Graphify is temporarily unavailable:

- do not fabricate capability discovery
- mark capability resolution as unavailable
- retry according to infrastructure policy
- allow manual fallback if required

---

## 10. Security

Graphify should normally be read-only for discovery agents.

If using a shared HTTP MCP server:

- internal network only
- authentication required
- project scoping
- query logging
- no source secrets in indexed artifacts
- review indexed file types
- restrict cross-project graph access

---

## 11. Capability Quality Metrics

Track:

- percentage of test actions resolved to existing capabilities
- missing capability rate
- ambiguous match rate
- top missing capabilities
- capability reuse count
- capability discovery latency
- tests broken by capability changes
- duplicate capability implementation attempts

These metrics tell the automation team where to improve the reusable framework.
