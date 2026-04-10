# Enterprise GenAI Architecture

This document presents the complete architecture for an enterprise-grade GenAI platform designed for a global bank serving 100M+ customers across 60+ countries.

## System Context

```mermaid
graph TB
    subgraph "External Users"
        RetailCustomers[100M+ Retail Customers]
        BusinessCustomers[5M Business Customers]
        Employees[50K+ Employees]
        Partners[Partner Organizations]
    end

    subgraph "Channels"
        MobileApp[Mobile Banking App]
        WebApp[Online Banking]
        ContactCenter[Contact Center]
        InternalPortal[Employee Portal]
        APIChannels[Partner APIs]
    end

    subgraph "GenAI Platform"
        Gateway[API Gateway + Auth]
        Orchestrator[Request Orchestrator]
        ModelLayer[Model Layer<br/>Multi-provider]
        DataLayer[Data Layer<br/>RAG + Vector DB]
        SafetyLayer[Safety & Guardrails]
        Observability[Observability Stack]
    end

    subgraph "Banking Systems"
        CoreBanking[Core Banking System]
        CRM[CRM System]
        ComplianceSys[Compliance Systems]
        DocumentMgmt[Document Management]
        TransactionSys[Transaction Systems]
    end

    RetailCustomers --> MobileApp
    RetailCustomers --> WebApp
    BusinessCustomers --> WebApp
    Employees --> InternalPortal
    Employees --> ContactCenter
    Partners --> APIChannels

    MobileApp --> Gateway
    WebApp --> Gateway
    ContactCenter --> Gateway
    InternalPortal --> Gateway
    APIChannels --> Gateway

    Gateway --> Orchestrator
    Orchestrator --> ModelLayer
    Orchestrator --> DataLayer
    Orchestrator --> SafetyLayer

    Orchestrator --> CoreBanking
    Orchestrator --> CRM
    Orchestrator --> ComplianceSys
    Orchestrator --> DocumentMgmt
    Orchestrator --> TransactionSys

    ModelLayer --> Observability
    DataLayer --> Observability
    SafetyLayer --> Observability
    Orchestrator --> Observability
```

## Platform Architecture

```mermaid
graph TB
    subgraph "Ingress Layer"
        CDN[CDN / Load Balancer]
        WAF[Web Application Firewall]
        APIGW[API Gateway]
        Auth[Auth Service<br/>OAuth2 / mTLS]
        RateLimit[Rate Limiter]
    end

    subgraph "Application Layer"
        ChatService[Chat Service]
        SummarizationSvc[Summarization Service]
        ClassificationSvc[Classification Service]
        AnalysisService[Analysis Service]
        CodeAssistSvc[Code Assistant Service]
    end

    subgraph "Orchestration Layer"
        PromptMgr[Prompt Manager<br/>Versioning + A/B]
        ModelRouter[Model Router<br/>Intelligent routing]
        Cache[Response Cache<br/>Exact + Semantic]
        ContextMgr[Context Manager<br/>History + Summarization]
        ToolOrchestrator[Tool Orchestrator<br/>Function calling]
    end

    subgraph "Safety Layer"
        InputValidator[Input Validator<br/>Injection detection]
        OutputValidator[Output Validator<br/>PII + Hallucination]
        ContentFilter[Content Filter<br/>Harm categories]
        HumanReview[Human Review Queue<br/>HITL]
    end

    subgraph "Model Layer"
        OpenAI[OpenAI<br/>GPT-4o, GPT-4o-mini]
        Anthropic[Anthropic<br/>Claude 3.5 Sonnet/Haiku]
        Google[Google<br/>Gemini 1.5 Pro/Flash]
        SelfHosted[Self-Hosted<br/>Llama 3, Mistral]
        EmbeddingSvc[Embedding Service<br/>Multi-model]
    end

    subgraph "Data Layer"
        VectorDB[(Vector Database<br/>pgvector / Pinecone)]
        DocStore[(Document Store<br/>S3 / MinIO)]
        PromptDB[(Prompt Repository<br/>Git + DB)]
        CacheDB[(Cache Store<br/>Redis)]
        AuditDB[(Audit Log<br/>Immutable Store)]
    end

    subgraph "Observability Layer"
        Tracing[Distributed Tracing<br/>OpenTelemetry]
        Metrics[Metrics<br/>Prometheus / StatsD]
        Logging[Structured Logging<br/>ELK / Splunk]
        Dashboards[Dashboards<br/>Grafana]
        Alerts[Alerting<br/>PagerDuty]
    end

    CDN --> WAF --> APIGW --> Auth --> RateLimit

    RateLimit --> ChatService
    RateLimit --> SummarizationSvc
    RateLimit --> ClassificationSvc
    RateLimit --> AnalysisService
    RateLimit --> CodeAssistSvc

    ChatService --> PromptMgr
    ChatService --> ModelRouter
    ChatService --> Cache
    ChatService --> ContextMgr
    ChatService --> ToolOrchestrator

    PromptMgr --> InputValidator
    ModelRouter --> OutputValidator
    OutputValidator --> ContentFilter
    ContentFilter --> HumanReview

    InputValidator --> OpenAI
    InputValidator --> Anthropic
    InputValidator --> Google
    InputValidator --> SelfHosted

    ModelRouter --> EmbeddingSvc
    EmbeddingSvc --> VectorDB
    PromptMgr --> PromptDB
    Cache --> CacheDB
    HumanReview --> AuditDB

    OpenAI --> Tracing
    Anthropic --> Tracing
    Google --> Tracing
    SelfHosted --> Tracing

    Tracing --> Metrics --> Dashboards
    Logging --> Dashboards
    Dashboards --> Alerts
```

