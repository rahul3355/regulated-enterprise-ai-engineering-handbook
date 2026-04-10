# Mentorship Program

> "The best engineers aren't just individual contributors — they're force multipliers. Mentorship is how we multiply."

## Why Mentorship Matters in Banking GenAI

Building GenAI systems for a global bank requires skills that no single person has. The domain knowledge (banking regulations, compliance, risk models) is separate from the technical skills (LLM systems, RAG, distributed systems), and both are separate from the platform skills (Kubernetes, CI/CD, observability).

Mentorship is how we transfer this knowledge across the team and grow engineers who can operate across all three domains.

## Mentorship Models

### 1:1 Mentorship

**What it is:** A senior engineer is paired with a less experienced engineer for regular guidance.

**Duration:** 3-6 months, renewable.

**Cadence:** Weekly 30-minute sessions, plus ad-hoc availability for questions.

**How it works:**

```
Week 1: Goal Setting
├── Mentor and mentee discuss career aspirations
├── Identify 2-3 specific growth areas
├── Create a development plan with milestones
└── Schedule recurring sessions

Weeks 2-12: Execution
├── Weekly check-ins on progress
├── Review of code, designs, or documents
├── Discussion of blockers and challenges
└── Adjustment of goals as needed

Week 12+: Review and Renew
├── Assess progress against goals
├── Celebrate wins
├── Identify next growth areas
└── Decide whether to continue or rotate
```

**Pairing guidelines:**

| Mentee Level | Ideal Mentor | Focus Areas |
|--------------|-------------|-------------|
| Junior (0-2 years) | Senior (5+ years) | Code quality, debugging, banking basics |
| Mid-level (2-4 years) | Staff (7+ years) | Architecture, system design, cross-team work |
| Senior (5+ years) | Principal (10+ years) | Technical strategy, influence, ambiguity |

### Pair Programming Partnership

**What it is:** Two engineers work together on the same code, same screen, same problem.

**Cadence:** 2-4 hours per week, scheduled.

**How it works:**

- **Driver/Navigator model:** The less experienced engineer drives (types), the more experienced navigates (thinks aloud, suggests direction).
- **Rotate roles:** Even within a session, switch who is driving every 25 minutes.
- **No silent reviewing:** The navigator should think aloud, not just watch the driver code.

**Remote pairing setup:**

```
Tools:
├── VS Code Live Share (preferred)
├── Terminal sharing (tmux pair or Tmate)
├── Video call (always on, for non-verbal cues)
└── Shared whiteboard (Miro or Excalidraw)

Etiquette:
├── Both parties prepare before the session
├── Clear goal for the session (not open-ended exploration)
├── Regular breaks (5 min every 25 min)
└── Debrief at the end (what did we learn?)
```

### Group Mentorship

**What it is:** One senior engineer mentors a group of 4-6 engineers on a specific topic.

**Examples:**
- "GenAI Architecture Office Hours" — 1 hour weekly, open to all
- "Security Review Workshop" — Monthly deep-dive
- "System Design Study Group" — Bi-weekly case studies

## Growth Plan Template

Every mentee should have a written growth plan:

```markdown
# Growth Plan: [Name]
# Period: Q1 2026
# Mentor: [Name]

## Current State
- Strengths: [3-5 specific strengths]
- Growth Areas: [2-3 areas to develop]

## Goals (SMART)

### Goal 1: [Specific, Measurable Goal]
- **What:** Design and implement a production-grade RAG pipeline
- **Why:** Needed for the GenAI platform team
- **Success Criteria:** Pipeline deployed, tested, documented, reviewed
- **Timeline:** End of Q1
- **Support Needed:** Access to vector DB, time for study, code reviews

### Goal 2: [Specific, Measurable Goal]
- **What:** Lead a cross-team design review
- **Why:** Developing influence and communication skills
- **Success Criteria:** Design doc written, review conducted, decisions captured
- **Timeline:** Mid Q1
- **Support Needed:** Mentor review of design doc before presentation

## Check-in Schedule
- Weekly: 30-min 1:1 with mentor
- Bi-weekly: Code review session with mentor
- Monthly: Progress review with engineering manager

## Resources
- Reading: `genai-platforms/rag-pipeline-design.md`
- Course: Internal "Advanced RAG Patterns" workshop
- Practice: `exercises/coding-exercise-04-rag-pipeline.md`
```

