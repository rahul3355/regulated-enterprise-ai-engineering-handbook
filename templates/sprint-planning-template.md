# Sprint Planning Template

> Use this template to plan and commit to sprint deliverables.

## Template

```markdown
# Sprint [N] Planning: [Team Name]

**Sprint Dates:** [Start Date] — [End Date]
**Planning Date:** [YYYY-MM-DD]
**Facilitator:** [Name]
**Attendees:** [Team members]

## Sprint Goal

[One sentence: What is the primary outcome this sprint?
If we accomplish only one thing, what should it be?]

## Capacity

| Team Member | Available Days | Notes |
|-------------|---------------|-------|
| [Name] | [X] days | [PTO, on-call, meetings, etc.] |
| [Name] | [X] days | [...] |
| **Total** | **[X] days** | **[Utilization: X%]** |

## Sprint Backlog

| Priority | Item | Story Points | Owner | Notes |
|----------|------|-------------|-------|-------|
| 1 | [Story/ticket] | [X] | [Name] | [Dependencies, risks] |
| 2 | [Story/ticket] | [X] | [Name] | [...] |
| 3 | [Story/ticket] | [X] | [Name] | [...] |
| | **Total** | **[X] points** | | |

## Carry-over from Previous Sprint

| Item | Reason for Carry-over | New Commitment |
|------|----------------------|----------------|
| [Ticket] | [Why it wasn't completed] | [When it will be done] |

## Dependencies and Blockers

| Dependency | On Whom | Due Date | Status |
|------------|---------|----------|--------|
| [What we need] | [Team/person] | [Date] | ⬜ Pending / ✅ Received |

## Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|-----------|
| [What could go wrong] | High/Med/Low | High/Med/Low | [What we'll do] |

## Definition of Done

For this sprint, "done" means:
- [ ] Code written and reviewed
- [ ] Tests passing (coverage > 80%)
- [ ] Documentation updated
- [ ] Deployed to staging
- [ ] [Any sprint-specific criteria]

## Commitment

The team commits to delivering the above backlog items within this sprint,
with the understanding that priorities may shift if urgent issues arise.

**Team Confidence:** 😟 Low / 😐 Medium / 😊 High
```

## Example: Filled Sprint Planning

```markdown
# Sprint 12 Planning: GenAI Platform Team

**Sprint Dates:** April 8 — April 21, 2026
**Planning Date:** April 8, 2026
**Facilitator:** Sarah Chen
**Attendees:** Sarah, James, Priya, Alex

## Sprint Goal

Complete the compliance review for the GenAI Compliance Assistant and
begin production readiness activities for the May 1 launch.

## Capacity

| Team Member | Available Days | Notes |
|-------------|---------------|-------|
| Sarah Chen | 8 days | 2 days PTO (April 15-16) |
| James Wu | 10 days | On-call April 14-21 (~1 day equivalent) |
| Priya Patel | 9 days | Tech talk prep (0.5 day) |
| Alex Rodriguez | 10 days | Full availability |
| **Total** | **37 days** | **~85% utilization** |

## Sprint Backlog

| Priority | Item | Story Points | Owner | Notes |
|----------|------|-------------|-------|-------|
| 1 | Address compliance review findings | 8 | Priya | Dependent on compliance team feedback |
| 2 | Implement audit log staleness monitoring | 5 | James | GENAI-1245 |
| 3 | Load testing and capacity report | 5 | Alex | Requires staging env (ticket with platform) |
| 4 | UAT support for HR pilot group | 3 | Sarah | April 14-18 |
| 5 | Response caching deployment to staging | 3 | James | RFC-047, feature-flagged |
| 6 | Launch communication plan draft | 2 | Sarah | With product team |
| | **Total** | **26 points** | | |

## Carry-over from Previous Sprint

| Item | Reason for Carry-over | New Commitment |
|------|----------------------|----------------|
| GENAI-1230: Multi-provider failover test | OpenAI maintenance delayed | Complete during April 12 maintenance window |

## Dependencies and Blockers

| Dependency | On Whom | Due Date | Status |
|------------|---------|----------|--------|
| Compliance review feedback | Compliance team | April 12 | ⬜ Pending |
| Staging environment for load testing | Platform team | April 9 | ⬜ Pending |
| HR pilot test scenarios | Product team | April 11 | ✅ Received |

## Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|-----------|
| Compliance review takes longer than 2 weeks | High — delays launch | Medium | Escalate to compliance lead if no feedback by April 12 |
| Load testing environment not ready | Medium — delays capacity planning | Low | Platform team confirmed April 8 delivery |
| OpenAI maintenance causes unexpected outage | Medium — affects demo | Low | Fallback to Anthropic is configured |

## Definition of Done

For this sprint, "done" means:
- [ ] Code written and reviewed (at least 1 approval)
- [ ] Tests passing (coverage > 80%)
- [ ] Runbook updated (for operational changes)
- [ ] Deployed to staging
- [ ] Compliance review findings have remediation plan

## Commitment

The team commits to delivering the above backlog items within this sprint.
The #1 priority is addressing compliance review findings — if that takes
more effort than estimated, lower-priority items may be deferred.

**Team Confidence:** 😊 High
```
