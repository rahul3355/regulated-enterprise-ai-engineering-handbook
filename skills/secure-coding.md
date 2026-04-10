# Skill: Secure Coding

## Core Principles

1. **Input Is Guilty Until Proven Innocent** — Every input from every source (user, API, database, config file) must be validated before use. Trust is earned through validation, not assumed through origin.
2. **Least Privilege in Code** — Every function, service, and credential operates with the minimum privileges needed. A database connection for reading should not have write permissions.
3. **Fail Securely, Fail Visibly** — Errors must not leak sensitive information, but they must be logged with full context for debugging. The user sees "Internal error"; the log sees the full stack trace.
4. **Security Is Not an Afterthought** — Security decisions are made at design time, implemented at coding time, and verified at review time. "We'll fix it in the next sprint" is how breaches happen.
5. **Dependencies Are Attack Surface** — Every imported library is code you didn't write, didn't review, and may contain vulnerabilities. Pin versions, scan regularly, and update promptly.

## Mental Models

### The Secure Coding Checklist
```
□ Input validation on all external inputs
□ Output encoding on all responses
□ Parameterized queries (no string concatenation for SQL)
□ Authentication required for all endpoints
□ Authorization checked for all data access
□ Secrets from Vault, never hardcoded or in env vars
□ Error handling that doesn't leak sensitive data
□ Logging without logging sensitive data
□ Rate limiting on all public endpoints
□ CSRF protection on state-changing operations
□ CORS configured with specific origins
□ Dependencies scanned for known vulnerabilities
□ Cryptographic operations use standard libraries (never custom crypto)
□ Passwords hashed with bcrypt/argon2 (never MD5, SHA-1, or plaintext)
□ TLS enforced for all network communication
□ Session tokens are random, rotated, and have expiry
```

### The Data Flow Trust Model
```
┌─────────────────────────────────────────────────────────────┐
│                  Data Flow Security Checkpoints              │
│                                                             │
│  User Input ──▶ [Validate] ──▶ [Sanitize] ──▶ [Process]    │
│       │                              │                      │
│       │                              ▼                      │
│       │                         [Authorize] ──▶ [Execute]   │
│       │                              │                      │
│       │                              ▼                      │
│       │                    [Log (no PII)] ──▶ [Respond]     │
│       │                                      │              │
│       ▼                                      ▼              │
│  [Validate]                            [Encode]             │
│       │                                      │              │
│       ▼                                      ▼              │
│  [Sanitize]                          [Return safely]        │
│                                                             │
│  Every arrow is a security checkpoint.                      │
│  Never skip a checkpoint.                                   │
└─────────────────────────────────────────────────────────────┘
```

### The OWASP Top 10 Mapping for GenAI Applications
```
OWASP Top 10 (2021)          GenAI-Specific Addition
─────────────────────        ─────────────────────
A01 Broken Access Control    A01 Prompt Injection
A02 Cryptographic Failures    A02 Insecure Output Handling
A03 Injection                 A03 Training Data Poisoning
A04 Insecure Design           A04 Model Denial of Service
A05 Security Misconfiguration A05 Supply Chain Vulnerability (models)
A06 Vulnerable Components     A06 Sensitive Information Disclosure (via AI)
A07 Auth Failures             A07 Insecure Plugin Design
A08 Data Integrity Failures   A08 Unauthorized Code Execution (via AI)
A09 Logging Failures          A09 Overreliance (AI hallucination impact)
A10 SSRF                      A10 Excessive Agency (AI acting without oversight)
```

## Step-by-Step Approach

### 1. Secure Input Validation (Python)

