# AI Governance Framework

## What Is AI Governance and Why It Matters

AI governance is the system of policies, processes, controls, and accountabilities that ensures AI systems are developed, deployed, and operated responsibly. For a global bank, AI governance is not optional -- it is required by:

- **SR 11-7** (Federal Reserve/OCC) -- Model risk management
- **FCA expectations** (UK) -- Consumer Duty, SM&CR
- **EU AI Act** (EU) -- Risk-based AI regulation
- **NIST AI RMF** (US) -- Voluntary framework, widely adopted
- **ISO/IEC 42001** -- AI management system standard
- **Internal policies** -- Bank's own AI usage policy

Without a governance framework, AI systems can:
- Produce discriminatory outcomes
- Make decisions the bank cannot explain
- Violate consumer protection laws
- Expose the bank to regulatory enforcement
- Cause reputational damage
- Create systemic risks

## EU AI Act

The EU AI Act, effective August 2024, is the world's first comprehensive AI regulation. It takes a risk-based approach with phased enforcement starting 2025-2026.

### Risk Classification

| Risk Level | Description | Examples | Requirements |
|-----------|-------------|----------|-------------|
| **Unacceptable** | Prohibited AI systems | Social scoring, real-time biometric surveillance in public | BANNED |
| **High** | Significant risk to health, safety, or rights | Credit scoring, employment decisions, law enforcement | Full compliance required |
| **Limited** | Specific transparency risks | Chatbots, deepfakes | Transparency obligations |
| **Minimal** | Low risk | Spam filters, video games | No specific requirements |

### Banking AI Systems -- Likely Classifications

| AI System | Classification | Rationale |
|-----------|---------------|-----------|
| Credit scoring using AI | High | Determines access to financial services |
| Fraud detection using AI | High | Affects consumer account access |
| AML screening using AI | High | Regulatory compliance, account restrictions |
| AI chatbot for customer service | Limited | Transparency obligation (must disclose AI interaction) |
| AI document summarization (internal) | Minimal | Internal use, limited risk |
| AI trading algorithm | High (likely) | Significant financial impact |
| AI hiring tool | High | Employment decisions |

### High-Risk AI Requirements

If your AI system is classified as high-risk:

```
HIGH-RISK AI REQUIREMENTS:
├── Risk management system:
│   ├── Identify and analyze risks throughout lifecycle
│   ├── Implement risk mitigation measures
│   └── Document and review risk management
├── Data governance:
│   ├── Training data must be relevant, representative, and of high quality
│   ├── Data must be examined for biases
│   └── Data processing must respect data protection rules
├── Technical documentation:
│   ├── Comprehensive documentation of the AI system
│   └── Must enable conformity assessment
├── Record-keeping:
│   ├── Automatically log events relevant for traceability
│   └── Retain logs for 6 months minimum (longer for banking)
├── Transparency:
│   ├── Provide clear instructions for use
│   └── Enable users to interpret and use output appropriately
├── Human oversight:
│   ├── Enable humans to understand and intervene
│   └── Prevent or minimize risks
├── Accuracy, robustness, cybersecurity:
│   ├── Achieve appropriate levels of accuracy
│   ├── Be robust against errors and inconsistencies
│   └── Be resilient against attacks and misuse
├── Conformity assessment:
│   ├── Internal conformity assessment (self-assessment)
│   └── For some systems, third-party conformity assessment
├── Registration:
│   └── Register high-risk AI systems in EU database
└── Post-market monitoring:
    ├── Monitor AI system performance in production
    ├── Report serious incidents to authorities
    └── Take corrective action when needed
```

### Technical Documentation for High-Risk AI

```
TECHNICAL DOCUMENTATION:
├── General information:
│   ├── Name and details of provider
│   ├── Intended purpose
│   └── Version number
├── Intended purpose:
│   ├── Description of intended use
│   ├── Target users
│   └── Application areas
├── System description:
│   ├── Architecture and components
│   ├── Methodology
│   ├── Training methodology
│   └── System limitations
├── Data:
│   ├── Training data description
│   ├── Data collection methodology
│   ├── Data preparation processes
│   ├── Data quality assessment
│   └── Bias assessment
├── Model:
│   ├── Model architecture
│   ├── Training process
│   ├── Hyperparameters
│   ├── Performance metrics
│   └── Validation results
├── Human oversight:
│   ├── Measures enabling human oversight
│   └── Instructions for human oversight
├── Performance:
│   ├── Accuracy metrics
│   ├── Robustness testing
│   ├── Cybersecurity measures
│   └── Known limitations
├── Post-market monitoring:
│   ├── Monitoring methodology
│   └── Incident reporting procedures
└── Conformity:
    ├── Standards applied
    └── Conformity assessment results
```

