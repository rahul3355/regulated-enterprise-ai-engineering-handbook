# Psychological Safety

> "Psychological safety is not about being nice. It's about creating an environment where people can speak up, take risks, and be vulnerable — without fear of punishment or humiliation."

## What Is Psychological Safety?

Psychological safety is the shared belief that the team is safe for interpersonal risk-taking. It means:

- You can admit a mistake without fear of punishment
- You can ask a "stupid" question without being ridiculed
- You can disagree with a senior person without career consequences
- You can propose a wild idea without being laughed at
- You can say "I don't know" or "I need help"

**It does NOT mean:**
- Everyone is always nice
- No one is held accountable
- Conflict is avoided
- Performance standards are lowered
- Everyone gets a trophy

## Why Psychological Safety Matters in Banking GenAI

In a high-stakes, high-pressure environment like banking GenAI engineering, psychological safety is not a luxury — it's a **safety requirement**.

Without psychological safety:

| Scenario | With Safety | Without Safety |
|----------|------------|----------------|
| **Engineer makes a mistake** | Reports it immediately, team fixes the system | Hides it, hopes nobody notices, it becomes an incident |
| **Engineer sees a design flaw** | Speaks up in the design review | Stays quiet, "not my decision," flaw causes production issues |
| **Engineer doesn't understand a requirement** | Asks for clarification | Guesses, builds the wrong thing, wastes weeks |
| **Engineer disagrees with architecture decision** | Explains their reasoning, team debates | Silently implements their own approach, creates divergence |
| **Model produces problematic output** | Flags it for investigation | Ignores it, "the model is always right," bad output reaches users |

In banking, the "without safety" column is where regulatory violations, data breaches, and reputational damage happen.

## The Four Stages of Psychological Safety

```
Stage 1: INCLUSION SAFETY
"I belong here."
├── You feel accepted as a member of the team
├── You can be yourself without fear of exclusion
└── Foundation for all other stages

Stage 2: LEARNER SAFETY
"I can learn and grow here."
├── You can ask questions without feeling stupid
├── You can make mistakes as part of learning
└── Feedback is given constructively

Stage 3: CONTRIBUTOR SAFETY
"I can contribute my own ideas."
├── You can share your perspectives and ideas
├── You can use your skills and strengths
└── Your contributions are valued

Stage 4: CHALLENGER SAFETY
"I can challenge the status quo."
├── You can question existing practices
├── You can suggest changes, even unpopular ones
└── You can disagree with authority
```

Most teams are at Stage 2 or 3. Stage 4 is rare and powerful — it's where innovation happens.

## Building Psychological Safety

### For Team Leads and Managers

| Action | How | Why |
|--------|-----|-----|
| **Model vulnerability** | Admit your own mistakes publicly | Shows it's safe to be imperfect |
| **Respond productively to mistakes** | "What can we learn?" not "Who did this?" | Determines whether people report future mistakes |
| **Ask for input actively** | "What am I missing?" "Does anyone see this differently?" | Invites dissenting opinions |
| **Reward candor** | Thank people who deliver bad news or disagree | Reinforces the behavior you want |
| **Never shoot the messenger** | Even if the message is uncomfortable | One instance of punishment kills safety for months |
| **Create structured opportunities for input** | Round-robin in meetings, anonymous surveys | Not everyone speaks up voluntarily |
| **Follow through on feedback** | When someone raises an issue, address it | Shows that speaking up leads to change |

### For Individual Contributors

| Action | How | Why |
|--------|-----|-----|
| **Be curious, not judgmental** | "Help me understand your thinking" instead of "That's wrong" | Invites dialogue instead of defensiveness |
| **Admit your own gaps** | "I don't know enough about this — can someone explain?" | Models the behavior you want from others |
| **Acknowledge others' contributions** | "That's a great point, I hadn't thought of that" | Validates participation |
| **Speak up when you see silence** | "We haven't heard from Priya yet — Priya, what do you think?" | Draws out quiet voices |
| **Push back on blame language** | "Let's focus on what in our system allowed this" | Redirects to systems thinking |

### For Incident Response

Incidents are the ultimate test of psychological safety:

**Do:**
- Thank the person who discovered the issue
- Thank the person who caused it (they'll likely never report again if they're blamed)
- Focus on the chain of events and conditions, not the individual
- Ask "what were you seeing/thinking at the time?" not "why did you do that?"
- Share learnings broadly

**Don't:**
- Ask "how did this happen?" in a tone that implies negligence
- Use language like "dropped the ball," "fell through the cracks," "should have known"
- Allow anyone to say "I would have caught that" (hindsight bias)
- Discuss individual performance in the retrospective

