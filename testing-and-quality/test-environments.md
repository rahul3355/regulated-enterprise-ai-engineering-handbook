# Test Environments in Banking GenAI Systems

## Overview

Test environments are the infrastructure on which tests execute. In banking GenAI systems, the complexity comes from:

- **GPU availability**: LLM inference requires GPUs that may be scarce in test environments
- **External dependencies**: LLM providers, payment networks, credit bureaus
- **Data volume**: Vector databases need production-scale data to test accurately
- **Environment parity**: Differences between dev, test, staging, and production cause "works on my machine" bugs
- **Ephemeral environments**: PR-level environments must spin up quickly and tear down cleanly

---

## Environment Strategy

```mermaid
graph TD
    A[Local Dev<br/>Docker Compose] --> B[CI Pipeline<br/>Ephemeral]
    B --> C[Integration Env<br/>Shared services]
    C --> D[Staging<br/>Production mirror]
    D --> E[Production<br/>Live traffic]

    style A fill:#E8F5E9
    style B fill:#E3F2FD
    style C fill:#FFF3E0
    style D fill:#FCE4EC
    style E fill:#FFEBEE
```

---

## Ephemeral Environments

### Per-PR Environment with Helm

```yaml
# .github/workflows/ephemeral-env.yaml
name: Ephemeral Environment
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-ephemeral:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create ephemeral namespace
        run: |
          NAMESPACE="pr-${{ github.event.pull_request.number }}-${{ github.sha }}"
          kubectl create namespace "$NAMESPACE"
          kubectl label namespace "$NAMESPACE" pr=${{ github.event.pull_request.number }}

      - name: Deploy with Helm
        run: |
          helm install banking-genai ./charts/banking-genai \
            --namespace "pr-${{ github.event.pull_request.number }}" \
            --set image.tag="${{ github.sha }}" \
            --set resources.requests.gpu="0" \
            --set resources.limits.gpu="0" \
            --set embedding.model="all-MiniLM-L6-v2" \
            --set llm.provider="mock" \
            --set environment="ephemeral"

      - name: Run smoke tests
        run: |
          ENDPOINT=$(kubectl get ingress -n "pr-${{ github.event.pull_request.number }}" -o jsonpath='{.items[0].spec.rules[0].host}')
          curl -f "https://$ENDPOINT/health"
          pytest tests/smoke/ --base-url "https://$ENDPOINT"

      - name: Post environment URL
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const endpoint = process.env.ENDPOINT;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Ephemeral environment deployed: https://${endpoint}\nHealth: ${{ job.status }}`
            });

      - name: Cleanup on PR close
        if: always()
        run: |
          helm uninstall banking-genai -n "pr-${{ github.event.pull_request.number }}" || true
          kubectl delete namespace "pr-${{ github.event.pull_request.number }}" || true
```

### Docker Compose for Local Development

```yaml
# docker-compose.local.yaml
version: '3.8'
services:
  banking-rag-api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - LLM_PROVIDER=mock
      - EMBEDDING_MODEL=all-MiniLM-L6-v2
      - VECTOR_DB_URL=http://qdrant:6333
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/banking_test
      - REDIS_URL=redis://redis:6379/0
      - ENVIRONMENT=local
      - DEBUG=true
    depends_on:
      - qdrant
      - postgres
      - redis
    volumes:
      - ./app:/app
      - ./config:/config

  qdrant:
    image: qdrant/qdrant:v1.7.0
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage

  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=banking_test
    volumes:
      - ./test_data/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mock-llm:
    build: ./test_tools/mock-llm
    ports:
      - "8081:8081"
    environment:
      - MOCK_RESPONSES=./test_data/mock_llm_responses.json
    volumes:
      - ./test_data:/test_data

  mock-payment-network:
    build: ./test_tools/mock-payment-network
    ports:
      - "8082:8082"
    environment:
      - MOCK_SCENARIOS=./test_data/payment_scenarios.json

  otel-collector:
    image: otel/opentelemetry-collector:0.96.0
    ports:
      - "4317:4317"
      - "55679:55679"
    volumes:
      - ./config/otel-collector-config.yaml:/etc/otel/config.yaml

