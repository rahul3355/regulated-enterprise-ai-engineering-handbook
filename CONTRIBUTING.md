# Contributing Guide

Thank you for your interest in contributing to the **Regulated Enterprise AI Engineering Handbook**.

This handbook exists to help engineers build secure, compliant, production-grade AI systems in regulated environments. Every contribution improves the quality, accuracy, and usefulness of this resource for thousands of engineers.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [What We're Looking For](#what-were-looking-for)
- [How to Contribute](#how-to-contribute)
- [Content Standards](#content-standards)
- [Writing Guide](#writing-guide)
- [Technical Content Requirements](#technical-content-requirements)
- [Pull Request Process](#pull-request-process)
- [Review Process](#review-process)
- [Development Setup](#development-setup)
- [Reporting Issues](#reporting-issues)

## Code of Conduct

Please read and follow our [Code of Conduct](CODE_OF_CONDUCT.md). We expect all contributors to maintain a respectful, inclusive environment.

## What We're Looking For

### High-Value Contributions

- **New content** in areas that are underdeveloped or missing
- **Corrections** to factual errors, outdated guidance, or broken links
- **Code examples** that are production-grade and banking-appropriate
- **Exercises** with complete solutions and extensions
- **Case studies** from real production experience
- **Templates** that engineers can use directly
- **Cross-references** between related sections
- **Glossary terms** with clear definitions and examples
- **Mermaid diagrams** that clarify complex architectures

### What We Don't Accept

- Opinion pieces without technical substance
- Vendor-specific tutorials or product promotion
- Theoretical content without practical application
- Code examples that aren't production-ready
- Content that duplicates existing sections

## How to Contribute

### Small Changes (Typos, Fixes, Clarifications)

1. Navigate to the file on GitHub
2. Click **Edit this file** (pencil icon)
3. Make your change
4. Write a clear commit message
5. Open a pull request

### Larger Changes (New Content, Major Rewrites)

1. **Open an issue first** using the [Feature Request](.github/ISSUE_TEMPLATE/feature_request.md) or [Bug Report](.github/ISSUE_TEMPLATE/bug_report.md) template
2. Discuss the scope and approach with maintainers
3. Once agreed, fork the repository
4. Make your changes on a feature branch
5. Submit a pull request using the [PR template](.github/PULL_REQUEST_TEMPLATE.md)

## Content Standards

### Writing Style

- **Direct and practical** — Avoid academic language. Write for working engineers.
- **Opinionated but fair** — State your position clearly, acknowledge tradeoffs.
- **Example-driven** — Every concept should have a concrete example.
- **Banking-context** — Where possible, use banking, fintech, or regulated environment examples.
- **Second person** — Use "you" rather than "one" or "the engineer."

### Markdown Conventions

- Use ATX-style headers (`## Header`, not `Header\n======`)
- Use fenced code blocks with language tags
- Use Mermaid for diagrams (test rendering locally)
- Use tables for comparison data
- Use internal relative links: `[Section](../section/file.md)`
- Maximum line length: 120 characters (soft wrap)

### File Naming

- Use `kebab-case` for all file names
- Use descriptive, action-oriented names where appropriate
- Prefix exercises: `coding-exercise-01-rate-limiter.md`

## Writing Guide

### Every New File Should Include

1. **Core Principles** (3-5 key principles)
2. **Mental Models** (frameworks, checklists, decision trees)
3. **Step-by-Step Approach** (practical implementation with code)
4. **Common Pitfalls** (table of pitfall → consequence → fix)
5. **Banking-Specific Concerns** (where applicable)
6. **GenAI-Specific Concerns** (where applicable)
7. **Metrics to Monitor** (table of metric → threshold → why it matters)
8. **Interview Questions** (5-8 questions with strong answer guidance)
9. **Hands-On Exercise** (problem, constraints, expected output, hints, solution, extension)
10. **Related Files** (cross-references to other handbook sections)

### Code Example Standards

All code examples must be:
- **Complete** — Not fragments that won't compile or run
- **Production-grade** — Error handling, logging, validation included
- **Commented** — Explain the "why," not the "what"
- **Secure** — No hardcoded secrets, parameterized queries, input validation
- **Banking-appropriate** — Use banking domain models and terminology

### Banking Context Requirements

When writing banking-specific content:
- Use real banking terminology correctly (KYC, AML, SAR, Basel III, etc.)
- Reference actual regulations when applicable (GDPR Article 5, PCI-DSS Req 3, etc.)
- Include concrete scale (transaction volumes, user counts, data sizes)
- Reference actual fines or incidents when illustrating risk

## Technical Content Requirements

### Accuracy

- All technical claims must be verifiable
- Code examples must be tested or reviewed by someone familiar with the technology
- Version numbers and API references must be current
- If uncertain, mark content as `[TODO: Verify]` rather than guessing

### Cross-References

- Link to related sections within the handbook
- Update cross-references when moving or renaming files
- At minimum, reference: the relevant `security/`, `testing-and-quality/`, and `observability/` sections

### Diagrams

- Use Mermaid for all architecture diagrams
- Test diagrams render correctly on GitHub
- Include text descriptions for accessibility
- Keep diagrams focused and readable

## Pull Request Process

1. **Fork** the repository
2. **Create a branch** from `main`: `git checkout -b feat/add-prompt-testing-guide`
3. **Make your changes** following the content standards above
4. **Validate locally:**
   ```bash
   # Run markdown lint
   npx markdownlint-cli2 "**/*.md"
   
   # Run spellcheck
   npx cspell "**/*.md"
   
   # Check links
   lychee "**/*.md"
   ```
5. **Commit** with a descriptive message (see conventions below)
6. **Push** to your fork
7. **Open a PR** using the [PR template](.github/PULL_REQUEST_TEMPLATE.md)

### Commit Message Conventions

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): description

[optional body]

[optional footer]
```

**Types:**
- `docs:` — Documentation content changes
- `fix:` — Bug fixes
- `feat:` — New content or features
- `refactor:` — Restructuring without behavior change
- `style:` — Formatting, typos, presentation
- `ci:` — CI/CD workflow changes
- `chore:` — Maintenance, tooling

**Examples:**
```
docs(genai): add model evaluation framework guide
fix(security): correct OAuth2 flow diagram
feat(exercises): add prompt injection defense exercise
style(README): fix heading levels and broken links
ci: add spellcheck workflow
```

## Review Process

All PRs require at least **one approval** from a code owner before merging.

### Review Criteria

1. **Technical accuracy** — Is the content correct and current?
2. **Practical value** — Will engineers actually use this?
3. **Banking appropriateness** — Does it reflect regulated environment realities?
4. **Writing quality** — Is it clear, well-organized, and accessible?
5. **Security soundness** — Are all code examples secure?
6. **Completeness** — Does it cover the topic adequately?

### Review Timeline

- Small PRs (typos, fixes): 1-3 business days
- Medium PRs (new files, significant additions): 3-7 business days
- Large PRs (major sections, new categories): 1-2 weeks

## Development Setup

For local validation:

```bash
# Install validation tools
npm install -g markdownlint-cli2
npm install -g cspell
# Install lychee (link checker)
cargo install lychee
# Or use the brew package
brew install lychee

# Run all checks
npx markdownlint-cli2 "**/*.md"
npx cspell "**/*.md"
lychee "**/*.md"
```

## Reporting Issues

- **Content errors:** Use the [Bug Report](.github/ISSUE_TEMPLATE/bug_report.md) template
- **Security concerns:** Use the [Security Issue](.github/ISSUE_TEMPLATE/security_issue.md) template or email maintainers directly
- **Questions:** Use the [Question](.github/ISSUE_TEMPLATE/question.md) template
- **Suggestions:** Use the [Feature Request](.github/ISSUE_TEMPLATE/feature_request.md) template

## Thank You

Every contribution — from a single typo fix to a major new section — makes this handbook more valuable to engineers building critical systems. We appreciate your time and expertise.
