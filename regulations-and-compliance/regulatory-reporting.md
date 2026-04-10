# Regulatory Reporting

## What Regulatory Reporting Is and Why It Matters

Regulatory reporting is the process of submitting required information to regulatory bodies in specified formats and timeframes. For banks, regulatory reports cover:

- **Financial condition**: Capital adequacy, liquidity, leverage
- **Risk exposures**: Market risk, credit risk, operational risk
- **Compliance**: AML, KYC, sanctions, consumer protection
- **Prudential**: Stress test results, internal capital assessments

**AI's role**: GenAI is increasingly used to draft, analyze, and quality-check regulatory reports. This introduces unique risks because regulatory reports must be accurate, complete, and submitted on time. Errors can result in significant fines and enforcement actions.

**Regulatory framework**:
- **BCBS 239**: Risk data aggregation and reporting accuracy (see `bcbs-239.md`)
- **COREP/FINREP** (EU): Capital and financial reporting
- **FR Y-9C** (US): Bank holding company financial reports
- **FCA returns** (UK): Various regulatory submissions
- **Basel III**: Capital adequacy reporting

## Regulatory Report Types

### Capital Adequacy Reports

| Report | Regulator | Frequency | Content |
|--------|-----------|-----------|---------|
| COREP | ECB/EBA | Quarterly | Capital requirements, risk-weighted assets |
| FR Y-9C | Federal Reserve | Quarterly | Consolidated financial statements |
| Pillar 3 | Multiple | Semi-annual | Market disclosure of risk and capital |

### Liquidity Reports

| Report | Regulator | Frequency | Content |
|--------|-----------|-----------|---------|
| Liquidity Coverage Ratio (LCR) | Multiple | Monthly | Short-term liquidity resilience |
| Net Stable Funding Ratio (NSFR) | Multiple | Quarterly | Long-term funding stability |
| Daily liquidity monitoring | Internal | Daily | Intraday liquidity position |

### Risk Reports

| Report | Regulator | Frequency | Content |
|--------|-----------|-----------|---------|
| Credit risk | Multiple | Quarterly | Exposure, provisions, risk-weighted assets |
| Market risk | Multiple | Quarterly | VaR, stress testing, trading book |
| Operational risk | Multiple | Quarterly | Loss events, RCSA results |
| AML/SAR | FIU/FinCEN | As events occur | Suspicious activity reports |

## AI in Regulatory Reporting

### How GenAI Is Used

```
GENAI IN REGULATORY REPORTING:
├── Report drafting:
│   ├── AI drafts narrative sections of reports
│   ├── AI summarizes complex data for executive summaries
│   └── AI translates technical findings into regulatory language
├── Data analysis:
│   ├── AI identifies anomalies and outliers in reporting data
│   ├── AI flags potential data quality issues
│   └── AI suggests explanations for variances
├── Quality assurance:
│   ├── AI checks report consistency with prior periods
│   ├── AI validates calculations against source data
│   └── AI identifies missing or incomplete sections
├── Regulatory research:
│   ├── AI summarizes regulatory requirements
│   ├── AI tracks regulatory changes
│   └── AI maps requirements to internal processes
└── Submission preparation:
    ├── AI formats reports to regulatory specifications
    ├── AI validates submission package completeness
    └── AI generates submission cover letters
```

### Risks of AI in Regulatory Reporting

```
RISKS:
├── Inaccuracy:
│   ├── AI generates incorrect figures or calculations
│   ├── AI misinterprets data or trends
│   └── AI hallucinates regulatory requirements
├── Incompleteness:
│   ├── AI omits required sections or data points
│   ├── AI fails to include required disclosures
│   └── AI does not cover all required risk types
├── Inconsistency:
│   ├── AI produces reports inconsistent with prior periods
│   ├── AI uses different methodologies period over period
│   └── AI contradicts information in other reports
├── Timeliness:
│   ├── AI processing delays report generation
│   └── AI errors require rework, consuming time
├── Audit trail:
│   ├── AI-generated content must be traceable to source data
│   ├── Human review must be documented
│   └── Changes from AI draft to final must be tracked
└── Regulatory perception:
    ├── Regulators may not accept AI-generated reports without human sign-off
    ├── Regulators may require disclosure of AI use
    └── Regulators may have specific AI governance expectations
```

### Controls for AI-Generated Regulatory Reports

```
CONTROLS:
├── Human review mandatory:
│   ├── Every AI-generated section reviewed by qualified analyst
│   ├── Reviewer verifies figures against source data
│   ├── Reviewer validates narrative accuracy
│   └── Reviewer signs off on content
├── Source data traceability:
│   ├── Every number in report traceable to source system
│   ├── Reconciliation between report totals and source totals
│   └── Variance analysis for period-over-period changes
├── Version control:
│   ├── AI draft versions preserved
│   ├── Human edits tracked and documented
│   └── Final version clearly identified
├── Quality checks:
│   ├── Automated validation (completeness, arithmetic, consistency)
│   ├── Independent review (four-eyes principle)
│   └── Management sign-off before submission
├── AI monitoring:
│   ├── Track AI accuracy in regulatory reporting context
│   ├── Monitor for AI errors or hallucinations
│   └── Regular validation of AI output quality
└── Documentation:
    ├── AI use in regulatory reporting documented
    ├── AI limitations acknowledged
    ├── Compensating controls documented
    └── Process available for audit review
```