## Mentorship for GenAI-Specific Skills

GenAI engineering requires unique skills that are hard to learn from books. Mentorship is essential:

### Prompt Engineering Mentorship

**What a mentor does:**
- Reviews prompts with the mentee, discussing why certain formulations work better
- Shares examples of prompt injection attacks and how to defend against them
- Walks through prompt versioning and testing strategies
- Co-creates a prompt library with the mentee

### RAG Pipeline Mentorship

**What a mentor does:**
- Pairs on building a retrieval pipeline from scratch
- Discusses chunking strategies and their trade-offs
- Reviews embedding quality with concrete examples
- Explains how to evaluate retrieval quality (not just "it seems good")

### Production Deployment Mentorship

**What a mentor does:**
- Walks through the deployment pipeline for a GenAI service
- Explains how to set up monitoring for LLM response quality
- Discusses incident response when a model starts producing problematic outputs
- Reviews rollback procedures for model changes

## Mentor-mentee Matching

### How We Match

1. **Self-nomination:** Engineers express interest in mentoring or being mentored
2. **Skill gap analysis:** Identify what the mentee needs to learn
3. **Availability check:** Ensure mentor has capacity (max 2 mentees at once)
4. **Personality fit:** Consider communication styles and time zones
5. **Trial period:** First month is a trial — either party can request a re-match

### Re-matching

It's okay if a match doesn't work out. Signs that re-matching might be needed:

- Sessions are consistently cancelled or rescheduled
- No progress on growth goals after 6 weeks
- Communication styles clash significantly
- Either party feels the relationship isn't valuable

**Re-matching is not a failure.** It's an optimization.

## Mentor Recognition

Mentorship is work, and it should be recognized:

- **Performance reviews:** Mentorship contributions count toward promotion criteria
- **Engagement surveys:** Include questions about mentorship quality
- **Engineering awards:** Quarterly recognition for outstanding mentors
- **Visibility:** Mentors present their mentees' wins in team demos

## Anti-Patterns

| Anti-Pattern | What It Looks Like | Better Approach |
|--------------|-------------------|-----------------|
| **Spoon-feeding** | Mentor writes the code, mentee watches | Mentor asks guiding questions, mentee writes |
| **Gatekeeping** | "You're not ready for that yet" | "Here's what you need to be ready" |
| **Overloading** | Mentor takes on 5+ mentees | Max 2 mentees per mentor |
| **No structure** | "Let's just chat each week" | Have a growth plan with goals |
| **One-way street** | Mentor always teaches, never learns | Mentors learn from mentees too |
| **No end date** | Indefinite mentorship | 3-6 month cycles with renewal |

## Measuring Mentorship Success

We don't use metrics to judge individuals, but we track program health:

| Metric | Target | Why |
|--------|--------|-----|
| Participation rate | > 60% of engineers | Is the program accessible? |
| Retention rate | > 80% renew or continue | Is it valuable? |
| Goal completion rate | > 70% of goals met | Is it effective? |
| Satisfaction score | > 4/5 from both parties | Is the experience positive? |

## Starting a Mentorship Relationship

### First Session Agenda

1. **Introductions (10 min):** Background, role, interests
2. **Career discussion (10 min):** Where does the mentee want to be in 1-2 years?
3. **Strengths assessment (5 min):** What is the mentee already good at?
4. **Growth areas (5 min):** What does the mentee want to improve?
5. **Goal drafting (10 min):** Sketch 2-3 initial goals
6. **Logistics (5 min):** Cadence, communication channel, availability
7. **Action items (5 min):** What happens before next session?

### Ongoing Session Agenda

1. **Wins (5 min):** What went well since last session?
2. **Blockers (10 min):** What's stuck? What needs help?
3. **Code/design review (10 min):** Review specific work
4. **Learning (5 min):** Discuss a concept, pattern, or article
5. **Action items (5 min):** What happens before next session?

## Cross-References

- `engineering-culture/pair-programming.md` — Pair programming practices
- `engineering-culture/knowledge-sharing.md` — Broader knowledge sharing programs
- `leadership-and-collaboration/coaching-junior-engineers.md` — Coaching framework
- `engineering-philosophy/growth-mindset.md` — Growth mindset and continuous learning
- `interview-prep/career-progression.md` — Career ladders and growth expectations
