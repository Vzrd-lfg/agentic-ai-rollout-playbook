# Orchestration Patterns

The orchestration spine contains four patterns: Planner, Router, Worker, and Tool Executor. They compose rather than compete — a single agent deployment may use all four, and the relationship between them is fixed: the Planner decomposes, the Router dispatches, the Worker executes, and the Tool Executor acts. Understanding each pattern in isolation is necessary, but the architecture only becomes coherent when the composition model is understood.

## Pattern Overview Table

| Pattern | Responsibility | When to use |
|---|---|---|
| Planner | Decompose a goal into a typed, validated sequence of steps | Multi-step tasks where the sequence is not known in advance |
| Router | Select the appropriate specialist agent or model for a given request | When requests vary in type and each type requires a different capability |
| Worker | Execute a single well-scoped step with typed inputs and outputs | Every leaf-node task in a plan |
| Tool Executor | Invoke enterprise tools with risk-class enforcement and audit | Any call that reads from or writes to a system outside the model |

## Composition Model

In a fully composed deployment, the Planner receives the user's goal and decomposes it into an ordered, schema-validated plan. The Router receives each plan step in sequence and dispatches it to the appropriate specialist Worker — the one whose contract matches the step's type. The Worker executes the step, invoking the Tool Executor for any action that touches a system outside the model. Tool results flow back to the Worker, which validates them against its output schema and returns the typed result to the Planner. The Planner advances to the next step, applies dependency logic, and issues the next dispatch. If a step fails and replanning is permitted, the Planner re-invokes the model with the failed step's error as context and produces a revised plan. After the maximum replanning attempts are exhausted, the Planner escalates to the human-in-the-loop gate.

## Scope x Complexity Quadrant

```
         High Complexity (many steps)
                    │
    Multi-Agent     │    Planner +
    Orchestration   │    Multi-Worker
                    │
────────────────────┼────────────────────
  Low Scope         │            High Scope
  (narrow task)     │            (broad task)
                    │
    Single Worker   │    Router +
    (leaf node)     │    Worker Pool
                    │
          Low Complexity (few steps)
```

The quadrant guides the choice of composition depth. A narrow, single-step task — a document classification, a status lookup — warrants a single Worker with no Planner overhead. A broad, multi-step task with conditional branching warrants a Planner coordinating multiple Workers. Multi-agent orchestration, described in `multi-agent-orchestration.md`, applies when the breadth of use cases exceeds what a single agent deployment can handle coherently.

## Pattern Files

- [planner-pattern.md](./planner-pattern.md) — Goal decomposition into typed, validated plans with dependency graphs and replanning logic.
- [router-pattern.md](./router-pattern.md) — Request classification and specialist selection with explicit, testable routing criteria.
- [worker-pattern.md](./worker-pattern.md) — Narrowly scoped step execution with typed contracts, bounded tool sets, and output validation.
- [tool-executor-pattern.md](./tool-executor-pattern.md) — Risk-class enforcement, policy evaluation, audit emission, and HITL gating for all tool invocations.
