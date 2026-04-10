# Case Study: RAG Data Exfiltration Incident

## Executive Summary

A Retrieval-Augmented Generation (RAG) system used for internal customer support leaked personally identifiable information (PII) across customer boundaries. The incident exposed account details of 2,347 customers through AI-generated responses and was classified as SEV-1. The root cause was a combination of insufficient document-level access control in the vector database, overly permissive retrieval thresholds, and missing output filtering.

**Severity:** SEV-1 (Critical Data Exposure)
**Duration:** 14 hours (detection to containment)
**Exposure window:** Estimated 11 days before detection
**Customers affected:** 2,347
**Regulatory notification:** Required under GDPR Article 33, notified within 48 hours
**Financial impact:** $1.2M (fines, remediation, legal)

---

## Background and Context

### The System

Our internal customer support assistant, "AssistBot," used a RAG architecture to answer questions from the customer support team. The system:

1. Received natural language queries from support agents
2. Embedded the query and searched a vector database (Pinecone) for relevant documents
3. Retrieved top-K documents (K=10) with cosine similarity threshold of 0.72
4. Constructed a prompt with retrieved context and sent to GPT-4
5. Returned the generated answer to the support agent

### The Data

The vector database contained:
- Customer account summaries (5.2M documents)
- Transaction histories (12.8M documents)
- Support ticket resolutions (3.1M documents)
- KYC (Know Your Customer) profiles (890K documents)
- Internal policy documents (45K documents)

All documents were chunked at 512 tokens with 50-token overlap and indexed in a single Pinecone namespace.

### Access Model (Flawed)

The system relied on application-level access control:
- Support agents authenticated via SSO
- The application layer was supposed to filter results by agent permissions
- **However**, the vector retrieval step had no access control enforcement
- All chunks from all customers existed in the same namespace with no tenant isolation

---

## Timeline of Events

```mermaid
timeline
    title RAG Data Exfiltration Incident Timeline
    section Day -11 to Day 0
        Document migration : All customer data<br/>migrated to single<br/>Pinecone namespace
        : Access control<br/>filtering skipped<br/>in code review
        : Similarity threshold<br/>set too low (0.72)
    section Day 0 - Incident Day
        09:14 AM : Agent queries about<br/>customer "Smith"
        09:14:23 : RAG retrieves chunks<br/>from unrelated customer<br/>"Smythe" due to similarity
        09:14:25 : LLM synthesizes answer<br/>including Smythe's SSN,<br/>account balance, loan details
        09:15 AM : Agent notices wrong<br/>customer data, reports<br/>to team lead
        09:32 AM : Team lead escalates<br/>to engineering Slack channel
        09:47 AM : On-call engineer paged<br/>SEV-1 declared
        10:05 AM : Incident commander<br/>assigned, war room opened
        10:23 AM : Root cause identified:<br/>no tenant isolation in<br/>vector DB namespace
        10:45 AM : Containment decision:<br/>take AssistBot offline
        11:02 AM : AssistBot disabled,<br/>support reverts to<br/>manual processes
        12:30 PM : Log analysis begins<br/>to determine exposure<br/>scope
        03:15 PM : Initial scope:<br/>~800 queries may have<br/>returned cross-customer data
        06:00 PM : Scope expanded to<br/>2,347 customers after<br/>full log review
        08:00 PM : Legal and compliance<br/>notified, GDPR 48-hour<br/>clock starts
        11:30 PM : Containment complete,<br/>system remains offline
    section Day 1
        09:00 AM : Executive briefing,<br/>regulatory notification<br/>draft prepared
        02:00 PM : Customer notification<br/>process begins
        06:00 PM : GDPR notification filed<br/>with supervisory authority
    section Day 2-7
        Remediation begins : Per-customer namespaces<br/>designed and implemented
        : Output filtering<br/>pipeline added
        : Access control<br/>enforcement at DB layer
    section Day 14
        AssistBot restored : With full tenant isolation,<br/>output filtering, and<br/>audit logging
```

