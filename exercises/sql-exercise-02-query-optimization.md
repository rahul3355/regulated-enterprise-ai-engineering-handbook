# SQL Exercise 02: Query Optimization

> Optimize slow queries using EXPLAIN ANALYZE, indexing strategies, and query rewriting — critical for banking data platforms.

## Problem Statement

You are investigating slow queries in the GenAI platform's analytics database. Several dashboard queries are taking 10+ seconds, causing timeouts. Your task is to:

1. Analyze the slow queries using `EXPLAIN ANALYZE`
2. Identify the bottlenecks (sequential scans, nested loops, missing indexes)
3. Design indexes to fix the issues
4. Rewrite queries where indexing alone is insufficient
5. Measure the improvement

**Banking Context:** The GenAI platform generates millions of query records. Analytics dashboards for executives, compliance reports for auditors, and real-time monitoring all depend on fast queries. Slow dashboards mean slow decisions.

## Schema

```sql
CREATE TABLE queries (
    query_id UUID PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    department VARCHAR(100) NOT NULL,
    query_text TEXT NOT NULL,
    model VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    latency_ms INT NOT NULL,
    tokens_total INT NOT NULL,
    cost_cents DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL
);

-- Table has 10 million rows
-- ~500 distinct users, ~20 departments, 5 models
-- Data spans 12 months
```

## Slow Queries to Optimize

### Query 1: Department Monthly Summary
```sql
-- Takes 8.2 seconds
SELECT
    department,
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*) AS total_queries,
    AVG(latency_ms) AS avg_latency,
    SUM(cost_cents) AS total_cost
FROM queries
WHERE created_at >= '2025-06-01'
GROUP BY department, DATE_TRUNC('month', created_at)
ORDER BY month DESC, total_queries DESC;
```

### Query 2: User Activity Lookup
```sql
-- Takes 3.5 seconds
SELECT
    u.user_id,
    u.name,
    COUNT(q.query_id) AS total_queries,
    MAX(q.created_at) AS last_active,
    AVG(q.latency_ms) AS avg_latency
FROM queries q
JOIN users u ON q.user_id = u.user_id
WHERE q.department = 'engineering'
  AND q.created_at >= NOW() - INTERVAL '30 days'
GROUP BY u.user_id, u.name
HAVING COUNT(q.query_id) > 10
ORDER BY total_queries DESC
LIMIT 20;
```

### Query 3: Model Performance Over Time
```sql
-- Takes 12.1 seconds
SELECT
    model,
    DATE(created_at) AS query_date,
    COUNT(*) AS queries,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY latency_ms) AS p50,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) AS p99
FROM queries
WHERE status = 'success'
  AND created_at >= NOW() - INTERVAL '90 days'
GROUP BY model, DATE(created_at)
ORDER BY query_date DESC, model;
```

### Query 4: High-Cost Query Detection
```sql
-- Takes 15.3 seconds
SELECT
    q.query_id,
    q.user_id,
    q.query_text,
    q.cost_cents,
    u.name,
    u.department
FROM queries q
JOIN users u ON q.user_id = u.user_id
WHERE q.cost_cents > (
    SELECT AVG(cost_cents) * 3 FROM queries
    WHERE created_at >= NOW() - INTERVAL '7 days'
)
  AND q.created_at >= NOW() - INTERVAL '7 days'
ORDER BY q.cost_cents DESC
LIMIT 50;
```

## Hints

### Hint 1: Reading EXPLAIN ANALYZE Output

```
EXPLAIN ANALYZE output shows:
- Scan type: Seq Scan (bad) vs. Index Scan (good) vs. Index Only Scan (best)
- Join type: Nested Loop (bad for large tables) vs. Hash Join vs. Merge Join
- Actual time: start..end (look for the largest numbers)
- Rows: estimated vs. actual (big gaps mean bad statistics)
- Buffers: shared hit/read (disk reads are slow)

Key bottleneck indicators:
- "Seq Scan" on a 10M row table → needs index
- "Nested Loop" with large outer table → needs different join strategy
- "actual time=5000..8000" → this is where time is spent
```

### Hint 2: Index Design

```sql
-- Composite index for filtered + grouped queries
CREATE INDEX idx_queries_dept_date ON queries(department, created_at);

-- Covering index (includes all columns needed)
CREATE INDEX idx_queries_covering ON queries(user_id, created_at)
    INCLUDE (latency_ms, cost_cents, status, model);

-- Partial index (only index relevant rows)
CREATE INDEX idx_queries_recent_success ON queries(created_at, model, latency_ms)
    WHERE status = 'success' AND created_at >= NOW() - INTERVAL '90 days';
```

