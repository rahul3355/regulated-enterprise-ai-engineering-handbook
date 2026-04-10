# pgvector Deep-Dive

## Overview

pgvector is a PostgreSQL extension that adds vector similarity search capabilities to your existing PostgreSQL database. It is the most popular choice for teams already running PostgreSQL, as it requires no new infrastructure.

```
PostgreSQL + pgvector = SQL queries + ACID transactions + Vector search + Metadata filtering
```

**Key advantages for banking**:
- Leverage existing PostgreSQL expertise and infrastructure
- Complex metadata filtering with full SQL expressiveness
- ACID transactions ensure consistency
- Backup/restore uses standard PostgreSQL tools
- Row-level security integrates with vector search

## Installation

### PostgreSQL 15+ (Recommended)

```sql
-- Enable the extension
CREATE EXTENSION vector;

-- Verify installation
SELECT extversion FROM pg_extension WHERE extname = 'vector';
-- Expected: 0.5.0 or higher
```

### Docker

```yaml
version: '3.8'
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: bank_knowledge
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
```

### Docker Compose with pgvector

```yaml
services:
  postgres:
    image: ankane/pgvector:latest
    environment:
      POSTGRES_USER: bank_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: bank_knowledge
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```

## Schema Design

### Basic Table

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_chunks (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    doc_id varchar(255) NOT NULL,
    doc_title varchar(500),
    content text NOT NULL,
    metadata jsonb NOT NULL DEFAULT '{}',
    embedding vector(1536),  -- Match your embedding dimension
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now()
);

-- Index for vector similarity (HNSW)
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops);

-- Index for metadata filtering
CREATE INDEX idx_doc_id ON document_chunks(doc_id);
CREATE INDEX idx_metadata ON document_chunks USING gin(metadata);
CREATE INDEX idx_status ON document_chunks((metadata->>'status'));
```

### Advanced Schema with Partitions

For large-scale deployments (5M+ chunks), partition by department:

```sql
CREATE TABLE document_chunks (
    id uuid NOT NULL,
    doc_id varchar(255) NOT NULL,
    content text NOT NULL,
    metadata jsonb NOT NULL DEFAULT '{}',
    embedding vector(1536),
    department varchar(50) NOT NULL,
    created_at timestamptz DEFAULT now()
) PARTITION BY LIST (department);

-- Create partitions
CREATE TABLE chunks_retail PARTITION OF document_chunks
    FOR VALUES IN ('retail_banking', 'personal_loans', 'credit_cards');
CREATE TABLE chunks_corporate PARTITION OF document_chunks
    FOR VALUES IN ('corporate_banking', 'commercial_loans');
CREATE TABLE chunks_compliance PARTITION OF document_chunks
    FOR VALUES IN ('compliance', 'aml', 'kyc', 'regulatory');
CREATE TABLE chunks_default PARTITION OF document_chunks DEFAULT;

-- Each partition gets its own HNSW index
CREATE INDEX ON chunks_retail USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON chunks_corporate USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON chunks_compliance USING hnsw (embedding vector_cosine_ops);
```

## HNSW Index

### How HNSW Works

Hierarchical Navigable Small World (HNSW) builds a multi-layer graph where:
- **Top layers**: Few nodes, long-distance connections (coarse navigation)
- **Bottom layers**: Many nodes, short-distance connections (fine-grained search)
- **Search**: Start at top layer, greedily move to nearest neighbor, descend layers

```
Layer 3:  A -------- B -------- C          (few nodes, long jumps)
Layer 2:  A -- D -- B -- E -- C -- F       (more nodes)
Layer 1:  A-D-G-H-B-E-I-J-C-F-K-L         (all nodes, short jumps)
```

### HNSW Parameters

| Parameter | Description | Default | Recommended | Tradeoff |
|---|---|---|---|---|
| **M** | Max connections per node | 16 | 16-32 | Higher = better quality, more memory |
| **ef_construction** | Candidates during index build | 64 | 256-512 | Higher = better index quality, slower build |
| **ef_search** | Candidates during search | 40 | 100-200 | Higher = better recall, slower search |

### Creating HNSW Index

```sql
-- Default HNSW index
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops);

-- Custom M and ef_construction
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 32, ef_construction = 512);
```

### Memory Requirements

```
HNSW index size (bytes) = n_vectors * M * 4 * sizeof(float) + overhead
                         = n_vectors * M * 16 + overhead

For 1M vectors, M=16:
  Index size = 1,000,000 * 16 * 16 + overhead ≈ 256 MB + overhead

For 5M vectors, M=32:
  Index size = 5,000,000 * 32 * 16 + overhead ≈ 2.5 GB + overhead
