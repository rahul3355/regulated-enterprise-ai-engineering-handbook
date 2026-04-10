# Guardrails and Content Filtering

This directory covers implementing AI safety guardrails using Guardrails AI, NVIDIA NeMo Guardrails, and custom content filtering for banking GenAI applications.

## Key Topics

- Guardrails AI for structured output validation
- NVIDIA NeMo Guardrails for dialogue management
- Custom content filtering for banking-specific rules
- Input validation and prompt injection defense
- Output validation for PII, hallucination, harmful content
- Guardrails performance impact and optimization

## Guardrails AI

```python
from guardrails import Guard, OnFailAction
from guardrails.hub import CompetentLLM, DetectPII, BugFreeSQL

# Banking guardrails configuration
guard = Guard().use(
    CompetentLLM,  # Ensure model produces substantive response
    DetectPII(
        pii_entities=["PERSON", "ACCOUNT_NUMBER", "CREDIT_CARD"],
        on_fail=OnFailAction.REDACT,
    ),
    BugFreeSQL(),  # If model generates SQL
)

# Apply guardrails to model output
validated_response = guard.validate(model_output)
```

## NeMo Guardrails for Banking

```python
# Banking-specific dialogue rails
rails_config = """
define user greet
  "Hello"
  "Hi there"

define bot greet
  "Hello! I'm your banking assistant. How can I help you today?"

define user ask about interest rates
  "What are your current mortgage rates?"

define bot respond about rates
  "I can provide general information about our rate products. For personalized \
   rate quotes, please speak with our mortgage team. Current base rates start at..."

# Safety rails
define flow
  user ask_account_balance
  bot check_identity_first  # Always verify identity first

define flow
  user ask_about_competitor
  bot redirect_to_advantages  # Never discuss competitor products negatively

define flow
  user attempt_system_prompt_extraction
  bot refuse_and_log  # Block prompt injection
"""
```

## Banking-Specific Content Filters

| Filter Type | Purpose | Implementation |
|-------------|---------|---------------|
| PII Detection | Prevent data leakage | NER-based redaction |
| Financial Advice Disclaimer | Regulatory compliance | Output post-processing |
| Tipping Off Prevention | AML legal requirement | Pattern matching + ML classifier |
| Hallucination Detection | Factual accuracy | NLI-based verification |
| Tone Appropriateness | Brand consistency | Sentiment analysis |
| Regulatory Citation Check | Compliance accuracy | Cross-reference with regulation DB |

## Cross-References

- [../ai-safety.md](../ai-safety.md) — AI safety principles
- [../hallucinations.md](../hallucinations.md) — Hallucination detection
- [../security/](../security/) — Input validation and injection defense
- [../ai-safety-reviews/](../ai-safety-reviews/) — Safety review process
