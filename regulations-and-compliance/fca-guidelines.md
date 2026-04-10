# FCA Guidelines -- AI/ML in Financial Services

## What the FCA Expects and Why It Matters

The UK Financial Conduct Authority (FCA) regulates financial services firms and financial markets in the UK. While the FCA has not issued a single comprehensive "AI guideline," it has issued multiple statements, guidance documents, and enforcement actions that collectively establish expectations for AI/ML use in financial services.

**Key FCA sources**:
- FCA Discussion Paper DP5/22 (Machine Learning in UK Financial Services, 2022)
- FCA Policy Statement PS23/1 (Consumer Duty)
- FCA guidance on AI and public expectations
- FCA enforcement actions related to technology failures
- Bank of England/FCA joint work on AI

**Maximum penalties**: The FCA can impose unlimited fines on firms and individuals. Recent enforcement totals exceed £1.4 billion (2020-2024).

For GenAI engineers at banks operating in the UK, FCA expectations are non-negotiable. The FCA has been explicit that firms must ensure their AI/ML systems:
1. Do not harm consumers
2. Can be explained to consumers and regulators
3. Are governed and controlled appropriately
4. Are resilient and reliable
5. Comply with all existing regulations

## The Consumer Duty (PS22/9)

The Consumer Duty, effective July 2023, is the FCA's most significant recent rule change. It establishes a higher standard of care for consumer financial services.

### Four Outcomes

| Outcome | Description | AI Relevance |
|---------|-------------|-------------|
| **Products and services** | Products must be fit for purpose and designed for specific consumer groups | AI-designed products must be tested for target consumers |
| **Price and value** | Consumers must receive fair value | AI pricing must not produce unfair outcomes |
| **Consumer understanding** | Communications must be clear, fair, and not misleading | AI-generated communications must meet clarity standards |
| **Consumer support** | Consumers must receive appropriate support | AI chatbots must escalate appropriately |

### AI Implications for Consumer Duty

```
CONSUMER DUTY COMPLIANCE FOR AI SYSTEMS:
├── Products and services:
│   ├── AI models must be tested on target consumer groups
│   ├── Products must work as intended for all consumer segments
│   └── Identify and remediate products that harm specific groups
├── Price and value:
│   ├── AI pricing must be fair and transparent
│   ├── Dynamic pricing must not exploit consumer vulnerabilities
│   └── Value must be demonstrable
├── Consumer understanding:
│   ├── AI-generated communications must be clear and not misleading
│   ├── AI must not use consumer biases against them
│   ├── Information must be presented in accessible formats
│   └── Key information must not be hidden or buried
└── Consumer support:
    ├── AI chatbots must handle complaints appropriately
    ├── Escalation to human agents must be timely
    ├── Consumers must be able to reach humans when needed
    └── AI must not create barriers to consumer support
```

### Consumer Duty -- Good vs. Bad Outcomes

```
GOOD OUTCOMES (what FCA expects):
├── AI identifies vulnerable customers and routes them to specialized support
├── AI communications are tested for comprehension by target consumers
├── AI pricing produces fair outcomes across all consumer segments
├── AI systems are monitored for consumer harm outcomes
└── Consumers can easily access human support when AI cannot help

BAD OUTCOMES (what FCA will act against):
├── AI chatbot gives incorrect financial advice that harms consumers
├── AI pricing charges higher prices to less sophisticated consumers
├── AI communications are confusing, misleading, or exploit behavioral biases
├── AI systems create barriers to accessing customer support
└── Firm cannot explain why AI made a specific decision affecting a consumer
```

## FCA Expectations for AI Governance

Based on DP5/22 and subsequent statements, the FCA expects firms to have:

### 1. Clear Accountability

**Who is responsible for AI decisions?**

```
ACCOUNTABILITY FRAMEWORK:
├── Senior Manager responsible for AI governance (SMF designation)
├── Clear ownership of each AI system
├── Defined roles for development, validation, monitoring, and remediation
├── Escalation procedures for AI-related incidents
└── Board-level reporting on AI risks and performance
```

### 2. Robust Risk Management

**How are AI risks identified and managed?**

