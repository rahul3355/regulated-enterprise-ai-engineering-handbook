# Cost Optimization

## RAG Cost Components

```
RAG Cost = Embedding Cost + Retrieval Cost + Re-Ranking Cost + LLM Generation Cost
```

| Component | Cost Driver | Typical % of Total |
|---|---|---|
| Embedding (ingestion) | Number of chunks | 5-10% |
| Embedding (query) | Number of queries | 2-5% |
| Retrieval | Vector DB hosting | 10-15% |
| Re-Ranking | Number of pairs re-ranked | 5-15% |
| LLM Generation | Input + output tokens | 50-70% |

**LLM generation is the dominant cost.** Optimizing it yields the biggest savings.

## Cost Analysis: Banking RAG Example

### Scenario: 100,000 queries/month, 500K indexed chunks

```
Ingestion (one-time):
  500K chunks x 400 tokens = 200M tokens
  OpenAI text-embedding-3-small: 200M x $0.02/M = $4.00

Query embedding (monthly):
  100K queries x 20 tokens = 2M tokens
  OpenAI text-embedding-3-small: 2M x $0.02/M = $0.04

Re-Ranking (monthly):
  100K queries x 20 docs re-ranked
  Cohere Rerank v3.5: 2M reranks x $0.0015 = $3.00

LLM Generation (monthly):
  100K queries x (500 input + 200 output) = 70M tokens
  gpt-4o-mini: 70M x $0.15/M (input) + 20M x $0.60/M (output) = $22.50
  
Total monthly: ~$25.54 (excluding vector DB hosting)
```

### Cost Comparison: Model Selection

| LLM | Input Cost/M | Output Cost/M | 100K queries/month | Quality |
|---|---|---|---|---|
| gpt-4o | $2.50 | $10.00 | $445 | Best |
| gpt-4o-mini | $0.15 | $0.60 | $22.50 | Very Good |
| Claude Haiku | $0.25 | $1.25 | $37.50 | Good |
| Claude Sonnet | $3.00 | $15.00 | $450 | Best |
| Llama 3 70B (self-hosted) | ~$0.05* | ~$0.05* | ~$7.50* | Good |
| Mistral Medium | $0.27 | $0.81 | $40.20 | Good |

*Self-hosted GPU cost estimated at $0.50/hour for A10G

## Cost Optimization Strategies

### 1. Optimize Context Window Size

Every extra token in context costs money. Minimize context while maintaining quality.

```python
def optimize_context_size(docs: list, max_context_tokens: int = 1500) -> str:
    """Select minimum context that contains the answer."""
    
    # Sort docs by relevance
    sorted_docs = sorted(docs, key=lambda d: d.metadata.get("score", 0), reverse=True)
    
    context = ""
    for doc in sorted_docs:
        new_context = context + "\n\n" + doc.page_content
        if len(tiktoken.encode(new_context)) > max_context_tokens:
            break
        context = new_context
    
    return context.strip()

# Cost impact analysis:
# 2000 token context -> 1000 token context = 50% input cost savings
```

### 2. Use the Right Model for the Task

```python
def select_llm(query: str, complexity: str) -> str:
    """Route to appropriate LLM based on query complexity."""
    
    if complexity == "simple_factual":
        return "gpt-4o-mini"    # $0.15/M input
    elif complexity == "complex_analysis":
        return "gpt-4o"          # $2.50/M input
    elif complexity == "drafting":
        return "gpt-4o-mini"     # Good enough for drafts
    elif complexity == "compliance_critical":
        return "gpt-4o"          # Highest quality for compliance
    else:
        return "gpt-4o-mini"     # Default
```

### 3. Cache Frequently Asked Questions

```python
import hashlib
import redis

redis_client = redis.Redis(host="localhost", port=6379, db=0)

def cached_rag_response(query: str, context: str, llm) -> str:
    """Cache responses to identical query+context combinations."""
    
    # For exact matches
    cache_key = f"rag:{hashlib.sha256((query + context).encode()).hexdigest()}"
    cached = redis_client.get(cache_key)
    if cached:
        return cached.decode()
    
    response = llm.generate(build_prompt(query, context))
    redis_client.setex(cache_key, 86400 * 7, response)  # 7 day TTL
    return response

def fuzzy_cached_rag_response(query: str, context: str, llm) -> str:
    """Cache responses to semantically similar queries."""
    
    # Normalize query for fuzzy matching
    normalized = query.lower().strip().rstrip("?.")
    
    # Check for similar cached queries
    similar_key = redis_client.get(f"query_map:{hashlib.md5(normalized.encode()).hexdigest()}")
    if similar_key:
        cached = redis_client.get(f"rag:{similar_key.decode()}")
        if cached:
            return cached.decode()
    
    # Generate new response
    response = llm.generate(build_prompt(query, context))
    cache_key = hashlib.sha256((query + context).encode()).hexdigest()
    redis_client.setex(f"rag:{cache_key}", 86400 * 7, response)
    redis_client.setex(f"query_map:{hashlib.md5(normalized.encode()).hexdigest()}", 
                       86400 * 7, cache_key)
    
    return response
```

**Expected cache hit rates**: 20-40% for employee-facing RAG (many repeat questions). 5-15% for customer-facing.

### 4. Batch Embedding Operations

```python
# Inefficient: One API call per chunk
for chunk in chunks:
    embedding = openai.Embedding.create(input=chunk, model="text-embedding-3-small")
    # 100 API calls for 100 chunks

# Efficient: Batch chunks
for i in range(0, len(chunks), 100):
    batch = chunks[i:i+100]
    embeddings = openai.Embedding.create(input=batch, model="text-embedding-3-small")
    # 1 API call for 100 chunks
```

