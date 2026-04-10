# Understanding Regulated Environments

> **Audience:** All engineers on the GenAI Platform Team
> **Purpose:** How compliance works in a global bank, what engineers must do, audit reality
> **Prerequisites:** None — read this during Week 1

---

## Why This Document Exists

If you have never worked in a regulated industry, this section is critical. If you have, read it anyway — every bank is different, and the consequences of non-compliance here are real.

**The short version:** In a global bank, every line of code you write could be examined by a regulator, an auditor, or a court. This is not theoretical — it happens regularly. Your work must be defensible.

### Real Consequence

> In 2023, a UK bank was fined £68 million partly because its automated decision-making system (not GenAI, but the principle applies) made discriminatory lending decisions. During the investigation, regulators requested:
> - The source code of the decision engine
> - The training data used
> - The testing records
> - The change history
> - The approval records for each deployment
> - The incident reports related to the system
>
> The bank could produce most of this, but the training data records were incomplete. This gap contributed to the fine.
>
> **For our GenAI platform:** The same scrutiny applies. If our system produces harmful output, regulators will ask for the guardrails code, the testing records, the deployment approvals, and the incident history. Everything must be documented.

---

## Regulatory Frameworks That Apply to Us

| Regulation | Jurisdiction | What It Covers | How It Affects Us |
|-----------|-------------|----------------|-------------------|
| SOX (Sarbanes-Oxley) | US | Financial reporting controls | Change management for systems that affect financial data |
| GDPR | UK/EU | Personal data protection | PII handling, data subject rights, data residency |
| DORA (Digital Operational Resilience Act) | EU | IT risk management | Operational resilience, incident reporting, third-party risk |
| FCA Handbook | UK | Conduct of business | Fair treatment of customers, financial promotion rules |
| Basel III | Global | Risk management | Model risk management, operational risk |
| EU AI Act | EU | AI system regulation | Risk classification, transparency, human oversight |
| NIST AI RMF | US (voluntary) | AI risk management | Governance, transparency, accountability |
| Internal Audit | Global (bank policy) | Internal controls | All of the above, plus bank-specific policies |

### What This Means in Practice

**Every engineer must:**

1. **Follow change management** — Every change tracked, approved, and documented
2. **Maintain audit trails** — Every decision logged, every action traceable
3. **Protect personal data** — PII handling follows strict rules
4. **Report incidents** — All incidents reported within SLA, not hidden
5. **Complete mandatory training** — Annual compliance training is not optional
6. **Cooperate with audits** — When auditors ask for evidence, you provide it

---

## Change Management

### The Golden Rule

**No change reaches production without a documented, approved change request.**

This applies to:
- Code deployments
- Configuration changes
- Infrastructure changes
- Database migrations
- Firewall rule changes
- Certificate rotations
- Secret rotations (even though automated, the process must be documented)

### The Change Types

| Type | Description | Approval | Lead Time |
|------|-------------|----------|-----------|
| Standard | Pre-approved, low-risk, documented procedure | Auto-approved | None |
| Normal | Planned change with full assessment | CAB review | 3-5 business days |
| Emergency | Urgent fix for a production issue | Post-hoc review | Immediate (review within 24h) |

**Standard changes** (auto-approved):
- Deploying a change that passed the full CI/CD pipeline
- Restarting a failed pod (Kubernetes handles this)
- Scaling up within pre-approved limits
- Rotating secrets via automated process

**Normal changes** (require CAB):
- New feature deployment
- Infrastructure changes (new services, network changes)
- Database schema changes
- Guardrails model updates
- AI provider changes

**Emergency changes** (post-hoc review):
- Hotfixes for production incidents
- Emergency rollbacks
- Security patches for critical vulnerabilities

### The Change Record

Every change creates a record that must survive for the audit retention period (7 years):

