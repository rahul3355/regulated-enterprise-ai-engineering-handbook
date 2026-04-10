# Metrics in Banking GenAI Systems

## Metric Types

Metrics are numerical measurements aggregated over time. They are the cheapest observability signal to store and the primary input for alerting. Understanding the four metric types is essential for designing an effective metrics strategy.

### Counter

A counter is a cumulative value that only increases. It represents a running total of events. Counters are reset on service restart.

```
# Good counter examples
http_requests_total{method="POST", status="200", endpoint="/api/v1/mortgage-advice"} = 15432
llm_calls_total{model="gpt-4", provider="openai", status="success"} = 8921
token_usage_total{direction="input", model="gpt-4"} = 11234567
errors_total{service="rag-pipeline", type="VectorDBTimeout"} = 23
```

**Use counters when**: You need to count how many times something happened over a time period. Rate of change is typically more useful than the absolute value.

**Banking example**: Track total API calls to the LLM provider. By computing the rate of change (requests per second), you can detect traffic spikes, provider outages, and cost anomalies.

```python
from prometheus_client import Counter

llm_calls = Counter(
    'llm_calls_total',
    'Total LLM API calls made',
    ['model', 'provider', 'status']
)

def call_llm(model, provider, prompt):
    try:
        response = make_api_call(model, prompt)
        llm_calls.labels(model=model, provider=provider, status='success').inc()
        return response
    except Exception:
        llm_calls.labels(model=model, provider=provider, status='error').inc()
        raise
```

### Gauge

A gauge is a value that can go up or down. It represents the current state of something.

```
# Good gauge examples
active_connections{service="auth-service"} = 234
memory_usage_bytes{service="llm-gateway", instance="pod-abc123"} = 2147483648
queue_depth{queue="document-processing"} = 45
model_cache_size_items{cache_type="embedding"} = 1247
vector_db_collection_size{collection="banking-policies"} = 89234
```

**Use gauges when**: You need to know the current value of something that fluctuates.

**Banking example**: Monitor the number of active customer sessions in the GenAI advisory platform. A sudden drop may indicate an authentication service failure. A sudden spike may indicate a bot attack.

```python
from prometheus_client import Gauge

active_sessions = Gauge(
    'active_sessions_count',
    'Number of active customer sessions',
    ['user_tier']
)

def update_session_metrics():
    tiers = get_active_sessions_by_tier()
    for tier, count in tiers.items():
        active_sessions.labels(user_tier=tier).set(count)
```

### Histogram

A histogram samples observations (usually things like request durations or response sizes) and counts them in configurable buckets. It also provides a sum of all observed values and a count of observations.

```
# Good histogram examples
http_request_duration_seconds{method="POST", endpoint="/api/v1/chat", le="0.1"} = 1200
http_request_duration_seconds{method="POST", endpoint="/api/v1/chat", le="0.5"} = 4800
http_request_duration_seconds{method="POST", endpoint="/api/v1/chat", le="1.0"} = 5200
http_request_duration_seconds{method="POST", endpoint="/api/v1/chat", le="+Inf"} = 5234
http_request_duration_seconds_sum{method="POST", endpoint="/api/v1/chat"} = 2845.67
http_request_duration_seconds_count{method="POST", endpoint="/api/v1/chat"} = 5234
```

From this data, you can calculate percentiles:
- p50 latency: ~250ms
- p90 latency: ~800ms
- p99 latency: ~2100ms

**Use histograms when**: You need to understand the distribution of values, not just the average. In banking, p99 latency matters because the slowest requests often represent the most complex (and highest-value) customer interactions.

**Banking example**: Track LLM response time distribution to detect degradation:

```python
from prometheus_client import Histogram

llm_response_time = Histogram(
    'llm_response_time_seconds',
    'Time to receive LLM response',
    ['model', 'endpoint'],
    buckets=[0.1, 0.25, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0]
)

def call_llm_with_metrics(model, endpoint, prompt):
    start = time.time()
    try:
        response = call_model_api(model, prompt)
        llm_response_time.labels(model=model, endpoint=endpoint).observe(
            time.time() - start
        )
        return response
    except Exception:
        llm_response_time.labels(model=model, endpoint=endpoint).observe(
            time.time() - start
        )
        raise
```

### Summary

Similar to a histogram but computes quantiles on the client side. Use summaries when you need accurate quantiles across multiple services (histograms cannot be aggregated across instances).

```python
from prometheus_client import Summary

rag_retrieval_time = Summary(
    'rag_retrieval_time_seconds',
    'Time for RAG retrieval',
    ['vector_db', 'embedding_model']
)
```

**When to use summary vs histogram**: Use histograms for server-side latency percentiles. Use summaries when you need accurate cross-service quantiles. In practice, histograms are preferred 90% of the time.

## Metric Naming Conventions

Follow a consistent naming convention. Inconsistent metric names are the primary cause of dashboard and alert confusion.

### Format

```
<namespace>_<name>_<unit>
```

- **namespace**: The subsystem or domain (`http`, `llm`, `db`, `auth`)
- **name**: What is being measured (`requests`, `response_time`, `errors`)
- **unit**: The unit of measurement (`seconds`, `bytes`, `total`, `ratio`)

