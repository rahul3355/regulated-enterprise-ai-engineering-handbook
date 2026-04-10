# Skill: CI/CD Pipeline Design

## Core Principles

1. **Fast Feedback First** — The first 2 minutes of a pipeline must catch obvious failures (lint, type-check, unit tests). No one waits 20 minutes to learn they forgot a semicolon.
2. **Gated Progression** — Each stage must pass before the next runs. Failed unit tests should never trigger integration tests. Failed security scans should never reach production.
3. **Everything as Code** — Pipeline definitions live in Git, alongside the code they build. No Jenkins freestyle jobs. No manual deployment steps.
4. **Reproducible Artifacts** — A commit SHA must always produce the same artifact. If you rebuild from the same SHA, you get the same binary/image. This is non-negotiable for banking audits.
5. **Security Scanning at Every Stage** — SAST, DAST, dependency scanning, container scanning, and secret scanning are not optional. They are quality gates.

## Mental Models

### The Banking CI/CD Pipeline
```
┌──────────────────────────────────────────────────────────────────────┐
│                     CI/CD Pipeline Stages                            │
│                                                                      │
│  Commit ──▶ Stage 1: Quality Gate (2-5 min)                         │
│             ├── Lint (ruff, eslint, golangci-lint)                   │
│             ├── Type check (mypy, tsc)                               │
│             ├── Unit tests (pytest, jest, go test)                   │
│             ├── Secret scanning (gitleaks, trufflehog)              │
│             └── License check                                       │
│                                                                      │
│  Merge ──▶ Stage 2: Integration (5-15 min)                          │
│             ├── Build artifacts (Docker image, wheel, JAR)           │
│             ├── Integration tests (testcontainers)                   │
│             ├── API contract tests (Pact)                            │
│             ├── Dependency vulnerability scan (Trivy, Snyk)         │
│             └── Container image scan (Trivy, Prisma)                 │
│                                                                      │
│  Deploy to Dev ──▶ Stage 3: Deploy & Verify (5-10 min)              │
│             ├── Deploy to dev namespace via ArgoCD                   │
│             ├── Smoke tests (health checks, basic API calls)        │
│             ├── E2E tests (Playwright, Cypress)                      │
│             └── Performance baseline check                           │
│                                                                      │
│  Promote to UAT ──▶ Stage 4: Compliance Gate                        │
│             ├── Security sign-off (automated scan results)           │
│             ├── Compliance checklist (auto-generated report)         │
│             ├── Change request created (ServiceNow integration)      │
│             └── Manual approval (security/compliance team)           │
│                                                                      │
│  Deploy to Prod ──▶ Stage 5: Production Deployment                  │
│             ├── Blue-green or canary deployment                      │
│             ├── Automated canary analysis (Prometheus metrics)       │
│             ├── Post-deployment smoke tests                          │
│             └── Rollback on SLO breach                               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Pipeline Toolchain Options
```
GitHub Actions + ArgoCD (Modern, recommended for new projects)
├── GitHub Actions: CI (build, test, scan)
├── GitHub Actions: Push image to internal registry
├── ArgoCD: CD (GitOps sync to OpenShift)
└── Tekton: Advanced pipelines (complex multi-stage workflows)

Tekton (Bank-standard, already deployed)
├── Tekton Pipeline: Full CI/CD in-cluster
├── Triggers: Auto-start on PR/merge
├── Tasks: Reusable pipeline steps
└── Workspaces: Shared storage between tasks

Harness (Enterprise CD platform)
├── Continuous Integration
├── Continuous Delivery with verification stages
├── Feature flags integration
└── Automated rollback on failure
```

### The Pipeline Design Checklist
```
□ Pipeline triggers on PR and merge to main
□ Lint and type-check run first (fast feedback)
□ Unit tests run in parallel (sharding)
□ Build produces a single versioned artifact
□ All artifacts pushed to internal registry (Quay/Harbor)
□ Container image scanned for vulnerabilities (Trivy)
□ Dependency scan for known CVEs (Snyk/Dependabot)
□ Secret scanning on every commit (gitleaks)
□ Integration tests run against real dependencies (testcontainers)
□ E2E tests run against deployed dev environment
□ Deployment is declarative (ArgoCD/Helm/Kustomize)
□ Production deployment requires manual approval
□ Rollback is automated on health check failure
□ Pipeline artifacts are retained for audit (build logs, scan reports)
□ Pipeline itself is version-controlled and reviewed
□ Pipeline has timeouts on every stage
□ Pipeline sends notifications (Slack/Teams) on failure
□ Pipeline run time is monitored (< 30 min total)
```

## Step-by-Step Approach

### 1. GitHub Actions CI Pipeline (Python/TypeScript)

```yaml
# .github/workflows/ci.yaml
name: CI Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

