# Async-First Communication

> "In a distributed team, if it isn't written down, it didn't happen. Async communication isn't a compromise — it's the default."

## Why Async-First Matters

Our GenAI engineering teams are distributed across time zones. Even when everyone is in the same office, deep work requires uninterrupted focus time that synchronous communication destroys.

**Async-first means:**
- Default to written communication
- Decisions happen in documents, not meetings
- Meetings are for discussion, not for information sharing
- Information is findable, not trapped in someone's DMs

**Async-first does NOT mean:**
- No meetings ever
- No real-time collaboration
- No human connection
- Slow communication

It means **intentional** communication. Synchronous when it adds value. Async when it doesn't need to be live.

## The Communication Hierarchy

```
Urgency: HIGH              Urgency: LOW
Impact: HIGH               Impact: HIGH
    │                          │
    ▼                          ▼
Phone call / Page          Written document
(Incident response)        (Design doc, RFC)
                               │
                               ▼
                          Post in channel
                          (for discussion)

Urgency: HIGH              Urgency: LOW
Impact: LOW                Impact: LOW
    │                          │
    ▼                          ▼
Direct message             Email / Thread
(Quick question)           (FYI, non-urgent)
```

## Communication Channels and Their Purposes

### Slack / Teams

| Channel Type | Purpose | Response Expectation |
|--------------|---------|---------------------|
| `#incidents` | Active incident coordination | Immediate |
| `#team-genai` | Team-wide announcements | Same day |
| `#team-genai-discussion` | Technical discussions | 4 business hours |
| `#security-alerts` | Security notifications | Same day |
| `#random` | Social, non-work | No expectation |

**Rules:**
- Use threads. Always.
- Don't DM for work questions — post in the channel so others can learn
- Use `@here` sparingly (only for time-sensitive team-wide matters)
- Use `@channel` only with team lead approval
- Status messages are your friend: "Deep work until 2 PM", "In meetings all day", "On-call"

### Email

**Use for:**
- Formal communications (policy changes, organizational announcements)
- External communications (vendors, partners)
- Communications that need a formal record

**Don't use for:**
- Technical discussions (use Slack threads or GitHub discussions)
- Quick questions (use Slack DMs)
- Anything that needs a response today (use Slack)

### GitHub Discussions / Confluence

**Use for:**
- RFCs and design discussions
- Architecture decisions
- Technical debates that span days
- Knowledge that needs to persist

**Don't use for:**
- Urgent decisions
- Personal feedback
- Sensitive conversations

### Meetings

**Use for:**
- Complex discussions with many viewpoints
- Relationship building and team bonding
- Sensitive conversations
- Brainstorming (where building on ideas is valuable)

**Don't use for:**
- Status updates (write them instead)
- Information sharing (send a doc instead)
- Decisions that one person could make (just make the decision)

## The "Write It Down" Discipline

### Decision Logs

Every significant decision should be recorded:

```markdown
# Decision Log: [Team/Project Name]

| Date | Decision | Rationale | Deciders |
|------|----------|-----------|----------|
| 2026-04-10 | Use pgvector for embeddings | Lower operational complexity, already run Postgres | @sarah, @james |
| 2026-04-08 | Pin model to gpt-4-turbo-2024-04-09 | Reproducibility, can't risk model changing | @priya |
```

### Status Updates

Weekly status updates should be written, not presented:

```markdown
# Weekly Status: [Name] — Week of [Date]

## This Week
- Completed: [What you finished]
- In Progress: [What you're working on]
- Blockers: [What's stopping you]

## Next Week
- Planned: [What you'll work on]

## Metrics (if applicable)
- [Key numbers: uptime, latency, error rate, etc.]

## Risks
- [Anything stakeholders should be aware of]
```

### Meeting Alternatives

Before scheduling a meeting, ask: **Could this be an email/Slack message/doc?**

| Meeting Type | Async Alternative |
|--------------|-------------------|
| Status update | Written status doc with comments |
| Information sharing | Recorded video or written doc |
| Simple decision | RFC with comment period |
| Code discussion | PR review comments |
| Design alignment | Design doc with review cycle |

## Async Decision-Making Process

1. **Write the proposal.** Use the RFC template or a brief doc.
2. **Share with stakeholders.** Post in the relevant channel, tag the people whose input matters.
3. **Set a deadline.** "Please share feedback by EOD Thursday."
4. **Respond to all comments.** Even if it's "Good point, addressed in the doc."
5. **Make the decision.** After the deadline, the proposal owner decides.
6. **Document the outcome.** Update the doc with the decision and rationale.

### Handling Disagreements Async

