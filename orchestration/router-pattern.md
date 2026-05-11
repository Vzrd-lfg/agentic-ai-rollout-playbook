# The Router Pattern

## Problem

Agents receiving heterogeneous requests cannot handle all types well when they share a single system prompt. A prompt that attempts to cover billing inquiries, technical support, policy questions, and triage classification simultaneously degrades in quality on all of them — the model must maintain competing personas, competing retrieval strategies, and competing output formats within a single context. The router pattern is the single front door that prevents this. Rather than building one agent that tries to be everything, we build one Router that knows where to send each request, and multiple specialist agents that each do one thing well.

## Solution

The Router receives a normalised request from the orchestration spine and selects the appropriate specialist agent or model based on routing criteria. The routing criteria are explicit, enumerable, and testable — they are not emergent from the Router model's reasoning about what the request might mean. This is the most important constraint in the pattern: a routing decision that cannot be unit-tested is a routing decision that cannot be trusted in production.

## Routing Decision Table

The following table illustrates an example routing configuration. Actual routing tables are defined in the agent's configuration manifest and are version-controlled alongside the agent.

| Condition | Selected Agent/Model | Rationale |
|---|---|---|
| Request type = "document_extraction" | Document Intake Worker (Bedrock/Claude) | Structured extraction task |
| Request type = "policy_question" | Policy QA Agent (Vertex/Gemini) | RAG-based knowledge retrieval |
| Request type = "ticket_triage" | Triage Agent (Bedrock/Haiku) | Classification, low cost |
| Confidence < threshold | Escalation to human | Ambiguity too high to route safely |
| Unknown type | Default Worker | Graceful fallback |

## Routing Logic

Routing logic is implemented outside the primary model. The Router may use a lightweight classifier model to predict the request type from the normalised request payload, or it may apply a rules engine that matches on structured metadata attached to the request at the Experience Layer boundary. What it does not do is ask the primary task model to select its own routing destination — that conflates the routing decision with the task execution and makes routing behaviour untestable. A routing decision produced by the same model invocation that performs the task cannot be independently evaluated, cannot be A/B tested without also changing the task, and cannot be monitored in isolation. Keeping routing logic in a separate, deterministic layer preserves each concern's testability.

## Fallback Handling

Three fallback paths are defined for every Router deployment and configured explicitly in the routing manifest. First, when a classifier returns a confidence score below the configured threshold, the request is not routed to the lowest-confidence candidate — it is sent to the HITL gate with the classifier's top candidates and scores attached, and a human operator makes the routing decision. Second, when the selected Worker is unavailable — due to a circuit breaker open state or a deployment in progress — the request is placed in a queue with a configured time-to-live, and the user receives an acknowledgment that the request is pending. Third, when all Workers are unavailable simultaneously, the Router returns a graceful degradation response directly to the user, acknowledging the request and providing an estimated recovery window, rather than silently failing or returning an error code.
