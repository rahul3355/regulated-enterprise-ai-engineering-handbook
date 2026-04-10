# Incident Retrospectives

> "Every incident is a learning opportunity. The question is: will we learn, or will we repeat?"

## The Purpose of Retrospectives

An incident retrospective (also called a postmortem or blameless postmortem) has one goal: **understand what happened well enough to prevent it from happening again.**

It does NOT have these goals:
- Assigning blame
- Determining who made the mistake
- Proving that someone should have known better
- Justifying why the team is understaffed

**If a retrospective feels like a trial, we have failed.**

## Blamelessness: The Core Principle

### What Blamelessness Means

Blamelessness does NOT mean nobody made a mistake. It means:

1. **We assume everyone did the best they could with the information they had.**
2. **We focus on the system, not the person.** If a person made a mistake, the question is "what about our system allowed this mistake to cause an outage?"
3. **We recognize that hindsight bias is real and destructive.** It's easy to say "they should have known" after the fact.

### The "Second Victim" Problem

In incident response, there are often two victims:
1. **The users** who experienced the outage
2. **The engineer** who caused it (often accidentally)

If we punish the second victim, we create a culture where people hide mistakes. Hidden mistakes cannot be fixed.

### Real Example: Blame vs. Blameless

**Blame-oriented (BAD):**
> "John ran the migration without checking the disk space first. This was careless. He needs to be more careful next time."

**Blameless (GOOD):**
> "The migration script did not check available disk space before execution. The runbook does not include a disk space prerequisite step. Our monitoring did not alert on low disk space until it was too late. We should: (1) add disk space check to the migration script, (2) update the runbook, (3) add monitoring alert for disk space < 20%."

Notice: John is not mentioned. The SYSTEM is the focus.

## Retrospective Process

### Timeline

| When | What | Who |
|------|------|-----|
| **During incident** | Take notes, save logs, preserve evidence | Incident commander |
| **Within 24 hours** | Send incident summary to stakeholders | Incident commander |
| **Within 48 hours** | Schedule retrospective (within 5 business days) | Engineering manager |
| **Within 5 business days** | Hold retrospective meeting | Facilitator + team |
| **Within 2 business days after meeting** | Publish retrospective document | Retrospective author |
| **Within 2 weeks** | Complete action items | Assigned owners |
| **Within 30 days** | Follow up on action item completion | Engineering manager |

### Retrospective Meeting Structure (60 minutes)

```
Meeting: Incident Retrospective for [Incident Name]
Duration: 60 minutes
Attendees: Incident responders, affected team members, facilitator

0:00 - 0:05  Welcome and ground rules
             - This is blameless
             - We focus on systems, not people
             - Everyone's perspective is valuable

0:05 - 0:20  Timeline reconstruction
             - What happened, in order
             - Build the timeline together
             - "What did you know at each point?"

0:20 - 0:35  Contributing factors
             - What conditions allowed this to happen?
             - What systems failed?
             - What signals were missed?

0:35 - 0:50  Action items
             - What can we do to prevent recurrence?
             - Assign owners and deadlines
             - Prioritize by impact and effort

0:50 - 0:55  Lessons learned
             - What did we learn about our systems?
             - What did we learn about our processes?
             - What did we learn about each other?

0:55 - 1:00  Close
             - Thank everyone
             - Confirm action item owners
             - Schedule follow-up
```

### Facilitator Guidelines

The facilitator is NOT the incident commander. The facilitator should be someone who was not directly involved in the incident.

**Facilitator responsibilities:**
- Set the blameless tone from the start
- Ensure everyone has a chance to speak
- Keep the conversation focused on systems, not people
- Redirect blame language: "Instead of 'who did this,' let's ask 'what in our process allowed this?'"
- Ensure action items are specific and assigned
- Watch for hindsight bias: "It's easy to see the answer now. The question is what was knowable at the time."

**Facilitator phrases to use:**
- "Walk us through what you were seeing at that point."
- "What information did you have available?"
- "What were you optimizing for at that moment?"
- "What would have helped you make a different decision?"

**Facilitator phrases to avoid:**
- "Why did you do that?"
- "Shouldn't you have checked...?"
- "Anyone would have known to..."
- "How did we miss this?"

## Retrospective Document Template

See `templates/postmortem-template.md` for the full template.

Key sections:
1. **Summary** — What happened, impact, duration
2. **Timeline** — Chronological events
3. **Root cause** — The chain of events and conditions
4. **Contributing factors** — What made this possible
5. **Impact** — Who was affected and how
6. **Action items** — Specific, assigned, time-bound
7. **Lessons learned** — What we now know that we didn't before

## The "Five Whys" — Used Carefully

The Five Whys technique can be useful but is often misused:

### Good Use

