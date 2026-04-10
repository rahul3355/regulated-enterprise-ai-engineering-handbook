# Performance Engineering for Banking GenAI Platforms

## Why Performance Matters

In banking, performance directly impacts revenue and compliance. A payment API that exceeds 2-second P99 latency causes customer complaints and SLA breaches. A GenAI document analysis service that processes 100 documents/minute instead of 1000 creates operational bottlenecks. Performance engineering ensures services meet latency targets under production load.

## Performance Metrics

### Latency Percentiles

```
P50  (median):  50% of requests faster than this
P90:            90% of requests faster than this
P95:            95% of requests faster than this
P99:            99% of requests faster than this (target for SLAs)
P999:           99.9% of requests faster than this (outlier detection)

Banking SLA targets:
- Read APIs:     P99 < 200ms
- Write APIs:    P99 < 1000ms
- AI Inference:  P99 < 10000ms
- Batch Jobs:    Complete within scheduled window
```

### Throughput

```
Requests per second (RPS):  Number of requests handled per second
Transactions per second:    Number of database transactions per second
Tokens per second:          AI model throughput (GenAI specific)
Documents per hour:         Batch processing throughput

Capacity planning:
- Current peak: 5000 RPS
- Expected growth: 20% per year
- Target capacity: 5000 * 1.2^3 = 8640 RPS (3-year planning)
```

## Profiling

### CPU Profiling (Python)

```python
import cProfile
import pstats
import io

def profile_endpoint():
    """Profile a specific endpoint."""
    profiler = cProfile.Profile()
    profiler.enable()

    # Execute the code to profile
    result = expensive_operation()

    profiler.disable()

    # Print stats
    stream = io.StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats('cumulative')
    stats.print_stats(20)  # Top 20 functions

    print(stream.getvalue())
    return result


# Production profiling with pyinstrument
from pyinstrument import Profiler

profiler = Profiler()
profiler.start()

# Code to profile
result = process_payment_batch(payments)

profiler.stop()
print(profiler.output_text(unicode=True, color=True))

# Save HTML report
profiler.write_html('profile_output.html')
```

### Memory Profiling (Python)

```python
import tracemalloc
import gc

def profile_memory():
    """Profile memory usage."""
    tracemalloc.start()

    # Snapshot before
    snapshot_before = tracemalloc.take_snapshot()

    # Code to profile
    result = load_large_dataset()

    # Snapshot after
    snapshot_after = tracemalloc.take_snapshot()

    # Compare
    stats = snapshot_after.compare_to(snapshot_before, 'lineno')
    for stat in stats[:10]:
        print(stat)

    # Find memory leaks
    gc.collect()
    snapshot_gc = tracemalloc.take_snapshot()
    top_stats = snapshot_gc.statistics('lineno')
    for stat in top_stats[:10]:
        print(f'{stat.count} objects: {stat.size / 1024:.1f} KB')


# Track object growth
import objgraph

objgraph.show_growth(limit=10)
# Output shows which object types are growing:
# dict        15234    +1234
# list        8765     +567
# str         45678    +234
```

### CPU Profiling (Go)

```go
import (
    "net/http"
    _ "net/http/pprof"
    "runtime/pprof"
    "os"
)

func main() {
    // Enable pprof endpoints
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // Use: go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
    // Then: go tool pprof -http=:8080 profile.pb.gz

    // CPU profile to file
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    // Run application
    runApp()
}

// Memory profile
func writeMemoryProfile(filename string) {
    f, _ := os.Create(filename)
    defer f.Close()
    runtime.GC() // Force GC to get accurate stats
    pprof.WriteHeapProfile(f)
}
```

## Benchmarking

### Microbenchmarking (Python)

```python
import timeit

# Benchmark single operation
def benchmark_payment_validation():
    payment = {'amount': 100.00, 'currency': 'USD', 'to': 'ACC-001'}
    time = timeit.timeit(
        lambda: validate_payment(payment),
        number=10000,
    )
    print(f'Average: {time / 10000 * 1000:.3f}ms per validation')

# Compare implementations
def benchmark_implementations():
    setup = """
import json
data = {'amount': 100.00, 'currency': 'USD', 'items': [1,2,3,4,5]}
"""

    impl1 = timeit.timeit('json.dumps(data)', setup=setup, number=100000)
    impl2 = timeit.timeit('orjson.dumps(data)', setup=setup, number=100000)

    print(f'json.dumps: {impl1:.3f}s for 100k calls')
    print(f'orjson.dumps: {impl2:.3f}s for 100k calls')
    print(f'Speedup: {impl1/impl2:.1f}x')
```

