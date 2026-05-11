# The Tool Executor Pattern

## Problem

Agents calling enterprise tools directly create an uncontrolled blast radius. A tool call that queries a read-only knowledge base carries a fundamentally different risk profile than one that sends an external communication, modifies an access control list, or executes a financial transaction — yet without a centralised enforcement layer, both would be subject only to whatever the calling Worker happened to check before making the call. The Tool Executor exists because risk-class enforcement cannot be left to the discretion of individual Workers. It must be applied uniformly, at invocation time, by a component that runs outside the model and cannot be influenced by model output.

## Four Risk Classes

| Risk Class | Examples | Default Approval | Audit Level |
|---|---|---|---|
| READ | Database queries, document retrieval, status checks | Auto-approved | Standard |
| WRITE_INTERNAL | Creating internal records, updating workflow state | Auto-approved (with schema validation) | Enhanced |
| WRITE_EXTERNAL | Sending email, posting to Slack, calling partner APIs | HITL required | Full |
| IRREVERSIBLE | Deleting records, archiving documents, financial transactions | HITL required + second approver | Full + immutable |

READ operations are auto-approved in all deployments. They carry no side effects on external systems and their audit records are retained at standard retention. WRITE_INTERNAL operations are auto-approved but subject to schema validation of the action parameters before invocation; their audit records are retained at enhanced retention to support change history queries. WRITE_EXTERNAL operations require a human approver before execution. The approval request includes the full action context — calling Worker identity, requesting user identity, tool ID, and the action parameters — so the approver can make an informed decision. IRREVERSIBLE operations require both a primary approver and a second approver from a separate role; their audit records are written to an immutable store and retained at the full statutory period.

## Invocation Flow

```
  Worker           Tool Executor        Policy Engine      Audit Store
    │                    │                   │                 │
    │──tool_request──────►│                   │                 │
    │                    │──check_risk_class──►│                 │
    │                    │◄──class: READ──────│                 │
    │                    │──evaluate_policy───►│                 │
    │                    │◄──permit───────────│                 │
    │                    │──────────────────────────emit_event──►│
    │                    │──invoke_tool()                        │
    │◄──tool_result───────│                   │                 │
```

The audit event is emitted before tool invocation, not after. This ensures a record exists even if the invocation fails mid-execution — a property that is essential for IRREVERSIBLE operations where partial execution must be detectable.

## Policy Enforcement

Policies are evaluated by Cedar or OPA, running as a separate process outside the model, before any tool invocation. The policy engine receives: the Worker identity (a service principal), the tool ID, the risk class, the requesting user identity propagated via an on-behalf-of token, and the action parameters. The policy engine evaluates these inputs against the configured policy set and returns a permit or deny decision. The model cannot observe this evaluation, cannot influence its inputs, and cannot override its output. If the policy engine is unavailable, the Tool Executor fails closed — no tool invocations are permitted until the policy engine is reachable.

## Audit Event Structure

```json
{
  "event_type": "tool_invocation",
  "timestamp": "ISO-8601",
  "trace_id": "string",
  "span_id": "string",
  "worker_id": "string",
  "tool_id": "string",
  "risk_class": "READ|WRITE_INTERNAL|WRITE_EXTERNAL|IRREVERSIBLE",
  "user_identity": "string",
  "agent_identity": "string",
  "policy_decision": "permit|deny",
  "hitl_required": "boolean",
  "hitl_approver": "string|null",
  "outcome": "success|failure",
  "error": "string|null"
}
```

Every field in the audit event is populated before the invocation result is known, except `outcome` and `error`, which are updated when the invocation completes. The `trace_id` and `span_id` fields link the audit event to the distributed trace for the parent agent run, enabling full reconstruction of a multi-step execution from the audit log alone.
