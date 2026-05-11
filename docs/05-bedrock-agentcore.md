# AWS Bedrock AgentCore

AgentCore is AWS's managed runtime for enterprise-grade agentic applications. It provides nine independently composable primitives that together constitute a production agent lifecycle — from secure invocation and session management through tool governance, identity propagation, observability, and continuous evaluation. This document describes each primitive and the sequence in which to adopt them. The argument for AgentCore as the default Bedrock spine is straightforward: building equivalent infrastructure from scratch is a six-to-twelve-week investment in undifferentiated engineering. AgentCore collapses that investment to configuration, returning the platform team's capacity to the layers that genuinely differentiate the enterprise's agents — the prompts, eval sets, BU-specific tools, and change management programme.

## The Nine Primitives

### 1. Memory

AgentCore Memory provides persistent agent memory across sessions at two scopes. Short-term memory manages the within-session context window: it handles summarisation of long conversations, selective retention of relevant earlier turns, and context compression to keep token costs bounded across multi-turn interactions. Long-term memory provides cross-session knowledge storage in a managed vector store: facts, preferences, and decisions from prior sessions are retrieved at the start of new sessions and injected into the model's context. Both scopes are managed — the platform team does not operate conversation tables, embedding pipelines, or summarisation jobs. Memory is the last primitive to adopt because most agents do not require it initially; session-level context is sufficient until it demonstrably is not.

### 2. Sessions

AgentCore Sessions provides managed session state with configurable TTL, session resumption after interruption, and multi-turn coherence across the full lifetime of an agent interaction. Session state is persisted in managed storage and is isolated per session — one agent run cannot read another's state. The practical benefit is that the orchestration service does not need to implement session lifecycle: creation, persistence, expiry, and resumption are handled by the primitive, and the agent code receives a session context object at invocation time. This decouples session management from application code, which reduces the surface area for bugs and simplifies long-running interactions that span HITL approval steps.

### 3. Executor

The Executor is the managed compute layer for agent invocation. It provides built-in retry logic with configurable backoff, timeout enforcement, concurrency controls per agent, and session isolation that prevents one agent run from affecting another on shared infrastructure. Cold start latency is managed by the runtime; the platform team configures the concurrency and timeout parameters and does not operate the underlying compute. The Executor is the entry point for all agent invocations — the primitive that the experience plane talks to when it forwards an authenticated request to the orchestration plane.

### 4. Tools

AgentCore Tools is the typed tool registry. Every tool the agent can invoke is registered with an input schema (JSON Schema), an output schema, and metadata including the tool's owner, its risk class, and whether invocation requires HITL approval. The registry is the authoritative source of what actions are available to each agent and what the constraints on those actions are. Tool registration is a platform-team function; BU teams propose new tools but do not self-register them into the registry. The Executor enforces invocation policies derived from the registry — a tool registered as IRREVERSIBLE cannot be invoked without a HITL gate, regardless of what the agent's system prompt instructs.

### 5. Guardrails

AgentCore Guardrails applies content filtering, topic restriction, and grounding checks at the model boundary, independently of the agent's system prompt. This independence is the key property: guardrails are configured by the platform team and cannot be overridden by prompt injection into the user's input or the system prompt. On AWS, Bedrock Guardrails provides PII detection and redaction, denied-topic enforcement, and jailbreak resistance. The orchestration spine additionally applies its own output validator — faithfulness checks, schema validation, refusal detection — on every model response, providing a second layer of protection that is uniform across both Bedrock and Vertex model invocations.

### 6. Observability

AgentCore Observability emits OpenTelemetry-compatible traces for every agent step. Each span carries attribution data: which model was invoked, which tool was called, latency at each step, token count and cost, and the outcome of each HITL decision. Traces are collected in CloudWatch and can be forwarded to any OTel-compatible collector — a SIEM, a Datadog or Honeycomb backend, or a custom observability stack. Per-span cost attribution is the feature that makes the cost-as-first-class-design-dimension principle operational: the platform team can see exactly which agent, which step, and which model is driving cost, and route accordingly.

### 7. Model Access

AgentCore provides unified model invocation via the Bedrock Converse API. The Converse API presents a single interface across Claude, Titan, Llama, Mistral, Cohere, and other models available on Bedrock, normalising tool-calling syntax, message structure, and response formats so that no model-specific code appears in agent logic. Swapping the model behind an agent is a configuration change to the per-agent routing policy; the orchestration plane code does not change. This is the implementation of the "models are interchangeable, the spine is not" design principle at the platform level.

### 8. Identity

AgentCore Identity provides agent service identities via IAM roles, with no shared service accounts across agents. On-behalf-of token propagation allows agents to make downstream tool calls carrying the authenticated user's identity — so that CRM APIs, ticketing systems, and document stores can apply per-user access controls on agent-initiated requests, not just per-agent controls. A token vault managed by AgentCore stores credentials for downstream systems and handles rotation; agent code never handles raw credentials. This is the control that satisfies most enterprise security reviews on the first pass, because it answers the question "what identity does the agent act as, and can we scope it?" with a clear and auditable answer.

### 9. Policies

AgentCore Policies integrates with Cedar — AWS's open-source policy language — and AWS Verified Permissions to enforce policy-as-code on tool invocations. Cedar policies are evaluated outside the model: the policy engine receives the action the agent is attempting to take, the attributes of the requesting agent, and the attributes of the target resource, and it returns an allow or deny decision before the tool executor makes the call. This means that a prompt injection that succeeds in manipulating the model's reasoning cannot bypass the policy boundary — the enforcement is independent of the model's output. Policies are authored and owned by the platform team; they are not part of the agent's system prompt and are not visible to the model.

## Adoption Sequence

The following diagram shows the recommended order in which to adopt AgentCore primitives, with the rationale for each phase.

```
Phase 1 — Foundation
  [Model Access] --> [Sessions] --> [Executor]

  Get basic agent invocation working with persistent session state.
  This is the minimum viable spine: a model you can call, sessions
  that persist across turns, and managed compute that handles retries
  and timeouts. Instrument from day one — do not defer Observability.

  [Observability] added here, not in Phase 4, for all new programmes.

Phase 2 — Safety
  [Guardrails] --> [Policies]

  Add content filtering and Cedar policy enforcement before any
  external tool calls are wired in. Safety controls are harder to
  retrofit than to build in; adding them here means every tool call
  that comes later is already covered.

Phase 3 — Tools and Identity
  [Tools] --> [Identity]

  Register tools with typed schemas and risk classes. Wire up
  on-behalf-of identity so that tool calls carry the authenticated
  user's identity to downstream systems. This is the phase where
  the agent begins taking actions in external systems; the controls
  from Phase 2 must be in place first.

Phase 4 — Memory
  [Memory]

  Add long-term memory only when session-level context is
  demonstrably insufficient — when users are reporting that the
  agent does not remember relevant facts from prior sessions, or
  when the use case explicitly requires cross-session continuity.
  Memory adds operational complexity and cost; most agents do not
  need it at initial rollout.
```

## What AgentCore Does Not Provide

AgentCore does not replace a CI/CD pipeline for agent versioning, a golden-set eval harness, or a change-management programme. The primitives handle the runtime infrastructure: invocation, session state, tool governance, identity, observability, and policy enforcement. What they do not handle is the programme-level work: authoring and maintaining the golden eval set, running red-team exercises, operating the Approval Gate, maintaining the AI System Register, designing the HITL queue workflow, and driving adoption within business units. Those are the components this playbook addresses in `evaluation/`, `rollout/`, and `governance/`. AgentCore reduces the undifferentiated infrastructure build; it does not reduce the differentiated programme work.
