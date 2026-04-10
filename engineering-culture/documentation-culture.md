# Documentation Culture

> "If it isn't documented, it didn't happen. If it's documented but wrong, it's worse than not documented at all."

## Documentation as Engineering Responsibility

In a banking GenAI platform, documentation is not an optional extra. It is an engineering deliverable.

**Code without documentation is incomplete code.** A feature is not "done" when it works. It is "done" when it works AND is documented.

This is not unique to banking — it's true of all professional engineering. But in banking, the stakes are higher:

- **Auditors** will read our documentation
- **Compliance teams** will verify our processes against it
- **New engineers** will learn the system from it
- **On-call engineers** will debug incidents using it at 3 AM
- **Security teams** will assess our risk based on it

## The Documentation Pyramid

```
                    ┌──────────────────┐
                    │   Tutorials      │  Step-by-step guides for beginners
                    ├──────────────────┤
                    │   How-To Guides  │  Goal-oriented instructions
                    ├──────────────────┤
                    │   Reference      │  Technical specs, API docs, schemas
                    ├──────────────────┤
                    │   Architecture   │  Design docs, ADRs, system diagrams
                    ├──────────────────┤
                    │   Runbooks       │  Operational procedures, incident guides
                    └──────────────────┘
```

Each layer serves a different audience and has different quality requirements.

## What Must Be Documented

### Before Production Deployment

| Document | Owner | Reviewer | Audience |
|----------|-------|----------|----------|
| Architecture design doc | Author | Tech lead + affected teams | Engineering |
| API documentation (OpenAPI) | Author | API team | Engineering + consumers |
| Runbook | Author | On-call team | Operations |
| Deployment guide | Author | Platform team | Engineering + DevOps |
| Security review | Security reviewer | Security team | Security + Engineering |
| Compliance review | Compliance reviewer | Compliance team | Compliance + Engineering |
| Monitoring dashboard | Author | SRE team | Operations |

### Ongoing Documentation

| Document | Update Trigger | Owner |
|----------|---------------|-------|
| Architecture diagrams | System changes | Team |
| API specs | API changes | API owner |
| Runbooks | Incident learnings, system changes | On-call lead |
| Onboarding docs | Process changes | Engineering manager |
| Glossary | New terms discovered | Team |
| Troubleshooting guides | New failure patterns discovered | Team |

## Documentation Standards

### Code Documentation

**Docstrings:** Every public function, class, and module must have a docstring:

```python
async def generate_response(
    prompt: str,
    user_id: str,
    max_tokens: int = 1024,
    temperature: float = 0.7
) -> ChatResponse:
    """Generate a response from the LLM with safety guardrails.

    This function handles prompt validation, safety checks
    (injection detection, PII redaction), LLM API call with
    retry logic, and response filtering.

    Args:
        prompt: The user's input prompt (already validated for length).
        user_id: Hashed user identifier for audit logging.
        max_tokens: Maximum tokens in the response. Default 1024.
        temperature: Sampling temperature. Default 0.7.

    Returns:
        ChatResponse containing the generated text, token usage,
        and safety flags.

    Raises:
        PromptInjectionError: If prompt injection is detected.
        ModelAPIError: If the LLM API is unavailable after retries.
        ValidationError: If parameters are out of range.

    Example:
        >>> response = await generate_response(
        ...     prompt="What is our travel policy?",
        ...     user_id="abc123",
        ...     max_tokens=512
        ... )
    """
```

### README Requirements

Every repository and significant subdirectory must have a README:

```markdown
# [Project/Component Name]

## What is this?
[One paragraph description]

## Who is this for?
[Target audience]

## Quick Start
[How to get running in 5 minutes or less]

## Architecture
[Brief description with diagram]

## How to Deploy
[Link to deployment guide]

## How to Monitor
[Link to dashboard and key metrics]

## How to Debug
[Common issues and how to diagnose them]

## Dependencies
[What this depends on, and version requirements]

## Contributing
[How to contribute, coding standards, review process]

## Contacts
[Who owns this, who to contact for questions]
```

### Runbook Requirements

See `templates/runbook-template.md` for the full template.

Every production service must have a runbook that answers:
- What is this service?
- How do I know if it's healthy?
- What are the common alerts and what do they mean?
- What are the troubleshooting steps for each alert?
- Who do I escalate to?
- What are the known issues?

## Documentation Ownership

### Every Document Has an Owner

| Role | Responsibility |
|------|---------------|
| **Author** | Creates the initial document |
| **Owner** | Maintains accuracy over time (usually the team that owns the system) |
| **Reviewer** | Validates accuracy (security team for security docs, etc.) |

### Ownership Transfer

When a system changes hands, documentation ownership transfers too:

1. Old owner reviews docs with new owner
2. New owner validates all docs are accurate
3. New owner updates any outdated sections
4. Ownership is formally transferred in the doc header

## Documentation Quality Bar

### The "New Engineer" Test

