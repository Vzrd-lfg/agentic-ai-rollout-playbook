# Observability

Agent observability is traces-first. A trace captures the full execution tree of a single request — every model call, every tool invocation, every HITL gate — with timing, cost, and outcome at each node. Metrics and logs are derived from traces, not collected independently. A programme that collects logs and metrics without traces has data without attribution: it can tell you that error rate is elevated but not which model call, which tool, or which retrieval step caused it.

## OpenTelemetry-First Architecture

The orchestration spine emits OpenTelemetry spans for every component — Planner, Router, Worker, Tool Executor, HITL Gate, and the online eval sampler. Each span carries attributes specific to agent operations: model identifiers, token counts, tool risk classes, and policy decisions. Spans are exported to any OTel-compatible backend — AWS CloudWatch, Grafana Tempo, Honeycomb, Datadog, or Google Cloud Trace — through a standard OTel Collector. No vendor-specific SDK is required in agent code. The orchestration layer handles instrumentation; the agent implementation remains portable. Backend selection is an operational decision, not an architectural constraint.

## Per-Span Attributes

| Component | Key Attributes |
|---|---|
| Planner | goal_hash, plan_step_count, model_id, token_count |
| Router | route_decision, confidence_score, selected_agent |
| Worker | step_type, tool_calls_made, output_valid, retry_count |
| Tool Executor | tool_id, risk_class, policy_decision, latency_ms, outcome |
| HITL Gate | gate_id, action_type, approver_id, decision, wait_time_ms |
| Online Eval | trace_id, faithfulness_score, accuracy_score, sampled |

Every span carries the shared context attributes that make cross-span queries possible: `agent_id`, `trace_id`, `span_id`, `user_id` (hashed), and `environment`. These are set once at the root span and propagated automatically through the OTel context. A query of the form "show me every tool invocation in the last 24 hours for agent X where outcome was failure" is answerable without joins because every Tool Executor span already carries `agent_id` and `outcome` as first-class attributes.

## Key Dashboards

Four dashboards cover the operational signals that matter. Each is described in terms of what it shows and when it alerts; specific panel configuration is backend-dependent.

**Latency.** Displays p50, p95, and p99 end-to-end latency for each agent, broken down by component so that a latency increase can be attributed to a model call, a tool invocation, or HITL wait time. The p99 view is the most operationally significant: it captures the tail of user experience that average latency conceals. Alert threshold is agent-specific and registered in the AI System Register at the Approval Gate; the dashboard reads the threshold dynamically so that the alert boundary moves with the contract rather than requiring manual reconfiguration when the threshold changes.

**Tool Error Rate.** Displays the percentage of tool invocations resulting in failure, segmented by tool identifier and by risk class. READ-class tool failures indicate retrieval or integration issues; WRITE-class tool failures are higher-severity because they represent a failed action with potential side effects. Alert threshold: error rate exceeding 2% on any WRITE-class tool triggers an immediate page. READ-class threshold is agent-specific and typically set higher, as retrieval fallbacks are more common and recoverable.

**Faithfulness Trend.** Displays the rolling seven-day faithfulness score from the online eval sampler, per agent. Faithfulness is computed by the LLM judge as described in `eval-suite/judge-rubric.md` and aggregated daily. The trend view is more informative than the point-in-time value: a score that has been stable at 3.9 for four weeks and drops to 3.7 in a single day is a different signal than a score that has been declining by 0.05 per week for a month. Alert threshold: a drop of more than five percentage points relative to the deployment-day baseline fires an alert for investigation.

**Cost per Request.** Displays average and p95 token cost per agent invocation, trended over time. Cost is computed from per-span token counts and current model pricing, updated daily. The p95 view surfaces outlier requests that are disproportionately expensive — typically caused by context window exhaustion, retrieval returning an unusually large payload, or a multi-step plan that expanded unexpectedly. Alert threshold: when average cost exceeds the Approval Gate budget by more than 20%, an alert fires and the token efficiency levers documented in `evaluation/token-efficiency.md` are the first remediation reference.

## Alert Design

Alerts fire to the on-call engineer responsible for the agent, not to a shared inbox. The routing is agent-specific: the AI System Register records the on-call owner for each agent, and the alerting system reads that mapping at fire time. Each alert payload includes: the `agent_id`, the metric name, the registered threshold, the current value, the direction of breach, and a deep link to the relevant dashboard pre-filtered to the agent and time window. The person paged is the one who knows the agent — not a general platform on-call who must first determine which agent is implicated before beginning diagnosis.

## Log Correlation

Every log line emitted by the orchestration spine includes the `trace_id` and `span_id` from the active OpenTelemetry context. This is injected automatically by the logging middleware; individual components do not manage context propagation. The result is that a high-level trace in the OTel backend — showing a failed request with a specific `trace_id` — links directly to every log line generated during that request without a complex join query. An engineer investigating a production failure navigates from the trace view to the failing span to the associated log lines in a single click, regardless of which backend the organisation uses.
