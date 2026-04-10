# Skill: Stakeholder Communication

## Core Principles

1. **Know Your Audience** — Engineers need technical details. Managers need timelines and risks. Executives need business impact and options. Never give the same message to everyone.
2. **Lead with the Bottom Line** — Start with the conclusion, then provide context. Busy stakeholders will not read five paragraphs to find the one sentence that matters.
3. **Bad News Early Is Better Than Bad News Late** — Stakeholders can handle delays and problems. They cannot handle surprises. Communicate risks as soon as you know them.
4. **Data Beats Opinion** — Back every claim with evidence. "The API is slow" is an opinion. "The API p99 latency is 3.2s, up from 0.8s last week" is a fact.
5. **Communication Is a Skill, Not a Trait** — Good communicators are not born that way. They practice. They get feedback. They iterate. Treat it like any other engineering skill.

## Mental Models

### The Stakeholder Communication Matrix
```
┌───────────────────────────────────────────────────────────────────────┐
│                  Stakeholder Communication Matrix                      │
│                                                                       │
│  Audience          What They Want          How to Communicate         │
│  ────────          ──────────────          ──────────────────         │
│  Engineering       Technical details,      Slack, PR comments,        │
│  Team              code, architecture      architecture docs          │
│                                                                       │
│  Engineering       Timelines, risks,       Sprint review, Slack,      │
│  Manager           blockers, dependencies  1:1 updates                  │
│                                                                       │
│  Product Manager   Feature status,         Sprint review, roadmap     │
│                    user impact, tradeoffs  review, written summary    │
│                                                                       │
│  Security Team     Threat model,           Security review doc,       │
│                    vulnerabilities,        Jira tickets, meeting      │
│                    compliance gaps                                    │
│                                                                       │
│  Compliance        Evidence, audit trail,  Compliance report,         │
│                    regulatory alignment    review meeting             │
│                                                                       │
│  VP / Director     Business impact,        Executive summary,         │
│                    cost, risk, options     1-page brief               │
│                                                                       │
│  C-Suite           Strategic alignment,    Verbal brief,              │
│                    financial impact,       3-bullet summary           │
│                    reputational risk                                  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### The BLUF Framework (Bottom Line Up Front)
```
BLUF: Bottom Line Up Front

Structure every communication as:

1. BLUF (1-2 sentences): The key message
2. Context (1 paragraph): What led to this
3. Details (as needed): Supporting data, technical details
4. Action requested (if any): What you need from the reader

Example:

BLUF: The GenAI assistant rollout will be delayed by 2 weeks due to
a security finding in the document retrieval layer.

Context: During our threat modeling review, the security team identified
that the RAG service does not enforce document-level access control.
A user could retrieve documents they are not authorized to see by
crafting specific search queries.

Details: The fix requires changes to the vector database query layer
to include user clearance checks. We estimate 5 days of development
and 3 days of testing. The change is behind a feature flag and can
be deployed independently of other work.

Action: I need your approval to adjust the sprint scope and delay
the rollout from March 15 to March 29. I've attached the updated
project plan for review.
```

### The Communication Checklist
```
□ Message is appropriate for the audience (technical level, focus)
□ BLUF is clear in the first sentence
□ Claims are backed by data (metrics, screenshots, logs)
□ Risks and uncertainties are communicated honestly
□ Options are presented with trade-offs, not just problems
□ Action items are explicit (who, what, when)
□ Tone is professional and constructive
□ Follow-up is scheduled if the situation is evolving
□ Sensitive information is shared through appropriate channels
□ Written communication is concise and scannable (headings, bullets)
```

## Step-by-Step Approach

### 1. Communicating a Project Delay

```markdown
# Email: GenAI Assistant Rollout — Schedule Update

**To:** Engineering Manager, Product Manager, Director of Engineering
**CC:** Security Team Lead
**Subject:** Update: GenAI Assistant Rollout Delayed by 2 Weeks

BLUF: The GenAI assistant v2.0 rollout, currently planned for March 15,
will move to March 29 due to a security finding that requires 8
additional days of development and testing.

## What Happened

