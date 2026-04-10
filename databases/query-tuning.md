# Database Query Tuning with EXPLAIN

## Overview

Query tuning is the skill that distinguishes effective database operators from those who treat the database as a black box. This guide covers EXPLAIN analysis, execution plan interpretation, optimization strategies, and banking-specific query patterns.

## EXPLAIN Basics

```sql
-- EXPLAIN: Shows planned execution without running
EXPLAIN
SELECT c.customer_id, SUM(t.amount) AS total
FROM customers c
JOIN transactions t ON c.customer_id = t.customer_id
WHERE t.transaction_time >= '2025-01-01'
GROUP BY c.customer_id;

-- EXPLAIN ANALYZE: Runs query and shows actual stats
EXPLAIN ANALYZE
SELECT c.customer_id, SUM(t.amount) AS total
FROM customers c
JOIN transactions t ON c.customer_id = t.customer_id
WHERE t.transaction_time >= '2025-01-01'
GROUP BY c.customer_id;

-- EXPLAIN with additional details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT * FROM transactions 
WHERE account_id = 12345 
ORDER BY transaction_time DESC 
LIMIT 100;

-- EXPLAIN output formats: TEXT (default), JSON, XML, YAML
EXPLAIN (ANALYZE, FORMAT JSON)
SELECT * FROM transactions WHERE account_id = 12345;
```

## Reading EXPLAIN ANALYZE Output

```
EXPLAIN ANALYZE
SELECT * FROM transactions 
WHERE account_id = 12345 AND transaction_time >= '2025-01-01';

                                                  QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Index Scan using idx_txn_account_time on transactions  
   (cost=0.56..234.56 rows=1234 width=128) 
   (actual time=0.045..2.345 rows=1234 loops=1)
   Index Cond: (account_id = 12345 AND transaction_time >= '2025-01-01'::date)
   Buffers: shared hit=456 read=12
 Planning Time: 0.234 ms
 Execution Time: 2.567 ms
```

### Key Metrics Explained

```
cost=0.56..234.56
  ├── 0.56: Startup cost (time before first row)
  └── 234.56: Total cost (arbitrary units, comparable between plans)

rows=1234
  └── Planner's estimate of output rows

width=128
  └── Estimated row width in bytes

actual time=0.045..2.345
  ├── 0.045: Startup time (ms)
  └── 2.345: Total time (ms)

rows=1234
  └── Actual rows returned

loops=1
  └── How many times this node executed (multiply rows × loops for total)

Buffers: shared hit=456 read=12
  ├── hit: Pages found in shared buffer cache
  ├── read: Pages read from disk
  └── High "read" = cold cache or insufficient shared_buffers
```

## Identifying Problems

### Sequential Scan on Large Table

```
Seq Scan on transactions  (cost=0.00..150234.56 rows=1234 width=128)
  (actual time=1.234..567.890 rows=1234 loops=1)
  Filter: (account_id = 12345)
  Rows Removed by Filter: 49998766

Problem: Scanning 50M rows to find 1,234 matching rows
Solution: CREATE INDEX idx_transactions_account_id ON transactions (account_id);
```

### Nested Loop with Large Inner Side

```
Nested Loop  (cost=0.56..56789.01 rows=100 width=256)
  (actual time=0.056..2345.678 rows=100 loops=1)
  ->  Index Scan using idx_customers_pkey on customers
      Index Cond: (customer_id = 100)
  ->  Seq Scan on transactions  (cost=0.00..500.00 rows=100 width=128)
      Filter: (customer_id = 100)
      Rows Removed by Filter: 49999900

Problem: Inner loop scans 50M rows for each outer row
Solution: Add index on transactions(customer_id)
```

### Hash Join with Spilling

```
Hash Join  (cost=1234.56..5678.90 rows=10000 width=256)
  (actual time=23.456..345.678 rows=10234 loops=1)
  Hash Cond: (t.customer_id = c.customer_id)
  ->  Seq Scan on transactions
  ->  Hash  (actual time=12.345..12.345 rows=50000 loops=1)
        Batches: 4  Memory Usage: 25600kB  Disk: 204800kB
        ->  Seq Scan on customers

Problem: Hash doesn't fit in memory, spills to disk (4 batches)
Solution: Increase work_mem to fit hash in memory
  SET work_mem = '256MB';  -- Before running the query
```

### Sort Spilling to Disk

```
Sort  (cost=5678.90..5789.01 rows=100000 width=128)
  (actual time=45.678..67.890 rows=100000 loops=1)
  Sort Key: amount DESC
  Sort Method: external merge  Disk: 12345kB
  ->  Seq Scan on transactions

Problem: Sort spilled to disk (external merge)
Solution: Increase work_mem, or add index for ORDER BY
  CREATE INDEX idx_transactions_amount ON transactions (amount DESC);
```

## Optimization Patterns

### Pattern 1: Replace Subquery with JOIN

```sql
-- Slow: Correlated subquery (runs once per row)
SELECT 
    account_id,
    (SELECT SUM(amount) FROM transactions t 
     WHERE t.account_id = a.account_id 
       AND t.transaction_time >= '2025-01-01') AS total
FROM accounts a;

-- Fast: JOIN with aggregation
SELECT 
    a.account_id,
    COALESCE(SUM(t.amount), 0) AS total
FROM accounts a
LEFT JOIN transactions t ON a.account_id = t.account_id
    AND t.transaction_time >= '2025-01-01'
GROUP BY a.account_id;
```

