# Query Optimization for Banking Data Pipelines

## Overview

Query optimization is the skill that separates junior and senior data engineers. In banking environments handling millions of daily transactions, a poorly optimized query can cause:
- Pipeline timeouts and SLA breaches
- Table locks affecting production systems
- Excessive resource consumption driving up infrastructure costs
- Downstream delays for fraud detection and compliance reporting

This guide covers EXPLAIN ANALYZE, index usage, query plans, and practical optimization strategies for production banking workloads.

## Understanding EXPLAIN and EXPLAIN ANALYZE

`EXPLAIN` shows the query planner's chosen execution strategy without running the query. `EXPLAIN ANALYZE` actually executes the query and provides real timing statistics.

```sql
-- Basic EXPLAIN: Shows planned execution
EXPLAIN
SELECT c.customer_id, SUM(t.amount) AS total_spend
FROM customers c
JOIN transactions t ON c.customer_id = t.customer_id
WHERE t.transaction_time >= '2025-01-01'
GROUP BY c.customer_id;

-- EXPLAIN ANALYZE: Shows actual execution stats
EXPLAIN ANALYZE
SELECT c.customer_id, SUM(t.amount) AS total_spend
FROM customers c
JOIN transactions t ON c.customer_id = t.customer_id
WHERE t.transaction_time >= '2025-01-01'
GROUP BY c.customer_id;

-- EXPLAIN with buffers: Shows I/O statistics
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT c.customer_id, SUM(t.amount) AS total_spend
FROM customers c
JOIN transactions t ON c.customer_id = t.customer_id
WHERE t.transaction_time >= '2025-01-01'
GROUP BY c.customer_id;

-- Verbose output with additional details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT * FROM transactions 
WHERE account_id = 12345 
ORDER BY transaction_time DESC 
LIMIT 100;
```

### Reading an EXPLAIN ANALYZE Output

```
EXPLAIN ANALYZE
SELECT * FROM transactions 
WHERE account_id = 12345 AND transaction_time >= '2025-01-01';

                                                QUERY PLAN                                                  
------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on transactions  (cost=45.32..1234.56 rows=5234 width=128) 
   (actual time=2.345..15.678 rows=4892 loops=1)
   Recheck Cond: (account_id = 12345)
   Filter: (transaction_time >= '2025-01-01'::date)
   Rows Removed by Filter: 234
   Heap Blocks: exact=1456
   Buffers: shared hit=234 read=1222
   ->  Bitmap Index Scan on idx_transactions_account_id  
       (cost=0.00..44.01 rows=5500 width=0) 
       (actual time=1.234..1.234 rows=5126 loops=1)
         Index Cond: (account_id = 12345)
         Buffers: shared hit=45 read=12
 Planning Time: 0.456 ms
 Execution Time: 16.234 ms
```

**Key Metrics to Read**:
- **cost**: Planner's estimated cost (start..end). First number is startup cost, second is total cost.
- **rows**: Estimated rows the planner expects.
- **actual time**: Real execution time in milliseconds (startup..total).
- **rows (actual)**: Real number of rows returned.
- **loops**: How many times this node was executed.
- **Buffers**: `hit` = from cache, `read` = from disk. High disk reads indicate missing indexes or cold cache.
- **Rows Removed by Filter**: Rows read but discarded. High numbers indicate selective filtering after scan.

## Common Access Methods

### Sequential Scan

```
Seq Scan on transactions  (cost=0.00..150234.56 rows=1234 width=128)
  (actual time=1.234..567.890 rows=1234 loops=1)
  Filter: (account_id = 12345)
  Rows Removed by Filter: 49998766
```

**Problem**: Scanning 50M rows to find 1,234 matching rows. This is extremely inefficient.

**Solution**: Add an index on the filter column.

```sql
-- Create index to avoid sequential scan
CREATE INDEX idx_transactions_account_id 
ON transactions (account_id);

-- For range queries, include the time column
CREATE INDEX idx_transactions_account_time 
ON transactions (account_id, transaction_time);
```

### Index Scan

```
Index Scan using idx_transactions_account_time on transactions  
  (cost=0.56..234.56 rows=1234 width=128)
  (actual time=0.045..2.345 rows=1234 loops=1)
  Index Cond: (account_id = 12345 AND transaction_time >= '2025-01-01')
```

**Good**: Direct index lookup. Very efficient for point queries and narrow ranges.

### Bitmap Index Scan

```
Bitmap Heap Scan on transactions  (cost=45.32..1234.56 rows=5234 width=128)
  ->  Bitmap Index Scan on idx_transactions_account_id  
      (cost=0.00..44.01 rows=5500 width=0)
```

**Good**: Efficient for moderate selectivity queries. The bitmap collects matching row locations, then fetches heap pages in order.

### Index Only Scan

