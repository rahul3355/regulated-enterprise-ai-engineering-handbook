# Data Retention

## What Data Retention Is and Why It Matters

Data retention policies define how long data is kept and when it must be deleted. For banking GenAI systems, data retention is complex because different data types have different legal, regulatory, and business requirements -- and AI systems introduce new retention challenges.

**Key principle**: Keep data no longer than necessary. Every day you retain data is a day you bear the risk and cost of protecting it.

## Retention Requirements by Data Type

### Customer Data

| Data Type | Minimum Retention | Maximum Retention | Regulatory Driver |
|-----------|------------------|------------------|-------------------|
| Account records | 5 years after closure | 7 years | Banking regulations |
| Transaction records | 5 years | 7 years | AML/KYC regulations |
| KYC documentation | 5 years after relationship ends | 7 years | AML regulations |
| Credit application records | 25 months (adverse action) | 7 years | ECOA/Regulation B |
| Credit reports | 25 months after adverse action | 7 years | FCRA |
| Customer communications | 5 years | 7 years | FCA/MiFID II |
| Complaint records | 5 years after resolution | 7 years | FCA Consumer Duty |

### AI-Specific Data

| Data Type | Minimum Retention | Maximum Retention | Rationale |
|-----------|------------------|------------------|-----------|
| Model training data | Duration of model use + 1 year | 3 years after model retirement | Model validation, reproducibility |
| Model validation data | Duration of model use + 1 year | 3 years after model retirement | Re-validation reference |
| Model predictions | 1 year | 3 years | Dispute resolution, audit |
| Model performance logs | 2 years | 5 years | Ongoing monitoring evidence |
| AI chat logs (customer) | 90 days | 2 years | Complaint resolution, training |
| AI chat logs (internal) | 90 days | 1 year | Operational purposes |
| Prompt logs (redacted) | 90 days | 1 year | Debugging, monitoring |
| AI incident records | 5 years | 7 years | Regulatory reporting |
| Model cards and documentation | Duration of model use + 1 year | 5 years after retirement | Audit trail |
| Vector embeddings | Same as source documents | Same as source documents | Derived data |

### Security and Audit Data

| Data Type | Minimum Retention | Regulatory Driver |
|-----------|------------------|-------------------|
| Authentication logs | 3 years | PCI-DSS |
| Access logs | 3 years | SOC 2, ISO 27001 |
| Security event logs | 3 years | PCI-DSS, ISO 27001 |
| Audit trail records | 7 years | Multiple regulations |
| Backup logs | 3 years | SOC 2 |
| Penetration test reports | 3 years | PCI-DSS |
| Vulnerability scan reports | 3 years | PCI-DSS |
| Incident response records | 7 years | FCA, GDPR |

## Retention Policy Implementation

### Automated Retention

