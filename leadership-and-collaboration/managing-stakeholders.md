# Managing Stakeholders

> "Stakeholder management is not about keeping everyone happy. It's about ensuring the right people have the right information at the right time to make good decisions."

## Who Are Your Stakeholders?

In a banking GenAI platform, your stakeholders include anyone affected by or interested in your work:

```
                    ┌──────────────────────┐
                    │      Executives      │  Strategic alignment, budget, risk
                    ├──────────────────────┤
                    │  Product Management  │  Features, priorities, users
                    ├──────────────────────┤
     ┌──────────────┤    Engineering       ├──────────────┐
     │              │  (Your Team + Others)│              │
     │              └──────────────────────┘              │
     ▼                                                    ▼
┌─────────┐                                          ┌─────────┐
│Security │                                          │Compliance│
│& Risk   │                                          │& Legal  │
└─────────┘                                          └─────────┘
     │                                                    │
     ▼                                                    ▼
┌─────────┐                                          ┌─────────┐
│Platform │                                          │  End     │
│& Infra  │                                          │  Users   │
└─────────┘                                          └─────────┘
```

## Stakeholder Mapping

For any significant initiative, create a stakeholder map:

```markdown
# Stakeholder Map: [Initiative Name]

| Stakeholder | Interest | Influence | Current Stance | Engagement Strategy |
|-------------|----------|-----------|----------------|---------------------|
| CTO | Strategic alignment with AI strategy | High | Supportive | Monthly executive summary |
| Head of Compliance | Regulatory risk | High | Neutral → Skeptical | Early engagement, evidence-based updates |
| Security Lead | Vulnerability exposure | High | Supportive (with conditions) | Bi-weekly technical sync |
| Product Manager | Feature delivery timeline | Medium | Supportive | Weekly status update |
| Platform Team | Infrastructure impact | Medium | Neutral | RFC + design review |
| End Users (employees) | Quality and reliability | Low (individually) | Unknown | User testing, feedback channels |
```

### The Power-Interest Grid

```
                    Interest in Your Work
                    Low              High
              ┌────────────────────────────────────┐
        High  │  KEEP SATISFIED    │  MANAGE CLOSELY │
              │  • Regulators      │  • CTO          │
    Power     │  • Audit teams     │  • Security     │
              │                    │  • Compliance   │
              ├────────────────────┼─────────────────┤
        Low   │  MONITOR           │  KEEP INFORMED  │
              │  • Other teams     │  • End users    │
              │  • Vendors         │  • Support team │
              └────────────────────────────────────┘
```

## Communication Plans

### For Each Stakeholder Group, Define

| Element | Description | Example |
|---------|-------------|---------|
| **What** | What information they need | Progress, risks, decisions, metrics |
| **How** | Channel for communication | Email, Slack, meeting, dashboard |
| **When** | Frequency and timing | Weekly Monday 10 AM, monthly summary |
| **Who** | Who sends the communication | Tech lead, engineering manager, author |
| **Feedback** | How they provide input | Reply email, office hours, survey |

### Communication Plan Template

```markdown
# Communication Plan: [Initiative Name]

## Stakeholder Communications

### Executive Sponsors (CTO, VP Engineering)
- **What:** Monthly progress summary with risks and decisions needed
- **How:** 1-page written summary + 15-min call
- **When:** First Tuesday of each month
- **Who:** Engineering manager
- **Feedback:** Email response or ad-hoc call

### Security Team
- **What:** Design docs, security review findings, remediation progress
- **How:** Design doc review + bi-weekly 30-min sync
- **When:** Bi-weekly, Thursdays
- **Who:** Tech lead
- **Feedback:** Direct comments on design docs

### Compliance Team
- **What:** Compliance review findings, remediation status, regulatory updates
- **How:** Written report + monthly review meeting
- **When:** Monthly, last Wednesday
- **Who:** Engineering manager + compliance liaison
- **Feedback:** Compliance sign-off process

### Product Team
- **What:** Sprint progress, blockers, scope changes
- **How:** Slack update + weekly 15-min sync
- **When:** Weekly, Mondays
- **Who:** Tech lead
- **Feedback:** Slack thread, ad-hoc discussion

### End Users
- **What:** Feature announcements, known issues, tips
- **How:** Internal newsletter + Slack announcement
- **When:** At each feature release
- **Who:** Product manager (engineering provides technical content)
- **Feedback:** User feedback channel, quarterly survey
```

## Banking-Specific Stakeholder Challenges

### The Compliance Stakeholder

Compliance teams are not adversaries. They are protecting the bank from regulatory risk. But their incentives differ from engineering's:

| Engineering Priority | Compliance Priority | Tension |
|---------------------|---------------------|---------|
| Ship features quickly | Ensure every regulation is met | Speed vs. thoroughness |
| Technical elegance | Audit evidence | Different definitions of "done" |
| Experiment and iterate | Prove safety before experimentation | Innovation vs. caution |

