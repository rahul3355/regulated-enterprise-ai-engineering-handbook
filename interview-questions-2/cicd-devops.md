# CI/CD & DevOps Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | CI/CD & DevOps (GitHub Actions, Tekton, Harness, Branching, Progressive Delivery, Testing, Rollbacks, Container Scanning, Secrets, IaC) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | cicd-devops/ (21 files: README, ci-cd-design, github-workflows, tekton, harness, branching-strategies, pull-request-process, testing-in-pipelines, progressive-delivery, production-approvals, release-processes, rollbacks, change-management, release-communication, drift-detection, container-scanning, security-scanning, sbom-generation, secrets-handling, infrastructure-as-code, terraform) |
| Citi Relevance | Citi uses GitHub, Tekton, and Harness for their CI/CD. Progressive delivery and automated testing gates are critical for deploying GenAI features safely in a regulated banking environment. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

### Q1: 🔵 What are the core principles of CI/CD and why do they matter specifically in a regulated banking environment?

**Strong Answer:**

CI/CD rests on seven core principles: automate everything, fail fast, reproducible builds, immutable artifacts, deploy small changes, always be rollback-ready, and bake compliance into the pipeline. In a consumer startup, "move fast and break things" might be acceptable. In banking, every principle must be balanced against regulatory requirements (SOX, PCI-DSS, GDPR) that demand audit trails, separation of duties, and change management processes.

The banking impact is significant. **Automate Everything** means no manual SCP deployments — every step from build to deploy must be codified so regulators can see exactly what ran. **Reproducible Builds** means the same git commit SHA always produces the same container image, which is essential when auditors ask "what exact code was running in production on March 15?" **Immutable Artifacts** means you build once, tag with the commit SHA, and deploy that same image to staging and production — you never rebuild for production.

For a GenAI platform at Citi, this also extends to AI-specific concerns: every model version, prompt template, and embedding index must be versioned and traceable. If a GenAI response causes a compliance issue, you need to be able to reproduce the exact build that produced it. The CI/CD pipeline becomes your compliance evidence engine.

```yaml
# Banking GenAI CI/CD Pipeline — core structure
stages:
  1. Source:        # PR with branch protection, 2+ approvals
  2. Build:         # Container image + SBOM, tagged with git SHA
  3. Test:          # Unit, integration, contract, GenAI evaluation
  4. Security:      # SAST, SCA, container scan, secret detection
  5. Deploy Staging: # Deploy, smoke tests, e2e tests
  6. Approval:       # Automated (low-risk) or CAB (high-risk)
  7. Deploy Prod:    # Canary/blue-green with auto-rollback
  8. Post-Deploy:    # Monitor SLOs, verify alerts, notify stakeholders
```

**Key Points to Hit:**
- [ ] Seven core principles (automate, fail fast, reproducible, immutable, small deploys, rollback-ready, compliance baked in)
- [ ] Regulatory requirements (SOX, PCI-DSS, GDPR) demand audit trails and change tracking
- [ ] Immutable artifacts: build once, tag with SHA, deploy everywhere
- [ ] GenAI-specific versioning (models, prompts, embedding indexes)
- [ ] CI/CD pipeline as compliance evidence engine for auditors

**Follow-Up Questions:**
1. How do you balance deployment speed with compliance gate requirements?
2. What specific audit evidence would a regulator expect from your CI/CD pipeline?
3. How do you handle emergency hotfixes that need to bypass normal CI/CD gates?

**Source:** `banking-genai-engineering-academy/cicd-devops/README.md`, `banking-genai-engineering-academy/cicd-devops/ci-cd-design.md`

---

### Q2: 🔵 Explain how GitHub Actions reusable workflows work and show a production-ready example for a banking application.

**Strong Answer:**

Reusable workflows in GitHub Actions let you define common CI/CD patterns once and call them from multiple repositories, eliminating copy-paste drift. They use `workflow_call` as the trigger type and support typed inputs, outputs, and secrets — critical for banking where every team should use the same security scanning and SBOM generation process.

In a banking context, you would have a central repository (e.g., `banking-ci-templates`) that owns the reusable workflows. Each GenAI service repo references these workflows, ensuring consistent security posture across all teams. The reusable workflow handles building, scanning, SBOM generation, and publishing — while the caller workflow orchestrates which reusable workflows to invoke and with what parameters.

```yaml
# .github/workflows/reusable-build.yml (central template repo)
name: Reusable Build
on:
  workflow_call:
    inputs:
      image_name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: Dockerfile
    outputs:
      image_digest:
        value: ${{ jobs.build.outputs.image_digest }}
    secrets:
      registry_username:
        required: true
      registry_password:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Build and push container image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ${{ inputs.dockerfile }}
          push: true
          tags: ${{ inputs.image_name }}:${{ github.sha }}
          labels: |
            org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
            org.opencontainers.image.revision=${{ github.sha }}

      - name: Generate SBOM (required for banking compliance)
        uses: anchore/sbom-action@v0
        with:
          image: ${{ inputs.image_name }}:${{ github.sha }}

      - name: Scan image — block on CRITICAL/HIGH
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ inputs.image_name }}:${{ github.sha }}
          format: sarif
          exit-code: '1'
          severity: CRITICAL,HIGH
```

The caller workflow in a GenAI service repo then becomes very clean:

```yaml
# .github/workflows/ci.yml (service repo)
jobs:
  build:
    uses: banking-ci-templates/.github/workflows/reusable-build.yml@main
    with:
      image_name: quay.io/banking/genai-api
    secrets:
      registry_username: ${{ secrets.REGISTRY_USER }}
      registry_password: ${{ secrets.REGISTRY_PASS }}
```

**Key Points to Hit:**
- [ ] `workflow_call` trigger enables reusable workflows
- [ ] Typed inputs, outputs, and secrets for strong contracts
- [ ] Central template repo ensures consistent security posture across teams
- [ ] SBOM generation and container scanning baked into every build
- [ ] Caller workflows stay clean and focused on orchestration

**Follow-Up Questions:**
1. How do you version reusable workflows and handle breaking changes?
2. How do you pass the image digest from the build job to the deploy job across workflows?
3. What permissions model should you use for reusable workflows in a banking organization?

**Source:** `banking-genai-engineering-academy/cicd-devops/github-workflows.md`, `banking-genai-engineering-academy/cicd-devops/ci-cd-design.md`

---

### Q3: 🔵 What is Tekton and how does its architecture (Tasks, Pipelines, PipelineRuns) differ from traditional Jenkins pipelines?

**Strong Answer:**

Tekton is a cloud-native CI/CD framework that runs pipelines as Kubernetes Custom Resource Definitions (CRDs). Unlike Jenkins, which uses a master-agent architecture with Groovy-based pipeline DSL, Tekton is entirely Kubernetes-native — every Task, Pipeline, and PipelineRun is a declarative YAML resource applied to the cluster. This means Tekton pipelines benefit from Kubernetes RBAC, resource quotas, and the full Kubernetes ecosystem.

The core abstractions are: **Tasks** are the smallest unit — a sequence of steps that run in containers (e.g., build an image, run tests). **Pipelines** compose multiple Tasks with explicit ordering via `runAfter` and share data through **Workspaces** (persistent volumes mounted into Tasks). **PipelineRuns** are executions of a Pipeline — they parameterize the Pipeline and create actual pods. **Triggers** respond to external events (git push, webhook) and create PipelineRuns automatically.

