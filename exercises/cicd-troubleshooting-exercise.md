# CI/CD Troubleshooting Exercise

> Debug a failing CI/CD pipeline for a GenAI service — from build failure to production deployment issue.

## Problem Statement

The CI/CD pipeline for the GenAI chat service is failing. Multiple stages are broken. Diagnose and fix each issue.

## Pipeline Definition

```yaml
# .github/workflows/genai-chat.yml
name: GenAI Chat CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run linter
        run: ruff check .

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: pgvector/pgvector:pg15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: genai_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/genai_test
        run: pytest tests/ -v --cov=genai_service --cov-report=xml

  security:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        run: |
          pip install bandit safety
          bandit -r genai_service/ -f json -o bandit-report.json
          safety check -r requirements.txt

  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    steps:
      - uses: actions/checkout@v4
      - name: Build container image
        run: |
          docker build -t genai-chat:${{ github.sha }} .
      - name: Push to registry
        run: |
          docker tag genai-chat:${{ github.sha }} \
            registry.bank.internal/genai-chat:${{ github.sha }}
          docker push registry.bank.internal/genai-chat:${{ github.sha }}

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to staging
        run: |
          kubectl set image deployment/genai-chat \
            chat=registry.bank.internal/genai-chat:${{ github.sha }} \
            -n genai-staging
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/genai-chat \
            -n genai-staging --timeout=300s
      - name: Run smoke tests
        run: |
          curl -f https://genai-chat-staging.bank.internal/health

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - name: Deploy to production
        run: |
          kubectl set image deployment/genai-chat \
            chat=registry.bank.internal/genai-chat:${{ github.sha }} \
            -n genai-production
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/genai-chat \
            -n genai-production --timeout=300s
```

## Failures to Diagnose

### Failure 1: Lint Job Failing

```
Run ruff check .
genai_service/chat.py:45:1: F401 'os' imported but unused
genai_service/chat.py:78:20: F821 undefined name 'settings'
genai_service/retriever.py:12:1: E501 line too long (127 > 88)
genai_service/config.py:23:5: F841 local variable 'temp_key' is assigned to but never used
Error: Process completed with exit code 1.
```

**Diagnosis and fix:**

```bash
# The linter found actual issues in the code:
# 1. Unused import (F401) — remove 'import os'
# 2. Undefined name (F821) — 'settings' is referenced but not imported/defined
# 3. Line too long (E501) — format the line
# 4. Unused variable (F841) — remove or use temp_key

# Fix: Run ruff with --fix for auto-fixable issues
ruff check --fix .

# Then manually fix the remaining issues (F821, F841)
# Commit the fixes and re-run the pipeline
```

### Failure 2: Test Job Failing

```
Run pytest tests/ -v --cov=genai_service
============================= test session starts ==============================
collected 47 items

tests/test_chat.py ...........F...                                       [ 34%]
tests/test_retriever.py .......E...                                      [ 59%]
tests/test_auth.py ..........                                            [ 80%]
tests/test_audit.py ..........                                           [100%]

==================================== ERRORS ====================================
____________ ERROR at setup of test_retrieval_with_access_control ____________

fixturedef = <FixtureDef argname='db_session' scope='function' baseid='tests'>
request = <SubRequest 'db_session' for <Function test_retrieval_with_access_control>>

    @pytest.fixture
    def db_session():
        engine = create_engine(os.environ["TEST_DATABASE_URL"])
>       with engine.begin() as conn:
E       psycopg2.OperationalError: connection to server at "localhost" (::1),
E       port 5432 failed: FATAL:  password authentication failed for user "test"

ERROR tests/test_retriever.py::test_retrieval_with_access_control -
  psycopg2.OperationalError
=================================== FAILURES ===================================
_______________________ test_chat_endpoint_auth_required _______________________

    def test_chat_endpoint_auth_required():
        client = TestClient(app)
        response = client.post("/v1/chat", json={
            "query": "test",
            "api_key": "invalid-key"
        })
>       assert response.status_code == 401
E       assert 500 == 401

FAILED tests/test_chat.py::test_chat_endpoint_auth_required
============== 1 failed, 1 error, 45 passed in 12.34s ==============
```

**Diagnosis and fix:**

```python
# Error 1: Database connection failure
# The fixture uses TEST_DATABASE_URL but the env var in the pipeline is
# DATABASE_URL. The fixture should use the same variable name.

# Fix in conftest.py:
@pytest.fixture
def db_session():
    # Use the same env var name as the pipeline sets
    engine = create_engine(os.environ.get("DATABASE_URL"))
    with engine.begin() as conn:
        yield conn

# Failure 2: Auth test returns 500 instead of 401
# The auth middleware raises an exception that isn't caught properly.

# Fix in chat.py:
# The authenticate() function raises HTTPException but it's not being
# caught in the endpoint. Add exception handling:

@app.post("/v1/chat")
async def chat(request: ChatRequest):
    try:
        user = await authenticate(request.api_key)
    except HTTPException as e:
        return JSONResponse(status_code=e.status_code, content=e.detail)
    # ... rest of the handler
```

### Failure 3: Security Scan Failing

