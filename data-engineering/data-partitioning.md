# Data Partitioning and Index Strategies

## Overview

Partitioning splits large tables into smaller, manageable pieces while maintaining a unified query interface. Combined with appropriate index strategies, partitioning enables banking data systems to scale to billions of rows while maintaining sub-second query response times. This guide covers partitioning methods, index types, and practical strategies for banking data.

## Partitioning Methods

### Range Partitioning (Time-Series)

```sql
-- Most common for banking: partition by date/time
CREATE TABLE transactions (
    transaction_id BIGINT,
    account_id BIGINT NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3),
    transaction_time TIMESTAMPTZ NOT NULL,
    status VARCHAR(20),
    PRIMARY KEY (transaction_id, transaction_time)
) PARTITION BY RANGE (transaction_time);

-- Monthly partitions
CREATE TABLE transactions_2024_01 PARTITION OF transactions
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE transactions_2024_02 PARTITION OF transactions
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... continue for each month

-- Partition pruning: query only relevant partitions
EXPLAIN SELECT * FROM transactions 
WHERE transaction_time >= '2025-01-01' AND transaction_time < '2025-02-01';
-- Query plan shows only transactions_2025_01 is scanned

-- Old data: detach and archive
ALTER TABLE transactions DETACH PARTITION transactions_2023_01;
-- Move to cold storage, then drop
```

### List Partitioning (Categorical)

```sql
-- Partition by region or business unit
CREATE TABLE customer_records (
    customer_id BIGINT,
    region VARCHAR(20) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    segment VARCHAR(20),
    PRIMARY KEY (customer_id, region)
) PARTITION BY LIST (region);

CREATE TABLE customers_na PARTITION OF customer_records
    FOR VALUES IN ('US_EAST', 'US_WEST', 'CANADA');
CREATE TABLE customers_eu PARTITION OF customer_records
    FOR VALUES IN ('UK', 'GERMANY', 'FRANCE', 'EU_OTHER');
CREATE TABLE customers_apac PARTITION OF customer_records
    FOR VALUES IN ('JAPAN', 'AUSTRALIA', 'SINGAPORE', 'APAC_OTHER');
CREATE TABLE customers_other PARTITION OF customer_records
    FOR VALUES IN (DEFAULT);
```

### Hash Partitioning (Even Distribution)

```sql
-- Partition by hash for even distribution without natural ranges
CREATE TABLE account_events (
    event_id BIGINT,
    account_id BIGINT NOT NULL,
    event_type VARCHAR(50),
    event_data JSONB,
    created_at TIMESTAMPTZ,
    PRIMARY KEY (event_id, account_id)
) PARTITION BY HASH (account_id);

-- 8 hash partitions for parallelism
CREATE TABLE account_events_p0 PARTITION OF account_events
    FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE account_events_p1 PARTITION OF account_events
    FOR VALUES WITH (MODULUS 8, REMAINDER 1);
-- ... through p7
```

## Index Strategies

### B-Tree Indexes (Default)

```sql
-- B-Tree: Best for equality and range queries
-- Good for: account_id, date ranges, sorted access

-- Single column
CREATE INDEX idx_txn_account ON transactions (account_id);

-- Composite: order matters! Most selective first
CREATE INDEX idx_txn_account_time ON transactions (account_id, transaction_time DESC);

-- Covering index: include columns needed by query
CREATE INDEX idx_txn_covering ON transactions (account_id, transaction_time) 
    INCLUDE (amount, currency, status);
-- Enables index-only scan for queries selecting only these columns
```

### Partial Indexes

```sql
-- Index only active/important rows
CREATE INDEX idx_txn_active ON transactions (account_id, transaction_time)
    WHERE status = 'PENDING';

-- Index only high-value transactions
CREATE INDEX idx_txn_high_value ON transactions (customer_id, amount)
    WHERE amount > 10000;

-- Index only recent data (most queried)
CREATE INDEX idx_txn_recent ON transactions (account_id, transaction_time)
    WHERE transaction_time >= CURRENT_DATE - INTERVAL '90 days';
```

