# Stakeholder Update Template

> Use this template for regular status updates to project stakeholders.

## Template

```markdown
# Stakeholder Update: [Project/Initiative Name]

**Date:** [YYYY-MM-DD]
**Period:** Week of [Date] / Month of [Date]
**Author:** [Name, role]
**Audience:** [Who this is for]
**Overall Status:** 🟢 On Track / 🟡 At Risk / 🔴 Off Track

## Bottom Line

[One sentence: How are things going? A busy reader should understand
the full status from this sentence alone.]

## Progress Since Last Update

### Completed ✅
- [Item 1 — what was done and its impact]
- [Item 2]

### In Progress ⏳
- [Item 1 — % complete, expected completion]
- [Item 2]

### Planned Next 📋
- [Item 1 — when]
- [Item 2]

## Risks and Blockers

| Risk/Blocker | Impact | Likelihood | Mitigation | Help Needed |
|-------------|--------|-----------|-----------|-------------|
| [Description] | High/Med/Low | High/Med/Low | [What we're doing] | [From whom, by when] |

If no risks: "No significant risks or blockers."

## Key Metrics

| Metric | Current | Target | Trend | Notes |
|--------|---------|--------|-------|-------|
| [Metric 1] | [Value] | [Target] | ↑/→/↓ | [Context] |
| [Metric 2] | [Value] | [Target] | ↑/→/↓ | [...] |

## Decisions Needed

| Decision | Options | Recommendation | Due Date |
|----------|---------|---------------|----------|
| [What needs to be decided] | [Option A vs B] | [Our recommendation] | [Date] |

If no decisions needed: "No decisions needed at this time."

## Timeline

| Milestone | Original Date | Current Forecast | Status |
|-----------|--------------|-----------------|--------|
| [Milestone 1] | [Date] | [Date] | ✅ On track / ⚠️ At risk / ❌ Slipped |
| [Milestone 2] | [Date] | [Date] | [...] |

## Appendix

[Links to supporting documents, dashboards, demos, etc.]
```

## Example: Filled Stakeholder Update

```markdown
# Stakeholder Update: GenAI Compliance Assistant

**Date:** 2026-04-07
**Period:** Week of April 1-7, 2026
**Author:** Sarah Chen, GenAI Platform Lead
**Audience:** Product, Compliance, Engineering Management
**Overall Status:** 🟢 On Track

## Bottom Line

On track for the May 1 launch. Security review passed with zero open
findings. Compliance review starts this week. User acceptance testing
with the HR pilot group begins April 14.

## Progress Since Last Update

### Completed ✅
- Security review passed — all 3 findings from the initial review
  have been addressed and verified
- Performance optimization: p95 latency reduced from 1.8s to 1.2s
  through response caching implementation
- User documentation drafted and reviewed by the training team

### In Progress ⏳
- Compliance review (started April 5, estimated 2 weeks) — 10% complete
- HR pilot group onboarding (15 users registered, orientation session
  scheduled for April 14) — 60% complete
- Load testing in staging environment — 40% complete

### Planned Next 📋
- Begin UAT with HR pilot group (week of April 14)
- Complete load testing and capacity planning (week of April 14)
- Prepare launch communication plan (week of April 21)

## Risks and Blockers

| Risk/Blocker | Impact | Likelihood | Mitigation | Help Needed |
|-------------|--------|-----------|-----------|-------------|
| Compliance review may take longer than 2 weeks | High — delays launch | Medium | Provided all documentation upfront; engaged compliance lead early | Compliance team to confirm resource allocation by April 9 |
| Load testing environment not yet provisioned | Medium — delays capacity planning | Low | Platform team has ticket, prioritized for April 8 | None |

## Key Metrics

| Metric | Current | Target | Trend | Notes |
|--------|---------|--------|-------|-------|
| Security findings (open) | 0 | 0 | → | All findings resolved |
| p95 latency | 1200ms | < 1500ms | ↓ | Improved from 1800ms last week |
| Staging uptime | 99.95% | 99.9% | → | Stable |
| Pilot users registered | 15 | 50 | ↑ | Growing |

## Decisions Needed

| Decision | Options | Recommendation | Due Date |
|----------|---------|---------------|----------|
| Expand pilot from HR to Finance in May? | HR only vs. HR + Finance | Expand to Finance — high demand, compliance review covers both | April 18 |

## Timeline

| Milestone | Original Date | Current Forecast | Status |
|-----------|--------------|-----------------|--------|
| Security review complete | April 4 | April 4 | ✅ Complete |
| Compliance review complete | April 19 | April 23 | ⚠️ At risk (may slip 4 days) |
| UAT complete | April 26 | April 26 | ✅ On track |
| Production launch | May 1 | May 1 | ✅ On track |
```