```
RISK MANAGEMENT:
├── AI risk register maintained and reviewed regularly
├── Risk assessments before deploying new AI systems
├── Ongoing monitoring of AI system performance
├── Incident response procedures for AI failures
├── Regular stress testing of AI systems
└── Independent challenge of AI risk assessments
```

### 3. Transparency and Explainability

**Can you explain how your AI makes decisions?**

```
TRANSPARENCY REQUIREMENTS:
├── Ability to explain individual AI decisions to consumers
├── Ability to explain AI system behavior to regulators
├── Documentation of AI methodology and limitations
├── Clear communication to consumers when they interact with AI
├── Records of AI decision logic available for review
└── Third-party AI systems: understand and document limitations
```

### 4. Testing and Validation

**Has the AI been adequately tested?**

```
TESTING REQUIREMENTS:
├── Pre-deployment testing on representative data
├── Testing across diverse consumer groups
├── Testing for bias and unfair outcomes
├── Stress testing under adverse conditions
├── Continuous post-deployment monitoring
├── Regular re-validation and re-testing
└── Independent validation (not just by development team)
```

### 5. Data Governance

**Is the data used by AI appropriate and well-managed?**

```
DATA GOVERNANCE:
├── Data provenance documented
├── Data quality monitored
├── Bias in training data assessed
├── Consumer consent for data use verified
├── Data retention and deletion policies applied
├── Third-party data assessed for quality and appropriateness
└── Data lineage from source to AI output
```

### 6. Third-Party Risk Management

**Are AI vendors appropriately managed?**

```
VENDOR MANAGEMENT:
├── Due diligence on AI vendors
├── Contractual requirements for transparency
├── Ongoing monitoring of vendor performance
├── Contingency plans for vendor failure
├── Understanding of vendor's model development practices
├── Access to vendor's model documentation and testing results
└── Right to audit vendor's systems
```

## FCA Enforcement Actions Related to Technology

### Notable Cases

| Firm | Fine | Year | Issue |
|------|------|------|-------|
| Barclays | £72M | 2021 | Failure to manage risks in electronic trading systems |
| Goldman Sachs | £29M | 2022 | Systemic control failures in platform engineering |
| Various firms | £200M+ total | 2020-2024 | Technology-related failures affecting consumers |

### FCA Enforcement Themes

1. **Inadequate governance** -- Firms did not have adequate oversight of technology systems
2. **Poor change management** -- Changes deployed without adequate testing
3. **Insufficient monitoring** -- Firms did not detect issues promptly
4. **Weak controls** -- Controls were not effective in preventing harm
5. **Inadequate resourcing** -- Firms did not invest sufficient resources in technology risk

## SM&CR (Senior Managers and Certification Regime)

The SM&CR holds senior individuals personally accountable for failures in their areas of responsibility.

### Implications for AI

```
SM&CR AND AI:
├── Senior Manager responsible for technology/AI must be designated
├── Statement of Responsibilities must cover AI governance
├── Management Responsibilities Map must include AI systems
├── Certification regime applies to staff with significant AI roles
├── Conduct Rules apply to all staff working on AI systems
└── Personal liability for failures in AI governance
```

### Conduct Rules Relevant to AI Engineers

| Rule | Description | Application to AI |
|------|-------------|-------------------|
| **Rule 1** | Act with integrity | Do not misrepresent AI capabilities |
| **Rule 2** | Act with due skill, care, and diligence | Test AI systems thoroughly |
| **Rule 3** | Be open and cooperative with regulators | Disclose AI issues to FCA |
| **Rule 4** | Pay due regard to customer interests and treat them fairly | Ensure AI does not harm consumers |

## FCA Reporting Expectations

### Incidents

Firms must report significant operational incidents to the FCA:

```
REPORTING THRESHOLDS:
├── Incidents affecting significant numbers of consumers
├── Incidents causing consumer harm
├── Incidents affecting market integrity
├── Cybersecurity incidents
├── System outages affecting critical services
└── Any incident that could damage market confidence
```

### AI-Specific Incidents

```
AI INCIDENTS TO REPORT:
├── AI system produces materially incorrect financial advice
├── AI bias results in unfair treatment of a consumer group
├── AI system failure causes consumer harm
├── AI model drift detected after deployment
├── AI vendor breach affects customer data
├── AI system exploited by adversarial attack
└── AI system used for unauthorized purposes
```

