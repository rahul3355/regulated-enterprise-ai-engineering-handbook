# GenAI Observability for Banking Systems

## Why GenAI Needs Specialized Observability

Traditional observability covers latency, errors, and traffic. But a GenAI system can have perfect traditional metrics while delivering terrible user experiences:

- The API responds in 200ms, but the answer is wrong
- Error rate is 0%, but the model is hallucinating
- All services are healthy, but the response contains biased advice

GenAI observability extends traditional observability with model-specific, quality-specific, and cost-specific metrics.

## GenAI Observability Framework

```
┌─────────────────────────────────────────────────────────────┐
│              GenAI Observability Layers                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LAYER 5: Business Impact                           │   │
│  │  Customer satisfaction, conversion, compliance      │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LAYER 4: Output Quality                            │   │
│  │  Hallucination rate, factual accuracy, relevance    │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LAYER 3: Model Performance                         │   │
│  │  Token usage, cost, latency by model component      │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LAYER 2: Pipeline Performance                      │   │
│  │  RAG retrieval quality, embedding latency,          │   │
│  │  vector DB performance, cache hit rate              │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LAYER 1: Infrastructure (traditional)              │   │
│  │  CPU, memory, network, disk, container health       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Model Performance Metrics

### Token Usage

```python
from prometheus_client import Counter, Histogram

# Token counts
token_usage_total = Counter(
    'token_usage_total',
    'Total tokens consumed',
    ['model', 'direction', 'product_line']
    # direction = input (prompt) or output (completion)
)

# Token usage distribution
tokens_per_request = Histogram(
    'tokens_per_request',
    'Tokens per request',
    ['model', 'direction'],
    buckets=[10, 50, 100, 250, 500, 1000, 2000, 5000, 10000]
)

def record_token_usage(model, direction, count, product_line=None):
    token_usage_total.labels(
        model=model, direction=direction, product_line=product_line
    ).inc(count)
    tokens_per_request.labels(model=model, direction=direction).observe(count)
```

### Cost Tracking

```python
# Cost tracking per model
MODEL_COSTS = {
    'gpt-4-turbo': {'input': 0.01 / 1000, 'output': 0.03 / 1000},
    'gpt-3.5-turbo': {'input': 0.0005 / 1000, 'output': 0.0015 / 1000},
    'claude-3-sonnet': {'input': 0.003 / 1000, 'output': 0.015 / 1000},
    'claude-3-haiku': {'input': 0.00025 / 1000, 'output': 0.00125 / 1000},
}

llm_cost_usd = Counter(
    'llm_cost_usd_total',
    'Total LLM API cost in USD',
    ['model', 'provider', 'product_line']
)

def calculate_and_record_cost(model, prompt_tokens, completion_tokens, product_line=None):
    costs = MODEL_COSTS.get(model, {'input': 0, 'output': 0})
    cost = (prompt_tokens * costs['input']) + (completion_tokens * costs['output'])

    provider = get_provider_for_model(model)
    llm_cost_usd.labels(
        model=model, provider=provider, product_line=product_line
    ).inc(cost)

    return cost
```

### Model Latency Breakdown

```python
# Break down latency by component
model_latency_breakdown = Histogram(
    'model_latency_component_seconds',
    'Model latency broken down by component',
    ['model', 'component'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0]
)

# Components:
# - prompt_preparation: assembling prompt from template + context
# - embedding: generating query embedding
# - retrieval: vector DB search + reranking
# - api_call: network time to LLM provider
# - model_inference: actual model processing (provider-side)
# - response_parsing: parsing and validating response
# - total: end-to-end
```

## Output Quality Metrics

### Hallucination Detection

```python
hallucination_rate = Counter(
    'hallucination_detected_total',
    'Detected hallucinations',
    ['model', 'product_line', 'severity']
    # severity = critical (wrong financial advice), warning, info
)

def check_for_hallucination(response, source_documents) -> bool:
    """Check if response contains claims not supported by sources."""
    claims = extract_claims(response)
    supported_claims = verify_claims_against_sources(claims, source_documents)

    unsupported = [c for c in claims if c not in supported_claims]

    if unsupported:
        severity = assess_hallucination_severity(unsupported)
        hallucination_rate.labels(
            model=response.model,
            product_line=response.product_line,
            severity=severity
        ).inc()
        return True
    return False
```

### Response Quality Scoring

```python
quality_score_distribution = Histogram(
    'response_quality_score',
    'Distribution of response quality scores',
    ['model', 'eval_metric'],
    buckets=[i/10 for i in range(11)]  # 0.0 to 1.0
)

def evaluate_response_quality(response, ground_truth=None):
    """Evaluate response quality using multiple metrics."""
    scores = {}

    # Relevance: how well does the response address the query?
    scores['relevance'] = compute_relevance_score(
        response.query, response.content
    )

    # Faithfulness: are claims supported by retrieved documents?
    scores['faithfulness'] = compute_faithfulness_score(
        response.content, response.source_documents
    )

    # Context precision: were the retrieved documents relevant?
    scores['context_precision'] = compute_context_precision(
        response.query, response.source_documents
    )

    # Answer relevance: does the answer match the question?
    scores['answer_relevance'] = compute_answer_relevance(
        response.query, response.content
    )

    # Record all scores
    for metric, score in scores.items():
        quality_score_distribution.labels(
            model=response.model, eval_metric=metric
        ).observe(score)

    return scores
