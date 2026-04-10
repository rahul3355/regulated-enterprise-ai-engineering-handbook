# Postmortem Process

## Overview

The postmortem is a structured review of an incident that identifies root causes, contributing factors, and actionable improvements. Our postmortem process is blameless, action-oriented, and focused on systemic learning.

This document defines the postmortem process, facilitation techniques, and document standards.

---

## Postmortem Principles

### 1. Blameless

The postmortem examines the system, not the people. We assume everyone acted with good intentions and with the information they had at the time. The question is not "who made the mistake?" but "why did the system allow this outcome?"

### 2. Factual

The postmortem is based on evidence, not speculation. The timeline is constructed from logs, metrics, and documented events. Hypotheses are clearly labeled as such.

### 3. Action-Oriented

Every postmortem produces concrete, actionable improvements. Each action item has an owner, a deadline, and a measurable outcome.

### 4. Timely

Postmortems are conducted while the incident is still fresh in everyone's memory:
- SEV-1: Within 5 business days
- SEV-2: Within 10 business days
- SEV-3: Within 15 business days (or combined in sprint retrospective)

### 5. Transparent

Postmortems are published internally (blameless summaries). Learnings are shared across the engineering organization.

---

## Postmortem Roles

| Role | Responsibility |
|------|---------------|
| **Facilitator** | Leads the postmortem session, ensures blameless culture, drives action items |
| **Author** | Writes the postmortem document (may be the Facilitator or the IC) |
| **Participants** | People involved in the incident who provide context and perspective |
| **Reviewers** | Engineering leads who review and approve the postmortem |
| **Action Owners** | People assigned to complete action items |

---

## Postmortem Process

### Step 1: Schedule the Postmortem

**When:** Within the SLA for the incident severity.

**Who:**
- Facilitator (assigned by IC or engineering lead)
- All participants who were involved in the incident response
- Optional: Observers who want to learn

**Duration:**
- SEV-1: 60-90 minutes
- SEV-2: 45-60 minutes
- SEV-3: 30 minutes (or combined in retrospective)

### Step 2: Prepare the Document

**Before the meeting**, the Author prepares:
1. Incident timeline (from [incident-timeline.md](incident-timeline.md))
2. Impact assessment (customers, data, finance, regulatory)
3. Initial root cause hypothesis
4. List of questions to explore during the session

### Step 3: Conduct the Postmortem Session

#### Opening (5 minutes)

> "Welcome to the postmortem for [incident name]. I'm [facilitator name], and I'll be facilitating this session.
>
> Ground rules:
> 1. This is a blameless postmortem. We are here to understand the system, not to assign blame.
> 2. We assume everyone acted with good intentions and with the information they had.
> 3. We focus on what happened, how it happened, and how we prevent it from happening again.
> 4. Everything discussed here stays in this room. The published summary will be blameless.
>
> Let's start by reviewing the timeline."

#### Timeline Review (10-15 minutes)

- Walk through the incident timeline
- Ask participants to fill in gaps
- Confirm timestamps and events
- Identify key decision points

#### Root Cause Analysis (20-30 minutes)

Use the **5 Whys** technique:

```
Problem: AssistBot returned PII from the wrong customer.

Why? The RAG system retrieved documents from multiple customers.

Why? The vector database query had no customer ID filter.

Why? The migration to a single namespace removed the namespace-level isolation.

Why? The migration was done for performance, and access control was assumed to be handled at the application layer.

Why? The security review did not cover the migration change, and there was no formal risk assessment for the architecture change.

Root cause: Missing security review for architectural change + insufficient defense-in-depth in RAG design.
```

Use the **Fishbone Diagram** for complex incidents:

```
                  People               Process
                    |                     |
          No GenAI security        No migration review
          expertise on team        process for security
                    |                impact assessment
                    |                     |
                    |                     |
  ──────────────────┼─────────────────────┼─────────────  Problem
                    |                     |
          No tenant isolation      No output PII
          in vector DB             filtering pipeline
                    |                     |
              Technical             Organizational
```

#### Contributing Factors (10 minutes)

Identify factors that contributed to the incident but were not the root cause:
- Missing monitoring
- Insufficient testing
- Process gaps
- Tooling limitations
- Communication issues

#### Action Items (15-20 minutes)

For each root cause and contributing factor, identify action items:
- **Immediate fixes**: Can be done within 1 sprint
- **Systemic fixes**: Require architectural or process changes
- **Monitoring improvements**: New alerts, dashboards
- **Documentation updates**: Runbooks, playbooks, guides

