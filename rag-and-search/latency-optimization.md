# Latency Optimization

## Latency Budget in RAG

A typical RAG request has these components:

```
User Query --> [Query Rewriting: 200-800ms] --> [Embedding: 50-200ms] 
           --> [Retrieval: 30-100ms] --> [Re-ranking: 100-300ms] 
           --> [Context Assembly: 5ms] --> [LLM Generation: 500-3000ms]
           --> Response
```

**Total P95 target**: < 3000ms for banking RAG

| Component | P50 | P95 | Optimization Impact |
|---|---|---|---|
| Query rewriting | 200ms | 600ms | High (can skip) |
| Embedding | 30ms | 100ms | Low |
| Retrieval | 20ms | 80ms | Medium |
| Re-ranking | 80ms | 250ms | Medium |
| LLM generation | 800ms | 2500ms | Very High |
| **Total** | **1130ms** | **3530ms** | |

## Optimization Strategies by Component

### 1. Query Rewriting Optimization

**Problem**: LLM-based query rewriting adds 200-800ms.

**Solutions**:

```python
# Option A: Skip rewriting for simple queries
def smart_query_rewriting(query: str, llm) -> list[str]:
    """Only rewrite complex queries."""
    
    # Simple heuristic: short queries with clear intent don't need rewriting
    if len(query.split()) <= 4 and not any(c in query for c in ["?", "and", "or", "vs"]):
        return [query]  # Skip rewriting
    
    # Cache common query patterns
    cache_key = hashlib.md5(query.lower().encode()).hexdigest()
    cached = redis.get(f"query_rewrite:{cache_key}")
    if cached:
        return json.loads(cached)
    
    # Rewrite for complex queries
    variations = expand_query(query, llm)
    redis.setex(f"query_rewrite:{cache_key}", 3600, json.dumps(variations))
    
    return variations

# Option B: Use a smaller model for rewriting
# gpt-4o-mini instead of gpt-4o for query rewriting saves ~300ms

# Option C: Parallel rewriting
async def parallel_rewrite(query: str, llm) -> list[str]:
    """Generate variations in parallel."""
    tasks = [
        llm.generate_async(f"Rewrite: {query}. Use synonyms."),
        llm.generate_async(f"Rewrite: {query}. Use banking terminology."),
        llm.generate_async(f"Rewrite: {query}. Make it more specific."),
    ]
    results = await asyncio.gather(*tasks)
    return [query] + [r.strip() for r in results if r.strip()]
```

### 2. Embedding Optimization

**Problem**: Embedding adds 30-200ms per query.

**Solutions**:

```python
# Option A: Self-hosted embedding model (no network latency)
# ONNX-optimized model on CPU can embed in < 10ms

from optimum.onnxruntime import ORTModelForFeatureExtraction
from transformers import AutoTokenizer

class LocalEmbedder:
    def __init__(self, model_name="BAAI/bge-large-en-v1.5"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = ORTModelForFeatureExtraction.from_pretrained(
            model_name, 
            provider="CPUExecutionProvider"
        )
    
    def embed(self, text: str) -> list[float]:
        inputs = self.tokenizer(text, return_tensors="pt", truncation=True, max_length=512)
        outputs = self.model(**inputs)
        # Mean pooling
        return outputs.last_hidden_state.mean(dim=1).squeeze().tolist()

# Latency: ~10ms on CPU (vs 50-200ms for API call)

# Option B: Batch embeddings
def embed_batch(texts: list[str], embedder, batch_size: int = 32) -> list[list[float]]:
    """Embed multiple texts in a single batch call."""
    all_embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        embeddings = embedder.encode_batch(batch)  # Single API call
        all_embeddings.extend(embeddings)
    return all_embeddings
```

### 3. Retrieval Optimization

**Problem**: Vector search can be slow for large databases.

**Solutions**:

