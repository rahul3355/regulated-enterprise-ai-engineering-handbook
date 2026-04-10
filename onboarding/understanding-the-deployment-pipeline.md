# Understanding the Deployment Pipeline

> **Audience:** All engineers on the GenAI Platform Team
> **Purpose:** From PR to production — every stage, gate, and approval
> **Prerequisites:** Basic Git and CI/CD knowledge

---

## Overview

Our deployment pipeline follows a GitOps model with progressive delivery. Every change goes through the same pipeline, regardless of who writes it — from interns to the CTO.

```
PR Created
    │
    ▼
┌─────────────────┐
│  1. PRE-COMMIT  │  Local hooks: lint, format, secrets scan
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2. CI BUILD    │  Jenkins: build, unit tests, SAST scan
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  3. CI TEST     │  Integration tests, DAST scan, OWASP dep check
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  4. CODE REVIEW │  Min 2 approvals, 1 senior+, security if guardrails
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  5. MERGE       │  Squash merge to main, semantic version tag
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  6. BUILD IMAGE │  Container build, Trivy scan, sign + push to registry
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  7. DEPLOY DEV  │  ArgoCD auto-deploys to dev cluster
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  8. DEV VERIFY  │  Smoke tests, e2e tests on dev
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  9. CAB CHECK   │  Change Advisory Board review (if change type requires)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 10. DEPLOY STG  │  ArgoCD deploys to staging (mirrors prod)
└────────┬────────┘
         │
         │
         ▼
┌─────────────────┐
│ 11. STG VERIFY  │  Full e2e, performance, security tests
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 12. PROD APPROVAL│  Manual gate: Tech Lead + Eng Manager approve
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 13. PROD DEPLOY │  Canary (5%) -> Progressive (25%, 50%, 100%)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 14. POST-DEPLOY │  Health checks, monitoring, feature flag activation
└─────────────────┘
```

**Total pipeline time (typical):**
- PR to dev: 15-20 minutes
- Dev to staging: 1-2 hours (including CAB if required)
- Staging to prod: Scheduled (typically 18:00-20:00 GMT, outside banking hours)

---

## Stage 1: Pre-Commit Hooks

Before you even push, pre-commit hooks run locally:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-merge-conflict
      - id: detect-private-key        # Prevents committing secrets
  
  - repo: https://github.com/psf/black
    rev: 24.2.0
    hooks:
      - id: black
        language_version: python3.11
  
  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
        args: ["--max-line-length=120", "--extend-ignore=E203"]
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies:
          - types-requests
          - types-redis
          - pydantic
  
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.1
    hooks:
      - id: gitleaks
        # Scans for secrets, tokens, keys
        # Bank policy: no secrets in code, ever
  
  - repo: local
    hooks:
      - id: pytest-quick
        name: Quick test suite
        entry: make test-unit
        language: system
        types: [python]
        pass_filenames: false
        # Runs unit tests (< 30 seconds total)
```

**What happens if a hook fails:**

```
$ git commit -m "feat: add new guardrails check"

trim trailing whitespace.................................................Passed
fix end of files.........................................................Passed
check yaml...............................................................Passed
check for merge conflicts................................................Passed
detect private key.......................................................Passed
black....................................................................Passed
flake8...................................................................Failed
- hook id: flake8
- exit code: 1

src/genai_platform/guardrails/checks/new_check.py:45:80: E501 line too long
src/genai_platform/guardrails/checks/new_check.py:67:1: F841 local variable 'unused' is assigned to but never used

Commit blocked. Fix the issues and try again.
```

**Bypassing hooks (emergency only):**

```bash
# NEVER do this for production code
# Only acceptable for: documentation-only changes with broken hook config
git commit --no-verify -m "docs: fix typo"

