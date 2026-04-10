# Skill: Code Review Best Practices

## Core Principles

1. **Review the Problem, Not Just the Solution** — Before reviewing code, understand what problem it solves. A correct solution to the wrong problem is still a bug.
2. **Prioritize by Impact** — Security, correctness, and architecture issues matter more than style. Comment on what matters first. Don't nitpick formatting when there's a race condition.
3. **Be Kind, Be Specific, Be Constructive** — Code review is a collaboration, not a trial. Explain why, suggest alternatives, and acknowledge good work.
4. **Small Reviews Catch More Bugs** — Reviews over 400 lines of code miss defects exponentially. Push for small, focused PRs. One feature, one PR.
5. **Automate What You Can** — Linters, type checkers, formatters, and security scanners should catch mechanical issues. Human reviewers should focus on architecture, logic, and edge cases.

## Mental Models

### The Code Review Priority Pyramid
```
┌─────────────────────────────────────────────────────┐
│            Code Review Priority Pyramid             │
│                                                     │
│              ┌───────────────┐                      │
│              │  Architecture  │  Design patterns,    │
│              │  & Design     │  modularity,          │
│              │               │  coupling             │
│             ┌┴───────────────┴┐                     │
│             │  Correctness     │  Logic errors,       │
│             │  & Security      │  edge cases, vulns  │
│            ┌┴──────────────────┴┐                    │
│            │  Performance &     │  N+1 queries,       │
│            │  Scalability      │  memory, caching    │
│           ┌┴────────────────────┴┐                   │
│           │  Readability &       │  Naming, docs,     │
│           │  Maintainability    │  complexity         │
│          ┌┴──────────────────────┴┐                  │
│          │  Style & Formatting     │  Linting, format  │
│          │                        │  (automated)      │
│          └────────────────────────┘                   │
│                                                       │
│  Bottom layers = automated. Top layers = human.       │
│  Never block a PR for bottom-layer issues.            │
└─────────────────────────────────────────────────────┘
```

### The Reviewer's Checklist
```
□ Does this code solve the stated problem?
□ Is the approach the right one? (vs. alternatives considered)
□ Are there security vulnerabilities? (injection, auth, data exposure)
□ Are there correctness issues? (race conditions, off-by-one, null handling)
□ Are there edge cases not handled? (empty input, large input, concurrent access)
□ Is error handling appropriate? (fail securely, log context)
□ Is the code testable? (unit tests, integration tests)
□ Are tests sufficient? (happy path, error path, edge cases)
□ Are there performance concerns? (N+1 queries, unbounded loops, large payloads)
□ Is the code readable and maintainable? (naming, structure, comments)
□ Are there new dependencies? (are they necessary, vetted, pinned?)
□ Does this change require documentation updates?
□ Does this change require migration scripts?
□ Does this change require feature flags?
```

### The Author's Pre-Review Checklist
```
□ I've tested this manually (not just automated tests)
□ I've written tests for the happy path, error path, and edge cases
□ I've run the linter and type checker locally
□ I've reviewed my own diff and removed debug code
□ I've updated documentation if needed
□ The PR description explains the problem, the solution, and how to test
□ The PR is focused (one feature/fix per PR)
□ I've linked the Jira ticket or issue being addressed
□ I've flagged areas I'm unsure about for specific feedback
```

## Step-by-Step Approach

### 1. Writing a Good PR Description

