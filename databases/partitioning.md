# PostgreSQL Partitioning

## Overview

Table partitioning splits large tables into smaller physical pieces while maintaining a single logical table. This is essential for banking tables with billions of rows (transactions, audit logs, events) to maintain query performance and enable efficient data lifecycle management.

## Declarative Partitioning

```sql
-- Range partitioning: most common for time-series banking data
CREATE TABLE transactions (
    transaction_id BIGINT,
    account_id BIGINT NOT NULL,
    transaction_type VARCHAR(20) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    transaction_time TIMESTAMPTZ NOT NULL,
    channel VARCHAR(20),
    status VARCHAR(20) DEFAULT 'PENDING',
    PRIMARY KEY (transaction_id, transaction_time)  -- Partition key must be in PK
) PARTITION BY RANGE (transaction_time);

-- Create monthly partitions
CREATE TABLE transactions_2025_01 
    PARTITION OF transactions 
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE transactions_2025_02 
    PARTITION OF transactions 
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE TABLE transactions_2025_03 
    PARTITION OF transactions 
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

-- Default partition catches rows that don't match any partition
CREATE TABLE transactions_default 
    PARTITION OF transactions DEFAULT;
```

## Partition Pruning

```sql
-- Partition pruning: query planner skips irrelevant partitions

-- This query only scans transactions_2025_01
EXPLAIN SELECT * FROM transactions
WHERE transaction_time >= '2025-01-01' AND transaction_time < '2025-02-01';

-- Verify pruning is working
SET enable_partition_pruning = off;  -- Disable to compare
EXPLAIN SELECT * FROM transactions
WHERE transaction_time >= '2025-01-01';

SET enable_partition_pruning = on;  -- Re-enable
EXPLAIN SELECT * FROM transactions
WHERE transaction_time >= '2025-01-01';

-- Pruning also works with parameters (prepared statements)
PREPARE get_month_txns(DATE) AS
    SELECT * FROM transactions 
    WHERE transaction_time >= $1 
      AND transaction_time < $1 + INTERVAL '1 month';

EXECUTE get_month_txns('2025-01-01');

-- When pruning does NOT work:
-- 1. Partition key not in WHERE clause
-- 2. Function wraps partition key: WHERE DATE(transaction_time) = '2025-01-15'
-- 3. Implicit type conversion: WHERE transaction_time = '2025-01-15' (without cast)
-- 4. Complex expressions the planner can't simplify

-- Fix: Use partition key directly
WHERE transaction_time >= '2025-01-15' AND transaction_time < '2025-01-16';
```

## Automated Partition Management

```sql
-- Automated partition creation and archival
CREATE OR REPLACE FUNCTION manage_partitions(
    p_parent_table TEXT,
    p_partition_interval TEXT DEFAULT '1 month',
    p_future_partitions INT DEFAULT 6,
    p_retention_partitions INT DEFAULT 36
) RETURNS TABLE(action TEXT, partition_name TEXT) AS $$
DECLARE
    v_partition_start DATE;
    v_partition_end DATE;
    v_partition_name TEXT;
    v_record RECORD;
    v_cutoff DATE;
BEGIN
    -- Create future partitions
    FOR i IN 0..p_future_partitions LOOP
        v_partition_start := DATE_TRUNC('month', CURRENT_DATE + (i || ' months')::INTERVAL)::DATE;
        v_partition_end := (v_partition_start + p_partition_interval::INTERVAL)::DATE;
        v_partition_name := p_parent_table || '_' || TO_CHAR(v_partition_start, 'YYYY_MM');
        
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
            v_partition_name, p_parent_table, v_partition_start, v_partition_end
        );
        
        -- Create indexes on new partition
        EXECUTE format(
            'CREATE INDEX IF NOT EXISTS %I ON %I (account_id)',
            v_partition_name || '_account_idx', v_partition_name
        );
        
        action := 'CREATE';
        partition_name := v_partition_name;
        RETURN NEXT;
    END LOOP;
    
    -- Drop old partitions
    v_cutoff := DATE_TRUNC('month', CURRENT_DATE - (p_retention_partitions || ' months')::INTERVAL)::DATE;
    
    FOR v_record IN
        SELECT inhrelid::regclass::text AS part_name
        FROM pg_inherits
        WHERE inhparent = p_parent_table::regclass
        ORDER BY inhrelid::regclass::text
    LOOP
        -- Extract date from partition name
        DECLARE
            v_part_date DATE;
        BEGIN
            v_part_date := TO_DATE(RIGHT(v_record.part_name, 7), 'YYYY_MM');
            
            IF v_part_date < v_cutoff AND v_record.part_name !~ '_default$' THEN
                -- Detach first, then drop (safer)
                EXECUTE format(
                    'ALTER TABLE %I DETACH PARTITION %I',
                    p_parent_table, v_record.part_name
                );
                EXECUTE format('DROP TABLE %I', v_record.part_name);
                
                action := 'DROP';
                partition_name := v_record.part_name;
                RETURN NEXT;
            END IF;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Run monthly via cron or Airflow
SELECT * FROM manage_partitions('transactions');
```