### BRIN Indexes (Block Range)

```sql
-- BRIN: Very small index for naturally ordered data
-- Good for: time-series, append-only tables, very large tables

CREATE INDEX idx_txn_time_brin ON transactions USING brin (transaction_time);
-- Size: ~1MB vs ~500MB for B-Tree on 500M rows
-- Trade-off: Less precise, but much smaller

-- Use when:
-- - Data is physically ordered by the indexed column
-- - Table is very large (billions of rows)
-- - Queries filter on the indexed column with range conditions
```

## Partition Maintenance

```sql
-- Automated partition management
CREATE OR REPLACE FUNCTION manage_partitions(
    p_table_name TEXT,
    p_retention_months INT DEFAULT 24,
    p_future_months INT DEFAULT 6
) RETURNS TABLE(action TEXT, detail TEXT) AS $$
DECLARE
    v_partition_record RECORD;
    v_cutoff_date DATE;
    v_future_date DATE;
BEGIN
    v_cutoff_date := DATE_TRUNC('month', CURRENT_DATE - (p_retention_months || ' months')::INTERVAL)::DATE;
    v_future_date := DATE_TRUNC('month', CURRENT_DATE + (p_future_months || ' months')::INTERVAL)::DATE;
    
    -- Create future partitions
    FOR i IN 0..p_future_months LOOP
        DECLARE
            v_start DATE := DATE_TRUNC('month', CURRENT_DATE + (i || ' months')::INTERVAL)::DATE;
            v_end DATE := v_start + INTERVAL '1 month';
            v_part_name TEXT := p_table_name || '_' || TO_CHAR(v_start, 'YYYY_MM');
        BEGIN
            EXECUTE format(
                'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
                v_part_name, p_table_name, v_start, v_end
            );
            action := 'CREATE';
            detail := v_part_name;
            RETURN NEXT;
        END;
    END LOOP;
    
    -- Drop old partitions
    FOR v_partition_record IN
        SELECT inhrelid::regclass::text AS partition_name
        FROM pg_inherits
        WHERE inhparent = p_table_name::regclass
    LOOP
        DECLARE
            v_part_date DATE;
        BEGIN
            -- Extract date from partition name (assumes YYYY_MM suffix)
            v_part_date := TO_DATE(SUBSTR(v_partition_record.partition_name, -7), 'YYYY_MM');
            
            IF v_part_date < v_cutoff_date THEN
                EXECUTE format('DROP TABLE %I', v_partition_record.partition_name);
                action := 'DROP';
                detail := v_partition_record.partition_name;
                RETURN NEXT;
            END IF;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Run monthly via cron/Airflow
SELECT * FROM manage_partitions('transactions', p_retention_months => 36, p_future_months => 6);
```

## Cross-References

- **Query Optimization**: See [query-optimization.md](query-optimization.md) for EXPLAIN plans
- **Indexing**: See [indexing.md](../databases/indexing.md) for index types
- **Postgres Best Practices**: See [postgres-best-practices.md](postgres-best-practices.md) for production patterns

## Interview Questions

1. **When does partition pruning NOT kick in even though the partition key is in the WHERE clause?**
2. **How do you choose between range, list, and hash partitioning?**
3. **Your partitioned table query is still slow. What could be the issue?**
4. **What is a covering index and when should you use it?**
5. **How do you migrate a 1B row non-partitioned table to a partitioned table with zero downtime?**
6. **When would you use a BRIN index instead of a B-Tree?**

## Checklist: Partitioning and Indexing

- [ ] Partitioning strategy aligned with common query patterns
- [ ] Partition key included in primary key
- [ ] Automated partition creation scheduled
- [ ] Old partition archival/deletion policy implemented
- [ ] Indexes created on foreign key columns
- [ ] Composite indexes ordered by selectivity
- [ ] Partial indexes for selective queries
- [ ] Covering indexes for index-only scans
- [ ] Index bloat monitored
- [ ] Unused indexes identified and removed quarterly
- [ ] Partition pruning verified with EXPLAIN