```
CHANGE RECORD: CR-2024-5678

Created: 2024-12-01 10:00 UTC
Requester: jane.smith@bank.com (Employee ID: 458921)
Team: GenAI Platform
Service: genai-platform-core
Change Type: Normal

Description:
Deploy v2.14.0 — Add financial advice disclaimer, update reranker model

Risk Assessment: LOW
Reasoning: All changes behind feature flags, rollback tested

Testing:
- Unit tests: 347/347 passed
- Integration tests: 89/89 passed
- E2E tests: 23/23 passed
- Security scan: 0 vulnerabilities
- Performance: Within acceptable bounds

Approvals:
- Tech Lead: sarah.chen@bank.com (2024-12-02 14:30 UTC)
- Eng Manager: james.wright@bank.com (2024-12-02 15:00 UTC)
- CAB: Approved at CAB-2024-47 (2024-12-03)

Deployment:
- Scheduled: 2024-12-05 18:00 UTC
- Executed: 2024-12-05 18:00-18:40 UTC
- Result: Successful
- Rollback required: No
- Post-deployment issues: None

Audit Trail:
- Jenkins build: https://jenkins.bank.internal/job/genai-platform-core/1234
- ArgoCD deployment: https://argocd.bank.internal/applications/genai-platform-core
- PR: https://github.com/bank-org/genai-platform-core/pull/8845
- CAB minutes: https://confluence.bank.internal/CAB-2024-47
```

**During an audit, this record will be examined to verify:**
1. The change was properly requested and documented
2. The change was properly tested
3. The change was properly approved
4. The change was executed as planned
5. The result was recorded

---

## Data Protection and PII

### What Counts as PII in Our Context

```
DEFINITE PII (must never appear in logs or prompts):
- Customer names
- Account numbers
- Sort codes
- National Insurance numbers / SSNs
- Credit card numbers
- Dates of birth
- Addresses
- Phone numbers
- Email addresses
- Biometric data

LIKELY PII (treat as PII unless confirmed otherwise):
- Transaction descriptions (may contain merchant names + locations)
- Portfolio holdings (financial position is personal data under GDPR)
- Salary information
- Employment details

NOT PII:
- Product names (ISA, Mortgage, Credit Card)
- Interest rates
- Public market data
- Anonymized, aggregated statistics
```

### How We Handle PII

**1. PII Detection in Prompts**

Every prompt is scanned for PII before being sent to the LLM:

```python
# genai_platform/guardrails/checks/pii_detection.py
"""
PII Detection Check

Scans prompts and responses for personally identifiable information.
If PII is detected, the content is redacted before being sent to the LLM.

PII patterns detected:
- UK National Insurance number: AB 12 34 56 C
- UK account number: 8 digits
- UK sort code: XX-XX-XX
- Credit card numbers: 16 digits (with Luhn check)
- UK phone numbers
- Email addresses
- UK postcodes (when adjacent to a name)

Regulatory basis: GDPR Article 5(1)(c) — Data minimization
Approved by: Data Protection Officer (ref: DPO-2024-0234)
"""

import re
from typing import List, Tuple
from ..pipeline import GuardrailCheck, GuardResult, GuardAction

class PIIDetectionCheck(GuardrailCheck):
    name = "pii_detection"
    
    PATTERNS = {
        "ni_number": r'[A-Z]{2}\s*\d{2}\s*\d{2}\s*\d{2}\s*[A-Z]',
        "account_number": r'\b\d{8}\b',
        "sort_code": r'\b\d{2}-\d{2}-\d{2}\b',
        "credit_card": r'\b(?:\d[ -]*?){13,19}\b',  # Then Luhn check
        "uk_phone": r'\b(?:\+44|0)\d{10,11}\b',
        "email": r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
    }
    
    async def evaluate(self, content: str, context: dict) -> GuardResult:
        detected: List[Tuple[str, str]] = []
        redacted = content
        
        for pii_type, pattern in self.PATTERNS.items():
            matches = re.findall(pattern, content)
            for match in matches:
                # Additional validation (e.g., Luhn check for credit cards)
                if pii_type == "credit_card" and not self._luhn_check(match):
                    continue
                
                detected.append((pii_type, match))
                redacted = redacted.replace(match, f"[REDACTED:{pii_type}]")
        
        if not detected:
            return GuardResult(
                action=GuardAction.ALLOW,
                confidence=0.0,
                reason="No PII detected",
                details={},
            )
        
        # PII detected — redact and continue
        return GuardResult(
            action=GuardAction.REDACT,
            confidence=1.0,
            reason=f"PII detected: {len(detected)} instances",
            details={
                "pii_types": [t for t, _ in detected],
                "pii_count": len(detected),
            },
            redacted_content=redacted,
        )
```

**2. PII Redaction in Logs**

All logging goes through a PII-redaction filter:

