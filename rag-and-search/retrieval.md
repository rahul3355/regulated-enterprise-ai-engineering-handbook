# Retrieval Strategies

## Overview

Retrieval is the heart of RAG. The retriever takes a user query and finds the most relevant documents from the knowledge base. The quality of retrieval directly determines the quality of the final response -- no LLM can compensate for irrelevant context.

```
User Query --> [Embed] --> [Search Vector DB] --> [Top-K Chunks] --> [Re-rank] --> [Select Top-K'] --> Context
```

## Retrieval Strategies

### 1. Simple Top-K Retrieval

The baseline approach: embed the query, find the K most similar vectors.

```python
def simple_retrieval(query: str, vectorstore, k: int = 4) -> list:
    """Basic similarity search."""
    query_embedding = embed(query)
    results = vectorstore.similarity_search_by_vector(
        query_embedding, k=k
    )
    return results
```

**Pros**: Simple, fast, works for 80% of cases
**Cons**: Fails on complex/multi-part queries; sensitive to embedding quality
**When to use**: Prototyping, simple Q&A, small document collections (< 10K chunks)

### 2. Hybrid Retrieval (BM25 + Vector)

Combine keyword-based search (BM25) with semantic search (vector similarity).

```python
def hybrid_retrieval(query: str, k: int = 10, 
                     alpha: float = 0.5) -> list:
    """Combine BM25 and vector search with weighted scoring."""
    # BM25 (keyword) search
    bm25_results = bm25_search(query, k=k)  # [(score, doc), ...]
    
    # Vector (semantic) search  
    vector_results = vector_search(query, k=k)  # [(score, doc), ...]
    
    # Normalize scores to [0, 1]
    bm25_normalized = normalize_scores(bm25_results)
    vector_normalized = normalize_scores(vector_results)
    
    # Merge with weighted combination
    combined = {}
    for doc_id, score in bm25_normalized:
        combined[doc_id] = alpha * score
    for doc_id, score in vector_normalized:
        combined[doc_id] = combined.get(doc_id, 0) + (1 - alpha) * score
    
    # Sort and return top-k
    sorted_results = sorted(combined.items(), key=lambda x: x[1], reverse=True)
    return [doc_id for doc_id, _ in sorted_results[:k]]
```

**Why hybrid works**: BM25 catches exact keyword matches that embeddings might miss (product codes, specific policy numbers, named entities). Vector search catches semantic matches that keyword search misses.

**Alpha tuning**: In banking, alpha = 0.3-0.4 often works best (favor vector search slightly). For documents with lots of specific terms (contract clauses), alpha = 0.5-0.6.

**Banking recommendation**: Always use hybrid search in production. It is the single biggest improvement over pure vector search.

### 3. Multi-Query Retrieval

Expand the original query into multiple variations, retrieve from each, and deduplicate.

```python
def multi_query_retrieval(query: str, llm, vectorstore, 
                          k_per_query: int = 4) -> list:
    """Generate query variations and retrieve from each."""
    
    # Generate variations
    prompt = f"""Generate 3 alternative versions of this query.
    Each should capture the same intent but use different wording.
    Original: {query}
    
    Return only the 3 queries, one per line."""
    
    variations = llm.generate(prompt).strip().split("\n")
    all_queries = [query] + [v.strip() for v in variations if v.strip()]
    
    # Retrieve from each query
    all_results = {}
    for q in all_queries:
        results = vectorstore.similarity_search(q, k=k_per_query)
        for doc in results:
            doc_id = doc.metadata.get("id")
            if doc_id not in all_results:
                all_results[doc_id] = {
                    "doc": doc,
                    "queries_matched": [],
                    "max_score": 0
                }
            all_results[doc_id]["queries_matched"].append(q)
    
    # Sort by number of queries that matched (consensus)
    sorted_results = sorted(
        all_results.values(),
        key=lambda x: len(x["queries_matched"]),
        reverse=True
    )
    
    return [r["doc"] for r in sorted_results[:k_per_query]]
```

**When to use**: Complex queries with multiple concepts, ambiguous queries, when initial retrieval fails.

