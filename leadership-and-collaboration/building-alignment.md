# Building Alignment

> "Alignment is not agreement. Alignment is commitment to a shared direction, even when people have different opinions about the best path."

## The Alignment Problem

In a banking GenAI platform, no team operates in isolation. Decisions affect:

- Other engineering teams (API contracts, shared infrastructure)
- Security and compliance teams (review workload, risk exposure)
- Product teams (feature dependencies, user impact)
- Platform teams (resource requirements, operational burden)

**Without alignment:** Teams build incompatible systems, duplicate work, or block each other.
**With alignment:** Teams move in the same direction, even if they started with different ideas.

## Alignment vs. Consensus

| Aspect | Consensus | Alignment |
|--------|-----------|-----------|
| **Definition** | Everyone agrees it's the best option | Everyone commits to the chosen direction |
| **Requirement** | Unanimous agreement | Willingness to support the decision |
| **Speed** | Slow (everyone must be convinced) | Faster (decision-maker decides after input) |
| **Quality** | Can be mediocre (lowest common denominator) | Can be excellent (best option, fully supported) |
| **When to use** | Small, reversible decisions | Significant, cross-team decisions |

**Goal:** Alignment, not consensus. Seek input from everyone, but don't require everyone to agree.

## The Alignment Process

### Step 1: Frame the Problem (Not the Solution)

Start with a shared understanding of the problem before discussing solutions:

```markdown
# Problem Frame: [Topic]

## What problem are we solving?
[Clear, specific problem statement]

## Why does this matter?
[Business/technical/user impact]

## Who is affected?
[Teams, users, systems]

## What constraints exist?
[Technical, regulatory, timeline, resource]

## What would success look like?
[Measurable outcomes]
```

**Example:**

```markdown
# Problem Frame: Model Provider Selection

## What problem are we solving?
Our current LLM provider (OpenAI) has announced pricing changes
that would increase our monthly costs by 40%. We need to evaluate
alternative providers to maintain our budget.

## Why does this matter?
At current usage, the price increase adds $180K/year to our
operating costs. This threatens the business case for the GenAI
platform.

## Who is affected?
- All GenAI platform users (response quality may change)
- Security team (new provider = new security review)
- Compliance team (data handling may differ)
- Engineering team (integration work required)

## What constraints exist?
- Data must not leave US/EU regions (compliance)
- Provider must support our required model capabilities
- Migration must complete within Q2 (budget cycle)
- Any new provider must pass security review

## What would success look like?
- Cost reduction of at least 20% vs. current pricing
- Response quality maintained or improved
- Zero user-visible disruption during migration
- Security and compliance sign-off
```

### Step 2: Gather Input

Collect perspectives from all affected parties:

| Method | When | Who |
|--------|------|-----|
| 1:1 conversations | Early, before positions harden | Key stakeholders, dissenting voices |
| Group discussion | After individual input | Full team, affected teams |
| Written proposals | For complex topics | Each option's advocate writes a brief |
| Prototypes/spikes | When data is needed | Engineering team |

### Step 3: Present Options

Present options honestly, including the one you don't prefer:

```markdown
# Options Analysis: [Topic]

## Option A: [Description]
- **Pros:** [...]
- **Cons:** [...]
- **Effort:** [...]
- **Risk:** [...]

## Option B: [Description]
- **Pros:** [...]
- **Cons:** [...]
- **Effort:** [...]
- **Risk:** [...]

## Option C: Status Quo
- **Pros:** No change effort
- **Cons:** [Why the current situation is untenable]
- **Effort:** 0
- **Risk:** [Risk of not changing]

## Recommendation
[Your recommended option with rationale]
```

### Step 4: Address Concerns

Every concern deserves a response:

```markdown
# Concerns and Responses

| # | Concern | Raised By | Response | Resolution |
|---|---------|-----------|----------|-----------|
| 1 | New provider may have worse quality | Product team | Benchmarks show comparable quality on our document set | Accepted — will validate with user testing |
| 2 | Security review will take 4 weeks | Security team | We've started the review process and provided full technical details | Accepted — timeline adjusted |
| 3 | Migration may cause downtime | Platform team | We'll run both providers in parallel during migration | Accepted — dual-provider support added to plan |
```

### Step 5: Decide

Someone must decide. The decision-maker depends on scope:

| Scope | Decision-Maker |
|-------|---------------|
| Team-internal | Tech lead |
| Cross-team | Engineering manager or director |
| Strategic | VP/CTO |
| Security-significant | Security lead + engineering lead |
| Compliance-significant | Compliance lead + engineering lead |

The decision-maker:
1. Reviews all input
2. Makes a clear decision
3. Documents the rationale
4. Acknowledges dissenting views

### Step 6: Commit

Once the decision is made, everyone commits:

```markdown
# Decision: [What was decided]

## Rationale
[Why this option was chosen]

## Dissenting Views
[What concerns remain, how they're being addressed]

## Commitment
We are aligned on this decision. Even those who had concerns
commit to supporting the implementation and making it succeed.

## Revisit Criteria
We will re-evaluate if:
- [Specific condition that would trigger re-evaluation]
- [Specific metric that, if missed, means we should reconsider]
```