```python
# genai_platform/audit/redaction.py
"""
Log PII Redaction

All log entries pass through this filter before being written.
This ensures PII never appears in Splunk, Grafana, or any log store.

IMPORTANT: This is a defense-in-depth measure.
PII should already be removed by the guardrails PII check.
This filter catches anything that slips through.
"""

import re
import structlog

class PIIRefactingProcessor:
    """
    structlog processor that redacts PII from log entries.
    
    Runs on every log entry, every field, every time.
    Performance impact: < 0.1ms per log entry.
    """
    
    PATTERNS = [
        (re.compile(r'\b\d{8}\b'), '[REDACTED:ACCOUNT_NUMBER]'),
        (re.compile(r'[A-Z]{2}\s*\d{2}\s*\d{2}\s*\d{2}\s*[A-Z]'), '[REDACTED:NI_NUMBER]'),
        (re.compile(r'\b\d{2}-\d{2}-\d{2}\b'), '[REDACTED:SORT_CODE]'),
        # ... more patterns
    ]
    
    def __call__(self, logger, method, event_dict):
        for key, value in event_dict.items():
            if isinstance(value, str):
                for pattern, replacement in self.PATTERNS:
                    value = pattern.sub(replacement, value)
                event_dict[key] = value
        return event_dict
```

**3. Data Subject Rights (GDPR)**

If a customer exercises their right to access or delete their data, we must be able to:

- [ ] Identify all data we hold about them
- [ ] Provide it in a machine-readable format (within 30 days)
- [ ] Delete it if requested (within 30 days, subject to retention requirements)

**For the GenAI platform:** We do not store customer prompts or responses long-term. The audit log contains:
- Request ID
- Consumer ID
- Guardrails decisions (allow/deny/redact)
- Token usage
- Timestamp

**No prompt content or response content is stored in the audit log.** This is by design — it protects both the customer (PII) and the bank (liability).

---

## AI Governance

### The EU AI Act and Us

The EU AI Act classifies AI systems by risk level. Our platform supports systems that could fall into multiple categories:

| Use Case | Risk Level (Our Assessment) | Requirements |
|----------|---------------------------|--------------|
| Wealth Management Chat | High (financial advice) | Human oversight, transparency, accuracy |
| Investment Research Bot | Limited (informational) | Transparency (AI-generated content label) |
| Compliance Assistant | High (regulatory decisions) | Human oversight, accuracy, documentation |
| Code Review Assistant | Minimal (developer tool) | None specific |
| HR Policy Chatbot | Limited (employment decisions) | Transparency |
| Fraud Detection | High (financial decisions) | Human oversight, accuracy, appeal rights |
| KYC Document Processing | High (identity verification) | Human oversight, accuracy, documentation |

**What this means for engineers:**

1. **High-risk use cases require human oversight** — The AI never makes final decisions
2. **Transparency** — Users must know they are interacting with AI
3. **Accuracy** — We must measure and report accuracy for high-risk use cases
4. **Documentation** — Technical documentation must be maintained and available
5. **Incident reporting** — AI-related incidents must be reported to the regulator within 15 days (for high-risk systems in the EU)

### AI Model Risk Management

Every AI model used on the platform goes through a model risk assessment:

```
MODEL RISK ASSESSMENT: injection-scanner-v2.4

Model: Bank-internal prompt injection detector v2.4
Type: Binary classifier (malicious / not malicious)
Training data: 500K labeled prompts (curated by Security team)
Use case: Pre-flight guardrails check

Risk Assessment:
- Impact of wrong decision: HIGH (security breach or false blocking)
- Model complexity: Medium (fine-tuned transformer)
- Data sensitivity: Low (no PII in training data)
- Overall risk: MEDIUM-HIGH

Validation Results:
- Accuracy: 94.2% (threshold: 95%) — FAILED
- False positive rate: 2.1% (threshold: 3%) — PASSED
- False negative rate: 3.7% (threshold: 5%) — PASSED
- Bias assessment: No significant bias across languages — PASSED
- Robustness: Resists adversarial attacks (tested) — PASSED

Decision: NOT APPROVED for production deployment
Reason: Accuracy below threshold
Remediation: Retrain with additional adversarial examples
Re-test date: Sprint 25
```

**Model governance board:** All AI models are reviewed by the AI Governance Board before production use. This board includes:
- Chief Risk Officer (or delegate)
- Chief Information Security Officer (or delegate)
- Head of Data Science
- Head of Engineering
- Data Protection Officer
- Compliance representative

