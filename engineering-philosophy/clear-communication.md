# Clear Communication

> **Writing design docs, status updates, escalation communication.**
> **Audience:** All engineers on the GenAI Platform team
> **Owner:** Engineering Leadership

---

## Core Principle

**If you cannot explain what you are doing, why it matters, and what the risks are — you do not understand it well enough to build it.**

Engineering communication is not about writing beautiful prose. It is about **transmitting complex technical information accurately and efficiently** to people who need to make decisions based on it.

In a global bank, poor communication is not just an annoyance. It is a **risk vector**:

- A poorly written design doc leads to building the wrong thing.
- A vague status update causes a delayed escalation.
- An unclear incident message keeps the wrong people paged for hours.
- An ambiguous handoff causes a production outage.

Clear communication is a technical skill. Treat it that way.

---

## Writing Design Documents

### The Purpose of a Design Doc

A design document is not a novel. It is a **decision-making tool**. Its purpose is to:

1. Align stakeholders on what will be built and why
2. Record the reasoning behind architectural decisions
3. Surface risks and dependencies early
4. Provide a reference for future engineers

If your design doc does not lead to a decision, it has failed.

### The Design Doc Structure

```markdown
# Design Doc: [Title]

**Author:** [Name]
**Reviewers:** [Names]
**Status:** Draft / In Review / Approved / Superseded
**Target Date:** [YYYY-MM-DD]
**JIRA:** [PROJ-XXXX]

---

## 1. Problem Statement
[2-3 paragraphs. What is broken, who is affected, what is the impact.
 Include data. No solutions yet.]

## 2. Goals and Non-Goals
### Goals
- [What this design aims to achieve — measurable outcomes]
- [Specific user or system impacts]

### Non-Goals
- [What is explicitly out of scope — prevents scope creep]

## 3. Proposed Solution
[The architecture. Diagrams are mandatory here.
 Describe components, data flow, and interactions.]

### 3.1 Architecture Overview
[Mermaid diagram or architecture description]

### 3.2 Component Design
[Detail each component: responsibility, interfaces, dependencies]

### 3.3 Data Model
[Schema changes, data migration plan, data retention policy]

### 3.4 API Design
[Endpoints, request/response schemas, error codes]

## 4. Alternatives Considered
| Option | Pros | Cons | Why Rejected |
|--------|------|------|-------------|
| ... | ... | ... | ... |

## 5. Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| ... | ... | ... | ... |

## 6. Security and Compliance
[Data classification, access controls, audit requirements,
 regulatory implications, privacy considerations]

## 7. Observability
[Logging, metrics, alerts, dashboards]

## 8. Rollout Plan
[Phased deployment, feature flags, canary strategy, rollback plan]

## 9. Open Questions
[Unresolved items that need input from others]

## 10. Timeline and Dependencies
[Milestones, team dependencies, critical path]
```

### Design Doc Anti-Patterns

```
The Encyclopedia:
  40+ pages of exhaustive detail. Nobody reads it. Nobody approves it.
  Fix: Keep it under 10 pages. Link to appendices for deep detail.

The Solution Without a Problem:
  "Here is the architecture." (But what problem does it solve?)
  Fix: Start with the problem. Always.

The No-Alternatives Doc:
  "The solution is X." (No other options considered.)
  Fix: Always present at least 2 alternatives, even if one is obvious.

The Rubber Stamp:
  Written after the decision was already made.
  Fix: Write the doc BEFORE building. If you have already built it,
       call it a "retrospective design doc" and be honest about it.

The Living Document:
  Updated continuously as implementation diverges from design.
  Fix: A design doc is a snapshot of the design at decision time.
       If implementation diverges, write a new doc or an ADR.
```

### Real Story: The Design Doc That Saved Us 3 Months

> **Situation (Q2 2025):** The platform team needed to implement a multi-tenant RAG system so that different business units could have isolated knowledge bases with different access controls.
>
> **What happened:** A staff engineer (Meera) wrote a 7-page design doc with three alternative architectures. She spent 2 weeks socializing it with stakeholders before the formal review.
>
> **During the review,** the security team pointed out that alternative A (shared vector store with row-level access control) would require a security model that the database team could not support within the compliance timeline. Alternative B (separate vector store per tenant) would exceed OpenShift resource quotas.
>
> **Alternative C** (hybrid: shared infrastructure with namespace-level isolation) was the only viable path, but required coordination with the platform team for OpenShift namespace provisioning.
>
> **Without the design doc,** the team would have built alternative A and discovered the security issue 2 months into implementation — requiring a complete rewrite.
>
> **The design doc saved an estimated 3 months of wasted effort.** This is the highest-ROI engineering activity Meera performed that quarter.

---

## Writing Status Updates

### The Weekly Update Format

Every engineer should send a weekly update to their team. Keep it to 5 minutes of reading:

