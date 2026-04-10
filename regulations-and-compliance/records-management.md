# Records Management

## What Records Management Is and Why It Matters

Records management is the systematic control of records throughout their lifecycle -- from creation through disposal. A "record" is any information created, received, or maintained as evidence of business activities.

For banking GenAI systems, records management is complex because AI systems create new types of records (model outputs, training data records, prompt logs) and process existing records in new ways.

**Regulatory drivers**:
- **Banking regulations**: Require retention of specific records for defined periods
- **SEC Rule 17a-4** (US): Electronic records retention for broker-dealers
- **FCA Handbook** (UK): Record-keeping requirements
- **MiFID II** (EU): Transaction and communication records
- **GDPR**: Right to erasure vs. retention requirements
- **Sarbanes-Oxley**: Financial records retention
- **Litigation hold**: Legal requirements to preserve records

## Records Classification

### Record Types in Banking

| Record Type | Examples | Retention | Classification |
|------------|----------|-----------|---------------|
| Customer account records | Account applications, KYC documents | 5-7 years after closure | Vital |
| Transaction records | Payment instructions, confirmations | 5-7 years | Vital |
| Credit records | Loan applications, credit decisions, collateral docs | 7 years | Vital |
| Communication records | Emails, letters, call recordings with customers | 5-7 years | Important |
| Compliance records | SARs, CTRs, compliance reviews | 5-7 years | Vital |
| Regulatory reports | COREP, FINREP, FR Y-9C | 7 years | Vital |
| Board minutes | Board and committee meeting minutes | Permanent | Vital |
| Financial records | Financial statements, audit reports | 7 years | Vital |
| Model records | Model documentation, validation reports, performance logs | 5 years after retirement | Important |
| AI system records | AI chat logs, prompt logs, model outputs | 1-3 years | Important |
| System logs | Access logs, security event logs, audit trails | 3-7 years | Important |

### Classification Scheme

```
RECORD CLASSIFICATION LEVELS:
├── Vital:
│   ├── Essential for legal, regulatory, or business continuity
│   ├── Loss would cause severe regulatory or legal consequences
│   ├── Examples: Customer accounts, regulatory reports, board minutes
│   └── Controls: Multiple backups, integrity verification, restricted access
├── Important:
│   ├── Significant business value
│   ├── Loss would cause operational difficulties
│   ├── Examples: Model records, AI system records, internal reports
│   └── Controls: Regular backups, access controls
├── Useful:
│   ├── Supports business operations
│   ├── Loss would cause inconvenience but not severe consequences
│   ├── Examples: Working papers, drafts, reference materials
│   └── Controls: Standard backup
└── Temporary:
    ├── Short-term value only
    ├── Examples: Duplicates, convenience copies, rough drafts
    └── Controls: Dispose when no longer needed
```

## Records Lifecycle

```
RECORDS LIFECYCLE
══════════════════

  ┌───────────┐    ┌───────────┐    ┌───────────┐
  │ CREATION  ├───►│ MAINTENANCE├───►│ RETENTION │
  │ & CAPTURE │    │ & USE     │    │           │
  └───────────┘    └───────────┘    └─────┬─────┘
                                          │
  ┌───────────┐    ┌───────────┐          │
  │ DISPOSAL  │◄───┤ ARCHIVE   │◄─────────┘
  │           │    │           │
  └───────────┘    └───────────┘
```

### Phase 1: Creation and Capture

```
RECORD CREATION CONTROLS:
├── Unique identifier assigned to each record
├── Metadata captured at creation:
│   ├── Creator/author
│   ├── Creation date
│   ├── Record type
│   ├── Classification level
│   ├── Retention category
│   └── Access permissions
├── Record linked to related records (parent-child relationships)
├── Record stored in approved repository
├── Version control for record updates
└── Audit trail of creation
```

### Phase 2: Maintenance and Use

```
RECORD MAINTENANCE CONTROLS:
├── Access controls based on classification
├── Version management (no overwriting of original records)
├── Audit trail of all access and modifications
├── Integrity verification (checksums, digital signatures)
├── Backup according to classification requirements
├── Search and retrieval capabilities
├── Cross-references to related records
└── Usage monitoring
```

### Phase 3: Retention

```
RETENTION CONTROLS:
├── Retention schedule defined for each record category
├── Retention period starts from defined trigger event
│   ├── Account closure for customer records
│   ├── Transaction date for transaction records
│   ├── Report submission for regulatory reports
│   └── Model retirement for model records
├── Automated retention tracking
├── Legal hold overrides automated disposal
├── Periodic review of retention compliance
└── Disposition scheduling
```

### Phase 4: Archive

```
ARCHIVE CONTROLS:
├── Records moved to archive when no longer actively used
├── Archive storage meets security requirements
│   ├── Encryption at rest
│   ├── Access controls
│   └── Environmental protection
├── Archive index maintained for discovery
├── Retrieval procedures documented and tested
├── Archive integrity verified periodically
└── Archive retention period tracked
```

### Phase 5: Disposal

```
DISPOSAL CONTROLS:
├── Disposal authorization (records manager sign-off)
├── Legal hold check before disposal
├── Disposal method based on classification:
│   ├── Vital records: Secure destruction with certificate
│   ├── Important records: Secure destruction
│   ├── Useful records: Standard deletion
│   └── Temporary records: Routine deletion
├── Disposal logged in audit trail
│   ├── Record identifier
│   ├── Disposal date
│   ├── Disposal method
│   ├── Authorization reference
│   └── Verification of successful disposal
└── Disposal certificate generated for vital records
```

