# Engineering in Large Organisations

> **Navigating bureaucracy, building alignment, driving change.**
> **Audience:** All engineers on the GenAI Platform team
> **Owner:** Engineering Leadership

---

## Core Principle

**Large organizations are not broken. They are optimized for different things than you are.**

As an engineer, you are optimized for:
- Shipping working software
- Technical quality
- User impact
- Iteration speed

A global bank is optimized for:
- Managing risk
- Regulatory compliance
- Audit readiness
- Stability and predictability

Neither is wrong. They are different. The engineer who fights the organization's priorities is the engineer who gets frustrated, burned out, and eventually leaves. The engineer who **learns to work within the system while still driving change** is the engineer who has impact at scale.

---

## Understanding the Organization

### The Org Chart Is Not the Real Organization

```
What the org chart shows:           What the real organization looks like:

  VP Engineering                    VP Eng ─────┐
     │                                            │
  Director ── Director ── Director         ───────┼───────
     │           │           │                    │
  Team A      Team B      Team C           ───────┼───────
                                                    │
                                            Influencers, decision-makers,
                                            gatekeepers, and connectors
                                            span across all teams

The real organization is a network, not a tree.
Your ability to get things done depends on the network, not the chart.
```

### The Decision-Makers Map

For any significant change, identify:

```
Decision Roles (RACI + Influence):

Responsible (R): Who does the work? (You.)
Accountable (A): Who signs off on the outcome? (Your engineering manager or tech lead.)
Consulted (C): Who needs to give input? (Security, compliance, platform team, etc.)
Informed (I): Who needs to know the outcome? (Stakeholders, downstream teams.)

+ Two hidden roles:

The Gatekeeper: The person who can say "no" even if they are not formally
                accountable. (Often: security lead, compliance officer,
                platform team architect.)

The Champion: The person who says "yes" and advocates for your change
              when you are not in the room. (Often: a senior engineer
              or director who sees the strategic value.)

You need at least one Champion and zero hostile Gatekeepers for any
significant change to succeed.
```

### How Decisions Actually Get Made

```
The formal process:
  Design doc → Review meeting → Approval → Implementation

The actual process:
  1. You socialize the idea informally with key stakeholders.
  2. You incorporate their feedback before the formal review.
  3. You write the design doc (which everyone has already seen).
  4. The formal review is a formality because alignment is already built.
  5. Approval is granted.
  6. Implementation begins.

The engineer who skips steps 1-2 and goes straight to step 3
will have their design doc rejected or endlessly revised because
the stakeholders are seeing it for the first time in the review
meeting and will raise concerns that could have been addressed earlier.

This is not bureaucracy. This is how human organizations work.
```

---

## Navigating Bureaucracy

### Bureaucracy Exists for a Reason

```
Every process in a bank exists because something went wrong in the past:

Process                        Exists Because
─────────                      ──────────────
Change Advisory Board (CAB)    Someone deployed on Friday and broke production
Security Review                Someone shipped a vulnerability
Compliance Sign-off            Someone handled regulated data incorrectly
Architecture Review Board      Someone built something that could not scale
Vendor Review                  Someone signed a contract with unacceptable terms
Privacy Impact Assessment      Someone exposed personal data

When you complain about "bureaucracy," you are complaining about
the scars from previous injuries. The scars are not going away.
Learn to work with them.
```

### The Bureaucracy Navigation Playbook

```
Step 1 — Understand the process.
  Read the documentation. Ask someone who has done it before.
  Do not guess. The process is documented for a reason.

Step 2 — Start early.
  If the security review SLA is 5 business days, start 7 days before
  you need approval. The 2-day buffer is for when your submission
  is rejected and you need to resubmit.

Step 3 — Make it easy for the reviewers.
  Provide all required information upfront.
  Anticipate their questions and answer them in your submission.
  A complete submission gets reviewed faster than an incomplete one
  that requires back-and-forth.

Step 4 — Build relationships with the process owners.
  The security reviewer is not your enemy. They are your partner.
  If they trust you and know your systems, the review is faster
  and more productive.

Step 5 — Improve the process from within.
  If a process is genuinely broken, do not complain about it.
  Propose an improvement. Volunteer to pilot it.
  The people who improve processes are the ones who get promoted.
```

