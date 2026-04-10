# Backend Observability for Banking GenAI

## Service-Level Observability

Backend observability focuses on the internal health of services: APIs, databases, message queues, caches, and integrations. While frontend observability captures user experience, backend observability enables engineers to understand why the system is behaving a certain way.

## The RED Method

For every service, monitor the RED metrics:

```
Rate:      Number of requests per second
Errors:    Number of failed requests per second
Duration:  Time each request takes (as a histogram)
```

### Implementation (Prometheus + FastAPI)

```python
from prometheus_client import Histogram, Counter
from fastapi import FastAPI, Request
import time

# Metrics
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['service', 'method', 'endpoint', 'status']
)

http_request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['service', 'method', 'endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start_time = time.time()
    service = "loan-advisor-api"

    response = await call_next()

    duration = time.time() - start_time
    endpoint = request.url.path
    method = request.method
    status = str(response.status_code)

    http_requests_total.labels(
        service=service, method=method,
        endpoint=endpoint, status=status
    ).inc()

    http_request_duration.labels(
        service=service, method=method, endpoint=endpoint
    ).observe(duration)

    return response
```

## Database Observability

### Query Performance

```python
from prometheus_client import Histogram
from sqlalchemy import event
from sqlalchemy.engine import Engine

db_query_duration = Histogram(
    'db_query_duration_seconds',
    'Database query duration',
    ['db', 'operation', 'table'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
)

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    conn.info.setdefault('query_start_time', []).append(time.time())

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - conn.info['query_start_time'].pop(-1)
    # Parse table from statement (simplified)
    table = extract_table_name(statement)
    operation = statement.split()[0].upper()
    db_query_duration.labels(
        db='postgresql', operation=operation, table=table
    ).observe(total)
```

### Connection Pool Health

```python
db_pool_available = Gauge(
    'db_connection_pool_available',
    'Available database connections',
    ['db']
)

db_pool_total = Gauge(
    'db_connection_pool_total',
    'Total database connections',
    ['db']
)

def update_pool_metrics(engine):
    pool = engine.pool
    db_pool_available.labels(db='postgresql').set(pool.checkedin())
    db_pool_total.labels(db='postgresql').set(pool.size())
```

### Slow Query Detection

```python
SLOW_QUERY_THRESHOLD_MS = 100

@event.listens_for(Engine, "after_cursor_execute")
def log_slow_queries(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - conn.info['query_start_time'].pop(-1)
    total_ms = total * 1000

    if total_ms > SLOW_QUERY_THRESHOLD_MS:
        logger.warning(
            "Slow query detected",
            extra={
                "duration_ms": total_ms,
                "operation": statement.split()[0].upper(),
                "table": extract_table_name(statement),
                "parameter_count": len(parameters) if parameters else 0
            }
        )
```

## Message Queue Observability

### Kafka Consumer/Producer Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

kafka_messages_published = Counter(
    'kafka_messages_published_total',
    'Total Kafka messages published',
    ['topic']
)

kafka_messages_consumed = Counter(
    'kafka_messages_consumed_total',
    'Total Kafka messages consumed',
    ['topic', 'consumer_group']
)

kafka_produce_latency = Histogram(
    'kafka_produce_latency_seconds',
    'Time to publish a Kafka message',
    ['topic']
)

kafka_consumer_lag = Gauge(
    'kafka_consumer_lag',
    'Kafka consumer lag (messages behind)',
    ['topic', 'consumer_group', 'partition']
)

kafka_queue_depth = Gauge(
    'kafka_queue_depth',
    'Number of messages waiting in topic',
    ['topic', 'partition']
)
```

### Consumer Lag Alerting

```yaml
- alert: KafkaConsumerLagHigh
  expr: |
    kafka_consumer_lag{topic="document-processing"} > 1000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Kafka consumer lag high on document-processing"
    description: "Consumer group {{ $labels.consumer_group }} is {{ $value }} messages behind on topic {{ $labels.topic }}"
```

## Cache Observability

```python
cache_hits_total = Counter('cache_hits_total', 'Cache hits', ['cache_type'])
cache_misses_total = Counter('cache_misses_total', 'Cache misses', ['cache_type'])
cache_evictions_total = Counter('cache_evictions_total', 'Cache evictions', ['cache_type'])
cache_items = Gauge('cache_items_count', 'Items in cache', ['cache_type'])
cache_memory_bytes = Gauge('cache_memory_usage_bytes', 'Cache memory usage', ['cache_type'])

def record_cache_hit(cache_type: str):
    cache_hits_total.labels(cache_type=cache_type).inc()

def record_cache_miss(cache_type: str):
    cache_misses_total.labels(cache_type=cache_type).inc()

# Cache hit rate
# cache_hits_total / (cache_hits_total + cache_misses_total)
```

### LLM Response Cache

```python
class LLMMetricsCache:
    """Track LLM response cache metrics."""

    def __init__(self):
        self.hits = Counter('llm_cache_hits_total', 'LLM cache hits', ['model'])
        self.misses = Counter('llm_cache_misses_total', 'LLM cache misses', ['model'])
        self.size = Gauge('llm_cache_size_items', 'LLM cache size', ['model'])
        self.memory = Gauge('llm_cache_memory_bytes', 'LLM cache memory', ['model'])

    def hit_rate(self, model: str) -> float:
        # Use rate() for accurate hit rate over time windows
        return self.hits  # Calculated via PromQL
