# Evaluation Harness

Evaluation is not a launch activity — it is a continuous operational practice. Every agent deployment generates a contract: a set of pass/fail thresholds, agreed at the Approval Gate, that define whether the agent is fit for production. The eval harness enforces that contract on every deployment event. An agent that passed six months ago but has not been re-evaluated since has no current fitness assurance; the harness treats that as an open risk.

## Five Evaluation Axes

| Axis | What It Measures | Minimum Threshold (example) | Judge |
|---|---|---|---|
| Accuracy | Correctness of factual claims | Set per agent at Approval Gate | Ground-truth comparison |
| Faithfulness | Grounding in retrieved context | ≥95% for RAG agents | LLM-as-judge against source |
| Safety | Absence of harmful, biased, or non-compliant outputs | 0 critical failures | Guardrail log + red-team eval |
| Cost | Token cost per request | Agent-specific budget | Cost tracking + token counter |
| Adoption | User satisfaction and task completion rate | ≥80% task completion after 30 days | Usage analytics + survey |

**Accuracy** measures whether the agent's factual claims are correct relative to ground truth. For classification agents, it is the proportion of outputs that match the annotated label. For generative agents, it is scored by comparing the response to a human-written authoritative answer using an LLM judge or string-match criteria. Thresholds are set per agent at the Approval Gate because different agents carry different error tolerances: a content summarisation agent may tolerate ≥90% accuracy where a compliance classification agent requires ≥99%.

**Faithfulness** is the degree to which every factual claim in a RAG agent's response is traceable to the retrieved source material. A faithful response does not introduce facts from model parametric memory. An LLM judge is passed both the response and the retrieved chunks and asked to score whether each claim is supported. Faithfulness is distinct from accuracy: a response can be faithful to a source that is itself wrong.

**Safety** measures whether the agent produces harmful, discriminatory, or policy-non-compliant outputs. It is the only axis with a zero-tolerance threshold: a single critical safety failure blocks deployment regardless of scores on other axes. Safety evaluation combines automated guardrail logs with a dedicated red-team eval surface that exercises adversarial inputs specifically designed to probe policy boundaries.

**Cost** measures the token cost per request against the budget set at the Approval Gate. Cost is tracked at the span level (per model call) and aggregated per agent run. A passing cost score means the agent is operating within its approved budget. Cost threshold violations do not necessarily block deployment but trigger a mandatory review of the token efficiency levers before the next release.

**Adoption** is the only axis that cannot be measured before production. It is evaluated at 30 days post-deployment using usage analytics (task completion rate) and optional user surveys. An adoption score below threshold does not trigger a rollback but initiates a structured review with the Business Unit AI Owner to identify whether the gap is a quality issue, a workflow integration issue, or a change management issue.

## Four Evaluation Surfaces

### Unit Eval (per-prompt)

Individual prompt-response pairs are evaluated in isolation. Unit evals run as part of the CI pipeline on every change to the system prompt, the tool definitions, or the model version. Each test case exercises a single known-good behaviour: a specific input category, a known edge case, or a previously-failing regression. Unit evals are fast — they do not require a full agent run — and catch prompt regressions before they can compound into agent-level failures. A failing unit eval blocks the merge.

### Agent Eval (golden set)

The full agent is run end-to-end against a curated golden set of 100–500 test cases with ground-truth answers. This is the primary gate for deployment. Each case is a complete input-to-output workflow, not a decomposed sub-step. The eval harness scores each case across all five axes (where applicable), aggregates, and compares against the thresholds from the AI System Register. All axes must pass for deployment to proceed. A partial pass — for example, accuracy above threshold but cost above budget — constitutes a failing run.

### Red-Team Eval (adversarial)

A set of adversarial inputs designed to trigger prompt injection, boundary violations, harmful outputs, and policy bypasses. Red-team evals run before every major release and whenever the agent's threat model changes — for example, when new tool categories are added, when the user population expands, or when a new risk class is introduced. The adversarial input set is maintained separately from the golden set and is reviewed by the AI Risk & Compliance function at each Approval Gate. A red-team failure blocks deployment regardless of golden-set scores.

### Online Eval (production sampling)

One to five percent of live production traffic is sampled and scored by an LLM judge against the faithfulness and accuracy axes. Sampled traces — including the model's response and any retrieved context — are scored asynchronously and fed to a monitoring dashboard. Alerts fire when performance drifts below a configurable threshold relative to the deployment-day baseline. Online eval is the only surface that catches real-world distribution shift: cases that were not anticipated in the golden set, and model behaviour changes introduced silently by the model provider.

## Contract-Based Evaluation

At the Approval Gate, the pass/fail thresholds for all five axes are agreed between the AI Platform team, the Business Unit AI Owner, and the AI Risk & Compliance function. These thresholds are recorded in the AI System Register and constitute the evaluation contract for that agent. They are not negotiable post-deployment without a new Gate review. The eval harness reads the thresholds from the register at runtime and reports pass/fail against the registered values — not against a global standard. Global defaults exist as a floor; registered thresholds override them upward. This design means that a mission-critical agent can be held to a higher standard than the platform minimum without modifying the harness itself.

## Eval Pipeline

```
  Code / Prompt Change
          │
          ▼
    ┌──────────────┐
    │  Unit Eval   │── fail ──► Block merge
    └──────┬───────┘
           │ pass
           ▼
    ┌──────────────┐
    │  Agent Eval  │── fail ──► Block deployment
    │ (golden set) │
    └──────┬───────┘
           │ pass
           ▼
    ┌──────────────┐
    │  Red-Team    │── fail ──► Block deployment
    │    Eval      │
    └──────┬───────┘
           │ pass
           ▼
    Deploy to Production
           │
           ▼
    ┌──────────────┐
    │ Online Eval  │── drift alert ──► Incident review
    │  (1–5%)      │
    └──────────────┘
```

The pipeline is sequential by design. Unit eval runs first because it is fast and catches the most common regression pattern — a system prompt edit that breaks a known behaviour — before the more expensive agent eval is invoked. Red-team eval runs last in the pre-production sequence because it exercises scenarios that depend on the agent's full configuration; running it on partially-integrated components produces misleading results. Online eval is the only surface that is always active; the pre-production surfaces run on change events.
