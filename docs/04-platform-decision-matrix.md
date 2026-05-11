# Platform Decision Matrix

This playbook treats AWS Bedrock with AgentCore as the default platform for regulated, tool-use-heavy workloads and GCP Vertex AI as the secondary platform for knowledge Q&A and Google Workspace-integrated use cases. Neither is exclusive. Many production enterprises run both, arriving at a dual-cloud posture not by design but by following the workload: document intake and support automation land on Bedrock because of Claude's extraction quality and AgentCore's managed tool lifecycle; policy Q&A and document search land on Vertex because of Vertex AI Search's grounding quality on enterprise corpora. The orchestration spine — planner, router, worker, tool executor — is platform-agnostic at the pattern level. What changes across platforms is the model invocation layer and the retrieval backend.

## Capability Comparison

| Capability | AWS Bedrock + AgentCore | GCP Vertex AI | Notes |
|---|---|---|---|
| Regulated data residency | VPC-contained, PrivateLink; no cross-region egress by default | VPC-SC, regional endpoints, EU sovereign cloud option | Both satisfy most enterprise data-residency requirements; verify against specific regulatory jurisdiction |
| Structured tool use (function calling) | Native via Converse API; abstracts across model families | Native via Gemini function calling | Bedrock Converse normalises tool-calling syntax across Claude, Titan, Llama, and others |
| Knowledge retrieval (RAG) | Knowledge Bases (OpenSearch Serverless, Aurora PostgreSQL pgvector) | Vertex AI Search, Vertex AI RAG Engine | Vertex AI Search is stronger on Google Workspace corpora; Bedrock Knowledge Bases integrates more cleanly with S3 data estates |
| Agent orchestration primitives | AgentCore (nine primitives: Runtime, Memory, Sessions, Executor, Tools, Guardrails, Observability, Identity, Policies) | Vertex AI Reasoning Engine, Agent Builder | AgentCore is purpose-built for enterprise agent lifecycle; Vertex Reasoning Engine requires more custom wiring |
| Memory across sessions | AgentCore Memory (short-term and long-term; managed vector store) | Vertex Agent Builder memory | AgentCore memory provides both session-scoped and cross-session long-term storage |
| Identity integration | IAM, Cognito, IAM Identity Center; on-behalf-of token propagation | Identity Platform, Workforce Identity Federation | AWS integrates more cleanly with enterprise Active Directory via IAM Identity Center |
| Model breadth | Claude (Sonnet, Haiku, Opus), Titan, Llama, Mistral, Cohere, DeepSeek | Gemini (Pro, Flash, Ultra), Claude via Model Garden | Both offer Claude; Bedrock has wider third-party model breadth; Gemini's long context window (up to 2M tokens) suits large-codebase use cases |
| Cost model | Per-token inference; Provisioned Throughput for predictable high-volume; AgentCore Runtime priced separately | Per-token inference; Provisioned Throughput; Vertex AI Search index pricing | Similar structure; negotiate enterprise agreements on both; total cost of ownership includes AgentCore primitives on Bedrock |
| Observability integration | CloudWatch, OpenTelemetry; AgentCore Observability for per-span agent traces | Cloud Monitoring, OpenTelemetry | Both are OTel-compatible; CloudWatch has deeper integration with Lambda and ECS workloads |
| Policy enforcement | AgentCore Policy (Cedar); deterministic, evaluated outside the model | Manual OPA/Rego implementation; native IAM for AWS resources only | Cedar on AgentCore enforces action boundaries without requiring custom policy infrastructure |
| Audit trail | S3 with Object Lock (WORM, configurable retention) | GCS with Bucket Lock (WORM) | Both provide immutable audit storage; S3 Object Lock has more granular retention modes |

## Decision Flowchart

The following flowchart covers the majority of workload routing decisions. Edge cases — air-gapped deployments, sovereignty constraints that neither cloud satisfies, on-premises requirements — are handled separately and documented in the AI System Register for the relevant agent.

```
Is the primary corpus Google Workspace (Drive, Docs, Meet)?
  |
  +-- Yes --> Consider Vertex AI Search as the retrieval layer.
  |           Orchestration spine can remain on Bedrock if the
  |           broader estate is AWS; retrieval is pluggable.
  |
  +-- No
       |
       Is the data subject to strict residency or sovereignty requirements?
       |
       +-- Yes --> Bedrock with PrivateLink (no cross-region egress).
       |           Verify the specific regulatory jurisdiction against
       |           Bedrock's compliance posture before committing.
       |
       +-- No
            |
            Is the use case tool-use-heavy (more than three external tools,
            or tools with IRREVERSIBLE risk class)?
            |
            +-- Yes --> Bedrock + AgentCore.
            |           AgentCore's tool registry, Cedar policy enforcement,
            |           and managed Gateway reduce the build cost significantly.
            |
            +-- No
                 |
                 Is the existing cloud footprint primarily AWS or GCP?
                 |
                 +-- AWS  --> Bedrock + AgentCore. Default for new workloads.
                 |
                 +-- GCP  --> Vertex AI. Reduces cross-cloud operational overhead.
                 |
                 +-- Both --> Route by workload shape:
                              knowledge Q&A on Vertex, action-taking on Bedrock.
```

## Running Both Platforms

Many production deployments use Bedrock for intake, triage, and action-taking agents — where Claude's instruction-following quality, AgentCore's tool lifecycle, and Cedar's policy enforcement provide the most value — and Vertex AI for knowledge Q&A and document search agents, where Vertex AI Search's grounding quality on enterprise corpora and Gemini's long context window provide the most value. This split is not a failure of platform selection; it is the correct outcome of routing workloads to the platform that fits them. The orchestration spine operates across both: the planner, router, worker, and tool executor are platform-agnostic at the pattern level, and the model invocation layer swaps between the Bedrock Converse API and the Vertex AI prediction API based on the per-agent routing policy in YAML configuration. What the enterprise operates is one governance framework, one eval harness, one audit trail architecture, and two model invocation backends — not two separate AI programmes.
