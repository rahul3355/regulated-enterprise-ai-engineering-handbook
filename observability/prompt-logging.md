# Prompt Logging for Banking GenAI Systems

## The Challenge

In traditional systems, logging inputs and outputs is straightforward. In GenAI systems, prompts may contain customer financial data, and logging them creates a PII and compliance risk. Yet you need prompt data for debugging, auditing, and model improvement.

This document defines safe prompt logging practices for banking environments.

## What NOT to Log

The following must NEVER appear in any log, trace, or observability system:

```python
# NEVER log these
REDACTED_PATTERNS = {
    # Personal identifiers
    'ssn': r'\d{3}-\d{2}-\d{4}',
    'account_number': r'\d{8,17}',  # Bank account numbers
    'routing_number': r'\d{9}',
    'card_number': r'\d{13,19}',
    'date_of_birth': r'\d{4}-\d{2}-\d{2}',  # When in DOB context

    # Authentication secrets
    'password': r'.*',
    'api_key': r'sk-[a-zA-Z0-9]{20,}',
    'token': r'eyJ[a-zA-Z0-9._-]+',

    # Contact information
    'email': r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
    'phone': r'\+?\d{10,15}',

    # Financial data
    'salary': r'\$?\d{5,}',  # Specific salary figures
    'balance': r'\$?\d{4,}',  # Account balances
    'transaction_amount': r'\$?\d{3,}',  # Transaction amounts
}
```

## Safe Prompt Logging

### Hashing for Deduplication

If you need to detect duplicate prompts without logging the content:

```python
import hashlib

def hash_prompt(prompt: str) -> str:
    """Create a deterministic hash of a prompt for deduplication."""
    return hashlib.sha256(f"bank-prompt-salt-{prompt}".encode()).hexdigest()[:32]

# Usage
log_entry = {
    "prompt_hash": hash_prompt(user_prompt),
    "prompt_length": len(user_prompt),
    "prompt_token_count": count_tokens(user_prompt),
    # Never log the actual prompt
}
```

### Redaction Before Logging

```python
import re

class PromptRedactor:
    """Redact sensitive information from prompts before logging."""

    PATTERNS = {
        'ssn': r'\b\d{3}-\d{2}-\d{4}\b',
        'account': r'\b\d{8,17}\b',
        'card': r'\b\d{13,19}\b',
        'email': r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
        'phone': r'\b\+?\d{10,15}\b',
        'ssn_spaced': r'\b\d{3}\s\d{2}\s\d{4}\b',
        'dollar_amount': r'\$[\d,]+(?:\.\d{2})?',
    }

    def redact(self, text: str) -> str:
        """Redact all sensitive patterns from text."""
        result = text
        for name, pattern in self.PATTERNS.items():
            result = re.sub(pattern, f'[REDACTED:{name}]', result)
        return result

redactor = PromptRedactor()

# Before: "My salary is $95,000 and my SSN is 123-45-6789"
# After:  "My salary is [REDACTED:dollar_amount] and my SSN is [REDACTED:ssn]"
safe_prompt = redactor.redact(user_prompt)
```

### What IS Safe to Log

The following prompt metadata is safe and valuable for debugging:

```json
{
  "level": "INFO",
  "timestamp": "2025-03-15T14:32:01.234Z",
  "service": "llm-gateway",
  "correlation_id": "req-abc123",
  "message": "LLM call completed",
  "prompt_hash": "a3f2c8d1e5b9f7a4c2d8e6f1a3b5c7d9",
  "prompt_length_chars": 1245,
  "prompt_token_count": 312,
  "prompt_category": "mortgage_inquiry",
  "prompt_language": "en",
  "system_prompt_version": "mortgage-advisor-v3.2",
  "context_documents_count": 5,
  "context_documents_total_tokens": 890,
  "contains_financial_advice_request": true,
  "requires_compliance_disclaimer": true,
  "response_tokens": 456,
  "response_finish_reason": "stop",
  "model": "gpt-4-turbo",
  "duration_ms": 2890,
  "cost_usd": 0.0423
}
```

## Audit Logging for Compliance

Banking regulations require audit trails for AI-driven financial advice. The audit log must record:

```python
def create_audit_log_entry(interaction: GenAIInteraction) -> dict:
    """Create a compliance-grade audit log entry."""
    return {
        "audit_id": str(uuid.uuid4()),
        "timestamp": datetime.utcnow().isoformat(),
        "interaction_type": interaction.type,
        "product_line": interaction.product_line,
        "user_tier": interaction.user_tier,
        "user_id_hash": hash_user_id(interaction.user_id),
        "session_id": interaction.session_id,
        "correlation_id": interaction.correlation_id,

        # Model information
        "model_name": interaction.model,
        "model_version": interaction.model_version,
        "provider": interaction.provider,
        "temperature": interaction.temperature,
        "max_tokens": interaction.max_tokens,

        # Token usage
        "prompt_tokens": interaction.prompt_tokens,
        "completion_tokens": interaction.completion_tokens,
        "total_tokens": interaction.total_tokens,
        "estimated_cost_usd": interaction.cost_usd,

        # Content metadata (NOT content itself)
        "prompt_length": interaction.prompt_length,
        "prompt_category": interaction.prompt_category,
        "response_length": interaction.response_length,
        "response_contains_disclaimer": interaction.disclaimer_present,
        "response_contains_sources": interaction.sources_present,
        "source_document_count": len(interaction.source_documents),

        # Quality indicators
        "confidence_score": interaction.confidence,
        "requires_human_review": interaction.requires_review,
        "hallucination_risk": interaction.hallucination_risk_level,

        # Retention
        "retention_years": 7,
        "data_classification": "confidential",
        "regulatory_framework": ["TILA", "RESPA", "ECOA"]
    }
```

## Prompt Categories

Classifying prompts enables analysis without logging content:

```python
PROMPT_CATEGORIES = {
    'mortgage_inquiry': 'Questions about mortgage products, rates, eligibility',
    'account_inquiry': 'Questions about existing accounts, balances, status',
    'investment_advice': 'Questions about investment products, portfolio advice',
    'credit_card_inquiry': 'Questions about credit card products, rates',
    'loan_application': 'Questions about loan application process',
    'general_banking': 'General banking product questions',
    'complaint': 'Customer complaints or issues',
    'fraud_report': 'Fraud or unauthorized transaction reports',
    'greeting': 'Greetings, small talk, system tests',
    'off_topic': 'Questions outside banking domain',
    'adversarial': 'Attempts to manipulate the AI system',
}

def classify_prompt(prompt: str) -> str:
    """Classify a prompt into a category without logging content."""
    # Use a lightweight classifier or keyword matching
    # Never log the actual prompt text
    for category, keywords in CATEGORY_KEYWORDS.items():
        if any(kw in prompt.lower() for kw in keywords):
            return category
    return 'general_banking'
```

## Log Retention for Prompts

```
┌─────────────────────────────────────────────────────────────┐
│  PROMPT LOG RETENTION POLICY                                │
├─────────────────────┬───────────────┬───────────────────────┤
│  Data Type          │  Retention    │  Storage              │
├─────────────────────┼───────────────┼───────────────────────┤
│  Prompt hashes      │  7 years     │  Cold storage (S3)    │
│  Prompt metadata    │  7 years     │  Cold storage (S3)    │
│  Audit log entries  │  7 years     │  Immutable storage    │
│  Token usage data   │  2 years     │  Metrics storage      │
│  Redacted prompts   │  90 days     │  Hot storage (Elastic)│
│  Full prompts       │  NEVER       │  N/A                  │
└─────────────────────┴───────────────┴───────────────────────┘
```

## Testing Redaction

```python
class TestPromptRedaction:
    """Verify that redaction works correctly."""

    def test_ssn_redaction(self):
        redactor = PromptRedactor()
        text = "My SSN is 123-45-6789 and I want a loan"
        result = redactor.redact(text)
        assert '123-45-6789' not in result
        assert '[REDACTED:ssn]' in result

    def test_email_redaction(self):
        redactor = PromptRedactor()
        text = "Contact me at john.doe@example.com please"
        result = redactor.redact(text)
        assert 'john.doe@example.com' not in result
        assert '[REDACTED:email]' in result

    def test_dollar_amount_redaction(self):
        redactor = PromptRedactor()
        text = "I earn $95,000 per year"
        result = redactor.redact(text)
        assert '$95,000' not in result
        assert '[REDACTED:dollar_amount]' in result

    def test_multiple_redactions(self):
        redactor = PromptRedactor()
        text = "SSN: 123-45-6789, email: test@test.com, salary: $85,000"
        result = redactor.redact(text)
        assert result.count('[REDACTED:') == 3
```

## Regulatory Requirements

### Key Regulations Affecting Prompt Logging

| Regulation | Requirement | Impact on Logging |
|------------|-------------|-------------------|
| GLBA | Protect customer financial data | No financial data in logs |
| CCPA/CPRA | Right to know/delete | Must be able to identify and delete customer data |
| ECOA | Fair lending | Must not log protected class information |
| TILA | Truth in lending | Must log advice given, but not customer data |
| SOX | Audit trails | Must log AI decisions affecting financial products |

### Prompt Logging Compliance Checklist

- [ ] All PII patterns redacted before logging
- [ ] Prompt hashing uses salted, one-way function
- [ ] Audit log entries created for every financial advice interaction
- [ ] Audit logs stored in immutable, tamper-evident storage
- [ ] Retention policy enforced (7 years minimum for audit)
- [ ] Access to prompt audit logs restricted and logged
- [ ] Redaction patterns reviewed quarterly
- [ ] False negative rate of redaction measured and reported
