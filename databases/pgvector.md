# pgvector for Embedding Storage and Retrieval

## Overview

pgvector is a PostgreSQL extension that enables vector similarity search, making Postgres a viable option for storing and querying GenAI embeddings. This guide covers installation, index types (HNSW, IVFFlat), query patterns, and production considerations for banking RAG systems.

## Setup and Configuration

```sql
-- Install pgvector (PostgreSQL 13+)
-- Ubuntu/Debian: apt install postgresql-15-pgvector
-- Docker: pgvector/pgvector:pg15 image

-- Enable extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify installation
SELECT * FROM pg_extension WHERE extname = 'vector';

-- Vector type supports 1 to 16,000 dimensions
-- Common embedding models:
-- text-embedding-3-small: 1536 dimensions
-- text-embedding-3-large: 3072 dimensions (or 1536, 1024, 512, 256)
-- text-embedding-ada-002: 1536 dimensions
```

## Storing Embeddings

```sql
-- Document chunks with embeddings
CREATE TABLE document_chunks (
    chunk_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),
    metadata JSONB DEFAULT '{}',
    embedding_model VARCHAR(50),
    embedding_version VARCHAR(20),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(document_id, chunk_index)
);

-- Insert embedding
INSERT INTO document_chunks (chunk_id, document_id, chunk_index, content, embedding, metadata)
VALUES (
    gen_random_uuid(),
    'doc_001',
    0,
    'Interest rates for savings accounts start at 4.5% APY...',
    '[0.023, -0.045, 0.012, ...]'::vector,  -- 1536-dim vector
    '{"doc_type": "product_info", "source": "products.pdf"}'
);

-- Bulk insert embeddings
-- Use COPY or psycopg2's execute_values for performance
```

## HNSW Index for Fast Similarity Search

```sql
-- HNSW (Hierarchical Navigable Small World) index
-- Best for: High recall, moderate dataset size (< 10M vectors)

-- Create HNSW index
CREATE INDEX ON document_chunks 
USING hnsw (embedding vector_cosine_ops);

-- HNSW parameters (tune for speed/accuracy trade-off)
-- m: max connections per layer (default 16, range 2-100)
-- Higher m = better recall but larger index and slower build
CREATE INDEX ON document_chunks 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 24, ef_construction = 128);

-- Query-time search width (ef_search)
-- Higher = better recall but slower queries
SET hnsw.ef_search = 100;  -- Default is 40

-- Cosine similarity query (most common for text embeddings)
SELECT 
    chunk_id,
    content,
    metadata,
    1 - (embedding <=> '[0.023, -0.045, ...]'::vector) AS similarity
FROM document_chunks
ORDER BY embedding <=> '[0.023, -0.045, ...]'::vector
LIMIT 5;

-- Combined: Vector search + metadata filter
SELECT 
    chunk_id,
    content,
    metadata->>'doc_type' AS doc_type,
    1 - (embedding <=> $1::vector) AS similarity
FROM document_chunks
WHERE metadata @> '{"doc_type": "product_info"}'::jsonb
ORDER BY embedding <=> $1::vector
LIMIT 5;
```

## IVFFlat Index

```sql
-- IVFFlat (Inverted File with Flat Quantization)
-- Best for: Very large datasets, faster build time

-- Requires training data (lists built from existing vectors)
-- First, insert some data
-- Then create the index

-- Number of lists: sqrt(N) where N = number of vectors
-- For 1M vectors: ~1000 lists
CREATE INDEX ON document_chunks 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 1000);

-- Query-time probes (higher = better recall, slower)
SET ivfflat.probes = 10;  -- Default is 1

-- IVFFlat requires vectors in the table before creation
-- HNSW does not (can create on empty table)
```

## Distance Operators

