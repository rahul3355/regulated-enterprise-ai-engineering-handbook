# Consent and Data Usage

## What Consent Is and Why It Matters

Consent is the legal basis for processing personal data when no other lawful basis applies. In banking, consent is one of several lawful bases (alongside contract performance, legal obligation, and legitimate interests), but it is the most commonly misunderstood.

**Key principle**: Consent must be FREELY GIVEN, SPECIFIC, INFORMED, and UNAMBIGUOUS. If any of these is missing, the consent is not valid under GDPR and similar regulations.

For GenAI systems, consent is particularly relevant for:
- Optional AI features (not required for core banking)
- Using customer data to improve AI models
- Sharing customer data with AI vendors
- AI-driven profiling and personalization
- Retaining customer conversations for training

## Lawful Bases for Processing (GDPR Article 6)

Before considering consent, check if another lawful basis applies:

| Basis | When It Applies | Banking Example |
|-------|----------------|-----------------|
| **Contract performance** | Processing is necessary for a contract with the individual | Processing transactions, account management |
| **Legal obligation** | Processing is necessary to comply with the law | AML/KYC checks, tax reporting |
| **Vital interests** | Processing is necessary to protect someone's life | Emergency fraud prevention |
| **Public task** | Processing is necessary for a task in the public interest | Regulatory reporting to central bank |
| **Legitimate interests** | Processing is necessary for legitimate interests, not overridden by individual's rights | Fraud detection analytics, security monitoring |
| **Consent** | The individual has given clear consent | Optional AI financial advisor, marketing personalization |

### When to Use Consent for AI

```
USE CONSENT WHEN:
├── AI feature is optional (not needed for core banking service)
├── Customer data is used in ways beyond the original purpose
├── Data is shared with third-party AI providers
├── Customer conversations are used for model training
├── AI profiling goes beyond what is expected for the service
└── Special category data must be processed (requires explicit consent)

DO NOT USE CONSENT WHEN:
├── Processing is required for core banking service (use contract performance)
├── Processing is required by law (use legal obligation)
├── Processing is for fraud detection (use legitimate interests)
├── You would deny service if consent is withdrawn (not freely given)
└── You cannot offer the service without the processing (use contract performance)
```

## Consent Requirements

### Valid Consent Must Be

```
VALID CONSENT CHECKLIST:
├── Freely given:
│   ├── No detriment for refusing consent
│   ├── No bundled consent (must be separate from terms of service)
│   ├── No pre-ticked boxes (must be active choice)
│   └── Service not conditional on consent (if not necessary for service)
├── Specific:
│   ├── Separate consent for separate processing activities
│   ├── Granular options (not "accept all or nothing")
│   └── Clear description of what is being consent to
├── Informed:
│   ├── Identity of data controller clearly stated
│   ├── Purpose of processing clearly explained
│   ├── Types of data to be processed specified
│   ├── Third parties with access identified
│   ├── Right to withdraw consent explained
│   └── Consequences of not consenting explained
├── Unambiguous:
│   ├── Clear affirmative action (not silence or inaction)
│   ├── Opt-in mechanism (not opt-out)
│   └── No ambiguity about what the user is consenting to
└── Easy to withdraw:
    ├── Withdrawal as easy as giving consent
    ├── No detriment for withdrawal
    └── Processing stops promptly after withdrawal
```

### Consent Collection Implementation

```python
class ConsentManager:
    """Manage consent collection and enforcement."""

    def record_consent(self, request: ConsentRequest) -> ConsentRecord:
        """Record a user's consent decision."""
        record = ConsentRecord(
            user_id=request.user_id,
            consent_type=request.consent_type,
            granted=request.granted,
            timestamp=datetime.utcnow(),
            ip_address=request.ip_address,
            user_agent=request.user_agent,
            privacy_notice_version=self.get_current_notice_version(
                request.consent_type
            ),
            granular_choices=request.granular_choices,
        )

        # Store in consent database
        self.consent_db.store(record)

        # Notify downstream systems
        if not request.granted:
            self.notify_consent_denied(request.user_id, request.consent_type)
        else:
            self.notify_consent_granted(request.user_id, request.consent_type)

        return record

    def check_consent(self, user_id: str, processing_type: str) -> ConsentCheck:
        """Check if valid consent exists for a processing type."""
        consent = self.consent_db.get_latest(user_id, processing_type)

        if not consent:
            return ConsentCheck(valid=False, reason="No consent on record")

        if not consent.granted:
            return ConsentCheck(valid=False, reason="Consent was declined")

        if consent.withdrawn:
            return ConsentCheck(
                valid=False,
                reason=f"Consent withdrawn on {consent.withdrawal_date}",
            )

        # Check consent is still valid (not expired)
        if consent.expiry_date and datetime.utcnow() > consent.expiry_date:
            return ConsentCheck(valid=False, reason="Consent has expired")

        return ConsentCheck(valid=True, consent_record=consent)
```

## Consent for AI-Specific Processing

### Using Customer Data for Model Training

```
CONSENT FOR AI TRAINING:
├── Separate consent required from core service consent
├── Must specify:
│   ├── What data will be used (conversations, transactions, profile)
│   ├── How it will be used (model training, evaluation, testing)
│   ├── Who will have access (internal team, vendor, open source)
│   ├── How long data will be retained
│   ├── Whether data will be anonymized
│   └── Right to withdraw consent and consequences
├── Must be easy to withdraw (and data removed from training pipeline)
├── Must not condition core service on training consent
└── Must re-request consent if training purpose changes
```

### Using Customer Conversations for AI Improvement

