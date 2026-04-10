# On-Call Handoff Template

> Use this template for on-call rotation handoffs. Ensures continuity between shifts.

## Template

```markdown
# On-Call Handoff: [Service/Team Name]

**Outgoing:** [Name]
**Incoming:** [Name]
**Date/Time:** [YYYY-MM-DD HH:MM]
**Rotation Period:** [Start Date] — [End Date]

## Open Incidents

| Incident ID | Status | Summary | Next Action | Owner |
|-------------|--------|---------|-------------|-------|
| INC-XXXX | Active/Monitoring/Resolved | [2-3 sentences] | [What needs to happen next] | [Who] |

If no open incidents: "No open incidents."

## Recent Changes (last 7 days)

| Date | Change | Impact |
|------|--------|--------|
| YYYY-MM-DD | [Deployment/config change/migration] | [Any observed impact] |

If no changes: "No recent changes."

## Known Issues

| Issue | Impact | Workaround | Tracking Ticket |
|-------|--------|-----------|----------------|
| [Description] | [Who/what is affected] | [What users can do] | [Jira/GitHub link] |

If no known issues: "No known issues beyond what's in the runbook."

## Upcoming Changes (next 7 days)

| Date | Planned Change | Risk | Rollback Plan |
|------|---------------|------|--------------|
| YYYY-MM-DD | [Deployment/release/maintenance] | Low/Med/High | [How to undo] |

If no upcoming changes: "No upcoming changes planned."

## Metric Trends

| Metric | Last Week | This Week | Trend | Notes |
|--------|-----------|-----------|-------|-------|
| Error rate | [Value] | [Value] | ↑/→/↓ | [Any notable changes] |
| p95 latency | [Value] | [Value] | ↑/→/↓ | [...] |
| Throughput | [Value] | [Value] | ↑/→/↓ | [...] |
| Pager alerts | [Count] | [Count] | ↑/→/↓ | [...] |

## Anything Unusual

[Anything the incoming on-call should be aware of that doesn't fit
the categories above. Patterns, hunches, external events, etc.]

## Tips and Gotchas

[Anything non-obvious that would help the incoming on-call:
- "The Grafana dashboard X has a bug that shows stale data"
- "If you see alert Y, it's usually a false positive from Z"
- "The runbook step for W is outdated — do X instead"]

## Escalation Contacts

| Role | Name | Contact |
|------|------|---------|
| Tech lead | [Name] | [Phone/Slack] |
| Engineering manager | [Name] | [Phone/Slack] |
| External dependency contact | [Name, team] | [Contact] |

## Handoff Confirmation

- [ ] Outgoing has shared all relevant context
- [ ] Incoming has access to all systems/tools
- [ ] Incoming understands open items
- [ ] Both parties confirm handoff is complete
```

## Example: Filled Handoff

```markdown
# On-Call Handoff: GenAI Platform

**Outgoing:** Sarah Chen
**Incoming:** James Wu
**Date/Time:** 2026-04-07 09:00 AM
**Rotation Period:** April 7 — April 14

## Open Incidents

| Incident ID | Status | Summary | Next Action | Owner |
|-------------|--------|---------|-------------|-------|
| None | — | No open incidents | — | — |

## Recent Changes (last 7 days)

| Date | Change | Impact |
|------|--------|--------|
| Apr 3 | Deployed prompt injection detector v1.0 | Blocking ~2% of queries (mostly false positives from legitimate system admin queries) |
| Apr 5 | Increased memory limit on retriever pods from 512Mi to 1Gi | Resolved OOMKill issues |

## Known Issues

| Issue | Impact | Workaround | Tracking Ticket |
|-------|--------|-----------|----------------|
| Prompt injection false positives on IT admin queries | ~2% of legitimate queries blocked | Users can retry with rephrased query | GENAI-1240 |
| Stale document index for POL-2847 was fixed but no staleness monitoring exists | Risk of future stale data incidents | Manual daily check planned | GENAI-1245 |

## Upcoming Changes (next 7 days)

| Date | Planned Change | Risk | Rollback Plan |
|------|---------------|------|--------------|
| Apr 10 | Deploy response caching (RFC-047) | Medium — new Redis dependency | Disable cache via feature flag |
| Apr 12 | OpenAI maintenance window (02:00-04:00 UTC) | Low — fallback to Anthropic | Automatic failover configured |

## Metric Trends

| Metric | Last Week | This Week | Trend | Notes |
|--------|-----------|-----------|-------|-------|
| Error rate | 0.08% | 0.12% | ↑ | Due to injection detector false positives |
| p95 latency | 1200ms | 1150ms | ↓ | Slight improvement after retriever memory increase |
| Throughput | 110 req/min | 125 req/min | ↑ | Growing adoption |
| Pager alerts | 3 | 5 | ↑ | 2 were injection detector false positive alerts |

## Anything Unusual

The OpenAI maintenance window on Apr 12 is the first time our automatic
failover will be tested in production. Watch the failover metrics closely
during that window. If failover doesn't trigger automatically within
5 minutes of the outage starting, manually enable it.

## Tips and Gotchas

- The Grafana "GenAI Overview" dashboard has a broken panel for cache
  hit rate (panel 7). It shows 0% even when cache is working. Use the
  "Cache Metrics" dashboard instead.
- If you get an alert about "high injection block rate," check if it's
  a real spike or just the false positive issue. Threshold is >5% of
  total queries.
- The runbook step for "LLM API timeout" says to restart pods, but that
  doesn't help if the provider is down. Just monitor and wait for
  provider recovery or failover.

## Handoff Confirmation

- [x] Outgoing has shared all relevant context
- [x] Incoming has access to all systems/tools
- [x] Incoming understands open items
- [x] Both parties confirm handoff is complete
```
