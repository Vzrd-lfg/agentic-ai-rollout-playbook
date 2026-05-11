# Workflow Engine Integration

Agents operate most effectively when embedded in a durable workflow engine rather than when they attempt to implement workflow concerns themselves. The engine provides state persistence, retry logic, timeout management, and compensation — capabilities that are expensive and error-prone to implement in agent code, and that workflow engines have been designed and hardened to handle. The division of responsibility is clear: the workflow engine owns the business process, the long-running state, and the audit trail at the process level; the agent owns the cognitive step inside a single node, including the micro-orchestration of model calls and tool calls within its turn.

## Three Integration Patterns

The pattern chosen for a given agent depends on the scope of the agent's role in the broader process. An agent that produces a bounded artifact fits the Task pattern; an agent whose execution is itself multi-step fits the Subprocess pattern; an agent that determines a branch condition fits the Decision Gateway pattern. These patterns are not mutually exclusive — a complex process may use all three at different nodes.

### Agent as Task

The agent is invoked as a single step in a larger workflow. The workflow engine calls the agent API, waits for a response (synchronous for fast agents, polling or callback for agents that exceed a synchronous timeout), and routes the result to the next step as a workflow variable. This is the simplest integration and appropriate when the agent's output is a discrete, bounded artifact — a classification, a draft, an extraction — that the rest of the process consumes as a data value. The workflow engine handles retries and timeouts around the agent call; the agent need not implement these concerns internally.

### Agent as Subprocess

A workflow step spawns an entire agent workflow as a child workflow. The parent workflow waits for the child to complete and receives a structured result envelope. This pattern is appropriate when an agent's execution itself has multiple stages that benefit from durable state and retry logic — for example, a document intake pipeline where ingestion, parsing, entity extraction, and validation each need to survive a failure in any subsequent stage without re-running completed stages. The child workflow's state is managed by the engine, not by the agent process, which means the agent can be restarted mid-execution without data loss.

### Agent as Decision Gateway

The agent evaluates a condition and the workflow branches based on the result. The agent receives the relevant context, applies reasoning, and returns a structured decision — approve, reject, escalate, route-to-team — that the workflow engine uses to select the next path. This pattern is powerful for encoding judgment that is too contextual for a simple rule engine but too consequential to be embedded invisibly in agent logic. Because the agent's decision determines which workflow branch executes, it must be logged as a named, timestamped audit event with the full input context preserved, since it is a governance-relevant action that may be reviewed in a compliance or dispute context.

## State Handoff

The workflow engine and the agent exchange structured envelopes at the boundary of each invocation. The envelope content at each direction is defined below.

| Direction | Data | Format |
|---|---|---|
| Engine → Agent | workflow_id, step_id, input payload, context (prior steps' outputs), user identity | JSON envelope |
| Agent → Engine | output payload, status (complete/failed/hitl_required), next_step_hint, audit_reference | JSON envelope |

The `workflow_id` and `step_id` serve as the correlation keys that tie the workflow engine's process-level audit log to the agent's model-and-tool-level audit log. The `audit_reference` in the agent's response is the agent_run_id that can be joined against the agent's own audit trail in the observability stack. The `next_step_hint` is advisory; the workflow engine retains authority over routing.

## Compensation Patterns

When an agent step fails after a WRITE_INTERNAL or WRITE_EXTERNAL tool call has already executed, the workflow engine cannot simply retry from scratch — the write has already occurred and retrying would produce a duplicate. Each agent step that performs a write must declare a compensation action in its contract: the undo operation that the workflow engine should invoke if the step or any subsequent step fails. The workflow engine invokes compensation actions in reverse order on failure, unwinding the partial execution. This is the saga pattern applied to agent steps. Agents that perform only READ operations require no compensation; agents that perform writes without a declared compensation action must not be placed in a workflow that uses compensation.

## Engine Compatibility

| Engine | Agent Integration Point | Notes |
|---|---|---|
| AWS Step Functions | Lambda task or HTTP task | Native retry and catch; use Express Workflows for high-volume |
| Temporal | Activity | Full compensation via Sagas; strong typing on workflow inputs/outputs |
| Camunda / BPMN | Service task | Suitable for business-process-centric workflows; requires BPMN modelling overhead |
| Apache Airflow | Python operator or HTTP operator | Batch-oriented; not suitable for real-time agent invocation |
