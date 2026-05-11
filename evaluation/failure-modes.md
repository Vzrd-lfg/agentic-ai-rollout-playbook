# Failure Modes

Production data across multiple enterprise agentic deployments shows that agents fail on 60–65% of complex multi-step tasks measured end-to-end against ground truth. This is not a model quality problem — it is an architectural problem. The same agents succeed at 85–95% accuracy on their decomposed sub-tasks in isolation. The gap between sub-task performance and end-to-end performance is the central challenge of production agentic systems. The five failure categories below account for the majority of that gap.

## The 63% Argument

The mathematics of sequential error composition explain the production failure rate precisely. If each step in a ten-step pipeline succeeds with 95% probability, end-to-end accuracy is 0.95^10 ≈ 60%. At 90% per step, it falls to 0.90^10 ≈ 35%. The intuitive response is to push per-step accuracy higher — but even at 99% per step, ten steps yields 0.99^10 ≈ 90%, and most real-world agent tasks involve more than ten steps. The correct architectural response is fewer steps, not higher per-step accuracy. Narrowing the scope of each agent reduces the number of steps in the critical path. HITL gates inserted at the highest-risk hand-offs break the error propagation chain at the points where human judgment is most valuable. These two interventions — narrow scope and strategic HITL placement — are the primary tools for addressing the composition problem.

## Five Failure Categories

### Scope Failure

**Definition:** The task assigned to the agent is broader than the agent's design, tools, or context can reliably handle.

**Production signal:** High replanning rate in the Planner span; user dissatisfaction scores below threshold even when individual step spans show success; escalating HITL volume that the agent's design did not anticipate.

**Leading indicator:** Mean plan step count exceeding 8 across production requests. A plan that consistently requires more than 8 steps to execute is a signal that the task boundary is too wide for the agent's current architecture.

**Remediation:** Decompose the use case further at the design stage. Assign each bounded sub-task to a dedicated Worker with a specific input contract, a specific output schema, and a specific set of tools. The orchestrating Planner coordinates narrower Workers rather than expanding scope within a single Worker.

---

### Composition Failure

**Definition:** Individual steps succeed in isolation but the multi-step chain fails end-to-end due to error propagation, context loss between steps, or incompatible output formats at step boundaries.

**Production signal:** High end-to-end failure rate measured in agent eval despite low per-step failure rate measured in unit eval. The trace shows each span completing with a success outcome; the aggregate output does not match ground truth.

**Leading indicator:** Output schema validation failures at step boundaries — the schema validation middleware rejecting a Worker's output before it is passed to the next step. These validation failures are lower-severity than end-to-end failures but are the earliest signal of composition brittleness.

**Remediation:** Tighten output schemas at every step boundary using JSON Schema or equivalent. Add explicit context-forwarding logic that carries the relevant fields from one step's output to the next step's input contract. Insert HITL gates at the hand-offs with the highest error-amplification risk — specifically, hand-offs where an error in step N renders the output of steps N+1 through N+k incorrect regardless of their individual quality.

---

### Observability Failure

**Definition:** The agent fails and the failure cannot be attributed to a specific component without manual reproduction.

**Production signal:** Post-incident root-cause analyses that take days rather than hours; post-mortems with the finding "we could not reliably reproduce"; recurring incidents with similar surface presentations but different root causes.

**Leading indicator:** Spans missing from traces in the OTel backend — gaps in the execution tree that indicate a component is not emitting instrumentation. Also: `trace_id` absent from log lines, meaning log correlation is not possible.

**Remediation:** Observability is a pre-production requirement, not a post-incident retrofit. Every component must emit a span before the agent is deployed to production. The pre-deployment checklist includes a trace completeness verification step: invoke the agent against a test case and confirm that every expected span is present in the backend, that every span carries the required attributes defined in `evaluation/observability.md`, and that every log line includes a `trace_id`.

---

### Reliability Failure

**Definition:** The same input produces different outputs across runs on tasks where consistency is a correctness requirement.

**Production signal:** User-reported inconsistency on classification, extraction, or routing tasks — tasks where a deterministic output is expected. High output variance in the unit eval suite across repeated runs with identical inputs.

**Leading indicator:** Standard deviation of output scores across unit eval repeated runs exceeding the threshold configured in the eval harness. A classification agent that produces the same label on 95 of 100 identical runs and a different label on 5 is exhibiting a reliability failure at a measurable rate before production.

**Remediation:** Set model temperature to 0 for classification, extraction, and routing Workers. Apply structured output constraints — JSON Schema with enumerated field values — that eliminate the space of possible outputs the model can generate. Add output schema validation as a post-inference step that rejects outputs that do not conform and triggers a structured retry with a fixed fallback before escalating to HITL.

---

### Drift Failure

**Definition:** The agent's quality degrades over time after a deployment that was initially successful, without an explicit change to the agent's configuration.

**Production signal:** Faithfulness score on the online eval dashboard declining monotonically over weeks; user satisfaction scores declining at a rate inconsistent with usage growth; HITL approval rate decreasing without an increase in task complexity.

**Leading indicator:** The weekly faithfulness trend alert firing for two or more consecutive weeks. A single week's dip may reflect sampling variance; two consecutive weeks indicates a directional trend.

**Remediation:** Re-run the full golden-set eval to identify which test cases are now failing. Drift has three common sources: model update (the model provider has changed the model's behaviour without a version bump), corpus drift (the retrieval index content has changed and the system prompt no longer aligns with the available documents), and prompt drift (the system prompt has not been updated to reflect domain changes that have occurred since launch). Each source requires a different fix. Identify the source from the failing test cases before changing anything.

---

## Failure Mode Radar

```
                    Scope
                    ████
                   ██████
         Drift   ████████   Composition
         ████   ██████████   ████
         ████   ██████████   ████
         ████   ██████████   ████
         Reliability   Observability
                ████████
                  ████
```

In practice, most programmes present with Scope and Composition as the dominant failure modes in months 1–3, transitioning to Drift and Reliability as the primary concerns in months 6–12. Observability Failure is a pre-condition that amplifies all others: when observability is absent, every other failure mode takes longer to diagnose and generates more production impact before it is contained. Establishing full trace coverage before launch is the single intervention that most improves mean time to recovery across all five categories.
