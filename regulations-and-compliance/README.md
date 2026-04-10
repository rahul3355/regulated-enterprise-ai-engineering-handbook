# Regulations and Compliance for GenAI in Banking

## Overview

This folder is the definitive engineering reference for every regulation, standard, and compliance framework that affects how you design, build, deploy, and operate GenAI systems at a global bank.

**Compliance is not optional.** A single violation can result in fines ranging from millions to billions of dollars, personal liability for executives, loss of banking licenses, and irreparable reputational damage. As an engineer working on banking GenAI systems, you are on the front line of compliance.

## Why This Matters to Engineers

Regulations are often written in legal language, but their enforcement happens through **code**. Every API endpoint you design, every log you write, every encryption key you manage, every prompt you log -- these are the artifacts that auditors examine and regulators scrutinize.

### Real-World Impact

| Incident | Fine / Cost | Root Cause |
|----------|------------|------------|
| Wells Fargo fake accounts (2016) | $3.7 billion | Inadequate internal controls |
| Capital One data breach (2019) | $190 million | Misconfigured cloud storage, inadequate access controls |
| GDPR violations by Meta (2023) | €1.2 billion ($1.3B) | Illegal data transfers |
| BaFin AI guidance violations (2021) | Multiple enforcement actions | Inadequate model governance |
| FCA fines 2020-2024 | £1.4 billion total | Various compliance failures |

## How to Use This Folder

### For Daily Development

1. **Before writing any code** that handles customer data, read `gdpr.md` and `pii-handling.md`.
2. **Before integrating any external AI API** (OpenAI, Anthropic, etc.), read `vendor-risk-management.md` and `third-party-ai-risk.md`.
3. **Before deploying any model** to production, read `sr11-7-model-risk.md`, `model-risk-management.md`, and `ai-governance.md`.
4. **Before designing any data pipeline**, read `data-retention.md`, `consent-and-data-usage.md`, and `secure-data-sharing.md`.
5. **Before setting up logging**, read `audit-trails.md` and `data-retention.md`.

### For Architecture Reviews

- `iso-27001.md` -- Information security baseline
- `soc2.md` -- Trust service criteria
- `pci-dss.md` -- If handling payment card data
- `bcbs-239.md` -- Risk data aggregation requirements
- `fca-guidelines.md` -- UK regulatory expectations

### For Interviews

Each file includes a "Common Interview Questions" section. These are actual questions asked in banking GenAI engineering interviews at institutions like JPMorgan Chase, Goldman Sachs, HSBC, Barclays, and Morgan Stanley.

## The Compliance Mindset

### Principle 1: Assume Everything Is Audited

Every system you build will be audited. Design for auditability from day one:

```
GOOD:    Every API request is logged with user ID, timestamp, data accessed, and purpose.
BAD:     "We'll add logging later if we need it."
```

### Principle 2: Data Minimization Is Your Friend

Collect, store, and process the minimum data necessary. This reduces your attack surface and compliance burden:

```
GOOD:    Hash the customer ID before sending it to the vector database.
BAD:     Send the full customer record "just in case we need it."
```

### Principle 3: Document Your Decisions

Every architecture decision that affects compliance must be documented:

```
GOOD:    ADR-042: "We use AES-256-GCM for PII at rest because PCI-DSS v4.0 requires strong cryptography."
BAD:     "We encrypted the database because security said so."
```

### Principle 4: Compliance Is a Feature

Treat compliance requirements as product requirements. A system that passes audit is more valuable than a system that doesn't, regardless of how many features it has.

## Regulatory Landscape Map

```
                    BANKING GENAI COMPLIANCE LANDSCAPE
                    ==================================

    DATA PROTECTION           FINANCIAL REGULATIONS
    ┌─────────────────┐       ┌─────────────────────┐
    │ GDPR (EU/UK)    │       │ SR 11-7 (US)        │
    │ PCI-DSS         │       │ BCBS 239 (Global)   │
    │ Consent Mgmt    │       │ FCA Guidelines (UK) │
    │ Data Retention  │       │ Model Risk Mgmt     │
    └────────┬────────┘       └──────────┬──────────┘
             │                           │
             │    AI-SPECIFIC            │
             │    ┌──────────────┐       │
             └───►│ EU AI Act    │◄──────┘
                  │ NIST AI RMF │
                  │ AI Gov Frame│
                  └──────┬───────┘
                         │
    SECURITY STANDARDS   │    OPERATIONAL
    ┌─────────────────┐  │    ┌─────────────────────┐
    │ ISO 27001       │  │    │ Audit Trails        │
    │ SOC 2 Type II   │──┘    │ Access Control      │
    └─────────────────┘       │ Vendor Risk Mgmt    │
                              │ Secure Data Sharing │
                              └─────────────────────┘
```

