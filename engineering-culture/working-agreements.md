# Working Agreements

> "Working agreements aren't rules imposed from above. They're promises we make to each other about how we'll work together."

## What Are Working Agreements?

Working agreements are explicit commitments that a team makes about how they will work together. They cover things that are usually implicit (and therefore inconsistent):

- How quickly do we respond to messages?
- What makes a PR "ready to review"?
- When do we hold meetings, and when do we write docs?
- How do we handle on-call handoffs?
- What does "done" mean?

**Implicit agreements are fragile.** They break when new people join, when teams merge, when stress is high. **Explicit agreements are resilient.**

## How Working Agreements Are Created

### Initial Creation (New Teams)

1. **Workshop session (90 minutes):** The team brainstorms agreements together
2. **Categorize:** Group into communication, code quality, meetings, on-call, etc.
3. **Prioritize:** Each person votes for their top 5 — focus on what matters most
4. **Draft:** Write specific, actionable agreements (not vague aspirations)
5. **Commit:** Each person verbally commits to the agreements

### Evolution (Existing Teams)

1. **Retrospective trigger:** "This agreement isn't working — should we change it?"
2. **New member trigger:** "Here's how we work — does this make sense for you too?"
3. **Quarterly review:** Revisit all agreements during quarterly team retro

## Sample Working Agreements

### Communication

| Agreement | Rationale |
|-----------|-----------|
| We respond to Slack messages within 4 business hours | Respects deep work time while ensuring responsiveness |
| We use threads for all Slack conversations | Keeps channels readable and searchable |
| P1/P0 issues are escalated via phone call, not Slack | Ensures urgent issues get immediate attention |
| We document decisions in writing, not in DMs | Knowledge must be findable by others |
| Status updates are posted by 10 AM local time each Tuesday | Keeps stakeholders informed without meetings |

### Code Quality

| Agreement | Rationale |
|-----------|-----------|
| All PRs include tests for new logic | Tests are the safety net for future changes |
| PRs are reviewed within 24 hours | Unreviewed PRs block delivery |
| We don't merge without CI passing | Green tests are the minimum bar |
| Security-sensitive changes require security team review | Defense in depth |
| We use the PR template for every PR | Consistency and completeness |

### Meetings

| Agreement | Rationale |
|-----------|-----------|
| Every meeting has an agenda and a goal | Meetings without purpose waste time |
| We decline meetings where we're not needed | Protects focus time |
| No-meeting Wednesdays (except P1/P0 incidents) | Guarantees uninterrupted work time |
| Meetings start and end on time | Respects everyone's schedule |
| Meeting notes are shared within 1 hour | Decisions are captured and shared |

### On-Call

| Agreement | Rationale |
|-----------|-----------|
| On-call engineers mute non-urgent notifications | Reduces cognitive load during incidents |
| Incident postmortems are completed within 5 business days | Learning happens while details are fresh |
| On-call handoff includes written summary | Continuity between shifts |
| We rotate on-call fairly (max 1 week per 6 weeks) | Prevents burnout |
| Post-incident, the on-call engineer gets the next day off | Recovery time after stressful incidents |

### GenAI-Specific Agreements

| Agreement | Rationale |
|-----------|-----------|
| All model version changes are pinned, never "latest" | Reproducibility and rollback capability |
| All prompts are version-controlled, never hardcoded in production | Auditability and change tracking |
| RAG source documents are classified before inclusion | Compliance with data access policies |
| Prompt injection tests are run in CI for all GenAI endpoints | Security-critical for AI systems |
| Model response quality is monitored, not just latency | Users care about answer quality, not just speed |

## Working Agreement Anti-Patterns

| Anti-Pattern | What It Looks Like | Why It Fails |
|--------------|-------------------|--------------|
| **Management-imposed** | "Here are the team's working agreements" (from manager) | Team doesn't own them |
| **Too many** | 40 agreements that nobody remembers | Dilutes focus |
| **Too vague** | "We communicate well" | Not actionable |
| **Never reviewed** | Agreements set in stone for 2 years | Stale and irrelevant |
| **No accountability** | Agreements exist but nobody follows them | Worse than no agreements |
| **Punitive enforcement** | "You violated agreement #7" | Creates fear, not commitment |

## How to Handle Agreement Violations

When someone breaks a working agreement:

### Good Response

> "Hey, I noticed this PR didn't have tests. We agreed that all new logic includes tests. Is there a reason this one is different? No worries if there's a good reason — just want to check."

### Bad Response

> "You didn't include tests. That's against our agreements."

The difference: the good response assumes positive intent and seeks understanding. The bad response is accusatory.

### When Agreements Need to Change

Sometimes an agreement is broken not because someone is careless, but because the agreement is unrealistic:

> "We agreed to review PRs within 4 hours, but with the current team size and workload, that's not sustainable. Should we change it to 24 hours?"

**Breaking an agreement is data.** It tells you something about the system.

## Team-Specific vs. Organization-Wide Agreements

| Level | Scope | Example |
|-------|-------|---------|
| **Team-specific** | Agreements within a single team | "We do standups at 10:30 AM" |
| **Cross-team** | Agreements between teams that work together | "API changes are communicated 2 weeks before deployment" |
| **Organization-wide** | Agreements that apply to all engineering | "All P1 incidents get a postmortem within 5 days" |

Team-specific agreements can be stricter than organization-wide ones, but never looser. (Organization-wide agreements set the floor, not the ceiling.)

## Working Agreement Template for Teams

```markdown
# Team Working Agreements: [Team Name]
# Last Reviewed: [Date]
# Next Review: [Date]

## How We Communicate
- [Agreement 1]
- [Agreement 2]

## How We Write Code
- [Agreement 1]
- [Agreement 2]

## How We Run Meetings
- [Agreement 1]
- [Agreement 2]

## How We Handle On-Call
- [Agreement 1]
- [Agreement 2]

## How We Define "Done"
- [Agreement 1]
- [Agreement 2]

## Exceptions and Context
- [When and how we bend these agreements]

## Review History
- [Date]: [What changed and why]
```

## Quarterly Review Process

Every quarter, the team spends 30 minutes reviewing working agreements:

1. **Read each agreement aloud** (or display them)
2. **Ask: Is this still working?** (thumbs up/down/sideways)
3. **For thumbs down or sideways:** Discuss what needs to change
4. **Update the document** with any changes
5. **Add new agreements** if gaps are identified
6. **Remove agreements** that are no longer relevant

## The "Why" Behind Agreements

Every agreement should have a clear rationale. If you can't explain WHY the agreement exists, it probably shouldn't be an agreement.

Example:

| Agreement | Why |
|-----------|-----|
| PRs are reviewed within 24 hours | Because unreviewed PRs block deployment, and blocked deployments delay features and fixes that users need. In banking, delayed security patches can be a compliance issue. |
| No-meeting Wednesdays | Because engineers need 4+ hours of uninterrupted focus time to do deep work on complex problems like debugging production issues or designing new systems. |
| On-call gets next day off | Because incident response is cognitively exhausting, and working while fatigued leads to more mistakes and burnout. |

When team members understand the WHY, they follow the agreement even when it's inconvenient.

## Cross-References

- `engineering-culture/team-rituals.md` — Team rituals and ceremonies
- `engineering-culture/async-communication.md` — Async communication practices
- `engineering-culture/psychological-safety.md` — Creating psychological safety
- `engineering-culture/working-agreements.md` — This document
- `incident-management/on-call.md` — On-call procedures
