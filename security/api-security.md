# API Security

## Overview

APIs are the attack surface of modern banking. Every endpoint is a potential entry point for attackers. This guide covers authentication, authorization, rate limiting, and input validation for production-grade banking APIs that handle billions of dollars in transactions.

## Threat Landscape

### API-Specific Attack Vectors

| Attack | Description | Impact |
|---|---|---|
| Broken Object Level Authorization (BOLA) | Accessing objects by manipulating IDs | Data breach |
| Broken Authentication | Credential stuffing, token theft | Account takeover |
| Broken Object Property Level Authorization | Mass assignment, over-posting | Privilege escalation |
| Unrestricted Resource Consumption | No rate limiting, large payloads | DoS, cost explosion |
| Broken Function Level Authorization | Accessing admin functions as regular user | Full system compromise |
| Unrestricted Access to Sensitive Business Flows | Automating purchases, ticket scalping | Business logic abuse |
| Server Side Request Forgery | Forcing API to make internal requests | Network reconnaissance |
| Security Misconfiguration | Verbose errors, missing headers | Information disclosure |
| Improper Inventory Management | Shadow APIs, dev endpoints in production | Undocumented attack surface |
| Unsafe AI/LLM Consumption | Prompt injection via API | AI manipulation |

*Based on OWASP API Security Top 10:2023*

### Real-World API Breaches

- **T-Mobile (2022)**: Compromised API endpoint exposed data of 37M customers. The attacker exploited an API with no authentication.
- **Optus (2022)**: Unauthenticated API endpoint exposed 9.8M customer records including passport numbers.
- **Venmo (2019)**: API vulnerability allowed anyone to see full transaction history of any user.
- **Samsung (2023)**: API breach exposed personal data of customers due to unauthorized API access.

## API Authentication

### API Keys

```python
# BAD: API key in URL parameter
# GET /api/v1/transactions?api_key=sk_live_abc123

# GOOD: API key in header
# GET /api/v1/transactions
# Authorization: Bearer sk_live_abc123

# API key validation with rate limiting per key
import time
import hashlib
import hmac
from collections import defaultdict

class APIKeyManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.rate_limits = defaultdict(lambda: {"count": 0, "reset_at": 0})

    def validate_key(self, api_key: str) -> dict:
        # Constant-time comparison to prevent timing attacks
        stored_key = self._get_key_hash(api_key)
        if not stored_key or not hmac.compare_digest(api_key, stored_key):
            raise AuthenticationError("Invalid API key")

        # Check if key is revoked or expired
        metadata = self.redis.get(f"apikey:{api_key[:8]}:meta")
        if not metadata or metadata.get("revoked"):
            raise AuthenticationError("API key revoked")

        return metadata

    def _get_key_hash(self, api_key: str) -> str:
        """Store only hashes, never raw keys"""
        prefix = api_key[:8]  # For identification
        return self.redis.get(f"apikey:{prefix}:hash")
```

### API Key Best Practices

```python
# API key format: prefix_hash
# Example: sk_live_a1b2c3d4_abc123...
# Prefix helps with identification and debugging

import secrets
import hashlib

def generate_api_key(prefix: str = "sk_live") -> tuple[str, str]:
    """Returns (display_key, stored_hash)"""
    raw_key = secrets.token_urlsafe(32)
    display_prefix = ''.join(secrets.choice('abcdefghijklmnopqrstuvwxyz0123456789') for _ in range(8))
    display_key = f"{prefix}_{display_prefix}_{raw_key[:4]}"
    stored_hash = hashlib.sha256(raw_key.encode()).hexdigest()
    return display_key, stored_hash

def verify_api_key(provided_key: str, stored_hash: str) -> bool:
    # Extract the raw key portion (after prefix_display_)
    raw_key = provided_key.split("_", 2)[2]  # Get everything after prefix_display_
    computed_hash = hashlib.sha256(raw_key.encode()).hexdigest()
    return hmac.compare_digest(computed_hash, stored_hash)
```

