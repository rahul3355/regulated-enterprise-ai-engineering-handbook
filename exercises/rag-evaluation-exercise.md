# RAG Evaluation Exercise

> Evaluate RAG pipeline quality using a golden dataset — measuring retrieval and generation quality.

## Problem Statement

You are the quality engineer for the GenAI platform's RAG pipeline. The product team wants to know: "How good are our AI assistant's answers?"

Build an evaluation framework using a golden dataset of query-document-answer triples.

## Golden Dataset

You have a dataset of 100 curated examples created by compliance and HR experts:

```json
[
  {
    "id": "eval-001",
    "query": "What is the deadline for submitting expense reports?",
    "expected_documents": ["POL-001", "POL-015"],
    "expected_answer": "Expense reports must be submitted within 30 days of the expense date.",
    "category": "finance",
    "difficulty": "easy",
    "created_by": "compliance-team",
    "created_at": "2026-03-01"
  },
  {
    "id": "eval-042",
    "query": "Can I work remotely from another country?",
    "expected_documents": ["POL-002", "POL-033", "POL-045"],
    "expected_answer": "Remote work from another country requires approval from both your manager and the tax team, as it may create tax implications for the company. Submit a request through the HR portal at least 30 days in advance.",
    "category": "hr",
    "difficulty": "hard",
    "created_by": "hr-team",
    "created_at": "2026-03-15"
  }
]
```

## Evaluation Metrics

### Retrieval Quality

| Metric | Definition | How to Calculate |
|--------|-----------|-----------------|
| **Recall@K** | Fraction of relevant documents retrieved in top K | Count expected_documents found in top K results / total expected_documents |
| **Precision@K** | Fraction of retrieved documents that are relevant | Count relevant documents in top K / K |
| **MRR** | Mean Reciprocal Rank of first relevant document | Average of 1/rank for each query |
| **NDCG@K** | Normalized Discounted Cumulative Gain | Ranking quality considering position |

### Generation Quality

| Metric | Definition | How to Calculate |
|--------|-----------|-----------------|
| **Factuality** | Is the answer factually correct vs. expected answer | LLM-as-judge or human review |
| **Completeness** | Does the answer cover all key points | Compare key points in expected vs. actual |
| **Conciseness** | Is the answer appropriately brief | Length ratio, redundancy check |
| **Citation accuracy** | Do the cited sources actually support the answer | Verify citations against documents |
| **Safety** | Does the response avoid harmful content | Content filter, manual review |

## Tasks

### Task 1: Build the Evaluation Pipeline

```python
def evaluate_rag(golden_dataset: list[dict], rag_pipeline) -> dict:
    """
    Run the full evaluation pipeline.

    Returns:
    {
        "retrieval": {
            "recall_at_1": 0.65,
            "recall_at_3": 0.82,
            "recall_at_5": 0.91,
            "precision_at_3": 0.54,
            "mrr": 0.71,
            "ndcg_at_3": 0.76,
        },
        "generation": {
            "factuality": 0.88,
            "completeness": 0.75,
            "conciseness": 0.82,
            "citation_accuracy": 0.91,
            "safety": 1.0,
        },
        "by_category": {
            "finance": {"recall_at_3": 0.85, "factuality": 0.92},
            "hr": {"recall_at_3": 0.78, "factuality": 0.85},
            "security": {"recall_at_3": 0.83, "factuality": 0.87},
        },
        "by_difficulty": {
            "easy": {"recall_at_3": 0.90, "factuality": 0.95},
            "medium": {"recall_at_3": 0.80, "factuality": 0.88},
            "hard": {"recall_at_3": 0.65, "factuality": 0.72},
        }
    }
    """
```

### Task 2: Calculate Retrieval Metrics

