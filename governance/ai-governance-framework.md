# AI Governance Framework

Governance is not a compliance formality — it is the operational mechanism that allows an enterprise to deploy agents at scale without losing control. Without it, risk classification is a spreadsheet exercise, approval decisions are informal, and production incidents have no structured response path. This framework comprises four components: a risk classification system, an eight-check Approval Gate, an AI System Register, and continuous operational controls. Together they form the governance spine that every agent in production passes through and remains accountable to.

## Risk Classification

Every agent is classified before it enters the Approval Gate. Classification determines which controls apply, which reviews are mandatory, and how the agent is monitored in production. The four risk classes below align to EU AI Act categories and extend them with internal operational definitions.

| Risk Class | Definition | Examples | Default Control Level |
|---|---|---|---|
| Prohibited | Capability that must not be deployed under any circumstances | Social scoring, real-time biometric surveillance of staff | Block at Approval Gate |
| High-Risk | Automated decisions with significant impact on individuals | Credit scoring, recruitment screening, benefits determination | Mandatory HITL + quarterly audit |
| Limited-Risk | Agent interacts with humans; transparency required | Chatbots, content recommendation, summarisation | Disclose AI involvement to end user |
| Minimal-Risk | No interaction with individuals; internal workflow only | Document classification, log analysis, code review tools | Standard Approval Gate |

Classification is assigned by the AI Governance Lead at intake, recorded in the AI System Register, and reviewed annually or on any material change to the agent's scope, model, or data access. Downward reclassification (e.g., High to Limited) requires written justification and a second sign-off. Classification creep — where agents accumulate scope without re-classification — is the most common governance failure mode; the quarterly register audit exists to catch it.

## The Eight-Check Approval Gate

No agent reaches production without passing all eight checks. The gate is a structured review, not a rubber stamp. The AI Governance Lead convenes the review, collects evidence against each check, and records the outcome. A check may not be waived; it may be deferred only if a remediation plan with a named owner and deadline is in place. Any single failed check returns the agent to design.

```
  ┌─────────────────────────────────────────────┐
  │           APPROVAL GATE                      │
  │                                             │
  │  1. Business Case ──► does value justify    │
  │  2. Risk Class    ──► classify & document   │
  │  3. Data Access   ──► data inventory + DPA  │
  │  4. Model Approval ──► model on approved list│
  │  5. Eval Baseline ──► pass threshold set    │
  │  6. Security Review ──► pen test if High    │
  │  7. HITL Policy   ──► HITL rules defined    │
  │  8. Rollback Plan ──► disable path tested   │
  │                                             │
  │  ALL 8 CHECKS PASS ──► Deploy to Production │
  │  ANY CHECK FAILS ──► Return to Design       │
  └─────────────────────────────────────────────┘
```

**Business Case.** The agent's purpose, the metric it improves, the baseline it is measured against, and the owner accountable for outcomes must be documented before any technical work begins. A use case without a named business owner does not proceed. This check prevents the accumulation of agents that no one is accountable for.

**Risk Classification.** The agent is classified against the four risk classes using the criteria above. The classification is signed by the AI Governance Lead. High-risk agents require a separate impact assessment that documents the population affected, the nature of the automated decision, and the safeguards in place. The impact assessment is retained in the AI System Register.

**Data Access.** An inventory of every data source the agent reads from or writes to is required. A data processing agreement is in place for any personal data. Data residency requirements are met. For High-risk agents processing personal data, a data protection impact assessment is completed under Art. 35 GDPR before the gate review. Data minimisation is verified: the agent is granted access only to the specific records it needs, not to the table or system that contains them.

**Model Approval.** The model or models the agent will invoke are on the organisation's approved model list. The list is maintained by the AI Governance Lead and reviewed quarterly. It records the model provider, the model version, the contractual terms governing its use, and the jurisdictions in which it may be used. An agent may not invoke a model that is not on the list; adding a model to the list is itself a governed process.

**Eval Baseline.** The agent's pass/fail thresholds for accuracy, faithfulness, safety, and cost are defined before the first eval run. The agent must meet those thresholds on the initial run against the designated evaluation dataset. Thresholds are not set retrospectively to match results. The eval report, including the dataset version and threshold values, is retained and attached to the AI System Register entry.

**Security Review.** The agent's architecture has been reviewed for the OWASP LLM Top-10 threats. The review covers tool input/output schemas, IAM scopes, secrets handling, network egress paths, and prompt injection surface. High-risk agents undergo a penetration test conducted or verified by the information security function. The security review sign-off is a prerequisite for the gate passing; it may not be performed by the team that built the agent.

**HITL Policy.** The conditions under which the agent escalates to a human are explicitly defined. This includes: which tool risk classes trigger a HITL gate, which user roles may act as approvers, the timeout period after which the gate expires, and the default action taken on timeout (approve, reject, or abort). Every HITL gate must have a named default action; a gate that blocks indefinitely on timeout is a production reliability risk.

**Rollback Plan.** A tested mechanism exists to disable the agent without disrupting downstream systems. The rollback procedure is documented in the agent's runbook, the person responsible for executing it is named, and the procedure has been tested in a pre-production environment. The rollback test result is attached to the gate review record. An agent whose rollback has not been tested does not pass this check.

## AI System Register

The AI System Register is the authoritative inventory of every agent operating in production. It is queryable, version-controlled, and updated on every material change to an agent. An agent not in the register should not be in production; the register audit process identifies and remediates discrepancies. The minimum record structure for each entry is defined by the following schema.

```yaml
agent_register_entry:
  agent_id: string
  name: string
  version: string
  owner: string              # named individual, not a team
  purpose: string
  risk_class: enum           # Prohibited|High|Limited|Minimal
  models_used: [string]
  data_sources: [string]
  tools: [string]
  hitl_policy_ref: string    # link to HITL policy document
  eval_status:
    last_run: date
    accuracy: float
    faithfulness: float
    safety: float
  approval_date: date
  next_review_date: date
  rollback_owner: string
```

The `owner` field requires a named individual. A team name does not satisfy this field because teams cannot be held accountable; individuals can. The `next_review_date` is set at approval time based on the agent's risk class: High-risk agents are reviewed quarterly, Limited-risk agents semi-annually, and Minimal-risk agents annually.

## Continuous Controls

The Approval Gate is a point-in-time control. Continuous controls maintain governance assurance across the agent's operational life. The table below defines the minimum set of ongoing controls, their frequency, and the trigger that initiates each.

| Control | Frequency | Trigger |
|---|---|---|
| Eval run | Every deployment | Any change to model, prompt, or tool |
| Online sampling | Continuous (1–5% of traffic) | Production traffic |
| Drift monitoring | Weekly | Automated; alert if faithfulness drops >5pp |
| Annual recertification | Annual | Calendar |
| Incident review | Per incident | Any production anomaly |

Eval runs are automated and block deployment if thresholds are not met. Online sampling results feed into the weekly drift report reviewed by the AI Governance Lead. Drift alerts page the agent owner; if faithfulness has dropped more than five percentage points from the approved baseline, the agent is flagged for immediate review and may be taken offline pending investigation. Annual recertification re-runs the full Approval Gate process: risk class is re-confirmed, the eval baseline is updated, and the business owner re-affirms accountability. Retention periods for all continuous control records align to the agent's risk class and applicable regulatory retention requirements.
