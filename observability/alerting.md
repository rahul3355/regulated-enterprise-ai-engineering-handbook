# Alerting Design for Banking GenAI Systems

## Alerting Philosophy

Alerts interrupt humans to demand attention. Each alert represents a bet that:
1. Something is wrong
2. A human can do something about it
3. The urgency justifies the interruption

In banking, alert fatigue is not just an engineering problem -- it is a regulatory risk. If engineers ignore alerts because 90% are false positives, the 10% that matter will also be ignored.

**Principle: Alert on symptoms, not causes.**

Alert when users are affected. Do not alert because a specific component is unhealthy -- alert because that component's failure is causing user-visible errors.

## Alert Categories

### Pages (Immediate Response Required)

Pages wake someone up. They should be rare, actionable, and urgent.

```
Criteria for a page:
- Users are actively affected (not "might be affected")
- A human can take action within minutes
- Waiting until morning would significantly worsen the situation

Expected page rate: 1-2 per week per team, not per day
```

**Banking GenAI page-worthy alerts**:

```
PAGE: Loan Advisor API availability below 99% (currently 94.2%)
  -> Users cannot get mortgage advice. Business impact is immediate.
  -> Action: Check service health, consider failover.

PAGE: PII detected in LLM prompt logs (compliance violation)
  -> Regulatory violation in progress. Must stop and remediate.
  -> Action: Disable logging, redact data, file compliance incident.

PAGE: LLM provider returning 500 errors for > 5 minutes
  -> All GenAI features degraded. Customer-facing impact.
  -> Action: Switch to fallback model or cached responses.
```

### Tickets (Response Within Business Hours)

Tickets indicate a problem that needs fixing but is not an emergency.

```
Criteria for a ticket:
- Something is wrong but not user-facing yet
- The issue will likely worsen if not addressed
- It can wait until the next business day

Expected ticket rate: 5-10 per week per team
```

**Banking GenAI ticket-worthy alerts**:

```
TICKET: Error budget burn rate at 3x for RAG pipeline latency SLO
  -> Not yet breaching SLO, but will breach in ~10 hours if trend continues.
  -> Action: Investigate latency increase, optimize vector DB queries.

TICKET: Disk usage on log storage at 80%
  -> Will fill in ~2 weeks at current rate.
  -> Action: Plan storage expansion or implement log rotation.

TICKET: Model version gpt-3.5-turbo-0125 deprecated by provider
  -> Provider will discontinue in 60 days.
  -> Action: Plan migration to newer model version.
```

### Notifications (Informational)

Notifications provide context without demanding action.

```
Criteria for a notification:
- Something noteworthy happened
- No immediate action needed
- Useful for debugging or trend analysis

Expected notification rate: Variable
```

**Examples**:

```
NOTIFY: Deployment of loan-advisor-api v2.4.1 completed
NOTIFY: Monthly token cost summary: $12,450 (within budget)
NOTIFY: Weekly SLO report generated and sent to #eng-reliability
```

## Alert Routing Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Alert Routing                             │
│                                                             │
│  Alert ──> Alertmanager ──> Routing Rules ──> Receiver      │
│  Manager                                                  │
│                                                             │
│  Routing Rules:                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  IF severity=critical AND env=production            │   │
│  │    -> Page on-call rotation (PagerDuty)             │   │
│  │                                                     │   │
│  │  IF severity=warning AND env=production             │   │
│  │    -> Ticket (Jira) + Slack #alerts-warning        │   │
│  │                                                     │   │
│  │  IF severity=info                                   │   │
│  │    -> Slack #alerts-info                           │   │
│  │                                                     │   │
│  │  IF team=ml-platform AND severity=critical          │   │
│  │    -> Page ML on-call + escalate to platform lead   │   │
│  │                                                     │   │
│  │  IF compliance=true                                 │   │
│  │    -> Page on-call + notify compliance-team         │   │
│  │    -> Create Jira in compliance project             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Alert Configuration (Alertmanager)

