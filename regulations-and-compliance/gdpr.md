# General Data Protection Regulation (GDPR)

## What Is GDPR and Why It Matters

The General Data Protection Regulation (EU 2016/679) is the most comprehensive data protection law in the world. It applies to **any organization** that processes the personal data of individuals in the EU/EEA, regardless of where the organization is located.

**Maximum fine**: €20 million or 4% of global annual revenue (whichever is higher).

For GenAI systems in banking, GDPR is not just about customer databases -- it affects every component of your AI pipeline:

- **Training data**: Must have a lawful basis for processing every record
- **Prompt inputs**: Customer prompts may contain PII and must be handled accordingly
- **Model outputs**: Generated content may constitute profiling or automated decision-making
- **Vector databases**: Embeddings of personal data ARE personal data
- **Fine-tuning datasets**: Subject to the same consent and purpose limitations as raw data

### Key GDPR Principles (Article 5)

1. **Lawfulness, fairness, transparency** -- Process data legally and inform data subjects
2. **Purpose limitation** -- Only collect data for specified, explicit purposes
3. **Data minimization** -- Collect only what you need
4. **Accuracy** -- Keep personal data accurate and up to date
5. **Storage limitation** -- Delete data when no longer needed
6. **Integrity and confidentiality** -- Secure the data
7. **Accountability** -- Demonstrate compliance

## Lawful Bases for Processing (Article 6)

Every piece of personal data processed by your GenAI system must have a valid lawful basis:

| Basis | Description | Applicable to GenAI? |
|-------|-------------|---------------------|
| **Consent** | User explicitly agrees | Yes -- for optional AI features |
| **Contract performance** | Needed to fulfill a contract | Yes -- for core banking AI features |
| **Legal obligation** | Required by law | Yes -- for compliance AI |
| **Vital interests** | Life-or-death situations | Rarely |
| **Public task** | Official authority | Rarely for banks |
| **Legitimate interests** | Your legitimate interests, balanced against individual rights | Yes -- but requires LIA documentation |

**Engineering implication**: Your system MUST track the lawful basis for each data element. This is not optional -- without a lawful basis, all subsequent processing is illegal.

```python
# Example: Track lawful basis for each data field
class CustomerDataRecord:
    customer_id: str
    transaction_history: List[Transaction]
    lawful_basis: str = "contract_performance"  # For core banking
    purpose: str = "fraud_detection_ai"
    consent_record: Optional[ConsentRecord] = None  # If basis is "consent"
```

## Data Subject Rights and Engineering Requirements

### Right of Access (Article 15) -- DSAR

Individuals can request all personal data you hold about them. Your GenAI system must be able to:

1. **Identify** all personal data across all systems (databases, vector stores, logs, model training data, prompt logs)
2. **Export** it in a structured, machine-readable format
3. **Respond** within 30 calendar days (extendable by 60 for complex requests)

**Engineering requirements**:

```
MANDATORY CONTROLS:
├── Data catalog mapping all PII storage locations
├── Automated DSAR workflow system
├── Prompt log search by customer identifier
├── Vector database PII index (if embeddings contain personal data)
├── Model training data provenance records
├── Third-party data sharing records
└── SLA monitoring for 30-day response deadline
```

**Implementation pattern**:

```python
# DSAR handler for a GenAI customer service system
class DSARHandler:
    def collect_customer_data(self, customer_id: str) -> DataExport:
        """Collect ALL personal data for a customer."""
        return DataExport(
            core_profile=self.database.get_profile(customer_id),
            transaction_history=self.database.get_transactions(customer_id),
            chat_history=self.ai_service.get_chat_logs(customer_id),
            prompt_history=self.ai_service.get_prompts(customer_id),
            model_inputs=self.ai_service.get_model_inputs(customer_id),
            model_outputs=self.ai_service.get_model_outputs(customer_id),
            vector_embeddings=self.vector_store.get_customer_embeddings(customer_id),
            training_data_records=self.ml_platform.get_training_records(customer_id),
            third_party_shares=self.consent_manager.get_shares(customer_id),
        )
```

### Right to Erasure (Right to be Forgotten) (Article 17)

Individuals can request deletion of their personal data when:
- The data is no longer necessary for the original purpose
- They withdraw consent
- They object to processing (and no overriding legitimate interest exists)
- The data was unlawfully processed
- Erasure is required by legal obligation

**Engineering requirements**:

```
MANDATORY CONTROLS:
├── Delete from primary database (with cascade)
├── Delete from backup systems (or anonymize)
├── Delete from vector database (re-embed without PII or delete collection)
├── Delete from prompt logs
├── Delete from model training data (may require model retraining)
├── Delete from cache layers (Redis, CDN)
├── Delete from message queues
├── Notify downstream systems
└── Generate deletion certificate
```

