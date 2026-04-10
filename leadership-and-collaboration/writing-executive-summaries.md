# Writing Executive Summaries

> "An executive summary is not a summary of a document. It's a standalone document that lets a busy executive understand the situation and make a decision in 2 minutes."

## What Is an Executive Summary?

An executive summary is a **self-contained, decision-ready document** that:

- Communicates the essential information from a longer document or situation
- Allows the reader to understand the issue and make a decision without reading the full document
- Stands alone — if the attachment is lost, the summary is still actionable

**It is NOT:**
- A table of contents
- A chapter-by-chapter summary
- A teaser that requires reading the full document
- A collection of bullet points without narrative flow

## When to Write One

| Situation | Why |
|-----------|-----|
| Escalating to VP/CTO | They don't have time for the full context |
| Monthly/quarterly updates | Busy leaders need the compressed version |
| Before a meeting | Sets context so meeting time is used for decisions, not briefings |
| After an incident | Quick summary for leadership before the full postmortem |
| Proposing a significant investment | Leaders need the business case quickly |

## The Structure

### The 5-Paragraph Executive Summary

```
Paragraph 1: SITUATION
What is happening? What is the context?
(1-2 sentences)

Paragraph 2: COMPLICATION
What is the problem, risk, or opportunity?
(1-2 sentences)

Paragraph 3: IMPLICATION
Why does this matter? What is the impact?
(1-2 sentences, with data)

Paragraph 4: RECOMMENDATION
What should we do? What is the ask?
(1-2 sentences)

Paragraph 5: NEXT STEPS
What happens next? What decision is needed?
(1-2 sentences)
```

### Example: Budget Increase Request

```markdown
# Executive Summary: GenAI Platform Budget Adjustment Request

## Situation
The GenAI platform has grown from 2,000 to 12,000 weekly active
users in 6 months — 6x our original projection. User satisfaction
remains high at 4.3/5, and 8 additional teams have requested
onboarding.

## Complication
This growth has driven costs 40% over budget. Our current monthly
run rate is $48K against a $30K budget, primarily due to LLM API
usage. At current growth rates, we will exceed the annual budget
by Q3.

## Implication
Without a budget adjustment, we will need to throttle usage in
Q3, which would impact 12,000 employees and delay onboarding of
8 teams (estimated $500K in productivity savings). Alternatively,
we can reduce per-query cost by 50% through architectural changes
(multi-provider routing, response caching, query optimization),
but this requires a one-time $80K infrastructure investment.

## Recommendation
Approve a Q3-Q4 budget increase of $110K ($80K infrastructure
investment + $30K additional operating costs through year-end).
The infrastructure investment will reduce per-query cost by 50%,
bringing the platform back within budget by Q1 2027 at current
usage levels.

## Decision Needed
Budget approval of $110K for Q3-Q4 by May 15 to begin
infrastructure work before peak usage season. Supporting
detail: [linked strategy document, 4 pages].
```

### Example: Incident Escalation

```markdown
# Executive Summary: GenAI Service Degradation — April 8, 2026

## Situation
At 14:32 UTC, the GenAI chat service began experiencing elevated
error rates. Approximately 30% of user queries are returning errors
instead of responses. The issue was detected by monitoring and the
on-call team is actively working on resolution.

## Complication
Root cause has been traced to the external LLM API provider, which
is experiencing degraded performance in the US-East region. The
provider has not yet provided an ETA for resolution.

## Implication
12,000 weekly active users are affected. The service is degraded
but not fully down — users can still access cached responses and
some queries succeed. No data loss or security impact. This is
the second provider-related degradation in 30 days.

## Action Taken
- Incident declared at 14:45 UTC (P2)
- Provider escalation filed; their engineering team is investigating
- Evaluating failover to secondary provider (Anthropic) — estimated
  2-hour switchover time
- User communication posted on internal status page

## Decision Needed
Should we proceed with provider failover? Pros: restores full
service. Cons: 2-hour switchover, Anthropic has 15% lower accuracy
on our document set. Recommendation: wait 1 hour for provider
resolution, then failover if unresolved.
```

## Writing Principles

