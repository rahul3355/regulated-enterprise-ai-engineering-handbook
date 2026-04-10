# Audit Trails and Logging

## What Audit Trails Are and Why They Matter

Audit trails are chronological records of activities, events, and data changes within a system. For banking GenAI systems, audit trails are not just good practice -- they are a legal and regulatory requirement under virtually every compliance framework.

**Regulatory requirements**:
- **GDPR** Article 30: Records of processing activities
- **PCI-DSS** Requirement 10: Log and monitor all access
- **SOC 2** CC7.2: Detection and monitoring of security events
- **ISO 27001** A.8.15: Logging
- **SR 11-7**: Model documentation and monitoring records
- **BCBS 239**: Data lineage and reconciliation records
- **FCA**: Incident reporting and governance records

**Without proper audit trails, you cannot**:
- Prove compliance during an audit
- Investigate a security incident
- Fulfill a data subject access request
- Trace a regulatory report back to source data
- Detect unauthorized access
- Prove model validation was performed

## What to Log

### Security Events

| Event | Required Fields | Priority |
|-------|----------------|----------|
| Authentication success | User ID, timestamp, source IP, method, MFA status | HIGH |
| Authentication failure | User ID (if known), timestamp, source IP, method, reason | HIGH |
| Authorization failure | User ID, timestamp, resource, action attempted, source IP | CRITICAL |
| Account lockout | User ID, timestamp, reason, failed attempt count | HIGH |
| Privilege escalation | User ID, timestamp, old privilege, new privilege, reason, approver | CRITICAL |
| Session creation | Session ID, user ID, timestamp, source IP, user agent | MEDIUM |
| Session termination | Session ID, user ID, timestamp, reason | LOW |

### Data Access Events

| Event | Required Fields | Priority |
|-------|----------------|----------|
| PII read | User ID, timestamp, data category, record ID, purpose, lawful basis | HIGH |
| PII create | User ID, timestamp, data category, record ID | HIGH |
| PII update | User ID, timestamp, data category, record ID, old value (masked), new value (masked) | HIGH |
| PII delete | User ID, timestamp, data category, record ID, reason | HIGH |
| Bulk data export | User ID, timestamp, record count, data categories, export destination | CRITICAL |
| Data import | User ID, timestamp, source, record count, validation results | MEDIUM |

### Model-Related Events

| Event | Required Fields | Priority |
|-------|----------------|----------|
| Model training started | Model ID, user ID, timestamp, training data reference, hyperparameters | MEDIUM |
| Model training completed | Model ID, timestamp, training duration, metrics, data size | MEDIUM |
| Model deployed | Model ID, user ID, timestamp, version, environment, approval reference | HIGH |
| Model prediction | Model ID, timestamp, input hash (not raw input), output, confidence | MEDIUM |
| Model performance alert | Model ID, timestamp, metric name, current value, threshold, severity | HIGH |
| Model validation completed | Model ID, validator ID, timestamp, result, findings | HIGH |

### AI-Specific Events

| Event | Required Fields | Priority |
|-------|----------------|----------|
| Prompt received | Timestamp, user/session ID, prompt hash (redacted), PII detected (Y/N) | HIGH |
| Prompt sent to AI API | Timestamp, API endpoint, model used, tokens in, PII redacted (Y/N) | HIGH |
| AI response received | Timestamp, model used, tokens out, safety flags, content filter results | HIGH |
| AI response returned to user | Timestamp, user/session ID, response status, safety checks passed | MEDIUM |
| Content filter triggered | Timestamp, filter name, reason, severity, action taken | HIGH |
| Prompt injection detected | Timestamp, pattern matched, confidence, action taken | CRITICAL |
| Hallucination detected | Timestamp, detection method, confidence, action taken | HIGH |

### System Events

| Event | Required Fields | Priority |
|-------|----------------|----------|
| Configuration change | User ID, timestamp, config item, old value, new value, reason | HIGH |
| System startup/shutdown | Timestamp, reason, duration, status | MEDIUM |
| Backup created | Timestamp, type, size, retention period, verification status | MEDIUM |
| Backup restored | Timestamp, reason, scope, verification results | HIGH |
| Security scan completed | Timestamp, scanner, findings count by severity | MEDIUM |

## Log Format

All logs must be structured, machine-readable, and consistent.