## Compliance by System Component

### APIs
- Must authenticate and authorize every request (`access-control.md`)
- Must log all access to regulated data (`audit-trails.md`)
- Must validate and sanitize all inputs
- Must enforce data usage policies (`consent-and-data-usage.md`)

### Frontend
- Must not display PII in URLs or browser storage
- Must implement proper session management
- Must display required consent notices
- Must not log PII in browser console

### Backend
- Must encrypt PII at rest and in transit
- Must implement RBAC/ABAC (`access-control.md`)
- Must not log PII in application logs
- Must handle data subject requests (DSARs)

### Databases
- Must encrypt all PII columns at rest
- Must support right-to-erasure (GDPR Article 17)
- Must maintain audit trails for all modifications
- Must implement column-level access controls

### CI/CD
- Must scan for secrets in every commit
- Must require compliance sign-off before production deployment
- Must maintain artifact provenance
- Must enforce infrastructure-as-code reviews

### Monitoring
- Must detect unauthorized access to regulated data
- Must alert on compliance policy violations
- Must track model performance drift
- Must monitor data retention policy compliance

### AI Systems
- Must classify models under SR 11-7
- Must validate models before production use
- Must monitor for bias, drift, and adversarial attacks
- Must document model cards and data lineage

### Prompt Logging
- Must never log PII in prompts
- Must redact sensitive data before storage
- Must apply same retention policies as other data
- Must be included in DSAR fulfillment

### Vector Databases
- Must hash or tokenize PII before embedding
- Must implement access controls at collection level
- Must support data deletion (not trivial with embeddings)
- Must maintain data lineage for embedded content

### Model Training
- Must verify training data consent
- Must document data sources and lineage
- Must test for bias across protected characteristics
- Must maintain reproducibility

## Quick Reference: Fines by Regulation

| Regulation | Maximum Fine | Notable Enforcement Action |
|-----------|-------------|---------------------------|
| GDPR | €20M or 4% of global revenue | Meta €1.2B (2023) |
| PCI-DSS | $500K-$100K per month | Multiple merchant breaches |
| SR 11-7 | Enforcement orders, cease & desist | Multiple bank consent orders |
| FCA | Unlimited | Barclays £72M (2021) |
| BCBS 239 | Pillar 2 add-ons, public censure | Several European banks |
| EU AI Act | €35M or 7% of global revenue | First enforcement 2026+ |

## File Index

| File | Topic | Priority |
|------|-------|----------|
| `gdpr.md` | General Data Protection Regulation | CRITICAL |
| `pci-dss.md` | Payment Card Industry Data Security Standard | HIGH |
| `soc2.md` | SOC 2 Type II Compliance | HIGH |
| `iso-27001.md` | Information Security Management | HIGH |
| `bcbs-239.md` | Risk Data Aggregation | HIGH |
| `sr11-7-model-risk.md` | Model Risk Management (US) | CRITICAL |
| `fca-guidelines.md` | UK Financial Conduct Authority | CRITICAL |
| `ai-governance.md` | AI Governance Framework | CRITICAL |
| `model-risk-management.md` | Ongoing Model Risk | CRITICAL |
| `audit-trails.md` | Audit Logging Requirements | CRITICAL |
| `explainability.md` | AI Explainability | HIGH |
| `data-retention.md` | Data Retention Policies | HIGH |
| `pii-handling.md` | PII Handling | CRITICAL |
| `consent-and-data-usage.md` | Consent Management | CRITICAL |
| `vendor-risk-management.md` | Third-Party Risk | HIGH |
| `third-party-ai-risk.md` | External AI Provider Risks | CRITICAL |
| `regulatory-reporting.md` | Regulatory Reporting | HIGH |
| `records-management.md` | Records Management | HIGH |
| `access-control.md` | Access Control Systems | CRITICAL |
| `secure-data-sharing.md` | Secure Data Sharing | HIGH |

## Glossary

- **DSAR**: Data Subject Access Request
- **DPIA**: Data Protection Impact Assessment
- **PII**: Personally Identifiable Information
- **RBAC**: Role-Based Access Control
- **ABAC**: Attribute-Based Access Control
- **DPO**: Data Protection Officer
- **CDE**: Cardholder Data Environment
- **MRM**: Model Risk Management
- **RMF**: Risk Management Framework
- **DLP**: Data Loss Prevention
- **HSM**: Hardware Security Module
- **SIEM**: Security Information and Event Management

## Related Folders

- `../incident-management/` -- How to handle compliance breaches
- `../interview-prep/` -- Compliance-related interview questions
- `../engineering-philosophy/` -- Building with compliance in mind