```
CONSENT FOR CONVERSATION RETENTION:
├── Separate consent for:
│   ├── Storing conversations beyond service delivery period
│   ├── Using conversations to improve AI models
│   ├── Sharing conversations with AI vendors for improvement
│   └── Human review of conversations for quality
├── Default: Do NOT retain conversations without consent
├── Default: Do NOT use conversations for training without consent
├── Must offer granular options (storage vs. training vs. sharing)
├── Must honor withdrawal promptly
└── Must delete conversations when consent is withdrawn
```

### Consent for Third-Party AI Processing

```
CONSENT FOR THIRD-PARTY AI:
├── Disclose that data will be processed by [specific vendor]
├── Describe what data will be sent (prompts, metadata, customer ID)
├── Describe how the vendor will process the data
├── Disclose if the vendor may use data for their own purposes
│   (e.g., OpenAI using prompts to improve their models)
├── Provide opt-out (use internal AI model instead)
├── If no alternative available: cannot process without consent
└── Document the data transfer mechanism (SCCs, adequacy decision)
```

## Purpose Limitation

Data collected for one purpose cannot be used for a different purpose without a new lawful basis.

```
PURPOSE LIMITATION EXAMPLES:
├── Customer provided data for account opening:
│   ├── OK: Use for account management (same purpose)
│   ├── OK: Use for fraud detection (compatible purpose)
│   ├── NOT OK: Use for AI marketing model (new purpose, need consent)
│   └── NOT OK: Share with AI vendor for their training (new purpose, need consent)
├── Customer chat logs from support conversations:
│   ├── OK: Use to resolve the customer's issue (original purpose)
│   ├── NOT OK: Use for AI model training (new purpose, need consent)
│   └── NOT OK: Share with vendor for product improvement (new purpose, need consent)
```

### Compatible Purpose Assessment

Before using data for a new purpose, assess compatibility:

```
COMPATIBLE PURPOSE ASSESSMENT:
├── Is there a link between the original and new purpose?
├── Was the data collected in a way the individual would expect?
├── What is the nature of the data?
├── What are the possible consequences of the new processing?
├── Are there appropriate safeguards in place?
└── Would a reasonable person expect this use?

If YES to all: May be compatible (document assessment)
If NO to any: Need new consent or other lawful basis
```

## Consent Documentation

```
CONSENT RECORD REQUIREMENTS:
├── Who: User ID of the individual
├── What: Specific consent type and scope
├── When: Timestamp of consent
├── How: Mechanism used to obtain consent (UI screenshot, form version)
├── Which version: Privacy notice version at time of consent
├── From where: IP address, user agent (for audit)
├── Granular choices: Which specific options were selected
├── Changes: History of consent updates and withdrawals
└── Withdrawal: If withdrawn, when and how
```

## Enforcement and Monitoring

### Consent Enforcement in Data Pipeline

```python
class ConsentEnforcer:
    """Enforce consent decisions across data pipelines."""

    def __init__(self, consent_manager: ConsentManager):
        self.consent_manager = consent_manager

    def filter_by_consent(self, data: DataFrame, processing_type: str) -> DataFrame:
        """Filter data to only include records with valid consent."""
        valid_users = []
        for user_id in data["user_id"].unique():
            if self.consent_manager.check_consent(user_id, processing_type).valid:
                valid_users.append(user_id)

        return data[data["user_id"].isin(valid_users)]

    def block_processing(self, user_id: str, processing_type: str) -> bool:
        """Check if processing should be blocked due to consent."""
        check = self.consent_manager.check_consent(user_id, processing_type)
        if not check.valid:
            self.audit_logger.log_blocked_processing(
                user_id=user_id,
                processing_type=processing_type,
                reason=check.reason,
            )
            return True
        return False
```

### Consent Monitoring

```
CONSENT MONITORING CHECKLIST:
├── Are consent records complete and accurate?
├── Are consent withdrawals being honored promptly?
├── Is processing being blocked for users who declined consent?
├── Are consent forms still current (privacy notice up to date)?
├── Are there any processing activities without documented lawful basis?
├── Is consent granularity being enforced (not treating partial consent as full)?
├── Are consent records being retained for required periods?
├── Is the consent collection UX still compliant (no dark patterns)?
└── Are third-party processing activities covered by consent?
```

## Common Interview Questions

### Question 1: "Can you use customer support chat logs to train an AI model?"

**Good answer structure**:
Not without a specific lawful basis. The original purpose of collecting chat logs was to resolve the customer's support issue. Using them for model training is a different purpose. Options: (1) Obtain specific consent for training use, clearly explaining what data will be used and how. (2) Anonymize the data so it is no longer personal data. (3) Assess if training is a "compatible purpose" -- this is unlikely for training without explicit disclosure. The safest approach is to obtain separate, informed consent and allow customers to opt out without any detriment to their core service.

### Question 2: "A customer withdraws consent for AI processing. What happens?"

**Good answer structure**:
Withdrawal must be as easy as giving consent, and processing must stop promptly. Specifically: (1) Update the consent record immediately. (2) Notify all downstream systems to stop processing. (3) Delete data that was processed solely on the basis of consent. (4) If data also has another lawful basis (e.g., contract performance), that processing can continue. (5) For AI training data, assess whether the customer's data can be removed from trained models -- this may require retraining or fine-tuning corrections. (6) Document the withdrawal and actions taken.

### Question 3: "When is consent NOT the right lawful basis for AI processing?"

**Good answer**:
Consent is not appropriate when: (1) The processing is necessary for core banking services -- use contract performance instead. (2) The processing is required by law (AML, KYC) -- use legal obligation. (3) Denying consent would mean denying service -- consent is not "freely given." (4) The processing is for fraud detection -- use legitimate interests. (5) The bank cannot practically honor withdrawal requests -- consent requires easy withdrawal. In these cases, use the appropriate alternative lawful basis and document the assessment.
