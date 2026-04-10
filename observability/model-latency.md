# Model Latency Monitoring for Banking GenAI

## Understanding Model Latency

Model latency in a GenAI system has multiple components. Understanding the breakdown is essential for optimization:

```
Total Latency = Prompt Preparation + Embedding + Retrieval + Reranking
              + API Call (network) + Model Inference (provider)
              + Response Parsing + Streaming Overhead
```

In a typical banking RAG system:

```
┌─────────────────────────────────────────────────────────────┐
│  LATENCY BREAKDOWN (typical RAG mortgage query)             │
│                                                             │
│  Prompt Preparation:    12ms   (1%)                         │
│  Embedding Generation:  45ms   (2%)                         │
│  Vector DB Search:     123ms   (5%)                         │
│  Reranking:             67ms   (3%)                         │
│  API Network Time:     156ms   (7%)                         │
│  Model Inference:    1,890ms  (82%)  <-- LLM provider time  │
│  Response Parsing:     12ms   (1%)                         │
│  ─────────────────────────────────                          │
│  Total:              2,305ms  (100%)                        │
│                                                             │
│  Key insight: 82% of time is in model inference.            │
│  Optimizing vector DB saves 123ms.                          │
│  Optimizing the LLM call could save 1,890ms.                │
└─────────────────────────────────────────────────────────────┘
```

## Per-Component Latency Tracking

```python
from prometheus_client import Histogram
from contextlib import contextmanager
import time

# Latency histograms per component
latency_histograms = {
    'prompt_preparation': Histogram(
        'latency_prompt_preparation_seconds',
        'Time to assemble prompt',
        ['model', 'product_line'],
        buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1]
    ),
    'embedding': Histogram(
        'latency_embedding_seconds',
        'Time to generate embedding',
        ['embedding_model'],
        buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25]
    ),
    'retrieval': Histogram(
        'latency_retrieval_seconds',
        'Time for vector DB retrieval',
        ['vector_db', 'collection'],
        buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
    ),
    'reranking': Histogram(
        'latency_reranking_seconds',
        'Time for document reranking',
        ['reranker_model'],
        buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5]
    ),
    'api_call': Histogram(
        'latency_llm_api_call_seconds',
        'Network time for LLM API call',
        ['model', 'provider'],
        buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
    ),
    'model_inference': Histogram(
        'latency_model_inference_seconds',
        'Model processing time (provider-side)',
        ['model'],
        buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0]
    ),
    'response_parsing': Histogram(
        'latency_response_parsing_seconds',
        'Time to parse LLM response',
        ['model'],
        buckets=[0.001, 0.005, 0.01, 0.025, 0.05]
    ),
    'total': Histogram(
        'latency_total_seconds',
        'End-to-end latency',
        ['model', 'product_line', 'endpoint'],
        buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0, 60.0]
    ),
}

@contextmanager
def measure_latency(component: str, **labels):
    """Context manager for measuring component latency."""
    histogram = latency_histograms[component]
    start = time.time()
    try:
        yield
    finally:
        duration = time.time() - start
        histogram.labels(**labels).observe(duration)
```

## Streaming vs Non-Streaming Latency

For streaming responses, two latency metrics matter:

1. **Time to First Token (TTFT)**: When does the user start seeing output?
2. **Total Response Time**: When is the complete response received?

```python
async def measure_streaming_latency(model, endpoint):
    """Measure TTFT and total time for streaming response."""
    ttft = None
    start = time.time()
    token_count = 0

    async for token in stream_llm_response(model, prompt):
        if ttft is None:
            ttft = time.time() - start
            streaming_ttft_seconds.labels(model=model).observe(ttft)

        token_count += 1

    total_time = time.time() - start
    streaming_total_seconds.labels(model=model, endpoint=endpoint).observe(total_time)
    streaming_tokens_per_second.labels(model=model).observe(
        token_count / total_time if total_time > 0 else 0
    )

    return {
        'ttft_ms': ttft * 1000,
        'total_ms': total_time * 1000,
        'tokens': token_count,
        'tokens_per_second': token_count / total_time if total_time > 0 else 0
    }
```

## Latency by Model and Provider

Track latency per model and provider to make informed routing decisions:

```
┌──────────────────────────────────────────────────────────────┐
│  MODEL LATENCY COMPARISON (p50 / p95 / p99)                  │
├────────────────────────┬───────────┬───────────┬─────────────┤
│  Model                 │  p50      │  p95      │  p99        │
├────────────────────────┼───────────┼───────────┼─────────────┤
│  gpt-4-turbo           │  1.8s     │  4.2s     │  8.1s       │
│  gpt-3.5-turbo         │  0.4s     │  1.2s     │  2.8s       │
│  claude-3-sonnet       │  1.5s     │  3.8s     │  7.2s       │
│  claude-3-haiku        │  0.3s     │  0.8s     │  1.5s       │
│  custom-rag-model      │  0.6s     │  1.5s     │  3.2s       │
└────────────────────────┴───────────┴───────────┴─────────────┘
```

