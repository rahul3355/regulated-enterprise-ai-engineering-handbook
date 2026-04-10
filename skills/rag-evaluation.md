# Skill: RAG Evaluation

## Core Principles

1. **RAG Quality = Retrieval Quality + Generation Quality** — Both must be good for the system to work.
2. **Measure, Don't Guess** — Use metrics, not vibes, to assess RAG quality.
3. **Golden Datasets Are Essential** — You need known good query-answer pairs to evaluate.
4. **Human Judgment Is the Gold Standard** — Automated metrics supplement, not replace, human review.

## Mental Models

### RAG Evaluation Pyramid

```
┌─────────────────────────────────────────────────┐
│  End-to-End Quality (Human Rated)               │  ← Full pipeline evaluation
├─────────────────────────────────────────────────┤
│  Generation Quality (Groundedness, Relevance)   │  ← Is the AI response good?
├─────────────────────────────────────────────────┤
│  Retrieval Quality (Precision, Recall, MRR)     │  ← Did we find the right docs?
├─────────────────────────────────────────────────┤
│  Component Quality (Embedding similarity,       │  ← Individual component health
│  chunk quality, index freshness)                │
└─────────────────────────────────────────────────┘
```

## Key Metrics

### Retrieval Metrics

| Metric | Formula | Target | What It Measures |
|--------|---------|--------|-----------------|
| Precision@K | Relevant retrieved / Total retrieved | > 0.8 | How many retrieved docs are relevant? |
| Recall@K | Relevant retrieved / Total relevant | > 0.9 | How many relevant docs did we find? |
| MRR (Mean Reciprocal Rank) | avg(1/rank of first relevant) | > 0.7 | How early do relevant docs appear? |
| NDCG@K | Discounted cumulative gain | > 0.8 | Ranking quality |

### Generation Metrics

| Metric | How Measured | Target | What It Measures |
|--------|-------------|--------|-----------------|
| Groundedness | LLM judge: "Is this response supported by the context?" | > 0.9 | Response based on retrieved docs |
| Relevance | LLM judge: "Does this answer the user's question?" | > 0.85 | Response addresses the query |
| Completeness | LLM judge: "Does this cover all aspects of the query?" | > 0.8 | Thoroughness of response |
| Harmfulness | Classifier: toxic, biased, dangerous content | 0% | Safety |
| PII Leakage | PII scanner on responses | 0% | Data protection |
| Hallucination Rate | Claims not supported by context / Total claims | < 2% | Fabrication |

## Step-by-Step Approach

### 1. Build a Golden Dataset

```python
"""
Golden Dataset Schema for RAG Evaluation

Each entry:
- query: The user's question
- relevant_documents: List of document IDs that contain the answer
- ideal_answer: A gold-standard answer (for comparison)
- expected_citations: Document IDs the response should cite
- difficulty: easy | medium | hard
- category: policy | procedure | compliance | general
"""

golden_dataset = [
    {
        "query": "What is the maximum amount I can expense for a client lunch?",
        "relevant_documents": ["TEP-2024-01", "EXP-POLICY-v3"],
        "ideal_answer": "According to the Travel and Expenses Policy, business meals are reimbursable up to £75 per person. Receipts are required.",
        "expected_citations": ["TEP-2024-01§5.1"],
        "difficulty": "easy",
        "category": "policy",
    },
    {
        "query": "How do I handle a suspicious transaction report?",
        "relevant_documents": ["AML-PROC-2024", "SAR-GUIDELINES"],
        "ideal_answer": "Suspicious Transaction Reports (STRs) must be filed with the MLRO within 24 hours of detection. Use the SAR portal and include all transaction details, customer information, and reason for suspicion.",
        "expected_citations": ["AML-PROC-2024§3.2", "SAR-GUIDELINES§2"],
        "difficulty": "medium",
        "category": "procedure",
    },
]
```

### 2. Evaluate Retrieval

```python
def evaluate_retrieval(golden_dataset, retriever):
    """Evaluate retrieval quality against golden dataset."""
    precisions = []
    recalls = []
    rr_scores = []  # Reciprocal Rank
    
    for entry in golden_dataset:
        retrieved = retriever.search(entry["query"], top_k=5)
        retrieved_ids = [doc.id for doc in retrieved]
        relevant_ids = set(entry["relevant_documents"])
        
        # Precision@5
        relevant_retrieved = len(set(retrieved_ids) & relevant_ids)
        precision = relevant_retrieved / len(retrieved_ids) if retrieved_ids else 0
        precisions.append(precision)
        
        # Recall@5
        recall = relevant_retrieved / len(relevant_ids) if relevant_ids else 0
        recalls.append(recall)
        
        # Reciprocal Rank
        for i, doc_id in enumerate(retrieved_ids):
            if doc_id in relevant_ids:
                rr_scores.append(1.0 / (i + 1))
                break
        else:
            rr_scores.append(0.0)
    
    return {
        "precision_at_5": sum(precisions) / len(precisions),
        "recall_at_5": sum(recalls) / len(recalls),
        "mrr": sum(rr_scores) / len(rr_scores),
    }
```