### Timeline

- Initial notification: As soon as possible, within 24 hours for critical incidents
- Detailed report: Within 72 hours
- Final report: When incident is fully resolved

## AI Transparency and Disclosure

The FCA expects firms to be transparent about their use of AI:

```
TRANSPARENCY DISCLOSURES:
├── Inform consumers when they are interacting with AI
├── Explain what the AI can and cannot do
├── Provide option to speak with a human
├── Do not misrepresent AI capabilities
├── Disclose when AI influences financial decisions
├── Maintain records of AI interactions for complaint resolution
└── Publish AI usage policy (public-facing)
```

## The FCA and the EU AI Act

Post-Brexit, the UK is developing its own AI regulatory approach. The FCA has indicated it will:

1. **Align with international standards** where appropriate
2. **Take a proportionate, risk-based approach** (similar to EU AI Act)
3. **Leverage existing regulatory frameworks** (Consumer Duty, SM&CR, PRIN)
4. **Coordinate with other regulators** (Bank of England, ICO, CMA)

**Expectation**: The FCA will classify AI systems by risk level and impose proportionate requirements, similar to the EU AI Act framework.

## How FCA Expectations Affect Each System Component

### APIs
- Must include consumer identification and appropriate routing (vulnerable customers to specialized flows)
- Must log all AI-driven decisions for complaint handling
- Must support consumer data access requests
- Must enforce fair treatment across all consumer segments

### Frontend
- Must clearly indicate when consumers are interacting with AI
- Must provide clear, non-misleading information
- Must offer human escalation option
- Must be accessible to all consumers (WCAG compliance)
- Must not use dark patterns or exploitative design

### Backend
- Must implement fair treatment logic
- Must detect and prevent AI-driven discrimination
- Must support consumer complaint workflows
- Must maintain audit trails for all consumer-affecting decisions

### AI Systems
- Must be tested for consumer harm before deployment
- Must be monitored for unfair outcomes post-deployment
- Must support explanation of individual decisions
- Must handle vulnerable consumers appropriately
- Must escalate complaints and complex cases to humans

### Monitoring
- Must track consumer outcome metrics
- Must detect patterns of consumer harm
- Must alert on significant deviations from expected behavior
- Must support complaint trend analysis
- Must feed into Management Information (MI) for senior management

## Common Interview Questions

### Question 1: "How does the FCA Consumer Duty affect an AI-powered mortgage recommendation system?"

**Good answer structure**:
1. Products must be fit for purpose: Test the AI recommendations on actual target consumers, not just historical data
2. Fair value: Ensure the AI doesn't recommend higher-cost products to less sophisticated consumers
3. Consumer understanding: AI explanations must be clear, not misleading, and tested for comprehension
4. Consumer support: Consumers must be able to reach humans if they disagree with AI recommendations
5. Monitoring: Track actual consumer outcomes, not just model accuracy metrics
6. Documentation: Be able to explain why the AI recommended a specific product to a specific consumer

### Question 2: "What would you do if the FCA asked about a specific AI decision that harmed a consumer?"

**Good answer structure**:
1. Produce the complete audit trail: input data, model version, processing logic, output, and any human review
2. Explain the model methodology and its limitations
3. Show testing results and validation evidence
4. Demonstrate monitoring data showing the model's overall performance
5. If a systemic issue: identify scope, affected consumers, and remediation plan
6. If an isolated issue: explain why it's isolated and what controls prevent recurrence
7. Show governance: who approved the model, who monitors it, who can escalate issues

### Question 3: "How would you design an AI system that meets FCA expectations for transparency?"

**Good answer structure**:
1. Pre-deployment: Document the model methodology, training data, testing results, and limitations
2. At point of interaction: Disclose AI use to consumers, explain capabilities and limitations
3. Post-decision: Provide meaningful explanation of the decision in consumer-friendly language
4. On request: Provide detailed information about how the decision was made
5. Ongoing: Monitor for consumer harm, track complaints, report incidents
6. Governance: Senior manager accountability, regular board reporting, independent validation
