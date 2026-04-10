# Solutions-First Mindset

> **Problem-first thinking. Outcome orientation. Measuring impact.**
> **Audience:** All engineers on the GenAI Platform team
> **Owner:** Engineering Leadership

---

## Core Principle

**The best engineers do not start by writing code. They start by understanding the problem.**

In consumer software, you can afford to build first and measure later. In a global bank, a GenAI platform serving hundreds of thousands of employees, every line of code you write has downstream consequences: regulatory implications, security review requirements, operational overhead, and real financial risk.

The solutions-first mindset is not a natural talent. It is a **learned discipline** that separates senior engineers from junior ones.

---

## What Problem-First Thinking Looks Like

### The Wrong Way

```
Engineer: "We need to add LangGraph to our RAG pipeline."
Manager: "Why?"
Engineer: "Because it's the latest thing. Everyone is using it."
```

This is **technology-first thinking**. It is the most expensive way to build software. You end up with a solution looking for a problem, and you will retrofit requirements to justify your investment.

### The Right Way

```
Engineer: "Our RAG retrieval accuracy is 62% for compliance documents.
           Users are complaining that the assistant gives incomplete answers
           for regulatory queries. I analyzed the failure modes and found
           that single-pass retrieval misses multi-hop relationships
           between policy documents. I propose we implement a multi-step
           reasoning approach. LangGraph is one option, but we should
           also evaluate a simpler iterative refinement loop before
           committing to a new framework."
```

This is **problem-first thinking**. It starts with a measured problem, proposes a measurable outcome, and considers multiple solutions.

---

## The Problem Statement Framework

Before writing any code for a non-trivial change, write a problem statement:

```markdown
## Problem Statement

**What is happening:**
- RAG retrieval accuracy for compliance documents is 62%
- User satisfaction score for compliance queries: 3.2/5.0
- 18% of compliance queries result in "I don't know" or incomplete answers

**Why it matters:**
- Compliance team relies on this assistant for regulatory interpretation
- Incomplete answers could lead to incorrect compliance decisions
- Regulatory risk if employees act on partial information

**Who is affected:**
- 2,400 compliance officers across 14 business units
- Legal review team (42 people)

**Current state:**
- Single-pass dense vector retrieval against pgvector
- No re-ranking step
- No multi-hop document linking

**Desired state:**
- Retrieval accuracy > 85% for compliance documents
- User satisfaction > 4.0/5.0
- Incomplete answer rate < 5%

**Constraints:**
- Cannot change the embedding model without 4-week security review
- Must work within existing OpenShift resource quotas
- Must maintain current latency SLO (p95 < 3s)
```

### Checklist: Is This a Well-Formed Problem?

- [ ] Can you quantify the current state with data?
- [ ] Can you identify who is affected and how?
- [ ] Can you describe the impact of NOT solving this?
- [ ] Have you considered the constraints?
- [ ] Can you define what "solved" looks like with numbers?

If you cannot check all of these, you do not understand the problem yet. **Do not write code.**

---

## Outcome Orientation

### Output vs. Outcome

```
OUTPUT:  "We shipped LangGraph integration."
OUTCOME: "Compliance query accuracy improved from 62% to 87%,
          reducing compliance team escalation tickets by 34%."
```

Banks reward outcomes. Nobody gets promoted for shipping technology. You get promoted for **changing measurable business metrics**.

### The Outcome Ladder

```
Level 0: Activity        "I worked on the RAG pipeline this sprint."
Level 1: Output          "I shipped LangGraph integration."
Level 2: Outcome         "Retrieval accuracy improved from 62% to 87%."
Level 3: Impact          "Compliance escalation tickets reduced by 34%,
                          saving an estimated 240 engineering hours/month."
Level 4: Business Value  "Reduced regulatory risk exposure. Avoided
                          potential fine of $2-5M based on OCC precedent."
```

Most engineers operate at Level 1. Staff engineers operate at Level 3. Principal engineers operate at Level 4.

