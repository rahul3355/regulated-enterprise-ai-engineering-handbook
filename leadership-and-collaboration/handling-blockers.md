# Handling Blockers

> "A blocker unaddressed is a commitment unkept. The speed at which you identify and escalate blockers determines your reliability."

## What Is a Blocker?

A blocker is anything preventing you from making progress on a committed deliverable:

| Type | Example | Resolution Owner |
|------|---------|-----------------|
| **Technical** | API dependency not available, bug in platform service | Engineering |
| **Cross-team** | Waiting for another team's deliverable | Other team |
| **Resource** | Need a security reviewer but none available | Management |
| **Decision** | Can't proceed without architecture decision | Tech lead / Architect |
| **External** | Vendor API down, compliance review delayed | External party |
| **Knowledge** | Don't know how to solve a technical problem | Team / Mentor |

**Not a blocker:** "I'm working on something else first." That's a prioritization issue, not a blocker.

## Blocker Identification

### Signs You're Blocked

- You can't complete a task because something external is required
- You've been waiting for a response for more than the expected timeframe
- You don't have information/decision/access needed to proceed
- You've tried multiple approaches and none work

### The "Two-Hour Rule"

If you've been stuck on a technical problem for **2 hours** without progress:
1. Document what you've tried
2. Ask for help (team channel, pair with someone)
3. If still stuck after another 2 hours, escalate to tech lead

This prevents the "I've been stuck for 3 days but didn't tell anyone" problem.

## Blocker Resolution Framework

### Step 1: Self-Resolve (0-4 hours)

```
Can I unblock myself?
├── Do I have all the information I need? → Search docs, check past decisions
├── Do I have all the access I need? → Request access, check onboarding docs
├── Is this a knowledge gap? → Ask team, pair with someone, read docs
├── Is this a technical problem? → Debug systematically, try alternative approaches
└── Have I tried for 2+ hours? → Move to Step 2
```

### Step 2: Team Help (4-24 hours)

```
Can someone on the team help?
├── Post in team channel with specific question
├── Pair with a teammate who has relevant expertise
├── Bring up in standup
├── Schedule a 30-min debugging session with a senior engineer
└── Still blocked after team help? → Move to Step 3
```

### Step 3: Escalate (24-48 hours)

```
Does this require action outside the team?
├── Another team's deliverable → Contact their lead directly
├── Resource need (reviewer, environment, tool) → Tell engineering manager
├── Decision needed from leadership → Write up the decision needed, send to decision-maker
├── External dependency (vendor, compliance) → Tell engineering manager
└── Still blocked after 48 hours total? → Move to Step 4
```

### Step 4: Management Escalation (48+ hours)

```
Does this require management intervention?
├── Engineering manager: Resource allocation, priority conflicts
├── Director: Cross-team priority conflicts
├── VP: Strategic alignment, budget decisions
└── Document the blocker, impact, and attempted resolutions
```

## Blocker Communication

### When You're Blocked

Tell someone immediately. The longer you wait, the worse the impact:

```
Good (immediate):
"Blocker: I can't complete the API integration because the auth
service endpoint is returning 503s. Ticket filed with platform
team (#4567). This blocks my current sprint commit. ETA from
platform team: unknown."

Bad (3 days later):
"Oh yeah, I couldn't finish that. The auth service was broken.
I figured someone else was handling it."
```

### Blocker Report Format

```markdown
# Blocker Report: [Name] — [Date]

## Active Blockers

### Blocker 1: [Brief Description]
- **Impact:** [What deliverable is affected]
- **Duration:** [How long you've been blocked]
- **Root Cause:** [What's causing the block]
- **Attempted Resolutions:** [What you've tried]
- **Help Needed:** [Specific: who needs to do what]
- **Status:** 🔴 Blocked / 🟡 Partially blocked / 🟢 Resolving

## Resolved Blockers (since last report)
- [Blocker] → Resolved by [how] on [date]
```

## Common Blockers in GenAI Engineering

### Model API Unavailable

```
Blocker: LLM API is returning 503 errors
Impact: All GenAI features non-functional
Resolution: 
├── Check provider status page
├── Activate fallback model if available
├── Communicate impact to users
├── Escalate to provider if SLA is breached
└── Post-incident: evaluate multi-provider strategy
```

### Compliance Review Delayed

