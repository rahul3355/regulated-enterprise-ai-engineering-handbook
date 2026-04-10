# Third-Party AI Risk

## Overview

Using third-party AI services (OpenAI, Anthropic, Google, Mistral, etc.) introduces risks beyond standard vendor risk management. This document focuses on the unique risks of external AI providers in a banking context.

## Risk Categories

### 1. Data Privacy Risk

**Risk**: Customer data sent to third-party AI APIs may be stored, processed, or used in ways the bank does not control.

**Specific concerns**:
- Vendor may log prompts and responses (which may contain PII)
- Vendor may use customer data to improve their models
- Model training on customer data could expose patterns to other customers
- Data may be transferred to jurisdictions with different privacy laws
- Vendor employees may have access to customer data
- Data breach at vendor exposes customer conversations

**Real-world incidents**:
- **Samsung (2023)**: Employees pasted confidential code into ChatGPT, company banned ChatGPT after discovering the leak
- **ChatGPT vulnerability (March 2023)**: Bug in Redis library allowed users to see other users' chat titles and payment info
- **OpenAI data incident (March 2023)**: Users could see titles and contents of other users' conversations

**Mitigations**:
```
DATA PRIVACY MITIGATIONS:
├── Use enterprise/API offerings with zero-retention options
│   ├── OpenAI API: Data not used for training by default
│   ├── Azure OpenAI: Data not used for training, EU data boundary option
│   └── Anthropic API: Data not used for training
├── Do NOT send PII to third-party AI APIs
│   ├── PII detection and redaction before API call
│   ├── Tokenize customer identifiers
│   └── Use internal models for PII-containing prompts
├── Implement data processing agreements
│   ├── GDPR Article 28 compliant DPA
│   ├── SCCs for international transfers
│   └── Data location commitments
├── Monitor vendor's privacy policy for changes
├── Conduct periodic vendor data handling audits
└── Maintain data transfer inventory
```

### 2. Model Integrity Risk

**Risk**: The vendor's model may produce incorrect, biased, harmful, or inconsistent outputs.

**Specific concerns**:
- Hallucinations: Model generates plausible-sounding but false information
- Bias: Model produces outputs that discriminate against protected groups
- Inconsistency: Same prompt produces different answers at different times
- Prompt injection: Malicious inputs manipulate model behavior
- Jailbreak: Users bypass safety guardrails
- Prompt leakage: Model reveals its own system instructions or training data
- Adversarial attacks: Specially crafted inputs cause specific failures

**Real-world incidents**:
- **Air Canada chatbot (2024)**: AI chatbot provided incorrect refund policy information; court ruled airline liable for chatbot's statements
- **ChatGPT hallucination lawsuits (2023-2024)**: Multiple cases of AI-generated false information in legal filings
- **Bing Chat "Sydney" incidents (2023)**: Chatbot exhibited manipulative behavior, threatened users

**Mitigations**:
```
MODEL INTEGRITY MITIGATIONS:
├── Input validation and sanitization
│   ├── Prompt injection detection
│   ├── Input length and format validation
│   └── Content filtering on inputs
├── Output validation
│   ├── Fact-checking against known sources
│   ├── Content filtering on outputs
│   ├── Format validation (expected structure)
│   └── Confidence scoring
├── Safety guardrails
│   ├── System prompts defining boundaries
│   ├── Temperature control for consistency
│   ├── Response length limits
│   └── Prohibited topic lists
├── Human review
│   ├── Human-in-the-loop for significant decisions
│   ├── Spot-checking of AI outputs
│   └── Customer feedback collection
├── Testing
│   ├── Adversarial prompt testing
│   ├── Bias testing across demographic groups
│   ├── Consistency testing (same prompt, multiple runs)
│   └── Edge case testing
└── Monitoring
    ├── Hallucination rate tracking
    ├── Safety violation tracking
    ├── Output quality metrics
    └── Customer complaint trends
```

### 3. Availability Risk

**Risk**: Third-party AI services may experience outages, rate limiting, or degradation.

**Specific concerns**:
- Vendor service outage (complete or partial)
- Rate limiting during peak usage
- API version deprecation
- Model deprecation
- Network connectivity issues
- Vendor capacity constraints