```python
# Option A: Tune HNSW parameters
# Lower ef_search for faster (less accurate) search
# SET hnsw.ef_search = 50;  -- Fast, less accurate
# SET hnsw.ef_search = 200; -- Slower, more accurate

# Option B: Partition by department
# Instead of searching all documents, route to department-specific index
def partitioned_search(query: str, department: str, vectorstore, k: int = 4):
    """Search only within relevant department partition."""
    # Smaller index = faster search
    return vectorstore.similarity_search(query, k=k, filter={"department": department})

# Option C: Approximate search with IVF
# IVF is faster to build and search than HNSW for very large datasets
# Trade-off: ~5% accuracy loss for 2x speedup

# Option D: Pre-computed query cache
CACHE = {}

def cached_search(query: str, vectorstore, k: int = 4):
    """Cache results for common queries."""
    # Normalize query for caching
    normalized = query.lower().strip().rstrip('?')
    
    if normalized in CACHE:
        return CACHE[normalized]
    
    results = vectorstore.similarity_search(query, k=k)
    
    # Cache only high-frequency queries
    CACHE[normalized] = results
    if len(CACHE) > 10000:
        # Evict least recently used
        evict_oldest(CACHE)
    
    return results
```

### 4. Re-Ranking Optimization

**Problem**: Re-ranking 20-50 documents adds 100-300ms.

**Solutions**:

```python
# Option A: Reduce re-rank candidate count
# Re-rank top-10 instead of top-20 (saves ~50% re-rank time)
# Quality impact: ~2-3% precision loss

# Option B: Batched re-ranking
def batched_rerank(query: str, documents: list[str], reranker, 
                   batch_size: int = 16) -> list:
    """Re-rank in batches for better GPU utilization."""
    # Instead of one call per doc, batch them
    pairs = [(query, doc) for doc in documents]
    return reranker.rerank_batch(pairs, batch_size=batch_size)

# Option C: Two-stage re-ranking
# First pass: fast, lightweight re-ranker
# Second pass: accurate re-ranker on top-10 only
def two_stage_rerank(query: str, documents: list[str]) -> list:
    # Stage 1: Fast re-rank (small model)
    stage1_scores = fast_reranker.score(query, documents)
    top_10_indices = np.argsort(stage1_scores)[-10:][::-1]
    top_10_docs = [documents[i] for i in top_10_indices]
    
    # Stage 2: Accurate re-rank on top-10
    stage2_results = accurate_reranker.rerank(query, top_10_docs, top_k=4)
    
    return stage2_results

# Option D: Skip re-ranking for high-confidence retrieval
def conditional_rerank(query: str, retrieved: list, reranker, k: int = 4):
    """Skip re-ranking if initial retrieval is high confidence."""
    if len(retrieved) == 0:
        return []
    
    top_score = retrieved[0].metadata.get("score", 0)
    if top_score > 0.85:  # High confidence
        return retrieved[:k]  # Skip re-ranking
    
    return reranker.rerank(query, retrieved, top_k=k)
```

### 5. LLM Generation Optimization

**Problem**: LLM generation is the largest latency component (500-3000ms).

**Solutions**:

```python
# Option A: Use a faster model
# gpt-4o-mini is 2-3x faster than gpt-4o with acceptable quality for policy Q&A

# Option B: Streaming responses
def stream_response(query: str, context: str, llm):
    """Stream response token by token for better perceived latency."""
    prompt = build_prompt(query, context)
    
    first_token_time = None
    for chunk in llm.stream(prompt):
        if first_token_time is None:
            first_token_time = time.time()
        yield chunk.content
    
    # Log time-to-first-token (TTFT)
    ttft = first_token_time - start_time
    log_metric("ttft_ms", ttft * 1000)

# Time to first token (TTFT) is what users perceive as latency
# Target TTFT: < 500ms

# Option C: Response caching
def cached_llm_response(query_hash: str, context_hash: str, llm, prompt: str):
    """Cache LLM responses for identical queries and context."""
    cache_key = f"llm:{query_hash}:{context_hash}"
    
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    response = llm.generate(prompt)
    redis.setex(cache_key, 86400, json.dumps(response))  # 24 hour TTL
    return response

# Option D: Constrained generation (max tokens)
# Limit response length to cap generation time
llm = ChatOpenAI(
    model="gpt-4o-mini",
    max_tokens=500,  # Cap response length
    temperature=0.0,
)

# Option E: Speculative decoding (advanced)
# Use a small draft model to generate tokens, verify with large model
# 2-3x speedup for generation
```

## End-to-End Latency Optimization

### Parallel Pipeline Execution

