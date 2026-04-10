# Writing Strategy Documents

> "A strategy document is not a wish list. It's a coherent argument for a course of action, backed by evidence, that aligns a team around a shared plan."

## What Is a Strategy Document?

A strategy document answers three questions:

1. **Where are we now?** (Current state)
2. **Where do we want to be?** (Vision/goals)
3. **How do we get there?** (Plan/priorities)

It's different from:
- **Design doc:** How to build a specific system
- **RFC:** A lightweight proposal for discussion
- **Status update:** What's happening right now
- **Roadmap:** A timeline of deliverables

A strategy document is the **argument for why the roadmap is what it is.**

## When to Write a Strategy Document

| Situation | Why |
|-----------|-----|
| Planning a quarter or year | Align the team on priorities and trade-offs |
| Proposing a technical investment | Make the case for spending engineering time on infrastructure |
| Responding to a strategic shift | Explain how the team will adapt to new direction |
| Onboarding new leaders | Provide context on team direction and rationale |
| Cross-team alignment | Get multiple teams moving in the same direction |

## Strategy Document Structure

```markdown
# Strategy: [Title]
# Author: [Name, role]
# Date: [Date]
# Audience: [Who this is for]
# Review Status: [Draft / In Review / Approved]

## Executive Summary
[One paragraph: What is this strategy about, and what do you want
the reader to do or understand?]

## Current State

### What We Have
[Describe the current situation honestly and specifically]

### What's Working
[Acknowledge what's going well]

### What's Not Working
[Be candid about problems. Use data.]

### Why Now
[Why is this the right time for this strategy? What's changed?]

## Vision

### Where We're Going
[Describe the future state. Be specific about what success looks like.]

### Guiding Principles
[3-5 principles that will guide decisions. Examples:
 "Safety over speed," "Build on existing platform," "Automate everything"]

### What Success Looks Like
[Measurable outcomes. How will we know the strategy worked?]

## Strategic Priorities

### Priority 1: [Name]
**Why:** [Why this matters]
**What:** [What we'll do]
**How we'll measure it:** [Metrics]
**Timeline:** [When]
**Dependencies:** [What we need]
**Risks:** [What could go wrong]

### Priority 2: [Name]
[same structure]

### Priority 3: [Name]
[same structure]

### What We're NOT Doing
[Explicit non-priorities. This is as important as what we ARE doing.]

## Trade-offs and Risks

### Trade-offs
| Trade-off | What we gain | What we give up |
|-----------|-------------|-----------------|
| [Trade-off 1] | [Gain] | [Loss] |

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| [Risk 1] | [H/M/L] | [H/M/L] | [Mitigation] |

## Resource Requirements

| Resource | Current | Needed | Gap |
|----------|---------|--------|-----|
| Engineers | X | Y | Z |
| Budget | $X | $Y | $Z |
| Infrastructure | [Current] | [Needed] | [Gap] |

## Implementation Plan

### Phase 1: [Name] — [Timeline]
[What happens first]

### Phase 2: [Name] — [Timeline]
[What happens next]

### Phase 3: [Name] — [Timeline]
[What happens last]

### Milestones
| Milestone | Date | Success Criteria |
|-----------|------|-----------------|
| [Milestone 1] | [Date] | [How we'll know it's done] |

## Alternatives Considered
[What other strategies did you consider? Why did you reject them?]

## Appendix
[Supporting data, technical details, links to design docs, etc.]
```

## Real Example: GenAI Platform Strategy

Here's an excerpt from a real strategy document:

```markdown
# Strategy: GenAI Platform — Q2-Q3 2026

## Executive Summary

Our GenAI platform has proven product-market fit internally: 12,000
active weekly users, 4.3/5 satisfaction. But we're hitting scaling
limits: latency SLO breaches increased 3x in Q1, costs are 40% over
budget, and we can't onboard new use cases fast enough. This strategy
outlines our plan to rebuild the platform for scale, targeting 10x
user growth while reducing per-query cost by 50%.

## Current State

### What We Have
- Single-tenant architecture serving 12K weekly active users
- OpenAI as sole LLM provider
- pgvector-based RAG pipeline with 2M documents
- 99.7% uptime (target: 99.9%)
- Average cost per query: $0.012

### What's Working
- User satisfaction is high (4.3/5)
- Security and compliance reviews pass cleanly
- Team velocity is strong (average 2-week sprint delivery)

### What's Not Working
| Problem | Data | Impact |
|---------|------|--------|
| Latency SLO breaches | 15% of queries exceed 2s p99 (target: < 1s) | User frustration, adoption ceiling |
| Cost overruns | $48K/month vs. $30K budget | Unsustainable at current growth |
| Single provider risk | 100% dependent on OpenAI | Outages, pricing changes |
| Onboarding bottleneck | 6 weeks to onboard a new use case | Can't meet demand from 8 requesting teams |

### Why Now
- Q2 budget cycle allows infrastructure investment
- OpenAI's pricing change makes cost optimization urgent
- 8 teams are waiting to onboard use cases (revenue impact)
- Competitor banks are launching similar platforms — we have a
  6-month lead we should extend
```

