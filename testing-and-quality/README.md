# Testing and Quality Engineering in Banking GenAI

## Philosophy

Quality is not a phase. It is a continuous engineering discipline that spans every stage of development, from the first line of code to the last production deployment. In banking GenAI, quality means two things simultaneously:

1. **Software Quality**: The system is reliable, secure, performant, and maintainable.
2. **AI Quality**: The model produces accurate, safe, fair, and helpful outputs.

Both are non-negotiable. A fast, secure application that gives incorrect financial advice is a regulatory liability. An accurate AI running on an unreliable platform is equally dangerous.

## The Testing Pyramid for GenAI

```
                    ┌─────────────────┐
                    │  Human Review   │  Subjective quality assessment
                    │  (Red Teaming)  │  Adversarial testing
                    └────────┬────────┘
                   ┌─────────┴─────────┐
                   │  LLM Evaluation   │  RAGAS, DeepEval, custom evals
                   │  (Golden Datasets)│  Automated quality scoring
                   └─────────┬─────────┘
                  ┌──────────┴──────────┐
                  │  Integration Tests  │  RAG pipeline, API contracts
                  │                     │  Multi-component flows
                  └──────────┬──────────┘
                 ┌───────────┴───────────┐
                 │     Unit Tests        │  Individual functions, classes
                 │  (Traditional + AI)   │  Prompt template tests
                 └───────────────────────┘
              ┌─────────────────────────────┐
              │    Static Analysis          │  Linting, type checking
              │    (SAST, Dependency Scan)  │  Security scanning
              └─────────────────────────────┘
```

The pyramid is wider at the base because unit tests and static analysis are cheap and fast. The apex (human review and red teaming) is expensive and should be used strategically.

## Quality Dimensions

### 1. Functional Correctness

Does the code do what it is supposed to do?

- Unit tests verify individual functions
- Integration tests verify component interactions
- End-to-end tests verify complete user flows

### 2. AI Output Quality

Does the AI produce accurate, relevant, and safe responses?

- Factual accuracy: Claims are supported by source documents
- Relevance: Responses address the user's actual question
- Completeness: Responses cover all aspects of the query
- Safety: Responses do not contain harmful or biased content

### 3. Security

Is the system protected against attacks?

- Input validation and sanitization
- Prompt injection defense
- Authentication and authorization
- Data encryption and PII protection

### 4. Performance

Does the system meet latency and throughput requirements?

- API response time within SLO
- Token usage within budget
- Resource utilization acceptable
- Scalability under load

### 5. Reliability

Does the system handle failures gracefully?

- Dependency failures do not cascade
- Degraded mode is functional
- Data is not lost during failures
- Recovery is automated where possible

## Quality Gates in CI/CD

```
┌─────────────────────────────────────────────────────────────┐
│  CI/CD QUALITY GATES                                        │
│                                                             │
│  Code Push ──> Pre-commit Checks                             │
│                ├── Linting (ruff, black, mypy)              │
│                ├── Secret scanning                           │
│                └── PII detection in test data                │
│                                                             │
│                ↓                                            │
│                CI Pipeline                                   │
│                ├── Unit tests (> 80% coverage)              │
│                ├── Integration tests                        │
│                ├── Security scan (SAST, DAST)               │
│                ├── Dependency audit                         │
│                └── Build verification                        │
│                                                             │
│                ↓                                            │
│                Staging Deployment                            │
│                ├── E2E tests                                │
│                ├── Performance tests                        │
│                ├── LLM evaluation (golden dataset)          │
│                ├── Contract tests                           │
│                └── Smoke tests                               │
│                                                             │
│                ↓                                            │
│                Production Deployment                         │
│                ├── Canary analysis                          │
│                ├── Synthetic monitoring                     │
│                └── Rollback if SLO breached                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## GenAI-Specific Testing Challenges

### Non-Deterministic Outputs

LLMs produce different outputs for the same input. Traditional assertions (expected == actual) do not work directly.

```python
# Traditional test (deterministic)
def test_add():
    assert add(2, 3) == 5  # Always passes or always fails

