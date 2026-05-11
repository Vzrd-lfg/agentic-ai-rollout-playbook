# Policy Q&A Agent

The Policy Q&A Agent answers questions about internal policies, compliance requirements, and regulatory guidance. It runs on GCP Vertex AI, uses a Hybrid RAG pattern, and applies a cite-or-refuse discipline: every answer either cites the source document and passage, or explicitly refuses to answer when grounding confidence is insufficient. This discipline is not optional; it is enforced at the architectural level by a rule-based gate that inspects every generator output before it is returned to the caller. An agent that occasionally answers without citation is an agent that will eventually hallucinate policy.

## Pipeline Overview

```
  User Question
        │
        ▼
  ┌──────────────┐
  │ Query Rewrite │  Gemini Flash (function call)
  │  Worker      │  → rewritten_query (improves retrieval recall)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Vertex AI    │  Hybrid retrieval (dense + keyword)
  │ Search       │  → top_k_chunks with source_ids
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Re-rank     │  Cross-encoder re-ranker
  │  Worker      │  → ranked_chunks (top 3–5)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Gemini Pro   │  Grounded generation
  │ (generator)  │  → response + citation_ids + confidence_score
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Cite or      │  if confidence >= threshold → answer with citations
  │ Refuse       │  if confidence < threshold → "I cannot answer this
  └──────────────┘    with confidence. Please consult [source]."
```

## Component Map

| Component | Spine Role | GCP Service | Risk Class |
|---|---|---|---|
| Query Rewrite Worker | Worker | Vertex AI (Gemini Flash) | READ |
| Retrieval | Tool Executor | Vertex AI Search | READ |
| Re-rank Worker | Worker | Vertex AI Ranking API | READ |
| Generator | Worker | Vertex AI (Gemini Pro) | READ |
| Cite/Refuse Gate | Worker (rule-based) | N/A | READ |

This agent is read-only end to end. No worker in this pipeline issues a WRITE_INTERNAL or WRITE_EXTERNAL action. The Cite/Refuse Gate is implemented as a rule-based worker, not a model call, to ensure that the enforcement logic is deterministic and auditable. The gate inspects the generator's output for citation_ids and confidence_score; it does not ask a model to judge whether the answer is well-grounded.

## Cite-or-Refuse Discipline

Every response from this agent includes a citation to the source document and passage, or it refuses to answer. The confidence threshold for grounding is set at the Approval Gate and recorded in the AI System Register. Reducing the threshold increases answer rate but reduces faithfulness; the eval harness measures the trade-off on the golden set, which must include questions that should be refused (out-of-scope, unanswerable from corpus) as well as questions that should be answered. The refusal message includes a pointer to where the user can find an authoritative answer — a named team or document — so the refusal is actionable rather than a dead end.

## Failure Modes Specific to This Agent

| Failure | Signal | Remediation |
|---|---|---|
| Retrieval corpus out of date | Faithfulness score drops on policy-update questions | Implement corpus refresh pipeline triggered by document publish events |
| Query vocabulary mismatch | Low retrieval recall on technical questions | Contextual RAG or HyDE for technical domains; corpus curation |
| Over-refusal | Refusal rate > 20% on in-scope questions | Lower confidence threshold; improve re-ranking quality; expand golden set |
