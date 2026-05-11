# 90-Day Rollout Plan

The 90-day plan is not a project plan — it is a sequencing discipline. It specifies what must be true at the end of each phase before the programme advances. A phase that overruns by two weeks is a diagnostic signal; a phase that is skipped is a governance failure.

## Phase Overview

```
Week  1  2  3  4  5  6  7  8  9  10 11 12 13+
      ├──Discovery──┤
         ├────Architecture────┤
                     ├──────────Pilot──────────┤
                                    ├──────Scale──────┤
                                                ├──Operate──►
```

Discovery and Architecture overlap by design: use case prioritisation informs infrastructure decisions, and infrastructure constraints sharpen the use case selection. The Pilot phase begins when both prior phases have met their gate criteria — not before. Scale and Operate similarly overlap: operational discipline must begin the moment the first agent is live.

## Phase 1 — Discovery (Weeks 0–2)

**Objective**: Establish the use case portfolio, risk classification, and executive mandate.

| Deliverable | Owner | Gate Criterion |
|---|---|---|
| Use case map (≥5 prioritised use cases, scored by value × feasibility) | AI Programme Owner | Approved by Steering Committee |
| Risk classification for top 3 use cases | AI Governance Lead | Documented in AI System Register |
| Data landscape assessment | AI Platform Lead | All data sources identified; DPAs initiated |
| Executive framing (3-point brief: problem, approach, ask) | AI Programme Owner | Steering Committee aligned |

Discovery is a rapid intake exercise, not a six-week consulting engagement. The workshops that produce the use case map should run within the first five business days. The Programme Owner selects the Phase 1 candidate before Discovery closes; Architecture cannot begin with full clarity until that selection is made.

The data landscape assessment is the deliverable most frequently deferred and most frequently regretted. DPAs, data residency constraints, and access control requirements have long lead times. Initiating them at week one rather than week four is the difference between shipping the pilot at day 30 and shipping it at day 60.

## Phase 2 — Architecture (Weeks 0–4, overlaps Discovery)

**Objective**: Build the orchestration spine and governance scaffolding before any use case is deployed.

| Deliverable | Owner | Gate Criterion |
|---|---|---|
| Orchestration spine deployed in non-production | AI Platform Lead | Planner/Router/Worker/Tool Executor all invocable |
| Approval Gate process documented and socialised | AI Governance Lead | Gate checklist approved |
| AI System Register created | AI Governance Lead | First entry: reference implementation |
| Reference implementation (one synthetic agent on the spine) | AI Platform Lead | Passes eval harness baseline |
| Observability stack live | AI Platform Lead | Traces visible in dashboard |

The spine is the prerequisite for everything that follows. An agent built before the spine exists is a bespoke system, not a programme asset. The Architecture phase exists to prevent that outcome. The reference implementation is not a demo — it is the first agent that actually exercises the spine end-to-end, including the eval harness, the audit log, and the observability stack.

## Phase 3 — Pilot (Days 1–30): Gate 1

**Objective**: First production agent live; governance working; eval passing; rollback tested.

Gate 1 is the first hard stop. All eight Approval Gate checks must pass before the agent enters production. There are no partial passes.

**Gate 1 Criteria:**

1. Business case approved
2. Risk class documented
3. Data access inventory complete and DPAs signed
4. Model on approved model list
5. Eval baseline met (all five axes)
6. Security review complete (architecture review; pen test if High-risk)
7. HITL policy defined and tested
8. Rollback plan tested (disable agent in under 5 minutes)

| Deliverable | Owner | Gate Criterion |
|---|---|---|
| First agent live in production | AI Platform Lead + BU AI Owner | Real users, not a demo environment |
| Audit trail active | AI Platform Lead | Immutable log producing queryable records |
| HITL gate tested in production | BU AI Owner | At least one escalation exercised and resolved |
| First online eval sample captured | AI Platform Lead | Online metrics visible in dashboard |

The pilot phase ends when Gate 1 is passed, not when the calendar reaches day 30. A pilot that is feature-complete but has not passed the eval baseline does not exit the phase. The Programme Owner documents the gap and the target date; the Steering Committee is notified.

## Phase 4 — Scale (Days 31–60): Gate 2

**Objective**: Three agents live across two platforms; demand emerging from business units; self-service path defined.

**Gate 2 Criteria:** Three agents have passed Gate 1. At least one Bedrock agent and one Vertex agent. BU AI Owner role filled for at least one business unit. Use case intake process — how business units request new agents — documented and communicated to all in-scope BUs.

| Deliverable | Owner | Gate Criterion |
|---|---|---|
| Three agents in production | AI Platform Lead + BU AI Owners | Each has passed Gate 1 independently |
| BU AI Owner onboarded | AI Programme Owner | Named individual; AI System Register entries held |
| Use case intake process live | AI Programme Owner | Intake form published; at least one BU submission received |
| Cost dashboard live | AI Platform Lead | Per-agent cost visible; model-level breakdown available |

The critical discipline of this phase is that each new agent goes through the full Approval Gate. The spine being established does not accelerate the gate. The gate evaluates the agent, not the infrastructure.

## Phase 5 — Operate (Days 61–90+): Gate 3

**Objective**: Programme is self-sustaining. Platform team supports rather than builds. AI Council established.

**Gate 3 Criteria:** AI Council has met at least once. At least one BU team has shipped a use case without platform team implementation support. Drift monitoring has run for 30 or more days without a critical alert. Enablement programme across all three populations has run at least one cohort.

| Deliverable | Owner | Gate Criterion |
|---|---|---|
| AI Council established | AI Programme Owner | Terms of reference signed; at least one meeting held |
| Monthly programme reporting cadence live | AI Programme Owner | Dashboard published; first report distributed |
| At least one BU-originated use case in production | BU AI Owner | Built without platform team as primary implementors |

Gate 3 marks the transition from programme delivery to programme operations. The measure of success is not how many agents the platform team has built; it is whether the programme can sustain itself without the platform team as the bottleneck for every new use case.

## Common Failure Patterns

| Failure Pattern | Root Cause | Recovery |
|---|---|---|
| Architecture phase skipped | Pressure to show a demo | Stop. Build the spine. A demo without a spine does not scale. |
| Gate criteria softened | Stakeholder impatience | Gate criteria are not negotiable. Softening them is a governance failure. |
| Pilot phase extended indefinitely | Fear of the second use case | Pilot purgatory is the problem this plan solves. Gate 1 is the exit. |
