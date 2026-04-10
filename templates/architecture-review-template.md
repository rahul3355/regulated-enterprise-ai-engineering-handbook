# Architecture Review Template

> Use this template to review architecture designs for completeness, security, reliability, and banking compliance.

## Template

```markdown
# Architecture Review: [System/Feature Name]

**Reviewer:** [Name, role]
**Date:** [YYYY-MM-DD]
**Design Doc:** [Link]
**Author:** [Name, team]

## Overall Assessment

🟢 Approved / 🟡 Approved with Conditions / 🔴 Needs Revision

[1-paragraph summary of your assessment and the most important findings.]

## Review by Category

### 1. Functional Requirements

| Question | Assessment | Notes |
|----------|-----------|-------|
| Does the design meet the stated requirements? | ✅/⚠️/❌ | [...] |
| Are non-goals clearly stated? | ✅/⚠️/❌ | [...] |
| Are edge cases considered? | ✅/⚠️/❌ | [...] |
| Is the user experience clear? | ✅/⚠️/❌ | [...] |

### 2. Security

| Question | Assessment | Notes |
|----------|-----------|-------|
| Authentication design is clear | ✅/⚠️/❌ | [...] |
| Authorization/access control is defined | ✅/⚠️/❌ | [...] |
| Data is protected at rest and in transit | ✅/⚠️/❌ | [...] |
| Input validation is specified | ✅/⚠️/❌ | [...] |
| GenAI-specific threats addressed (prompt injection, data exfil) | ✅/⚠️/❌ | [...] |
| Dependencies have been security-reviewed | ✅/⚠️/❌ | [...] |

### 3. Reliability

| Question | Assessment | Notes |
|----------|-----------|-------|
| Failure modes are identified | ✅/⚠️/❌ | [...] |
| Fallback/degradation behavior is defined | ✅/⚠️/❌ | [...] |
| Retry and idempotency are addressed | ✅/⚠️/❌ | [...] |
| Single points of failure identified | ✅/⚠️/❌ | [...] |
| Recovery procedures are documented | ✅/⚠️/❌ | [...] |

### 4. Scalability

| Question | Assessment | Notes |
|----------|-----------|-------|
| Expected load is estimated | ✅/⚠️/❌ | [...] |
| Bottlenecks are identified | ✅/⚠️/❌ | [...] |
| Scaling strategy is defined | ✅/⚠️/❌ | [...] |
| Capacity planning is addressed | ✅/⚠️/❌ | [...] |

### 5. Observability

| Question | Assessment | Notes |
|----------|-----------|-------|
| Key metrics are defined | ✅/⚠️/❌ | [...] |
| Alerting strategy is specified | ✅/⚠️/❌ | [...] |
| Logging captures sufficient context | ✅/⚠️/❌ | [...] |
| Distributed tracing is included | ✅/⚠️/❌ | [...] |
| Dashboards are planned | ✅/⚠️/❌ | [...] |

### 6. Operations

| Question | Assessment | Notes |
|----------|-----------|-------|
| Deployment strategy is defined | ✅/⚠️/❌ | [...] |
| Rollback plan exists | ✅/⚠️/❌ | [...] |
| Runbook is planned | ✅/⚠️/❌ | [...] |
| On-call implications are understood | ✅/⚠️/❌ | [...] |
| Feature flags are used (if applicable) | ✅/⚠️/❌ | [...] |

### 7. Compliance (Banking-Specific)

| Question | Assessment | Notes |
|----------|-----------|-------|
| Audit logging design is specified | ✅/⚠️/❌ | [...] |
| Data retention is addressed | ✅/⚠️/❌ | [...] |
| PII handling is defined | ✅/⚠️/❌ | [...] |
| Regulatory requirements are identified | ✅/⚠️/❌ | [...] |
| Model versioning is tracked (GenAI) | ✅/⚠️/❌ | [...] |
| Response citations are included (GenAI) | ✅/⚠️/❌ | [...] |

### 8. Alternatives & Trade-offs

| Question | Assessment | Notes |
|----------|-----------|-------|
| Alternatives were considered | ✅/⚠️/❌ | [...] |
| Trade-offs are honestly assessed | ✅/⚠️/❌ | [...] |
| Decision rationale is clear | ✅/⚠️/❌ | [...] |

## Findings

### Critical (Must Fix Before Implementation)

1. **[Finding title]**
   - **Category:** [Security/Reliability/Compliance/etc.]
   - **Description:** [What the issue is]
   - **Recommendation:** [How to fix it]

### Significant (Should Fix Before Implementation)

1. **[Finding title]**
   - [...]

### Minor (Can Fix During Implementation)

1. **[Finding title]**
   - [...]

## Open Questions

- [ ] [Question that needs resolution before implementation]
- [ ] [Question assigned to specific person/team]

## Recommendation

[Final recommendation: proceed, proceed with conditions, or revise and re-review.]

---
*This review is based on the design document as of [date]. Implementation
details may reveal additional considerations. This review does not replace
security or compliance sign-off.*
```

## Banking-Specific Review Checklist

For GenAI systems, ensure these are addressed:

### GenAI-Specific
- [ ] Prompt versioning and management
- [ ] Model version pinning (not "latest")
- [ ] Prompt injection defense
- [ ] Output content filtering
- [ ] Response hallucination mitigation
- [ ] PII detection in prompts and responses
- [ ] Document-level access control in RAG
- [ ] Audit logging of prompts, responses, model versions
- [ ] Response quality monitoring
- [ ] Fallback behavior for model failures

### Data Flow
- [ ] Complete data flow diagram
- [ ] Data classification at each stage
- [ ] Encryption at rest and in transit
- [ ] Data retention and deletion strategy
- [ ] Cross-border data transfer (if applicable)

### External Dependencies
- [ ] All external dependencies listed
- [ ] SLA/availability requirements for each
- [ ] Fallback if dependency is unavailable
- [ ] Security review of external services
- [ ] Cost model and budget
