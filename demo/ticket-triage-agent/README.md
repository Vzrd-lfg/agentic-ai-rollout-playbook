# Ticket Triage Agent

The Ticket Triage Agent classifies, enriches, and routes incoming IT and operational tickets. It runs on AWS Bedrock and applies a three-layer model strategy: a lightweight classifier for standard cases, a planner for ambiguous cases that require decomposition, and a drafter for cases requiring a response before routing. The layered strategy is the primary cost control mechanism: the goal is to resolve the highest volume of tickets at the lowest model tier while reserving more capable models for the cases where additional reasoning demonstrably improves the outcome.

## Pipeline Overview

```
  Inbound Ticket (webhook / API)
          │
          ▼
  ┌──────────────────┐
  │ Normalise Worker │  Rule-based (no model)
  │                  │  → structured_ticket (priority, category_hint, metadata)
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ Haiku Classifier │  Bedrock Converse (Haiku) — READ
  │ Worker           │  → category, confidence, suggested_assignee
  └────────┬─────────┘
           │
     ┌─────┴──────┐
     │ confident? │
     Yes          No
     │             │
     ▼             ▼
  ┌──────┐   ┌──────────────────┐
  │Route │   │ Planner Worker   │  Sonnet — decomposes ambiguous ticket
  │ out  │   │ (if ambiguous)   │  → clarification_plan or sub-tasks
  └──────┘   └────────┬─────────┘
                      │
                      ▼
             ┌──────────────────┐
             │ Draft Response   │  Sonnet (if acknowledgment needed)
             │ Worker           │  → draft_response (HITL before send)
             └────────┬─────────┘
                      │
                      ▼
             ┌──────────────────┐
             │ HITL Gate        │  Human reviews draft before WRITE_EXTERNAL
             └────────┬─────────┘
                      │ approved
                      ▼
             ┌──────────────────┐
             │ Execute Worker   │  Route + (optionally) send response
             │                  │  WRITE_INTERNAL + WRITE_EXTERNAL
             └──────────────────┘
```

## Three-Layer Model Strategy

| Layer | Model | Task | Rationale |
|---|---|---|---|
| Layer 1 | Haiku (Bedrock) | Classification and confidence scoring | Low cost; high speed; >90% of tickets resolved here |
| Layer 2 | Sonnet (Bedrock) | Ambiguity resolution (planning) and drafting | Used only when Haiku confidence is below threshold |
| Layer 3 | Human | HITL review of WRITE_EXTERNAL actions | HITL gate on all external communications |

The model strategy is the primary cost control in this agent. In a well-tuned deployment, 85–90% of tickets are classified and routed by Haiku without invoking Sonnet. The threshold for escalating to Sonnet is tunable; the eval harness measures the accuracy-cost trade-off at each threshold setting. The threshold is owned jointly by the operations team (who care about routing accuracy) and the product owner (who owns the cost budget); changes to it are governed by the same change-control process as changes to the routing policy.

## HITL Policy

The HITL gate fires before any WRITE_EXTERNAL action — sending a response to the ticket submitter. It does not fire for WRITE_INTERNAL actions such as routing the ticket within the system or updating the internal record. This asymmetry is intentional: internal routing errors are recoverable and auditable; external communications to submitters are not. The timeout is 30 minutes; if the HITL gate is not resolved within that window, the ticket is routed without a response and flagged for human follow-up. The flag is written to the audit log and surfaced on the operations dashboard.

## Failure Modes Specific to This Agent

| Failure | Signal | Remediation |
|---|---|---|
| High Sonnet escalation rate | Cost per ticket above budget | Lower classifier confidence threshold; expand Haiku training data |
| Incorrect routing | Misrouted ticket rate in feedback | Add routing eval to golden set; improve category taxonomy |
| Draft response quality | HITL rejection rate > 15% | Tighten drafter system prompt; add failed drafts to eval set |
