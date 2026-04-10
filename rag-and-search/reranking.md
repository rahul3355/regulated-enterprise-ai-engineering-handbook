# Re-Ranking Models

## What Is Re-Ranking?

Re-ranking is a second-stage retrieval step that takes the top-N candidates from initial retrieval (usually 20-50 documents) and re-scores them with a more accurate model to select the final top-K.

```
Query --> [Initial Retrieval: 50 docs via vector search] --> [Re-ranker: score all 50] --> [Top 4] --> LLM Context
```

**Why re-rank?** Initial retrieval uses approximate methods (cosine similarity on embeddings) that are fast but imprecise. Re-rankers use cross-encoder models that jointly process the query and document, producing much more accurate relevance scores.

**Performance gain**: Adding a re-ranker typically improves retrieval precision by 10-25 percentage points. This is one of the highest-ROI improvements in a RAG pipeline.

## Bi-Encoders vs. Cross-Encoders

### Bi-Encoders (Initial Retrieval)

Process query and document independently, then compare embeddings.

```
Query: "What are the loan fees?"  -> embed -> [0.12, -0.34, ...]
Doc: "Processing charges are 1.5%" -> embed -> [0.14, -0.31, ...]
Similarity = cosine([0.12, ...], [0.14, ...]) = 0.92
```

**Pros**: Fast (pre-compute document embeddings); scalable to millions of docs
**Cons**: Less accurate; cannot model query-document interaction

**Used for**: First-stage retrieval (vector search)

### Cross-Encoders (Re-Ranking)

Process query and document together through a single transformer.

```
Input: "[CLS] What are the loan fees? [SEP] Processing charges are 1.5% [SEP]"
Model: BERT/RoBERTa cross-encoder
Output: relevance score (0-1)
```

**Pros**: Much more accurate; captures fine-grained interactions
**Cons**: Slow (must process each query-doc pair); not scalable to large collections

**Used for**: Second-stage re-ranking (top-N candidates only)

### Comparison

| Property | Bi-Encoder | Cross-Encoder |
|---|---|---|
| **Accuracy** | Good | Excellent |
| **Speed** | Very fast (pre-computed) | Slow (per-pair computation) |
| **Scalability** | Millions of docs | Hundreds of pairs |
| **Use case** | First-stage retrieval | Re-ranking top-N |
| **Model size** | 100M-400M params | 100M-400M params |
| **Latency per doc** | < 1ms (lookup) | 10-50ms |

## Popular Re-Ranking Models

### Open Source Models

| Model | Size | MTEB Rerank Score | Latency (A10 GPU) | Notes |
|---|---|---|---|---|
| BGE-Reranker-v2-M3 | 560M | 62.4 | 15ms | Best open source, multilingual |
| BGE-Reranker-large | 560M | 59.8 | 15ms | Strong English performance |
| Cohere Rerank v3.5 | N/A (API) | 64.2 | 100ms (API) | Best overall, API only |
| Jina Reranker v2 | 278M | 58.5 | 10ms | Fast, good quality |
| RankGPT (GPT-4) | N/A | 65.0+ | 2000ms (API) | LLM-as-reranker, expensive |

### Cloud API Models

| Model | Cost per 1K reranks | Latency | Multilingual | Notes |
|---|---|---|---|---|
| Cohere Rerank v3.5 | $1.50 | 100ms | Yes (50+ languages) | Best quality |
| Voyage AI Rerank | $0.50 | 150ms | No | Good quality, cheaper |
| Jina Rerank API | $0.30 | 120ms | Yes | Cost-effective |

### Banking Recommendation

| Scenario | Re-Ranker | Rationale |
|---|---|---|
| Cloud, best quality | Cohere Rerank v3.5 | Highest accuracy, multilingual |
| Cloud, cost-sensitive | Jina Rerank API | Good quality at lower cost |
| Self-hosted | BGE-Reranker-v2-M3 | Best open source |
| Self-hosted, speed priority | Jina Reranker v2 | Fast inference |
| Budget unlimited | RankGPT (GPT-4) | LLM-as-reranker, highest accuracy |