## Data Flow Architecture

### Request Flow

```mermaid
sequenceDiagram
    participant Client as Client App
    participant Gateway as API Gateway
    participant Orchestrator as Orchestrator
    participant Safety as Safety Layer
    participant Cache as Cache
    participant Router as Model Router
    participant Models as Model Provider
    participant VectorDB as Vector DB
    participant Audit as Audit Log

    Client->>Gateway: POST /api/chat
    Gateway->>Gateway: Authenticate + Rate Limit
    Gateway->>Orchestrator: Forward request

    Orchestrator->>Cache: Check exact cache
    alt Cache Hit
        Cache-->>Orchestrator: Cached response
        Orchestrator->>Client: Return cached response
    else Cache Miss
        Orchestrator->>Safety: Validate input
        Safety-->>Orchestrator: Input OK

        Orchestrator->>VectorDB: Retrieve relevant context
        VectorDB-->>Orchestrator: Top-K documents

        Orchestrator->>Router: Select model
        Router-->>Orchestrator: gpt-4o

        Orchestrator->>Models: Complete(prompt, context)
        Models-->>Orchestrator: Response

        Orchestrator->>Safety: Validate output
        Safety-->>Orchestrator: Output OK

        Orchestrator->>Cache: Store in cache
        Orchestrator->>Audit: Log request/response
        Orchestrator->>Client: Return response
    end
```

### RAG Pipeline

```mermaid
graph LR
    subgraph "Ingestion Pipeline"
        Sources[Data Sources<br/>Policies, Regulations, FAQs] --> Parser[Document Parser]
        Parser --> Chunker[Chunker<br/>Semantic boundaries]
        Chunker --> Cleaner[Cleaner<br/>Remove noise, PII check]
        Cleaner --> Embedder[Embedder]
        Embedder --> VectorDB[(Vector DB)]
    end

    subgraph "Query Pipeline"
        Query[User Query] --> QueryEmbed[Query Embedding]
        QueryEmbed --> HybridSearch[Hybrid Search<br/>Semantic + Keyword]
        HybridSearch --> Reranker[Reranker<br/>Cross-encoder]
        Reranker --> ContextBuilder[Context Builder]
        ContextBuilder --> Prompt[Prompt Assembly]
        Prompt --> LLM[LLM Generation]
        LLM --> Response[Response to User]
    end

    VectorDB --> HybridSearch
```

## Multi-Tenancy Architecture

```mermaid
graph TB
    subgraph "Tenant Isolation"
        TenantA[Tenant: Retail Banking]
        TenantB[Tenant: Compliance]
        TenantC[Tenant: Engineering]
    end

    subgraph "Shared Platform"
        SharedGateway[Shared API Gateway]
        SharedModels[Shared Model Pool]
        SharedCache[Shared Cache]
    end

    subgraph "Tenant-Specific Resources"
        APromptsA[Prompts A]
        APromptsB[Prompts B]
        APromptsC[Prompts C]
        ADataA[Data A<br/>Isolated]
        ADataB[Data B<br/>Isolated]
        ADataC[Data C<br/>Isolated]
        ABudgetA[Budget A]
        ABudgetB[Budget B]
        ABudgetC[Budget C]
    end

    TenantA --> SharedGateway
    TenantB --> SharedGateway
    TenantC --> SharedGateway

    SharedGateway --> TenantA
    SharedGateway --> TenantB
    SharedGateway --> TenantC

    TenantA --> APromptsA
    TenantA --> ADataA
    TenantA --> ABudgetA
    TenantB --> APromptsB
    TenantB --> ADataB
    TenantB --> ABudgetB
    TenantC --> APromptsC
    TenantC --> ADataC
    TenantC --> ABudgetC

    SharedGateway --> SharedModels
    SharedGateway --> SharedCache
```

### Tenant Isolation Model