# Prevent concurrent runs on the same PR
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: registry.internal/banking
  IMAGE_NAME: genai/rag-service
  PYTHON_VERSION: "3.11"
  NODE_VERSION: "20"

jobs:
  # ─── Stage 1: Quality Gate ───
  lint-and-type-check:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements-dev.txt

      - name: Lint (ruff)
        run: ruff check . --output-format=github

      - name: Format check (ruff)
        run: ruff format --check .

      - name: Type check (mypy)
        run: mypy src/ --config-file pyproject.toml

      - name: Frontend lint (if applicable)
        if: hashFiles('frontend/package.json') != ''
        run: |
          cd frontend
          npm ci
          npm run lint
          npm run type-check

  # ─── Stage 1: Security Scan ───
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Secret scanning (gitleaks)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Dependency vulnerability scan
        run: |
          pip install safety
          safety check --json --output safety-report.json || true

      - name: Upload security reports
        uses: actions/upload-artifact@v4
        with:
          name: security-reports
          path: |
            safety-report.json
            gitleaks-report.json

  # ─── Stage 1: Unit Tests ───
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    timeout-minutes: 10
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Run tests (sharded)
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=html \
            --junitxml=test-results.xml \
            --numprocesses=auto \
            --slices=4 --slice=${{ matrix.shard }}

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-shard-${{ matrix.shard }}
          path: coverage.xml

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-shard-${{ matrix.shard }}
          path: test-results.xml

  # ─── Stage 2: Build & Integration Tests ───
  build-and-integration:
    name: Build & Integration
    needs: [lint-and-type-check, security-scan, unit-tests]
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to internal registry
        uses: docker/login-action@v3
        with:
          registry: registry.internal
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
          build-args: |
            BUILD_COMMIT=${{ github.sha }}
            BUILD_TIMESTAMP=${{ github.event.head_commit.timestamp }}

      - name: Container vulnerability scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          exit-code: "1"
          severity: CRITICAL,HIGH

      - name: Run integration tests
        run: |
          docker compose -f docker-compose.test.yml up -d
          pip install -r requirements-dev.txt
          pytest tests/integration/ --junitxml=integration-results.xml
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0

      - name: Upload integration test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: integration-test-results
          path: integration-results.xml
```

### 2. Tekton Pipeline (In-Cluster CI/CD)

```yaml
# tekton/pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: genai-service-pipeline
  namespace: genai-platform
spec:
  params:
    - name: repo-url
      type: string
    - name: revision
      type: string
      default: main
    - name: image-name
      type: string
      default: registry.internal/banking/genai/rag-service
    - name: deploy-namespace
      type: string
      default: genai-platform
  workspaces:
    - name: shared-workspace
    - name: docker-credentials
  tasks:
    # Step 1: Clone source
    - name: fetch-source
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.revision)

    # Step 2: Run tests
    - name: run-tests
      runAfter: [fetch-source]
      taskRef:
        name: python-test
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: test-command
          value: ["pytest", "tests/", "-v", "--cov=src", "--cov-fail-under=80"]

    # Step 3: Build and push image
    - name: build-and-push
      runAfter: [run-tests]
      taskRef:
        name: buildah
      workspaces:
        - name: source
          workspace: shared-workspace
        - name: dockerconfig
          workspace: docker-credentials
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-source.results.commit)
        - name: DOCKERFILE
          value: ./Dockerfile
        - name: CONTEXT
          value: .
        - name: TLSVERIFY
          value: "true"
        - name: EXTRA_ARGS
          value: ["--label=org.opencontainers.image.source=$(params.repo-url)"]

    # Step 4: Scan image
    - name: scan-image
      runAfter: [build-and-push]
      taskRef:
        name: trivy-scanner
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-source.results.commit)
        - name: SEVERITY
          value: CRITICAL,HIGH
        - name: FAIL_ON_DETECTION
          value: "true"

    # Step 5: Deploy to dev
    - name: deploy-dev
      runAfter: [scan-image]
      taskRef:
        name: openshift-client
      params:
        - name: SCRIPT
          value: |
            oc set image deployment/rag-service \
              rag-service=$(params.image-name):$(tasks.fetch-source.results.commit) \
              -n $(params.deploy-namespace)
            oc rollout status deployment/rag-service -n $(params.deploy-namespace) --timeout=300s

    # Step 6: Run smoke tests
    - name: smoke-tests
      runAfter: [deploy-dev]
      taskRef:
        name: python-test
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: test-command
          value: ["pytest", "tests/smoke/", "-v"]
        - name: env-vars
          value:
            - BASE_URL=https://rag-service-dev.apps.bank.internal

    # Step 7: Deploy to production (manual approval gate)
    - name: deploy-prod
      runAfter: [smoke-tests]
      taskRef:
        name: manual-approval
      params:
        - name: MESSAGE
          value: "Approve deployment to production for $(tasks.fetch-source.results.commit)"