**The goal is not to reach Level 4 for everything.** Some changes are Level 1 or 2, and that is fine. But you must **know which level you are working at** and communicate accordingly.

---

## Measuring Impact

### What Gets Measured Gets Improved

In our GenAI platform, we measure everything that matters:

```python
# Example: RAG Pipeline Impact Metrics
from dataclasses import dataclass
from datetime import datetime

@dataclass
class RagImpactMetrics:
    """Metrics that matter when changing the RAG pipeline."""

    # Quality metrics
    retrieval_precision: float       # % of retrieved docs that are relevant
    answer_accuracy: float           # % of answers rated correct by human review
    hallucination_rate: float        # % of answers containing fabricated facts
    citation_coverage: float         # % of answers with valid source citations

    # User impact metrics
    user_satisfaction: float         # Average user rating (1-5)
    repeat_usage_rate: float         # % of users who return within 7 days
    escalation_reduction: float      # % reduction in human escalation tickets

    # Operational metrics
    p50_latency_ms: float            # Median response time
    p95_latency_ms: float            # Tail latency
    cost_per_query: float            # Total cost / total queries
    token_efficiency: float          # Useful output tokens / total tokens

    # Risk metrics
    pii_leak_rate: float             # Incidents of PII in responses (target: 0)
    prompt_injection_blocks: int     # Number of blocked injection attempts
    safety_flag_rate: float          # % of responses flagged by safety filter
```

### Real Example: How We Measured the Impact of a Guardrails Change

> **Situation (Q3 2025):** The security team requested that we add a new layer of output filtering to detect financial advice hallucinations. The concern: the assistant was occasionally generating plausible-sounding but incorrect investment guidance.
>
> **Before the change:**
> - Hallucination rate on financial advice queries: 4.2%
> - Average latency added by existing guardrails: 120ms
> - User satisfaction: 4.1/5.0
>
> **After adding the new guardrail:**
> - Hallucination rate: 0.8% (81% reduction)
> - Added latency: 185ms total (65ms from new guardrail)
> - User satisfaction: 4.3/5.0 (improved because answers were more reliable)
> - False positive rate: 2.1% (some legitimate advice was being blocked)
>
> **The key insight:** We did not just ship the guardrail and walk away. We measured the before and after, found the false positive rate was too high, and tuned the detection rules. Without measurement, we would not have known about the false positives.

---

## The Five Whys for Problem Discovery

When a problem is reported, do not accept the first statement. Dig deeper.

```
Reported Problem: "The GenAI assistant is too slow."

Why?         -> "It takes 8 seconds to respond to compliance queries."
Why?         -> "The retrieval step is searching 50,000 documents."
Why?         -> "We never scoped the compliance knowledge base separately."
Why?         -> "The initial design assumed one unified knowledge base."
Why?         -> "Nobody anticipated compliance would become the largest user."

Root Cause:  The platform was designed for scale, not for workload isolation.
             The solution is NOT "make it faster." The solution is
             "partition knowledge bases by domain and implement query routing."
```

The solution to "make it faster" might be caching, better hardware, or query optimization. The solution to "partition knowledge bases" is an architecture change. These are very different investments.

---

## Banking Context: Why Solutions-First Matters More Here

In a startup, you can pivot. In a global bank:

- **Architecture changes require months of review.** A bad solution wastes months, not weeks.
- **Compliance reviews are triggered by changes.** Every unnecessary change is an unnecessary compliance overhead.
- **Resource quotas are fixed.** You cannot spin up more GPUs because a solution is inefficient.
- **Vendor contracts are locked.** You cannot switch embedding providers on a whim.

**Every solution you propose must be justified against these constraints.**

---

## What Good Looks Like vs. What Bad Looks Like

### Good

