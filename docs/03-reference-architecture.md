# Reference Architecture

The enterprise reference architecture is structured across five planes, one orchestration spine, and two cloud platforms. The planes are vertically stacked: user requests enter at the experience plane, traverse the orchestration spine, invoke models and tools through the middle planes, and are governed at every step by the control plane. Each plane has a defined responsibility boundary and communicates with adjacent planes through typed interfaces.

## System Diagram

```
┌─────────────────────────────────────────────────────────┐
│  EXPERIENCE PLANE                                        │
│  Web UI │ Slack/Teams │ Email │ REST API │ Embedded      │
└────────────────────────┬────────────────────────────────┘
                         │ authenticated request
                         │ (SSO / OAuth2 / JWT)
┌────────────────────────▼────────────────────────────────┐
│  ORCHESTRATION PLANE                                     │
│  ┌─────────┐  ┌────────┐  ┌────────┐  ┌─────────────┐  │
│  │ Planner │→ │ Router │→ │ Worker │→ │Tool Executor│  │
│  └─────────┘  └────────┘  └────────┘  └─────────────┘  │
│                              ↕ HITL Gate                 │
│                         Audit emitter (every step)       │
└────────────┬────────────────────────────────────────────┘
             │ model invocation
             │ (Converse API / Vertex SDK)
┌────────────▼───────────────────────────────────────────┐
│  MODEL PLANE                                            │
│  AWS Bedrock (Claude, Titan, Llama) │ Vertex (Gemini)  │
└────────────┬───────────────────────────────────────────┘
             │ retrieval / tool execution
             │ (typed schemas, risk-class enforcement)
┌────────────▼───────────────────────────────────────────┐
│  DATA & TOOL PLANE                                      │
│  Vector Store │ Graph DB │ Enterprise APIs │ Workflows  │
└────────────┬───────────────────────────────────────────┘
             │ telemetry / policy / cost
┌────────────▼───────────────────────────────────────────┐
│  CONTROL PLANE                                          │
│  IAM │ Secrets │ Audit │ Eval │ Cost Tracking │ Guard  │
└────────────────────────────────────────────────────────┘
```

## Plane Descriptions

### Experience Plane

The experience plane exposes the agent to users and systems through multiple surfaces: a web UI, Slack and Teams bots, email ingestion, a REST API for programmatic access, and an embedded SDK for integration into existing business-unit applications. All surfaces are thin. No surface owns business logic. Every request is authenticated before it reaches the orchestration plane — SSO via the enterprise identity provider, JWT propagated end-to-end so that downstream tool calls can act on behalf of the authenticated user rather than as a generic service account. The experience plane also enforces the first line of input validation: WAF rules and prompt-injection pre-filters that block obviously malicious inputs before they reach the orchestration service.

### Orchestration Plane

The orchestration plane contains the agent loop and the four components of the spine. The **Planner** receives the authenticated request and decomposes it into a plan: a sequence of steps, the tools each step requires, and the model tier appropriate for each step. The **Router** resolves each step to a specific model and retrieval strategy based on a per-agent routing policy defined in YAML configuration. The **Worker** executes each step by invoking the model with the retrieved context and the step's system prompt. The **Tool Executor** handles all outbound calls to external systems, validating tool inputs against typed schemas, enforcing risk-class policies, emitting an audit record for every invocation, and routing requests that require human approval to the HITL queue. The orchestration plane persists state across steps — in Redis, DynamoDB, or Firestore depending on the deployment — so that long-running agent runs survive restarts and can be resumed after a HITL decision.

### Model Plane

The model plane provides foundation model inference behind a single abstract interface. On AWS, this is the Bedrock Converse API, which abstracts across Claude, Titan, Llama, Mistral, and other available model families without model-specific code in the orchestration layer. On GCP, this is the Vertex AI prediction API, which abstracts across Gemini model variants. Both platforms are accessed via VPC endpoints — Bedrock PrivateLink on AWS, Private Service Connect on GCP — so model invocation traffic does not traverse the public internet. The per-agent routing policy determines which model handles which task class; the abstraction ensures that changing that policy is a configuration change, not a code change.

### Data and Tool Plane

The data and tool plane contains the resources that agents read from and act on: vector stores for dense retrieval, a graph database for entity-relationship queries, enterprise APIs for CRM, ticketing, document management, and workflow engines, and workflow automation platforms for multi-step business process actions. Agents never communicate directly with systems of record. Every interaction passes through a thin tool layer that enforces authentication, validates inputs, emits an audit record, and enforces idempotency on write operations. The tool registry — the catalogue of available tools with their schemas and risk classes — is maintained centrally by the platform team. An agent's system prompt exposes only the tools its risk class is permitted to invoke.

### Control Plane

