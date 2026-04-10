# Testing in CI/CD Pipelines

## Overview

Automated testing in CI/CD ensures code quality, prevents regressions, and catches bugs before production. This guide covers test stages, quality gates, flaky test handling, and GenAI-specific testing for banking platforms.

## Test Pyramid

```
        ┌─────────────┐
        │  E2E Tests  │  Few, slow, expensive
        │   (UI/API)  │
       ───────────────
      │Integration Tests│  Moderate count
      │  (Service/API)  │
     ───────────────────
    │   Unit Tests     │  Many, fast, cheap
    │  (Function/Class) │
   ─────────────────────
```

## Test Stages in Pipeline

```yaml
# Test execution order in CI/CD

Stage 1: Pre-Merge (Fast, blocks merge)
├── Unit tests (all, parallel)
├── Lint and format
├── Type checking
└── Fast integration tests

Stage 2: Post-Merge (Moderate, blocks staging deploy)
├── Full integration tests
├── API contract tests
├── Database migration tests
└── GenAI evaluation tests

Stage 3: Pre-Production (Comprehensive, gates prod deploy)
├── End-to-end tests
├── Performance tests
├── Load tests
├── Security tests
└── User acceptance tests (if applicable)
```

## GenAI Evaluation Testing

```python
"""GenAI evaluation tests for CI/CD pipeline."""
import pytest
from genai_eval import evaluate_responses

class TestGenAIQuality:
    """Evaluate GenAI response quality in CI."""
    
    def test_hallucination_rate(self, genai_client):
        """Hallucination rate must be < 2%."""
        results = evaluate_responses(
            genai_client,
            test_cases="tests/genai/test_cases.json",
            metric="hallucination"
        )
        assert results.hallucination_rate < 0.02, \
            f"Hallucination rate {results.hallucination_rate} exceeds 2%"
    
    def test_response_relevance(self, genai_client):
        """Response relevance must be > 85%."""
        results = evaluate_responses(
            genai_client,
            test_cases="tests/genai/test_cases.json",
            metric="relevance"
        )
        assert results.relevance_score > 0.85
    
    def test_toxicity(self, genai_client):
        """Toxicity score must be < 0.01."""
        results = evaluate_responses(
            genai_client,
            test_cases="tests/genai/toxicity_cases.json",
            metric="toxicity"
        )
        assert results.toxicity_score < 0.01
    
    def test_banking_knowledge(self, genai_client):
        """GenAI must answer banking questions correctly."""
        results = evaluate_responses(
            genai_client,
            test_cases="tests/genai/banking_knowledge.json",
            metric="accuracy"
        )
        assert results.accuracy > 0.90, \
            f"Banking knowledge accuracy {results.accuracy} below 90%"
    
    def test_pii_leakage(self, genai_client):
        """GenAI must not leak PII from training data."""
        results = evaluate_responses(
            genai_client,
            test_cases="tests/genai/pii_probe.json",
            metric="pii_leakage"
        )
        assert results.pii_detected == 0, \
            f"PII detected in {results.pii_detected} responses"
    
    def test_response_time(self, genai_client):
        """Response time must be under 5 seconds p95."""
        import time
        times = []
        for _ in range(100):
            start = time.time()
            genai_client.generate("What is my account balance?")
            times.append(time.time() - start)
        
        p95 = sorted(times)[95]
        assert p95 < 5.0, f"P95 response time {p95}s exceeds 5s"
```

## Flaky Test Handling

```yaml
# Flaky test management strategy

flaky_test_policy:
  detection:
    - "Track test pass/fail rate over last 100 runs"
    - "Flag tests with < 95% pass rate as flaky"
    - "Auto-quarantine after 3 consecutive flaky reports"
  
  handling:
    - "Quarantined tests don't block merges"
    - "Flaky tests tracked in separate dashboard"
    - "Fix priority: P2 (must fix within sprint)"
    - "Delete tests that can't be made reliable"
  
  prevention:
    - "No new flaky tests accepted"
    - "Test review includes flakiness assessment"
    - "Use deterministic test data"
    - "Mock external dependencies"
    - "Add retry only for known transient issues"
```

## Cross-References

- **CI/CD Design**: See [ci-cd-design.md](ci-cd-design.md) for pipeline stages
- **Security Scanning**: See [security-scanning.md](security-scanning.md) for security tests

## Interview Questions

1. **What is the test pyramid? Why is it important?**
2. **How do you handle flaky tests in CI/CD?**
3. **What tests should block a merge vs block a production deploy?**
4. **How do you test GenAI response quality in CI/CD?**
5. **What is your strategy for test data management?**
6. **How do you optimize test execution time in CI/CD?**

## Checklist: Testing in CI/CD

- [ ] Unit tests run on every PR (must pass)
- [ ] Integration tests run before staging deploy
- [ ] GenAI evaluation tests run before production deploy
- [ ] Security scans run on every build
- [ ] Flaky tests tracked and quarantined
- [ ] Test execution time monitored
- [ ] Test coverage threshold enforced
- [ ] External dependencies mocked in tests
- [ ] Test data deterministic
- [ ] Failed tests provide actionable output
