# Synthetic Monitoring for Banking GenAI

## What Is Synthetic Monitoring?

Synthetic monitoring proactively tests your system by simulating user interactions. Unlike real-user monitoring (which observes actual customer traffic), synthetic monitoring runs on a schedule to verify availability and functionality even when real traffic is low.

```
Real-user monitoring:  "Are customers experiencing problems?"
Synthetic monitoring:  "Would a customer experience problems if they tried now?"
```

In banking, synthetic monitoring is critical because:
- Some services have low overnight traffic but must remain available
- Regulatory systems must be verified 24/7
- External provider degradations must be detected before customers notice

## Types of Synthetic Checks

### 1. HTTP Health Checks

The simplest form: send an HTTP request and verify the response.

```python
import httpx
from datetime import datetime

async def check_api_health():
    """Basic health check for the loan advisor API."""
    async with httpx.AsyncClient(timeout=10) as client:
        response = await client.get(
            "https://api.bank.example/health",
            headers={"User-Agent": "synthetic-monitor/1.0"}
        )

    assert response.status_code == 200
    body = response.json()
    assert body["status"] == "healthy"
    assert body["version"] is not None
    assert response.elapsed.total_seconds() < 1.0
```

**Where to run from**: Multiple geographic locations, including inside your banking network and from external vantage points.

### 2. Multi-Step Transaction Checks

Simulate a complete user flow:

```python
async def synthetic_mortgage_query():
    """Full user journey simulation."""

    # Step 1: Authenticate
    auth_response = await client.post(
        "https://sso.bank.example/oauth/token",
        json={"grant_type": "client_credentials", ...}
    )
    assert auth_response.status_code == 200
    token = auth_response.json()["access_token"]

    # Step 2: Query mortgage advice
    advice_response = await client.post(
        "https://api.bank.example/api/v1/mortgage-advice",
        json={
            "salary": 95000,
            "loan_amount": 450000,
            "term_years": 30
        },
        headers={"Authorization": f"Bearer {token}"}
    )
    assert advice_response.status_code == 200

    # Step 3: Verify response quality
    advice = advice_response.json()
    assert "recommendation" in advice
    assert "disclaimer" in advice  # Compliance requirement
    assert len(advice["recommendation"]) > 50  # Not a truncated response

    # Step 4: Verify audit trail
    assert response.headers.get("X-Audit-Id") is not None
```

### 3. LLM Provider Availability Checks

```python
async def check_llm_provider():
    """Verify LLM provider is responsive and returning valid responses."""

    prompt = "Respond with exactly the word 'OK' and nothing else."

    response = await call_llm(
        model="gpt-3.5-turbo",
        prompt=prompt,
        temperature=0,
        max_tokens=10
    )

    # Verify response quality
    text = response.text.strip().lower()
    assert "ok" in text

    # Verify latency
    assert response.latency_ms < 5000

    # Verify token usage is reasonable for this simple prompt
    assert response.tokens_prompt < 100
    assert response.tokens_completion < 20
```

### 4. RAG Pipeline Verification

```python
async def check_rag_pipeline():
    """Verify the complete RAG pipeline returns relevant documents."""

    # Use a known query with a known answer
    query = "What is the maximum loan-to-value ratio for a conventional mortgage?"

    response = await rag_pipeline.retrieve_and_answer(query)

    # Check that documents were retrieved
    assert len(response.source_documents) >= 3

    # Check that the answer contains expected terms
    answer_lower = response.answer.lower()
    assert any(term in answer_lower for term in ["80%", "loan-to-value", "ltv"])

    # Check latency
    assert response.total_latency_ms < 8000

    # Check compliance
    assert response.disclaimer_present
```

## Check Frequency

```
┌─────────────────────────────────────────────────────────────┐
│  SYNTHETIC CHECK FREQUENCY                                  │
├─────────────────────────────┬───────────────┬───────────────┤
│  Check Type                 │  Frequency    │  Timeout      │
├─────────────────────────────┼───────────────┼───────────────┤
│  HTTP health check          │  Every 1 min  │  5 seconds    │
│  Multi-step transaction     │  Every 5 min  │  30 seconds   │
│  LLM provider check         │  Every 2 min  │  15 seconds   │
│  RAG pipeline verification  │  Every 10 min │  30 seconds   │
│  Full user journey          │  Every 15 min │  60 seconds   │
│  Compliance check           │  Every 5 min  │  15 seconds   │
└─────────────────────────────┴───────────────┴───────────────┘
```

## Geographic Distribution

Run synthetic checks from multiple locations:

