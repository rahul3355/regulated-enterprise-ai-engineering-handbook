# Skill: Secure API Design

## Core Principles

1. **Trust Nothing** — Every input is untrusted. Every request must be authenticated and authorized.
2. **Least Privilege** — Every endpoint exposes the minimum data and functionality needed.
3. **Defense in Depth** — Multiple layers of security (network, auth, validation, logging).
4. **Fail Securely** — Errors never leak sensitive information or system internals.

## Mental Models

### The API Security Checklist
```
□ Authentication required (OAuth2/OIDC, never custom auth)
□ Authorization enforced per-endpoint (RBAC/ABAC)
□ Input validated (type, length, format, range)
□ Output sanitized (no PII, no stack traces)
□ Rate limiting applied (per user, per endpoint)
□ CORS configured (specific origins, not wildcard)
□ HTTPS enforced (TLS 1.2+, no HTTP)
□ Secrets from Vault (never env vars or code)
□ Audit logging on sensitive operations
□ Dependencies scanned for vulnerabilities
□ SQL injection prevented (parameterized queries)
□ XSS prevented (output encoding)
□ CSRF protection on state-changing operations
□ Request size limited (prevent DoS)
□ Timeouts configured (prevent hanging requests)
```

## Step-by-Step Approach

### 1. Define the Contract
```python
# Good: Explicit request/response schemas
from pydantic import BaseModel, Field, validator

class CreateDocumentRequest(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    content: str = Field(min_length=1, max_length=50000)
    classification: str = Field(pattern="^(public|internal|confidential|restricted)$")
    
    @validator('content')
    def no_pii(cls, v):
        if detect_pii(v):
            raise ValueError("Content contains PII")
        return v
```

### 2. Enforce Authentication
```python
from fastapi import Depends, HTTPException
from app.auth import verify_token

@router.post("/documents")
async def create_document(
    request: CreateDocumentRequest,
    current_user: User = Depends(verify_token),
):
    # current_user is None if token is invalid — 401 returned automatically
    ...
```

### 3. Enforce Authorization
```python
from app.auth import require_scope

@router.post("/documents")
async def create_document(
    request: CreateDocumentRequest,
    current_user: User = Depends(verify_token),
):
    require_scope(current_user, "documents.write")
    # Returns 403 if user lacks the required scope
    ...
```

### 4. Apply Rate Limiting
```python
from app.ratelimit import rate_limit

@router.post("/documents")
@rate_limit(key="user_id", calls=50, window_seconds=3600)
async def create_document(...):
    ...
```

### 5. Handle Errors Securely
```python
try:
    doc = await db.create_document(request, current_user)
except ValidationError as e:
    # Safe — tells client what's wrong with their request
    raise HTTPException(400, detail=str(e))
except DatabaseError as e:
    # Safe — never expose internal error details
    logger.exception("Database error creating document")
    raise HTTPException(500, detail="Internal server error")
```

## Common Pitfalls

| Pitfall | Risk | Fix |
|---------|------|-----|
| Trusting user IDs from request body | Horizontal privilege escalation | Use auth token identity only |
| No input length limits | DoS via massive payloads | Enforce max_length on all fields |
| Returning full error objects | Leaking stack traces, SQL queries | Return generic error messages |
| Wildcard CORS (`*`) | Any website can call your API | Specify exact allowed origins |
| Secrets in environment variables | Leaked via logs, debug endpoints | Use Vault or K8s Secrets with encryption |
| No request timeouts | Resource exhaustion via hanging requests | Set timeout on all downstream calls |

## Banking-Specific Concerns

1. **PII Handling** — Never log PII. Validate and redact PII in inputs and outputs.
2. **Audit Requirements** — Every data access logged with user ID, timestamp, resource.
3. **Data Classification** — API must enforce document classification access rules.
4. **Regulatory APIs** — Some endpoints may have specific regulatory requirements (GDPR Article 15 access requests, etc.).
5. **GenAI Considerations** — If API feeds AI responses, ensure output safety and grounding.

## GenAI-Specific Concerns

1. **Prompt Injection via API Input** — User input in prompts must be isolated from system instructions.
2. **Token Cost Control** — Rate limit must account for token costs, not just request counts.
3. **Model API Key Protection** — Model API keys must never be exposed to API consumers.
4. **Response Sanitization** — AI responses must be scanned for PII before returning to clients.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Authentication failure rate | > 5% over 5 min | Possible credential stuffing attack |
| Authorization failure rate | > 1% over 5 min | Possible privilege escalation attempt |
| Rate limit trigger rate | > 10% of requests | Possible abuse or misconfigured client |
| 5xx error rate | > 1% over 5 min | Service degradation |
| Input validation failure rate | > 5% over 5 min | Possible attack or client bug |
| Average response size | > 2x baseline | Possible data exfiltration |

## Interview Questions

1. How do you prevent SQL injection in Python APIs?
2. What is the difference between authentication and authorization?
3. How would you design rate limiting for a GenAI API?
4. What CORS configuration is appropriate for an internal banking API?
5. How do you securely store and rotate API keys?
6. What input validation would you apply to a chat endpoint?
7. How do you prevent a user from accessing another user's data?
8. What is the principle of least privilege and how does it apply to APIs?

## Hands-On Exercise

### Exercise: Secure a RAG Retrieval API

**Problem:** The following endpoint has multiple security issues. Identify and fix them.

```python
@router.post("/api/search")
async def search(request: dict):
    query = request.get("query", "")
    results = vector_db.search(query, top_k=10)
    return {"results": results}
```

**Constraints:**
- The API is used by authenticated bank employees
- Results may contain sensitive documents
- Must comply with audit logging requirements
- Must prevent abuse (rate limiting, input validation)

**Expected Output:**
- Pydantic request/response models with validation
- Authentication and authorization
- Input validation and sanitization
- Rate limiting
- Audit logging
- Secure error handling

**Hint:** See `agents/staff-backend-engineer-agent.md` for a complete production-grade example.

**Extension:** Add document-level access control — users can only see documents matching their role.

---

**Related files:**
- `security/api-security.md` — Full API security guide
- `security/authn-and-authz.md` — Authentication and authorization
- `backend-engineering/api-design/` — API design patterns
- `skills/secure-coding.md` — Secure coding practices