---

## Audit Reality

### What Auditors Actually Do

Auditors do not read your code. They read your **records about your code**.

**What auditors request (and what you need to produce):**

| Auditor Request | Where to Find It | Who Provides It |
|----------------|-----------------|-----------------|
| Show me the change record for deployment X | Change management system (ServiceNow) | Engineering Manager |
| Show me the test results for change Y | Jenkins build artifacts | DevOps Engineer |
| Show me the approvals for PR Z | GitHub PR review records | Tech Lead |
| Show me the access controls for service A | Kubernetes RBAC + Vault policies | SRE Engineer |
| Show me the incident history for service B | Incident management system (ServiceNow) | On-call Lead |
| Show me the PII handling process | Guardrails code + test records | Security Engineer |
| Show me the model validation records | AI Governance Board records | ML Platform Lead |
| Show me the training records for the team | LMS (Learning Management System) | Engineering Manager |

### Audit Survival Tips

**Do:**

1. **Answer only the question asked.** If the auditor asks "Show me the change record for deployment X," show them the change record. Do not volunteer "but we had some issues with the testing."
2. **Be honest.** If you do not know, say "I do not know, but I can find out." Never guess. Never make something up.
3. **Be timely.** Respond to audit requests within the stated deadline. Delaying looks suspicious.
4. **Keep good records.** If it is not documented, it did not happen. This is the auditor's mantra, and it is true.

**Do Not:**

1. **Delete anything.** Ever. Even if it looks bad. Even if it was a mistake. Deletion is itself a compliance violation.
2. **Change records retroactively.** If you need to correct something, add an amendment with a timestamp and explanation. Do not overwrite.
3. **Be defensive.** Auditors are not your enemies. They are doing their job. Cooperate professionally.
4. **Speculate.** "I think," "probably," and "maybe" are audit red flags. Stick to facts.

### Story: The Audit That Found a Gap

> In 2024, the internal audit team reviewed the GenAI platform's change management process. They sampled 20 changes from the previous quarter and checked each one against the change management policy.
>
> 19 out of 20 were fully compliant. The 20th was a feature flag activation that had been approved by the Product Owner but not by the CAB. The feature flag changed guardrails behavior (added a new check).
>
> **The finding:** "Feature flag activations that change guardrails behavior should require CAB approval, but the current process does not enforce this."
>
> **The remediation:** A new rule was added: any feature flag that affects guardrails, security, or compliance behavior requires CAB approval before activation. This rule was added to the feature flag policy and the CAB checklist.
>
> **The lesson:** Auditors think laterally. They do not just check the obvious paths — they look for edge cases and gaps. Assume they will find the one thing you forgot.

---

## Mandatory Training

### Annual Requirements

Every engineer must complete these annually (tracked by HR):

| Training | Duration | Deadline | Consequence of Non-Completion |
|----------|----------|----------|------------------------------|
| Information Security Awareness | 2 hours | March 31 | System access revoked |
| Data Protection and GDPR | 1.5 hours | March 31 | System access revoked |
| Code of Conduct | 1 hour | March 31 | Disciplinary action |
| Anti-Money Laundering | 2 hours | June 30 | Cannot work on relevant systems |
| AI Ethics and Governance | 2 hours | June 30 | Cannot work on AI systems |
| Operational Resilience | 1 hour | September 30 | Cannot be on-call |

**These are not optional.** If you do not complete them on time, your system access will be revoked automatically. Your manager will be notified. It is embarrassing and entirely avoidable.

### Setting Reminders

```
Calendar reminders you should set:
- 60 days before deadline: "Complete annual training: [list]"
- 30 days before deadline: "Training deadline in 30 days — schedule time"
- 7 days before deadline: "Training deadline in 7 days — complete this week"
```

---

## Incident Reporting Requirements

### Regulatory Timelines

| Incident Type | Internal Reporting | External Reporting | Regulator Notified |
|--------------|-------------------|-------------------|-------------------|
| Data breach (PII) | 1 hour | 72 hours (GDPR) | ICO within 72 hours |
| AI system producing harmful output | 4 hours | 15 days (EU AI Act) | Relevant authority |
| Service outage affecting consumers | 1 hour | Per SLA | Depends on impact |
| Security vulnerability exploited | 1 hour | Per DORA requirements | Depends on impact |
| Model producing biased decisions | 4 hours | Per AI Act | Depends on impact |

