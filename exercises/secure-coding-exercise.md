# Secure Coding Exercise

> Find and fix security vulnerabilities in a GenAI service code review — the kind of issues security teams flag in banking.

## Problem Statement

You are reviewing a PR for the GenAI chat service. The code works functionally, but you need to identify and flag security vulnerabilities before it can be deployed to production.

Review the code below, identify all security issues, and provide fixes.

## Code Under Review

```python
# genai_service/chat.py — PR #892
import os
import sqlite3
import requests
import logging

logger = logging.getLogger(__name__)

# Database connection
DB_PATH = os.environ.get("DB_PATH", "/tmp/genai.db")

def get_db():
    conn = sqlite3.connect(DB_PATH)
    return conn

# LLM API configuration
LLM_API_KEY = os.environ.get("LLM_API_KEY", "")
LLM_API_URL = os.environ.get("LLM_API_URL", "https://api.openai.com/v1/chat/completions")

def call_llm(messages: list[dict]) -> str:
    """Call the LLM API and return the response."""
    headers = {
        "Authorization": f"Bearer {LLM_API_KEY}",
        "Content-Type": "application/json",
    }
    data = {
        "model": "gpt-4-turbo",
        "messages": messages,
        "max_tokens": 4096,
    }
    response = requests.post(LLM_API_URL, headers=headers, json=data)
    return response.json()["choices"][0]["message"]["content"]

def save_query(user_id: str, query: str, response: str):
    """Save the query and response to the database for history."""
    conn = get_db()
    cursor = conn.cursor()
    # Build the SQL query
    sql = f"""
        INSERT INTO query_history (user_id, query, response, timestamp)
        VALUES ('{user_id}', '{query}', '{response}', datetime('now'))
    """
    cursor.execute(sql)
    conn.commit()
    conn.close()

def get_user_history(user_id: str):
    """Get query history for a user."""
    conn = get_db()
    cursor = conn.cursor()
    cursor.execute(f"SELECT * FROM query_history WHERE user_id = '{user_id}'")
    rows = cursor.fetchall()
    conn.close()
    return rows

def process_query(user_id: str, query: str) -> dict:
    """Process a user query through the GenAI pipeline."""
    # Log the query
    logger.info(f"Processing query for user {user_id}: {query}")

    # Build messages for the LLM
    messages = [
        {"role": "system", "content": "You are a helpful banking assistant."},
        {"role": "user", "content": query},
    ]

    # Call the LLM
    response = call_llm(messages)

    # Save to history
    save_query(user_id, query, response)

    return {
        "user_id": user_id,
        "query": query,
        "response": response,
        "model": "gpt-4-turbo",
    }

def admin_panel(request):
    """Admin endpoint to view all queries."""
    admin_key = request.headers.get("X-Admin-Key", "")
    if admin_key == "supersecretkey123":
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM query_history")
        rows = cursor.fetchall()
        conn.close()
        return {"queries": rows}
    return {"error": "unauthorized"}, 401

def debug_endpoint(request):
    """Debug endpoint for development."""
    return {
        "llm_api_key": LLM_API_KEY[:10] + "...",
        "db_path": DB_PATH,
        "llm_url": LLM_API_URL,
        "env": dict(os.environ),
    }
```

## Security Issues to Find

There are **at least 10 security vulnerabilities** in this code. Find them all.

## Security Findings

| # | Vulnerability | Severity | Location | Fix |
|---|--------------|----------|----------|-----|
| 1 | **SQL Injection** | Critical | `save_query()`, `get_user_history()` | Use parameterized queries |
| 2 | **Hardcoded admin secret** | Critical | `admin_panel()` | Use proper auth, store secret securely |
| 3 | **Debug endpoint exposes secrets** | Critical | `debug_endpoint()` | Remove entirely or protect with auth |
| 4 | **Environment variable dump** | Critical | `debug_endpoint()` | Remove `dict(os.environ)` |
| 5 | **API key in logs** | High | `logger.info()` | Don't log user queries with user IDs |
| 6 | **No input validation** | High | `process_query()` | Validate query length, content |
| 7 | **No rate limiting** | Medium | `process_query()` | Add per-user rate limiting |
| 8 | **No error handling for LLM call** | Medium | `call_llM()` | Handle API errors, don't crash |
| 9 | **No TLS verification** | Medium | `call_llm()` | Ensure requests verifies TLS |
| 10 | **Database in /tmp** | Medium | `DB_PATH` default | Use persistent, secured storage |
| 11 | **No authentication on user history** | Low | `get_user_history()` | Verify user can only see own data |
| 12 | **Response data not sanitized** | Medium | `save_query()` | Sanitize before storage |