### 1. The "BLUF" Principle: Bottom Line Up Front

The first sentence should tell the reader everything they need to know:

```
BLUF examples:

"We need a $110K budget increase to keep the GenAI platform
on track for Q3."

"The GenAI service is degraded due to a provider outage. We're
monitoring and have a failover option ready."

"I recommend we hire 2 additional engineers to deliver the Q2
GenAI platform rebuild on schedule."
```

### 2. One Document, One Purpose

Each executive summary should focus on a single decision or situation. Don't combine multiple unrelated topics.

```
Bad: Combining budget request, hiring plan, and incident report
     in one summary.

Good: Three separate summaries, each with a clear focus and ask.
```

### 3. Data-Driven, Not Emotion-Driven

```
Bad: "The situation is critical and we need immediate action."
Good: "30% of queries are failing. At current rates, we'll exceed
       budget by $180K this quarter."

Bad: "Users are very unhappy with the service."
Good: "User satisfaction dropped from 4.3 to 3.7/5 in the last
       two weeks."
```

### 4. Include the Ask

Every executive summary should end with a clear request:

| Type of Summary | The Ask |
|-----------------|---------|
| Proposal | "Approve X by [date]" |
| Escalation | "Decision needed: [options]. Recommendation: [X]" |
| Status update | "No action needed. FYI only." or "Awareness: [risk]" |
| Incident | "Decision needed: [options]. Recommendation: [X]" |

### 5. Keep It to One Page

If your executive summary is more than one page, it's not a summary — it's the full document.

**Length targets:**
- **Budget/resource requests:** 1 page + supporting attachment
- **Incident escalations:** Half page
- **Status updates:** Half page
- **Strategy summaries:** 1-2 pages + full document attachment

## Formatting Guidelines

```markdown
# Executive Summary: [Topic]

**Date:** [Date]
**Author:** [Name, role]
**Audience:** [Who this is for]
**Classification:** [Internal / Confidential]

## Summary
[5 paragraphs following the SCIRN structure above]

## Supporting Details
[Brief bullets with key data points]

## Decision / Ask
[Clear statement of what you need]

## Attachments
- [Full strategy document]
- [Budget breakdown]
- [Technical detail if relevant]
```

## Executive Summary Review Checklist

Before sending:

- [ ] Can the reader understand the situation without the attachment?
- [ ] Is the ask clear and specific?
- [ ] Is there a deadline for the decision?
- [ ] Are all claims backed by data?
- [ ] Is it one page or less?
- [ ] Would a busy executive be able to act on this in 2 minutes?
- [ ] Is the tone professional and factual (not emotional or alarmist)?
- [ ] Have I included my recommendation (not just options)?

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Too much detail** | Defeats the purpose of a summary | Move detail to attachments |
| **No clear ask** | Reader doesn't know what to do | End with a specific request |
| **No deadline** | Decisions drift | Include "by [date]" |
| **Multiple topics** | Confuses the reader | One summary per topic |
| **Jargon without explanation** | Non-technical executives can't understand | Explain acronyms and technical terms |
| **No recommendation** | Forces the executive to decide without guidance | Always include your recommendation |
| **Alarmist tone** | Creates panic, reduces credibility | Be factual and measured |
| **Optimistic bias** | Understating the problem destroys trust when reality hits | Be honest about severity |

## Adapting to the Audience

| Audience | Adjustments |
|----------|-------------|
| **CTO** | Include technical context, architecture implications |
| **CFO** | Focus on cost, ROI, budget impact |
| **COO** | Focus on operational impact, user impact, timeline |
| **CEO** | Focus on strategic alignment, competitive position, reputational risk |
| **General counsel** | Focus on regulatory, legal, and liability implications |

## Cross-References

- `leadership-and-collaboration/communicating-with-executives.md` — Executive communication
- `leadership-and-collaboration/writing-strategy-docs.md` — Strategy document writing
- `leadership-and-collaboration/escalation-strategies.md` — Escalation strategies
- `leadership-and-collaboration/giving-status-updates.md` — Status update formats
- `templates/executive-summary-template.md` — Executive summary template
- `incident-management/incident-communication.md` — Incident communication
