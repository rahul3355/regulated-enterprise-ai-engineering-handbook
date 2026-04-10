# Release Plan Template

> Use this template to plan and coordinate a production release of a GenAI service.

## Template

```markdown
# Release Plan: [Service Name] v[Version]

| Field | Value |
|-------|-------|
| **Service** | [Name] |
| **Version** | [Semantic version] |
| **Release Date** | [YYYY-MM-DD] |
| **Release Manager** | [Name] |
| **Engineering Owner** | [Name] |
| **Status** | Planning / Ready / In Progress / Complete / Rolled Back |

## Overview

[1-paragraph description of what's in this release and why it matters.]

## Changes

| Type | Description | Jira/Ticket | Risk |
|------|-------------|-------------|------|
| feat | [...] | [Link] | Low/Med/High |
| fix | [...] | [Link] | Low/Med/High |
| infra | [...] | [Link] | Low/Med/High |

## Pre-Release Checklist

- [ ] All tests passing (unit, integration, load)
- [ ] Security scan passed (no new CRITICAL/HIGH findings)
- [ ] Compliance review completed (if applicable)
- [ ] Performance regression check passed
- [ ] Database migrations tested (if applicable)
- [ ] API contract validated (if applicable)
- [ ] Documentation updated (runbooks, API docs, README)
- [ ] Feature flags configured (if applicable)
- [ ] Monitoring alerts configured (if applicable)
- [ ] Rollback plan tested
- [ ] Stakeholders notified

## Release Steps

| Step | Action | Owner | Time | Status |
|------|--------|-------|------|--------|
| 1 | Deploy to staging | [Name] | HH:MM | ⬜ |
| 2 | Run smoke tests on staging | [Name] | HH:MM | ⬜ |
| 3 | Run load tests on staging | [Name] | HH:MM | ⬜ |
| 4 | Stakeholder sign-off | [Name] | HH:MM | ⬜ |
| 5 | Deploy to production (canary: 10%) | [Name] | HH:MM | ⬜ |
| 6 | Monitor canary for 15 min | [Name] | HH:MM | ⬜ |
| 7 | Expand to 50% | [Name] | HH:MM | ⬜ |
| 8 | Monitor for 15 min | [Name] | HH:MM | ⬜ |
| 9 | Expand to 100% | [Name] | HH:MM | ⬜ |
| 10 | Final validation | [Name] | HH:MM | ⬜ |

## Rollback Plan

**Trigger for rollback:** [What conditions would cause us to roll back?]
- Error rate exceeds X%
- Latency p95 exceeds X ms
- Specific functionality broken
- Data integrity issue

**Rollback steps:**
1. [Step 1]
2. [Step 2]
3. [Step 3]

**Estimated rollback time:** [X minutes]

## Communication Plan

| Audience | Message | Channel | Timing |
|----------|---------|---------|--------|
| Engineering team | Release starting | Slack | Before step 1 |
| Stakeholders | Release complete | Email | After step 10 |
| Users (if applicable) | New feature announcement | Slack/Email | After validation |
| On-call team | Updated runbook | Wiki + Slack | Before release |

## Post-Release Validation

| Check | Method | Success Criteria |
|-------|--------|-----------------|
| Health endpoint | curl /health | 200 OK |
| Key functionality | [Specific test] | [Expected result] |
| Error rate | Grafana dashboard | < X% |
| Latency | Grafana dashboard | p95 < X ms |
| [Business metric] | [Dashboard/tool] | [Expected range] |

## Known Issues

| Issue | Impact | Workaround | Fix Planned |
|-------|--------|-----------|-------------|
| [...] | [...] | [...] | [Version/date] |
```
