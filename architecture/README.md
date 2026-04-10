# Architecture Practice and ADR Process

## Architecture Philosophy

Architecture in a banking GenAI organization is not about creating perfect diagrams. It is about making decisions that balance competing concerns -- reliability vs. speed, innovation vs. compliance, simplicity vs. capability -- and documenting those decisions so the organization can learn from them.

### Core Principles

1. **Decisions over diagrams**: Architecture is a series of decisions. Document the decisions. Diagrams support understanding, but the decision is the artifact.

2. **Context-driven**: There are no universally right architecture decisions. The right decision depends on your constraints: team size, regulatory environment, budget, timeline, and risk tolerance.

3. **Reversible vs. irreversible**: Classify decisions by how hard they are to reverse. Irreversible decisions (technology selection, data model) deserve more rigor than reversible ones (configuration, naming).

4. **Evolutionary over big-design-up-front**: Architecture evolves with the system. Start simple, add complexity when forced to, and document each step.

5. **Compliance is a first-class concern**: In banking, regulatory compliance is not a constraint to work around. It is a design requirement from day one.

## Architecture Decision Records (ADRs)

ADRs are the primary mechanism for documenting architectural decisions. See [Architecture Decision Records](architecture-decision-records.md) for the format and process.

### ADR Workflow

```
1. Identify a decision that needs documenting
   -> Is it cross-team? Hard to reverse? Compliance-relevant?
   -> If yes, write an ADR.

2. Draft the ADR
   -> Describe context, decision, consequences, alternatives

3. Review with stakeholders
   -> Engineering team, security, compliance, operations

4. Merge to architecture/decisions/ directory
   -> Status: Proposed

5. Implement the decision
   -> Status: Accepted

6. Review at scheduled date
   -> Is the decision still valid? Supersede or deprecate if needed.
```

### ADR Review Cadence

```
┌─────────────────────────────────────────────────────────────┐
│  ADR REVIEW SCHEDULE                                        │
├─────────────────────┬───────────────────────────────────────┤
│  Frequency          │  Action                               │
├─────────────────────┼───────────────────────────────────────┤
│  Monthly            │  Review ADRs in "Proposed" status     │
│  Quarterly          │  Review ADRs approaching review date  │
│  Per-release        │  Check if any ADRs are superseded     │
│  Annually           │  Full ADR log audit                   │
└─────────────────────┴───────────────────────────────────────┘
```

## Architecture Review Process

### When Architecture Review Is Required

New services or significant changes require architecture review when:
- A new service is being created
- An existing service is undergoing significant redesign
- A new technology or framework is being adopted
- Data flow between services is changing
- Security or compliance posture is affected

### Review Committee

```
┌─────────────────────────────────────────────────────────────┐
│  ARCHITECTURE REVIEW BOARD                                  │
│                                                             │
│  Core Members:                                              │
│  - Principal Engineer (chair)                               │
│  - Staff Engineer from affected domain                      │
│  - Security Engineer                                        │
│  - SRE / Platform Engineer                                  │
│                                                             │
│  Ad-hoc Members:                                            │
│  - Compliance Officer (if regulatory impact)                │
│  - Data Engineer (if data pipeline changes)                 │
│  - ML Engineer (if model/ML infrastructure changes)         │
│  - Frontend Engineer (if API contract changes)              │
│                                                             │
│  Meeting Cadence: Weekly, 60 minutes                        │
│  Decision Method: Consensus, with chair as tiebreaker       │
└─────────────────────────────────────────────────────────────┘
```

### Review Checklist