## Measuring Psychological Safety

### Team Health Survey Questions

These questions (answered anonymously) measure psychological safety:

| Question | What It Measures |
|----------|-----------------|
| "If I make a mistake on this team, it is not held against me." | Mistake tolerance |
| "Members of this team are able to bring up problems and tough issues." | Candor safety |
| "People on this team sometimes reject others for being different." (reverse) | Inclusion safety |
| "It is safe to take a risk on this team." | Risk tolerance |
| "I can ask for help when I need it." | Help-seeking safety |
| "No one on this team would deliberately act in a way that undermines my efforts." | Trust |
| "My unique skills and talents are valued on this team." | Contribution safety |
| "I can challenge the team's approach if I disagree." | Challenger safety |

### Behavioral Indicators

| Behavior | Safety Level |
|----------|-------------|
| People report their own mistakes | High |
| People ask "stupid" questions in public channels | High |
| Junior engineers disagree with senior engineers | High |
| Incidents are reported quickly | High |
| Retrospectives generate honest feedback | High |
| | |
| People hide mistakes | Low |
| Meetings are dominated by 1-2 voices | Low |
| Disagreements happen in DMs, not in the open | Low |
| Incidents are discovered by users, not the team | Low |
| Retrospectives are silent or performative | Low |

## Psychological Safety and Performance

A common fear: "If we create psychological safety, will people become complacent?"

**Research answer:** No. Psychological safety and performance standards are not in tension — they're complementary.

```
High Standards + Low Safety = Anxiety
    People are afraid to make mistakes, but are held to a high bar.
    Result: Burnout, hidden mistakes, turnover.

Low Standards + High Safety = Comfort Zone
    People feel safe but aren't pushed to grow.
    Result: Stagnation, mediocrity.

Low Standards + Low Safety = Apathy
    People don't feel safe and aren't pushed to grow.
    Result: Disengagement, worst outcome.

High Standards + High Safety = High Performance
    People are pushed to excel AND feel safe to take risks.
    Result: Learning, innovation, growth. ← This is the goal.
```

**The key insight:** Psychological safety is not about lowering standards. It's about creating the conditions where people can meet high standards without fear.

## Psychological Safety in Code Reviews

Code reviews are a common place where psychological safety breaks down:

**Unsafe code review:**
> "This code has multiple issues. The approach is wrong, there's no error handling, and the variable names are meaningless. Please rewrite."

**Safe code review:**
> "Thanks for putting this together. I have a few suggestions: (1) What happens if the model API returns an error? We might need a retry here. (2) Could we use `response_text` instead of `x` for clarity? (3) Have you considered using the circuit breaker pattern here? Happy to pair on this if it's helpful."

**The difference:** Both reviews identify the same issues. The safe review treats the author as a competent professional who made reasonable choices and invites collaboration on improvements.

## Handling Low Psychological Safety

If your team has low psychological safety:

### Immediate Actions

1. **Name it.** "I've noticed that people aren't speaking up in meetings. I want to understand why."
2. **Model the behavior you want.** Admit a mistake. Ask a "stupid" question. Disagree with someone respectfully.
3. **Create structured input.** "Before we move on, I want to hear from everyone. Go around the room."
4. **Thank people who take risks.** When someone speaks up, acknowledge it: "Thank you for bringing that up."

### Longer-Term Actions

1. **Work with the team lead/manager.** Psychological safety starts with leadership.
2. **Address specific behaviors.** If someone regularly shuts down others, that needs to be addressed directly.
3. **Bring in an external facilitator.** Sometimes an outside perspective is needed.
4. **Track progress.** Survey the team quarterly on safety measures.

## Psychological Safety for Remote Teams

Remote work makes psychological safety harder but more important:

| Challenge | Solution |
|-----------|----------|
| Harder to read the room | Use video, pay attention to tone in text |
| Silence is invisible | Actively solicit input from quiet members |
| DMs replace open discussion | Encourage public discussion, discourage decision-making in DMs |
| Time zone isolation | Ensure everyone has a buddy in their timezone |
| "Camera fatigue" | Allow camera-off for some meetings, but camera-on for important ones |

## Cross-References

- `engineering-culture/incident-retrospectives.md` — Blameless retrospectives require psychological safety
- `engineering-culture/code-reviews.md` — Safe code review practices
- `engineering-culture/building-trust.md` — Trust building (closely related)
- `engineering-culture/high-standards-with-empathy.md` — High standards with compassion
- `leadership-and-collaboration/coaching-junior-engineers.md` — Coaching in a safe environment