### Real Story: The Engineer Who Fixed the Process

> **Situation:** The security review process for GenAI platform changes had a 10-business-day SLA. The actual average time was 18 business days because submissions were frequently incomplete.
>
> **The complaint:** "Security takes forever. They are the bottleneck."
>
> **What one engineer (Priya) did instead:**
> 1. Analyzed the last 20 security review submissions.
> 2. Found that 60% were delayed due to missing information (threat model, data flow diagram, or test evidence).
> 3. Created a pre-submission checklist that engineers could use before filing.
> 4. Partnered with the security team to create a "pre-review" office hour where engineers could ask questions before submitting.
> 5. Tracked the impact over 3 months.
>
> **Result:** Average review time dropped from 18 days to 8 days. The security team was happier (fewer incomplete submissions). The engineers were happier (faster reviews). Priya was recognized in the quarterly engineering all-hands.
>
> **Lesson:** Do not fight the process. Improve it. The engineer who improves the process has more impact than the engineer who works around it.

---

## Building Alignment

### The Alignment Building Process

```
For any significant change (> 1 sprint of effort, > 1 team affected):

Week 1: Socialize
  → Talk to the key stakeholders individually.
  → Present the problem and your initial thinking.
  → Listen more than you talk. Understand their concerns.
  → "What would need to be true for you to support this?"

Week 2: Incorporate
  → Update your proposal based on stakeholder feedback.
  → Address concerns explicitly in the design doc.
  → If a concern cannot be addressed, document why and the mitigation.
  → Share the updated draft with key stakeholders for a second round.

Week 3: Formalize
  → Write the final design doc (which stakeholders have already seen).
  → Schedule the review meeting.
  → The meeting should have no surprises.

Week 4: Decide
  → The review meeting is a formality.
  → Approval is granted (or concerns are already known and addressed).
  → Implementation begins.
```

### The Pre-Wire Technique

```
"Pre-wiring" means getting buy-in before the formal meeting.

How to pre-wire:

1. Identify the 3-5 people whose opinion matters most.
   (Tech leads, architects, team leads, security leads.)

2. Meet with each of them individually (15-20 min each).
   "I am thinking about [topic]. Here is my initial approach.
    What concerns do you have? What am I missing?"

3. Incorporate their feedback.
   If they raise a valid concern, address it in the design.
   If they raise an invalid concern, understand why they have it
   and address it with data.

4. In the formal meeting:
   "I have spoken with [names] and incorporated their feedback.
    The design addresses [concerns]. Are there any new concerns?"

The pre-wire technique is not manipulation. It is respect for
people's time and expertise. It ensures that the formal meeting
is a decision point, not a discovery session.
```

### Real Story: The Change That Failed Because Nobody Pre-Wired

> **Situation:** A senior engineer (David) designed a new observability platform for the GenAI team. It was technically excellent — better metrics, better dashboards, better alerts.
>
> **What David did:** Built a prototype, wrote a 20-page design doc, scheduled a review meeting.
>
> **What happened in the meeting:**
> - The SRE team lead: "This overlaps with our existing observability strategy."
> - The platform team lead: "This requires infrastructure changes we have not planned for."
> - The security team: "This sends data to a new endpoint that has not been reviewed."
> - The engineering manager: "Why was this not coordinated with the other teams?"
>
> **Result:** The design was sent back for "further alignment." David spent 6 more weeks meeting with teams individually, revising the design, and rebuilding trust. The total time was 10 weeks instead of 4.
>
> **What David should have done:** Pre-wire with the SRE lead, platform lead, and security lead BEFORE writing the design doc. Their concerns would have shaped the design from the start.
>
> **Lesson:** The design doc is the last step of alignment, not the first.