### Detailed Minute-by-Minute (First 2 Hours)

| Time | Event | Actor | Notes |
|------|-------|-------|-------|
| 09:14:00 | Agent Sarah submits query: "What is the account status for customer reference CR-44821?" | Support Agent | Normal query |
| 09:14:03 | Query embedded: `[0.023, -0.156, 0.089, ...]` (1536-dim) | Embedding API | Using text-embedding-ada-002 |
| 09:14:05 | Vector search executed with top_k=10, threshold=0.72 | Pinecone | Returns 10 chunks |
| 09:14:06 | **BUG:** Chunks returned include data from CR-44819 (customer "Smythe") | Pinecone | Similar last names caused high similarity scores |
| 09:14:08 | No access control filter applied | Application | **Critical gap** - app did not filter by customer ID |
| 09:14:10 | Prompt constructed with retrieved context | Application | Context includes Smythe's SSN, account number, loan balance |
| 09:14:15 | GPT-4 generates response incorporating Smythe's PII | LLM | Model faithfully reproduces PII from context |
| 09:14:23 | Response displayed to agent | UI | Agent sees wrong customer's data |
| 09:15:12 | Agent notices discrepancy: "This doesn't look like Smith's account" | Support Agent | |
| 09:16:00 | Agent screenshots the response and messages team lead | Support Agent | Evidence preserved |
| 09:32:00 | Team lead posts in #eng-support Slack: "AssistBot is returning wrong customer data, possible PII leak" | Team Lead | |
| 09:35:00 | Engineering manager sees message, starts investigation | Engineering Manager | |
| 09:47:00 | SEV-1 declared, on-call paged via PagerDuty | Engineering Manager | |
| 10:05:00 | Incident Commander (IC) Maria joins, opens Zoom war room | IC | |

---

## Root Cause Analysis

### Technical Root Causes

1. **Missing Tenant Isolation in Vector Database**
   - All customer documents were indexed in a single Pinecone namespace
   - No metadata filtering by customer ID at the retrieval layer
   - Pinecone supports metadata filtering, but it was not configured

2. **Overly Permissive Similarity Threshold**
   - Threshold of 0.72 allowed semantically similar but wrong-customer documents to be retrieved
   - Customer names with similar spellings ("Smith" vs "Smythe") scored above threshold
   - No business-logic validation on retrieved documents before prompt construction

3. **Missing Output Filtering**
   - No PII detection on LLM output before display
   - No regex-based redaction of SSNs, account numbers, or other sensitive patterns
   - No post-generation validation step

4. **Insufficient Code Review**
   - The vector retrieval code bypassed the normal review process during a migration sprint
   - The PR that moved data to a single namespace was approved by a single reviewer
   - No security review was triggered for the data architecture change

### Organizational Root Causes

1. **Siloed Security Review**
   - Security team reviewed the initial RAG design but not the migration changes
   - Change management process did not flag vector DB namespace consolidation as a security-relevant change

2. **Performance Over Security Trade-off**
   - Single namespace was chosen for query performance (avoiding namespace switching overhead)
   - Security implications were discussed but dismissed as "handled at the application layer"
   - No formal risk acceptance was documented

3. **Missing Observability**
   - No alerting on cross-customer data retrieval patterns
   - No audit logging of which documents were retrieved per query
   - No anomaly detection on retrieval patterns

4. **Inadequate Testing**
   - No test cases for cross-customer data leakage
   - Load testing focused on latency and throughput, not data isolation
   - No red team testing of the RAG system before production launch

---

## What Went Wrong Technically

### The Code Path

```python
# BEFORE (vulnerable code)
async def retrieve_context(query: str, agent_id: str) -> List[Document]:
    # Embed the query
    embedding = await embedding_model.encode(query)

    # Search vector DB - NO customer filtering!
    results = await pinecone_index.query(
        vector=embedding,
        top_k=10,
        include_metadata=True,
        # BUG: No filter by customer_id
        # BUG: No filter by agent's permitted customers
    )

    # Threshold filtering - too permissive
    documents = [
        Document.from_pinecone(match)
        for match in results["matches"]
        if match["score"] >= 0.72  # Too low!
    ]

    return documents  # May contain any customer's data
```

