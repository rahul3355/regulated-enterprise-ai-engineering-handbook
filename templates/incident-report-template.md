# Incident Report Template

> Use this template for real-time incident communication during active incidents.

## Template

```markdown
# Incident Report: [Brief Title]

**Incident ID:** INC-[Number]
**Status:** 🔴 Investigating / 🟡 Identified / 🟠 Monitoring / 🟢 Resolved
**Severity:** P0 (Critical) / P1 (High) / P2 (Medium) / P3 (Low)
**Started:** [YYYY-MM-DD HH:MM UTC]
**Last Updated:** [YYYY-MM-DD HH:MM UTC]
**Incident Commander:** [Name]
**Communications Lead:** [Name]

## Summary

[2-3 sentences: What is happening, what is the impact, what are we doing?]

## Impact

- **Who is affected:** [Users, services, systems]
- **How many:** [Number of users, % of traffic, specific systems]
- **What they experience:** [Error messages, degraded functionality]
- **Business impact:** [If known: revenue, compliance, reputational]

## Current Status

[What we know right now. 3-5 bullet points maximum.]

## What We're Doing

[Current actions. 2-3 bullet points.]

## Next Update

[When the next status update will be posted — typically every 30-60 minutes for P1/P0.]

## Timeline (for resolved incidents)

| Time (UTC) | Event |
|------------|-------|
| HH:MM | [What happened / what we did] |
| HH:MM | [...] |

## Resolution (for resolved incidents)

[What fixed the issue? What was the root cause?]

## Actions for Stakeholders

[If any action is needed from readers: "Do not deploy to production",
"Avoid using the chat assistant until further notice", etc.]
```

## Example: Filled Incident Report

```markdown
# Incident Report: GenAI Chat Service — Elevated Error Rates

**Incident ID:** INC-2026-0408
**Status:** 🟡 Identified
**Severity:** P1 (High)
**Started:** 2026-04-08 14:32 UTC
**Last Updated:** 2026-04-08 15:15 UTC
**Incident Commander:** Priya Patel
**Communications Lead:** James Wu

## Summary

The GenAI chat assistant is returning 503 errors for approximately 30%
of user queries due to degraded performance from our LLM API provider
(OpenAI) in the US-East region. The provider has acknowledged the issue
and is investigating. Our fallback provider (Anthropic) is available
but not yet activated.

## Impact

- **Who is affected:** All GenAI chat users (internal employees)
- **How many:** ~12,000 weekly active users; ~30% of queries failing
- **What they experience:** "Service temporarily unavailable" error message
- **Business impact:** Employees cannot access policy information via AI;
  no customer-facing impact; alternative: manual policy portal search

## Current Status

- Root cause identified: OpenAI US-East region degradation
- Provider status page: "Investigating elevated error rates" — no ETA
- Fallback provider (Anthropic) is healthy and available
- 70% of queries still succeeding (load balanced across regions)

## What We're Doing

- Monitoring provider recovery (OpenAI engineering team engaged)
- Evaluating failover to Anthropic (estimated 2-hour activation)
- Preparing user communication if outage extends beyond 1 hour

## Next Update

15:45 UTC (30 minutes)

## Actions for Stakeholders

- Engineering: No action needed. Do not deploy to GenAI services during incident.
- Product: Communicate to department heads if outage extends beyond 2 hours.
- All users: If the assistant is unavailable, use the policy portal at
  policies.bank.internal for manual search.
```

## Incident Severity Guide

| Severity | Criteria | Response Time | Communication |
|----------|----------|--------------|---------------|
| **P0 (Critical)** | Customer-facing outage, data breach, regulatory impact | Immediate | Every 15-30 min |
| **P1 (High)** | Significant feature unavailable, many users affected | 15 minutes | Every 30-60 min |
| **P2 (Medium)** | Partial degradation, workaround available | 1 hour | Every 2 hours |
| **P3 (Low)** | Minor issue, few users affected, no data loss | 4 hours | At resolution |

## Communication Channels

| Audience | Channel |
|----------|---------|
| Engineering team | `#incidents` Slack channel |
| Affected users | Internal status page + Slack announcement |
| Product/Management | Email + Slack DM to leads |
| Executives | Email (executive summary format) |
| External (if applicable) | Status page + social media (approved messaging) |

## Status Definitions

| Status | Meaning |
|--------|---------|
| 🔴 **Investigating** | We know something is wrong; investigating root cause |
| 🟡 **Identified** | Root cause identified; working on fix |
| 🟠 **Monitoring** | Fix applied; monitoring for stability |
| 🟢 **Resolved** | Service restored; incident closed |
