# On-Call Best Practices

## Overview

A well-designed on-call rotation ensures incidents are responded to quickly while maintaining engineer wellbeing and sustainability. This document covers on-call rotation design, handoff procedures, escalation policies, and sustainability practices.

---

## On-Call Philosophy

### Principles

1. **Sustainable**: On-call should not cause burnout. If it does, the system is broken, not the engineer.
2. **Fair**: Workload is distributed equitably across the team.
3. **Prepared**: On-call engineers have the tools, runbooks, and training to respond effectively.
4. **Compensated**: On-call work is recognized and compensated (time off, bonus, or other).
5. **Improving**: On-call feedback drives system reliability improvements.

---

## Rotation Design

### Rotation Types

| Type | Duration | Pros | Cons | Best For |
|------|----------|------|------|----------|
| **Weekly** | 7 days | Fewer handoffs, longer recovery | Longer stretches of on-call | Mature systems, low alert volume |
| **Daily** | 24 hours | Short stretches, quick recovery | Frequent handoffs | High alert volume, new systems |
| **Follow-the-Sun** | Regional handoff | 24/7 coverage with daytime only | Requires global team | Large, distributed organizations |

### Recommended: Weekly Rotation

For the GenAI engineering organization, we use a weekly rotation:

- **Start**: Monday at 10:00 AM
- **End**: Following Monday at 10:00 AM
- **Primary**: One engineer responsible for responding to pages
- **Backup**: Next engineer in rotation (paged if primary does not respond within 5 minutes)

### Rotation Schedule

```
Rotation: genai-platform-oncall

Week 1 (Jan 1-7):  Alice (Platform Squad)
Week 2 (Jan 8-14): Bob (Inference Squad)
Week 3 (Jan 15-21): Carol (Data Squad)
Week 4 (Jan 22-28): Dave (Frontend Squad)
Week 5 (Jan 29-Feb 4): Eve (Platform Squad)
Week 6 (Feb 5-11): Frank (Inference Squad)
...
```

### Rotation Rules

1. **Minimum 4 weeks between on-call weeks**: Engineers should not be on-call more than once per month.
2. **No consecutive weeks**: Engineers should have at least one week off between on-call stretches.
3. **No on-call during PTO**: If an engineer is on leave, swap with the next person in rotation.
4. **Maximum 2 weeks per quarter**: No engineer should be on-call more than 2 weeks per quarter.

---

## Escalation Policy

### Standard Escalation Policy

```yaml
# PagerDuty escalation policy for genai-platform

escalation_policy:
  name: genai-platform-escalation
  steps:
    - step: 1
      timeout: 5 minutes
      notify:
        - on-call-primary
      urgency: high

    - step: 2
      timeout: 5 minutes
      notify:
        - on-call-backup
      urgency: high

    - step: 3
      timeout: 10 minutes
      notify:
        - engineering-lead
      urgency: high

    - step: 4
      timeout: 15 minutes
      notify:
        - vp-engineering
      urgency: high
```

### Escalation Policy per Severity

| Severity | Step 1 | Step 2 | Step 3 | Step 4 |
|----------|--------|--------|--------|--------|
| SEV-1 | Page primary (immediate) | Page backup (5 min) | Engineering lead (10 min) | VP Engineering (15 min) |
| SEV-2 | Page primary (immediate) | Page backup (5 min) | Engineering lead (15 min) | -- |
| SEV-3 | Slack notification | Ticket created | Triage during business hours | -- |
| SEV-4 | Ticket created | Triage next business day | -- | -- |

---

## On-Call Responsibilities

### During the On-Call Week

1. **Be Available**
   - Keep phone on and charged at all times
   - Respond to pages within the SLA (5 minutes for SEV-1/SEV-2)
   - Stay within a reasonable response distance to a computer
   - Notify the backup if you will be temporarily unavailable

2. **Monitor Proactively**
   - Review dashboards at the start of each day
   - Check for trends in error rates, latency, and cost
   - Look for upcoming maintenance or deployments that may cause issues

3. **Respond to Pages**
   - Acknowledge pages within the SLA
   - Triage per [triage-playbooks.md](triage-playbooks.md)
   - Escalate if needed
   - Document the incident in the incident ticket

