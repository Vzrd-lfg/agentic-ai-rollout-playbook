# The Planner Pattern

## Problem

Complex tasks cannot be dispatched to a single model invocation without quality degradation. When a model must simultaneously decompose a goal, select tools, sequence steps, apply conditional logic, and produce a coherent output, each of those concerns competes for the model's effective working capacity. The result is plans embedded in prose, tool selections that skip prerequisite steps, and sequences that cannot be mechanically executed without re-interpreting the model's intent at each stage. The only reliable fix is to separate goal decomposition from execution — a single model call whose sole output is a typed, machine-readable plan.

The model's working memory is finite, and long chains of reasoning degrade mid-sequence. An eight-step plan that the model constructs and executes in a single context rapidly loses coherence at steps five through eight, as the model's attention distributes across the growing context. By separating the Planner from the Worker, we ensure the Planner sees only the goal and tool catalog, and each Worker sees only its own step — neither is burdened with the other's context.

## Solution

The Planner is the component that receives a user's goal, decomposes it into a typed, ordered sequence of steps, validates that sequence against a schema, and hands the validated plan to the execution layer. On simple deployments, the execution layer is a single Worker. On complex deployments, the Router dispatches each step to the appropriate specialist Worker. The Planner does not execute steps, call tools, or produce user-facing output — its only product is the plan.

## Plan Schema

```yaml
# Typed plan output schema
plan:
  goal: string            # original user intent
  steps:
    - id: string          # unique step identifier
      type: enum          # [retrieve, transform, validate, write, decide, notify]
      input: object       # typed input matching the step's Worker contract
      depends_on: [id]    # explicit dependency graph
      hitl_required: bool # whether human approval is needed before execution
      timeout_seconds: int
  replanning_allowed: bool
  max_replanning_attempts: int
```

The `type` field constrains the plan to a closed set of step types, each of which maps to a Worker contract. Free-form step descriptions are rejected at validation time. The `depends_on` field makes the execution graph explicit — the orchestrator does not infer ordering from step IDs or narrative sequence; it reads the dependency graph directly. The `hitl_required` flag is set by the Planner but is also enforced independently by the Tool Executor: even if the Planner omits it, the Tool Executor will block on HITL for tools whose risk class requires it.

## Sequence Diagram

```
User        Planner     Schema      Router/Worker   HITL Gate
 │             │        Validator       │               │
 │──goal──────►│                       │               │
 │             │──decompose────────────►               │
 │             │◄──typed plan──────────                │
 │             │──validate──►│                         │
 │             │◄──valid─────│                         │
 │             │─────────────────────step──────────────►│
 │             │             │        │           approve/reject
 │             │◄────────────────────────────────result│
 │◄──response──│             │        │               │
```

## Validation Rules

The following constraints are evaluated deterministically, by the schema validator, before any step is dispatched to the execution layer:

- All step `type` values are members of the permitted enum: `retrieve`, `transform`, `validate`, `write`, `decide`, `notify`. Steps with unrecognised types are rejected without retry.
- All `depends_on` references resolve to valid step IDs within the same plan. A reference to a non-existent step ID is a hard validation failure.
- The dependency graph contains no circular references. Cycles are detected via topological sort before execution begins.
- All steps whose `type` is `write` and whose target tool has a risk class of `WRITE_EXTERNAL` or `IRREVERSIBLE` must have `hitl_required: true`. Plans that omit this flag for those steps are rejected; the Planner is re-invoked with the omission injected into context as an explicit constraint.
- The total step count does not exceed the configured maximum (typically eight). Plans exceeding this limit indicate that the goal was not sufficiently scoped and should be escalated to the user rather than retried.

## Failure Handling

On step failure, the Planner can replan if `replanning_allowed` is true. Replanning re-invokes the model with the original goal, the validated plan, the failed step's identifier, and the error returned by the Worker or Tool Executor injected as structured context. The model produces a revised plan covering the remaining steps. The revised plan is validated by the same schema validator before execution resumes. After `max_replanning_attempts` consecutive failures, the Planner does not retry further — it emits a structured escalation event to the HITL gate, suspends the agent state to the checkpoint store, and waits for a human operator to intervene.
