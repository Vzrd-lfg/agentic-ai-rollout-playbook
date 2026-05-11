# Agentic AI Rollout Playbook

A reusable, opinionated blueprint for deploying agentic AI at enterprise scale — on AWS Bedrock + AgentCore and GCP Vertex AI — in regulated environments.

![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)

---

##Why I built this — and what the playbook is for

I've spent the last several years building production agentic AI systems in compliance-sensitive environments — sanctions screening, proptech matching, logistics document intelligence — and the same conversation keeps recurring.

A new programme starts. A pilot succeeds. The pilot doesn't scale. Six months in, the team has shipped one impressive demo and is stuck on the second. Or worse: they've shipped widely, an incident happens, and the whole programme is paused while Legal, InfoSec, and Compliance work out what just happened.

The pattern is so consistent that I started writing down the operating discipline that does work — the architectural decisions, the governance scaffolding, the rollout sequencing, the ways agents actually fail in production — as a reusable playbook. This repository is that playbook.

---

## Three things the playbook argues

**1. The failure rate on complex agent tasks runs at 60–65%, and smarter foundation models will not, on their own, fix this.** The dominant failure modes are architectural: scope (the agent's job is too broad), composition (95% × 10 steps = 60% end-to-end accuracy), observability (the agent fails and nobody knows where), reliability (the same input produces different outputs), and drift (the agent worked at launch and is worse six months later). See [evaluation/failure-modes.md](evaluation/failure-modes.md).

**2. The orchestration spine is the product, not the agent.** Planner → Router → Worker → Tool Executor, with four risk classes on the tool boundary (READ / WRITE_INTERNAL / WRITE_EXTERNAL / IRREVERSIBLE) and Cedar or OPA policy enforced *outside the model*. New use cases are configuration on the spine — tools, prompts, eval sets — not bespoke architecture. McKinsey calls this the "agentic mesh"; LinkedIn's published tech stack is the same idea operationalised. See [orchestration/](orchestration/) for the patterns and [docs/02-design-principles.md](docs/02-design-principles.md) for the operating positions.

**3. Governance is scaffolded in on day one, not bolted on the week before audit.** An eight-check binary Approval Gate. An AI System Register. EU AI Act / ISO 42001 / NIST AI RMF / SR 11-7 / GDPR / DORA mapped to specific artefacts. The OWASP LLM Top-10 mapped to the controls already in the architecture. None of this is theoretical — it's what regulators expect to see. See [governance/](governance/).

## What's published, and what isn't

The patterns, frameworks, taxonomies, rollout discipline, and case-study evidence are published. The operational depth that *actually* differentiates production rollouts — the iterated prompts, the curated golden sets, the policy packs, the IaC modules, the change-management collateral — is not.

---

## How to engage

- **If you're a hiring manager** looking at this because you're evaluating me for an AI Programme / Director / Adoption role: the [49-page PDF playbook and 8-slide deck](mailto:vishnu.sathiap@gmail.com) expand on every section here and are available on request.
- **If you're a peer practitioner** running an agentic programme: use the patterns freely under [CC BY-NC 4.0](LICENSE). If you'd like to compare notes, open a Discussion.
- **If you're an organisation** that would like the operational depth — Approval Gate templates, eval harness implementations, prompt libraries, policy packs, change-management collateral — those are part of consulting engagements. [Get in touch](mailto:vishnu.sathiap@gmail.com).

Thanks for reading. The repository is a living document; the failure-mode catalogue alone grows every quarter from production incidents.

---

## The Core Argument

Most agentic AI programmes fail in one of three ways: a successful proof-of-concept that cannot survive the transition to production (*pilot purgatory*); a single over-scoped agent attempting too many tasks, failing unpredictably, and impossible to debug (*hero agents*); or agents touching production systems with no audit trail, no approval process, and no rollback plan until an incident forces a programme pause (*ungoverned rollouts*).

The failure rate on complex multi-step agent tasks runs at 60–65% in production. This is not a model quality problem. The dominant failure modes — scope, composition, observability, reliability, drift — are architectural. A well-scoped agent on a capable-but-not-frontier model consistently outperforms an over-scoped agent on the most powerful model available.

This playbook documents the operating discipline that addresses those failures: a reusable orchestration spine, a governance scaffolding that satisfies EU AI Act and ISO 42001 from day one, and a 90-day rollout plan with gate criteria drawn from fifteen production deployments.

---

## What This Playbook Contains

| Section | What It Covers |
|---|---|
| `docs/` | Executive summary, design principles, reference architecture, platform decision matrix, Bedrock AgentCore primitives |
| `architecture/` | System context and component interaction diagrams |
| `orchestration/` | Four patterns: Planner, Router, Worker, Tool Executor, and multi-agent topology guidance |
| `integrations/` | API gateway, ten RAG patterns with decision tree, workflow engine integration, identity and secrets |
| `governance/` | Eight-check Approval Gate, AI System Register, access control, auditability, OWASP LLM Top-10, EU AI Act / ISO 42001 / NIST AI RMF / SR 11-7 / GDPR / DORA mapping |
| `evaluation/` | Eval harness (five axes, four surfaces), observability, token efficiency, five failure modes, eval-suite scaffold |
| `rollout/` | 90-day rollout plan, operating model, enablement playbook, change management, fifteen case studies |
| `demo/` | Architecture documentation for three reference agents: document intake, policy Q&A, ticket triage |
| `examples/` | Illustrative pseudo-code (NOT_PRODUCTION — DEMONSTRATION ONLY) for Bedrock, AgentCore, and Vertex AI |
| `infrastructure/` | Pattern-layer infrastructure documentation (no IaC modules) |

---

## Platform Stance

**Primary: AWS Bedrock + AgentCore.** The default for regulated, tool-use-heavy workloads. Bedrock's Converse API abstracts across Claude, Titan, Llama, and third-party models. AgentCore provides nine managed primitives — Memory, Sessions, Executor, Tools, Guardrails, Observability, Model Access, Identity, Policies — that together constitute a production agent lifecycle. PrivateLink egress contains data within the enterprise boundary.

**Secondary: GCP Vertex AI.** The preferred platform for knowledge Q&A and Google Workspace-integrated use cases. Vertex AI Search provides enterprise-grade retrieval over structured and unstructured corpora. Gemini's function-calling interface is well-suited to RAG-heavy agents. Many production deployments use Bedrock for intake and triage agents and Vertex for policy and knowledge agents.

Neither platform is exclusive. The orchestration spine is platform-agnostic; what changes is the model invocation layer and the retrieval backend.

---

## Reading Paths

| Persona | Start Here | Then Read |
|---|---|---|
| CTO / Chief Architect | `docs/01-executive-summary.md` → `docs/02-design-principles.md` | `docs/03-reference-architecture.md` → `docs/05-bedrock-agentcore.md` → `rollout/90-day-plan.md` |
| CISO / Compliance Lead | `governance/ai-governance-framework.md` | `governance/agent-security.md` → `governance/eu-ai-act-mapping.md` → `evaluation/failure-modes.md` |
| AI Platform Lead | `orchestration/README.md` (all patterns) | `integrations/rag-architecture.md` → `evaluation/eval-harness.md` → `evaluation/observability.md` |
| Programme / Delivery Lead | `rollout/90-day-plan.md` | `rollout/operating-model.md` → `rollout/change-management.md` → `rollout/case-studies.md` |

---

## Using This Playbook

This repository is licensed under [CC BY-NC 4.0](LICENSE). You may use, adapt, and share the patterns and frameworks for non-commercial purposes with attribution. Commercial use requires written permission.

See [DISCLAIMER.md](DISCLAIMER.md) for a precise statement of what is and is not published here. The patterns and frameworks are sufficient to save four to six weeks of trial-and-error design; the operational depth that differentiates production rollouts — curated golden sets, Cedar policy packs, IaC modules, change-management collateral — is reserved for consulting engagements.

Feedback and issue reports are welcome via GitHub Issues. If you encounter a failure mode not covered in `evaluation/failure-modes.md`, or a regulatory framework not mapped in `governance/eu-ai-act-mapping.md`, please open an issue.

---

## Citation

If you use this playbook in research or published work, please cite it using the metadata in [CITATION.cff](CITATION.cff).

---

*Built on fifteen production deployments across financial services, logistics, healthcare, and enterprise SaaS. Author: Vishnu Sathiapathi.*
