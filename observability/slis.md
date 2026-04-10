# Service Level Indicators (SLIs) for Banking GenAI

## What Is an SLI?

An SLI (Service Level Indicator) is the actual measured value of a service aspect. It is the raw measurement that you compare against an SLO target.

```
SLI = Measured value of a service aspect

Examples:
  Availability SLI = 99.95% (actual measured availability)
  Latency SLI = p95 response time is 2.1s
  Quality SLI = 96.2% of responses rated acceptable
```

## SLI Selection Framework

Not everything should be an SLI. Good SLIs meet all of these criteria:

1. **User-relevant**: The measurement directly correlates with user experience
2. **Measurable**: You can reliably measure it with existing tooling
3. **Actionable**: If the SLI degrades, the team knows what to do
4. **Stable**: The measurement is not noisy or easily gamed

### SLI Selection Matrix

```
                    User-Relevant
                         │
                    Yes  │  No
                  ┌──────┼──────┐
                  │      │      │
             Measurable│      │ DO NOT │
                  │  A  │  B   │ MEASURE│
                  │GOOD │ OK   │       │
             SLIs │ SLIs│ SLIs │       │
                  │     │      │       │
    ──────────────┼─────SLI────┼───────┼────────
                  │     │      │       │
                  │  C  │  D   │       │
                  │GOOD │ NEED  │       │
                  │ SLIs│ SLIs  │       │
             Not  │     │      │       │
          Measurable│     │      │       │
                  │      │      │       │
                  └──────┴──────┘
                         │
                       Yes  No
                     Actionable
```

**Quadrant A (User-relevant + Measurable + Actionable)**: These are your best SLIs. Use them.

**Quadrant B (Measurable + Actionable but not user-relevant)**: These are internal metrics. Track them as operational metrics, not SLIs.

**Quadrant C (User-relevant + Actionable but not measurable)**: Invest in measurement capability. These often become the most valuable SLIs once measurable.

**Quadrant D (User-relevant only)**: Cannot be an SLI until you can measure and act on it.

## Recommended SLIs for Banking GenAI

### Availability SLIs

**Definition**: Percentage of valid requests that are successfully processed.

```
Availability SLI = (Valid successful requests / Valid total requests) * 100
```

