# System Design Interview Questions (20+)

## Question Framework for Any System Design Problem

1. **Clarify requirements** (5 min): Users, scale, latency, availability, constraints
2. **High-level design** (10 min): Draw architecture, identify main components
3. **Deep dive** (15 min): Data models, APIs, scaling, failure modes
4. **Tradeoffs** (5 min): Alternatives considered, why you chose what you chose

## 20+ System Design Questions with Answer Frameworks

### Q1: Design an enterprise chatbot for bank employees (50,000 users)

**Key points to cover**:
- RAG architecture: vector DB, embedding model, LLM, retriever
- Access control: RBAC, department-based filtering
- PII redaction before external API calls
- Caching strategy for common queries
- Audit logging of all interactions
- Fallback when AI can't answer (human escalation)
- Multi-channel (web, Teams, Slack)
- **Target metrics**: P95 < 3s, 99.9% availability, 500K queries/day

### Q2: Design a policy search and Q&A system for 10,000+ documents

**Key points**:
- Hybrid search (BM25 + vector) for best retrieval quality
- Document versioning and effective date tracking
- Incremental indexing when policies change
- Highlight relevant sections in search results
- Browse by category (retail, corporate, compliance)
- **Key tradeoff**: Elasticsearch + pgvector vs. single vector DB

### Q3: Design a regulatory compliance assistant

**Key points**:
- Highest accuracy bar -- zero tolerance for factual errors
- Strict verification: NLI-based claim checking, citation validation
- Human-in-the-loop for compliance-critical queries
- Regulation change tracking (Federal Register API, CFPB)
- Historical regulation lookup (what did Reg E say in 2020?)
- gpt-4o only -- no cost optimization for accuracy
- **Key design**: Verification engine that blocks unverified responses

### Q4: Design a customer service AI assistant (10M customers)

**Key points**:
- Intent classification to route queries (balance, dispute, product info)
- Core banking system integration (read-only account access)
- Action execution (transfers, payments) with confirmation
- Escalation to human agent with full context summary
- Omnichannel: web, mobile, phone (STT/TTS)
- Customer data NEVER sent to external LLMs
- **Key tradeoff**: Self-hosted for account-specific, cloud for general Q&A

### Q5: Design an AI model gateway serving 20+ internal applications

**Key points**:
- Model routing: intelligent selection based on cost, quality, availability
- Semantic caching of LLM responses
- Per-team budget tracking and enforcement
- Fallback chain (if primary fails, try secondary)
- Usage tracking and cost dashboard
- PII detection before external API calls
- **Key design**: Router that scores models per request

### Q6: Design a secure multi-tenant RAG platform

**Key points**:
- Tenant isolation at every layer (DB, cache, API)
- Row-level security in PostgreSQL
- Response validation (no cross-tenant citations)
- Per-tenant cache keys
- Admin console for managing access
- Audit logging per tenant
- **Key design**: Defense-in-depth isolation

### Q7: Design a multi-agent orchestration platform

**Key points**:
- Workflow definition (DAG of agent steps)
- Agent framework with tool access
- Human checkpoints in workflows
- State management and recovery
- Retry with exponential backoff
- Workflow versioning
- **Key tradeoff**: Custom orchestrator vs. Temporal/Argo

### Q8: Design an observability platform for GenAI

**Key points**:
- SDK instrumented in every application
- Event ingestion at 1M+/hour
- Cost tracking per team, per model
- Quality monitoring (groundedness, hallucination rate)
- Security monitoring (prompt injection, PII leaks)
- Dashboards for different stakeholders
- **Key design**: Two-store approach (ES for logs, ClickHouse for metrics)

### Q9: Design a prompt management platform

**Key points**:
- Prompt versioning and approval workflow
- A/B testing with traffic splitting
- Local caching in applications (< 10ms retrieval)
- Performance tracking per prompt
- Rollback to any version
- CI/CD integration
- **Key design**: Cache with short TTL + API fallback

### Q10: Design a developer code assistant for 2,000 internal developers

**Key points**:
- Self-hosted LLM (no proprietary code to external APIs)
- Code indexing pipeline (parse, embed, index all repos)
- Code graph for relationship tracking (call graphs, imports)
- IDE integration (VS Code, IntelliJ plugins)
- Code review, test generation, debugging
- **Key tradeoff**: CodeLlama/StarCoder self-hosted vs. GitHub Copilot

### Q11: Design a real-time fraud detection system

**Key points**:
- Stream processing (Kafka + Flink) for real-time scoring
- Feature store for transaction features
- ML model scoring with < 100ms latency
- Rule engine + ML model ensemble
- False positive handling (manual review queue)
- Model retraining pipeline
- **Key metric**: < 200ms end-to-end, < 1% false positive rate

### Q12: Design a payment processing system

**Key points**:
- Idempotent API (idempotency keys)
- Database transactions with SELECT FOR UPDATE
- Double-entry bookkeeping
- Audit trail (append-only)
- Reconciliation jobs
- Circuit breaker for external payment networks
- **Key design**: Exactly-once semantics via outbox pattern

### Q13: Design a notification system for 10M users

**Key points**:
- Multi-channel: email, SMS, push, in-app
- Template management and localization
- Rate limiting per user (don't spam)
- Delivery tracking and retry
- Preference management (opt-out, frequency)
- **Key scaling**: Queue-based, batch sending per channel

### Q14: Design a document management system

**Key points**:
- Version control for documents
- Access control (RBAC, document-level permissions)
- Full-text search with metadata filtering
- Audit trail (who accessed what, when)
- Retention policies and legal holds
- **Key design**: Object storage for files + metadata in PostgreSQL

### Q15: Design a rate limiting system

**Key points**:
- Token bucket or sliding window algorithm
- Redis-based for distributed state
- Per-user, per-endpoint, per-IP limits
- Lua scripts for atomic operations
- Response headers (X-RateLimit-Remaining)
- **Key challenge**: Distributed rate limiting consistency

## Answer Framework Template

For any system design question, use this structure:

```
1. Requirements clarification
   - Who are the users?
   - What is the scale (users, requests, data)?
   - What are the latency/availability requirements?
   - Any specific constraints (security, compliance)?

2. High-level architecture
   - Draw the main components
   - Show data flow between components

3. Component deep-dive
   - Data model/schema
   - API design
   - Scaling strategy for each component

4. Tradeoffs
   - What alternatives did I consider?
   - Why did I choose what I chose?

5. Failure modes
   - What can go wrong?
   - How does the system handle it?

6. Monitoring
   - What metrics matter?
   - When do we alert?
```

## Banking-Specific Design Considerations

Always mention these when relevant:
- **Audit logging**: Every action must be logged
- **Access control**: RBAC with department and clearance levels
- **Data encryption**: At rest and in transit
- **Regulatory compliance**: SOC 2, PCI DSS, GDPR
- **High availability**: Multi-AZ deployment minimum
- **Disaster recovery**: RPO < 1 hour, RTO < 4 hours
- **PII handling**: Redaction, masking, access logging