```
┌─────────────────────────────────────────────────────────────┐
│              Synthetic Check Locations                       │
│                                                             │
│     ┌──────┐                            ┌──────┐           │
│     │  US  │                            │  EU  │           │
│     │East  │────── Internet ────────────│West  │           │
│     └──┬───┘                            └──┬───┘           │
│        │                                    │               │
│        └────────── Banking Network ─────────┘               │
│                            │                                │
│                     ┌──────┴──────┐                         │
│                     │  Internal   │                         │
│                     │  Health     │                         │
│                     │  Checks     │                         │
│                     └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

Internal checks verify service-to-service connectivity. External checks verify customer-facing availability.

## Blackbox Monitoring

Blackbox monitoring tests the system from outside, with no internal knowledge:

```python
# Blackbox: Only knows the public API contract
async def blackbox_api_check():
    """Test the API without any internal access."""
    response = await client.get("https://api.bank.example/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"
    # That is all a blackbox check knows
```

Contrast with whitebox monitoring:

```python
# Whitebox: Knows internal state
async def whitebox_api_check():
    """Test the API with internal access."""
    response = await client.get("https://api.bank.example/health")
    body = response.json()
    assert body["status"] == "healthy"
    assert body["db_connection_pool_available"] > 0
    assert body["vector_db_latency_ms"] < 200
    assert body["llm_provider_status"] == "connected"
    # Checks internal component health
```

Both are valuable. Blackbox catches what users experience. Whitebox catches what is about to break.

## Alerting on Synthetic Failures

```yaml
# Prometheus rules for synthetic monitoring
groups:
  - name: synthetic-monitoring
    rules:
      - alert: SyntheticHealthCheckFailing
        expr: |
          synthetic_check_success{check="api-health"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "API health check failing from {{ $external_labels.location }}"
          description: "Synthetic health check to {{ $labels.endpoint }} is failing. Users may be affected."

      - alert: SyntheticLLMCheckHighLatency
        expr: |
          synthetic_check_duration_seconds{check="llm-provider"} > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "LLM provider response time > 10s"
          description: "Synthetic LLM check from {{ $external_labels.location }} taking {{ $value }}s"

      - alert: SyntheticRAGCheckFailing
        expr: |
          synthetic_check_success{check="rag-pipeline"} == 0
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "RAG pipeline synthetic check failing"
          description: "Full RAG pipeline check failing. Document retrieval may be broken."
```

## Implementation with K6

```javascript
// k6 synthetic test
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend } from 'k6/metrics';

const apiLatency = new Trend('api_latency_ms');

export const options = {
  stages: [
    { duration: '30s', target: 1 },  // Ramp up
    { duration: '5m', target: 1 },   // Steady state
    { duration: '30s', target: 0 },  // Ramp down
  ],
  thresholds: {
    'api_latency_ms': ['p(95)<3000'],
    'http_req_duration': ['p(95)<5000'],
    'http_req_failed': ['rate<0.01'],
  },
};

export default function () {
  // Health check
  const healthRes = http.get('https://api.bank.example/health');
  check(healthRes, {
    'health status is 200': (r) => r.status === 200,
    'health response < 1s': (r) => r.timings.duration < 1000,
  });
  apiLatency.add(healthRes.timings.duration);

  // Auth + query flow
  const tokenRes = http.post('https://sso.bank.example/oauth/token', {
    grant_type: 'client_credentials',
    client_id: __ENV.SYNTHETIC_CLIENT_ID,
    client_secret: __ENV.SYNTHETIC_CLIENT_SECRET,
  });

  const token = tokenRes.json('access_token');

  const queryRes = http.post('https://api.bank.example/api/v1/chat', {
    query: 'What are your mortgage rates?',
  }, {
    headers: { Authorization: `Bearer ${token}` },
  });

  check(queryRes, {
    'query status is 200': (r) => r.status === 200,
    'response has recommendation': (r) => r.json('recommendation') !== null,
    'response has disclaimer': (r) => r.json('disclaimer') !== null,
    'query response < 5s': (r) => r.timings.duration < 5000,
  });

  sleep(60);  // Wait 60 seconds between iterations
}
```

## Common Mistakes

1. **Synthetic checks that always pass**: If your synthetic test uses a trivial query that never exercises the critical path, it provides false confidence.

2. **No authentication in synthetic tests**: If your production traffic requires auth but your synthetic check does not, you are not testing the real path.

3. **Only testing from one location**: A service may be healthy from your datacenter but unreachable from customer locations.

4. **Not alerting on synthetic failures**: A synthetic check that fails without alerting is worse than no check at all -- it creates false confidence.

5. **Too-frequent expensive checks**: Running a full RAG pipeline check every 30 seconds wastes tokens and costs money. Balance frequency with cost.

6. **Hard-coded expected responses**: If your synthetic check expects an exact response string, it will fail when the LLM rephrases. Check for semantic correctness, not exact matches.
