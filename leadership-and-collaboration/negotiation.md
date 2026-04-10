# Negotiation

> "Technical negotiation is not about winning arguments. It's about finding the best path forward that the team can commit to."

## Why Engineers Need Negotiation Skills

Every day, engineers negotiate:

- With product managers about scope and timeline
- With security teams about findings and remediation timelines
- With other teams about API contracts and integration timelines
- With their own team about architecture decisions and code standards
- With managers about workload, priorities, and career growth

**If you can't negotiate effectively, you'll either:**
- Always give in (your technical standards erode)
- Always fight (you become impossible to work with)

The goal is neither. The goal is **finding the best technical outcome that everyone can commit to.**

## Types of Technical Negotiation

### 1. Scope vs. Timeline

**The classic trade-off:** "We need all 5 features by the end of the quarter."

**Bad negotiation:**
> Engineering: "That's impossible. We can do 2."
> Product: "We need all 5. The business depends on it."
> [Stalemate]

**Good negotiation:**
> Engineering: "Let me break down the effort for each feature.
> Feature 1: 2 weeks, Feature 2: 3 weeks, Feature 3: 4 weeks,
> Feature 4: 2 weeks, Feature 5: 3 weeks. Total: 14 weeks of
> work. We have 10 weeks. So either we need more engineers,
> a longer timeline, or a smaller scope.
>
> If the timeline is fixed, I recommend we do features 1, 2,
> and 4 in Q2 (7 weeks), and features 3 and 5 in early Q3.
> Features 1 and 4 are the highest-value and lowest-effort."

**The principle:** Use data, not opinions. Break down the problem. Offer options.

### 2. Technical Standards vs. Delivery Speed

**The scenario:** Security found 5 findings. Fixing all of them delays launch by 2 weeks.

**Bad negotiation:**
> Security: "All 5 findings must be fixed before launch."
> Engineering: "We can't delay launch. Can we fix them later?"
> Security: "No. Policy requires all findings resolved."
> [Escalation]

**Good negotiation:**
> Engineering: "Let's look at each finding:
>
> 1. Critical — SQL injection risk → We'll fix this before launch (2 days)
> 2. High — Missing rate limiting → We'll fix this before launch (1 day)
> 3. Medium — Verbose error messages → We'll fix this before launch (0.5 days)
> 4. Low — Missing security headers → Can we fix this within 1 week post-launch?
>    The risk is low and we have WAF protection in place.
> 5. Low — Outdated dependency → Can we fix this within 1 week post-launch?
>    No known exploit for our usage pattern.
>
> This way, all critical and high risks are addressed before launch,
> and the low risks are committed to a firm timeline within 1 week."

**The principle:** Categorize by risk, not by finding count. Offer firm commitments for deferred items.

### 3. Architecture Choices Between Teams

**The scenario:** Team A wants REST, Team B wants gRPC for inter-service communication.

**Bad negotiation:**
> Team A: "REST is simpler and we already use it."
> Team B: "gRPC is faster and has better type safety."
> Team A: "We're not learning a new technology for this."
> Team B: "Your choice will bottleneck our performance."
> [Stalemate, escalation to architects]

**Good negotiation:**
> "Let's define our requirements:
> - Latency: p99 < 100ms
> - Throughput: 1000 requests/second
> - Development speed: Both teams need to iterate quickly
> - Operational complexity: We share on-call
>
> REST can meet latency and throughput with our current load.
> gRPC would give us 3x better performance but adds operational
> complexity (new tooling, new monitoring, new debugging skills).
>
> Recommendation: Start with REST. If we hit performance limits
> (measured by our SLOs), migrate to gRPC. The migration path
> is clear because we'll design clean interfaces.
>
> This gives us delivery speed now with a performance escape hatch."

**The principle:** Define objective criteria. Evaluate both options against them. Propose a decision with an escape hatch.

### 4. Resource Allocation

**The scenario:** Your manager wants you to take on an additional project.

**Bad negotiation:**
> Manager: "Can you also lead the GenAI governance project?"
> You: "Sure." (Then burn out from overcommitment)

**Good negotiation:**
> "I'd be interested in the governance project. Let me show you
> my current commitments:
>
> - GenAI platform launch: 60% (through May)
> - On-call rotation: 15%
> - Mentoring two junior engineers: 10%
> - Tech talks and documentation: 5%
>
> That's 90%. If I take on the governance project, something
> needs to give. Options:
> 1. Reduce my platform commitment to 40% (delay launch by 2 weeks)
> 2. Pause mentoring until the governance project stabilizes
> 3. Bring in another engineer for the platform launch
>
> Which trade-off works best for the team?"

**The principle:** Show the math. Offer trade-offs. Let the decision-maker choose.

## Negotiation Framework

