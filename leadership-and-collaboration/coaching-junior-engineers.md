# Coaching Junior Engineers

> "Coaching is not about telling someone what to do. It's about helping them figure it out themselves — and building the confidence to do it again next time."

## Coaching vs. Mentoring vs. Managing

| Role | Focus | Relationship | Timeframe |
|------|-------|-------------|-----------|
| **Coaching** | Skill development, performance improvement | Structured, goal-oriented | Weeks to months |
| **Mentoring** | Career guidance, wisdom sharing | Informal, relationship-based | Months to years |
| **Managing** | Delivery, performance, career progression | Authority-based, accountability | Ongoing |

This document focuses on **coaching** — the structured development of specific skills.

## The Coaching Framework: GROW

```
G — Goal: What do we want to achieve?
R — Reality: Where are we now?
O — Options: What could we do?
W — Will: What will we do?
```

### G — Goal

```
Coach asks:
- "What do you want to get better at?"
- "What would success look like in 3 months?"
- "How does this connect to your career goals?"

Good goals are specific:
❌ "Get better at backend engineering"
✅ "Build and deploy a production-grade FastAPI service independently"

❌ "Improve code quality"
✅ "Reduce review iterations from 4 to 2 by catching issues before PR"
```

### R — Reality

```
Coach asks:
- "What's your current level with this skill?"
- "What have you tried so far?"
- "What's getting in the way?"
- "On a scale of 1-10, how confident are you?"

Reality assessment should be honest and specific:
- "I can build APIs but struggle with error handling and testing"
- "I've read the docs but haven't built anything production-grade"
- "I'm not sure how to structure a multi-module project"
```

### O — Options

```
Coach asks:
- "What approaches could work?"
- "Who could help you?"
- "What resources are available?"
- "What's worked for you in the past when learning something new?"

Brainstorm options together:
- Pair programming with a senior engineer
- Complete a hands-on exercise from the academy
- Build a prototype project
- Read specific chapters of relevant docs
- Attend a tech talk on the topic
- Shadow someone building a similar system
```

### W — Will

```
Coach asks:
- "Which option will you commit to?"
- "When will you do it?"
- "How will I know you've done it?"
- "What support do you need from me?"

Commitments are specific and time-bound:
- "I'll build the secure API endpoint exercise by Friday"
- "I'll pair with James on Tuesday and Thursday this week"
- "I'll deploy my first service to staging by end of sprint"
```

## Coaching Techniques

### Ask, Don't Tell

**Telling:** "You need to add input validation here."
**Asking:** "What happens if a user sends an empty string to this endpoint?"

**Telling:** "Use a context manager for the database connection."
**Asking:** "How do we make sure this connection gets cleaned up if an exception is raised?"

**Telling:** "This query is slow because of the N+1 pattern."
**Asking:** "How many database queries does this code make for 100 users?"

Asking develops the engineer's own problem-solving ability. Telling creates dependency.

### The Socratic Method for Code Reviews

Instead of listing all the issues in a PR, guide the engineer to find them:

```
Reviewer: "This looks good overall. I have a few questions:

1. What happens if the model API returns a 500 error here?
2. I notice this function is 80 lines — is there a natural place
   to split it into smaller functions?
3. What level of logging would help someone debug this at 3 AM?
4. Is there a test for the happy path? What about edge cases?

Take your time to think through these. Happy to discuss any of them."
```

### Scaffolding

Provide support that gradually decreases as competence increases:

```
Level 1: I do, you watch (demonstration)
├── Senior engineer builds a feature while narrating
├── Junior engineer observes patterns and decisions

Level 2: I do, you help (guided participation)
├── Senior engineer drives, junior engineer suggests and reviews
├── Junior engineer starts to participate in decisions

Level 3: You do, I help (supported execution)
├── Junior engineer builds, senior engineer reviews and guides
├── Strong-Style pairing: junior drives, senior navigates

Level 4: You do, I watch (monitored independence)
├── Junior engineer builds independently
├── Senior engineer reviews the PR

Level 5: You do (full independence)
├── Junior engineer owns the area end-to-end
├── Senior engineer is available for questions
```

### Feedback Delivery: The SBI Model

```
SITUATION: "In yesterday's PR (#342)..."
BEHAVIOR: "...you didn't include error handling for the database call..."
IMPACT: "...which means if the database is unavailable, the error
propagates to the user as a 500 with a stack trace. In production,
this would expose internal details and make debugging harder."

Then:
- Pause. Let them respond.
- Ask: "What do you think about that?"
- Collaborate: "How can we catch this class of issue before PR next time?"
```

## Coaching for Specific Skills

### Technical Skills

| Skill | Coaching Approach | Practice |
|-------|------------------|----------|
| **API design** | Review existing APIs, design a new one together, critique | Build a CRUD API with auth, validation, error handling |
| **Testing** | Write tests together, discuss what to test and why | Add tests to an untested service |
| **Debugging** | Think aloud while debugging, then watch them debug | Debug a production-like issue in staging |
| **System design** | Walk through existing systems, design a new one together | Complete architecture exercise from the academy |
| **Code review** | Review a PR together, explain your thinking | Review 3 PRs independently, then compare notes |

### Non-Technical Skills