```python
from pydantic import BaseModel, Field, field_validator, model_validator
import re
from typing import Annotated

# Custom type validators
NonEmptyString = Annotated[str, Field(min_length=1)]
SafeString = Annotated[str, Field(pattern=r'^[a-zA-Z0-9\s\-\.\,\'\"\(\)]+$')]

class DocumentSearchRequest(BaseModel):
    """Secure input validation for a document search endpoint."""

    query: str = Field(
        min_length=1,
        max_length=500,
        description="Search query for document retrieval"
    )
    classification_level: str = Field(
        pattern=r'^(public|internal|confidential|restricted)$'
    )
    max_results: int = Field(
        ge=1,
        le=100,
        default=10
    )
    department: str | None = Field(
        default=None,
        max_length=100,
        pattern=r'^[a-z\-]+$'
    )

    @field_validator('query')
    @classmethod
    def validate_query(cls, v: str) -> str:
        # Check for SQL injection patterns
        sql_patterns = [
            r"(\b(union|select|insert|update|delete|drop|alter|create|execute|exec)\b)",
            r"(--|;|/\*|\*/)",
            r"(\b(or|and)\b\s+\d+\s*=\s*\d+)",
        ]
        for pattern in sql_patterns:
            if re.search(pattern, v, re.IGNORECASE):
                raise ValueError("Query contains disallowed patterns")

        # Check for XSS patterns
        xss_patterns = [
            r'<script',
            r'javascript:',
            r'on\w+\s*=',
            r'<iframe',
            r'<object',
        ]
        for pattern in xss_patterns:
            if re.search(pattern, v, re.IGNORECASE):
                raise ValueError("Query contains disallowed patterns")

        return v.strip()

    @model_validator(mode='after')
    def validate_access(self) -> 'DocumentSearchRequest':
        # Users can't search restricted documents without explicit authorization
        if self.classification_level == 'restricted' and not self.department:
            raise ValueError("Department required for restricted document search")
        return self
```

### 2. Secure Database Access (Python/SQLAlchemy)

```python
from sqlalchemy import create_engine, text, select
from sqlalchemy.orm import Session

# BAD: String concatenation — SQL injection vulnerability
def get_user_bad(username: str) -> dict:
    query = f"SELECT * FROM users WHERE username = '{username}'"
    # If username = "' OR '1'='1", this returns ALL users
    return db.execute(text(query)).fetchone()

# GOOD: Parameterized query — safe from SQL injection
def get_user_good(username: str, session: Session) -> dict:
    query = text("SELECT id, username, email, role FROM users WHERE username = :username")
    return session.execute(query, {"username": username}).fetchone()

# BETTER: ORM with explicit column selection
from sqlalchemy.orm import Session
from app.models import User

def get_user_best(username: str, session: Session) -> User | None:
    return session.query(User).filter(
        User.username == username
    ).with_entities(
        User.id, User.username, User.email, User.role  # Explicit columns
    ).first()

# SECURE: Row-level access control
def get_account(account_id: str, user_id: str, session: Session) -> dict:
    """Only return the account if it belongs to the requesting user."""
    result = session.execute(
        text("""
            SELECT id, account_number, balance, currency
            FROM accounts
            WHERE id = :account_id AND customer_id = :user_id
        """),
        {"account_id": account_id, "user_id": user_id}
    )
    row = result.fetchone()
    if not row:
        # Don't reveal whether the account exists or the user lacks access
        raise AccountNotFoundError("Account not found")
    return dict(row._mapping)
```

### 3. Secure Secret Management

```python
# BAD: Hardcoded secrets
OPENAI_API_KEY = "sk-abc123..."  # NEVER DO THIS
DATABASE_PASSWORD = "supersecret"  # NEVER DO THIS

# BAD: Secrets in environment variables (visible in /proc/*/environ, logs, debug endpoints)
import os
api_key = os.environ["OPENAI_API_KEY"]  # Better but not ideal

# GOOD: Secrets from Vault (HashiCorp Vault example)
import hvac

class VaultSecretManager:
    def __init__(self, vault_url: str, token: str):
        self.client = hvac.Client(url=vault_url, token=token)

    def get_secret(self, path: str, key: str) -> str:
        """Retrieve a secret from Vault."""
        response = self.client.secrets.kv.v2.read_secret_version(path=path)
        return response["data"]["data"][key]

# Usage
vault = VaultSecretManager(
    vault_url="https://vault.bank.internal:8200",
    token=os.environ["VAULT_TOKEN"],  # Vault token itself from secure source
)

openai_api_key = vault.get_secret("genai-platform/openai", "api_key")
db_password = vault.get_secret("genai-platform/postgres", "password")

# BETTER: Dynamic secrets with automatic rotation
# Vault can generate database credentials that are time-limited
def get_temp_db_credentials() -> dict:
    """Generate short-lived database credentials from Vault."""
    response = vault.client.write("database/creds/genai-platform", ttl="1h")
    return {
        "username": response["data"]["username"],
        "password": response["data"]["password"],
        "ttl": response["data"]["lease_duration"],
    }
```