In a banking environment like Citi's, Tekton is particularly valuable because: (1) it runs on OpenShift, which Citi uses extensively; (2) Workspaces provide isolated, ephemeral storage per run — no shared state contamination between builds; (3) every PipelineRun is a Kubernetes resource with full audit logging; and (4) Tasks are composable and reusable across teams.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: genai-ci-pipeline
spec:
  params:
    - name: repo-url
    - name: revision
      default: main
    - name: image-name
  workspaces:
    - name: shared-workspace
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace

    - name: test
      taskRef:
        name: run-tests
      runAfter: [clone]
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: build
      taskRef:
        name: build-and-push
      runAfter: [clone]
      params:
        - name: image
          value: $(params.image-name)
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: deploy-staging
      taskRef:
        name: deploy-to-openshift
      runAfter: [test, build]
      workspaces:
        - name: source
          workspace: shared-workspace
```

The key advantage over Jenkins: no Groovy DSL to maintain, no master node bottleneck, and pipelines scale elastically with Kubernetes. The trade-off is steeper learning curve and more YAML to manage.

**Key Points to Hit:**
- [ ] Tekton uses Kubernetes CRDs — everything is declarative YAML
- [ ] Tasks (steps in containers), Pipelines (compose Tasks), PipelineRuns (executions)
- [ ] Workspaces replace Jenkins workspace — persistent volumes shared between Tasks
- [ ] Triggers respond to events and create PipelineRuns automatically
- [ ] Advantages over Jenkins: no Groovy, no master bottleneck, Kubernetes-native, elastic scaling

**Follow-Up Questions:**
1. How do you pass results between Tekton Tasks?
2. How do you handle secrets in Tekton Pipelines?
3. How would you migrate a Jenkins pipeline to Tekton?

**Source:** `banking-genai-engineering-academy/cicd-devops/tekton.md`, `banking-genai-engineering-academy/cicd-devops/ci-cd-design.md`

---

### Q4: 🔵 Compare trunk-based development with GitFlow. Which do you recommend for a banking GenAI platform and why?

**Strong Answer:**

Trunk-based development uses a single `main` branch that is always production-ready. Developers create short-lived feature branches (max 1-2 days) and merge via Pull Request. Incomplete features are hidden behind feature flags. GitFlow uses a `develop` branch for integration, `main` for releases only, plus feature, release, and hotfix branches.

For a banking GenAI platform, I recommend **trunk-based development** for several reasons. First, GenAI features evolve rapidly — model versions, prompt templates, and RAG pipelines change frequently. Long-lived feature branches in GitFlow create massive merge conflicts and slow feedback loops. Second, trunk-based development enables true continuous delivery — every merge to main is deployable, which is essential when you need to respond to model degradation or security vulnerabilities quickly. Third, feature flags allow compliance teams to review changes before they are visible to users, satisfying banking change management requirements without branching complexity.

However, banking adds constraints: branch protection rules must be strict (2+ approvals, CI must pass, CODEOWNERS review), and every merge must link to a change management ticket. The "always deployable main" principle means the CI pipeline must have comprehensive gates — you cannot afford a broken main in a regulated environment.

```
Trunk-Based (Recommended for Banking GenAI):
main ──┬── A ── B ── C ── D ── E (always production-ready)
       │
       ├── short-lived feature branches (max 1-2 days)
       └── feature flags for incomplete features

Branch Protection:
- 2+ approvals required
- CI must pass (unit, security, GenAI eval)
- CODEOWNERS review required
- No force push, no direct pushes
- Linear history (squash or rebase merge)
```

GitFlow's release stabilization branches make sense for traditional on-premise banking software with quarterly releases, but GenAI services deployed to Kubernetes can and should deploy daily. The overhead of managing develop, release, and hotfix branches slows delivery without adding meaningful risk reduction when you have strong CI gates.

**Key Points to Hit:**
- [ ] Trunk-based: single main always deployable, short-lived branches, feature flags
- [ ] GitFlow: develop + main + release + hotfix branches, more overhead
- [ ] Trunk-based recommended for GenAI: rapid iteration, continuous delivery
- [ ] Feature flags enable compliance review without branching complexity
- [ ] Strict branch protection required (2+ approvals, CI gates, CODEOWNERS, change tickets)

**Follow-Up Questions:**
1. How do you handle long-running feature development in trunk-based development?
2. What happens when a hotfix is needed in production under trunk-based development?
3. How do feature flags interact with banking change management processes?

**Source:** `banking-genai-engineering-academy/cicd-devops/branching-strategies.md`, `banking-genai-engineering-academy/cicd-devops/README.md`

---

### Q5: 🔵 What stages should a CI/CD pipeline have for a GenAI application in a banking environment?

**Strong Answer:**

A GenAI CI/CD pipeline in banking needs eight stages, organized to fail fast and provide comprehensive quality gates. The stages run in this order:

**Stage 1 — Fast Checks (2-5 min):** Lint, type check, license check, commit message validation. These run first because they are fast and catch obvious issues before wasting compute on builds.

**Stage 2 — Build & Test (10-20 min, parallel):** Build the container image, run unit tests (with coverage >= 80%), integration tests, API contract tests, and critically, **GenAI evaluation tests** — hallucination rate, response relevance, toxicity, PII leakage, and banking knowledge accuracy. These run in parallel after the build artifact is available.

**Stage 3 — Security (5-10 min, parallel):** SAST (CodeQL), SCA (Trivy filesystem scan), container image scanning, secret detection (gitleaks), and IaC security scanning (Checkov). All must pass — zero CRITICAL vulnerabilities allowed.

**Stage 4 — Deploy to Staging (5-10 min):** Deploy the container to a staging environment that mirrors production. Run smoke tests and end-to-end tests.

**Stage 5 — Verify Staging (5-10 min):** Performance tests (P95 latency < 2s), load tests, and GenAI quality validation on real data. This catches issues that only appear in a deployed environment.

**Stage 6 — Approval Gate:** Low-risk changes auto-approve. Medium-risk require team lead + SRE review. High-risk go to the Change Advisory Board (CAB).

**Stage 7 — Deploy to Production (10-30 min):** Canary deployment starting at 10% traffic, progressive rollout to 100% with continuous verification. Automated rollback on SLO breach.

**Stage 8 — Post-Deployment (5 min):** Monitor SLOs, check for new alerts, notify stakeholders via Slack/email.

```yaml
quality_gates:
  unit_tests: "All pass, coverage >= 80%"
  security: "Zero CRITICAL, zero HIGH in prod code"
  performance: "P95 latency < 2s, error rate < 0.1%"
  genai_quality:
    hallucination_rate: "< 2%"
    response_relevance: "> 85%"
    toxicity_score: "< 0.01%"
    pii_leakage: "0 detections"
    response_time_p95: "< 5 seconds"
