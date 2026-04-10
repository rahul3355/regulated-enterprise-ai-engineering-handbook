# Service Level Objectives (SLOs) for Banking GenAI

## What Is an SLO?

An SLO (Service Level Objective) is a target for a service's reliability, expressed as a percentage over a time window. It is the agreement between the service team and its users about how reliable the service needs to be.

```
SLO = SLI <= Target over Window

Example:
  SLI: Availability (successful requests / total requests)
  Target: 99.9%
  Window: 30-day rolling
```

## SLI vs SLO vs SLA

```
┌─────────────────────────────────────────────────────┐
│               SLI / SLO / SLA                        │
│                                                     │
│  SLI (Indicator):  What you measure                 │
│    "Our availability is 99.95%"                     │
│                                                     │
│  SLO (Objective):  Your internal target             │
│    "We target 99.9% availability"                   │
│                                                     │
│  SLA (Agreement):  Contractual commitment to users  │
│    "We guarantee 99.5% availability or credit"      │
│                                                     │
│  SLA < SLO < 100%                                   │
│  The gap between SLA and SLO is your safety margin. │
└─────────────────────────────────────────────────────┘
```

In banking, SLAs often have financial penalties and regulatory implications. SLOs should be stricter than SLAs to provide a buffer.

## Why SLOs Matter in Banking GenAI

1. **Objective Launch Decisions**: "We cannot launch the mortgage advisor because it is at 98.5% availability and our SLO is 99.9%" is an objective statement, not an opinion.

2. **Error Budget Policy**: SLOs define how much unreliability is acceptable. If the error budget is exhausted, the team must focus on reliability over features.

3. **Regulatory Confidence**: Banking regulators ask "How reliable is your AI advisory system?" An SLO with historical compliance data is a defensible answer.

4. **Prioritization**: When deciding between building a new feature and fixing technical debt, the error budget status is the tiebreaker.

## Choosing the Right SLOs

### The Four Golden Signals

Start with these four SLO categories:

| Signal | What It Measures | Banking GenAI Example |
|--------|-----------------|----------------------|
| **Latency** | Time to serve requests | p95 response time < 3s for chat |
| **Traffic** | Demand on the system | Query volume handled per minute |
| **Errors** | Rate of failed requests | Error rate < 0.1% |
| **Saturation** | How full the system is | GPU utilization < 85% |

### Banking-Specific SLOs

Beyond the golden signals, banking GenAI systems need domain-specific SLOs:

```
┌────────────────────────────────────────────────────────────────┐
│  BANKING GENAI SLO CATALOG                                     │
├─────────────────────┬────────────┬──────────┬──────────────────┤
│  SLO                │  Target    │  Window  │  Rationale       │
├─────────────────────┼────────────┼──────────┼──────────────────┤
│  Availability       │  99.9%     │  30 day  │  Customer trust  │
│  Latency (p95)      │  < 3s      │  30 day  │  UX requirement  │
│  Response Quality   │  > 95%     │  7 day   │  Advice accuracy   │
│  Compliance Logging │  100%      │  30 day  │  Regulatory req  │
│  PII Redaction      │  100%      │  30 day  │  Privacy law     │
│  Data Freshness     │  < 1h      │  30 day  │  Accurate advice   │
│  Token Cost/Query   │  < $0.05   │  30 day  │  Cost control    │
│  Audit Trail        │  100%      │  30 day  │  Compliance      │
└─────────────────────┴────────────┴──────────┴──────────────────┘
```

**Critical**: Some SLOs in banking have a target of 100%. This does not mean perfection is achievable -- it means any failure is treated as a severity-1 incident requiring immediate response.

## SLO Calculation

### Availability SLO

```
Availability SLI = (Successful requests / Total requests) * 100

Example for 30-day window:
  Total requests: 15,000,000
  Failed requests: 4,500
  Availability = (15,000,000 - 4,500) / 15,000,000 * 100
  Availability = 99.97%

  SLO Target: 99.9%
  Result: PASSING (99.97% > 99.9%)
  Error Budget Remaining: (99.97 - 99.9) / (100 - 99.9) = 70%
```

### Latency SLO

```
Latency SLI = (Requests served within threshold / Total requests) * 100

Example:
  Total requests: 15,000,000
  Requests < 3s: 14,700,000
  Latency SLI = 14,700,000 / 15,000,000 * 100 = 98.0%

  SLO Target: 99.0%
  Result: FAILING (98.0% < 99.0%)
  Error Budget Remaining: 0% (exhausted)
```

## Error Budgets

The error budget is the amount of unreliability allowed before the SLO is violated.

