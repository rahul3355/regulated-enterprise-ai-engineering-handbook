# Influencing Without Authority

> "You don't need a title to lead. You need credibility, relationships, and the ability to make others want to follow your lead."

## The Reality of Engineering Leadership

In a matrixed banking organization, you will regularly need to:

- Get another team to prioritize your integration requirement
- Convince the security team to approve your approach
- Align three teams on a common API standard
- Persuade your manager to invest in technical debt reduction
- Get compliance to fast-track your review

None of these people report to you. You cannot command them. You can only **influence** them.

## The Influence Equation

```
Influence = Credibility × Relationships × Communication

Where:
- Credibility: Do people believe you know what you're talking about?
- Relationships: Do people trust your intentions?
- Communication: Can you articulate your case effectively?

If any factor is zero, influence is zero.
```

### Building Credibility

| Method | How | Timeframe |
|--------|-----|-----------|
| **Technical depth** | Know your domain deeply. Be the person others come to for answers. | 6-12 months |
| **Track record** | Consistently deliver quality work on time. | Ongoing |
| **Public contributions** | Tech talks, design docs, open-source contributions | Ongoing |
| **Admitting limits** | "I don't know, but I'll find out" increases credibility | Immediate |
| **Cross-domain knowledge** | Understanding banking regulations AND GenAI AND infrastructure | 1-2 years |

### Building Relationships

| Method | How | Timeframe |
|--------|-----|-----------|
| **Help others succeed** | Volunteer to help other teams with their challenges | Ongoing |
| **Learn about people** | Know your colleagues' goals, challenges, and motivations | 1-3 months |
| **Give before you ask** | Offer help before you need something | Ongoing |
| **Be reliable** | When you commit to something, deliver | Immediate and ongoing |
| **Social connection** | Coffee chats, team events, informal conversations | Ongoing |

### Improving Communication

| Method | How | Timeframe |
|--------|-----|-----------|
| **Write clearly** | Design docs, RFCs, emails that are concise and persuasive | Ongoing |
| **Speak effectively** | Presentations, meetings, 1:1 conversations | Ongoing |
| **Tailor the message** | Executives want outcomes. Engineers want details. Compliance wants evidence. | Immediate |
| **Use data** | Numbers are more persuasive than opinions | Immediate |
| **Tell stories** | "Here's what happened at 3 AM" is more compelling than "latency increased" | Immediate |

## Influence Strategies

### 1. The Data-Driven Approach

**When to use:** Technical decisions, architecture choices, tool selection.

**How:**
1. Gather objective data
2. Present alternatives with honest trade-offs
3. Let the data make the case

**Example:**
> "I evaluated three embedding databases for our RAG pipeline. Here's the comparison:
>
> | Database | Query Latency (p99) | Memory Usage | Operational Complexity | Cost (annual) |
> |----------|---------------------|--------------|----------------------|---------------|
> | pgvector | 45ms | 8GB | Low (already run Postgres) | $0 |
> | Pinecone | 30ms | Managed | Low | $48K |
> | Milvus | 25ms | 16GB | High (new cluster) | $72K |
>
> Given our latency requirement (p99 < 100ms) and our operational capacity, pgvector meets our needs at zero incremental cost. I recommend we start with pgvector and reassess if we hit scale limits."

### 2. The Alignment Approach

**When to use:** Cross-team standards, shared infrastructure, platform decisions.

**How:**
1. Identify the shared goal
2. Show how your proposal advances it
3. Involve stakeholders in shaping the proposal

**Example:**
> "We all want the GenAI platform to be secure and compliant. The issue we're seeing — inconsistent prompt logging across teams — creates a compliance gap. I'm proposing a standard logging interface that every team can use. I've already talked to the security team and they support this direction. Can we work together on the design?"

### 3. The Coalition Approach

**When to use:** Organizational change, process improvements, cultural shifts.

**How:**
1. Find 2-3 people who share your view
2. Build a unified proposal together
3. Present as a group, not as an individual

**Example:**
> "Sarah from Platform, James from Security, and I have been discussing our deployment process. We've noticed that the current manual approval step adds 3 days to every release with no measurable quality improvement. We've drafted a proposal for automated approval gates based on test results. We'd like to pilot it with our three teams and measure the impact."

### 4. The Pilot Approach

**When to use:** When people are skeptical about a change.

**How:**
1. Propose a small, time-boxed experiment
2. Define success criteria upfront
3. Let the results drive the decision

**Example:**
> "I know moving to async-first communication is a big change. Instead of committing to it team-wide, can we try it for two weeks? We'll continue all our current processes, but additionally try written status updates instead of the Monday status meeting. After two weeks, we compare: did information flow improve? Did people feel more or less informed? If it didn't work, we go back. If it did, we expand."

