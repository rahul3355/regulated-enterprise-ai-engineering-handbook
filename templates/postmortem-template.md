# Blameless Postmortem Template

> Use this template for post-incident learning. The goal is understanding, not blame.

## Template

```markdown
# Postmortem: [Incident Title]

| Field | Value |
|-------|-------|
| **Incident ID** | INC-[Number] |
| **Date of Incident** | [YYYY-MM-DD] |
| **Postmortem Date** | [YYYY-MM-DD] |
| **Author** | [Name] |
| **Attendees** | [Names of retrospective participants] |
| **Status** | Draft / In Review / Complete |

## Executive Summary

[2-3 paragraphs: What happened, what was the impact, what was the
root cause, and what are we doing about it? A busy reader should
understand the full story from this section.]

## Impact

| Metric | Value |
|--------|-------|
| **Duration** | [X minutes/hours] |
| **Users Affected** | [Number or description] |
| **Services Affected** | [List of services] |
| **Data Impact** | [Any data loss or corruption?] |
| **Financial Impact** | [If quantifiable] |
| **Compliance Impact** | [Any regulatory implications?] |

## Timeline

[Chronological list of events from detection through resolution.
Include what happened, who did what, and key decisions.]

| Time (UTC) | Event |
|------------|-------|
| HH:MM | [What happened] |
| HH:MM | [What was detected / who detected it] |
| HH:MM | [Investigation findings] |
| HH:MM | [Decision made] |
| HH:MM | [Action taken] |
| HH:MM | [Resolution confirmed] |

## Root Cause Analysis

[Describe the chain of events and conditions that led to the incident.
Use the "Five Whys" technique if helpful. Focus on SYSTEMS, not people.]

### What Went Wrong

[Technical description of the failure.]

### Why It Wasn't Caught

[Why didn't testing, monitoring, or review catch this?]

### Why It Wasn't Contained Faster

[What slowed detection, response, or resolution?]

## Contributing Factors

[What conditions made this incident possible? Think beyond the
immediate trigger.]

| Factor | Category | Description |
|--------|----------|-------------|
| [...] | Technical / Process / Human / Organizational | [...] |

## Action Items

[Specific, assigned, time-bound actions to prevent recurrence or
reduce impact of similar incidents.]

| # | Action | Owner | Priority | Due Date | Status |
|---|--------|-------|----------|----------|--------|
| 1 | [...] | @name | P0/P1/P2 | YYYY-MM-DD | Open |

## Lessons Learned

### What Went Well
- [...]

### What Went Poorly
- [...]

### Where We Got Lucky
- [...]

## Appendix

### Supporting Data
[Links to graphs, logs, dashboards, screenshots]

### Related Incidents
[Has this happened before? Link to previous postmortems.]

### References
[Links to relevant design docs, runbooks, ADRs]
```

## Example: Filled Postmortem

