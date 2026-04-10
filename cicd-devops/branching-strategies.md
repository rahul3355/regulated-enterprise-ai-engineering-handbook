# Branching Strategies for Banking Software

## Overview

Branching strategies define how developers organize code changes, collaborate, and integrate work. In banking, branching must support code review, compliance tracking, and controlled releases.

## Trunk-Based Development (Recommended)

```
main ──┬── A ── B ── C ── D ── E (production)
       │
       ├── short-lived feature branches (max 1-2 days)
       └── direct commits for small changes
       
Rules:
- main is always deployable
- Feature branches merged within 1-2 days
- CI runs on every commit to main
- Feature flags for incomplete features
```

## GitFlow

```
main ────────── v1.0 ────── v1.1 ──── (releases)
  │              │           │
develop ──┬── A ── B ── C ── D ──── (integration)
          │         │
          └─feature1 └─ feature2
          
Rules:
- develop for integration
- main for releases only
- feature branches from develop
- release branches for stabilization
- hotfix branches from main
```

## GitHub Flow (Simple)

```
main ── A ── B ── C ── D ── E
        │         │
        PR1       PR2
        
Rules:
- Branch from main for features
- PR with review
- Merge to main
- Deploy from main
```

## Branch Protection Rules

```yaml
# GitHub branch protection for main
branch_protection:
  main:
    require_pull_request: true
    required_approving_review_count: 2
    require_code_owner_review: true
    dismiss_stale_reviews: true
    require_status_checks:
      strict: true  # Branch must be up to date
      checks:
        - ci-build
        - unit-tests
        - security-scan
        - integration-tests
    enforce_admins: true
    require_linear_history: true  # No merge commits
    allow_force_pushes: false
    allow_deletions: false
```

## Cross-References

- **PR Process**: See [pull-request-process.md](pull-request-process.md) for review requirements
- **CI/CD Design**: See [ci-cd-design.md](ci-cd-design.md) for pipeline integration

## Interview Questions

1. **Compare trunk-based development with GitFlow. Which do you prefer for banking?**
2. **What branch protection rules should be enforced for a banking codebase?**
3. **How do you handle long-running feature development in trunk-based development?**
4. **What is the role of feature flags in branching strategy?**
5. **How do you manage releases across multiple environments?**
6. **What happens when a hotfix is needed in production?**

## Checklist: Branching Strategy

- [ ] Strategy chosen and documented (trunk-based recommended)
- [ ] Branch protection rules enforced
- [ ] Required review count set (minimum 2)
- [ ] CI status checks required before merge
- [ ] Force push disabled on main
- [ ] Linear history enforced
- [ ] Feature flag strategy for incomplete features
- [ ] Branch naming convention documented
- [ ] Stale branch cleanup automated
- [ ] Hotfix process documented