```json
{
  "timestamp": "2024-01-15T14:32:01.123Z",
  "level": "INFO",
  "service": "ai-chat-service",
  "event_type": "ai_prompt_received",
  "correlation_id": "req-abc-123-xyz",
  "user": {
    "user_id": "user-456",
    "role": "customer_service_agent",
    "session_id": "sess-789"
  },
  "action": {
    "type": "prompt_received",
    "resource": "customer_chat",
    "result": "success"
  },
  "data": {
    "pii_detected": false,
    "prompt_length": 150,
    "language": "en"
  },
  "source": {
    "ip": "10.0.1.50",
    "hostname": "chat-frontend-01",
    "user_agent": "BankingChat/2.1"
  },
  "metadata": {
    "environment": "production",
    "region": "us-east-1",
    "version": "1.4.2"
  }
}
```

## What NOT to Log

```
NEVER LOG:
├── Raw passwords or password hashes
├── API keys, tokens, or secrets
├── Unencrypted PII (name, email, phone, SSN, PAN)
├── Full prompt text containing PII
├── Full AI responses containing PII
├── Credit card numbers (even masked, in some contexts)
├── Bank account numbers
├── Health information
├── Biometric data
├── Private keys
├── Encryption keys
└── Any data classified as "Restricted" or higher
```

## Log Protection

### Immutability

```
IMMUTABILITY CONTROLS:
├── Write-once storage for audit logs
├── Append-only log streams
├── Cryptographic signing of log entries
├── No delete or update operations on logs
├── Separate permissions (writers cannot delete)
├── Tamper detection (hash chains)
└── Regular integrity verification
```

```python
class ImmutableAuditLog:
    """Write-once audit log with integrity verification."""

    def __init__(self, storage: WriteOnceStorage, key: bytes):
        self.storage = storage
        self.key = key
        self.last_hash = self.get_last_hash()

    def append(self, entry: LogEntry) -> str:
        """Append a log entry with integrity protection."""
        # Include previous hash to create a chain
        entry.previous_hash = self.last_hash

        # Sign the entry
        entry.signature = self.sign_entry(entry)

        # Compute this entry's hash
        entry_hash = self.compute_hash(entry)
        self.last_hash = entry_hash

        # Write to immutable storage
        entry_id = self.storage.write(entry)

        return entry_id

    def sign_entry(self, entry: LogEntry) -> str:
        """Sign a log entry using HMAC."""
        data = entry.to_json()
        return hmac.new(self.key, data.encode(), hashlib.sha256).hexdigest()

    def verify_integrity(self) -> IntegrityReport:
        """Verify the entire log chain has not been tampered with."""
        entries = self.storage.read_all()
        current_hash = None
        for entry in entries:
            # Verify previous hash link
            if entry.previous_hash != current_hash and current_hash is not None:
                return IntegrityReport(
                    valid=False,
                    reason="Hash chain broken",
                    entry_id=entry.id,
                )
            # Verify signature
            expected_sig = self.sign_entry(entry)
            if entry.signature != expected_sig:
                return IntegrityReport(
                    valid=False,
                    reason="Signature mismatch",
                    entry_id=entry.id,
                )
            current_hash = self.compute_hash(entry)
        return IntegrityReport(valid=True)
```

### Centralized Log Storage

```
LOG ARCHITECTURE:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ App Log  │  │ App Log  │  │ App Log  │
│ Service A│  │ Service B│  │ Service C│
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
             ┌──────┴──────┐
             │ Log Shipper │
             │ (Fluentd/   │
             │  Filebeat)  │
             └──────┬──────┘
                    │
             ┌──────┴──────┐
             │ Log Aggregator │
             │ (Elasticsearch │
             │  / CloudWatch  │
             │  / Splunk)     │
             └──────┬──────┘
                    │
             ┌──────┴──────┐
             │ Immutable   │
             │ Storage     │
             │ (S3 Object  │
             │  Lock /     │
             │  WORM)      │
             └─────────────┘
```

### Log Retention

| Log Type | Minimum Retention | Immediately Available |
|----------|------------------|----------------------|
| Security events | 3 years | 90 days |
| Authentication events | 3 years | 90 days |
| Data access events | 3 years | 90 days |
| Model events | 5 years | 1 year |
| System events | 1 year | 30 days |
| Audit trail changes | 7 years | 1 year |
| Incident-related logs | 7 years | Full period |
| Regulatory report lineage | 7 years | Full period |

## Log Monitoring and Alerting

### Automated Log Analysis

