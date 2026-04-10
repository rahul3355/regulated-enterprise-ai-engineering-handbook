# Threat Modeling Exercise: GenAI RAG System

> Perform a threat model for a Retrieval-Augmented Generation system used in a banking environment.

## Problem Statement

You are the security lead for the GenAI platform. The team has built a RAG-based internal assistant that:

1. Accepts employee queries via a web interface
2. Retrieves relevant documents from a vector database
3. Sends query + documents to an external LLM API
4. Returns a response with citations

Perform a threat model using the **STRIDE** framework. Identify threats, assess risk, and recommend mitigations.

## System Description

```
Employee → Web App (React) → API Gateway → GenAI Service → LLM API (external)
                                    ↓
                              Vector DB (pgvector)
                                    ↑
                              Document Store (PostgreSQL)
                                    ↓
                              Audit Logger → Audit DB
```

Data flows:
- Employee authenticates via corporate SSO (SAML)
- API Gateway validates JWT tokens
- GenAI Service retrieves documents the employee is authorized to access
- Query + documents are sent to external LLM API over TLS
- Response is content-filtered before returning to employee
- All interactions are logged to audit database

## STRIDE Framework

| Threat Category | What It Means | GenAI Example |
|----------------|---------------|---------------|
| **S**poofing | Pretending to be someone else | Stealing another employee's JWT token |
| **T**ampering | Modifying data or code | Altering retrieved documents before sending to LLM |
| **R**epudiation | Denying an action occurred | Employee denies making a specific query |
| **I**nformation Disclosure | Exposing data to unauthorized parties | LLM API logs containing confidential bank data |
| **D**enial of Service | Making the system unavailable | Flooding the API with queries to exhaust LLM quota |
| **E**levation of Privilege | Gaining unauthorized access | Prompt injection to access restricted documents |

## Expected Deliverables

For each STRIDE category, provide:

1. **Threat description** — Specific attack scenario
2. **Affected component** — Which part of the system is vulnerable
3. **Impact** — What happens if the threat is realized (Low/Medium/High/Critical)
4. **Likelihood** — How likely is this attack (Low/Medium/High)
5. **Mitigation** — Specific technical control to reduce risk
6. **Residual risk** — Risk level after mitigation

## Example Threat Analysis

```
Category: Spoofing
Threat: An attacker steals an employee's JWT token from their browser's
        local storage and uses it to query the GenAI assistant, gaining
        access to all documents the employee is authorized to see.
Component: API Gateway / JWT validation
Impact: High — Unauthorized access to internal documents
Likelihood: Medium — Token theft via XSS is common
Mitigation:
  1. Store tokens in httpOnly, secure cookies (not localStorage)
  2. Implement short token TTL (15 min) with refresh tokens
  3. Bind tokens to device fingerprint
  4. Implement Content Security Policy to prevent XSS
Residual Risk: Low
```

## Your Task: Complete the Threat Model

Find at least 2 threats per STRIDE category (12 total minimum). Include:

### Must-Cover Scenarios

- Prompt injection to access unauthorized documents
- Data exfiltration through the LLM API provider
- Vector database access without authentication
- Audit log tampering
- Model response manipulation
- Document store data leakage

## Deliverable Format

```markdown
# Threat Model: GenAI RAG System
# Date: [Date]
# Author: [Name]
# Classification: Confidential

## System Overview
[Architecture diagram and data flow description]

## Trust Boundaries
[Identify trust boundaries between components]

## Threat Analysis

### Spoofing
| # | Threat | Component | Impact | Likelihood | Mitigation | Residual |
|---|--------|-----------|--------|-----------|-----------|----------|
| 1 | [...] | [...] | High | Medium | [...] | Low |

### Tampering
[...] (same format for each category)

## Risk Summary
| Risk Level | Count |
|------------|-------|
| Critical | X |
| High | X |
| Medium | X |
| Low | X |

## Recommendations
[Prioritized list of mitigations to implement]

## Open Questions
[Unresolved concerns needing further analysis]
```

## Extensions

1. **Attack tree:** Build an attack tree for the highest-risk threat scenario.

2. **DREAD scoring:** Apply DREAD (Damage, Reproducibility, Exploitability, Affected users, Discoverability) scoring to rank threats numerically.

3. **Threat model the LLM provider:** What threats exist on the external LLM API side? What assurances do you need from the provider?

4. **Red team plan:** Design a red team engagement plan to validate your threat model. What would you actually test?

5. **Compare to OWASP Top 10 for LLMs:** Map your threats to the OWASP Top 10 for Large Language Model Applications.

## Interview Relevance

Threat modeling is a key skill for senior engineers in banking:

| Skill | Why It Matters |
|-------|---------------|
| Systematic threat identification | Security must be proactive, not reactive |
| STRIDE framework | Industry-standard methodology |
| Risk assessment | Prioritizing mitigations by risk |
| GenAI-specific threats | New attack surfaces with AI systems |
| Mitigation design | Not just finding problems, solving them |

**Follow-up questions:**
- "What's the difference between STRIDE and DREAD?"
- "How do you threat model a system that uses an external AI API?"
- "What's the hardest threat to mitigate in a RAG system?"
- "How often should you revisit a threat model?"