### The Data Flow

```
Support Agent Query
    |
    v
Embedding API (text-embedding-ada-002)
    |
    v
Pinecone Vector Search (no tenant filter)
    |
    v
Top-10 Results (may include ANY customer)
    |
    v
[NO ACCESS CONTROL CHECK]
    |
    v
Prompt Construction with Retrieved Context
    |
    v
LLM Generation (faithfully reproduces PII)
    |
    v
[NO PII FILTERING ON OUTPUT]
    |
    v
Response to Agent (contains wrong customer's PII)
```

---

## What Went Wrong Organizationally

1. **Migration Process Failure**: The decision to consolidate namespaces was made in a sprint planning meeting without architectural review. The migration was treated as an "infra optimization" rather than a "security architecture change."

2. **Blind Spot in Security Model**: The team assumed application-level access control was sufficient. They did not apply defense-in-depth principles to the RAG architecture.

3. **Velocity Pressure**: The migration was rushed to meet a quarterly OKR for query latency improvement. The team skipped the security review to "save a week."

4. **Missing Champion for Data Governance**: No data protection officer (DPO) was involved in the RAG system design or changes.

---

## Immediate Response and Mitigation

### First 4 Hours (Containment)

1. **System Disabled**: AssistBot was taken offline at 11:02 AM
2. **Log Preservation**: All query logs, retrieval results, and LLM responses were frozen and copied to secure storage
3. **Scope Determination**: Engineering team ran a script to identify all queries where retrieved documents came from a different customer than the queried customer
4. **Support Team Notified**: All 147 support agents were instructed to stop using AssistBot immediately

### Hours 4-14 (Assessment)

1. **Full Log Analysis**: Every AssistBot query since the migration was analyzed
2. **Cross-Customer Retrieval Detection**: A script identified 3,891 queries where documents from multiple customers were retrieved together
3. **PII Exposure Confirmation**: 1,247 of those queries resulted in responses containing PII from a customer other than the queried one
4. **Unique Customer Count**: 2,347 unique customers had their data potentially exposed

### Notification Process

