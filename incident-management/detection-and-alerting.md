# Detection and Alerting

## Overview

Effective detection and alerting ensures that incidents are identified quickly and the right people are notified with actionable information. This document covers alert design, routing strategies, and noise reduction principles for the GenAI engineering organization.

---

## Alert Design Principles

### 1. Alerts Must Be Actionable

Every alert must answer these questions:
- **What is wrong?** Clear description of the condition
- **How severe is it?** Severity classification (SEV-1 through SEV-4)
- **What should I do?** Recommended first steps or runbook link
- **Who else needs to know?** Escalation path

**Bad alert**: "Error rate elevated"
**Good alert**: "[SEV-2] AssistBot error rate at 8% (threshold: 2%). Last deployment 45 minutes ago. Runbook: https://runbooks/assistbot-error-rate"

### 2. Alerts Must Be Specific

- Include exact metric values and thresholds
- Include the scope of impact (how many users, which services)
- Include relevant context (recent deployments, traffic changes)

### 3. Alerts Must Be Timely

- SEV-1: Alert within 1 minute of condition
- SEV-2: Alert within 3 minutes
- SEV-3: Alert within 10 minutes
- SEV-4: Alert within 30 minutes (or batched into digest)

### 4. Alerts Must Be deduplicated

- Multiple symptoms of the same incident should produce a single alert
- Alert grouping rules prevent alert storms
- Suppression rules prevent duplicate pages

---

## Alert Categories

### Infrastructure Alerts

| Alert Name | Condition | Severity | Runbook |
|-----------|-----------|----------|---------|
| HighCPUUsage | CPU > 85% for 5 minutes | SEV-3 | `/runbooks/high-cpu` |
| HighMemoryUsage | Memory > 90% for 5 minutes | SEV-2 | `/runbooks/high-memory` |
| DiskSpaceLow | Disk < 15% free | SEV-3 | `/runbooks/disk-space` |
| PodCrashLooping | Pod restarts > 3 in 10 minutes | SEV-2 | `/runbooks/pod-crashloop` |
| NodeNotReady | Node NotReady for > 2 minutes | SEV-2 | `/runbooks/node-not-ready` |

### Application Alerts

| Alert Name | Condition | Severity | Runbook |
|-----------|-----------|----------|---------|
| HighErrorRate | 5xx rate > 2% for 3 minutes | SEV-2 | `/runbooks/high-error-rate` |
| HighLatency | P95 latency > 500ms for 5 minutes | SEV-3 | `/runbooks/high-latency` |
| ServiceDown | Health check failing for > 1 minute | SEV-1 | `/runbooks/service-down` |
| HighRequestQueue | Request queue depth > 1000 | SEV-2 | `/runbooks/high-queue` |

### GenAI-Specific Alerts

| Alert Name | Condition | Severity | Runbook |
|-----------|-----------|----------|---------|
| ModelQualityDegraded | Accuracy drop > 5% from baseline | SEV-2 | `/runbooks/model-quality` |
| VectorDBLatency | P95 > 200ms for 5 minutes | SEV-2 | `/runbooks/vectordb-latency` |
| VectorDBDown | Vector DB unreachable for > 1 minute | SEV-1 | `/runbooks/vectordb-down` |
| TokenCostAnomaly | Daily cost > 150% of forecast | SEV-3 | `/runbooks/token-cost` |
| PromptInjectionDetected | Injection pattern matched in input | SEV-1 | `/runbooks/prompt-injection` |
| PIIInOutput | PII pattern detected in LLM output | SEV-1 | `/runbooks/pii-in-output` |
| EmbeddingPipelineStalled | Embedding queue not processing for > 30 minutes | SEV-3 | `/runbooks/embedding-stalled` |
| ModelDriftDetected | PSI > 0.2 for input distribution | SEV-2 | `/runbooks/model-drift` |
| SystemPromptModified | System prompt hash changed without approval | SEV-2 | `/runbooks/system-prompt-change` |

### Compliance and Security Alerts

| Alert Name | Condition | Severity | Runbook |
|-----------|-----------|----------|---------|
| AuditLogGap | No audit logs for > 10 minutes | SEV-2 | `/runbooks/audit-log-gap` |
| UnusualDataAccess | Access pattern anomaly detected | SEV-2 | `/runbooks/unusual-access` |
| UnauthorizedAPIKey | API key used from unauthorized IP | SEV-1 | `/runbooks/unauthorized-key` |
| RetentionPolicyViolation | Logs deleted before retention period | SEV-1 | `/runbooks/retention-violation` |