```markdown
## Problem

The RAG service occasionally returns stale document content because
it retrieves documents from a read replica that has fallen behind
the primary database by up to 30 seconds.

This affects ~2% of search queries, causing users to receive outdated
policy information.

## Solution

Add a fallback mechanism: if the read replica returns stale data
(replication lag > 5 seconds), fall back to the primary database
for that specific query.

Changes:
- Added replication lag monitoring to the database connection pool
- Added automatic fallback to primary when lag exceeds threshold
- Added metric `db_fallback_count` to monitor fallback frequency
- Updated health endpoint to report replica lag status

## Alternatives Considered

1. **Always use primary** — Simple but increases primary load by 40%.
2. **Wait for replica sync** — Blocks the request, degrading latency.
3. **Cache documents in Redis** — Complex invalidation, stale cache risk.

Chosen: Fallback to primary — balances freshness with read offloading.

## Testing

- Unit tests: `pytest tests/unit/test_db_fallback.py` (12 tests)
- Integration tests: `pytest tests/integration/test_replica_fallback.py` (5 tests)
- Manual test: Set replication lag > 5s, verify fallback triggers

## Rollout

- Behind feature flag `enable_replica_fallback` (default: false)
- Monitor `db_fallback_count` metric — expect ~2% of queries initially
- If fallback rate > 10%, investigate replica performance

## Related
- Jira: GENAI-1234
- Architecture decision: docs/adr/042-replica-read-strategy.md
```

### 2. Writing Effective Review Comments

```python
# BAD COMMENT: Nitpicking style while missing a security issue
# "Please use double quotes instead of single quotes for consistency."
# (This should be caught by the linter, not a human reviewer)

# GOOD COMMENT: Security-focused, specific, and constructive
"""
Security: This query is vulnerable to SQL injection because it uses
string concatenation to build the query.

Current code (vulnerable):
    query = f"SELECT * FROM accounts WHERE id = '{account_id}'"

Suggested fix (parameterized):
    query = text("SELECT * FROM accounts WHERE id = :account_id")
    result = session.execute(query, {"account_id": account_id})

See OWASP A03:2021 — Injection. This is a critical fix that must
be addressed before merge.
"""

# GOOD COMMENT: Architecture-focused with alternatives
"""
Architecture: This function mixes data access with business logic.
The access control check is embedded in the retrieval function,
making it hard to test and reuse independently.

Suggestion: Separate concerns:

    # Data access layer
    async def fetch_documents(query: str, top_k: int) -> list[Document]:
        ...

    # Business logic layer
    async def retrieve_authorized_documents(
        query: str, user: User, top_k: int = 10
    ) -> list[Document]:
        docs = await fetch_documents(query, top_k)
        return [d for d in docs if user.can_access(d)]

This makes both functions independently testable and allows
fetch_documents to be reused in contexts where access control
is handled differently (e.g., admin bulk operations).
"""

# GOOD COMMENT: Performance-focused with evidence
"""
Performance: This loop makes one database query per document to check
access control. With 10 documents per query, that's 10 additional
queries per request (N+1 pattern).

At 500 requests/second, this adds 5,000 queries/second to the database.

Suggestion: Batch the access control check:

    # Single query for all documents
    doc_ids = [d.id for d in documents]
    accessible = await db.check_access_batch(user.id, doc_ids)
    return [d for d in documents if d.id in accessible]

This reduces N queries to 1. At 500 req/s, that's 500 queries/second
instead of 5,000.
"""

# GOOD COMMENT: Positive reinforcement
"""
Nice use of the circuit breaker pattern for the LLM API calls.
The fallback to cached responses during outages is exactly the
right behavior. The configuration is clear and the timeout values
are well-justified.
"""
```

### 3. Reviewing for Security

```python
# Review checklist for security issues:

# 1. Authentication & Authorization
def review_auth_endpoints(code):
    """Check: Is authentication required on all endpoints?"""
    # Look for: @router.post/get without Depends(get_current_user)
    # Look for: Missing authorization check after authentication
    pass

# 2. Input Validation
def review_input_validation(code):
    """Check: Is all external input validated before use?"""
    # Look for: Direct use of request.body, request.params, request.headers
    # Look for: SQL string concatenation (injection risk)
    # Look for: eval(), exec(), subprocess with user input
    pass

# 3. Data Exposure
def review_data_exposure(code):
    """Check: Are sensitive values exposed in responses or logs?"""
    # Look for: return {"password": ..., "api_key": ..., "token": ...}
    # Look for: logging passwords, tokens, PII
    # Look for: SELECT * (returns more columns than needed)
    pass

# 4. Secret Management
def review_secrets(code):
    """Check: Are secrets handled securely?"""
    # Look for: Hardcoded API keys, passwords, tokens
    # Look for: os.environ["SECRET_KEY"] (should use Vault)
    # Look for: Secrets logged or returned in error messages
    pass

# 5. Dependency Security
def review_dependencies(pr_diff):
    """Check: Are new dependencies necessary and secure?"""
    # Look for: New packages in requirements.txt / package.json
    # Verify: Package is maintained, has no known CVEs
    # Verify: Package version is pinned (not using ^ or ~)
    pass
```