# This is logged and audited. If you bypass hooks for code changes,
# you will be called out in the PR review.
```

---

## Stage 2-3: CI Build and Test (Jenkins)

When you push a branch, Jenkins starts the pipeline:

```groovy
// Jenkinsfile (in each repository)
pipeline {
    agent {
        kubernetes {
            yamlFile 'Jenkins-agent.yaml'
            // Each build runs in an isolated container
        }
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                // Shallow clone for speed
            }
        }
        
        stage('Build') {
            steps {
                sh 'make build'
                // Creates virtual environment, installs deps, compiles
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'make test-unit'
                // Fast tests only, < 5 minutes
            }
            post {
                always {
                    junit 'reports/unit-tests/*.xml'
                    // Publishes test results to Jenkins UI
                }
            }
        }
        
        stage('Code Quality') {
            steps {
                sh 'make lint'
                sh 'make type-check'
                sh 'make security-scan'
                // SonarQube analysis
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh 'make test-integration'
                // Uses Testcontainers for Redis, PostgreSQL, etc.
                // Test environment mirrors production configuration
            }
            post {
                always {
                    junit 'reports/integration-tests/*.xml'
                }
            }
        }
        
        stage('Security Scans') {
            parallel {
                stage('SAST') {
                    steps {
                        // Static Application Security Testing
                        sh 'semgrep --config bank-security-rules --json -o reports/semgrep.json src/'
                    }
                }
                stage('DAST Prep') {
                    steps {
                        // Dynamic Application Security Testing
                        // Build the application and run security scans
                        sh 'make dast-scan'
                    }
                }
                stage('Dependency Scan') {
                    steps {
                        // OWASP Dependency-Check
                        sh 'make dependency-check'
                        // Fails if any CRITICAL or HIGH vulnerabilities in dependencies
                    }
                }
                stage('Container Scan') {
                    steps {
                        // Trivy container scan (runs after image build)
                        sh 'trivy fs --severity CRITICAL,HIGH --exit-code 1 .'
                    }
                }
            }
        }
        
        stage('Compliance Gate') {
            steps {
                // Bank-specific compliance checks
                sh 'make compliance-check'
                // Checks:
                // - No production secrets in code or env vars
                // - All config values come from Vault or ConfigMap
                // - Audit logging is not disabled in any code path
                // - PII handling follows data protection policy
                // - Change has associated JIRA ticket (enforced via commit message)
            }
        }
    }
    
    post {
        success {
            // Notify Slack, update PR status
            sh 'curl -X POST $SLACK_WEBHOOK -d "{\"text\": \"CI passed for $JOB_NAME\"}"'
        }
        failure {
            // Notify PR author and reviewers
            sh 'curl -X POST $SLACK_WEBHOOK -d "{\"text\": \"CI FAILED for $JOB_NAME: $BUILD_URL\"}"'
        }
    }
}
```

**CI Gate Summary:**

| Gate | Tool | What It Checks | Failure Action |
|------|------|---------------|----------------|
| Unit Tests | pytest | Test correctness | Block merge |
| Integration Tests | pytest + Testcontainers | Service interactions | Block merge |
| Code Quality | SonarQube | Code smells, coverage, duplications | Warning (blocks if quality gate fails) |
| SAST | Semgrep | Security vulnerabilities in code | Block merge |
| DAST | OWASP ZAP | Runtime vulnerabilities | Block merge |
| Dependency Scan | OWASP Dep-Check | Vulnerable dependencies | Block merge if CRITICAL/HIGH |
| Container Scan | Trivy | Container vulnerabilities | Block merge if CRITICAL |
| Compliance | Custom scripts | Bank policy adherence | Block merge |

---

## Stage 4: Code Review

### Minimum Approval Requirements

| Change Type | Approvals Required | Special Requirements |
|------------|-------------------|---------------------|
| Documentation only | 1 | None |
| Bug fix (non-critical) | 2 | At least 1 senior+ |
| Feature addition | 2 | At least 1 senior+, test coverage check |
| Guardrails change | 2 | **Security team approval required** |
| Infrastructure change | 2 | **SRE team approval required** |
| Database migration | 2 | **DBA approval required** |
| Security-sensitive change | 3 | Security + SRE + Tech Lead |
| Emergency hotfix | 1 | **Post-hoc review within 24 hours** |

### PR Template

Every PR auto-populates this template:

```markdown
## JIRA Ticket
[Link to ticket]

## What
[One sentence: what does this change do?]

## Why
[Business justification, link to requirements]

## How
[Brief description of the approach]

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed (describe)
- [ ] Tested in dev environment
- [ ] Tested in staging environment (if applicable)

