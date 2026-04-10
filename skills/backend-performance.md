# Skill: Backend Performance

## Core Principles

1. **Profile Before You Optimize** — The bottleneck is never where you think it is. Use profilers, APM traces, and query plans to find the actual bottleneck, then fix it.
2. **The 80/20 Rule of Performance** — 80% of the latency comes from 20% of the code. Find the hot paths and optimize those first. Everything else is distraction.
3. **Caching Is the Biggest Win** — A cached result is always faster than a computed one. But caching introduces complexity (staleness, invalidation). Cache selectively, with clear TTL strategies.
4. **Database Is Usually the Bottleneck** — Slow queries, missing indexes, N+1 queries, and connection pool exhaustion cause more performance issues than application code.
5. **Concurrency Is Free, Until It Isn't** — Async/await and parallelism improve throughput, but only up to the point where you exhaust database connections, file descriptors, or memory.

## Mental Models

### The Backend Performance Hierarchy
```
┌─────────────────────────────────────────────────────────────┐
│            Backend Performance Optimization Order            │
│                                                             │
│  Level 1: Eliminate Unnecessary Work (10-100x improvement)  │
│  ├── Remove N+1 queries                                     │
│  ├── Remove redundant API calls                             │
│  ├── Remove unused data from responses                      │
│  └── Remove synchronous calls that can be async             │
│                                                             │
│  Level 2: Optimize the Hot Path (2-10x improvement)         │
│  ├── Add database indexes                                   │
│  ├── Optimize slow queries (EXPLAIN ANALYZE)                │
│  ├── Cache frequently accessed data                         │
│  └── Batch database operations                              │
│                                                             │
│  Level 3: Scale Out (2-5x improvement)                      │
│  ├── Add read replicas for read-heavy workloads             │
│  ├── Use connection pooling (PgBouncer)                     │
│  ├── Add horizontal scaling (more pods)                     │
│  └── Use CDN for static content                             │
│                                                             │
│  Level 4: Optimize Code (1.5-3x improvement)               │
│  ├── Optimize algorithms (O(n²) → O(n log n))               │
│  ├── Reduce memory allocations                              │
│  ├── Use faster serialization (orjson vs json)              │
│  └── Use compiled extensions for hot paths (Cython, Rust)   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Latency Budget for a GenAI Request
```
┌─────────────────────────────────────────────────────────────┐
│         Latency Budget: GenAI Chat Request (Target: < 3s)   │
│                                                             │
│  API Gateway overhead:          10ms                        │
│  Authentication:                20ms                        │
│  Input validation:              5ms                         │
│  Guardrail checks:              100ms                       │
│  Cache lookup (Redis):          5ms                         │
│  Document retrieval (Vector DB): 200ms                      │
│  Prompt assembly:               10ms                        │
│  LLM API call:                  1500ms                      │
│  Output guardrails:             100ms                       │
│  Response serialization:        5ms                         │
│  Audit logging (async):         20ms                        │
│                                                             │
│  Total: ~1975ms (leaving 1s buffer for network variance)    │
│                                                             │
│  If any component exceeds its budget, the user sees lag.    │
│  Monitor each component independently.                      │
└─────────────────────────────────────────────────────────────┘
```

### The Backend Performance Checklist
```
□ Database queries use indexes (EXPLAIN ANALYZE)
□ No N+1 query patterns (use JOINs or batch queries)
□ Connection pool properly configured (PgBouncer)
□ Slow queries logged and alertable (> 100ms)
□ Caching strategy defined (what, where, TTL, invalidation)
□ API responses paginated (never return unlimited results)
□ Response compression enabled (gzip/brotli)
□ Async processing for non-critical work (Celery, background tasks)
□ Rate limiting prevents abuse
□ Timeout set on all downstream calls
□ Memory usage monitored (no memory leaks)
□ CPU usage monitored (no hot loops)
□ Garbage collection impact measured (for JVM/Go)
□ Request size limited (prevent large payload DoS)
□ Response streaming for large responses (don't buffer in memory)
```

## Step-by-Step Approach

### 1. Identify and Fix N+1 Queries

```python
# BAD: N+1 query pattern — one query for accounts, then N queries for transactions
async def get_accounts_with_transactions(user_id: str) -> list[dict]:
    accounts = await db.fetch_all(
        "SELECT id, account_number, balance FROM accounts WHERE customer_id = :user_id",
        {"user_id": user_id}
    )

    result = []
    for account in accounts:
        # This runs once per account — N additional queries!
        transactions = await db.fetch_all(
            "SELECT id, amount, description FROM transactions WHERE account_id = :account_id ORDER BY created_at DESC LIMIT 10",
            {"account_id": account["id"]}
        )
        result.append({
            **account,
            "recent_transactions": transactions,
        })
    return result
    # For 50 accounts: 1 + 50 = 51 queries

# GOOD: Single query with JOIN
async def get_accounts_with_transactions_optimized(user_id: str) -> list[dict]:
    rows = await db.fetch_all("""
        SELECT
            a.id as account_id,
            a.account_number,
            a.balance,
            t.id as transaction_id,
            t.amount,
            t.description,
            t.created_at as transaction_date
        FROM accounts a
        LEFT JOIN LATERAL (
            SELECT id, amount, description, created_at
            FROM transactions
            WHERE account_id = a.id
            ORDER BY created_at DESC
            LIMIT 10
        ) t ON true
        WHERE a.customer_id = :user_id
        ORDER BY a.id, t.created_at DESC
    """, {"user_id": user_id})

    # Group results in application code
    accounts = {}
    for row in rows:
        if row["account_id"] not in accounts:
            accounts[row["account_id"]] = {
                "id": row["account_id"],
                "account_number": row["account_number"],
                "balance": row["balance"],
                "recent_transactions": [],
            }
        if row["transaction_id"]:
            accounts[row["account_id"]]["recent_transactions"].append({
                "id": row["transaction_id"],
                "amount": row["amount"],
                "description": row["description"],
                "created_at": row["transaction_date"],
            })

    return list(accounts.values())
    # 1 query regardless of number of accounts

# GOOD: Using batched IN query (simpler but less elegant)
async def get_transactions_batched(account_ids: list[str]) -> dict:
    """Fetch all transactions for multiple accounts in one query."""
    rows = await db.fetch_all("""
        SELECT account_id, id, amount, description, created_at
        FROM transactions
        WHERE account_id = ANY(:account_ids)
        ORDER BY account_id, created_at DESC
        LIMIT 500
    """, {"account_ids": account_ids})

    # Group by account_id
    result = defaultdict(list)
    for row in rows:
        result[row["account_id"]].append({
            "id": row["id"],
            "amount": row["amount"],
            "description": row["description"],
            "created_at": row["created_at"],
        })
    return dict(result)
```

### 2. Optimize Slow Queries with EXPLAIN ANALYZE

```sql
-- Identify slow queries from the slow query log
-- In Postgres: log_min_duration_statement = 100 (log queries > 100ms)

-- Analyze the slow query
EXPLAIN ANALYZE
SELECT d.id, d.title, d.content
FROM documents d
WHERE d.department = 'risk-management'
  AND d.classification = 'confidential'
  AND d.embedding <=> '[0.1, 0.2, ...]' < 0.3
ORDER BY d.created_at DESC
LIMIT 20;

-- Typical output showing the problem:
-- Limit (cost=1234.56..1234.61 rows=20) (actual time=450.123..450.145 rows=20)
--   ->  Sort (cost=1234.56..1240.56 rows=2400) (actual time=450.120..450.130 rows=20)
--         Sort Key: created_at DESC
--         Sort Method: top-N heapsort  Memory: 35kB
--         ->  Seq Scan on documents d  (cost=0.00..1180.00 rows=2400) (actual time=2.345..440.567 rows=2400)
--               Filter: (department = 'risk-management' AND classification = 'confidential')
--               Rows Removed by Filter: 47600
--
-- Problem: Sequential scan scanning 50,000 rows, filtering to 2,400
-- Solution: Add a composite index

-- Create a partial index (only for relevant rows)
CREATE INDEX idx_documents_dept_classification
  ON documents(department, classification)
  WHERE classification IN ('confidential', 'restricted');

-- Re-run EXPLAIN ANALYZE — should now use the index:
-- Limit (cost=0.43..8.45 rows=20) (actual time=0.123..0.145 rows=20)
--   ->  Index Scan using idx_documents_dept_classification on documents d
--       Index Cond: (department = 'risk-management' AND classification = 'confidential')

-- For vector similarity searches, ensure the IVFFlat or HNSW index exists
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Check index usage statistics
SELECT
    schemaname,
    relname as table_name,
    indexrelname as index_name,
    idx_scan as index_scans,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;  -- Low scan count = candidate for removal
```

### 3. Connection Pool Management with PgBouncer

```ini
# pgbouncer.ini — Production PgBouncer configuration
[databases]
genai_platform = host=pg-primary port=5432 dbname=genai_platform

[pgbouncer]
; Use transaction pooling (best for most web applications)
pool_mode = transaction

; Connection limits
max_client_conn = 1000
default_pool_size = 25

; Timeouts
server_lifetime = 3600
server_idle_timeout = 600
server_connect_timeout = 15
server_login_retry = 15

; Query timeout
query_timeout = 30
query_wait_timeout = 120

; Logging and monitoring
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 10

; Security
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; Listen address
listen_addr = 0.0.0.0
listen_port = 6432
admin_users = pgbouncer_admin
```

```python
# Application connection string points to PgBouncer, not directly to Postgres
DATABASE_URL = "postgresql://user:password@pgbouncer.genai-db.svc:6432/genai_platform"

# Monitoring PgBouncer
# Connect to the PgBouncer admin database
# psql -h pgbouncer -p 6432 -U admin pgbouncer

# SHOW pools; — Current pool status
# SHOW stats; — Query statistics
# SHOW clients; — Connected clients
# SHOW servers; — Backend server connections
```

### 4. Caching Strategy with Redis

```python
import json
import hashlib
from redis.asyncio import Redis
from functools import wraps

redis = Redis.from_url("redis://redis-cache.genai-platform.svc:6379/0")

def cache_response(ttl: int = 300):
    """
    Decorator that caches API response in Redis.
    Cache key includes user ID (for personalized responses).
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Build cache key from function name + arguments
            request = kwargs.get("request")
            user_id = request.user.id if request else "anonymous"

            cache_key = f"api:{func.__name__}:{user_id}:{hashlib.md5(
                json.dumps(kwargs, default=str).encode()
            ).hexdigest()}"

            # Try cache hit
            cached = await redis.get(cache_key)
            if cached:
                return json.loads(cached)

            # Cache miss — compute and store
            result = await func(*args, **kwargs)
            await redis.setex(cache_key, ttl, json.dumps(result, default=str))
            return result
        return wrapper
    return decorator

# Usage
@cache_response(ttl=600)  # Cache for 10 minutes
async def get_policy_documents(user_id: str, department: str) -> list[dict]:
    """This function's result is cached per user + department."""
    return await db.fetch_policy_documents(user_id, department)

# Cache invalidation
async def invalidate_user_cache(user_id: str) -> None:
    """Invalidate all cached data for a user."""
    pattern = f"api:*:{user_id}:*"
    async for key in redis.scan_iter(match=pattern):
        await redis.delete(key)

# Cache warming: pre-populate cache for commonly accessed data
async def warm_cache() -> None:
    """Pre-populate cache for top departments."""
    departments = ["risk-management", "compliance", "hr", "legal"]
    for dept in departments:
        docs = await db.fetch_policy_documents_raw(dept)
        cache_key = f"api:get_policy_documents:system:{hashlib.md5(dept.encode()).hexdigest()}"
        await redis.setex(cache_key, 3600, json.dumps(docs))  # Cache for 1 hour
```

### 5. Async Processing for Non-Critical Work

```python
from celery import Celery

celery_app = Celery(
    "genai_platform",
    broker="redis://redis-cache.genai-platform.svc:6379/1",
    backend="redis://redis-cache.genai-platform.svc:6379/2",
)

@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def embed_document(self, document_id: str) -> dict:
    """
    Async task to embed a document for RAG retrieval.
    This is not on the critical path — the user doesn't wait for it.
    """
    try:
        document = db.get_document(document_id)
        embedding = openai.embeddings.create(
            model="text-embedding-3-small",
            input=document["content"],
        )
        db.update_document_embedding(document_id, embedding.data[0].embedding)
        return {"status": "success", "document_id": document_id}
    except Exception as exc:
        # Retry on failure
        raise self.retry(exc=exc)

# Trigger from the API endpoint (don't wait for the result)
@app.post("/api/documents")
async def upload_document(request: DocumentUploadRequest):
    document = await db.create_document(request)
    # Trigger embedding asynchronously
    embed_document.delay(document["id"])
    # Return immediately — embedding happens in background
    return {"id": document["id"], "status": "processing", "embedding": "pending"}
```

### 6. Response Streaming for Large Payloads

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

@app.get("/api/documents/{doc_id}/export")
async def export_document(doc_id: str):
    """
    Stream large document exports instead of loading into memory.
    This prevents memory issues for documents > 10MB.
    """
    async def document_stream():
        async for chunk in db.stream_document(doc_id, chunk_size=8192):
            yield chunk

    return StreamingResponse(
        document_stream(),
        media_type="application/octet-stream",
        headers={
            "Content-Disposition": f'attachment; filename="{doc_id}.pdf"',
            "X-Content-Type-Options": "nosniff",
        },
    )

# Streaming GenAI responses (token-by-token)
@app.post("/api/chat/stream")
async def chat_stream(request: ChatRequest):
    async def generate():
        async for token in llm_client.stream_chat(request.messages):
            yield f"data: {json.dumps({'token': token})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        },
    )
```

### 7. Performance Profiling

```python
# Python profiling with pyinstrument
from pyinstrument import Profiler

@app.middleware("http")
async def profile_slow_requests(request: Request, call_next):
    """Profile requests that take longer than 500ms."""
    import time
    start = time.time()

    response = await call_next(request)

    duration = (time.time() - start) * 1000
    if duration > 500:
        # Log slow request
        logger.warning(
            "Slow request: method=%s path=%s duration=%.0fms",
            request.method, request.url.path, duration,
        )
        # Add header for debugging (only in dev/staging)
        if os.getenv("ENV") != "production":
            response.headers["X-Request-Duration-Ms"] = str(int(duration))

    return response

# Manual profiling for specific endpoints
@app.get("/api/debug/profile")
async def profile_endpoint():
    profiler = Profiler()
    profiler.start()

    # Run the code you want to profile
    result = await expensive_operation()

    profiler.stop()
    return HTMLResponse(profiler.output_html())
```

```go
// Go profiling with pprof
import (
    _ "net/http/pprof"
    "runtime/pprof"
)

func main() {
    // Enable pprof endpoints
    // Access at http://localhost:8080/debug/pprof/
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // Application code...
}

// Profile specific functions
func BenchmarkRetrieveDocuments(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _, err := retriever.Search(context.Background(), "test query", 10)
        if err != nil {
            b.Fatal(err)
        }
    }
}
// Run: go test -bench=. -benchmem -cpuprofile=cpu.prof
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| N+1 queries | O(n) database calls, linear latency growth | Use JOINs or batch queries |
| No database indexes | Full table scans, slow queries grow with data | EXPLAIN ANALYZE, add indexes |
| Missing connection pool limits | Connection exhaustion, cascading failures | Use PgBouncer, set pool sizes |
| Synchronous calls for non-critical work | Increased latency, poor throughput | Use async task queues (Celery) |
| Not setting timeouts on downstream calls | Requests hang indefinitely | Set timeout on every external call |
| Returning unlimited results | Memory exhaustion, slow responses | Always paginate with LIMIT |
| Caching without invalidation strategy | Stale data, hard-to-debug inconsistencies | Define TTL and invalidation upfront |
| Buffering large responses in memory | OOM kills for large payloads | Use streaming responses |
| Not monitoring query latency | Slow queries go unnoticed | Log and alert on queries > 100ms |
| Premature optimization | Wasted time on non-bottlenecks | Profile first, optimize the hot path |

## Banking-Specific Concerns

1. **Regulatory Query Performance** — Compliance queries (e.g., "show all transactions for customer X in the last 7 years") can be extremely expensive. Pre-compute compliance reports on a schedule rather than running ad-hoc queries.
2. **Audit Log Write Performance** — Audit logs are write-heavy and append-only. Use partitioned tables, batch inserts, and consider a separate database for audit data to avoid impacting the primary database.
3. **Data Residency Query Routing** — In multi-region banking deployments, queries must route to the correct regional database. Misrouted queries add cross-region latency and may violate data residency requirements.
4. **Peak Load Patterns** — Banking systems have predictable peak loads (market open, payroll days, month-end close). Capacity plan for peaks, not averages.
5. **Batch Processing Windows** — End-of-day batch processing (interest calculations, reconciliation) must complete within a time window. Monitor batch job duration and alert if approaching the window limit.

## GenAI-Specific Concerns

1. **LLM API Latency** — The LLM API call is typically the slowest component (1-5 seconds). Mitigate with response caching for common queries, streaming for perceived speed, and fallback responses for timeout scenarios.
2. **Embedding Pipeline Throughput** — Document embedding is CPU/GPU intensive. Use async processing with a queue (Celery, RabbitMQ) and monitor queue depth. Scale embedding workers independently.
3. **Prompt Token Count** — Longer prompts cost more and take longer to process. Optimize prompt templates to remove unnecessary context. Use retrieved document snippets instead of full documents.
4. **Conversation Context Size** — Long conversations accumulate token context. Implement context window management: summarize older messages, truncate, or use a separate retrieval step.
5. **Vector Search Performance** — IVFFlat index accuracy depends on the number of lists. HNSW is more accurate but uses more memory. Choose based on the accuracy/performance trade-off for your use case.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| API response time (p99) | > 3s | User-perceived slowness |
| Database query time (p99) | > 100ms | Query performance degradation |
| Cache hit rate | < 80% | Caching strategy ineffective |
| Connection pool utilization | > 80% | Approaching connection exhaustion |
| N+1 query count | > 0 in hot paths | Unnecessary database calls |
| Queue depth (async tasks) | > 1000 | Async processing backlog |
| Memory usage | > 80% of limit | Approaching OOM |
| CPU usage | > 70% sustained | Compute bottleneck |
| LLM API latency (p99) | > 5s | Model API degradation |
| Embedding queue wait time | > 5 minutes | Embedding pipeline backlog |
| Error rate | > 1% | Service degradation |
| Throughput (requests/sec) | Track trend | Capacity planning |

## Interview Questions

1. How do you identify and fix N+1 query problems?
2. A database query that used to take 10ms now takes 500ms. How do you debug it?
3. What is the difference between connection pooling and connection multiplexing?
4. How would you design a caching strategy for a GenAI RAG system?
5. When would you use async processing vs. synchronous processing?
6. How do you handle a slow endpoint that can't be optimized further?
7. Explain how you would profile a slow Python API endpoint.
8. What strategies would you use to reduce the latency of a GenAI chat response?

## Hands-On Exercise

### Exercise: Optimize a Slow RAG Retrieval API

**Problem:** The `/api/search` endpoint that retrieves documents for the GenAI assistant is slow. At peak load (500 requests/second), the p99 latency is 2.5 seconds (target: < 500ms). The endpoint:

1. Validates the search query
2. Generates an embedding for the query (calls external embedding API)
3. Searches the vector database for similar documents
4. Fetches full document content from Postgres
5. Checks access control for each document
6. Returns the top 10 results

**Constraints:**
- You cannot change the embedding model or API
- You cannot reduce the number of results returned
- Access control checks must remain (regulatory requirement)
- The database has 500,000 documents and growing
- The embedding API has a p99 latency of 120ms

**Expected Output:**
- EXPLAIN ANALYZE output showing the slow queries
- Index recommendations with SQL statements
- Caching strategy (what to cache, where, TTL, invalidation)
- Code changes to eliminate N+1 queries
- Connection pool configuration
- Performance budget for each component
- Expected improvement estimate

**Hints:**
- The embedding API call is a good candidate for caching (same query → same embedding)
- The access control check runs per document — can it be pushed down to the database query?
- The vector search may not be using the right index
- Document content fetching may be an N+1 pattern

**Extension:**
- Implement a read replica strategy for the document database
- Design a pre-computation pipeline that caches popular search results
- Add a performance regression test that fails the CI build if p99 exceeds 500ms
- Implement response streaming for the search results

---

**Related files:**
- `backend-engineering/api-performance.md` — API performance guide
- `databases/postgres-performance.md` — Postgres query optimization
- `databases/caching-strategies.md` — Redis caching patterns
- `skills/frontend-performance.md` — Frontend performance
- `observability/apm.md` — Application performance monitoring
- `genai-platforms/rag-optimization.md` — RAG performance