```

## RAG Pipeline Metrics

### Retrieval Quality

```python
# How many documents were retrieved?
documents_retrieved = Histogram(
    'documents_retrieved_per_query',
    'Number of documents retrieved per query',
    ['vector_db', 'collection'],
    buckets=[1, 2, 3, 5, 8, 10, 15, 20, 30]
)

# Similarity score distribution
similarity_scores = Histogram(
    'retrieval_similarity_score',
    'Similarity scores of retrieved documents',
    ['vector_db'],
    buckets=[i/100 for i in range(101)]  # 0.00 to 1.00
)

# Reranking effectiveness
reranking_impact = Histogram(
    'reranking_position_change',
    'How much reranking changes document positions',
    ['reranker_model']
)
```

### Embedding Metrics

```python
# Embedding generation latency
embedding_latency = Histogram(
    'embedding_generation_seconds',
    'Time to generate query embedding',
    ['embedding_model'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5]
)

# Embedding dimension count (for monitoring model changes)
embedding_dimensions = Gauge(
    'embedding_dimension_count',
    'Number of dimensions in embedding',
    ['embedding_model']
)
```

## GenAI Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  GenAI PLATFORM - QUALITY & PERFORMANCE                      │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  Avg Quality │  Hallucination│  Cost/Query  │  Cache Hit     │
│  Score       │  Rate         │              │  Rate          │
│              │              │              │                │
│  0.92/1.0    │  2.1%         │  $0.034      │  23%           │
│  (target:    │  (target:    │  (target:    │  (target:      │
│   > 0.85)    │   < 5%)      │   < $0.05)   │   > 20%)       │
└──────────────┴──────────────┴──────────────┴─────────────────┘

┌──────────────────────────┬───────────────────────────────────┐
│  Token Usage by Model    │  Cost Trend (daily)              │
│  gpt-4:    4.2M tokens   │  $450 ┤    ╱╲  ╱╲               │
│  gpt-3.5:  6.8M tokens   │       │   ╱  ╲╱  ╲  ╱╲          │
│  claude-3: 1.5M tokens   │  $300 ┤  ╱       ╲╱  ╲          │
│                           │       │ ╱         ╲             │
│                           │  $150 ┤╱           ╲            │
└──────────────────────────┴───────┴──────────────────────────┘

┌──────────────────────────┬───────────────────────────────────┐
│  Quality Scores          │  Latency Breakdown               │
│  Relevance:     0.94     │  Prompt prep:   12ms (1%)        │
│  Faithfulness:  0.89     │  Embedding:     45ms (2%)        │
│  Context prec:  0.87     │  Retrieval:    234ms (10%)       │
│  Answer relev:  0.91     │  LLM inference: 1,890ms (82%)    │
│                          │  Response parse: 112ms (5%)       │
└──────────────────────────┴───────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Hallucination Incidents (last 7 days)                       │
│  Date       | Severity | Product Line | Category             │
│  2025-03-12 | CRITICAL | Mortgage     | Incorrect rate quote  │
│  2025-03-13 | WARNING  | Investment   | Outdated fund data    │
│  2025-03-14 | WARNING  | Credit Card  | Missing fee info      │
└──────────────────────────────────────────────────────────────┘
```

## GenAI-Specific Alerting

```yaml
- alert: HighHallucinationRate
  expr: |
    rate(hallucination_detected_total{severity="critical"}[15m])
    /
    rate(llm_calls_total[15m])
    > 0.05
  for: 5m
  labels:
    severity: critical
    team: ml-platform
  annotations:
    summary: "Critical hallucination rate > 5%"
    description: "{{ $value | humanizePercentage }} of responses contain critical hallucinations"

- alert: LLMSpikeInCost
  expr: |
    sum(rate(llm_cost_usd_total[1h]))
    >
    sum(rate(llm_cost_usd_total[1h])) offset 24h * 2
  for: 15m
  labels:
    severity: warning
    team: ml-platform
  annotations:
    summary: "LLM cost doubled compared to 24 hours ago"

- alert: QualityScoreDegradation
  expr: |
    histogram_quantile(0.5, rate(response_quality_score[1h]))
    < 0.80
  for: 30m
  labels:
    severity: warning
    team: ml-platform
  annotations:
    summary: "Median quality score below 0.80"
```

## Common GenAI Observability Mistakes

1. **Only tracking infrastructure metrics**: A healthy LLM provider with degraded model quality is invisible to traditional monitoring.

2. **No hallucination tracking**: Hallucinations are the GenAI equivalent of data corruption. Track them with the same rigor.

3. **Ignoring token costs**: Without per-query cost tracking, you cannot detect cost anomalies or optimize spending.

4. **No quality SLOs**: If you do not define and measure quality targets, you cannot tell if model quality is degrading.

5. **Not tracking prompt-to-response chain**: When quality is poor, you need to know if the issue was the prompt, the retrieved context, or the model itself.

6. **No feedback loop**: User feedback (thumbs up/down) is the most valuable quality signal. Capture it and feed it into observability.
