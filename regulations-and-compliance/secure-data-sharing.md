# Secure Data Sharing

## What Secure Data Sharing Is and Why It Matters

Secure data sharing is the controlled exchange of data between teams, systems, organizations, or jurisdictions. In banking, data sharing is essential for:
- Cross-team collaboration (risk, compliance, operations, business lines)
- Third-party services (AI vendors, payment processors, auditors)
- Regulatory reporting (submitting data to regulators)
- Business partnerships (correspondent banking, joint ventures)
- Mergers and acquisitions

**Key principle**: Share the minimum data necessary, with appropriate controls, for a defined purpose, for a defined period.

**Regulatory requirements**:
- **GDPR** Chapter V: International data transfers
- **GDPR** Article 28: Data processor agreements
- **PCI-DSS** Requirement 3/4: Protection of stored/transmitted cardholder data
- **ISO 27001** A.5.14: Information transfer
- **BCBS 239**: Data sharing for risk aggregation
- **Banking secrecy laws**: Confidentiality obligations

## Data Sharing Scenarios

### Internal Data Sharing (Cross-Team)

```
INTERNAL DATA SHARING SCENARIOS:
├── Business line to Risk Management:
│   ├── Data: Customer exposures, transactions, positions
│   ├── Purpose: Risk aggregation and reporting
│   ├── Controls: Role-based access, audit logging
│   └── Classification: Confidential
├── Operations to Compliance:
│   ├── Data: Transaction records, customer interactions
│   ├── Purpose: Regulatory compliance monitoring
│   ├── Controls: Need-to-know access, retention per regulation
│   └── Classification: Confidential
├── IT to AI Engineering:
│   ├── Data: Anonymized customer data, system logs
│   ├── Purpose: Model development and testing
│   ├── Controls: Anonymized only, no PII, no production access
│   └── Classification: Internal (anonymized)
├── Customer Service to Fraud Team:
│   ├── Data: Customer account details, flagged transactions
│   ├── Purpose: Fraud investigation
│   ├── Controls: Case-based access, time-limited, audited
│   └── Classification: Confidential
└── All teams to Internal Audit:
    ├── Data: All data relevant to audit scope
    ├── Purpose: Independent audit
    ├── Controls: Audit-specific access, confidentiality agreement
    └── Classification: Restricted (audit)
```

### External Data Sharing (Third-Party)

```
EXTERNAL DATA SHARING SCENARIOS:
├── Bank to AI vendor (OpenAI, Anthropic, etc.):
│   ├── Data: Prompts (PII-redacted), metadata
│   ├── Purpose: AI inference
│   ├── Controls: Zero-retention agreement, encrypted transit,
│   │              PII detection and redaction, access logging
│   └── Classification: Confidential (minimum)
├── Bank to regulator:
│   ├── Data: Regulatory reports, supporting data
│   ├── Purpose: Regulatory compliance
│   ├── Controls: Secure submission channel, encryption,
│   │              submission tracking, receipt confirmation
│   └── Classification: Restricted
├── Bank to auditor:
│   ├── Data: Audit scope data, system access, records
│   ├── Purpose: Independent audit
│   ├── Controls: Time-limited access, confidentiality agreement,
│   │              escorted access (if on-site), activity logging
│   └── Classification: Restricted
├── Bank to correspondent bank:
│   ├── Data: Payment instructions, customer information
│   ├── Purpose: Payment processing
│   ├── Controls: Secure messaging (SWIFT), encryption,
│   │              minimum necessary data, retention agreement
│   └── Classification: Confidential
└── Bank to cloud provider:
    ├── Data: Application data, backups, logs
    ├── Purpose: Infrastructure hosting
    ├── Controls: Encryption (customer-managed keys),
    │              access controls, data processing agreement,
    │              data location requirements
    └── Classification: Confidential
```

## Data Sharing Controls

### Pre-Sharing Assessment

```
DATA SHARING ASSESSMENT:
├── Purpose assessment:
│   ├── Is there a legitimate business purpose?
│   ├── Is the data sharing necessary for that purpose?
│   └── Is there a less invasive alternative?
├── Data assessment:
│   ├── What data needs to be shared?
│   ├── What is the sensitivity classification?
│   ├── Does it contain PII?
│   ├── Does it contain cardholder data?
│   └── Does it contain special category data?
├── Recipient assessment:
│   ├── Who is the recipient?
│   ├── What is their security posture?
│   ├── Do they have a DPA in place (if personal data)?
│   ├── Are they in an adequate jurisdiction (if international)?
│   └── Do they have appropriate access controls?
├── Legal assessment:
│   ├── Is there a lawful basis for sharing?
│   ├── Is consent required (and obtained)?
│   ├── Is there a data processing agreement?
│   ├── Are SCCs required (for international transfers)?
│   └── Are there any regulatory notification requirements?
└── Technical assessment:
    ├── How will data be transferred securely?
    ├── How will data be protected at the destination?
    ├── How will access be controlled at the destination?
    ├── How will data be deleted when no longer needed?
    └── How will compliance be monitored?
```

