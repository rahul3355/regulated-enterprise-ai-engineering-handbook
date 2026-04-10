# Code Review Culture

> "Code review is the single most effective engineering practice for improving code quality, sharing knowledge, and building a shared understanding of the system."

## Why Code Reviews Matter in Banking GenAI

In consumer software, a bad merge might cause a bug. In banking GenAI:

- A leaked API key in code → **credential compromise across all environments**
- A missing input validation → **prompt injection vulnerability**
- An unhandled edge case → **PII exposure in GenAI responses**
- A poorly designed database migration → **downtime during trading hours**

Code reviews are not bureaucracy. They are **the last line of defense before production.**

## Goals of Code Review

| Goal | Description | Banking Context |
|------|-------------|-----------------|
| **Catch defects** | Find bugs before they reach production | Especially security, compliance, and data-handling bugs |
| **Share knowledge** | Multiple people understand the code | Critical for systems that 100K+ employees depend on |
| **Enforce standards** | Consistent patterns across the codebase | Required for auditability and maintainability |
| **Mentor** | Junior engineers learn from seniors | Mentorship happens naturally through review comments |
| **Document decisions** | Review thread captures reasoning | Audit trail for "why was this done this way?" |

## The Review Mindset

### For Authors

1. **Your code is not your identity.** Criticism of your code is not criticism of you as an engineer.
2. **Small PRs get better reviews.** A 200-line PR gets thoughtful feedback. A 2000-line PR gets a rubber stamp.
3. **Context is king.** Your reviewer doesn't have the context you do. Provide it in the PR description.
4. **Review is asynchronous by default.** Don't expect instant review. Use the review process to improve the code, not defend it.
5. **You are responsible for follow-up.** Address every comment, even if it's "Agreed, will fix in a follow-up."

### For Reviewers

1. **You are reviewing the code, not the author.** Tone matters. "This should be..." not "Why did you...?"
2. **Assume positive intent.** The author is trying to solve a real problem.
3. **Ask questions, don't give commands.** "What happens if X?" is better than "You need to handle X."
4. **Distinguish between blocking and non-blocking.** Not every comment is a showstopper.
5. **Your job is to catch what the author can't see.** Fresh eyes find bugs the author has been staring at for hours.

## PR Size Guidelines

```
PR Size    | Expected Review Quality | Review Time | Risk
──────────────────────────────────────────────────────────
< 50 lines | Excellent               | 15 minutes  | Low
50-200     | Good                    | 30 minutes  | Medium
200-500    | Declining               | 1-2 hours   | High
500-1000   | Poor                    | 2-4 hours   | Very High
> 1000     | Effectively rubber      | 4+ hours    | Critical
```

**Rule of thumb:** If a PR touches more than 5 files, it should probably be split.

## PR Description Requirements

Every PR in the banking GenAI platform **must** include:

```markdown
## What does this PR do?
[1-2 sentence description of the change]

## Why is this change needed?
[Business or technical context. Link to Jira/Confluence if applicable.]

## How was this tested?
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed
- [ ] Load testing performed (for performance-sensitive changes)

## Security & Compliance Impact
- [ ] No new security implications
- [ ] Security implications documented and reviewed
- [ ] PII handling reviewed (if applicable)
- [ ] Audit logging reviewed (if applicable)
- [ ] Compliance team notified (if applicable)

## Rollback Plan
[How to revert this change if it causes issues in production]

## Screenshots / Logs (if applicable)
```

## Review Comment Etiquette

### Good Comments

```
# Asks a question that reveals a gap
"What happens if the model API returns a 429 response here?
 I don't see a retry mechanism."

# Points to a specific line with context
"Consider using `typing.Optional[str]` here instead of `str | None`
 to maintain consistency with the rest of the codebase. See
 `backend-engineering/skills/python-typing.md` for our conventions."

# Suggests an improvement with rationale
"Using `secrets.token_urlsafe(32)` here would be better than
 `uuid4()` for the API key, as it provides more entropy and
 is designed for cryptographic use."

# Flags a security concern clearly
"BLOCKING: This endpoint is missing input validation on the
 `user_query` parameter. An attacker could send a prompt
 injection attack through this field. See
 `security/prompt-injection.md` for the required checks."
```

### Bad Comments

```
# Vague
"Fix this."

# Nit-picking on style when there are bigger issues
"You missed a space after the comma on line 42."

# Prescriptive without explanation
"Use a class here."

# Condescending
"Obviously this should be async."
```

## Banking-Specific Review Checklist

### Security Review (every PR)

- [ ] No hardcoded credentials, API keys, or tokens
- [ ] All user inputs are validated and sanitized
- [ ] Database queries use parameterized statements (no SQL injection)
- [ ] API endpoints have authentication and authorization checks
- [ ] Sensitive data is not logged or exposed in error messages
- [ ] Dependencies are pinned to specific versions
- [ ] No new dependencies without security review approval
- [ ] Rate limiting is in place for public-facing endpoints

### GenAI-Specific Review (for GenAI PRs)

- [ ] Prompts are validated for injection patterns
- [ ] PII detection is in place for both inputs and outputs
- [ ] Model responses are bounded (max tokens, timeout)
- [ ] Fallback behavior is defined for model failures
- [ ] Audit logging captures prompt, response, and model version
- [ ] Citations are included in RAG responses
- [ ] Response content filtering is applied
- [ ] Model version is pinned, not using `latest`