## Latency Alerting

```yaml
- alert: ModelLatencyDegradation
  expr: |
    histogram_quantile(0.95, rate(latency_total_seconds_bucket[10m]))
    >
    histogram_quantile(0.95, rate(latency_total_seconds_bucket[1h])) offset 1d * 1.5
  for: 10m
  labels:
    severity: warning
    team: ml-platform
  annotations:
    summary: "Model p95 latency 50% higher than 24 hours ago"
    description: "Current p95: {{ $value }}s"

- alert: ModelLatencyAboveSLO
  expr: |
    histogram_quantile(0.95, rate(latency_total_seconds_bucket{endpoint="/api/v1/chat"}[5m]))
    > 3
  for: 5m
  labels:
    severity: critical
    team: ml-platform
  annotations:
    summary: "Chat endpoint p95 latency above 3s SLO"
    description: "Current p95: {{ $value }}s"
```

## Latency Optimization Levers

When latency is too high, these are the available levers:

```
┌─────────────────────────────────────────────────────────────┐
│  LATENCY OPTIMIZATION LEVERS                                │
├─────────────────────────────┬───────────────┬───────────────┤
│  Lever                      │  Impact       │  Complexity   │
├─────────────────────────────┼───────────────┼───────────────┤
│  Use faster model           │  HIGH         │  LOW          │
│  Enable response caching    │  HIGH         │  LOW          │
│  Reduce context documents   │  MEDIUM       │  LOW          │
│  Optimize prompt template   │  LOW-MEDIUM   │  LOW          │
│  Use streaming              │  MEDIUM (TTFT)│  MEDIUM       │
│  Improve vector DB indexing │  MEDIUM       │  MEDIUM       │
│  Add semantic cache         │  HIGH         │  MEDIUM       │
│  Parallelize retrievals     │  MEDIUM       │  HIGH         │
│  Model distillation         │  HIGH         │  HIGH         │
│  GPU provisioning           │  HIGH         │  HIGH         │
└─────────────────────────────┴───────────────┴───────────────┘
```

## Banking Context: Latency by Product

Different banking products have different latency expectations:

```
┌─────────────────────────────────────────────────────────────┐
│  LATENCY EXPECTATIONS BY BANKING PRODUCT                     │
├─────────────────────────────┬───────────────┬───────────────┤
│  Product                    │  p95 Target   │  Rationale    │
├─────────────────────────────┼───────────────┼───────────────┤
│  Quick balance inquiry      │  < 1s        │  Simple lookup │
│  Chat (general question)    │  < 2s        │  Conversational│
│  Mortgage advice            │  < 5s        │  Complex RAG   │
│  Document analysis          │  < 10s       │  Heavy processing│
│  Investment portfolio review│  < 8s        │  Multi-source  │
│  Fraud report analysis      │  < 15s       │  Deep analysis │
│  Batch report generation    │  < 60s       │  Async, not real-time│
└─────────────────────────────┴───────────────┴───────────────┘
```

## Latency Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  MODEL LATENCY - DETAILED VIEW                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Overall Latency:                                            │
│  p50: 1.8s  p90: 3.2s  p95: 4.1s  p99: 7.8s                │
│  (SLO: p95 < 3s)  [BREACHING]                               │
│                                                              │
│  Latency by Component (p50):                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Prompt prep     [████] 12ms                          │ │
│  │ Embedding       [████████] 45ms                      │ │
│  │ Vector DB       [██████████████████] 123ms            │ │
│  │ Reranking       [██████████] 67ms                     │ │
│  │ API Call        [███████████████████████] 156ms       │ │
│  │ Model inference [████████████████████████████] 1,890ms│ │
│  │ Response parse  [██] 12ms                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Latency Trend (p95, last 24h):                              │
│  5.0s ┤                    ╱╲                               │
│       │              ╱╲  ╱  ╲  ╱╲                          │
│  3.0s ┤─────────────╱──╲╱────╲╱──╲── SLO line               │
│       │    ╱╲     ╱              ╲                          │
│  1.0s ┤───╱──╲───╱────────────────╲───                      │
│       └────────────────────────────────────                  │
│       00:00  06:00  12:00  18:00  00:00                     │
└──────────────────────────────────────────────────────────────┘
```

## Common Latency Mistakes

1. **Only measuring total latency**: Without component breakdown, you do not know where to optimize.

2. **Using average latency**: Average hides tail latency. A p99 of 30 seconds with an average of 2 seconds means 1% of users have a terrible experience.

3. **Not measuring TTFT for streaming**: Users perceive TTFT, not total time. A 10-second response with 500ms TTFT feels responsive. A 3-second response with 2-second TTFT feels slow.

4. **Ignoring provider variability**: LLM providers have variable latency. A model with p50 of 1s but p99 of 10s is less reliable than a model with consistent 2s latency.

5. **No baseline comparison**: "Latency is 4 seconds" means nothing without comparison. Is it higher than yesterday? Than last week? Than the SLO?
