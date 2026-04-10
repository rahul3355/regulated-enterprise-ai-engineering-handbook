# Banking Incident Requirements

## Overview

As a banking GenAI platform, incidents carry additional regulatory, audit, legal, and compliance obligations beyond standard software incident management. This document details the specific requirements for incident management in a banking context, including regulatory frameworks, reporting obligations, and audit expectations.

---

## Regulatory Frameworks

### FCA (Financial Conduct Authority)

#### Relevant Regulations

| Regulation | Topic | Incident Relevance |
|-----------|-------|-------------------|
| **PRIN 2.1.1** | Adequate resources and systems | System outages, operational failures |
| **PRIN 2.1.2** | Conduct with due skill, care, and diligence | AI giving incorrect financial advice |
| **SYSC 4.1.1** | Systems and controls for compliance | Missing audit logs, process gaps |
| **SYSC 8.1** | Outsourcing (including cloud and AI providers) | LLM provider outage, vendor incidents |
| **SYSC 13** | Risk control | Model risk, AI risk incidents |
| **COBS 2.1.1** | Clear, fair, not misleading communications | AI giving misleading financial advice |
| **SS1/21** | Operational Resilience | IBS disruption, impact tolerance breach |
| **FG 22/1** | Guidance on operational resilience | Testing, impact tolerance setting |

#### FCA Notification Obligations

| Trigger | Timeline | Method |
|---------|----------|--------|
| IBS breach of impact tolerance | 24 hours | RegData portal |
| Significant operational incident | As soon as practicable | RegData portal |
| Outsourcing failure (vendor incident) | As soon as practicable | RegData portal |
| Breach of threshold indicator | Immediately | Phone + written follow-up |

#### FCA Expectations for GenAI Incidents

The FCA expects firms using AI/ML to:

1. **Understand Model Risk**: Know when the model may produce incorrect outputs
2. **Maintain Human Oversight**: Ensure humans can review and override AI decisions
3. **Ensure Fairness**: Monitor for discriminatory or biased AI outputs
4. **Maintain Records**: Keep records of AI decisions and recommendations
5. **Test Resilience**: Regularly test AI system resilience under stress conditions

### PRA (Prudential Regulation Authority)

#### Relevant Requirements

| Requirement | Description |
|-------------|-------------|
| **SS2/21** | Climate risk (if AI affects climate-related financial decisions) |
| **SS1/20** | Third-party risk (AI vendor dependencies) |
| **Operational Resilience** | Consistent with FCA policy |

#### PRA Notification

Notify PRA for incidents affecting:
- Capital adequacy decisions made by AI
- Credit risk model failures
- Liquidity management system failures

### ICO (Information Commissioner's Office) -- GDPR

#### Relevant Articles

| Article | Topic | Incident Relevance |
|---------|-------|-------------------|
| **Article 33** | Breach notification | Notify within 72 hours of personal data breach |
| **Article 34** | Communicating breach to data subjects | Notify individuals if high risk |
| **Article 22** | Automated decision-making | AI making decisions without human review |
| **Article 5** | Data processing principles | AI processing data unfairly or inaccurately |
| **Article 25** | Data protection by design | AI systems not designed with privacy |
| **Article 35** | Data protection impact assessment | DPIA not conducted or inadequate |

#### ICO Notification Thresholds

| Severity | Criteria | Action |
|----------|----------|--------|
| High Risk | Likely to adversely affect individuals' rights | Notify ICO within 72 hours + notify individuals |
| Medium Risk | Possible risk, assessment needed | Notify ICO within 72 hours if confirmed |
| Low Risk | Unlikely to result in risk | Document internally, no notification required |

### NCSC (National Cyber Security Centre)

#### Reportable Incidents

| Incident Type | Report? | Timeline |
|--------------|---------|----------|
| State-sponsored cyber attack | Yes | Immediately |
| Ransomware attack | Yes | Immediately |
| Significant data breach | Yes | Within 72 hours |
| Supply chain compromise | Yes | Within 72 hours |
| Critical infrastructure disruption | Yes | Immediately |

### MiFID II (Markets in Financial Instruments Directive)

#### Relevant Requirements

| Article | Topic | Incident Relevance |
|---------|-------|-------------------|
| **Article 16** | Recording of communications | AI advice recording obligations |
| **Article 17** | Algorithmic trading | AI-driven trading decisions |
| **Article 25** | Suitibility assessment | AI providing investment advice |

---