```python
class RetentionPolicy:
    """Define and enforce data retention policies."""

    POLICIES = {
        "customer_account": {
            "retention_period": timedelta(days=5 * 365),
            "start_trigger": "account_closure",
            "deletion_method": "secure_erase",
            "legal_hold_override": False,
            "exceptions": ["active_dispute", "regulatory_investigation"],
        },
        "transaction_record": {
            "retention_period": timedelta(days=7 * 365),
            "start_trigger": "transaction_date",
            "deletion_method": "secure_erase",
            "legal_hold_override": False,
            "exceptions": ["active_investigation"],
        },
        "ai_chat_log": {
            "retention_period": timedelta(days=2 * 365),
            "start_trigger": "creation_date",
            "deletion_method": "secure_delete",
            "legal_hold_override": False,
            "exceptions": ["active_complaint", "active_investigation"],
        },
        "model_prediction": {
            "retention_period": timedelta(days=3 * 365),
            "start_trigger": "prediction_date",
            "deletion_method": "secure_delete",
            "legal_hold_override": False,
            "exceptions": ["active_dispute"],
        },
        "prompt_log": {
            "retention_period": timedelta(days=365),
            "start_trigger": "creation_date",
            "deletion_method": "secure_delete",
            "legal_hold_override": False,
            "exceptions": [],
        },
        "vector_embedding": {
            "retention_period": "same_as_source",
            "start_trigger": "source_document_deletion",
            "deletion_method": "cascade_delete",
            "legal_hold_override": False,
            "exceptions": [],
        },
    }

    def evaluate(self, record: DataRecord) -> RetentionDecision:
        """Determine if a record should be deleted."""
        policy = self.POLICIES.get(record.type)
        if not policy:
            return RetentionDecision(action="review", reason="No policy defined")

        # Check for legal hold
        if self.is_under_legal_hold(record):
            return RetentionDecision(
                action="retain",
                reason="Under legal hold",
                hold_reference=record.legal_hold_ref,
            )

        # Check for exceptions
        for exception in policy["exceptions"]:
            if self.check_exception(record, exception):
                return RetentionDecision(
                    action="retain",
                    reason=f"Exception: {exception}",
                )

        # Check retention period
        start_date = self.get_start_date(record, policy["start_trigger"])
        expiry_date = start_date + policy["retention_period"]

        if datetime.utcnow() > expiry_date:
            return RetentionDecision(
                action="delete",
                reason="Retention period expired",
                expiry_date=expiry_date,
                deletion_method=policy["deletion_method"],
            )

        return RetentionDecision(
            action="retain",
            reason="Within retention period",
            expiry_date=expiry_date,
        )
```

### Legal Hold

```
LEGAL HOLD PROCESS:
├── Trigger: Litigation, regulatory investigation, audit
├── Identify all relevant data sources
├── Place hold on all affected data
│   ├── Suspend automated deletion
│   ├── Notify data owners
│   └── Document hold scope and duration
├── Maintain hold until released
│   ├── Monitor for hold violations
│   └── Regular hold review
├── Release hold when no longer needed
│   ├── Formal release authorization
│   ├── Resume automated deletion
│   └── Document release
└── Audit trail: All hold and release actions logged
```

### Deletion Methods

```
DELETION METHODS BY SENSITIVITY:
├── Standard delete:
│   └── Database DELETE, file unlink
│   └── For: Non-sensitive operational data
├── Secure delete:
│   ├── Cryptographic erasure (delete encryption key)
│   ├── Overwrite storage blocks (for direct storage)
│   └── For: PII, customer data
├── Physical destruction:
│   ├── Degaussing (magnetic media)
│   ├── Shredding (physical media)
│   └── For: Classified/restricted data on physical media
└── Verification:
    ├── Confirm deletion was successful
    ├── Document deletion in audit log
    └── Notify relevant systems
```

## Retention Challenges for AI Systems

### Training Data Retention

```
CHALLENGE: Training data must be retained for model validation and reproducibility,
but it may contain PII subject to shorter retention periods.

SOLUTION:
├── Anonymize training data where possible
│   ├── Remove direct identifiers
│   ├── Generalize quasi-identifiers
│   └── Perturb sensitive values
├── Separate model-relevant data from incidental data
│   ├── Retain only features used by the model
│   └── Delete raw source data per its own retention policy
├── Document the training data composition
│   ├── Data sources, timeframes, volumes
│   └── Retention status of each source
└── If source data must be deleted before model retirement:
    ├── Document why retraining is not feasible
    ├── Assess risk of continued model use without source data
    └── Plan for model retirement or retraining
```

### Prompt Log Retention

```
CHALLENGE: Prompt logs may contain PII, but are needed for debugging and monitoring.

SOLUTION:
├── Redact PII from prompts BEFORE logging
│   ├── Use automated PII detection and redaction
│   ├── Validate redaction effectiveness
│   └── Log redaction actions for audit
├── Apply shortest practical retention
│   ├── 90 days for operational debugging
│   ├── 1 year for compliance monitoring
│   └── Extend only for specific investigations
├── Separate metadata from content
│   ├── Retain metadata (timestamps, user IDs, model info) longer
│   └── Retain content (prompt text) for shorter period
└── Include in DSAR fulfillment
    ├── Prompt logs must be searched for customer data
    └── Customer can request deletion of their prompts
```

