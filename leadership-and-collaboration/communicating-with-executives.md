# Communicating with Executives

> "Executives don't need less information — they need the right information, compressed."

## The Executive Communication Challenge

Senior engineers often struggle with executive communication because the skills that make you a good engineer (depth, thoroughness, technical precision) are the opposite of what executives need (brevity, clarity, business impact).

**Engineer's instinct:** "Let me explain the architecture so they understand why this is hard."
**Executive need:** "Should we invest in this? What's the risk? What do you need from me?"

## What Executives Care About

When communicating with executives (VP, SVP, CTO, CIO), they are thinking about:

| Question | Why They Care |
|----------|--------------|
| **Does this support our strategy?** | Their job is strategic alignment |
| **What's the risk?** | Their job is managing organizational risk |
| **What does it cost?** | Budget is always constrained |
| **When will it be done?** | They have dependencies and commitments |
| **What do you need from me?** | They want to help but need specific asks |
| **How does this compare to competitors?** | Market positioning matters |

They are NOT thinking about:
- Which framework you chose
- How the RAG pipeline works
- Why Kubernetes is better than VMs
- The details of the prompt injection vulnerability

## The Pyramid Principle

Start with the conclusion. Then support it.

```
BAD (Bottom-Up):
"We've been working on the RAG pipeline and we noticed that
the embedding model we're using has some accuracy issues with
financial documents. We evaluated three alternative models
and ran benchmarks on our document set. The new model from
Anthropic performs 15% better. We'd need to change our API
integration and re-run the embedding pipeline, which takes
about 3 days. So we think we should switch."

GOOD (Top-Down - Pyramid):
"We should switch our RAG embedding model to improve accuracy
by 15%. The change costs 3 engineering days and has no user
impact. I recommend we make the switch this sprint.

Here's why: Our current model struggles with financial
terminology, causing relevant documents to be missed in 18%
of queries. We benchmarked three alternatives and Anthropic's
new embedding model is the clear winner.

The switch requires updating our API integration (2 days)
and re-running the embedding pipeline (1 day, automated).
No user-facing changes are needed.

Should I proceed?"
```

## Communication Formats

### The 30-Second Update (Elevator Pitch)

```
Situation: [Context in one sentence]
Progress: [What's happened]
Ask/Risk: [What they need to know or do]

Example:
"The GenAI compliance assistant is on track for the May launch.
We completed security review and are starting compliance review
this week. The only risk is the compliance review timeline — if
it takes longer than 2 weeks, we'll slip the launch date. I'd
like your help ensuring compliance has the bandwidth to review
by April 20."
```

### The 1-Page Summary

For written executive updates:

```markdown
# GenAI Platform — Monthly Update — April 2026

## Bottom Line
On track for Q2 launch. Security review complete. Compliance
review in progress. One risk: model API capacity constraints.

## Progress
✅ Security review passed with zero open findings
✅ Performance targets met (p99 < 1s, 99.9% uptime)
✅ User acceptance testing started with HR pilot group (50 users)
⏳ Compliance review started April 5 (target completion: April 25)

## Risks and Mitigations
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Compliance review slips | Medium | High | Engaged compliance lead; escalated priority |
| Model API rate limits during launch | Low | Medium | Reserved capacity agreement with provider |
| User adoption lower than expected | Medium | Medium | Change management plan with comms team |

## Decisions Needed
1. Should we expand the pilot from HR to Finance in May?
   → Recommendation: Yes, Finance has high demand. Requires
     additional compliance review for financial documents.

## Resource Ask
- Need 2 additional engineers for Q3 scaling work (budget: $X)
- Need compliance team bandwidth for Finance expansion in May

## Key Metrics
| Metric | Current | Target | Trend |
|--------|---------|--------|-------|
| Uptime | 99.95% | 99.9% | ✅ |
| p99 Latency | 890ms | 1000ms | ✅ |
| User Satisfaction | 4.2/5 | 4.0/5 | ✅ |
| Query Volume (daily) | 2,340 | 5,000 | ⏳ |
```

### The Escalation

When you need to escalate to an executive:

```markdown
# Escalation: [Issue]

## What's Happening
[2-3 sentences describing the issue]

## Impact
[Who is affected and how. Quantify if possible.]

## What We've Tried
[Bullet points of resolution attempts]

## What We Need
[Specific ask: decision, resource, intervention]

## Recommended Action
[Your recommendation with rationale]

## Timing
[When this needs to be resolved by]

Example:
Escalation: Compliance Review Bottleneck

What's Happening:
The compliance review for the GenAI HR assistant has been
queued for 3 weeks with no start date. The review is on the
critical path for our May 15 launch date.

Impact:
If the review doesn't start by April 20, the launch will slip
by at least 2 weeks. This affects the CHRO's Q2 initiative
commitment to the board.

What We've Tried:
- Contacted compliance team lead directly (no response)
- Engaged our engineering manager to follow up (queued behind
  3 other reviews)
- Proposed a streamlined review scope (under consideration)

What We Need:
Prioritization of our compliance review from the Head of
Compliance. This requires VP-level alignment.

Recommended Action:
Reach out to the Head of Compliance to prioritize the GenAI
HR assistant review for this week. The streamlined scope
(half the normal review surface) should make this feasible.

Timing:
Need prioritization decision by April 10 to maintain launch date.
```