```
Blocker: Compliance review hasn't started, launch date at risk
Impact: May delay product launch by 2+ weeks
Resolution:
├── Contact compliance lead directly
├── Understand their queue and priorities
├── Offer to reduce review scope
├── Escalate to engineering manager for VP-level alignment
└── Post-incident: Build earlier engagement into next project plan
```

### Security Finding Blocks Deployment

```
Blocker: Security review found critical vulnerability
Impact: Cannot deploy until fixed
Resolution:
├── Understand the finding fully (ask security for details if unclear)
├── Estimate fix effort
├── Implement fix or implement workaround
├── Request re-review
└── Post-incident: Address the class of vulnerability proactively
```

### Environment Issues

```
Blocker: Staging environment doesn't match production
Impact: Can't validate changes before deployment
Resolution:
├── Document the differences
├── File ticket with platform team
├── Find workaround if possible (local testing, mock services)
├── Escalate if blocking multiple teams
└── Post-incident: Add environment parity to platform requirements
```

## Preventing Blockers

### Proactive Blocker Prevention

| Strategy | How | When |
|----------|-----|------|
| **Dependency mapping** | List all external dependencies at sprint planning | Sprint planning |
| **Early engagement** | Contact other teams about their timelines before you need them | Project kickoff |
| **Pre-flight checks** | Verify access, environments, and dependencies before starting work | Task start |
| **Buffer time** | Include buffer for external dependencies in estimates | Estimation |
| **Regular check-ins** | Weekly status check with dependent teams | Ongoing |
| **Escalation criteria** | Define when a delay becomes an escalation | Project kickoff |

### The "Pre-Mortem" Technique

Before starting a project, imagine it has failed. Ask: "What blocked us?"

```
Pre-mortem: GenAI Compliance Assistant Project

"What if we miss our May 1 launch date? What blocked us?"

Team responses:
- "Compliance review took 4 weeks instead of 2"
- "The security team found a critical issue in week 3"
- "We couldn't get the production environment set up in time"
- "The model API provider changed their pricing model"

Actions:
- Engage compliance NOW, not when we're ready for review
- Start security review at design time, not at deployment time
- Request environment access in week 1
- Sign a 12-month pricing agreement with the provider
```

## Blocker Metrics

Track blocker patterns at the team level:

| Metric | Why |
|--------|-----|
| **Blocker frequency** | How often does the team get blocked? |
| **Average blocker duration** | How long do blocks last? |
| **Blocker type distribution** | What types of blockers are most common? |
| **Self-resolved vs. escalated** | What percentage does the team resolve on its own? |

**Goal:** Reduce blocker frequency and duration over time. If the same type of blocker keeps appearing, fix the system.

## Blocker Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact |
|--------------|-------------------|--------|
| **Silent blocking** | Person is stuck but doesn't tell anyone | Days of wasted effort |
| **Premature escalation** | Escalating before trying to self-resolve | Wastes other people's time |
| **Late escalation** | Waiting until the deadline to mention a blocker | No time to resolve |
| **Vague blocker description** | "I'm blocked on the API" (which API? what's wrong?) | Can't help without specifics |
| **Blocker hoarding** | "I'll figure it out myself" for days | Inefficient, risky |
| **No follow-up** | Escalated but never checked back | Blocker may have been forgotten |

## Escalation Templates

### Direct Escalation (to another team's lead)

```
Hi [Name],

I'm blocked on [specific issue] which is affecting [specific deliverable].

What I need: [specific request]
By when: [deadline]
Impact of delay: [what happens if this isn't resolved]

What I've tried: [brief list]

Can you help? If this isn't the right person, could you point me
to who can help?

Thanks,
[Name]
```

### Management Escalation

```
Hi [Manager],

I need your help unblocking [deliverable].

The blocker: [specific description]
Impact: [timeline, users, business impact]
Duration: [how long blocked]
Attempted resolutions: [what's been tried]

I need: [specific: what the manager should do]
Timeline: [when it needs to be resolved by]

I've already contacted [person/team] on [date] and [result].

Thanks,
[Name]
```

## Cross-References

- `leadership-and-collaboration/escalation-strategies.md` — Escalation strategies
- `leadership-and-collaboration/giving-status-updates.md` — Status updates (blocker reporting)
- `leadership-and-collaboration/managing-stakeholders.md` — Stakeholder management
- `engineering-culture/async-communication.md` — Async communication for blocker resolution
- `incident-management/incident-response.md` — Incident response (blockers during incidents)