**The vector database problem**: If a customer's data was embedded into a vector database, you cannot simply "delete" their data -- the embedding is a mathematical representation spread across the index.

**Solutions**:
1. **Hash customer identifiers** before embedding (preferred)
2. **Use collection-level deletion** -- if each customer has their own collection
3. **Rebuild the index** without the customer's data (expensive but definitive)
4. **Use machine unlearning** techniques (emerging field, not yet production-ready)

### Right to Rectification (Article 16)

Individuals can request correction of inaccurate personal data.

**Engineering requirements**:
- Implement data correction workflows that propagate to all systems
- Update vector embeddings when source data changes
- Log all corrections with timestamps and reasons

### Right to Object (Article 21)

Individuals can object to processing based on legitimate interests, including profiling.

**Engineering requirements**:
- Implement opt-out mechanisms for AI-driven profiling
- Honor objections within a reasonable timeframe
- Document the objection and any overriding legitimate interests

### Right Not to Be Subject to Automated Decision-Making (Article 22)

This is the most critical article for GenAI systems. Individuals have the right **not to be subject to a decision based solely on automated processing** (including profiling) that produces legal effects or similarly significantly affects them.

**Applies to**:
- Credit decisions
- Loan approvals/rejections
- Insurance pricing
- Employment decisions
- Fraud flagging that restricts account access

**Engineering requirements**:

```
MANDATORY CONTROLS FOR ARTICLE 22:
├── Human-in-the-loop for all significant automated decisions
├── Clear explanation of logic used (see explainability.md)
├── Ability to contest the decision
├── Ability to express their point of view
├── Right to human intervention
└── Documentation of meaningful information about the logic involved
```

```python
# Example: Credit decision with Article 22 compliance
class CreditDecisionService:
    def evaluate_credit_application(self, application: CreditApplication) -> Decision:
        # AI model provides recommendation
        ai_score = self.model.score(application)
        ai_recommendation = self.model.recommend(ai_score)

        # Human reviewer MUST be involved for significant decisions
        if ai_recommendation.is_significant():  # e.g., loan rejection
            human_review = HumanReviewQueue.create(
                application=application,
                ai_score=ai_score,
                ai_recommendation=ai_recommendation,
                explanation=self.explainer.explain(ai_score),
            )
            return Decision(
                status="pending_human_review",
                review_id=human_review.id,
            )
        else:
            return Decision(status=ai_recommendation.status)
```

## Special Category Data (Article 9)

Special category data requires **additional** protection:
- Racial or ethnic origin
- Political opinions
- Religious or philosophical beliefs
- Trade union membership
- Genetic data
- Biometric data (for identification purposes)
- Health data
- Sex life or sexual orientation
- Criminal convictions and offences (Article 10)

**Processing special category data is PROHIBITED** unless a specific exception applies. Most banking GenAI systems should NOT process this data.

**Engineering requirements**:
```
MANDATORY CONTROLS:
├── Automatic detection and flagging of special category data in inputs
├── Automatic rejection or redaction of prompts containing special category data
├── Explicit consent workflow (separate from general consent)
├── Enhanced access controls (need-to-know basis only)
├── Enhanced encryption
├── Separate audit trail
└── DPO notification workflow
```

```python
# Detect and handle special category data in AI inputs
class SpecialCategoryDetector:
    SPECIAL_CATEGORY_PATTERNS = [
        r'\b(disability|disabled|wheelchair)\b',       # Health data
        r'\b(pregnant|pregnancy)\b',                     # Health data
        r'\b(race|ethnicity)\b',                         # Racial origin
        r'\b(biometric|fingerprint|facial recognition)\b', # Biometric
        # ... many more patterns
    ]

    def detect(self, text: str) -> List[CategoryMatch]:
        matches = []
        for pattern in self.SPECIAL_CATEGORY_PATTERNS:
            for match in re.finditer(pattern, text, re.IGNORECASE):
                matches.append(CategoryMatch(
                    category=self.classify(pattern),
                    text=match.group(),
                    position=match.span(),
                ))
        return matches

    def handle(self, prompt: str) -> ProcessingDecision:
        matches = self.detect(prompt)
        if matches:
            # Log detection, redact, and require explicit consent
            self.audit_logger.log_detection(matches)
            redacted = self.redact_special_categories(prompt, matches)
            return ProcessingDecision(
                allowed=False,
                reason="Special category data detected",
                redacted_prompt=redacted,
                requires_explicit_consent=True,
            )
        return ProcessingDecision(allowed=True)
```