### Load Testing

```python
# Locust load test configuration
from locust import HttpUser, task, between

class BankingApiUser(HttpUser):
    wait_time = between(1, 3)  # Wait 1-3 seconds between requests

    @task(3)  # 3x more likely
    def get_account(self):
        self.client.get('/v1/accounts/ACC-001')

    @task(2)
    def get_transactions(self):
        self.client.get('/v1/accounts/ACC-001/transactions?limit=50')

    @task(1)
    def create_payment(self):
        self.client.post('/v1/payments', json={
            'to': 'ACC-002',
            'amount': 100.00,
            'currency': 'USD',
        })

# Run: locust -f load_test.py --headless -u 1000 -r 100
# -u: 1000 concurrent users
# -r: 100 users spawned per second
```

## Performance Optimization Techniques

### Database Query Optimization

```python
# N+1 Query Problem
def get_accounts_with_customers_bad(account_ids: list[str]) -> list[dict]:
    """N+1 queries: 1 for accounts + N for customers."""
    accounts = db.query(Account).filter(Account.id.in_(account_ids)).all()
    result = []
    for account in accounts:
        customer = db.query(Customer).get(account.customer_id)  # N queries!
        result.append({**account.to_dict(), 'customer': customer.to_dict()})
    return result

# Solution: Eager loading
def get_accounts_with_customers_good(account_ids: list[str]) -> list[dict]:
    """2 queries total: 1 for accounts + 1 for customers."""
    accounts = db.query(Account).filter(Account.id.in_(account_ids)).all()
    customer_ids = [a.customer_id for a in accounts]
    customers = db.query(Customer).filter(Customer.id.in_(customer_ids)).all()
    customer_map = {c.id: c for c in customers}

    return [
        {**a.to_dict(), 'customer': customer_map[a.customer_id].to_dict()}
        for a in accounts
    ]
```

### Connection Pooling

```python
from sqlalchemy import create_engine

# Connection pool configuration
engine = create_engine(
    'postgresql://user:pass@db:5432/banking',
    pool_size=20,              # Persistent connections
    max_overflow=10,           # Temporary overflow connections
    pool_timeout=30,           # Wait time for connection
    pool_recycle=1800,         # Recycle connections every 30 min
    pool_pre_ping=True,        # Check connection health before use
)

# Monitor pool usage
from sqlalchemy.pool import QueuePool

pool = engine.pool
print(f'Pool size: {pool.size()}')
print(f'Checked out: {pool.checkedout()}')
print(f'Overflow: {pool.overflow()}')
```

### Caching for Performance

```python
# Multi-level caching strategy
class PerformanceOptimizedAccountService:
    def __init__(self):
        self.local_cache = {}      # L1: In-process (microseconds)
        self.redis_cache = redis.Redis()  # L2: Redis (milliseconds)
        self.database = Database()  # L3: Database (tens of milliseconds)

    def get_account(self, account_id: str) -> dict:
        # L1: Local cache
        if account_id in self.local_cache:
            return self.local_cache[account_id]

        # L2: Redis cache
        cached = self.redis_cache.get(f'account:{account_id}')
        if cached:
            account = json.loads(cached)
            self.local_cache[account_id] = account  # Promote to L1
            return account

        # L3: Database
        account = self.database.get_account(account_id)

        # Write through caches
        self.redis_cache.set(f'account:{account_id}', json.dumps(account), ex=300)
        self.local_cache[account_id] = account

        return account
```

### Batch Processing Optimization