```
Index Only Scan using idx_transactions_covering on transactions  
  (cost=0.56..123.45 rows=500 width=32)
  (actual time=0.023..0.567 rows=500 loops=1)
  Index Cond: (account_id = 12345)
  Heap Fetches: 0
```

**Excellent**: All needed columns are in the index. No heap fetch required. Achieved with covering indexes.

```sql
-- Covering index: includes all columns needed by the query
CREATE INDEX idx_transactions_covering 
ON transactions (account_id, transaction_time) 
INCLUDE (amount, transaction_type, currency);
```

## Join Strategies

### Nested Loop Join

```
Nested Loop  (cost=0.56..567.89 rows=100 width=256)
  (actual time=0.056..12.345 rows=100 loops=1)
  ->  Index Scan using idx_customers_pkey on customers  
      (cost=0.28..8.29 rows=1 width=128)
      Index Cond: (customer_id = 100)
  ->  Index Scan using idx_transactions_customer on transactions  
      (cost=0.28..550.00 rows=100 width=128)
      Index Cond: (customer_id = 100)
```

**When Used**: Small outer table with indexed inner table. Efficient for small result sets.

**Problem**: O(n*m) complexity. Devastating when both tables are large.

### Hash Join

```
Hash Join  (cost=1234.56..5678.90 rows=10000 width=256)
  (actual time=23.456..89.012 rows=10234 loops=1)
  Hash Cond: (t.customer_id = c.customer_id)
  ->  Seq Scan on transactions t  (cost=0.00..3456.78 rows=100000 width=128)
  ->  Hash  (cost=1000.00..1000.00 rows=50000 width=128)
        (actual time=12.345..12.345 rows=50000 loops=1)
        Buckets: 65536  Batches: 1  Memory Usage: 5120kB
        ->  Seq Scan on customers c  (cost=0.00..1000.00 rows=50000 width=128)
```

**When Used**: Large tables without selective filters. Builds hash table on smaller input.

**Optimization**: Increase `work_mem` to allow larger hash tables in memory.

```sql
-- Increase work_mem for this session
SET work_mem = '256MB';

-- Or at server level (postgresql.conf)
-- work_mem = 256MB
```

### Merge Join

```
Merge Join  (cost=0.56..3456.78 rows=50000 width=256)
  (actual time=0.056..45.678 rows=50000 loops=1)
  Merge Cond: (t.customer_id = c.customer_id)
  ->  Index Scan using idx_transactions_customer on transactions t
  ->  Index Scan using idx_customers_pkey on customers c
```

**When Used**: Both inputs sorted on join keys. Very efficient for sorted data or when indexes provide ordering.

## Aggregation Strategies

### HashAggregate

```
HashAggregate  (cost=5678.90..5789.01 rows=1000 width=128)
  (actual time=34.567..34.890 rows=1000 loops=1)
  Group Key: customer_id
  Batches: 1  Memory Usage: 512kB
  ->  Seq Scan on transactions
```

**When Used**: Grouping on unsorted data. Builds hash table for groups.

### GroupAggregate

```
GroupAggregate  (cost=0.56..6789.01 rows=1000 width=128)
  (actual time=0.056..56.789 rows=1000 loops=1)
  Group Key: customer_id
  ->  Index Scan using idx_transactions_customer_time on transactions
```

**When Used**: Input is already sorted by group key. No hash table needed.

## Query Optimization Patterns

### Avoiding SELECT *

```sql
-- Bad: Fetches all columns, prevents index-only scans
SELECT * FROM transactions 
WHERE account_id = 12345;

-- Good: Only needed columns, enables index-only scans
SELECT transaction_id, amount, transaction_time, transaction_type
FROM transactions 
WHERE account_id = 12345;
```

### Filtering Before Joining

```sql
-- Bad: Joins full tables, then filters
SELECT c.customer_id, t.amount
FROM customers c
JOIN transactions t ON c.customer_id = t.customer_id
WHERE t.transaction_time >= '2025-01-01'
  AND c.customer_segment = 'PREMIUM';

-- Good: Filter before join, reduce data early
WITH premium_customers AS (
    SELECT customer_id FROM customers 
    WHERE customer_segment = 'PREMIUM'
),
recent_transactions AS (
    SELECT customer_id, amount, transaction_time 
    FROM transactions 
    WHERE transaction_time >= '2025-01-01'
)
SELECT pc.customer_id, rt.amount
FROM premium_customers pc
JOIN recent_transactions rt USING (customer_id);
```

### Optimizing LIKE Queries

