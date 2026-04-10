# API Design for Banking GenAI Platforms

## Why API Design Matters

In a global banking context, REST APIs are the primary contract between services. A poorly designed API creates cascading maintenance costs, security vulnerabilities, and integration friction. For a GenAI platform serving millions of transactions, API design directly impacts developer velocity, system reliability, and regulatory compliance.

## Core REST API Design Principles

### Resource-Oriented Design

```
# Correct - resource nouns
GET    /v1/accounts/{id}/transactions
POST   /v1/payments
PUT    /v1/customers/{id}
DELETE /v1/sessions/{id}

# Incorrect - action verbs
GET    /v1/getAccountTransactions
POST   /v1/createPayment
POST   /v1/updateCustomer
```

### HATEOAS for Banking Workflows

Hypermedia As The Engine Of Application State guides clients through valid state transitions:

```json
{
  "account_id": "ACC-001",
  "balance": 15000.00,
  "currency": "USD",
  "_links": {
    "self": { "href": "/v1/accounts/ACC-001" },
    "transactions": { "href": "/v1/accounts/ACC-001/transactions" },
    "transfer": {
      "href": "/v1/accounts/ACC-001/transfers",
      "method": "POST",
      "template": {
        "to_account": "...",
        "amount": "...",
        "currency": "..."
      }
    }
  }
}
```

### Idempotency

Critical for banking operations where duplicate requests must not cause duplicate effects:

```python
# Idempotency via request key
POST /v1/payments
Headers:
  Idempotency-Key: req-7f3a2b1c-uuid

# Server stores key -> response mapping
# Subsequent requests with same key return cached response
# Key expires after 24 hours (banking SLA)
```

### Conditional Requests for Concurrency

```
GET /v1/accounts/ACC-001
Response:
  ETag: "v1-abc123"
  Last-Modified: Thu, 10 Apr 2026 08:00:00 GMT

PUT /v1/accounts/ACC-001
Headers:
  If-Match: "v1-abc123"

# Returns 412 Precondition Failed if version mismatch
# Prevents lost-update problem in concurrent modifications
```

## API Versioning Strategy

### URL Path Versioning (Primary)

```
/v1/accounts      # Current stable
/v2/accounts      # New version with breaking changes
```

**Rules:**
- Major versions increment on breaking changes (field removal, type change, auth change)
- Non-breaking changes (new fields, new endpoints) go into same version
- Each version supported for minimum 12 months per banking compliance
- Deprecation communicated via `Sunset` header

### Header-Based Versioning (Alternative)

```
GET /accounts
Headers:
  API-Version: 2026-04-10
  Accept: application/vnd.bank.v2+json
```

### Version Transition Pattern

```
Phase 1 (Month 0-3):  v2 available, v1 fully supported
Phase 2 (Month 3-9):  v2 recommended, v1 warning headers added
Phase 3 (Month 9-12): v1 deprecation notices, migration deadlines
Phase 4 (Month 12+):  v1 sunset, returns 410 Gone
```

## Pagination

### Cursor-Based Pagination (Recommended)

```python
# Request
GET /v1/transactions?limit=50&cursor=eyJpZCI6MTIzNDV9

# Response
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIzOTV9",
    "has_more": true,
    "limit": 50
  }
}
```

**Why cursor over offset:**
- Consistent performance regardless of dataset size
- No duplicate/missing records on inserts during pagination
- Safe for real-time banking transaction streams
- O(1) lookup vs O(n) offset scan

### Offset Pagination (Legacy Support)

```python
# Only for admin dashboards, not API-first
GET /v1/transactions?page=1&per_page=25

# Response includes metadata
{
  "data": [...],
  "pagination": {
    "page": 1,
    "per_page": 25,
    "total": 1500,
    "total_pages": 60
  }
}
```

**Anti-pattern:** Never use offset for APIs serving >100k records. Performance degrades linearly.

## Filtering and Sorting