**Savings**: Batching reduces API call overhead by 99%. Some providers (OpenAI) support batch inputs natively.

### 5. Reduce Re-Ranking Cost

```python
def cost_effective_reranking(query: str, retrieved: list, reranker, k: int = 4):
    """Optimize re-ranking cost."""
    
    # Only re-rank if initial retrieval confidence is moderate
    top_score = retrieved[0].metadata.get("score", 0)
    
    if top_score > 0.9:
        return retrieved[:k]  # Skip re-ranking, high confidence
    
    if top_score < 0.3:
        return retrieved[:k]  # Skip re-ranking, nothing is relevant anyway
    
    # Re-rank only when needed
    return reranker.rerank(query, retrieved[:10], top_k=k)

# Cost impact:
# Full re-ranking: 100K queries x 20 docs = 2M reranks x $0.0015 = $3,000/month
# Selective re-ranking (skip 40%): 1.2M reranks = $1,800/month
# Savings: $1,200/month
```

### 6. Dimension Reduction for Storage

```python
# Store embeddings in lower precision to reduce vector DB cost
import numpy as np

def compress_embedding(embedding: list[float]) -> bytes:
    """Compress embedding from FP32 to FP16."""
    return np.array(embedding, dtype=np.float16).tobytes()

def decompress_embedding(data: bytes) -> list[float]:
    """Decompress FP16 back to FP32."""
    return np.frombuffer(data, dtype=np.float16).astype(np.float32).tolist()

# Storage savings:
# 1536-dim FP32: 6,144 bytes per vector
# 1536-dim FP16: 3,072 bytes per vector (50% savings)
# For 500K vectors: 3GB -> 1.5GB
```

### 7. Self-Hosted vs. Cloud Cost Analysis

```
Monthly costs at 100K queries:

Cloud (OpenAI + Pinecone + Cohere):
  Embeddings:     $0.04
  Vector DB:      $50.00 (Pinecone serverless)
  Re-Ranking:     $3.00 (Cohere)
  LLM (gpt-4o-mini): $22.50
  Total:          $75.54/month

Self-Hosted (BGE + pgvector + Llama 3):
  Embeddings:     $15.00 (GPU hours for ingestion + queries)
  Vector DB:      $200.00 (RDS PostgreSQL)
  Re-Ranking:     $10.00 (GPU hours for BGE-reranker)
  LLM (Llama 3):  $150.00 (A10G GPU, always on)
  Total:          $375.00/month

Cloud wins at 100K queries/month.

Break-even at ~1M queries/month:
  Cloud:          ~$750/month
  Self-Hosted:    ~$375/month
```

**Rule of thumb**: Cloud API is cheaper below 500K-1M queries/month. Self-hosted becomes cost-effective above that.

### 8. Monitor and Alert on Cost Anomalies

```python
def monitor_daily_cost() -> dict:
    """Track daily RAG costs and alert on anomalies."""
    
    today_costs = get_costs_for_date(datetime.now().date())
    yesterday_costs = get_costs_for_date(datetime.now().date() - timedelta(days=1))
    avg_daily_cost = get_avg_daily_cost(last_days=30)
    
    report = {
        "today": today_costs,
        "yesterday": yesterday_costs,
        "avg_30d": avg_daily_cost,
        "variance_from_avg": (today_costs - avg_daily_cost) / avg_daily_cost if avg_daily_cost > 0 else 0,
    }
    
    # Alert if cost is 2x above average
    if today_costs > avg_daily_cost * 2:
        send_alert(
            f"RAG cost anomaly: ${today_costs:.2f} today vs ${avg_daily_cost:.2f} avg",
            severity="warning"
        )
    
    return report
```

## Cost Dashboard

```python
def generate_cost_dashboard() -> dict:
    """Generate comprehensive cost breakdown."""
    
    return {
        "monthly_total": calculate_monthly_cost(),
        "breakdown": {
            "embedding_ingestion": get_embedding_ingestion_cost(),
            "embedding_queries": get_embedding_query_cost(),
            "vector_db": get_vector_db_cost(),
            "reranking": get_reranking_cost(),
            "llm_input": get_llm_input_cost(),
            "llm_output": get_llm_output_cost(),
        },
        "per_query": {
            "avg_cost": calculate_monthly_cost() / get_monthly_query_count(),
            "p50_cost": get_percentile_query_cost(50),
            "p95_cost": get_percentile_query_cost(95),
        },
        "trends": {
            "cost_per_query_7d": get_cost_per_query_trend(7),
            "cost_per_query_30d": get_cost_per_query_trend(30),
        },
        "optimization_opportunities": identify_cost_optimizations(),
    }
```

## Best Practices Summary

| Practice | Savings | Effort |
|---|---|---|
| Use gpt-4o-mini instead of gpt-4o | 60-80% | Low |
| Cache common queries | 20-40% | Medium |
| Minimize context window | 20-30% | Low |
| Batch embeddings | 50% on embedding cost | Low |
| Selective re-ranking | 30-50% on rerank cost | Medium |
| Reduce retrieval K | 10-15% | Low |
| Self-host at scale (>1M queries) | 50%+ | High |
| Compress embeddings (FP16) | 50% storage | Low |
| Route by complexity | 20-40% | Medium |
| Monitor and alert | Prevents waste | Medium |

## Banking-Specific Cost Considerations

1. **Compliance responses cost more**: They need gpt-4o level quality, not gpt-4o-mini. Budget accordingly.
2. **Multi-tenant costs multiply**: Each department/region may have separate vector DB instances.
3. **Audit logging adds tokens**: Logging every query-response pair for compliance increases token count.
4. **Multilingual multiplies cost**: Each language may need separate processing.
5. **Document ingestion is a one-time cost**: Budget $5-50 for initial ingestion of 100K documents, then only incremental costs.