```

The GenAI-specific gates are what differentiate this pipeline from a traditional microservice pipeline. Hallucination rate and PII leakage tests are mandatory in banking — a GenAI model that leaks customer data or provides hallucinated financial advice is a regulatory violation.

**Key Points to Hit:**
- [ ] Eight stages: fast checks, build & test, security, staging deploy, staging verify, approval, prod deploy, post-deploy
- [ ] GenAI evaluation tests are mandatory: hallucination, relevance, toxicity, PII, response time
- [ ] Quality gates with specific thresholds (hallucination < 2%, P95 < 2s, coverage >= 80%)
- [ ] Security scanning in parallel: SAST, SCA, container, secrets, IaC
- [ ] Risk-based approval: automated (low), team lead (medium), CAB (high)

**Follow-Up Questions:**
1. How do you optimize pipeline execution time when running all these stages?
2. What metrics determine whether to promote a canary to full production?
3. How do you handle flaky GenAI evaluation tests that produce inconsistent results?

**Source:** `banking-genai-engineering-academy/cicd-devops/ci-cd-design.md`, `banking-genai-engineering-academy/cicd-devops/testing-in-pipelines.md`, `banking-genai-engineering-academy/cicd-devops/README.md`

---

### Q6: 🟡 How do you configure Harness for a canary deployment with automated verification and rollback for a GenAI API?

**Strong Answer:**

Harness excels at canary deployments with built-in verification and automated rollback. For a GenAI API in banking, you want a pipeline that deploys a canary instance, verifies it against real production metrics (error rate, latency, and critically, GenAI-specific metrics like hallucination rate), and automatically rolls back if any metric exceeds thresholds.

The Harness pipeline has four key stages in the deployment phase: (1) CanaryDeploy — deploys 1 instance alongside production, (2) Verify — checks metrics for 10 minutes with retry, (3) RollingDeploy — full rollout if verification passes, (4) RollbackSteps — automatically invoked if verification fails.

What makes Harness particularly powerful for GenAI is its ability to query custom Prometheus metrics. You can verify hallucination rate, response relevance, and toxicity scores during the canary phase — not just traditional SRE metrics.

```yaml
pipeline:
  stages:
    - stage:
        name: Deploy
        identifier: deploy
        type: Deployment
        spec:
          deploymentType: Kubernetes
          execution:
            steps:
              - step:
                  type: CanaryDeploy
                  identifier: canary_deploy
                  spec:
                    instanceUnitType: Count
                    instances: 1

              - step:
                  type: Verify
                  identifier: canary_verify
                  spec:
                    timeout: 10m
                    retry:
                      count: 3
                      atRuntimeMultiplier: 2
                    criteria:
                      metrics:
                        - metric: error_rate
                          condition: LESS_THAN
                          value: 1.0
                        - metric: p99_latency
                          condition: LESS_THAN
                          value: 2000
                        - metric: hallucination_rate
                          condition: LESS_THAN
                          value: 5.0

            rollbackSteps:
              - step:
                  type: RollbackRollingDeployment
                  identifier: rollback
```

The verification criteria connect to Prometheus, which exposes both SRE metrics and GenAI-specific metrics. If the canary's error rate exceeds 1%, P99 latency exceeds 2000ms, or hallucination rate exceeds 5%, Harness automatically triggers the rollback step — no human intervention needed. In a banking context, this automated rollback is critical: a degraded GenAI model that starts hallucinating financial advice must be rolled back within seconds, not waiting for a human to notice.

The trade-off: you need robust monitoring infrastructure (Prometheus + custom exporters for GenAI metrics) before you can use Harness verification effectively. Without good metrics, verification is blind.

**Key Points to Hit:**
- [ ] Harness stages: CanaryDeploy, Verify, RollingDeploy, RollbackSteps
- [ ] Verification against Prometheus metrics: error rate, latency, hallucination rate
- [ ] Automated rollback on metric threshold breach — no human intervention
- [ ] Retry logic with atRuntimeMultiplier for transient issues
- [ ] Requires robust monitoring (Prometheus + custom GenAI metric exporters)

**Follow-Up Questions:**
1. How do you define and export custom GenAI metrics (hallucination rate) to Prometheus?
2. What happens if Harness verification is inconclusive (metrics are borderline)?
3. How do you configure Harness approval gates for CAB-required changes?

**Source:** `banking-genai-engineering-academy/cicd-devops/harness.md`, `banking-genai-engineering-academy/cicd-devops/progressive-delivery.md`

---

### Q7: 🟡 How do you design branch protection rules and CODEOWNERS for a banking GenAI repository?

**Strong Answer:**

Branch protection rules are your first line of defense against unreviewed code reaching production. In banking, they are non-negotiable — regulators expect evidence that every production change underwent review. The rules must be comprehensive enough to catch issues but not so restrictive that they bottleneck delivery.

For a GenAI repository at Citi, the `main` branch should have these protections: (1) Pull requests required — no direct pushes; (2) Minimum 2 approving reviews — one from the team, one from CODEOWNERS; (3) Dismiss stale reviews — if new commits are pushed after approval, the approval is invalidated; (4) Require status checks to pass — CI build, unit tests, security scan, integration tests, and GenAI evaluation; (5) Require branches to be up-to-date before merging — prevents stale merges; (6) Enforce on administrators — no exceptions, even for VPs; (7) Require linear history — no merge commits, use squash or rebase; (8) Disable force pushes and branch deletion.

The CODEOWNERS file defines who must review changes to specific paths. In a GenAI platform, different parts of the codebase require different expert reviews:

```yaml
# GitHub branch protection for main
branch_protection:
  main:
    require_pull_request: true
    required_approving_review_count: 2
    require_code_owner_review: true
    dismiss_stale_reviews: true
    require_status_checks:
      strict: true
      checks:
        - ci-build
        - unit-tests
        - security-scan
        - integration-tests
        - genai-eval
    enforce_admins: true
    require_linear_history: true
    allow_force_pushes: false
    allow_deletions: false
```

```
# CODEOWNERS file
# Security-sensitive code requires security team review
/src/auth/        @banking/security-team
/src/secrets/     @banking/security-team

# Database changes require DBA review
/db/migrations/   @banking/dbas

# Infrastructure changes require SRE review
/terraform/       @banking/sre-team
/k8s/             @banking/sre-team

# GenAI prompt changes require compliance review
/prompts/         @banking/ai-compliance
/prompts/         @banking/legal-review

# CI/CD pipeline changes require platform team review
/.github/         @banking/platform-team
/tekton/          @banking/platform-team
```

The banking context means CODEOWNERS should include compliance and legal reviewers for prompt changes — GenAI prompts that give financial advice have regulatory implications. Similarly, infrastructure changes require SRE review because they affect production stability.

**Key Points to Hit:**
- [ ] 8 branch protection rules: PR required, 2+ approvals, CODEOWNERS, dismiss stale, status checks, up-to-date, enforce admins, linear history
- [ ] CODEOWNERS routes different paths to different expert teams (security, DBA, SRE, compliance, legal)
- [ ] GenAI prompt changes require compliance and legal review — regulatory implications
- [ ] Enforce on administrators: no exceptions for senior staff
- [ ] Linear history (squash/rebase) for clean audit trail

**Follow-Up Questions:**
1. How do you handle situations where a CODEOWNER is unavailable and blocks a critical PR?
2. What is the difference between squash, rebase, and merge strategies and which should banking use?
3. How do you automate linking PRs to change management tickets?

**Source:** `banking-genai-engineering-academy/cicd-devops/branching-strategies.md`, `banking-genai-engineering-academy/cicd-devops/pull-request-process.md`

---

### Q8: 🟡 How do you structure testing in CI/CD pipelines — what runs when and what blocks what?

**Strong Answer:**

Testing in CI/CD follows a three-stage gating model, organized by speed and impact. The key principle: fast tests run first and block merges, moderate tests run after merge and block staging deployments, and comprehensive tests run before production deployment.

**Stage 1 — Pre-Merge (blocks PR merge):** Unit tests (all, run in parallel via matrix sharding), lint/format, type checking. These must complete in under 5 minutes. If they fail, the PR cannot be merged. This is the fastest feedback loop.

**Stage 2 — Post-Merge (blocks staging deploy):** Full integration tests against staging services, API contract tests (ensuring API consumers are not broken), database migration tests, and GenAI evaluation tests (hallucination rate, relevance, toxicity, PII leakage). These run after the merge to main and must pass before the artifact is deployed to staging.

**Stage 3 — Pre-Production (blocks production deploy):** End-to-end tests, performance/load tests, security penetration tests (DAST), and user acceptance tests. These are the slowest but most comprehensive. They run against the staging environment and must pass before the approval gate is opened.

```yaml
# GitHub Actions: parallel test execution with matrix sharding
jobs:
  test-unit:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: pytest --shard=${{ matrix.shard }} --total-shards=4

  test-integration:
    needs: build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
    steps:
      - run: pytest tests/integration/

  genai-eval:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: python scripts/evaluate_genai.py

  security-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: CRITICAL,HIGH
          exit-code: '1'