```sql
-- <=> : Cosine distance (1 - cosine similarity)
-- Best for: Text embeddings where direction matters more than magnitude
SELECT embedding <=> '[...]'::vector AS cosine_distance FROM document_chunks;

-- <-> : Euclidean (L2) distance
-- Best for: Embeddings where magnitude is meaningful
SELECT embedding <-> '[...]'::vector AS l2_distance FROM document_chunks;

-- <#> : Negative inner product
-- Best for: Maximum inner product search (equivalent to cosine for normalized vectors)
SELECT embedding <#> '[...]'::vector AS neg_inner_product FROM document_chunks;

-- For RAG with OpenAI embeddings, use cosine distance (<=>)
-- OpenAI embeddings are normalized, so cosine similarity = inner product
```

## RAG Query Pattern

```sql
-- Complete RAG retrieval query with hybrid scoring
WITH vector_results AS (
    SELECT 
        chunk_id,
        document_id,
        content,
        metadata,
        1 - (embedding <=> $1::vector) AS vector_score,
        1 AS source_weight  -- Vector search weight
    FROM document_chunks
    WHERE metadata @> $2::jsonb  -- Metadata filter
    ORDER BY embedding <=> $1::vector
    LIMIT 20
),
keyword_results AS (
    SELECT 
        chunk_id,
        document_id,
        content,
        metadata,
        ts_rank(
            to_tsvector('english', content),
            plainto_tsquery('english', $3)
        ) AS keyword_score,
        0.5 AS source_weight  -- Keyword search weight
    FROM document_chunks
    WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $3)
    LIMIT 20
),
combined AS (
    SELECT 
        COALESCE(v.chunk_id, k.chunk_id) AS chunk_id,
        COALESCE(v.document_id, k.document_id) AS document_id,
        COALESCE(v.content, k.content) AS content,
        COALESCE(v.metadata, k.metadata) AS metadata,
        COALESCE(v.vector_score, 0) * COALESCE(v.source_weight, 0) +
        COALESCE(k.keyword_score, 0) * COALESCE(k.source_weight, 0) AS combined_score
    FROM vector_results v
    FULL OUTER JOIN keyword_results k USING (chunk_id)
)
SELECT chunk_id, content, metadata, combined_score
FROM combined
ORDER BY combined_score DESC
LIMIT 5;
```

## Production Considerations

```sql
-- Monitor index size
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'document_chunks'
  AND indexname LIKE '%embedding%';

-- HNSW index build time grows with data volume
-- For 1M vectors with m=24: ~30 minutes
-- Use CONCURRENTLY to avoid blocking
CREATE INDEX CONCURRENTLY ON document_chunks 
USING hnsw (embedding vector_cosine_ops);

-- WAL growth with vector updates
-- Each vector update generates WAL entries
-- Monitor WAL size during bulk embedding operations
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/00000000'::pg_lsn));

-- VACUUM after bulk deletes
VACUUM ANALYZE document_chunks;
```

## Cross-References

- **Embedding Pipelines**: See [embedding-pipelines.md](../data-engineering/embedding-pipelines.md) for generation
- **Indexing**: See [indexing.md](indexing.md) for HNSW index details
- **Vector Databases**: See [vector-databases.md](vector-databases.md) for comparison

## Interview Questions

1. **How does pgvector compare to dedicated vector databases like Milvus or Pinecone?**
2. **What is the difference between HNSW and IVFFlat indexes? When do you use each?**
3. **How do you tune HNSW parameters for the best speed/accuracy trade-off?**
4. **Your vector similarity query is slow on 5M documents. What do you check?**
5. **How do you handle embedding model version changes with pgvector?**
6. **Can you combine vector search with metadata filtering efficiently?**

## Checklist: pgvector Production Readiness

- [ ] pgvector extension installed and enabled
- [ ] Embedding column dimension matches model output
- [ ] HNSW or IVFFlat index created on embedding column
- [ ] Index built with CONCURRENTLY to avoid locks
- [ ] ef_search/probes tuned for recall requirements
- [ ] Combined vector + keyword search implemented
- [ ] Metadata filtering with GIN index on JSONB
- [ ] Index size monitored
- [ ] WAL growth monitored during bulk operations
- [ ] Embedding version tracked for each document
