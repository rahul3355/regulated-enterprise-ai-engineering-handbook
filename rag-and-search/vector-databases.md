# Vector Databases

## Overview

Vector databases (or vector stores) are specialized databases that store and search high-dimensional embedding vectors efficiently. They are the backbone of the retrieval layer in RAG systems.

```
Text Chunks --> [Embedding Model] --> Vectors (1024-3072 dimensions) --> [Vector Database]
                                                                    |
Query --> [Embedding Model] --> Query Vector --> [Vector Database] --> Similar Vectors
```

## Key Concepts

### Indexing Algorithms

| Algorithm | Type | Speed | Memory | Accuracy | Best For |
|---|---|---|---|---|---|
| **HNSW** | Graph-based | Fast | High | Excellent | General purpose, in-memory |
| **IVF** | Clustering-based | Medium | Medium | Good | Large datasets, disk-based |
| **IVF-PQ** | IVF + Product Quantization | Medium | Low | Good | Memory-constrained |
| **DiskANN** | Disk-based graph | Fast | Low | Excellent | Billion-scale datasets |
| **Flat/Brute-force** | Exact | Slow | Low | Perfect | Small datasets (< 10K vectors) |

### Distance Metrics

| Metric | Formula | Use Case |
|---|---|---|
| **Cosine Similarity** | cos(A,B) = (A.B) / (||A|| * ||B||) | Most embeddings (OpenAI, Cohere) |
| **L2 (Euclidean)** | sqrt(SUM((ai - bi)^2)) | When magnitude matters |
| **Inner Product (Dot)** | SUM(ai * bi) | When vectors are normalized = cosine |
| **Hamming Distance** | Count of different bits | Binary embeddings |

**Banking recommendation**: Most embedding models (OpenAI, BGE, E5) are optimized for cosine similarity. Use cosine unless your model specifically recommends otherwise.

## Vector Database Comparison

### Overview Matrix

| Database | Type | Open Source | Managed | HNSW | Metadata Filter | Hybrid Search | Best For |
|---|---|---|---|---|---|---|---|
| **pgvector** | Extension | Yes | No | Yes | SQL/JSONB | Via full-text | PostgreSQL teams |
| **Pinecone** | SaaS | No | Yes | Yes | Yes | Yes (Sparse) | Fastest deployment |
| **Milvus** | Standalone | Yes | Yes (Zilliz) | Yes | Yes | Yes | Large scale |
| **Weaviate** | Standalone | Yes | Yes (WCS) | Yes | Yes | Yes | Rich features |
| **Chroma** | Library | Yes | No | HNSW | Simple | No | Prototyping |
| **Qdrant** | Standalone | Yes | Yes | Yes | Yes | Yes | Performance |
| **FAISS** | Library | Yes | No | IVF/HNSW | No | No | Research/embedding only |
| **Redis VL** | Extension | Yes | Yes | HNSW | Yes | No | Redis users |

### Detailed Comparison

#### pgvector

```sql
-- Simple setup if you already have PostgreSQL
CREATE EXTENSION vector;
CREATE TABLE documents (
    id uuid PRIMARY KEY,
    content text,
    metadata jsonb,
    embedding vector(1536)
);
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);
```

**Pros**: No new infrastructure (uses existing PostgreSQL); SQL for complex filtering; ACID transactions; familiar tooling
**Cons**: Scaling requires PostgreSQL scaling (read replicas, partitioning); HNSW build is single-threaded; not as fast as dedicated vector DBs
**Cost**: Free (open source), just PostgreSQL hosting cost
**Scale**: Up to ~10M vectors per instance
**Best for**: Teams already using PostgreSQL; moderate scale; need complex SQL queries

#### Pinecone

```python
from pinecone import Pinecone

pc = Pinecone(api_key="...")
index = pc.create_index(
    name="banking-knowledge",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1")
)

index.upsert(vectors=[
    {"id": "chunk-001", "values": embedding, "metadata": {"doc_type": "policy"}}
])

results = index.query(vector=query_embedding, top_k=4, include_metadata=True)
```

**Pros**: Fully managed; no infrastructure to maintain; auto-scaling; fastest time-to-production; built-in hybrid search
**Cons**: Proprietary; vendor lock-in; higher cost at scale; less control over indexing
**Cost**: $0.075/hour for serverless + $0.04/1K writes + $0.04/1M reads
**Scale**: Billions of vectors
**Best for**: Startups; teams without ML infrastructure expertise; rapid prototyping

#### Milvus

```python
from pymilvus import connections, Collection, CollectionSchema, FieldSchema

connections.connect("localhost", "19530")

fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1536),
    FieldSchema(name="content", dtype=DataType.VARCHAR, max_length=65535),
    FieldSchema(name="metadata", dtype=DataType.JSON),
]
schema = CollectionSchema(fields, "Banking knowledge")
collection = Collection("banking_docs", schema)

# Create HNSW index
index_params = {
    "index_type": "HNSW",
    "metric_type": "COSINE",
    "params": {"M": 16, "efConstruction": 256}
}
collection.create_index("embedding", index_params)
```

**Pros**: Open source; scales to billions; rich feature set; supports multiple index types; good performance
**Cons**: Complex to deploy and operate (requires etcd, MinIO); steep learning curve; operational overhead
**Cost**: Free (open source); managed version (Zilliz Cloud) starts at $99/month
**Scale**: Billions of vectors
**Best for**: Large enterprises; need full control; billion-scale datasets

#### Weaviate