During the scheduled threat modeling review on March 1, the security team
identified that the RAG service does not enforce document-level access
control at the database query layer. While the application layer checks
user permissions, a crafted query could bypass these checks and retrieve
unauthorized documents.

**Severity:** High (potential data exposure)
**Affected users:** All 100,000+ employees who would use the assistant
**Regulatory impact:** Possible compliance finding if deployed without fix

## The Fix

The fix adds user clearance enforcement at the PostgreSQL query level
using row-level security policies. This is a defense-in-depth measure
that ensures access control even if the application layer has a bug.

**Effort:** 5 days development + 3 days testing
**Risk:** Low — change is isolated to the retrieval layer
**Testing:** New integration tests + security team validation

## Updated Timeline

| Milestone | Original | Updated |
|-----------|----------|---------|
| Development complete | Mar 8 | Mar 15 |
| Security review | Mar 10 | Mar 18 |
| UAT deployment | Mar 12 | Mar 20 |
| Production rollout | Mar 15 | Mar 29 |

## Options Considered

1. **Deploy as-is, fix in next sprint** — Rejected. The security risk
   is too high for a production deployment.
2. **Partial rollout (internal users only)** — Possible, but the
   access control gap affects all users equally. No safe subset.
3. **Delay by 2 weeks** — Selected. Allows time for fix, testing,
   and security review without compromising quality.

## Next Steps

- [ ] Updated sprint plan attached — please review by EOD March 3
- [ ] Security team validation scheduled for March 18
- [ ] Stakeholder demo rescheduled to March 25
- [ ] I'll provide a weekly progress update every Friday

Happy to discuss any of this on a call. I'm available today from
2-4 PM or tomorrow from 10 AM-12 PM.

— [Your Name]
```

### 2. Communicating During an Incident

```markdown
# Slack: Incident Update (SEV-2)

**Channel:** #incident-2025-03-01-rag-errors
**Audience:** Engineering team, management, stakeholder channel

---

**Update 1 — Initial (14:30 UTC)**

SEV-2: RAG service error rate elevated at 8% (normal: < 0.1%).
Chat assistant returning errors for ~15% of requests.
Team is investigating. Next update in 30 minutes.

---

**Update 2 — Investigation (15:00 UTC)**

Root cause identified: Database read replica crashed.
RAG service cannot retrieve documents from the replica.
Mitigation: Switching back to primary database.
ETA for recovery: ~10 minutes.
No data loss or security impact.

---

**Update 3 — Resolved (15:15 UTC)**

RESOLVED: Service recovered at 15:10 UTC.
Error rate returned to normal (< 0.1%).
All synthetic tests passing.

Duration: 40 minutes.
Impact: ~15% of chat requests failed during this window.
No data loss. No security impact.

Postmortem will be completed by March 3.
Root cause summary and action items will follow.
```

### 3. Writing an Executive Summary

```markdown
# Executive Brief: GenAI Platform Q1 2025 Status

**Audience:** VP of Engineering, CTO
**Length:** 1 page
**Date:** March 31, 2025

---

## Key Highlights

- GenAI assistant now serves 45,000 daily active users (up from 12,000 in Q4)
- 99.95% availability achieved in Q1 (target: 99.9%)
- Average response time: 1.8s (target: < 3s)
- Zero security incidents
- Token cost per query reduced by 30% through prompt optimization

## Risks and Concerns

1. **Model dependency risk:** We rely on a single LLM provider (OpenAI).
   A service outage or pricing change would directly impact all users.
   Mitigation: Multi-provider integration in progress (ETA: Q2).

2. **Scaling pressure:** User growth is 3x quarterly. Current infrastructure
   supports up to 75,000 DAU. We will need to scale in Q2.
   Mitigation: Capacity planning exercise scheduled for April.

3. **Regulatory uncertainty:** The EU AI Act may impose new requirements
   on enterprise AI assistants. Our compliance team is monitoring.
   Mitigation: Quarterly compliance review with legal team.

## Budget and Resources

- Q1 spend: $180K (within budget of $200K)
- Primary cost driver: LLM API usage ($120K)
- Q2 forecast: $250K (based on projected user growth)
- Headcount: 8 engineers (2 open positions — recruiting in progress)

