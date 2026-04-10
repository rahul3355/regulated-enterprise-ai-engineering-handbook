# Escalation Strategies

> "Escalation is not a failure. It's a tool for resolving problems that can't be resolved at the current level. The skill is knowing when and how."

## When to Escalate

Escalate when:

| Condition | Example |
|-----------|---------|
| **You've exhausted self-resolution** | Tried for 4+ hours, asked the team, still stuck |
| **The blocker is outside your control** | Need a decision from another team's lead |
| **Timeline risk is materializing** | Compliance review hasn't started, launch is in 2 weeks |
| **Stakeholders need to make a decision** | Two options, you need leadership to choose |
| **Resources are insufficient** | Need 2 more engineers, team can't deliver |
| **Risk is materializing** | Security vulnerability found, need immediate attention |
| **Conflict can't be resolved at your level** | Two teams disagree on approach, need arbitrator |

**Don't escalate when:**

| Condition | Better Approach |
|-----------|----------------|
| You haven't tried to resolve it yourself | Spend 2-4 hours attempting resolution |
| You're mildly annoyed | Handle it directly with the person/team |
| You want visibility for credit | Escalation for visibility is abuse of the process |
| It's a preference, not a blocker | Make the decision yourself or discuss with peers |

## Escalation Levels

```
Level 0: Self-Resolution (0-4 hours)
├── Debug the problem
├── Search documentation
├── Ask teammates
└── Try alternative approaches

Level 1: Team Resolution (4-24 hours)
├── Pair with senior teammate
├── Bring up in standup
├── Team lead helps directly
└── Team reallocates work

Level 2: Cross-Team Resolution (24-48 hours)
├── Contact other team's lead directly
├── Write formal request with context
├── Propose specific solution
└── Set deadline for response

Level 3: Management Escalation (48+ hours)
├── Engineering manager intervenes
├── Priority conflict resolution
├── Resource allocation
└── Cross-team alignment

Level 4: Executive Escalation (strategic impact)
├── VP/CTO involvement
├── Strategic priority decisions
├── Budget approvals
└── Organizational changes
```

## How to Escalate

### The Escalation Formula

Every escalation should include:

```
1. PROBLEM: What is blocked, clearly and specifically
2. IMPACT: Who is affected, how, and by when
3. CONTEXT: What caused the block
4. ATTEMPTS: What you've already tried
5. ASK: What specific action you need from the escalation target
6. URGENCY: When this needs to be resolved by
```

### Example: Cross-Team Escalation

```
To: Platform Team Lead
Cc: My Engineering Manager

Subject: BLOCKER: GenAI Service Cannot Deploy — Environment Access Needed by April 12

Problem:
Our GenAI chat service cannot be deployed to staging because
the service account lacks permission to create OpenShift Routes
in the genai-staging namespace.

Impact:
- Blocks deployment of the compliance review feature
- Compliance review is scheduled for April 15
- If we can't deploy by April 12, we miss the compliance window
- May delay product launch by 2 weeks

Context:
We requested the service account permissions in ticket #3421
on March 28. The ticket is still in "pending" status.

Attempts:
- Commented on ticket #3421 on April 3 (no response)
- Messaged @platform-support in Slack on April 5 (acknowledged, no action)
- Checked with our team — nobody has the permissions to self-serve

Ask:
Please grant Route creation permissions to service account
genai-chat-sa in namespace genai-staging by EOD April 11.

Ticket: #3421
Service account: genai-chat-sa
Namespace: genai-staging
Required permissions: routes create, list, get

If this isn't the right person to approve this, please redirect
to whoever can.

Thanks,
Sarah Chen
GenAI Platform Team
```

### Example: Management Escalation

```
To: Engineering Manager
Subject: Escalation: Compliance Review Timeline Risk

Hi [Manager],

I need your help with a compliance review timeline risk.

Problem:
The compliance review for the GenAI HR Assistant hasn't started.
Our launch date is May 1, and the review takes a minimum of 2 weeks.
If it doesn't start by April 15, we will miss our launch date.

Impact:
- CHRO has committed to the board that this launches in Q2
- May 1 is the latest date that qualifies as Q2
- A slip would push us to mid-May (Q3)

Context:
We submitted the review request on March 20. The compliance team
acknowledged receipt but has not assigned a reviewer. They are
currently working on 3 other reviews ahead of us.

Attempts:
- I contacted the compliance team lead on April 1 — they said
  they're understaffed and our review is queued for late April
- I offered to reduce the review scope — they're considering it
  but haven't committed to a timeline
- Our product manager contacted their product counterpart — same
  response

Ask:
Could you reach out to the Head of Engineering (who has a direct
relationship with the Head of Compliance) to request priority
for our review? We need it to start by April 15 at the latest.

I've prepared a summary of what the review covers and why the
timeline is critical, which you can forward.

Thanks,
Sarah
```