## Data Protection Impact Assessment (DPIA) (Article 35)

A DPIA is **mandatory** when processing is likely to result in a high risk to individuals, including:
- Systematic and extensive profiling
- Large-scale processing of special category data
- Systematic monitoring of publicly accessible areas

**GenAI systems in banking will ALWAYS require a DPIA** because they involve:
- Large-scale processing of financial data
- Automated decision-making with significant effects
- Profiling and behavioral analysis

**DPIA must document**:
1. Systematic description of processing operations and purposes
2. Necessity and proportionality assessment
3. Risk assessment for individuals' rights and freedoms
4. Mitigating measures (safeg, security measures, mechanisms)

**Engineering contribution**: You must provide technical details for the DPIA:
- Data flows (source, destination, format, volume)
- Security controls (encryption, access controls)
- Data retention periods
- Third-party data sharing
- Risk mitigation measures

## International Data Transfers (Chapter V)

Personal data can only be transferred outside the EU/EEA if:
- The destination country has an adequacy decision
- Appropriate safeguards are in place (SCCs, BCRs)
- A specific derogation applies

**GenAI implications**:
- If you use OpenAI's API (US-based), customer prompts are being transferred to the US
- This requires Standard Contractual Clauses (SCCs) or an adequacy decision
- The EU-US Data Privacy Framework (2023) provides an adequacy basis for certified companies

**Engineering requirements**:
```
MANDATORY CONTROLS:
├── Data transfer impact assessment (TIA)
├── SCCs in place for all cross-border transfers
├── Encryption before transfer (supplementary measures)
├── Document all transfer mechanisms
├── Monitor adequacy decision changes
└── Implement data localization where required
```

## Breach Notification (Articles 33-34)

**Notify the supervisory authority** within **72 hours** of becoming aware of a personal data breach.

**Notify affected individuals** without undue delay if the breach is likely to result in a high risk to their rights.

**Engineering requirements**:
```
MANDATORY CONTROLS:
├── Automated breach detection and alerting
├── Breach severity classification workflow
├── 72-hour notification SLA tracking
├── Individual notification workflow
├── Forensic data collection capabilities
├── Breach response runbook
└── Post-incident review process
```

## Privacy by Design and by Default (Article 25)

Data protection must be integrated into the design of processing operations **from the outset**:

```
GOOD:    Design the data model with GDPR compliance built in.
BAD:     "We'll add GDPR compliance later."

GOOD:    Default settings minimize data collection.
BAD:     "Users can opt out if they want."
```

**Engineering checklist**:
- [ ] Data minimization: Only collect what you need
- [ ] Purpose limitation: Each data field has a documented purpose
- [ ] Storage limitation: Automatic deletion schedules in place
- [ ] Access controls: Least privilege by default
- [ ] Encryption: PII encrypted at rest and in transit
- [ ] Audit logging: All access and modifications logged
- [ ] DSAR capabilities: Automated data export and deletion
- [ ] DPIA: Completed and approved before development

## Common Violations and Real-World Fines

### Major GDPR Enforcement Actions

| Organization | Fine | Year | Violation |
|-------------|------|------|-----------|
| Meta (Facebook) | €1.2 billion | 2023 | Illegal data transfers to US |
| Amazon | €746 million | 2021 | Inadequate legal basis for advertising |
| WhatsApp | €225 million | 2021 | Transparency failures |
| Google | €90 million | 2022 | Cookie consent violations |
| H&M (Germany) | €35 million | 2020 | Employee surveillance data |
| British Airways | £20 million | 2020 | Data breach (initially £183M) |
| Marriott | £18.4 million | 2020 | Data breach (initially £99M) |
| Italian DPA -- various | €28 million total | 2023 | AI/automated decision violations |

### Common Violation Categories

1. **Inadequate legal basis** -- Processing data without valid lawful basis
2. **Failure to honor DSARs** -- Not responding to access/erasure requests
3. **Inadequate consent mechanisms** -- Pre-ticked boxes, bundled consent
4. **Insufficient transparency** -- Privacy notices too vague or hidden
5. **Inadequate security measures** -- Leading to data breaches
6. **Unauthorized international transfers** -- Sending data to non-adequate countries
7. **Failure to conduct DPIA** -- Not assessing risks before processing

## Required Controls and Implementation

### Control: Data Inventory and Mapping