### 4. Secure Error Handling

```python
import logging
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse

logger = logging.getLogger(__name__)

# BAD: Leaking internal details
@app.get("/accounts/{account_id}")
async def get_account(account_id: str, request: Request):
    try:
        account = await db.get_account(account_id, request.user.id)
        return account
    except Exception as e:
        # NEVER expose internal error details to the user
        return {"error": str(e), "traceback": traceback.format_exc()}
        # This reveals: database schema, file paths, stack traces, SQL queries

# GOOD: User-safe error, detailed logging
@app.get("/accounts/{account_id}")
async def get_account(account_id: str, request: Request):
    try:
        account = await db.get_account(account_id, request.user.id)
        if not account:
            raise HTTPException(404, detail="Account not found")
        return account
    except HTTPException:
        raise  # Re-raise HTTP exceptions (they already have safe messages)
    except AccountAccessDeniedError:
        logger.warning(
            "Access denied: user=%s account=%s",
            request.user.id, account_id,
            extra={"user_id": request.user.id, "account_id": account_id}
        )
        raise HTTPException(403, detail="You do not have access to this account")
    except DatabaseError as e:
        logger.exception(
            "Database error retrieving account: user=%s account=%s",
            request.user.id, account_id,
            extra={"user_id": request.user.id, "account_id": account_id}
        )
        raise HTTPException(500, detail="An internal error occurred")
    except Exception as e:
        logger.exception(
            "Unexpected error: user=%s account=%s error_type=%s",
            request.user.id, account_id, type(e).__name__,
        )
        raise HTTPException(500, detail="An internal error occurred")

# Custom exception handler for unhandled exceptions
async def unhandled_exception_handler(request: Request, exc: Exception):
    logger.critical(
        "Unhandled exception: method=%s path=%s error=%s",
        request.method, request.url.path, exc,
        exc_info=True,
    )
    return JSONResponse(
        status_code=500,
        content={"detail": "An internal error occurred. Please try again later."},
    )
```

### 5. Secure Authentication and Session Management

```python
import secrets
import hashlib
from datetime import datetime, timedelta, timezone
import jwt

class SessionManager:
    """Secure session management for banking application."""

    def __init__(self, secret_key: str, token_expiry_minutes: int = 30):
        self._secret_key = secret_key
        self._token_expiry = timedelta(minutes=token_expiry_minutes)

    def create_token(self, user_id: str, role: str, scopes: list[str]) -> str:
        """Create a JWT token with secure settings."""
        now = datetime.now(timezone.utc)
        payload = {
            "sub": user_id,
            "role": role,
            "scopes": scopes,
            "iat": now,
            "exp": now + self._token_expiry,
            "nbf": now,
            "jti": secrets.token_hex(16),  # Unique token ID
        }
        return jwt.encode(payload, self._secret_key, algorithm="HS256")

    def verify_token(self, token: str) -> dict:
        """Verify and decode a JWT token."""
        try:
            payload = jwt.decode(
                token,
                self._secret_key,
                algorithms=["HS256"],
                options={
                    "require": ["exp", "iat", "sub", "jti"],
                    "verify_exp": True,
                    "verify_nbf": True,
                }
            )
            return payload
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token has expired")
        except jwt.InvalidTokenError as e:
            raise AuthenticationError(f"Invalid token: {e}")

# Usage in FastAPI
from fastapi import Depends, HTTPException

async def get_current_user(
    authorization: str = Header(...),
    session: SessionManager = Depends(get_session_manager),
) -> User:
    """Extract and validate the current user from the Authorization header."""
    try:
        scheme, token = authorization.split()
        if scheme.lower() != "bearer":
            raise HTTPException(401, "Invalid authentication scheme")

        payload = session.verify_token(token)
        return User(
            id=payload["sub"],
            role=payload["role"],
            scopes=payload["scopes"],
        )
    except (ValueError, AuthenticationError) as e:
        raise HTTPException(401, str(e))
```

