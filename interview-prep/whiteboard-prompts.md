# Whiteboard Prompts (20)

## System Design Prompts

### Prompt 1: Enterprise Chatbot
"Design an AI chatbot for 50,000 bank employees to find policy information. Show the architecture, data flow, and how you handle access control."

**Key elements to draw**:
- Client apps (web, Teams, Slack) -> API Gateway -> Auth -> Chat Service
- RAG pipeline: Query -> Embed -> Vector DB -> Re-rank -> LLM -> Response
- Access control filter at retrieval layer
- Audit logging for every interaction
- Cache layer for common queries

### Prompt 2: RAG Ingestion Pipeline
"Draw the document ingestion pipeline for a RAG system. Show how raw documents become searchable vector embeddings."

**Key elements**: Source systems -> Fetcher -> Parser -> Cleaner -> Deduplicator -> Chunker -> Embedder -> Vector DB + Metadata Store

### Prompt 3: Multi-Tenant Vector DB
"Design the data isolation architecture for a vector database serving 10 departments with different access levels."

**Key elements**: Row-level security, metadata filtering, per-tenant cache keys, response validation, defense-in-depth layers

### Prompt 4: AI Model Gateway
"Design a centralized gateway that routes requests to multiple LLM providers with cost optimization, caching, and fallback."

**Key elements**: API Gateway -> Auth -> Rate Limiter -> PII Redactor -> Semantic Cache -> Model Router -> Provider (with fallback chain) -> Response Validator -> Usage Tracker

### Prompt 5: Observability Architecture
"Draw the observability architecture for monitoring 20+ GenAI applications. Show data flow from SDK to dashboards."

**Key elements**: SDK in each app -> Log/Trace/Metric collectors -> Stream processor -> PII masking -> Storage (ES for logs, TSDB for metrics) -> Dashboards + Alerting

### Prompt 6: Hybrid Search Architecture
"Draw the architecture for a hybrid search system combining BM25 and vector search with re-ranking."

**Key elements**: Query -> BM25 index + Vector index -> Score normalization -> Fusion -> Re-ranker -> Top-K results

### Prompt 7: Secure Document Access
"Design the access control flow for a document search system where different employees see different documents based on their role and clearance level."

**Key elements**: User -> Auth -> RBAC Engine -> Filtered retrieval (metadata filter) -> LLM -> Response validation (verify citations) -> User

### Prompt 8: Real-Time Fraud Detection
"Design a real-time fraud detection system for credit card transactions."

**Key elements**: Transaction -> Stream (Kafka) -> Feature extraction -> ML scoring + Rule engine -> Decision (approve/flag/reject) -> Notification + Audit log

### Prompt 9: Prompt Management System
"Design a prompt versioning and A/B testing platform."

**Key elements**: Prompt Editor -> Version Manager -> A/B Test Engine -> Cache (Redis) -> Application SDK -> LLM -> Performance Tracker -> Analytics Dashboard

### Prompt 10: Multi-Agent Workflow
"Draw a workflow where multiple AI agents collaborate: document extractor -> risk assessor -> compliance checker -> summary generator."

**Key elements**: Workflow definition (DAG) -> Orchestrator -> Agent 1 (with tools) -> State -> Agent 2 -> Human checkpoint -> Agent 3 -> Agent 4 -> Result

---

## Data Structure and Algorithm Prompts

### Prompt 11: Token Bucket Rate Limiter
"Draw the token bucket algorithm and show how it handles a burst of requests followed by a steady stream."

### Prompt 12: LRU Cache
"Draw the data structure for an LRU cache with O(1) get and put. Show what happens on a cache miss when the cache is full."

### Prompt 13: HNSW Index
"Draw a simplified HNSW graph with 3 layers. Show how a search navigates from the top layer to the bottom."

### Prompt 14: Sliding Window
"Draw the sliding window algorithm for rate limiting. Show a 1-minute window with requests at different timestamps."

### Prompt 15: Hash Chain
"Draw a hash chain for tamper-proof audit logs. Show how tampering with one entry breaks the chain."

---

## Banking Domain Prompts

### Prompt 16: Payment Flow
"Draw the flow of a domestic wire transfer from initiation to settlement, showing all systems involved."

### Prompt 17: Loan Approval
"Draw the loan approval process flow from application to funding, showing decision points and systems."

### Prompt 18: Compliance Check
"Draw the compliance checking flow for a new customer onboarding, showing KYC, AML screening, and sanctions checks."

### Prompt 19: Regulatory Reporting
"Draw the data flow for generating a regulatory report: source systems -> data warehouse -> aggregation -> report -> submission."

### Prompt 20: Incident Response
"Draw the incident response flow for a data breach: detection -> containment -> investigation -> remediation -> notification -> post-mortem."

---

## Tips for Whiteboard Interviews

### Before You Start Drawing
1. **Clarify the problem**: Make sure you understand what's being asked
2. **State your assumptions**: "I'll assume 50,000 users and 500K queries per day"
3. **Outline your approach**: "I'll start with the high-level architecture, then drill into each component"

### While Drawing
4. **Start big, go small**: Draw boxes and arrows first, then add details
5. **Label everything**: Every box should have a name, every arrow should show data flow
6. **Talk while you draw**: Explain your thinking as you go
7. **Use standard notation**: Rectangles for services, cylinders for databases, arrows for data flow

### Common Mistakes
8. **Don't start drawing immediately**: Always clarify requirements first
9. **Don't skip the "why"**: Explain why you chose each component
10. **Don't forget failure modes**: Show what happens when components fail
11. **Don't ignore security**: Always include auth, encryption, audit logging
12. **Don't forget scaling**: Show how the system handles growth

### What Great Candidates Do
13. **Draw iteratively**: "Here's my first cut. Now let me refine the data layer..."
14. **Self-correct**: "Actually, that won't work because... Let me reconsider."
15. **Ask for feedback**: "Does this architecture make sense so far?"
16. **Quantify**: "This component handles 1000 QPS, so we'll need 3 instances"
17. **Show tradeoffs**: "I chose PostgreSQL over MongoDB because we need ACID transactions for account data"

### Practice Framework
For each prompt, practice:
1. **2 minutes**: Clarify requirements and state assumptions
2. **3 minutes**: Draw high-level architecture
3. **5 minutes**: Deep dive into 2-3 key components
4. **3 minutes**: Discuss tradeoffs and alternatives
5. **2 minutes**: Address failure modes and scaling

Total: 15 minutes per prompt. Time yourself.