```markdown
## Weekly Update — [Name] — [Week of YYYY-MM-DD]

### Completed This Week
- [PROJ-1234] Implemented cross-encoder re-ranking for compliance retriever
  - Retrieval accuracy improved from 62% to 78%
  - PR merged, deployed to staging
- [PROJ-1240] Fixed guardrails false positive on legitimate financial queries
  - Tuned detection rules, false positives reduced from 5.2% to 1.8%

### In Progress
- [PROJ-1245] Adding cache invalidation for RAG response cache
  - 60% complete. Targeting end of week.
  - Blocked on: need OpenShift config change from platform team (ticket opened)

### Blockers / Risks
- [RISK] PROJ-1245 may slip if platform team cannot prioritize the config change.
  I have messaged their lead; awaiting response.

### Next Week
- Complete PROJ-1245 (cache invalidation)
- Begin PROJ-1250 (prompt template versioning design doc)

### Help Needed
- Looking for a reviewer for the re-ranking integration tests.
  Any volunteer for a 30-minute review this week?
```

### Status Update Anti-Patterns

```
The Diary:
  "Monday I did X. Tuesday I did Y. Wednesday I had a meeting..."
  Fix: Group by work item, not by day. Focus on outcomes, not activities.

The Omission:
  No mention of blockers or risks. Everything is "on track."
  Fix: If there is a risk, state it. Silence is not optimism — it is denial.

The Wall of Text:
  Three paragraphs with no structure or formatting.
  Fix: Use headers, bullet points, and ticket references.

The Zero-Update:
  "Same as last week."
  Fix: Even if progress was minimal, say WHY. "Blocked on X. Waiting for Y."
```

---

## Escalation Communication

### When and How to Escalate

```
Escalate when:

- A blocker has been unresolved for > 1 business day
- A production incident is ongoing and you need more expertise
- A dependency from another team is blocking your critical path
- A risk has materialized or is about to
- You need a decision from someone with more authority
```

### The Escalation Template

```
Subject: [ESCALATION] [Severity] — [Brief Description] — [Ticket/Incident ID]

Team: [Team Name]
Impact: [Who is affected and how]
Blocker: [What is preventing progress]
Attempted: [What you have tried]
Needed: [What you need and from whom]
Deadline: [When you need it by]

Example:

Subject: [ESCALATION] Medium — RAG Pipeline Deployment Blocked on
         OpenShift Quota — PROJ-1245

Impact: 2,400 compliance users will not receive improved retrieval
        accuracy by the Q3 deadline if this slips past Friday.

Blocker: The cross-encoder re-ranking model requires 4 GPU pods. Current
         namespace quota is 2 GPU pods. We need a quota increase.

Attempted:
- Opened platform team ticket (PLAT-892) on Monday
- Messaged platform team lead on Slack (no response)
- Checked quota request process — SLA is 3 business days

Needed: Platform team to approve quota increase to 4 GPU pods for
        genai-compliance namespace by end of day Thursday.

Deadline: Thursday EOD (deployment window is Friday 6 AM).
```

### Escalation Anti-Patterns

```
The Complaint Without Context:
  "The platform team is blocking us."
  Fix: Include what you have tried, ticket numbers, and timeline.

The Emotional Escalation:
  "Nobody is helping us. This is unacceptable."
  Fix: State facts, not feelings. The facts are compelling enough.

The Late Escalation:
  Waiting until the deadline is tomorrow to escalate a 3-day blocker.
  Fix: Escalate as soon as the blocker is identified, not when the
       deadline is at risk.

The Nuclear Option:
  Copying VPs on the first escalation.
  Fix: Start with the team lead. Only involve senior leadership
       if the team-level escalation fails after 1 business day.
```

---

## Being Proactive with Communication

### Proactive communication means informing people before they need to ask.

```
Proactive:
  "The deployment is running now. Expect 10 minutes of degraded
   performance. I will update when it is complete."

Reactive:
  "Why is the service degraded?"
  "Oh, I was deploying. Sorry."

The difference: In the proactive version, nobody pages you.
In the reactive version, you get three pages and a message from
your manager asking what is going on.
```

### Communication Triggers

```
Always communicate proactively when:

- Deploying to production (what, when, expected impact, rollback plan)
- A dependency change will affect other teams (API changes, schema changes)
- A risk has increased (even if it has not materialized)
- Your timeline has changed (slip early, slip often)
- You discover information that affects other teams' decisions
- An incident is resolved (even if nobody asked)
```

### The "No Surprises" Rule

```
Your manager should never be surprised by:
- A missed deadline they did not know was at risk
- A production incident you have been handling silently for hours
- A conflict with another team they could have helped prevent
- A technical decision that has political implications

If your manager is surprised, you have failed to communicate.
This is not about blame. It is about ensuring the people with
more context and authority can help you navigate risks.
```

### Real Story: The Silent Failure

> **Situation:** An engineer discovered that the embedding model was producing degraded quality after a provider update. The model provider had silently changed the model behavior.
>
> **What the engineer did:** They spent 3 days investigating and confirming the issue. They built a fix (switching to a pinned model version). They deployed it.
>
> **What the engineer did NOT do:** Inform anyone about the degradation during those 3 days.
>
> **Consequence:** During those 3 days, 800+ compliance queries were answered with lower-quality retrievals. The compliance team noticed the quality drop independently and filed complaints. By the time the fix was deployed, the team had already lost trust.
>
> **What should have happened:** On Day 1, when the degradation was first detected:
> "Heads up: the embedding provider updated their model. I am seeing a 5-point
> drop in retrieval accuracy. I am investigating and will update by EOD."
>
> This simple message would have set expectations, prevented surprise, and
> given the compliance team context for the quality drop they were seeing.
>
> **Lesson:** Proactive communication is not admitting failure. It is managing expectations and enabling others to make informed decisions.