```
WHAT:    Complete inventory of all personal data processed by GenAI systems
WHY:     Required for Articles 30 (records of processing) and 35 (DPIA)
HOW:
  ├── Automated data discovery tools
  ├── Manual documentation of data flows
  ├── Regular validation of inventory accuracy
  └── Integration with data governance platform

MINIMUM DATA ELEMENTS PER RECORD:
  - Data category (customer, employee, vendor, etc.)
  - Specific data fields (name, email, transaction data, etc.)
  - Storage location (database, vector store, cache, logs)
  - Lawful basis for processing
  - Purpose of processing
  - Retention period
  - Third parties with access
  - International transfer mechanism
  - Encryption status
```

### Control: Consent Management System

```
WHAT:    System to capture, store, and enforce user consent
WHY:     Required when consent is the lawful basis for processing
HOW:
  ├── Granular consent options (not bundled)
  ├── Explicit opt-in (not pre-ticked boxes)
  ├── Timestamp and IP recording
  ├── Consent withdrawal mechanism
  ├── Integration with all downstream systems
  └── Regular consent validity checks

ENGINEERING IMPLEMENTATION:
  - Consent API for frontend integration
  - Consent database with versioning
  └── Webhook/notification system for consent changes
  - Automatic data processing halt on withdrawal
```

### Control: Data Subject Request Portal

```
WHAT:    Self-service or ticketed system for DSARs
WHY:     Required for Articles 15-22
HOW:
  ├── Identity verification workflow
  ├── Automated data collection from all systems
  ├── Data export in machine-readable format
  ├── Deletion workflow with verification
  ├── SLA monitoring (30 days)
  └── Escalation procedures for complex requests
```

## Logging and Audit Requirements

### What to Log

| Event | Required Fields | Retention |
|-------|----------------|-----------|
| Data access | User ID, timestamp, data category, purpose, lawful basis | Minimum 3 years |
| DSAR received | Request type, customer ID, timestamp, SLA deadline | 5 years |
| DSAR fulfilled | Request type, data exported, date, fulfillment method | 5 years |
| Consent captured | Customer ID, consent type, timestamp, version, IP | 5 years |
| Consent withdrawn | Customer ID, consent type, timestamp | 5 years |
| Data deletion | Customer ID, data categories, timestamp, verification | 5 years |
| Data transfer | Source, destination, data categories, transfer mechanism | 5 years |
| Breach detected | Breach type, severity, data affected, detection time | 10 years |
| DPIA completed | System name, assessor, date, risk level, mitigations | 5 years |

### What NOT to Log

- Raw PII in application logs
- Unencrypted personal data in error messages
- Personal data in debug output
- Full prompt text containing PII
- Vector embeddings of personal data (unless access-controlled and encrypted)

## How GDPR Affects Each System Component

### APIs
- Must include consent verification for PII access
- Must support DSAR endpoints
- Must log all access to personal data with purpose
- Must enforce data minimization in responses (no over-fetching)
- Rate limiting on DSAR endpoints to prevent abuse

### Frontend
- Must display privacy notices before collecting data
- Must implement consent collection with granular options
- Must not expose PII in URLs, error messages, or client-side logs
- Must support privacy settings interface
- Must implement secure session management

### Backend
- Must encrypt PII at rest (AES-256 minimum)
- Must implement purpose-based access controls
- Must support DSAR data collection APIs
- Must support deletion cascades
- Must not log PII in application logs
- Must validate and redact PII from AI inputs

### Databases
- Must encrypt all PII columns at rest
- Must support efficient DSAR queries across tables
- Must support cascading deletion
- Must maintain audit trails for all PII modifications
- Must implement column-level access controls
- Must support data anonymization for backups

### CI/CD
- Must scan for hardcoded PII in code and configs
- Must require DPIA sign-off before deployment of systems processing PII
- Must enforce encryption standards in infrastructure definitions
- Must validate logging configurations do not capture PII

### Monitoring
- Must detect unauthorized access to personal data
- Must alert on DSAR SLA breaches
- Must track consent status changes
- Must monitor data transfer destinations
- Must detect anomalous data access patterns

### AI Systems
- Must classify AI processing under Article 22 (automated decision-making)
- Must implement human-in-the-loop for significant decisions
- Must provide meaningful explanations of AI logic
- Must track training data provenance and consent
- Must support deletion of personal data from training sets (or document why not possible)

### Prompt Logging
- Must redact PII before logging prompts
- Must apply same retention as other personal data
- Must be searchable for DSAR fulfillment
- Must support deletion of individual customer's prompts
- Must not expose prompt logs to unauthorized personnel

### Vector Databases
- Must hash customer identifiers before embedding
- Must implement collection-level access controls
- Must support deletion of customer-specific embeddings
- Must document data lineage (source documents -> embeddings)
- Must encrypt embeddings at rest