### 4. Reviewing for Correctness

```python
# Common correctness issues to look for:

# 1. Race Conditions
async def transfer_money_bad(from_account: str, to_account: str, amount: Decimal):
    """Race condition: two concurrent transfers can overdraft the account."""
    balance = await db.get_balance(from_account)  # Read
    if balance < amount:
        raise InsufficientFundsError()
    await db.update_balance(from_account, balance - amount)  # Write
    # Another transfer may have read the same balance before this write

# FIX: Use database-level locking
async def transfer_money_good(from_account: str, to_account: str, amount: Decimal, session):
    """Correct: Uses SELECT FOR UPDATE to lock the row."""
    row = await session.execute(
        text("SELECT balance FROM accounts WHERE id = :id FOR UPDATE"),
        {"id": from_account}
    )
    balance = row.scalar()
    if balance < amount:
        raise InsufficientFundsError()
    await session.execute(
        text("UPDATE accounts SET balance = :balance WHERE id = :id"),
        {"balance": balance - amount, "id": from_account}
    )

# 2. Off-by-One Errors
def paginate(items: list, page: int, page_size: int) -> list:
    """Check: Are pagination calculations correct?"""
    start = (page - 1) * page_size  # Correct: page 1 starts at 0
    end = start + page_size
    return items[start:end]  # Slicing is exclusive of end, which is correct

# 3. Null/None Handling
def get_user_display_name(user: User) -> str:
    """Check: What if user.first_name is None?"""
    # BAD: user.first_name + " " + user.last_name  # TypeError if None
    # GOOD:
    parts = [user.first_name, user.last_name]
    return " ".join(p for p in parts if p) or "Anonymous User"

# 4. Resource Cleanup
async def process_file_bad(file_path: str) -> dict:
    """Check: Are resources (files, connections) properly cleaned up?"""
    f = open(file_path)  # Never closed if an exception occurs
    data = json.load(f)
    return parse_data(data)

# GOOD: Use context managers
async def process_file_good(file_path: str) -> dict:
    with open(file_path) as f:
        data = json.load(f)
    return parse_data(data)
```

### 5. Reviewing for Performance

```python
# Performance issues to look for:

# 1. N+1 Queries (see backend-performance.md for details)
def get_users_with_departments_bad():
    users = db.query(User).all()
    for user in users:
        user.department = db.query(Department).get(user.department_id)  # N queries!
    return users

# 2. Unbounded Queries
def search_documents_bad(query: str) -> list[Document]:
    """No limit — could return millions of results."""
    return db.query(Document).filter(Document.content.contains(query)).all()

# GOOD: Always paginate
def search_documents_good(query: str, limit: int = 100) -> list[Document]:
    return db.query(Document).filter(
        Document.content.contains(query)
    ).limit(limit).all()

# 3. Inefficient Data Structures
def find_duplicates_bad(items: list) -> set:
    """O(n²) — checking membership in a list is O(n)."""
    duplicates = set()
    for i, item in enumerate(items):
        if item in items[i+1:]:  # O(n) for each item
            duplicates.add(item)
    return duplicates

# GOOD: Use a set for O(1) lookups
def find_duplicates_good(items: list) -> set:
    """O(n) — checking membership in a set is O(1)."""
    seen = set()
    duplicates = set()
    for item in items:
        if item in seen:
            duplicates.add(item)
        seen.add(item)
    return duplicates

# 4. Missing Caching
def get_policy_document_bad(doc_id: str) -> dict:
    """Hits the database on every call, even though policies rarely change."""
    return db.query(Policy).get(doc_id)

# GOOD: Cache with appropriate TTL
@cache_response(ttl=3600)  # Cache for 1 hour
def get_policy_document_good(doc_id: str) -> dict:
    return db.query(Policy).get(doc_id)
```

