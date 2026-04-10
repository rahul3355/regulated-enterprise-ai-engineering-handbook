# Retrieval Metrics

## Overview

Retrieval metrics measure how well the retrieval component of your RAG pipeline finds relevant documents. They evaluate the retriever independently of the LLM generator.

**Why separate retrieval metrics?** Because a bad generation result could be caused by bad retrieval or bad generation. Isolating retrieval quality lets you pinpoint the problem.

## Core Retrieval Metrics

### 1. Precision@K

What fraction of the top-K retrieved documents are actually relevant?

```
Precision@K = (Number of relevant docs in top-K) / K
```

```python
def precision_at_k(retrieved: list, relevant_ids: set, k: int) -> float:
    """Calculate precision at K."""
    retrieved_ids = [doc.metadata["id"] for doc in retrieved[:k]]
    relevant_retrieved = sum(1 for doc_id in retrieved_ids if doc_id in relevant_ids)
    return relevant_retrieved / k

# Example
# Retrieved: [A, B, C, D, E]  (top 5)
# Relevant:  {A, C, F, G}     (4 relevant docs exist)
# Precision@5 = 2/5 = 0.40  (A and C are relevant)
```

### 2. Recall@K

What fraction of all relevant documents were retrieved in the top-K?

```
Recall@K = (Number of relevant docs in top-K) / (Total number of relevant docs)
```

```python
def recall_at_k(retrieved: list, relevant_ids: set, k: int) -> float:
    """Calculate recall at K."""
    retrieved_ids = set(doc.metadata["id"] for doc in retrieved[:k])
    found_relevant = retrieved_ids & relevant_ids
    return len(found_relevant) / len(relevant_ids) if relevant_ids else 1.0

# Example
# Retrieved: [A, B, C, D, E]
# Relevant:  {A, C, F, G}
# Recall@5 = 2/4 = 0.50  (found A and C out of 4 total relevant)
```

### 3. Mean Reciprocal Rank (MRR)

How high is the first relevant document in the ranking?

```
RR = 1 / rank_of_first_relevant_doc
MRR = average(RR) across all queries
```

```python
def reciprocal_rank(retrieved: list, relevant_ids: set) -> float:
    """Calculate reciprocal rank for a single query."""
    for i, doc in enumerate(retrieved):
        if doc.metadata["id"] in relevant_ids:
            return 1.0 / (i + 1)
    return 0.0  # No relevant doc found

def mean_reciprocal_rank(queries_results: list) -> float:
    """Calculate MRR across multiple queries."""
    rrs = []
    for retrieved, relevant_ids in queries_results:
        rrs.append(reciprocal_rank(retrieved, relevant_ids))
    return sum(rrs) / len(rrs) if rrs else 0.0

# Example
# Query 1: First relevant doc at rank 1 -> RR = 1/1 = 1.0
# Query 2: First relevant doc at rank 3 -> RR = 1/3 = 0.33
# Query 3: First relevant doc at rank 2 -> RR = 1/2 = 0.50
# MRR = (1.0 + 0.33 + 0.50) / 3 = 0.61
```

### 4. NDCG@K (Normalized Discounted Cumulative Gain)

Measures ranking quality, rewarding systems that place relevant documents higher.

```python
def dcg_at_k(scores: list, k: int) -> float:
    """Discounted Cumulative Gain."""
    dcg = 0.0
    for i, score in enumerate(scores[:k]):
        dcg += score / np.log2(i + 2)  # i+2 because position is 1-indexed
    return dcg

def ndcg_at_k(retrieved: list, relevant_ids: set, k: int, 
              relevance_scores: dict = None) -> float:
    """Normalized DCG@K."""
    
    if relevance_scores is None:
        # Binary relevance: 1 if relevant, 0 if not
        relevance_scores = {doc_id: 1 for doc_id in relevant_ids}
    
    # Actual DCG
    retrieved_ids = [doc.metadata["id"] for doc in retrieved[:k]]
    actual_scores = [relevance_scores.get(doc_id, 0) for doc_id in retrieved_ids]
    dcg = dcg_at_k(actual_scores, k)
    
    # Ideal DCG (all relevant docs at top)
    ideal_scores = sorted(relevance_scores.values(), reverse=True)[:k]
    # Pad with zeros if fewer relevant docs than k
    ideal_scores.extend([0] * (k - len(ideal_scores)))
    idcg = dcg_at_k(ideal_scores, k)
    
    return dcg / idcg if idcg > 0 else 0.0

# Example
# Retrieved relevance: [1, 0, 1, 0, 1]  (positions 1,3,5 are relevant)
# DCG = 1/log2(2) + 0/log2(3) + 1/log2(4) + 0/log2(5) + 1/log2(6)
#     = 1.0 + 0 + 0.5 + 0 + 0.39 = 1.89
# Ideal: [1, 1, 1, 0, 0]
# IDCG = 1/log2(2) + 1/log2(3) + 1/log2(4) = 1.0 + 0.63 + 0.5 = 2.13
# NDCG = 1.89 / 2.13 = 0.89
```

