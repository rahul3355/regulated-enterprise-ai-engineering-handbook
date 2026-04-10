# Service Catalog

## Overview

The service catalog is the authoritative registry of all services running on the GenAI platform. It provides discoverability, ownership information, dependency mapping, and operational status for every service. This document covers service catalog design principles, metadata schema, and adoption strategies.

---

## Service Catalog Purpose

### For Engineers

- **Discoverability**: Find existing services before building new ones
- **Onboarding**: Understand the service landscape of the organization
- **Debugging**: Find service owners and dependencies during incidents
- **Impact Analysis**: Understand what will break if I change this service

### For Platform Team

- **Governance**: Track service compliance with platform standards
- **Adoption**: Monitor golden path adoption rates
- **Cost**: Aggregate cost data per service
- **Capacity**: Plan infrastructure based on service requirements

### For Leadership

- **Visibility**: Understand the platform's service landscape
- **Risk**: Identify single points of failure and bus factor
- **Investment**: Understand where engineering effort is spent

---

## Service Catalog Architecture

### Data Model

```yaml
# Service metadata schema
service:
  # Identity
  id: "assistbot"
  name: "AssistBot"
  description: "AI-powered customer support assistant"
  lifecycle: "production"  # experimental, development, staging, production, deprecated

  # Ownership
  owner:
    team: "platform-squad"
    tech_lead: "@alice"
    product_manager: "@bob"
    on_call: "genai-platform-oncall"

  # Classification
  tier: 1  # 1 = critical customer-facing, 2 = important, 3 = internal
  domain: "customer-experience"
  capabilities:
    - "customer-support"
    - "rag-retrieval"
    - "natural-language-understanding"

  # Infrastructure
  infrastructure:
    compute:
      type: kubernetes
      cluster: "genai-prod-aks"
      namespace: "assistbot"
    database:
      - type: postgresql
        instance: "assistbot-db-prod"
    vector_db:
      provider: pinecone
      index: "assistbot-knowledge"
    cache:
      type: redis
      instance: "assistbot-redis"

  # Dependencies
  dependencies:
    - service: "azure-openai-proxy"
      type: llm-api
      criticality: critical
    - service: "customer-data-api"
      type: data-source
      criticality: critical
    - service: "pinecone-prod"
      type: vector-db
      criticality: critical
    - service: "notification-service"
      type: downstream
      criticality: optional

  # Endpoints
  endpoints:
    - name: "chat-api"
      url: "https://api.genai.bank.com/v1/chat"
      type: public
    - name: "admin-api"
      url: "https://assistbot.internal.genai.bank.internal/admin"
      type: internal

  # SLIs and SLOs
  reliability:
    slos:
      - name: "availability"
        target: 99.95
        window: "30d"
      - name: "latency_p95"
        target: "500ms"
        window: "30d"
      - name: "error_rate"
        target: "0.1%"
        window: "30d"
    current_performance:
      availability: 99.97
      latency_p95: "420ms"
      error_rate: "0.05%"

  # Deployment
  deployment:
    repository: "github.com/bank/genai-assistbot"
    ci_cd: "github-actions"
    deployment_tool: "argocd"
    last_deployment:
      version: "2.5.1"
      deployed_at: "2024-03-15T10:30:00Z"
      deployed_by: "@alice"
      status: "successful"

  # Cost
  cost:
    monthly_budget: £18,000
    monthly_actual: £16,500
    cost_center: "genai-platform"
    cost_breakdown:
      llm_api: £8,200
      gpu_compute: £5,100
      application_compute: £1,800
      database: £800
      vector_db: £500
      other: £100

  # Compliance
  compliance:
    data_classification: "confidential"
    regulatory_scope:
      - "FCA"
      - "GDPR"
    audit_logging: true
    data_residency: "UK"
    last_security_review: "2024-02-01"
    next_security_review: "2024-08-01"

  # Documentation
  documentation:
    runbook: "https://wiki.bank.com/runbooks/assistbot"
    api_docs: "https://api.genai.bank.com/docs"
    architecture: "https://wiki.bank.com/architecture/assistbot"
    postmortems:
      - title: "RAG data exfiltration incident"
        date: "2023-11-15"
        url: "https://wiki.bank.com/postmortems/2023-11-15-assistbot"
      - title: "Vector DB outage"
        date: "2024-01-20"
        url: "https://wiki.bank.com/postmortems/2024-01-20-vectordb"

  # Links
  links:
    dashboard: "https://grafana.bank.com/d/assistbot"
    logs: "https://kibana.bank.com/app/discover#/assistbot"
    traces: "https://jaeger.bank.com/search?service=assistbot"
    alerts: "https://pagerduty.com/services/assistbot"
    repository: "https://github.com/bank/genai-assistbot"
```