### 6. Automated Code Review with Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  # Python linting and formatting
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.8
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  # Python type checking
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, types-requests]

  # Security scanning
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  # TypeScript linting
  - repo: local
    hooks:
      - id: eslint
        name: eslint
        entry: npm run lint
        language: system
        files: \.tsx?$
        pass_filenames: false

  # TypeScript type checking
  - repo: local
    hooks:
      - id: tsc
        name: tsc
        entry: npx tsc --noEmit
        language: system
        files: \.tsx?$
        pass_filenames: false

  # Secret scanning
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.57.0
    hooks:
      - id: trufflehog
        entry: trufflehog filesystem --fail
```

### 7. Code Review Metrics and Team Practices

```markdown
# Team Code Review Standards

## Review SLAs
- SEV fix reviews: Within 1 hour
- Regular PR reviews: Within 4 business hours
- Large PRs (> 300 lines): Within 1 business day

## Review Assignment
- Every PR requires at least 2 approvals
- At least one reviewer must be familiar with the changed code area
- The code owner for the directory must approve (CODEOWNERS file)

## PR Size Guidelines
- Ideal: < 200 lines of code
- Acceptable: < 400 lines of code
- Requires reviewer notification if > 400 lines
- Must be split into multiple PRs if > 600 lines (except auto-generated code)

## Review Quality Standards
- Reviewers must read every line they are assigned to
- Comments must be specific and actionable
- "Nit:" prefix for non-blocking suggestions
- "Blocking:" prefix for issues that must be fixed before merge
- Praise good patterns and solutions

## What to Automate
- Formatting (ruff, prettier, gofmt)
- Linting (ruff, eslint, golangci-lint)
- Type checking (mypy, tsc)
- Security scanning (gitleaks, Trivy)
- Dependency updates (Dependabot)
- Changelog updates (auto-generated from PR titles)

## What Humans Review
- Architecture and design decisions
- Business logic correctness
- Security logic (auth, authorization, data handling)
- Error handling and edge cases
- Test coverage and quality
- Readability and maintainability
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Reviewing only style, not substance | Bugs and security issues slip through | Focus on the priority pyramid top |
| Large PRs that nobody reviews thoroughly | Defects reach production | Enforce PR size limits |
| Rubber-stamping approvals | Review process is meaningless | Require substantive comments |
| Reviewer is the only author on the code | Knowledge silo, biased review | Require a reviewer familiar with the area |
| No automated checks | Humans waste time on mechanical issues | Linters, formatters, type checkers in CI |
| Review comments without explanation | Author doesn't understand the issue | Explain why, not just what |
| Ignoring test quality | Tests pass but don't catch real bugs | Review test cases, not just test count |
| Not checking for new dependencies | Unvetted packages enter the codebase | Require justification for new deps |
| Reviewing in isolation (no PR description) | Reviewer doesn't understand the problem | Require good PR descriptions |
| Code review as a bottleneck | Reviews take longer than writing code | Set review SLAs, distribute reviews |

## Banking-Specific Concerns

1. **Security Review Requirement** — Code that touches authentication, authorization, or data handling must be reviewed by the security team in addition to the engineering team.
2. **Compliance Code Paths** — Code that implements regulatory requirements (GDPR, PCI-DSS, AML) must include evidence of compliance review in the PR.
3. **Audit Trail** — All code reviews are part of the audit trail. PR discussions must be professional, clear, and free of ambiguous references.
4. **Change Advisory Board (CAB)** — Some changes require CAB approval in addition to code review. The PR description should note whether CAB approval is needed.
5. **Segregation of Duties** — The author of a change cannot be the person who approves it for production. Enforce this through CODEOWNERS and branch protection rules.

## GenAI-Specific Concerns