## API Authorization

### Object-Level Authorization (BOLA Prevention)

```python
# BAD: No ownership check
@app.get("/api/transactions/{transaction_id}")
async def get_transaction(transaction_id: str, current_user: User = Depends(get_current_user)):
    transaction = await db.get_transaction(transaction_id)
    if not transaction:
        raise HTTPException(404, "Not found")
    return transaction  # Returns ANY transaction, not just user's

# GOOD: Enforce ownership
@app.get("/api/transactions/{transaction_id}")
async def get_transaction(transaction_id: str, current_user: User = Depends(get_current_user)):
    transaction = await db.get_transaction(transaction_id)
    if not transaction or transaction.owner_id != current_user.id:
        raise HTTPException(404, "Not found")  # Don't leak existence
    return transaction

# MIDDLEWARE APPROACH: Centralized authorization
class AuthorizationMiddleware:
    def __init__(self, policy_engine):
        self.policy_engine = policy_engine

    async def __call__(self, request: Request, call_next):
        user = request.state.user
        resource = self._extract_resource(request)
        action = self._extract_action(request)

        if not self.policy_engine.check(user, action, resource):
            return JSONResponse(
                status_code=403,
                content={"error": "Forbidden", "code": "INSUFFICIENT_PERMISSIONS"}
            )

        return await call_next(request)
```

### Role-Based Access Control (RBAC)

```python
from enum import Enum
from functools import wraps

class Role(Enum):
    CUSTOMER = "customer"
    TELLER = "teller"
    MANAGER = "manager"
    ADMIN = "admin"

class Permission(Enum):
    VIEW_OWN_ACCOUNT = "account:view:own"
    VIEW_ANY_ACCOUNT = "account:view:any"
    TRANSFER_OWN = "transfer:create:own"
    TRANSFER_ANY = "transfer:create:any"
    VIEW_TRANSACTIONS_OWN = "transaction:view:own"
    VIEW_TRANSACTIONS_ANY = "transaction:view:any"
    ADMIN_PANEL = "admin:access"

ROLE_PERMISSIONS = {
    Role.CUSTOMER: {
        Permission.VIEW_OWN_ACCOUNT,
        Permission.TRANSFER_OWN,
        Permission.VIEW_TRANSACTIONS_OWN,
    },
    Role.TELLER: {
        Permission.VIEW_OWN_ACCOUNT,
        Permission.VIEW_ANY_ACCOUNT,
        Permission.VIEW_TRANSACTIONS_OWN,
        Permission.VIEW_TRANSACTIONS_ANY,
    },
    Role.MANAGER: {
        Permission.VIEW_OWN_ACCOUNT,
        Permission.VIEW_ANY_ACCOUNT,
        Permission.TRANSFER_OWN,
        Permission.TRANSFER_ANY,
        Permission.VIEW_TRANSACTIONS_OWN,
        Permission.VIEW_TRANSACTIONS_ANY,
    },
    Role.ADMIN: {
        Permission.VIEW_OWN_ACCOUNT,
        Permission.VIEW_ANY_ACCOUNT,
        Permission.TRANSFER_OWN,
        Permission.TRANSFER_ANY,
        Permission.VIEW_TRANSACTIONS_OWN,
        Permission.VIEW_TRANSACTIONS_ANY,
        Permission.ADMIN_PANEL,
    },
}

def require_permission(permission: Permission):
    def decorator(f):
        @wraps(f)
        async def decorated_function(*args, **kwargs):
            user = kwargs.get("current_user") or get_current_user()
            user_permissions = ROLE_PERMISSIONS.get(user.role, set())
            if permission not in user_permissions:
                raise HTTPException(403, "Insufficient permissions")
            return await f(*args, **kwargs)
        return decorated_function
    return decorator

@app.post("/api/transfers")
@require_permission(Permission.TRANSFER_OWN)
async def create_transfer(request: TransferRequest, current_user: User):
    return await process_transfer(request, current_user)
```