```markdown
# Postmortem: Stale Policy Data in GenAI Compliance Assistant

| Field | Value |
|-------|-------|
| **Incident ID** | INC-2026-0408 |
| **Date of Incident** | 2026-04-08 |
| **Postmortem Date** | 2026-04-12 |
| **Author** | Priya Patel |
| **Attendees** | Sarah Chen, James Wu, Alex Rodriguez, Maria Santos (Compliance) |
| **Status** | Complete |

## Executive Summary

On April 8, 2026, the GenAI compliance assistant returned incorrect
regulatory information to loan officers for 47 minutes. A capital
requirement policy had been updated from 8% to 12%, but the RAG
pipeline's document index was not updated because a webhook delivery
failed silently.

Two loan officers received the incorrect 8% figure. One processed a
$50M corporate loan with $2M less capital than required. The error
was caught by a compliance analyst within 45 minutes and manually
corrected. No regulatory filing was affected.

Root cause: The document indexing webhook endpoint had been moved
from /api/v1/index to /api/v2/index in a deployment 2 weeks prior,
but the document management system was never updated. The webhook
failure was logged but not alerted on.

## Impact

| Metric | Value |
|--------|-------|
| **Duration** | 47 minutes (first incorrect answer to resolution) |
| **Users Affected** | 2 loan officers received incorrect information |
| **Services Affected** | GenAI compliance assistant |
| **Data Impact** | No data loss; one loan record corrected |
| **Financial Impact** | $2M capital shortfall (corrected within 1 hour) |
| **Compliance Impact** | Moderate — internal process error, no external impact |

## Timeline

| Time (UTC) | Event |
|------------|-------|
| 09:15 | Compliance team updates POL-2847 (capital requirements: 8% → 12%) |
| 09:16 | Updated document uploaded to document management system |
| 09:17 | Webhook to RAG indexer fails with 500 (wrong endpoint) |
| 09:17 | Failure logged; no alert generated |
| 09:20 | Loan officer asks about capital requirement; receives stale 8% answer |
| 09:22 | $50M loan processed with 8% capital requirement |
| 09:30 | Second loan officer receives same incorrect answer |
| 09:45 | Compliance analyst notices discrepancy (8% vs. 12%) |
| 09:47 | Incident report filed |
| 09:50 | On-call acknowledges, confirms stale index |
| 09:55 | Manual re-index triggered |
| 10:00 | Re-index complete; new version searchable |
| 10:02 | Verified: assistant returns correct 12% answer |
| 10:30 | Loan capital requirement manually corrected |

## Root Cause Analysis

### What Went Wrong
The RAG pipeline's document indexer receives updates via webhook from
the document management system. The webhook endpoint was changed during
a deployment (v1 → v2 API migration) but the document management
system's webhook configuration was not updated. The webhook failed
with a 500 error, meaning the updated document was never re-indexed.

### Why It Wasn't Caught
1. The webhook failure was logged but not alerted on.
2. No process exists to verify that document changes are reflected
   in the RAG index.
3. The API migration checklist did not include updating external
   webhook consumers.

### Why It Wasn't Contained Faster
1. The assistant has no mechanism to detect that retrieved documents
   are outdated.
2. No "last updated" timestamp is shown in responses.
3. Detection relied on a human noticing the discrepancy manually.

## Contributing Factors

| Factor | Category | Description |
|--------|----------|-------------|
| Webhook endpoint change without consumer update | Process | Migration checklist missed this dependency |
| No alerting on webhook failures | Technical | Failed webhooks logged but not monitored |
| No index staleness detection | Technical | No way to detect outdated documents in the index |
| No response confidence indicator | Technical | Users don't know how current the information is |
| No human-in-the-loop for compliance answers | Process | High-stakes compliance answers go directly to users |

## Action Items

| # | Action | Owner | Priority | Due Date | Status |
|---|--------|-------|----------|----------|--------|
| 1 | Fix webhook endpoint in document management system | @james | P0 | 2026-04-09 | ✅ Done |
| 2 | Add alerting for webhook delivery failures | @james | P0 | 2026-04-12 | ✅ Done |
| 3 | Implement index staleness monitoring and alerting | @sarah | P1 | 2026-04-26 | In Progress |
| 4 | Add "last updated" timestamp to assistant responses | @sarah | P1 | 2026-04-26 | Open |
| 5 | Create API migration checklist that includes webhook consumers | @alex | P1 | 2026-05-03 | Open |
| 6 | Implement human-in-the-loop review for compliance-category responses | @priya + @maria | P1 | 2026-05-10 | Open |
| 7 | Add periodic full re-index as safety net (daily) | @james | P2 | 2026-05-17 | Open |

## Lessons Learned

### What Went Well
- The error was caught quickly by a vigilant compliance analyst
- The manual re-index process worked and was fast (5 minutes)
- The loan was corrected before any regulatory filing
- Incident response was calm and effective

### What Went Poorly
- Webhook failures were silently logged for 2 weeks
- No automated detection existed for stale index data
- The API migration process had a gap that wasn't caught by review

### Where We Got Lucky
- A compliance analyst happened to ask the same question on the same
  day as the policy update. If this had gone unnoticed for days, the
  impact could have been much larger.
- Only 2 loan officers asked the question during the window. Higher
  usage would have multiplied the impact.

## Appendix

### Supporting Data
- Webhook failure logs: [link]
- Index query showing stale document: [link]
- Incident Slack thread: [link]

### Related Incidents
- INC-2026-0315: Model API timeout causing stale cached responses
- INC-2025-1201: Document indexing delay after database migration
```

## Postmortem Facilitation Guide

### Before the Meeting
- Schedule within 5 business days of the incident
- Invite incident responders and affected stakeholders
- Prepare the timeline from incident logs
- Assign a facilitator (NOT the incident commander)

### During the Meeting (60 minutes)
1. **Ground rules (5 min):** Blameless, focus on systems
2. **Timeline review (15 min):** Build the timeline together
3. **Root cause discussion (15 min):** What conditions allowed this?
4. **Action items (15 min):** Specific, assigned, time-bound
5. **Lessons learned (10 min):** What did we learn?

### After the Meeting
- Publish the postmortem within 2 business days
- Track action items in Jira with label `postmortem-action`
- Review action item completion in weekly team sync
- Share learnings in engineering all-hands if broadly applicable
