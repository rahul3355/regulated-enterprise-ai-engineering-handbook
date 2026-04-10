# Prompt Libraries

This directory covers building a centralized prompt management system for the enterprise, including templating, versioning, and sharing prompts across teams.

## Key Topics

- Centralized prompt repository architecture
- Prompt templating with variable substitution
- Version control and change management
- Cross-team prompt sharing and reuse
- Prompt quality benchmarking
- Prompt lifecycle management

## Prompt Template System

```yaml
# prompts/transaction_risk_analysis.yaml
id: transaction_risk_analysis
version: "2.1.0"
status: production
owner: fraud-engineering
description: >
  Analyze a transaction for potential fraud risk factors.
  Used by fraud investigation team and transaction monitoring.

model: claude-3-5-sonnet
temperature: 0.1
max_tokens: 1500

system_prompt: |
  You are a Senior Fraud Analyst at a global bank. Analyze the provided \
  transaction for potential fraud indicators.

user_template: |
  Transaction Details:
  - Amount: {{amount}}
  - Type: {{transaction_type}}
  - Source: {{source_account}}
  - Destination: {{destination_account}}
  - Customer tenure: {{customer_tenure_days}} days

  Customer Profile:
  - Risk rating: {{customer_risk_rating}}
  - Typical transaction range: £{{typical_min}} - £{{typical_max}}
  - Known beneficiaries: {{known_beneficiaries}}

  Analyze for fraud indicators and provide risk assessment.

variables:
  amount:
    type: number
    required: true
    description: "Transaction amount in GBP"
  transaction_type:
    type: enum
    enum: ["wire_transfer", "fps", "chaps", "card_payment", "direct_debit", "standing_order"]
    required: true
  customer_tenure_days:
    type: integer
    required: true
    minimum: 0

tags: [fraud, transaction-monitoring, risk-assessment]
last_reviewed: "2024-03-15"
next_review: "2024-06-15"
```

## Cross-References

- [../prompt-versioning.md](../prompt-versioning.md) — Prompt version management
- [../prompt-engineering.md](../prompt-engineering.md) — Prompt design patterns
- [../safe-rollout-strategies.md](../safe-rollout-strategies.md) — Rollout for prompt changes
- [../evaluation-frameworks/](../evaluation-frameworks/) — Prompt quality testing