# GenAI test (non-deterministic)
def test_mortgage_advice():
    response = get_mortgage_advice(salary=95000, loan=450000)
    # Cannot assert exact response
    # Instead, assert properties of the response:
    assert response.recommendation is not None
    assert response.disclaimer_present
    assert len(response.recommendation) > 100
    assert_response_quality(response, min_quality=0.8)
```

### Evaluation Frameworks

Use established evaluation frameworks for LLM testing:

- **RAGAS**: Evaluates RAG pipelines on faithfulness, answer relevance, context precision, and context recall
- **DeepEval**: Comprehensive LLM evaluation with hallucination detection, bias detection, and toxicity scoring
- **Custom Evaluators**: Domain-specific evaluators for banking accuracy, compliance, and fairness

### Golden Datasets

Maintain curated datasets of known-good question-answer pairs for regression testing:

```python
GOLDEN_DATASET = [
    {
        "query": "What is the maximum loan-to-value ratio for a conventional mortgage?",
        "expected_topics": ["loan_to_value", "conventional_mortgage", "80_percent"],
        "min_quality": 0.85,
        "must_not_contain": ["guaranteed approval", "no credit check"],
    },
    {
        "query": "Can I get a mortgage with a credit score of 620?",
        "expected_topics": ["credit_score", "fha_loan", "conventional_minimum"],
        "min_quality": 0.80,
        "must_not_contain": ["you will be approved", "guaranteed"],
    },
]
```

## Banking-Specific Quality Requirements

### Regulatory Compliance Testing

Every AI response that contains financial advice must:
- Include appropriate disclaimers
- Not guarantee approval or outcomes
- Present information factually without pressure
- Comply with fair lending requirements

### Bias and Fairness Testing

Test AI responses across demographic segments to ensure consistent quality:

```python
async def test_fairness_across_segments():
    """Verify response quality is consistent across demographic segments."""
    query = "What mortgage products do you offer?"

    profiles = [
        {"age_group": "25-34", "income": "high"},
        {"age_group": "25-34", "income": "medium"},
        {"age_group": "55-64", "income": "high"},
        {"age_group": "55-64", "income": "medium"},
    ]

    scores = []
    for profile in profiles:
        response = await get_advice(query, profile)
        score = evaluate_response_quality(response)
        scores.append(score)

    # Quality should not vary by more than 10% across segments
    assert max(scores) - min(scores) < 0.10
```

## Quality Culture

Testing is not the QA team's responsibility. It is every engineer's responsibility. Code without tests is incomplete code. This principle applies equally to Python services, prompt templates, and evaluation datasets.

The most effective quality organizations invest in:
1. **Automated checks** that run on every change
2. **Fast feedback** so developers know immediately if they broke something
3. **Quality metrics** that are visible and actionable
4. **A blameless culture** where bugs are system failures, not people failures

## Table of Contents

- [Unit Testing](unit-testing.md)
- [Integration Testing](integration-testing.md)
- [Contract Testing](contract-testing.md)
- [End-to-End Testing](end-to-end-testing.md)
- [UI Testing](ui-testing.md)
- [API Testing](api-testing.md)
- [Performance Testing](performance-testing.md)
- [Load Testing](load-testing.md)
- [Security Testing](security-testing.md)
- [Chaos Engineering](chaos-engineering.md)
- [Test Data Management](test-data-management.md)
- [Test Environments](test-environments.md)
- [Regression Testing](regression-testing.md)
- [Snapshot Testing](snapshot-testing.md)
- [Mutation Testing](mutation-testing.md)
- [LLM Evaluation](llm-evaluation.md)
- [Red Teaming](red-teaming.md)
- [Adversarial Testing](adversarial-testing.md)
- [Golden Datasets](golden-datasets.md)
- [Human Review Workflows](human-review-workflows.md)
- [Quality Gates](quality-gates.md)
- [Release Readiness](release-readiness.md)