### Data Sharing Agreement

```
DATA SHARING AGREEMENT REQUIREMENTS:
├── Parties: Clear identification of data provider and recipient
├── Purpose: Specific purpose for data sharing
├── Data description: What data is being shared (specific categories)
├── Permitted uses: What the recipient can do with the data
├── Prohibited uses: What the recipient cannot do with the data
├── Security requirements:
│   ├── Encryption requirements (in transit and at rest)
│   ├── Access control requirements
│   ├── Logging and monitoring requirements
│   └── Incident response requirements
├── Retention and deletion:
│   ├── How long the recipient can retain the data
│   └── How and when the data must be deleted
├── Sub-processing:
│   ├── Whether the recipient can share with subprocessors
│   └── Approval process for subprocessors
├── Audit rights:
│   ├── Right to audit recipient's handling of shared data
│   └── Frequency and scope of audits
├── Breach notification:
│   ├── Timeline for notification (e.g., within 24 hours)
│   └── Cooperation requirements
├── Liability:
│   ├── Liability for data misuse or breach
│   └── Indemnification provisions
├── Termination:
│   ├── Conditions for terminating data sharing
│   └── Data return/deletion on termination
└── Governing law: Applicable jurisdiction
```

### Technical Controls

```
TECHNICAL CONTROLS FOR DATA SHARING:
├── Encryption:
│   ├── Data in transit: TLS 1.2+ (TLS 1.3 preferred)
│   ├── Data at rest: AES-256-GCM
│   ├── Customer-managed encryption keys (for cloud)
│   └── Key exchange via secure key management
├── Authentication and authorization:
│   ├── Mutual TLS for system-to-system sharing
│   ├── OAuth 2.0 / OIDC for API-based sharing
│   ├── API keys for service authentication
│   └── Per-user authentication for human access
├── Access controls:
│   ├── Recipient access limited to authorized individuals
│   ├── Need-to-know principle
│   ├── Time-limited access
│   └── Access logging by recipient
├── Data masking and anonymization:
│   ├── PII redacted before sharing
│   ├── Cardholder data tokenized
│   ├── Sensitive fields masked or generalized
│   └── Anonymization for analytics sharing
├── Transfer mechanisms:
│   ├── Secure file transfer (SFTP, managed file transfer)
│   ├── Secure API with authentication and encryption
│   ├── Secure messaging (SWIFT for banking messages)
│   └── Encrypted physical media (rare, for large datasets)
├── Monitoring:
│   ├── Transfer volume monitoring
│   ├── Access pattern monitoring
│   ├── Anomaly detection
│   └── Compliance verification
└── Data loss prevention:
    ├── DLP policies for outbound data
    ├── Content inspection for sensitive data
    ├── Automated blocking of unauthorized transfers
    └── Alerting on policy violations
```

## International Data Transfers

### Transfer Mechanisms

```
LAWFUL TRANSFER MECHANISMS (GDPR):
├── Adequacy decision:
│   ├── European Commission has determined the country provides
│       adequate data protection
│   ├── Current adequacy decisions include: UK, Japan, South Korea,
│       Canada (commercial), Israel, New Zealand, Switzerland,
│       EU-US Data Privacy Framework participants
│   └── No additional safeguards required
├── Standard Contractual Clauses (SCCs):
│   ├── Pre-approved contractual terms for data transfers
│   ├── Updated by European Commission (2021)
│   ├── Requires transfer impact assessment
│   └── Supplementary measures may be required
├── Binding Corporate Rules (BCRs):
│   ├── Internal rules for intra-group transfers
│   ├── Requires approval by competent authority
│   └── Suitable for multinational banks
├── Derogations (specific situations):
│   ├── Explicit consent
│   ├── Contract performance
│   ├── Important reasons of public interest
│   └── Legal claims
└── Approved codes of conduct / certification mechanisms
    ├── Industry-specific codes
    └── Still emerging for AI/cloud
```

### Transfer Impact Assessment (TIA)