```

Flaky test management is critical. Tests with < 95% pass rate over the last 100 runs are auto-quarantined — they do not block merges but are tracked in a dashboard. Fix priority is P2 (must fix within sprint). No new flaky tests are accepted. This prevents the "the build is always red so nobody cares" anti-pattern.

For GenAI evaluation tests specifically, you need deterministic test data. Use a fixed set of questions with known correct answers, and measure hallucination rate by asking the model questions where the answer is verifiable from a knowledge base.

**Key Points to Hit:**
- [ ] Three-stage gating: pre-merge (blocks merge), post-merge (blocks staging), pre-production (blocks prod)
- [ ] Parallel test execution with matrix sharding for speed
- [ ] Flaky test policy: auto-quarantine at < 95% pass rate, P2 fix priority
- [ ] GenAI evaluation tests: hallucination, relevance, toxicity, PII, banking knowledge
- [ ] Deterministic test data for GenAI — fixed questions with known answers

**Follow-Up Questions:**
1. How do you handle test data management for integration tests in a banking environment?
2. What is your strategy for optimizing test execution time when the pipeline takes 30+ minutes?
3. How do you test GenAI response quality deterministically when LLMs are non-deterministic by nature?

**Source:** `banking-genai-engineering-academy/cicd-devops/testing-in-pipelines.md`, `banking-genai-engineering-academy/cicd-devops/ci-cd-design.md`

---

### Q9: 🟡 Explain progressive delivery patterns — when would you use canary vs blue-green vs feature flags in a banking context?

**Strong Answer:**

Progressive delivery deploys changes incrementally, reducing blast radius and enabling fast rollback. The three primary patterns serve different use cases:

**Canary deployments** route a small percentage of traffic (10% → 25% → 50% → 100%) to the new version while monitoring metrics. Use canary for: routine service updates, model version changes, configuration changes. Canary is the default choice for GenAI API deployments at Citi because it provides continuous verification — if the new model's hallucination rate spikes at 25% traffic, you only affect 25% of users and auto-rollback is instant.

**Blue-green deployments** maintain two identical production environments. You deploy to the idle environment (green), verify it, then switch 100% of traffic via load balancer. Use blue-green for: major version releases, database schema migrations, infrastructure changes. Blue-green is preferred when you need instant rollback (a single load balancer switch) and when the change is too risky for gradual rollout — for example, switching to a new embedding model across the entire RAG pipeline.

**Feature flags** deploy code to production but keep features hidden behind runtime toggles. Use feature flags for: incomplete features, A/B testing, gradual user rollout, kill switches. Feature flags are essential in banking because they decouple deployment from release — you can deploy on Tuesday but enable the feature for users on Thursday after CAB approval and stakeholder communication.

```python
# Feature flag with percentage rollout for GenAI model
class FeatureFlagManager:
    def is_enabled(self, flag_name: str, user_id: str) -> bool:
        flag = self.flags.get(flag_name, {})
        if not flag.get('enabled', False):
            return False
        if 'percentage' in flag and user_id:
            hash_value = int(hashlib.md5(
                f"{flag_name}:{user_id}".encode()
            ).hexdigest(), 16) % 100
            return hash_value < flag['percentage']
        return False

flags = FeatureFlagManager({
    'new_genai_model': {'enabled': True, 'percentage': 25},
    'new_ui': {'enabled': True, 'percentage': 100},
})
```

The banking context adds important constraints: feature flags must have an expiration policy (no orphaned flags), flag state must be auditable (who enabled what when), and critical kill switches must be testable in production drills. Feature flags for GenAI models are particularly important because model behavior can differ across user segments — you can roll out a new model to internal users first, then to retail customers, then to institutional clients.

**Key Points to Hit:**
- [ ] Canary: gradual traffic shift (10→25→50→100%), default for routine updates, auto-rollback
- [ ] Blue-green: two identical environments, instant switch, for major releases and DB migrations
- [ ] Feature flags: code deployed but hidden, decouples deployment from release, for A/B testing and gradual rollout
- [ ] Canary = default for GenAI API; Blue-green = major changes; Feature flags = incomplete features
- [ ] Banking constraints: flag expiration policy, auditable flag state, testable kill switches

**Follow-Up Questions:**
1. How do you measure success during a canary deployment — what metrics trigger promotion vs rollback?
2. What are the risks of feature flags and how do you prevent flag sprawl?
3. How do you handle database schema changes in a blue-green deployment?

**Source:** `banking-genai-engineering-academy/cicd-devops/progressive-delivery.md`, `banking-genai-engineering-academy/cicd-devops/harness.md`

---

### Q10: 🟡 What production approval gates are needed for a banking GenAI deployment and how do you automate them?

**Strong Answer:**

Production approval gates in banking follow a risk-based classification model. Not every change needs CAB review — that would bottleneck delivery. Instead, changes are classified as low, medium, or high risk, with corresponding approval requirements.

**Low-risk changes** (bug fixes, no DB or infra changes, security scan clean, all tests passing, rollback tested) are auto-approved by the CI/CD pipeline. The pipeline itself is the gate — if all quality checks pass, the deployment proceeds automatically.

**Medium-risk changes** (new features behind feature flags, non-breaking DB migrations, configuration changes) require team lead and SRE review. In GitHub Actions, this is implemented via Environment Protection Rules — the deployment job waits for the specified reviewers to approve before proceeding.

**High-risk changes** (breaking API changes, DB schema changes, infrastructure changes, GenAI model changes) require CAB review. The Change Advisory Board meets weekly to review change requests, assess impact, verify rollback procedures, and approve or reject.

```yaml
risk_levels:
  low:
    criteria:
      - "Bug fix only, no new features"
      - "No database or infrastructure changes"
      - "Security scan clean"
      - "All tests passing"
      - "Rollback procedure tested"
    approval: "Automated (CI gates only)"

  medium:
    criteria:
      - "New features with feature flags"
      - "Non-breaking database migrations"
    approval: "Team lead + SRE review"

  high:
    criteria:
      - "Breaking API changes"
      - "Database schema changes"
      - "GenAI model changes"
    approval: "CAB review required"
```

In GitHub Actions, environment protection rules enforce the approval requirements:

```yaml
# GitHub environment protection rules
environments:
  production:
    protection_rules:
      required_reviewers: 2
      reviewers:
        - type: Team
          name: sre-team
        - type: Team
          name: genai-leads
      wait_timer: 5  # Minutes delay before deployment starts
      restrictions:
        teams:
          - release-managers
