# Template Usage Guide

## Purpose

These templates provide starting points for common engineering artifacts in the Banking GenAI organization. Using templates ensures consistency, reduces cognitive load, and helps new engineers produce professional-quality documentation quickly.

## When to Use Templates

Templates are guides, not straitjackets. Use them when:

- You are creating a new document and do not know where to start
- You want to ensure all required sections are included
- You are preparing for a formal review (architecture review, compliance audit)
- You want consistent formatting across the organization

Skip or modify sections when they do not apply to your specific situation.

## Available Templates

### Engineering Artifacts

| Template | When to Use | File |
|----------|-------------|------|
| Design Document | Before building a non-trivial feature | [design-doc-template.md](design-doc-template.md) |
| RFC | Proposing a significant change | [rfc-template.md](rfc-template.md) |
| ADR | Documenting an architecture decision | [adr-template.md](adr-template.md) |
| Pull Request | Opening a PR for review | [pull-request-template.md](pull-request-template.md) |

### Operational Artifacts

| Template | When to Use | File |
|----------|-------------|------|
| Runbook | Documenting operational procedures | [runbook-template.md](runbook-template.md) |
| On-Call Handoff | Transitioning on-call responsibility | [oncall-handoff-template.md](oncall-handoff-template.md) |
| Release Plan | Planning a production release | [release-plan-template.md](release-plan-template.md) |

### Incident Artifacts

| Template | When to Use | File |
|----------|-------------|------|
| Incident Report | During/after an incident | [incident-report-template.md](incident-report-template.md) |
| Postmortem | After incident resolution | [postmortem-template.md](postmortem-template.md) |

### Review Artifacts

| Template | When to Use | File |
|----------|-------------|------|
| Architecture Review | Presenting design for feedback | [architecture-review-template.md](architecture-review-template.md) |
| Threat Model | Security analysis of a system | [threat-model-template.md](threat-model-template.md) |
| Compliance Review | Regulatory compliance check | [compliance-review-template.md](compliance-review-template.md) |

### Communication Artifacts

| Template | When to Use | File |
|----------|-------------|------|
| Stakeholder Update | Regular stakeholder communication | [stakeholder-update-template.md](stakeholder-update-template.md) |
| Executive Summary | Summarizing for leadership | [executive-summary-template.md](executive-summary-template.md) |

### Team Rituals

| Template | When to Use | File |
|----------|-------------|------|
| Sprint Planning | Starting a new sprint | [sprint-planning-template.md](sprint-planning-template.md) |
| Retrospective | End of sprint retrospective | [retrospective-template.md](retrospective-template.md) |

## Template Customization

Each template includes:
- **Required sections**: Must be present for the document to be complete
- **Optional sections**: Include if relevant to your situation
- **Banking-specific sections**: Required for banking compliance

### Example: Customizing the Design Doc Template

```markdown
# Standard template sections:

## Problem Statement          [REQUIRED]
## Proposed Solution          [REQUIRED]
## Alternatives Considered    [REQUIRED]
## Implementation Plan        [REQUIRED]
## Risks and Mitigations      [REQUIRED]
## Compliance Impact          [REQUIRED for banking]
## Observability Plan         [REQUIRED for production services]
## Rollback Plan              [REQUIRED for production services]
## Cost Estimate              [OPTIONAL - add if significant cost]
## Team Impact                [OPTIONAL - add if staffing changes]
```

## Storage Conventions

Store completed templates in their appropriate locations:

```
service-repo/
├── docs/
│   ├── design-docs/
│   │   └── mortgage-advisor-redesign.md
│   ├── runbooks/
│   │   └── high-latency-troubleshooting.md
│   └── postmortems/
│       └── 2025-03-12-vector-db-outage.md

architecture/
├── decisions/
│   └── 042-use-pgvector.md
├── threat-models/
│   └── genai-platform.md
└── reviews/
    └── 2025-Q1-mortgage-advisor.md
```

## Template Quality Checklist

When completing any template, verify:

- [ ] All required sections are present
- [ ] Content is specific (not generic or placeholder text)
- [ ] Banking-specific sections are completed for production services
- [ ] Links to related documents are included
- [ ] Document has an owner and a date
- [ ] Review/approval section is completed (if required)