## Fixes

### Fix 1: SQL Injection (Critical)

```python
# BEFORE (vulnerable):
sql = f"""
    INSERT INTO query_history (user_id, query, response, timestamp)
    VALUES ('{user_id}', '{query}', '{response}', datetime('now'))
"""
cursor.execute(sql)

# AFTER (safe):
cursor.execute(
    """INSERT INTO query_history (user_id, query, response, timestamp)
       VALUES (?, ?, ?, datetime('now'))""",
    (user_id, query, response)
)

# Same fix for get_user_history:
# BEFORE:
cursor.execute(f"SELECT * FROM query_history WHERE user_id = '{user_id}'")
# AFTER:
cursor.execute("SELECT * FROM query_history WHERE user_id = ?", (user_id,))
```

### Fix 2: Hardcoded Admin Secret (Critical)

```python
# BEFORE:
if admin_key == "supersecretkey123":

# AFTER:
import hmac
ADMIN_KEY = os.environ.get("ADMIN_API_KEY")

def admin_panel(request):
    admin_key = request.headers.get("X-Admin-Key", "")
    if not ADMIN_KEY:
        return {"error": "admin not configured"}, 500
    if hmac.compare_digest(admin_key, ADMIN_KEY):
        # ... authorized access
```

### Fix 3 & 4: Remove Debug Endpoint (Critical)

```python
# REMOVE entirely — no debug endpoints in production
# If needed for development, guard with:
if os.environ.get("ENVIRONMENT") == "development":
    # debug endpoints
```

### Fix 5: Secure Logging

```python
# BEFORE:
logger.info(f"Processing query for user {user_id}: {query}")

# AFTER:
logger.info(
    "Processing query: user=%s, query_length=%d",
    user_id,
    len(query)
)
# Don't log query content or associate with user ID in plain text
```

### Fix 6: Input Validation

```python
MAX_QUERY_LENGTH = 4000
MIN_QUERY_LENGTH = 1

def process_query(user_id: str, query: str) -> dict:
    if not query or len(query) < MIN_QUERY_LENGTH:
        raise ValueError("Query cannot be empty")
    if len(query) > MAX_QUERY_LENGTH:
        raise ValueError(f"Query exceeds maximum length of {MAX_QUERY_LENGTH}")

    # Check for prompt injection patterns
    from .injection_detector import PromptInjectionDetector
    detector = PromptInjectionDetector()
    if not detector.is_safe(query):
        logger.warning("Prompt injection detected for user %s", user_id)
        return {"error": "Query contains potentially malicious patterns"}
```

### Fix 8: Error Handling for LLM Call

```python
def call_llm(messages: list[dict]) -> str:
    try:
        response = requests.post(
            LLM_API_URL,
            headers={"Authorization": f"Bearer {LLM_API_KEY}"},
            json={"model": "gpt-4-turbo", "messages": messages, "max_tokens": 4096},
            timeout=30,
            verify=True,  # TLS verification
        )
        response.raise_for_status()
        return response.json()["choices"][0]["message"]["content"]
    except requests.exceptions.Timeout:
        logger.error("LLM API request timed out")
        raise ServiceUnavailableError("LLM API is not responding")
    except requests.exceptions.RequestException as e:
        logger.error("LLM API request failed: %s", e)
        raise ServiceUnavailableError("LLM API request failed")
    except (KeyError, IndexError) as e:
        logger.error("Unexpected LLM API response format: %s", e)
        raise ServiceError("Unexpected response from LLM API")
```

## Extensions

1. **Add dependency scanning:** Run `safety check` and `bandit` on the code. What additional issues do they find?

2. **Implement a security review checklist:** Create a reusable checklist for reviewing GenAI Python code.

3. **Add pre-commit security hooks:** Configure pre-commit to run security checks on every commit.

4. **Write security unit tests:** Add tests that verify SQL injection is prevented, auth is enforced, etc.

5. **Threat model the service:** Use the STRIDE framework to identify threats beyond code-level vulnerabilities.

## Interview Relevance

Security code review is tested in senior and security-focused roles:

| Skill | Why It Matters |
|-------|---------------|
| SQL injection detection | Most common web vulnerability |
| Secret management | API keys, passwords, tokens |
| Input validation | First line of defense |
| Secure logging | Don't log sensitive data |
| Error handling | Don't leak information in errors |

**Follow-up questions:**
- "What's the difference between authentication and authorization?"
- "How do you securely store API keys in a cloud environment?"
- "What's the OWASP Top 10 for LLM applications?"
- "How do you review code for prompt injection vulnerabilities?"