```

The banking context adds SOX compliance requirements: segregation of duties (the developer who wrote the code cannot approve the deployment), audit trail of all approvals, and quarterly access reviews. Every approval decision, reviewer identity, and timestamp must be logged and retained for the audit period.

For emergency changes that bypass CAB, there must be a documented emergency change process: the deployment can proceed, but a retroactive change request must be filed within 24 hours with full justification and post-implementation review.

**Key Points to Hit:**
- [ ] Risk-based classification: low (auto-approve), medium (team lead + SRE), high (CAB)
- [ ] GitHub Environment Protection Rules enforce reviewer requirements
- [ ] SOX compliance: segregation of duties, audit trail, quarterly access review
- [ ] Emergency change process: deploy first, retroactive change request within 24 hours
- [ ] Change request template: ID, title, description, risk, rollback, deployment window, stakeholders

**Follow-Up Questions:**
1. How do you automatically classify change risk based on the PR content?
2. What happens if a CAB meeting is delayed but the deployment is urgent?
3. How do you implement a wait timer in GitHub Actions and why is it useful?

**Source:** `banking-genai-engineering-academy/cicd-devops/production-approvals.md`, `banking-genai-engineering-academy/cicd-devops/change-management.md`

---

### Q11: 🟡 What are the rollback strategies for Kubernetes deployments and when should rollback be automated vs manual?

**Strong Answer:**

Rollback strategies exist on a spectrum from fully automated to fully manual. The choice depends on the deployment pattern, failure type, and complexity of the change.

**Automated rollback** (seconds, fully automatic) is triggered by health metric degradation during a canary or progressive deployment. When error rate exceeds threshold, latency spikes, or GenAI hallucination rate jumps, the system automatically reverts to the previous version. This is implemented via Argo Rollouts or Harness with predefined rollback analysis templates:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: genai-api
spec:
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 5m}
        - analysis:
            templates:
              - templateName: canary-check
            failingFast: 2  # Auto-rollback after 2 consecutive failures
      abortScaleDownDelaySeconds: 30

apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-check
spec:
  metrics:
    - name: error-rate
      interval: 1m
      failureCondition: result[0] > 0.05
      failingLimit: 2
      provider:
        prometheus:
          query: |
            sum(rate(http_requests_total{status=~"5.."}[1m]))
            / sum(rate(http_requests_total[1m]))
```

**Manual rollback** (minutes, human-driven) is used when the failure is complex — for example, a database migration that partially completed, or a configuration change that caused cascading failures across multiple services. The manual procedure is: (1) identify the previous good revision via `kubectl rollout history`, (2) execute `kubectl rollout undo deployment/genai-api`, (3) verify health endpoint, (4) monitor for 15 minutes, (5) document the incident.

**Database migration rollback** is the trickiest case. Forward-only migrations (adding columns, creating indexes) are safe. Destructive migrations (dropping columns, changing types) require a rollback strategy: either keep the old column alongside the new one (expand-and-contract pattern), or have a reverse migration script ready. In banking, database migration rollback scripts must be tested quarterly as part of rollback drills.

**Blue-green switchback** is the fastest rollback for major releases — simply switch the load balancer back to the blue environment. This takes seconds and is the preferred rollback method for high-risk deployments.

The rule of thumb: automate rollback when the failure is detectable by metrics and the previous version is known to be healthy. Use manual rollback when the failure requires human diagnosis or when multiple services are affected.

**Key Points to Hit:**
- [ ] Automated rollback: triggered by metrics, implemented via Argo Rollouts/Harness, seconds
- [ ] Manual rollback: kubectl rollout undo, for complex failures requiring human diagnosis
- [ ] Database migration rollback: expand-and-contract pattern, reverse scripts tested quarterly
- [ ] Blue-green switchback: instant load balancer switch, preferred for high-risk deployments
- [ ] Rule: automate when failure is metric-detectable and previous version is healthy

**Follow-Up Questions:**
1. How do you rollback a GenAI model deployment that causes subtle quality degradation (not obvious errors)?
2. What is the expand-and-contract pattern for database migrations?
3. How do you test your rollback procedures to ensure they work when needed?

**Source:** `banking-genai-engineering-academy/cicd-devops/rollbacks.md`, `banking-genai-engineering-academy/cicd-devops/progressive-delivery.md`

---

### Q12: 🟡 How do you handle container scanning in CI/CD — what tools, severity thresholds, and policy enforcement?

**Strong Answer:**

Container scanning is mandatory in banking to prevent deploying images with known vulnerabilities. The primary tool is **Trivy** (by Aqua Security), which scans for OS package vulnerabilities, language dependency vulnerabilities, misconfigurations, and secrets.

The scanning workflow runs on every PR and every build. It has two phases: (1) a blocking scan that fails the CI on CRITICAL and HIGH severity vulnerabilities, and (2) a reporting scan that generates SARIF output for the GitHub Security dashboard.

```yaml
# Trivy container scan in GitHub Actions
- name: Run Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'genai-api:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
    ignore-unfixed: true

- name: Generate SARIF report
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'genai-api:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

For policy enforcement, use OPA (Open Policy Agent) Rego policies:

```rego
# policy.rego — block images with CRITICAL vulnerabilities
package container_security

deny[msg] {
  input.vulnerabilities[_].severity == "CRITICAL"
  msg := sprintf("CRITICAL vulnerability found: %s in %s",
    [input.vulnerabilities[_].vulnerabilityID, input.vulnerabilities[_].pkgName])
}

deny[msg] {
  count([v | v = input.vulnerabilities[_]; v.severity == "HIGH"]) > 5
  msg := "More than 5 HIGH vulnerabilities found"
}

deny[msg] {
  input.image.user == "root"
  msg := "Container runs as root user"
}
```

Severity thresholds in banking: **CRITICAL** vulnerabilities always block merges — zero tolerance. **HIGH** vulnerabilities are tracked with a defined SLA (e.g., fix within 7 days for production code, 30 days for non-production). **MEDIUM** and **LOW** are tracked but do not block deployment.

Base image best practices reduce vulnerabilities: use minimal base images (distroless, Alpine), run as non-root user, and regularly update base images. In banking, you should also maintain an internal approved base image list — developers cannot use arbitrary base images from Docker Hub.

For false positives, there should be a documented exception process: the vulnerability is reviewed by the security team, documented in the change request, and an exception is granted with an expiration date. This prevents scanner noise from blocking legitimate deployments.

**Key Points to Hit:**
- [ ] Trivy scans every PR and build — CRITICAL/HIGH block with exit-code 1
- [ ] SARIF output uploaded to GitHub Security dashboard for tracking
- [ ] OPA Rego policies enforce: no CRITICAL, no > 5 HIGH, no root user
- [ ] Severity thresholds: CRITICAL = always block, HIGH = SLA, MEDIUM/LOW = track
- [ ] Base image best practices: minimal (distroless/Alpine), non-root, approved list, regular updates

**Follow-Up Questions:**
1. How do you handle a CRITICAL vulnerability in a dependency that has no fix available?
2. What is the difference between build-time scanning and runtime scanning?
3. How do you integrate SBOM data with container scanning results for supply chain security?

**Source:** `banking-genai-engineering-academy/cicd-devops/container-scanning.md`, `banking-genai-engineering-academy/cicd-devops/security-scanning.md`

---

### Q13: 🟡 How do you manage secrets in CI/CD pipelines — what are the options and what is the enterprise best practice?

**Strong Answer:**

Secrets management in CI/CD is about ensuring credentials, API keys, and tokens are never stored in plaintext in source code, CI configuration, or logs. In banking, a leaked secret is a P1 security incident.

The options range from basic to enterprise-grade:

**GitHub Secrets** — the simplest option. Secrets are stored in the repository or organization settings and accessed via `${{ secrets.SECRET_NAME }}` in workflows. They are encrypted at rest and masked in logs. Suitable for CI/CD variables like registry credentials and deployment tokens. However, they lack fine-grained access control and audit logging, which regulators expect.

**HashiCorp Vault** — the enterprise gold standard. Vault provides dynamic secrets (short-lived credentials generated on demand), fine-grained ACL policies, audit logging of every secret access, and secret rotation. In CI/CD, the pipeline authenticates to Vault via Kubernetes auth method or OIDC, then retrieves the secrets it needs for that specific run. Secrets are never stored in GitHub — they exist only in memory during pipeline execution.

```yaml
# Vault integration in GitHub Actions
- name: Authenticate with Vault
  uses: hashicorp/vault-action@v2
  with:
    url: https://vault.bank.com:8200
    method: kubernetes
    role: genai-ci-role
    secrets: |
      secret/data/banking/genai/database url | DATABASE_URL ;
      secret/data/banking/genai/openai api_key | OPENAI_API_KEY ;