```python
async def parallel_rag_pipeline(query: str, user: User) -> dict:
    """Run independent pipeline stages in parallel."""
    
    # Stage 1: Query rewriting + embedding (can be parallel)
    rewritten_task = asyncio.create_task(rewrite_query_async(query))
    embedding_task = asyncio.create_task(embed_async(query))
    
    rewritten_queries, query_embedding = await asyncio.gather(
        rewritten_task, embedding_task
    )
    
    # Stage 2: Retrieval (can parallelize BM25 + vector)
    bm25_task = asyncio.create_task(bm25_search_async(query))
    vector_task = asyncio.create_task(vector_search_async(query_embedding))
    
    bm25_results, vector_results = await asyncio.gather(
        bm25_task, vector_task
    )
    
    # Stage 3: Combine, re-rank, generate
    combined = combine_results(bm25_results, vector_results)
    reranked = reranker.rerank(query, combined, top_k=4)
    context = assemble_context(reranked)
    
    # Stage 4: Generate with streaming
    response = await llm.generate_async_stream(context, query)
    
    return {"response": response, "sources": reranked}
```

### Latency Monitoring

```python
import time
from contextlib import contextmanager

@contextmanager
def timed_stage(stage_name: str):
    """Context manager for timing pipeline stages."""
    start = time.time()
    try:
        yield
    finally:
        duration = time.time() - start
        log_metric(f"latency.{stage_name}_ms", duration * 1000)

def rag_pipeline_with_timing(query: str) -> dict:
    """Pipeline with per-stage latency tracking."""
    
    with timed_stage("query_rewrite"):
        rewritten = rewrite_query(query)
    
    with timed_stage("embedding"):
        query_embedding = embed(query)
    
    with timed_stage("retrieval"):
        retrieved = vector_search(query_embedding, k=20)
    
    with timed_stage("reranking"):
        reranked = reranker.rerank(query, retrieved, top_k=4)
    
    with timed_stage("generation"):
        response = llm.generate(build_prompt(query, reranked))
    
    return response
```

### SLO Targets

| Metric | Target | Alert Threshold |
|---|---|---|
| P50 end-to-end | < 1500ms | > 2000ms |
| P95 end-to-end | < 3000ms | > 5000ms |
| P99 end-to-end | < 5000ms | > 8000ms |
| Time to first token | < 500ms | > 1000ms |
| Retrieval P95 | < 200ms | > 500ms |
| Re-ranking P95 | < 300ms | > 500ms |

## Latency vs. Quality Tradeoffs

| Optimization | Latency Savings | Quality Impact | Recommended? |
|---|---|---|---|
| Skip query rewriting | -400ms | -10% precision | For simple queries only |
| Self-hosted embeddings | -100ms | 0% (same model) | Always |
| Reduce retrieval K (20->10) | -30ms | -3% precision | If re-ranker is used |
| Skip re-rank for high confidence | -150ms | -2% precision | Yes, with threshold > 0.85 |
| Use gpt-4o-mini vs gpt-4o | -1000ms | -5% quality | For policy Q&A, yes |
| Response caching | -2000ms (cache hit) | 0% (identical) | For common queries |
| Limit max tokens to 300 | -500ms | May truncate long answers | Only if answers are short |

## Production Latency Debugging

When latency spikes occur, debug systematically:

```python
def debug_latency_spike(trace_id: str) -> dict:
    """Analyze a slow request trace."""
    trace = get_trace(trace_id)
    
    stages = trace["stages"]
    total = sum(s["duration_ms"] for s in stages)
    
    # Find the bottleneck
    bottleneck = max(stages, key=lambda s: s["duration_ms"])
    
    # Compare with baseline
    baseline = get_baseline_latencies()
    
    slow_stages = []
    for stage in stages:
        p95_baseline = baseline[stage["name"]]["p95"]
        if stage["duration_ms"] > p95_baseline * 1.5:
            slow_stages.append({
                "stage": stage["name"],
                "actual_ms": stage["duration_ms"],
                "baseline_p95_ms": p95_baseline,
                "ratio": stage["duration_ms"] / p95_baseline
            })
    
    return {
        "total_latency_ms": total,
        "bottleneck": bottleneck["name"],
        "slow_stages": slow_stages,
        "stages": {s["name"]: s["duration_ms"] for s in stages}
    }
```