### 5. Hit Rate@K

What fraction of queries returned at least one relevant document?

```python
def hit_rate_at_k(results_list: list, k: int) -> float:
    """Fraction of queries with at least one relevant result in top-K."""
    hits = 0
    for retrieved, relevant_ids in results_list:
        retrieved_ids = set(doc.metadata["id"] for doc in retrieved[:k])
        if retrieved_ids & relevant_ids:  # Any overlap?
            hits += 1
    return hits / len(results_list) if results_list else 0.0
```

## Computing All Metrics Together

```python
def compute_all_retrieval_metrics(retrieved: list, relevant_ids: set, 
                                   k: int = 4) -> dict:
    """Compute all retrieval metrics for a single query."""
    
    retrieved_ids = [doc.metadata["id"] for doc in retrieved]
    
    # Precision and Recall at K
    p_at_k = precision_at_k(retrieved, relevant_ids, k)
    r_at_k = recall_at_k(retrieved, relevant_ids, k)
    
    # MRR
    rr = reciprocal_rank(retrieved, relevant_ids)
    
    # NDCG at K
    relevance = {doc_id: 1 for doc_id in relevant_ids}
    ndcg = ndcg_at_k(retrieved, relevant_ids, k, relevance)
    
    # Hit Rate
    hit = 1.0 if any(doc_id in relevant_ids for doc_id in retrieved_ids[:k]) else 0.0
    
    return {
        "precision_at_k": p_at_k,
        "recall_at_k": r_at_k,
        "reciprocal_rank": rr,
        "ndcg_at_k": ndcg,
        "hit_at_k": hit,
        "num_retrieved": len(retrieved),
        "num_relevant": len(relevant_ids),
        "num_relevant_retrieved": len(set(retrieved_ids[:k]) & relevant_ids)
    }

def evaluate_retrieval_pipeline(pipeline, golden_queries, k: int = 4) -> dict:
    """Evaluate retrieval pipeline across full golden dataset."""
    
    all_metrics = {
        "precision_at_k": [],
        "recall_at_k": [],
        "reciprocal_rank": [],
        "ndcg_at_k": [],
        "hit_at_k": [],
    }
    
    per_category = {}
    
    for gq in golden_queries:
        # Retrieve
        retrieved = pipeline.retrieve(gq.query, k=k * 2)
        
        # Metrics
        metrics = compute_all_retrieval_metrics(retrieved, set(gq.relevant_doc_ids), k)
        
        for metric_name, value in metrics.items():
            if metric_name in all_metrics:
                all_metrics[metric_name].append(value)
        
        # Per-category
        category = gq.category
        if category not in per_category:
            per_category[category] = {m: [] for m in all_metrics}
        for m in all_metrics:
            per_category[category][m].append(all_metrics[m][-1])
    
    # Aggregate
    results = {}
    for metric_name, values in all_metrics.items():
        results[metric_name] = {
            "mean": np.mean(values),
            "median": np.median(values),
            "std": np.std(values),
            "min": np.min(values),
            "max": np.max(values),
        }
    
    results["per_category"] = {}
    for category, metrics in per_category.items():
        results["per_category"][category] = {
            m: np.mean(v) for m, v in metrics.items()
        }
    
    return results
```

## Target Values for Banking RAG

| Metric | Minimum | Good | Excellent |
|---|---|---|---|
| Precision@4 | 0.50 | 0.70 | 0.85 |
| Recall@4 | 0.30 | 0.50 | 0.70 |
| MRR | 0.40 | 0.60 | 0.80 |
| NDCG@4 | 0.45 | 0.65 | 0.85 |
| Hit Rate@4 | 0.70 | 0.85 | 0.95 |

**Note**: These targets assume a well-constructed golden dataset. If your golden dataset has many hard queries, lower targets may be appropriate.

## Metric Selection Guide