```

**Sealed Secrets** (for Kubernetes) — encrypts Kubernetes Secrets using a public key so they can be safely stored in Git. The controller running in the cluster decrypts them. This enables GitOps workflows where secrets are version-controlled without being readable.

**External Secrets Operator** — syncs secrets from external secret managers (AWS Secrets Manager, Vault, Azure Key Vault) into Kubernetes Secrets automatically. This is the cleanest approach for Kubernetes workloads — the CI/CD pipeline writes secrets to AWS Secrets Manager, and ESO syncs them to the cluster.

The enterprise best practice for a banking GenAI platform: **Vault for CI/CD pipeline secrets** (API keys, database credentials), **Sealed Secrets or ESO for Kubernetes runtime secrets** (service-to-service credentials), and **GitHub Secrets only for bootstrap credentials** (Vault authentication tokens, registry credentials). Never store OpenAI API keys, database passwords, or any production credential in GitHub Secrets alone.

Secret rotation should be automated — Vault can generate short-lived database credentials that expire after 1 hour. If a secret is suspected to be leaked, it should be immediately revoked and rotated without service downtime.

**Key Points to Hit:**
- [ ] Never store secrets in plaintext in code, CI config, or logs
- [ ] Vault: dynamic secrets, fine-grained ACL, audit logging, rotation — enterprise gold standard
- [ ] GitHub Secrets: simple, encrypted at rest, masked in logs — only for bootstrap credentials
- [ ] Sealed Secrets: encrypt K8s secrets for Git storage, decrypted by cluster controller
- [ ] Best practice: Vault for CI/CD, Sealed Secrets/ESO for K8s, GitHub Secrets for bootstrap only

**Follow-Up Questions:**
1. How do you rotate secrets without causing deployment downtime?
2. How do you audit who accessed which secret and when?
3. What is SOPS and how does it work with GitOps workflows?

**Source:** `banking-genai-engineering-academy/cicd-devops/secrets-handling.md`, `banking-genai-engineering-academy/cicd-devops/security-scanning.md`

---

### Q14: 🟡 How do you manage Infrastructure as Code with Terraform — module design, state management, and GitOps?

**Strong Answer:**

Infrastructure as Code (IaC) means managing all infrastructure through version-controlled, declarative configuration files. Terraform is the dominant tool, and in banking, IaC is mandatory for audit compliance — every infrastructure change must be reviewable in git.

**Module design** follows a layered approach. Reusable modules define resource patterns (e.g., a GenAI platform module that provisions OpenShift cluster, PostgreSQL, Redis, Kafka). Environment-specific configurations (dev, staging, production) consume these modules with different parameters:

```hcl
# modules/genai_platform/main.tf
module "genai_platform" {
  source = "./modules/genai-platform"

  environment = var.environment
  region      = var.region

  cluster_name     = "banking-genai-${var.environment}"
  node_count       = var.environment == "production" ? 6 : 3
  db_multi_az      = var.environment == "production"
  db_backup_retention = 30

  tags = {
    Project     = "banking-genai"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

**State management** is critical. Terraform state contains the mapping between configuration and real infrastructure, including sensitive data. It must be stored remotely (S3) with encryption and locking (DynamoDB):

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "banking-terraform-state"
    key            = "genai-platform/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "banking-terraform-locks"
  }
}
```

**GitOps for infrastructure** means: all Terraform code is in git, `terraform plan` runs on PR (showing what will change), the plan is reviewed by the team, merge to main triggers `terraform apply` via CI/CD. This provides full audit trail — every infrastructure change is a commit with reviewers.

The banking context adds: state files must be encrypted at rest, access to state must be logged, secrets are never in Terraform code (use Vault provider or data sources), and destroy plans must be tested for cleanup procedures. Resource tagging is mandatory for cost allocation and compliance reporting.

**Key Points to Hit:**
- [ ] Module design: reusable modules + environment-specific configs (dev/staging/prod)
- [ ] State management: remote S3 backend, encryption, DynamoDB locking
- [ ] GitOps: terraform plan on PR, review, merge triggers apply — full audit trail
- [ ] Banking: encrypted state, logged access, secrets via Vault (not in code), tested destroy plans
- [ ] Resource tagging mandatory for cost allocation and compliance

**Follow-Up Questions:**
1. What happens when Terraform state drifts from actual infrastructure?
2. How do you handle Terraform state that contains sensitive data (database passwords)?
3. How do you coordinate Terraform changes across multiple teams working on the same infrastructure?

**Source:** `banking-genai-engineering-academy/cicd-devops/infrastructure-as-code.md`, `banking-genai-engineering-academy/cicd-devops/terraform.md`

---

### Q15: 🟡 How do you handle release processes — semantic versioning, changelog generation, and environment promotion?

**Strong Answer:**

Release processes ensure every deployment is versioned, documented, and traceable. In banking, releases must link to change requests and include rollback procedures.

**Semantic versioning** (MAJOR.MINOR.PATCH) provides a standard way to communicate change impact. MAJOR for breaking API changes, MINOR for new backward-compatible features, PATCH for bug fixes. For GenAI APIs, model version changes that affect response format are MAJOR, new capabilities are MINOR, and prompt tuning improvements are PATCH.

Automated versioning uses **conventional commits** + **semantic-release**: commit messages like `feat: add new embedding model` trigger a MINOR bump, `fix: resolve tokenization bug` triggers PATCH, and `feat!: change API contract` triggers MAJOR. This eliminates human error in version numbering.

```yaml
# Automated release with semantic-release
jobs:
  release:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.release.outputs.version }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Semantic Release
        id: release
        uses: cycjimmy/semantic-release-action@v3
        with:
          extra_plugins: |
            @semantic-release/changelog
            @semantic-release/git
```

**Changelog generation** is automated from commit messages, categorized into Features, Bug Fixes, Security, and Breaking Changes. Every release also needs a manually reviewed release notes document that includes: summary, what's new, bug fixes, breaking changes, impact assessment, rollback plan, testing evidence, and contacts.

**Environment promotion** follows the pipeline: staging verification must pass before production deployment. In banking, the promotion is gated by the approval process (auto for low-risk, team lead for medium, CAB for high). Each environment promotion creates an audit record: what version, who approved, when deployed, test results.

For GenAI releases, the release notes must also include model performance metrics: hallucination rate comparison, response relevance scores, and any changes to the knowledge base or prompt templates.

**Key Points to Hit:**
- [ ] Semantic versioning: MAJOR (breaking), MINOR (new features), PATCH (bug fixes)
- [ ] Automated versioning via conventional commits + semantic-release
- [ ] Changelog auto-generated from commits + manually reviewed release notes
- [ ] Environment promotion gated by approval process with full audit records
- [ ] GenAI release notes include model metrics: hallucination rate, relevance scores

**Follow-Up Questions:**
1. How do you handle breaking changes in a banking API that has multiple consumer teams?
2. How do you coordinate releases across multiple microservices in the GenAI platform?
3. What is your release frequency for a production banking GenAI platform?

**Source:** `banking-genai-engineering-academy/cicd-devops/release-processes.md`, `banking-genai-engineering-academy/cicd-devops/release-communication.md`

---

### Q16: 🔴 How do you implement drift detection for Terraform-managed infrastructure and what is your remediation strategy?

**Strong Answer:**

Infrastructure drift occurs when actual infrastructure differs from the desired state defined in Terraform code. It happens through: manual console changes (emergency hotfix), another team modifying resources, Terraform bugs, or cloud provider changes. In banking, untracked drift is a compliance violation because the code no longer reflects reality.

**Detection** is implemented as a scheduled GitHub Actions workflow that runs `terraform plan -detailed-exitcode` daily. Exit code 0 means no drift, exit code 2 means drift detected. The workflow generates a report and alerts via Slack:

```yaml
name: Terraform Drift Detection
on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -detailed-exitcode
        id: plan
        continue-on-error: true

      - name: Alert on drift
        if: steps.plan.outcome == 'failure'
        run: |
          terraform plan -no-color > drift-report.txt
          curl -X POST $SLACK_WEBHOOK \
            -d "{\"text\": \"Terraform drift detected in banking-genai\"}"
