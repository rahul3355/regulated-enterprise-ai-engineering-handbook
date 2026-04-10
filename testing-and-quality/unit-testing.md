# Unit Testing in Banking GenAI Systems

## Philosophy

Unit tests verify the smallest testable units of code -- typically individual functions or methods. They are the fastest, cheapest, and most reliable tests in your suite. In banking GenAI, unit tests cover:

1. **Traditional code**: Business logic, data transformations, validators
2. **AI-adjacent code**: Prompt template assembly, response parsing, token counting
3. **Quality gates**: Redaction functions, compliance checks, data validators

### What NOT to Unit Test

- LLM calls themselves (use integration tests or evaluation)
- Database queries (use integration tests)
- Network calls (use integration tests with mocks)
- UI rendering (use UI tests)

## Test Structure: Arrange-Act-Assert

```python
# Good unit test
def test_calculate_monthly_payment():
    # Arrange
    principal = 450_000
    annual_rate = 0.065
    term_years = 30

    # Act
    monthly_payment = calculate_monthly_payment(principal, annual_rate, term_years)

    # Assert
    assert monthly_payment == pytest.approx(2844.52, rel=0.01)
```

## Test Naming Convention

Test names should read as specifications:

```python
# GOOD: Name describes behavior
def test_loan_to_value_ratio_exceeds_80_percent_requires_pmi():
    ltv = calculate_ltv(loan_amount=400_000, property_value=450_000)
    assert ltv > 0.80
    assert requires_pmi(ltv) is True

def test_redact_ssn_replaces_nine_digit_pattern():
    result = redact_pii("My SSN is 123-45-6789")
    assert "123-45-6789" not in result
    assert "[REDACTED:ssn]" in result

# BAD: Name adds no information
def test_ltv():
    ...

def test_redaction():
    ...
```

## Testing Banking Business Logic

### Loan Calculation Tests

```python
class TestMortgageCalculations:
    """Tests for mortgage calculation functions."""

    def test_monthly_payment_standard_case(self):
        """Verify monthly payment for a standard 30-year fixed mortgage."""
        payment = calculate_monthly_payment(
            principal=450_000,
            annual_rate=0.065,
            term_years=30
        )
        # Expected: $2,844.52
        assert payment == pytest.approx(2844.52, rel=0.001)

    def test_monthly_payment_zero_interest(self):
        """With 0% interest, payment is principal divided by months."""
        payment = calculate_monthly_payment(
            principal=300_000,
            annual_rate=0.0,
            term_years=15
        )
        assert payment == pytest.approx(300_000 / (15 * 12), rel=0.001)

    def test_total_interest_paid(self):
        """Total interest = (monthly payment * months) - principal."""
        monthly = calculate_monthly_payment(450_000, 0.065, 30)
        total_paid = monthly * 30 * 12
        total_interest = total_paid - 450_000
        assert total_interest == pytest.approx(574_027, rel=0.01)

    def test_loan_to_value_ratio(self):
        """LTV = loan amount / property value."""
        ltv = calculate_ltv(loan_amount=360_000, property_value=450_000)
        assert ltv == pytest.approx(0.80, rel=0.001)

    def test_debt_to_income_ratio(self):
        """DTI = total monthly debt / gross monthly income."""
        dti = calculate_dti(
            monthly_debt=2500,  # mortgage + car + student loans
            gross_monthly_income=7916  # $95,000 / 12
        )
        assert dti == pytest.approx(0.316, rel=0.01)

    @pytest.mark.parametrize("dti,eligible", [
        (0.28, True),   # Well within limit
        (0.36, True),   # At typical limit
        (0.43, True),   # At FHA limit
        (0.44, False),  # Above limit
        (0.50, False),  # Well above limit
    ])
    def test_dti_eligibility(self, dti, eligible):
        """DTI above threshold makes borrower ineligible."""
        assert is_eligible_for_mortgage(dti=dti) == eligible
```

### PII Redaction Tests

```python
class TestPIIRedaction:
    """Tests for PII redaction functions."""

    def test_redact_ssn_standard_format(self):
        redactor = PromptRedactor()
        result = redactor.redact("SSN: 123-45-6789")
        assert result == "SSN: [REDACTED:ssn]"

    def test_redact_ssn_no_spaces(self):
        redactor = PromptRedactor()
        result = redactor.redact("SSN: 123456789")
        assert "123456789" not in result

    def test_redact_email(self):
        redactor = PromptRedactor()
        result = redactor.redact("Email me at john.doe@example.com")
        assert "john.doe@example.com" not in result
        assert "[REDACTED:email]" in result

    def test_redact_credit_card(self):
        redactor = PromptRedactor()
        result = redactor.redact("Card: 4532015112830366")
        assert "4532015112830366" not in result

    def test_redact_multiple_patterns(self):
        redactor = PromptRedactor()
        text = "SSN: 123-45-6789, email: test@test.com, card: 4532015112830366"
        result = redactor.redact(text)
        assert result.count("[REDACTED:") == 3

    def test_redact_no_pii_unchanged(self):
        redactor = PromptRedactor()
        text = "What mortgage rates do you offer?"
        result = redactor.redact(text)
        assert result == text
```

### Prompt Template Tests