| Situation | Most Important Metric |
|---|---|
| Users need the right answer fast | **MRR** (first relevant doc rank) |
| Multiple documents needed for answer | **Recall@K** (coverage) |
| Quality of top results matters most | **Precision@K** (accuracy) |
| Overall ranking quality | **NDCG@K** (weighted ranking) |
| System must always find something | **Hit Rate@K** (coverage) |

**Banking recommendation**: Focus on **Precision@4** and **MRR** as primary metrics. Banking users need accurate top results, and the first relevant document should appear at rank 1 or 2.

## Interpreting Results

### Scenario 1: High Precision, Low Recall

```
Precision@4 = 0.80  (4/5 retrieved docs are relevant)
Recall@4 = 0.20     (but only 20% of all relevant docs found)
```

**Diagnosis**: The retriever finds good docs but misses many relevant ones.
**Fix**: Increase initial retrieval K, use query expansion, or improve chunking to capture more relevant content.

### Scenario 2: Low Precision, High Recall

```
Precision@4 = 0.25  (only 1/4 retrieved docs relevant)
Recall@4 = 0.70     (but 70% of relevant docs are in the top results)
```

**Diagnosis**: The retriever finds the right docs but buries them among irrelevant ones.
**Fix**: Add re-ranking, improve metadata filtering, tune hybrid search alpha.

### Scenario 3: Good MRR, Poor NDCG

```
MRR = 0.75          (first relevant doc appears at rank ~1.3)
NDCG@4 = 0.45       (overall ranking is poor)
```

**Diagnosis**: The system finds one good document quickly but the rest of the ranking is noisy.
**Fix**: Improve re-ranking to push more relevant docs higher.

### Scenario 4: Low Across All Metrics

```
Precision@4 = 0.20, Recall@4 = 0.10, MRR = 0.15, NDCG@4 = 0.15
```

**Diagnosis**: Fundamental retrieval issue -- likely wrong embedding model, poor chunking, or query mismatch.
**Fix**: 
1. Check that embedding model is appropriate for domain
2. Evaluate chunking strategy
3. Try query rewriting
4. Test hybrid search vs. vector-only

## Per-Category Analysis

Always break down metrics by query category:

```python
# Example results by category
results = {
    "per_category": {
        "factual": {"precision_at_k": 0.85, "mrr": 0.80},
        "procedural": {"precision_at_k": 0.72, "mrr": 0.65},
        "comparative": {"precision_at_k": 0.55, "mrr": 0.45},
        "multi_part": {"precision_at_k": 0.50, "mrr": 0.40},
    }
}
```

**Insight**: Factual queries perform well; multi-part queries perform poorly.
**Action**: Implement query decomposition for multi-part queries.

## Monitoring in Production

```python
# Track retrieval metrics over time
def log_retrieval_metrics(query: str, retrieved: list, user_feedback: str = None):
    """Log retrieval metrics for monitoring."""
    
    metrics = {
        "timestamp": datetime.utcnow().isoformat(),
        "query": query,
        "num_results": len(retrieved),
        "top_score": retrieved[0].metadata.get("score") if retrieved else None,
        "avg_score": np.mean([d.metadata.get("score", 0) for d in retrieved]),
    }
    
    if user_feedback:
        metrics["user_feedback"] = user_feedback
    
    retrieval_metrics_logger.info(json.dumps(metrics))

# Weekly aggregation
def weekly_retrieval_report() -> dict:
    """Generate weekly retrieval performance report."""
    
    logs = get_logs_since(datetime.now() - timedelta(days=7))
    
    return {
        "total_queries": len(logs),
        "avg_results_per_query": np.mean([l["num_results"] for l in logs]),
        "avg_top_score": np.mean([l["top_score"] for l in logs if l["top_score"]]),
        "queries_with_no_results": sum(1 for l in logs if l["num_results"] == 0),
        "user_satisfaction_rate": compute_satisfaction_from_feedback(logs),
    }
```

## Golden Dataset Maintenance

Retrieval metrics are only as good as your golden dataset. Maintain it actively:

1. **Review quarterly**: Ensure relevant doc IDs are still correct (documents may have been updated or removed)
2. **Add failure cases**: Every time the retriever fails in production, add that query to the golden dataset
3. **Expand coverage**: Add queries for new document types as they are added to the knowledge base
4. **Balance categories**: Ensure each query category has at least 30-50 queries for statistical significance
5. **Remove duplicates**: Deduplicate very similar queries that artificially inflate metrics