---

## Driving Change Without Authority

### The Change Driving Framework

You do not need a management title to drive change. You need:

```
1. A clear problem statement (from solutions-first-mindset.md)
2. Data to support the problem
3. A proposed solution with alternatives
4. A champion at the director level or above
5. Willingness to do the work yourself
```

### The Influence Ladder

```
Level 1: Suggest
  "Have we considered [idea]?"
  → Low commitment, low impact.
  → Good for planting seeds.

Level 2: Prototype
  "I built a prototype to test [idea]. Here are the results."
  → Medium commitment, medium impact.
  → Data speaks louder than opinions.

Level 3: Propose
  "Here is a design doc for [idea]. I have spoken with [stakeholders]
   and addressed their concerns. I am willing to own the implementation."
  → High commitment, high impact.
  → This is how significant changes happen.

Level 4: Lead
  "Here is the initiative. I have the champion's support. Here is the
   timeline, the team, and the plan. Let us begin."
  → Highest commitment, highest impact.
  → This is Staff/Principal engineer territory.
```

### Real Story: The Engineer Who Changed the Team's Testing Culture

> **Situation:** The GenAI Platform team had inconsistent testing practices. Some services had comprehensive tests. Others had none. The overall test coverage was 34%.
>
> **The engineer:** A mid-level engineer (Ana) who believed this was unacceptable for a compliance-critical platform.
>
> **Ana's approach:**
>
> Phase 1 — Prove the value (2 weeks):
> - Wrote tests for the most bug-prone service (the guardrails module).
> - Found 3 real bugs that had caused production incidents.
> - Presented the findings in a team meeting: "Here are 3 bugs that tests
>   would have caught before production."
>
> Phase 2 — Make it easy (1 week):
> - Created a testing checklist and template for the team.
> - Added a CI gate that blocked PRs with < 70% coverage on new code.
> - Wrote a "Getting Started with Testing" guide for the team.
>
> Phase 3 — Get the champion (1 week):
> - Presented the results to the engineering manager: "Test coverage is
>   now 52%. Production bugs from the guardrails module dropped to zero.
>   I would like to make 70% coverage a team standard."
> - The manager agreed and announced the standard.
>
> Phase 4 — Sustain (ongoing):
> - Ana volunteered to review testing PRs for the first month.
> - She ran a 30-minute "testing office hour" twice a week.
> - She tracked coverage monthly and reported progress.
>
> **Result:** Within 3 months, team-wide test coverage reached 72%. The CI gate was adopted by two other teams. Ana was promoted to Senior Engineer.
>
> **Lesson:** Change is driven by evidence, not opinion. Ana proved the value, made it easy, got the champion's support, and sustained the effort. She had no authority — but she had credibility, data, and willingness to do the work.

---

## How Senior Engineers Influence Teams Without Formal Authority

This topic was introduced in `collaborative-engineering.md`. Here is the deeper treatment specific to large organizations.

### The Organizational Influence Model

```
In a large organization, influence requires three things:

1. Credibility (earned through past results)
   → You need a track record of good decisions and successful deliveries.
   → New engineers have limited credibility. Build it through small wins.

2. Network (built through collaboration)
   → You need relationships across teams and levels.
   → The engineer who only talks to their immediate team has limited reach.

3. Strategic alignment (understanding what matters to leadership)
   → Your proposal must align with organizational priorities.
   → "This would be cool" is not a strategic argument.
   → "This reduces regulatory risk by X%" is.
```

### Navigating the Political Landscape

