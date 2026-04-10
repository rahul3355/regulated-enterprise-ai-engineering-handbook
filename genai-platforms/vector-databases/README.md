# Vector Databases

This directory covers vector database selection and deployment for GenAI applications, comparing pgvector, Pinecone, Milvus, Weaviate, and other options in a banking context.

## Key Topics

- Vector database comparison for banking use cases
- pgvector for in-PostgreSQL vector storage
- Pinecone for managed vector search
- Milvus for high-scale vector search
- Weaviate for hybrid search
- Data residency and compliance considerations
- Index configuration and tuning

## Comparison Matrix

| Feature | pgvector | Pinecone | Milvus | Weaviate |
|---------|----------|----------|--------|----------|
| **Type** | PostgreSQL extension | Managed SaaS | Open source / Managed | Open source / Cloud |
| **Deployment** | Self-hosted | Cloud only | Self-hosted or Cloud | Self-hosted or Cloud |
| **Data Residency** | Full control (in existing DB) | Limited to provider regions | Full control | Full control |
| **Scalability** | Moderate (single node) | High (managed) | Very high (distributed) | High |
| **Hybrid Search** | Via full-text search | Limited | Yes | Yes (built-in) |
| **Metadata Filtering** | Full SQL | Yes | Yes | Yes |
| **Banking Recommendation** | **Primary choice** | Backup option | Evaluate for scale | Evaluate for hybrid |

## pgvector Setup (Recommended for Banking)

```sql
-- Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Create table
CREATE TABLE embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id TEXT NOT NULL,
    content TEXT NOT NULL,
    embedding VECTOR(1536),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW()
);

-- HNSW index for fast similarity search
CREATE INDEX ON embeddings
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Semantic search with metadata filtering
SELECT content, metadata,
       1 - (embedding <=> $1::vector) AS similarity
FROM embeddings
WHERE metadata->>'department' = 'compliance'
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

## Cross-References

- [../embeddings.md](../embeddings.md) — Embedding generation
- [../rag-and-search/](../rag-and-search/) — Vector search in RAG pipelines
- [../databases/](../databases/) — Database architecture patterns
- [../infrastructure/](../infrastructure/) — Infrastructure for vector DB deployment
