# Knowledge Sharing

> "Knowledge that isn't shared is knowledge that leaves when the person does. Document everything, share constantly."

## Why Knowledge Sharing Is Non-Negotiable

In a banking GenAI platform, tribal knowledge is a **single point of failure**.

If only one person knows how the prompt injection detection system works, and they leave the team, you have:
- A security-critical system nobody understands
- Inability to debug incidents in that system
- Risk of breaking changes during unrelated work
- Bus factor of 1

We treat knowledge sharing as an **engineering responsibility**, not an optional extra.

## Knowledge Sharing Channels

### 1. Tech Talks

**Format:** 45-minute presentation + 15-minute Q&A, recorded.

**Frequency:** Bi-weekly, Thursdays at 3 PM.

**Audience:** All engineers (mandatory attendance, optional for those on P1/P0 incidents).

**Topics rotate through:**
- Deep-dives into production systems we've built
- Post-incident learnings (what went wrong, what we fixed)
- New technology evaluations (should we adopt X?)
- Banking domain education (Basel III, PSD2, etc.)
- Security threat walkthroughs
- GenAI research papers applied to our context

**Tech talk template:**

```markdown
# Tech Talk: [Topic]
# Presenter: [Name]
# Date: [Date]

## Context (5 min)
- Why are we talking about this?
- What problem does it solve?
- Who should care?

## Deep Dive (30 min)
- Architecture / design / code
- Trade-offs and decisions
- Production data and metrics

## Banking/GenAI Relevance (5 min)
- How does this apply to our platform?
- What are the regulatory implications?
- What are the security considerations?

## Q&A (15 min)
- Open questions
- Discussion
- Next steps

## Resources
- Links to docs, code, papers
- Recording link
```

### 2. Brown Bag Sessions

**Format:** 30-minute informal session during lunch, no recording.

**Frequency:** Weekly, Wednesdays at 12:30 PM.

**Topics:** Lighter, more exploratory than tech talks.
- "I tried building X over the weekend, here's what I learned"
- "Here's a cool paper I read about Y"
- "Demo of a side project"
- "Book club discussion"

**No preparation required.** The informality is the point.

### 3. Engineering All-Hands

**Format:** 60-minute session, monthly.

**Agenda:**
- Platform health dashboard review (10 min)
- Recent incidents and learnings (10 min)
- New feature demos (15 min)
- Architecture decision highlights (10 min)
- Open Q&A (15 min)

### 4. Documentation Sprints

**Format:** Dedicated time for documentation improvement.

**Frequency:** One week per quarter.

**Rules during doc sprint:**
- No new feature work
- Existing docs are reviewed and updated
- Missing docs are created
- Every engineer contributes at least one significant doc
- Docs are treated with the same rigor as code (reviewed, tested where possible)

## Documentation Standards

### What Must Be Documented

| Artifact | When | Where | Reviewer |
|----------|------|-------|----------|
| Architecture decisions | Before implementation | `architecture/` | Tech lead |
| API contracts | Before implementation | OpenAPI specs + docs | API team |
| Runbooks | Before going to production | `runbooks/` | On-call team |
| Incident reports | Within 48 hours of incident | `incidents/` | Engineering manager |
| Design documents | Before implementation | `designs/` | Peers + tech lead |
| On-call procedures | Before being on-call | On-call wiki | On-call lead |
| Deployment procedures | Before deployment | CI/CD docs | Platform team |
| Security reviews | Before production launch | Security wiki | Security team |

### Documentation Quality Bar

Good documentation is:

1. **Accurate** — It matches the actual system, not the intended system
2. **Current** — Updated when the system changes (part of the PR)
3. **Actionable** — A reader can do something with it
4. **Findable** — In the right place, with the right title and tags
5. **Concise** — As short as possible, no shorter

### The "Two-Week Test"

Documentation passes the two-week test if: **An engineer who has been away for two weeks can read it and understand what changed and why.**

## Learning Budget

Every engineer has a **$3,000 annual learning budget**:

| Category | Allocation | Examples |
|----------|-----------|----------|
| **Conferences** | $1,500 | NeurIPS, KubeCon, PyCon, Strata |
| **Courses** | $500 | Online courses, certifications |
| **Books** | $200 | Technical books, subscriptions |
| **Workshops** | $500 | External workshops, training days |
| **Tools** | $300 | Personal dev tools, subscriptions |

### Budget Rules

- No pre-approval needed for items under $100
- Items $100-$500 need team lead approval
- Items over $500 need engineering manager approval
- Conference attendance requires a post-attendance tech talk to share learnings
- All purchases must be work-relevant (but "work-relevant" is interpreted broadly)