```python
# Standard query parameters
GET /v1/transactions?
  status=completed&
  date_from=2026-01-01&
  date_to=2026-04-10&
  min_amount=1000&
  sort=-created_at&
  fields=id,amount,currency,created_at
```

**Security constraint:** Maximum 100 records per response. Server enforces hard limit regardless of client `limit` parameter. Regulatory requirement for audit trail completeness.

## Error Handling

### Standard Error Response

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Account balance insufficient for requested transfer",
    "details": {
      "account_id": "ACC-001",
      "available_balance": 5000.00,
      "requested_amount": 7500.00,
      "currency": "USD"
    },
    "request_id": "req-7f3a2b1c",
    "documentation_url": "https://docs.bank.api/errors/INSUFFICIENT_FUNDS"
  }
}
```

### HTTP Status Code Mapping

| Status | Usage | Banking Example |
|--------|-------|----------------|
| 400 | Validation failure | Invalid account number format |
| 401 | Authentication failure | Expired OAuth token |
| 403 | Authorization failure | User lacks transfer permission |
| 404 | Resource not found | Account does not exist |
| 409 | Conflict | Duplicate idempotency key |
| 422 | Business rule violation | Daily transfer limit exceeded |
| 429 | Rate limited | Too many requests |
| 500 | Internal server error | Database connection pool exhausted |
| 502 | Bad gateway | Downstream service unavailable |
| 503 | Service unavailable | Scheduled maintenance |

### Error Code Taxonomy

```
VALIDATION_ERROR     - Input validation failed
AUTHENTICATION_ERROR - Token invalid or expired
AUTHORIZATION_ERROR  - Permission denied
NOT_FOUND_ERROR      - Resource does not exist
CONFLICT_ERROR       - Version mismatch, duplicate
BUSINESS_ERROR       - Business rule violation
RATE_LIMIT_ERROR     - Too many requests
INTERNAL_ERROR       - Unexpected server error
DEPENDENCY_ERROR     - Downstream service failure
```

## Banking-Specific API Patterns

### Payment Initiation Flow

```python
# Step 1: Create payment intent
POST /v1/payments/intents
{
  "amount": 5000.00,
  "currency": "USD",
  "recipient_account": "GB29NWBK60161331926819",
  "reference": "Invoice INV-2026-0042",
  "execution_date": "2026-04-11"
}

# Step 2: Confirm with strong customer authentication
POST /v1/payments/intent/{intent_id}/confirm
{
  "sca_token": "eyJhbGc..."
}

# Step 3: Poll status (or use webhook)
GET /v1/payments/{payment_id}/status

# Webhook notification
POST /webhooks/payment-status
{
  "event": "payment.completed",
  "payment_id": "PAY-001",
  "status": "settled",
  "settled_at": "2026-04-11T09:15:00Z"
}
```

### Bulk Operations

```python
# Bulk transfer with partial failure handling
POST /v1/payments/batch
{
  "transfers": [
    {"to": "ACC-002", "amount": 1000, "reference": "Salary"},
    {"to": "ACC-003", "amount": 2500, "reference": "Vendor"},
    {"to": "ACC-999", "amount": 500, "reference": "Refund"}
  ]
}

# Response - partial success
{
  "batch_id": "BATCH-001",
  "total": 3,
  "succeeded": 2,
  "failed": 1,
  "results": [
    {"index": 0, "payment_id": "PAY-001", "status": "accepted"},
    {"index": 1, "payment_id": "PAY-002", "status": "accepted"},
    {"index": 2, "error": "ACCOUNT_NOT_FOUND", "status": "rejected"}
  ]
}
```

## GenAI-Specific API Patterns

### Streaming Completions

```python
# Server-Sent Events for GenAI token streaming
GET /v1/ai/chat/completions?stream=true

# SSE Response
event: chunk
data: {"role": "assistant", "content": "The", "token_index": 0}

event: chunk
data: {"role": "assistant", "content": " regulatory", "token_index": 1}

