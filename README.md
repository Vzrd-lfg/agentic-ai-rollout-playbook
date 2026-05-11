# Agentic AI Rollout Playbook

A reusable, opinionated blueprint for deploying agentic AI at enterprise scale — on AWS Bedrock + AgentCore and GCP Vertex AI — in regulated environments.

![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)

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