---

## Service Catalog Implementation

### Backstage Service Catalog

```yaml
# Backstage catalog-info.yaml for AssistBot
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: assistbot
  description: AI-powered customer support assistant
  annotations:
    github.com/project-slug: bank/genai-assistbot
    backstage.io/techdocs-ref: dir:.
    grafana/dashboard-selector: tags @= 'assistbot'
    pagerduty.com/service-id: P123456
  tags:
    - genai
    - customer-facing
    - rag
    - tier-1
  links:
    - url: https://api.genai.bank.com/v1/chat
      title: API Endpoint
    - url: https://wiki.bank.com/runbooks/assistbot
      title: Runbook
      icon: book
spec:
  type: service
  lifecycle: production
  owner: platform-squad
  dependsOn:
    - resource:azure-openai-proxy
    - resource:pinecone-prod
    - resource:customer-data-api
  system: genai-platform
  providesApis:
    - assistbot-chat-api

---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: assistbot-chat-api
  description: Chat API for customer support
spec:
  type: openapi
  lifecycle: production
  owner: platform-squad
  definition:
    $text: https://api.genai.bank.com/openapi/assistbot.yaml
```

---

## Service Discoverability

### Search and Filter

Engineers should be able to find services by:

| Search Criteria | Example |
|----------------|---------|
| **Name** | "assistbot" |
| **Team** | "platform-squad" |
| **Domain** | "customer-experience" |
| **Tier** | "tier-1" |
| **Lifecycle** | "production" |
| **Dependency** | "depends on pinecone" |
| **Technology** | "uses FastAPI" |
| **Tag** | "genai", "rag" |

### Service Graph

Visualize service dependencies:

```
┌───────────────┐     ┌───────────────┐
│  Web Frontend │────►│  AssistBot    │
└───────────────┘     │               │
                      ├───────┬───────┤
┌───────────────┐     │       │       │
│Mobile App     │────►│       │       │
└───────────────┘     │       │       │
                      ▼       ▼       ▼
              ┌──────────┐ ┌─────┐ ┌──────┐
              │OpenAI    │ │Pine-│ │Post- │
              │Proxy     │ │cone │ │greSQL│
              └──────────┘ └─────┘ └──────┘
```

### Dependency Impact Analysis

When planning a change to a service, show:

- **Upstream services**: Services that depend on this service
- **Downstream services**: Services this service depends on
- **Blast radius**: What would break if this service goes down?

---

## Service Tiering

### Tier Definitions

| Tier | Description | Requirements | Examples |
|------|-------------|-------------|---------|
| **Tier 1** | Critical customer-facing service | 99.95% SLO, 24/7 on-call, DR plan, quarterly chaos testing | AssistBot, WealthAdvisor AI |
| **Tier 2** | Important service | 99.9% SLO, business hours on-call, backup plan | ComplianceBot, Embedding Pipeline |
| **Tier 3** | Internal/supporting service | 99.5% SLO, best-effort support | Internal tools, dashboards |

### Tier Requirements