```
ALERT RULES:
├── Authentication:
│   ├── 5+ failed logins from same IP in 5 minutes
│   ├── Login from new geographic location
│   ├── Login outside business hours for role
│   ├── Privileged account used without ticket
│   └── Service account used interactively
├── Data access:
│   ├── Bulk data export (>1000 records)
│   ├── Access to data outside user's role
│   ├── Access to PII without documented purpose
│   ├── Data access from unusual source
│   └── After-hours PII access
├── Model events:
│   ├── Model deployed without validation approval
│   ├── Model performance below threshold
│   ├── Prediction volume anomaly
│   ├── Model input data quality alert
│   └── Model serving errors spike
├── AI events:
│   ├── Prompt injection detected
│   ├── PII detected in prompt
│   ├── Content filter triggered
│   ├── Hallucination rate spike
│   └── AI API error rate spike
└── System events:
    ├── Configuration change without ticket
    ├── Backup failure
    ├── Service availability degradation
    ├── Disk space below threshold
    └── Memory/CPU anomaly
```

## Log Review

### Daily Review (Automated)

```
DAILY REVIEW CHECKLIST:
├── Security event summary
├── Authentication failure trends
├── Data access anomalies
├── Model performance status
├── AI system health
├── System availability
├── Log volume anomalies
└── Alert follow-up status
```

### Quarterly Review (Manual)

```
QUARTERLY REVIEW CHECKLIST:
├── Log coverage assessment (are we logging everything required?)
├── Log format compliance (are logs in the correct format?)
├── Log retention compliance (are logs retained for required periods?)
├── Log integrity verification (have we verified log chain integrity?)
├── Access to logs (who has access, is it appropriate?)
├── Alert tuning (are alerts effective, too noisy, missing things?)
├── Log storage capacity planning
└── Gap analysis against regulatory requirements
```

## AI-Specific Logging Considerations

### Prompt Logging

```
PROMPT LOGGING POLICY:
├── Log metadata only (timestamp, user, model, tokens)
├── Do NOT log raw prompt text (may contain PII)
├── If logging prompts for debugging:
│   ├── Redact PII before logging
│   ├── Apply same retention as PII
│   ├── Restrict access to authorized personnel
│   └── Include in DSAR fulfillment
├── Log safety filter results
├── Log injection detection results
└── Hash prompts for deduplication (one-way, no reversal)
```

### Model Input/Output Logging

```
MODEL I/O LOGGING:
├── Input: Log feature statistics, not raw data
│   ├── Feature count, types
│   ├── Missing value count
│   ├── Out-of-range count
│   └── Input hash for correlation
├── Output: Log prediction, confidence, metadata
│   ├── Prediction value
│   ├── Confidence score
│   ├── Model version
│   ├── Processing latency
│   └── Output hash for correlation
└── Correlation: Use correlation IDs to link input to output
    ├── Request ID -> Input hash -> Output hash
    └── Enables tracing without exposing raw data
```

### Vector Database Logging

```
VECTOR DB LOGGING:
├── Log all queries (metadata: user, timestamp, collection, result count)
├── Log all index modifications (user, timestamp, operation, document count)
├── Log access control decisions (allow/deny, user, collection)
├── Do NOT log vector embeddings (they may encode sensitive data)
├── Log data source lineage (which documents were embedded)
├── Log retention policy compliance (auto-delete old entries)
└── Log collection-level access separately from query-level access
```

## Common Interview Questions

### Question 1: "What should you log for an AI system, and what should you not log?"

**Good answer structure**:
Log all metadata: timestamps, user IDs, model versions, input/output statistics, safety filter results, and access decisions. Do NOT log raw data that contains PII, secrets, or sensitive content -- this includes full prompts, full AI responses, and raw vector embeddings. If you need to log content for debugging, redact PII first and apply the same retention and access controls as the source data. Use hashes for correlation and deduplication.

### Question 2: "How do you ensure audit logs are tamper-proof?"

**Good answer structure**:
1. Write-once storage (WORM, S3 Object Lock, append-only streams)
2. Cryptographic hash chains (each entry includes hash of previous entry)
3. Digital signatures on log entries (HMAC or asymmetric signatures)
4. Separate permissions (log writers cannot delete or modify)
5. Regular integrity verification (verify hash chains and signatures)
6. Ship logs to centralized, access-controlled storage
7. Monitor for log gaps (missing entries indicate tampering)

### Question 3: "How long should you retain audit logs?"

**Good answer**:
It depends on the log type and regulatory requirements. Security and authentication logs: minimum 3 years per PCI-DSS. Model-related logs: 5 years to support model validation and SR 11-7 compliance. Incident-related logs: 7 years for legal hold. The longest requirement governs. I recommend standardizing on the longest applicable requirement (7 years for banking) with 90 days immediately available for investigation and the rest in cold storage.