4. **Handoff Preparation**
   - Document any open incidents or concerns for the next on-call
   - Update runbooks if gaps were discovered
   - Complete the on-call handoff checklist

### What On-Call Is NOT

- **Not 24/7 coding**: On-call is about responding to incidents, not building features.
- **Not a punishment**: On-call is a shared responsibility, not a penalty.
- **Not a solo effort**: On-call engineers should escalate and get help when needed.

---

## Handoff Procedure

### On-Call Handoff Checklist

**Completed by outgoing on-call, reviewed by incoming on-call:**

- [ ] **Open incidents**: Any active incidents being managed?
- [ ] **Recent incidents**: Any incidents from the past week that the incoming should be aware of?
- [ ] **Deployments**: Any deployments planned during the incoming's week?
- [ ] **Maintenance**: Any scheduled maintenance during the incoming's week?
- [ ] **Known issues**: Any known issues that may page the incoming?
- [ ] **Runbook updates**: Any runbooks updated during the week?
- [ ] **Alert changes**: Any alert threshold changes or new alerts added?
- [ ] **PagerDuty check**: Confirm escalation policy is correct and incoming is listed as backup.
- [ ] **Tools access**: Confirm incoming has access to all required tools (Grafana, PagerDuty, kubectl, etc.).
- [ ] **Slack channel**: Incoming is added to the on-call Slack channel.

### Handoff Meeting (15 minutes, Monday 10:00 AM)

```
Outgoing: "Here is what happened this week:
- 2 SEV-3 incidents, both resolved
- 1 SEV-2 on Wednesday, resolved postmortem scheduled for Thursday
- Deployment of AssistBot v2.5 on Friday, monitoring looks good
- No open concerns going into your week

Incoming, do you have any questions?"

Incoming: "I see there's a deployment planned for Wednesday. Anything I should know?"

Outgoing: "Yes, the deployment is for the new RAG retrieval optimization. The team
has tested it in staging. If you see retrieval latency spike after the deployment,
consider a rollback."

Incoming: "Got it. I'm good to take over."

Outgoing: "Handoff complete. I'll update PagerDuty."
```

---

## Compensation and Wellbeing

### Compensation

On-call work is compensated through:

1. **On-call allowance**: Additional payment for each week of on-call
2. **Comp time**: If pages occur outside business hours, equivalent time off is provided
3. **SEV-1 response bonus**: Additional compensation for SEV-1 responses outside business hours
4. **Recognition**: On-call contributions recognized in performance reviews

### Wellbeing Practices

1. **Sleep Protection**: If an on-call engineer is paged during the night, they should take the following morning off.

2. **Weekend Recovery**: If a SEV-1 occurs on a weekend, the on-call engineer gets the following Monday off.

3. **On-Call Load Limit**: If an engineer receives more than 5 pages in a week, the team reviews and addresses the root cause.

4. **Opt-Out Option**: If an engineer needs a break from on-call (personal reasons, health, etc.), they can opt out for one rotation with no questions asked.

5. **Burnout Check**: Engineering leads check in with on-call engineers monthly about on-call sustainability.

---

## On-Call Preparation

### On-Call Readiness Checklist

Before starting an on-call rotation, engineers must:

- [ ] Complete on-call training (4-hour course)
- [ ] Shadow an experienced on-call engineer for 1 week
- [ ] Reverse shadow (lead with supervision) for 1 week
- [ ] Review all runbooks and playbooks
- [ ] Confirm access to all required tools
- [ ] Confirm PagerDuty escalation policy is correct
- [ ] Install PagerDuty mobile app and test notifications
- [ ] Join the on-call Slack channel
- [ ] Review recent incidents and postmortems

### Required Tools Access

| Tool | Purpose | Access Level |
|------|---------|-------------|
| Grafana | Dashboards and metrics | Read access |
| PagerDuty | Alert management | On-call role |
| Kubernetes (kubectl) | Pod management, rollback | Namespace-level access |
| ELK Stack | Log analysis | Read access |
| ArgoCD | Deployment management | Deploy and rollback access |
| Slack | Communication | On-call channel access |
| Incident Platform | Incident tracking | Full access |
| Runbook Wiki | Runbooks and playbooks | Read and edit access |

