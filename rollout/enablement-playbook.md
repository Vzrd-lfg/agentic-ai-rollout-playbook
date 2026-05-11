# Enablement Playbook

Adoption is the product. An agent that exists but is not used has zero value. The enablement playbook defines what each population needs to succeed with agentic AI and how the programme delivers it. The most common failure is treating enablement as a single event — a training day, a launch email, a town hall — rather than as a continuous operating function. Enablement works when it is specific to a population, sustained over time, and measured by outcomes rather than attendance.

## Three Populations

### Executives

| Item | Detail |
|---|---|
| Need | Understand the strategic opportunity and the risk; be able to ask the right questions |
| Curriculum | What agentic AI is; where it works and where it fails; what the programme requires from them (mandate, budget, governance participation) |
| Format | Briefing (live, 60 minutes) or self-paced (video + one-pager) |
| Duration | 60 minutes |
| Artefact | Executive brief: "What your AI programme needs from you" |

The executive briefing is not the platform architecture deck with the technical content removed. It addresses three questions: what are we doing, why does it matter to this organisation's specific situation, and what do we need from you. The third question is the one most commonly omitted. Executives who understand what the programme needs from them — mandate clarity, budget flexibility, willingness to participate in the AI Council — are significantly more effective sponsors than those who receive only status updates.

### Operators (Domain Users)

| Item | Detail |
|---|---|
| Need | Understand what the agent does and does not do; know when to trust it and when to escalate |
| Curriculum | How the agent works at a functional level; trust calibration (what it gets right, what it misses); how to escalate; how to give feedback |
| Format | Half-day workshop (live, hands-on with the agent) |
| Duration | 4 hours |
| Artefact | Agent user guide (what it does, known limitations, escalation path) |

The operator workshop is delivered per agent, not as a generic AI session. It must include hands-on time with the specific agent the operator will use, including deliberate exposure to the cases where the agent is known to be unreliable. Operators who have seen the failure modes before encountering them in production are more likely to escalate appropriately and less likely to either over-trust or reject the agent entirely.

### Technical Builders (BU Developers, Platform Engineers)

| Item | Detail |
|---|---|
| Need | Be able to build and operate agents on the spine without platform team support |
| Curriculum | Spine architecture; orchestration patterns; eval harness; governance process; common failure modes |
| Format | Two-day bootcamp (live) + self-paced reference materials |
| Duration | 2 days + ongoing |
| Artefact | agent-template repo; prompt pattern library; Approval Gate checklist |

The two-day bootcamp is the gateway to the self-service path. Graduates can build a Tier-3 (low-risk) agent with the platform team in advisory mode. They cannot submit a Tier-1 or Tier-2 agent for the Approval Gate without an architecture review. Day 1 covers the platform fundamentals and the reference implementation. Day 2 covers production-readiness: eval harness, observability instrumentation, and the Approval Gate checklist. The hands-on lab on Day 2 takes the agent built on Day 1 from a working prototype to a system that can pass the platform's pre-production gate.

## Artefact Library

| Artefact | Audience | Purpose |
|---|---|---|
| agent-template repo | Technical builders | Scaffold for a new agent on the spine |
| Use case intake template | BU AI Owners | Standardised form for requesting a new agent |
| Approval Gate checklist | All | Eight-check checklist in Google Sheets / Confluence |
| Prompt pattern reference | Technical builders | Common system prompt structures (not the prompts themselves) |
| Case study compendium | All | 15 documented rollouts with lessons learned |
| FAQ (technical) | Technical builders | Common integration and debugging questions |
| FAQ (governance) | BU AI Owners, Executives | Common compliance and approval questions |

All artefacts are versioned and kept current. A stale artefact is worse than no artefact: builders who follow outdated guidance and produce non-compliant agents create rework for the platform team and delay the Approval Gate. The Platform Lead owns the technical artefacts; the Governance Lead owns the compliance artefacts; the Programme Owner owns the business-facing artefacts.

## Communities of Practice

**Prompt Engineering CoP.** Monthly. Peer-led. Practitioners share what is working and what is not in system prompt design across different use cases and model versions. No platform team facilitation is required after the first three sessions. The platform team attends but does not chair. The signal that this community is working is practitioners referencing each other's patterns in the artefact library without being prompted.

**Evaluation Design CoP.** Monthly. Peer-led. Practitioners share golden set construction approaches, judge prompt patterns, and lessons from eval failures. The Evaluation Design CoP is the mechanism by which the programme's eval quality improves faster than any individual team could achieve in isolation. A BU that has struggled with faithfulness scoring in a particular domain shares what they learned; the next BU starting a similar use case benefits.

**RAG Practitioners CoP.** Monthly. Peer-led. Focused on retrieval quality: chunking strategies, re-ranking experiments, corpus maintenance, and the practical differences between dense-vector, sparse, and graph retrieval across different knowledge corpus structures. RAG quality is the single most common source of faithfulness failures in production; this community is the ongoing response to that fact.

## Office Hours

| Session | Cadence | Facilitator | Audience |
|---|---|---|---|
| Technical Office Hours | Weekly | AI Platform Lead or delegate | Technical builders with implementation questions |
| Governance Office Hours | Fortnightly | AI Governance Lead | BU AI Owners preparing for Approval Gate |
| ROI / Value Office Hours | Monthly | AI Programme Owner | BU AI Owners with adoption or value questions |

Office hours are drop-in, unscheduled, and recorded for internal access. The absence of agenda is deliberate: the questions that matter most are the ones practitioners did not know they had until they hit the problem. Declining attendance at office hours is a signal worth investigating — it may mean the enablement is working, or it may mean practitioners have stopped asking because they do not expect useful answers.