| Skill | Coaching Approach | Practice |
|-------|------------------|----------|
| **Communication** | Review their written updates, suggest improvements | Write a status update, get feedback |
| **Estimation** | Compare estimates to actuals, discuss variance | Estimate 3 tasks, track accuracy |
| **Prioritization** | Discuss how you prioritize your own work | Plan a sprint, justify priorities |
| **Stakeholder management** | Share your stakeholder maps, discuss strategies | Draft a stakeholder update for review |

## The Coaching Plan

```markdown
# Coaching Plan: [Engineer Name]
# Coach: [Name]
# Period: [Start Date] — [End Date]

## Development Goal
[Specific, measurable goal]

## Current State
- Strengths: [What they're already good at]
- Growth Areas: [What needs development]
- Self-Assessment: [Their own view of their level, 1-10]

## Coaching Activities
| Activity | Frequency | Owner | Success Measure |
|----------|----------|-------|----------------|
| Pair programming | 2x/week | Coach + Coachee | Coachee can complete task independently |
| Code review coaching | Every PR | Coach | Review iterations reduce from 4 to 2 |
| Design doc writing | 1 per month | Coachee (Coach reviews) | Design doc approved with minor feedback |
| Reading assignments | Weekly | Coachee | Discussion of key learnings |

## Check-in Schedule
- Weekly: 30-min 1:1 coaching session
- Bi-weekly: Progress review against plan
- Monthly: Update plan based on progress

## Milestones
| Milestone | Target Date | Success Criteria |
|-----------|-------------|-----------------|
| [Milestone 1] | [Date] | [How we'll know it's achieved] |
| [Milestone 2] | [Date] | [How we'll know it's achieved] |
| [Final Goal] | [Date] | [How we'll know it's achieved] |

## Progress Notes
[Ongoing notes from each coaching session]
```

## Common Coaching Challenges

### Challenge: The Engineer Lacks Confidence

**Symptoms:** Second-guesses every decision, asks for permission on small choices, apologizes excessively.

**Approach:**
1. **Acknowledge the feeling.** "It's normal to feel uncertain when you're learning something new."
2. **Point to evidence.** "Your last three PRs were approved with minor feedback. That's good work."
3. **Create safe decision space.** "For the next sprint, make all decisions about error handling on your own. We'll review the outcomes together."
4. **Celebrate independent decisions.** "You handled that incident start-to-finish without asking for help. That's growth."

### Challenge: The Engineer Doesn't Accept Feedback

**Symptoms:** Defends every decision, argues with every review comment, takes feedback personally.

**Approach:**
1. **Have a meta-conversation.** "I've noticed some tension in our code review interactions. I want to understand how the feedback feels for you."
2. **Separate the person from the code.** "The feedback is about the code, not about you as an engineer."
3. **Model receiving feedback.** "In my last PR, Priya found three issues I'd missed. That made the code better. I'm grateful for it."
4. **Give feedback on the feedback reception.** "When you defend every comment, it makes reviewers hesitant to give feedback. That actually hurts your growth."

### Challenge: The Engineer Is Not Progressing

**Symptoms:** Same mistakes in review after review, no improvement after weeks of coaching.

**Approach:**
1. **Check the coaching approach.** Maybe the method doesn't match their learning style.
2. **Check for external factors.** Personal issues, burnout, mismatched role.
3. **Be direct.** "I've noticed that [specific pattern] has come up in 5 reviews over 4 weeks. Help me understand what's making it hard to change."
4. **Escalate to manager if needed.** Persistent non-progress may be a role fit issue.

## Coaching Remote Engineers

Remote coaching requires more intentionality:

| Challenge | Solution |
|-----------|----------|
| Can't look at the same screen | VS Code Live Share, screen sharing |
| Hard to read non-verbal cues | Camera on, check in explicitly: "How is this landing?" |
| Less casual coaching moments | Schedule intentional "office hours" |
| Time zone differences | Record coaching sessions for review |
| Less relationship building | Virtual coffee chats, personal check-ins before coaching |

## Measuring Coaching Effectiveness

### Indicators of Effective Coaching

| Indicator | What It Shows |
|-----------|--------------|
| Engineer ships increasingly complex work independently | Growing capability |
| Review iterations decrease | Improving quality before review |
| Engineer starts coaching others | Knowledge transfer is working |
| Engineer speaks up more in meetings | Growing confidence |
| Engineer proposes designs, not just implements | Growing ownership |

### Coaching Metrics

| Metric | Baseline | Target | Actual |
|--------|----------|--------|--------|
| PR review iterations | 4.2 | 2.0 | 2.5 |
| Independent feature delivery | 0 per sprint | 2 per sprint | 1.5 per sprint |
| Design docs written | 0 | 1 per quarter | 1 (in review) |
| Self-assessed confidence (1-10) | 3 | 7 | 6 |

## Cross-References

- `engineering-culture/mentorship.md` — Mentorship program (broader than coaching)
- `engineering-culture/pair-programming.md` — Pair programming (coaching technique)
- `engineering-culture/code-reviews.md` — Code review as coaching opportunity
- `engineering-culture/high-standards-with-empathy.md` — Standards with compassion
- `engineering-culture/psychological-safety.md` — Safe environment for learning