## Submission Process

```
REGULATORY SUBMISSION WORKFLOW:
┌─────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
│ DATA    ├───►│ REPORT    ├───►│ REVIEW    ├───►│ APPROVAL  │
│ COLLECTION│   │ GENERATION│    │ & QA      │    │           │
└─────────┘    └───────────┘    └───────────┘    └───────────┘
                                                        │
┌─────────┐    ┌───────────┐    ┌───────────┐          │
│ RECEIPT │◄───┤ SUBMISSION│◄───┤ PACKAGING │◄─────────┘
│ CONFIRM │    │           │    │           │
└─────────┘    └───────────┘    └───────────┘
```

### Data Collection

```
DATA COLLECTION REQUIREMENTS:
├── Automated extraction from source systems
├── Data quality validation at point of extraction
├── Reconciliation to general ledger
├── Documentation of data sources and extraction methods
├── Timestamp of extraction
└── Sign-off by data owner
```

### Review and QA

```
REVIEW CHECKLIST:
├── Completeness: All required sections present
├── Accuracy: All figures verified to source data
├── Consistency: Figures consistent across report sections
├── Period-over-period: Variances explained and reasonable
├── Regulatory requirements: All requirements met
├── Formatting: Meets regulatory specifications
├── Narrative: Clear, accurate, not misleading
└── Sign-offs: All required approvals obtained
```

### Submission

```
SUBMISSION CONTROLS:
├── Submission through approved channel (regulatory portal, secure file transfer)
├── Submission package validated before sending
├── Submission timestamp recorded
├── Receipt confirmation obtained and recorded
├── Copy of submitted report archived
├── Submission logged in regulatory tracking system
└── Submission deadline compliance verified
```

## Common Reporting Errors and Consequences

### Notable Enforcement Actions

| Institution | Fine | Year | Issue |
|-------------|------|------|-------|
| Various banks | €10M+ each | 2020-2024 | Inaccurate COREP/FINREP reporting |
| US banks | Multiple MRAs | 2019-2024 | FR Y-9C reporting errors |
| Multiple firms | Varies | Ongoing | Late regulatory submissions |
| Several banks | Pillar 2 add-ons | 2018-2024 | BCBS 239 deficiencies affecting reporting |

### Common Error Categories

1. **Data quality**: Source data errors flow into reports
2. **Calculation errors**: Incorrect formulas or methodologies
3. **Classification errors**: Misclassifying exposures or instruments
4. **Completeness**: Missing required data or sections
5. **Timeliness**: Late submissions
6. **Consistency**: Inconsistencies between related reports
7. **Narrative**: Inaccurate or misleading explanations

## Monitoring and Reporting on Regulatory Reporting

```
MONITORING METRICS:
├── Report accuracy:
│   ├── Error rate per report type
│   ├── Restatement frequency
│   └── Root cause analysis of errors
├── Timeliness:
│   ├── On-time submission rate
│   ├── Days before deadline submitted
│   └── Submission process cycle time
├── Data quality:
│   ├── Data completeness rate
│   ├── Data accuracy rate
│   └── Reconciliation break rate
├── Process efficiency:
│   ├── Manual effort hours
│   ├── AI assistance utilization
│   └── Review cycle count
└── Regulatory feedback:
    ├── Regulatory inquiries received
    ├── Data requests from regulators
    └── Findings from regulatory examinations
```

## Common Interview Questions

### Question 1: "What controls would you implement for an AI system that drafts regulatory reports?"

**Good answer structure**:
1. Mandatory human review of every AI-generated section by qualified analysts
2. Source data traceability: every number must trace back to source systems with reconciliation
3. Version control: preserve AI drafts, track human edits, clearly identify final version
4. Automated validation: completeness checks, arithmetic validation, consistency with prior periods
5. Independent review: four-eyes principle, different person reviews than prepares
6. Management sign-off before submission
7. AI performance monitoring: track AI accuracy, error types, improvement trends
8. Full audit trail available for regulatory examination

### Question 2: "How do you ensure BCBS 239 compliance for regulatory reports generated with AI assistance?"

**Good answer structure**:
BCBS 239 requires accurate, complete, timely, and adaptable reporting. For AI-assisted reports: (1) Maintain complete data lineage from source data through AI processing to final report. (2) Validate AI output against source data reconciliation. (3) Ensure reports cover all required risk dimensions. (4) Meet reporting timeliness SLAs. (5) Document the AI processing logic. (6) Have human review for all significant reports. (7) Support drill-down capabilities. (8) Have stress scenario readiness. The AI is a tool -- the bank remains responsible for the report's accuracy and compliance.

### Question 3: "What would you do if you discovered an error in a regulatory report after submission?"

**Good answer structure**:
1. Assess the error: What is wrong? How material is it? What is the impact on the regulatory assessment?
2. Notify: Internal escalation to management, compliance, and legal
3. Regulator: Proactively notify the regulator of the error and its impact -- regulators value transparency
4. Correct: Prepare and submit a corrected report as soon as possible
5. Root cause: Investigate why the error occurred -- was it a data issue, calculation error, AI error, human error?
6. Remediate: Fix the root cause to prevent recurrence
7. Document: Complete incident record for audit purposes