```

**Tuning rule**: Set `effective_cache_size` and `maintenance_work_mem` appropriately:

```sql
-- In postgresql.conf
maintenance_work_mem = '2GB'     -- For index building
effective_cache_size = '8GB'     -- For query planning
work_mem = '256MB'               -- For sort operations
shared_buffers = '4GB'           -- 25% of RAM for dedicated DB server
```

## Querying

### Basic Similarity Search

```sql
-- Find top 5 most similar chunks
SELECT id, doc_title, content, metadata,
       embedding <=> $1 as distance
FROM document_chunks
ORDER BY embedding <=> $1
LIMIT 5;
```

### Search with Metadata Filter

```sql
-- Hybrid: metadata filter + vector search
SELECT id, doc_title, content, metadata,
       embedding <=> $1 as distance
FROM document_chunks
WHERE metadata->>'status' = 'active'
  AND metadata->>'department' = ANY($2)
  AND (metadata->>'min_clearance')::int <= $3
ORDER BY embedding <=> $1
LIMIT 5;
```

### BM25 Full-Text Search + Vector (Hybrid)

```sql
-- Combined hybrid search with score fusion
WITH bm25_results AS (
    SELECT id, content, metadata,
           ts_rank(to_tsvector('english', content), 
                   plainto_tsquery('english', $1)) as bm25_score,
           RANK() OVER (ORDER BY ts_rank(...) DESC) as bm25_rank
    FROM document_chunks
    WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $1)
      AND metadata->>'status' = 'active'
    LIMIT 50
),
vector_results AS (
    SELECT id, content, metadata,
           embedding <=> $2::vector as vector_distance,
           RANK() OVER (ORDER BY embedding <=> $2) as vector_rank
    FROM document_chunks
    WHERE metadata->>'status' = 'active'
    ORDER BY embedding <=> $2
    LIMIT 50
)
SELECT COALESCE(b.id, v.id) as id,
       COALESCE(b.content, v.content) as content,
       COALESCE(b.metadata, v.metadata) as metadata,
       -- RRF (Reciprocal Rank Fusion)
       (1.0 / (60 + COALESCE(b.bm25_rank, 999))) +
       (1.0 / (60 + COALESCE(v.vector_rank, 999))) as rrf_score
FROM bm25_results b
FULL OUTER JOIN vector_results v ON b.id = v.id
ORDER BY rrf_score DESC
LIMIT 5;
```

### KNN with IVFFlat (for very large datasets)

```sql
-- IVF index (faster build, slightly less accurate than HNSW)
CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 1000);  -- sqrt(n) is a good starting point

-- Set probes for search (higher = more accurate, slower)
SET ivfflat.probes = 20;

-- Query (same as HNSW)
SELECT id, content, metadata,
       embedding <=> $1 as distance
FROM document_chunks
ORDER BY embedding <=> $1
LIMIT 5;
```

## Performance Tuning

### Configuration

```sql
-- postgresql.conf tuning for pgvector

# Memory settings
shared_buffers = '4GB'           -- 25% of total RAM
effective_cache_size = '12GB'    -- 75% of total RAM
maintenance_work_mem = '2GB'     -- For index creation
work_mem = '256MB'               -- Per-operation memory
huge_pages = on                   -- Use huge pages

# HNSW-specific
-- ef_search is set per-session or per-query
SET hnsw.ef_search = 200;

# Parallelism
max_parallel_workers = 8
max_parallel_workers_per_gather = 4

# Autovacuum tuning (for frequently updated tables)
autovacuum_vacuum_scale_factor = 0.05   -- Vacuum after 5% dead tuples
autovacuum_analyze_scale_factor = 0.02  -- Analyze after 2% changes
```

### Query Optimization

```sql
-- Use EXPLAIN ANALYZE to understand query plans
EXPLAIN ANALYZE
SELECT id, content, metadata,
       embedding <=> $1 as distance
FROM document_chunks
WHERE metadata->>'status' = 'active'
ORDER BY embedding <=> $1
LIMIT 5;
```

Expected output with HNSW:
```
Limit  (cost=0.42..12.34 rows=5 width=1234)
  ->  Index Scan using document_chunks_embedding_idx on document_chunks
        (cost=0.42..12345.67 rows=5123 width=1234)
        Order By: (embedding <=> '...'::vector)
        Index Cond: (metadata->>'status' = 'active')
```

If you see `Seq Scan` instead of `Index Scan`:
- Check that the HNSW index exists
- Ensure enough rows to make index worthwhile
- Run `ANALYZE document_chunks` to update statistics

### Handling Metadata Filtering + Vector Search

PostgreSQL may not use the vector index when metadata filters are very selective. Solution:

```sql
-- Option 1: Use a partial index
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops)
    WHERE metadata->>'status' = 'active';