## Writing Principles

### 1. Lead with the Problem

If the reader doesn't agree there's a problem, they won't care about your strategy.

```
Bad: "We propose rebuilding the RAG pipeline with a multi-provider
abstraction layer."

Good: "Our RAG pipeline costs $48K/month and is growing 15%
quarterly. At this rate, it will consume the entire GenAI budget
by Q4. We need to reduce per-query cost by 50% while handling
10x more documents."
```

### 2. Use Data, Not Adjectives

```
Bad: "Performance is bad and the system is slow."
Good: "p99 latency is 2.1s against a 1s SLO. 15% of queries
breach the SLO, up from 5% last quarter."

Bad: "Users love the platform."
Good: "User satisfaction is 4.3/5 from 1,247 respondents. NPS is 62."
```

### 3. Be Honest About Trade-offs

A strategy that claims there are no trade-offs is not credible.

```
Trade-offs:
1. Rebuilding for scale means pausing new feature development
   for 6 weeks. This delays the Finance team onboarding by 1 month.
2. Multi-provider support adds 4 weeks of engineering effort but
   reduces our single-provider risk and gives us pricing leverage.
3. Hiring 2 additional engineers is required to deliver this
   strategy on our timeline. Without them, the timeline extends
   by 8 weeks.
```

### 4. Make the "No" Clear

```
What We're NOT Doing:
- We're not onboarding new use cases until the platform rebuild
  is complete (estimated: 8 weeks)
- We're not evaluating new LLM providers beyond OpenAI and
  Anthropic during this period
- We're not building custom model fine-tuning (this is Q4 at
  earliest)
```

### 5. End with a Clear Ask

```
Resource Ask:
- 2 additional senior engineers (approved in Q2 budget)
- $50K for infrastructure (multi-provider API credits, additional
  Redis and Postgres capacity)
- Compliance team bandwidth for provider security review in April
- Security team engagement for threat model review in April

Decision Needed:
- Approve this strategy and resource allocation by April 15
```

## Strategy Document Review Process

### Draft Review

1. **Share with your team first.** Get internal alignment.
2. **Share with directly affected teams.** Get their input on assumptions and dependencies.
3. **Share with leadership.** Get their input on strategic alignment and resources.

### Review Feedback

Collect feedback in a structured way:

```markdown
# Review Feedback: [Strategy Title]

| Reviewer | Concern/Suggestion | Author Response | Resolution |
|----------|-------------------|-----------------|-----------|
| Security | Multi-provider review timeline is aggressive | Agreed. Adjusted from 2 to 4 weeks | Updated |
| Finance | Cost projections don't include training expenses | Good catch. Added $10K for training | Updated |
| Product | Pausing new features delays 3 commitments | True. Mitigation: communicate delays to affected teams | Will do |
```

### Approval

- **Team-level strategy:** Approved by engineering manager
- **Cross-team strategy:** Approved by director
- **Organization-level strategy:** Approved by VP/CTO

## Strategy Document Anti-Patterns

| Anti-Pattern | What It Looks Like | Why It Fails |
|--------------|-------------------|--------------|
| **The novel** | 30 pages of detail | Nobody reads it |
| **The slide deck** | Strategy as a presentation | Lacks the depth and permanence of a written document |
| **The wish list** | Goals with no plan or trade-offs | Not a strategy, just aspirations |
| **The technical deep-dive** | 20 pages of architecture details | Loses the strategic narrative |
| **The hidden document** | Written but never shared or discussed | No alignment if nobody reads it |
| **The set-and-forget** | Written once, never revisited | Strategy needs to evolve with reality |

## Strategy Document Length Guidelines

| Scope | Length | Reading Time |
|-------|--------|-------------|
| Team-level (quarterly) | 3-5 pages | 15 minutes |
| Cross-team (half-yearly) | 5-8 pages | 20 minutes |
| Organization-level (yearly) | 8-12 pages | 30 minutes |

**Rule:** If you can't explain your strategy in 5 pages, you don't understand it well enough. The appendix can contain supporting detail.

## Keeping Strategy Alive

A strategy document is useless if it sits in a folder. Keep it alive:

| Activity | Frequency | Purpose |
|----------|-----------|---------|
| Reference in sprint planning | Every sprint | Ensure work aligns with strategy |
| Review in monthly team meeting | Monthly | Track progress on milestones |
| Update leadership | Monthly | Maintain visibility and support |
| Revisit and revise | Quarterly | Adjust based on changing reality |
| Share with new team members | Onboarding | Provide context and direction |

## Cross-References

- `leadership-and-collaboration/writing-executive-summaries.md` — Executive summary writing
- `leadership-and-collaboration/building-alignment.md` — Getting teams aligned on strategy
- `engineering-culture/design-docs.md` — Design docs (more detailed, implementation-focused)
- `engineering-culture/rfcs.md` — RFCs (lighter-weight proposals)
- `leadership-and-collaboration/communicating-with-executives.md` — Executive communication
- `templates/executive-summary-template.md` — Executive summary template
