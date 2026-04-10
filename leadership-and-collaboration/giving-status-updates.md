# Giving Status Updates

> "A good status update answers the questions before they're asked. A great status update prevents the need for a status meeting."

## Why Status Updates Matter

In a banking GenAI platform, stakeholders need to know:
- Is the project on track?
- Are there risks they should be aware of?
- Do they need to make decisions or provide resources?
- Are users/customers being affected?

**Bad status updates create anxiety.** Stakeholders who don't have clear information assume the worst.

**Good status updates create confidence.** Even bad news, communicated early and clearly, allows stakeholders to help.

## Status Update Principles

### 1. Lead with the Bottom Line

```
Bad: "This week we worked on the API integration and we found some
interesting issues with the model response format and we're looking
into a few options and we'll probably have something next week..."

Good: "On track. API integration is 80% complete. One issue found
with response parsing — fix in progress. Expected completion: Friday."
```

### 2. Be Honest About Risks

```
Bad: "No risks to report." (When there clearly are risks)

Good: "Risk: The compliance review may take longer than planned.
We've not yet received confirmation of their start date. If it
doesn't start by April 15, we'll slip our May 1 launch. Mitigation:
Escalating to the compliance lead this week."
```

### 3. Quantify Everything

```
Bad: "Performance is better now."
Good: "p99 latency dropped from 2.1s to 1.4s (33% improvement)."

Bad: "Users seem happy with it."
Good: "User satisfaction score: 4.2/5 from 127 respondents (target: 4.0/5)."
```

### 4. Include the Ask

Every status update should clearly state what you need from the reader:

```
"No action needed from you."
"Decision needed: Should we proceed with option A or B? Need answer by Friday."
"Resource needed: We need a security reviewer assigned by April 12."
"Awareness only: FYI that the model API has scheduled maintenance on April 15."
```

## Status Update Formats

### Weekly Written Update (for team leads and engineering managers)

```markdown
# Weekly Status: [Name/Team] — Week of [Date]

## Bottom Line
[One sentence: On track / At risk / Off track]

## Progress This Week
- ✅ [Completed item 1]
- ✅ [Completed item 2]
- ⏳ [In-progress item, % complete]

## Plan for Next Week
- 📋 [Planned item 1]
- 📋 [Planned item 2]

## Risks and Blockers
| Risk/Blocker | Impact | Mitigation | Help Needed |
|-------------|--------|-----------|-------------|
| [Description] | [High/Med/Low] | [What we're doing] | [From whom] |

## Key Metrics
| Metric | Current | Target | Trend |
|--------|---------|--------|-------|
| [Metric 1] | [Value] | [Target] | [Up/Down/Stable] |

## Decisions Needed
- [ ] [Decision 1] — by [Date]
- [ ] [Decision 2] — by [Date]

## Shout-outs
- [Recognition for someone's contribution]
```

### Executive Status Update (monthly, for VP/CTO level)

See `communicating-with-executives.md` for the detailed format. Key differences from weekly updates:
- Higher-level (no task-level detail)
- Focus on strategic alignment and risk
- Include budget and resource information
- Trend data (month-over-month)

### Incident Status Update (during active incidents)

```markdown
# Incident Status: [Incident Name] — [Time]

## Current Status
[Active / Monitoring / Resolved]

## Impact
- [Who is affected]
- [How many users/systems]
- [Duration so far]

## What We Know
- [Root cause if known, or "under investigation"]
- [What's broken]
- [What's working]

## What We're Doing
- [Current action]
- [ETA for next update]

## What We Need
- [Any resources or decisions needed]

## Next Update
[When the next status will be posted — typically every 30-60 min]
```

### Sprint Status (for product and stakeholders)

```markdown
# Sprint [N] Status — [Dates]

## Sprint Goal
[One sentence: what this sprint is trying to achieve]

## Progress
| Item | Status | Notes |
|------|--------|-------|
| [Story 1] | ✅ Done | [Brief note] |
| [Story 2] | ⏳ In Progress | [70% complete, on track] |
| [Story 3] | ⚠️ At Risk | [Blocked on X, need Y] |
| [Story 4] | ❌ Not Started | [Carried over from previous sprint] |

## Sprint Health
- Scope: ✅ On track / ⚠️ Scope creep / ❌ Over-committed
- Timeline: ✅ On track / ⚠️ Some items at risk / ❌ Will not complete
- Quality: ✅ No issues / ⚠️ Some bugs / ❌ Quality concerns

## Carry-over Items
- [What's not completing and why]

## Next Sprint Preview
- [What's planned for next sprint]
```

