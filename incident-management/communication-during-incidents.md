# Communication During Incidents

## Overview

Effective communication during incidents ensures that all stakeholders -- engineers, executives, customers, and regulators -- have the information they need at the right time. This document defines communication procedures, templates, and responsibilities for all incident severities.

---

## Communication Roles

### Incident Commander (IC)

- Approves all external communications
- Ensures regular internal updates
- Decides when to escalate to executives

### Communications Lead

- Drafts and distributes stakeholder updates
- Manages the status page
- Coordinates customer and regulatory notifications
- Monitors social media and public sentiment

### Scribe

- Documents the incident timeline with all decisions and communications
- Ensures all communications are captured in the incident ticket

---

## Communication Channels

| Audience | Channel | Frequency | Owner |
|----------|---------|-----------|-------|
| Engineering team | Incident Slack channel | Continuous (real-time updates) | IC |
| Stakeholders (internal) | Email/Slack update | Every 30 min (SEV-1), every hour (SEV-2) | Comms Lead |
| Executives | Brief email or briefing call | Every hour (SEV-1), every 2 hours (SEV-2) | IC |
| Customers | Status page | Every 30 min during active incident | Comms Lead |
| Customers (notification) | Email/in-app notification | Once at detection, once at resolution | Comms Lead |
| Regulators | Formal notification | Per regulatory requirements (see [regulatory-notification.md](regulatory-notification.md)) | Legal/Compliance |
| Media | Press statement | If media attention exceeds threshold | PR/Communications |

---

## Communication Timeline

### SEV-1 Communication Schedule

| Time | Action | Audience | Content |
|------|--------|----------|---------|
| T+0 | Alert received | Engineering team | "Investigating potential service outage" |
| T+5 min | Triage complete | Engineering team | "Confirmed SEV-1: [description]. IC assigned. War room open." |
| T+15 min | First stakeholder update | Internal stakeholders | Brief summary of impact and response |
| T+30 min | First status page update | Customers | "We are investigating an issue affecting [service]. More info soon." |
| T+30 min | First stakeholder update | Internal stakeholders | Current status, impact, actions being taken |
| T+1 hour | Executive briefing | Executives | Detailed briefing with IC |
| T+1 hour | Status page update | Customers | Update with current understanding |
| T+1.5 hours | Stakeholder update | Internal stakeholders | Progress update |
| T+2 hours | Status page update | Customers | Update or "still investigating" |
| Every 30 min | Status page update | Customers | Regular updates until resolved |
| Every 30 min | Stakeholder update | Internal stakeholders | Regular updates until resolved |
| Every hour | Executive briefing | Executives | Regular updates until resolved |
| Resolution | Resolution notification | All audiences | "Issue resolved. Details to follow." |
| T+24 hours | Post-incident summary | Internal stakeholders | Blameless summary and action items |

### SEV-2 Communication Schedule

| Time | Action | Audience |
|------|--------|----------|
| T+15 min | Triage complete | Engineering team |
| T+1 hour | First stakeholder update | Internal stakeholders |
| T+2 hours | Status page update (if customer-facing) | Customers |
| Every hour | Stakeholder update | Internal stakeholders |
| Resolution | Resolution notification | All audiences |

### SEV-3/SEV-4 Communication

- No regular status page updates required
- Stakeholder notification at detection and resolution
- Resolution summary in sprint retrospective

---

## Status Page Guidelines

### What to Publish

1. **Incident title**: Clear, non-technical description
2. **Status**: Investigating / Identified / Monitoring / Resolved
3. **Affected services**: Which services are impacted
4. **Impact description**: What customers are experiencing (in plain language)
5. **What we are doing**: Brief description of response actions
6. **Next update**: When the next update will be posted

### What NOT to Publish

1. Root cause details (save for post-incident report)
2. Internal technical details (infrastructure, code, configurations)
3. Blame or attribution (never say "caused by a vendor" or "engineer error")
4. Speculative timelines ("should be fixed in 30 minutes" -- only commit when certain)
5. Security vulnerability details (until the vulnerability is patched)

### Status Page Templates

#### Initial Post (Investigating)

```
**Title**: We are investigating an issue affecting [Service Name]

**Status**: Investigating

**Impact**: Some users may experience [specific impact, e.g., "delayed responses from our AI assistant" or "errors when uploading documents"].

**What we are doing**: Our engineering team is actively investigating the issue. We will provide an update within 30 minutes.

**Started at**: [Timestamp]
```

#### Update Post (Identified)

```
**Title**: Update on [Service Name] issue

**Status**: Identified

**Impact**: Approximately [X]% of users are affected. [Specific description of impact].

**What we have identified**: We have identified the cause as [high-level description, non-technical]. [Optional: "A fix is being deployed."]

**What we are doing**: [Brief description of mitigation actions].

**Next update**: [Time]
```

#### Resolution Post

```
**Title**: [Service Name] issue resolved

**Status**: Resolved

**Summary**: The issue affecting [Service Name] has been resolved. All users should now see normal behavior.

**What happened**: [Brief, non-technical explanation. Save details for post-incident report.]

**What we are doing to prevent recurrence**: We are implementing [brief description of fix]. A detailed post-incident report will be published within 5 business days.

**Resolved at**: [Timestamp]
```

---

## Stakeholder Update Templates

### Internal Stakeholder Update (SEV-1)

```
Subject: SEV-1 Incident Update - [Service Name] - [Timestamp]

**Current Status**: [Investigating / Identified / Mitigating / Monitoring]

**Impact**:
- [Number] customers affected
- [Description of customer impact]
- [Financial impact if known]

**What We Know**:
- [Key finding 1]
- [Key finding 2]

**What We Are Doing**:
- [Action 1]
- [Action 2]

**ETA for Resolution**: [Estimate or "Too early to tell"]

**Next Update**: [Time]

**War Room**: [Link]
```