When people disagree in an async discussion:

1. **Acknowledge the disagreement.** "I see we have different views on X."
2. **Clarify the disagreement.** "The disagreement is about whether Y is a real risk."
3. **Propose a path to resolution.** "Let me run a spike to test this and share results by Friday."
4. **Decide.** If consensus isn't possible, the decision-maker decides and documents the rationale.
5. **Commit.** Once decided, everyone commits to the decision even if they disagreed.

## Time Zone Considerations

Our teams span multiple time zones. Async communication must be inclusive:

### Guidelines

| Practice | Rationale |
|----------|-----------|
| No expectation of instant replies | Someone is always off-hours |
| Rotate meeting times | Don't always disadvantage the same timezone |
| Record all meetings | For those who can't attend |
| Use 24-hour time in writing | Avoids AM/PM confusion |
| Include timezone in status | "GMT+5:30, available 9 AM - 6 PM IST" |
| Write complete context | The person reading at 2 AM their time can't ask clarifying questions |

### Meeting Time Fairness

```
If your team spans:
- US East (ET)
- UK (GMT)
- India (IST)

Rotation schedule:
Week 1: 9 AM ET = 2 PM GMT = 7:30 PM IST (US-friendly)
Week 2: 8 AM GMT = 3 AM ET = 1:30 PM IST (India-friendly)
Week 3: 6 PM IST = 1:30 PM GMT = 8:30 AM ET (UK-friendly)

No single timezone is always the "inconvenient" one.
```

## Writing for Async: Best Practices

### Be Complete

When writing async communication, include ALL the context. The reader can't ask you a follow-up question right now.

**Bad:**
> "The API is broken. Can someone look?"

**Good:**
> "The `/v2/chat` endpoint is returning 503 errors since the deployment at 14:32 UTC.
> Logs show connection refused to the model-api service. The deployment was PR #342
> (model version upgrade). Rolling back now. Need someone from the platform team
> to check if there's a known issue with the new model-api image."

### Use Formatting

- **Bold** for emphasis
- `code` for technical terms and commands
- Bullet points for lists
- Headers for sections
- Links for context

### Assume No Context

Write as if the reader wasn't in the previous conversations. Link to relevant docs, don't assume they know the history.

## Async Communication Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact | Fix |
|--------------|-------------------|--------|-----|
| **DM decisions** | Important decisions made in 1:1 DMs | Nobody else knows | Post in channels |
| **Incomplete context** | "Can you help with the thing?" | Wastes time asking for context | Write complete context |
| **Ambiguous deadlines** | "Let me know what you think" with no deadline | Discussions drag on forever | Set explicit deadlines |
| **Meeting by default** | Scheduling a meeting for every discussion | Calendar overload, timezone pain | Write a doc first |
| **Ghost channels** | Channels where nobody posts | Information goes elsewhere | Revive or archive the channel |
| **Over-notification** | @channel for everything | Notification fatigue, ignored alerts | Use @channel only for urgent team-wide matters |

## Measuring Async Communication Health

| Metric | Target | Why |
|--------|--------|-----|
| Decision documentation rate | > 90% of decisions logged | Knowledge persists |
| Response time (non-urgent) | < 4 business hours | Respectful without being urgent |
| Meeting-to-doc ratio | Fewer meetings than docs | Information should be async |
| Thread usage rate | > 80% of messages in threads | Channels stay readable |
| Cross-timezone participation | All time zones contribute to discussions | Inclusive decision-making |

## The "No Meeting Wednesday" Experiment

Many teams in our organization have adopted **No-Meeting Wednesdays**:

- No scheduled meetings on Wednesdays (except P1/P0 incident response)
- Full day for deep work: coding, design, writing
- Meetings that would fall on Wednesday are moved to Tuesday or Thursday

**Results after 6 months:**
- 40% increase in PR throughput on Wednesdays
- Engineers report higher satisfaction and lower burnout
- Fewer complaints about "too many meetings"
- Design doc quality improved (engineers have time to think)

**How to implement:**
1. Team agrees to the experiment for one quarter
2. Calendar blocks are set for the whole team
3. Exceptions require team lead approval
4. Review at end of quarter — continue, modify, or stop

## Cross-References

- `engineering-culture/documentation-culture.md` — Documentation as engineering responsibility
- `engineering-culture/design-docs.md` — Writing design docs for async review
- `engineering-culture/rfcs.md` — RFC process (async decision-making)
- `leadership-and-collaboration/writing-strategy-docs.md` — Strategy document writing
- `leadership-and-collaboration/giving-status-updates.md` — Status update formats