## Knowledge Repository Organization

```
banking-genai-engineering-academy/
├── README.md                     ← Start here
├── engineering-culture/          ← How we work
├── engineering-philosophy/       ← How we think
├── banking-domain/               ← What we're building for
├── genai-platforms/              ← GenAI system design
├── architecture/                 ← Design decisions, patterns
├── security/                     ← Security practices, reviews
├── incident-management/          ← Incidents and learnings
└── ...
```

**Rules:**
- Every folder has a `README.md` that explains what's in it
- Every document has a last-updated date
- Stale documents (> 6 months without review) are flagged
- Documents reference each other with relative links

## Knowledge Sharing Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact | Fix |
|--------------|-------------------|--------|-----|
| **Hero culture** | Only the "smartest" people present at tech talks | Others feel they can't contribute | Rotate presenters, encourage junior engineers |
| **Documentation debt** | "We'll document it after we ship" | Docs never get written | Docs are part of "done" |
| **Meeting overload** | Too many knowledge-sharing meetings, no time to code | Burnout, resentment | Cap at 2 knowledge-sharing events per week |
| **Stale docs** | Documentation that hasn't been updated in a year | Misleading, dangerous | Flag stale docs, assign owners |
| **Siloed knowledge** | Only one person knows a system | Bus factor of 1 | Mandatory cross-training |
| **Presentation-only** | Tech talks with no recording or notes | Knowledge doesn't persist | Always record, always publish notes |

## Knowledge Sharing for Remote Teams

Our teams are distributed across time zones. Knowledge sharing must work asynchronously:

### Asynchronous Practices

1. **Record everything:** All tech talks, all-hands, and brown bags are recorded and transcribed.
2. **Write decisions down:** Architecture decisions are written as ADRs, not discussed only in meetings.
3. **Use threaded discussions:** Decisions happen in writing (GitHub discussions, Confluence comments), not in DMs.
4. **Create "day in the life" docs:** For each role, document what a typical day looks like, including links to all relevant systems.

### Time Zone Considerations

- Tech talks are recorded — attendance is not mandatory for all time zones
- Meeting rotation: Important meetings rotate times so no single timezone is always disadvantaged
- Async-first: Decisions can be made without a meeting, through written proposals and comments

## Measuring Knowledge Sharing Health

| Metric | How We Measure | Target |
|--------|---------------|--------|
| **Doc freshness** | % of docs updated in last 90 days | > 80% |
| **Talk participation** | % of engineers who presented at least once per quarter | > 60% |
| **Cross-team learning** | Number of engineers attending talks outside their team | > 40% |
| **Onboarding speed** | Time for new hire to ship first PR | < 2 weeks |
| **Bus factor** | Minimum number of people who know each critical system | >= 3 |

## New Hire Knowledge Transfer

Every new hire gets a **30-60-90 day learning plan**:

### Days 1-30: Foundation
- [ ] Read `README.md` and `onboarding/` docs
- [ ] Complete local environment setup
- [ ] Shadow an on-call engineer for one rotation
- [ ] Read the top 5 most-referenced ADRs
- [ ] Understand the production architecture (draw it from memory)
- [ ] Ship first small PR (bug fix or docs)

### Days 31-60: Contribution
- [ ] Own a small feature end-to-end
- [ ] Present a tech talk or brown bag (any topic)
- [ ] Complete a code review of someone else's PR
- [ ] Understand the deployment pipeline
- [ ] Write a runbook for something you've learned

### Days 61-90: Ownership
- [ ] Lead a design review for a medium-sized feature
- [ ] Participate in an incident response
- [ ] Mentor a newer team member (even if just on onboarding)
- [ ] Propose and implement an improvement to the platform
- [ ] Write a postmortem for any incident you were involved in

## Knowledge Sharing as Career Growth

Contributing to knowledge sharing is part of the **promotion criteria**:

| Level | Knowledge Sharing Expectation |
|-------|------------------------------|
| **Junior** | Actively learns, asks questions, documents own work |
| **Mid** | Presents tech talks, writes docs, mentors juniors |
| **Senior** | Leads knowledge-sharing initiatives, creates learning programs |
| **Staff** | Sets knowledge-sharing culture, cross-team education |
| **Principal** | Industry-level sharing (papers, talks, open source) |

## Cross-References

- `engineering-culture/documentation-culture.md` — Documentation as engineering responsibility
- `engineering-culture/design-docs.md` — Design document process
- `engineering-culture/async-communication.md` — Async-first communication
- `engineering-culture/mentorship.md` — Mentorship program
- `leadership-and-collaboration/writing-strategy-docs.md` — Strategy document writing