```
Error Budget = 100% - SLO Target

For a 99.9% SLO:
  Error Budget = 0.1% = 1 in 1,000 requests can fail

  Over 30 days with 15M requests:
  Allowed failures = 15,000,000 * 0.001 = 15,000

  If you have 4,500 failures so far:
  Budget remaining = (15,000 - 4,500) / 15,000 = 70%
```

### Error Budget Burn Rate

Burn rate measures how fast you are consuming the error budget.

```
Burn Rate = (Current error rate) / (Error budget allocation rate)

For 99.9% SLO (0.1% budget) over 30 days:
  Budget allocation rate = 0.1% / 30 days = 0.0033%/day

  If current error rate is 0.1%:
  Burn Rate = 0.1% / 0.0033% = 30x

  This means: at the current rate, the monthly budget is exhausted in 1 day.
```

### Burn Rate Alerting

Alert at multiple burn rate thresholds:

```
┌─────────────┬─────────────┬─────────────────────────────────────┐
│  Burn Rate  │  Time to    │  Action                             │
│  Multiplier │  Exhaustion│                                     │
├─────────────┼─────────────┼─────────────────────────────────────┤
│  14.4x      │  2 hours   │  Page on-call (critical)             │
│  6x         │  5 hours   │  Page on-call (urgent)               │
│  3x         │  10 hours  │  Ticket + Slack alert                │
│  1x         │  30 days   │  Report in weekly review             │
└─────────────┴─────────────┴─────────────────────────────────────┘
```

## Implementing Burn Rate Alerts (Prometheus)

```yaml
# SLO: 99.9% availability over 30 days
# Error budget = 0.001 (0.1%)

groups:
  - name: loan-advisor-slo-alerts
    rules:
      # Fast burn: 14.4x over 5 minutes
      - alert: HighErrorBudgetBurnRate_Fast
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{service="loan-advisor", status=~"2..|3..|4.."}[5m]))
              /
              sum(rate(http_requests_total{service="loan-advisor"}[5m]))
            )
          ) / 0.001 > 14.4
        for: 5m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "Error budget burning at 14.4x normal rate"
          description: "At this rate, the monthly error budget will be exhausted in 2 hours"

      # Slow burn: 3x over 30 minutes
      - alert: HighErrorBudgetBurnRate_Slow
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{service="loan-advisor", status=~"2..|3..|4.."}[30m]))
              /
              sum(rate(http_requests_total{service="loan-advisor"}[30m]))
            )
          ) / 0.001 > 3
        for: 30m
        labels:
          severity: warning
          slo: availability
        annotations:
          summary: "Error budget burning at 3x normal rate"
          description: "At this rate, the monthly error budget will be exhausted in 10 hours"
```

## SLO Reporting

### Weekly SLO Report Template

```
## SLO Report: Loan Advisor API
## Week of 2025-03-10 to 2025-03-16

| SLO | Target | Actual | Status | Budget Remaining |
|-----|--------|--------|--------|-----------------|
| Availability | 99.9% | 99.95% | PASS | 82% |
| Latency (p95 < 3s) | 99.0% | 98.7% | FAIL | 0% |
| Response Quality | 95.0% | 96.2% | PASS | 68% |
| Compliance Logging | 100% | 100% | PASS | 100% |

### Key Events This Week
- 2025-03-12: Vector DB upgrade caused 45 minutes of elevated latency
- 2025-03-14: LLM provider outage caused 12 minutes of errors
- 2025-03-15: Deployed v2.4.1 with improved caching

### Error Budget Status
- Latency SLO error budget exhausted. Team will focus on latency improvements next sprint.
- Availability SLO budget healthy at 82%.
```

## SLO Governance

### Who Sets SLOs?

SLOs are a collaboration between:
- **Service team**: Proposes realistic targets based on system capabilities
- **Users/customers**: Defines what reliability they need
- **Business stakeholders**: Communicates regulatory and contractual requirements
- **SRE team**: Ensures targets are measurable and alertable

### SLO Review Cadence

- **Weekly**: SLO status reviewed in team meetings
- **Monthly**: Error budget status reported to engineering leadership
- **Quarterly**: SLO targets reviewed and adjusted if needed
- **Annually**: SLO targets reviewed against SLA requirements and customer feedback

### When to Change an SLO

Change an SLO when:
- User needs have demonstrably changed
- The architecture has fundamentally changed
- Regulatory requirements have changed
- The SLO has been consistently met with >50% budget remaining for 3+ months (consider tightening)
- The SLO is consistently missed despite focused effort (investigate root cause first)

Do NOT change an SLO because:
- The team wants to ship features and the budget is exhausted
- A recent incident caused a breach
- It is "too hard" to meet the current target
