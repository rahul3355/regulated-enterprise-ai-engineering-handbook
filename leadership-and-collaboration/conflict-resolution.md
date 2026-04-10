# Conflict Resolution

> "Conflict is inevitable in engineering. Disagreements about architecture, priorities, and approaches are signs of a team that cares. The question is whether conflict produces better decisions or broken relationships."

## Types of Engineering Conflict

### Technical Conflicts

| Type | Example | Resolution Approach |
|------|---------|-------------------|
| **Architecture disagreement** | REST vs. gRPC, monolith vs. microservices | Data-driven evaluation, prototype, decide |
| **Implementation approach** | Build vs. buy, which library to use | Benchmark, evaluate trade-offs, decide |
| **Code style/patterns** | Naming conventions, error handling patterns | Agree on standards, enforce with tooling |
| **Performance vs. readability** | Optimized but complex code vs. clear but slower | Context-dependent, document the trade-off |

### Process Conflicts

| Type | Example | Resolution Approach |
|------|---------|-------------------|
| **Timeline disagreement** | "We can ship in 2 weeks" vs. "It needs 4" | Break down the work, estimate together |
| **Priority conflict** | "This is the most important thing" vs. "No, this is" | Refer to team/organizational priorities |
| **Quality bar** | "This is good enough" vs. "This needs more work" | Define objective quality criteria |
| **Ownership** | "That's your team's responsibility" vs. "No, it's yours" | Clarify RACI, escalate if needed |

### Interpersonal Conflicts

| Type | Example | Resolution Approach |
|------|---------|-------------------|
| **Communication style clash** | Direct vs. diplomatic, written vs. verbal | Acknowledge differences, adapt |
| **Credit and recognition** | "I did most of the work" | Acknowledge all contributions, move forward |
| **Trust breakdown** | "They always drop the ball" | Address specific instances, rebuild through delivery |
| **Respect issues** | Dismissive comments, talking over people | Direct conversation, mediation if needed |

## Conflict Resolution Framework

### Step 1: Direct Conversation (Between the Parties)

Most conflicts should be resolved directly between the people involved:

```
"I'd like to talk about [specific issue]. I noticed [observable
behavior] and it had [impact] on me/the work. I'd like to understand
your perspective and figure out how we can work better together."

Key principles:
- Focus on behavior, not personality
- Use "I" statements, not "you" statements
- Listen to understand, not to respond
- Seek the underlying interest, not just the position
```

**Example:**

> **Instead of:** "You never respond to my PR reviews. You're blocking my work."
>
> **Say:** "I've noticed my last three PR reviews haven't had comments from you. I'm not sure if you haven't had time or if there's something about the approach you disagree with. Can we talk about it?"

### Step 2: Facilitated Discussion (With a Neutral Third Party)

If direct conversation doesn't resolve it:

```
A neutral third party (tech lead, engineering manager, or trusted
colleague) facilitates a structured conversation:

1. Each person shares their perspective (without interruption)
2. The facilitator summarizes both views
3. Together, identify the underlying interests
4. Brainstorm options that address both parties' needs
5. Agree on a path forward
```

**Facilitator rules:**
- No interruptions
- No character attacks ("you always," "you never")
- Focus on the future, not the past
- Document the agreement

### Step 3: Management Decision

If facilitated discussion doesn't resolve it:

```
The manager makes a decision after:
1. Hearing both sides
2. Understanding the impact on the work
3. Considering team dynamics
4. Making a clear, documented decision

Both parties are expected to "disagree and commit."
```

## Resolving Technical Conflicts

### The Data-Driven Approach

For technical disagreements, data usually resolves it:

```
Step 1: Define the criteria for success
├── Performance requirements
├── Development speed
├── Operational complexity
├── Team expertise
└── Long-term maintainability

Step 2: Evaluate each option against the criteria
├── Option A scores: [specific scores per criterion]
├── Option B scores: [specific scores per criterion]
└── Total: Option A = X, Option B = Y

Step 3: If scores are close, consider:
├── Can we prototype both?
├── Is the decision reversible?
├── What's the cost of being wrong?
└── Who will own the decision?

Step 4: Decide and document
├── Decision: [Option chosen]
├── Rationale: [Why, referencing the criteria]
├── Dissenting view: [What the other side believed]
└── Revisit criteria: [When we'll re-evaluate]
```

### Real Example: RAG Retrieval Architecture

```
Conflict: Team wants dense retrieval (embeddings only).
Product wants hybrid retrieval (BM25 + embeddings).

Criteria:
| Criterion | Dense Only | Hybrid |
|-----------|-----------|--------|
| Recall (our benchmark) | 72% | 89% |
| Query latency (p99) | 45ms | 78ms |
| Implementation complexity | 2 weeks | 4 weeks |
| Operational overhead | Low (one system) | Medium (two systems) |
| User satisfaction (projected) | Good | Better |

Analysis:
- Hybrid has 17% better recall (significant for user experience)
- Both meet our latency requirement (p99 < 100ms)
- Hybrid costs 2 extra weeks but improves quality significantly
- We already run Elasticsearch for other services (lower marginal overhead)

Decision: Hybrid retrieval.
Rationale: Recall improvement justifies the additional complexity
and 2-week delay. We already operate Elasticsearch, so operational
overhead is manageable.
Dissenting view: Dense-only would be simpler and faster to ship.
Revisit: After 3 months of production use, if recall is acceptable
with dense-only, we can simplify.
```

