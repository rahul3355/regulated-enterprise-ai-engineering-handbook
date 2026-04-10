# Quality Gates for Banking GenAI CI/CD

## What Are Quality Gates?

Quality gates are automated checks that must pass before code progresses through the CI/CD pipeline. They enforce minimum quality standards objectively, removing human judgment from release decisions.

## Quality Gate Stages

```
┌─────────────────────────────────────────────────────────────┐
│  QUALITY GATE PIPELINE                                       │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  PRE-COMMIT  │───>│  CI GATE     │───>│  STAGING     │  │
│  │  GATE        │    │  GATE        │    │  GATE        │  │
│  │              │    │              │    │              │  │
│  │  Lint        │    │  Unit tests  │    │  E2E tests   │  │
│  │  Format      │    │  Int. tests  │    │  Perf tests  │  │
│  │  Type check  │    │  Security    │    │  LLM eval    │  │
│  │  Secrets     │    │  Build       │    │  SLO verify  │  │
│  │  PII check   │    │  Coverage    │    │  Contract    │  │
│  └──────────────┘    └──────────────┘    └──────┬───────┘  │
│                                                  │          │
│                                           ┌──────┴───────┐  │
│                                           │  PROD GATE   │  │
│                                           │              │  │
│                                           │  Canary      │  │
│                                           │  Synthetic   │  │
│                                           │  SLO check   │  │
│                                           └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Gate 1: Pre-Commit

```yaml
# .pre-commit-config.yaml
repos:
  # Code formatting
  - repo: https://github.com/psf/black
    rev: 24.2.0
    hooks:
      - id: black
        args: ["--line-length", "100"]

  # Linting
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: ["--fix", "--exit-non-zero-on-fix"]

  # Type checking
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: ["pydantic"]

  # Secret scanning
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  # PII detection
  - repo: local
    hooks:
      - id: pii-check
        name: Check for PII in test data
        entry: python scripts/check_pii.py
        language: system
        files: "tests/.*\\.py$"
```

## Gate 2: CI

```yaml
# .github/workflows/ci.yml
name: CI Quality Gate

on:
  pull_request:
    branches: [main]

jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Unit Tests
        run: |
          pytest tests/unit/ \
            --cov=src \
            --cov-fail-under=85 \
            --cov-branch \
            --junitxml=unit-results.xml

      - name: Integration Tests
        run: |
          pytest tests/integration/ \
            --junitxml=integration-results.xml

      - name: Security Scan
        run: |
          bandit -r src/ -f json -o bandit-report.json
          pip-audit -r requirements.txt

      - name: Contract Tests
        run: |
          pytest tests/contracts/

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
          fail_ci_if_error: true

      - name: Check Quality Gate
        run: |
          # All checks must pass
          python scripts/check_quality_gate.py \
            --unit-results unit-results.xml \
            --integration-results integration-results.xml \
            --security-scan bandit-report.json \
            --coverage-report coverage.xml
```

### Quality Gate Thresholds

```python
# scripts/check_quality_gate.py
QUALITY_THRESHOLDS = {
    'unit_test_pass_rate': 1.0,        # 100% of unit tests must pass
    'integration_test_pass_rate': 1.0,  # 100% of integration tests must pass
    'coverage_minimum': 0.85,           # 85% line coverage
    'coverage_branch_minimum': 0.75,    # 75% branch coverage
    'security_critical': 0,             # Zero critical security findings
    'security_high': 0,                 # Zero high security findings
    'contract_test_pass_rate': 1.0,     # 100% contract tests must pass
    'llm_eval_faithfulness': 0.90,      # Minimum faithfulness score
    'llm_eval_compliance': 1.0,         # Must be fully compliant
}
```

## Gate 3: Staging

```yaml
# .github/workflows/staging-gate.yml
name: Staging Quality Gate

on:
  push:
    branches: [main]

jobs:
  staging-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: ./scripts/deploy-staging.sh

      - name: Wait for Deployment
        run: ./scripts/wait-for-deployment.sh staging

      - name: E2E Tests
        run: pytest tests/e2e/ -v

      - name: Performance Tests
        run: |
          k6 run tests/performance/api.js \
            --out json=perf-results.json

      - name: LLM Evaluation
        run: |
          python scripts/run_llm_evaluation.py \
            --model gpt-4-turbo \
            --dataset golden-mortgage-qa \
            --output eval-results.json

      - name: Verify SLOs
        run: |
          python scripts/verify_slos.py \
            --endpoint https://staging.bank.example \
            --duration 5m

      - name: Check Staging Gate
        run: |
          python scripts/check_staging_gate.py \
            --e2e-results e2e-results.xml \
            --perf-results perf-results.json \
            --eval-results eval-results.json
