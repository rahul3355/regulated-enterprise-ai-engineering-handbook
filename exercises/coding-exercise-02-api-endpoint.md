# Coding Exercise 02: Secure API Endpoint

> Build a production-grade FastAPI endpoint with authentication, rate limiting, and audit logging — the foundation of every banking GenAI service.

## Problem Statement

You are building a FastAPI endpoint for the internal GenAI assistant used by bank employees. The endpoint must:

1. Accept user queries and return AI-generated responses
2. Authenticate requests using API keys (simulated)
3. Rate-limit requests per user (max 10 requests per minute)
4. Log every request and response for audit purposes
5. Handle errors gracefully without exposing internal details

**Banking Context:** This endpoint will be used by 50,000+ employees. Every query must be auditable for compliance. Rate limiting prevents abuse and cost overruns. Authentication ensures only authorized employees can access the system.

## Constraints

- Use FastAPI (Python 3.11+)
- No external model API call needed — simulate with a mock response
- API keys are stored in a simple dictionary (no database needed)
- Rate limiting must use an in-memory store (Redis not required for this exercise)
- Audit logs must be written to a file in JSON Lines format
- All endpoints must return consistent error responses
- Maximum query length: 4,000 characters
- Response must include request ID for traceability

## Expected Output

```python
# Endpoint: POST /v1/chat
# Request:
{
    "query": "What is our expense reporting policy?",
    "api_key": "sk-employee-001"
}

# Success Response (200):
{
    "request_id": "uuid",
    "response": "According to the company expense policy...",
    "model": "mock-gpt-4",
    "tokens_used": 120,
    "latency_ms": 45
}

# Rate Limited Response (429):
{
    "error": "rate_limit_exceeded",
    "message": "Too many requests. Please try again in 60 seconds.",
    "retry_after": 45
}

# Auth Error (401):
{
    "error": "authentication_failed",
    "message": "Invalid or expired API key."
}

# Validation Error (400):
{
    "error": "validation_failed",
    "message": "Query must be between 1 and 4000 characters.",
    "details": {"query_length": 0}
}
```

## Hints

### Hint 1: Start with the Data Models
```python
from pydantic import BaseModel, Field
from datetime import datetime
import uuid

class ChatRequest(BaseModel):
    query: str = Field(..., min_length=1, max_length=4000)
    api_key: str

class ChatResponse(BaseModel):
    request_id: str
    response: str
    model: str
    tokens_used: int
    latency_ms: int

class ErrorResponse(BaseModel):
    error: str
    message: str
    details: dict | None = None
    retry_after: int | None = None
```

### Hint 2: Use FastAPI Dependencies for Authentication
```python
from fastapi import Depends, HTTPException, Request
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

VALID_API_KEYS = {
    "sk-employee-001": {"user_id": "emp-001", "department": "engineering"},
    "sk-employee-002": {"user_id": "emp-002", "department": "hr"},
}

async def authenticate(api_key: str = Depends(api_key_header)):
    if api_key not in VALID_API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return VALID_API_KEYS[api_key]
```

### Hint 3: Implement Rate Limiting with a Sliding Window
```python
from collections import defaultdict
import time

class RateLimiter:
    def __init__(self, max_requests: int = 10, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests: defaultdict[str, list[float]] = defaultdict(list)

    def is_allowed(self, key: str) -> tuple[bool, int]:
        now = time.time()
        # Remove expired entries
        self.requests[key] = [
            t for t in self.requests[key]
            if now - t < self.window_seconds
        ]
        if len(self.requests[key]) >= self.max_requests:
            retry_after = int(self.window_seconds - (now - self.requests[key][0]))
            return False, retry_after
        self.requests[key].append(now)
        return True, 0
```

### Hint 4: Audit Logging
```python
import json
from pathlib import Path

class AuditLogger:
    def __init__(self, log_path: str = "audit.log"):
        self.log_path = Path(log_path)

    def log(self, event: dict):
        event["timestamp"] = datetime.utcnow().isoformat()
        with open(self.log_path, "a") as f:
            f.write(json.dumps(event) + "\n")
```

## Example Solution

