# Vendor Risk Management

## What Vendor Risk Management Is and Why It Matters

Vendor risk management (also called third-party risk management or TPRM) is the process of assessing, monitoring, and controlling risks introduced by third-party vendors. For a bank using GenAI services, vendor risk management is critical because:

- **You are responsible for your vendors' failures** -- regulators hold the bank accountable
- **AI vendors introduce unique risks** -- model changes, data handling, opacity
- **Concentration risk** -- many banks using the same AI vendors creates systemic risk
- **Data sharing** -- customer data leaves the bank's control

**Regulatory requirements**:
- **OCC 2013-29** (US): Third-party risk management guidance
- **EBA Guidelines** (EU): Outsourcing arrangements
- **FCA** (UK): Third-party risk expectations
- **ISO 27001** A.5.19-A.5.23: Supplier relationships
- **SOC 2** CC9: Vendor risk management
- **GDPR** Article 28: Data processing agreements
- **EU AI Act**: Obligations for deployers of third-party AI systems

## Vendor Risk Management Lifecycle

```
VENDOR RISK MANAGEMENT LIFECYCLE
════════════════════════════════

  ┌───────────┐    ┌───────────┐    ┌───────────┐
  │ IDENTIFY  ├───►│ ASSESS    ├───►│ APPROVE   │
  │ & CLASSIFY│    │ & DUE     │    │ & ONBOARD │
  │           │    │ DILIGENCE │    │           │
  └───────────┘    └───────────┘    └───────────┘
                                         │
  ┌───────────┐    ┌───────────┐         │
  │ TERMINATE │◄───┤ MONITOR   │◄────────┘
  │ & OFFBOARD│    │ & REVIEW  │
  └───────────┘    └───────────┘
```

### Phase 1: Identify and Classify

```
VENDOR CLASSIFICATION:
├── Critical:
│   ├── Vendor failure would materially impact bank operations
│   ├── Vendor handles sensitive customer data
│   ├── Vendor provides core banking services
│   ├── Examples: Core AI model provider, cloud infrastructure
│   └── Requires: Full due diligence, board approval, ongoing monitoring
├── Significant:
│   ├── Vendor failure would significantly impact operations
│   ├── Vendor handles internal (not customer) data
│   ├── Examples: AI development tools, testing platforms
│   └── Requires: Enhanced due diligence, regular monitoring
├── Standard:
│   ├── Vendor failure would have limited impact
│   ├── No sensitive data involved
│   ├── Examples: General productivity tools
│   └── Requires: Standard due diligence, periodic review
└── Low:
    ├── Minimal impact if vendor fails
    ├── Public data only
    ├── Examples: Public API services, open-source tools
    └── Requires: Basic assessment
```

### Phase 2: Assessment and Due Diligence

#### Information Security Assessment

```
SECURITY ASSESSMENT CHECKLIST:
├── Security certifications:
│   ├── SOC 2 Type II report (review findings)
│   ├── ISO 27001 certification
│   ├── PCI-DSS compliance (if handling payment data)
│   └── Industry-specific certifications
├── Security practices:
│   ├── Encryption standards (at rest and in transit)
│   ├── Access controls and authentication
│   ├── Vulnerability management program
│   ├── Incident response capabilities
│   ├── Penetration testing frequency
│   └── Security team size and expertise
├── Data handling:
│   ├── Where is data stored (geographic location)?
│   ├── Is data encrypted?
│   ├── Who has access to data?
│   ├── Is data used for the vendor's own purposes?
│   ├── Can data be returned/deleted on request?
│   └── How is data segregated between customers?
├── Business continuity:
│   ├── Uptime SLA and historical performance
│   ├── Disaster recovery capabilities
│   ├── Backup procedures
│   └── Business continuity plan testing
└── Compliance:
    ├── Regulatory compliance attestations
    ├── Privacy policy and practices
    ├── Data processing agreement willingness
    └── Audit rights
```

#### AI-Specific Due Diligence

```
AI VENDOR DUE DILIGENCE:
├── Model transparency:
│   ├── Model architecture and capabilities documented
│   ├── Training data sources and methods disclosed
│   ├── Model limitations and known issues disclosed
│   ├── Model versioning and change notification process
│   └── Performance benchmarks available
├── Data practices:
│   ├── Does the vendor train on customer data?
│   ├── Does the vendor store customer inputs?
│   ├── Does the vendor share data with third parties?
│   ├── Does the vendor use data to improve their models?
│   └── Can the vendor delete customer data on request?
├── Security:
│   ├── API security measures
│   ├── Input/output filtering capabilities
│   ├── Content moderation and safety features
│   ├── Rate limiting and abuse prevention
│   └── Data breach history and response
├── Reliability:
│   ├── Historical uptime and incident record
│   ├── SLA terms and penalties
│   ├── Support responsiveness
│   └── Escalation procedures
├── Legal:
│   ├── IP ownership of outputs
│   ├── Indemnification for IP infringement
│   ├── Liability limitations
│   └── Regulatory compliance commitments
└── Exit strategy:
    ├── Data export capabilities
    ├── Data deletion guarantees
    ├── Contract termination terms
    └── Transition support
```

### Phase 3: Approval and Onboarding

#### Approval Process

