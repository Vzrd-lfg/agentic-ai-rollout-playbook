# Auditability

Every agent action that matters to a regulator, an auditor, or an incident investigator must be recorded in an immutable, structured, queryable log. The audit log is not an afterthought — it is a first-class output of the orchestration spine. It is the mechanism by which the organisation can answer, under pressure and on demand: what did this agent do, why, at whose instruction, and with what outcome. An agent system without a complete audit trail is not a production system; it is a prototype operating under production load.

## What to Log

The minimum set of event types and their required fields is defined below. Additional fields may be added per agent, but none of the required fields may be omitted. The `trace_id` field follows the W3C TraceContext specification and links all events within a single agent invocation into a reconstructable sequence.

| Event Type | Required Fields |
|---|---|
| Agent invocation | timestamp, trace_id, user_identity, agent_id, input_hash, session_id |
| Model call | timestamp, span_id, model_id, token_count, latency_ms, guardrail_triggered |
| Tool invocation | timestamp, span_id, tool_id, risk_class, policy_decision, outcome |
| HITL gate | timestamp, span_id, gate_id, action_requested, approver_id, decision, latency_ms |
| Agent response | timestamp, trace_id, output_hash, faithfulness_score (if online eval), latency_ms |
| Policy denial | timestamp, span_id, tool_id, policy_rule_id, user_identity, agent_identity |
| Error | timestamp, span_id, error_code, error_message, retry_count |

The `input_hash` and `output_hash` fields are SHA-256 hashes of the raw input and output respectively. They allow verification of what was processed without retaining the full content in the primary log path — full content is retained separately where required by the agent's risk class. The `faithfulness_score` on agent response events is populated only when online evaluation is active for that agent, which is determined by the agent's configuration in the AI System Register.

## Structured Log Schema

All events share a common envelope. Event-type-specific fields are carried in the `payload` object. The schema version allows the log pipeline to handle schema evolution without breaking existing consumers.

```json
{
  "schema_version": "1.0",
  "event_type": "string",
  "timestamp": "ISO-8601",
  "trace_id": "string (W3C TraceContext)",
  "span_id": "string",
  "environment": "string",
  "agent_id": "string",
  "agent_version": "string",
  "user_identity": "string (hashed if PII)",
  "session_id": "string",
  "payload": { }
}
```

The `user_identity` field is hashed when the raw value constitutes personal data. The mapping from hash to identity is maintained in a separately access-controlled identity store, queryable only by authorised investigators. This satisfies data minimisation while preserving the ability to reconstruct the full picture during a legitimate investigation.

## Immutability

Audit logs are written to append-only storage — S3 Object Lock in compliance mode, GCS with a retention policy, or an equivalent WORM-semantics store. No process, including the agent itself and including administrative accounts in the application tier, has delete or overwrite access to the audit log bucket. Logs are encrypted at rest with a key managed by the control plane. The agent's service identity does not have access to the encryption key used for audit log storage; the separation ensures that a compromised agent cannot destroy the evidence of its own compromise. Log integrity is further protected by a hash chain: each event record contains the hash of the preceding event, so tampering with any record breaks the chain at read time.

## Retention Tiers

Retention is managed in three tiers. Movement between tiers is automated by the log pipeline; no manual intervention is required to meet retention commitments. The hot tier supports real-time investigation; the warm tier supports compliance queries; the cold tier satisfies long-tail regulatory requirements.

| Tier | Retention | Contents | Access |
|---|---|---|---|
| Hot | 30 days | All events | Real-time query (CloudWatch Logs Insights, BigQuery) |
| Warm | 1 year | All events | Batch query (Athena, BigQuery) |
| Cold | 7 years | All events | Archive (S3 Glacier, GCS Archive) |

Retention periods may be extended by regulatory requirement. GDPR does not prescribe a retention period for processing records but requires them to be available to the supervisory authority on request; we retain for seven years as a conservative default. DORA requires ICT incident records for five years. Sector-specific requirements — FINRA, MiFID II, HIPAA — may impose longer periods and these override the defaults where applicable.

## Query Patterns for Incident Investigation

The audit log supports a consistent set of investigative queries. The following three patterns cover the majority of incident and compliance scenarios. All three are expressible in the standard query interfaces of the warm-tier store and return results within the time bounds required for incident response.

**"What did this agent do during session X?"** Filter by `session_id`, order by `timestamp`, and reconstruct the full action sequence from agent invocation through all model calls, tool invocations, HITL gates, and the final response. This query produces the step-by-step narrative required for a regulatory explanation of an automated decision.

**"All WRITE_EXTERNAL tool calls in the last 24 hours."** Filter by `event_type = tool_invocation` and `risk_class = WRITE_EXTERNAL`, group by `agent_id`. This query supports anomaly detection — a spike in external write calls from an agent not associated with a known deployment or scheduled task is a signal that warrants investigation.

**"Which policy denials occurred for this user this month?"** Filter by `event_type = policy_denial` and `user_identity`, count by `policy_rule_id`. This query supports both security investigation — a user probing for policy gaps — and access control tuning — a legitimate user repeatedly denied access they need to do their job.