### Model Training
- Must verify lawful basis for all training data
- Must document data sources and processing purposes
- Must test for bias against protected characteristics
- Must support removal of specific individual's data from training set
- Must maintain model cards documenting training data composition

## Common Interview Questions

### Question 1: "How would you design a GDPR-compliant customer service chatbot?"

**What they're testing**: Your understanding of data minimization, consent, and data subject rights in an AI context.

**Good answer structure**:
1. Data collection: Only collect data needed for the conversation, with clear privacy notice
2. Consent: Obtain explicit consent for chat logging and AI processing
3. PII handling: Redact PII from prompts before sending to AI model
4. Data retention: Auto-delete chat logs after defined period (e.g., 90 days)
5. DSAR support: Index chats by customer ID for easy export
6. Article 22: Human escalation for any decision affecting the customer
7. International transfers: Use EU-hosted AI model or implement SCCs
8. Security: Encrypt all data at rest and in transit

### Question 2: "A customer requests deletion of their data. Your AI model was fine-tuned on their data. What do you do?"

**What they're testing**: Understanding of the limitations of deletion in ML systems and how to handle them.

**Good answer structure**:
1. Delete all directly accessible data (database, logs, vector store)
2. Document that the model was trained on their data
3. Assess the risk: Can the model reproduce their personal data? (memorization risk)
4. If risk is low: Document the assessment, keep the model
5. If risk is high: Consider model retraining or fine-tuning a correction model
6. Communicate with the customer about limitations
7. Implement prevention: Use differential privacy or data hashing for future training

### Question 3: "How do you handle DSARs for data stored in vector embeddings?"

**What they're testing**: Technical understanding of vector databases and GDPR compliance.

**Good answer structure**:
1. Preventive approach: Hash customer IDs before embedding (so you can find all their data)
2. If already embedded: Use metadata tagging to identify customer-specific vectors
3. For deletion: Delete the metadata entries and source documents
4. For the embeddings themselves: Either delete the collection and rebuild, or use approximate deletion techniques
5. Document the approach and any residual risk
6. Future-proof: Design embedding pipeline with deletion in mind

### Question 4: "What is Article 22 and how does it affect ML systems?"

**What they're testing**: Knowledge of automated decision-making rules.

**Good answer structure**:
1. Article 22 gives individuals the right not to be subject to solely automated decisions with legal/significant effects
2. This applies to credit decisions, fraud flagging, employment decisions
3. Engineering response: Implement human-in-the-loop for all significant decisions
4. Provide meaningful information about the logic involved (explainability)
5. Allow individuals to contest decisions and express their point of view
6. Document the decision-making process and human review

### Question 5: "How would you implement a data inventory for GDPR compliance?"

**What they're testing**: Systematic approach to data governance.

**Good answer structure**:
1. Automated discovery: Use tools to scan databases, file stores, APIs
2. Manual documentation: Interview teams to find shadow data stores
3. Classify data: PII categories, sensitivity levels, lawful basis
4. Map data flows: Source -> processing -> storage -> deletion
5. Integrate with data governance platform for ongoing management
6. Validate regularly: Automated scans to find new data stores
7. Document everything: Required for Article 30 records of processing

## Compliance Checklist

### Pre-Development
- [ ] DPIA completed and approved
- [ ] Lawful basis documented for each data element
- [ ] Data minimization review (are we collecting more than needed?)
- [ ] International transfer assessment completed
- [ ] Special category data assessment completed
- [ ] Privacy notice drafted and approved by DPO

### Development
- [ ] PII encrypted at rest (AES-256 minimum)
- [ ] PII encrypted in transit (TLS 1.2+)
- [ ] Access controls implemented (RBAC/ABAC)
- [ ] Audit logging for all PII access
- [ ] PII redaction from application logs
- [ ] Consent management system integrated
- [ ] DSAR data collection APIs implemented
- [ ] Deletion cascades implemented
- [ ] Prompt logging redacts PII
- [ ] Vector embeddings use hashed identifiers

### Pre-Deployment
- [ ] Penetration testing completed
- [ ] DSAR end-to-end test passed
- [ ] Deletion end-to-end test passed
- [ ] Consent withdrawal test passed
- [ ] DPO sign-off obtained
- [ ] Privacy notice published
- [ ] Incident response runbook updated
- [ ] Team trained on GDPR requirements

### Post-Deployment
- [ ] DSAR SLA monitoring active
- [ ] Unauthorized access alerting active
- [ ] Data retention policy enforcement active
- [ ] Regular audit of data inventory
- [ ] Annual DPIA review scheduled
- [ ] Breach response drill completed
- [ ] Vendor compliance reviews scheduled