### PREPARE: Before the Negotiation

| Step | Action | Example |
|------|--------|---------|
| **Understand their position** | What do they want? Why? What are their constraints? | "Security wants all findings fixed because their KPI is zero open findings" |
| **Clarify your position** | What do you want? Why? What are your constraints? | "We want to launch on time because the CHRO has a board commitment" |
| **Identify shared interests** | What do you both want? | "Both want a secure, successful launch" |
| **Define your walk-away** | What's the minimum acceptable outcome? | "Critical and high findings MUST be fixed before launch" |
| **Prepare options** | Multiple proposals, not just one | "Option A: Fix all, delay 2 weeks. Option B: Fix critical, commit to timeline for rest" |

### EXECUTE: During the Negotiation

| Principle | How | Example |
|-----------|-----|---------|
| **Lead with shared interests** | "We both want a secure launch" | Establishes collaboration, not conflict |
| **Use data, not opinions** | "Our benchmarks show..." | Harder to argue with numbers |
| **Ask questions** | "Help me understand your concern" | Reveals the real issue |
| **Acknowledge their constraints** | "I understand you have 3 other reviews queued" | Shows empathy, builds goodwill |
| **Propose, don't demand** | "What if we..." not "We need..." | Invites collaboration |
| **Be willing to give ground** | "We can accept that" when it's not critical | Shows flexibility |
| **Get commitments in writing** | "So we agreed: X by date Y" | Prevents "that's not what I agreed to" |

### FOLLOW UP: After the Negotiation

| Action | Why |
|--------|-----|
| **Document the agreement** | Write it down, share with all parties |
| **Fulfill your commitments** | Trust is built on delivered promises |
| **Monitor the agreement** | Is the other party fulfilling theirs? |
| **Renegotiate if needed** | If circumstances change, reopen the discussion early |

## Negotiation in Banking Context

### The Regulatory Non-Negotiable

Some things cannot be negotiated in banking:

| Non-Negotiable | Why |
|----------------|-----|
| Regulatory compliance requirements | Legal obligation, non-discretionary |
| Security critical findings | Regulatory and reputational risk |
| Data protection standards | GDPR, PCI-DSS, banking regulations |
| Audit trail requirements | SOX, regulatory examination |

**How to handle:** Don't try to negotiate these. Instead, negotiate **how** to achieve compliance most efficiently.

### The Cost of "No" in Banking

In consumer tech, saying "no" to a requirement might mean a lost feature. In banking:

- Saying "no" to a compliance requirement → Regulatory fine
- Saying "no" to a security finding → Data breach
- Saying "no" to an audit requirement → Failed audit, remediation order

**The negotiation is never about WHETHER to comply. It's always about HOW and WHEN.**

## Negotiation Anti-Patterns

| Anti-Pattern | What It Looks Like | Why It Fails |
|--------------|-------------------|--------------|
| **Zero-sum thinking** | "If they win, I lose" | Most negotiations can create value for both sides |
| **Positional bargaining** | "I won't budge from my position" | Creates stalemates, forces escalation |
| **Emotional negotiation** | Getting defensive, angry, or personal | Destroys relationships, derails the discussion |
| **Negotiating in public** | Arguing in a large meeting or channel | Creates performative positions, harder to compromise |
| **Negotiating without data** | "I feel like this will take 3 weeks" | Opinions are negotiable, data is harder to dispute |
| **Agreeing without commitment** | "Sure, we'll try" (no timeline, no specifics) | Creates false expectations |
| **Escalating too early** | "I'll talk to your manager" on the first disagreement | Destroys the relationship, makes future negotiation harder |

## Getting to "Disagree and Commit"

Sometimes negotiation doesn't produce consensus. That's okay. The "disagree and commit" principle:

```
Before the decision:
├── Express your honest view with supporting evidence
├── Listen to other views with an open mind
├── Debate the merits, not the people
└── Accept the decision-maker's call gracefully

After the decision:
├── Support the decision fully in public
├── Don't undermine it with passive-aggressive compliance
├── Give it your best effort to succeed
└── If it fails, focus on learning, not "I told you so"
```

**Example:**
> "I still think pgvector will hit performance limits at our scale,
> but the team has decided on Pinecone. I'll fully support that
> decision and make it work. If we do hit the cost issues I'm
> worried about, we can revisit then."

## Cross-References

- `leadership-and-collaboration/influencing-without-authority.md` — Influence strategies
- `leadership-and-collaboration/conflict-resolution.md` — Conflict resolution (when negotiation breaks down)
- `leadership-and-collaboration/escalation-strategies.md` — Escalation (when negotiation fails)
- `leadership-and-collaboration/building-alignment.md` — Getting teams aligned
- `engineering-culture/rfcs.md` — RFC process (structured technical negotiation)