### Vector Embedding Retention

```
CHALLENGE: Vector embeddings are derived from source documents. When source documents
are deleted, the embeddings should be too -- but identifying which embeddings
correspond to which source documents is not always straightforward.

SOLUTION:
├── Maintain source-to-embedding mapping
│   ├── Each embedded document has metadata linking to source
│   ├── Source document ID stored with embedding
│   └── Cascading delete: delete embeddings when source is deleted
├── Use collection-level organization
│   ├── Group embeddings by source system and retention class
│   └── Delete entire collections when retention expires
├── Implement automated sync
│   ├── When source document is deleted, trigger embedding deletion
│   └── Verify embedding deletion was successful
└── If source-to-embedding mapping is lost:
    ├── Assess risk of continued embedding retention
    ├── Consider index rebuild if risk is unacceptable
    └── Document the issue and remediation plan
```

## Archival

When data must be retained but is no longer actively used, it should be archived:

```
ARCHIVAL PROCESS:
├── Identify data ready for archival (no longer actively used, but must be retained)
├── Move to lower-cost storage (cold storage, tape)
├── Maintain metadata for discovery
│   ├── What data was archived
│   ├── When and why
│   ├── Where it is stored
│   └── How to retrieve it
├── Ensure archival storage meets security requirements
│   ├── Encryption at rest
│   ├── Access controls
│   └── Integrity verification
├── Test retrieval periodically
│   ├── Verify data can be restored
│   └── Measure retrieval time against SLA
└── Delete from archival storage when retention expires
```

## Retention Compliance Monitoring

```
MONITORING CHECKLIST:
├── Are retention policies defined for all data types?
├── Are automated deletion processes running on schedule?
├── Are legal holds being honored (no accidental deletion)?
├── Are exceptions properly documented?
├── Is deletion being logged with audit trail?
├── Are archival retrievals successful and timely?
├── Are retention periods aligned with regulatory requirements?
├── Are there data stores without retention policies?
├── Is deletion actually removing data (not just marking)?
└── Are backups also being cleaned up per retention policy?
```

## Common Interview Questions

### Question 1: "How do you handle data retention for AI training data?"

**Good answer structure**:
Training data has competing requirements: you need it for model validation and reproducibility, but it may contain PII with shorter retention periods. My approach: (1) Anonymize training data before use where possible. (2) Retain only the features actually used by the model, not the full raw data. (3) Document the training data composition thoroughly. (4) If source data must be deleted before the model retires, assess whether the model can still be validated and document the risk. (5) Plan for model retirement when training data expires.

### Question 2: "What happens when a customer requests deletion under GDPR but their data was used to train an AI model?"

**Good answer structure**:
This is a known challenge. I would: (1) Delete all directly accessible data (databases, logs, vector store). (2) Assess whether the model can reproduce the individual's personal data (memorization risk). (3) If risk is low, document the assessment and retain the model. (4) If risk is high, consider fine-tuning a correction model or full retraining. (5) Communicate with the customer about the limitations. (6) For future prevention: use differential privacy during training, hash identifiers before embedding, and maintain data provenance records.

### Question 3: "How do you enforce data retention across multiple storage systems?"

**Good answer structure**:
Centralized retention management: (1) Define retention policies centrally with legal/compliance input. (2) Implement a retention service that evaluates records against policies. (3) Each storage system integrates with the retention service. (4) Automated deletion processes run on schedule. (5) Legal holds override automated deletion. (6) All deletion actions are logged. (7) Regular compliance audits verify the system is working. (8) Exception: backups must also be cleaned up, which requires coordination with the backup system's lifecycle.