```python
# Process in batches instead of one at a time
def process_payments_batch(payments: list[dict]) -> list[dict]:
    """Process payments in bulk with single database transaction."""
    if not payments:
        return []

    # Batch insert
    batch_size = 1000
    results = []

    for i in range(0, len(payments), batch_size):
        batch = payments[i:i + batch_size]

        # Single bulk insert instead of N individual inserts
        db.execute(
            """
            INSERT INTO payments (account_id, amount, currency, status, created_at)
            VALUES (:account_id, :amount, :currency, 'pending', NOW())
            """,
            batch,
        )

        results.extend([
            {'payment_id': generate_id(), 'status': 'pending'}
            for _ in batch
        ])

    return results
```

## GenAI Performance

### Model Inference Optimization

```python
# Batch inference for better GPU utilization
class BatchInferenceOptimizer:
    def __init__(self, model, batch_size: int = 32):
        self.model = model
        self.batch_size = batch_size
        self.queue = []
        self.lock = threading.Lock()

    def add_request(self, prompt: str) -> asyncio.Future:
        """Add request to batch queue."""
        future = asyncio.get_event_loop().create_future()
        with self.lock:
            self.queue.append((prompt, future))

            # If batch is full, process it
            if len(self.queue) >= self.batch_size:
                asyncio.create_task(self.process_batch())

        return future

    async def process_batch(self):
        """Process accumulated requests as a batch."""
        with self.lock:
            batch = self.queue[:self.batch_size]
            self.queue = self.queue[self.batch_size:]

        if not batch:
            return

        prompts = [p for p, _ in batch]
        futures = [f for _, f in batch]

        # Batch inference (much faster than individual)
        results = await self.model.generate_batch(prompts)

        for future, result in zip(futures, results):
            future.set_result(result)
```

### Embedding Caching

```python
# Cache embeddings to avoid recomputation
class EmbeddingCache:
    def __init__(self, redis_client, model_version: str):
        self.redis = redis_client
        self.model_version = model_version

    def get_or_compute(self, text: str) -> list[float]:
        """Get embedding from cache or compute."""
        import hashlib
        key = f'emb:{self.model_version}:{hashlib.sha256(text.encode()).hexdigest()}'

        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Compute embedding
        embedding = compute_embedding(text)

        # Cache for 30 days
        self.redis.set(key, json.dumps(embedding), ex=86400 * 30)

        return embedding
```

## Performance Budgets

```
API Performance Budgets:

| Endpoint                    | P50    | P95    | P99    |
|-----------------------------|--------|--------|--------|
| GET /accounts/{id}          | 10ms   | 50ms   | 100ms  |
| GET /accounts/{id}/txns     | 20ms   | 100ms  | 200ms  |
| POST /payments              | 50ms   | 200ms  | 500ms  |
| POST /ai/chat               | 500ms  | 2000ms | 5000ms |
| POST /ai/analyze            | 1000ms | 5000ms | 10000ms|

Database Performance Budgets:
- Simple SELECT: < 5ms
- Indexed lookup: < 10ms
- Complex aggregation: < 100ms
- Bulk insert (1000 rows): < 500ms

Cache Performance Budgets:
- Local cache hit: < 1ms
- Redis cache hit: < 5ms
- Cache miss penalty: Database query time + 5ms
```

## Common Mistakes

1. **No baseline measurements**: Optimizing without profiling first
2. **Premature optimization**: Optimizing code that isn't a bottleneck
3. **Ignoring P99**: Only looking at average latency
4. **Not testing under load**: Performance degrades non-linearly at scale
5. **Connection pool exhaustion**: Not sizing pools for peak load
6. **N+1 queries**: Hidden performance killer in ORMs
7. **No capacity planning**: Surprised when traffic doubles
8. **Ignoring GC pauses**: Garbage collection causing latency spikes

## Interview Questions

1. How do you identify the bottleneck in an API endpoint with P99 of 5 seconds?
2. Design a load testing strategy for a payment API expecting 10k RPS.
3. How would you reduce the latency of a GenAI inference from 10s to 2s?
4. Explain how connection pool sizing affects performance under load.
5. How do you detect and fix N+1 query problems in production?

## Cross-References

- [[caching/README.md]] - Caching for performance optimization
- [[concurrency/README.md]] - Concurrency for throughput
- [[backend-testing/README.md]] - Performance testing strategies
- `../python/performance.md` - Python-specific optimization
- `../go/performance.md` - Go-specific profiling and optimization