**How to work effectively with compliance:**

1. **Understand their framework.** Learn the regulations that apply to your work. Don't make them teach you from scratch.
2. **Provide evidence, not assurances.** "We tested it thoroughly" means nothing. "Here are our test results, audit logs, and access control design" means everything.
3. **Engage at design time.** A compliance concern caught in a design doc costs hours. Caught in production, it costs months.
4. **Respect their timeline.** Compliance reviews take time because regulatory risk is real. Plan for it.

### The Security Stakeholder

Security teams share compliance's caution but focus on different risks:

| Security Concern | How to Address |
|-----------------|----------------|
| "Your GenAI system could expose customer data" | Show PII redaction, access controls, audit logging |
| "External LLM APIs are a data exfiltration risk" | Show data classification, prompt sanitization, contractual protections |
| "Your API has no rate limiting" | Implement and demonstrate rate limiting |
| "Your dependencies have known CVEs" | Show dependency scanning and patching process |

### The Executive Stakeholder

Executives need different information than engineers:

| What Executives Want | What They Don't Need |
|---------------------|---------------------|
| Progress toward strategic goals | Technical implementation details |
| Risks and mitigation plans | Blame for problems |
| Resource needs and trade-offs | Complaints without solutions |
| Competitive/market context | Jargon without explanation |
| Clear asks and decisions | Open-ended discussions |

See `communicating-with-executives.md` for detailed guidance.

## Stakeholder Engagement Strategies

### For Supportive Stakeholders

- Keep them informed and engaged
- Ask them to advocate for your initiative
- Leverage their influence with skeptical stakeholders
- Don't take their support for granted

### For Neutral Stakeholders

- Educate them on why the initiative matters to their area
- Involve them early in design and planning
- Address their questions proactively
- Convert them to supporters through transparency and results

### For Skeptical Stakeholders

- Understand their concerns (schedule a 1:1 to listen)
- Acknowledge valid concerns openly
- Address concerns with evidence, not persuasion
- Find areas of agreement and build from there
- If they remain skeptical, document their concerns and how you're addressing them

### For Blocking Stakeholders

- Understand WHY they're blocking (it's always about something they're responsible for)
- Find the minimal condition that would remove the block
- Escalate only after genuine attempts to resolve
- Document the block and your resolution attempts

## Stakeholder Management Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact |
|--------------|-------------------|--------|
| **Surprise stakeholder** | Someone you didn't identify as a stakeholder blocks your launch | Delayed launch, damaged relationships |
| **Over-communication** | CC'ing 20 people on every update | Ignored communications, noise |
| **Under-communication** | Not telling stakeholders about problems until it's too late | Broken trust, escalation |
| **One-size-fits-all** | Same status update for engineers and executives | Nobody gets what they need |
| **Managing up only** | Only considering senior stakeholders | Missing ground-level supporters |
| **Set-and-forget** | Stakeholder map created once, never updated | Stakeholders change, your map should too |

## Stakeholder Updates: What to Include

Every stakeholder update should answer:

1. **What happened since the last update?** (Progress)
2. **What's happening next?** (Plan)
3. **What are the risks?** (Awareness)
4. **What decisions or actions are needed from the stakeholder?** (Ask)
5. **What are the key metrics?** (Evidence)

```markdown
# Weekly Update: GenAI Chat Platform — Week of April 6, 2026

## Progress This Week
✅ Shipped prompt injection detection v2 (reduced false positives by 40%)
✅ Completed security review — all findings addressed
✅ Performance improvement: p99 latency from 2.1s to 1.4s

## Next Week
📋 Begin compliance review process
📋 Start user acceptance testing with HR department
📋 Deploy to staging environment for load testing

## Risks
⚠️ Compliance review may take 2 weeks (longer than the 1 week we planned)
   Mitigation: Engaged compliance lead early, provided all documentation upfront
⚠️ Model API provider announced maintenance window for April 15
   Mitigation: Scheduled our load testing for April 12-14

## Actions Needed
🔸 Compliance team: Confirm review start date (due: April 8)
🔸 Product team: Provide UAT test scenarios (due: April 9)

## Key Metrics
- Uptime: 99.97% (target: 99.9%)
- Average response time: 890ms (target: < 1s)
- User satisfaction (internal beta): 4.2/5 (target: 4.0/5)
- Open security findings: 0
```

## Cross-References

- `leadership-and-collaboration/communicating-with-executives.md` — Executive communication
- `leadership-and-collaboration/giving-status-updates.md` — Status update formats
- `leadership-and-collaboration/influencing-without-authority.md` — Influence strategies
- `leadership-and-collaboration/handling-blockers.md` — Blocker management
- `engineering-culture/async-communication.md` — Async communication practices
- `templates/stakeholder-update-template.md` — Stakeholder update template
