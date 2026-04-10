# Pair Programming

> "Two brains, one keyboard. Pair programming isn't about writing code faster — it's about writing better code and sharing knowledge faster than any other method."

## What Pair Programming Is (and Isn't)

**Pair programming is:** Two engineers working together on the same code, at the same time, with one goal.

**Pair programming is NOT:**
- One person typing while the other watches silently
- A teaching session where one person lectures
- A code review after the code is written
- A way to "supervise" a junior engineer

## Pair Programming Models

### Driver-Navigator (Most Common)

```
Driver (types)                    Navigator (thinks)
─────────────────                ──────────────────────
- Writes the code                - Sets the direction
- Focuses on syntax              - Thinks about design
- Handles the mechanics          - Anticipates problems
- Implements the details         - Reviews as code is written
```

**How it works:**
1. Navigator describes what needs to be done
2. Driver writes the code
3. Navigator reviews each line as it's written
4. Both discuss alternatives as they arise
5. Roles swap every 25-30 minutes

**When to use:** Knowledge transfer, complex debugging, onboarding.

### Strong-Style Pairing

> "For an idea to go from your head into the computer, it must go through someone else's hands."

**How it works:**
- The less experienced person drives
- The more experienced person navigates
- The navigator doesn't touch the keyboard
- Knowledge flows from navigator to driver through the code

**When to use:** Mentoring junior engineers, building team capability.

### Ping-Pong Pairing

**How it works:**
1. Person A writes a failing test
2. Person B writes code to make the test pass
3. Person B writes the next failing test
4. Person A writes code to make it pass
5. Repeat

**When to use:** TDD sessions, building well-tested features, keeping both people engaged.

### Tour Guide Pairing

**How it works:**
- The expert drives and narrates
- The learner asks questions and observes
- The expert explains decisions as they make them

**When to use:** Very first introduction to an unfamiliar codebase or technology. Should transition to Driver-Navigator as soon as possible.

## When to Pair Program

### High-Value Pairing Scenarios

| Scenario | Why Pair |
|----------|----------|
| Onboarding new engineers | Fastest way to learn the codebase and culture |
| Complex debugging | Two perspectives on a hard problem |
| Security-critical code | Extra eyes on auth, encryption, data handling |
| GenAI prompt engineering | Two minds on prompt quality and edge cases |
| Learning a new technology | Shared learning, shared notes |
| Cross-team collaboration | Knowledge transfer between teams |
| Tackling a gnarly refactor | Shared confidence for risky changes |
| Designing a complex algorithm | Real-time design review |

### Low-Value Pairing Scenarios

| Scenario | Why Not |
|----------|---------|
| Simple CRUD endpoint | Straightforward, no learning opportunity |
| Documentation updates | Can be done independently efficiently |
| Configuration changes | Doesn't benefit from two perspectives |
| Repetitive tasks | Boring for the navigator |

## Remote Pair Programming Setup

Since our teams are distributed, remote pairing is the norm:

### Tool Stack

| Tool | Purpose | Alternatives |
|------|---------|-------------|
| VS Code Live Share | Shared editing | CodeTogether, GitPod |
| Video call (camera on) | Non-verbal communication | Zoom, Meet, Teams |
| Shared terminal | Command execution | tmate, Tmux + SSH |
| Shared whiteboard | Architecture diagrams | Excalidraw, Miro |

### Environment Setup

```
Physical Setup:
├── Dual monitors (one for code, one for docs/video)
├── Good microphone (headset preferred)
├── Camera at eye level
├── Minimal background noise
└── Reliable internet connection

Software Setup:
├── Same editor (VS Code with Live Share extension)
├── Same linter/formatter configuration
├── Shared terminal access
└── Shared browser for documentation
```

### Remote Pairing Etiquette

1. **Camera on.** Non-verbal cues matter. If you can't have camera on, acknowledge it.
2. **Good audio.** Invest in a decent microphone. Bad audio is exhausting.
3. **Take breaks.** 5 minutes every 25 minutes. Two brains working for 2 hours need more breaks than one.
4. **Switch roles.** Every 25-30 minutes, swap driver and navigator.
5. **No multitasking.** If you're pairing, you're pairing. No checking email on the side.
6. **Prepare before.** Both people should understand the goal before the session starts.
7. **Debrief after.** 5 minutes at the end: what did we learn? what's next?

## Pair Programming Rotation