---

## Sharing Knowledge

### Knowledge Sharing Is a Multiplier

Individual knowledge is a team risk. If only one person knows how a system works, that person is a single point of failure.

### Knowledge Sharing Mechanisms

```
Formal:
- Design docs and ADRs (architectural knowledge)
- Runbooks and playbooks (operational knowledge)
- Architecture diagrams (system knowledge)
- On-call handoff notes (contextual knowledge)
- Tech talks and demos (broad knowledge dissemination)

Informal:
- Pair programming sessions (deep knowledge transfer)
- Code review comments (contextual decision records)
- Team channel posts (discovery sharing)
- Brown bag sessions (cross-team knowledge sharing)
```

### The "Bus Factor" Metric

```
Bus Factor: The minimum number of team members who would need to be
            hit by a bus (or go on vacation, or leave the company)
            before the team cannot function.

Goal: Every critical system should have a bus factor of >= 3.
      At least 3 people should be able to operate and debug it.

How to improve bus factor:
1. Document the system (runbook, architecture, deployment)
2. Train at least 2 other people through pair debugging
3. Rotate on-call so multiple people gain operational experience
4. Record tech talks and share recordings
```

### Real Example: The Knowledge Transfer Session

> **Situation:** Meera was the only person who understood the prompt injection detection system. She was planning parental leave.
>
> **What she did:**
> 1. Wrote a comprehensive runbook (3 hours)
> 2. Conducted 3 pair-debugging sessions with different team members (6 hours)
> 3. Recorded a 45-minute tech talk on the injection detection architecture
> 4. Created a "debugging guide" with common failure modes and their fixes
>
> **Result:** During her leave, the team handled two injection-related incidents without her involvement. The bus factor went from 1 to 4.
>
> **Time invested:** ~12 hours.
> **Time saved:** The alternative would have been interrupting Meera's leave for incidents, which would have cost more time and damaged trust.

---

## Cross-References

- **Collaborative Engineering** (`collaborative-engineering.md`) — How knowledge sharing and helping peers fits into the broader collaborative mindset.
- **Ownership and Accountability** (`ownership-and-accountability.md`) — Communication is part of owning something end-to-end.
- **Solutions-First Mindset** (`solutions-first-mindset.md`) — The one-page proposal template for communicating solutions.
- **Leadership and Collaboration** (leadership-and-collaboration/ folder) — Broader communication and influence patterns.

---

## Interview Preparation

### Questions You Might Be Asked

1. **"Tell me about a time you had to communicate a technical risk to non-technical stakeholders."**
   - Focus on translation: technical details → business impact → recommended action.

2. **"Describe a time a project was at risk and how you communicated it."**
   - Use the "slip early" principle. Show proactive communication.

3. **"How do you handle a disagreement about a technical approach?"**
   - Write a design doc with alternatives. Let data and structured reasoning decide.

4. **"Tell me about a time you improved team communication."**
   - Use the knowledge transfer or runbook writing examples.

### STAR Story: Risk Communication

```
Situation:  "Three weeks before a critical compliance deadline, I discovered
             that our retrieval accuracy was 15 points below the target,
             and the fix required a cross-encoder model that needed GPU
             quota increases from the platform team."

Task:       "Communicate the risk to engineering leadership and the
             compliance stakeholders without causing panic, while being
             honest about the timeline impact."

Action:     "I sent a status update that day: here is the current state,
             here is the gap, here is the proposed fix, here is the
             dependency on the platform team, here is the risk to the
             deadline. I provided three scenarios: best case (platform
             team approves in 2 days), likely case (5 business days),
             worst case (quota denied, need alternative approach). I
             recommended the likely case path with a contingency plan."

Result:     "Engineering leadership escalated the quota request with the
             platform team lead. It was approved in 3 days. We met the
             deadline with 2 days to spare. The compliance team appreciated
             the transparency and had time to prepare contingency processes."
```

---

## Summary

1. **Communication is a technical skill.** Clear, structured, efficient.
2. **Design docs are decision tools.** If they do not lead to a decision, they failed.
3. **Status updates are for risk visibility.** Highlight blockers and risks, not just progress.
4. **Escalate early with context.** Facts, not emotions. What you tried, what you need.
5. **Proactive communication prevents surprises.** Inform before people ask.
6. **Knowledge sharing reduces team risk.** Bus factor >= 3 for every critical system.
7. **The "no surprises" rule.** Your stakeholders should never learn about problems from incidents, not from you.

> "The best engineers are not just the ones who can build complex systems.
> They are the ones who can explain them, get alignment on them, and
> ensure everyone affected knows what is happening and why."