| Requirement | Tier 1 | Tier 2 | Tier 3 |
|------------|--------|--------|-------|
| **SLO** | 99.95% | 99.9% | 99.5% |
| **On-call** | 24/7 | Business hours | Best effort |
| **Multi-AZ** | Required | Required | Recommended |
| **DR plan** | Required | Recommended | Optional |
| **Chaos testing** | Quarterly | Semi-annually | Optional |
| **Security review** | Annually | Annually | As needed |
| **Load testing** | Before launch, quarterly | Before launch | Before launch |
| **Capacity planning** | Monthly | Quarterly | As needed |

---

## Service Lifecycle Management

### Service States

| State | Description | Actions |
|-------|-------------|---------|
| **Proposed** | Service is being planned | RFC, architecture review |
| **Development** | Service is being built | CI/CD pipeline active, staging deployment |
| **Staging** | Service is being tested | UAT, load testing, security review |
| **Production** | Service is live | Monitoring, on-call, SLO tracking |
| **Deprecated** | Service is being retired | Migration plan, sunset date |
| **Decommissioned** | Service is shut down | Resources freed, data archived |

### Service Creation Workflow

```
1. RFC submitted (why build this? what does it do?)
    │
2. Architecture review (is this the right approach?)
    │
3. Choose golden path template (or justify custom)
    │
4. Scaffold service from template
    │
5. Build and test (CI/CD pipeline active)
    │
6. Deploy to staging
    │
7. UAT and load testing
    │
8. Security review
    │
9. Deploy to production
    │
10. Register in service catalog
    │
11. On-call rotation added
    │
12. Monitoring and alerting active
```

### Service Decommissioning Workflow

```
1. Decision to decommission (product/tech review)
    │
2. Mark as "deprecated" in service catalog
    │
3. Notify dependent services
    │
4. Set sunset date (minimum 30 days for Tier 1/2)
    │
5. Migration plan for dependent services
    │
6. Execute migration
    │
7. Shut down service
    │
8. Archive data (per retention policy)
    │
9. Free resources
    │
10. Mark as "decommissioned" in catalog
```

---

## Service Catalog Governance

### Compliance Tracking

The service catalog tracks compliance requirements:

| Check | How It Is Enforced |
|-------|-------------------|
| **All production services have an owner** | Catalog validation (cannot deploy without owner) |
| **All Tier 1 services have 24/7 on-call** | PagerDuty integration check |
| **All services have audit logging** | Platform policy enforcement |
| **All services are tagged for cost** | Cloud provider policy enforcement |
| **All services have health checks** | Deployment gate (cannot deploy without health checks) |
| **Security review is current** | Calendar-based alerting to service owner |

### Catalog Health Checks

Regular checks on catalog data quality:

| Check | Frequency | Action |
|-------|-----------|--------|
| **Missing owner** | Weekly | Page team lead to assign owner |
| **Stale documentation** | Monthly | Notify owner to update |
| **Outdated dependencies** | Weekly | Notify owner to update dependency list |
| **Missing SLOs** | Monthly | Require SLO definition for production services |
| **Incorrect tier** | Quarterly | Review tier assignments |

---

## Service Catalog Adoption

### Driving Adoption

1. **Make it the source of truth**: All operational information must be in the catalog
2. **Integrate with workflows**: Deployment, on-call, and incident response workflows use catalog data
3. **Make it easy**: Auto-populate as much data as possible from infrastructure
4. **Show value**: Demonstrate how the catalog helps engineers (discovery, debugging, impact analysis)

### Adoption Metrics

| Metric | Target | Current |
|--------|--------|---------|
| **Services registered** | 100% of production services | 85% |
| **Complete metadata** | > 90% of fields populated | 78% |
| **Up-to-date dependencies** | > 95% accuracy | 88% |
| **Catalog searches per week** | > 10 per engineer | 6 per engineer |

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [platform-engineering.md](platform-engineering.md) -- Platform engineering principles
- [golden-paths.md](golden-paths.md) -- Golden path design
- [../engineering-culture/service-ownership.md](../engineering-culture/service-ownership.md) -- Service ownership principles
- [../incident-management/README.md](../incident-management/README.md) -- Incident management
