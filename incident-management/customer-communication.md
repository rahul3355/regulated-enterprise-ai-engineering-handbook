# Customer Communication

## Overview

Timely, honest, and empathetic communication with customers during incidents is essential for maintaining trust. This document provides customer notification templates, timing guidance, and content standards for all incident severities.

---

## Communication Principles

### 1. Be Honest

Never minimize or hide the impact. Customers can tell when something is wrong, and dishonesty damages trust far more than the incident itself.

**Bad**: "We are experiencing minor technical difficulties."
**Good**: "Our AI assistant is currently unavailable. We are working to restore service."

### 2. Be Timely

Communicate as soon as you confirm customer impact. Do not wait until you have a fix or an ETA.

**Rule**: First customer communication within 30 minutes of confirmed impact (SEV-1) or 2 hours (SEV-2).

### 3. Be Specific

Describe what customers are experiencing, not what is happening technically.

**Bad**: "Our Kubernetes cluster is experiencing pod restarts."
**Good**: "You may see errors when using our AI assistant. Our team is working to fix this."

### 4. Be Empathetic

Acknowledge the impact on customers and apologize sincerely.

**Bad**: "We apologize for any inconvenience."
**Good**: "We understand this is frustrating, and we apologize for the impact on your experience. We are working hard to resolve this."

### 5. Set Realistic Expectations

Do not commit to an ETA unless you are confident. It is better to say "we do not have an estimate yet" than to miss a promised time.

---

## Timing Guidelines

| Severity | First Communication | Update Frequency | Resolution Communication |
|----------|-------------------|-----------------|------------------------|
| SEV-1 | Within 30 minutes | Every 30 minutes | Within 15 minutes of resolution |
| SEV-2 | Within 2 hours | Every 1 hour | Within 30 minutes of resolution |
| SEV-3 | Within 4 hours (if impact persists > 2 hours) | Every 2 hours | At resolution |
| SEV-4 | No proactive communication required | N/A | If customer reported, respond directly |

---

## Communication Channels

| Channel | When to Use | Audience |
|---------|-------------|----------|
| **Status Page** | All customer-facing incidents | All customers |
| **In-App Notification** | Active sessions during incident | Currently active users |
| **Email** | SEV-1 with significant impact | Affected customers |
| **Social Media** | If incident is discussed publicly | Public |
| **Direct Contact** | If specific customers are individually impacted | Named customers |
| **Phone** | For enterprise/VIP customers during SEV-1 | Account managers to customers |

---

## Customer Notification Templates

### Template 1: SEV-1 -- Complete Service Outage

#### Status Page (Initial)

```
Status: Major Outage

Our AI assistant is currently unavailable for all users.

We are aware of the issue and our engineering team is actively
working to restore service.

We will provide another update within 30 minutes.

Started at: [Time, Timezone]
```

#### Status Page (Update)

```
Status: Major Outage

Update: We have identified the cause of the outage and are
implementing a fix. We expect service to begin recovering
within the next [timeframe / or "we will provide another
update in 30 minutes"].

We apologize for the continued disruption.

Next update: [Time]
```

#### Status Page (Resolved)

```
Status: Resolved

Our AI assistant is fully operational again. All users should
see normal behavior.

What happened: [Brief, non-technical explanation]

We are investigating the root cause to prevent this from
happening again. A detailed report will be published within
5 business days.

We sincerely apologize for this disruption.

Resolved at: [Time]
```

#### Email (SEV-1, Significant Impact)

```
Subject: Important: Issue affecting [Service Name] -- Now Resolved

Dear [Customer Name],

I am writing to update you on the issue that affected [Service Name]
today between [start time] and [end time].

**What happened:**
[Brief, non-technical explanation. E.g., "A technical issue caused
our AI assistant to be unavailable for approximately X hours."]

**How you were affected:**
[Specific impact. E.g., "During this time, you may have been unable
to get answers from our AI assistant or received error messages."]

**What we did:**
Our engineering team identified and resolved the issue. Service
has been fully restored since [time].

**What we are doing to prevent recurrence:**
[Brief description. E.g., "We are implementing additional monitoring
and safeguards to detect and prevent this issue in the future."]

We understand how important [Service Name] is to your experience,
and we sincerely apologize for the disruption.

If you have any questions, please contact us at [support contact].

Sincerely,
[Name, Title]
```

### Template 2: SEV-2 -- Partial Degradation

#### Status Page (Initial)

```
Status: Degraded Performance

Some users may experience slower response times from our AI
assistant.

We are investigating the issue and will provide an update
within 1 hour.

Started at: [Time]
```

#### Status Page (Resolved)

```
Status: Resolved

The performance issue affecting our AI assistant has been
resolved. Response times have returned to normal.

We apologize for the inconvenience.

Resolved at: [Time]
```

### Template 3: Data Exposure Incident

#### Email (Data Exposure)

**Note:** This template must be reviewed by Legal before use.