```python
import weaviate

client = weaviate.connect_to_local()

# Create collection
client.collections.create(
    name="BankingDocuments",
    vectorizer_config=Configure.Vectorizer.none(),  # Bring your own embeddings
    properties=[
        Property(name="content", data_type=DataType.TEXT),
        Property(name="doc_title", data_type=DataType.TEXT),
        Property(name="department", data_type=DataType.TEXT),
        Property(name="metadata", data_type=DataType.OBJECT),
    ]
)

# Query with hybrid search
collection = client.collections.get("BankingDocuments")
results = collection.query.hybrid(
    query="processing fee for personal loans",
    alpha=0.3,
    limit=10,
    return_properties=["content", "doc_title", "department"]
)
```

**Pros**: Rich GraphQL API; built-in hybrid search; good documentation; multi-tenancy; generative search (built-in RAG)
**Cons**: Newer ecosystem than pgvector; GraphQL may be unfamiliar; moderate scaling complexity
**Cost**: Free (open source); managed (WCS) starts at $25/month
**Scale**: Hundreds of millions of vectors
**Best for**: Teams wanting rich features out of the box; hybrid search; multi-tenancy

## Selection Decision Framework

```
Do you already run PostgreSQL at scale?
├── Yes -> pgvector (minimal new infrastructure)
└── No -> Continue

Do you need to manage your own infrastructure?
├── No (want managed) -> Pinecone (fastest) or Weaviate Cloud (more features)
└── Yes (must self-host) -> Continue

How many vectors?
├── < 10M -> pgvector or Qdrant or Weaviate
├── 10M-100M -> Milvus or Weaviate or Qdrant
└── > 100M -> Milvus (best at extreme scale)

Is hybrid search required?
├── Yes -> Weaviate (built-in) or Pinecone (sparse vectors) or pgvector (FTS)
└── No -> Any vector DB
```

## Banking-Specific Requirements

### Security

| Requirement | pgvector | Pinecone | Milvus | Weaviate |
|---|---|---|---|---|
| Encryption at rest | PostgreSQL-level | Yes | Yes (configurable) | Yes |
| Encryption in transit | TLS (PostgreSQL) | Yes | Yes (configurable) | Yes |
| RBAC | PostgreSQL roles | API keys + RBAC | RBAC | API keys + RBAC |
| VPC isolation | Yes | Yes | Yes | Yes |
| SOC 2 | PostgreSQL vendor | Yes | Zilliz: Yes | Weaviate Cloud: Yes |
| Data residency | Your server | Selectable region | Your server | Selectable region |

### Compliance

- **Audit logging**: pgvector (PostgreSQL audit) > Weaviate > Milvus > Pinecone
- **Data retention**: All support deletion; pgvector has strongest ACID guarantees
- **Access control**: pgvector (SQL GRANT/REVOKE) is most mature; Pinecone API-level RBAC
- **Backup/restore**: All support; pgvector uses standard PostgreSQL backup tools

## Scaling Strategies

### pgvector Scaling

```
Single Instance (< 5M vectors)
├── Read replicas for query load
├── Connection pooling (PgBouncer)
└── HNSW index tuning (ef_search, M)

Multiple Instances (5M-50M vectors)
├── Partition by department or date
├── Citus for distributed PostgreSQL
└── Cross-shard query routing

Large Scale (> 50M vectors)
├── Consider migrating to Milvus
├── Or use specialized vector DB
└── pgvector is not designed for 100M+ vectors
```

### Pinecone Scaling

```python
# Serverless scales automatically
# For provisioned, adjust pods
index = pc.Index("banking-knowledge")
index.configure_index(pods=4, pod_type="p2.x4")  # Scale up
```

## Cost Comparison (1M vectors, 100K queries/month)

| Database | Infrastructure | Monthly Cost | Notes |
|---|---|---|---|
| pgvector | AWS RDS db.r6g.large | ~$200 | Shared with other PostgreSQL workloads |
| Pinecone (Serverless) | Managed | ~$50 | $0.04/1M reads + storage |
| Milvus (self-hosted) | 2x EC2 + etcd + MinIO | ~$300 | Operational overhead included |
| Weaviate (self-hosted) | 2x EC2 | ~$200 | Simpler than Milvus |
| Qdrant (self-hosted) | 1x EC2 | ~$100 | Lightweight |

## Migration Between Vector Databases

```python
def migrate_vector_store(source, target, batch_size=500):
    """Migrate from one vector DB to another."""
    offset = 0
    total_migrated = 0
    
    while True:
        batch = source.get_all(offset=offset, limit=batch_size, include_vectors=True)
        if not batch:
            break
        
        # Insert into target
        target.upsert(batch)
        total_migrated += len(batch)
        offset += batch_size
        
        print(f"Migrated {total_migrated} vectors...")
    
    print(f"Migration complete: {total_migrated} vectors")
```

**Migration tips**:
1. Run both old and new databases in parallel
2. Dual-write for new documents during migration
3. Validate by comparing query results between old and new
4. Cutover during low-traffic period

## Best Practices

1. **Start with pgvector if using PostgreSQL**: Lowest barrier to entry
2. **Index after loading all data**: HNSW index building is faster on complete datasets
3. **Tune HNSW parameters**: M=16, efConstruction=256 are good defaults; increase for better accuracy
4. **Monitor index build time**: For 1M vectors, expect 10-30 minutes
5. **Use appropriate dimension**: Don't use 3072 dimensions if 768 achieves similar quality
6. **Metadata filtering impacts performance**: Index frequently-filtered fields
7. **Backup regularly**: Vector databases are stateful; have disaster recovery plans
