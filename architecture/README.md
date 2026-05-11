# System Architecture

The playbook's architecture is a five-plane stack with a four-component orchestration spine. Every agent deployed under this playbook is a configuration of that spine — Planner, Router, Worker, and Tool Executor — rather than a bespoke architecture designed from scratch. This constraint is intentional: it concentrates governance controls in a small number of well-defined seams, makes agents independently testable, and ensures that lessons learned on one agent transfer directly to the next.

## System Context Diagram

```
                    ┌──────────────────────────────────┐
                    │         Enterprise Users          │
                    │  (staff, system accounts, APIs)  │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │         Experience Layer          │
                    │  Web · Slack · Email · REST API  │
                    └──────────────┬───────────────────┘
                                   │  authenticated, normalised
                    ┌──────────────▼───────────────────┐
                    │       Orchestration Spine         │◄── Governance &
                    │  Planner→Router→Worker→Tool Exec  │    Eval Controls
                    └──────┬──────────────┬────────────┘
                           │              │
             ┌─────────────▼──┐    ┌──────▼────────────────┐
             │  Model Plane   │    │   Data & Tool Plane    │
             │ Bedrock/Vertex │    │ Vector·Graph·APIs·Wkfl │
             └────────────────┘    └───────────────────────┘
                           │              │
                    ┌──────▼──────────────▼────────────┐
                    │         Control Plane             │
                    │  IAM · Audit · Cost · Observ.    │
                    └──────────────────────────────────┘
```

The Experience Layer is the only surface exposed to users. It normalises requests to a canonical format before they reach the orchestration spine, ensuring the spine never has to reason about channel-specific quirks. The Control Plane sits beneath every other plane and is accessible only in read from the application zone — audit logs are immutable and cannot be modified by any component in the orchestration spine.

## Component Interaction Diagram

The following diagram traces how a single request flows through the four spine components and the supporting infrastructure.

```
  Request In
      │
      ▼
 ┌──────────┐    typed plan    ┌──────────┐
 │  Planner │─────────────────►│  Router  │
 │          │                  │          │
 │ Decomposes goal             │ Selects  │
 │ into steps │                │ specialist│
 └──────────┘                  └────┬─────┘
                                    │ routes to
                               ┌────▼─────┐    tool request   ┌───────────────┐
                               │  Worker  │──────────────────►│ Tool Executor │
                               │          │                    │               │
                               │ Executes │◄───tool result─────│ Risk class    │
                               │ one step │                    │ enforcement   │
                               └────┬─────┘                    └───────────────┘
                                    │                                  │
                                    │ HITL gate (if required)          │ Audit event
                                    ▼                                  ▼
                               ┌──────────┐                    ┌───────────────┐
                               │ Human    │                    │  Audit Store  │
                               │ Review   │                    │  (immutable)  │
                               └──────────┘                    └───────────────┘
```

The Tool Executor is the only component that reaches outside the orchestration spine. Workers cannot call enterprise systems directly; every external action is mediated by the Tool Executor, which evaluates risk class and policy before invocation and emits an audit event regardless of outcome.

## Data Flow Walkthrough

The following steps trace a representative request from the user's submission to the final response, annotating the acting component, the trust boundary crossed, and what is logged at each transition.

1. **User submits request** via the Experience Layer (web, Slack, REST API). The layer authenticates the user against the enterprise identity provider, attaches a verified user identity and role claim, and normalises the request to a canonical payload. The Experience → Orchestration boundary is crossed. Logged: request receipt, user identity, channel, timestamp, trace ID.

2. **Orchestration spine receives the normalised request.** Rate limiting and quota checks are applied against the user identity before any model call is made. If quota is exceeded, the request is rejected at this point. Logged: pre-flight policy evaluation result.

3. **Planner decomposes the goal.** The Planner invokes a model call with the user's goal and the available tool catalog. The output is a typed, schema-validated plan. No trust boundary is crossed at this step; the Orchestration → Model boundary is the model call itself. Logged: plan schema validation result, plan step count, estimated cost.

4. **Router selects the appropriate specialist Worker** for the first executable step. The routing decision is made against the plan's step type and the available Worker registry — not against free-form model output. Logged: routing decision, selected Worker, confidence score.

5. **Worker executes the assigned step.** The Worker invokes the model with a bounded, single-purpose system prompt and the typed step input. The Orchestration → Model boundary is crossed for this model call. Logged: model call, input token count, output token count, latency.

6. **Worker requests a tool invocation** via the Tool Executor. The Orchestration → Tool boundary is crossed. The Tool Executor checks the risk class of the requested tool, evaluates the Cedar/OPA policy against the Worker identity and user identity, and either permits or denies the call. Logged: tool ID, risk class, policy decision, Worker identity, user identity (on-behalf-of).

7. **HITL gate is evaluated** if the tool's risk class requires human approval. The agent state is persisted to the checkpoint store; an approval request is emitted to the configured review channel. Execution resumes only after an approver acts. Logged: HITL request emission, approver identity, approval or rejection, latency of human decision.

8. **Response is returned to the user** through the Experience Layer. The Worker's typed output is rendered into the channel-appropriate format. The audit event chain is closed. Logged: final response, total token consumption, total cost, end-to-end latency, outcome.

## Trust Boundaries

**Experience to Orchestration.** Every request entering the orchestration spine must carry a verified identity token issued by the enterprise identity provider. The Experience Layer is responsible for authentication, authorisation against the user's role and permitted agent scope, request normalisation into the canonical payload schema, and rate limiting. The orchestration spine trusts the claims on the token but does not perform its own authentication — the boundary is a single, auditable chokepoint.

**Orchestration to Model.** When the Planner or Worker invokes a model, it constructs the system prompt programmatically from versioned templates. User-supplied content is injected into the user turn only and is never permitted to overwrite the system prompt. Context window management — chunking, truncation, retrieval augmentation — is handled by the orchestration layer before the model call. Guardrail checks (prompt injection pattern matching, PII detection) are applied to both input and output at this boundary.

**Orchestration to Tool.** The Tool Executor is the sole component permitted to cross the Orchestration → Tool boundary. Before any invocation, it verifies the calling Worker's identity against the Tool Registry, determines the tool's risk class, evaluates the Cedar or OPA policy with the full request context (Worker identity, user identity, tool ID, action parameters), and emits an audit event. The model cannot influence or override a policy decision — the policy engine runs as a separate process outside the model's context window.