### What Engineers Must Do

When an incident occurs:

1. **Follow the runbook** — Restore service first
2. **Declare the incident** — Create the incident ticket within 15 minutes
3. **Classify the severity** — Use the severity matrix (see `incident-management/severity-matrix.md`)
4. **Report within SLA** — The incident management system auto-notifies the right people
5. **Complete the post-mortem** — Within the deadline (48 hours for P1/P2, 1 week for P3/P4)
6. **Track remediation actions** — Until all actions are closed

**Never:**
- Handle an incident "informally" without a ticket
- Delay reporting because you are still investigating
- Assume someone else has reported it
- Minimize the severity to avoid scrutiny

---

## Vendor Risk Management

### Azure OpenAI as a Third-Party Vendor

Microsoft Azure OpenAI is a critical third-party vendor for our platform. This creates specific compliance obligations:

```
VENDOR RISK ASSESSMENT: Microsoft Azure OpenAI

Vendor: Microsoft Corporation
Service: Azure OpenAI (GPT-4o, GPT-3.5-turbo, Embeddings)
Criticality: CRITICAL (platform depends on this service)
Data shared: Prompts (PII-redacted), consumer metadata

Due Diligence Completed:
- Security assessment: Microsoft SOC 2 Type II reviewed — PASSED
- Data protection: Microsoft DPA reviewed and bank addendum signed — PASSED
- Data residency: UK South region confirmed — PASSED
- Business continuity: Microsoft DR plan reviewed — PASSED
- Regulatory compliance: Microsoft compliance certifications reviewed — PASSED

Ongoing Monitoring:
- Monthly: Review Microsoft security advisories
- Quarterly: Review service level adherence
- Annually: Re-assess vendor risk

Contingency Plan:
- If Azure OpenAI is unavailable for > 1 hour:
  → Traffic shifts to internal LLM (Llama 3 70B)
  → Degraded functionality (some capabilities unavailable)
  → Communications sent to affected consumers
- If Azure OpenAI is unavailable for > 24 hours:
  → Full traffic shift to internal LLM
  → Consumer notification of reduced capabilities
  → Executive escalation with Microsoft
- If Azure OpenAI relationship is terminated:
  → Full migration to internal LLM + alternative providers
  → Estimated timeline: 4-6 weeks
  → Business impact assessment: MEDIUM
```

### What Engineers Need to Know

1. **Never share production data with external AI providers** — Only use approved APIs with PII-redacted data
2. **Report vendor incidents immediately** — If Azure OpenAI has an outage, it is our incident too
3. **Do not evaluate new AI providers without vendor risk assessment** — The procurement team must assess any new vendor

---

## Compliance Checklist for Every Change

Before any change reaches production, verify:

```
COMPLIANCE PRE-FLIGHT CHECKLIST

For EVERY change:
[ ] JIRA ticket exists and is linked to the change record
[ ] Testing evidence is documented
[ ] Approval from required approvers is recorded
[ ] Rollback plan is documented
[ ] Feature flags are configured (if deploying new behavior)
[ ] Monitoring is configured (if new failure modes are possible)

For GUARDRAILS changes:
[ ] Security team has reviewed and approved
[ ] Model validation records are complete (if model change)
[ ] AI Governance Board has approved (if model change)
[ ] Impact on all consumers has been assessed
[ ] Rollback tested in staging

For DATA changes:
[ ] DBA has reviewed and approved
[ ] Migration is reversible (rollback script tested)
[ ] No PII is exposed during migration
[ ] Backup taken before migration

For INFRASTRUCTURE changes:
[ ] SRE team has reviewed and approved
[ ] Network security impact assessed
[ ] Capacity impact assessed
[ ] Disaster recovery impact assessed

For AI PROVIDER changes:
[ ] Vendor risk assessment is complete
[ ] Data protection impact assessment is complete
[ ] Legal/procurement has approved the contract
[ ] Contingency plan is updated
```

---

## Further Reading

- `incident-management/` — Incident response procedures and reporting
- `glossary/` — Compliance terminology
- `interview-prep/` — Compliance-related interview questions
- `understanding-the-deployment-pipeline.md` — How changes reach production
- `first-30-days.md` — Your first compliance experience

---

*Last updated: April 2025 | Document owner: Compliance Engineering Team | Review cycle: Quarterly or upon regulatory change*