## Risk Assessment
- Risk level: [LOW / MEDIUM / HIGH]
- Rollback plan: [How to revert if something goes wrong]
- Impact on consumers: [Any consumer-facing changes?]
- Impact on compliance: [Any compliance-relevant changes?]
- Impact on cost: [Any AI provider cost implications?]

## Checklist
- [ ] Code follows style guide
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated (if applicable)
- [ ] Feature flag created (if deploying new behavior)
- [ ] Runbook updated (if operational change)
- [ ] Monitoring alerts configured (if new failure mode)
- [ ] CAB ticket created (if required for change type)

## Screenshots / Output
[If applicable: screenshots, test output, curl commands]
```

### Common Review Comments

```
# Comment types you will see frequently:

# 1. Security concern
"🔒 SECURITY: This input is not validated before being passed to the LLM.
 An attacker could inject a prompt. Please add input validation using
 the InputValidator class. See: src/genai_platform/security/validator.py"

# 2. Compliance concern
"⚖️ COMPLIANCE: This change modifies audit logging behavior. Please ensure
 the compliance team has reviewed this. Reference: COMP-2024-0892"

# 3. Missing test
"🧪 TEST: Please add a test for the edge case where the provider returns
 an empty response. This is covered by the circuit breaker but not tested."

# 4. Performance concern
"⚡ PERFORMANCE: This loop makes a database query per iteration. Consider
 batching with a single query. See: https://wiki.bank.internal/n-plus-one"

# 5. Observability gap
"📊 OBSERVABILITY: Please add structured logging here so we can debug
 this in production. Include: request_id, consumer_id, and the decision."

# 6. Feature flag reminder
"🚩 FEATURE FLAG: This behavior change should be behind a flag. Please
 create one in LaunchDarkly and add the check. Flag name suggestion:
 new-{feature-name}-{jira-ticket-number}"
```

---

## Stage 5-6: Merge and Container Build

After approval and merge:

```bash
# The merge triggers the post-merge pipeline automatically:

1. Jenkins detects the merge to main (webhook)
2. Builds the container image:
   docker build -t genai-platform-core:2.14.0-rc3 .

3. Tags the image:
   docker tag genai-platform-core:2.14.0-rc3 registry.bank.internal/genai-platform-core:2.14.0-rc3
   docker tag genai-platform-core:2.14.0-rc3 registry.bank.internal/genai-platform-core:latest

4. Runs Trivy scan on the final image:
   trivy image registry.bank.internal/genai-platform-core:2.14.0-rc3
   # Must have 0 CRITICAL vulnerabilities

5. Signs the image (cosign):
   cosign sign --key $SIGNING_KEY registry.bank.internal/genai-platform-core:2.14.0-rc3

6. Pushes to registry:
   docker push registry.bank.internal/genai-platform-core:2.14.0-rc3
   docker push registry.bank.internal/genai-platform-core:latest

7. Updates ArgoCD manifest:
   # genai-platform-infra/argocd/applications/core.yaml
   spec:
     source:
       targetRevision: "2.14.0-rc3"  # Updated automatically
```

---

## Stage 7-8: Deploy to Dev and Verify

ArgoCD detects the manifest change and deploys automatically:

```
ArggoCD Application Status: genai-platform-core (dev)

Health:     Healthy
Sync:       Synced
Revision:   2.14.0-rc3 (5 minutes ago)

Resources:
  Deployment/genai-platform-core
    ├── Pod/genai-platform-core-7d8f9b6c4-x2k9p   Running (ready: 1/1)
    ├── Pod/genai-platform-core-7d8f9b6c4-m4n7j   Running (ready: 1/1)
    └── Pod/genai-platform-core-7d8f9b6c4-p8q3r   Running (ready: 1/1)

Events:
  5m  Normal  ScalingReplicaSet  Deployment  Scaled up replica set to 3
  5m  Normal  Pulling            Pod         Pulling image "registry.bank.internal/genai-platform-core:2.14.0-rc3"
  4m  Normal  Created            Pod         Created container genai-platform-core
  4m  Normal  Started            Pod         Started container genai-platform-core
  4m  Normal  HealthCheckPassed  Pod         Liveness probe succeeded
```

**Post-deployment smoke tests:**

```bash
# Automated smoke test suite runs after dev deployment
make smoke-test ENV=dev