```sql
-- Bad: Leading wildcard prevents index usage
SELECT * FROM customers WHERE email LIKE '%@gmail.com';

-- Good: Use trigram index for pattern matching
CREATE INDEX idx_customers_email_trgm 
ON customers USING gin (email gin_trgm_ops);

-- Good: Use reversed column for suffix matching
-- Create a reversed index
CREATE INDEX idx_customers_email_reversed 
ON customers (reverse(email));
-- Query: WHERE reverse(email) LIKE reverse('%@gmail.com')

-- Best: Use generated column for domain extraction
ALTER TABLE customers ADD COLUMN email_domain 
    GENERATED ALWAYS AS (split_part(email, '@', 2)) STORED;
CREATE INDEX idx_customers_email_domain 
ON customers (email_domain);
-- Query: WHERE email_domain = 'gmail.com'
```

### Optimizing ORDER BY with LIMIT

```sql
-- Bad: Sorts all rows, then takes 10
EXPLAIN ANALYZE
SELECT * FROM transactions 
WHERE transaction_time >= '2025-01-01'
ORDER BY amount DESC
LIMIT 10;

-- Good: Index supports both filter and sort
CREATE INDEX idx_transactions_time_amount 
ON transactions (transaction_time, amount DESC);

-- Now the query uses index for filter AND sort
```

## Banking-Specific Optimization

### High-Volume Transaction Queries

```sql
-- Problem: Daily reconciliation query on 50M row table
-- Original: 45 seconds
EXPLAIN ANALYZE
SELECT 
    a.account_number,
    SUM(t.amount) AS daily_total,
    COUNT(*) AS txn_count
FROM accounts a
JOIN transactions t ON a.account_id = t.account_id
WHERE t.transaction_date = CURRENT_DATE - 1
GROUP BY a.account_number;

-- Optimizations applied:
-- 1. Partition transactions by date
-- 2. Add covering index
-- 3. Pre-aggregate in materialized view

-- Partitioned table
CREATE TABLE transactions (
    transaction_id BIGINT,
    account_id BIGINT,
    amount DECIMAL,
    transaction_date DATE,
    ...
) PARTITION BY RANGE (transaction_date);

-- Create monthly partitions
CREATE TABLE transactions_2025_01 
    PARTITION OF transactions 
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Covering index on partition
CREATE INDEX idx_txn_account_covering 
ON transactions (account_id, transaction_date) 
INCLUDE (amount);

-- Materialized view for daily aggregation (refreshed nightly)
CREATE MATERIALIZED VIEW daily_account_summary AS
SELECT 
    account_id,
    transaction_date,
    SUM(amount) AS daily_total,
    COUNT(*) AS txn_count
FROM transactions
GROUP BY account_id, transaction_date;

CREATE UNIQUE INDEX ON daily_account_summary (account_id, transaction_date);

-- Now query runs in milliseconds
SELECT a.account_number, d.daily_total, d.txn_count
FROM accounts a
JOIN daily_account_summary d ON a.account_id = d.account_id
WHERE d.transaction_date = CURRENT_DATE - 1;
```

## Configuration Tuning

```sql
-- Check current settings
SHOW work_mem;
SHOW shared_buffers;
SHOW effective_cache_size;
SHOW maintenance_work_mem;
SHOW random_page_cost;

-- Recommended settings for analytics workloads (4-server with 64GB RAM)
-- postgresql.conf:
-- shared_buffers = 16GB           -- 25% of RAM
-- effective_cache_size = 48GB     -- 75% of RAM
-- work_mem = 256MB                -- Per-operation memory
-- maintenance_work_mem = 2GB      -- For VACUUM, CREATE INDEX
-- random_page_cost = 1.1          -- SSD storage
-- effective_io_concurrency = 200  -- SSD
-- default_statistics_target = 200 -- Better planner estimates
-- max_parallel_workers_per_gather = 4
```

## Cross-References

- **Advanced SQL**: See [advanced-sql.md](advanced-sql.md) for CTEs
- **Indexing**: See [indexing.md](../databases/indexing.md) for index types
- **Postgres Performance**: See [postgres-performance.md](../databases/postgres-performance.md) for tuning

## Interview Questions

1. **How do you read an EXPLAIN ANALYZE output? What metrics matter most?**
2. **Your query is doing a sequential scan on a 50M row table. What do you check and how do you fix it?**
3. **Explain the difference between Nested Loop, Hash Join, and Merge Join. When does the planner choose each?**
4. **What is an index-only scan and how do you enable it?**
5. **A query runs fast in isolation but slow in production. What could be the cause?**
6. **How does partition pruning work? When does it not kick in?**

## Checklist: Query Optimization

- [ ] Run EXPLAIN (ANALYZE, BUFFERS) on slow queries
- [ ] Check for sequential scans on large tables (add indexes)
- [ ] Verify join order and strategy (small table first)
- [ ] Ensure WHERE filters use indexed columns
- [ ] Check for unnecessary columns in SELECT (prevents index-only scans)
- [ ] Verify statistics are up to date (ANALYZE table)
- [ ] Review work_mem settings for hash/sort operations
- [ ] Consider partitioning for time-series data
- [ ] Use covering indexes for frequently accessed column combinations
- [ ] Test with production data volumes (not just development datasets)