## Rate Limiting

### Token Bucket Algorithm

```python
import time
from dataclasses import dataclass
import threading

@dataclass
class TokenBucket:
    capacity: int        # Maximum tokens
    refill_rate: float   # Tokens per second
    tokens: float        # Current tokens
    last_refill: float   # Last refill timestamp

class RateLimiter:
    def __init__(self):
        self.buckets: dict[str, TokenBucket] = {}
        self.lock = threading.Lock()

    def allow_request(self, client_id: str, tokens: int = 1) -> bool:
        with self.lock:
            now = time.time()

            if client_id not in self.buckets:
                self.buckets[client_id] = TokenBucket(
                    capacity=100,
                    refill_rate=10,  # 10 tokens/second
                    tokens=100,
                    last_refill=now,
                )

            bucket = self.buckets[client_id]

            # Refill tokens
            elapsed = now - bucket.last_refill
            bucket.tokens = min(
                bucket.capacity,
                bucket.tokens + elapsed * bucket.refill_rate
            )
            bucket.last_refill = now

            # Check if enough tokens available
            if bucket.tokens >= tokens:
                bucket.tokens -= tokens
                return True
            return False

# Usage in middleware
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_id = request.client.host
    if not limiter.allow_request(client_id):
        return JSONResponse(
            status_code=429,
            headers={
                "Retry-After": "10",
                "X-RateLimit-Limit": "100",
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(int(time.time()) + 10),
            },
            content={"error": "Rate limit exceeded"}
        )
    return await call_next(request)
```

### Tiered Rate Limiting for Banking APIs

```yaml
# Rate limits by endpoint tier
rate_limits:
  public:               # Health checks, docs
    requests_per_minute: 60
    burst: 10

  authenticated:        # Standard API calls
    requests_per_minute: 300
    burst: 50

  financial:            # Transfers, payments
    requests_per_minute: 30
    burst: 5
    additional_controls:
      - daily_limit: 50
      - max_amount_per_request: 10000
      - cooldown_between_transfers: 5s

  admin:                # Administrative endpoints
    requests_per_minute: 60
    burst: 10
    ip_whitelist_required: true
```

## Input Validation

### Request Validation Pipeline

```python
from pydantic import BaseModel, Field, validator
from typing import Optional
from enum import Enum

class Currency(str, Enum):
    USD = "USD"
    EUR = "EUR"
    GBP = "GBP"

class TransferRequest(BaseModel):
    from_account: str = Field(
        ...,
        min_length=12,
        max_length=12,
        pattern=r"^ACC\d{9}$",
        description="Source account number"
    )
    to_account: str = Field(
        ...,
        min_length=12,
        max_length=12,
        pattern=r"^ACC\d{9}$",
        description="Destination account number"
    )
    amount: float = Field(
        ...,
        gt=0,
        le=100000,
        description="Transfer amount (max $100,000)"
    )
    currency: Currency = Field(default=Currency.USD)
    reference: Optional[str] = Field(
        None,
        max_length=140,
        description="Transfer reference"
    )

    @validator("from_account")
    def validate_from_account(cls, v):
        if not v.startswith("ACC"):
            raise ValueError("Account must start with ACC")
        return v

    @validator("amount")
    def validate_amount_precision(cls, v):
        # Prevent floating point issues
        if round(v, 2) != v:
            raise ValueError("Amount must have at most 2 decimal places")
        return v

# Usage
@app.post("/api/transfers")
async def create_transfer(request: TransferRequest):
    # Pydantic has already validated the request
    # Any validation errors return 400 automatically
    return await process_transfer(request)
```

### Input Sanitization