## Alignment Techniques

### Pre-Wiring

Before a big decision meeting, talk to key stakeholders individually:

```
"I'm planning to propose X at Thursday's meeting. I wanted to
get your thoughts beforehand. What concerns do you have? Is
there anything that would make this easier for you to support?"

Benefits:
- You hear concerns before they're public
- You can adjust the proposal before presenting it
- People feel heard, not ambushed
- The actual meeting is more efficient
```

### Finding the "Third Option"

When two parties are stuck on opposing options:

```
Party A: "We should use approach X."
Party B: "We should use approach Y."

Mediator: "What problem is X trying to solve for you?"
Party A: "We need low latency."

Mediator: "What problem is Y trying to solve for you?"
Party B: "We need type safety."

Mediator: "Is there an approach that gives us both low latency
and type safety? What about gRPC with protobuf?"

Third option identified. Both parties' underlying needs are met.
```

### The "Disagree and Commit" Ask

```
"I know you prefer Option A, and I understand your reasoning.
I've decided on Option B because [rationale]. I'm not asking you
to agree that B is better. I'm asking you to commit to making B
succeed. Can you do that?"

If they say yes: Great. Follow through.
If they say no: Understand why. Is there a real blocker?
```

### The "Sincerity Test"

After a decision is made, check for genuine alignment:

```
In the next team meeting:
"We decided on X last week. I want to check in — how is everyone
feeling about that direction? Any concerns that haven't been
addressed? Anyone having trouble committing to this?"

Watch for:
- Enthusiastic support: Good alignment
- Quiet acceptance: Adequate alignment
- Continued advocacy for the rejected option: Poor alignment —
  needs a follow-up conversation
```

## Alignment Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact |
|--------------|-------------------|--------|
| **Design by committee** | Everyone must agree, so the decision is the lowest common denominator | Mediocre decisions that nobody is excited about |
| **Dictatorship** | One person decides without input | People don't commit to decisions they didn't shape |
| **False alignment** | People nod in the meeting but don't follow through | Decisions are made but not executed |
| **Re-litigating** | After a decision, people keep arguing for their preferred option | Undermines the decision, wastes time |
| **Ambush decisions** | Surprising people with a decision they weren't part of | Resistance, resentment |
| **Infinite discussion** | "Let's discuss more" for weeks without deciding | Paralysis, missed deadlines |

## Measuring Alignment

| Indicator | Aligned Team | Misaligned Team |
|-----------|-------------|-----------------|
| **After a decision** | Everyone supports it publicly | Some continue advocating their position |
| **Implementation** | Teams build compatible systems | Teams build incompatible systems |
| **Communication** | Teams reference the decision accurately | Teams have different understanding of the decision |
| **Speed** | Decisions lead to action | Decisions are questioned and delayed |
| **Retrospectives** | "We chose well" or "We chose wrong, let's learn" | "I told you so" or silent resentment |

## Building Alignment Across Teams

### Cross-Team Alignment Checklist

- [ ] All affected teams have been identified
- [ ] Each team has been consulted individually
- [ ] Concerns from each team are documented and addressed
- [ ] A single decision-maker has been identified
- [ ] The decision is documented and shared
- [ ] Each team has a named owner for their part
- [ ] Timeline and dependencies are agreed upon
- [ ] Escalation path is defined if teams fall out of alignment

### The Alignment Document

```markdown
# Cross-Team Alignment: [Topic]

## Decision
[What was decided, by whom, when]

## Team Commitments

### GenAI Platform Team
- Build the dual-provider abstraction layer
- Deliver by May 15
- Owner: @sarah

### Security Team
- Complete security review of new provider by April 20
- Owner: @james

### Compliance Team
- Assess data handling implications by April 25
- Owner: @priya

### Platform Team
- Provision infrastructure for new provider by April 30
- Owner: @alex

## Dependencies
| Team A | Depends on | Team B | Due Date |
|--------|-----------|--------|----------|
| GenAI Platform | Security review complete | Security | April 20 |
| GenAI Platform | Infra provisioned | Platform | April 30 |
| Product | Migration complete | GenAI Platform | May 15 |

## Escalation
If any team cannot meet their commitment, they escalate to
[Name] by [Date] to allow replanning.

## Check-in Cadence
- Weekly sync: Thursdays 2 PM
- Status updates: Written, Mondays
```

## Cross-References

- `leadership-and-collaboration/influencing-without-authority.md` — Influence strategies
- `leadership-and-collaboration/negotiation.md` — Technical negotiation
- `leadership-and-collaboration/running-technical-meetings.md` — Running alignment meetings
- `engineering-culture/rfcs.md` — RFC process (lightweight alignment)
- `engineering-culture/design-docs.md` — Design docs (detailed alignment)
- `leadership-and-collaboration/writing-strategy-docs.md` — Strategy document writing