## AI-Specific Records

### Model Records

```
MODEL RECORDS INVENTORY:
├── Model development records:
│   ├── Model design documents
│   ├── Training data documentation
│   ├── Development testing results
│   └── Code repository records
├── Model validation records:
│   ├── Independent validation reports
│   ├── Validation findings and remediation
│   └── Re-validation records
├── Model deployment records:
│   ├── Deployment approval
│   ├── Deployment configuration
│   └── Go-live sign-off
├── Model operation records:
│   ├── Performance monitoring reports
│   ├── Incident records
│   ├── Change records
│   └── User feedback
├── Model retirement records:
│   ├── Retirement decision and rationale
│   ├── Data disposition records
│   └── Post-retirement review
└── Retention: 5 years after model retirement
```

### AI Interaction Records

```
AI INTERACTION RECORDS:
├── Customer-facing AI interactions:
│   ├── Chat transcripts (redacted of PII)
│   ├── AI responses
│   ├── User feedback
│   ├── Escalation records (human handoff)
│   └── Satisfaction scores
│   └── Retention: 2 years minimum
├── Internal AI interactions:
│   ├── Prompt logs (redacted)
│   ├── AI outputs
│   ├── Human modifications
│   └── Usage statistics
│   └── Retention: 1 year minimum
├── AI training records:
│   ├── Training datasets used
│   ├── Training parameters
│   ├── Training results
│   └── Evaluation metrics
│   └── Retention: 3 years after model retirement
└── AI governance records:
    ├── Risk classification records
    ├── Governance committee minutes
    ├── Policy documents
    └── Compliance assessments
    └── Retention: 5 years minimum
```

## Electronic Records Management System

```
ERMS REQUIREMENTS:
├── Record declaration:
│   ├── Automatic classification based on content/type
│   ├── Manual classification override with audit trail
│   └── Metadata capture at declaration
├── Storage:
│   ├── Secure, access-controlled storage
│   ├── Multiple copies for vital records
│   ├── Format preservation
│   └── Integrity verification
├── Retrieval:
│   ├── Search by metadata, content, date range
│   ├── Access logging
│   ├── Export in standard formats
│   └── Response time SLAs
├── Retention management:
│   ├── Automated retention scheduling
│   ├── Legal hold management
│   ├── Disposition workflow
│   └── Disposition audit trail
├── Security:
│   ├── Encryption at rest and in transit
│   ├── Role-based access controls
│   ├── Tamper detection
│   └── Disaster recovery
└── Compliance:
    ├── Regulatory retention schedules
    ├── Audit reporting
    ├── Legal discovery support
    └── Records management reporting
```

## Legal Discovery

```
LEGAL DISCOVERY PROCESS:
├── Litigation hold initiated
│   ├── Identify all relevant record types
│   ├── Suspend disposal for affected records
│   ├── Notify record owners
│   └── Document hold scope
├── Discovery request received
│   ├── Define scope of request
│   ├── Identify relevant records
│   ├── Collect records from all sources
│   └── Preserve chain of custody
├── Records reviewed
│   ├── Privilege review
│   ├── Relevance review
│   └── Redaction of privileged content
├── Records produced
│   ├── Production format as specified
│   ├── Production log maintained
│   └── Certificate of production
└── Hold released
    ├── Authorization from legal
    ├── Resume normal retention
    └── Document release
```

## Common Interview Questions

### Question 1: "How do you manage records created by an AI system?"

**Good answer structure**:
AI-created records should be managed the same way as human-created records of the same type. A regulatory report draft by AI is still a regulatory record. An AI chat log with a customer is still a communication record. Key considerations: (1) Classify AI records by their content and purpose, not by their creator. (2) Capture metadata about AI creation (model version, timestamp, prompt). (3) Apply the same retention schedules as equivalent human-created records. (4) Ensure AI records are included in legal discovery. (5) Document the AI's role in record creation for audit purposes.

### Question 2: "How do you handle the conflict between GDPR's right to erasure and regulatory retention requirements?"

**Good answer structure**:
Regulatory retention requirements generally override GDPR's right to erasure when there is a legal obligation to retain data. The approach: (1) Identify the legal basis for retention (specific regulation requiring the data). (2) Document this as the basis for not erasing the specific data. (3) Erase any data NOT subject to retention requirements. (4) Restrict processing of retained data to the retention purpose only. (5) Delete when the retention period expires. (6) Communicate to the individual why some data cannot be erased and when it will be deleted.

### Question 3: "What's the minimum set of records you need to retain for an AI model?"

**Good answer structure**:
For model risk management compliance: (1) Model documentation (design, methodology, assumptions, limitations). (2) Training data documentation (sources, preparation, quality assessment). (3) Validation reports (independent validation findings, remediation). (4) Deployment records (approval, configuration). (5) Performance monitoring records (regular reports, incidents, changes). (6) Change records (all modifications with impact assessments). (7) Retirement records (decision, data disposition). Retention: 5 years minimum after model retirement. These records must be sufficient for an auditor to understand what the model did, how it was validated, how it performed, and what changes were made.
