# Compliance Review Template

> Use this template to review GenAI systems against banking regulations and internal policies.

## Template

```markdown
# Compliance Review: [System Name]

| Field | Value |
|-------|-------|
| **System** | [Name, version] |
| **Team** | [Team name] |
| **Reviewer** | [Name, compliance team] |
| **Engineering Contact** | [Name, role] |
| **Review Date** | [YYYY-MM-DD] |
| **Review Type** | Initial / Periodic / Change-driven |
| **Status** | Pass / Conditional Pass / Fail |

## Executive Summary

[1-paragraph summary of compliance posture, key findings, and recommendation.]

## Scope

[What is being reviewed? Which components, data flows, and use cases?]

## Regulatory Framework

[Which regulations and policies apply?]

| Regulation/Policy | Applicability | Evidence Reviewed |
|-------------------|--------------|-------------------|
| GDPR | [Yes/Partial/N/A] | [DPA, data map, privacy notice] |
| Banking Regulations | [Yes/Partial/N/A] | [...] |
| AI Governance Policy | [Yes/Partial/N/A] | [...] |
| Data Classification Policy | [Yes/Partial/N/A] | [...] |
| Vendor Risk Policy | [Yes/Partial/N/A] | [...] |

## Findings

### Finding 1: [Title]

| Field | Value |
|-------|-------|
| **ID** | FIND-[Number] |
| **Category** | Data Protection / Access Control / Audit / AI Governance / Vendor Risk |
| **Severity** | Critical / High / Medium / Low |
| **Status** | Open / In Progress / Resolved / Accepted Risk |
| **Requirement** | [What regulation/policy requires this] |
| **Finding** | [What was observed that doesn't comply] |
| **Evidence** | [Screenshots, configurations, code, interview notes] |
| **Recommendation** | [How to achieve compliance] |
| **Remediation Owner** | [Name] |
| **Remediation Due Date** | [YYYY-MM-DD] |

## Summary

| Severity | Count | Block Launch? |
|----------|-------|--------------|
| Critical | X | Yes |
| High | X | Yes |
| Medium | X | No (fix within 30 days) |
| Low | X | No (fix within 90 days) |

## Detailed Review

### 1. Data Protection

| Control | Status | Notes |
|---------|--------|-------|
| Data minimization | ✅/⚠️/❌ | [...] |
| Purpose limitation | ✅/⚠️/❌ | [...] |
| Consent/notice | ✅/⚠️/❌ | [...] |
| Right to access | ✅/⚠️/❌ | [...] |
| Right to erasure | ✅/⚠️/❌ | [...] |
| Data residency | ✅/⚠️/❌ | [...] |
| PII detection/redaction | ✅/⚠️/❌ | [...] |

### 2. Access Control

| Control | Status | Notes |
|---------|--------|-------|
| Authentication | ✅/⚠️/❌ | [...] |
| Authorization (document-level) | ✅/⚠️/❌ | [...] |
| Data isolation | ✅/⚠️/❌ | [...] |
| Privileged access management | ✅/⚠️/❌ | [...] |

### 3. Audit & Traceability

| Control | Status | Notes |
|---------|--------|-------|
| Complete audit logging | ✅/⚠️/❌ | [...] |
| Log integrity | ✅/⚠️/❌ | [...] |
| Retention (7+ years) | ✅/⚠️/❌ | [...] |
| Model version tracking | ✅/⚠️/❌ | [...] |
| E-discovery capability | ✅/⚠️/❌ | [...] |

### 4. AI Governance

| Control | Status | Notes |
|---------|--------|-------|
| User transparency (AI disclosure) | ✅/⚠️/❌ | [...] |
| Hallucination mitigation | ✅/⚠️/❌ | [...] |
| Human oversight process | ✅/⚠️/❌ | [...] |
| Bias assessment | ✅/⚠️/❌ | [...] |
| Prompt security | ✅/⚠️/❌ | [...] |
| Response quality monitoring | ✅/⚠️/❌ | [...] |

### 5. Vendor Risk (External LLM Provider)

| Control | Status | Notes |
|---------|--------|-------|
| Data processing agreement | ✅/⚠️/❌ | [...] |
| Data usage restrictions | ✅/⚠️/❌ | [...] |
| Security certifications | ✅/⚠️/❌ | [...] |
| Incident notification SLA | ✅/⚠️/❌ | [...] |
| Right to audit | ✅/⚠️/❌ | [...] |
| Data deletion capability | ✅/⚠️/❌ | [...] |

## Recommendation

- [ ] **Pass** — No Critical or High findings. System may proceed to production.
- [ ] **Conditional Pass** — No Critical findings. High findings must be resolved within 30 days. System may proceed with documented acceptance of residual risk.
- [ ] **Fail** — Critical findings exist. System may NOT proceed to production until resolved.

## Sign-off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Compliance Reviewer | | | |
| Engineering Owner | | | |
| Security Reviewer (if applicable) | | | |
| Data Protection Officer (if applicable) | | | |
```

## Common Compliance Findings for GenAI Systems

| Finding | Typical Severity | Typical Fix |
|---------|-----------------|-------------|
| No data processing agreement with LLM provider | Critical | Execute DPA before launch |
| PII sent to external LLM API without redaction | Critical | Implement PII detection and redaction |
| No audit logging of prompts and responses | High | Implement async audit logging |
| Document-level access control not enforced in RAG | Critical | Implement pre-retrieval access filtering |
| No model version tracking | High | Log model name and version with each response |
| No user disclosure that they're interacting with AI | Medium | Add AI disclosure in the UI |
| No process for correcting AI errors | Medium | Implement feedback mechanism + correction workflow |
| Training data includes confidential documents | High | Document data sources; exclude confidential data |