**Real-world incidents**:
- **ChatGPT outage (November 2022)**: Major outage affecting millions of users during peak adoption
- **OpenAI API degradation (multiple 2023)**: Periods of elevated error rates and latency
- **Azure OpenAI regional outage (2023)**: Regional service disruption affecting enterprise customers

**Mitigations**:
```
AVAILABILITY MITIGATIONS:
├── Multi-vendor strategy
│   ├── Primary vendor: OpenAI
│   ├── Secondary vendor: Anthropic or Google
│   └── Internal fallback model (open-source)
├── Abstraction layer
│   ├── Standardized API interface
│   ├── Easy vendor switching
│   └── Automatic failover
├── Monitoring
│   ├── Real-time availability monitoring
│   ├── Latency monitoring
│   ├── Error rate tracking
│   └── SLA compliance tracking
├── Rate management
│   ├── Request queuing and throttling
│   ├── Priority routing for critical requests
│   └── Batch processing for non-urgent requests
├── Offline fallback
│   ├── Graceful degradation when AI unavailable
│   ├── Cached responses for common queries
│   └── Human escalation when AI unavailable
└── Contractual
    ├── SLA with uptime guarantees
    ├── Penalties for SLA breaches
    └── Support escalation procedures
```

### 4. Regulatory and Compliance Risk

**Risk**: Using third-party AI may violate regulatory requirements or create new compliance obligations.

**Specific concerns**:
- EU AI Act: Vendor may not comply with high-risk AI requirements
- GDPR: International data transfers may not be adequately protected
- SR 11-7: Limited visibility into vendor's model development and validation
- FCA: Inability to explain AI decisions to consumers or regulators
- Banking regulations: Outsourcing of critical functions may require notification
- Industry regulations: Specific restrictions on third-party data processing

**Mitigations**:
```
REGULATORY MITIGATIONS:
├── Vendor compliance assessment
│   ├── EU AI Act compliance status
│   ├── GDPR compliance verification
│   ├── SOC 2 Type II review
│   └── Industry-specific certifications
├── Regulatory notifications
│   ├── Notify regulators of critical AI vendor relationships
│   └── Report material changes to vendor arrangements
├── Documentation
│   ├── Document vendor's model development practices
│   ├── Document data flows and processing
│   └── Document risk assessment and mitigations
├── Explainability
│   ├── Understand and document what can be explained
│   ├── Acknowledge limitations to regulators
│   └── Implement compensating controls (human review)
└── Contingency
    ├── Plan for vendor regulatory action
    ├── Alternative vendors pre-approved
    └── Internal capability as fallback
```

### 5. Reputational Risk

**Risk**: Vendor actions or AI output failures may damage the bank's reputation.

**Specific concerns**:
- AI generates offensive or inappropriate content to customers
- Vendor is involved in controversy (e.g., copyright lawsuits, political bias claims)
- AI makes commitments the bank cannot honor
- AI provides incorrect financial information
- Customer data exposed through vendor breach

**Real-world incidents**:
- **Air Canada (2024)**: Chatbot provided incorrect refund information, bank found liable
- **Various companies (2023-2024)**: Chatbots making inappropriate commitments to customers

**Mitigations**:
```
REPUTATIONAL MITIGATIONS:
├── Content filtering
│   ├── Input content filtering
│   ├── Output content filtering
│   └── Prohibited response patterns
├── Response boundaries
│   ├── System prompts defining scope
│   ├── "I don't know" over hallucination
│   ├── Referral to human for complex topics
│   └── No financial commitments without human approval
├── Monitoring and alerting
│   ├── Inappropriate content detection
│   ├── Customer sentiment monitoring
│   └── Complaint trend analysis
├── Customer communication
│   ├── Clear disclosure of AI interaction
│   ├── Explanation of AI capabilities and limitations
│   └── Easy access to human support
└── Incident response
    ├── Rapid response to inappropriate outputs
    ├── Customer notification and remediation
    └── Root cause analysis and prevention
```

### 6. Strategic Risk

**Risk**: Dependency on third-party AI creates strategic vulnerability.

