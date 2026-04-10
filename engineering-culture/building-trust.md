# Building Trust

> "Trust is the currency of engineering. Without it, every decision is questioned, every estimate is doubted, every incident is investigated as negligence."

## What Trust Means in Engineering

Trust is not about being liked. Trust is about being **reliable, competent, and honest.**

In banking GenAI engineering, trust manifests as:

| Trust Dimension | What It Looks Like | What Breaks It |
|----------------|-------------------|----------------|
| **Technical trust** | Your code works, your designs are sound | Bugs, outages, poor designs |
| **Reliability trust** | You do what you say you'll do, when you say | Missed commitments, surprises |
| **Judgment trust** | Your recommendations are well-reasoned | Repeatedly bad calls |
| **Communication trust** | You tell the truth, even when it's uncomfortable | Sugar-coating, hiding problems |
| **Team trust** | You support your teammates, share credit | Blame, credit-stealing |

## The Trust Equation

```
Trust = (Credibility + Reliability + Intimacy) / Self-Orientation

Where:
- Credibility: Do you know what you're talking about?
- Reliability: Do you do what you say you'll do?
- Intimacy: Do people feel safe sharing concerns with you?
- Self-Orientation: Are you motivated by team success or personal gain?
  (Higher self-orientation = lower trust)
```

### Building Each Component

**Credibility:**
- Demonstrate technical competence consistently
- Admit when you don't know something (counterintuitively, this increases credibility)
- Stay current with technology and domain knowledge
- Share your knowledge generously

**Reliability:**
- Make commitments you can keep
- Communicate early when commitments are at risk
- Follow through on every action item
- Be predictable in your quality of work

**Intimacy:**
- Be genuinely interested in others' success
- Share your own struggles and learnings
- Create safe spaces for others to be vulnerable
- Remember personal details (ask about their kids, their hobbies)

**Low Self-Orientation:**
- Give credit to others
- Take responsibility for failures
- Advocate for team needs over personal preferences
- Make decisions based on what's right, not what's visible

## Trust in the Banking Context

### Why Trust Matters More in Banking

In a consumer tech company, a trust breach might mean a frustrated user. In banking:

- **Regulatory trust:** If regulators don't trust our systems, we face fines, restrictions, or license revocation
- **Customer trust:** If customers don't trust us with their data, they leave (and tell others)
- **Internal trust:** If other teams don't trust our GenAI platform, they build their own (shadow IT)
- **Executive trust:** If executives don't trust engineering, they impose constraints, hire consultants, or reorganize

### Building Trust with Non-Technical Stakeholders

Non-technical stakeholders (compliance, legal, product, executives) evaluate trust differently than engineers:

| Stakeholder | What Builds Trust | What Breaks It |
|-------------|------------------|----------------|
| **Compliance** | Proactive engagement, documented processes, honest risk assessments | Surprises, undocumented changes, dismissive attitude |
| **Security** | Self-assessment before review, prompt remediation, transparency | Finding vulnerabilities yourself, delayed fixes, hiding issues |
| **Product** | Realistic estimates, early communication of risks, delivery on commitments | Over-promising, late surprises, feature creep without communication |
| **Executives** | Clear communication, honest bad news early, solutions with problems | Sugar-coating, surprises, complaints without solutions |
| **Legal** | Clear technical explanations, documented processes, early engagement | Jargon without explanation, assuming legal understands technical trade-offs |

## Practical Trust-Building Behaviors

### Daily

| Behavior | Impact |
|----------|--------|
| Respond to messages within the agreed timeframe | Shows reliability |
| Update tickets/status without being asked | Shows ownership |
| Help a teammate who's stuck | Shows team orientation |
| Admit a mistake immediately | Shows integrity |

### Weekly

| Behavior | Impact |
|----------|--------|
| Share what you've learned | Shows generosity with knowledge |
| Acknowledge someone else's contribution | Shows lack of self-orientation |
| Follow up on an action item | Shows reliability |
| Share a risk or concern early | Shows communication trust |

### Monthly

| Behavior | Impact |
|----------|--------|
| Review and update your team on long-running work | Shows ownership |
| Mentor someone on a skill you have | Shows credibility + generosity |
| Propose an improvement (not just execute assigned work) | Shows investment in the team |
| Reflect on a mistake and share the learning | Shows growth mindset |