## Incident Documentation Requirements

### Mandatory Documentation

For every incident, the following must be documented and retained:

| Document | Retention | Purpose |
|----------|-----------|---------|
| Incident timeline | 5 years | Regulatory evidence |
| Root cause analysis | 5 years | Audit trail |
| Impact assessment | 5 years | Customer impact records |
| Action items and completion | 5 years | Demonstrate remediation |
| Customer notifications | 5 years | Proof of notification |
| Regulatory notifications | 5 years | Proof of compliance |
| Postmortem document | 5 years | Organizational learning |
| War room recording | 1 year | Investigation reference |
| Log archives | 5 years | Evidence preservation |

### Audit Trail Requirements

The following must be recorded for every GenAI interaction:

| Data Point | Why | Retention |
|-----------|-----|-----------|
| Customer ID | Identify affected parties | 5 years |
| Timestamp | Chronological audit trail | 5 years |
| Query/Prompt | What was asked | 5 years |
| Retrieved context | What data was used | 5 years |
| LLM response | What was answered | 5 years |
| Model version | Reproducibility | 5 years |
| System prompt version | Behavioral context | 5 years |
| Confidence score | Quality assessment | 5 years |
| Human review outcome (if applicable) | Human oversight record | 5 years |
| Customer consent ID | Consent verification | 5 years |

---

## Banking-Specific Incident Classification

### Additional Classification Dimensions

Beyond standard SEV levels, banking incidents are classified by:

#### Regulatory Impact

| Level | Description |
|-------|-------------|
| **R-1** | Active regulatory breach requiring mandatory notification |
| **R-2** | Potential regulatory breach requiring investigation |
| **R-3** | Compliance gap identified, no active breach |
| **R-4** | No regulatory impact |

#### Customer Financial Impact

| Level | Description |
|-------|-------------|
| **F-1** | Direct financial loss to customers > £100,000 |
| **F-2** | Direct financial loss to customers £10,000 - £100,000 |
| **F-3** | Direct financial loss to customers £1,000 - £10,000 |
| **F-4** | Direct financial loss to customers < £1,000 |
| **F-5** | No direct financial loss |

#### Data Sensitivity

| Level | Description |
|-------|-------------|
| **D-1** | Special category data exposed (biometric, health, criminal) |
| **D-2** | Financial PII exposed (SSN, account numbers, credit data) |
| **D-3** | Standard PII exposed (name, address, email) |
| **D-4** | Non-personal sensitive data exposed (business data) |
| **D-5** | No data exposure |

### Combined Classification

A complete incident classification includes:

```
SEV-1 / R-1 / F-2 / D-2
Meaning: Critical incident, regulatory breach, £10K-100K customer loss, financial PII exposed
```

---

## Incident Response Requirements

### Mandatory Participants

For banking incidents, the following must be involved:

| Severity | Required Participants |
|----------|----------------------|
| SEV-1 | IC, SMEs, Compliance Officer, Legal Counsel, DPO (if data involved), Engineering Lead, VP Engineering |
| SEV-2 | IC, SMEs, Compliance Officer (if regulatory impact), DPO (if data involved) |
| SEV-3 | On-call engineer, SME, Compliance Officer (if regulatory impact) |

### Mandatory Actions

| Action | SEV-1 | SEV-2 | SEV-3 |
|--------|-------|-------|-------|
| War room opened | Required | Required | Optional |
| Compliance notified | Within 1 hour | Within 2 hours | If regulatory impact |
| Legal notified | Within 1 hour | Within 4 hours | If legal risk |
| DPO notified | If data involved | If data involved | If data involved |
| Regulatory notification assessed | Immediately | Within 4 hours | Within 24 hours |
| Evidence preserved | Mandatory | Mandatory | Recommended |
| Executive briefing | Within 1 hour | Within 4 hours | Daily summary |

---

## Post-Incident Requirements

### Regulatory Follow-Up

After incidents with regulatory notification:

1. **Full Investigation Report**: Complete root cause analysis with evidence
2. **Remediation Plan**: Detailed plan to prevent recurrence
3. **Progress Updates**: Regular updates to the regulator on remediation progress
4. **Lessons Learned Report**: What was learned and how it is being applied
5. **Evidence of Completion**: Proof that remediation actions were completed

### Internal Audit

Internal audit will review:
- Whether the incident was properly classified
- Whether notification obligations were met
- Whether evidence was properly preserved
- Whether action items were completed on time
- Whether similar incidents have recurred

