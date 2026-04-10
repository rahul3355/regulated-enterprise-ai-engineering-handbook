# Postgres Best Practices for Banking Data at Scale

## Overview

PostgreSQL is the default database for banking data engineering. Its ACID compliance, rich feature set, and extensibility (pgvector, logical replication, row-level security) make it ideal for both transactional and analytical workloads. This guide covers production-grade Postgres patterns for handling banking-scale data: connection pooling, partitioning, query patterns, and operational considerations.

## Connection Pooling

```yaml
# PgBouncer configuration for banking workloads
# PgBouncer sits between application servers and Postgres
# Prevents connection storms during traffic spikes

[databases]
banking_prod = host=db-primary port=5432 dbname=banking
analytics_db = host=db-analytics port=5432 dbname=analytics

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pooling mode: transaction vs session
# Transaction: connection released after each transaction (recommended)
# Session: connection held for entire session (needed for prepared statements)
pool_mode = transaction

# Connection limits
max_client_conn = 5000        # Max connections from apps
default_pool_size = 50        # Connections per DB
min_pool_size = 10            # Keep warm connections
reserve_pool_size = 10        # Extra connections under load
reserve_pool_timeout = 3      # Seconds before using reserve pool

# Timeouts
server_lifetime = 3600        # Recycle server connections hourly
server_idle_timeout = 600     # Close idle server connections
client_idle_timeout = 0       # Don't timeout clients

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1

# Stats
stats_period = 60
```

```python
# SQLAlchemy connection pool configuration
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql+psycopg2://user:pass@pgbouncer-host:6432/banking_prod',
    poolclass=QueuePool,
    pool_size=20,              # Connections in pool
    max_overflow=10,           # Extra connections allowed
    pool_pre_ping=True,        # Verify connection health
    pool_recycle=1800,         # Recycle connections every 30 min
    pool_timeout=30,           # Wait time for connection
    echo_pool=False,
    
    # Query-level settings
    connect_args={
        'options': '-c statement_timeout=30000'  # 30 second timeout
    }
)

# For batch/ETL workloads, use separate engine with larger pool
etl_engine = create_engine(
    'postgresql+psycopg2://etl_user:pass@pgbouncer-host:6432/banking_prod',
    poolclass=QueuePool,
    pool_size=50,
    max_overflow=20,
    connect_args={
        'options': '-c statement_timeout=300000'  # 5 min for ETL
    }
)
```

## Table Partitioning

```sql
-- Range partitioning for time-series data (most common in banking)
CREATE TABLE transactions (
    transaction_id BIGINT,
    account_id BIGINT NOT NULL,
    transaction_type VARCHAR(20) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    transaction_time TIMESTAMPTZ NOT NULL,
    channel VARCHAR(20),
    status VARCHAR(20) DEFAULT 'PENDING',
    reference VARCHAR(100),
    PRIMARY KEY (transaction_id, transaction_time)  -- Partition key must be in PK
) PARTITION BY RANGE (transaction_time);

-- Create monthly partitions
CREATE TABLE transactions_2025_01 
    PARTITION OF transactions 
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE transactions_2025_02 
    PARTITION OF transactions 
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Automate partition creation
CREATE OR REPLACE FUNCTION create_monthly_partition(
    table_name TEXT,
    start_date DATE,
    months_ahead INT DEFAULT 6
) RETURNS void AS $$
DECLARE
    partition_start DATE;
    partition_end DATE;
    partition_name TEXT;
BEGIN
    FOR i IN 0..months_ahead LOOP
        partition_start := start_date + (i || ' months')::INTERVAL;
        partition_end := partition_start + '1 month'::INTERVAL;
        partition_name := table_name || '_' || TO_CHAR(partition_start, 'YYYY_MM');
        
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
            partition_name, table_name, partition_start, partition_end
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Call monthly via cron/Airflow
SELECT create_monthly_partition('transactions', CURRENT_DATE, months_ahead => 6);

-- Indexes on partitions (each partition has its own indexes)
CREATE INDEX idx_txn_account_2025_01 
    ON transactions_2025_01 (account_id);
CREATE INDEX idx_txn_time_2025_01 
    ON transactions_2025_01 (transaction_time);
```

## Query Patterns for Banking

