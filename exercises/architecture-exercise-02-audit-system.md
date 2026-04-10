# Architecture Exercise 02: Audit System Design

> Design a tamper-resistant audit logging system for GenAI interactions — a regulatory requirement in banking.

## Problem Statement

The bank needs an audit system that logs every GenAI interaction for compliance, security, and forensic analysis. The system must:

1. Capture every request, response, and metadata
2. Be tamper-resistant (logs cannot be modified or deleted after writing)
3. Support 7-year retention with search capability
4. Handle 1 million audit events per day
5. Be queryable for investigations (by user, date, model, risk flags)
6. Support regulatory e-discovery requests
7. Encrypt sensitive data with key rotation

Design the system architecture and data model.

## Audit Event Schema

```json
{
  "event_id": "uuid",
  "event_type": "query|response|error|block|access_change",
  "timestamp": "ISO8601",
  "user_id": "hashed-employee-id",
  "session_id": "uuid",
  "request": {
    "query_text_hash": "sha256",
    "query_length": 1234,
    "query_language": "en",
    "injection_score": 0.02,
    "pii_detected": false,
    "prompt_template_version": "v2.1",
    "model_requested": "gpt-4-turbo"
  },
  "response": {
    "response_text_hash": "sha256",
    "response_length": 567,
    "tokens_used": 89,
    "content_filtered": true,
    "filter_reasons": ["profanity"],
    "latency_ms": 342,
    "model_used": "gpt-4-turbo-2024-04-09",
    "model_version": "v1.3"
  },
  "context": {
    "documents_retrieved": 3,
    "document_ids_hashed": ["hash1", "hash2", "hash3"],
    "cache_hit": false,
    "fallback_used": false
  },
  "compliance": {
    "risk_score": 0.1,
    "flags": [],
    "reviewed": false,
    "reviewer_id": null,
    "review_decision": null
  }
}
```

## Constraints

- **Tamper resistance:** Once written, logs cannot be modified. Use append-only storage.
- **Encryption:** Sensitive fields (user_id, query_hash) encrypted at rest.
- **Retention:** 7 years for standard events, 10 years for flagged events.
- **Search:** Investigators must be able to search by user, date range, risk score, and flags.
- **Performance:** Audit logging must not add more than 10ms to the user-facing request.
- **Durability:** Zero data loss. Events must survive process and zone failures.

## Expected Deliverables

1. **Architecture diagram** showing the audit pipeline
2. **Storage design** with partitioning and retention strategy
3. **Tamper-resistance mechanism** (how you prevent modification/deletion)
4. **Encryption strategy** (which fields, which keys, rotation schedule)
5. **Query patterns** for common investigation scenarios
6. **Cost estimate** for 7-year storage of 1M events/day

## Hints

### Hint 1: Pipeline Design

```
Async pipeline (doesn't block user request):

User Request → GenAI Service → Response to User
                     │
                     └──→ Audit Event (async)
                               │
                               ├──→ Message Queue (Kafka)
                               │       │
                               │       ├──→ Consumer → Hot Storage (Postgres, 90 days)
                               │       │
                               │       ├──→ Consumer → Warm Storage (S3, 7 years)
                               │       │
                               │       └──→ Consumer → Security Stream (real-time alerts)
                               │
                               └──→ Write-Ahead Log (local, crash recovery)
```

### Hint 2: Tamper Resistance

```
Options for tamper-resistant storage:
1. Append-only database table (PostgreSQL with row-level security)
2. Write-once object storage (S3 Object Lock / WORM)
3. Blockchain/hash chain (each log entry hashes the previous)
4. Immutable ledger (Amazon QLDB)

Recommended for banking: S3 Object Lock + hash chain for verification
```

### Hint 3: Cost Estimation

```
1M events/day × 2KB/event = 2GB/day
2GB/day × 365 days = 730GB/year
730GB × 7 years = 5.1TB total

Storage costs:
- Hot (SSD, 90 days): 180GB × $0.10/GB/month = $18/month
- Warm (S3 Standard, 1 year): 730GB × $0.023/GB/month = $17/month
- Cold (S3 Glacier, 6 years): 4.4TB × $0.004/GB/month = $18/month
- Total storage: ~$53/month

Processing costs (Kafka, consumers): ~$200/month
Total estimated: ~$250-500/month
```

## Extensions

1. **Real-time alerting:** Build a stream processor that flags high-risk events in real-time and alerts the security team.

2. **GDPR deletion:** How do you handle a "right to erasure" request when regulations require 7-year retention?

3. **Cross-region replication:** Design multi-region audit log storage for disaster recovery.

4. **Log integrity verification:** Implement a periodic verification process that checks the hash chain and reports any tampering.

5. **E-discovery interface:** Design a search interface for compliance investigators that supports complex queries with audit trail of the searches themselves.

## Interview Relevance

Audit system design tests understanding of compliance engineering:

| Skill | Why It Matters |
|-------|---------------|
| Async pipeline design | Don't block user requests for logging |
| Tamper resistance | Regulatory requirement |
| Data retention | GDPR vs. banking regulations |
| Encryption | Protecting sensitive audit data |
| Cost estimation | Infrastructure budgeting |

**Follow-up questions:**
- "How do you handle audit log writes failing?"
- "What happens if the message queue is down?"
- "How do you verify that logs haven't been tampered with?"
- "How would you handle a subpoena for specific user logs?"