# Output:
Running smoke tests against dev environment...

✓ Health check: GET /health -> 200 (45ms)
✓ Readiness check: GET /ready -> 200 (12ms)
✓ API version: GET /version -> 200, body matches expected version
✓ Consumer auth: POST /v1/chat with valid API key -> 200
✓ Consumer auth: POST /v1/chat with invalid API key -> 401
✓ Guardrails: Prompt with injection pattern -> denied
✓ Guardrails: Normal prompt -> allowed
✓ RAG pipeline: Query returns documents (retrieval count > 0)
✓ LLM generation: Response is non-empty and well-formed

All 9 smoke tests passed in 3.2 seconds.
```

---

## Stage 9: CAB Review

### What is CAB?

The **Change Advisory Board** (CAB) reviews changes before they reach staging/production. This is a regulatory requirement for banks (part of SOX, Basel III, and internal audit frameworks).

### Which Changes Need CAB?

| Change Type | CAB Required? | Review Time |
|------------|---------------|-------------|
| Documentation only | No | N/A |
| Bug fix (tested, low risk) | No (auto-approved) | N/A |
| Feature addition | Yes | 1-2 business days |
| Guardrails change | Yes (expedited) | Same day |
| Infrastructure change | Yes | 2-3 business days |
| Database migration | Yes | 2-3 business days |
| Emergency hotfix | No (post-hoc review) | Within 24 hours |

### CAB Submission Template

```
CHANGE REQUEST: CR-2024-5678

Title: Deploy genai-platform-core v2.14.0
Requester: Jane Smith (jane.smith@bank.com)
Team: GenAI Platform
Date: 2024-12-05

Change Description:
- Add new financial advice disclaimer guardrail check
- Update reranker model to v2 for improved multilingual support
- Fix token counting bug in cost tracker

Risk Assessment: LOW
- All changes are behind feature flags
- Guardrails change approved by Security (ref: SEC-REV-2024-0891)
- Rollback plan: revert to v2.13.0 via ArgoCD (tested, < 5 minutes)

Testing Evidence:
- Unit tests: 347 passed, 0 failed
- Integration tests: 89 passed, 0 failed
- E2E tests: 23 passed, 0 failed
- Security scan: 0 vulnerabilities
- Performance test: P50 latency unchanged, P99 improved by 5%

Deployment Plan:
- Scheduled: 2024-12-05, 18:00-20:00 GMT
- Method: Progressive rollout via ArgoCD
  - 18:00 Canary (5% traffic)
  - 18:15 25% traffic
  - 18:30 50% traffic
  - 18:45 100% traffic
- Rollback: ArgoCD rollback to previous revision
- Stakeholders notified: Yes (wealth-digital, investment-analytics teams)

Backout Plan:
If any of the following occur, rollback immediately:
- Error rate exceeds 1% for more than 2 minutes
- P99 latency exceeds 5 seconds for more than 2 minutes
- Guardrails detection rate drops below 95%
- Any consumer reports functional regression
```

---

## Stage 10-11: Deploy to Staging and Verify

Staging mirrors production:

```
Staging Cluster Configuration:
- Same Kubernetes version as production (OpenShift 4.14)
- Same node pool configuration (scaled down: 3 nodes vs 12 in prod)
- Same service mesh configuration (Istio 1.20)
- Same external dependencies (Azure OpenAI, vector DB, Redis)
- Same feature flags (copied from production, but can be overridden)
- Realistic data: Anonymized production data (refreshed weekly)
```

**Staging verification includes:**

```bash
# Full test suite against staging
make verify ENV=staging

# Includes:
# 1. Smoke tests (same as dev)
# 2. Full e2e test suite (real API calls, real LLM responses)
# 3. Performance test (load test at 2x expected production traffic)
# 4. Security test (penetration test suite, if significant changes)
# 5. Data validation (verify no data leakage between consumers)
# 6. Disaster recovery test (simulated pod failure, verify self-healing)
```

**Performance test results (example):**

```
Load Test Report: genai-platform-core v2.14.0
Date: 2024-12-05
Environment: staging

Configuration:
- Duration: 30 minutes
- Ramp-up: 5 minutes
- Concurrent users: 500 (equivalent to 2x peak production)
- Target: https://staging-api-gateway.platform.bank