---

## Alert Routing

### Routing Rules

```yaml
# alert-routing.yaml

routes:
  # SEV-1: Page immediately, all the time
  - match:
      severity: sev-1
    receiver: page-primary-and-backup
    continue: true
    group_wait: 0s
    group_interval: 1m
    repeat_interval: 5m

  # SEV-2: Page during business hours, Slack after hours
  - match:
      severity: sev-2
    receiver: page-during-hours
    active_time_intervals:
      - business-hours
    group_wait: 2m
    group_interval: 5m
    repeat_interval: 15m

  - match:
      severity: sev-2
    receiver: slack-alerts
    active_time_intervals:
      - after-hours
    group_wait: 5m
    group_interval: 15m
    repeat_interval: 1h

  # SEV-3: Slack only, no page
  - match:
      severity: sev-3
    receiver: slack-alerts
    group_wait: 10m
    group_interval: 30m
    repeat_interval: 4h

  # SEV-4: Daily digest
  - match:
      severity: sev-4
    receiver: email-digest
    group_wait: 1h
    group_interval: 1h
    repeat_interval: 24h

time_intervals:
  - name: business-hours
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '08:00'
            end_time: '20:00'
        location: 'Europe/London'

  - name: after-hours
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '20:00'
            end_time: '08:00'
        location: 'Europe/London'
      - weekdays: ['saturday', 'sunday']
```

### Escalation Policies

```yaml
# escalation-policies.yaml

policies:
  genai-platform:
    - step: 1
      timeout: 5m
      target: on-call-primary
    - step: 2
      timeout: 5m
      target: on-call-backup
    - step: 3
      timeout: 10m
      target: engineering-lead
    - step: 4
      timeout: 15m
      target: vp-engineering

  genai-inference:
    - step: 1
      timeout: 5m
      target: inference-on-call
    - step: 2
      timeout: 5m
      target: ml-lead
    - step: 3
      timeout: 10m
      target: engineering-lead
```

---

## Noise Reduction

### The Problem

Alert fatigue is the #1 cause of missed incidents. When engineers receive too many alerts:
- They start ignoring them
- They disable notifications
- They miss real incidents in the noise
- On-call becomes unsustainable

### Noise Reduction Strategies

#### 1. Alert Grouping

Group related alerts into a single notification:

```yaml
group_by:
  - alertname
  - service
  - severity

group_wait: 30s      # Wait for related alerts before sending
group_interval: 5m   # Wait before sending additional alerts for same group
repeat_interval: 4h  # Wait before repeating the same alert
```

#### 2. Alert Suppression

Suppress alerts during known conditions:

```yaml
# Suppress alerts during planned maintenance
inhibit_rules:
  - source_match:
      alertname: PlannedMaintenance
    target_match_re:
      alertname: ".*"
    equal:
      - service

# Suppress low-priority alerts during active SEV-1
  - source_match:
      severity: sev-1
    target_match:
      severity: sev-4
    equal:
      - service
```

#### 3. Burn Rate Alerting

Use error budget burn rate instead of raw thresholds:

```
# Instead of: "Error rate > 2%"
# Use: "Error budget will be exhausted in 1 hour at current burn rate"

Burn rate = current_error_rate / error_budget_rate

If burn_rate > 14.4:  # Error budget exhausted in 1 hour
  severity: sev-1
  page: true

If burn_rate > 6:  # Error budget exhausted in 2 hours
  severity: sev-2
  page: true

If burn_rate > 3:  # Error budget exhausted in 6 hours
  severity: sev-3
  slack: true
```

#### 4. Multi-Window, Multi-Burn-Rate Alerts

Use multiple time windows to distinguish between spikes and sustained issues:

```
# Short window, high burn rate: fast incidents
short_window = 5m
short_burn_rate = 14.4
alert: short_window_error_rate / short_window_budget_rate > 14.4

# Long window, low burn rate: slow degradation
long_window = 1h
long_burn_rate = 6
alert: long_window_error_rate / long_window_budget_rate > 6

# Only alert if BOTH conditions are met (reduces false positives)
```

#### 5. Alert Triage Scoring

Score each alert on actionability:

| Criteria | Score |
|----------|-------|
| Requires immediate action? | +3 |
| Has a clear runbook? | +2 |
| Happened before? | +1 |
| Can be automated? | -2 |
| Happens daily? | -3 |