### 6. Secure TypeScript Coding (Next.js)

```typescript
// BAD: Directly rendering user input (XSS vulnerability)
function UserProfile({ user }: { user: { bio: string } }) {
  return <div dangerouslySetInnerHTML={{ __html: user.bio }} />;
}

// GOOD: React automatically escapes content
function UserProfileSafe({ user }: { user: { bio: string } }) {
  return <div>{user.bio}</div>;
}

// BAD: Exposing sensitive data in client-side code
// Never put API keys, secrets, or internal URLs in client-side code
const API_KEY = process.env.OPENAI_API_KEY; // This gets bundled to the browser!

// GOOD: Server-side only code
// Server Component (Next.js App Router)
async function generateResponse(message: string) {
  // This runs on the server — environment variables are safe here
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ messages: [{ role: 'user', content: message }] }),
  });
  return response.json();
}

// BAD: SQL injection via string concatenation (in API route)
async function getUser(req: NextRequest) {
  const username = req.nextUrl.searchParams.get('username');
  const user = await db.query(`SELECT * FROM users WHERE username = '${username}'`);
  return NextResponse.json(user);
}

// GOOD: Parameterized queries in API route
async function getUserSafe(req: NextRequest) {
  const username = req.nextUrl.searchParams.get('username');

  // Validate input
  if (!username || typeof username !== 'string' || username.length > 100) {
    return NextResponse.json({ error: 'Invalid username' }, { status: 400 });
  }

  // Sanitize: allow only alphanumeric characters and underscores
  if (!/^[a-zA-Z0-9_]+$/.test(username)) {
    return NextResponse.json({ error: 'Invalid username format' }, { status: 400 });
  }

  const user = await db.query(
    'SELECT id, username, email FROM users WHERE username = $1',
    [username]
  );
  return NextResponse.json(user.rows[0]);
}
```

### 7. Secure Dependency Management