```yaml
# alertmanager.yaml
route:
  receiver: default-slack
  group_by: ['alertname', 'service', 'team']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
        environment: production
      receiver: pagerduty-critical
      continue: true
      group_wait: 10s
      repeat_interval: 15m

    - match:
        severity: warning
        environment: production
      receiver: jira-warning
      group_wait: 5m
      repeat_interval: 12h

    - match:
        compliance: "true"
      receiver: compliance-team
      group_wait: 1m
      repeat_interval: 1h

receivers:
  - name: default-slack
    slack_configs:
      - channel: '#alerts-info'
        send_resolved: true

  - name: pagerduty-critical
    pagerduty_configs:
      - service_key: $PAGERDUTY_KEY

  - name: jira-warning
    webhook_configs:
      - url: 'http://jira-webhook/alert'

  - name: compliance-teams
    slack_configs:
      - channel: '#compliance-alerts'
        send_resolved: true
    pagerduty_configs:
      - service_key: $COMPLIANCE_PAGERDUTY_KEY
```

## Burn Rate Alerting for SLOs

The most effective alerting strategy for production services is burn rate alerting:

```yaml
# SLO burn rate alerts for 99.9% availability (30-day window)
groups:
  - name: slo-burn-rates
    rules:
      # Page: 14.4x burn rate (budget exhausted in 2 hours)
      - alert: SLOBurnRateCritical
        expr: |
          multiwindow:
            # Short window confirms rapid burn
            (1 - avg_over_time(slo_error_ratio[5m])) / 0.001 > 14.4
            and
            # Long window confirms it is not a brief blip
            (1 - avg_over_time(slo_error_ratio[1h])) / 0.001 > 14.4
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "SLO error budget burning at 14.4x normal rate"
          runbook_url: "https://wiki.bank.internal/runbooks/slo-burn-rate"

      # Ticket: 3x burn rate (budget exhausted in 10 hours)
      - alert: SLOBurnRateWarning
        expr: |
          (1 - avg_over_time(slo_error_ratio[30m])) / 0.001 > 3
        for: 15m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "SLO error budget burning at 3x normal rate"
```

## Noise Reduction Strategies

### 1. Alert Grouping

Group related alerts to avoid alert storms:

```yaml
group_by: ['alertname', 'service', 'team']
group_wait: 30s      # Wait before sending first alert
group_interval: 5m   # Wait before sending update
```

### 2. Inhibition Rules

Suppress lower-priority alerts when a higher-priority alert covers the same issue:

```yaml
inhibit_rules:
  # If a critical alert is firing, suppress warning alerts for the same service
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'service']
```

### 3. Dead Man's Switch

Ensure the alerting system itself is working:

```yaml
# Always-firing alert that acts as a dead man's switch
- alert: DeadMansSwitch
  expr: vector(1)
  labels:
    severity: none
  annotations:
    description: "This alert is always firing. If you receive this notification, it means the alerting system is working."
```

### 4. Maintenance Windows

Suppress alerts during planned maintenance:

```yaml
# Silence all alerts for a service during maintenance
POST /api/v2/silences
{
  "matchers": [{"name": "service", "value": "loan-advisor-api"}],
  "startsAt": "2025-03-20T02:00:00Z",
  "endsAt": "2025-03-20T06:00:00Z",
  "createdBy": "deploy-pipeline",
  "comment": "Planned maintenance - database migration"
}
```

## Alert Quality Metrics

Track the quality of your alerting:

```
Alert Quality Metrics:
  - Page-to-action ratio: % of pages that result in action (target: > 80%)
  - Mean time to acknowledge (MTTA): Time from page to on-call response
  - Mean time to resolve (MTTR): Time from page to resolution
  - False positive rate: % of pages that were not actionable (target: < 5%)
  - Alert volume trend: Total alerts per week (should decrease over time)
  - Duplicate alert rate: % of alerts that are duplicates of existing alerts
```

## Common Alerting Mistakes

1. **Alerting on causes, not symptoms**: "CPU at 90%" is a cause. "Request latency increased" is a symptom. Alert on the symptom.

2. **No alert ownership**: Every alert must map to a specific team. Alerts without owners are ignored.

3. **Alert fatigue from staging**: Staging alerts should go to a different channel with different expectations. Never page on-call for staging issues.

4. **Missing runbook links**: Every page must include a link to a runbook. If there is no runbook, the page should not exist.

5. **Alerting on metrics without baselines**: "Disk at 70%" means nothing without context. Is that normal? Is it growing? Alert on trend, not just threshold.

6. **No escalation path**: If the primary on-call does not acknowledge within 15 minutes, who is next? Define escalation chains.

7. **Ignoring alert history**: If an alert fires 20 times a week and is always acknowledged without action, delete or fix it. It is noise.