Results:
                    P50     P90     P99     Max     Error Rate
Request latency:    340ms   890ms   2,100ms 3,400ms 0.02%
Guardrails check:   45ms    120ms   250ms   400ms   0.00%
Embedding gen:      35ms    80ms    150ms   200ms   0.00%
RAG retrieval:      120ms   340ms   680ms   1,100ms 0.01%
LLM generation:     800ms   1,800ms 3,200ms 5,000ms 0.03%

Comparison with v2.13.0:
- P50: +2% (within noise)
- P99: -5% (improved due to reranker update)
- Error rate: -0.01% (improved due to bug fix)

Verdict: PASS — Performance meets or exceeds previous version.
```

---

## Stage 12-13: Production Approval and Deployment

### The Production Gate

This is a **manual** approval step. ArgoCD will NOT deploy to production automatically.

```
Production Deployment Request
Application: genai-platform-core
Version: 2.14.0-rc3
Scheduled: 2024-12-05, 18:00 GMT

Approvals Required:
┌─────────────────────┬──────────┬─────────────────────────────────────┐
│ Approver            │ Status   │ Comments                            │
├─────────────────────┼──────────┼─────────────────────────────────────┤
│ Tech Lead           │ ✓        │ Reviewed staging results, approved  │
│ Eng Manager         │ ✓        │ CAB approved, stakeholders notified │
│ Security (optional) │ ✓        │ Guardrails change reviewed          │
│ SRE (optional)      │ N/A      │ No infrastructure changes           │
└─────────────────────┴──────────┴─────────────────────────────────────┘

Deployment Window:
- Start: 18:00 GMT (outside core banking hours)
- Expected duration: 45 minutes
- Rollback deadline: 19:30 GMT (must be stable before morning operations)

Click [APPROVE] to begin deployment.
Click [REJECT] to cancel and provide feedback.
```

### Progressive Rollout

Once approved, ArgoCD executes the progressive rollout:

```yaml
# ArgoCD Rollout configuration
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: genai-platform-core
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5       # 5% traffic (canary)
      - pause: {duration: 5m}   # Monitor for 5 minutes
      - setWeight: 25      # 25% traffic
      - pause: {duration: 5m}
      - setWeight: 50      # 50% traffic
      - pause: {duration: 10m}  # Longer pause at 50%
      - setWeight: 100     # 100% traffic
      - pause: {duration: 15m}  # Final monitoring period
      
      # Automated rollback conditions
      analysis:
        templates:
        - templateName: deployment-health
        args:
        - name: error-rate-threshold
          value: "1.0"
        - name: latency-p99-threshold
          value: "5000"
        - name: guardrails-detection-min
          value: "95"
```

**Real deployment timeline:**

```
18:00 — Deployment initiated by ArgoCD
18:01 — Canary pods (5% traffic) deployed and health-checked
18:02 — Canary serving traffic, metrics pipeline activated
18:07 — 5-minute monitoring window:
        - Error rate: 0.02% (threshold: 1.0%) ✓
        - P99 latency: 2,100ms (threshold: 5,000ms) ✓
        - Guardrails detection: 96.2% (threshold: 95%) ✓
18:07 — Auto-promoted to 25% traffic
18:12 — 5-minute monitoring window: all green ✓
18:12 — Auto-promoted to 50% traffic
18:22 — 10-minute monitoring window: all green ✓
18:22 — Auto-promoted to 100% traffic
18:37 — Final 15-minute monitoring window: all green ✓
18:37 — Deployment complete
18:38 — Feature flags activated for new functionality
18:40 — Stakeholder notification sent: "Deployment PROD-2024-1205 complete"
```

---

## Stage 14: Post-Deployment

### Immediate Actions (First 30 Minutes)

```
POST-DEPLOYMENT CHECKLIST:

1. [ ] Verify health checks (all pods healthy)
2. [ ] Verify error rate (below threshold)
3. [ ] Verify latency percentiles (P50, P90, P99 within bounds)
4. [ ] Verify guardrails metrics (detection rate, false positive rate)
5. [ ] Verify consumer functionality (spot-check with real queries)
6. [ ] Verify audit logging (decisions being logged)
7. [ ] Verify cost tracking (token counts accurate)
8. [ ] Check Splunk for any error patterns
9. [ ] Check Grafana dashboards for anomalies
10. [ ] Post deployment completion message in Slack
```

### Monitoring Period (First 24 Hours)

The deploying engineer is responsible for monitoring the deployment for 24 hours:

```
24-Hour Post-Deployment Monitoring:

Hour 1: Active monitoring (stay at desk)
- Watch Grafana dashboards
- Respond to any alerts immediately
- Be available in Slack for consumer questions

Hours 2-4: Periodic checking (every 15 minutes)
- Check error rate trend
- Check latency trend
- Check consumer feedback

Hours 5-8: Relaxed monitoring (every 30 minutes)
- Check for any overnight alerts
- Verify no consumer-reported issues

Hours 9-24: Normal on-call
- Standard alerting applies
- Next day: Check 24-hour metrics summary
```

---

## Emergency Hotfix Process

Sometimes you cannot wait for the full pipeline. For emergencies:

```
EMERGENCY HOTFIX PROCESS

1. Create hotfix branch from the production tag:
   git checkout -b hotfix/INC-2024-1205 prod-2.13.0

2. Make the minimal fix (one file change if possible)

3. Run critical tests locally:
   make test-unit && make security-scan

4. Commit with emergency message:
   git commit -m "HOTFIX: INC-2024-1205 - Roll back guardrails model
   [emergency] [inc:INC-2024-1205]"

5. Push and create PR with HOTFIX label:
   git push origin hotfix/INC-2024-1205
   # PR template auto-detects HOTFIX label and adjusts requirements

6. HOTFIX PR requirements:
   - 1 approval (from on-call lead or Tech Lead)
   - CI must pass (same gates, but can be overridden by Tech Lead with justification)
   - Security review can be post-hoc (within 24 hours)

7. After merge:
   - Deploy to staging (skip dev, save time)
   - Run smoke tests on staging (5 minutes)
   - Manual production approval (Tech Lead + Eng Manager, can be via phone)
   - Deploy to production (no progressive rollout, direct deploy if urgent)

8. Post-hotfix (within 24 hours):
   - File incident report
   - Create proper PR with full review
   - Run full test suite
   - Document why the hotfix was needed and how to prevent recurrence
```

**Hotfix example:**

```
INCIDENT: INC-2024-1205
Issue: Guardrails model regression (detection rate dropped to 78%)
Hotfix: Roll back to previous model version

Hotfix PR: #8847
Approvals: Tech Lead (phone approval, documented in PR)
Deploy time: 30 minutes from incident acknowledgment to production rollback
Result: Detection rate restored to 95.5%

Post-mortem finding: The regression should have been caught by the CI accuracy gate.
The gate was configured as WARNING instead of BLOCKING. Fixed in PR #8850.
```

---

## Common Deployment Failures and Fixes

| Failure | Cause | Fix |
|---------|-------|-----|
| CI fails on dependency scan | New dependency has known vulnerability | Update dependency version or request security exception |
| Integration tests timeout | Test container not starting | Check Dockerfile, ensure test DB version matches |
| SonarQube quality gate fails | Code coverage dropped below 80% | Add tests for new code |
| ArgoCD sync fails | Kubernetes manifest invalid (wrong API version) | Validate manifest with `kubectl --dry-run` |
| Canary auto-rollback | Error rate exceeded threshold | Investigate errors, fix, redeploy |
| CAB rejected | Insufficient testing evidence | Add more test results, re-submit |
| Production health check fails | Deployment has a bug not caught by tests | Rollback, fix in new PR, redeploy |
| Feature flag not working | Flag not created in LaunchDarkly | Create flag, verify configuration |

---

## Further Reading

- `cicd-devops/` — CI/CD configuration and tooling
- `kubernetes-openshift/` — Kubernetes deployment details
- `incident-management/` — Incident response and rollback procedures
- `infrastructure/` — Infrastructure as Code
- `first-30-days.md` — Your first production deployment experience
- `common-mistakes-new-engineers-make.md` — Deployment mistakes to avoid

---

*Last updated: April 2025 | Document owner: DevOps Engineering Team | Review cycle: Monthly*