### Examples

```
# Counters: _total suffix
http_requests_total
llm_calls_total
errors_total
token_usage_total

# Gauges: _count or _bytes suffix
active_connections_count
memory_usage_bytes
queue_depth
cache_hit_ratio

# Histograms: _seconds or _bytes suffix
http_request_duration_seconds
db_query_duration_seconds
llm_response_time_seconds
```

### Suffix Conventions

| Suffix | Type | Meaning |
|--------|------|---------|
| `_total` | Counter | Cumulative count |
| `_count` | Gauge/Histogram | Number of items |
| `_seconds` | Histogram/Summary | Time duration |
| `_bytes` | Gauge/Histogram | Size in bytes |
| `_ratio` | Gauge | Value between 0 and 1 |
| `_bucket` | Histogram | Histogram bucket (automatic) |
| `_sum` | Histogram/Summary | Sum of observed values |

## Labels (Dimensions)

Labels add cardinality to metrics. They allow you to slice metrics by dimensions like endpoint, method, or user tier.

### Label Best Practices

```
# Good: Low cardinality, meaningful for alerting
http_requests_total{method="POST", status="200", endpoint="/api/v1/chat"}

# Bad: High cardinality, will explode metric storage
http_requests_total{method="POST", status="200", user_id="user-12345", correlation_id="req-abc123"}
```

**Rules for labels**:

1. **Low cardinality**: Each label should have fewer than 100 unique values. User IDs, correlation IDs, and timestamps are NOT valid label values.

2. **Meaningful for operations**: Labels should help you diagnose and alert on operational issues. `user_tier` is useful. `user_id` is not.

3. **Consistent across services**: If you use `status` in one service, use `status` in all services. Not `status_code`, `http_status`, or `result`.

4. **Avoid PII**: Never use customer names, emails, or account numbers as label values.

### Recommended Labels by Metric

```
# HTTP metrics
http_requests_total{method, status, endpoint}
http_request_duration_seconds{method, endpoint}

# LLM metrics
llm_calls_total{model, provider, status}
llm_response_time_seconds{model, endpoint}
token_usage_total{model, direction}  # direction = input/output
llm_cost_usd_total{model, provider}

# Database metrics
db_query_duration_seconds{db, operation, table}
db_connection_pool_available{db}

# Business metrics
customer_queries_total{product_line, user_tier}
satisfaction_score{product_line}  # from thumbs up/down
```

## GenAI-Specific Metrics

### Model Performance

```
# Token usage
token_usage_total{model, direction="input"}    # prompt tokens
token_usage_total{model, direction="output"}   # completion tokens

# Cost tracking
llm_cost_usd_total{model, provider, customer_segment}

# Model quality
model_response_quality_score{model, eval_metric="relevance"}
hallucination_rate_ratio{model, product_line}

# Caching
cache_hits_total{cache_type="llm_response"}
cache_misses_total{cache_type="llm_response"}
cache_hit_ratio{cache_type="llm_response"}
```

### RAG Pipeline

```
# Retrieval
rag_retrieval_time_seconds{vector_db, embedding_model}
documents_retrieved_count{vector_db, collection}
reranking_time_seconds{reranker_model}

# Embedding
embedding_generation_time_seconds{embedding_model}
embedding_dimension_count{embedding_model}
```

### Banking Business Metrics

```
# Customer interaction
customer_advisor_sessions_total{product_line, user_tier, outcome}
advisor_session_duration_seconds{product_line}
compliance_disclaimer_shown_total{product_line}
customer_satisfaction_score{product_line}  # 1-5 from feedback
```

## Metric Storage and Retention

```
┌─────────────────────────────────────────────────────┐
│              Metric Retention Tiers                  │
├──────────────┬────────────────┬─────────────────────┤
│  Raw Data    │  Downsampled   │  Long-term          │
│  15 days     │  90 days       │  2 years            │
│  15s resol.  │  5m resolution │  1h resolution      │
│  (Prometheus)│  (Thanos/Cortex)│  (S3/Object Store)  │
└──────────────┴────────────────┴─────────────────────┘
```

Banking regulatory requirements may mandate longer retention for certain metrics (cost, audit-related). Tag these metrics with `retention_category: regulatory` and configure extended retention.

## Common Mistakes

1. **High cardinality labels**: Using `user_id`, `correlation_id`, or `timestamp` as labels creates millions of time series. Use them in traces and logs, not metrics.

2. **No unit suffix**: `http_request_duration` is ambiguous. Is it seconds, milliseconds, or microseconds? Use `http_request_duration_seconds`.

3. **Inconsistent naming**: One service uses `llm_response_time`, another uses `model_latency`. Agree on names platform-wide.

4. **Missing status labels**: Tracking `http_requests_total` without a `status` label means you cannot distinguish successful from failed requests.

5. **Tracking averages only**: Average latency hides the tail latency experience. Use histograms to understand p95 and p99.

6. **Not tracking cost metrics**: In GenAI systems, token usage directly translates to dollars. Track it as rigorously as you track latency.