### 3. Evaluate Generation

```python
def evaluate_generation(golden_dataset, rag_pipeline):
    """Evaluate end-to-end RAG quality."""
    scores = {
        "groundedness": [],
        "relevance": [],
        "completeness": [],
        "hallucinations": [],
        "citation_accuracy": [],
    }
    
    for entry in golden_dataset:
        response = rag_pipeline.query(entry["query"])
        
        # Groundedness: Are claims supported by retrieved documents?
        grounded = check_groundedness(response.text, response.retrieved_docs)
        scores["groundedness"].append(grounded)
        
        # Relevance: Does the response answer the query?
        relevance = llm_judge_relevance(entry["query"], response.text)
        scores["relevance"].append(relevance)
        
        # Hallucination detection
        hallucinations = detect_hallucinations(response.text, response.retrieved_docs)
        scores["hallucinations"].append(len(hallucinations))
        
        # Citation accuracy: Did it cite the right documents?
        citation_score = check_citations(response.citations, entry["expected_citations"])
        scores["citation_accuracy"].append(citation_score)
    
    return {
        "groundedness_rate": sum(scores["groundedness"]) / len(scores["groundedness"]),
        "relevance_rate": sum(scores["relevance"]) / len(scores["relevance"]),
        "avg_hallucinations": sum(scores["hallucinations"]) / len(scores["hallucinations"]),
        "citation_accuracy": sum(scores["citation_accuracy"]) / len(scores["citation_accuracy"]),
    }
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| No golden dataset | No way to measure quality | Build one, start small, grow iteratively |
| Only testing easy queries | Inflated quality scores | Include medium and hard queries |
| Only automated metrics | Missing nuanced quality issues | Include human review |
| Stale golden dataset | Metrics don't reflect current behavior | Update golden dataset regularly |
| Not testing edge cases | Surprises in production | Include adversarial and edge-case queries |
| Not testing across categories | Blind spots in specific areas | Ensure category diversity |

## Banking-Specific Concerns

1. **Compliance Accuracy** — Wrong compliance answers are not just bad UX — they're regulatory risk.
2. **Citation Traceability** — Every AI claim must be traceable to a source document for audit.
3. **Recency** — Policy documents change. RAG must retrieve the current version, not outdated versions.
4. **Role-Based Retrieval** — The same query from different roles may require different answers based on access.
5. **Regulatory Language** — AI responses must use correct regulatory terminology.

## Metrics to Monitor in Production

| Metric | Alert Threshold | Action |
|--------|----------------|--------|
| Groundedness rate | < 85% over 7 days | Review retrieval quality, prompt, model |
| Hallucination rate | > 3% over 7 days | Urgent: Investigate root cause |
| PII leakage | Any occurrence | CRITICAL: Incident response |
| Average retrieval latency | > 500ms p99 | Optimize vector search, indexing |
| User thumbs-down rate | > 10% over 7 days | Review quality, update golden dataset |
| Zero-result rate | > 5% | Improve retrieval coverage |

## Interview Questions

1. How do you measure RAG quality?
2. What is groundedness and how do you measure it?
3. How large should a golden dataset be?
4. How do you handle evaluation when the "correct answer" changes (policy updates)?
5. What is the difference between precision and recall in retrieval?
6. How do you detect hallucinations automatically?
7. Why is human review still necessary for RAG evaluation in banking?

## Hands-On Exercise

### Exercise: Evaluate a RAG Pipeline

**Problem:** You have a RAG pipeline for banking policy search. Build an evaluation suite.

**Given:**
- A golden dataset of 50 queries (mix of easy, medium, hard)
- Access to the RAG pipeline
- An LLM-as-judge function

**Tasks:**
1. Calculate Precision@5, Recall@5, and MRR for retrieval
2. Calculate groundedness and relevance rates for generation
3. Identify the 5 worst-performing queries and explain why they fail
4. Recommend 3 improvements based on your analysis

**Expected Output:**
- Evaluation report with metrics
- Analysis of failure modes
- Prioritized improvement recommendations

---

**Related files:**
- `rag-and-search/evaluation.md` — Full RAG evaluation guide
- `genai-platforms/evaluation-frameworks/` — Evaluation framework implementations
- `skills/prompt-engineering.md` — Prompt engineering skill
- `testing-and-quality/llm-evaluation.md` — LLM testing strategies