```
Politics in engineering is not dirty. It is simply the reality that:

- Decisions are made by people, not by technical merit alone.
- People have different priorities, risk tolerances, and incentives.
- Resources are有限, and competition for them is real.
- Timing matters as much as the idea itself.

Navigating this landscape means:

1. Understanding what each stakeholder cares about.
   → The security team cares about risk reduction.
   → The product team cares about user impact.
   → The platform team cares about operational burden.
   → The engineering manager cares about delivery predictability.

2. Framing your proposal in terms of what they care about.
   → To security: "This reduces our risk exposure by X."
   → To product: "This improves user satisfaction by Y."
   → To platform: "This reduces operational burden by Z."
   → To the manager: "This is delivered on time, within budget."

3. Timing your proposal when the organization is receptive.
   → After an incident is the best time to propose reliability improvements.
   → After a compliance audit is the best time to propose compliance tooling.
   → During budget planning is the best time to propose infrastructure investment.
```

### The Long Game

```
Organizational change takes time. The engineer who expects immediate
results will be disappointed.

Timeline for significant change:
  Small change (one team):     2-4 weeks
  Medium change (multiple      4-8 weeks
  teams, moderate complexity)
  Large change (org-wide,      8-16 weeks
  significant investment)
  Cultural change (testing     3-6 months
  culture, deployment culture)

The engineer who starts the conversation today and ships the change
in 3 months is not slow. They are realistic.

The engineer who complains that "nothing ever changes around here"
is usually the engineer who gave up after 2 weeks.
```

---

## Cross-References

- **Collaborative Engineering** (`collaborative-engineering.md`) — Cross-team collaboration and building relationships.
- **Clear Communication** (`clear-communication.md`) — Writing design docs, status updates, and escalations that build alignment.
- **Solutions-First Mindset** (`solutions-first-mindset.md`) — Framing proposals as solutions to problems the organization cares about.
- **Bias for Action** (`bias-for-action.md`) — Driving change requires action bias, even in a large organization.
- **Leadership and collaboration** (leadership-and-collaboration/ folder) — Broader leadership patterns and influence techniques.

---

## Interview Preparation

### Questions You Might Be Asked

1. **"Tell me about a time you navigated organizational complexity to deliver something."**
   - Use the pre-wiring story (David's failure and what he should have done).

2. **"Describe a time you improved a team process or culture."**
   - Use Ana's testing culture story. Show the phased approach.

3. **"How do you handle a situation where another team is blocking your progress?"**
   - Discuss the escalation framework, relationship building, and alignment.

4. **"Tell me about a time you drove change without formal authority."**
   - Use Ana's story or any example where you influenced through credibility, not mandate.

5. **"How do you build alignment for a technical proposal?"**
   - Discuss the pre-wire technique and the socialize-incorporate-formalize process.

### STAR Story: Driving Change

```
Situation:  "Our team's test coverage was 34%. Some services had no tests.
             Production bugs were frequent, especially in the guardrails
             module which was compliance-critical."

Task:       "Improve testing culture and coverage without any formal
             mandate to do so."

Action:     "I started by writing tests for the most bug-prone service
             and found 3 real production bugs. I presented these findings
             in a team meeting. I created a testing checklist, added a
             CI gate for 70% coverage on new code, and got the engineering
             manager's support for the standard. I ran testing office hours
             and reviewed testing PRs for the first month."

Result:     "Test coverage went from 34% to 72% in 3 months. Production
             bugs from the guardrails module dropped to zero. The CI gate
             was adopted by two other teams. I was promoted to Senior
             Engineer for this initiative."
```

---

## Summary

1. **Large organizations are optimized for risk, not speed.** Work with this, not against it.
2. **The org chart is not the real organization.** Build relationships across the network.
3. **Pre-wire before the formal meeting.** The design doc is the last step, not the first.
4. **Bureaucracy exists for a reason.** Improve processes from within, do not fight them.
5. **Drive change with evidence, not opinion.** Data beats opinion every time.
6. **Frame proposals in terms of what stakeholders care about.** Risk, impact, burden, predictability.
7. **The long game wins.** Significant change takes months, not weeks.
8. **Influence requires credibility, network, and strategic alignment.** All three are necessary.

> "The most impactful engineers in large organizations are not the ones
> with the best technical ideas. They are the ones who can get the
> organization to believe in those ideas, invest in them, and sustain
> them after the original champion moves on."