## Resolving Interpersonal Conflicts

### The SBI Model (Situation-Behavior-Impact)

When addressing interpersonal conflict:

```
SITUATION: Describe the specific situation (when, where)
BEHAVIOR: Describe the observable behavior (not your interpretation)
IMPACT: Describe the impact on you or the work

Example:
"Yesterday in the design review meeting (situation),
when you interrupted me three times while I was presenting
(behavior), I felt like my contribution wasn't valued, and
I stopped sharing my ideas (impact)."
```

### Common Interpersonal Conflict Scenarios

#### Scenario 1: The Dismissive Reviewer

**Problem:** A senior engineer's code review comments are perceived as dismissive and condescending.

**Resolution:**
1. **Direct conversation:** "I've noticed your review comments often start with 'Obviously...' or 'This should be...'. I know you're trying to be efficient, but it comes across as dismissive. Could you phrase suggestions as questions or with more context?"
2. **Senior engineer's response:** "I didn't realize that. I'm trying to be direct, not dismissive. I'll adjust my tone."
3. **Follow-up:** Check in after a week. "I noticed the improvement — thank you. The reviews feel much more collaborative now."

#### Scenario 2: The Credit Dispute

**Problem:** Two engineers both claim primary ownership of a feature.

**Resolution:**
1. **Acknowledge both contributions:** "Both of you made significant contributions. Alice designed the architecture and wrote the core. Bob built the integration layer and debugging tooling."
2. **Clarify credit attribution:** "In the release notes, I'll credit both of you with specific contributions. In the future, let's be clearer about ownership from the start."
3. **Process improvement:** Implement a CONTRIBUTING.md file that clarifies ownership conventions.

#### Scenario 3: The Trust Breakdown

**Problem:** One engineer consistently misses commitments, and teammates no longer trust their estimates.

**Resolution:**
1. **Private conversation:** "I've noticed your last three estimates were significantly off. Help me understand what's happening."
2. **Root cause:** "I keep getting pulled into incident response, and I'm not accounting for that in my estimates."
3. **Solution:** "Let's explicitly account for on-call load in sprint planning. If you're on-call, reduce your committed capacity by 40%."
4. **Rebuild trust:** Start with smaller, achievable commitments. Demonstrate reliability. Gradually increase scope.

## Conflict Resolution Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact |
|--------------|-------------------|--------|
| **Avoidance** | Pretending the conflict doesn't exist | Conflict festers and grows |
| **Public confrontation** | Arguing in meetings or channels | Creates sides, escalates emotions |
| **Triangulation** | Complaining about someone to a third party instead of to them | Creates gossip, erodes trust |
| **Scorekeeping** | "You always..." "You never..." | Makes the conflict about the person, not the issue |
| **Forced agreement** | "Can't we all just get along?" | Suppresses the conflict without resolving it |
| **Taking sides** | A manager picks a person, not an argument | Creates winners and losers |
| **Post-conflict punishment** | Treating someone differently after a conflict | Destroys psychological safety |

## When Conflict Is Healthy

Not all conflict is bad. Healthy conflict:

| Sign | What It Looks Like |
|------|-------------------|
| **Debate is about ideas, not people** | "I disagree with that approach" not "You're wrong" |
| **Everyone participates** | Multiple viewpoints are expressed |
| **Decisions are made** | Conflict leads to better decisions, not paralysis |
| **Relationships survive** | People can disagree strongly and still work together |
| **The best idea wins** | Junior engineers can convince senior engineers |
| **Post-decision commitment** | After a decision, everyone supports it |

**If your team has no conflict, that's actually a warning sign.** It may mean:
- People don't care enough to disagree
- People are afraid to speak up
- One person dominates the discussion
- The team is not diverse enough in thinking

## Building Conflict Resolution Skills

### For Individuals

1. **Practice the SBI model.** Use it in low-stakes situations first.
2. **Learn to separate behavior from intent.** Assume positive intent.
3. **Develop active listening.** Repeat back what you heard: "So what you're saying is..."
4. **Manage your emotions.** If you're angry, wait before responding.
5. **Seek feedback.** "How did that conversation feel for you?"

### For Teams

1. **Establish norms for disagreement.** "We debate ideas, not people."
2. **Practice "disagree and commit."** Make it an explicit team value.
3. **Debrief after conflicts.** "How did we handle that disagreement? What could be better?"
4. **Rotate meeting facilitation.** Different facilitation styles prevent dominance.
5. **Celebrate good conflicts.** "That debate led to a much better design."

## Cross-References

- `leadership-and-collaboration/negotiation.md` — Negotiation (structured approach to disagreement)
- `engineering-culture/psychological-safety.md` — Psychological safety (prerequisite for healthy conflict)
- `engineering-culture/code-reviews.md` — Code review (common conflict source)
- `engineering-culture/high-standards-with-empathy.md` — High standards with empathy
- `leadership-and-collaboration/escalation-strategies.md` — Escalation (when conflict can't be resolved)
