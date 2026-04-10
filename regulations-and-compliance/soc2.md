# SOC 2 Type II Compliance

## What Is SOC 2 and Why It Matters

SOC 2 (System and Organization Controls 2) is an auditing framework developed by the AICPA that evaluates how organizations protect customer data. Unlike PCI-DSS or GDPR, SOC 2 is not a regulation -- it is a **voluntary** framework. However, it is effectively mandatory for:

- Selling to enterprise customers (they will demand your SOC 2 report)
- Cloud service providers (AWS, Azure, GCP require SOC 2 of their critical vendors)
- Banking technology vendors (banks require SOC 2 from all critical vendors)
- Any organization handling sensitive customer data

**SOC 2 Type I** evaluates the design of controls at a point in time.
**SOC 2 Type II** evaluates the **operating effectiveness** of controls over a period (minimum 6 months, typically 12 months).

For GenAI systems in banking, SOC 2 Type II is expected by:
- Internal audit teams evaluating AI platform vendors
- External auditors assessing the bank's control environment
- Regulators reviewing third-party risk management

## The Trust Service Criteria (TSC)

SOC 2 audits are based on the Trust Services Criteria, organized into five categories:

| Category | Symbol | Required? | Focus |
|----------|--------|-----------|-------|
| **Security** | Common Criteria | YES (always) | Protection against unauthorized access |
| **Availability** | A | Optional | System availability for operation and use |
| **Processing Integrity** | PI | Optional | System processing is complete, valid, accurate, timely, authorized |
| **Confidentiality** | C | Optional | Information designated as confidential is protected |
| **Privacy** | P | Optional | Personal information is collected, used, retained, disclosed, and disposed of in conformity with commitments |

Most GenAI systems in banking will be audited against **Security, Confidentiality, and Privacy** at minimum.

## Security (Common Criteria) -- CC Series

The Security category consists of 9 common criteria (CC1-CC9) with 62 control points. These are **always** included in a SOC 2 audit.

### CC1: Control Environment

**What auditors examine**: Does the organization demonstrate a commitment to integrity and ethical values? Is there board independence and oversight? Does management establish structures, reporting lines, and appropriate authorities?

**Engineering implications**:
- Document your team's security responsibilities
- Participate in security awareness training
- Report security concerns through proper channels
- Follow the principle of least privilege
- Document and follow secure development lifecycle

### CC2: Communication and Information

**What auditors examine**: Does the organization communicate internally and externally about internal controls? Do individuals understand their responsibilities?

**Engineering implications**:
- Document security procedures and runbooks
- Communicate security incidents promptly
- Maintain up-to-date architecture documentation
- Document API contracts and data flows
- Participate in post-incident reviews

### CC3: Risk Assessment

**What auditors examine**: Does the organization identify, analyze, and manage risks? Does it consider fraud risk? Does it assess changes that could impact internal controls?

**Engineering implications**:
- Participate in threat modeling sessions
- Document risks in architecture decision records
- Assess security impact of all design changes
- Report new vulnerabilities through proper channels
- Maintain risk register for your systems

**GenAI-specific risks to document**:
```
AI SYSTEM RISK REGISTER EXAMPLE:
┌─────────────────────────────────┬──────────┬───────────┬────────────────────┐
│ Risk Description                │ Likeli-  │ Impact    │ Mitigation         │
│                                 │ hood     │           │                    │
├─────────────────────────────────┼──────────┼───────────┼────────────────────┤
│ Prompt injection exposes         │ High     │ Critical  │ Input validation,  │
│ system data or modifies          │          │           │ output filtering,  │
│ behavior                         │          │           │ prompt sandboxing  │
├─────────────────────────────────┼──────────┼───────────┼────────────────────┤
│ Training data contains PII       │ Medium   │ High      │ Data classification│
│ without consent                  │          │           │ review, consent    │
│                                  │          │           │ verification       │
├─────────────────────────────────┼──────────┼───────────┼────────────────────┤
│ Model output contains            │ Medium   │ High      │ Output validation, │
│ sensitive information from       │          │           │ content filtering, │
│ training data memorization       │          │           │ redaction          │
├─────────────────────────────────┼──────────┼───────────┼────────────────────┤
│ AI vendor service outage         │ Medium   │ High      │ Multi-vendor       │
│ disrupts customer service        │          │           │ strategy, offline  │
│                                  │          │           │ fallback           │
├─────────────────────────────────┼──────────┼───────────┼────────────────────┤
│ Model drift causes               │ High     │ Medium    │ Continuous         │
│ inaccurate customer responses    │          │           │ monitoring,        │
│                                  │          │           │ human review       │
└─────────────────────────────────┴──────────┴───────────┴────────────────────┘
```

