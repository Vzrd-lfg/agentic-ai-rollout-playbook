# Design Principles

Twelve operating positions that shape every architectural decision in this playbook. They are not abstract values — each one was extracted from a category of production failure or a pattern that consistently worked. Violating them is not always wrong, but it should always be deliberate, documented, and accompanied by a rationale that survives a post-incident review.

---

### 1. Deterministic Where Possible, Agentic Only Where Necessary

LLM invocation is expensive, slow, and non-deterministic. The same input can produce different outputs across calls, and the difference matters when the output drives an action in a production system. Rule-based logic, regex, SQL, and workflow engines are deterministic, auditable, and cheap. They are the right answer for any task that can be expressed as rules. Model calls are reserved for tasks that involve genuine ambiguity, natural language understanding, or multi-step planning that resists deterministic encoding. Treating every automation problem as an agent problem is the most expensive form of architectural laziness — it inflates cost, reduces reliability, and makes the agent harder to evaluate because there is no ground truth to compare against.

> **Violation looks like:** an agent invoking a model to parse a structured JSON payload that a schema validator would handle without error.

---

### 2. One Spine, Many Use Cases

The orchestration spine — planner, router, worker, tool executor — is fixed infrastructure. It is built once, governed centrally, and extended by configuration. New use cases arrive as new tools registered in the tool registry, new system prompts scoped to the worker, and new eval sets added to the evaluation harness. They do not arrive as new architectures. This is what makes the fifth and tenth agent dramatically faster to deliver than the first: the platform team is not rebuilding session management, audit emission, HITL routing, and observability wiring for each new agent. The architecture is the product; individual agents are instances of it.

> **Violation looks like:** a bespoke orchestration architecture built for a single use case that cannot be extended to the next one without starting over.

---

### 3. Typed Tools, Never Free-Form Function Calls

Every tool the agent can invoke has a typed input schema, a typed output schema, and a risk class. The model never constructs raw API calls. Tool invocation is a structured operation: the model selects a tool by name and provides arguments that are validated against the schema before execution. Risk classes — READ, WRITE_INTERNAL, WRITE_EXTERNAL, IRREVERSIBLE — determine what additional controls apply: idempotency keys, HITL approval gates, dual-control requirements. Policy enforcement sits outside the model, via Cedar (AgentCore Policy) or OPA, so a prompt injection that compromises the model's reasoning cannot bypass the enforcement boundary.

> **Violation looks like:** allowing the model to synthesise arbitrary HTTP requests, construct dynamic SQL, or call external APIs in free-form code that is executed directly.

---

### 4. Retrieve Before Reason

Agents operating on domain knowledge — policy documents, product catalogues, case history, regulatory text — retrieve the relevant context before they reason over it. The model is not a knowledge base. Model weights encode patterns from training data; they do not encode the organisation's current policy, the current state of a case, or the specific clause of a contract that governs a decision. Generation that cannot be traced to retrieved context is flagged by a faithfulness validator. This is not because hallucinations are categorically intolerable — it is because ungrounded outputs destroy the trust that adoption depends on, and trust, once lost in a business context, is not recovered quickly.

> **Violation looks like:** a policy Q&A agent that answers questions from model memory with no retrieval step, producing answers that are plausible but not grounded in current policy text.

---

### 5. Human-in-the-Loop by Design

HITL is not a workaround for agent unreliability. It is a first-class architectural component that every agent has, specified before the agent ships. The HITL policy for each agent defines which action classes require human approval, what the timeout is, and what the default action is when a response is not received within the timeout. HITL queues have SLAs and are monitored like any other production queue. Agents handling irreversible actions — sending customer-facing communications, initiating financial transactions, updating records in a system of record — have HITL gates by default. Removing a HITL gate from an irreversible action class is a governance decision, not an engineering decision, and it requires a documented risk acceptance.

> **Violation looks like:** deploying an agent with write access to an irreversible tool class — external email, financial workflow, production database — with no HITL gate and no documented risk acceptance.

---

### 6. Continuous Evaluation, Not Just at Launch

Agents are not software libraries that can be deployed and monitored by uptime checks. Every model update, prompt change, tool change, or retrieval corpus change is a deployment event that requires regression evaluation. Every agent has a golden set of historical cases with ground truth, an LLM-as-judge harness for subjective quality dimensions, and a red-team suite for adversarial inputs. The harness runs in CI. Online metrics — faithfulness, refusal rate, HITL approval rate, edit distance on drafted outputs — are tracked in production. The Approval Gate at launch is the first quality check, not the final one; the eval set grows as production incidents add cases to it.

> **Violation looks like:** treating the initial Approval Gate review as the final quality check, with no ongoing eval and no mechanism to detect that agent performance has drifted after a model update.

---

### 7. Cost as a First-Class Design Dimension