1. **Prompt Review** — Changes to prompt templates should be reviewed with the same rigor as code changes. Prompts are code. Review for: clarity, safety, grounding, and edge cases.
2. **Model Change Impact** — Changes to the model (version, provider, parameters) should include evaluation test results showing the impact on quality metrics.
3. **Guardrail Changes** — Any change to guardrail rules must be reviewed by both engineering and security/compliance teams.
4. **Evaluation Test Coverage** — PRs that affect GenAI behavior should include evaluation test results (RAGAS scores, hallucination rate, grounding score).
5. **Data Handling in Prompts** — Review that prompts do not inadvertently include PII or sensitive data. Check that input sanitization is applied before prompts.

## Metrics to Monitor

| Metric | Target | Why It Matters |
|--------|--------|----------------|
| PR size (median) | < 200 lines | Smaller PRs get better reviews |
| Review time (median) | < 4 hours | Fast feedback loop |
| Comments per PR | > 3 per 100 lines | Active engagement |
| Defects caught in review | > 80% of total | Review effectiveness |
| Review iteration count | < 2 per PR | Clear initial implementation |
| Automated check pass rate | > 95% | CI is reliable |
| Security issues found in review | Track trend | Security awareness |
| PRs requiring security review | Track percentage | Security engagement |

## Interview Questions

1. What do you look for when reviewing a PR that adds a new API endpoint?
2. How do you handle a PR where the solution is technically correct but the approach is overly complex?
3. What is the difference between a blocking and a non-blocking review comment?
4. How would you review a PR that changes the prompt template for a GenAI assistant?
5. What automated checks would you require before a PR can be reviewed by humans?
6. How do you balance review speed with review quality?
7. What would you do if you disagree with a reviewer's blocking comment?
8. How do you ensure that security considerations are reviewed in every PR?

## Hands-On Exercise

### Exercise: Review a PR for a GenAI RAG Service

**Problem:** You are asked to review the following PR that adds a new endpoint to the RAG service. The endpoint allows users to ask follow-up questions about a previously retrieved document.

**PR Description:**
```
## Problem
Users want to ask follow-up questions about specific documents returned
by the RAG search. Currently, they need to rephrase their entire query.

## Solution
Added POST /api/chat/followup endpoint that takes the previous search
results and the follow-up question, combines them into a prompt, and
calls the LLM.
```

**Code changes (simplified):**
```python
@router.post("/api/chat/followup")
async def followup(request: FollowupRequest):
    # Get the previous search results
    previous_results = redis.get(f"search:{request.session_id}")
    documents = json.loads(previous_results)

    # Build the prompt
    prompt = f"""Based on the following documents, answer the question:

{documents}

Question: {request.question}"""

    # Call the LLM
    response = await httpx.AsyncClient().post(
        "http://model-gateway.inference.svc/v1/chat",
        json={"messages": [{"role": "user", "content": prompt}]},
    )

    return {"answer": response.json()["choices"][0]["message"]["content"]}
```

**Constraints:**
- Review for security, correctness, performance, and design
- Identify all issues and write constructive comments
- Prioritize your comments by impact
- Suggest specific fixes

**Expected Output:**
- List of issues found, prioritized by severity
- Specific review comments with suggested fixes
- Identification of what was done well
- Decision: Approve, Request Changes, or Comment (with justification)

**Hints:**
- Look for security issues (input handling, data exposure)
- Look for error handling issues (what if Redis returns None?)
- Look for performance issues (unbounded document context)
- Look for design issues (hardcoded prompt format)
- Look for missing features (access control on previous results)

**Extension:**
- Write the improved version of the endpoint incorporating all review feedback
- Add unit tests and integration tests for the endpoint
- Write a runbook for common issues with this endpoint

---

**Related files:**
- `skills/secure-coding.md` — Secure coding practices
- `skills/secure-api-design.md` — API security review
- `testing-and-quality/code-review-automation.md` — Automated review tools
- `engineering-culture/code-review-culture.md` — Building a review culture
- `skills/genai-guardrails.md` — GenAI-specific security review