volumes:
  qdrant_data:
```

---

## Environment Parity

The goal is to minimize differences between environments while managing costs.

```python
# scripts/env-parity-check.py
"""
Compare environment configurations to detect parity drift.
Run in CI to ensure staging and production stay aligned.
"""
import yaml
import json
import sys

def load_env_config(env: str) -> dict:
    """Load environment configuration from the config repository."""
    with open(f"config/environments/{env}.yaml") as f:
        return yaml.safe_load(f)

def compare_configs(env1: str, env2: str) -> list:
    """Compare two environment configurations and report differences."""
    config1 = load_env_config(env1)
    config2 = load_env_config(env2)

    differences = []

    # Compare service versions
    for service in set(list(config1.get("services", {}).keys()) + list(config2.get("services", {}).keys())):
        v1 = config1.get("services", {}).get(service, {}).get("image", "")
        v2 = config2.get("services", {}).get(service, {}).get("image", "")
        if v1 != v2:
            differences.append({
                "service": service,
                "env1_version": v1,
                "env2_version": v2,
                "severity": "high" if "production" in [env1, env2] else "medium",
            })

    # Compare resource allocations (scaled)
    for service in config1.get("services", {}):
        r1 = config1["services"][service].get("resources", {})
        r2 = config2["services"][service].get("resources", {})
        if r1 and r2:
            # Staging should have at least 50% of production resources
            for resource_type in ["cpu", "memory", "gpu"]:
                val1 = r1.get("requests", {}).get(resource_type, 0)
                val2 = r2.get("requests", {}).get(resource_type, 0)
                if val1 and val2 and val1 < val2 * 0.5:
                    differences.append({
                        "service": service,
                        "resource": resource_type,
                        "staging": val1,
                        "production": val2,
                        "severity": "warning",
                        "message": f"Staging has <50% of production {resource_type} for {service}",
                    })

    return differences

# Run comparison
diffs = compare_configs("staging", "production")
if diffs:
    print(f"Found {len(diffs)} configuration differences between staging and production:")
    for d in diffs:
        print(f"  [{d['severity'].upper()}] {json.dumps(d)}")
    # Fail CI if high-severity differences exist
    high_severity = [d for d in diffs if d["severity"] == "high"]
    if high_severity:
        print("\nFailing: High-severity parity differences detected")
        sys.exit(1)
else:
    print("Environment parity check passed")
```

### Environment Configuration Template

```yaml
# config/environments/staging.yaml
environment: staging
replicas:
  banking-rag-api: 3
  embedding-service: 2
  vector-db: 3
  redis: 2
resources:
  banking-rag-api:
    cpu: "2"
    memory: "4Gi"
    gpu: "1"
  embedding-service:
    cpu: "4"
    memory: "8Gi"
    gpu: "1"
  vector-db:
    cpu: "4"
    memory: "16Gi"
llm:
  provider: openai
  model: gpt-4-turbo
  max_tokens: 2048
  fallback_provider: anthropic
  fallback_model: claude-3-sonnet
vector_db:
  url: "https://qdrant-staging.banking-genai.internal:6333"
  collection: "banking-policies-v2"
  top_k: 5
external_services:
  payment_network: sandbox
  credit_bureau: test
  fraud_detection: staging
data:
  seed_from_production: false
  synthetic_data: true
  data_volume_scale: 0.1  # 10% of production