```sql
-- Pagination for large result sets (avoid OFFSET for performance)
-- Use keyset pagination instead
SELECT 
    transaction_id,
    account_id,
    amount,
    transaction_time
FROM transactions
WHERE transaction_time < '2025-01-15 10:30:00'  -- Last seen timestamp
ORDER BY transaction_time DESC
LIMIT 100;

-- Bulk upsert for ETL loads
INSERT INTO account_balances (account_id, balance_date, balance)
SELECT 
    account_id,
    CURRENT_DATE,
    SUM(CASE WHEN type IN ('DEPOSIT', 'TRANSFER_IN') THEN amount 
             ELSE -amount END) AS balance
FROM transactions
WHERE DATE(transaction_time) = CURRENT_DATE
GROUP BY account_id
ON CONFLICT (account_id, balance_date) 
DO UPDATE SET
    balance = EXCLUDED.balance,
    updated_at = NOW();

-- Batch delete old data (avoid deleting millions of rows in one transaction)
CREATE OR REPLACE FUNCTION delete_old_transactions(cutoff_date DATE)
RETURNS bigint AS $$
DECLARE
    deleted_count BIGINT := 0;
    batch_count BIGINT;
BEGIN
    LOOP
        DELETE FROM transactions
        WHERE transaction_id IN (
            SELECT transaction_id 
            FROM transactions 
            WHERE transaction_time < cutoff_date
            LIMIT 10000
        );
        
        GET DIAGNOSTICS batch_count = ROW_COUNT;
        deleted_count := deleted_count + batch_count;
        
        COMMIT;  -- Commit each batch
        
        EXIT WHEN batch_count = 0;
        PERFORM pg_sleep(0.1);  -- Brief pause to reduce replication lag
    END LOOP;
    
    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;

-- Materialized view for dashboard queries
CREATE MATERIALIZED VIEW mv_daily_branch_summary AS
SELECT 
    DATE(t.transaction_time) AS txn_date,
    b.branch_id,
    b.branch_name,
    COUNT(*) AS txn_count,
    SUM(t.amount) AS total_volume,
    COUNT(DISTINCT t.account_id) AS active_accounts
FROM transactions t
JOIN accounts a ON t.account_id = a.account_id
JOIN branches b ON a.branch_id = b.branch_id
GROUP BY DATE(t.transaction_time), b.branch_id, b.branch_name
WITH DATA;

CREATE UNIQUE INDEX ON mv_daily_branch_summary (txn_date, branch_id);

-- Refresh concurrently (allows reads during refresh)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_branch_summary;
```

## Maintenance and Operations

```sql
-- Vacuum and analyze configuration
-- Auto-vacuum tuning for high-throughput tables
ALTER TABLE transactions SET (
    autovacuum_vacuum_scale_factor = 0.05,    -- Vacuum after 5% dead rows
    autovacuum_analyze_scale_factor = 0.02,   -- Analyze after 2% changes
    autovacuum_vacuum_threshold = 10000,       -- Minimum threshold
    autovacuum_analyze_threshold = 5000
);

-- Check bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename) 
                   - pg_relation_size(schemaname || '.' || tablename)) AS index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- Monitor replication lag
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    EXTRACT(EPOCH FROM replay_lag) AS replay_lag_seconds
FROM pg_stat_replication;

-- Lock monitoring
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

## Cross-References

- **Query Optimization**: See [query-optimization.md](query-optimization.md) for performance tuning
- **Partitioning**: See [partitioning.md](../databases/partitioning.md) for partition strategies
- **Replication**: See [replication.md](../databases/replication.md) for HA setup

## Interview Questions

1. **How does PgBouncer's transaction mode differ from session mode? When would you use each?**
2. **Your transactions table has 500M rows and queries are slow. What do you do?**
3. **How do you handle bulk data loads without impacting production queries?**
4. **What is partition pruning and how do you verify it is working?**
5. **A query is blocked waiting for a lock. How do you diagnose and resolve it?**
6. **How do you safely delete 100M rows from a production table?**

## Checklist: Postgres at Scale

- [ ] PgBouncer configured with appropriate pool sizes
- [ ] Tables partitioned for data exceeding 100M rows
- [ ] Automated partition creation scheduled
- [ ] Auto-vacuum tuned for high-churn tables
- [ ] Materialized views with CONCURRENTLY refresh for dashboards
- [ ] Bulk operations use batched approach
- [ ] Keyset pagination used instead of OFFSET for large result sets
- [ ] Connection timeouts configured (statement, idle, lock)
- [ ] Replication lag monitored
- [ ] Lock contention monitored and alerted
- [ ] Index usage stats reviewed quarterly
- [ ] Table and index bloat checked regularly
