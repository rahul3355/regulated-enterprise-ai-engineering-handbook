# Executive Summary Template

> Use this template to communicate complex technical topics to executives in one page or less.

## Template

```markdown
# Executive Summary: [Topic]

**Date:** [YYYY-MM-DD]
**Author:** [Name, role]
**Audience:** [Executive name(s) or title(s)]
**Classification:** [Internal / Confidential]
**Reading Time:** 2 minutes

## Bottom Line

[One sentence. The most important thing the reader needs to know.
If they read nothing else, they should understand this.]

## Situation

[2-3 sentences of context. What is the current state? What prompted
this summary? Assume the reader knows nothing about the topic.]

## Complication

[2-3 sentences. What is the problem, risk, or opportunity? Quantify
the impact with numbers if possible.]

## Recommendation

[1-2 sentences. What are you proposing? Be specific about what you
want the reader to do or decide.]

## Supporting Points

[Bullet points with key data. 3-5 bullets maximum.]

- [Point 1 with data]
- [Point 2 with data]
- [Point 3 with data]

## Trade-offs

[Brief acknowledgment of trade-offs. Shows you've thought critically.]

- **We gain:** [Benefits of the recommendation]
- **We give up:** [Costs or risks of the recommendation]

## Ask

[Clear, specific request with deadline.]

- **What:** [What you need from the reader]
- **By when:** [Deadline]
- **If not:** [Consequence of inaction]

## Appendix

[Link to detailed document if the reader wants more information.]
```

## Example: Filled Executive Summary

```markdown
# Executive Summary: GenAI Platform Budget Adjustment

**Date:** 2026-04-07
**Author:** Sarah Chen, GenAI Platform Lead
**Audience:** VP of Engineering, CTO
**Classification:** Confidential
**Reading Time:** 2 minutes

## Bottom Line

We need a $110K budget increase for Q3-Q4 to keep the GenAI platform
operational and on track for 10x user growth.

## Situation

The GenAI platform has grown from 2,000 to 12,000 weekly active users
in 6 months — 6x our original projection. User satisfaction is 4.3/5,
and 8 additional teams have requested onboarding. The platform is
delivering measurable value across the organization.

## Complication

This growth has driven LLM API costs 40% over budget. Our current
monthly run rate is $48K against a $30K budget. At current growth
rates, we will exceed the annual budget by Q3. Without additional
funding, we must either throttle usage (impacting 12,000 employees)
or delay onboarding of 8 teams (estimated $500K in productivity savings).

## Recommendation

Approve a Q3-Q4 budget increase of $110K: $80K for infrastructure
investment (multi-provider routing, response caching) that will reduce
per-query cost by 50%, and $30K for additional operating costs through
year-end. The infrastructure investment pays for itself by Q1 2027.

## Supporting Points

- **Usage is growing 15% monthly** — we'll hit 50K weekly users by Q4
  if current trends continue
- **Multi-provider routing saves 30-40%** — by routing queries to the
  most cost-effective provider, we reduce costs without reducing quality
- **Response caching saves 20-25%** — 22% of queries are repeats that
  can be served from cache at near-zero cost
- **Combined effect: 50% cost reduction** — bringing per-query cost
  from $0.012 to $0.006, within the original budget at current usage
- **Investment pays back in 6 months** — the $80K infrastructure cost
  is recovered through savings by Q1 2027

## Trade-offs

- **We gain:** Sustainable costs, 10x scalability, multi-provider resilience
- **We give up:** 6 weeks of new feature development to build infrastructure
- **Risk:** Multi-provider routing adds complexity — mitigated by starting
  with 2 providers and a simple routing strategy

## Ask

- **What:** Approve $110K budget increase for Q3-Q4
- **By when:** May 15 (start of Q3 budget cycle)
- **If not:** Must throttle usage in Q3, impacting 12,000+ employees
  and delaying 8 team onboardings

## Appendix

- Full strategy document: [link, 4 pages]
- Cost analysis spreadsheet: [link]
- Multi-provider benchmark results: [link]
```

## Executive Summary Writing Tips

1. **Write it last.** Write the detailed document first, then distill it.
2. **Test the "elevator" rule.** If you can't communicate the essence in 30 seconds, it's not distilled enough.
3. **Use the reader's language.** Executives think in terms of strategy, risk, cost, and timeline — not technical details.
4. **Always include a recommendation.** Executives want your judgment, not just your analysis.
5. **Quantify everything.** "Significant improvement" means nothing. "30% cost reduction" means everything.
