# Change Management

Agentic AI does not fail because of the technology. It fails because the humans who are supposed to use it do not trust it, do not understand it, or have not been given a role in shaping it. Change management is the practice of managing that transition deliberately. Most programmes underspend on this dimension by an order of magnitude, treating it as a communications task rather than as the primary adoption mechanism. The result is technically sound agents that knowledge workers route around, and programmes that report agent deployments rather than agent adoption.

## Three Framings

### Augmentation

The agent handles the boring 70% — the high-volume, low-complexity, low-judgment tasks that currently consume disproportionate time. The operator focuses on the interesting 30% that actually requires their expertise. This framing works with knowledge workers who understand that their value is in their judgment, not in processing volume. It is the most broadly applicable framing and the least likely to generate resistance. Its limitation is that it loses credibility if the programme subsequently needs to reduce headcount; operators will remember the framing and the gap will cost trust.

### Capacity

The same team can handle three times the volume without hiring. The agent scales the team's capacity without replacing headcount. This framing works in growth contexts where demand is outpacing headcount and the business case is clear. It is the most honest framing when the actual driver is growth, and it lands well with operators who understand their team is under pressure. It is harder to sustain in flat or shrinking businesses where the capacity expansion has no obvious destination.

### Cost

The cost per transaction falls as the agent handles more of the workflow. This framing works in operational contexts where the finance case is the primary driver. Caution: leading with cost in a public-facing way triggers job security anxiety among the teams affected. Use this framing internally with finance stakeholders; use the augmentation or capacity framing with the teams doing the work. The framing must be selected per agent based on the actual business context. Drifting between framings across different conversations about the same agent erodes credibility faster than any technical failure.

## Five-Step Launch Sequence

1. **Co-design.** Involve the team that will use the agent in its design. They know the edge cases. They will find the gaps that the builders did not anticipate. Co-design generates ownership. An agent that the team helped design is "our agent", not "a system imposed on us." Two to three workshops with operators before the design is locked is the minimum. Skipping this step is the single most common cause of adoption failure.

2. **Pilot volunteers.** Recruit three to five enthusiastic early adopters from the target team. Run the agent with them for two weeks. Gather feedback. Fix the most obvious problems. The volunteers become internal advocates. Their endorsement carries more weight with their colleagues than any communication from the programme team. The pilot generates the testimonial that the broader rollout rides on.

3. **Phased rollout.** Expand from volunteers to the full team in two or three cohorts. Each cohort receives a dedicated onboarding session — the half-day operator workshop from the enablement playbook. Do not roll out to the full team at once; the ability to observe, learn, and adjust between cohorts is worth the additional time. The go/no-go between cohorts is driven by adoption metrics, not by the calendar.

4. **Visible feedback loop.** Make it easy for users to flag errors and wrong answers. A simple thumbs-down in the UI that routes to a tracked queue is sufficient. Close the loop visibly and publicly: "We fixed the issue you reported on Tuesday." Users who see their feedback acted on trust the system more. Users who flag problems and hear nothing stop flagging — and stop trusting. Silence on flagged issues is the fastest mechanism for destroying adoption.

5. **Celebrate the humans.** When the agent saves the team time, make the win visible — and attribute it to the team, not the technology. "The claims team processed three times more cases this month" is a team win; the agent is the tool they used. Internal communications that celebrate the agent rather than the team create the impression that the technology is the protagonist and the operators are incidental. That impression is both inaccurate and corrosive.

## Hard Conversations

| Concern | Honest Answer |
|---|---|
| "Will this replace my job?" | The agent replaces tasks, not roles. The tasks it takes are the ones you find least interesting. The judgment calls it cannot make are the ones that define your expertise. |
| "What if it gives wrong answers?" | It will give wrong answers. That is why the HITL gate exists and why we are measuring faithfulness continuously. When it makes a mistake, we catch it, log it, and fix it. |
| "I don't trust it." | Trust is earned by evidence. Run it on low-stakes cases first. Review its outputs. When you see it getting things right consistently, extend more trust — based on data, not assumption. |
| "This is too much change at once." | Phased rollout exists precisely for this. You will not be asked to change everything at once. |

The common failure in hard conversations is deflection. Operators who receive non-answers conclude — correctly — that the programme team either does not know or will not say. The cost of a direct honest answer, even an uncomfortable one, is significantly lower than the cost of the trust deficit that deflection produces.

## Adoption Health Metrics

| Metric | Target (30 days post-launch) |
|---|---|
| Task completion rate | ≥80% of agent-initiated tasks completed without user override |
| Escalation rate | ≤20% of cases escalated to human (if HITL is optional) |
| Feedback submission rate | ≥10% of sessions with at least one feedback signal |
| Net Promoter Score (internal) | ≥+20 among the first cohort |

These metrics are measured per agent, not across the programme. An aggregate adoption rate that looks healthy can conceal a single agent with a failing adoption trajectory. The Programme Owner reviews per-agent adoption metrics monthly; the BU AI Owner reviews them weekly. When metrics are below target at the 30-day mark, the response is investigation before intervention: understand why before prescribing a fix.
