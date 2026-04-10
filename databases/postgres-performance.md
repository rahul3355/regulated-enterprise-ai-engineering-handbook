# PostgreSQL Performance Tuning

## Overview

PostgreSQL performance tuning is critical for banking systems handling millions of daily transactions. This guide covers connection management, caching, memory configuration, query optimization, and operational tuning for production banking workloads.

## Memory Configuration

```yaml
# postgresql.conf - Memory settings for 64GB server
# Rule of thumb: allocate memory in this order of priority

# 1. shared_buffers: PostgreSQL's buffer pool
# Recommended: 25% of RAM
shared_buffers = 16GB

# 2. effective_cache_size: Planner's estimate of available cache
# Recommended: 75% of RAM (shared_buffers + OS cache)
effective_cache_size = 48GB

# 3. work_mem: Per-operation memory for sorts, hashes, joins
# Careful: multiplied by operations × connections
# Recommended: 64MB - 256MB for OLTP, 256MB - 1GB for analytics
work_mem = 256MB

# 4. maintenance_work_mem: For VACUUM, CREATE INDEX, ALTER TABLE
# Recommended: 1-2GB for large databases
maintenance_work_mem = 2GB

# 5. wal_buffers: WAL write buffer
# Recommended: 64MB (auto-tuned to 1/32 of shared_buffers up to 16MB)
wal_buffers = 64MB

# 6. huge_pages: Use OS huge pages for shared_buffers
# Recommended: try for large shared_buffers
huge_pages = try
```

## Connection Management

```yaml
# postgresql.conf - Connection settings
max_connections = 200           # Max direct connections
# Use PgBouncer for connection pooling instead of increasing this

# superuser_reserved_connections: Reserve for admin
superuser_reserved_connections = 5

# tcp_keepalives: Detect dead connections
tcp_keepalives_idle = 60
tcp_keepalives_interval = 10
tcp_keepalives_count = 6

# Statement timeout: Prevent runaway queries
statement_timeout = 30000       # 30 seconds default
idle_in_transaction_session_timeout = 300000  # 5 min idle in transaction
lock_timeout = 10000            # 10 seconds for lock waits

# Per-role overrides
ALTER ROLE analytics_user SET statement_timeout = '300000';  # 5 min for analytics
ALTER ROLE etl_user SET work_mem = '512MB';
ALTER ROLE api_user SET statement_timeout = '5000';  # 5 seconds for API
```

## PgBouncer Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
banking_prod = host=localhost port=5432 dbname=banking pool_mode=transaction
analytics_db = host=localhost port=5432 dbname=analytics pool_mode=session

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool settings
pool_mode = transaction           # Release connection after each transaction
max_client_conn = 10000           # Max app connections
default_pool_size = 50            # Per-database pool size
min_pool_size = 10                # Keep connections warm
reserve_pool_size = 10            # Extra pool under load
reserve_pool_timeout = 3          # Use reserve after 3s wait

# Timeouts
server_lifetime = 3600            # Recycle server connections hourly
server_idle_timeout = 600         # Close idle server connections
server_connect_timeout = 15
server_login_retry = 15
client_idle_timeout = 0           # Don't timeout clients
client_login_timeout = 60

# Logging and stats
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60
```

## Query Performance

### Index Utilization

```sql
-- Check index usage
SELECT 
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC
LIMIT 20;

-- Identify unused indexes (candidates for removal)
SELECT 
    schemaname,
    relname,
    indexrelname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND pg_relation_size(indexrelid) > 1024 * 1024  -- > 1MB
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check for missing indexes (sequential scans on large tables)
SELECT 
    relname AS table_name,
    seq_scan AS sequential_scans,
    seq_tup_read AS tuples_read,
    idx_scan AS index_scans,
    n_live_tup AS estimated_rows,
    pg_size_pretty(pg_relation_size(relid)) AS table_size
FROM pg_stat_user_tables
WHERE seq_scan > 0
  AND n_live_tup > 100000
ORDER BY seq_tup_read DESC
LIMIT 20;
```

### Query Plan Analysis

```sql
-- Enable timing for all queries
SET track_activities = on;

-- Find slow queries
SELECT 
    pid,
    usename,
    query,
    state,
    wait_event_type,
    wait_event,
    EXTRACT(EPOCH FROM (now() - query_start)) AS duration_seconds,
    EXTRACT(EPOCH FROM (now() - state_change)) AS state_duration
FROM pg_stat_activity
WHERE state != 'idle'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration_seconds DESC
LIMIT 20;

-- Analyze query plans for common patterns
-- Use pg_stat_statements extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT 
    queryid,
    SUBSTR(query, 1, 100) AS query_preview,
    calls,
    ROUND(total_exec_time::numeric / 1000, 2) AS total_seconds,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(min_exec_time::numeric, 2) AS min_ms,
    ROUND(max_exec_time::numeric, 2) AS max_ms,
    rows AS total_rows,
    shared_blks_hit,
    shared_blks_read,
    CASE WHEN calls > 0 
        THEN ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2)
        ELSE 0 
    END AS cache_hit_pct
FROM pg_stat_statements
WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database())
ORDER BY total_exec_time DESC
LIMIT 20;
```

## Write Performance

### Bulk Operations

```sql
-- Bulk insert: Use COPY instead of INSERT for large loads
-- COPY is 10-50x faster than INSERT