### Penalties

| Violation | Fine |
|-----------|------|
| Prohibited AI systems | Up to €35M or 7% of global revenue |
| High-risk AI non-compliance | Up to €15M or 3% of global revenue |
| Providing incorrect information | Up to €7.5M or 1.5% of global revenue |

## NIST AI Risk Management Framework (AI RMF)

The NIST AI RMF 1.0 (January 2023) is a voluntary framework for managing AI risks. It is widely adopted by US organizations and referenced by regulators.

### The AI RMF Core: Four Functions

```
                    NIST AI RMF
                    ════════════

    ┌──────────────────────────────────────────────────┐
    │                 GOVERN                           │
    │  Culture, policies, processes, and accountabilities │
    │  that enable and inform AI risk management          │
    └──────────────────────┬───────────────────────────┘
                           │
    ┌──────────────────────┴───────────────────────────┐
    │                 MAP                              │
    │  Context, capabilities, and impacts of AI systems  │
    │  to categorize risks                               │
    └──────────────────────┬───────────────────────────┘
                           │
    ┌──────────────────────┴───────────────────────────┐
    │                 MEASURE                          │
    │  Analyze, assess, and track AI risks              │
    │  and impacts                                       │
    └──────────────────────┬───────────────────────────┘
                           │
    ┌──────────────────────┴───────────────────────────┐
    │                 MANAGE                           │
    │  Allocate resources and respond to risks          │
    │  identified through the other functions            │
    └──────────────────────────────────────────────────┘
```

### GOVERN Function

```
GOVERN.1: AI RMF is integrated into organizational risk management
GOVERN.2: Policies, processes, and procedures for AI risk are in place
GOVERN.3: Organizational teams are diverse and empowered
GOVERN.4: Ongoing monitoring of AI systems is implemented
GOVERN.5: AI risk management processes are documented
GOVERN.6: Incident response and recovery processes are in place
GOVERN.7: Third-party AI risks are managed
```

**Engineering implementation**:
```python
class AIGovernance:
    """Implement GOVERN function of NIST AI RMF."""

    def govern_ai_system(self, system: AISystem) -> GovernanceReport:
        return GovernanceReport(
            system_id=system.id,
            policies=self.verify_policies_in_place(system),
            processes=self.verify_processes(system),
            team_diversity=self.assess_team_diversity(system),
            monitoring=self.verify_monitoring(system),
            documentation=self.verify_documentation(system),
            incident_response=self.verify_incident_response(system),
            third_party=self.verify_third_party_management(system),
            recommendations=self.generate_recommendations(system),
        )
```

### MAP Function

```
MAP.1: AI system context, capabilities, and limitations are understood
MAP.2: Intended purposes and contexts are defined
MAP.3: AI system impacts (positive and negative) are identified
MAP.4: Risks are categorized and prioritized
```

**Engineering implementation**:
```python
class AIMapping:
    """Implement MAP function of NIST AI RMF."""

    def map_ai_system(self, system: AISystem) -> MappingReport:
        return MappingReport(
            system_id=system.id,
            context=self.describe_context(system),
            capabilities=self.describe_capabilities(system),
            limitations=self.describe_limitations(system),
            intended_purpose=system.intended_purpose,
            intended_users=system.intended_users,
            impacts=self.identify_impacts(system),
            risks=self.categorize_risks(system),
            risk_priority=self.prioritize_risks(system),
        )

    def identify_impacts(self, system: AISystem) -> ImpactAnalysis:
        """Identify positive and negative impacts of the AI system."""
        return ImpactAnalysis(
            positive_impacts=[
                Impact(type="efficiency", description="Reduced processing time"),
                Impact(type="accuracy", description="Improved fraud detection rate"),
            ],
            negative_impacts=[
                Impact(type="bias", description="Potential disparate impact on group X"),
                Impact(type="transparency", description="Limited explainability"),
                Impact(type="privacy", description="Training data includes personal information"),
            ],
            affected_stakeholders=[
                "consumers", "employees", "regulators", "shareholders",
            ],
            severity_assessment=self.assess_severity(system),
        )
```

