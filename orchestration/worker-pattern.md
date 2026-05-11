# The Worker Pattern

## Problem

Broad-scope agents fail because the model must simultaneously reason about task decomposition, domain knowledge, tool selection, error handling, and output formatting within a single context. Each of those concerns adds noise to the others. A model asked to decompose a complex task, retrieve relevant documents, select the right tool, and format the result in a single invocation will perform each of those functions worse than a model asked to do only one of them. The Worker pattern is the antidote to the broad-scope agent: it isolates one well-defined task, gives the model a single-purpose system prompt, and produces a typed output that the orchestration layer can mechanically verify.

## Solution

A Worker is a narrowly scoped agent that executes exactly one step type. It has a typed input contract, a typed output contract, and a bounded set of tools it may invoke. The input contract defines precisely what the Worker receives from the Planner or Router; the output contract defines precisely what it returns. The Worker cannot negotiate the scope of its task at runtime, cannot invoke tools outside its allowed set, and cannot return output that does not conform to its schema. These constraints make the Worker independently testable, replaceable without risk to adjacent steps, and observable in isolation.

## Worker Contract

```yaml
worker_contract:
  name: string
  step_type: enum           # matches plan step types
  input_schema: object      # JSON Schema
  output_schema: object     # JSON Schema
  allowed_tools: [string]   # tool IDs from the Tool Registry
  max_tool_calls: int       # prevents runaway tool use
  model: string             # specific model or model family
  timeout_seconds: int
  retry_policy:
    max_attempts: int
    backoff: exponential
```

The `step_type` field binds the Worker to a specific plan step type, ensuring the Router can select the correct Worker for each step without reasoning about the step's content. The `allowed_tools` list is the binding that the Tool Executor enforces at invocation time — it is not advisory. The `max_tool_calls` field prevents a Worker from entering a tool-calling loop when the model is uncertain about the correct action.

## Output Schema Validation

The Worker's output is validated against the `output_schema` before being returned to the Planner or Router. Validation is deterministic — a schema check, not a model-based quality assessment. If the output does not conform, the step is retried with the schema error injected into the model's context as a structured correction prompt. After `max_attempts`, the step is marked failed, the Worker returns a typed failure result, and the Planner handles escalation according to its replanning policy. Output validation failures that recur across retries are flagged as a prompt quality issue in the eval harness.

## Tool Permission Boundary

The Worker can only invoke tools listed in its `allowed_tools`. The Tool Executor enforces this at invocation time by verifying the calling Worker's identity — carried in the invocation context — against the Tool Registry entry for the requested tool. A Worker that requests a tool not in its allowed list receives a structured denial, not an error. The denial is logged as a policy event. A Worker cannot escalate its own tool permissions at runtime, cannot request tools on behalf of another Worker, and cannot modify its own contract.

## Retry Policy

| Condition | Action |
|---|---|
| Model timeout | Retry with exponential backoff |
| Invalid output schema | Retry with schema error injected into context |
| Tool call failure (transient) | Retry tool call |
| Tool call failure (permanent) | Mark step failed; Planner handles replanning |
| Max attempts reached | Escalate to HITL |