-- From file
COPY transactions FROM '/data/transactions.csv' WITH (FORMAT csv, HEADER true);

-- From stdin (programmatic)
-- Use psycopg2's copy_expert for Python

-- Bulk insert with VALUES
INSERT INTO transactions (account_id, amount, type, txn_time)
VALUES 
    (1, 100, 'DEPOSIT', NOW()),
    (2, 200, 'WITHDRAWAL', NOW()),
    -- ... thousands of rows
ON CONFLICT DO NOTHING;

-- For very large loads:
-- 1. Drop indexes before load
-- 2. Load data
-- 3. Recreate indexes
-- 4. ANALYZE table

-- Disable autovacuum during massive loads
ALTER TABLE transactions SET (autovacuum_enabled = false);
-- Load data...
ALTER TABLE transactions SET (autovacuum_enabled = true);
VACUUM ANALYZE transactions;
```

### WAL Optimization

```sql
-- For write-heavy workloads
-- wal_level = replica (minimal for no replication, replica for standby, logical for CDC)
-- synchronous_commit = off (for bulk loads, risk of losing last transactions)
-- commit_delay = 100 (microseconds, group commits)
-- commit_siblings = 5 (only delay if this many concurrent transactions)

-- Temporary: disable synchronous commit for bulk load
SET synchronous_commit = off;
-- Load data...
SET synchronous_commit = on;

-- Check WAL write rate
SELECT 
    pg_walfile_name(pg_current_wal_lsn()) AS current_wal_file,
    pg_size_pretty(pg_current_wal_lsn() - '0/00000000'::pg_lsn) AS total_wal_written;
```

## Read Performance

### Materialized Views

```sql
-- Pre-compute expensive queries
CREATE MATERIALIZED VIEW mv_customer_360 AS
SELECT 
    c.customer_id,
    c.customer_segment,
    COUNT(DISTINCT a.account_id) AS account_count,
    SUM(a.balance) AS total_balance,
    COUNT(DISTINCT t.transaction_id) AS txn_count_90d,
    SUM(t.amount) AS total_spend_90d
FROM customers c
LEFT JOIN accounts a ON c.customer_id = a.customer_id
LEFT JOIN transactions t ON a.account_id = t.account_id
    AND t.transaction_time >= CURRENT_DATE - 90
GROUP BY c.customer_id, c.customer_segment;

CREATE UNIQUE INDEX ON mv_customer_360 (customer_id);

-- Refresh concurrently (allows reads during refresh)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_360;

-- Schedule refresh via cron/Airflow
```

## Configuration Tuning Script

```sql
-- Quick configuration audit
SELECT name, setting, unit, category, short_desc
FROM pg_settings
WHERE name IN (
    'shared_buffers',
    'effective_cache_size',
    'work_mem',
    'maintenance_work_mem',
    'wal_buffers',
    'max_connections',
    'random_page_cost',
    'effective_io_concurrency',
    'default_statistics_target',
    'checkpoint_timeout',
    'checkpoint_completion_target',
    'wal_level',
    'max_wal_size',
    'min_wal_size',
    'autovacuum_max_workers',
    'autovacuum_naptime',
    'huge_pages',
    'parallel_tuple_cost',
    'parallel_setup_cost',
    'max_parallel_workers_per_gather',
    'max_parallel_workers'
)
ORDER BY category, name;

-- Calculate recommended settings for your server
SELECT 
    'shared_buffers' AS parameter,
    (total_memory_bytes * 0.25)::bigint || ' bytes' AS recommended,
    current_setting('shared_buffers') AS current
FROM (SELECT pg_total_relation_size('pg_class') AS total_memory_bytes) t
UNION ALL
SELECT 
    'effective_cache_size',
    (total_memory_bytes * 0.75)::bigint || ' bytes',
    current_setting('effective_cache_size')
FROM (SELECT pg_total_relation_size('pg_class') AS total_memory_bytes) t;
```

## Cross-References

- **Postgres Fundamentals**: See [postgres-fundamentals.md](postgres-fundamentals.md) for architecture
- **Query Tuning**: See [query-tuning.md](query-tuning.md) for EXPLAIN analysis
- **Indexing**: See [indexing.md](indexing.md) for index types

## Interview Questions

1. **Your PostgreSQL server has 90% cache hit ratio. What does this mean and what do you do?**
2. **How do you size shared_buffers, work_mem, and effective_cache_size?**
3. **A query that was fast is now slow. How do you diagnose the issue?**
4. **What is the impact of setting synchronous_commit = off?**
5. **How do you optimize a bulk load of 100M rows into PostgreSQL?**
6. **When would you use a materialized view vs a regular view vs an index?**

## Checklist: Performance Tuning

- [ ] Memory settings sized for available RAM
- [ ] PgBouncer configured for connection pooling
- [ ] Statement timeouts configured per role
- [ ] Unused indexes identified and removed
- [ ] Slow queries identified and optimized
- [ ] pg_stat_statements enabled and monitored
- [ ] Cache hit ratio > 99% for OLTP
- [ ] Autovacuum tuned for high-churn tables
- [ ] Materialized views for expensive aggregations
- [ ] Bulk loads use COPY or batched INSERT
- [ ] WAL archiving configured for PITR
- [ ] Configuration changes tested before production