```

**Remediation strategy** depends on the cause:

1. **If the manual change was necessary** (e.g., emergency scaling of database): update the Terraform code to match the new desired state, then apply. This makes the code the source of truth again.

2. **If the manual change was unauthorized**: run `terraform apply` to revert the infrastructure back to the code-defined state, then investigate the unauthorized change through audit logs.

3. **If drift is caused by cloud provider behavior**: this is a Terraform bug — report it and apply a workaround in the code.

**Prevention** is better than cure: restrict manual changes through RBAC (only CI/CD service accounts can modify production infrastructure), audit all manual changes, and run drift detection on every CI/CD run (not just scheduled). ArgoCD can also detect Kubernetes drift by comparing the declared manifest against the actual cluster state.

In banking, drift detection reports must be reviewed weekly by the platform team, and any unresolved drift older than 7 days must be escalated to engineering leadership. The compliance team includes drift detection results in their quarterly audit reports.

**Key Points to Hit:**
- [ ] Drift: actual infra differs from Terraform code — caused by manual changes, bugs, cloud provider
- [ ] Detection: scheduled `terraform plan -detailed-exitcode`, alert on exit code 2
- [ ] Remediation: if change was needed, update code; if unauthorized, apply to revert back
- [ ] Prevention: RBAC restrictions, audit all manual changes, drift detection on every CI/CD run
- [ ] Banking: weekly review, 7-day escalation, quarterly compliance audit inclusion

**Follow-Up Questions:**
1. How do you handle emergency manual changes in production without causing drift?
2. How does ArgoCD detect and remediate Kubernetes configuration drift?
3. What metrics do you track to measure drift trends over time?

**Source:** `banking-genai-engineering-academy/cicd-devops/drift-detection.md`, `banking-genai-engineering-academy/cicd-devops/terraform.md`

---

### Q17: 🔴 How do you design a complete security scanning pipeline covering SAST, SCA, DAST, secret detection, container scanning, and IaC scanning?

**Strong Answer:**

A complete security scanning pipeline for a banking GenAI platform runs six distinct scan types, each at the optimal point in the CI/CD flow. The goal is to catch vulnerabilities as early as possible (shift-left) while maintaining comprehensive coverage.

**SAST (Static Application Security Testing)** runs on every PR using CodeQL. It analyzes source code for security vulnerabilities like SQL injection, XSS, and insecure crypto. It runs in the CI pipeline, blocking merges on CRITICAL findings.

**SCA (Software Composition Analysis)** runs on every build using Trivy filesystem scan. It checks third-party dependencies (requirements.txt, package.json) for known CVEs. It is faster than SAST and should run in parallel.

**DAST (Dynamic Application Security Testing)** runs nightly against the staging environment using OWASP ZAP. Unlike SAST/SCA, DAST tests the running application — it sends malicious requests to find vulnerabilities like authentication bypass and injection flaws. DAST cannot run on every PR because it needs a deployed application.

**Secret Detection** runs on every PR using gitleaks. It scans commit history for leaked API keys, tokens, and credentials. In banking, a leaked secret is a P1 incident.

**Container Scanning** runs on every image build using Trivy. It checks the container image for OS-level and language-level vulnerabilities.

**IaC Scanning** runs on infrastructure PRs using Checkov or tfsec. It checks Terraform files for misconfigurations like publicly accessible S3 buckets or unencrypted databases.

```yaml
# Complete security scanning pipeline
jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ['python', 'javascript']
          queries: security-extended
      - uses: github/codeql-action/analyze@v3

  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2

  container-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'genai-api:${{ github.sha }}'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform/
          framework: terraform
```

When a CRITICAL vulnerability is found: the pipeline fails immediately, the PR is blocked, the security team is notified via Slack, and a ticket is auto-created with the vulnerability details. The developer must fix the vulnerability (upgrade dependency, patch code) before the PR can be merged. For false positives, the security team reviews and documents an exception with an expiration date.

**Key Points to Hit:**
- [ ] Six scan types: SAST (CodeQL, on PR), SCA (Trivy fs, on build), DAST (ZAP, nightly), secrets (gitleaks, on PR), container (Trivy image, on build), IaC (Checkov, on infra PR)
- [ ] CRITICAL always blocks, HIGH tracked with SLA, false positive exception process
- [ ] DAST runs nightly against staging — needs deployed application
- [ ] All scan results uploaded to GitHub Security dashboard via SARIF
- [ ] CRITICAL finding: pipeline fails, security team notified, auto-ticket created

**Follow-Up Questions:**
1. How do you reduce the time impact of running all six scans without sacrificing coverage?
2. What is your process for handling a CVE that has no available fix but is flagged as CRITICAL?
3. How do you integrate SBOM data with vulnerability scanning for supply chain security?

**Source:** `banking-genai-engineering-academy/cicd-devops/security-scanning.md`, `banking-genai-engineering-academy/cicd-devops/container-scanning.md`

---

### Q18: 🔴 How do you generate and use SBOMs for supply chain security in a banking CI/CD pipeline?

**Strong Answer:**

An SBOM (Software Bill of Materials) is a comprehensive inventory of all components, libraries, and dependencies in a software artifact. In banking, SBOMs are increasingly required for regulatory compliance and are essential for supply chain security — when a new CVE like Log4Shell emerges, you need to know within minutes which of your services are affected.

There are three main formats: **CycloneDX** (OWASP-maintained, JSON/XML, recommended for banking), **SPDX** (Linux Foundation, more verbose), and **Syft** (Anchore's format, can output CycloneDX).

In a banking CI/CD pipeline, you generate **two SBOMs** and merge them: (1) application-level SBOM from the dependency files (requirements.txt, package.json) using cyclonedx-bom, and (2) container image SBOM using Syft, which includes OS-level packages. The merged SBOM represents the complete dependency tree.

```yaml
# SBOM generation in CI/CD
jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Application-level SBOM
      - name: Generate Python SBOM
        run: |
          pip install cyclonedx-bom
          cyclonedx-py --requirements requirements.txt \
            --output sbom-cyclonedx.json --output-format json

      # Container image SBOM with Syft
      - name: Generate Image SBOM
        uses: anchore/sbom-action@v0
        with:
          image: quay.io/banking/genai-api:${{ github.sha }}
          format: cyclonedx-json
          output-file: sbom-image.json

      # Upload SBOM as build artifact
      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom-cyclonedx.json

      # Store in artifact registry for compliance retention
      - name: Upload to S3
        run: |
          aws s3 cp sbom-cyclonedx.json \
            s3://banking-sboms/genai-api/${{ github.sha }}/sbom.json