## Escalation Do's and Don'ts

### Do

| Action | Why |
|--------|-----|
| **Escalate early** | The sooner the problem is visible, the more options exist |
| **Be specific** | Vague escalations can't be acted on |
| **Include what you've tried** | Shows you're not escalating prematurely |
| **Propose a solution** | Makes it easier for the escalation target to help |
| **Be factual, not emotional** | "The ticket has been pending for 2 weeks" not "Nobody cares about our ticket" |
| **Follow up** | After escalating, check back. Don't assume it's handled |
| **Thank the resolver** | When the escalation is resolved, acknowledge the help |

### Don't

| Action | Why |
|--------|-----|
| **Escalate as a threat** | "I'll escalate this if you don't..." destroys relationships |
| **Skip levels unnecessarily** | Go to your direct manager first, unless it's about them |
| **Escalate publicly** | Escalations are sensitive. Don't post them in public channels |
| **Blame in the escalation** | Focus on the problem, not the person who caused it |
| **Escalate and abandon** | You still own the problem. Escalation is getting help, not handing off |
| **Cry wolf** | Escalating non-issues makes future escalations less credible |

## Escalation Decision Tree

```
Is something blocking a committed deliverable?
├── No → Don't escalate
└── Yes
    │
    ├── Have you tried to resolve it yourself?
    │   ├── No → Try for 2-4 hours first
    │   └── Yes
    │       │
    │       ├── Can your team resolve it?
    │       │   ├── Yes → Work on it as a team
    │       │   └── No
    │       │           │
    │       │           ├── Does it need management authority?
    │       │           │   ├── Yes → Escalate to manager
    │       │           │   └── No
    │       │           │           │
    │       │           │           ├── Does it need another team's action?
    │       │           │           │   ├── Yes → Contact their lead directly
    │       │           │           │   └── No → Identify the right resolver
    │       │           │
    │       │           └── Has the resolver been unresponsive for 48+ hours?
    │       │               ├── Yes → Escalate to your manager
    │       │               └── No → Wait and follow up
    │       │
    │       └── Is there a timeline risk?
    │           ├── Yes → Escalate with urgency
    │           └── No → Normal resolution process
```

## Escalation Tracking

At the team level, track escalations to identify patterns:

```markdown
# Escalation Log: [Team Name] — Q1 2026

| Date | Issue | Escalated To | Resolved? | Resolution Time | Root Cause |
|------|-------|-------------|-----------|-----------------|------------|
| Mar 5 | Env access blocked | Platform lead | ✅ | 2 days | Process gap in onboarding |
| Mar 18 | Compliance review delayed | Engineering manager | ✅ | 5 days | Compliance understaffing |
| Apr 2 | Security finding dispute | Security lead + eng mgr | ✅ | 3 days | Unclear security requirements |

Patterns:
- 2 of 3 escalations involve external team dependencies
- Average resolution time: 3.3 days
- Root cause trend: Process gaps, not technical issues

Actions:
- Create pre-flight checklist for environment access (prevents Mar 5 type)
- Engage compliance 4 weeks before review needed (prevents Mar 18 type)
- Publish security requirements earlier in design process (prevents Apr 2 type)
```

## Escalation and Relationships

Escalation can strain relationships if handled poorly:

### Preserving Relationships During Escalation

1. **Talk to the person first.** Before escalating about someone, talk to them directly. "Hey, I noticed the ticket has been pending — is there anything I can do to help move it forward?"

2. **Frame it as a process issue, not a person issue.** "Our process for requesting access seems to have a gap" not "Your team is ignoring our requests."

3. **Include the person in the escalation.** "I'm escalating this to get more visibility — it's not about anyone's performance, it's about getting the resources we need."

4. **Follow up with gratitude.** When the escalation resolves, thank everyone involved. "Thanks for helping us get unblocked. We really appreciate it."

5. **Debrief after.** "How did that escalation feel from your side? Is there a better way I could have handled it?"

## The "No Surprise" Rule

Your manager should never be surprised by an escalation. Before escalating to your manager:

1. Give them a heads-up. "I'm planning to escalate X. Here's the situation and what I'm asking for. Does this seem right?"
2. Incorporate their feedback. They may suggest a different approach.
3. Then escalate. With their awareness and input.

This applies at every level. Before your manager escalates to their manager, they should give their manager a heads-up.

## Cross-References

- `leadership-and-collaboration/handling-blockers.md` — Blocker identification and resolution
- `leadership-and-collaboration/giving-status-updates.md` — Status updates (escalation communication)
- `leadership-and-collaboration/managing-stakeholders.md` — Stakeholder management
- `leadership-and-collaboration/negotiation.md` — Negotiation (before escalation)
- `leadership-and-collaboration/communicating-with-executives.md` — Executive escalation format