```
Run safety check -r requirements.txt
+==============================================================================+

                               /$$$$$$            /$$
                              /$$__  $$          | $$
           /$$$$$$$  /$$$$$$ | $$  \__//$$$$$$  /$$$$$$
          | $$__  $$/$$__  $$| $$$$   /$$__  $$|_  $$_/
          | $$  \ $$ $$  \ $$| $$_/  | $$$$$$$$  | $$
          | $$  | $$ $$  | $$| $$    | $$_____/  | $$ /$$
          | $$  | $$  $$$$$$$| $$    |  $$$$$$$  |  $$$$/
          |__/  |__/\____  $$|__/     \_______/   \___/
                    /$$  \ $$
                   |  $$$$$$/
                    \______/

          safety v3.0.0
          Python 3.11.5

  found packages: 45
  scan timestamp: 2026-04-10 14:23:01

+==============================================================================+

VULNERABILITIES FOUND
=====================

  Package: urllib3
  Installed: 2.0.7
  Vulnerable: < 2.1.0
  CVE: CVE-2024-XXXX
  Advisory: urllib3 does not validate TLS certificates properly,
            allowing MITM attacks.
  Fix: Upgrade to >= 2.1.0

  Package: cryptography
  Installed: 41.0.7
  Vulnerable: < 42.0.0
  CVE: CVE-2024-YYYY
  Advisory: cryptography has a vulnerability in X.509 parsing.
  Fix: Upgrade to >= 42.0.0

Reported 2 vulnerabilities
```

**Diagnosis and fix:**

```bash
# Fix: Update the vulnerable dependencies
# Update requirements.txt:
urllib3>=2.1.0
cryptography>=42.0.0

# Then update the lock file and re-run
pip install -r requirements.txt
pip freeze > requirements.txt

# Re-run security scan to verify
safety check -r requirements.txt
# No vulnerabilities found ✓
```

### Failure 4: Deployment Failing

```
Run kubectl set image deployment/genai-chat ...
deployment.apps/genai-chat image updated

Run kubectl rollout status deployment/genai-chat -n genai-staging
Waiting for deployment "genai-chat" rollout to finish:
  0 of 4 updated replicas are available...
  1 of 4 updated replicas are available...
  2 of 4 updated replicas are available...
error: deployment "genai-chat" exceeded its progress deadline

$ kubectl get events -n genai-staging --sort-by='.lastTimestamp'
LAST SEEN   TYPE      REASON      OBJECT                      MESSAGE
2m          Warning   Failed      pod/genai-chat-abc12        Failed to pull image
                                   "registry.bank.internal/genai-chat:sha256:xyz":
                                   rpc error: code = Unknown desc =
                                   failed to pull and unpack image:
                                   unauthorized: authentication required
```

**Diagnosis and fix:**

```bash
# The staging cluster can't authenticate to the internal registry.
# The image pull secret is missing or expired.

# Check if the image pull secret exists
$ kubectl get secrets -n genai-staging | grep registry
registry-credentials   kubernetes.io/dockerconfigjson   1   30d

# The secret exists but may be expired. Regenerate it:
$ kubectl create secret docker-registry registry-credentials \
    --docker-server=registry.bank.internal \
    --docker-username=genai-deploy \
    --docker-password=$(cat /path/to/token) \
    -n genai-staging \
    --dry-run=client -o yaml | kubectl apply -f -

# Ensure the deployment references the secret:
$ kubectl get deployment genai-chat -n genai-staging -o yaml | grep imagePullSecrets
      imagePullSecrets:
      - name: registry-credentials

# If it's missing, add it:
$ kubectl patch deployment genai-chat -n genai-staging \
    -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"registry-credentials"}]}}}}'
```

## Prevention Measures

1. **Pre-commit hooks:** Run ruff and security checks before pushing code
2. **Dependency scanning:** Automated weekly dependency updates via Dependabot/Renovate
3. **Pipeline notifications:** Alert the team when any stage fails
4. **Deployment health checks:** Automated rollback if health check fails after deployment
5. **Image pull secret rotation:** Automate secret rotation before expiry

## Extensions

1. **Add a canary deployment stage:** Deploy to 10% of production pods first, monitor error rate, then full rollout.

2. **Add performance testing stage:** Run load tests in staging before production deployment. Fail if p95 latency regresses.

3. **Implement pipeline-as-code review:** Every pipeline change requires review before merging.

4. **Build a pipeline dashboard:** Visualize build times, failure rates, and flaky tests across all services.

5. **Add security gates:** Block deployment if any CRITICAL or HIGH security finding is open.

## Interview Relevance

CI/CD troubleshooting is tested in platform and senior engineering roles:

| Skill | Why It Matters |
|-------|---------------|
| Pipeline debugging | Can you diagnose build failures? |
| Security scanning | Do you treat security as a pipeline stage? |
| Deployment troubleshooting | Can you fix production deployment issues? |
| Image management | Container registry, pull secrets, tagging |
| Prevention design | Can you prevent recurring failures? |

**Follow-up questions:**
- "What stages should every GenAI pipeline have?"
- "How do you handle flaky tests in CI?"
- "When should you rollback a deployment?"
- "How do you secure the CI/CD pipeline itself?"