```

Once generated, the SBOM is used for: (1) **vulnerability impact analysis** — when a new CVE is published, query your SBOM database to find affected services; (2) **license compliance** — check that all dependencies have approved licenses (no GPL in proprietary banking code); (3) **regulatory compliance** — provide SBOMs to auditors as evidence of supply chain transparency; (4) **deployment linkage** — each deployment record includes the SBOM, so auditors can see exactly what dependencies were in production at any point in time.

The SBOM retention policy should match the regulatory retention period — typically 1-7 years depending on the jurisdiction. SBOMs should be stored in an immutable artifact registry with access logging.

**Key Points to Hit:**
- [ ] SBOM: complete inventory of all components/dependencies — required for supply chain security
- [ ] Two SBOMs: application-level (cyclonedx-bom) + image-level (Syft), merged for completeness
- [ ] CycloneDX format recommended (OWASP, machine-readable, industry standard)
- [ ] Uses: vulnerability impact analysis, license compliance, regulatory compliance, deployment linkage
- [ ] Retention: 1-7 years in immutable artifact registry with access logging

**Follow-Up Questions:**
1. How do you query SBOMs to find services affected by a newly disclosed CVE?
2. How do you handle license compliance checks from SBOM data?
3. What regulations currently require SBOMs and what is the enforcement timeline?

**Source:** `banking-genai-engineering-academy/cicd-devops/sbom-generation.md`, `banking-genai-engineering-academy/cicd-devops/security-scanning.md`

---

### Q19: 🔴 How do you design release communication for a banking GenAI platform — pre-deployment, during deployment, and post-deployment?

**Strong Answer:**

Release communication ensures all stakeholders — engineering teams, SRE, product management, compliance, customer support, and leadership — are informed about what is changing, when, and what the impact is. In banking, poor release communication can cause compliance escalations, customer complaints, and regulatory scrutiny.

**Pre-deployment (48-24 hours before):** Update the release calendar, send email to the stakeholder mailing list, and post in the #genai-releases Slack channel. The release notes should include: version, deployment window, risk level, change request ID, summary of changes, impact assessment, rollback plan, testing evidence, and contacts.

**During deployment:** Live updates in the #genai-releases channel, status page updates, and a war room conference bridge for major releases. Each pipeline stage completion posts an automated update: "Stage 3/8: Security scan passed, 0 vulnerabilities." If the deployment fails, the incident communication protocol activates: @channel in #genai-incidents, PagerDuty notification, and customer-facing status page update if users are affected.

**Post-deployment:** Deployment completion notification via Slack and email, release notes published to the internal documentation site, change request closed in the tracking system, and a post-release summary sent to leadership within 24 hours.

```yaml
communication_channels:
  pre_deployment:
    - "Slack: #genai-releases (24 hours before)"
    - "Email: Stakeholder mailing list (48 hours before)"
    - "Dashboard: Release calendar updated"

  during_deployment:
    - "Slack: Live updates in #genai-releases"
    - "Status page: Deployment status updated"
    - "War room: Conference bridge for major releases"

  post_deployment:
    - "Slack: Deployment completion notification"
    - "Email: Release summary to stakeholders"
    - "Dashboard: Release notes published"
    - "Jira: Change request closed"
```

Automated notifications are sent via GitHub Actions at each stage:

```yaml
- name: Notify Slack on completion
  if: success()
  run: |
    curl -X POST $SLACK_WEBHOOK \
      -H 'Content-Type: application/json' \
      -d "{
        \"blocks\": [
          {
            \"type\": \"header\",
            \"text\": {
              \"type\": \"plain_text\",
              \"text\": \"Deployment Successful: GenAI API v${{ steps.release.outputs.version }}\"
            }
          }
        ]
      }"
```

For GenAI releases specifically, the communication must include model behavior changes: "The new model version reduces hallucination rate from 2.1% to 1.2% and improves response relevance from 82% to 89%. No changes to response format or API contract."

**Key Points to Hit:**
- [ ] Pre-deployment (48h): email, Slack, release calendar with full release notes
- [ ] During deployment: live Slack updates, status page, war room for major releases
- [ ] Post-deployment: completion notification, published notes, change request closed, leadership summary
- [ ] Automated notifications via GitHub Actions at each pipeline stage
- [ ] GenAI-specific: communicate model behavior changes (hallucination rate, relevance scores)

**Follow-Up Questions:**
1. How do you handle communication when a deployment fails in the middle of the night?
2. What is your escalation communication protocol for a P1 incident caused by a bad release?
3. How do you collect and incorporate stakeholder feedback after a release?

**Source:** `banking-genai-engineering-academy/cicd-devops/release-communication.md`, `banking-genai-engineering-academy/cicd-devops/release-processes.md`

---

### Q20: 🔴 How do you handle the complete change management process in a regulated CI/CD environment — from change request to post-implementation review?

**Strong Answer:**

Change management in banking is not optional — it is a regulatory requirement under SOX, PCI-DSS, and GDPR. Every change to production systems must be documented, reviewed, approved, implemented, verified, and audited. The challenge is making this process rigorous without making it so slow that it blocks delivery.

The process has seven stages:

**1. Change Request Creation:** Automated from the Pull Request. When a PR is opened against main, a CI workflow creates a change request ticket with a unique ID (e.g., CR-2025-0015). The ticket includes: PR description, files changed, risk assessment, affected services, and rollback procedure.

**2. Risk Assessment:** Automated based on the change content. If the PR touches only bug fixes with no DB or infra changes, it is classified as low-risk. If it includes new features or DB migrations, it is medium-risk. Breaking API changes or GenAI model changes are high-risk.

**3. Review:** Low-risk changes are auto-approved if all CI gates pass. Medium-risk changes require team lead and SRE review. High-risk changes go to the CAB.

**4. CAB Approval:** The Change Advisory Board meets weekly. They review the change request, assess impact on customers and other services, verify the rollback procedure is tested, check that stakeholder notification was sent, and approve or reject.

**5. Implementation:** The CI/CD pipeline deploys the change. Every deployment step is logged and linked to the change request.

**6. Verification:** Post-deployment monitoring confirms the change is working as expected. SLOs are checked, alerts are reviewed, and the release manager confirms success.

**7. Post-Implementation Review:** Within 48 hours, the team reviews the change outcome. Was it successful? Were there unexpected issues? What lessons were learned? Failed changes get a formal incident review with root cause analysis.

```yaml
sox_compliance:
  - "All changes to financial systems documented"
  - "Segregation of duties (developer != approver)"
  - "Audit trail of all changes"
  - "Quarterly access review"

pci_dss:
  - "Change management procedure documented"
  - "Changes tested in non-production"
  - "Emergency changes documented within 24 hours"
  - "Annual procedure review"
```

The key to making this work at AVP level is **automation**. The change request is auto-created from the PR. Risk classification is automated based on file paths and change types. Approval workflows are automated through GitHub Environments. Deployment logs are automatically linked to the change request. The audit trail is generated without any manual effort from the development team.

For emergency changes, the process is: deploy first (to resolve the incident), file the change request within 24 hours, conduct a post-implementation review within 48 hours, and document why the normal process was bypassed. Emergency changes are reviewed quarterly by compliance to identify patterns that indicate the normal process needs improvement.

**Key Points to Hit:**
- [ ] Seven stages: change request, risk assessment, review, CAB approval, implementation, verification, post-implementation review
- [ ] Automated change request creation from PR with risk classification
- [ ] SOX: segregation of duties, audit trail, quarterly access review
- [ ] PCI-DSS: tested in non-prod, emergency changes documented within 24 hours
- [ ] Emergency changes: deploy first, retroactive change request within 24 hours, PIR within 48 hours

**Follow-Up Questions:**
1. How do you implement segregation of duties in a GitHub Actions workflow?
2. What happens if a change is approved by CAB but fails during deployment?
3. How do you prepare for a quarterly compliance audit of your CI/CD change management process?

**Source:** `banking-genai-engineering-academy/cicd-devops/change-management.md`, `banking-genai-engineering-academy/cicd-devops/production-approvals.md`, `banking-genai-engineering-academy/cicd-devops/release-communication.md`