### MEASURE Function

```
MEASURE.1: AI system properties and behaviors are analyzed
MEASURE.2: AI system risks are assessed
MEASURE.3: AI system performance is tracked
MEASURE.4: Unexpected or undesirable behaviors are identified
MEASURE.5: Testing and evaluation results are used to improve AI risk management
```

**Engineering implementation**: See `model-risk-management.md` for detailed monitoring implementation.

### MANAGE Function

```
MANAGE.1: AI risks are prioritized and allocated
MANAGE.2: Risk responses are planned and implemented
MANAGE.3: AI system incidents are tracked and managed
MANAGE.4: Risk responses are monitored and improved
```

## AI Governance Framework for Banking

A practical AI governance framework for a global bank should include:

### 1. AI Governance Board/Committee

```
AI GOVERNANCE COMMITTEE:
├── Composition:
│   ├── Chief Risk Officer (chair)
│   ├── Chief Technology Officer
│   ├── Chief Data Officer
│   ├── Chief Compliance Officer
│   ├── Data Protection Officer
│   ├── Head of Model Risk Management
│   └── Business line representatives
├── Responsibilities:
│   ├── Approve AI policy
│   ├── Classify AI systems by risk level
│   ├── Review high-risk AI deployments
│   ├── Review AI incident reports
│   ├── Approve AI vendor relationships
│   └── Report to Board on AI risks
├── Meeting frequency: Monthly
└── Escalation: Board Risk Committee
```

### 2. AI Policy

```
BANK AI POLICY -- KEY ELEMENTS:
├── Scope: All AI/ML systems used by the bank
├── Risk classification framework
├── Development standards
├── Validation requirements
├── Deployment approval process
├── Monitoring and maintenance requirements
├── Incident response procedures
├── Vendor management requirements
├── Training and awareness requirements
├── Compliance and enforcement
└── Review cycle (annual minimum)
```

### 3. AI System Registration

```
AI SYSTEM REGISTRY:
├── All AI systems must be registered
├── Registration includes:
│   ├── System name and description
│   ├── Intended purpose
│   ├── Risk classification
│   ├── System owner
│   ├── Development team
│   ├── Validation status
│   ├── Deployment status
│   ├── Data sources
│   ├── Dependencies
│   └── Review schedule
├── Updated with any material changes
├── Reviewed at least annually
└── Reported to AI Governance Committee quarterly
```

### 4. AI Risk Classification

```
RISK CLASSIFICATION MATRIX:
┌─────────────────────────┬─────────────┬─────────────┬──────────────┐
│ Factor                  │ Low Risk    │ Med Risk    │ High Risk    │
├─────────────────────────┼─────────────┼─────────────┼──────────────┤
│ Impact on consumers     │ Minimal     │ Moderate    │ Significant  │
│ Impact on bank finances │ Minimal     │ Moderate    │ Significant  │
│ Regulatory scrutiny     │ None        │ Some        │ Direct       │
│ Explainability required │ Low         │ Moderate    │ High         │
│ Human oversight         │ Not required│ Recommended │ Required     │
│ Data sensitivity        │ Public      │ Internal    │ Confidential │
│ Reversibility           │ Easy        │ Moderate    │ Difficult    │
└─────────────────────────┴─────────────┴─────────────┴──────────────┘

CLASSIFICATION PROCESS:
├── Self-assessment by system owner
├── Review by Model Risk Management
├── Approval by AI Governance Committee (for High Risk)
├── Documented classification rationale
└── Re-assessment with any material change
```

### 5. AI Development Lifecycle

