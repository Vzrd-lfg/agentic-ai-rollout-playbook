# Regulatory Mapping

This document maps the playbook's governance artefacts to the obligations imposed by the EU AI Act and four complementary frameworks: ISO 42001, NIST AI RMF, SR 11-7, and GDPR/DORA. The purpose is not to claim compliance — that determination belongs to the organisation's legal and compliance functions — but to make the mapping explicit so that a compliance officer, auditor, or regulator can trace each obligation to the specific artefact and process that addresses it.

## EU AI Act Alignment

The EU AI Act's obligation structure is tiered by risk. For an enterprise deploying agents, the practically significant tiers are Prohibited (Art. 5), High-Risk (Title III), and Limited-Risk (Art. 50). General-purpose AI model obligations (Title IIIA) sit primarily with model providers rather than deployers; the deployer's corresponding duty is to document model provenance and limitations in the AI System Register entry.

```
  EU AI Act
  ├── Art. 5 ──► Prohibited
  │              └── Block at Approval Gate intake
  ├── Title III ──► High-Risk
  │               ├── Conformity assessment ──► Approval Gate (8 checks)
  │               ├── Human oversight ──► HITL Policy
  │               └── Post-market monitoring ──► Online sampling + drift
  └── Art. 50 ──► Limited-Risk
                  └── Transparency obligation ──► AI disclosure in UI
```

| EU AI Act Obligation | Risk Tier | Playbook Artefact |
|---|---|---|
| Conformity assessment | High | Approval Gate (8 checks) |
| Technical documentation | High | AI System Register entry |
| Risk management system | High | Risk classification + continuous controls |
| Data governance | High | Data access check (Gate check 3) |
| Human oversight | High | HITL policy (Gate check 7) |
| Transparency to users | Limited | Disclosure requirement in agent UI |
| Accuracy, robustness | High | Eval harness (5 axes) + drift monitoring |
| Post-market monitoring | High | Online sampling + incident review |
| Incident reporting | High | Auditability + structured log |

The Approval Gate produces the evidence package that substantiates a conformity assessment for High-risk agents. The eight gate checks map directly to the documentation requirements of Art. 11 and the risk management requirements of Art. 9. An organisation seeking to demonstrate compliance to a notified body or national market surveillance authority should treat the gate review record — including the attached eval report, security review sign-off, and HITL policy — as the primary evidence artefact.

## ISO 42001 Mapping

ISO/IEC 42001 defines an AI Management System standard with a structure analogous to ISO 27001. Its clauses cover policy, leadership, planning, support, operations, evaluation, and improvement. The playbook addresses the operationally material clauses; the remaining clauses (leadership commitment, communication, document control) are organisation-level commitments outside the scope of a technical architecture playbook.

| ISO 42001 Clause | Playbook Artefact |
|---|---|
| 6.1 — AI risk assessment | Risk classification + Approval Gate |
| 8.4 — AI system lifecycle | 90-day rollout plan + operating model |
| 9.1 — Monitoring and measurement | Eval harness + observability |
| 10.2 — Incident management | Auditability + rollback plan |

Clause 6.1 requires that AI risks be identified, assessed, and treated. The risk classification system provides the identification and assessment; the Approval Gate and continuous controls provide the treatment. Clause 8.4 requires that AI system lifecycle processes be planned and controlled; the 90-day rollout plan is the planning artefact and the Approval Gate is the control point at transition from development to production.

## NIST AI RMF Mapping

The NIST AI Risk Management Framework organises AI risk management into four functions: Govern, Map, Measure, and Manage. The framework is voluntary but is referenced by US federal agencies and increasingly adopted by regulated financial institutions as a complement to sector-specific requirements.

| NIST AI RMF Function | Playbook Artefact |
|---|---|
| GOVERN | AI governance framework + AI System Register |
| MAP | Risk classification + use case mapping (rollout/90-day-plan) |
| MEASURE | Eval harness + online sampling + token efficiency |
| MANAGE | Approval Gate + HITL policy + rollback plan |

The MEASURE function is particularly well-supported by this architecture: the eval harness provides structured measurement against defined thresholds, online sampling extends measurement into production, and the AI System Register records measurement results alongside each agent's approval record. The GOVERN function depends on organisational commitments — named roles, board-level sponsorship, policy ownership — that the AI Governance Framework document defines but that require human execution to be real controls.

## SR 11-7 (Model Risk Management)

SR 11-7 applies to financial institutions supervised by the Federal Reserve that use models for decision-making. Its three-pillar model risk management framework — development and implementation, validation, and governance and control — maps directly to this playbook's structure. The eval harness addresses development and validation: it establishes baseline performance, defines acceptance thresholds, and provides independent measurement of the model's behaviour against those thresholds. The Approval Gate addresses governance: it is the formal control point at which an agent is reviewed before production, with documented evidence and named sign-offs. The continuous controls address ongoing monitoring: drift detection, online sampling, and annual recertification correspond to SR 11-7's expectation of ongoing model performance review. Financial institutions using this playbook should verify with their model risk management function that the specific agents in scope meet the SR 11-7 definition of a "model" — the definition is broader than many practitioners expect and includes systems that apply statistical or computational methods to inform consequential decisions.

## GDPR and DORA

**GDPR.** The data access check at Gate check 3 requires a data processing agreement for any personal data the agent processes. Where the agent processes personal data at scale or for automated decision-making, a data protection impact assessment under Art. 35 is completed before the gate review and retained as part of the gate record. PII is hashed in audit logs; the mapping from hash to identity is maintained in a separately access-controlled store. The agent's data minimisation is verified at the gate: it retrieves only the data necessary for the task, not the full dataset from which that data is drawn. Art. 22, which grants data subjects the right not to be subject to solely automated decisions with significant effects, applies to High-risk agents in scope; the HITL requirement at Gate check 7 satisfies the right to human review by ensuring a human decision-maker is in the loop for consequential automated decisions.

**DORA.** The Digital Operational Resilience Act applies to financial entities operating in the EU and imposes requirements for ICT risk management, incident management, operational resilience testing, and third-party ICT risk. The rollback plan at Gate check 8 addresses DORA's ICT business continuity requirements: it establishes a tested procedure for disabling a failing agent without service disruption. The incident review process — triggered by any production anomaly — addresses DORA's ICT incident management requirements, including the classification, reporting, and post-incident analysis obligations. The immutable audit trail satisfies DORA's record-keeping requirements for ICT incidents. Where the model provider is a critical ICT third-party service provider under DORA, the organisation's third-party risk management process must cover the model provider as such; the approved model list and model provider security assessment in the governance framework provide the required due diligence artefacts.
