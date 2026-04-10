# Pull Request Process

## Overview

Pull requests are the primary mechanism for code review and collaboration. In banking, PRs must ensure code quality, security compliance, and audit trails.

## PR Template

```markdown
## Description
<!-- What does this PR do and why? -->

## Change Type
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Infrastructure change
- [ ] Security fix

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed
- [ ] GenAI evaluation tests passed

## Security
- [ ] No new PII handling
- [ ] Security review required: Yes/No
- [ ] Secrets/credentials: None added

## Compliance
- [ ] Change ticket: [LINK]
- [ ] Rollback plan: [DESCRIBE]
- [ ] Monitoring impact: None/Describe

## Screenshots/Evidence
<!-- Add screenshots, logs, or metrics -->
```

## Review Requirements

```yaml
review_rules:
  minimum_approvals: 2
  code_owner_required: true
  ci_must_pass: true
  no_requested_changes: true
  stale_review_dismissal: true
  
  additional_requirements:
    - "Security-sensitive changes require security team review"
    - "Database migrations require DBA review"
    - "Infrastructure changes require SRE review"
    - "GenAI prompt changes require compliance review"

merge_strategies:
  preferred: squash
  allowed:
    - squash  # Most PRs
    - rebase  # For simple changes
  disabled:
    - merge  # No merge commits (keep linear history)
```

## Automated Checks

```yaml
# Required status checks before merge
required_checks:
  - name: CI Build
    description: "All code compiles and packages"
  - name: Unit Tests
    description: "All unit tests pass, coverage > 80%"
  - name: Integration Tests
    description: "Integration tests against staging services"
  - name: Security Scan
    description: "SAST, SCA, and secret detection pass"
  - name: Lint/Format
    description: "Code style and formatting checks"
  - name: GenAI Eval
    description: "Response quality and hallucination rate"
```

## Cross-References

- **Branching Strategies**: See [branching-strategies.md](branching-strategies.md) for branch models
- **Testing in Pipelines**: See [testing-in-pipelines.md](testing-in-pipelines.md) for test automation

## Interview Questions

1. **What should a good PR template include?**
2. **How many approvals should be required for a banking PR?**
3. **What automated checks should block PR merging?**
4. **How do you handle disagreements between reviewers?**
5. **What is the difference between squash, rebase, and merge strategies?**
6. **How do you ensure PR reviews are thorough but not slow?**

## Checklist: PR Process

- [ ] PR template used for all PRs
- [ ] Minimum 2 approvals required
- [ ] Code owners reviewed for relevant files
- [ ] All CI checks passing
- [ ] Description explains the "why" not just the "what"
- [ ] Tests included for new functionality
- [ ] Security review completed if applicable
- [ ] Rollback plan documented for infrastructure changes
- [ ] Linear history maintained (squash/rebase merge)
- [ ] PR linked to change management ticket