```python
"""
Secure GenAI Chat Endpoint — Banking Context
Builds a production-ready FastAPI endpoint with auth, rate limiting,
and audit logging suitable for a regulated financial environment.
"""

import time
import uuid
import json
from collections import defaultdict
from datetime import datetime
from pathlib import Path

from fastapi import Depends, FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field

app = FastAPI(title="GenAI Chat API", version="1.0.0")

# ─── Data Models ───────────────────────────────────────────────

class ChatRequest(BaseModel):
    query: str = Field(..., min_length=1, max_length=4000, description="User query text")
    api_key: str = Field(..., description="Employee API key")

class ChatResponse(BaseModel):
    request_id: str
    response: str
    model: str = "mock-gpt-4"
    tokens_used: int
    latency_ms: int

class ErrorResponse(BaseModel):
    error: str
    message: str
    details: dict | None = None
    retry_after: int | None = None

# ─── Authentication ────────────────────────────────────────────

VALID_API_KEYS = {
    "sk-employee-001": {"user_id": "emp-001", "department": "engineering", "role": "senior_engineer"},
    "sk-employee-002": {"user_id": "emp-002", "department": "hr", "role": "hr_analyst"},
    "sk-employee-003": {"user_id": "emp-003", "department": "compliance", "role": "compliance_officer"},
}

async def authenticate(api_key: str) -> dict:
    """Validate API key and return user context."""
    if api_key not in VALID_API_KEYS:
        raise HTTPException(
            status_code=401,
            detail={"error": "authentication_failed", "message": "Invalid or expired API key."}
        )
    return VALID_API_KEYS[api_key]

# ─── Rate Limiting ─────────────────────────────────────────────

class RateLimiter:
    """Sliding window rate limiter."""

    def __init__(self, max_requests: int = 10, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests: defaultdict[str, list[float]] = defaultdict(list)

    def is_allowed(self, key: str) -> tuple[bool, int]:
        now = time.time()
        self.requests[key] = [t for t in self.requests[key] if now - t < self.window_seconds]
        if len(self.requests[key]) >= self.max_requests:
            retry_after = int(self.window_seconds - (now - self.requests[key][0])) + 1
            return False, retry_after
        self.requests[key].append(now)
        return True, 0

rate_limiter = RateLimiter(max_requests=10, window_seconds=60)

# ─── Audit Logging ─────────────────────────────────────────────

class AuditLogger:
    """JSON Lines audit logger for compliance."""

    def __init__(self, log_path: str = "audit_chat.log"):
        self.log_path = Path(log_path)

    def log(self, event: dict):
        event["timestamp"] = datetime.utcnow().isoformat() + "Z"
        with open(self.log_path, "a") as f:
            f.write(json.dumps(event, default=str) + "\n")

audit_logger = AuditLogger()

# ─── Mock LLM Response ────────────────────────────────────────

MOCK_RESPONSES = {
    "expense": "According to the company expense policy (v3.2, updated Jan 2026): "
               "Expenses up to $50 per meal do not require receipts. "
               "All expenses over $50 must be submitted with itemized receipts "
               "within 30 days of the expense date. International expenses require "
               "manager approval.",
    "default": "I understand your question. Based on our internal knowledge base, "
               "here's what I found: [This is a mock response for testing. "
               "In production, this would call the LLM API with the user's query.]"
}

def generate_mock_response(query: str) -> tuple[str, int]:
    """Simulate an LLM response with token usage."""
    query_lower = query.lower()
    for keyword, response in MOCK_RESPONSES.items():
        if keyword != "default" and keyword in query_lower:
            return response, len(query.split()) + len(response.split())
    return MOCK_RESPONSES["default"], len(query.split()) + 50

# ─── Endpoint ──────────────────────────────────────────────────

@app.post("/v1/chat", response_model=ChatResponse)
async def chat(request_body: ChatRequest, request: Request):
    request_id = str(uuid.uuid4())
    start_time = time.time()

    # Authenticate
    try:
        user_context = await authenticate(request_body.api_key)
    except HTTPException as e:
        audit_logger.log({
            "event": "chat_request",
            "request_id": request_id,
            "status": "auth_failed",
            "api_key_prefix": request_body.api_key[:8] + "...",
            "query_length": len(request_body.query),
        })
        return JSONResponse(status_code=401, content=e.detail)

    # Rate limit
    rate_key = user_context["user_id"]
    allowed, retry_after = rate_limiter.is_allowed(rate_key)
    if not allowed:
        audit_logger.log({
            "event": "chat_request",
            "request_id": request_id,
            "status": "rate_limited",
            "user_id": rate_key,
            "retry_after": retry_after,
        })
        return JSONResponse(
            status_code=429,
            content={
                "error": "rate_limit_exceeded",
                "message": "Too many requests. Please try again in 60 seconds.",
                "retry_after": retry_after,
            }
        )

    # Generate response
    try:
        response_text, tokens_used = generate_mock_response(request_body.query)
        latency_ms = int((time.time() - start_time) * 1000)

        # Audit log (success)
        audit_logger.log({
            "event": "chat_request",
            "request_id": request_id,
            "status": "success",
            "user_id": user_context["user_id"],
            "department": user_context["department"],
            "query_length": len(request_body.query),
            "response_length": len(response_text),
            "tokens_used": tokens_used,
            "latency_ms": latency_ms,
        })

        return ChatResponse(
            request_id=request_id,
            response=response_text,
            tokens_used=tokens_used,
            latency_ms=latency_ms,
        )

    except Exception as e:
        # Generic error — never expose internals
        audit_logger.log({
            "event": "chat_request",
            "request_id": request_id,
            "status": "error",
            "user_id": user_context["user_id"],
            "error_type": type(e).__name__,
        })
        return JSONResponse(
            status_code=500,
            content={
                "error": "internal_error",
                "message": "An unexpected error occurred. Please try again.",
            }
        )

@app.get("/health")
async def health():
    return {"status": "healthy", "version": "1.0.0"}
```

## Extensions

1. **Add PII detection:** Scan the query for patterns that look like SSNs, credit card numbers, or account numbers. Reject queries containing PII and log the detection.

2. **Add prompt injection detection:** Implement a simple heuristic-based detector that flags queries containing phrases like "ignore previous instructions," "you are now," or system prompt patterns.

3. **Replace in-memory rate limiting with Redis:** Use `redis-py` with `INCR` and `EXPIRE` for distributed rate limiting.

4. **Add request ID propagation:** Accept an `X-Request-ID` header from upstream proxies and use it instead of generating a new UUID.

5. **Add response streaming:** Implement Server-Sent Events (SSE) for streaming responses, similar to how production LLM APIs work.

## Interview Relevance

This exercise tests skills commonly assessed in banking GenAI engineering interviews:

| Skill | Why It Matters |
|-------|---------------|
| API authentication | Every banking API requires auth |
| Rate limiting | Prevents abuse and controls costs |
| Audit logging | Regulatory requirement in banking |
| Error handling | Production systems must fail safely |
| Input validation | Prevents injection attacks and abuse |
| Consistent response formats | API contract compliance |

**Follow-up questions an interviewer might ask:**
- "How would you scale the rate limiter across multiple instances?"
- "What would you change if this needed to handle 10,000 requests/second?"
- "How do you ensure audit logs are not lost if the process crashes?"
- "What security concerns do you see with the audit log file?"