For each action item:
- **Owner**: Single person accountable
- **Deadline**: Specific date (not "soon")
- **Success criteria**: Measurable outcome
- **Priority**: P0 (within 1 sprint), P1 (within 2 sprints), P2 (within quarter)

#### Closing (5 minutes)

> "Thank you everyone. We have identified [N] action items. The Author will finalize the postmortem document and circulate it for review. Action items will be tracked in Jira.
>
> Does anyone have anything to add before we close?"

### Step 4: Finalize the Document

The Author finalizes the postmortem document within 2 business days of the session.

### Step 5: Review and Approve

Reviewers (engineering leads) review the document for:
- Completeness and accuracy
- Blameless tone
- Actionable and well-scoped action items
- Clear ownership and deadlines

### Step 6: Publish and Share

The postmortem is:
- Published on the internal wiki
- Summarized in the engineering all-hands
- Added to the incident archive
- Used as training material for new engineers

---

## Postmortem Document Template

```markdown
# Postmortem: [Incident Name]

**Date:** [Date of incident]
**Authors:** [Names]
**Status:** [Draft / In Review / Published]
**Severity:** [SEV-1 / SEV-2 / SEV-3]

## Summary

[2-3 sentence summary of the incident: what happened, impact, and resolution.]

## Impact

| Metric | Value |
|--------|-------|
| Duration | [X hours Y minutes] |
| Customers affected | [Number] |
| Data exposed | [Description, if applicable] |
| Financial impact | [Amount, if known] |
| Regulatory impact | [Description, if applicable] |

## Timeline

[Summary timeline. Reference full timeline in [incident-timeline.md](../incident-management/incident-timeline.md).]

| Time | Event |
|------|-------|
| ... | ... |

## Root Cause Analysis

### Root Cause

[Description of the root cause identified through 5 Whys analysis.]

### Contributing Factors

1. [Factor 1]
2. [Factor 2]
3. [Factor 3]

### What Went Well

1. [Positive observation 1]
2. [Positive observation 2]

### What Went Wrong

1. [Negative observation 1]
2. [Negative observation 2]

## Action Items

| # | Action | Owner | Deadline | Priority | Status |
|---|--------|-------|----------|----------|--------|
| 1 | [Description] | [@person] | [Date] | P0/P1/P2 | Open |
| 2 | [Description] | [@person] | [Date] | P0/P1/P2 | Open |
| 3 | [Description] | [@person] | [Date] | P0/P1/P2 | Open |

## Lessons Learned

### What did we learn?
[Key insights from the incident.]

### How can we apply this learning?
[How this applies to other systems or future incidents.]

## References

- [Incident ticket link]
- [War room recording]
- [Dashboards]
- [Related postmortems]
```

---

## Facilitation Best Practices

### Handling Blame

If a participant blames someone:
> "Let's step back. [Person] made the best decision they could with the information they had. Let's focus on why the system allowed this outcome and what we can change to prevent it."

### Handling Dominant Voices

If one person is dominating the discussion:
> "Thanks, [Name]. I'd like to hear from others. [Other name], what was your perspective on this?"

### Handling Silence

If no one is contributing:
> "Let me ask a specific question. [Name], when you saw [X], what was your thinking?"

### Handling Tangents

If the discussion goes off-topic:
> "That's an important topic, but it's separate from this incident. Let's capture it as a parking lot item and return to the incident."

### Handling Disagreement

If participants disagree on the root cause:
> "We have two hypotheses: [A] and [B]. What evidence supports each? What evidence would distinguish them?"

---

## Common Postmortem Anti-Patterns

### 1. The Blame Game

**Problem**: "Bob deployed without testing."
**Fix**: "The deployment process allowed an untested change to reach production. What gates should we add?"

### 2. The Root Cause Cop-Out

**Problem**: "Root cause: human error."
**Fix**: "Human error is a symptom, not a root cause. Why did the system allow the error to cause an incident?"

### 3. Action Item Bloat

**Problem**: 30 action items, none of which get done.
**Fix**: Prioritize ruthlessly. 3-5 high-impact action items are better than 30 aspirational ones.

### 4. The Postmortem That Goes Nowhere

**Problem**: Postmortem written and filed, never referenced again.
**Fix**: Track action items like product features. Review progress weekly. Share completed actions in all-hands.

### 5. The Blameless Postmortem That Is Not Really Blameless

**Problem**: "We are not blaming Bob, but Bob should have known better."
**Fix**: If you are naming individuals in a negative context, the postmortem is not blameless. Focus on systems.

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-command.md](incident-command.md) -- Incident commander role
- [action-item-tracking.md](action-item-tracking.md) -- Action item management
- [incident-timeline.md](incident-timeline.md) -- Timeline management
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