```python
import re
import html

class InputSanitizer:
    """Sanitize inputs to prevent injection attacks"""

    # Whitelist of allowed characters for different contexts
    ALPHANUMERIC = re.compile(r'^[a-zA-Z0-9]+$')
    ACCOUNT_NAME = re.compile(r'^[a-zA-Z0-9\s\-\.]+$')
    REFERENCE = re.compile(r'^[a-zA-Z0-9\s\-\.\/\(\),]+$')

    @classmethod
    def sanitize_string(cls, value: str, max_length: int = 255) -> str:
        """Generic string sanitization"""
        if not isinstance(value, str):
            raise ValueError("Expected string")
        # Truncate
        value = value[:max_length]
        # Strip null bytes
        value = value.replace('\x00', '')
        return value

    @classmethod
    def sanitize_html(cls, value: str) -> str:
        """HTML escape for rendering"""
        return html.escape(value, quote=True)

    @classmethod
    def validate_account_name(cls, name: str) -> str:
        """Validate account name against whitelist pattern"""
        name = cls.sanitize_string(name, max_length=100)
        if not cls.ACCOUNT_NAME.match(name):
            raise ValueError("Account name contains invalid characters")
        return name
```

## API Versioning Strategy

```python
# URL-based versioning
# /api/v1/transactions
# /api/v2/transactions

from fastapi import APIRouter

v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

@v1_router.get("/transactions")
async def get_transactions_v1():
    """V1 returns flat list"""
    return {"transactions": [...]}

@v2_router.get("/transactions")
async def get_transactions_v2():
    """V2 returns paginated response with metadata"""
    return {
        "data": [...],
        "pagination": {
            "page": 1,
            "per_page": 20,
            "total": 150,
        },
        "links": {
            "next": "/api/v2/transactions?page=2",
        }
    }

# Deprecation headers
@app.middleware("http")
async def deprecation_middleware(request: Request, call_next):
    response = await call_next(request)
    if "/api/v1/" in str(request.url):
        response.headers["Sunset"] = "2025-12-31"
        response.headers["Deprecation"] = "true"
        response.headers["Link"] = '</api/v2/>; rel="successor-version"'
    return response
```

## API Response Standards

```python
# Standard error response format
class ErrorResponse(BaseModel):
    error: str
    code: str
    message: str
    correlation_id: str
    details: Optional[dict] = None

# Standard success response format
class SuccessResponse(BaseModel):
    data: Any
    meta: Optional[dict] = None

# Security-relevant response headers
SECURITY_HEADERS = {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Content-Security-Policy": "default-src 'none'",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
    "Cache-Control": "no-store, no-cache, must-revalidate",
    "Pragma": "no-cache",
}
```

## Banking-Specific API Controls

### Transaction Integrity

```python
import uuid
from decimal import Decimal
from datetime import datetime, timezone

class TransactionProcessor:
    def __init__(self, db, audit_logger):
        self.db = db
        self.audit_logger = audit_logger

    async def process_transfer(self, request: TransferRequest, user: User) -> dict:
        correlation_id = str(uuid.uuid4())

        # Idempotency check
        idempotency_key = request.idempotency_key
        existing = await self.db.get_transfer_by_idempotency(idempotency_key)
        if existing:
            return existing  # Return original result

        # Begin transaction with isolation level SERIALIZABLE
        async with self.db.transaction(isolation="SERIALIZABLE") as txn:
            # Verify sufficient balance
            from_account = await txn.get_account(request.from_account, for_update=True)
            if from_account.balance < Decimal(str(request.amount)):
                raise InsufficientFundsError()

            # Double-entry bookkeeping
            debit = await txn.create_entry(
                account_id=request.from_account,
                amount=-Decimal(str(request.amount)),
                type="DEBIT",
                correlation_id=correlation_id,
            )
            credit = await txn.create_entry(
                account_id=request.to_account,
                amount=Decimal(str(request.amount)),
                type="CREDIT",
                correlation_id=correlation_id,
            )

            # Idempotency key stored
            transfer = await txn.create_transfer(
                debit_id=debit.id,
                credit_id=credit.id,
                idempotency_key=idempotency_key,
                correlation_id=correlation_id,
            )

        # Audit log (outside DB transaction)
        self.audit_logger.log({
            "event": "transfer_completed",
            "correlation_id": correlation_id,
            "user_id": user.id,
            "from_account": request.from_account,
            "to_account": request.to_account,
            "amount": str(request.amount),
            "timestamp": datetime.now(timezone.utc).isoformat(),
        })

        return {"id": transfer.id, "correlation_id": correlation_id}
```

