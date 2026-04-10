# Evaluation Frameworks

This directory covers evaluating GenAI application quality using RAGAS, DeepEval, and custom evaluation frameworks tailored for banking use cases.

## Key Topics

- RAGAS for RAG system evaluation
- DeepEval for LLM output quality assessment
- Custom evaluation metrics for banking
- Building evaluation datasets
- Automated evaluation pipelines
- Human evaluation integration

## RAGAS Metrics

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| Faithfulness | Factual consistency with context | >= 0.90 |
| Answer Relevance | How well answer addresses question | >= 0.85 |
| Context Precision | Quality of retrieved chunks | >= 0.80 |
| Context Recall | Completeness of retrieval | >= 0.80 |
| Answer Correctness | Overall answer accuracy | >= 0.85 |

## DeepEval Metrics

```python
from deeval.metrics import (
    AnswerRelevancyMetric,
    FaithfulnessMetric,
    ContextualRelevancyMetric,
    HallucinationMetric,
    BiasMetric,
    ToxicityMetric,
)

from deeval.evaluate import evaluate

# Banking compliance evaluation
test_cases = [
    LLMTestCase(
        input="Is this transaction suspicious?",
        actual_output="The transaction shows patterns consistent with...",
        retrieval_context=[retrieved_doc_1, retrieved_doc_2],
        expected_output="Based on the AML policy, this transaction...",
    ),
]

metrics = [
    AnswerRelevancyMetric(threshold=0.85),
    FaithfulnessMetric(threshold=0.90),
    ContextualRelevancyMetric(threshold=0.80),
    HallucinationMetric(threshold=0.05),
]

results = evaluate(test_cases, metrics)
```

## Custom Banking Evaluation Metrics

```python
class RegulatoryCitationAccuracy(BaseMetric):
    """Verify that all regulatory citations are to real regulations."""

    def measure(self, test_case: LLMTestCase) -> float:
        citations = extract_citations(test_case.actual_output)
        valid = sum(1 for c in citations if verify_citation(c))
        return valid / len(citations) if citations else 1.0

class TippingOffCompliance(BaseMetric):
    """Ensure response does not constitute tipping off."""

    def measure(self, test_case: LLMTestCase) -> float:
        score = self.tipping_off_classifier(test_case.actual_output)
        return 1.0 if score < THRESHOLD else 0.0
```

## Cross-References

- [../model-observability.md](../model-observability.md) — Quality monitoring in production
- [../prompt-versioning.md](../prompt-versioning.md) — Testing before prompt deployment
- [../safe-rollout-strategies.md](../safe-rollout-strategies.md) — Evaluation before rollout
- [../hallucinations.md](../hallucinations.md) — Hallucination measurement
- [../rag-and-search/](../rag-and-search/) — RAG quality evaluation