## Partitioning Strategies

### List Partitioning

```sql
-- Partition by categorical column
CREATE TABLE accounts (
    account_id BIGINT,
    account_type VARCHAR(20) NOT NULL,
    customer_id BIGINT,
    balance DECIMAL(15, 2),
    PRIMARY KEY (account_id, account_type)
) PARTITION BY LIST (account_type);

CREATE TABLE accounts_checking PARTITION OF accounts
    FOR VALUES IN ('CHECKING');
CREATE TABLE accounts_savings PARTITION OF accounts
    FOR VALUES IN ('SAVINGS');
CREATE TABLE accounts_loan PARTITION OF accounts
    FOR VALUES IN ('LOAN');
CREATE TABLE accounts_credit PARTITION OF accounts
    FOR VALUES IN ('CREDIT_CARD');
```

### Hash Partitioning

```sql
-- Partition by hash for even distribution
CREATE TABLE customer_events (
    event_id BIGINT,
    customer_id BIGINT NOT NULL,
    event_type VARCHAR(50),
    event_data JSONB,
    created_at TIMESTAMPTZ,
    PRIMARY KEY (event_id, customer_id)
) PARTITION BY HASH (customer_id);

-- 16 hash partitions for parallel processing
CREATE TABLE customer_events_p0 PARTITION OF customer_events
    FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE customer_events_p1 PARTITION OF customer_events
    FOR VALUES WITH (MODULUS 16, REMAINDER 1);
-- ... p2 through p15
```

### Sub-Partitioning

```sql
-- Partition by range (date), then by list (region)
CREATE TABLE transactions_regional (
    transaction_id BIGINT,
    account_id BIGINT,
    region VARCHAR(20) NOT NULL,
    transaction_time TIMESTAMPTZ NOT NULL,
    amount DECIMAL(15, 2),
    PRIMARY KEY (transaction_id, transaction_time, region)
) PARTITION BY RANGE (transaction_time);

-- Create monthly parent partitions
CREATE TABLE transactions_reg_2025_01 
    PARTITION OF transactions_regional
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01')
    PARTITION BY LIST (region);

-- Sub-partitions by region
CREATE TABLE transactions_reg_2025_01_na 
    PARTITION OF transactions_reg_2025_01
    FOR VALUES IN ('NA_EAST', 'NA_WEST');
CREATE TABLE transactions_reg_2025_01_eu 
    PARTITION OF transactions_reg_2025_01
    FOR VALUES IN ('UK', 'GERMANY', 'FRANCE');
```

## Cross-References

- **Indexing**: See [indexing.md](indexing.md) for index strategies on partitions
- **Retention Policies**: See [retention-policies.md](../data-engineering/retention-policies.md) for archival
- **Query Tuning**: See [query-tuning.md](query-tuning.md) for partition-aware queries

## Interview Questions

1. **How does partition pruning work? When does it NOT kick in?**
2. **Your partitioned table query is still doing a sequential scan. Why?**
3. **How do you migrate a 500M row non-partitioned table to a partitioned table with zero downtime?**
4. **What happens to indexes when you partition a table?**
5. **When would you use sub-partitioning vs a single-level partition?**
6. **How do you handle the default partition growing too large?**

## Checklist: Partitioning

- [ ] Partition key included in primary key
- [ ] Partition pruning verified with EXPLAIN
- [ ] Automated partition creation scheduled
- [ ] Old partition detachment and archival automated
- [ ] Indexes created on each partition
- [ ] Default partition monitored for unexpected data
- [ ] FK constraints considered (partitioning breaks traditional FKs)
- [ ] Unique constraints include partition key
- [ ] VACUUM and ANALYZE run on new partitions
- [ ] Backup strategy covers all partitions