```

### 3. ArgoCD Sync Waves for Ordered Deployment

```yaml
# argocd/sync-waves.yaml
# Wave 0: ConfigMaps and Secrets (must exist before pods start)
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rag-service-config
  namespace: genai-platform
  annotations:
    argocd.argoproj.io/sync-wave: "0"
data:
  DATABASE_HOST: pg-primary.genai-db.svc.cluster.local
  LOG_LEVEL: info

---
# Wave 1: Database migrations
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: genai-platform
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: registry.internal/banking/genai/rag-service:v2.3.1
          command: ["python", "-m", "alembic", "upgrade", "head"]
      restartPolicy: Never

---
# Wave 2: Application deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-service
  namespace: genai-platform
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  # ... deployment spec

---
# Wave 3: Post-deployment verification
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  namespace: genai-platform
  annotations:
    argocd.argoproj.io/sync-wave: "3"
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: smoke-test
          image: registry.internal/banking/genai/rag-service:v2.3.1
          command: ["python", "-m", "pytest", "tests/smoke/", "-v"]
          env:
            - name: BASE_URL
              value: "https://rag-service.genai-platform.svc.cluster.local"
      restartPolicy: Never
```

### 4. Automated Rollback on SLO Breach

```yaml
# argocd/analysis.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisRun
metadata:
  name: canary-analysis
  namespace: genai-platform
spec:
  metrics:
    - name: error-rate
      successCondition: result[0] < 0.01
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring.svc:9090
          query: >
            sum(rate(http_requests_total{service="rag-service",status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="rag-service"}[5m]))

    - name: latency-p99
      successCondition: result[0] < 2000
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring.svc:9090
          query: >
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{service="rag-service"}[5m]))
              by (le))

    - name: availability
      successCondition: result[0] >= 0.999
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring.svc:9090
          query: >
            sum(up{service="rag-service"}) / count(up{service="rag-service"})
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Sequential test execution | 30+ min pipeline, slow feedback | Parallelize with sharding/matrix |
| No pipeline timeouts | Pipeline hangs indefinitely, blocks releases | Set timeout on every stage |
| Flaky tests not quarantined | Pipeline becomes untrusted, ignored failures | Quarantine flaky tests, fix or remove |
| Building on every commit (no caching) | Wasted compute, slow builds | Use Docker layer caching, pip/wheels cache |
| Security scans after deployment | Vulnerable code reaches production | Security scans as quality gates before deploy |
| Manual production deployments | Human error, no audit trail, inconsistent process | Use GitOps (ArgoCD) for all deployments |
| No artifact versioning | Can't trace which code is in production | Tag images with commit SHA, not just `latest` |
| Ignoring pipeline cost | Hundreds of devs triggering expensive pipelines | Use concurrency groups, cancel redundant runs |
| Not testing the deployment itself | Tests pass but deployment fails | Run smoke tests against the actual deployed service |
| No rollback strategy | Failed deployments stay broken | Automated rollback on health check failure |

## Banking-Specific Concerns

1. **Change Management Integration** — Production deployments must create change requests in ServiceNow or equivalent. Automate this via API: pipeline creates the CR, attaches scan reports, and requests approval.
2. **Segregation of Duties** — The person who merges the PR should not be the person who approves production deployment. Configure separate approval gates.
3. **Audit Evidence** — Every pipeline run must produce evidence: build logs, scan reports, test results, deployment timestamps. Retain for 7+ years.
4. **Deployment Windows** — Many banks have restricted deployment windows (e.g., no deployments during month-end close). Configure pipeline schedules accordingly.
5. **Dual Control** — Some changes require two approvers. Configure ArgoCD manual approval gates with multiple required approvals.
6. **Emergency Hotfix Pipeline** — A separate, expedited pipeline for production incidents. Still requires security scanning but with reduced approval chain. Must produce a post-hoc change request.

