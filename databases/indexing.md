# Database Indexing: B-Tree, GIN, GiST, BRIN, and HNSW

## Overview

Choosing the right index type is critical for database performance. PostgreSQL offers multiple index types, each optimized for different access patterns. This guide covers when and how to use each index type in banking data systems.

## Index Type Comparison

| Index Type | Best For | Banking Use Case | Size | Speed |
|-----------|----------|-----------------|------|-------|
| B-Tree | Equality, range queries | account_id, date ranges | Medium | Fast |
| Hash | Exact equality only | UUID lookups | Small | Fastest |
| GIN | Composite values, arrays, JSONB | JSONB metadata, full-text | Large | Medium |
| GiST | Geometric, range overlap | Geospatial, IP ranges | Medium | Medium |
| BRIN | Naturally ordered data, very large tables | Time-series, append-only | Tiny | Good |
| HNSW | Vector similarity search | Embedding retrieval | Large | Fast |
| SP-GiST | Non-balanced structures | Phone directories, radix trees | Medium | Medium |

## B-Tree Index (Default)

```sql
-- B-Tree is the default index type in PostgreSQL
-- Supports: =, <, >, <=, >=, BETWEEN, IN, IS NULL, LIKE 'prefix%'

-- Single column
CREATE INDEX idx_txn_account ON transactions (account_id);

-- Composite index: column order matters!
-- Most selective/equality columns first, range columns last
CREATE INDEX idx_txn_account_time ON transactions (account_id, transaction_time DESC);

-- This query uses the composite index efficiently:
SELECT * FROM transactions 
WHERE account_id = 12345 
  AND transaction_time >= '2025-01-01'
ORDER BY transaction_time DESC;

-- This query ALSO uses the index (leftmost prefix):
SELECT * FROM transactions WHERE account_id = 12345;

-- This query does NOT fully use the index (skips first column):
SELECT * FROM transactions WHERE transaction_time >= '2025-01-01';

-- Covering index: includes additional columns for index-only scans
CREATE INDEX idx_txn_covering ON transactions (account_id, transaction_time)
    INCLUDE (amount, currency, status);

-- Now this query uses index-only scan (no heap fetch):
SELECT account_id, transaction_time, amount, status
FROM transactions 
WHERE account_id = 12345;

-- Partial index: index only a subset of rows
CREATE INDEX idx_txn_pending ON transactions (account_id, transaction_time)
    WHERE status = 'PENDING';

-- Unique index: enforce uniqueness
CREATE UNIQUE INDEX idx_unique_txn ON transactions (transaction_id);
```

## GIN Index (Generalized Inverted Index)

```sql
-- GIN: Index composite values (arrays, JSONB, full-text)

-- JSONB indexing
CREATE TABLE banking_documents (
    id BIGINT PRIMARY KEY,
    title VARCHAR(500),
    content TEXT,
    metadata JSONB,
    tags TEXT[]
);

-- GIN index on JSONB (supports @>, ?, ?&, ?| operators)
CREATE INDEX idx_doc_metadata ON banking_documents USING gin (metadata);

-- Query with GIN index
SELECT * FROM banking_documents
WHERE metadata @> '{"doc_type": "product_info"}';

SELECT * FROM banking_documents
WHERE metadata ? 'regulatory_flag';

-- GIN index on arrays
CREATE INDEX idx_doc_tags ON banking_documents USING gin (tags);

-- Array query
SELECT * FROM banking_documents
WHERE tags @> ARRAY['high-priority', 'compliance'];

-- Full-text search with GIN
CREATE INDEX idx_doc_content_fts ON banking_documents 
    USING gin (to_tsvector('english', content));

SELECT * FROM banking_documents
WHERE to_tsvector('english', content) @@ to_tsquery('interest & rate');

-- GIN index trade-offs:
-- Pros: Supports complex queries on JSONB, arrays, full-text
-- Cons: Larger index size, slower writes
```

## GiST Index (Generalized Search Tree)

```sql
-- GiST: Supports range types, geometric data, full-text

-- Range indexing (for date ranges, IP ranges)
CREATE INDEX idx_account_active_range ON accounts 
    USING gist (daterange(opened_date, COALESCE(closed_date, 'infinity')));

-- Query overlapping ranges
SELECT * FROM accounts
WHERE daterange(opened_date, COALESCE(closed_date, 'infinity')) && 
      daterange('2025-01-01', '2025-12-31');

-- IP address range indexing
CREATE INDEX idx_access_logs_ip ON access_logs 
    USING gist (ip_range);

-- Trigram index with GiST (for fuzzy text matching)
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_customer_name_trgm ON customers 
    USING gist (last_name gist_trgm_ops);

-- Fuzzy name search
SELECT * FROM customers
WHERE last_name % 'Smiith'  -- Similarity match
ORDER BY last_name <-> 'Smiith'  -- Order by distance
LIMIT 10;

-- Note: For trigram, GIN is usually faster than GiST
-- CREATE INDEX idx_customer_name_trgm_gin ON customers USING gin (last_name gin_trgm_ops);
```

## BRIN Index (Block Range Index)

