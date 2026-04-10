# LLM Evaluation for Banking GenAI

## Why Traditional Testing Is Not Enough

Traditional testing verifies deterministic code. LLM outputs are non-deterministic, making traditional assertions impractical. LLM evaluation uses specialized frameworks to assess AI output quality across multiple dimensions.

## Evaluation Dimensions

```
┌─────────────────────────────────────────────────────────────┐
│  LLM EVALUATION DIMENSIONS                                   │
├─────────────────────────┬───────────────────────────────────┤
│  Dimension              │  What It Measures                 │
├─────────────────────────┼───────────────────────────────────┤
│  Faithfulness           │  Are claims supported by sources?  │
│  Answer Relevance      │  Does the answer address the query?│
│  Context Precision     │  Are retrieved docs relevant?      │
│  Context Recall         │  Are all relevant docs retrieved? │
│  Hallucination          │  Does response contain fabricated │
│                         │  information?                     │
│  Toxicity              │  Does response contain harmful    │
│                         │  or offensive content?            │
│  Bias                  │  Does response show demographic   │
│                         │  or systemic bias?                │
│  Compliance            │  Does response meet regulatory    │
│                         │  requirements?                    │
└─────────────────────────┴───────────────────────────────────┘
```

## RAGAS Framework

RAGAS (Retrieval Augmented Generation Assessment) is purpose-built for evaluating RAG pipelines:

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# Evaluation dataset
data = {
    "question": [
        "What is the current 30-year fixed mortgage rate?",
        "What credit score do I need for a conventional mortgage?",
        "Can I get a mortgage with 10% down payment?",
    ],
    "answer": [
        "The current 30-year fixed mortgage rate is approximately 6.5% APR as of March 2025.",
        "For a conventional mortgage, a minimum credit score of 620 is typically required.",
        "Yes, you can get a mortgage with 10% down payment. However, a down payment below 20% typically requires Private Mortgage Insurance (PMI).",
    ],
    "contexts": [
        ["Current 30-year fixed rate: 6.5% APR. Last updated: March 1, 2025."],
        ["Minimum credit score for conventional mortgage: 620. FHA loans accept scores as low as 580."],
        ["Down payment requirements: Conventional loans accept as low as 3% down. PMI required for LTV > 80%."],
    ],
    "ground_truth": [
        "The 30-year fixed mortgage rate is 6.5%.",
        "Minimum credit score for conventional mortgage is 620.",
        "Yes, mortgages are available with 10% down but require PMI.",
    ],
}

dataset = Dataset.from_dict(data)

# Evaluate
result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)

print(result)
```

Output:
```
{'faithfulness': 0.92, 'answer_relevancy': 0.88,
 'context_precision': 0.85, 'context_recall': 0.90}
```

### Thresholds for Banking

```python
BANKING_EVAL_THRESHOLDS = {
    'faithfulness': {
        'minimum': 0.90,      # Claims must be 90%+ supported by sources
        'critical_minimum': 0.95,  # For financial advice, 95%+
    },
    'answer_relevancy': {
        'minimum': 0.80,      # Answers should be relevant
    },
    'context_precision': {
        'minimum': 0.75,      # Retrieved docs should be mostly relevant
    },
    'context_recall': {
        'minimum': 0.80,      # Should find most relevant documents
    },
}
```

## DeepEval Framework

DeepEval provides comprehensive LLM testing:

```python
from deeval import assert_test
from deeval.metrics import (
    AnswerRelevancyMetric,
    FaithfulnessMetric,
    HallucinationMetric,
    BiasMetric,
    ToxicityMetric,
)
from deeval.test_case import LLMTestCase

def test_mortgage_advice_quality():
    """Evaluate a mortgage advice response."""

    test_case = LLMTestCase(
        input="Can I afford a $450,000 home with a $95,000 salary?",
        actual_output=(
            "Based on your salary of $95,000, a $450,000 mortgage would result in "
            "a monthly payment of approximately $2,844. Your estimated debt-to-income "
            "ratio would be 36%, which is within acceptable limits for most lenders. "
            "However, this is for informational purposes only and not financial advice. "
            "Please consult with a licensed mortgage advisor."
        ),
        retrieval_context=[
            "30-year fixed rate: 6.5% APR",
            "Monthly payment for $450k at 6.5%: $2,844",
            "Maximum DTI for conventional: 43%",
        ],
        expected_output=(
            "The monthly payment would be approximately $2,844 with a DTI of 36%."
        ),
    )

    # Faithfulness: are claims supported by context?
    assert_test(test_case, [FaithfulnessMetric(threshold=0.9)])

    # Answer relevancy: does it address the query?
    assert_test(test_case, [AnswerRelevancyMetric(threshold=0.8)])

    # Hallucination: no fabricated information?
    assert_test(test_case, [HallucinationMetric(threshold=0.1)])

    # Bias: no demographic bias?
    assert_test(test_case, [BiasMetric(threshold=0.1)])
```

## Custom Banking Evaluators

```python
from deeval.metrics import BaseMetric

