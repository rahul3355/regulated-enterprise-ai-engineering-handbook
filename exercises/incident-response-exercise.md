# Incident Response Exercise: Model API Down

> Simulate an incident response when the external LLM API becomes unavailable during peak banking hours.

## Problem Statement

It's 10:30 AM on a Tuesday. The GenAI chat assistant used by 12,000 bank employees is returning errors. The external LLM API (OpenAI) is experiencing an outage in the US-East region. You are the incident commander.

Manage the incident from detection through resolution.

## Incident Timeline (Provided)

```
10:30 — Monitoring alert: GenAI chat endpoint error rate spikes to 85%
10:31 — On-call engineer acknowledges, begins investigation
10:33 — On-call identifies: LLM API returning 503 errors
10:35 — On-call checks provider status page: "Investigating elevated error rates"
10:37 — On-call declares P1 incident, pages incident commander
10:40 — Incident commander joins, assesses situation
10:42 — Impact assessment: 12,000 users affected, 95% of queries failing
10:45 — Decision point: Wait for provider recovery or activate fallback?
10:50 — [YOUR DECISIONS AND ACTIONS FROM HERE]
```

## Your Role: Incident Commander

As incident commander, you must:

1. **Assess the situation** — What do you need to know?
2. **Make decisions** — Wait or failover? What are the trade-offs?
3. **Communicate** — Who needs to know what, and when?
4. **Coordinate** — Who does what?
5. **Resolve** — How do you know the incident is over?
6. **Document** — What needs to be recorded?

## Context

```
System capabilities:
- Primary LLM: OpenAI GPT-4 Turbo (US-East)
- Fallback LLM: Anthropic Claude 3 (available, not yet activated)
- Fallback limitations:
  - 20% lower accuracy on banking documents
  - No support for some advanced prompt templates
  - Estimated 2-hour activation time (config change, testing, rollout)
  - 3x higher cost per token

Current situation:
- 95% of queries failing
- Provider ETA: "Investigating" — no ETA for resolution
- Historical recovery time for this provider: 1-4 hours
- Business impact: Employees cannot access policy information
- No customer-facing impact (internal tool only)
- Alternative: Employees can search the policy portal manually
```

## Expected Actions

Document your decisions and actions in incident command format:

```markdown
# Incident Command Log

## [Timestamp] Decision
**Decision:** [What you decided]
**Rationale:** [Why]
**Assigned to:** [Who executes]
**Expected outcome:** [What should happen]

## [Timestamp] Communication
**Audience:** [Who]
**Message:** [What you told them]
**Channel:** [How]

## [Timestamp] Status Update
**Current state:** [What's happening]
**Impact:** [Who is affected]
**Next update:** [When]
```

## Decision Points

### Decision 1: Wait vs. Failover

**Option A — Wait for provider recovery:**
- Pros: No activation risk, no quality degradation, no cost increase
- Cons: Users affected for unknown duration (1-4 hours historically)

**Option B — Activate fallback:**
- Pros: Restores service within 2 hours
- Cons: Lower accuracy, 3x cost, activation risk

**Your decision and rationale?**

### Decision 2: Communication Strategy

Who do you communicate to, what do you say, and through which channels?

- Engineering team?
- Product owner?
- Head of internal tools?
- All users (12,000 employees)?
- Executive leadership?

### Decision 3: Resolution Criteria

When do you declare the incident resolved?

- When the provider recovers?
- When the fallback is activated and stable?
- Something else?

## Extensions

1. **Write the postmortem:** After resolving the incident, write a blameless postmortem using the template in `templates/postmortem-template.md`.

2. **Design auto-failover:** Build an automatic failover system that detects provider degradation and switches to the fallback without human intervention. What are the risks?

3. **Multi-provider architecture:** Redesign the system to use multiple providers simultaneously, routing queries based on availability and cost.

4. **Graceful degradation:** Design a response system that works even when ALL LLM providers are down (cached responses, static fallback content).

5. **Chaos engineering:** Design a chaos engineering experiment to test failover behavior in production safely.

## Interview Relevance

Incident response is tested in SRE and senior engineering interviews:

| Skill | Why It Matters |
|-------|---------------|
| Incident command | Can you lead under pressure? |
| Decision making | Can you make trade-offs with incomplete information? |
| Communication | Can you keep stakeholders informed? |
| Trade-off analysis | Can you evaluate options honestly? |
| Post-incident learning | Can you turn incidents into system improvements? |

**Follow-up questions:**
- "What metrics would you monitor during this incident?"
- "How do you decide when to escalate?"
- "What would you change to prevent this incident?"
- "How do you handle the stress of being incident commander?"