```bash
# Python: Scan dependencies for known vulnerabilities
pip install safety
safety check --json --output safety-report.json

# Python: Pin dependencies with hashes (in requirements.txt)
requests==2.31.0 \
    --hash=sha256:58cd2971e3c3c6e7c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3c3
    --hash=sha256:abc123...

# Python: Use pip-audit for CVE scanning
pip install pip-audit
pip-audit --format=json --output pip-audit-report.json

# Node.js: Audit dependencies
npm audit --json --audit-level=high

# Node.js: Use Snyk for continuous monitoring
npx snyk test

# Go: Check for vulnerabilities
govulncheck ./...

# All languages: Use Dependabot/Renovate for automated PRs on vulnerability fixes
# .github/dependabot.yaml
# version: 2
# updates:
#   - package-ecosystem: "pip"
#     directory: "/"
#     schedule:
#       interval: "weekly"
#     open-pull-requests-limit: 10
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| String concatenation in SQL queries | SQL injection, full database compromise | Parameterized queries always |
| Rendering user input as HTML | XSS, session hijacking | Escape all output, use CSP headers |
| Hardcoded secrets in code | Credential exposure via Git history | Use Vault, never commit secrets |
| Catching all exceptions with `except: pass` | Silent failures, no debugging info | Log exceptions, re-raise or handle specifically |
| Storing passwords in plaintext/hash that's too fast | Credential theft in database breach | bcrypt or argon2 with proper cost factors |
| Using `eval()` or `exec()` on user input | Remote code execution | Never execute user input as code |
| Trusting `X-Forwarded-For` header | IP spoofing, bypassing IP-based controls | Use the actual socket peer IP |
| Not validating Content-Type | Content-type confusion attacks | Validate and enforce expected Content-Type |
| Using deprecated cryptographic functions | Weak encryption, data exposure | Use modern algorithms (AES-256-GCM, SHA-256+) |
| Not updating vulnerable dependencies | Known exploits in production | Automated dependency scanning and updates |

## Banking-Specific Concerns

1. **Audit Trail in Code** — Every security-relevant operation must log: who did what, when, and on which resource. This is not optional; it's a regulatory requirement.
2. **Data Minimization** — Only request, process, and store the minimum data needed. If you don't need the customer's SSN for a specific operation, don't request it.
3. **Segregation of Duties in Code** — The code that approves a transaction should not be the same code that initiates it. Implement separation in the codebase.
4. **Regulatory Code Paths** — Some banking operations have specific regulatory requirements (e.g., dual approval for large transfers). Code must enforce these requirements, not just suggest them.
5. **Data Retention in Code** — Code must not retain data longer than the retention policy allows. Implement automatic data purging based on classification.

## GenAI-Specific Concerns

1. **Prompt Injection via User Input** — Never concatenate user input directly into system prompts. Use structured templates that isolate user input from system instructions.
2. **PII in Prompts** — User messages may contain PII. Redact PII before sending to external LLM APIs. Log PII detection events for audit.
3. **Model Output Trust** — Never trust LLM output as authoritative. Verify critical information (account numbers, policy details) against authoritative sources.
4. **Token Leakage** — Sensitive data in the conversation context may be included in token counts. Monitor token usage for anomalies that may indicate data leakage.
5. **Plugin/Tool Security** — If the GenAI system has access to tools (APIs, databases), validate all tool call parameters and enforce access control at the tool level, not just the prompt level.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Dependency vulnerability count | > 0 critical, > 5 high | Known exploitable vulnerabilities |
| Secret detection in commits | > 0 per month | Credentials in source control |
| Input validation failure rate | > 5% of requests | Possible attack or client bug |
| Authentication failure rate | > 5% of requests | Possible credential stuffing |
| Authorization failure rate | > 1% of requests | Possible privilege escalation |
| SQL injection attempt rate | > 0 per day | Active attack |
| XSS attempt rate | > 0 per day | Active attack |
| TLS certificate expiry | < 30 days | Upcoming TLS failures |
| Code review security comment rate | Track trend | Security awareness in reviews |
| Time to patch critical vulnerability | > 7 days | Vulnerability exposure window |

## Interview Questions

1. What is SQL injection and how do you prevent it in Python?
2. How do you securely store and verify passwords in a banking application?
3. What is the difference between authentication and authorization? Give an example of each.
4. How would you handle secrets in a containerized application deployed to Kubernetes?
5. What is XSS and how do you prevent it in a React application?
6. How do you validate and sanitize user input in a REST API?
7. What is a parameterized query and why is it safer than string concatenation?
8. How do you ensure that error messages don't leak sensitive information?

## Hands-On Exercise

### Exercise: Secure a Vulnerable Banking API

**Problem:** The following API endpoint has multiple security vulnerabilities. Identify and fix them all.

```python
@router.get("/api/accounts/{account_id}")
async def get_account(account_id: str, request: Request):
    # Get the user ID from the request header (untrusted)
    user_id = request.headers.get("X-User-ID")

    # Build the query
    query = f"SELECT * FROM accounts WHERE id = '{account_id}'"

    # Execute the query
    result = db.execute(query)
    account = result.fetchone()

    # Return the full account data
    return {
        "account": dict(account),
        "debug": {"query": query, "user": user_id}
    }
```

**Constraints:**
- The API is used by authenticated bank employees
- Users should only see their own accounts
- The response must not include sensitive data (internal IDs, audit flags)
- Error messages must not reveal internal details
- All access must be logged for audit

**Expected Output:**
- Fixed endpoint with all vulnerabilities addressed
- Input validation for account_id
- Parameterized query
- Authorization check (user owns the account)
- Response filtering (no sensitive fields)
- Secure error handling
- Audit logging

**Hints:**
- The `X-User-ID` header is untrusted — use the authenticated user from the token
- The SQL query is vulnerable to injection
- `SELECT *` returns more columns than needed
- The debug field leaks the SQL query
- There is no authorization check

**Extension:**
- Add rate limiting to the endpoint
- Add input validation using Pydantic
- Implement row-level security in the database
- Write unit tests for the secure endpoint

---

**Related files:**
- `security/owasp-top-10.md` — OWASP Top 10 deep-dive
- `security/api-security.md` — API security guide
- `security/secret-management.md` — Vault and secret management
- `skills/secure-api-design.md` — Secure API design patterns
- `skills/threat-modeling.md` — Threat modeling
- `skills/genai-guardrails.md` — GenAI-specific security
- `testing-and-quality/security-testing.md` — Security testing practices