### How We Rotate

For ongoing pairing (e.g., building a feature together):

```
Week 1: Alice + Bob
Week 2: Alice + Carol
Week 3: Bob + Carol
Week 4: Bob + David
...
```

**Benefits:**
- Knowledge spreads across the team
- No single knowledge silo
- Fresh perspectives on the code
- Stronger team relationships

**Guidelines:**
- No two people should pair for more than 2 consecutive weeks
- Everyone should pair with everyone else on the team at least once per quarter
- Junior engineers should rotate through all senior engineers

## Measuring Pair Programming Effectiveness

### What We Track

| Metric | Why |
|--------|-----|
| Pairing hours per engineer per week | Is pairing happening regularly? |
| Cross-team pairing | Is knowledge spreading beyond the team? |
| Defect rate in paired code vs. solo code | Is pairing improving quality? |
| Onboarding time with vs. without pairing | Is pairing accelerating learning? |

### What We Don't Track

| Metric | Why Not |
|--------|---------|
| Lines of code per pair hour | Perverse incentive, discourages thoughtful coding |
| Number of pairs formed | Quantity over quality |
| Time to complete feature | Pairing often takes longer initially but produces better code |

## Common Pairing Challenges

### Challenge: One Person Dominates

**Symptoms:** Navigator talks 90% of the time, or Driver types everything without input.

**Fix:**
- Use a timer for role rotation
- Strong-Style: navigator is forbidden from typing
- Navigator asks questions instead of giving directions: "What if we tried X?"
- Check in: "How is this feeling for you?"

### Challenge: Personality Clash

**Symptoms:** Pairing sessions feel tense or unproductive.

**Fix:**
- Acknowledge it's okay — not everyone pairs well with everyone
- Try different pairing models (maybe Ping-Pong works better than Driver-Navigator)
- If it persists, don't force it — rotate to different pairs
- Consider: is this a communication skill gap that coaching could address?

### Challenge: Different Skill Levels

**Symptoms:** One person is bored, the other is lost.

**Fix:**
- Strong-Style pairing (junior drives, senior navigates)
- Senior navigator asks questions instead of telling: "What do you think comes next?"
- Set explicit learning goals for the session
- Take breaks more frequently — cognitive load is higher for the learner

### Challenge: "We Don't Have Time to Pair"

**Reality check:** Pairing feels slower in the moment but is faster overall because:
- Fewer bugs reach production
- Knowledge transfer reduces future questions
- Code is more maintainable (designed, not just written)
- Onboarding is accelerated

**Data point:** Studies show paired code has 15-50% fewer defects. In banking, where a production defect can mean a compliance violation, this is significant.

## Pairing and Neurodiversity

Pair programming can be challenging for neurodivergent engineers (ADHD, autism, social anxiety). We accommodate:

| Need | Accommodation |
|------|--------------|
| Sensory sensitivity | Allow breaks, reduce background noise, text-based pairing option |
| Social battery limits | Shorter sessions (15 min instead of 30), more frequent breaks |
| Processing differences | Provide context before the session, allow written communication alongside verbal |
| Focus needs | Clear agenda for the session, defined goals |

**Key principle:** Pairing should be a positive experience for both people. If it's consistently negative for someone, we find a different approach.

## Pair Programming Checklist

### Before the Session

- [ ] Both people understand the goal
- [ ] Environment is set up (editor sharing, video, audio)
- [ ] Time is blocked on both calendars
- [ ] Breaks are planned (every 25-30 minutes)

### During the Session

- [ ] Roles are clear (driver/navigator or other model)
- [ ] Both people are engaged (no multitasking)
- [ ] Navigator thinks aloud
- [ ] Driver asks questions when uncertain
- [ ] Roles are rotated regularly

### After the Session

- [ ] Debrief: what did we learn?
- [ ] Commit and push the code
- [ ] Note any follow-up items
- [ ] Schedule next session if needed

## Cross-References

- `engineering-culture/mentorship.md` — Mentorship program (pairing is a mentorship tool)
- `engineering-culture/code-reviews.md` — Code review (pairing is real-time review)
- `engineering-culture/knowledge-sharing.md` — Knowledge sharing (pairing spreads knowledge)
- `leadership-and-collaboration/coaching-junior-engineers.md` — Coaching (pairing is a coaching technique)
- `engineering-philosophy/craft.md` — Engineering craft (pairing improves code quality)