```

---

## Shared Services for Integration Testing

Some services are expensive or slow to spin up per-PR. These run as shared services.

```yaml
# config/shared-services.yaml
shared_services:
  # Vector database with pre-loaded test indices
  qdrant:
    url: "https://qdrant-shared.banking-genai.internal:6333"
    collections:
      - name: "banking-policies-test"
        vector_size: 384
        distance: "cosine"
      - name: "product-catalog-test"
        vector_size: 768
        distance: "cosine"
    isolation: "namespace"  # Each PR gets its own namespace within shared Qdrant

  # Mock LLM provider (deterministic responses)
  mock-llm:
    url: "https://mock-llm.banking-genai.internal:8081"
    scenarios:
      - name: "standard_response"
        latency_ms: 800
      - name: "slow_response"
        latency_ms: 5000
      - name: "error_response"
        status: 500
      - name: "rate_limited"
        status: 429

  # Mock payment network
  mock-payment-network:
    url: "https://mock-payments.banking-genai.internal:8082"
    supported_operations:
      - balance_inquiry
      - transaction_history
      - fund_transfer
```

---

## GPU Resource Management

```yaml
# kubernetes/gpu-pool.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banking-genai-inference
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: inference
          image: banking-genai-inference:latest
          resources:
            requests:
              nvidia.com/gpu: 1
              cpu: "4"
              memory: "16Gi"
            limits:
              nvidia.com/gpu: 1
              cpu: "8"
              memory: "32Gi"
          env:
            - name: NVIDIA_VISIBLE_DEVICES
              value: all
            - name: CUDA_VISIBLE_DEVICES
              value: "0"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        gpu-type: a100-80gb

---
# GPU quota management for test environments
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: banking-genai-test
spec:
  hard:
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"
```

---

## Environment Teardown and Cost Control

```python
# scripts/cleanup-ephemeral-envs.py
"""
Clean up ephemeral environments that are older than 48 hours.
Prevents cost creep from abandoned PR environments.
"""
import subprocess
import json
from datetime import datetime, timedelta

def get_all_namespaces():
    """Get all Kubernetes namespaces with PR labels."""
    result = subprocess.run(
        ["kubectl", "get", "namespaces", "-l", "pr", "-o", "json"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)

def namespace_age(namespace: dict) -> timedelta:
    """Calculate the age of a namespace."""
    creation = namespace["metadata"]["creationTimestamp"]
    creation_dt = datetime.fromisoformat(creation.replace("Z", "+00:00"))
    return datetime.now(creation_dt.tzinfo) - creation_dt

def cleanup_old_namespaces(max_age_hours: int = 48):
    """Delete namespaces older than max_age_hours."""
    namespaces = get_all_namespaces()
    deleted = 0

    for ns in namespaces.get("items", []):
        age = namespace_age(ns)
        if age > timedelta(hours=max_age_hours):
            name = ns["metadata"]["name"]
            print(f"Deleting namespace: {name} (age: {age})")
            subprocess.run(["kubectl", "delete", "namespace", name], check=False)
            deleted += 1

    print(f"Cleaned up {deleted} namespaces older than {max_age_hours} hours")

if __name__ == "__main__":
    cleanup_old_namespaces()
```

---

## Interview Questions

1. **How do you handle GPU constraints in ephemeral test environments?**
   - Use mock LLM providers for PR-level tests. Reserve real GPU for staging and nightly performance tests. Use smaller embedding models (all-MiniLM-L6-v2) in test vs. production (text-embedding-3-large).

2. **What is environment parity and why is it important?**
   - Environment parity means test environments mirror production in architecture, configuration ratios, and data characteristics. Without parity, tests pass in staging but fail in production due to differences in scale, network topology, or configuration.

3. **How do you manage database state across parallel test runs?**
   - Each test run gets an isolated database (separate schema or container). Use Flyway/Liquibase for migrations. Seed with deterministic data using fixed seeds. Clean up after test completion.

4. **What is your strategy for testing against external LLM providers in CI?**
   - Use mocks/deterministic stubs in CI. Run integration tests against real providers nightly in staging. Use provider sandbox environments for contract validation. Never hit production LLM endpoints from CI.

---

## Cross-References

- See [kubernetes-openshift/](../kubernetes-openshift/) for Kubernetes deployment patterns
- See [cicd-devops/](../cicd-devops/) for CI/CD pipeline configuration
- See [load-testing.md](./load-testing.md) for environment requirements for load testing
- See [architecture/multi-tenant-design.md](../architecture/multi-tenant-design.md) for namespace isolation