## Presenting to Executives

### Preparation

1. **Know your audience.** Who will be in the room? What do they care about?
2. **Prepare the narrative first, slides second.** The story matters more than the design.
3. **Anticipate questions.** What will they ask? Have the answers ready (in an appendix).
4. **Practice out loud.** If you can't explain it in 5 minutes, you don't understand it well enough.
5. **Bring the data.** Every claim should be backed by numbers.

### Delivery

1. **Start with the bottom line.** Don't build suspense.
2. **Watch the clock.** If you have 15 minutes, plan for 10. Leave room for questions.
3. **Don't read slides.** They can read. Talk to the points.
4. **Handle questions directly.** Answer the question asked, not the question you prepared for.
5. **Say "I don't know" when you don't know.** Then commit to finding the answer.
6. **End with the ask.** What do you need from them?

### Handling Difficult Questions

| Question | Bad Response | Good Response |
|----------|-------------|---------------|
| "Why is this taking so long?" | "Well, the technical complexity is..." | "We underestimated the integration effort. We've since adjusted our plan and the new date is X." |
| "What if this fails?" | "It won't fail." | "Here are our top 3 failure modes and how we're mitigating each. The highest risk is X, and we're addressing it by Y." |
| "Why should we invest in this vs. [alternative]?" | "Because our approach is better." | "Great question. The alternatives are A and B. Here's the trade-off analysis. We recommend our approach because..." |
| "Are you sure?" | "Yes." | "Here's the data that gives us confidence. The main uncertainty is X, and we'll know more by Y." |

## Email Communication with Executives

### Subject Lines

Make the subject line actionable:

| Bad Subject | Good Subject |
|-------------|-------------|
| "Update" | "GenAI Platform — On Track for Q2 Launch — April Update" |
| "Question" | "Decision Needed: GenAI Model Provider Selection by Friday" |
| "Issue" | "Risk: Compliance Review May Delay GenAI Launch by 2 Weeks" |

### Email Structure

```
Subject: [Action needed / FYI]: [Topic] — [Key message]

Body:
Bottom line (1 sentence)
Context (2-3 sentences max)
Details (bulleted, linked to full doc if needed)
Ask (specific, with deadline)

Example:
Subject: Decision Needed: GenAI Embedding Model Change — Approve by Friday

Bottom line: Recommend switching embedding models to improve
RAG accuracy by 15%.

Context: Our current embedding model misses relevant documents
in 18% of financial queries. The alternative model (Anthropic
embeddings-v3) benchmarks 15% better on our document set.

Details:
- Switch requires 3 engineering days
- No user-facing changes needed
- Cost increase: $2K/month (within existing budget)
- Full benchmark: [link]

Ask: Approve the model change by Friday so we can implement
this sprint.
```

## What Not to Do

| Mistake | Why It's Bad |
|---------|-------------|
| **Surprising executives** | They should never learn about problems from someone else first |
| **Hiding bad news** | Bad news doesn't get better with time. Early = options. Late = crises. |
| **Over-explaining** | Executives will disengage. Then they won't have the context to help. |
| **Under-explaining** | "Everything is fine" with no data is not reassuring |
| **Jargon without translation** | "Our p99 is above SLO" → "Users are experiencing slow responses" |
| **Complaints without solutions** | Always bring options, not just problems |
| **Asking for things in meetings** | Executives should not be put on the spot in public forums. Pre-wire asks. |

## Building Executive Credibility Over Time

Executive credibility is built through consistent behavior:

| Behavior | Impact |
|----------|--------|
| Always tell the truth, especially when it's uncomfortable | They trust your reporting |
| Bring solutions, not just problems | They trust your judgment |
| Follow through on commitments | They trust your word |
| Understand the business, not just the technology | They trust your priorities |
| Be concise and respect their time | They want to hear from you |
| Admit mistakes quickly and clearly | They trust your integrity |

## Cross-References

- `leadership-and-collaboration/managing-stakeholders.md` — Stakeholder management
- `leadership-and-collaboration/giving-status-updates.md` — Status update formats
- `leadership-and-collaboration/handling-blockers.md` — Blocker management
- `leadership-and-collaboration/escalation-strategies.md` — Escalation strategies
- `leadership-and-collaboration/writing-executive-summaries.md` — Executive summary writing
- `templates/executive-summary-template.md` — Executive summary template
