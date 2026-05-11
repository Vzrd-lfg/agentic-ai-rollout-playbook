# Multi-Agent Orchestration

When a single agent cannot cover the breadth of a programme's use cases, multiple specialist agents are composed. This document describes the three composition topologies and the trust, failure, and coordination patterns that govern them. Topology selection is not a stylistic choice — each topology has a distinct failure mode and governance surface, and the wrong choice for a given use case will make the system harder to operate, not easier.

## Topology 1 — Hub and Spoke

```
             ┌──────────────┐
             │  Orchestrator │
             │   (Router)   │
             └──────┬───────┘
        ┌───────────┼──────────┐
        │           │          │
   ┌────▼───┐  ┌────▼───┐  ┌──▼─────┐
   │Agent A │  │Agent B │  │Agent C │
   │Doc QA  │  │Triage  │  │Intake  │
   └────────┘  └────────┘  └────────┘
```

A central orchestrator receives all requests and routes each to the appropriate specialist agent. Specialist agents do not communicate with each other; all coordination passes through the orchestrator. This topology is suitable when use cases are genuinely independent — the Doc QA agent's output is never an input to the Triage agent, and neither has any reason to know the other exists. The hub-and-spoke model is the easiest to govern: the orchestrator is the single point of policy enforcement, and specialist agent failures are naturally isolated. It breaks down when agents need to pass context between them, at which point the topology forces that context through the orchestrator, adding latency and complexity that the pipeline topology handles more naturally.

## Topology 2 — Pipeline

```
  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
  │ Extract │───►│Classify │───►│ Enrich  │───►│ Route   │
  │ Agent   │    │ Agent   │    │ Agent   │    │ Agent   │
  └─────────┘    └─────────┘    └─────────┘    └─────────┘
```

Agents are chained in a fixed sequence; each agent's typed output is the next agent's typed input. The orchestrator controls sequencing deterministically — the pipeline order is defined in configuration, not inferred by any model at runtime. This topology is the natural fit for document processing workflows, where a document must be extracted, then classified, then enriched with metadata, then routed to the appropriate downstream handler. Each stage is independently testable and replaceable without risk to the others. The critical design constraint is failure handling: a failure in any stage halts the pipeline. Each stage must be designed with an explicit compensation step — what the system does when that stage fails — and compensation logic must be defined before the pipeline goes to production, not added reactively.

## Topology 3 — Mesh

```
  ┌─────────┐◄──────────────►┌─────────┐
  │ Agent A │                │ Agent B │
  └────┬────┘                └────┬────┘
       │                          │
       └──────────┬───────────────┘
                  ▼
            ┌─────────┐
            │ Agent C │
            └─────────┘
```

Agents communicate peer-to-peer to collaborate on a task. No single agent owns the control flow; coordination emerges from the inter-agent communication pattern. The mesh is the most powerful topology for tasks that require genuine collaboration across specialisations — a legal review agent, a compliance agent, and a risk assessment agent that must each independently evaluate a contract and then reconcile their findings. It is also the hardest to govern: trust between agents must be explicit and implemented at the invocation level, failure can propagate non-obviously across agent pairs, and the absence of a central control flow makes debugging significantly harder. Reserve this topology for use cases where no other topology provides an adequate fit.

## Trust Between Agents

Each agent has an IAM role or equivalent service identity that is distinct from every other agent's identity and from the end user's identity. When Agent A calls Agent B, it presents a scoped token that identifies Agent A as the caller — not the end user. Agent B's policy engine evaluates the calling agent's identity against its configured tool policies, applying the same Cedar or OPA evaluation it would apply to any other caller. On-behalf-of token chaining propagates the end-user context — the original user's identity and role claims — as a subordinate claim on the inter-agent call, so that Agent B can apply user-level policy without being granted the user's full permissions. An agent cannot grant itself or another agent permissions that exceed its own authorised scope.

## Failure Propagation

| Topology | Failure Behaviour | Mitigation |
|---|---|---|
| Hub and Spoke | Single agent failure is isolated to that request | Orchestrator retries or escalates to HITL |
| Pipeline | Upstream failure halts all downstream stages | Compensation steps defined per stage |
| Mesh | Failure can cascade across peer agent pairs | Circuit breakers between agent pairs |

## Coordination Patterns

**Handoff.** Agent A completes its task and passes the typed result to Agent B. Agent A has no further involvement once the handoff is made. This is the simplest coordination pattern: it requires no shared state, no waiting, and no synchronisation. The Planner orchestrates the handoff by dispatching steps sequentially according to the dependency graph.

**Delegation.** Agent A delegates a subtask to Agent B and suspends its own execution until Agent B returns a typed result. Agent A must persist its pending state to the checkpoint store for the duration of the delegation; the state is restored when Agent B's result arrives. Delegation is appropriate when Agent A needs Agent B's output to determine its own next action — when the subtask is genuinely blocking, not merely a prerequisite that could be sequenced by the Planner.

**Consensus.** Multiple agents independently produce outputs on the same input; an aggregator agent receives all outputs and selects or merges them into a final result. This pattern is appropriate for high-stakes decisions where a single agent's output carries unacceptable risk — regulatory analysis, medical triage, financial risk assessment — and where the cost of running multiple agents is justified by the reduction in wrong-answer risk. It is not appropriate as a general quality improvement strategy; for most tasks, a single well-scoped Worker with deterministic output validation is less expensive and no less reliable.