```
AI DEVELOPMENT LIFECYCLE:
├── Phase 1: Initiation
│   ├── Define purpose and scope
│   ├── Risk classification
│   ├── Feasibility assessment
│   └── Governance approval
├── Phase 2: Development
│   ├── Data sourcing and preparation
│   ├── Model development
│   ├── Testing (unit, integration, security)
│   ├── Documentation
│   └── Code review
├── Phase 3: Validation
│   ├── Independent model validation
│   ├── Bias and fairness testing
│   ├── Security assessment
│   ├── Performance testing
│   └── Validation report
├── Phase 4: Deployment
│   ├── Deployment approval
│   ├── Production configuration
│   ├── Monitoring setup
│   ├── Runbook creation
│   └── Go-live approval
├── Phase 5: Operation
│   ├── Continuous monitoring
│   ├── Periodic re-validation
│   ├── Incident management
│   ├── Performance reporting
│   └── User feedback collection
└── Phase 6: Retirement
    ├── Retirement decision
    ├── Data disposition
    ├── Knowledge retention
    ├── System decommissioning
    └── Post-retirement review
```

### 6. AI Incident Management

```
AI INCIDENT CLASSIFICATION:
├── Severity 1 (Critical):
│   ├── AI produces materially incorrect advice causing financial loss
│   ├── AI system breached exposing customer data
│   ├── AI system used for unauthorized purpose
│   └── Response: Immediate escalation, regulatory notification
├── Severity 2 (High):
│   ├── AI bias detected affecting a consumer group
│   ├── AI performance degradation beyond tolerance
│   ├── AI model drift requiring immediate attention
│   └── Response: Escalation within 4 hours, remediation within 24 hours
├── Severity 3 (Medium):
│   ├── AI output quality issue (no consumer harm)
│   ├── AI monitoring alert requiring investigation
│   └── Response: Investigation within 24 hours, remediation within 5 days
└── Severity 4 (Low):
    ├── Minor AI behavior issue
    ├── Documentation gap
    └── Response: Logged and addressed in regular review cycle
```

## How AI Governance Affects Each System Component

### APIs
- Must include risk classification metadata
- Must enforce governance-approved access controls
- Must log all usage for monitoring
- Must support incident detection

### Frontend
- Must disclose AI interaction to consumers (EU AI Act requirement)
- Must provide human escalation option
- Must be accessible and not misleading

### Backend
- Must implement governance-approved model lifecycle
- Must enforce risk-based controls
- Must support audit requirements

### AI Systems
- Must be registered in AI inventory
- Must have documented risk classification
- Must have completed validation before production
- Must have monitoring in place
- Must have incident response procedures

### Monitoring
- Must track AI system performance
- Must detect incidents
- Must feed governance reporting
- Must support regulatory requirements

## Common Interview Questions

### Question 1: "How would you classify a GenAI system that summarizes regulatory documents for internal use?"

**Good answer**:
Likely minimal or low risk. It's for internal use, doesn't make decisions affecting consumers, and errors can be caught by human reviewers. However, I would still register it in the AI inventory, document its limitations, implement basic monitoring, and train users not to rely on it blindly. If the summaries influence regulatory reporting or compliance decisions, the risk classification increases and additional controls are needed.

### Question 2: "What's the difference between the EU AI Act and the NIST AI RMF?"

**Good answer**:
The EU AI Act is a regulation with legal force -- non-compliance results in fines. It classifies AI systems by risk level and imposes mandatory requirements. The NIST AI RMF is a voluntary framework with no legal force, but it is widely adopted as best practice. The EU AI Act is more prescriptive ("you must do X"), while the NIST AI RMF is more flexible ("consider doing X, Y, Z"). Both emphasize risk management, transparency, and human oversight. In practice, a global bank should comply with both.

### Question 3: "How do you govern an AI system built by a vendor (e.g., OpenAI)?"

**Good answer**:
You cannot govern the vendor's internal development, but you can govern how the bank uses it: (1) Classify the risk of the use case, not the model itself. (2) Conduct vendor due diligence (security, compliance, transparency). (3) Implement input/output controls (validation, filtering, monitoring). (4) Document the system's limitations and train users. (5) Establish contractual protections (data handling, transparency, audit rights). (6) Maintain contingency plans if the vendor fails or changes the model. (7) Monitor the vendor's public announcements for model changes that affect your risk profile.