```python
class TestPromptTemplates:
    """Tests for prompt template assembly."""

    def test_mortgage_advisor_prompt_includes_context(self):
        """Prompt must include retrieved context documents."""
        context = [
            Document(content="Current 30-year fixed rate: 6.5%", source="rate_sheet"),
            Document(content="Minimum credit score for conventional: 620", source="policy"),
        ]
        prompt = build_mortgage_prompt(
            query="Can I get a mortgage with 620 credit score?",
            context=context
        )

        assert "6.5%" in prompt
        assert "620" in prompt
        assert "Current 30-year fixed rate: 6.5%" in prompt

    def test_mortgage_advisor_prompt_includes_disclaimer(self):
        """Prompt must include compliance disclaimer instruction."""
        prompt = build_mortgage_prompt(
            query="Can I afford a $450k home?",
            context=[]
        )

        assert "disclaimer" in prompt.lower()
        assert "not financial advice" in prompt.lower() or "consult" in prompt.lower()

    def test_prompt_token_count_within_limits(self):
        """Assembled prompt must not exceed model's context window."""
        context = [Document(content="x" * 5000) for _ in range(10)]
        prompt = build_mortgage_prompt(
            query="What rates do you offer?",
            context=context
        )

        tokens = count_tokens(prompt)
        assert tokens < 128_000  # GPT-4 Turbo context limit
```

### Response Parser Tests

```python
class TestResponseParser:
    """Tests for LLM response parsing."""

    def test_parse_json_response(self):
        """Parser should extract JSON from markdown code blocks."""
        raw_response = '''Here is the analysis:

```json
{
    "recommendation": "proceed",
    "risk_level": "low",
    "dti_ratio": 0.32
}
```

Let me know if you need more details.'''

        result = parse_json_response(raw_response)
        assert result["recommendation"] == "proceed"
        assert result["risk_level"] == "low"
        assert result["dti_ratio"] == pytest.approx(0.32)

    def test_parse_malformed_json_raises_error(self):
        """Parser should handle malformed JSON gracefully."""
        raw_response = "This is not JSON at all"

        with pytest.raises(ParseError):
            parse_json_response(raw_response, strict=True)

        # Non-strict mode returns None
        assert parse_json_response(raw_response, strict=False) is None

    def test_parse_response_validates_schema(self):
        """Parser should validate required fields."""
        raw_response = '{"recommendation": "proceed"}'
        required_fields = ["recommendation", "risk_level", "dti_ratio"]

        with pytest.raises(ValidationError, match="Missing required fields"):
            parse_json_response(raw_response, required_fields=required_fields)
```

## Testing with pytest Fixtures

```python
@pytest.fixture
def sample_mortgage_application():
    return MortgageApplication(
        applicant_income=95_000,
        loan_amount=450_000,
        property_value=500_000,
        credit_score=720,
        employment_years=8,
        existing_debt_monthly=1_500,
    )

@pytest.fixture
def redactor():
    return PromptRedactor()

@pytest.fixture
def mock_llm_client():
    with patch('src.llm.OpenAIClient') as mock:
        mock.return_value.chat.return_value = ChatResponse(
            content="Test response",
            tokens_prompt=100,
            tokens_completion=50,
        )
        yield mock

def test_mortgage_eligibility_check(sample_mortgage_application):
    eligibility = check_mortgage_eligibility(sample_mortgage_application)
    assert eligibility.eligible is True
    assert eligibility.reasons == []

def test_mortgage_ineligible_high_dti():
    application = MortgageApplication(
        applicant_income=40_000,
        loan_amount=450_000,
        property_value=500_000,
        credit_score=720,
        employment_years=8,
        existing_debt_monthly=5_000,  # Very high DTI
    )
    eligibility = check_mortgage_eligibility(application)
    assert eligibility.eligible is False
    assert "debt-to-income" in eligibility.reasons[0].lower()
```

## Test Coverage Requirements

```
┌─────────────────────────────────────────────────────────────┐
│  COVERAGE REQUIREMENTS                                      │
├─────────────────────────────┬───────────────────────────────┤
│  Code Type                  │  Minimum Coverage             │
├─────────────────────────────┼───────────────────────────────┤
│  Banking calculations       │  95% (branch coverage)        │
│  PII redaction              │  100% (all patterns tested)   │
│  Prompt templates           │  90%                          │
│  Response parsers           │  90%                          │
│  Validators                 │  95%                          │
│  API handlers               │  80%                          │
│  Utility functions          │  80%                          │
│  Configuration              │  70%                          │
└─────────────────────────────┴───────────────────────────────┘
```

Enforce coverage in CI:

```ini
# pyproject.toml
[tool.pytest.ini_options]
addopts = """
    --cov=src
    --cov-report=term-missing
    --cov-report=html
    --cov-fail-under=85
    --cov-branch
"""
```

## Common Unit Testing Mistakes

1. **Testing implementation details**: Test behavior, not implementation. `assert result == 5` is better than `assert internal_counter.value == 5`.

2. **No assertion messages**: When a test fails, `assert x == y` with no message makes debugging hard. Use `assert x == y, f"Expected {y}, got {x}"`.

3. **Shared state between tests**: Each test must be independent. Use fixtures and clean up after tests.

4. **Testing external calls**: Unit tests should not call the database, the network, or the LLM. Use mocks.

5. **Brittle tests**: Tests that break when refactoring (but behavior is unchanged) are testing the wrong thing.

6. **Not testing edge cases**: Zero values, negative values, empty strings, None -- these cause the most production bugs.