### CC4: Monitoring Activities

**What auditors examine**: Does the organization select, develop, and deploy controls to monitor internal controls? Does it remediate identified deficiencies?

**Engineering implications**:
- Implement automated monitoring for all production systems
- Set up alerting for security-relevant events
- Monitor for anomalous access patterns
- Track security metrics (MTTD, MTTR, vulnerability SLA compliance)
- Participate in regular access reviews

### CC5: Control Activities

**What auditors examine**: Does the organization select and develop control activities that mitigate risks? Does it deploy controls through policies and procedures?

**Engineering implications**:
- Implement technical controls (not just procedural controls)
- Automate security checks in CI/CD pipeline
- Implement infrastructure-as-code with security scanning
- Enforce code review requirements
- Implement secret scanning in repositories

### CC6: Logical and Physical Access Controls

**What auditors examine**: Does the organization restrict logical and physical access to authorized users? Does it protect information assets from unauthorized access?

**Engineering requirements**:

**Logical access**:
```
REQUIRED CONTROLS:
├── Unique user IDs for all personnel
├── Multi-factor authentication for all production access
├── Role-based access control (RBAC)
├── Principle of least privilege
├── Regular access reviews (quarterly minimum)
├── Automated provisioning/deprovisioning
├── Password complexity requirements
├── Session timeout (15 minutes for privileged access)
├── Privileged access management (PAM) for admin accounts
├── Service account management with credential rotation
└── Emergency access procedures (break-glass accounts)
```

**Physical access** (for on-prem infrastructure):
```
REQUIRED CONTROLS:
├── Badge access to data centers
├── Visitor logs and escorts
├── CCTV monitoring and retention
├── Environmental controls (temperature, humidity, fire suppression)
├── Power redundancy (UPS, generators)
└── Secure media disposal
```

### CC7: System Operations

**What auditors examine**: Does the organization detect, analyze, and respond to security events? Does it respond to identified security incidents?

**Engineering requirements**:
```
REQUIRED CONTROLS:
├── Intrusion detection/prevention
├── Malware detection
├── Unauthorized device/software detection
├── Monitoring for anomalous activity
├── Automated alerting on security events
├── Incident response procedures
├── Incident classification and escalation
├── Post-incident review process
├── Lessons learned documentation
└── Incident response testing (annual minimum)
```

### CC8: Change Management

**What auditors examine**: Does the organization authorize, design, develop, configure, test, approve, and deploy changes to infrastructure, data, and software?

**Engineering requirements**:
```
REQUIRED CONTROLS:
├── Change request documentation
├── Risk assessment for each change
├── Testing before deployment (unit, integration, security)
├── Peer review of all changes
├── Approval before production deployment
├── Rollback procedures documented and tested
├── Change implementation tracking
├── Emergency change procedures (with post-implementation review)
├── Segregation of duties (developers cannot deploy to production unreviewed)
└── Change success monitoring
```

**CI/CD compliance**:
```yaml
# Example CI/CD pipeline with SOC 2 controls
stages:
  - test:
      - unit_tests
      - integration_tests
      - security_scan:
          - sast: true
          - dependency_check: true
          - secret_scan: true
      - compliance_check:
          - infrastructure_validation: true
          - data_classification_check: true

  - review:
      - code_review:
          required_approvals: 2
          required_reviewers: [security_champion, team_lead]
      - compliance_review:
          required_for: [production, staging]

  - approve:
      - change_approval:
          required_for: production
          approvers: [change_advisory_board]

  - deploy:
      - canary_deployment: true
      - automated_rollback: true
      - post_deployment_validation: true
```