Alerts scoring < 2 should be reviewed and either improved or removed.

---

## Alert Testing

### Pre-Deployment Testing

Before deploying a new alert:
1. **Run against historical data**: Would this alert have fired in the past 30 days? How often?
2. **Validate the threshold**: Is the threshold based on actual baseline behavior?
3. **Test the runbook**: Does the linked runbook exist and is it accurate?
4. **Page a volunteer**: Send a test page and verify the full alert-to-resolution flow

### Quarterly Alert Review

Every quarter, review all alerts:
1. **Alert frequency**: How often did each alert fire?
2. **Action rate**: How often did alerts result in action vs. being false positives?
3. **Removal candidates**: Which alerts have < 10% action rate?
4. **Missing alerts**: What incidents were not detected by alerts?

### Alert Quality Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| Alert-to-Incident Ratio | < 5:1 | Alerts per actual incident (lower is better) |
| False Positive Rate | < 20% | Percentage of alerts that were not actionable |
| Mean Alerts per incident | < 3 | Alerts generated per incident (grouping effectiveness) |
| On-Call Pages per Week | < 5 | Sustainable on-call load |
| Alert Response Time | < 2 minutes | Time from page to acknowledgment |

---

## GenAI-Specific Detection Patterns

### Prompt Injection Detection

```python
def detect_prompt_injection(user_input: str) -> bool:
    """Detect common prompt injection patterns."""
    patterns = [
        r"ignore\s+all\s+previous\s+instructions",
        r"system\s*:\s*override",
        r"developer\s*mode",
        r"BEGIN\s*(?:EXECUTION|OUTPUT)",
        r"output\s+your\s+(?:system\s+)?prompt",
        r"CRITICAL\s+SYSTEM\s+OVERRIDE",
        r"\[INST\].*?\[/INST\].*?\[INST\]",  # Instruction hijacking
    ]
    for pattern in patterns:
        if re.search(pattern, user_input, re.IGNORECASE):
            return True
    return False
```

### Model Quality Degradation Detection

```python
def detect_model_drift(recent_predictions: List[Prediction],
                       baseline_distribution: Dict[str, float]) -> bool:
    """Detect if model prediction distribution has drifted."""
    current_distribution = compute_distribution(recent_predictions)
    psi = compute_psi(current_distribution, baseline_distribution)

    if psi > 0.25:
        trigger_alert("SEV-2", f"Model PSI drift detected: {psi:.3f}")
        return True
    elif psi > 0.1:
        trigger_alert("SEV-3", f"Model PSI drift warning: {psi:.3f}")
        return False
    return False
```

### Token Cost Anomaly Detection

```python
def detect_token_cost_anomaly(daily_cost: float,
                              historical_costs: List[float]) -> bool:
    """Detect unusual spike in daily token costs."""
    mean = np.mean(historical_costs[-30:])  # 30-day average
    std = np.std(historical_costs[-30:])
    z_score = (daily_cost - mean) / std

    if z_score > 3:  # 3 standard deviations above mean
        trigger_alert("SEV-3", f"Token cost anomaly: "
                      f"£{daily_cost:,.0f} vs £{mean:,.0f} avg (z={z_score:.1f})")
        return True
    return False
```

---

## Monitoring Stack

### Tools

| Tool | Purpose |
|------|---------|
| Prometheus | Metrics collection and alerting rules |
| Grafana | Dashboards and visualization |
| PagerDuty | Alert routing, escalation, on-call management |
| ELK Stack | Log aggregation and analysis |
| Jaeger | Distributed tracing |
| DCGM Exporter | GPU-specific metrics |
| Custom Monitors | GenAI-specific metrics (token cost, model quality, PII detection) |

### Dashboard Requirements

Every service must have dashboards covering:

1. **Golden Signals**: Latency, Traffic, Errors, Saturation
2. **GenAI Signals**: Token cost, model quality, retrieval accuracy, PII detection
3. **Infrastructure Signals**: CPU, Memory, GPU, Disk, Network
4. **Compliance Signals**: Audit log completeness, data retention status

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [triage-playbooks.md](triage-playbooks.md) -- Triage decision trees
- [incident-metrics.md](incident-metrics.md) -- MTTD, MTTR tracking
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
- [on-call-best-practices.md](on-call-best-practices.md) -- On-call rotation design