## Implementing Re-Ranking

### Basic Re-Ranking with BGE

```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer
import torch

class CrossEncoderReranker:
    def __init__(self, model_name="BAAI/bge-reranker-v2-m3", device="cuda"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(
            model_name
        ).to(device)
        self.model.eval()
        self.device = device
    
    def rerank(self, query: str, documents: list[str], 
               top_k: int = 4) -> list[tuple[float, str]]:
        """Re-rank documents by relevance to query."""
        
        # Create query-doc pairs
        pairs = [(query, doc) for doc in documents]
        
        # Tokenize
        inputs = self.tokenizer(
            pairs, padding=True, truncation=True,
            return_tensors="pt", max_length=512
        ).to(self.device)
        
        # Score
        with torch.no_grad():
            scores = self.model(**inputs).logits.squeeze(-1).cpu().numpy()
        
        # Sort by score
        scored_docs = list(zip(scores.tolist(), documents))
        scored_docs.sort(reverse=True, key=lambda x: x[0])
        
        return scored_docs[:top_k]

# Usage
reranker = CrossEncoderReranker()

# Initial retrieval (50 docs from vector search)
initial_docs = vector_store.similarity_search(query, k=50)
doc_texts = [doc.page_content for doc in initial_docs]

# Re-rank to top 4
ranked = reranker.rerank(query, doc_texts, top_k=4)
for score, text in ranked:
    print(f"Score: {score:.4f}")
```

### Re-Ranking with Cohere API

```python
import cohere

co = cohere.Client("your-api-key")

def cohere_rerank(query: str, documents: list[str], 
                  top_k: int = 4) -> list[dict]:
    """Re-rank using Cohere's API."""
    response = co.rerank(
        query=query,
        documents=documents,
        top_n=top_k,
        model="rerank-v3.5"
    )
    
    results = []
    for r in response.results:
        results.append({
            "index": r.index,
            "score": r.relevance_score,
            "document": documents[r.index]
        })
    
    return results
```

### Full RAG Pipeline with Re-Ranking

```python
class RAGPipeline:
    def __init__(self, vectorstore, reranker, llm, 
                 initial_k: int = 20, final_k: int = 4):
        self.vectorstore = vectorstore
        self.reranker = reranker
        self.llm = llm
        self.initial_k = initial_k
        self.final_k = final_k
    
    def query(self, user_query: str) -> dict:
        """Full pipeline with re-ranking."""
        
        # Stage 1: Initial retrieval
        candidates = self.vectorstore.similarity_search(
            user_query, k=self.initial_k
        )
        candidate_texts = [c.page_content for c in candidates]
        candidate_metadata = [c.metadata for c in candidates]
        
        # Stage 2: Re-ranking
        ranked = self.reranker.rerank(
            user_query, candidate_texts, top_k=self.final_k
        )
        
        # Stage 3: Context assembly
        context = "\n\n".join([
            f"Source: {candidate_metadata[r['index']].get('source', 'Unknown')}\n{r['document']}"
            for r in ranked
        ])
        
        # Stage 4: Generation
        response = self.llm.generate(
            system="Answer using the provided context. Cite sources.",
            context=context,
            question=user_query
        )
        
        return {
            "response": response,
            "sources": [
                candidate_metadata[r["index"]] 
                for r in ranked
            ],
            "ranked_scores": [r["score"] for r in ranked]
        }
```

## Re-Ranking Performance Optimization

### Batched Inference

```python
def batched_rerank(query: str, documents: list[str], 
                   batch_size: int = 16) -> list[float]:
    """Process documents in batches to maximize GPU utilization."""
    all_scores = []
    
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]
        pairs = [(query, doc) for doc in batch]
        
        inputs = tokenizer(
            pairs, padding=True, truncation=True,
            return_tensors="pt", max_length=512
        ).to(device)
        
        with torch.no_grad():
            scores = model(**inputs).logits.squeeze(-1).cpu().numpy()
        
        all_scores.extend(scores.tolist())
    
    return all_scores
```

