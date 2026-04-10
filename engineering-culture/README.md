# Engineering Culture Overview

## What Is Engineering Culture?

Engineering culture is the set of shared values, behaviors, and practices that determine how engineering work gets done. It is not what the company says on its careers page. It is what happens when no one is watching.

In a banking GenAI organization, culture directly impacts:
- **Reliability**: Do engineers take ownership of their services in production?
- **Innovation**: Can engineers experiment with new AI techniques safely?
- **Compliance**: Is doing the right thing the default, not an afterthought?
- **Retention**: Do great engineers want to work here?

## Core Cultural Values

### 1. Ownership

Engineers own their services from design through production. "Throw it over the wall" does not exist.

```
GOOD: "I deployed the feature and am monitoring it in production."
BAD: "I finished the code. QA is testing it."
```

### 2. Blamelessness

When things go wrong (and they will), the focus is on fixing the system, not blaming the person.

```
GOOD: "How did our process allow this error to reach production?"
BAD: "Who made this mistake?"
```

### 3. Continuous Learning

Banking GenAI is a rapidly evolving field. Engineers must continuously learn about new models, techniques, and regulations.

```
GOOD: "I spent Friday afternoon evaluating the new Claude model."
BAD: "We do not have time for that. We have sprint commitments."
```

### 4. High Standards with Empathy

We hold each other to high standards because our customers depend on us. We support each other because we are humans.

```
GOOD: "This code does not meet our standards. Let me show you why and help you fix it."
BAD: "This code is terrible. Fix it."
```

### 5. Compliance as Craft

Compliance is not a burden -- it is evidence of engineering maturity. A system that can prove its own reliability is a well-designed system.

```
GOOD: "Our audit trail automatically captures every AI decision."
BAD: "Compliance is slowing us down."
```

## Code Review Culture

Code reviews are the primary mechanism for maintaining quality and sharing knowledge. See [Code Reviews](code-reviews.md) for detailed guidance.

Key principles:
- **Reviews are for learning, not gatekeeping**: The goal is to improve the code and the engineer.
- **Small PRs get better reviews**: A 200-line PR gets a thoughtful review. A 2000-line PR gets a rubber stamp.
- **Timely reviews block progress**: Review within 4 business hours. A PR waiting for review is wasted effort.

## Mentorship and Growth

See [Mentorship](mentorship.md) for the mentorship program structure.

Every engineer should have:
- A mentor (more senior engineer)
- A mentee (more junior engineer, when ready)
- A growth plan with measurable goals

## Documentation Culture

See [Documentation Culture](documentation-culture.md) for details.

Code without documentation is incomplete documentation. The question is not "should we document this?" but "what is the minimum documentation that makes this understandable?"

### Minimum Documentation

```
Every service must have:
- README with purpose, setup, and deployment instructions
- API documentation (auto-generated from code)
- Runbook for common operational tasks
- ADR for significant architectural decisions

Every feature must have:
- Design doc (for non-trivial features)
- Release notes
- Operational impact assessment
```

## Incident Culture

How a team handles incidents reveals its culture.

### During an Incident

1. **Fix first, understand later**: Restore service before investigating root cause.
2. **Communicate continuously**: Update stakeholders every 15-30 minutes.
3. **No blame**: Focus on the system, not the person.

### After an Incident

1. **Write a blameless postmortem**: See [Incident Retrospectives](incident-retrospectives.md).
2. **Create actionable follow-ups**: Every postmortem must produce specific, tracked action items.
3. **Share learnings**: Postmortems are read by other teams. Knowledge sharing prevents repeat incidents.

## Working Agreements

See [Working Agreements](working-agreements.md) for team-level agreements.

Example working agreements:
- "We deploy on Tuesdays and Thursdays, never on Fridays."
- "On-call is respected: pages are answered within 5 minutes."
- "PRs are reviewed within 4 business hours."
- "We do not commit to deadlines we cannot meet."

## Team Rituals

See [Team Rituals](team-rituals.md) for ceremony details.

```
┌─────────────────────────────────────────────────────────────┐
│  WEEKLY RHYTHM                                              │
├──────────────┬──────────────────────────────────────────────┤
│  Monday      │  Sprint planning (30 min)                    │
│  Daily       │  Standup (15 min)                            │
│  Tuesday     │  Architecture review (60 min)                │
│  Wednesday   │  Tech talk / Brown bag (45 min)              │
│  Thursday    │  1:1s with manager                           │
│  Friday      │  Demo and retrospective (30 min)             │
│              │  No deployments after 3 PM                   │
└──────────────┴──────────────────────────────────────────────┘
```

## Psychological Safety

See [Psychological Safety](psychological-safety.md) for details.

Psychological safety is the foundation of high-performing teams. Engineers must feel safe to:
- Admit mistakes
- Ask questions
- Challenge decisions
- Propose ideas
- Say "I do not know"

Without psychological safety, engineers hide mistakes, avoid risks, and leave.

## Measuring Culture

Culture is not fuzzy. Measure it:

```
┌─────────────────────────────────────────────────────────────┐
│  CULTURE METRICS                                            │
├──────────────────────────┬──────────────────────────────────┤
│  Metric                  │  Target                          │
├──────────────────────────┼──────────────────────────────────┤
│  PR review time (median)│  < 4 business hours              │
│  Deploy frequency        │  Daily or on-demand              │
│  Lead time for changes   │  < 1 week from PR to production  │
│  Incident postmortem rate │  100% of incidents get postmortem│
│  Team health survey      │  > 4.0 / 5.0                     │
│  Retention rate          │  > 90% annually                  │
│  Internal mobility       │  > 20% of engineers change role  │
│                          │  or team annually                │
└──────────────────────────┴──────────────────────────────────┘
```

## Table of Contents

- [Code Reviews](code-reviews.md)
- [Mentorship](mentorship.md)
- [Knowledge Sharing](knowledge-sharing.md)
- [Design Docs](design-docs.md)
- [RFCs](rfcs.md)
- [Incident Retrospectives](incident-retrospectives.md)
- [Working Agreements](working-agreements.md)
- [Team Rituals](team-rituals.md)
- [Async Communication](async-communication.md)
- [Documentation Culture](documentation-culture.md)
- [Pair Programming](pair-programming.md)
- [Cross-Functional Collaboration](cross-functional-collaboration.md)
- [Building Trust](building-trust.md)
- [Psychological Safety](psychological-safety.md)
- [High Standards with Empathy](high-standards-with-empathy.md)
