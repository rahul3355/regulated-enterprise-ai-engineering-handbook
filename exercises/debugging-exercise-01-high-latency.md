# Debugging Exercise 01: High Latency

> Debug and resolve high latency in the GenAI chat API — a common production issue in GenAI systems.

## Problem Statement

The GenAI chat API is experiencing elevated latency. The p95 has increased from 1.2s to 4.5s over the past 48 hours. Users are complaining that the assistant is "too slow to be useful."

Investigate and resolve the latency degradation.

## Symptoms

```
Dashboard Observations:
- p50 latency: 1.1s → 1.3s (minor increase)
- p95 latency: 1.2s → 4.5s (major increase)
- p99 latency: 2.0s → 12.0s (critical increase)
- Error rate: Stable at 0.1%
- Throughput: 50 req/min → 120 req/min (2.4x increase)
- CPU usage: 45% → 70%
- Memory usage: Stable at 60%
- Database connections: 45/100 (increased from 20/100)

What's unchanged:
- LLM API response time (stable at 800ms average)
- Network latency to LLM API
- Cache hit rate (30%, unchanged)
- Number of service replicas (4, unchanged)

What's changed recently:
- New feature deployed 3 days ago: "document context expansion"
- User adoption increased 2.4x over past week (viral growth)
- No infrastructure changes
- No database schema changes
```

## Investigation Tools Available

```
# Service logs (last 100 lines)
kubectl logs deployment/genai-chat -n genai --tail=100

# Database query performance
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
WHERE query LIKE '%policy_documents%'
ORDER BY mean_time DESC LIMIT 10;

# Connection pool stats
SELECT state, count(*) FROM pg_stat_activity
WHERE datname = 'genai' GROUP BY state;

# Application profiling
# The service has a /debug/pprof endpoint

# LLM API timing
# Each request logs: model_api_latency_ms

# Vector DB query timing
# Each request logs: retrieval_latency_ms
```

## Hints

### Hint 1: Look at the Recent Change

The "document context expansion" feature was deployed 3 days ago. What could it do?

```python
# The feature code:
def expand_context(documents: list[Document], expansion_factor: int = 2) -> list[Document]:
    """Expand retrieved documents by fetching related documents."""
    expanded = []
    for doc in documents:
        expanded.append(doc)
        # Fetch documents that reference this document
        related = db.query(
            "SELECT * FROM policy_documents "
            "WHERE references @> ARRAY[%s]",  # Array containment query
            doc.doc_id
        ).all()
        expanded.extend(related[:expansion_factor])
    return expanded
```

### Hint 2: Check the Query Plan

```sql
-- The related documents query uses an array containment operator
-- Check if there's an index on the references column
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'policy_documents'
  AND indexdef LIKE '%references%';

-- Result: No index on references column
-- The array containment query is doing a sequential scan
-- For each retrieved document, it scans ALL 2M documents
```

### Hint 3: The Compound Effect

```
Original retrieval: 3 documents per query
After expansion: 3 + (3 × 2) = 9 documents per query

Each expansion requires a sequential scan of 2M rows:
- 9 sequential scans per query × 120 queries/minute
- = 1,080 sequential scans per minute
- = 64,800 sequential scans per hour

Each sequential scan on 2M rows: ~500ms
Total expansion time per query: 9 × 500ms = 4.5s

This explains the p95/p99 increase: the expansion adds
variable latency depending on how many related documents exist.
```

## Resolution

```sql
-- Fix 1: Add a GIN index on the references array
CREATE INDEX idx_policy_docs_references
ON policy_documents USING GIN (references);

-- Result: Query time drops from 500ms to 5ms
-- p95 latency returns to 1.4s

-- Fix 2: Add a limit on expansion
def expand_context(documents, expansion_factor=2):
    expanded = []
    for doc in documents:
        expanded.append(doc)
        related = db.query(...).limit(expansion_factor).all()
        expanded.extend(related)
    return expanded

-- Fix 3: Add circuit breaker for expansion
# If expansion takes > 200ms, skip it and log a warning
```

## Post-Resolution Analysis

```
Root Cause: The document context expansion feature added an
unindexed array containment query that performs sequential scans
on 2M rows. Combined with 2.4x traffic growth, this caused
database connection saturation and high tail latency.

Why it wasn't caught in testing:
1. Staging database has 10K rows (vs. 2M in production)
2. Load testing was done at 50 req/min (vs. current 120)
3. The index was never tested at production data scale

Prevention measures:
1. Staging database must be production-scale (or representative sample)
2. Load testing must include peak traffic projections
3. All new database queries must have EXPLAIN ANALYZE in code review
4. Add latency SLO alerts at p50, p95, and p99
5. Feature flag the expansion feature with automatic rollback on latency breach
```

## Extensions

1. **Reproduce in staging:** Create a script that loads staging with production-scale data and reproduces the latency issue.

2. **Build a latency budget:** Create a breakdown of where time is spent in each request (auth, retrieval, expansion, LLM call, filtering, response). Set a budget for each component.

3. **Add SLO-based alerting:** Configure alerts that fire when latency exceeds SLO, not just when the service is down.

4. **Design a caching strategy:** Cache expanded document sets to avoid repeated database queries for similar queries.

5. **Implement auto-scaling:** Design horizontal pod autoscaling based on latency SLO, not just CPU usage.

## Interview Relevance

Debugging is a core skill tested in interviews:

| Skill | Why It Matters |
|-------|---------------|
| Systematic investigation | Not random guessing, methodical approach |
| Reading metrics | Can you interpret dashboards? |
| Query plan analysis | Database performance debugging |
| Root cause identification | Finding the real cause, not just symptoms |
| Prevention design | Fixing the system, not just the bug |

**Follow-up questions:**
- "How would you have caught this before deployment?"
- "What monitoring would you add to detect this earlier?"
- "How do you decide between fixing vs. rolling back?"
- "What's the difference between p50, p95, and p99, and why does it matter?"