### Database Review (for migration PRs)

- [ ] Migration is backward-compatible (can run with old code)
- [ ] No `DROP COLUMN` without a deployment cycle
- [ ] New indexes are appropriate for query patterns
- [ ] Large data migrations are chunked and resumable
- [ ] Migration has been tested on a production-sized dataset
- [ ] Rollback migration exists and has been tested

## Example: Real Review from Production

Here's an abbreviated real review from our RAG pipeline:

```
PR: Add document-level access control to RAG retriever
Author: @sarah.chen
Reviewers: @james.wu (security), @priya.patel (backend)

---

@james.wu (security):
  "I see the `check_user_access()` call on line 87, but I don't see
   the corresponding test. Can you add a test that verifies a user
   CANNOT retrieve documents they don't have access to? This is the
   core security property we need to validate."

Author: "Good catch. Added test_user_cannot_access_restricted_docs()
         which confirms the retriever filters correctly."

---

@priya.patel (backend):
  "The N+1 query pattern here is concerning. For each retrieved
   document, we're making a separate DB call to check access. With
   50 documents in a retrieval batch, that's 51 queries. Can we
   batch this into a single query with a JOIN or subquery?"

Author: "Refactored to use a single query with EXISTS subquery.
         Query time dropped from 2.3s to 45ms for 50 docs."

---

@james.wu (security):
  "BLOCKING: The `embedding_model_version` is being passed from
   user input on line 134. This should come from a configuration
   constant or environment variable. An attacker could specify
   arbitrary model names to probe our system."

Author: "Fixed. Now using MODEL_VERSION from settings, validated
         at startup."

---

Review outcome: Approved after 3 rounds of feedback.
Time to merge: 4 hours (asynchronous).
Production issues: None.
```

## Review Turnaround Expectations

| PR Priority | Expected Review Time | Escalation After |
|-------------|---------------------|------------------|
| Hotfix (P0) | 30 minutes | 1 hour (page on-call reviewer) |
| Critical (P1) | 2 hours | 4 hours |
| Normal | 24 hours | 48 hours |
| Low (docs, chores) | 48 hours | 1 week |

**Rule:** If a PR has no reviewers after the escalation time, the author should escalate to their tech lead.

## When to Skip Review

Almost never. The only exceptions:

1. **Hotfix for a P0 production incident** — Review immediately after merge. Document why review was skipped.
2. **Automated dependency updates** (e.g., Renovate bot) — Only if tests pass and the change is well-understood.
3. **Documentation-only changes** — No code changes. Still needs an approval, but can be lightweight.

**Never skip review for:** Security changes, database migrations, authentication/authorization code, GenAI prompt changes, infrastructure changes.

## Code Review as Mentorship

For junior engineers, code reviews are one of the highest-leverage learning opportunities.

### For Senior Reviewers

- Explain **why**, not just **what**. "Use a context manager here because it guarantees cleanup even if an exception is raised" is better than "Use a context manager."
- Link to documentation. Point to `skills/python-async-patterns.md` or external docs.
- Ask guiding questions. "What happens to this connection if the function throws?" helps them think through the problem.
- Acknowledge good patterns. "Nice use of the circuit breaker pattern here" reinforces good behavior.

### For Junior Authors

- Don't take comments personally. Every comment is an opportunity to learn.
- Ask "why" if you don't understand a suggestion.
- Thank reviewers for thorough feedback.
- Keep a "lessons learned" note from recurring review comments.

## Anti-Patterns to Avoid

| Anti-Pattern | What It Looks Like | Why It's Bad |
|--------------|-------------------|--------------|
| **LGTM without reading** | "Looks good to me" on a 500-line PR | Defeats the purpose of review |
| **Bike-shedding** | 20 comments about variable names, none about the architecture | Wastes time on low-impact issues |
| **Review by committee** | 8 reviewers all asking for different changes | Paralysis by analysis |
| **Drive-by blocking** | A reviewer from another team blocking without context | Creates friction and resentment |
| **Review hoarding** | Only one person reviews everything | Knowledge bottleneck |
| **Review avoidance** | Splitting a big change into many small PRs to avoid scrutiny | Defeats the purpose of review |

## Metrics We Track (and Don't Track)

### We Track
- **Review turnaround time** — Are reviews happening fast enough?
- **Review depth** — Average comments per PR (aim for 3-8)
- **PR size distribution** — Are PRs staying small?
- **Review participation** — Is everyone both reviewing and being reviewed?

### We DO NOT Track
- **Number of comments as a performance metric** — This incentivizes nit-picking
- **Review time as a speed metric** — Fast reviews aren't always good reviews
- **Number of PRs as a productivity metric** — Encourages artificial splitting

## Tools and Automation

- **GitHub CODEOWNERS** — Automatically assign reviewers based on file paths
- **Semantic PR titles** — Enforced by CI for changelog generation
- **Pre-commit hooks** — Catch formatting, linting, and basic security issues before review
- **CI gates** — Tests, coverage, and security scans must pass before merge
- **Review bot** — Automatically flags common issues (missing tests, hardcoded secrets)

## Cross-References

- `skills/code-review-checklist.md` — Detailed review checklist
- `security/secure-coding.md` — Security requirements for all code
- `templates/pull-request-template.md` — PR description template
- `engineering-philosophy/craft.md` — Engineering craft and quality standards
- `leadership-and-collaboration/coaching-junior-engineers.md` — Using review as coaching