**Cost**: 3-4x LLM calls for query generation. Use a small model (gpt-4o-mini) to minimize cost.

### 4. Query Expansion with Sub-Questions

Decompose complex queries into sub-questions, retrieve for each, and combine.

```python
def sub_question_retrieval(query: str, llm, vectorstore) -> list:
    """Decompose query into sub-questions and retrieve for each."""
    
    prompt = f"""Break down this question into 2-4 sub-questions.
    Each sub-question should focus on a specific aspect.
    
    Question: {query}
    
    Return sub-questions as a numbered list."""
    
    sub_questions = parse_numbered_list(llm.generate(prompt))
    
    all_docs = {}
    for sq in sub_questions:
        results = vectorstore.similarity_search(sq, k=3)
        for doc in results:
            doc_id = doc.metadata["id"]
            if doc_id not in all_docs:
                all_docs[doc_id] = {"doc": doc, "sub_questions": []}
            all_docs[doc_id]["sub_questions"].append(sq)
    
    return sorted(all_docs.values(), 
                  key=lambda x: len(x["sub_questions"]),
                  reverse=True)[:6]
```

**Banking example**:
- Original: "What are the eligibility criteria and fees for a personal loan?"
- Sub-question 1: "What are the eligibility criteria for personal loans?"
- Sub-question 2: "What fees are charged for personal loans?"

### 5. HyDE (Hypothetical Document Embeddings)

Generate a hypothetical answer, embed it, and use it for retrieval.

```python
def hyde_retrieval(query: str, llm, vectorstore, k: int = 4) -> list:
    """Generate a fake answer, use its embedding for retrieval."""
    
    # Step 1: Generate hypothetical answer
    prompt = f"""Write a plausible answer to this question.
    It doesn't need to be accurate - just what a good answer might look like.
    
    Question: {query}"""
    
    hypothetical_answer = llm.generate(prompt)
    
    # Step 2: Embed the hypothetical answer
    hyde_embedding = embed(hypothetical_answer)
    
    # Step 3: Use it to search
    results = vectorstore.similarity_search_by_vector(hyde_embedding, k=k)
    
    return results
```

**Why it works**: Hypothetical answers tend to be in the same vector space as real answers, even if factually wrong. This bridges the vocabulary gap between questions and answers.

**When it fails**: For very specific factual queries (what is the exact processing fee?), hallucinated numbers can mislead retrieval.

### 6. Hierarchical/Two-Stage Retrieval

First retrieve from document summaries, then from detailed chunks within selected documents.

```python
def hierarchical_retrieval(query: str, summary_store, chunk_store, k: int = 4):
    """Two-stage retrieval: documents first, then chunks."""
    
    # Stage 1: Find relevant documents
    top_docs = summary_store.similarity_search(query, k=3)
    doc_ids = [d.metadata["id"] for d in top_docs]
    
    # Stage 2: Within those documents, find relevant chunks
    all_chunks = []
    for doc_id in doc_ids:
        chunks = chunk_store.similarity_search(
            query, k=k, filter={"doc_id": doc_id}
        )
        all_chunks.extend(chunks)
    
    # Sort by score and return top-k
    all_chunks.sort(key=lambda c: c.metadata["score"], reverse=True)
    return all_chunks[:k]
```

**Benefits**: More efficient for large collections; ensures document-level diversity.
**When to use**: > 100K chunks, document-level relevance matters.

## K-Selection: How Many Documents to Retrieve

### The K Tradeoff

| K Value | Context Window Usage | Relevance | Noise | Risk |
|---|---|---|---|---|
| 1-3 | Minimal | High | Low | May miss info |
| 4-6 | Moderate | High | Low-Med | Balanced |
| 7-10 | Significant | Med | Med-High | Noise overload |
| 10+ | Excessive | Low | High | Lost in middle |

### Lost in the Middle Problem

Research shows LLMs pay most attention to the beginning and end of context, less to the middle. Placing the most relevant documents at the beginning and end of the context window improves answer quality.