## GenAI-Specific Concerns

1. **Model Version Pinning** — The pipeline must record which model version (name, provider, version) was used during testing. A model API change can break the service without code changes.
2. **Evaluation Tests** — Include GenAI evaluation metrics (accuracy, hallucination rate, guardrail effectiveness) in the test suite. Use frameworks like RAGAS or DeepEval.
3. **Prompt Regression Testing** — Changes to prompt templates should trigger a full evaluation suite. Treat prompts as code: version them, test them, review them.
4. **Token Cost Monitoring** — Include token cost estimation in the pipeline. A new prompt template might be 3x more expensive. Flag this before deployment.
5. **Embedding Pipeline** — When deploying a new embedding model, the pipeline must trigger re-embedding of all documents. This is a separate, long-running job that must complete before the new model goes live.
6. **Guardrail Testing** — Include adversarial prompt tests in the pipeline. Verify that prompt injection, jailbreak attempts, and harmful content generation are blocked.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Pipeline duration (p95) | > 20 min | Slow pipelines reduce developer velocity |
| Pipeline failure rate | > 10% | Broken builds waste engineering time |
| Flaky test rate | > 2% | Erodes trust in the pipeline |
| Time from merge to production | > 1 hour (for approved changes) | Delivery bottleneck |
| Security scan findings (critical) | > 0 | Blocks deployment, must be fixed |
| Artifact build cache hit rate | < 80% | Wasted compute and time |
| Deployment success rate | < 99% | Unreliable deployments |
| Rollback rate | > 5% of deployments | Quality issues reaching production |
| Change failure rate | > 15% | Too many changes causing incidents |
| Mean time to recovery | > 1 hour | Slow rollback process |

## Interview Questions

1. Design a CI/CD pipeline for a GenAI RAG service that must pass security scanning before reaching production.
2. How do you handle flaky tests in a CI pipeline? When do you quarantine vs. fix vs. remove?
3. What is the difference between blue-green and canary deployments? When would you use each in a banking context?
4. How do you ensure reproducible builds in a CI/CD pipeline?
5. What security scanning tools would you include in a pipeline and at what stages?
6. How would you design an automated rollback mechanism based on SLO monitoring?
7. Explain how ArgoCD sync waves work and why they matter for ordered deployments.
8. How do you balance pipeline speed with thoroughness in a regulated environment?

## Hands-On Exercise

### Exercise: Design a CI/CD Pipeline for a Banking GenAI Service

**Problem:** Your team is building a GenAI service that answers employee questions about HR policies using RAG. The service must:
- Pass unit tests, integration tests, and E2E tests
- Pass security scanning (SAST, DAST, dependency, container)
- Deploy to dev automatically, UAT with approval, and prod with dual approval
- Support automated rollback if error rate exceeds 1% after deployment
- Produce audit artifacts for every pipeline run

**Constraints:**
- The team uses GitHub for source control and OpenShift for deployment
- The banking security team requires Trivy scans with zero CRITICAL/HIGH vulnerabilities
- The compliance team requires change requests in ServiceNow for every production deployment
- The pipeline must complete in under 30 minutes end-to-end (dev deployment)
- The service uses Python (FastAPI) backend and React (Next.js) frontend

**Expected Output:**
- GitHub Actions workflow file(s) for CI
- Tekton or ArgoCD configuration for CD
- A deployment runbook with pre-checks, execution steps, and rollback criteria
- A list of quality gates and their pass/fail criteria
- Audit artifact list and retention policy

**Hints:**
- Use matrix strategy to parallelize independent test suites
- Use ArgoCD sync waves for ordered deployment
- Use Prometheus-based canary analysis for automated rollback
- Separate the pipeline into fast (PR) and slow (merge) paths

**Extension:**
- Add GenAI evaluation tests (RAGAS metrics) to the pipeline
- Design an emergency hotfix pipeline with reduced approval chain
- Implement a pipeline cost dashboard showing compute time and resource usage per run

---

**Related files:**
- `cicd-devops/github-actions.md` — GitHub Actions guide
- `cicd-devops/tekton-pipelines.md` — Tekton pipeline design
- `cicd-devops/argocd-gitops.md` — ArgoCD and GitOps
- `cicd-devops/deployment-strategies.md` — Blue-green, canary, rolling
- `security/container-security.md` — Container scanning
- `testing-and-quality/test-pyramid.md` — Testing strategy
- `skills/genai-guardrails.md` — GenAI guardrail testing