1. **Legal Notification**: 08:00 PM on Day 0
2. **GDPR Notification**: Filed with the ICO (Information Commissioner's Office) within 48 hours
3. **Customer Notification**: Individual letters sent to all 2,347 affected customers within 7 days
4. **Credit Monitoring**: 2 years of credit monitoring offered to all affected customers

---

## Long-Term Fixes and Systemic Changes

### Technical Fixes

1. **Per-Customer Namespaces with Metadata Filtering**
   ```python
   # AFTER (fixed code)
   async def retrieve_context(query: str, agent_id: str, customer_id: str) -> List[Document]:
       embedding = await embedding_model.encode(query)

       agent_permissions = await get_agent_customer_permissions(agent_id)

       results = await pinecone_index.query(
           vector=embedding,
           top_k=10,
           filter={
               "customer_id": {"$eq": customer_id},
               "permitted_agents": {"$in": [agent_id]},
               "document_type": {"$in": agent_permissions.allowed_doc_types},
           },
           include_metadata=True,
       )

       documents = [
           Document.from_pinecone(match)
           for match in results["matches"]
           if match["score"] >= 0.85  # Raised threshold
       ]

       # Post-retrieval validation
       for doc in documents:
           assert doc.customer_id == customer_id, f"CRITICAL: Wrong customer data in retrieval"

       return documents
   ```

2. **Output PII Filtering Pipeline**
   - Regex-based PII detection (SSN patterns, account number formats, routing numbers)
   - Named Entity Recognition (NER) model for detecting person names, addresses
   - Pre-commit hook that blocks any LLM response containing detected PII from non-query customer

3. **Audit Logging**
   - Every retrieval logged with: query embedding hash, retrieved document IDs, customer IDs, scores
   - Every LLM response logged with: PII scan results, customer ID, agent ID, timestamp
   - Tamper-evident logging using append-only log storage

4. **Retrieval Anomaly Detection**
   - Alerting when a single query retrieves documents from more than one customer
   - Alerting when retrieval scores for a single query span an unusually wide range
   - Daily reconciliation of queries vs. expected customer boundaries

### Process Changes

1. **Mandatory Security Review for RAG Architecture Changes**
   - Any change to vector DB structure, namespaces, or indexing requires security team sign-off
   - Changes classified as "data architecture" rather than "infrastructure optimization"

2. **Formal Risk Acceptance Process**
   - Any security trade-off (performance vs. isolation) requires documented risk acceptance
   - Risk acceptance must be approved by CISO and DPO
   - Time-bound with mandatory review every 90 days

3. **Red Team Testing**
   - Quarterly red team exercises targeting the RAG system
   - Specific test cases for cross-customer data leakage
   - Prompt injection testing included in red team scope

4. **Access Control Auditing**
   - Automated weekly audit of vector DB access patterns
   - Monthly review of agent permissions and customer access assignments
   - Quarterly penetration test of the RAG system

### Cultural Changes

1. **Security Champions Program**: Each squad now has a designated security champion who participates in security reviews
2. **Blameless Postmortem Culture**: The incident postmortem was conducted with a blameless approach, focusing on system fixes
3. **"Stop the Line" Authority**: Any engineer can halt a release if they identify a potential data isolation issue
4. **Data Governance as a First-Class Concern**: Data protection is now part of the definition of done for all features

---

## Lessons Learned

1. **Defense in Depth Matters**: Relying on a single layer of access control (application layer) is insufficient. Every layer must enforce access control independently.

2. **Performance vs. Security Trade-offs Must Be Documented**: The single-namespace decision was made for performance but without formal risk documentation or approval.

3. **Testing Must Cover Data Isolation**: Load testing is not enough. Data isolation testing is critical for multi-tenant systems.

4. **PII Filtering on Output Is Non-Negotiable**: Even with perfect retrieval controls, output filtering provides a critical safety net.

5. **Observability Is a Security Control**: Without proper logging and alerting, the incident could have gone undetected for weeks longer.

6. **Code Review Gaps Are Exploitable**: The bypassed security review was the direct cause of the vulnerability going into production.

7. **Regulatory Timelines Are Tight**: The 48-hour GDPR notification window requires pre-built notification templates and processes.

---

## Interview Questions Derived From This Case Study

1. **System Design**: "Design a RAG system for a multi-tenant banking application where each customer's data must be strictly isolated. How would you architect the vector database, retrieval, and generation layers?"

2. **Security**: "What are the unique security challenges of RAG systems compared to traditional CRUD applications? How do you prevent data leakage across tenant boundaries?"

3. **Incident Response**: "You discover that your AI system has been returning incorrect customer data for 2 weeks. Walk me through your first 24 hours of response."

4. **Code Review**: "Here's a vector retrieval function. What security issues do you spot? How would you fix them?" (Show the vulnerable code)

5. **Architecture Trade-offs**: "Your team wants to consolidate vector DB namespaces for performance. What questions would you ask before approving this change?"

6. **Compliance**: "How would you design audit logging for a RAG system to satisfy GDPR requirements for data access tracking?"

7. **Testing**: "What test cases would you write to verify that a RAG system cannot leak data between customers?"

8. **Observability**: "What metrics and alerts would you set up to detect cross-customer data leakage in a RAG system in real-time?"

---

## Cross-References

- See `../incident-management/incident-classification.md` for SEV level definitions
- See `../incident-management/regulatory-notification.md` for GDPR notification procedures
- See `../databases/vector-databases.md` for vector database security best practices
- See `../security/output-filtering.md` for PII detection and filtering strategies
- See `../architecture/rag-architecture.md` for secure RAG design patterns