### External Audit

External auditors may request:
- Incident logs and timelines
- Evidence of regulatory compliance
- Action item completion records
- Risk assessment documentation
- System change records

---

## Senior Managers and Certification Regime (SMCR)

### Accountability Mapping

Under SMCR, specific senior managers are accountable for GenAI incidents:

| SMF Role | Responsibility | Incident Accountability |
|----------|---------------|------------------------|
| **SMF12** | Chief Operations | Operational resilience, service availability |
| **SMF16** | Chief Risk | Risk management, model risk |
| **SMF17** | Chief Technology | Technology systems, security |
| **SMF18** | Chief Data | Data management, data quality |
| **SMF4** | Chief Risk (PRA) | Prudential risk, capital adequacy |
| **SMF1** | CEO | Overall accountability |

### Statements of Responsibility

Each SMF holder must have a clear statement of responsibility that covers:
- GenAI system oversight
- Incident management accountability
- Regulatory notification obligations
- Remediation approval authority

---

## Third-Party / Vendor Incidents

### FCA Outsourcing Requirements

When an incident involves a third-party vendor (LLM provider, cloud provider, vector DB provider):

1. **Vendor Notification**: Vendor must notify the bank within the SLA defined in the contract
2. **Bank Assessment**: Bank must assess the impact on regulated activities
3. **FCA Notification**: If the incident affects IBS, the bank must notify the FCA (the bank remains responsible, not the vendor)
4. **Remediation Oversight**: Bank must oversee and verify vendor remediation
5. **Contract Review**: Review contractual obligations and penalties

### Vendor Incident Documentation

The bank must maintain:
- Vendor incident reports (from the vendor)
- Independent impact assessment (by the bank)
- Evidence of vendor remediation
- Updated risk assessment for the vendor relationship

---

## Incident Record-Keeping Checklist

For every banking incident:

### During the Incident
- [ ] Incident classified with SEV level and regulatory impact
- [ ] Compliance team notified (if regulatory impact)
- [ ] Legal team notified (if legal risk)
- [ ] DPO notified (if personal data involved)
- [ ] Evidence preservation initiated
- [ ] War room opened (SEV-1/SEV-2)
- [ ] Timeline documented in real-time
- [ ] Regulatory notification obligation assessed

### After the Incident
- [ ] Root cause analysis completed
- [ ] Impact assessment documented (customer, financial, regulatory, data)
- [ ] Regulatory notifications submitted (if required)
- [ ] Customer notifications sent (if required)
- [ ] Postmortem conducted (blameless)
- [ ] Action items captured with owners and deadlines
- [ ] Evidence archived per retention policy
- [ ] Internal audit notified (for significant incidents)
- [ ] SMF holder briefed (for incidents in their area of responsibility)

### Long-Term
- [ ] Action items tracked to completion
- [ ] Regulatory follow-up completed (if notification was made)
- [ ] Remediation verified and documented
- [ ] Lessons learned shared with organization
- [ ] Runbooks and playbooks updated
- [ ] Risk assessment updated
- [ ] Vendor risk assessment updated (if vendor incident)

---

## Key Contacts

| Role | Contact | When to Engage |
|------|---------|---------------|
| **Incident Commander** | PagerDuty rotation | All SEV-1/SEV-2 incidents |
| **Compliance Officer** | compliance-oncall@bank.com | Any regulatory impact |
| **Legal Counsel** | legal-emergency@bank.com | Any legal risk, regulatory notification |
| **DPO** | dpo@bank.com | Any personal data involvement |
| **VP Engineering** | PagerDuty escalation | SEV-1 incidents |
| **CTO** | On request | SEV-1 with significant impact |
| **CEO** | Via CTO/VP Engineering | SEV-1 with major reputational impact |
| **FCA Contact** | Via RegData portal | Regulatory notification |
| **ICO Contact** | Via ICO portal | Data breach notification |
| **NCSC Contact** | incident@ncsc.gov.uk | Cybersecurity incidents |

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-classification.md](incident-classification.md) -- Severity classification
- [regulatory-notification.md](regulatory-notification.md) -- Regulatory notification procedures
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
- [communication-during-incidents.md](communication-during-incidents.md) -- Communication procedures
- [customer-communication.md](customer-communication.md) -- Customer notification templates
- [postmortem-process.md](postmortem-process.md) -- Postmortem process
- [action-item-tracking.md](action-item-tracking.md) -- Action item management
