# Compliance Exercise: RAG Pipeline Review

> Perform a compliance review of a GenAI RAG pipeline against banking regulations and internal policies.

## Problem Statement

You are a compliance engineer reviewing the GenAI RAG pipeline before its production launch. The system answers employee questions about internal policies using retrieval-augmented generation.

Perform a compliance review covering data protection, regulatory alignment, audit readiness, and AI governance.

## System Under Review

```
Components:
- Frontend: React web app with SSO authentication
- Backend: Python FastAPI service on Kubernetes
- RAG: pgvector-based retrieval with document-level access control
- LLM: External API (OpenAI GPT-4) with data processing agreement
- Audit: Async logging to PostgreSQL with 7-year retention
- Content Filter: Input/output screening for PII and harmful content

Data Flow:
1. Employee authenticates via SSO
2. Submits query through web app
3. Backend validates user's document access permissions
4. Retrieves relevant documents from pgvector (filtered by access)
5. Sends query + documents to OpenAI API (over TLS)
6. Filters response for PII and harmful content
7. Returns response with citations to employee
8. Logs interaction to audit database
```

## Compliance Framework

Review against these requirements:

| Framework | Relevance |
|-----------|-----------|
| **GDPR** | Employee data processing, right to access/erasure |
| **Banking Regulations** | Model risk management, audit requirements |
| **AI Governance Policy** | Internal AI usage policy |
| **Data Classification Policy** | Handling of confidential documents |
| **Vendor Risk Policy** | External LLM provider assessment |

## Review Checklist

### Data Protection

- [ ] **Data minimization:** Is only necessary data sent to the LLM API?
- [ ] **Purpose limitation:** Is employee data used only for the stated purpose?
- [ ] **Consent/notice:** Are employees informed about how their data is processed?
- [ ] **Right to access:** Can an employee request a copy of all their interaction data?
- [ ] **Right to erasure:** How is a deletion request handled given 7-year retention?
- [ ] **Data residency:** Does data stay within approved geographic regions?
- [ ] **PII handling:** Is PII detected and redacted before sending to the LLM?

### Access Control

- [ ] **Authentication:** Is authentication robust (MFA, token TTL)?
- [ ] **Authorization:** Is document-level access control enforced?
- [ ] **Segregation:** Can one user's data influence another user's responses?
- [ ] **Privileged access:** Who has admin access to the system? Is it logged?

### Audit & Traceability

- [ ] **Completeness:** Is every interaction logged?
- [ ] **Integrity:** Can audit logs be tampered with?
- [ ] **Retention:** Are logs retained for the required period?
- [ ] **Searchability:** Can investigators find specific interactions?
- [ ] **Model versioning:** Is the model version logged for each response?

### AI-Specific

- [ ] **Hallucination risk:** What happens when the AI provides incorrect information?
- [ ] **Transparency:** Do users know they're interacting with an AI?
- [ ] **Human oversight:** Is there a process for reviewing AI-generated content?
- [ ] **Bias assessment:** Has the system been tested for biased responses?
- [ ] **Prompt security:** Are prompts protected from injection attacks?

### Vendor Risk

- [ ] **Data processing agreement:** Does the LLM provider have a signed DPA?
- [ ] **Data usage:** Does the provider use our data for training?
- [ ] **Data deletion:** Can we request deletion of our data from the provider?
- [ ] **Security certifications:** Does the provider have SOC 2, ISO 27001?
- [ ] **Incident notification:** How quickly does the provider notify us of breaches?

## Expected Deliverables

```markdown
# Compliance Review: GenAI RAG Pipeline
# Reviewer: [Name]
# Date: [Date]
# Status: Pass / Conditional Pass / Fail

## Executive Summary
[1-paragraph summary of overall compliance posture]

## Findings

### Finding 1: [Title]
- **Category:** [Data Protection / Access Control / Audit / AI-Specific / Vendor]
- **Severity:** [Critical / High / Medium / Low]
- **Description:** [What the issue is]
- **Requirement:** [What regulation/policy requires]
- **Evidence:** [What you found that indicates the issue]
- **Recommendation:** [How to fix it]
- **Remediation Timeline:** [When it needs to be fixed]

## Summary

| Severity | Count | Must Fix Before Launch? |
|----------|-------|------------------------|
| Critical | X | Yes |
| High | X | Yes |
| Medium | X | Within 30 days |
| Low | X | Within 90 days |

## Overall Recommendation
[Launch / Launch with conditions / Do not launch]

## Sign-off
- Compliance Reviewer: [Name, Date]
- Engineering Owner: [Name, Date] (acknowledges findings and timeline)
```

## Extensions

1. **Write the DPA requirements:** Draft the data processing agreement requirements for the LLM provider.

2. **Create a monitoring plan:** Design ongoing compliance monitoring (not just point-in-time review).

3. **Employee privacy notice:** Write the employee-facing privacy notice for the GenAI assistant.

4. **Model risk assessment:** Complete a model risk management assessment per SR 11-7 guidance.

5. **Cross-border data transfer:** The bank operates in 40 countries. Design the data transfer compliance strategy.

## Interview Relevance

Compliance review skills are tested for banking-specific engineering roles:

| Skill | Why It Matters |
|-------|---------------|
| Regulatory knowledge | Banking is heavily regulated |
| Systematic review | Structured approach to compliance |
| Risk assessment | Prioritizing findings by severity |
| AI governance | New and evolving requirement area |
| Vendor assessment | External dependencies in compliance scope |

**Follow-up questions:**
- "What's the biggest compliance risk in using an external LLM API?"
- "How do you balance GDPR right to erasure with banking retention requirements?"
- "What evidence would an auditor need to verify AI compliance?"
- "How often should compliance reviews be repeated?"
