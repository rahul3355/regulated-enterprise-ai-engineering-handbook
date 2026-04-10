# Structured Logging in Banking GenAI Systems

## Why Structured Logging Matters

In banking, logs serve two equally important purposes: **operational debugging** and **regulatory compliance**. Unstructured text logs fail at both when systems scale. A log line like `"Error processing request"` is useless when you have 50 million log entries and need to find all failures for a specific customer during an audit.

Structured logging means every log entry is a machine-readable data object (typically JSON) with consistent fields, typed values, and a schema that enables querying, aggregation, and automated analysis.

## Log Levels

Use the following log levels consistently across all services. Misuse of log levels is the primary cause of alert fatigue and log storage cost overruns.

### FATAL

The service cannot continue operating. Immediate action required.

```json
{
  "level": "FATAL",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "loan-advisor-api",
  "version": "2.4.1",
  "correlation_id": "req-7f8a9b2c",
  "message": "Database connection pool exhausted, cannot serve any requests",
  "error_type": "ConnectionPoolExhausted",
  "stack_trace": "...",
  "environment": "production",
  "region": "us-east-1"
}
```

**Banking context**: A FATAL log in a transaction processing service triggers an immediate page to on-call and automated failover.

### ERROR

A specific operation failed. The service continues operating. Requires investigation within SLA.

```json
{
  "level": "ERROR",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "document-parser",
  "version": "1.8.0",
  "correlation_id": "req-3c4d5e6f",
  "message": "Failed to parse mortgage document PDF",
  "error_type": "DocumentParseError",
  "document_type": "mortgage_application",
  "document_id": "doc-abc123",
  "file_size_bytes": 2458624,
  "environment": "production",
  "user_tier": "premium"
}
```

**Banking context**: ERROR logs on document parsing feed into daily quality reports. Persistent errors on a specific document type trigger a bug fix.

### WARN

Something unexpected happened but the service recovered. Indicates a potential future problem.

```json
{
  "level": "WARN",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "llm-gateway",
  "correlation_id": "req-9a8b7c6d",
  "message": "LLM response time exceeded threshold, using cached response",
  "threshold_ms": 5000,
  "actual_ms": 8234,
  "model": "gpt-4",
  "cache_hit": true,
  "fallback_used": true
}
```

**Banking context**: Warnings about LLM latency degradation often precede provider-side incidents. Track warning rates to detect trends.

### INFO

Normal operational information. Should describe state changes, not every step.

```json
{
  "level": "INFO",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "rag-pipeline",
  "correlation_id": "req-1a2b3c4d",
  "message": "RAG retrieval completed",
  "documents_retrieved": 8,
  "retrieval_time_ms": 245,
  "vector_db_query_time_ms": 89,
  "embedding_model": "text-embedding-3-large",
  "similarity_threshold": 0.72,
  "documents_above_threshold": 5
}
```

### DEBUG

Detailed information for development and troubleshooting. Never enabled in production banking systems unless actively debugging an incident with change management approval.

```json
{
  "level": "DEBUG",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "auth-service",
  "correlation_id": "req-5e6f7a8b",
  "message": "Token validation details",
  "token_type": "JWT",
  "token_issuer": "okta-bank-sso",
  "claims_valid": true,
  "token_expiry_in_seconds": 342,
  "validation_time_ms": 3
}
```

## Correlation IDs

Every request entering your system must receive a unique correlation ID. This ID must propagate through every service, every log entry, every trace span, and every metric label.

### Generation

```python
import uuid

def generate_correlation_id() -> str:
    """Generate a URL-safe correlation ID."""
    return f"req-{uuid.uuid4().hex[:12]}"
```

### Propagation

The correlation ID must travel in HTTP headers across service boundaries:

```
X-Correlation-ID: req-7f8a9b2c3d4e
```

In a banking GenAI system, the flow looks like:

```
Client ──[X-Correlation-ID: req-abc123]──> API Gateway
                                              │
                                              ├──> Auth Service
                                              ├──> RAG Pipeline
                                              │       ├──> Vector DB
                                              │       └──> Document Store
                                              ├──> LLM Gateway
                                              │       └──> External LLM Provider
                                              └──> Response Formatter
                                                       └──> Client
```