```python
def smart_context_assembly(docs: list, query: str) -> str:
    """Arrange documents to maximize attention."""
    if len(docs) <= 3:
        return "\n\n".join([d.content for d in docs])
    
    # Most relevant first
    # Second most relevant last
    # Remaining in between
    arranged = [docs[0]]  # Most relevant first
    if len(docs) >= 2:
        arranged.extend(docs[2:])  # Middle documents
        arranged.append(docs[1])   # Second most relevant last
    
    return "\n\n".join([
        f"Document {i+1}:\n{d.content}" 
        for i, d in enumerate(arranged)
    ])
```

### Recommended K Values by Use Case

| Use Case | K (before re-rank) | K (after re-rank) | Context Tokens |
|---|---|---|---|
| Simple Q&A | 10 | 3 | ~1200 |
| Policy lookup | 15 | 4 | ~1600 |
| Complex analysis | 20 | 5-6 | ~2400 |
| Code retrieval | 10 | 3 | ~2000 (code chunks are larger) |
| Multihop reasoning | 20 | 5 | ~2000 |

## Banking-Specific Retrieval Patterns

### Retrieval with Access Control

```python
def rbac_retrieval(query: str, user: User, vectorstore, k: int = 4) -> list:
    """Retrieve only documents the user has access to."""
    
    # User's permissions
    user_roles = user.get_roles()
    allowed_departments = user.get_departments()
    clearance_level = user.get_clearance()
    
    # Build filter
    filter_dict = {
        "$and": [
            {"department": {"$in": allowed_departments}},
            {"min_clearance": {"$lte": clearance_level}}
        ]
    }
    
    # Search with filter
    return vectorstore.similarity_search(
        query, k=k, filter=filter_dict
    )
```

### Retrieval with Document Freshness

```python
from datetime import datetime, timedelta

def fresh_retrieval(query: str, vectorstore, k: int = 4,
                    max_age_days: int = 365) -> list:
    """Prefer recent documents, but don't exclude older if relevant."""
    
    cutoff = datetime.now() - timedelta(days=max_age_days)
    
    # First try with freshness filter
    results = vectorstore.similarity_search(
        query, k=k,
        filter={"updated_at": {"$gte": cutoff.isoformat()}}
    )
    
    # If not enough results, relax the filter
    if len(results) < k:
        additional = vectorstore.similarity_search(
            query, k=k - len(results),
            filter={"updated_at": {"$lt": cutoff.isoformat()}}
        )
        results.extend(additional)
    
    return results
```

### Multi-Tenant Retrieval

In banks, different business units have separate knowledge bases.

```python
def multi_tenant_retrieval(query: str, tenant_id: str, vectorstore, k: int = 4):
    """Retrieve from specific tenant's knowledge base."""
    return vectorstore.similarity_search(
        query, k=k,
        filter={"tenant_id": tenant_id}
    )
```

## Performance Optimization

### Batch Embedding for Queries

```python
# Bad: Sequential embedding (slow)
for query in queries:
    emb = embed(query)  # 100ms each
    results = search(emb)

# Good: Batch embedding
query_embeddings = embed_batch(queries)  # 50ms total for 10 queries
for emb in query_embeddings:
    results = search(emb)
```

### Caching

```python
from functools import lru_cache

@lru_cache(maxsize=10000)
def cached_retrieval(query_hash: str, k: int) -> tuple:
    """Cache retrieval results by query hash."""
    # Returns tuple of doc_ids for caching
    ...
```

### Parallel Retrieval

```python
import asyncio

async def parallel_hybrid_retrieval(query: str, k: int = 10):
    """Run BM25 and vector search in parallel."""
    bm25_task = asyncio.create_task(bm25_search_async(query, k))
    vector_task = asyncio.create_task(vector_search_async(query, k))
    
    bm25_results, vector_results = await asyncio.gather(
        bm25_task, vector_task
    )
    
    return combine_results(bm25_results, vector_results)
```

## Retrieval Evaluation Checklist

- [ ] Precision@K > 70% on golden dataset
- [ ] Mean Reciprocal Rank (MRR) > 0.6
- [ ] Coverage: > 90% of queries return at least 1 relevant result
- [ ] Latency: P95 < 200ms
- [ ] Diversity: Results span multiple documents (not all from one doc)
- [ ] Freshness: Retrieved documents are within acceptable age range