What counts as "valid":
- Exclude requests with authentication failures (user's problem)
- Exclude requests with malformed input (client's problem)
- Include requests that failed due to service issues (your problem)
- Include requests that returned incorrect or hallucinated answers (your problem)

```python
def is_valid_request(response):
    """Determine if a request is valid for SLI calculation."""
    if response.status_code in [401, 403]:
        return False  # Auth failure - user's problem
    if response.status_code == 400:
        return False  # Bad request - client's problem
    # Everything else counts as valid, including:
    # - 500 errors (service problem)
    # - 200 with hallucinated content (quality problem)
    # - Timeouts (reliability problem)
    return True
```

### Latency SLIs

**Definition**: Percentage of valid requests served within a threshold.

```
Latency SLI = (Requests served within threshold / Valid total requests) * 100
```

Recommended thresholds for banking GenAI:

```
┌─────────────────────────────────────────────────────────┐
│  LATENCY SLO THRESHOLDS                                 │
├────────────────────────┬────────────────┬───────────────┤
│  Endpoint              │  Threshold     │  Target       │
├────────────────────────┼────────────────┼───────────────┤
│  Simple chat query     │  < 2s          │  99.0%        │
│  RAG-enhanced query    │  < 5s          │  95.0%        │
│  Document analysis     │  < 10s         │  90.0%        │
│  Batch report gen      │  < 60s         │  99.0%        │
│  Streaming first token │  < 1s          │  99.0%        │
└────────────────────────┴────────────────┴───────────────┘
```

Why different thresholds per endpoint: A simple greeting ("Hello") should be fast. A mortgage analysis requiring document retrieval and LLM reasoning takes longer. Users expect this.

### Quality SLIs

**Definition**: Percentage of AI responses that meet quality standards.

This is the hardest SLI to measure in GenAI systems. Use multiple signals:

```
Quality SLI Composite:
  - Human review score (sampled): > 95% acceptable
  - User satisfaction (thumbs up/down): > 80% positive
  - Factual accuracy (automated check): > 98%
  - Regulatory compliance (automated check): 100%
```

Implementation:

```python
def measure_response_quality(response, user_feedback=None):
    """Composite quality score for SLI measurement."""
    scores = {}

    # Automated factual accuracy check
    if response.claims:
        scores['factual'] = check_claims_against_knowledge_base(
            response.claims, response.sources
        )

    # Regulatory compliance check
    scores['compliance'] = check_compliance_rules(
            response.content, response.product_line
        )

    # User feedback if available
    if user_feedback:
        scores['satisfaction'] = user_feedback.rating

    # Composite: response is "good" if all critical checks pass
    # and at least one quality indicator is positive
    return (
        scores.get('compliance', 0) == 1.0 and
        scores.get('factual', 0) >= 0.98 and
        (user_feedback is None or user_feedback.rating >= 3)
    )
```

### Compliance SLIs

**Definition**: Percentage of interactions that meet regulatory requirements.

```
Compliance SLI = (Compliant interactions / Total interactions) * 100
Target: 100%
```

Compliance checks include:
- Disclaimer shown for financial advice
- PII redacted from logs
- Audit log entry created
- Data retention policy applied
- Model version documented

```python
def check_compliance(interaction: Interaction) -> bool:
    """Check all compliance requirements for an interaction."""
    checks = {
        'disclaimer_shown': interaction.disclaimer_was_displayed,
        'pii_in_logs_clean': audit_log_pii_scan(interaction.log_entries) == 0,
        'audit_log_created': interaction.audit_log_id is not None,
        'data_classified': interaction.data_classification is not None,
        'model_version_logged': interaction.model_version is not None,
    }
    return all(checks.values())
```

## Measuring SLIs

### From Logs

```
# Availability from access logs
sum(status_code >= 200 and status_code < 500) / sum(total_requests)

# Latency from access logs
histogram_quantile(0.95, rate(duration_ms_bucket[5m]))
```

### From Traces

```
# Availability from traces
count(spans where status == OK) / count(all root spans)

# Latency from traces
histogram_quantile(0.95, root_span_duration_seconds)
```

### From Metrics

```
# Availability from counters
sum(rate(http_requests_total{status=~"2..|3.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# Latency from histograms
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

## SLI Reporting Template

```
## SLI Report: RAG Pipeline
## Period: 2025-03-01 to 2025-03-31

### Availability SLI
- Measurement: (successful retrievals / total retrievals) * 100
- Measured: 99.93%
- SLO Target: 99.9%
- Status: PASSING
- Error Budget Remaining: 73%

### Latency SLI (p95 < 2s)
- Measurement: % of retrievals completed within 2s
- Measured: 97.8%
- SLO Target: 99.0%
- Status: FAILING
- Error Budget Remaining: 0%
- Root Cause: Vector DB cold start adds ~500ms for first query after deployment

### Quality SLI (relevance > 0.7)
- Measurement: % of responses with relevance score > 0.7 (human-rated sample)
- Measured: 94.2%
- SLO Target: 95.0%
- Status: FAILING
- Error Budget Remaining: 0%
- Action: Improve embedding model and increase chunk size
```

## Common SLI Mistakes

1. **Measuring the wrong thing**: Server-side availability is not the same as user-perceived availability. If the CDN is down, users cannot reach the service even if the backend is healthy.

2. **Including invalid requests**: Authentication failures and malformed requests dilute your SLI. They are not indicative of your service's reliability.

3. **Using averages for latency**: Average latency is meaningless. Always use percentiles (p50, p95, p99).

4. **No quality SLI**: In GenAI, a response that is fast and available but hallucinates is a failure. Quality SLIs are harder to measure but essential.

5. **SLIs that no one checks**: If an SLI has been passing at 99.999% for 6 months, the target may be too low. If it has been failing for 6 months, it may be unrealistic.

6. **Composite SLIs without weights**: If you combine multiple signals into one SLI, document how they are weighted and what constitutes failure.