Every service logs with the same correlation ID. Every metric carries it as a label when cardinality allows.

## Required Log Fields

Every log entry in production MUST include:

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `level` | string | Log level (FATAL/ERROR/WARN/INFO/DEBUG) | `"ERROR"` |
| `timestamp` | ISO 8601 | When the event occurred | `"2025-03-15T14:32:01.234Z"` |
| `service` | string | Service name | `"loan-advisor-api"` |
| `version` | string | Service version | `"2.4.1"` |
| `correlation_id` | string | Request correlation ID | `"req-7f8a9b2c"` |
| `message` | string | Human-readable description | `"Document parsing failed"` |

Optional but recommended fields:

| Field | Type | Description |
|-------|------|-------------|
| `environment` | string | `production`, `staging`, `development` |
| `region` | string | Deployment region |
| `error_type` | string | Categorized error type |
| `stack_trace` | string | Stack trace (ERROR/FATAL only) |
| `user_tier` | string | Customer tier (never PII) |
| `duration_ms` | number | Operation duration |

## Fields That Must NEVER Be Logged

In banking, logging the following data is a regulatory violation:

- **PII**: Full name, email, phone, SSN, date of birth
- **PCI**: Full card number, CVV, PIN
- **Account Data**: Account numbers, routing numbers (use hashed references)
- **Authentication**: Passwords, tokens, API keys, session cookies
- **Model Inputs/Outputs**: Raw prompts or responses containing customer financial data (see [Prompt Logging](prompt-logging.md))

Use tokenization or hashing for identifiers that need correlation:

```python
import hashlib

def safe_customer_id(customer_id: str) -> str:
    """Hash customer ID for log correlation without exposing PII."""
    return hashlib.sha256(f"cust-salt-{customer_id}".encode()).hexdigest()[:16]
```

## Sampling Strategies

In high-traffic banking systems, logging every request is expensive. Use sampling:

```python
import random

class LogSampler:
    def __init__(self, error_rate=1.0, warn_rate=0.1, info_rate=0.01):
        self.error_rate = error_rate    # Always log errors
        self.warn_rate = warn_rate      # 10% of warnings
        self.info_rate = info_rate      # 1% of info logs

    def should_log(self, level: str) -> bool:
        rates = {"FATAL": 1.0, "ERROR": 1.0, "WARN": self.warn_rate,
                 "INFO": self.info_rate, "DEBUG": 0.0}
        return random.random() < rates.get(level, 0.0)
```

## Log Aggregation Architecture

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│  Services   │───>│  Log Agent   │───>│  Log Buffer  │
│  (Python)   │    │  (Fluent Bit)│    │  (Kafka)     │
└─────────────┘    └──────────────┘    └──────┬───────┘
                                              │
                                    ┌─────────┴─────────┐
                                    │                   │
                              ┌─────┴─────┐      ┌─────┴─────┐
                              │  Hot      │      │  Cold     │
                              │  Storage  │      │  Storage  │
                              │ (Elastic) │      │ (S3)      │
                              │ 30 days   │      │ 7 years   │
                              └───────────┘      └───────────┘
```

Hot storage enables real-time query and alerting. Cold storage satisfies regulatory retention requirements (7 years for banking audit trails).

## Query Examples

Find all errors for a specific service in the last hour:

```
service:loan-advisor-api AND level:ERROR AND timestamp:>now-1h
```

Find all requests for a correlation ID across all services:

```
correlation_id:req-7f8a9b2c
```

Count errors by type per hour:

```
level:ERROR | stats count by error_type, bin(timestamp, 1h)
```

## Common Mistakes

1. **Logging at INFO what should be DEBUG**: Verbose info logs consume storage and make debugging harder.

2. **Inconsistent field names**: One service uses `user_id`, another uses `userId`, another uses `customerId`. Enforce a schema.

3. **Logging stack traces for expected errors**: A validation failure is not a stack trace situation. Use structured error fields.

4. **Missing correlation IDs at service boundaries**: If a service calls another without propagating the correlation ID, the trace is broken.

5. **Not redacting PII in test environments**: Test environments often use production-like data. The same redaction rules apply.