-- Option 2: Create department-specific indexes
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops)
    WHERE metadata->>'department' = 'retail_banking';

-- Option 3: Use CTE to pre-filter
WITH active_docs AS (
    SELECT id, embedding
    FROM document_chunks
    WHERE metadata->>'status' = 'active'
      AND metadata->>'department' = 'retail_banking'
)
SELECT id, embedding <=> $1 as distance
FROM active_docs
ORDER BY distance
LIMIT 5;
```

## Bulk Insertion

```python
import psycopg2
from psycopg2.extras import execute_values

def bulk_insert_chunks(chunks: list[dict], connection, batch_size=500):
    """Efficiently insert chunks in batches."""
    
    with connection.cursor() as cur:
        query = """
            INSERT INTO document_chunks (id, doc_id, doc_title, content, metadata, embedding)
            VALUES %s
            ON CONFLICT (id) DO NOTHING
        """
        
        for i in range(0, len(chunks), batch_size):
            batch = chunks[i:i + batch_size]
            values = [
                (
                    c["id"], c["doc_id"], c["doc_title"],
                    c["content"], json.dumps(c["metadata"]),
                    c["embedding"]
                )
                for c in batch
            ]
            execute_values(cur, query, values)
            connection.commit()
            
        print(f"Inserted {len(chunks)} chunks")
```

## Reindexing

When the document corpus changes significantly, rebuild the index:

```sql
-- Concurrent reindex (doesn't block reads/writes)
CREATE INDEX CONCURRENTLY document_chunks_embedding_idx_new
    ON document_chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 32, ef_construction = 512);

-- Swap indexes
BEGIN;
DROP INDEX document_chunks_embedding_idx;
ALTER INDEX document_chunks_embedding_idx_new 
    RENAME TO document_chunks_embedding_idx;
COMMIT;

-- Or simply:
REINDEX INDEX CONCURRENTLY document_chunks_embedding_idx;
```

## Monitoring

```sql
-- Index size
SELECT pg_size_pretty(pg_relation_size('document_chunks_embedding_idx'));

-- Table size
SELECT pg_size_pretty(pg_total_relation_size('document_chunks'));

-- Index usage statistics
SELECT idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE indexname = 'document_chunks_embedding_idx';

-- HNSW index statistics (pgvector 0.5+)
SELECT * FROM pg_stat_progress_create_index
WHERE index_relid = 'document_chunks_embedding_idx'::regclass;

-- Query performance
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
WHERE query LIKE '%document_chunks%'
ORDER BY total_time DESC;
```

## Row-Level Security

```sql
-- Enable row-level security
ALTER TABLE document_chunks ENABLE ROW LEVEL SECURITY;

-- Create policy based on user role
CREATE POLICY department_access ON document_chunks
    USING (
        metadata->>'department' = ANY(
            SELECT unnest(current_user_departments())
        )
    );

-- Create function to get user's departments
CREATE OR REPLACE FUNCTION current_user_departments()
RETURNS text[] AS $$
    SELECT ARRAY['retail_banking', 'wealth_management']
    -- In practice, look up from user session or JWT
$$ LANGUAGE SQL;
```

## Backup and Restore

```bash
# Backup
pg_dump -h localhost -U bank_user -d bank_knowledge \
    -t document_chunks --inserts > backup.sql

# Restore
psql -h localhost -U bank_user -d bank_knowledge < backup.sql

# Logical backup with pg_dump (includes indexes)
pg_dump -h localhost -U bank_user -d bank_knowledge -Fc > backup.dump
pg_restore -h localhost -U bank_user -d bank_knowledge backup.dump
```

## Common Issues

### Issue: Index build is slow

```sql
-- Increase maintenance work memory
SET maintenance_work_mem = '4GB';

-- Increase parallelism (for IVF, not HNSW)
SET max_parallel_maintenance_workers = 4;

-- For HNSW, reduce M and ef_construction
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 256);
```

### Issue: Queries are slow

```sql
-- Increase ef_search for better recall (at cost of speed)
SET hnsw.ef_search = 200;

-- Or decrease for faster (less accurate) search
SET hnsw.ef_search = 50;

-- Ensure autovacuum is running
SELECT last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'document_chunks';
```

### Issue: Out of memory

```sql
-- HNSW index too large for RAM
-- Options:
-- 1. Use IVFFlat instead (less memory)
CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 500);

-- 2. Reduce embedding dimension (PCA)
-- 3. Upgrade server RAM
-- 4. Partition the table
```