event: done
data: {"finish_reason": "stop", "usage": {"tokens": 145}}
```

### Long-Running AI Tasks

```python
# Asynchronous AI processing
POST /v1/ai/document-analysis
{
  "document_url": "s3://docs/kyc-form-001.pdf",
  "analysis_type": "kyc_extraction"
}

Response: 202 Accepted
{
  "job_id": "JOB-ai-001",
  "status": "processing",
  "status_url": "/v1/jobs/JOB-ai-001",
  "estimated_completion": "2026-04-10T08:05:00Z"
}

# Poll for result
GET /v1/jobs/JOB-ai-001
{
  "job_id": "JOB-ai-001",
  "status": "completed",
  "result": {
    "extracted_fields": {
      "customer_name": "John Doe",
      "account_number": "ACC-001",
      "risk_score": 0.15
    }
  }
}
```

## Security Considerations

### Authentication

```python
# OAuth 2.0 + mTLS for banking APIs
Headers:
  Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
  X-Client-Cert-Fingerprint: SHA256:a1b2c3d4...

# Token scope enforcement
# Tokens must include required scopes for endpoint access
# { "scope": "accounts:read payments:write" }
```

### Rate Limiting

```python
# Headers for rate limit transparency
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1712736000
X-RateLimit-Window: 3600

# Retry-After for 429 responses
Retry-After: 120
```

### Request/Response Signing

```python
# HMAC signature for webhook delivery
Headers:
  X-Webhook-Signature: sha256=abc123...
  X-Webhook-Timestamp: 1712736000

# Server verifies signature using shared secret
# Rejects requests with timestamp > 5 minutes
```

### Data Masking in API Responses

```python
# Sensitive data masking
{
  "account_number": "****6819",
  "card_number": "****-****-****-1234",
  "ssn": "***-**-6789"
}
```

## Performance Best Practices

### Response Time Budgets

| Endpoint Type | Target P99 | Example |
|--------------|-----------|---------|
| Read (cached) | 50ms | Account balance lookup |
| Read (uncached) | 200ms | Transaction history |
| Write (simple) | 500ms | Update customer profile |
| Write (complex) | 2000ms | Initiate payment |
| AI inference | 10000ms | Document analysis |

### Compression

```
Accept-Encoding: gzip, br
Content-Encoding: br  # Brotli for JSON responses
# Minimum 1KB response size for compression
```

## Common Mistakes and Anti-Patterns

1. **Leaky Abstractions**: Exposing database IDs, internal error stack traces, or table names in API responses
2. **Missing Pagination**: Returning unbounded result sets that crash clients and databases
3. **Inconsistent Naming**: Mixing `snake_case` and `camelCase` across endpoints
4. **No Idempotency**: POST endpoints without idempotency keys for retry safety
5. **Over-fetching**: Returning full objects when clients only need specific fields
6. **Ignoring Content Negotiation**: Not supporting `Accept` header for response format
7. **Silent Failures**: Returning 200 OK with error details in body instead of proper status codes

## Interview Questions

1. How do you design an API that supports both simple CRUD and complex banking workflows?
2. Explain the trade-offs between cursor-based and offset-based pagination for transaction history.
3. How would you implement idempotency for a payment API that processes $10M+ daily?
4. What happens when a client retries a POST request without an idempotency key?
5. How do you handle API versioning when 500+ client applications depend on your service?
6. Design an error handling strategy that distinguishes between transient and permanent failures.
7. How would you implement rate limiting that is fair but also protects against abuse?

## Cross-References

- [[resilience-patterns/README.md]] - Retry and circuit breaker patterns for API calls
- [[queues-and-events/README.md]] - Event-driven API communication
- [[caching/README.md]] - API response caching strategies
- [[microservices/README.md]] - Inter-service API communication patterns
- `../python/fastapi.md` - FastAPI-specific API design
- `../go/grpc.md` - gRPC design for backend services