```sql
-- BRIN: Very small index for naturally ordered, append-only data
-- Ideal for time-series data in banking

-- BRIN stores min/max values per block range (default 128 pages)
CREATE INDEX idx_txn_time_brin ON transactions USING brin (transaction_time);

-- Size comparison for 500M row transactions table:
-- B-Tree: ~11 GB
-- BRIN:   ~5 MB

-- BRIN works well when:
-- 1. Data is physically ordered by the indexed column
-- 2. Table is append-only (no updates/deletes that reorder)
-- 3. Queries use range conditions on the indexed column

-- Configure block size for BRIN
CREATE INDEX idx_txn_time_brin_64 ON transactions 
    USING brin (transaction_time) WITH (pages_per_range = 64);

-- When BRIN is NOT effective:
-- - Data is randomly ordered
-- - High cardinality with low clustering
-- - Frequent updates/reordering

-- Check BRIN effectiveness
SELECT 
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan
FROM pg_stat_user_indexes
WHERE indexrelname LIKE '%brin%';
```

## HNSW Index (Hierarchical Navigable Small World)

```sql
-- HNSW: For vector similarity search (requires pgvector extension)
-- Best for: Approximate nearest neighbor search on embeddings

CREATE EXTENSION IF NOT EXISTS vector;

-- Document embeddings table
CREATE TABLE document_embeddings (
    chunk_id UUID PRIMARY KEY,
    content TEXT,
    embedding vector(1536),
    metadata JSONB
);

-- HNSW index for fast similarity search
CREATE INDEX idx_embeddings_hnsw ON document_embeddings 
    USING hnsw (embedding vector_cosine_ops);

-- HNSW parameters (tune for speed vs accuracy trade-off)
-- m: max connections per layer (default 16, higher = more accurate but slower build)
-- ef_construction: build-time search width (default 64, higher = better quality)
CREATE INDEX idx_embeddings_hnsw_optimized ON document_embeddings 
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 24, ef_construction = 128);

-- Query: find top 5 most similar chunks
SELECT chunk_id, content, 
       embedding <=> '[0.1, 0.2, ...]'::vector AS distance
FROM document_embeddings
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;

-- Adjust query-time search width for speed/accuracy trade-off
SET hnsw.ef_search = 100;  -- Default 40, higher = more accurate but slower

-- Distance operators:
-- <=> : Cosine distance (1 - cosine similarity)
-- <-> : Euclidean (L2) distance
-- <#> : Negative inner product (for maximum inner product search)

-- HNSW trade-offs:
-- Pros: Fast approximate search, good recall, scales well
-- Cons: Index build time, memory usage, less accurate than exact search
```

## Index Maintenance

```sql
-- Check index bloat
SELECT 
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
    idx_scan AS index_scans,
    CASE WHEN idx_scan > 0 
        THEN pg_relation_size(i.indexrelid) / idx_scan 
        ELSE 0 
    END AS bytes_per_scan
FROM pg_stat_user_indexes i
JOIN pg_index USING (indexrelid)
WHERE idx_scan > 0
ORDER BY pg_relation_size(i.indexrelid) DESC
LIMIT 20;

-- Reindex bloated indexes
REINDEX INDEX CONCURRENTLY idx_txn_account;

-- Rebuild all indexes on a table
REINDEX TABLE CONCURRENTLY transactions;

-- Find duplicate indexes (same columns, different names)
SELECT 
    pg_size_pretty(SUM(pg_relation_size(idx))::bigint) AS total_size,
    array_agg(idx) AS indexes
FROM (
    SELECT 
        indexrelid::regclass::text AS idx,
        indrelid,
        indkey
    FROM pg_index
    JOIN pg_stat_user_indexes USING (indexrelid)
) t
GROUP BY indrelid, indkey
HAVING COUNT(*) > 1;

-- Remove unused indexes (saves space and speeds up writes)
DROP INDEX CONCURRENTLY idx_unused_example;
```

## Cross-References

- **Query Tuning**: See [query-tuning.md](query-tuning.md) for EXPLAIN analysis
- **pgvector**: See [pgvector.md](pgvector.md) for vector storage
- **Partitioning**: See [partitioning.md](partitioning.md) for partition-level indexing

## Interview Questions

1. **When would you use a BRIN index instead of a B-Tree?**
2. **How do you choose the column order in a composite B-Tree index?**
3. **What is a covering index and how does it enable index-only scans?**
4. **How does HNSW work for vector similarity search?**
5. **Your GIN index on JSONB is 10GB. Is this normal? How do you optimize it?**
6. **How do you identify and remove unused indexes without causing performance regressions?**

## Checklist: Index Strategy

- [ ] B-Tree indexes on foreign key columns
- [ ] Composite indexes ordered by selectivity (equality first, range last)
- [ ] Covering indexes for frequently accessed column combinations
- [ ] Partial indexes for selective queries
- [ ] GIN indexes on JSONB columns with containment queries
- [ ] BRIN indexes on append-only time-series tables
- [ ] HNSW indexes on embedding columns for RAG
- [ ] Regular review of unused indexes
- [ ] Index bloat monitored and reindexed as needed
- [ ] Duplicate indexes removed
- [ ] Index creation uses CONCURRENTLY to avoid locks