| Behavior | Example |
|----------|---------|
| Starts with data | "Our error rate is 3.2%, here is the trend over 90 days." |
| Quantifies impact | "This will save approximately 120 engineering hours/month." |
| Considers alternatives | "We evaluated options A, B, and C. Here is why A wins." |
| Defines success | "We will know this works when X metric moves from Y to Z." |
| Acknowledges risk | "The downside is N. Here is how we mitigate it." |

### Bad

| Behavior | Example |
|----------|---------|
| Starts with technology | "We should use LangGraph." |
| Vague benefits | "This will make things better." |
| No alternatives considered | "This is the only way to solve it." |
| No success criteria | "We will iterate and see." |
| Ignores risk | "It is simple, what could go wrong?" |

---

## The One-Page Proposal Template

For any change that affects other teams or requires approval, write a one-page proposal:

```markdown
# Proposal: [Title]

## Problem
[2-3 sentences with data]

## Proposed Solution
[1-2 paragraphs describing the approach]

## Alternatives Considered
| Option | Pros | Cons | Why Rejected |
|--------|------|------|-------------|
| ... | ... | ... | ... |

## Impact
- **Users affected:** [who and how many]
- **Metric change:** [what moves, by how much]
- **Cost:** [compute, engineering time, compliance overhead]

## Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| ... | ... | ... | ... |

## Rollout Plan
- Phase 1: [what, when, who]
- Phase 2: [what, when, who]
- Rollback plan: [how to undo if things break]

## Success Criteria
- [ ] Metric X improves from Y to Z
- [ ] User satisfaction > N
- [ ] Zero P1/P2 incidents related to this change

## Approvers Needed
- [ ] Engineering Manager
- [ ] Security Review (if handling sensitive data)
- [ ] Compliance (if affecting regulated workflows)
```

---

## Cross-References

- **Bias for Action** (`bias-for-action.md`) — Once you understand the problem, how fast should you act?
- **Pragmatism vs. Technical Excellence** (`pragmatism-vs-technical-excellence.md`) — When is the "right" solution worth the investment?
- **Thinking in Systems** (`thinking-in-systems.md`) — Understanding second-order effects of your solution
- **Clear Communication** (`clear-communication.md`) — How to write up your problem analysis and proposals

---

## Interview Preparation

### Questions You Might Be Asked

1. **"Tell me about a time you solved a problem that nobody else saw coming."**
   - Use the problem statement framework. Show data-driven thinking.

2. **"How do you decide what to build vs. buy vs. ignore?"**
   - Walk through your evaluation framework. Mention constraints specific to banking.

3. **"Describe a time you pushed back on a proposed solution."**
   - Show how you reframed the conversation from technology to problem.

4. **"How do you measure the impact of your work?"**
   - Discuss outcome metrics, not output metrics. Use the Outcome Ladder.

### STAR Story Template

```
Situation:  "Our GenAI assistant was generating hallucinated compliance
             answers at a 4.2% rate. This was a regulatory risk."

Task:       "I needed to reduce hallucination rate below 1% without
             adding more than 100ms of latency."

Action:     "I analyzed the failure modes, found that 73% of hallucinations
             came from a single low-quality document source. I proposed
             adding an output guardrail AND removing the bad source. I
             measured before/after, set up continuous monitoring, and
             defined rollback criteria."

Result:     "Hallucination rate dropped to 0.8%. User satisfaction improved
             from 4.1 to 4.3. The compliance team reduced escalation tickets
             by 34%. The guardrail is now a standard component in our pipeline."
```

---

## Summary

1. **Start with the problem, not the solution.** Write it down. Quantify it.
2. **Focus on outcomes, not outputs.** Measure what matters.
3. **Define success before you start.** Know when to stop.
4. **Consider alternatives.** The first idea is rarely the best one.
5. **Measure everything.** Without data, you are guessing.
6. **In banking, the cost of a wrong solution is amplified.** Be thorough.

> "The most dangerous kind of progress is making fast progress in the wrong direction."
>
> The solutions-first mindset is how you ensure you are heading in the right direction before you start running.