## Escalating in Status Updates

When a status update needs to convey escalation:

### The Escalation Format

```markdown
## ⚠️ ESCALATION: [Issue]

### What's Happening
[2-3 sentences, factual]

### Impact
[Quantified impact on users, timeline, or business]

### What We've Done
[Bullet points of resolution attempts]

### What We Need
[Specific: who needs to do what by when]

### Recommendation
[Your recommended course of action]

### Timing
[When this needs to be resolved by to avoid worse impact]
```

### Escalation Levels

| Level | When | Who | Format |
|-------|------|-----|--------|
| **Team-level** | Blocker within team's control | Team lead | Standup mention, Slack |
| **Cross-team** | Blocker requires another team's action | Other team's lead | Direct message + formal request |
| **Management** | Blocker requires resource/priority change | Engineering manager | Written escalation + meeting |
| **Executive** | Blocker affects strategic commitments | VP/CTO | Executive escalation format |

## Risk Communication

### The Risk Register

For significant projects, maintain a risk register:

```markdown
# Risk Register: [Project Name]
# Updated: [Date]

| # | Risk | Likelihood | Impact | Owner | Mitigation | Status |
|---|------|-----------|--------|-------|-----------|--------|
| 1 | Compliance review delayed | Medium | High | @sarah | Early engagement, scoped review | Active |
| 2 | Model API capacity limits | Low | High | @james | Reserved capacity agreement | Monitoring |
| 3 | Key engineer unavailable | Low | Medium | @manager | Cross-training, documentation | Accepted |
```

### Communicating Risk

When communicating risk to stakeholders:

**Do:**
- Be specific about the risk (not vague "there might be issues")
- Quantify likelihood and impact
- Describe the mitigation (what you're doing about it)
- State what help you need (if any)

**Don't:**
- List every conceivable risk (focus on the top 3-5)
- Present risks without mitigations (creates anxiety)
- Downplay risks to seem confident (destroys trust when they materialize)
- Surprise stakeholders with risks that materialize without warning

## Status Update Anti-Patterns

| Anti-Pattern | What It Looks Like | Why It's Bad |
|--------------|-------------------|--------------|
| **The novel** | 5 pages of detail | Nobody reads it |
| **The haiku** | "Everything's fine." | No useful information |
| **The surprise** | "Oh, and we're 2 weeks behind" | Stakeholders should never be surprised |
| **The blame** | "We're behind because the security team took forever" | Unprofessional and unhelpful |
| **The optimist** | Everything is green, but the project is actually on fire | Destroys trust when the truth comes out |
| **The pessimist** | Everything is red, but most things are actually fine | Creates unnecessary anxiety |
| **The orphan** | Status update posted where nobody reads it | Wasted effort |

## Frequency Guidelines

| Audience | Frequency | Format |
|----------|-----------|--------|
| Team | Daily (standup) | Verbal, 2 minutes per person |
| Team lead / Engineering manager | Weekly | Written update |
| Product manager | Weekly | Written update + 15-min sync |
| Stakeholders (other teams) | Bi-weekly | Written summary |
| Executives | Monthly | 1-page summary + 15-min call |
| During incidents | Every 30-60 minutes | Incident status format |

## Status Update Distribution

Post status updates in consistent, findable locations:

| Audience | Where |
|----------|-------|
| Team | Slack thread, team channel |
| Engineering manager | Email + Confluence page |
| Stakeholders | Confluence page, tagged notification |
| Executives | Email (they won't look in Confluence) |
| Incident responders | Incident channel, pinned message |

**Rule:** If you're posting a status update and nobody comments or asks questions, it might mean it's perfect (everyone has the info they need) or it might mean nobody is reading it. Check by asking: "Did the status update have the information you needed?"

## Cross-References

- `leadership-and-collaboration/managing-stakeholders.md` — Stakeholder communication plans
- `leadership-and-collaboration/communicating-with-executives.md` — Executive communication
- `leadership-and-collaboration/handling-blockers.md` — Blocker identification and escalation
- `leadership-and-collaboration/escalation-strategies.md` — Escalation strategies
- `engineering-culture/async-communication.md` — Async status updates
- `templates/stakeholder-update-template.md` — Stakeholder update template
