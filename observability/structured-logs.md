# Structured Log Format and Field Conventions

## Log Format Specification

All production services in the banking GenAI platform MUST emit logs in JSON format. This is non-negotiable. JSON enables machine parsing, schema validation, and efficient storage.

### Base Schema

Every log entry conforms to this schema:

```json
{
  "level": "string",
  "timestamp": "ISO8601",
  "service": "string",
  "version": "string",
  "environment": "string",
  "region": "string",
  "correlation_id": "string",
  "span_id": "string",
  "trace_id": "string",
  "message": "string",
  "logger": "string",
  "thread": "string",
  "extra": {}
}
```

### Field Details

#### level (required)

Allowed values: `FATAL`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`

Case-sensitive. Always uppercase.

#### timestamp (required)

ISO 8601 format with timezone, always UTC:

```
"2025-03-15T14:32:01.234Z"
```

Never use local time. Never omit the timezone. Include millisecond precision minimum.

#### service (required)

The canonical service name as registered in the service catalog. Use kebab-case:

```
"loan-advisor-api"
"document-parser"
"llm-gateway"
"rag-retrieval-service"
```

Do NOT use hostnames, pod names, or instance IDs.

#### version (required)

The deployed version of the service, typically the Git short SHA or semantic version:

```
"a3f2c8d"
"2.4.1"
```

#### correlation_id (required)

The unique identifier for the request chain. Format: `req-{12 hex chars}`

```
"req-7f8a9b2c3d4e"
```

#### span_id (recommended)

OpenTelemetry span ID for the current operation. 16 hex characters.

```
"5bd639e71c2a4f89"
```

#### trace_id (recommended)

OpenTelemetry trace ID for the full request. 32 hex characters.

```
"4bf92f3577b34da6a3ce929d0e0e4736"
```

#### message (required)

A concise, human-readable description of what happened. Write for an engineer who has never seen this code.

Good:
```json
{"message": "LLM response exceeded timeout, fallback to cached response"}
```

Bad:
```json
{"message": "Error occurred"}
{"message": "Something went wrong in the process"}
{"message": "Failed"}
```

## Contextual Fields by Domain

Different operational domains require different contextual fields. Add these as top-level fields, NOT nested in `extra`.

### HTTP Request/Response Logging

```json
{
  "level": "INFO",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "loan-advisor-api",
  "message": "HTTP request completed",
  "http_method": "POST",
  "http_url": "/api/v1/mortgage-advice",
  "http_status_code": 200,
  "http_request_size_bytes": 1245,
  "http_response_size_bytes": 3892,
  "duration_ms": 3421,
  "user_agent": "banking-mobile/4.2.1",
  "client_ip_hash": "a3f2c8d"
}
```

**Note**: `client_ip_hash` is a salted hash of the client IP for correlation without storing raw IPs.

### LLM Call Logging

```json
{
  "level": "INFO",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "llm-gateway",
  "message": "LLM call completed",
  "model": "gpt-4-turbo",
  "provider": "openai",
  "tokens_prompt": 1245,
  "tokens_completion": 456,
  "tokens_total": 1701,
  "cost_usd": 0.0423,
  "duration_ms": 2890,
  "temperature": 0.1,
  "max_tokens": 2048,
  "finish_reason": "stop",
  "streaming": false,
  "cache_hit": false,
  "retry_count": 0
}
```

**Critical**: Never log the actual prompt or response content in production. See [Prompt Logging](prompt-logging.md) for safe logging practices.

### RAG Pipeline Logging

```json
{
  "level": "INFO",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "rag-pipeline",
  "message": "RAG retrieval completed",
  "query_embedding_model": "text-embedding-3-large",
  "embedding_dimension": 3072,
  "embedding_time_ms": 45,
  "vector_db": "pgvector",
  "vector_db_query_time_ms": 123,
  "documents_retrieved": 12,
  "documents_above_threshold": 8,
  "reranker_model": "bge-reranker-large",
  "reranking_time_ms": 67,
  "final_documents_count": 5,
  "similarity_threshold": 0.72,
  "max_documents": 10
}
```

### Database Query Logging

```json
{
  "level": "INFO",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "customer-profile-service",
  "message": "Database query completed",
  "db_system": "postgresql",
  "db_operation": "SELECT",
  "db_table": "customer_profiles",
  "db_rows_affected": 1,
  "db_duration_ms": 12,
  "db_connection_pool_available": 8,
  "db_connection_pool_total": 20
}
```

### Authentication/Authorization Logging

```json
{
  "level": "INFO",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "auth-service",
  "message": "Authentication completed",
  "auth_method": "oauth2",
  "auth_provider": "okta-bank-sso",
  "auth_result": "success",
  "auth_duration_ms": 234,
  "user_tier": "premium",
  "permissions_granted": ["read:accounts", "read:transactions"],
  "token_remaining_ttl_seconds": 342
}
```

## Field Naming Conventions

1. **Use snake_case**: `http_status_code`, not `httpStatusCode` or `HttpStatusCode`
2. **Use dot notation for namespacing**: `http.method`, `db.operation`
3. **Use unit suffixes for measurements**: `duration_ms`, `size_bytes`, `cost_usd`
4. **Use boolean suffixes for flags**: `cache_hit`, `streaming`, `is_retry`
5. **Use `_count` suffix for counters**: `documents_retrieved`, `tokens_prompt`

## Field Type Conventions

| Type | JSON Type | Example | Notes |
|------|-----------|---------|-------|
| String | string | `"gpt-4"` | Always quoted |
| Integer | number | `245` | No decimal point |
| Float | number | `0.0423` | Use decimal notation |
| Boolean | boolean | `true` | Not `"true"` or `1` |
| Array | array | `["read:accounts"]` | Typed arrays only |
| Object | object | `{"key": "value"}` | Nested structured data |
| Null | null | `null` | Use instead of omitting |

## Log Field Restrictions

### Fields That Must Be Redacted Before Logging

```python
REDACTED_FIELDS = {
    "password", "secret", "api_key", "token", "authorization",
    "cookie", "session_id", "credit_card", "card_number", "cvv",
    "ssn", "social_security", "account_number", "routing_number",
    "email", "phone", "full_name", "address", "date_of_birth"
}