```
┌─────────────────────────────────────────────────────────────┐
│  ARCHITECTURE REVIEW CHECKLIST                               │
├─────────────────────────────────────────────────────────────┤
│  System Design:                                             │
│  [ ] Service boundaries are well-defined                    │
│  [ ] API contracts are documented                           │
│  [ ] Data flow is clear and documented                      │
│  [ ] Failure modes are identified and handled               │
│  [ ] Scalability considerations addressed                   │
│                                                             │
│  Security:                                                  │
│  [ ] Authentication and authorization design documented     │
│  [ ] Data encryption (in transit and at rest)               │
│  [ ] PII handling and redaction                             │
│  [ ] Threat model completed                                 │
│                                                             │
│  Compliance:                                                │
│  [ ] Regulatory requirements identified                     │
│  [ ] Audit trail design documented                          │
│  [ ] Data retention policy defined                          │
│  [ ] Model governance considerations (for AI components)    │
│                                                             │
│  Operations:                                                │
│  [ ] Observability design (logs, metrics, traces)           │
│  [ ] Deployment strategy defined                            │
│  [ ] Rollback plan documented                               │
│  [ ] Disaster recovery considerations                       │
│                                                             │
│  Cost:                                                      │
│  [ ] Infrastructure cost estimated                          │
│  [ ] LLM/API cost estimated (for GenAI components)          │
│  [ ] Operational cost (on-call, maintenance) considered     │
│                                                             │
│  Documentation:                                             │
│  [ ] ADR written for significant decisions                  │
│  [ ] README updated with new service documentation          │
│  [ ] Runbook created for operational procedures             │
└─────────────────────────────────────────────────────────────┘
```

## Architecture Principles for Banking GenAI

### 1. Compliance by Design

Architecture decisions should make compliance the default, not the exception.

```
GOOD: Audit logging is built into the platform. Every service inherits it.
BAD: Each service implements its own audit logging. Some forget.
```

### 2. Observability as a Platform Feature

Every service running on the platform automatically emits logs, metrics, and traces.

```
GOOD: The platform SDK configures OpenTelemetry automatically.
BAD: Engineers manually add logging and often forget.
```

### 3. Multi-Model Abstraction

Do not tie architecture to a single LLM provider. Abstract the model layer.

```
GOOD: Services call an internal LLM Gateway that routes to any provider.
BAD: Services directly call OpenAI API. Migration requires code changes in 12 services.
```

### 4. Data Minimization

Collect and retain only the data you need. Banking regulations impose strict requirements on data handling.

```
GOOD: Prompt hashes are stored, not full prompts. PII is redacted before logging.
BAD: Full prompts and responses are logged indefinitely.
```

### 5. Graceful Degradation

When dependencies fail, the system should degrade gracefully, not cascade.

```
GOOD: LLM provider failure triggers fallback to cached responses or simpler model.
BAD: LLM provider failure makes the entire advisory feature unavailable.
```

## Architecture Documentation Structure

```
architecture/
├── README.md                     # This file
├── architecture-decision-records.md  # ADR process
├── decisions/                    # Individual ADRs
│   ├── 001-use-python-for-backend.md
│   ├── 002-use-fastapi-over-django.md
│   └── ...
├── diagrams/                     # Architecture diagrams
│   ├── platform-overview.png
│   ├── data-flow.png
│   └── deployment.png
├── standards/                    # Architecture standards
│   ├── api-design-standards.md
│   ├── data-modeling-standards.md
│   └── service-design-standards.md
└── reviews/                      # Architecture review records
    ├── 2025-Q1-mortgage-advisor-review.md
    └── 2025-Q1-llm-gateway-review.md
```

## GenAI Architecture Reference Patterns

### Pattern 1: LLM Gateway

```
Client ──> API Gateway ──> LLM Gateway ──> [OpenAI | Anthropic | Custom Model]
                              │
                              ├──> Prompt Template Service
                              ├──> Response Validator
                              ├──> Token Cost Tracker
                              └──> Cache Layer
```

The LLM Gateway centralizes model access, enforces rate limits, tracks costs, and provides a consistent interface regardless of the underlying provider.

### Pattern 2: RAG Pipeline

```
Query ──> Embedding ──> Vector DB ──> Reranker ──> Context Assembly ──> LLM
```

Each stage is independently scalable and observable. The pipeline supports multiple embedding models and vector stores.

### Pattern 3: Guardrails

```
User Input ──> Input Guardrails ──> LLM ──> Output Guardrails ──> Response
                   │                                   │
                   ├── PII detection                   ├── Factual check
                   ├── Injection defense               ├── Compliance check
                   └── Toxicity filter                 └── Hallucination check
```

Input and output guardrails ensure safety at both ends of the LLM call.
