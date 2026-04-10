# Incident Metrics

## Overview

Tracking incident metrics enables the organization to measure reliability, identify trends, and demonstrate improvement over time. This document defines key incident metrics, measurement methods, and reporting standards.

---

## Key Metrics

### Detection Metrics

#### MTTD (Mean Time To Detect)

**Definition**: The average time from when an incident begins to when it is detected by monitoring or reported.

**Formula**: `MTTD = SUM(detection_time - incident_start) / N_incidents`

**Target**: < 5 minutes

**Why it matters**: Shorter detection time means less customer impact before response begins.

**Improvement strategies:**
- Add missing alerts for undetected incident types
- Tune alert thresholds to reduce detection lag
- Implement anomaly detection for unknown failure modes
- Regular review of incidents detected by users vs. monitoring

#### MTTA (Mean Time To Acknowledge)

**Definition**: The average time from when an alert fires to when an on-call engineer acknowledges it.

**Formula**: `MTTA = SUM(acknowledge_time - alert_time) / N_incidents`

**Target**: < 2 minutes

**Why it matters**: Acknowledgment confirms the alert was received and someone is looking at it.

**Improvement strategies:**
- Ensure PagerDuty escalation policy is correct
- Reduce alert noise to prevent fatigue
- On-call training and preparedness

### Response Metrics

#### MTTR (Mean Time To Resolve)

**Definition**: The average time from when an incident is detected to when it is resolved.

**Formula**: `MTTR = SUM(resolution_time - detection_time) / N_incidents`

**Targets by severity:**
- SEV-1: < 4 hours
- SEV-2: < 8 hours
- SEV-3: < 24 hours
- SEV-4: < 72 hours

**Why it matters**: Shorter resolution time means less total customer impact.

**Improvement strategies:**
- Better runbooks and playbooks
- Faster rollback and mitigation mechanisms
- Experienced ICs
- Regular game day exercises

#### Time to Mitigate

**Definition**: The time from detection to when mitigation actions begin reducing impact.

**Formula**: `Time to Mitigate = mitigation_start - detection_time`

**Target**: < 30 minutes (SEV-1), < 60 minutes (SEV-2)

**Why it matters**: Mitigation is when customer impact starts decreasing.

### Frequency Metrics

#### Incident Frequency

**Definition**: The number of incidents per time period, categorized by severity.

**Measurement:**
- Incidents per month by severity
- Incidents per service per month
- Incidents per root cause category

**Target**: Decreasing trend over time

**Why it matters**: Decreasing frequency indicates improving system reliability.

#### Repeat Incident Rate

**Definition**: The percentage of incidents that have the same root cause as a prior incident.

**Formula**: `Repeat Rate = N_repeat_incidents / N_total_incidents * 100`

**Target**: 0%

**Why it matters**: Repeat incidents indicate that postmortem action items were not completed or were ineffective.

### Quality Metrics

#### Action Item Completion Rate

**Definition**: The percentage of postmortem action items completed by their deadline.

**Formula**: `Completion Rate = N_completed_on_time / N_total_action_items * 100`

**Target**: > 90%

**Why it matters**: Uncompleted action items leave the system vulnerable to repeat incidents.

#### False Positive Alert Rate

**Definition**: The percentage of alerts that did not correspond to actual incidents.

**Formula**: `False Positive Rate = N_false_alerts / N_total_alerts * 100`

**Target**: < 20%

**Why it matters**: High false positive rates cause alert fatigue and missed incidents.

### Availability Metrics

#### Service Availability

**Definition**: The percentage of time the service is available to customers.

**Formula**: `Availability = (total_time - downtime) / total_time * 100`

**Targets:**
- Customer-facing services: 99.95% (approximately 4.38 hours downtime per year)
- Internal services: 99.9% (approximately 8.76 hours downtime per year)

**Error Budget:**
- For 99.95% availability: 4.38 hours of allowable downtime per year
- For 99.9% availability: 8.76 hours of allowable downtime per year

---

## Measurement and Tracking

### Data Collection

Incident data is collected from:

| Source | Data |
|--------|------|
| PagerDuty | Alert times, acknowledgment times, escalation events |
| Incident tickets | Incident start/end times, severity, impact |
| Postmortems | Root cause categories, action items |
| Monitoring systems | Service availability, error rates |
| Jira | Action item status, completion dates |

### Dashboard

The incident metrics dashboard displays:

```
Incident Metrics Dashboard
===========================

CURRENT MONTH:
  SEV-1 incidents: 0 (target: 0)
  SEV-2 incidents: 2 (target: < 3)
  SEV-3 incidents: 5 (target: < 8)
  SEV-4 incidents: 12 (target: < 15)

DETECTION:
  MTTD: 3.2 minutes (target: < 5 minutes) ✓
  MTTA: 1.5 minutes (target: < 2 minutes) ✓

RESPONSE:
  MTTR (SEV-1): N/A (no SEV-1 this month)
  MTTR (SEV-2): 4.2 hours (target: < 8 hours) ✓
  MTTR (SEV-3): 8.5 hours (target: < 24 hours) ✓

FREQUENCY:
  Total incidents: 19 (last month: 24) ↓
  Repeat incidents: 0 (target: 0) ✓

QUALITY:
  Action item completion rate: 92% (target: > 90%) ✓
  False positive alert rate: 15% (target: < 20%) ✓

AVAILABILITY:
  AssistBot: 99.97% (target: 99.95%) ✓
  WealthAdvisor AI: 99.93% (target: 99.95%) ⚠
  ComplianceBot: 99.99% (target: 99.9%) ✓
```