### Hint 3: Query Rewriting

```sql
-- Instead of a correlated subquery, use a CTE or window function
-- Bad (runs subquery for every row):
WHERE cost_cents > (SELECT AVG(cost_cents) * 3 FROM queries WHERE ...)

-- Better (computes average once):
WITH avg_cost AS (
    SELECT AVG(cost_cents) * 3 AS threshold
    FROM queries WHERE created_at >= NOW() - INTERVAL '7 days'
)
SELECT ... FROM queries q, avg_cost a
WHERE q.cost_cents > a.threshold
```

## Solutions

```sql
-- Query 1 Optimization: Add composite index
CREATE INDEX idx_queries_dept_month ON queries(department, created_at)
    INCLUDE (latency_ms, cost_cents);

-- After index: ~0.3 seconds (from 8.2s)
-- EXPLAIN shows: Index Only Scan instead of Seq Scan

-- Query 2 Optimization: Add index and rewrite
CREATE INDEX idx_queries_dept_date_user ON queries(department, created_at, user_id)
    INCLUDE (latency_ms);

-- After index: ~0.2 seconds (from 3.5s)

-- Query 3 Optimization: Partial index for success queries
CREATE INDEX idx_queries_success_recent ON queries(model, created_at, latency_ms)
    WHERE status = 'success' AND created_at >= NOW() - INTERVAL '90 days';

-- Materialized view for daily stats (refreshed hourly)
CREATE MATERIALIZED VIEW mv_daily_model_stats AS
SELECT
    model,
    DATE(created_at) AS query_date,
    COUNT(*) AS queries,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY latency_ms) AS p50,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) AS p99
FROM queries
WHERE status = 'success'
  AND created_at >= NOW() - INTERVAL '90 days'
GROUP BY model, DATE(created_at);

CREATE UNIQUE INDEX ON mv_daily_model_stats(model, query_date);

-- After materialized view: ~0.01 seconds (from 12.1s)

-- Query 4 Optimization: Rewrite subquery as CTE
WITH weekly_avg AS (
    SELECT AVG(cost_cents) * 3 AS threshold
    FROM queries
    WHERE created_at >= NOW() - INTERVAL '7 days'
)
SELECT
    q.query_id,
    q.user_id,
    q.query_text,
    q.cost_cents,
    u.name,
    u.department
FROM queries q
JOIN users u ON q.user_id = u.user_id
CROSS JOIN weekly_avg wa
WHERE q.cost_cents > wa.threshold
  AND q.created_at >= NOW() - INTERVAL '7 days'
ORDER BY q.cost_cents DESC
LIMIT 50;

-- Add index:
CREATE INDEX idx_queries_date_cost ON queries(created_at, cost_cents DESC)
    INCLUDE (user_id, query_text);

-- After rewrite + index: ~0.5 seconds (from 15.3s)
```

## Optimization Checklist

For every slow query:

- [ ] Run `EXPLAIN ANALYZE` to identify bottlenecks
- [ ] Check for sequential scans on large tables → add index
- [ ] Check join types → Nested Loop on large tables needs index on join column
- [ ] Check for redundant computations → use CTEs or materialized views
- [ ] Check if all columns are needed → remove unnecessary SELECT columns
- [ ] Check if date ranges can use partial indexes
- [ ] Check if the query can be pre-computed (materialized view)
- [ ] Re-run `EXPLAIN ANALYZE` after each change to verify improvement

## Extensions

1. **Partitioning:** Partition the `queries` table by month. Measure the improvement for date-range queries.

2. **Connection pooling:** The analytics dashboard opens 50 concurrent connections. Implement PgBouncer and measure the impact.

3. **Read replica:** Move analytics queries to a read replica. Set up streaming replication and measure read/write split performance.

4. **Query plan stability:** Use `pg_hint_plan` to stabilize query plans when statistics change.

5. **Cost-based optimization:** For each optimization, estimate the infrastructure cost savings (fewer CPU hours, less I/O).

## Interview Relevance

Query optimization is tested in data engineering interviews:

| Skill | Why It Matters |
|-------|---------------|
| EXPLAIN ANALYZE | Understanding query execution |
| Index design | Most common performance fix |
| Query rewriting | When indexes aren't enough |
| Materialized views | Pre-computation for expensive queries |
| Partitioning | Scaling large tables |

**Follow-up questions:**
- "When is a sequential scan actually better than an index scan?"
- "What's the difference between a B-tree index and a BRIN index?"
- "How do covering indexes work, and when are they useful?"
- "What causes query plan regression?"
