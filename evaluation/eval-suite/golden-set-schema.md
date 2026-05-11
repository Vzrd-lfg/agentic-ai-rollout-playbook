# Golden Set Schema

A golden test case is a curated input-output pair with ground truth. Golden sets are the primary artefact of agent evaluation: they define what correct behaviour looks like for the specific agent under test, across the range of inputs that the agent is expected to handle in production. This schema defines the structure of a single golden test case. Every test case in every agent's golden set must conform to this schema to be consumable by the eval runner.

The schema is intentionally broad — not every field applies to every agent type. A classification agent does not use `citations_required`; a RAG agent does not use `decision`. The `expected` and `ground_truth` sections are populated according to the agent's task type. Fields that are not applicable are set to `null`.

## YAML Schema

```yaml
# Golden test case schema — version 1.0
test_case:
  id: string                  # unique identifier, e.g. "tc-001"
  version: string             # schema version, e.g. "1.0"
  agent_id: string            # agent this case tests
  tags:                       # free-form tags for filtering
    - string
  input:
    user_message: string      # the user turn
    context: object           # optional structured context (e.g. document, prior turns)
    session_id: string        # optional, for multi-turn tests
  expected:
    output_contains: [string] # key phrases that must appear in the response
    output_excludes: [string] # phrases that must not appear
    citations_required: bool  # for RAG agents: must cite source?
    decision: string          # for classification agents: expected class
    tool_calls:               # expected tool invocations (order-independent)
      - tool_id: string
        risk_class: string
  ground_truth:
    source_document_id: string   # for RAG agents: the source the answer must be grounded in
    authoritative_answer: string # human-written reference answer for LLM judge comparison
  metadata:
    created_by: string
    created_date: date
    last_reviewed: date
    difficulty: enum          # [easy, medium, hard, adversarial]
    failure_mode_category: string  # which failure mode this case tests
```

The `failure_mode_category` field in metadata is used to filter the golden set by the failure mode categories defined in `evaluation/failure-modes.md`. A test case tagged `composition` exercises a multi-step hand-off; a case tagged `reliability` exercises a task where consistency across runs is the correctness criterion; a case tagged `adversarial` is part of the red-team surface. This tagging enables the eval runner to report pass/fail rates by failure mode category, which identifies which categories of failure the agent is most exposed to.

## Example Record

```yaml
test_case:
  id: "tc-042"
  version: "1.0"
  agent_id: "policy-qa-agent"
  tags: ["gdpr", "data-retention", "faq"]
  input:
    user_message: "How long do we retain customer data under our current policy?"
    context: {}
  expected:
    output_contains: ["retention period", "data controller"]
    output_excludes: ["I don't know", "I cannot answer"]
    citations_required: true
    decision: null
    tool_calls:
      - tool_id: "vertex_search"
        risk_class: "READ"
  ground_truth:
    source_document_id: "doc-privacy-policy-v4"
    authoritative_answer: "Customer data is retained for 7 years after account closure per policy section 4.2."
  metadata:
    created_by: "qa-team"
    created_date: "2026-01-15"
    last_reviewed: "2026-03-01"
    difficulty: "easy"
    failure_mode_category: "reliability"
```

This example illustrates a straightforward RAG test case: the agent must retrieve from a known source document, cite that source, and include specific phrases. The `failure_mode_category` is `reliability` because this is a high-frequency, deterministic query — the agent should return consistent, citation-backed answers on every run. The `output_excludes` list enforces that the agent does not refuse or hedge on a question it has clear source material to answer.