## Asks from Leadership

1. **Approval for Q2 budget increase** ($250K vs $200K) — driven by
   user growth, not inefficiency.
2. **Expedite recruiting** for 2 open backend positions — team is at
   80% capacity, velocity will drop if not filled by May.
3. **Cross-team alignment** with the Data Engineering team to improve
   document embedding pipeline throughput (currently a bottleneck).

## Next Quarter Focus

- Multi-model provider support (reduce single-provider risk)
- Embedding pipeline scale-up (10x throughput)
- Advanced analytics dashboard for usage and cost tracking
- Compliance review and AI Act readiness assessment
```

### 4. Communicating a Technical Trade-Off

```markdown
# ADR: Database Choice for GenAI Platform

**Status:** Accepted
**Date:** 2025-02-15
**Author:** @lead-engineer
**Stakeholders:** Engineering, Security, Compliance

## Context

We need a vector database for our RAG retrieval system. The team
has evaluated three options:

## Options

### Option A: PostgreSQL with pgvector (Recommended)
**Pros:**
- Already in our stack — no new technology to operate
- Supports row-level security (banking compliance requirement)
- ACID transactions for document metadata
- Team expertise in Postgres
- Audit logging already implemented

**Cons:**
- Vector search performance is slower than dedicated solutions
- Limited to ~1M vectors per node before performance degrades
- IVFFlat index requires periodic rebuilding

**Cost:** $0 additional (existing Postgres infrastructure)

### Option B: Pinecone (Managed Vector DB)
**Pros:**
- Best-in-class vector search performance
- Auto-scaling, fully managed
- No operational overhead

**Cons:**
- External SaaS — data leaves the banking network (compliance blocker)
- No row-level security
- Additional vendor to review and approve
- Vendor lock-in risk

**Cost:** $15K/month at projected scale

### Option C: Milvus (Self-Hosted)
**Pros:**
- High performance for large vector collections
- Open-source, no vendor lock-in
- Runs on-premise (data residency compliant)

**Cons:**
- New technology — no team expertise
- Significant operational overhead
- No native row-level security (must build access control layer)
- Additional infrastructure to manage and monitor

**Cost:** $30K (infrastructure) + 2 engineer-months to set up

## Decision

PostgreSQL with pgvector is the recommended choice.

The performance limitations are acceptable for our current scale
(500K documents, 500 queries/second). If we exceed pgvector's
capacity, we can partition across multiple Postgres nodes.

Compliance is the deciding factor: pgvector runs within our existing
Postgres infrastructure, which is already approved for banking workloads.
Pinecone is a compliance non-starter. Milvus would require 6+ months
of security review and operational setup.

## Consequences

- We accept slower vector search performance in exchange for compliance
  and operational simplicity.
- We must monitor pgvector performance and have a migration plan
  ready if we approach capacity limits.
- We save $180K/year vs. Pinecone and avoid a new operational burden.
```

### 5. Communicating Up: Status Report Template

```markdown
# Weekly Status Report — GenAI Platform Team

**Week:** March 3-7, 2025
**Team Lead:** @your-name

## This Week's Accomplishments
- Deployed RAG service v2.3.1 to production (zero incidents)
- Completed security review — all findings addressed
- Reduced token cost per query by 15% through prompt optimization
- Onboarded 3 new departments (HR, Legal, Compliance)

## Key Metrics
| Metric | This Week | Last Week | Trend |
|--------|-----------|-----------|-------|
| DAU | 45,000 | 42,000 | ↑ 7% |
| Availability | 99.97% | 99.95% | ↑ |
| p99 Latency | 1.8s | 2.1s | ↓ (improved) |
| Token cost/query | $0.012 | $0.014 | ↓ 14% |
| User satisfaction | 4.2/5 | 4.1/5 | ↑ |

## Risks and Blockers
- **Risk:** Embedding pipeline processing rate is 20% below target.
  May delay document freshness SLA. Mitigation: Adding worker pods.
- **No blockers.**

