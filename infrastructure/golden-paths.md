# Golden Paths

## Overview

Golden paths are opinionated, well-documented, and fully supported ways of building and deploying services on the platform. They provide the fastest route from idea to production by eliminating boilerplate work and ensuring best practices are followed by default.

---

## Golden Path Philosophy

### What Is a Golden Path?

A golden path is not a framework. It is a combination of:

1. **Templates**: Pre-built project scaffolding with sensible defaults
2. **Pipelines**: Standard CI/CD workflows that work out of the box
3. **Configuration**: Infrastructure-as-code with pre-configured resources
4. **Documentation**: Clear guides for common tasks
5. **Support**: Platform team available to help and improve the path

### Why Golden Paths?

| Without Golden Paths | With Golden Paths |
|---------------------|-------------------|
| Every team builds their own tooling | Shared, maintained tooling |
| Inconsistent practices across teams | Consistent practices by default |
| Security and compliance are manual | Security and compliance are automated |
| New engineers take weeks to onboard | New engineers deploy on day one |
| Platform team supports many patterns | Platform team supports one path + escape hatches |

### Golden Path vs. Paved Road vs. Template

| Term | Definition | Scope |
|------|-----------|-------|
| **Template** | Project scaffolding | Code structure only |
| **Paved Road** | Supported pattern with documentation | Template + pipeline + docs |
| **Golden Path** | The fastest, most supported way to ship | Paved road + automation + platform integration |

---

## Available Golden Paths

### 1. GenAI API Service

**Use Case**: Build a customer-facing API service that uses LLMs.

**What You Get:**
- FastAPI application with authentication
- LLM routing (multiple provider support)
- Prompt management and versioning
- Request/response audit logging
- PII detection and filtering
- Rate limiting and cost tracking
- Health checks, monitoring, alerting

**Components:**
```
genai-api-service/
├── src/
│   ├── main.py              # FastAPI application
│   ├── routes/              # API endpoints
│   ├── services/            # Business logic
│   │   ├── llm_router.py    # LLM provider routing
│   │   ├── prompt_manager.py # Prompt versioning
│   │   └── audit_logger.py  # Audit logging
│   ├── middleware/           # Auth, rate limiting, PII detection
│   └── models/              # Pydantic models
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── infrastructure/
│   ├── main.tf              # Terraform (EKS, RDS, etc.)
│   ├── variables.tf
│   └── kubernetes/          # K8s manifests
├── .github/workflows/
│   ├── ci.yml               # Build, test, scan
│   └── cd.yml               # Deploy to staging/production
├── monitoring/
│   ├── dashboards/          # Grafana
│   └── alerts/              # PagerDuty
├── Dockerfile
├── platform.yaml            # Platform metadata
└── docs/
    ├── runbook.md
    └── api-spec.yaml
```

### 2. RAG Service

**Use Case**: Build a retrieval-augmented generation service.

**What You Get:**
- Document ingestion pipeline
- Embedding generation (batch and real-time)
- Vector database integration
- Retrieval with metadata filtering
- RAG prompt construction
- Generation with LLM
- Citation and grounding

**Components:**
```
genai-rag-service/
├── src/
│   ├── ingestion/           # Document processing
│   ├── embedding/           # Embedding generation
│   ├── retrieval/           # Vector search
│   ├── generation/          # LLM generation
│   └── evaluation/          # RAG quality metrics
├── infrastructure/
│   ├── vector-db.tf         # Pinecone/Milvus
│   ├── embedding-pipeline.tf
│   └── ...
├── pipelines/
│   ├── batch-embedding.py   # Batch processing
│   └── incremental-sync.py  # Incremental updates
└── ...
```

### 3. Model Training Pipeline

**Use Case**: Fine-tune models on banking-specific data.

**What You Get:**
- Training job orchestration
- GPU cluster provisioning
- Experiment tracking (MLflow)
- Model registry
- Evaluation pipeline
- Deployment to inference

**Components:**
```
genai-training-pipeline/
├── src/
│   ├── data/                # Data preparation
│   ├── training/            # Training scripts
│   ├── evaluation/          # Model evaluation
│   └── deployment/          # Model deployment
├── infrastructure/
│   ├── gpu-cluster.tf       # GPU provisioning
│   ├── storage.tf           # Training data storage
│   └── mlflow.tf            # Experiment tracking
├── configs/
│   ├── experiments/          # Experiment configurations
│   └── models/              # Model configurations
└── ...
```

### 4. Embedding Pipeline

**Use Case**: Process and embed documents at scale.

**What You Get:**
- Document chunking strategies
- Embedding generation (batch/streaming)
- Vector database indexing
- Incremental update handling
- Quality validation

### 5. Monitoring Service

**Use Case**: Monitor GenAI service quality and costs.

**What You Get:**
- Model quality tracking
- Drift detection
- Cost monitoring and alerting
- User feedback collection
- A/B testing framework

---

## Golden Path Design Principles

### 1. Zero Configuration to Start