The control plane governs every other plane. IAM provides per-agent service identities with least-privilege scopes; no two agents share an identity. A secrets manager holds credentials for downstream tools and rotates them on a defined schedule, with no secrets stored in agent configuration or environment variables. The audit log receives an immutable record of every model invocation, tool call, HITL decision, and eval run; the log is written to object storage with object-lock retention (S3 with Object Lock on AWS, GCS with Bucket Lock on GCP). The eval harness runs the golden-set evaluation on every deployment event. Cost tracking meters token usage per agent, per model, and per task class, and surfaces the data to the steering committee dashboard. Guardrails — Bedrock Guardrails on AWS, Vertex AI Safety Filters on GCP, plus a custom output validator applied uniformly by the orchestration spine — enforce content and grounding policies at the model boundary.

## Traffic Flow Walkthrough

The following steps trace a single request from user input through all five planes to response.

1. **User submits a request** via the web UI or Slack interface. The experience plane authenticates the request against the enterprise identity provider and attaches a signed JWT containing the user's identity and permission scopes.

2. **The request crosses the first trust boundary** — experience to orchestration. The API gateway validates the JWT, applies WAF rules, runs a prompt-injection pre-filter, and forwards the validated request body to the orchestration service. Requests that fail these checks are rejected with a structured error; they do not reach the agent.

3. **The Planner receives the request** and decomposes it into a step sequence. For a policy Q&A request, this might be: retrieve relevant policy sections, generate a grounded answer, validate faithfulness, format the response.

4. **The Router resolves each step** to a model and retrieval strategy. Classification steps resolve to a smaller, faster model. Generation steps with complex reasoning resolve to a larger model. The routing policy is read from YAML configuration; no routing logic is hardcoded.

5. **The Worker executes each step.** For retrieval steps, it queries the vector store or knowledge base and appends the retrieved context to the model's input. For generation steps, it invokes the model through the model plane interface and receives a structured response.

6. **The model invocation crosses the second trust boundary** — orchestration to model. The request is sent via the VPC endpoint to the model plane. The model plane applies guardrails before returning a response. The orchestration plane applies its own output validator — faithfulness check, schema validation, refusal detection — on the returned response.

7. **The Tool Executor is invoked** when a step requires an external action. It validates the tool input against the typed schema, checks the risk class against the agent's permission policy, and — if the tool is classified IRREVERSIBLE — routes the request to the HITL queue and suspends the agent run pending approval.

8. **The tool call crosses the third trust boundary** — orchestration to tool. The tool executor authenticates to the downstream system using credentials from the secrets manager, propagating the user's identity via on-behalf-of token where the downstream system supports it. The call is executed, the result is validated against the tool's output schema, and an audit record is emitted before the result is returned to the Worker.

9. **The HITL gate fires** when a step produces output that requires human review — either because the tool's risk class mandates it or because the agent's confidence is below the configured threshold. The run is suspended, a notification is sent to the designated reviewer, and the run resumes when the reviewer approves or rejects the action.

10. **The response is returned** through the orchestration plane to the experience plane. The audit log contains a complete, immutable record of every step: model invocations with input and output, tool calls with arguments and results, HITL decisions with approver identity and timestamp, eval scores.

## Trust Boundaries

Three trust boundaries exist in the architecture, each with distinct controls.

**Experience to Orchestration.** The request enters the system here. Controls at this boundary are authentication (JWT validation), authorisation (permission scope check), input sanitisation (prompt-injection pre-filter), and rate limiting per user and per agent. A request that passes these controls is trusted within the orchestration plane to the extent that the JWT permits.

**Orchestration to Model.** The agent's reasoning crosses this boundary on every model invocation. Controls at this boundary are the model plane's guardrails (content filtering, topic restriction, grounding checks) and the orchestration plane's output validator (faithfulness, schema, refusal detection). Neither set of controls is sufficient alone; both are applied. The model's output is never executed directly — it is a structured value returned to the orchestration plane for further processing.

**Orchestration to Tool.** Every external action crosses this boundary. Controls at this boundary are typed schema validation (inputs and outputs), risk-class enforcement (Cedar or OPA policy evaluated outside the model), idempotency enforcement on WRITE operations, on-behalf-of identity propagation, and mandatory audit emission before and after execution. A tool call that fails schema validation or violates a policy rule is rejected before it reaches the downstream system.

## Platform Mapping

| Plane | AWS Services | GCP Services |
|---|---|---|
| Experience | API Gateway, Cognito (user pools), WAF, SES (email) | API Gateway, Identity Platform, Cloud Armor, SendGrid |
| Orchestration | AgentCore Runtime, DynamoDB (state), Lambda / ECS | Cloud Run, Firestore (state), Cloud Functions |
| Model | Bedrock (Converse API), PrivateLink endpoint | Vertex AI (prediction API), Private Service Connect |
| Data & Tool | OpenSearch / Aurora (RAG), Neptune (graph), Secrets Manager | Vertex AI Search, Cloud Spanner, Secret Manager |
| Control | IAM, CloudWatch + OpenTelemetry, S3 + Object Lock, Bedrock Guardrails, AgentCore Observability / Policy | IAM, Cloud Monitoring + OpenTelemetry, GCS + Bucket Lock, Vertex Safety Filters, Model Armor |
