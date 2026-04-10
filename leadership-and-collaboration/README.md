# Leadership in Engineering

## What Is Engineering Leadership?

Engineering leadership is not a title. It is a set of behaviors that any engineer can exhibit, regardless of seniority. Leadership means:

- Making technical decisions and standing behind them
- Influencing direction without requiring authority
- Elevating the engineers around you
- Communicating technical information to non-technical stakeholders
- Driving initiatives from idea to production

In a banking GenAI organization, leadership is particularly important because the field moves faster than any single person can track. Leaders create clarity from ambiguity.

## Leadership at Every Level

```
┌─────────────────────────────────────────────────────────────┐
│  LEADERSHIP BEHAVIORS BY LEVEL                               │
├────────────┬────────────────────────────────────────────────┤
│  Junior    │  - Own your tasks end-to-end                   │
│            │  - Ask questions when stuck                    │
│            │  - Share what you learn                        │
│            │  - Review code thoughtfully                    │
├────────────┼────────────────────────────────────────────────┤
│  Mid       │  - Design features independently               │
│            │  - Mentor junior engineers                     │
│            │  - Identify and raise risks early              │
│            │  - Improve team processes                      │
├────────────┼────────────────────────────────────────────────┤
│  Senior    │  - Set technical direction for services        │
│            │  - Coordinate across teams                     │
│            │  - Make architecture decisions                 │
│            │  - Represent team to stakeholders              │
├────────────┼────────────────────────────────────────────────┤
│  Staff+    │  - Set technical direction for org             │
│            │  - Solve problems that span multiple teams     │
│            │  - Develop senior engineers                    │
│            │  - Influence executive strategy                │
└────────────┴────────────────────────────────────────────────┘
```

## Core Leadership Skills

### 1. Influencing Without Authority

Most engineering leadership happens without formal authority. You cannot order someone to adopt your library. You must persuade them. See [Influencing Without Authority](influencing-without-authority.md).

Key tactics:
- **Build relationships**: People adopt ideas from people they trust.
- **Demonstrate value**: Show, do not tell. A working prototype is more persuasive than a design doc.
- **Understand their incentives**: What does the other team care about? Frame your proposal in their terms.

### 2. Stakeholder Management

Engineering does not happen in a vacuum. Stakeholders include product managers, designers, security teams, compliance officers, and executives. See [Managing Stakeholders](managing-stakeholders.md).

Key practices:
- Map stakeholders by influence and interest
- Communicate proactively, not reactively
- Tailor communication to the audience

### 3. Technical Decision-Making

Leaders make decisions. Good leaders make decisions that survive contact with reality. Key practices:
- Gather input from those closest to the problem
- Document decisions with ADRs
- Be decisive but willing to change course with new information

### 4. Communication

Engineering leaders communicate in multiple formats:
- **Written**: Design docs, RFCs, status updates, postmortems
- **Verbal**: Meetings, presentations, 1:1s
- **Code**: The code itself is communication

See [Communicating with Executives](communicating-with-executives.md) for executive-specific guidance.

## Driving Technical Initiatives

See [Driving Technical Initiatives](driving-technical-initiatives.md) for a detailed guide.

The pattern for driving any technical initiative:

```
1. Identify the problem
   -> "Our LLM costs are 40% over budget."

2. Propose a solution
   -> "Implement response caching and model routing."

3. Build consensus
   -> Write a design doc. Present to the team. Get feedback.

4. Execute incrementally
   -> Implement caching first. Measure impact. Then add routing.

5. Communicate progress
   -> Weekly updates. Demo results. Share metrics.

6. Close the loop
   -> Document the result. Share learnings. Update runbooks.
```

## Coaching and Mentorship

See [Coaching Junior Engineers](coaching-junior-engineers.md).

Every engineer benefits from coaching. Great leaders coach everyone, not just their direct reports.

Coaching is not the same as managing:
- **Managing**: "Here is what you should do."
- **Coaching**: "What do you think you should do? Let me help you think it through."

## Conflict Resolution

See [Conflict Resolution](conflict-resolution.md).

Technical conflict is healthy. Personal conflict is not. Leaders distinguish between the two and handle each appropriately.

```
Healthy:   "I think we should use pgvector because it simplifies operations."
           "I disagree. Milvus has better performance. Let me show you the benchmarks."

Unhealthy: "Your architecture is always overcomplicated."
           "You never consider operational complexity."
```

## Building Alignment

See [Building Alignment](building-alignment.md).

Alignment does not mean everyone agrees. It means everyone understands the decision and commits to it, even if they initially disagreed.

```
Disagree and commit:
  "I preferred the Milvus approach, but the team decided on pgvector.
   I fully support the decision and will make it work."
```

## Leadership in Banking GenAI Context

### Unique Challenges

1. **Regulatory pressure**: Decisions have compliance implications. Leaders must understand regulations and translate them into engineering requirements.

2. **Rapidly evolving technology**: LLM capabilities change monthly. Leaders must balance adopting new techniques with maintaining stability.

3. **Multi-disciplinary teams**: GenAI teams include software engineers, ML engineers, data engineers, and domain experts. Leaders must bridge these disciplines.

4. **Risk management**: In banking, a wrong decision can cost millions. Leaders must be comfortable making decisions under uncertainty while managing risk.

### Leadership Opportunities

Every engineer can lead in these areas:
- **Model evaluation**: Lead the effort to evaluate new models for production use
- **Cost optimization**: Drive initiatives to reduce LLM spending
- **Quality improvement**: Champion LLM evaluation frameworks and golden datasets
- **Compliance automation**: Build tools that make compliance automatic
- **Knowledge sharing**: Organize tech talks on new GenAI techniques

## Table of Contents

- [Influencing Without Authority](influencing-without-authority.md)
- [Managing Stakeholders](managing-stakeholders.md)
- [Communicating with Executives](communicating-with-executives.md)
- [Running Technical Meetings](running-technical-meetings.md)
- [Giving Status Updates](giving-status-updates.md)
- [Handling Blockers](handling-blockers.md)
- [Escalation Strategies](escalation-strategies.md)
- [Negotiation](negotiation.md)
- [Conflict Resolution](conflict-resolution.md)
- [Coaching Junior Engineers](coaching-junior-engineers.md)
- [Building Alignment](building-alignment.md)
- [Writing Strategy Docs](writing-strategy-docs.md)
- [Writing Executive Summaries](writing-executive-summaries.md)
- [Driving Technical Initiatives](driving-technical-initiatives.md)
