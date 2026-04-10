# Threat Model Template

## Instructions

Complete this threat model for any new service or significant feature change. Review and update the threat model quarterly or when the architecture changes significantly.

---

# Threat Model: [System/Service Name]

| Field | Value |
|-------|-------|
| **Author** | [Name] |
| **Reviewers** | [Security Engineer], [Service Owner] |
| **Date** | YYYY-MM-DD |
| **System Version** | [Version] |
| **Classification** | Internal / Confidential / Restricted |

## System Overview

**Description**: Brief description of the system and its purpose.

**Architecture Diagram**:

```
[Diagram showing trust boundaries, data flows, and components]
```

**Trust Boundaries**: Identify where trust changes between components.

```
Trust Boundary 1: Client <-> API Gateway
  - Untrusted network traffic enters our infrastructure
  - Authentication and rate limiting applied here

Trust Boundary 2: API Gateway <-> Internal Services
  - Authenticated requests only
  - Internal network (VPC)

Trust Boundary 3: LLM Gateway <-> External LLM Provider
  - Prompts leave our infrastructure
  - API keys used for authentication
  - PII must be redacted before crossing this boundary
```

## Assets to Protect

| Asset | Classification | Impact if Compromised |
|-------|---------------|----------------------|
| Customer PII | Confidential | Regulatory fines, reputational damage, customer harm |
| LLM API keys | Secret | Unauthorized usage, cost exposure |
| AI decisions | Confidential | Compliance violations, incorrect advice liability |
| Model prompts | Internal | Intellectual property leakage |
| Vector DB content | Internal | Exposure of proprietary banking policies |

## Threat Analysis (STRIDE)

### Spoofing

**Threat**: An attacker impersonates a legitimate user or service.

| # | Scenario | Likelihood | Impact | Mitigation |
|---|----------|-----------|--------|-----------|
| S1 | Attacker steals user credentials | Medium | High | MFA, session management, anomaly detection |
| S2 | Attacker spoofs internal service | Low | Critical | mTLS between services, service mesh |
| S3 | Attacker uses forged JWT | Medium | High | JWT signature validation, short expiry |

### Tampering

**Threat**: An attacker modifies data in transit or at rest.

| # | Scenario | Likelihood | Impact | Mitigation |
|---|----------|-----------|--------|-----------|
| T1 | Attacker modifies API request | Low | High | TLS everywhere, request signing |
| T2 | Attacker modifies stored data | Low | High | Database access controls, audit logging |
| T3 | Attacker modifies prompt before LLM | Low | Critical | Prompt integrity checks, checksums |
| T4 | Attacker modifies LLM response | Low | Critical | Response validation, schema checking |

### Repudiation

**Threat**: A user denies performing an action, and there is no evidence to the contrary.

| # | Scenario | Likelihood | Impact | Mitigation |
|---|----------|-----------|--------|-----------|
| R1 | User denies submitting a loan application | Medium | High | Immutable audit log with correlation IDs |
| R2 | Admin denies changing system config | Low | High | Admin action logging, tamper-evident logs |
| R3 | System cannot prove advice was given | Medium | High | Full interaction audit trail |

### Information Disclosure

**Threat**: Sensitive data is exposed to unauthorized parties.

| # | Scenario | Likelihood | Impact | Mitigation |
|---|----------|-----------|--------|-----------|
| I1 | PII logged in application logs | Medium | Critical | Automated PII scanning, redaction |
| I2 | Customer data sent to LLM provider | Medium | Critical | PII redaction before prompts, DLP |
| I3 | Database credentials exposed in repo | Low | Critical | Secret scanning, environment variables |
| I4 | Vector DB accessible from internet | Low | Critical | Network isolation, security groups |

### Denial of Service

**Threat**: The system becomes unavailable to legitimate users.

| # | Scenario | Likelihood | Impact | Mitigation |
|---|----------|-----------|--------|-----------|
| D1 | LLM provider rate limits us | Medium | High | Request queuing, caching, multi-provider |
| D2 | Volumetric attack on API | Low | High | WAF, rate limiting, CDN |
| D3 | Database connection exhaustion | Medium | High | Connection pooling, query timeouts |
| D4 | Expensive prompt causes resource drain | Medium | Medium | Input length limits, cost caps |

### Elevation of Privilege

**Threat**: An attacker gains higher permissions than intended.

| # | Scenario | Likelihood | Impact | Mitigation |
|---|----------|-----------|--------|-----------|
| E1 | Broken authorization allows access to other users' data | Medium | Critical | Authorization checks on every request, testing |
| E2 | IDOR vulnerability | Medium | High | Object-level authorization, UUIDs |
| E3 | Prompt injection grants unauthorized capabilities | Medium | Critical | Input sanitization, output validation, guardrails |

## Attack Tree

```
Goal: Obtain customer financial data
│
├─> Compromise user credentials
│   ├─> Phishing attack
│   ├─> Credential stuffing (reuse from other breaches)
│   └─> Brute force (mitigated by account lockout)
│
├─> Exploit application vulnerability
│   ├─> SQL injection (mitigated by parameterized queries)
│   ├─> XSS (mitigated by CSP, output encoding)
│   └─> IDOR (mitigated by object-level auth)
│
├─> Compromise infrastructure
│   ├─> Exploit unpatched dependency (mitigated by dependency scanning)
│   ├─> Stolen cloud credentials (mitigated by IAM policies, MFA)
│   └─> Insider threat (mitigated by access controls, audit logging)
│
└─> Intercept data in transit
    ├─> MITM attack (mitigated by TLS)
    └─> Compromised LLM provider (mitigated by PII redaction)
```

## Risk Assessment

| Risk | Likelihood | Impact | Risk Level | Owner | Status |
|------|-----------|--------|-----------|-------|--------|
| PII in LLM prompts | Medium | Critical | HIGH | [Name] | Mitigated (redaction) |
| Prompt injection | Medium | High | HIGH | [Name] | Mitigated (guardrails) |
| LLM provider DoS | Medium | High | MEDIUM | [Name] | Accepted (fallback plan) |
| Credential theft | Low | Critical | MEDIUM | [Name] | Mitigated (MFA) |

## Mitigations Summary

| Mitigation | Threats Addressed | Implementation Status |
|-----------|-------------------|----------------------|
| PII redaction in prompts | I1, I2 | Implemented |
| Prompt injection guardrails | E3, T3 | Implemented |
| LLM provider fallback | D1 | Implemented |
| Audit logging (immutable) | R1, R2, R3 | Implemented |
| mTLS between services | S2, T1 | Planned (Q2) |
| Automated PII scanning | I1 | Planned (Q2) |

## Compliance Mapping

| Regulation | Requirement | How This System Complies |
|-----------|-------------|-------------------------|
| GLBA | Protect customer financial data | PII redaction, encryption, access controls |
| SOX | Audit trails for financial decisions | Immutable audit log, correlation IDs |
| ECOA | Fair lending | Bias testing, demographic parity monitoring |

## Review and Sign-off

| Role | Name | Date | Status |
|------|------|------|--------|
| Security Engineer | | | Approved / Changes Required |
| Service Owner | | | Approved / Changes Required |
| Compliance Officer | | | Approved / Changes Required |

## Review Schedule

Next review date: YYYY-MM-DD (quarterly or after significant architecture change)