### CC9: Risk Mitigation

**What auditors examine**: Does the organization identify, select, and develop risk mitigation activities for risks that could impact achievement of objectives?

**Engineering implications**:
- Maintain risk register
- Implement controls identified in risk assessments
- Track remediation of identified risks
- Participate in regular risk review meetings

## Availability (A Series)

If your SOC 2 scope includes Availability, auditors examine:

### A1.1: Capacity Management
- Monitor system capacity and plan for growth
- Implement auto-scaling where appropriate
- Capacity testing before major releases

### A1.2: Environmental Protection
- Protect against environmental threats (fire, flood, power)
- Document environmental controls

### A1.3: Recovery Procedures
- Document and test disaster recovery procedures
- Meet defined Recovery Time Objectives (RTO)
- Meet defined Recovery Point Objectives (RPO)

**Engineering requirements for AI systems**:
```
RECOVERY REQUIREMENTS:
├── AI model inference service RTO: 1 hour
├── AI model inference service RPO: 15 minutes
├── Vector database RTO: 4 hours
├── Vector database RPO: 1 hour
├── Training pipeline RTO: 24 hours
├── Training pipeline RPO: 24 hours (last checkpoint)
├── DR testing frequency: Semi-annual
└── DR test results documented and reviewed
```

## Processing Integrity (PI Series)

If your SOC 2 scope includes Processing Integrity, auditors examine:

### PI1.1: Processing Completeness
- All transactions are processed completely

### PI1.2: Processing Accuracy
- All transactions are processed accurately

### PI1.3: Processing Timeliness
- All transactions are processed in a timely manner

### PI1.4: Processing Validity
- Only valid transactions are processed

**Engineering requirements for AI systems**:
```
PROCESSING INTEGRITY CONTROLS:
├── Input validation for all AI requests
├── Output validation for all AI responses
├── Duplicate detection and prevention
├── Ordering guarantees where required
├── Processing status tracking
├── Error handling and retry logic
├── Processing completeness verification
└── End-to-end traceability
```

## Confidentiality (C Series)

If your SOC 2 scope includes Confidentiality, auditors examine:

### C1.1: Confidential Information Identification
- Identify information classified as confidential

### C1.2: Confidential Information Disposal
- Dispose of confidential information securely

### C1.3: Confidential Information Access
- Restrict access to confidential information

**Engineering requirements**:
```
CONFIDENTIALITY CONTROLS:
├── Data classification system implemented
├── Confidential data encrypted at rest (AES-256)
├── Confidential data encrypted in transit (TLS 1.2+)
├── Access controls based on classification
├── Secure disposal procedures (crypto-shredding, secure erase)
├── Confidential data handling training
└── Confidential data access logging
```

## Privacy (P Series)

If your SOC 2 scope includes Privacy, auditors examine:

### P1: Notice and Communication
- Provide notice about privacy practices

### P2: Choice and Consent
- Obtain consent where required

### P3: Collection
- Collect only personal information needed

### P4: Retention and Disposal
- Retain and dispose of personal information appropriately

### P5: Access, Request, and Correction
- Provide access to personal information

### P6: Disclosure and Notification
- Disclose personal information only for identified purposes

### P7: Quality
- Maintain accurate personal information

### P8: Monitoring and Enforcement
- Monitor compliance with privacy commitments

The Privacy criteria closely align with GDPR requirements. See `gdpr.md` for engineering implementation details.

## Evidence Requirements

### What Auditors Will Ask For

| Evidence Type | Examples | Frequency |
|--------------|----------|-----------|
| **Policies** | Security policy, access control policy, incident response policy | Annual review |
| **Procedures** | Onboarding/offboarding procedures, change management procedures | Annual review |
| **Logs** | Access logs, change logs, security event logs | Ongoing |
| **Screenshots** | Access control configurations, monitoring dashboards | Quarterly |
| **Reports** | Vulnerability scan reports, penetration test reports, access reviews | Quarterly/Annual |
| **Tickets** | Change requests, incident tickets, access requests | Ongoing |
| **Training records** | Security awareness training completion | Annual |
| **Contracts** | Vendor agreements with security clauses | As needed |

