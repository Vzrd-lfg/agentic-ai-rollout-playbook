# Disclaimer

This repository publishes the architectural patterns, frameworks, taxonomies, and operating discipline behind production agentic AI deployments. It does not publish the operational depth that differentiates live programmes. The distinction is deliberate: the conceptual material is sufficient to save four to six weeks of design iteration; the implementation is the consulting engagement.

## What Is Published

- Orchestration patterns: Planner, Router, Worker, Tool Executor, multi-agent topology guidance
- Ten RAG patterns with a selection decision tree
- Four-component API gateway pattern and three-identity model for agent authentication
- Workflow engine integration patterns (Step Functions, Temporal, BPMN)
- AI governance framework: eight-check Approval Gate, AI System Register schema, risk classification table, continuous controls
- Three-layer access control model (user-to-agent, agent-to-tool, tool-to-system)
- Immutable audit trail design, structured log schema, retention tiers
- Five-stage agent security posture and OWASP LLM Top-10 mapping
- EU AI Act, ISO 42001, NIST AI RMF, SR 11-7, GDPR, and DORA regulatory mapping
- Evaluation harness design: five axes, four surfaces, contract-based pass/fail
- Evaluation suite scaffold: golden-set schema, judge rubric, eval runner pseudo-code
- Token efficiency framework: eight cost levers ranked by impact
- Five failure mode categories with production signals, leading indicators, and remediations
- 90-day rollout plan with phase objectives and gate criteria
- Operating model: four roles, three committees, RACI matrix, funding model options
- Enablement playbook: three populations, three curricula, artefact library, communities of practice
- Change management: three framings, five-step launch sequence, adoption health metrics
- Fifteen case study summaries with outcome metrics and transferable lessons
- Infrastructure pattern documentation: account topology, network boundaries, nine IaC module shapes (described, not implemented)
- Illustrative pseudo-code examples for Bedrock, AgentCore, and Vertex AI

## What Is Not Published

- Production system prompts or system message content
- Curated golden test sets or ground-truth answer corpora
- Tuned judge rubrics with proprietary scoring criteria
- Cedar policy packs or OPA/Rego rule libraries
- Terraform, CDK, or Pulumi IaC modules
- Grafana dashboards, alert rule sets, or runbooks
- Change-management training decks or facilitation guides for specific team programmes
- Incident runbooks derived from specific client engagements
- Client-specific architectures, data models, or configurations

## On the Examples

The three files in `examples/` — `bedrock_claude_agent.py`, `agentcore_strands_agent.py`, and `vertex_gemini_agent.py` — are illustrative pseudo-code. They demonstrate structural and architectural intent only. They cannot be executed as written, contain no real API credentials or endpoint configurations, and are not intended as production templates. Each file is clearly marked `NOT_PRODUCTION — DEMONSTRATION ONLY` at the top.

## Legal

This repository is provided "as is" without warranty of any kind, express or implied. The patterns and frameworks are derived from the author's experience across multiple production deployments; they are not representations of any specific client's systems, and specific client data, proprietary architectures, and confidential operational details are excluded. The author accepts no liability for decisions made on the basis of this material.

Licensed under [CC BY-NC 4.0](LICENSE). Non-commercial use with attribution is permitted. Commercial use — including use within a commercial consulting engagement, as the basis for a product, or as a deliverable to a client — requires written permission from the author. Contact: vishnu.sathiap@gmail.com.