```
TIA PROCESS:
├── Step 1: Identify the transfer
│   ├── What data is being transferred?
│   ├── To which country/organization?
│   ├── For what purpose?
├── Step 2: Assess the destination country
│   ├── Does it have an adequacy decision?
│   ├── If not, what is the rule of law regarding data protection?
│   ├── Are there government surveillance laws that could access the data?
│   ├── Are there effective remedies for individuals?
├── Step 3: Assess the recipient
│   ├── What security measures do they have?
│   ├── What is their data protection track record?
│   ├── Are they subject to conflicting legal obligations?
├── Step 4: Identify supplementary measures
│   ├── Technical: Encryption, pseudonymization
│   ├── Contractual: SCCs, additional commitments
│   └── Organizational: Access controls, training
├── Step 5: Document and approve
│   ├── TIA documented and reviewed by DPO
│   ├── Approved by data protection officer
│   └── Reviewed annually or when circumstances change
└── Step 6: Monitor
    ├── Monitor for changes in destination country laws
    ├── Monitor for changes in recipient's practices
    └── Re-assess if circumstances change
```

## Data Sharing with AI Vendors

### Specific Considerations

```
AI VENDOR DATA SHARING CONTROLS:
├── Data minimization:
│   ├── Send only what is needed for the API call
│   ├── Redact PII before sending
│   ├── Tokenize customer identifiers
│   └── Minimize metadata shared
├── Zero-retention:
│   ├── Confirm vendor's zero-retention policy
│   ├── Contractual prohibition on training with customer data
│   ├── Contractual prohibition on sharing with third parties
│   └── Regular verification of compliance
├── Encryption:
│   ├── TLS 1.2+ for all API calls
│   ├── Consider client-side encryption of sensitive fields
│   └── Customer-managed keys where available
├── Monitoring:
│   ├── Log all data sent to vendor (metadata, not content)
│   ├── Monitor for anomalous data volumes
│   ├── Track PII detection and redaction rates
│   └── Monitor vendor's privacy policy for changes
├── Incident response:
│   ├── Vendor must notify of any data breach
│   ├── Bank must be able to request data deletion
│   └── Contractual liability for vendor-caused breaches
└── Exit strategy:
    ├── Data export capability
    ├── Data deletion verification
    └── Transition to alternative vendor
```

## Data Sharing Logging and Audit

```
DATA SHARING LOG:
├── For each data sharing event:
│   ├── Timestamp
│   ├── Data provider (system, user)
│   ├── Data recipient (system, organization, individual)
│   ├── Data categories shared
│   ├── Volume (record count, data size)
│   ├── Purpose
│   ├── Transfer mechanism
│   ├── Legal basis (DPA reference, SCC reference)
│   ├── Approval reference (who approved)
│   └── Retention period at destination
├── Logs retained for audit period (minimum 3 years)
├── Logs reviewed regularly for compliance
├── Anomalies investigated and escalated
└── Logs included in regulatory and audit examinations
```

## Common Interview Questions

### Question 1: "How would you share customer data with an external AI vendor for model fine-tuning?"

**Good answer structure**:
First, I would question whether this is necessary -- can we fine-tune with internal data only? If external sharing is required: (1) Anonymize the data before sharing -- remove all PII. (2) Execute a data processing agreement with the vendor specifying permitted uses, security requirements, retention, and deletion. (3) If personal data must be shared, ensure SCCs are in place for international transfers and complete a transfer impact assessment. (4) Use encrypted transfer mechanisms. (5) Log the sharing event. (6) Verify the vendor deletes the data after fine-tuning is complete. (7) Monitor for compliance with the agreement.

### Question 2: "What controls would you implement for data shared with a correspondent bank?"

**Good answer structure**:
1. Share only the minimum data necessary for the payment transaction (payment amount, beneficiary details, reference -- not full customer profile). 2. Use secure messaging channels (SWIFT). 3. Encrypt the data in transit. 4. Have a data sharing agreement in place. 5. Ensure the correspondent bank has appropriate security controls. 6. Log the data sharing. 7. Monitor the correspondent bank relationship for any changes in risk profile. 8. Have procedures for data return or deletion when the relationship ends.

### Question 3: "How do you ensure data shared with a cloud provider is protected?"

**Good answer structure**:
1. Encryption: Customer-managed encryption keys so the cloud provider cannot access the data. 2. Access controls: Only authorized bank personnel can access the data, not cloud provider staff. 3. Data processing agreement: Contractual obligations for data handling. 4. Data location: Specify where data can be stored (specific regions). 5. Network security: Private endpoints, no public internet access. 6. Monitoring: Continuous monitoring of access patterns and configurations. 7. Audit: Right to audit the cloud provider's handling of our data. 8. Exit: Data export and deletion procedures defined in the contract.