Engineers should be able to create, build, and deploy a service without any configuration:

```bash
# Create a new service
platform create service --template genai-api --name my-service

# Deploy to staging
git push origin main  # CI/CD handles everything

# Deploy to production
platform deploy my-service --environment prod  # With approval gate
```

### 2. Sensible Defaults

All configuration has defaults that work for 80% of cases:

```yaml
# platform.yaml defaults
defaults:
  replicas: 3
  min_replicas: 2
  max_replicas: 10
  cpu_request: 500m
  memory_request: 1Gi
  health_check_interval: 10s
  log_level: info
  environment: staging
```

### 3. Easy to Customize

When defaults are not enough, customization should be straightforward:

```yaml
# Override defaults
overrides:
  replicas: 6
  max_replicas: 30
  gpu:
    type: A100
    count: 1
  environment:
    - name: CUSTOM_ENV_VAR
      value: "custom-value"
```

### 4. Escape Hatches

Golden paths are opinionated but not restrictive. Engineers can deviate when needed:

- **Bring Your Own Infrastructure**: Use custom Terraform modules instead of platform defaults
- **Custom CI/CD**: Replace standard pipelines with custom workflows
- **Custom Monitoring**: Add custom metrics and dashboards alongside platform defaults

---

## Golden Path Adoption Strategy

### Phase 1: Build Trust

- Ensure golden paths are the fastest way to ship
- Provide excellent documentation and support
- Fix bugs and improve based on user feedback

### Phase 2: Drive Adoption

- Make golden paths the default for new services
- Migrate existing services incrementally
- Show success metrics (deployment speed, reliability)

### Phase 3: Maintain Excellence

- Regular template updates with security patches and improvements
- Keep templates in sync with platform capabilities
- Deprecate old patterns with migration guides

### Adoption Metrics

| Metric | Target | Current |
|--------|--------|---------|
| New services using golden paths | > 95% | 88% |
| Existing services migrated | > 70% | 45% |
| Template satisfaction score | > 8/10 | 7.5/10 |
| Average time to first deployment | < 1 day | 1.5 days |

---

## Golden Path Maintenance

### Template Update Process

1. **Platform team owns templates**: Templates are maintained by the platform team, not individual product teams
2. **Automated testing**: Templates are tested regularly (CI on the template repository)
3. **Versioned templates**: Teams can pin to a specific template version
4. **Update notifications**: When templates are updated, users are notified with migration guides

### Template Versioning

```yaml
# Template version history
template: genai-api-service
versions:
  - version: 2.1.0
    date: 2024-03-15
    changes:
      - "Added PII detection middleware"
      - "Updated to Python 3.12"
      - "Fixed Dockerfile caching"
    breaking: false

  - version: 2.0.0
    date: 2024-01-15
    changes:
      - "Migrated from Flask to FastAPI"
      - "New authentication middleware"
    breaking: true
```

### Migration Guides

For each breaking template update, provide a migration guide:

```markdown
# Migration Guide: genai-api-service v1.x → v2.0

## Breaking Changes

### Flask → FastAPI
- Update your route definitions from Flask decorators to FastAPI routers
- Replace `request.json()` with `await request.json()`
- Update response models to use Pydantic

## Steps

1. Update platform.yaml template version:
   ```yaml
   template: genai-api-service@2.0.0
   ```

2. Run the migration script:
   ```bash
   platform migrate my-service --from 1.5.0 --to 2.0.0
   ```

3. Review the changes:
   ```bash
   git diff
   ```

4. Test locally:
   ```bash
   platform run my-service --local
   ```

5. Deploy to staging:
   ```bash
   platform deploy my-service --environment staging
   ```
```

---

## Golden Path Examples

### Example: Creating a RAG Service in 5 Minutes

```bash
# Step 1: Scaffold the service (30 seconds)
platform create service \
  --template genai-rag-template \
  --name knowledge-assistant \
  --team data-squad

# Step 2: Configure the service (2 minutes)
# Edit platform.yaml
cat > platform.yaml << EOF
service:
  name: knowledge-assistant
  team: data-squad
  tier: 2

infrastructure:
  vector-db:
    provider: pinecone
    index: knowledge-base
    dimension: 1536
  embedding:
    model: text-embedding-ada-002
    provider: azure-openai
  llm:
    model: gpt-4-turbo
    provider: azure-openai
EOF

# Step 3: Add your business logic (varies)
# Edit src/retrieval/custom_logic.py

# Step 4: Deploy to staging (1 minute)
git add .
git commit -m "Initial knowledge-assistant service"
git push origin main
# CI/CD: build → test → deploy to staging (automated, ~5 minutes)

# Step 5: Deploy to production (after verification)
platform deploy knowledge-assistant --environment prod
# Approval gate → deploy to production
```

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [platform-engineering.md](platform-engineering.md) -- Platform engineering principles
- [service-catalog.md](service-catalog.md) -- Service catalog design
- [../cicd-devops/README.md](../cicd-devops/README.md) -- CI/CD pipelines
