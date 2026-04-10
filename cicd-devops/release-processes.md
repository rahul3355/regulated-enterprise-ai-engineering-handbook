# Release Processes: Versioning and Changelogs

## Overview

Release processes define how software versions are created, tracked, and communicated. This guide covers semantic versioning, changelog management, and release automation for banking GenAI platforms.

## Semantic Versioning

```
MAJOR.MINOR.PATCH (e.g., 2.1.0)

MAJOR: Breaking changes (API changes, data model changes)
MINOR: New features (backward-compatible)
PATCH: Bug fixes (backward-compatible)

Examples:
1.0.0 -> 1.0.1: Bug fix
1.0.1 -> 1.1.0: New feature
1.1.0 -> 2.0.0: Breaking change
```

## Automated Versioning

```yaml
# Conventional commits for automated versioning
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.release.outputs.version }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Semantic Release
        id: release
        uses: cycjimmy/semantic-release-action@v3
        with:
          extra_plugins: |
            @semantic-release/changelog
            @semantic-release/git
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# Conventional commit format:
# feat: add new embedding model (MINOR bump)
# fix: resolve tokenization bug (PATCH bump)
# feat!: change API contract (MAJOR bump)
# docs: update README (no bump)
# chore: update dependencies (no bump)
```

## Changelog

```markdown
# Changelog

## [1.2.0] - 2025-01-15

### Features
- Added support for text-embedding-3-large model
- Improved response relevance by 15%
- Added banking product knowledge base

### Bug Fixes
- Fixed tokenization issue with special characters
- Resolved timeout on large document uploads

### Security
- Updated dependencies with known vulnerabilities
- Added rate limiting for API endpoints

### Breaking Changes
- API v1 deprecated, use /api/v2

## [1.1.0] - 2025-01-01
...
```

## Cross-References

- **Release Communication**: See [release-communication.md](release-communication.md) for stakeholder updates
- **Change Management**: See [change-management.md](change-management.md) for regulated changes

## Interview Questions

1. **What is semantic versioning? How do you decide MAJOR vs MINOR vs PATCH?**
2. **How do you automate versioning from commit messages?**
3. **What should a changelog include?**
4. **How do you handle breaking changes in a banking API?**
5. **What is your release frequency for a production banking platform?**
6. **How do you coordinate releases across multiple services?**

## Checklist: Release Process

- [ ] Semantic versioning followed
- [ ] Version bumping automated
- [ ] Changelog auto-generated
- [ ] Release notes reviewed before publish
- [ ] Release tagged in git
- [ ] Artifacts published to registry
- [ ] Release communicated to stakeholders
- [ ] Previous version support period defined
- [ ] Release rollback procedure tested