### Caching Re-Rank Scores

```python
import hashlib
import diskcache

cache = diskcache.Cache("./rerank_cache")

def cached_rerank(query: str, documents: list[str]) -> list[float]:
    """Cache re-rank results for identical query-doc pairs."""
    cache_key = hashlib.sha256(
        (query + "||" + "||".join(documents)).encode()
    ).hexdigest()
    
    if cache_key in cache:
        return cache[cache_key]
    
    scores = rerank(query, documents)
    cache[cache_key] = scores
    return scores
```

### Early Exit Optimization

```python
def early_exit_rerank(query: str, documents: list[str], 
                      top_k: int = 4) -> list:
    """Stop processing once we're confident about top-k."""
    # Process in small batches
    # Track current top-k scores
    # Stop if remaining documents can't beat the k-th best score
    ...
```

## Evaluating Re-Rankers

### Metrics

| Metric | Description | Target |
|---|---|---|
| **NDCG@K** | Normalized Discounted Cumulative Gain | > 0.7 |
| **MRR** | Mean Reciprocal Rank | > 0.6 |
| **Precision@K** | Fraction of top-K that are relevant | > 0.8 |
| **Recall@K** | Fraction of all relevant docs in top-K | > 0.6 |

### A/B Test Setup

```python
def compare_retrieval_strategies(golden_queries):
    """Compare with vs without re-ranking."""
    
    strategies = {
        "vector_only": lambda q: vector_search(q, k=4),
        "hybrid": lambda q: hybrid_search(q, k=4),
        "vector_reranked": lambda q: reranked_search(q, initial_k=20, final_k=4),
        "hybrid_reranked": lambda q: hybrid_reranked_search(q, initial_k=20, final_k=4),
    }
    
    results = {}
    for name, strategy in strategies.items():
        scores = evaluate_on_golden(golden_queries, strategy)
        results[name] = scores
    
    # Print comparison
    for name, scores in results.items():
        print(f"{name}: P@4={scores['precision']:.3f} "
              f"MRR={scores['mrr']:.3f} "
              f"NDCG@4={scores['ndcg']:.3f}")
```

### Expected Results (Banking Dataset)

| Strategy | P@4 | MRR | NDCG@4 | Latency |
|---|---|---|---|---|
| Vector only | 0.55 | 0.48 | 0.58 | 50ms |
| Hybrid | 0.68 | 0.62 | 0.71 | 80ms |
| Vector + Re-rank | 0.78 | 0.73 | 0.82 | 200ms |
| Hybrid + Re-rank | 0.85 | 0.80 | 0.88 | 230ms |

## When to Skip Re-Ranking

Re-ranking is not always necessary. Skip it when:

1. **Document collection is small** (< 5K chunks): Vector search alone is sufficient
2. **Latency is critical** (< 100ms budget): Re-ranking adds 100-300ms
3. **Documents are very similar**: Re-ranker has little signal to differentiate
4. **Cost constraints**: Self-hosted re-ranker needs GPU; API costs add up

**Rule of thumb**: If retrieval precision@K is already > 80% without re-ranking, the marginal benefit may not justify the added complexity and cost.

## Banking-Specific Considerations

1. **Regulatory documents**: Re-rankers trained on general web data may underperform on regulatory text. Fine-tune on banking query-document pairs if available.

2. **Access control integration**: Re-rank only documents the user has access to. Do not re-rank and then filter -- filter first to avoid showing titles of restricted documents.

3. **Multilingual re-ranking**: For global banks, use BGE-Reranker-v2-M3 or Cohere Rerank v3.5 which support 50+ languages.

4. **Compliance logging**: Log re-rank scores for audit purposes. Unusually low scores (< 0.3) may indicate the query is outside the knowledge base scope.

5. **Confidence scoring**: Use re-rank scores as confidence indicators. If the top document scores < 0.5, the system should say "I don't have enough information" rather than guessing.