Documentation passes the quality bar if a new engineer, on their second day, can read it and:
- Understand what the system does
- Set up a local development environment
- Deploy to staging
- Find and fix a common production issue
- Know who to ask for help

### The "3 AM" Test

Runbook documentation passes the quality bar if an on-call engineer, half-asleep at 3 AM, can:
- Understand what's happening from the alert
- Follow troubleshooting steps without guessing
- Know when to escalate
- Not make things worse

### The "Auditor" Test

Compliance documentation passes the quality bar if an external auditor can:
- Understand our processes
- Verify that we follow them
- Trace decisions back to their rationale
- See evidence of regular review and updates

## Keeping Documentation Current

### Documentation as Part of the PR

Every PR that changes system behavior must also update relevant documentation:

```markdown
## Documentation Updates
- [ ] README updated (if applicable)
- [ ] API docs updated (if API changed)
- [ ] Runbook updated (if operational behavior changed)
- [ ] Architecture diagram updated (if architecture changed)
- [ ] Changelog updated
```

### Documentation Review in Code Review

Reviewers should check documentation as part of code review:

- "The API response format changed — is the OpenAPI spec updated?"
- "This adds a new alert — is the runbook updated with troubleshooting steps?"
- "This changes the deployment process — is the deployment guide updated?"

### Periodic Documentation Audits

Every quarter, each team spends 2 hours on a **documentation audit**:

1. **Identify stale docs:** Last updated > 90 days ago
2. **Verify accuracy:** Does the doc match the actual system?
3. **Update or archive:** Update if still relevant, archive if obsolete
4. **Fill gaps:** What's missing that should be documented?

### Stale Documentation Detection

```markdown
<!-- This document was last reviewed on 2025-12-15.
     If the current date is more than 90 days later,
     this document may be stale. Please verify accuracy
     and update if needed. -->
```

## Documentation Tools

| Tool | Purpose | Location |
|------|---------|----------|
| Markdown files | Design docs, ADRs, guides | This repository |
| OpenAPI/Swagger | API specifications | API gateway + repo |
| Mermaid diagrams | Architecture diagrams | Embedded in markdown |
| Confluence | Team pages, meeting notes | Internal wiki |
| Grafana dashboards | Operational documentation | Monitoring platform |
| dbt docs | Data pipeline documentation | Data engineering repo |

## Documentation Anti-Patterns

| Anti-Pattern | What It Looks Like | Why It's Dangerous |
|--------------|-------------------|-------------------|
| **Write once, never update** | Doc says v1.0, system is on v4.2 | Misleading, worse than no docs |
| **Docs in someone's head** | "Oh yeah, you need to also set X" | Bus factor of 1 |
| **Docs only in tickets** | Critical information buried in Jira comments | Not findable by new engineers |
| **Over-documented trivialities** | 10 pages explaining a 5-line function | Wastes time, obscures important docs |
| **Copy-paste docs** | README from another project, never customized | Actively misleading |
| **Screenshot documentation** | Docs are screenshots of text | Not searchable, not linkable |

## Documentation Metrics

| Metric | Target | Why |
|--------|--------|-----|
| Docs updated within 90 days | > 85% | Currency |
| New services have runbooks before production | 100% | Operational readiness |
| New engineers can deploy unaided | Within 2 weeks | Onboarding effectiveness |
| Documentation audit completion | 100% per quarter | Maintenance discipline |
| On-call escalation due to missing runbook | 0 | Runbook coverage |

## Documentation as Career Growth

Contributing to documentation is valued in performance reviews:

| Level | Documentation Expectation |
|-------|--------------------------|
| **Junior** | Documents own work, updates docs when changing systems |
| **Mid** | Writes design docs, improves team documentation quality |
| **Senior** | Sets documentation standards, mentors others on documentation |
| **Staff** | Cross-team documentation initiatives, architecture documentation |
| **Principal** | Industry-level documentation (open source docs, public posts) |

**Note:** Writing clear, concise documentation is a skill that improves with practice. If you struggle with writing, that's a skill gap worth addressing — not a reason to avoid documentation.

## The Documentation Investment Framework

Not all documentation is equally valuable. Invest proportionally:

| Documentation | Investment Level | Why |
|---------------|-----------------|-----|
| Runbooks for production services | High | Used during incidents at 3 AM |
| Architecture decision records | Medium | Referenced during future design decisions |
| API specifications | High | Used by consumers to integrate |
| Onboarding guides | High | Used by every new engineer |
| Internal design docs | Medium | Referenced during implementation |
| Troubleshooting guides | High | Used during debugging |
| Meeting notes | Low | Rarely referenced after 2 weeks |
| Ad-hoc technical notes | Low | Personal reference |

## Cross-References

- `engineering-culture/design-docs.md` — Design document process
- `engineering-culture/async-communication.md` — Written communication practices
- `templates/runbook-template.md` — Runbook template
- `templates/architecture-review-template.md` — Architecture review template
- `engineering-philosophy/documentation-is-engineering.md` — Philosophy behind documentation
- `onboarding/new-engineer-checklist.md` — Onboarding documentation requirements