## Trust Repair

When trust is broken (and it will be), here's how to repair it:

### The Trust Repair Process

1. **Acknowledge.** "I missed the deadline." Not "The deadline was missed." Own it.
2. **Apologize.** "I'm sorry for the impact this had on the team." Not "I'm sorry if anyone was inconvenienced."
3. **Explain (briefly).** "I underestimated the complexity of the integration." Not a 10-minute justification.
4. **Commit to change.** "Going forward, I'll flag estimation uncertainty earlier."
5. **Demonstrate change.** The next time, flag estimation uncertainty earlier.

### Example: Trust Repair After an Incident

**BAD:**
> "The outage was caused by an unusual interaction between the new model and the caching layer. Nobody could have predicted this. We've made some changes to monitoring."

**GOOD:**
> "I deployed the model upgrade without testing it against the caching layer in staging. This was my mistake — our deployment checklist includes this step and I skipped it. The outage lasted 47 minutes and affected 12,000 users. I'm sorry for the impact on our users and the team. I've updated the deployment pipeline to enforce this check automatically so it can't be skipped. I'll share the full postmortem by Friday."

## Trust Metrics (Informal, Not Judgmental)

We don't score individuals on trust, but we track team-level indicators:

| Indicator | What It Signals |
|-----------|----------------|
| Team members volunteer for hard problems | High trust in team support |
| Incidents are reported quickly, not hidden | High communication trust |
| Code reviews are candid but respectful | High technical trust |
| Estimates are honest, not optimistic | High reliability trust |
| People admit mistakes without fear | High psychological safety |
| Cross-team collaborations happen smoothly | High cross-functional trust |

## Trust Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact |
|--------------|-------------------|--------|
| **Over-promising** | "Sure, we can do that in a week!" | When you miss, trust is damaged more than if you'd been honest |
| **Hidden bad news** | "Everything's fine" (it's not) | When the truth comes out, trust is severely damaged |
| **Blame shifting** | "That wasn't my part" | Destroys team trust |
| **Credit hoarding** | "I built the system" (the team did) | Destroys team trust |
| **Inconsistent quality** | Great work sometimes, sloppy other times | Unpredictable, trust erodes |
| **Knowledge hoarding** | "I'm the only one who knows how this works" | Creates dependency, not trust |

## Trust and Remote Work

Building trust remotely is harder but not impossible:

| Challenge | Solution |
|-----------|----------|
| No casual interactions | Schedule virtual coffee chats |
| Hard to read non-verbal cues | Camera on for important conversations |
| "Out of sight, out of mind" | Regular status updates, visible contributions |
| Miscommunication in text | Default to video for sensitive conversations |
| Time zone gaps | Rotate meeting times, async documentation |

## Trust as a Multiplier

Trust is not just a "nice to have" — it's a performance multiplier:

```
High-trust team:
├── Decisions are made quickly (less second-guessing)
├── Information flows freely (no knowledge hoarding)
├── Mistakes are caught early (no fear of blame)
├── Collaboration is natural (no territorial behavior)
└── Velocity is high (less process overhead needed)

Low-trust team:
├── Decisions require extensive justification
├── Information is guarded (knowledge is power)
├── Mistakes are hidden (fear of consequences)
├── Collaboration requires management intervention
└── Velocity is low (process overhead compensates for trust gap)
```

## Building Trust as a New Team Member

If you're new to the team:

1. **Start small.** Deliver on small commitments consistently before taking on big ones.
2. **Ask questions.** Asking good questions shows engagement and humility.
3. **Listen more than you speak.** Understand the context before proposing changes.
4. **Be helpful.** Volunteer for things others don't want to do.
5. **Be honest about what you don't know.** "I haven't worked with Kubernetes before" is better than faking it.
6. **Ship something small quickly.** A docs PR or bug fix shows you can deliver.

## Cross-References

- `engineering-culture/psychological-safety.md` — Psychological safety (trust's foundation)
- `engineering-culture/working-agreements.md` — Working agreements (trust in practice)
- `leadership-and-collaboration/influencing-without-authority.md` — Influence (built on trust)
- `engineering-philosophy/craft.md` — Engineering craft (trust through quality)
- `leadership-and-collaboration/communicating-with-executives.md` — Executive communication (trust with leadership)