Token cost is an engineering metric. It belongs on the same dashboard as latency and accuracy, reviewed in the same steering committee, and owned by the same team that owns model quality. Every agent has a cost-per-task target established before it is approved for production. Routing decisions are cost-aware: a smaller, faster model handles classification and triage; a larger model is invoked only when the task complexity justifies it. Prompt caching, output schemas, retrieval discipline, and context window management all have measurable cost impact and are designed for, not left to chance. An agent that hits its accuracy target but misses its cost target by 5x is not a production-ready agent.

> **Violation looks like:** choosing a frontier model for a classification task that a smaller, cheaper model handles with equivalent accuracy, with no documented rationale for the more expensive choice.

---

### 8. Models Are Interchangeable, the Spine Is Not

The orchestration spine outlasts any specific model. Agent code is written against an abstraction layer — a single `generate(task, context, policy) -> response` interface — and the model is a swappable implementation behind that interface. Swapping a model is a configuration change followed by an eval harness regression, not a code change. This protects the programme against single-vendor pricing shifts, model deprecations, and capability improvements that make last year's frontier model obsolete. Model-specific behaviour — particular formatting quirks, tool-calling syntax, instruction-following patterns — is isolated to the abstraction layer and does not leak into orchestration logic.

> **Violation looks like:** hardcoding model-specific API response shapes, token limits, or tool-calling conventions into orchestration logic, making a model swap a multi-week refactoring effort.

---

### 9. Build, Buy, Partner per Layer

No single vendor supplies every layer of a production agentic system at the quality the enterprise requires. The model plane, orchestration plane, retrieval plane, and control plane may each come from different providers, and the decision is made per layer against explicit criteria: capability fit, total cost, lock-in risk, compliance posture, and operational burden. On AWS in 2026, the platform team adopts AgentCore primitives rather than building session management, tool registries, and observability from scratch — that is a buy decision at the orchestration layer. The prompts, eval sets, BU-specific tools, and change management programme are built; they are the differentiated layers. Vendor lock-in at a single layer is acceptable when the interfaces are clean and the switching cost is bounded. Lock-in at the spine level is not acceptable.

> **Violation looks like:** accepting a vendor's orchestration layer that cannot be replaced without rebuilding every agent, because the tool schemas, memory format, and session state are all proprietary to that vendor.

---

### 10. Adoption Is the Product

An agent that ships and is not used has zero value regardless of its technical quality. Change management, enablement, and feedback loops are engineering work. They are funded, measured, and reported on with the same rigour as accuracy and latency. The adoption metric is defined before the agent ships — active weekly users, tasks deflected, time saved per user, user satisfaction score — and it is tracked from day one of production. Programmes that treat adoption as a soft concern downstream of technical delivery consistently find that their agents are technically sound and behaviourally unused.

> **Violation looks like:** shipping an agent without a defined adoption metric, no enablement material for the users who will use it, and no feedback channel for users to report problems.

---

### 11. Start Narrow, Design for Collaboration Not Replacement

The first version of any agent handles the bounded, high-volume, low-discretion work — the 70% of tasks where the right action is clear and the cost of error is low — and escalates the remaining 30% to a human via the HITL queue. Scope creep in the first version is the most reliable predictor of a Hero Agent failure mode. Framing agents as replacing a human function triggers resistance from the people whose adoption is required for the agent to succeed, and it sets the wrong success metric: the agent's job is to make the human more effective, not to make the human unnecessary. The agents in production that have sustained adoption are the ones that positioned themselves as removing the boring work, not the interesting work.

> **Violation looks like:** scoping the first version of an agent to handle the full breadth of a business process, including the edge cases and high-discretion decisions, before the agent has demonstrated reliability on the core volume.

---

### 12. Centralised Governance, Decentralised Execution

The governance framework, Approval Gate, AI System Register, eval standards, and orchestration spine are centralised. The platform team owns them, maintains them, and enforces them. Building agents, operating agents, iterating on prompts and tools, and driving adoption within a business unit are decentralised to the teams closest to the use case. This split is the mechanism that allows a programme to scale beyond three or four agents without the platform team becoming a bottleneck. Confusing the accountabilities in either direction — a central team that builds all agents because it does not trust BUs, or BUs that self-govern without oversight — produces the two most common failure modes: too slow to deliver value, or an ungoverned rollout.

> **Violation looks like:** a central platform team that reviews and approves every prompt change made by a BU team, treating prompt iteration as an architectural decision rather than an operational one.

---

## The Composition Problem

Narrow scope is not a soft preference — it is an arithmetic requirement. If each step in a ten-step agent pipeline succeeds with 95% probability, the end-to-end accuracy of the pipeline is 0.95^10, which is approximately 60%. This is where the widely observed 60–65% failure rate on complex multi-step agent tasks comes from. The fix is not a more capable model; a more capable model raises each step's success probability from 95% to, say, 97%, which raises the ten-step pipeline accuracy to 0.97^10, approximately 74% — better, but still far below what a production system requires. The real fixes are narrower scope (fewer steps per agent, with handoffs between agents at defined boundaries) and HITL gates on the steps where the cost of failure is highest (breaking the multiplicative chain). These are not optional optimisations; they are the architectural response to the mathematics of sequential uncertainty.