### 5. The Reciprocity Approach

**When to use:** Getting help from other teams.

**How:**
1. Help them with their priorities first
2. Build goodwill
3. When you need something, they're motivated to help

**Example:**
> Last quarter, our team helped the Platform team debug their Kubernetes networking issue. Now, when we need their help setting up our new service mesh, they prioritize our request. This isn't transactional — it's relational.

## Understanding Others' Motivations

To influence someone, you need to understand what motivates them:

| Stakeholder | What They Care About | How to Frame Your Ask |
|-------------|---------------------|----------------------|
| **Security team** | Risk reduction, vulnerability prevention | "This reduces our attack surface by..." |
| **Compliance team** | Regulatory alignment, audit readiness | "This ensures we can demonstrate compliance with..." |
| **Product team** | User value, time to market | "This unblocks the feature by..." |
| **Platform team** | Stability, standardization, operational efficiency | "This follows the platform standard and reduces custom infrastructure" |
| **Engineering manager** | Team health, delivery predictability, stakeholder satisfaction | "This improves our delivery timeline and reduces risk" |
| **Executives** | Strategic alignment, cost, risk, competitive advantage | "This supports our strategic priority of X and reduces risk of Y" |

## The "Currency" of Influence

Think of influence as a bank account. You make deposits and withdrawals:

**Deposits:**
- Helping someone with their problem
- Publicly crediting someone's contribution
- Taking on an unglamorous task nobody else wants
- Sharing useful information or contacts
- Defending someone who isn't in the room

**Withdrawals:**
- Asking someone to prioritize your request
- Asking someone to change their approach
- Asking for a favor
- Disagreeing with someone publicly
- Escalating an issue

**Rule of thumb:** Maintain a positive balance. If you're always withdrawing and never depositing, people will stop responding to your requests.

## Influence Anti-Patterns

| Anti-Pattern | What It Looks Like | Why It Fails |
|--------------|-------------------|--------------|
| **Escalation as default** | "I'll talk to your manager" | Destroys relationships, creates resentment |
| **Authority borrowing** | "The CTO thinks we should..." (when they don't) | Quickly discovered, credibility destroyed |
| **Data manipulation** | Cherry-picking data to support your position | When discovered, trust is permanently damaged |
| **Emotional manipulation** | Guilt-tripping, playing the victim | Short-term gain, long-term trust loss |
| **Going around people** | Making decisions with their team without them | Creates surprise and defensiveness |
| **All stick, no carrot** | "You have to do this because..." with no benefit to them | No motivation to comply |

## Building Influence as a New Team Member

If you're new, you have low credibility and few relationships. Here's the path:

**Weeks 1-4:** Listen and learn. Ask questions. Understand the landscape. Help where asked.

**Weeks 5-8:** Start contributing. Ship small things well. Build relationships through pairing and code reviews.

**Weeks 9-12:** Propose small improvements. Write your first design doc. Volunteer for cross-team work.

**Months 4-6:** You now have enough credibility to influence team-level decisions.

**Months 6-12:** You can influence cross-team decisions if you've invested in relationships.

## Case Study: Influencing a Security Policy Change

> **Situation:** The GenAI platform team needed to change the security review process for model updates. The current process required a full security review for every model version change, adding 2 weeks to every update cycle.
>
> **Challenge:** The security team had no incentive to change — the current process was safe and thorough.
>
> **Approach:**
> 1. **Understand their motivation:** Security team's job is risk prevention. They don't care about our deployment speed.
> 2. **Build data:** Analyzed 6 months of model update security reviews. Found that 95% were identical to the previous version (same prompt, same configuration, only model weights changed).
> 3. **Propose a pilot:** "Can we create a fast-track for model updates that don't change the prompt, configuration, or data access patterns? Full review for everything else. We'll track security incidents for 3 months."
> 4. **Get allies:** The compliance team supported this because faster model updates mean we can respond to model safety issues more quickly.
> 5. **Present jointly:** Met with the security team lead alongside the compliance representative.
>
> **Result:** Pilot approved. After 3 months, zero security incidents with fast-tracked updates. Review time dropped from 2 weeks to 2 days for qualifying updates. Process adopted organization-wide.
>
> **Key learning:** Influence started with understanding the security team's motivations, not with asserting our own needs.

## Cross-References

- `leadership-and-collaboration/managing-stakeholders.md` — Stakeholder management
- `leadership-and-collaboration/negotiation.md` — Technical negotiation
- `leadership-and-collaboration/building-alignment.md` — Getting teams aligned
- `engineering-culture/building-trust.md` — Trust building (foundation for influence)
- `engineering-culture/rfcs.md` — RFC process (influence through written proposals)
