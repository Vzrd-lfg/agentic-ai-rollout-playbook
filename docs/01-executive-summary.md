# Executive Summary

## The Problem

Enterprise agentic AI programmes fail in predictable ways. Three failure modes account for the vast majority of programmes that stall, get paused after an incident, or quietly produce nothing of value.

**Pilot Purgatory** is the most common. A proof-of-concept succeeds under controlled conditions — a well-chosen dataset, a permissive environment, a motivated champion, and a skilled individual who held it all together. That individual's institutional knowledge, their tolerance for prompt iteration, their workarounds for brittle dependencies — none of it survives the handoff to a production team. The agent cannot be explained, cannot be tested systematically, and cannot be maintained by anyone who was not in the room. The programme is paused while the organisation decides what to do next, and the decision is usually to start over.

**Hero Agents** are the architectural equivalent of a function that does everything. A single agent is scoped to handle the full breadth of a business process — intake, classification, retrieval, drafting, routing, escalation, and reporting — because the original design optimised for demo clarity rather than operational tractability. When the agent fails, no one can determine which step failed. The trace, if one exists, is a wall of text. The fix, if one is attempted, requires understanding the entire system at once. Debugging is replaced by prompt archaeology. The agent is quietly retired or handed back to a team that monitors it manually.

**Ungoverned Rollouts** produce incidents. An agent is given write access to a production system — a CRM, a ticketing platform, a customer-facing email service — without a defined approval process, without an audit trail that satisfies a compliance review, without a rollback plan, and without any ongoing evaluation of whether the agent's behaviour has drifted since it was first approved. The incident, when it arrives, is not a surprise to the engineers who built it. It is a surprise to the business. The programme is paused while governance is retrofitted — which is always harder than building it in.

| Failure Mode | Root Cause | Consequence | What Fixes It |
|---|---|---|---|
| Pilot Purgatory | Hero code, undocumented prompts, no eval harness | Cannot survive handoff to production team | Reusable spine, typed tools, golden-set eval from day one |
| Hero Agents | Over-scoped single agent with no step isolation | Unpredictable failure, impossible to debug | Planner/router/worker decomposition; narrow per-agent scope |
| Ungoverned Rollouts | Write access to production with no audit or approval process | Incident forces programme pause; governance retrofitted under pressure | Approval Gate, risk-classed tools, HITL by design, immutable audit log |

## The Hypothesis

Architecture matters more than model choice. An agent with a well-defined scope, a typed tool registry, a continuous eval harness, and a human-in-the-loop policy on irreversible actions will outperform an agent built on a more powerful model but without those properties. The evidence from production deployments is consistent on this point: the agents that fail do not fail because the model was insufficiently capable. They fail because the scope was too broad, the failure modes were not designed for, and no one was measuring whether the agent was still performing correctly three weeks after launch.

The orchestration spine — planner, router, worker, tool executor — is the durable investment. It is the structure that makes the second, fifth, and fifteenth agent fast to deliver, because each new use case is configuration on the spine: new tools, new system prompts, new eval sets. The model sitting behind that spine is a swappable implementation detail. Frontier models are replaced by the next frontier model on a cycle measured in months. The spine, if built correctly, is not replaced at all.

## What This Playbook Delivers

This playbook provides four things that an enterprise needs to move from a successful proof-of-concept to a governed, operating agentic AI programme.

**A reusable orchestration spine** applicable to any use case, structured as four composable components — planner, router, worker, tool executor — with typed tool schemas, risk classification, HITL gates, and an immutable audit log. The spine is platform-agnostic at the pattern level and maps onto AWS Bedrock AgentCore and GCP Vertex AI at the implementation level.

**A governance scaffolding** that satisfies EU AI Act, ISO 42001, and NIST AI RMF requirements from the first agent, not as a retrofit. This includes an eight-check Approval Gate, an AI System Register, a risk classification taxonomy, and an audit trail architecture that survives regulatory review.

**A 90-day rollout plan** with three phase gates — Pilot (day 30), Scale (day 60), Operate (day 90+) — each with explicit exit criteria, RACI assignments, and go/no-go decision authority. The plan is designed so that a programme can demonstrate measurable production value within 30 days without sacrificing the governance and observability properties that make it extensible.

**Fifteen documented case studies** drawn from published production deployments, anchoring every architectural pattern in demonstrated outcomes. The case studies are not aspirational. They document what was built, what the constraints were, and what the result was — including where the first version underperformed and what the fix was.

## Who This Is For

**Technical architects** will find the reference architecture, platform decision matrix, orchestration patterns, RAG architecture guide, and tool taxonomy directly applicable to design decisions. The reading path starts at `docs/03-reference-architecture.md` and proceeds through `orchestration/`, `integrations/`, and `evaluation/`. The platform-specific detail is in `docs/04-platform-decision-matrix.md` and `docs/05-bedrock-agentcore.md`.

**CISO and compliance leads** will find the governance framework, Approval Gate specification, AI System Register template, and regulatory mapping in `governance/`. The risk classification taxonomy and audit trail architecture are designed to be presented to regulators directly. The relevant regulatory frameworks — EU AI Act Article 9, ISO 42001 clause 6.1, NIST AI RMF Govern and Manage functions — are mapped explicitly to the artefacts this playbook produces.

**Programme leads** will find the 90-day rollout plan, operating model, use-case pipeline methodology, and RACI templates in `rollout/`. The case studies in `rollout/case-studies.md` provide reference points for scoping, resourcing, and setting expectations with executive sponsors. The reading path for programme leads is `docs/01-executive-summary.md` → `rollout/` → `docs/02-design-principles.md`.

The README.md reading-path table maps each audience to the specific sections most relevant to their role.

## What Is Not Here

This playbook does not constitute legal advice, regulatory guidance, or a compliance certification. It describes architectural and operational patterns that have been applied in regulated environments, but every enterprise's regulatory context is different. Before presenting any artefact from this playbook to a regulator, it must be reviewed by qualified legal and compliance counsel. See DISCLAIMER.md for the full scope of what this playbook does and does not represent.