### Evidence Organization

```
SOC 2 EVIDENCE REPOSITORY:
├── CC1-Control-Environment/
│   ├── security-policy-v2.3.pdf
│   ├── code-of-conduct.pdf
│   └── org-chart.pdf
├── CC6-Access-Controls/
│   ├── rbac-matrix.xlsx
│   ├── mfa-configuration.png
│   ├── quarterly-access-review-q3.pdf
│   └── pam-configuration.png
├── CC7-System-Operations/
│   ├── siem-dashboard-q3.png
│   ├── incident-response-runbook.pdf
│   └── ir-dr-test-results.pdf
├── CC8-Change-Management/
│   ├── change-request-samples.pdf
│   ├── ci-cd-pipeline-configuration.yaml
│   └── emergency-change-log.xlsx
└── CC9-Risk-Mitigation/
    ├── risk-register.xlsx
    ├── threat-model-ai-service.pdf
    └── vulnerability-remediation-tracking.xlsx
```

## Real-World SOC 2 Findings

### Common Deficiencies Found in SOC 2 Audits

| Deficiency | Severity | Common Cause |
|-----------|----------|-------------|
| Access not reviewed quarterly | Moderate | No automated process |
| Terminated employees retained access | High | No automated offboarding |
| Production changes deployed without approval | High | CI/CD bypass enabled |
| Vulnerability not remediated within SLA | Moderate | No tracking process |
| Logs not protected from modification | High | Append-only not enforced |
| Backup encryption not tested | Moderate | No regular restore testing |
| DR test not performed annually | Moderate | No scheduled DR testing |
| Vendor security assessments not current | Moderate | No vendor management process |

## How SOC 2 Affects Each System Component

### APIs
- Must implement authentication and authorization
- Must log all access attempts
- Must enforce rate limiting
- Must validate all inputs
- Must support audit logging

### Frontend
- Must implement secure session management
- Must not expose sensitive data in client-side storage
- Must enforce HTTPS
- Must implement CSP headers
- Must validate inputs client and server side

### Backend
- Must enforce access controls on all endpoints
- Must implement audit logging
- Must encrypt sensitive data
- Must implement input validation
- Must follow secure SDLC

### Databases
- Must encrypt at rest
- Must implement access controls
- Must log all administrative access
- Must support backup and recovery
- Must implement change tracking

### CI/CD
- Must enforce code review requirements
- Must implement security scanning
- Must require approvals for production deployment
- Must maintain change records
- Must support rollback

### Monitoring
- Must detect unauthorized access
- Must alert on security events
- Must track compliance metrics
- Must support forensic investigation

### AI Systems
- Must document model development lifecycle
- Must implement model validation
- Must monitor for model drift
- Must document training data provenance
- Must implement access controls on models and data

## Common Interview Questions

### Question 1: "What's the difference between SOC 2 Type I and Type II?"

**Good answer**:
Type I evaluates the design of controls at a specific point in time -- "are the controls designed appropriately?" Type II evaluates the operating effectiveness of controls over a period (minimum 6 months) -- "do the controls actually work consistently?" Type II is more valuable because it proves the organization follows its own controls over time.

### Question 2: "How would you prepare a GenAI platform for SOC 2 audit?"

**Good answer structure**:
1. Define the audit scope (which trust service categories)
2. Document all controls in a control matrix
3. Implement any missing controls (especially logging, access control, change management)
4. Begin collecting evidence immediately
5. Run internal assessments before the audit
6. Ensure all systems in scope have proper documentation
7. Train all team members on what to expect during the audit

### Question 3: "What evidence would you provide for change management controls?"

**Good answer structure**:
1. Sample of change requests from the audit period
2. Evidence of testing before deployment (test results, security scan results)
3. Evidence of peer review (pull request approvals)
4. Evidence of approval before production deployment (CAB approval, manager approval)
5. Evidence of successful deployment and post-deployment validation
6. Emergency change records with post-implementation review
7. CI/CD pipeline configuration showing enforced controls