class ComplianceMetric(BaseMetric):
    """Evaluate whether response meets banking compliance requirements."""

    def __init__(self, product_line: str):
        self.product_line = product_line
        self.threshold = 1.0  # Must be fully compliant

    def measure(self, test_case: LLMTestCase):
        score = 1.0
        reasons = []

        # Check for disclaimer
        disclaimer_terms = [
            "informational purposes", "not financial advice",
            "consult", "professional advisor", "for reference"
        ]
        has_disclaimer = any(
            term in test_case.actual_output.lower()
            for term in disclaimer_terms
        )
        if not has_disclaimer:
            score = 0.0
            reasons.append("Missing compliance disclaimer")

        # Check for guarantee language
        guarantee_terms = [
            "guaranteed", "will be approved", "definitely qualify",
            "no risk", "sure to"
        ]
        has_guarantee = any(
            term in test_case.actual_output.lower()
            for term in guarantee_terms
        )
        if has_guarantee:
            score = 0.0
            reasons.append("Contains guarantee language")

        # Check for accurate rate information
        if self.product_line == "mortgage":
            rate_match = re.search(r'(\d+\.?\d*)%', test_case.actual_output)
            if rate_match:
                rate = float(rate_match.group(1))
                if rate < 1.0 or rate > 20.0:
                    score = 0.0
                    reasons.append(f"Unrealistic rate: {rate}%")

        self.score = score
        self.reason = "; ".join(reasons) if reasons else "Compliant"
        self.success = score >= self.threshold
        return self.score

    def is_successful(self):
        return self.success

    @property
    def __name__(self):
        return "Compliance"
```

## Evaluation Pipeline

```python
class EvaluationPipeline:
    """Run evaluations on a regular schedule."""

    def __init__(self):
        self.dataset = load_golden_dataset()
        self.metrics = [
            faithfulness,
            answer_relevancy,
            context_precision,
            context_recall,
            ComplianceMetric(product_line="mortgage"),
        ]

    def run_evaluation(self, model: str, version: str):
        """Evaluate a model version and record results."""
        results = {}

        for row in self.dataset:
            # Generate response
            response = call_llm(model=model, prompt=row["prompt"])

            # Evaluate
            test_case = LLMTestCase(
                input=row["question"],
                actual_output=response,
                retrieval_context=row.get("contexts", []),
                expected_output=row.get("expected", ""),
            )

            for metric in self.metrics:
                metric.measure(test_case)
                metric_name = metric.__name__
                if metric_name not in results:
                    results[metric_name] = []
                results[metric_name].append(metric.score)

        # Aggregate
        summary = {
            metric: sum(scores) / len(scores)
            for metric, scores in results.items()
        }

        # Record to observability
        record_evaluation_results(model, version, summary)

        return summary

    def compare_models(self, model_a: str, model_b: str):
        """Compare two models side by side."""
        results_a = self.run_evaluation(model_a, "current")
        results_b = self.run_evaluation(model_b, "candidate")

        comparison = {}
        for metric in results_a:
            delta = results_b[metric] - results_a[metric]
            comparison[metric] = {
                "current": results_a[metric],
                "candidate": results_b[metric],
                "delta": delta,
                "improved": delta > 0,
            }

        return comparison
```

## Evaluation Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  LLM EVALUATION RESULTS - Mortgage Advisor                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Model: gpt-4-turbo-2024-04-09                               │
│  Evaluation Date: 2025-03-15                                 │
│  Dataset: Golden Mortgage QA (234 questions)                 │
│                                                              │
│  ┌─────────────────┬───────────┬───────────┬───────────────┐ │
│  │ Metric          │  Score    │  Threshold│  Status       │ │
│  ├─────────────────┼───────────┼───────────┼───────────────┤ │
│  │ Faithfulness    │  0.94     │  > 0.90   │  PASS         │ │
│  │ Answer Relev.   │  0.91     │  > 0.80   │  PASS         │ │
│  │ Context Prec.   │  0.87     │  > 0.75   │  PASS         │ │
│  │ Context Recall   │  0.88     │  > 0.80   │  PASS         │ │
│  │ Compliance      │  1.00     │  = 1.00   │  PASS         │ │
│  │ Hallucination    │  0.03     │  < 0.10   │  PASS         │ │
│  │ Bias            │  0.02     │  < 0.10   │  PASS         │ │
│  └─────────────────┴───────────┴───────────┴───────────────┘ │
│                                                              │
│  Trend (last 4 evaluations):                                 │
│  Faithfulness:  0.91 -> 0.92 -> 0.93 -> 0.94 [Improving]    │
│  Compliance:    1.00 -> 1.00 -> 1.00 -> 1.00 [Stable]        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Common LLM Evaluation Mistakes

1. **Evaluating only on easy questions**: Include edge cases, adversarial prompts, and ambiguous queries in your evaluation set.

2. **No compliance metric**: In banking, a compliant but irrelevant answer is better than an accurate but non-compliant one. Always test compliance.

3. **Static evaluation dataset**: As products change, the evaluation dataset must be updated. Review it quarterly.

4. **Using only LLM-as-judge**: LLMs evaluating LLMs can miss subtle errors. Combine automated evaluation with human review.

5. **No baseline comparison**: "Faithfulness score of 0.94" means nothing without knowing the previous score or the competitor's score.

6. **Evaluating in isolation**: Run evaluations as part of CI/CD, not as a quarterly exercise. Every model change should trigger evaluation.