```
Problem: The GenAI chatbot returned incorrect compliance information.

Why? The RAG pipeline retrieved outdated policy documents.
Why? The document index had not been updated after the policy change.
Why? The document update process requires manual trigger.
Why? We assumed the scheduled indexer would catch it.
Why? The scheduled indexer only runs weekly, but policy changes are ad-hoc.

Root cause: No mechanism to trigger re-indexing on policy document changes.
Fix: Event-driven re-indexing when compliance documents are updated.
```

### Bad Use

```
Problem: The GenAI chatbot returned incorrect compliance information.

Why? The RAG pipeline retrieved outdated policy documents.
Why? Sarah didn't update the index after the policy change.
Why? Sarah forgot.
Why? Sarah was distracted by the P1 incident.
Why? Sarah is the only person who knows how to do this.

Conclusion: Sarah needs to be more organized and we need backup.
```

This is bad because it stops at "Sarah" instead of examining the system that allowed a single person to be the only one who knows how to do something critical.

## Action Item Management

### Action Item Criteria

Every action item must be:

| Criterion | Example | Non-Example |
|-----------|---------|-------------|
| **Specific** | "Add disk space check to migration script" | "Be more careful with migrations" |
| **Assigned** | "Assigned to: @james.wu" | "Someone should fix this" |
| **Time-bound** | "Due: 2026-04-25" | "Soon" |
| **Testable** | "Migration script fails if disk space < 20%" | "Improve migration reliability" |

### Tracking

- Action items are tracked in Jira with the label `retrospective-action`
- Engineering manager reviews open retrospective actions in weekly team sync
- Actions overdue by 2 weeks are escalated to the engineering manager
- Actions overdue by 4 weeks are escalated to the director

### Prioritization

| Priority | Criteria | Response Time |
|----------|----------|---------------|
| **P0** | Prevents exact recurrence of a P0/P1 incident | 1 week |
| **P1** | Prevents similar class of incidents | 2 weeks |
| **P2** | Improves detection or response time | 4 weeks |
| **P3** | Improves documentation or training | Next sprint |

## Common Retrospective Patterns

### The "Swiss Cheese" Model

Incidents rarely have a single cause. They happen when multiple layers of defense all have holes that align:

```
Layer 1: Code Review        →   Missed the bug
Layer 2: Testing            →   No test for this case
Layer 3: Staging            →   Staging didn't match production
Layer 4: Monitoring         →   No alert for this condition
Layer 5: Incident Response  →   Took 45 min to identify root cause

                          INCIDENT
```

Each layer had a hole. The retrospective should identify how to strengthen EACH layer, not just the one closest to the bug.

### The "Normal Accident" Theory

Some systems are so complex that incidents are inevitable. The goal is not zero incidents (impossible), but:
- Reduce frequency of severe incidents
- Improve detection and response time
- Learn from every incident

### The "Drift Into Failure" Pattern

Systems slowly drift from safe to unsafe through small, individually reasonable decisions:

```
Month 1: Skip the migration dry-run to save time (it's simple)
Month 2: Don't update the runbook (it's still basically right)
Month 3: Ignore the disk space warning (it's been fine)
Month 4: Migration runs out of disk space → OUTAGE
```

Each decision was reasonable in isolation. Together, they created the conditions for failure.

## Retrospective Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact |
|--------------|-------------------|--------|
| **Blame game** | Focusing on who made the mistake | People stop sharing honestly |
| **Action item graveyard** | Long lists of actions that never get done | Cynicism about the process |
| **Root cause obsession** | Insisting on a single root cause | Misses systemic issues |
| **Too soon** | Holding retrospective immediately after incident | Emotions are too high |
| **Too late** | Holding retrospective 3 weeks after incident | Details are forgotten |
| **Too big** | 30 people in the room | Nobody can contribute meaningfully |
| **Too small** | Only the incident responders | Missing perspectives from affected teams |
| **No follow-through** | Meeting happens, document is published, nothing changes | Wastes everyone's time |

## Building a Learning Culture from Incidents

### Incident Review Rotation

Every engineer should review at least one incident retrospective per month from outside their team. This:
- Spreads knowledge about failure modes
- Builds empathy for other teams
- Generates ideas for improving your own systems

### Monthly Incident Review

Once a month, the team reviews:
- All incidents from the past month
- Status of retrospective action items
- Trends (are incidents increasing? decreasing? changing type?)
- Lessons that apply across multiple incidents

### Incident Pattern Library

Maintain a library of common incident patterns:

```
incident-management/patterns/
├── database-connection-exhaustion.md
├── cascading-failure-from-single-service.md
├── memory-leak-in-python-service.md
├── model-api-timeout-cascade.md
├── misconfigured-feature-flag.md
└── ...
```

Each pattern includes:
- Symptoms
- Root cause
- Resolution
- Prevention measures

## Cross-References

- `incident-management/` — Full incident management practices
- `templates/postmortem-template.md` — Postmortem template
- `incident-management/incident-response.md` — Incident response procedures
- `engineering-culture/psychological-safety.md` — Creating safe environment for retrospectives
- `observability/alerting.md` — Improving detection to reduce incident impact