```python
def calculate_recall_at_k(
    expected_docs: list[str],
    retrieved_docs: list[dict],
    k: int = 3
) -> float:
    """
    Calculate Recall@K.

    expected_docs: List of document IDs that should be retrieved
    retrieved_docs: List of dicts with 'doc_id' key, in retrieval order
    k: Number of top results to consider

    Returns: Fraction of expected docs found in top K
    """
    retrieved_ids = [doc["doc_id"] for doc in retrieved_docs[:k]]
    found = len(set(expected_docs) & set(retrieved_ids))
    return found / len(expected_docs) if expected_docs else 1.0
```

### Task 3: LLM-as-Judge for Factuality

```python
def evaluate_factuality(
    query: str,
    expected_answer: str,
    actual_answer: str,
    context: str,
) -> dict:
    """
    Use an LLM to judge if the actual answer is factually correct.

    Returns:
    {
        "score": 0.8,  # 0.0 to 1.0
        "reason": "The actual answer contains the key information about
                    the 30-day deadline and manager approval requirement.
                    However, it misses the detail about international
                    expenses being in USD.",
        "missing_points": ["international expenses in USD"],
        "incorrect_points": [],
    }
    """
```

## Expected Output

```
RAG Pipeline Evaluation — 2026-04-10
=====================================
Dataset: 100 queries, 4 categories, 3 difficulty levels

RETRIEVAL QUALITY:
  Recall@1:  0.65  ████████████░░░░░░░░
  Recall@3:  0.82  ████████████████░░░░
  Recall@5:  0.91  ██████████████████░░
  Precision@3: 0.54
  MRR:       0.71
  NDCG@3:    0.76

GENERATION QUALITY:
  Factuality:     0.88  ██████████████████░░
  Completeness:   0.75  ███████████████░░░░░
  Conciseness:    0.82  ████████████████░░░░
  Citation Acc:   0.91  ██████████████████░░
  Safety:         1.00  ████████████████████

BY CATEGORY:
  Finance:    Recall@3=0.85  Factuality=0.92  ← Best
  HR:         Recall@3=0.78  Factuality=0.85
  Security:   Recall@3=0.83  Factuality=0.87
  IT:         Recall@3=0.72  Factuality=0.80  ← Worst

BY DIFFICULTY:
  Easy:       Recall@3=0.90  Factuality=0.95
  Medium:     Recall@3=0.80  Factuality=0.88
  Hard:       Recall@3=0.65  Factuality=0.72  ← Needs improvement

ISSUES IDENTIFIED:
  1. Hard queries have low Recall@3 (0.65) — multi-topic queries struggle
  2. IT category has lowest scores — fewer/lower quality documents
  3. Completeness (0.75) is lowest generation metric — answers miss details
  4. 3 queries had safety flags — investigate false positives

RECOMMENDATIONS:
  1. Improve chunking for multi-topic documents (hybrid search)
  2. Review IT document coverage and quality
  3. Adjust prompt to encourage more complete answers
  4. Review safety filter false positives
```

## Extensions

1. **Continuous evaluation:** Run the evaluation pipeline nightly on the golden dataset. Alert on regression.

2. **A/B testing:** Compare two retrieval strategies (e.g., dense-only vs. hybrid) using the golden dataset.

3. **Human-in-the-loop:** Build a UI where subject matter experts can review and score AI-generated answers.

4. **Synthetic data generation:** Use an LLM to generate additional evaluation questions from policy documents.

5. **Production monitoring:** Derive production metrics that correlate with golden dataset scores (e.g., if cache hit rate drops, does retrieval quality drop?).

## Interview Relevance

RAG evaluation is a top topic in GenAI interviews:

| Skill | Why It Matters |
|-------|---------------|
| Retrieval metrics | Can you measure what matters? |
| Golden dataset design | Can you create evaluation data? |
| LLM-as-judge | Can you automate quality evaluation? |
| Breakdown analysis | Can you find weak spots? |
| Actionable insights | Can you turn metrics into improvements? |

**Follow-up questions:**
- "How many golden examples do you need?"
- "What are the limitations of LLM-as-judge?"
- "How do you keep the golden dataset current as policies change?"
- "What's the difference between offline evaluation and online monitoring?"
