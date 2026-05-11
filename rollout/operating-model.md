# Operating Model

The operating model defines who does what, how decisions are made, and how the programme is funded. Without it, the programme depends on heroic individuals rather than durable institutions. The most common failure pattern is a programme that ships three agents and then stalls because the Programme Owner is simultaneously the platform engineer, the governance reviewer, and the change manager. The operating model prevents that.

## Four Roles

### AI Programme Owner

Accountable for the programme's outcomes: use case portfolio, business value realisation, and stakeholder relationships. Not necessarily technical. Reports to a C-level sponsor. Chairs the AI Council. Signs off on the business case component of the Approval Gate. The Programme Owner is the role this playbook is written for: the person who must simultaneously manage upward (executive mandate), outward (business unit demand), and across (platform and governance functions). Their measure of success is not agents shipped — it is adoption outcomes sustained.

### AI Platform Lead

Accountable for the orchestration spine, shared infrastructure, and developer experience. Technical. Owns the reference architecture, the model abstraction layer, the eval harness, and the observability stack. Signs off on the architecture and security components of the Approval Gate. The Platform Lead's operating principle is that the spine is a shared good: every decision made in the platform either enables or constrains every agent built on it. Platform standards are not bureaucracy; they are the mechanism that makes the self-service path possible.

### BU AI Owner

Accountable for a business unit's use cases. Bridges between the BU's domain knowledge and the platform. Owns the use case backlog for their BU, the adoption metrics for their agents, and the BU-level HITL policy. Typically a senior operator, not a technical role. The BU AI Owner must belong to their BU — that is how adoption sticks. The programme owns the platform, the gate, and the portfolio view. The BU owns the agent, the users, and the outcome. A BU AI Owner who reports into the central platform team rather than into their BU will fail to generate the trust within the BU that adoption requires.

### AI Governance Lead

Accountable for the Approval Gate process, the AI System Register, regulatory alignment, and the risk classification framework. Owns the governance documentation. Signs off on the compliance, data, and HITL components of the Approval Gate. Reports to the General Counsel or Chief Risk Officer. The Governance Lead has the authority to block a production launch on compliance grounds, with appeal to the Steering Committee. This authority must be real: a governance function that can be overridden at will is not governance.

## Three Committees

### AI Council / Steering Committee

Meets monthly at steady state; fortnightly during initial build-out. Reviews: programme metrics (use cases in production, adoption rates, cost trends), the use case pipeline, budget allocation, and strategic priorities. Chaired by the AI Programme Owner. Members include the C-level sponsor, AI Platform Lead, AI Governance Lead, and BU AI Owners. The standing agenda covers portfolio status, decisions requiring committee authority, and any open risks or incidents. The AI Council is not a status meeting — it is the decision-making body for prioritisation, gate advancement, and material governance matters.

### Architecture Review Board

Meets bi-weekly. Reviews: proposed agent architectures, model additions to the approved model list, changes to the orchestration spine, and infrastructure changes. Chaired by the AI Platform Lead. Quorum requires three technical members. The ARB's remit is not to approve every agent — Tier-3 (low-risk) agents do not require ARB review. It exists to review architectural exceptions, dual-cloud decisions, and any system that deviates from the reference implementation. Deviations that are not reviewed become technical debt that the platform team inherits.

### Approval Gate Panel

Convened per agent deployment. Reviews the eight-check Approval Gate for the specific agent being submitted for production. Members: AI Programme Owner (business case), AI Platform Lead (architecture), AI Governance Lead (compliance). All three must sign off. The panel does not negotiate gate criteria — it verifies them. If a criterion is not met, the agent returns to the build phase with a documented gap and a target date.

## RACI Matrix

| Decision | Programme Owner | Platform Lead | BU AI Owner | Governance Lead |
|---|---|---|---|---|
| Use case prioritisation | A | C | R | C |
| Architecture standards | C | A | I | C |
| Model approval | C | R | I | A |
| Approval Gate pass/fail | A | R | I | R |
| Risk classification | C | C | I | A |
| Budget allocation | A | C | C | C |
| BU AI Owner appointment | A | I | R | C |
| Eval threshold setting | C | R | C | A |
| Incident response | A | R | C | R |
| Regulatory reporting | C | I | I | A |

R = Responsible, A = Accountable, C = Consulted, I = Informed

## Funding Models

**Centralised Platform + BU-Funded Agents.** The platform — spine, infrastructure, observability — is centrally funded. Individual agents are funded by the business unit that benefits. This is the cleanest alignment of cost and value: the BU that owns the outcome also owns the budget. It requires BU budgets to be flexible enough to absorb agent development costs, which may require early conversation with finance. This is the recommended model from Scale onwards.

**Fully Centralised.** Platform and agents are both centrally funded. Appropriate in the first 90 days when BU budgets have not been adjusted and the programme is establishing proof of concept. Does not scale beyond a small portfolio because the central team becomes the bottleneck for every funding decision as well as every build decision.

**Chargeback.** Platform is centrally funded; BUs are charged per agent invocation or per token consumed. Creates cost visibility and incentivises efficiency in prompt design and model selection. Requires cost telemetry infrastructure before it can be implemented — the per-agent cost breakdown in the observability stack is the prerequisite. Appropriate for larger organisations with mature FinOps practices.

**AI Innovation Fund.** A time-limited fund, typically 12 to 18 months, that co-invests with BUs by covering 50 to 80 percent of use case development costs to de-risk early adoption. The fund sunsets; BUs are expected to fund production operations themselves thereafter. Useful when the executive sponsor wants tight ROI accountability on early investments and when BU budgets are not yet structured to absorb agentic AI costs.
