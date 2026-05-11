# Document Intake Agent

The Document Intake Agent automates the ingestion of unstructured documents — contracts, invoices, regulatory filings — into enterprise systems. It runs on AWS Bedrock, uses a pipeline orchestration pattern, and applies a HITL gate before any write to the downstream system of record. The pipeline is intentionally decomposed rather than monolithic: a single "extract everything" model call collapses on scanned documents, multi-page layouts, and edge cases that benefit from rule-based validation. Decomposition makes each failure mode addressable independently.

## Pipeline Overview

```
  Inbound Document (S3 trigger / API)
          │
          ▼
  ┌───────────────┐
  │  Classify     │  Claude Haiku (Bedrock Converse)
  │  Worker       │  → document_type, confidence_score
  └───────┬───────┘
          │ if confidence < threshold → HITL: "Classify manually"
          │
          ▼
  ┌───────────────┐
  │  Extract      │  Amazon Textract (READ)
  │  Worker       │  → structured fields per document_type schema
  └───────┬───────┘
          │
          ▼
  ┌───────────────┐
  │  Validate     │  Schema validation (rule-based, no model call)
  │  Worker       │  → validation_result, field_errors
  └───────┬───────┘
          │ if validation_errors → HITL: "Correct extracted fields"
          │
          ▼
  ┌───────────────┐
  │  HITL Gate    │  Human reviews before write
  │  (if flagged) │  → approved / rejected
  └───────┬───────┘
          │ approved
          ▼
  ┌───────────────┐
  │  Persist      │  WRITE_INTERNAL (system of record API)
  │  Worker       │  → record_id, audit_reference
  └───────────────┘
```

## Component Map

| Component | Spine Role | AWS Service | Risk Class |
|---|---|---|---|
| Classify Worker | Worker | Bedrock Converse (Haiku) | READ (model invocation only) |
| Extract Worker | Tool Executor | Amazon Textract | READ |
| Validate Worker | Worker | Rule engine (no model) | READ |
| HITL Gate | HITL Gate | Step Functions wait-for-token | N/A |
| Persist Worker | Tool Executor | Internal API | WRITE_INTERNAL |

The Validate Worker uses no model call. Validation is deterministic: schema conformance, required-field presence, and cross-field rules (for example, origin port must not equal destination port). Keeping validation rule-based ensures that validation failures are reproducible and debuggable without model invocation costs or non-determinism. Only when validation passes does the pipeline escalate to the HITL gate or, if no gate is required, proceed directly to the Persist Worker.

## HITL Policy

Two HITL gates exist. The first fires when the classifier's confidence falls below a configured threshold (default: 0.85); the human assigns the document type, and the pipeline resumes from the Extract Worker with the corrected type. The second fires when field validation fails; the human corrects the extracted fields, and the pipeline resumes from the Validate Worker with the corrected values. Both gates have a configurable timeout and a default action — reject the document and notify the submitter — if the gate is not resolved within the TTL. Human corrections at either gate are written to the eval set to improve future classifier and extractor performance.

## Failure Modes Specific to This Agent

| Failure | Signal | Remediation |
|---|---|---|
| Low-quality scan | Textract returns sparse output | Pre-process with image enhancement; add scan-quality check before extraction |
| Novel document type | Classifier confidence low on unknown type | Expand classifier training data; add explicit "unknown" class to schema |
| Field schema drift | Validation failure rate increases | Update document_type schemas; re-run eval on updated schema |