## Next Week's Plan
- Begin multi-provider integration (OpenAI + Anthropic)
- Embedding pipeline scale-up (target: 10K docs/hour)
- Sprint planning for Q2 roadmap
- Compliance review meeting with legal team

## Asks
- Need decision on Q2 budget allocation by March 14
- Need help prioritizing between multi-provider support and embedding
  pipeline scale-up — both are on the critical path for Q2
```

### 6. Communicating Down: Translating Strategy to the Team

```markdown
# Team Update: Q2 Priorities and Context

**From:** @your-name (Engineering Manager)
**To:** GenAI Platform Team
**Date:** March 31, 2025

---

Hi team,

I want to share the Q2 priorities and the context behind them.
I know we've all been heads-down on the v2.0 rollout, and I want
to make sure we're aligned on what comes next.

## The Big Picture

Leadership is happy with the GenAI assistant. User adoption is
exceeding expectations (45K DAU vs. 20K target). The conversation
has shifted from "Can we build it?" to "Can we scale it safely?"

This means our Q2 focus is **reliability and risk reduction**, not
new features.

## Q2 Priorities (in order)

1. **Multi-model provider support** — We currently depend on OpenAI
   alone. If they have an outage, we have an outage. If they raise
   prices, our budget explodes. This is our #1 risk.

2. **Embedding pipeline at scale** — We're processing 5K docs/hour.
   We need 50K. The current pipeline won't handle this. We need to
   redesign it for horizontal scaling.

3. **Observability and alerting** — We had 2 incidents in Q1 that
   were detected by users before our monitoring caught them. This
   is unacceptable. We need better synthetic tests and SLO alerting.

4. **Compliance readiness** — The legal team needs us to demonstrate
   that our AI assistant meets the requirements of the EU AI Act.
   This is a "must-do" even though it's not technically exciting.

## What We're NOT Doing

