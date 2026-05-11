# Demonstration Agents

This directory documents three reference agents that illustrate the orchestration spine applied to representative enterprise use cases. Each agent is documented at the architectural level only — the intent is to provide a concrete instantiation of the patterns described in `orchestration/` and `integrations/`, not to provide a codebase. Readers should treat these as worked examples: each one shows how the spine's abstractions resolve into a specific pipeline, component map, failure-mode inventory, and HITL policy for a real problem class.

## Why These Three Agents

The three agents were chosen to cover the two primary platforms (Bedrock and Vertex AI), three distinct orchestration patterns (pipeline, RAG, multi-layer routing), and three distinct data types (structured documents, policy text, and operational tickets). No two agents share a primary pattern or a primary data type, which means the set collectively exercises the full configurability of the spine. Together they demonstrate that the orchestration spine, the eval harness, the audit schema, and the HITL gate are stable invariants; only the tools, prompts, and worker configurations change between them.

## Agent Overview

| Agent | Platform | Primary Pattern | Use Case Domain | Build Estimate |
|---|---|---|---|---|
| Document Intake | AWS Bedrock | Pipeline (extract → validate → persist) | Document processing | 2–3 weeks |
| Policy Q&A | GCP Vertex AI | Hybrid RAG + cite-or-refuse | Compliance / knowledge | 1–2 weeks |
| Ticket Triage | AWS Bedrock | Multi-layer routing + conditional planning | IT / operations | 2–3 weeks |

## Architecture Philosophy

Each demo directory contains an architecture README only. The README documents the agent's pipeline, its component map (which spine components it uses and how), its failure modes, and a data-flow diagram. There is no application code in this directory. The reason: the patterns are the transferable artefact; the code is the engagement. Any competent engineering team can implement a pipeline that matches the documented architecture; the value of this playbook is in the architectural decisions — the choice of orchestration pattern, the placement of HITL gates, the failure-mode inventory, and the eval harness design — not in any particular implementation.

## Build Sequencing

If building all three agents, the recommended sequence is: Policy Q&A first (simplest architecture; validates the retrieval stack and the cite-or-refuse discipline with minimal infrastructure), then Document Intake (introduces pipeline orchestration and Textract integration, and exercises the WRITE_INTERNAL risk class for the first time), then Ticket Triage (most complex; introduces multi-layer model tiering, conditional planning, and WRITE_EXTERNAL actions gated behind HITL). Each agent can be built independently; the sequencing recommendation exists to de-risk the build by ordering complexity.

```
  Build sequence (recommended)

  ┌───────────────────┐     ┌──────────────────────┐     ┌──────────────────────┐
  │  Policy Q&A       │ ──► │  Document Intake      │ ──► │  Ticket Triage       │
  │  (1–2 weeks)      │     │  (2–3 weeks)          │     │  (2–3 weeks)         │
  │                   │     │                       │     │                      │
  │  Validates:       │     │  Adds:                │     │  Adds:               │
  │  · retrieval stack│     │  · pipeline pattern   │     │  · model tiering     │
  │  · cite-or-refuse │     │  · Textract tooling   │     │  · conditional plan  │
  │  · eval harness   │     │  · WRITE_INTERNAL     │     │  · WRITE_EXTERNAL    │
  └───────────────────┘     └──────────────────────┘     └──────────────────────┘
```