```

PromQL for hit rate:
```
rate(llm_cache_hits_total[5m]) / (rate(llm_cache_hits_total[5m]) + rate(llm_cache_misses_total[5m]))
```

## Resource Utilization

### CPU and Memory

```
# Prometheus automatically collects via kube-state-metrics
container_cpu_usage_seconds_total{service="loan-advisor-api"}
container_memory_usage_bytes{service="loan-advisor-api"}
container_memory_working_set_bytes{service="loan-advisor-api"}

# Alert on memory approaching limit
- alert: ContainerMemoryHigh
  expr: |
    container_memory_working_set_bytes{service="loan-advisor-api"}
    /
    container_spec_memory_limit_bytes{service="loan-advisor-api"}
    > 0.85
  for: 5m
  labels:
    severity: warning
```

### Disk Usage

```
# Node filesystem
node_filesystem_avail_bytes{mountpoint="/data"}
node_filesystem_size_bytes{mountpoint="/data"}

# Alert
- alert: DiskSpaceLow
  expr: |
    node_filesystem_avail_bytes{mountpoint="/data"}
    /
    node_filesystem_size_bytes{mountpoint="/data"}
    < 0.15
  for: 10m
  labels:
    severity: warning
```

## Dependency Health

### External Service Checks

```python
from prometheus_client import Gauge

external_service_healthy = Gauge(
    'external_service_health',
    'External service health status (1=healthy, 0=unhealthy)',
    ['service']
)

async def check_dependencies():
    """Periodically check all external dependencies."""
    checks = {
        'openai-api': check_openai_api(),
        'vector-db': check_vector_db_connection(),
        'redis-cache': check_redis_connection(),
        'postgresql': check_postgres_connection(),
        'core-banking-api': check_core_banking_api(),
    }

    for service, check_coro in checks.items():
        try:
            await check_coro
            external_service_healthy.labels(service=service).set(1)
        except Exception:
            external_service_healthy.labels(service=service).set(0)
            logger.error(f"Dependency check failed: {service}")
```

### Dependency Latency Tracking

```python
# Track latency to each external dependency
dependency_latency = Histogram(
    'dependency_request_duration_seconds',
    'Request duration to external dependencies',
    ['dependency', 'operation'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0]
)

# Track dependency error rates
dependency_errors = Counter(
    'dependency_errors_total',
    'Errors from external dependencies',
    ['dependency', 'error_type']
)
```

## Backend Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  LOAN ADVISOR API - BACKEND HEALTH                           │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  Request/s   │  Error Rate  │  p95 Latency │  Active Conns  │
│  247         │  0.12%       │  2.1s        │  156           │
│  (vs 210 lw) │  (vs 0.08%   │  (vs 1.8s    │  (vs 140 lw)   │
│              │   lw)        │   lw)        │                 │
└──────────────┴──────────────┴──────────────┴─────────────────┘

┌──────────────────────────┬───────────────────────────────────┐
│  Request Rate + Errors   │  Latency Distribution            │
│  [Time series]           │  [Heatmap]                       │
│  200 ┤  ╱╲    ╱╲        │                                  │
│      │ ╱  ╲  ╱  ╲       │  10s ┤                           │
│  100 ┤╱    ╲╱    ╲      │   1s ┤  ████                      │
│      │                  │ 100ms┤  ██████████████              │
│    0 ┼───────────────── │  10ms┤  ████████████████████        │
│      00  04  08  12  16 │       └────────────────────────────│
└──────────────────────────┴───────────────────────────────────┘

┌──────────────────────────┬───────────────────────────────────┐
│  Database Connection Pool│  External Dependencies           │
│  Available: 12/20        │  OpenAI API:     Healthy (45ms)   │
│  [Gauge: 60%]           │  Vector DB:      Healthy (12ms)   │
│                          │  Redis Cache:    Healthy (2ms)    │
│                          │  Core Banking:   Healthy (89ms)   │
└──────────────────────────┴───────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Top 5 Slowest Endpoints (p95)                               │
│  /api/v1/mortgage-advice     4.2s    1,234 req/day           │
│  /api/v1/document-analysis   3.8s    456 req/day             │
│  /api/v1/chat                2.1s    8,901 req/day           │
│  /api/v1/account-summary     0.8s    2,345 req/day           │
│  /api/v1/rates               0.3s    5,678 req/day           │
└──────────────────────────────────────────────────────────────┘
```

## Common Backend Observability Mistakes

1. **No dependency tracking**: If the LLM provider is slow but your service code is fast, you still have a problem. Track external dependency latency separately.

2. **Missing connection pool metrics**: Connection pool exhaustion is a common cause of cascading failures. Monitor pool utilization.

3. **Not tracking queue depth**: Growing queue depth is an early warning of capacity problems. Alert on trends, not just thresholds.

4. **Ignoring resource limits**: A container at 95% memory usage is one traffic spike away from OOM kills. Alert at 85%.

5. **No cache metrics**: If your cache hit rate drops from 40% to 5%, your backend load just increased 8x. Track it.

6. **Dashboard without context**: A CPU metric without context is useless. Include the container's CPU limit and the trend.