### API Security Headers Implementation

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS: Restrict to known origins only
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://banking.example.com",
        "https://mobile.bank.example.com",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    max_age=3600,  # Cache preflight for 1 hour
)

# Security headers middleware
@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "0"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    response.headers["X-Permitted-Cross-Domain-Policies"] = "none"
    response.headers["Permissions-Policy"] = "geolocation=(), camera=(), microphone=()"

    # Remove identifying headers
    response.headers.pop("Server", None)
    response.headers.pop("X-Powered-By", None)

    return response
```

## API Gateway Security

```yaml
# Kong Gateway Configuration
services:
  - name: banking-api
    url: http://banking-api.production.svc.cluster.local:8080
    routes:
      - name: banking-api-route
        paths:
          - /api
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          minute: 300
          policy: redis
          redis:
            host: redis.production.svc
            port: 6379
      - name: jwt
        config:
          key_claim_name: iss
          secret_is_base64: true
          claims_to_verify:
            - exp
      - name: acl
        config:
          hide_groups_header: true
          allow:
            - banking-api-users
      - name: ip-restriction
        config:
          deny:
            - 0.0.0.0/0  # Deny all by default
          allow:
            - 10.0.0.0/8  # Allow internal only
      - name: bot-detection
        config:
          deny:
            - "MaliciousBot"
            - "SQLMap"
```

## API Security Testing

### Automated Testing Checklist

```yaml
api_security_tests:
  authentication:
    - test_missing_token
    - test_expired_token
    - test_invalid_token
    - test_revoked_token
    - test_token_reuse_detection

  authorization:
    - test_bola_horizontal
    - test_bola_vertical
    - test_mass_assignment
    - test_function_level_auth

  input_validation:
    - test_sql_injection
    - test_xss_injection
    - test_command_injection
    - test_path_traversal
    - test_oversized_payload
    - test_null_bytes
    - test_unicode_injection

  rate_limiting:
    - test_rate_limit_enforcement
    - test_rate_limit_bypass
    - test_burst_handling

  business_logic:
    - test_negative_amount
    - test_overflow_amount
    - test_double_spend
    - test_race_conditions
```

## Interview Questions

### Junior Level

1. What is the difference between authentication and authorization in APIs?
2. How do you prevent BOLA (Broken Object Level Authorization)?
3. What HTTP status code should you return for rate limiting?
4. Why should API keys not be passed in URL query parameters?

### Senior Level

1. Design a rate limiting system that works across multiple API gateway instances.
2. How would you handle idempotency for financial transactions?
3. What is the difference between symmetric and asymmetric rate limiting?
4. How do you detect and prevent API credential sharing?

### Staff Level

1. How would you design API security for a platform with 10,000 third-party integrations?
2. What is your strategy for detecting and responding to a mass data exfiltration via APIs?
3. How do you balance API security with developer experience for external partners?

## Cross-References

- [OAuth2 and OIDC](./oauth2-and-oidc.md) - API authentication protocols
- [JWT Security](./jwt.md) - Token security best practices
- [Rate Limiting](./rate-limiting.md) - Deep dive on rate limiting strategies
- [Abuse Detection](./abuse-detection.md) - Detecting API abuse patterns
- [Secure Coding Standards](./secure-coding.md) - Input validation patterns