def redact_sensitive_fields(log_entry: dict) -> dict:
    """Redact sensitive fields from a log entry."""
    sanitized = {}
    for key, value in log_entry.items():
        if key.lower() in REDACTED_FIELDS:
            sanitized[key] = "[REDACTED]"
        elif isinstance(value, str) and any(
            pattern in value.lower() for pattern in
            ["password", "secret", "token", "key", "authorization"]
        ):
            sanitized[key] = "[REDACTED]"
        else:
            sanitized[key] = value
    return sanitized
```

## Log Validation

Implement schema validation in your logging pipeline to catch non-conforming entries:

```python
import jsonschema

LOG_SCHEMA = {
    "type": "object",
    "required": ["level", "timestamp", "service", "version",
                 "correlation_id", "message"],
    "properties": {
        "level": {"enum": ["FATAL", "ERROR", "WARN", "INFO", "DEBUG", "TRACE"]},
        "timestamp": {"type": "string", "format": "date-time"},
        "service": {"type": "string", "pattern": "^[a-z0-9-]+$"},
        "version": {"type": "string"},
        "correlation_id": {"type": "string", "pattern": "^req-[a-f0-9]{12}$"},
        "message": {"type": "string", "minLength": 1},
        "duration_ms": {"type": "number", "minimum": 0}
    },
    "additionalProperties": True
}

def validate_log_entry(entry: dict) -> bool:
    """Validate a log entry against the schema."""
    try:
        jsonschema.validate(instance=entry, schema=LOG_SCHEMA)
        return True
    except jsonschema.ValidationError as e:
        # Log the validation error itself (meta!)
        print(f"Log validation failed: {e.message}")
        return False
```

## Performance Considerations

Logging has performance overhead. In a banking GenAI system where LLM calls already add latency, logging must be efficient:

1. **Asynchronous logging**: Write to a buffer, flush in background threads
2. **Batching**: Send logs in batches of 100-1000 entries, not individually
3. **Sampling**: Apply sampling rates by log level (100% for ERROR, 1% for INFO)
4. **Avoid synchronous disk I/O**: Use memory-mapped files or network buffers

```python
import logging
import json
from concurrent.futures import ThreadPoolExecutor
from queue import Queue

class AsyncJSONHandler(logging.Handler):
    """Asynchronous JSON log handler with batching."""

    def __init__(self, queue_size=10000, batch_size=100):
        super().__init__()
        self.queue = Queue(maxsize=queue_size)
        self.batch_size = batch_size
        self.executor = ThreadPoolExecutor(max_workers=2)
        self.executor.submit(self._flush_loop)

    def emit(self, record):
        try:
            log_entry = self.format(record)
            self.queue.put_nowait(log_entry)
        except Exception:
            self.handleError(record)

    def _flush_loop(self):
        while True:
            batch = []
            while len(batch) < self.batch_size:
                try:
                    batch.append(self.queue.get(timeout=1))
                except:
                    break
            if batch:
                self._send_batch(batch)
```

## Common Anti-Patterns

1. **Nested JSON in message field**: Do not put structured data in the message string. Use top-level fields.

2. **Inconsistent timestamp formats**: Enforce ISO 8601 with UTC timezone. Reject anything else.

3. **Logging the same event at multiple levels**: A single event produces one log entry at one level.

4. **Mutable objects in log context**: If you log a mutable dict and then modify it, the log entry may change before it is written.

5. **Missing service version**: Without version, you cannot correlate log patterns with deployments.