---

## Alert Quality for On-Call

### Good Alert Characteristics

1. **Actionable**: The on-call engineer can do something about it
2. **Specific**: Clear description of what is wrong
3. **Contextual**: Includes relevant information (service, metric value, threshold)
4. **Linked**: Includes a link to the relevant runbook
5. **Classified**: Includes the severity level

### Good Alert Example

```
[SEV-2] AssistBot: Error rate at 8.5% (threshold: 2%)
Service: assistbot-api
Endpoint: /v1/chat
Time: 2024-03-15 14:23 UTC
Runbook: https://runbooks/assistbot-error-rate
Dashboard: https://grafana/d/assistbot
```

### Bad Alert Example

```
Error rate high
```

### On-Call Alert Feedback

On-call engineers should provide feedback on alert quality:

- **False positive**: Alert fired but nothing was wrong
- **Not actionable**: Alert fired but there was nothing the on-call could do
- **Missing context**: Alert fired but the on-call could not determine what was wrong
- **Good alert**: Alert fired, was actionable, and led to resolution

This feedback is reviewed weekly and used to improve alert quality.

---

## On-Call Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| **Pages per week** | < 3 | Average number of pages per on-call week |
| **SEV-1 pages per quarter** | < 2 | Number of SEV-1 pages per quarter |
| **Response time** | < 2 minutes | Average time from page to acknowledgment |
| **On-call satisfaction** | > 7/10 | Engineer satisfaction with on-call experience |
| **Alert quality score** | > 80% | Percentage of alerts that were actionable |
| **Burnout indicators** | 0 | Engineers reporting on-call-related burnout |

---

## Common On-Call Anti-Patterns

### 1. The Hero On-Call

**Problem**: One engineer always gets the most pages because they are the most experienced.
**Fix**: Distribute knowledge through runbooks, training, and pair on-call.

### 2. The Silent On-Call

**Problem**: On-call engineer receives pages but does not acknowledge or respond.
**Fix**: Escalation policy pages the backup. Address the root cause (burnout, access issues).

### 3. The Overloaded On-Call

**Problem**: On-call engineer receives 20+ pages in a week.
**Fix**: This is a system reliability issue. Conduct a reliability review and fix root causes.

### 4. The Unprepared On-Call

**Problem**: On-call engineer does not know what to do when paged.
**Fix**: Better runbooks, training, and shadowing.

### 5. The Always-On On-Call

**Problem**: On-call engineer feels they can never disconnect.
**Fix**: Clear boundaries. The backup covers when the primary is unavailable. Trust the escalation policy.

---

## GenAI-Specific On-Call Considerations

### GenAI Alerts the On-Call Must Be Prepared For

| Alert | Response | Runbook |
|-------|----------|---------|
| Model quality degraded | Check model version, rollback if needed | `/runbooks/model-quality` |
| Vector DB unreachable | Activate fallback, check provider status | `/runbooks/vectordb-down` |
| Prompt injection detected | Disable affected feature, assess scope | `/runbooks/prompt-injection` |
| PII in output detected | Take service offline, assess data exposure | `/runbooks/pii-output` |
| Token cost anomaly | Identify cost driver, apply cost controls | `/runbooks/token-cost` |
| GPU memory exhaustion | Identify resource hog, evict or redistribute | `/runbooks/gpu-exhaustion` |
| Embedding pipeline stalled | Check embedding service, restart if needed | `/runbooks/embedding-stalled` |

### On-Call GenAI Knowledge Requirements

On-call engineers should understand:
- How the RAG pipeline works (embedding -> retrieval -> generation)
- How to check model quality and compare versions
- How to identify prompt injection attempts
- How to assess PII leakage scope
- How to check token cost breakdown by service
- How to diagnose GPU resource issues
- How to failover between LLM providers

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-command.md](incident-command.md) -- Incident commander role
- [detection-and-alerting.md](detection-and-alerting.md) -- Alert design
- [triage-playbooks.md](triage-playbooks.md) -- Triage patterns
- [game-days.md](game-days.md) -- Game day exercises
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
- [incident-metrics.md](incident-metrics.md) -- Incident metrics