```

## Gate 4: Production (Canary)

```yaml
# .github/workflows/prod-gate.yml
name: Production Quality Gate

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release'
        required: true

jobs:
  canary-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Canary (5% traffic)
        run: ./scripts/deploy-canary.sh ${{ inputs.version }}

      - name: Wait for Canary Warmup
        run: sleep 300  # 5 minutes

      - name: Check Canary SLOs
        run: |
          python scripts/check_canary_slos.py \
            --version ${{ inputs.version }} \
            --traffic-percentage 5 \
            --duration 15m

      - name: Check Error Rate
        run: |
          ERROR_RATE=$(python scripts/get_error_rate.py --version ${{ inputs.version }})
          if (( $(echo "$ERROR_RATE > 0.005" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE"
            ./scripts/rollback-canary.sh
            exit 1
          fi

      - name: Check Latency SLO
        run: |
          P95=$(python scripts/get_p95_latency.py --version ${{ inputs.version }})
          if (( $(echo "$P95 > 3.0" | bc -l) )); then
            echo "P95 latency too high: ${P95}s"
            ./scripts/rollback-canary.sh
            exit 1
          fi

      - name: Run Synthetic Checks
        run: |
          python scripts/run_synthetic_checks.py \
            --environment production-canary

  full-rollout:
    needs: canary-deploy
    runs-on: ubuntu-latest
    steps:
      - name: Full Rollout
        run: ./scripts/deploy-production.sh ${{ inputs.version }}
```

## Gate Enforcement

```python
class QualityGateEnforcer:
    """Enforce quality gates and block releases that do not meet standards."""

    def __init__(self, thresholds: dict):
        self.thresholds = thresholds
        self.violations = []

    def check_all(self, results: dict) -> bool:
        """Run all quality gate checks."""
        checks = [
            self._check_test_results(results.get("tests")),
            self._check_coverage(results.get("coverage")),
            self._check_security(results.get("security")),
            self._check_llm_evaluation(results.get("llm_eval")),
        ]

        if self.violations:
            self._block_release()
            return False

        return True

    def _check_test_results(self, results):
        if results and results["pass_rate"] < self.thresholds["unit_test_pass_rate"]:
            self.violations.append(
                f"Test pass rate {results['pass_rate']:.2%} below "
                f"threshold {self.thresholds['unit_test_pass_rate']:.2%}"
            )

    def _check_coverage(self, results):
        if results and results["line_coverage"] < self.thresholds["coverage_minimum"]:
            self.violations.append(
                f"Coverage {results['line_coverage']:.2%} below "
                f"threshold {self.thresholds['coverage_minimum']:.2%}"
            )

    def _check_security(self, results):
        if results and results["critical_findings"] > self.thresholds["security_critical"]:
            self.violations.append(
                f"Found {results['critical_findings']} critical security findings"
            )

    def _check_llm_evaluation(self, results):
        if results:
            if results.get("faithfulness", 0) < self.thresholds["llm_eval_faithfulness"]:
                self.violations.append(
                    f"LLM faithfulness {results['faithfulness']:.2f} below "
                    f"threshold {self.thresholds['llm_eval_faithfulness']:.2f}"
                )
            if results.get("compliance", 1.0) < self.thresholds["llm_eval_compliance"]:
                self.violations.append(
                    f"LLM compliance score {results['compliance']:.2f} below "
                    f"threshold {self.thresholds['llm_eval_compliance']:.2f}"
                )

    def _block_release(self):
        """Block the release and notify."""
        print("QUALITY GATE FAILED:")
        for violation in self.violations:
            print(f"  - {violation}")
        notify_quality_gate_failure(self.violations)
```

## Common Quality Gate Mistakes

1. **Gates that are too strict**: If gates fail 50% of the time, engineers learn to bypass them. Set achievable thresholds and raise them gradually.

2. **Gates that are too lenient**: A gate that always passes provides no value. Review gate effectiveness quarterly.

3. **No escalation path**: When a gate fails, what happens? Define the process: who investigates, who approves exceptions, how to fix.

4. **Exception process is too easy**: Gate exceptions should require leadership approval and have an expiration date.

5. **Gates not updated with changing requirements**: As the system matures, thresholds should tighten. A 60% coverage threshold makes no sense for a mature codebase.

6. **Missing GenAI-specific gates**: Traditional gates (coverage, linting) are necessary but insufficient. Add LLM evaluation and compliance gates.