### Pattern 2: Push Filters Into Subqueries

```sql
-- Slow: Filter applied after join
SELECT * FROM (
    SELECT c.*, a.* FROM customers c
    JOIN accounts a ON c.customer_id = a.customer_id
) sub
WHERE sub.customer_segment = 'PREMIUM'
  AND sub.balance > 10000;

-- Fast: Filter before join
SELECT c.*, a.* FROM 
    (SELECT * FROM customers WHERE customer_segment = 'PREMIUM') c
JOIN 
    (SELECT * FROM accounts WHERE balance > 10000) a
ON c.customer_id = a.customer_id;
```

### Pattern 3: Use EXISTS Instead of IN for Large Lists

```sql
-- Slow: IN with large subquery (materializes entire result)
SELECT * FROM accounts
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE risk_rating = 'HIGH'
);

-- Fast: EXISTS (short-circuits on first match)
SELECT * FROM accounts a
WHERE EXISTS (
    SELECT 1 FROM customers c 
    WHERE c.customer_id = a.customer_id 
      AND c.risk_rating = 'HIGH'
);
```

### Pattern 4: Lateral Join for Top-N-Per-Group

```sql
-- Get the 3 most recent transactions per account
SELECT a.account_id, a.account_number, t.*
FROM accounts a
CROSS JOIN LATERAL (
    SELECT *
    FROM transactions
    WHERE account_id = a.account_id
    ORDER BY transaction_time DESC
    LIMIT 3
) t
WHERE a.status = 'ACTIVE';

-- Without LATERAL, you'd need window functions:
SELECT * FROM (
    SELECT 
        a.account_id,
        t.*,
        ROW_NUMBER() OVER (PARTITION BY a.account_id ORDER BY t.transaction_time DESC) AS rn
    FROM accounts a
    JOIN transactions t ON a.account_id = t.account_id
    WHERE a.status = 'ACTIVE'
) ranked
WHERE rn <= 3;
```

## Banking Query Optimization Example

```sql
-- Original: 45 seconds
EXPLAIN ANALYZE
SELECT 
    a.account_number,
    c.customer_name,
    SUM(t.amount) AS monthly_total,
    COUNT(*) AS txn_count
FROM transactions t
JOIN accounts a ON t.account_id = a.account_id
JOIN customers c ON a.customer_id = c.customer_id
WHERE t.transaction_time >= '2025-01-01'
  AND t.transaction_time < '2025-02-01'
  AND a.status = 'ACTIVE'
GROUP BY a.account_number, c.customer_name
ORDER BY monthly_total DESC;

-- Optimizations:
-- 1. Add covering index
CREATE INDEX idx_txn_time_covering ON transactions (transaction_time, account_id)
    INCLUDE (amount);

-- 2. Filter accounts first (smaller dataset)
CREATE INDEX idx_accounts_status ON accounts (status, account_id)
    INCLUDE (account_number, customer_id);

-- 3. Use CTE to materialize filtered accounts
WITH active_accounts AS MATERIALIZED (
    SELECT account_id, account_number, customer_id
    FROM accounts
    WHERE status = 'ACTIVE'
)
SELECT 
    aa.account_number,
    c.customer_name,
    SUM(t.amount) AS monthly_total,
    COUNT(*) AS txn_count
FROM active_accounts aa
JOIN transactions t ON aa.account_id = t.account_id
    AND t.transaction_time >= '2025-01-01'
    AND t.transaction_time < '2025-02-01'
JOIN customers c ON aa.customer_id = c.customer_id
GROUP BY aa.account_number, c.customer_name
ORDER BY monthly_total DESC;

-- Result: 0.5 seconds (90x improvement)
```

## Cross-References

- **Indexing**: See [indexing.md](indexing.md) for index types
- **Postgres Performance**: See [postgres-performance.md](postgres-performance.md) for tuning
- **Query Optimization**: See [query-optimization.md](../data-engineering/query-optimization.md) for data engineering perspective

## Interview Questions

1. **How do you read an EXPLAIN ANALYZE output? What does cost mean?**
2. **Your query plan shows a sequential scan on a 500M row table. What do you do?**
3. **What is the difference between estimated and actual rows? Why does it matter?**
4. **A hash join is spilling to disk. How do you fix it?**
5. **When does the planner choose Nested Loop vs Hash Join vs Merge Join?**
6. **How do you use pg_stat_statements to find slow queries?**

## Checklist: Query Tuning

- [ ] EXPLAIN (ANALYZE, BUFFERS) run on all slow queries
- [ ] Sequential scans on large tables investigated
- [ ] Join strategies verified (small table should drive)
- [ ] work_mem sufficient for hash/sort operations
- [ ] Statistics up to date (ANALYZE run recently)
- [ ] pg_stat_statements enabled and monitored
- [ ] Query patterns reviewed for index opportunities
- [ ] Subqueries converted to JOINs where beneficial
- [ ] Filters pushed as close to data source as possible