### Executive Briefing (SEV-1)

```
SEV-1 Executive Briefing - [Timestamp]

INCIDENT: [Name]
SEVERITY: SEV-1
IC: [Name]

CURRENT STATUS:
[2-3 sentence summary]

CUSTOMER IMPACT:
- [Number] customers affected
- [Description]
- [Complaint volume / social media sentiment if relevant]

BUSINESS IMPACT:
- Revenue impact: [if known]
- Regulatory risk: [yes/no, description]
- Reputational risk: [assessment]

RESPONSE:
- Root cause: [if known, or "under investigation"]
- Mitigation: [what is being done]
- ETA: [if known]

REGULATORY NOTIFICATION:
- Required: [yes/no]
- Status: [if applicable]

NEXT UPDATE: [Time]
```

---

## Customer Notification Templates

### SEV-1 Customer Notification (Email)

```
Subject: Important: Issue affecting [Service Name]

Dear [Customer Name],

We are writing to inform you about an issue currently affecting [Service Name].

**What happened:**
At approximately [time], we detected [brief, non-technical description]. This may have caused [specific impact the customer may have noticed].

**What we are doing:**
Our engineering team is actively working to resolve the issue. [Brief description of actions, e.g., "We have identified the cause and are implementing a fix."]

**What this means for you:**
- [Specific impact on the customer]
- [Any action the customer needs to take]
- [Any data or service the customer should be aware of]

**When to expect resolution:**
We expect to resolve this issue by [estimate]. We will provide another update by [time].

**We apologize for the inconvenience.**
We understand the impact this has on your experience and are working to resolve it as quickly as possible.

If you have questions, please contact [support contact].

Sincerely,
[Name, Title]
```

### Data Breach Customer Notification (Legal Review Required)

```
Subject: Important: Notice of Data Security Incident

Dear [Customer Name],

We are writing to inform you about a data security incident that may have affected your personal information.

**What happened:**
On [date], we discovered [brief, factual description]. We immediately took steps to contain the issue and launched an investigation with the assistance of leading cybersecurity experts.

**What information was involved:**
Based on our investigation, the following information may have been affected:
- [Specific data types]
- [Timeframe of exposure]

**What we are doing:**
- [Containment actions taken]
- [Notification to regulators]
- [Remediation steps]

**What you can do:**
- [Recommended actions for the customer]
- [Credit monitoring offer if applicable]
- [Contact information for questions]

**For more information:**
Visit [URL] or call [phone number].

We sincerely apologize for this incident and the concern it may cause.

Sincerely,
[Name, Title]
```

---

## Regulatory Communication

See [regulatory-notification.md](regulatory-notification.md) for detailed regulatory notification requirements.

Key principles:
1. **Timing**: Follow regulatory deadlines strictly (ICO: 72 hours, FCA: 24 hours for IBS)
2. **Content**: Factual, complete, and reviewed by legal counsel
3. **Channel**: Formal written notification to the appropriate regulatory body
4. **Follow-up**: Ongoing updates as investigation progresses

---

## Media Communication

### When to Engage Media

Media engagement is required when:
- The incident is covered by national or industry media
- Social media mentions exceed 100 posts in 1 hour
- A prominent customer or public figure comments publicly
- Regulatory action is publicly announced

### Media Statement Template

```
FOR IMMEDIATE RELEASE

[Company] Statement on [Service] Issue

[Date]

[Company] is aware of an issue affecting [Service] that may have impacted
some customers' ability to [specific impact].

We identified the issue at approximately [time] and immediately began
working to resolve it. The issue was resolved at [time].

[If data breach:] Our investigation indicates that [factual description of
data involved]. We have notified [relevant regulators] and are taking steps
to [remediation actions].

We take our customers' trust seriously and apologize for this incident.
Customers with questions can contact [contact information] or visit
[URL] for more information.

###
```

---

## Communication Anti-Patterns

### 1. Radio Silence

**Problem**: No updates for hours during an active incident.
**Fix**: Even if there is no progress, post: "We are still investigating. No new information to share. Next update in 30 minutes."

### 2. Over-Promising on ETAs

**Problem**: "Fixed in 30 minutes" turns into 4 hours.
**Fix**: Only commit to timelines when the fix is in progress and well-understood. Otherwise: "Too early to estimate."

### 3. Technical Jargon in Customer Communications

**Problem**: "The Kubernetes deployment caused a ConfigMap misconfiguration."
**Fix**: "A configuration change caused some requests to fail."

### 4. Blaming External Parties

**Problem**: "This was caused by our vendor's outage."
**Fix**: "We experienced a service disruption. We are working to restore normal operation."

### 5. Inconsistent Messages

**Problem**: Status page says "resolved" but Slack says "still investigating."
**Fix**: Single source of truth. All communications coordinated through the Comms Lead.

---

## Post-Incident Communication

### Internal Post-Incident Summary

Within 24 hours of resolution:
- Blameless summary of what happened
- Timeline of events
- Root cause (if known)
- Action items identified
- Lessons learned

### Customer Post-Incident Report

For SEV-1 incidents, publish a customer-facing post-incident report within 5 business days:
- What happened (non-technical)
- Impact on customers
- What we did to resolve it
- What we are doing to prevent recurrence
- Apology and commitment to improvement

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-command.md](incident-command.md) -- Incident commander role
- [war-room-management.md](war-room-management.md) -- War room procedures
- [regulatory-notification.md](regulatory-notification.md) -- Regulatory notification requirements
- [customer-communication.md](customer-communication.md) -- Customer notification templates
- [incident-timeline.md](incident-timeline.md) -- Timeline management