- No new user-facing features in Q2 (except critical requests)
- No experimental projects (we'll revisit in Q3)
- No platform rewrites (we improve what we have)

## What This Means for Each of You

I'll discuss individual assignments in our 1:1s this week. Broadly:
- Backend engineers: Multi-provider integration and embedding pipeline
- Frontend engineers: Observability dashboard and error handling
- Platform engineers: OpenShift scaling and monitoring setup

## Questions?

I know this is a lot. Please come to me with any questions or concerns.
I'm especially interested in hearing about:
- Technical concerns I might be missing
- Capacity estimates for each priority
- Anything that would make you more effective

Thanks for all the great work on v2.0. Let's make Q2 about building
something that scales.

— [Your Name]
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Over-communicating to engineers | Wastes time, creates noise | Right-size the message for the audience |
| Under-communicating to executives | Loss of trust, surprise delays | BLUF format, regular cadence |
| Hiding bad news | Makes the problem worse, destroys trust | Communicate early, with options |
| Leading with technical details | Executives tune out | Lead with business impact |
| Presenting problems without options | Creates helplessness, not action | Always present 2-3 options with trade-offs |
| Using jargon with non-technical stakeholders | Confusion, misalignment | Translate technical terms to business impact |
| Not following up on action items | Promises broken, trust eroded | Track commitments, follow up visibly |
| Communicating sensitive info in public channels | Information leak, compliance risk | Use appropriate channels (private, encrypted) |
| Email for urgent matters | Slow response time | Use phone/Slack for urgent, email for records |
| No written record of decisions | Decisions forgotten or disputed | Document decisions in writing, even after verbal discussions |

## Banking-Specific Concerns

1. **Regulatory Communication** — When communicating with compliance teams, be precise. Regulatory requirements are not "guidelines" — they are mandates. Use the regulator's exact language when referencing requirements.
2. **Audit Communication** — During audits, provide exactly what is requested — no more, no less. Do not volunteer extra information that could be misinterpreted. Coordinate with the legal team.
3. **Executive Escalation** — In banking, issues escalate quickly to senior leadership. Be prepared to explain technical issues in business terms at any time.
4. **Cross-Functional Alignment** — Banking projects involve many teams (security, compliance, legal, risk, operations). Communication must be consistent across all groups.
5. **Confidentiality** — Banking project details may be confidential. Know what can be shared with whom and through which channels.

## GenAI-Specific Concerns

1. **Explaining AI Limitations** — Non-technical stakeholders may overestimate what GenAI can do. Communicate limitations clearly: "The assistant provides information based on documents; it does not make decisions or give advice."
2. **Cost Transparency** — GenAI costs are usage-based and unpredictable. Communicate cost projections with ranges, not fixed numbers. Explain the drivers (user count, query complexity).
3. **Quality Metrics** — Explain GenAI quality in terms stakeholders understand: "95% of responses are grounded in source documents" is better than "grounding score is 0.95."
4. **Risk Communication** — GenAI-specific risks (hallucination, prompt injection, data leakage) must be communicated to security and compliance teams in terms they understand: impact, likelihood, and mitigation.
5. **Regulatory Change** — AI regulations are evolving. Keep leadership informed about regulatory changes that could impact the platform.

## Metrics to Monitor

| Metric | Target | Why It Matters |
|--------|--------|----------------|
| Stakeholder satisfaction | > 4/5 in surveys | Communication effectiveness |
| Decision turnaround time | < 3 business days | Stakeholders making timely decisions |
| Escalation frequency | Track trend | Are issues being resolved at the right level? |
| Status report consistency | Weekly, on time | Communication reliability |
| Action item completion rate | > 90% | Commitment follow-through |
| Meeting efficiency | < 30 min, clear agenda | Respecting stakeholders' time |
| Written communication clarity | Feedback score > 4/5 | Messages are understood |

## Interview Questions

1. How would you communicate a 2-week project delay to your VP?
2. How do you explain a technical trade-off to a non-technical stakeholder?
3. What would you include in an incident update to the compliance team?
4. How do you handle a stakeholder who keeps asking for status updates?
5. How would you present a security finding that requires delaying a product launch?
6. How do you communicate team capacity constraints to a product manager who wants more features?
7. What is BLUF and when would you use it?
8. How do you adapt your communication style for engineers vs. executives?

## Hands-On Exercise

### Exercise: Communicate a Critical Security Finding to Leadership

**Problem:** You are the lead engineer on the GenAI platform. During a routine security review, your team discovered that the LLM provider (OpenAI) may be logging prompts that contain sensitive internal banking information. This includes:

- Employee names and IDs mentioned in queries
- Internal policy document content
- Department-specific operational details

The data is logged by OpenAI for "service improvement" purposes under their standard terms of service. Your banking data classification policy prohibits sending restricted data to external logging systems.

**Context:**
- The GenAI assistant has 45,000 daily active users
- The compliance team is not yet aware
- The engineering VP expects the multi-provider integration to be demoed next week
- The fix requires either: (a) a data processing agreement with OpenAI, (b) a proxy that redacts sensitive data before sending, or (c) switching to an on-premise model

**Constraints:**
- You need to communicate this to: your engineering manager, the VP of Engineering, and the security team
- Each audience needs different information
- The situation is sensitive and time-critical
- You must propose solutions, not just problems

**Expected Output:**
- A BLUF-format email to the VP of Engineering
- A technical briefing for the engineering manager
- A security incident report for the security team
- A proposed action plan with options, timelines, and trade-offs

**Hints:**
- The VP needs: business impact, options, recommended path, timeline
- The engineering manager needs: technical details, effort estimate, team impact
- The security team needs: what data was exposed, scope, regulatory implications
- Be honest about the severity but don't panic — this is a known risk of external LLM APIs

**Extension:**
- Draft the public-facing communication if this becomes a regulatory issue
- Design a technical architecture for the data redaction proxy
- Create a risk register entry for this finding with likelihood, impact, and mitigation

---

**Related files:**
- `leadership-and-collaboration/stakeholder-management.md` — Stakeholder management guide
- `leadership-and-collaboration/executive-communication.md` — Executive communication
- `incident-management/incident-communication.md` — Incident communication
- `engineering-culture/documentation-culture.md` — Writing culture
- `skills/code-review-best-practices.md` — Code review communication
