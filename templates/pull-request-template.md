# Pull Request Template

> Use this template for all PRs in the GenAI platform. It ensures consistent, thorough reviews that cover security, compliance, and operational concerns.

## Template

```markdown
# [Type]: [Brief Title]

> Types: feat / fix / docs / refactor / test / chore / security / infra

## What does this PR do?

[1-2 sentence description of the change. Link to Jira ticket if applicable.]

## Why is this change needed?

[Business or technical context. What problem does this solve?
What would happen if we DON make this change?]

## How was this tested?

- [ ] Unit tests added/updated (coverage: X%)
- [ ] Integration tests added/updated
- [ ] Manual testing performed (describe what was tested)
- [ ] Load/performance testing performed (if applicable)
- [ ] Tested in staging environment

### Test Details
[Describe the testing approach. What scenarios were tested?
What edge cases were considered?]

## Security & Compliance

- [ ] No new security implications
- [ ] Security implications documented below
- [ ] PII handling reviewed (if applicable)
- [ ] Audit logging reviewed (if applicable)
- [ ] Input validation reviewed (if applicable)
- [ ] Compliance team notified (if applicable)

### Security Details
[If any security-relevant changes, describe:
- What security controls are added/changed
- How input is validated/sanitized
- How secrets are handled
- How access is controlled]

## Change Type

- [ ] Breaking change (API, schema, behavior)
- [ ] Database migration included
- [ ] New dependency added
- [ ] Configuration change required
- [ ] Documentation update needed
- [ ] Infrastructure change required

## Rollback Plan

[How to revert this change if it causes issues in production.
If the rollback is non-trivial, describe the steps.]

## Deployment Notes

[Anything the deployment team needs to know:
- Feature flags to enable
- Environment variables to set
- Secrets to add
- Order of deployment (if multiple services)]

## Screenshots / Logs (if applicable)

[Visual evidence of the change working, or log output showing
the fix in action.]

## Reviewer Focus Areas

[Specific areas where you'd like reviewer attention:
- "Please pay special attention to the error handling in X"
- "I'm not confident about the retry logic in Y — does it look right?"
- "The access control check in Z is security-critical"]
```

## Example: Filled PR

```markdown
# feat: Add prompt injection detection to chat endpoint

**Jira:** GENAI-1234

## What does this PR do?

Adds prompt injection detection as a pre-processing step on the
chat endpoint. Queries flagged as potential injection attempts
are blocked with a 400 response and logged for security review.

## Why is this change needed?

The GenAI assistant is currently vulnerable to prompt injection
attacks. An attacker could craft queries that bypass access
controls, extract system prompts, or retrieve unauthorized
documents. This is a critical security control identified in
our recent threat model (see THREAT-2026-042).

## How was this tested?

- [x] Unit tests added (coverage: 94% for injection_detector.py)
- [x] Integration tests added (test endpoint blocks injection attempts)
- [x] Manual testing performed (tested 20 known injection patterns)
- [ ] Load testing performed — N/A (minimal performance impact)
- [x] Tested in staging environment

### Test Details
- Tested 50 safe queries to verify no false positives
- Tested 30 known injection patterns to verify detection
- Verified audit logging captures blocked attempts
- Verified response format matches error contract

## Security & Compliance

- [x] Security implications documented below
- [x] PII handling reviewed — no PII involved
- [x] Audit logging reviewed — blocked attempts are logged
- [x] Input validation reviewed — this IS input validation
- [ ] Compliance team notified — N/A (internal security control)

### Security Details
- New `PromptInjectionDetector` class with rule-based pattern matching
- Patterns cover: role-play override, system prompt extraction,
  data exfiltration, jailbreak attempts, command injection
- Risk scoring (0.0-1.0) with configurable threshold (default 0.7)
- Blocked attempts are logged to `injection_audit.log`
- Detection runs before any data is sent to the LLM

## Change Type

- [ ] Breaking change
- [ ] Database migration included
- [x] New dependency added (regex patterns in config file)
- [x] Configuration change required (INJECTION_THRESHOLD env var)
- [x] Documentation update needed (API docs updated)
- [ ] Infrastructure change required

## Rollback Plan

Simple: Revert this PR. The endpoint will return to accepting all
queries without injection screening. No data migration needed.

## Deployment Notes

1. Set `INJECTION_THRESHOLD=0.7` in all environments
2. No restart required — threshold is read from env on each request
3. Monitor `injection_blocked` metric in Grafana dashboard
4. Alert if false positive rate exceeds 1% (track via user reports)

## Reviewer Focus Areas

- `genai_service/injection_detector.py`: The pattern library is
  security-critical. Please review for coverage and false positive risk.
- `genai_service/chat.py`: Verify the injection check runs BEFORE
  the LLM call and doesn't affect normal query flow.
- `tests/test_injection_detector.py`: Check test coverage of
  edge cases (empty queries, non-English text, special characters).
```

## Banking PR Standards

### Required for All PRs
- [ ] CI passes (tests, linting, security scan)
- [ ] At least 1 approval from team member
- [ ] Security review for security-significant changes
- [ ] Compliance review for compliance-significant changes

### Additional for Database Migrations
- [ ] Migration is backward-compatible
- [ ] Tested on production-sized dataset
- [ ] Rollback migration exists and tested
- [ ] DBA team notified

### Additional for API Changes
- [ ] OpenAPI spec updated
- [ ] Backward-compatible or versioned
- [ ] Consumer teams notified of breaking changes
- [ ] API gateway configuration updated

### Additional for GenAI Changes
- [ ] Prompt changes are reviewed and versioned
- [ ] Model version is pinned (not "latest")
- [ ] Audit logging captures model version and prompt version
- [ ] Response quality evaluation considered