### Trend Analysis

#### Monthly Trend Report

```
Monthly Incident Report -- [Month Year]

INCIDENT SUMMARY:
  Total: [N] (previous: [N], trend: [up/down/stable])
  SEV-1: [N] (target: 0)
  SEV-2: [N] (target: < 3)
  SEV-3: [N] (target: < 8)
  SEV-4: [N] (target: < 15)

KEY METRICS:
  MTTD: [X] min (target: < 5 min)
  MTTA: [X] min (target: < 2 min)
  MTTR (SEV-2): [X] hours (target: < 8 hours)
  MTTR (SEV-3): [X] hours (target: < 24 hours)

TOP ROOT CAUSES:
  1. [Category]: [N] incidents ([X]%)
  2. [Category]: [N] incidents ([X]%)
  3. [Category]: [N] incidents ([X]%)

TOP AFFECTED SERVICES:
  1. [Service]: [N] incidents
  2. [Service]: [N] incidents
  3. [Service]: [N] incidents

ACTION ITEM STATUS:
  Open: [N]
  Completed this month: [N]
  Overdue: [N]
  Completion rate: [X]%

AVAILABILITY:
  [Service 1]: [X]% (budget remaining: [Y] hours)
  [Service 2]: [X]% (budget remaining: [Y] hours)

RECOMMENDATIONS:
  1. [Recommendation based on trends]
  2. [Recommendation based on root causes]
  3. [Recommendation based on action items]
```

### Quarterly Trend Analysis

Quarterly analysis focuses on:
- Long-term trend in incident frequency
- Systemic root cause patterns
- Action item completion trends
- Availability budget consumption
- Comparison to industry benchmarks

---

## Root Cause Categories

Standardized root cause categories enable trend analysis:

| Category | Description | Examples |
|----------|-------------|----------|
| **Deployment** | Issues caused by code or configuration changes | Bad deployment, missing migration |
| **Infrastructure** | Hardware, network, or platform failures | Node failure, network partition |
| **Dependency** | External service or library failures | LLM API outage, PyPI package issue |
| **Capacity** | Resource exhaustion or insufficient scaling | GPU memory exhaustion, connection pool full |
| **Configuration** | Incorrect settings or parameters | Wrong threshold, timezone bug |
| **Data** | Data quality, corruption, or drift issues | Model drift, training data contamination |
| **Security** | Security vulnerabilities or attacks | Prompt injection, supply chain compromise |
| **Process** | Process or procedure failures | Missed security review, skipped testing |
| **Human Error** | Mistakes made during operations | Wrong command executed, wrong config applied |
| **GenAI-Specific** | AI-specific failure modes | Hallucination, model degradation, token cost explosion |

---

## Reporting Cadence

| Report | Audience | Frequency | Content |
|--------|----------|-----------|---------|
| **Real-time dashboard** | All engineers | Continuous | Current metrics |
| **Weekly summary** | Engineering teams | Weekly | Weekly metrics, open actions |
| **Monthly report** | Engineering leadership | Monthly | Full monthly analysis |
| **Quarterly review** | VP Engineering, CTO | Quarterly | Trends, benchmarks, strategy |
| **Annual report** | Board, regulators | Annually | Annual reliability summary |

---

## Error Budget Policy

### What Is an Error Budget?

An error budget is the allowable amount of downtime or unreliability for a service, based on its SLO.

For a service with 99.95% availability SLO:
- Error budget = 0.05% of time = 4.38 hours per year
- Each incident consumes part of the budget

### Budget Consumption Rate

```
Burn Rate = Actual unreliability / Budgeted unreliability

If burn_rate > 1: Consuming budget faster than planned
If burn_rate = 1: On track
If burn_rate < 1: Under budget
```

### Budget Exhaustion Actions

| Budget Remaining | Action |
|-----------------|--------|
| > 50% | Normal operations |
| 25-50% | Increased monitoring, review action items |
| 10-25% | Freeze non-critical deployments, prioritize reliability work |
| < 10% | All-hands review, reliability sprint required |
| 0% | Service is below SLO. All engineering prioritizes reliability until budget is positive. |

---

## GenAI-Specific Metrics

### Model Quality Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| **Accuracy** | Model prediction accuracy vs. labeled test set | > 95% |
| **Drift (PSI)** | Population Stability Index for prediction distribution | < 0.1 |
| **Human Correction Rate** | % of AI outputs corrected by humans | < 5% |
| **Hallucination Rate** | % of responses containing fabricated information | < 1% |

### Token Cost Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| **Cost per Query** | Average token cost per user query | Within budget |
| **Daily Cost Variance** | Standard deviation of daily costs | < 20% of mean |
| **Budget Variance** | Actual vs. budgeted monthly cost | < 10% over |

### Retrieval Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| **Retrieval Accuracy** | % of queries where top-K contains relevant documents | > 90% |
| **Average Retrieval Score** | Mean cosine similarity of top result | > 0.80 |
| **Retrieval Latency (P95)** | P95 time for vector search | < 100ms |

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [action-item-tracking.md](action-item-tracking.md) -- Action item management
- [detection-and-alerting.md](detection-and-alerting.md) -- Alert design
- [postmortem-process.md](postmortem-process.md) -- Postmortem process
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