**Specific concerns**:
- Vendor pricing changes significantly
- Vendor changes business model (e.g., from API to product-only)
- Vendor is acquired by a competitor or controversial entity
- Vendor's technology is superseded by competitors
- Vendor exit from market or service discontinuation

**Mitigations**:
```
STRATEGIC RISK MITIGATIONS:
├── Multi-vendor strategy (avoid single-vendor dependency)
├── Abstraction layer for easy vendor switching
├── Internal AI capability development (open-source models)
├── Regular evaluation of alternative vendors
├── Contractual protections (price caps, continuity commitments)
├── Strategic relationship management (regular vendor meetings)
├── Market intelligence (monitor AI vendor landscape)
└── Escalation to senior management for strategic decisions
```

## Vendor Comparison Framework

### Key Evaluation Dimensions

```
VENDOR EVALUATION MATRIX:
┌─────────────────────────┬──────────┬──────────┬──────────┬──────────┐
│ Criterion               │ OpenAI   │ Anthropic│ Google   │ Internal │
├─────────────────────────┼──────────┼──────────┼──────────┼──────────┤
│ Data privacy            │ Good*    │ Strong   │ Strong   │ Best     │
│ Zero-retention option   │ Yes      │ Yes      │ Yes      │ N/A      │
│ Model transparency      │ Limited  │ Moderate │ Limited  │ Full     │
│ Explainability          │ Limited  │ Limited  │ Limited  │ Variable │
│ Performance             │ Leading  │ Leading  │ Leading  │ Variable │
│ Availability SLA        │ 99.9%    │ 99.9%    │ 99.95%   │ Internal │
│ Cost predictability     │ Moderate │ Moderate │ Moderate │ Controlled│
│ Regulatory commitment   │ Moderate │ Strong   │ Strong   │ Full     │
│ EU data boundary        │ Via Azure│ Yes      │ Yes      │ Internal │
│ Audit rights            │ Limited  │ Moderate │ Moderate │ Full     │
│ Model version pinning   │ Yes      │ Yes      │ Yes      │ Full     │
│ Customization           │ Fine-tune│ Fine-tune│ Fine-tune│ Full     │
└─────────────────────────┴──────────┴──────────┴──────────┴──────────┘

* OpenAI API: data not used for training by default
```

## Common Interview Questions

### Question 1: "What are the top risks of using ChatGPT's API for customer service in a bank?"

**Good answer structure**:
1. Data privacy: Customer prompts may contain PII sent to OpenAI's servers; must use zero-retention and redact PII
2. Model integrity: Hallucinations could provide incorrect financial advice; must implement output validation and human review
3. Availability: API outages would disrupt customer service; must have fallback options
4. Regulatory: Cannot fully explain ChatGPT's decisions to regulators; must document limitations and compensating controls
5. Reputational: Inappropriate responses could damage the bank; must implement content filtering and monitoring
6. Strategic: Dependency on OpenAI; must maintain multi-vendor strategy

### Question 2: "How would you handle a situation where the AI vendor's model starts producing biased outputs?"

**Good answer structure**:
1. Detect: Monitoring should catch the bias through regular fairness testing and customer feedback
2. Contain: Immediately implement additional output filtering or reduce automation level
3. Investigate: Determine root cause -- model change, input data shift, prompt issue
4. Notify: Inform vendor, internal stakeholders, and if affected customers were harmed
5. Remediate: Work with vendor on fix, adjust prompts, add additional guardrails
6. Validate: Test the fix before resuming normal operations
7. Document: Record the incident, actions taken, and lessons learned
8. Prevent: Add the bias pattern to automated testing to catch future occurrences

### Question 3: "Can a bank comply with SR 11-7 when using a black-box AI model from a vendor?"

**Good answer**:
It's challenging but possible with compensating controls. You cannot validate the vendor's internal model development, so you must: (1) Classify the model at an appropriate risk level (likely higher due to reduced visibility). (2) Validate what you can: input/output behavior, performance on your specific use case, fairness testing. (3) Document all limitations: you cannot verify training data quality, methodology soundness, or internal testing. (4) Implement stronger compensating controls: more rigorous ongoing monitoring, human review of decisions, more conservative use cases. (5) Require vendor documentation: model cards, testing results, change notifications. (6) Escalate the limitations to senior management and model risk governance.