```
Subject: Important: Notice Regarding Your Account Information

Dear [Customer Name],

We are writing to inform you about an incident that may have
affected your personal information.

**What happened:**
On [date], we discovered an issue with our AI assistant that
may have allowed some users to see information belonging to
other customers.

**What information was involved:**
Based on our investigation, the following types of information
may have been visible to other users:
- [Specific data types: e.g., "Account holder name, account
  number, transaction history"]
- [Timeframe of exposure]

**What we did:**
We immediately took steps to fix the issue and prevent further
exposure. Our investigation found that approximately [number]
customers may have been affected.

**What we are doing:**
- We have fixed the technical issue that caused this exposure
- We are conducting a thorough review of our systems
- We have notified [relevant regulators]
- [Optional: We are offering two years of free credit monitoring]

**What you can do:**
- Review your account activity for anything unusual
- Contact us immediately if you see any unauthorized activity
- [Additional recommended actions]

**For more information:**
Visit [URL] or call [phone number] for assistance.

We sincerely apologize for this incident and any concern it
may cause. Protecting your information is our highest priority.

Sincerely,
[Name, Title]
```

### Template 4: Incorrect AI Advice Incident

#### Email (Incorrect Advice)

```
Subject: Important: Update Regarding [Service Name] Advice

Dear [Customer Name],

We are writing to inform you about an issue that may have
affected the accuracy of advice provided by our AI assistant.

**What happened:**
Between [start date] and [end date], our AI assistant may have
provided advice that was incomplete or inaccurate regarding
[specific topic, e.g., "mortgage application requirements"].

**How you may have been affected:**
If you used our AI assistant during this period, the information
you received may not have been fully accurate. Specifically:
- [Specific description of what was wrong]

**What you should do:**
- If you received advice about [topic] during this period, please
  contact us to verify the information
- [Specific action if applicable]

**What we have done:**
We have corrected the issue and verified the accuracy of all
advice currently being provided by our AI assistant.

We apologize for any confusion this may have caused. If you have
questions, please contact [support contact].

Sincerely,
[Name, Title]
```

---

## Enterprise / VIP Customer Communication

For enterprise and VIP customers, account managers should provide direct communication:

### Phone Call Script (SEV-1)

```
"Hi [Customer Name], this is [Your Name] from [Bank]. I'm calling
to let you know about an issue affecting [Service Name].

Currently, [brief description of impact]. Our engineering team is
actively working on it.

I want to make sure you are aware of the situation and know that
we are on it. I'll personally update you [timeframe].

Do you have any immediate concerns I can help with?"
```

### Follow-Up Email

```
Subject: Update on [Service Name] Issue

Dear [Customer Name],

Following up on our call, here is the current status:

- Current status: [Investigating / Identified / Resolved]
- Impact: [Description]
- What we are doing: [Brief description]
- Next update: [Time]

My direct line is [number]. Please reach out if you have any
questions or concerns.

Best regards,
[Name]
[Title]
[Direct contact]
```

---

## Social Media Response

### Monitoring

During significant incidents, monitor:
- Twitter mentions of the bank and service
- Reddit discussions
- Trustpilot reviews
- News articles

### Response Template

```
Hi [Customer], we're aware of the issue affecting [Service] and
our team is working on it. We'll share updates on our status
page: [link]. We apologize for the inconvenience.
```

### When to Escalate to Media Statement

Escalate when:
- National or industry media picks up the story
- A prominent figure comments publicly
- Social media mentions exceed 100 posts per hour
- A customer complaint goes viral

See [communication-during-incidents.md](communication-during-incidents.md) for media statement templates.

---

## Post-Incident Customer Communication

### Post-Incident Report (SEV-1)

Within 5 business days, publish a customer-facing post-incident report:

```
Post-Incident Report: [Service Name] Outage on [Date]

**Summary:**
On [date], [Service Name] was unavailable from [start time] to
[end time] ([duration]).

**What happened:**
[Non-technical explanation of root cause]

**Impact:**
- [Number] customers affected
- [Duration] of service disruption
- [Any data impact, or "No data was affected"]

**What we did:**
[Description of response and resolution]

**What we are doing to prevent recurrence:**
[Description of systemic fixes]

**We apologize:**
We understand that [Service Name] is important to you, and we
are committed to earning back your trust.

[Link to status page for ongoing monitoring]
```

### Customer Compensation

For significant incidents, consider:
- SLA credits for enterprise customers
- Service fee waivers
- Free credit monitoring (for data exposure)
- Priority support for affected customers

Compensation decisions are made by the IC in consultation with product and finance.

---

## Customer Communication Anti-Patterns

### 1. "No Comment"

**Problem**: Refusing to comment makes customers assume the worst.
**Fix**: "We are aware of the issue and investigating. We will provide an update within [timeframe]."

### 2. Technical Jargon

**Problem**: "The vector DB index compaction caused IOPS exhaustion."
**Fix**: "A maintenance operation caused our search system to become overloaded."

### 3. Blaming Others

**Problem**: "This was caused by our cloud provider's outage."
**Fix**: "We experienced a service disruption. We are working to restore service."

### 4. Minimizing Impact

**Problem**: "Only a small number of users were affected."
**Fix**: "Approximately [X] users were affected. If you were one of them, we apologize."

### 5. Premise Resolution

**Problem**: Declaring resolution before all customers are restored.
**Fix**: "The fix is deployed and we are monitoring. Most users should see improvement within [time]. If you are still experiencing issues, please contact support."

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [communication-during-incidents.md](communication-during-incidents.md) -- Communication procedures
- [regulatory-notification.md](regulatory-notification.md) -- Regulatory notification requirements
- [incident-classification.md](incident-classification.md) -- Severity classification