```
APPROVAL REQUIREMENTS BY VENDOR CLASS:
├── Critical vendors:
│   ├── Information security approval
│   ├── Legal and compliance approval
│   ├── Data protection officer review
│   ├── Model risk management review (for AI vendors)
│   ├── Business continuity review
│   └── Board/senior management approval
├── Significant vendors:
│   ├── Information security approval
│   ├── Legal review
│   ├── Data protection review
│   └── Business unit head approval
├── Standard vendors:
│   ├── Security questionnaire completed
│   └── Procurement approval
└── Low vendors:
    └── Basic assessment documented
```

#### Contractual Requirements

```
REQUIRED CONTRACT TERMS FOR AI VENDORS:
├── Data processing agreement (GDPR Article 28 compliant)
├── Data location restrictions (data stays in approved regions)
├── Security requirements (encryption, access controls, monitoring)
├── Audit rights (right to audit vendor's systems)
├── Breach notification (within X hours of discovery)
├── Subprocessor approval (vendor must notify before adding subprocessors)
├── Data deletion (vendor must delete data on request or contract end)
├── Service levels (uptime, response time, resolution time)
├── Model change notification (vendor must notify of model changes)
├── IP terms (who owns outputs, indemnification for claims)
├── Liability (appropriate caps, exclusions)
├── Termination (transition support, data return)
├── Regulatory cooperation (vendor will cooperate with bank's regulators)
└── Insurance (cyber liability, errors and omissions)
```

### Phase 4: Ongoing Monitoring

```
ONGOING MONITORING PROGRAM:
├── Continuous monitoring:
│   ├── Vendor uptime monitoring
│   ├── Security incident alerts
│   ├── Public news and social media monitoring
│   └── Financial health monitoring
├── Periodic reviews:
│   ├── Critical vendors: Quarterly
│   ├── Significant vendors: Semi-annually
│   ├── Standard vendors: Annually
│   └── Low vendors: As needed
├── Annual reassessment:
│   ├── Updated security questionnaire
│   ├── Review of SOC 2/ISO certifications
│   ├── Review of incident history
│   ├── Contract compliance review
│   └── Risk classification re-assessment
├── Trigger events:
│   ├── Vendor security breach
│   ├── Vendor model change
│   ├── Vendor ownership change
│   ├── Regulatory action against vendor
│   ├── Significant SLA breach
│   └── Change in bank's use of vendor
└── Performance metrics:
    ├── SLA compliance rate
    ├── Incident count and severity
    ├── Support ticket resolution time
    ├── Customer satisfaction
    └── Issue resolution rate
```

## AI Vendor-Specific Risks

### Model Change Risk

AI vendors may change their models without notice, affecting your system's behavior.

```
MODEL CHANGE MITIGATION:
├── Contractual requirement for advance notification of model changes
├── Monitoring for behavior changes (automated regression testing)
├── Ability to pin to specific model versions
├── Testing protocol after any model change
├── Fallback to previous version if change causes issues
└── Documentation of model version in use at all times
```

### Training Data Risk

Vendor may use your data to train their models, potentially exposing your data to other customers.

```
TRAINING DATA MITIGATION:
├── Contractual prohibition on training on customer data
├── Zero-retention API options where available
├── Data processing agreement specifying data handling
├── Regular verification of data handling practices
├── Technical controls: Do not send sensitive data to vendor
└── Vendor audit rights to verify data handling
```

### Concentration Risk

If many banks use the same AI vendor, a vendor failure creates systemic risk.

```
CONCENTRATION RISK MITIGATION:
├── Multi-vendor strategy (primary + secondary AI providers)
├── Abstraction layer to switch between vendors
├── Internal fallback model capability
├── Regular testing of alternative vendors
├── Monitor vendor market share and concentration
└── Escalate concentration risk to senior management
```

## Common Interview Questions

### Question 1: "What due diligence would you perform before using OpenAI's API for a customer-facing banking application?"

**Good answer structure**:
1. Security: Review SOC 2 Type II report, encryption practices, access controls, incident history
2. Data handling: Confirm zero-retention or equivalent, no training on our data, data location
3. Legal: DPA signed, SCCs for data transfers, IP terms, liability terms, indemnification
4. AI-specific: Model transparency documentation, version pinning capability, change notification
5. Reliability: SLA terms, uptime history, support responsiveness, disaster recovery
6. Regulatory: Compliance commitments, regulator cooperation willingness
7. Exit: Data deletion guarantees, export capabilities, transition support
8. Internal: Get security, legal, compliance, and model risk management approval

### Question 2: "What would you do if your AI vendor had a data breach?"

**Good answer structure**:
1. Immediate: Assess impact -- was our data affected? What data? How many customers?
2. Contain: Stop sending data to the vendor if the breach is ongoing
3. Notify: Internal incident response, legal, compliance, DPO
4. Regulatory: Assess notification obligations (72-hour GDPR notification if applicable)
5. Customer: Assess whether customer notification is required
6. Vendor: Request breach details, remediation plan, prevention measures
7. Re-assess: Should we continue using this vendor? What additional controls are needed?
8. Document: Complete incident record, lessons learned, process improvements

### Question 3: "How do you manage the risk of an AI vendor changing their model?"

**Good answer structure**:
1. Contractual: Require advance notification of model changes
2. Technical: Pin to specific model versions where possible
3. Monitoring: Automated regression testing against known test cases
4. Testing: Re-validate model outputs after any change
5. Fallback: Maintain ability to switch to alternative model or vendor
6. Documentation: Track model version in use and change history
7. Communication: Notify users of significant model changes
8. Governance: Escalate significant model changes through change management