| Resource | Shared | Isolated | Rationale |
|----------|--------|----------|-----------|
| Model APIs | Yes | No | Multi-tenant by design |
| Prompts | No | Yes | Each tenant has own prompts |
| Vector Data | No | Yes | Data isolation required |
| Cache | Partial | Yes | Tenant-specific cache keys |
| Audit Logs | No | Yes | Compliance per tenant |
| Budget | No | Yes | Cost attribution |
| Rate Limits | No | Yes | Per-tenant throttling |

## Security Architecture

```mermaid
graph TB
    subgraph "Perimeter Security"
        WAF[WAF<br/>SQLi, XSS, Injection]
        DDoS[DDoS Protection]
        TLS[TLS 1.3<br/>Encryption in Transit]
    end

    subgraph "Application Security"
        AuthN[Authentication<br/>OAuth2 / OIDC]
        AuthZ[Authorization<br/>RBAC / ABAC]
        InputVal[Input Validation<br/>Schema, injection]
        OutputVal[Output Validation<br/>PII, hallucination]
    end

    subgraph "Data Security"
        EncryptAtRest[Encryption at Rest<br/>AES-256]
        Tokenization[Data Tokenization<br/>PII protection]
        Residency[Data Residency<br/>Regional isolation]
        KeyMgmt[Key Management<br/>HSM-backed]
    end

    subgraph "AI Security"
        PromptDefense[Prompt Injection<br/>Detection]
        ModelSecurity[Model Access<br/>Least privilege]
        AuditLogging[Audit Logging<br/>Immutable]
    end

    WAF --> AuthN --> AuthZ --> InputVal
    InputVal --> PromptDefense
    PromptDefense --> ModelSecurity
    ModelSecurity --> OutputVal
    OutputVal --> Tokenization
    TLS --> EncryptAtRest
    EncryptAtRest --> KeyMgmt
    Tokenization --> Residency
    AuditLogging -.-> InputVal
    AuditLogging -.-> OutputVal
    AuditLogging -.-> ModelSecurity
```

## Deployment Architecture

```mermaid
graph TB
    subgraph "Primary Region (London)"
        PrimaryLB[Load Balancer]
        PrimaryApp[App Pods<br/>Kubernetes]
        PrimaryModels[Model Gateway]
        PrimaryDB[(Primary DB<br/>PostgreSQL)]
        PrimaryCache[(Redis Cluster)]
        PrimaryVector[(Vector DB<br/>pgvector)]
    end

    subgraph "DR Region (Frankfurt)"
        DRApp[App Pods<br/>Warm Standby]
        DRDB[(Replicated DB)]
        DRVector[(Vector DB Replica)]
    end

    subgraph "External Services"
        OpenAI[OpenAI API]
        Anthropic[Anthropic API]
        Google[Google API]
    end

    PrimaryLB --> PrimaryApp
    PrimaryApp --> PrimaryModels
    PrimaryApp --> PrimaryDB
    PrimaryApp --> PrimaryCache
    PrimaryApp --> PrimaryVector

    PrimaryModels --> OpenAI
    PrimaryModels --> Anthropic
    PrimaryModels --> Google

    PrimaryDB -.->|Async Replication| DRDB
    PrimaryVector -.->|Async Replication| DRVector
    PrimaryApp -.->|Health Check| DRApp
```

## Key Design Decisions

### Decision 1: Build vs. Buy for Orchestration

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| Buy (LangChain Cloud) | Fast to start, managed | Vendor lock-in, less control | **No** |
| Build on OSS (LangGraph) | Full control, customizable | Engineering effort | **Yes** |
| Custom from scratch | Maximum flexibility | Very high effort | **No** |

### Decision 2: Vector Database Selection

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| pgvector | Existing infra, SQL integration, GDPR compliant | Less scalable than dedicated | **Primary** |
| Pinecone | Managed, highly scalable | External data, vendor lock-in | **Backup** |
| Milvus | Open source, scalable | Complex ops | **Evaluate** |

### Decision 3: Multi-Model Strategy

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| Single provider (OpenAI) | Simple, optimized | Vendor lock-in, no fallback | **No** |
| Multi-provider (3+) | Fallback, best-of-breed, negotiation | Complexity | **Yes** |
| Self-hosted only | Data residency, cost at scale | GPU investment, ops burden | **Supplementary** |

## Interview Questions

1. Design a GenAI platform for a global bank serving 100M customers.
2. How do you ensure data isolation between tenants in a shared platform?
3. Walk me through the complete request flow from user input to AI response.
4. How do you design for disaster recovery in a GenAI platform?
5. What are the key security layers in an enterprise GenAI architecture?

## Cross-References

- [model-routing.md](./model-routing.md) — Model routing architecture
- [multi-model-architecture.md](./multi-model-architecture.md) — Provider abstraction
- [ai-safety.md](./ai-safety.md) — Safety layer design
- [caching.md](./caching.md) — Caching architecture
- [../security/](../security/) — Security architecture details
- [../backend-engineering/](../backend-engineering/) — Service design patterns
- [../infrastructure/](../infrastructure/) — Infrastructure and deployment
