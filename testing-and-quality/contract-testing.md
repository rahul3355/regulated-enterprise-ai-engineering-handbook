# Contract Testing for Banking GenAI

## What Is Contract Testing?

Contract testing verifies that two services (a consumer and a provider) can communicate correctly by validating that they adhere to a shared contract. Unlike integration tests, contract tests do not require both services to be running simultaneously.

In banking GenAI, contracts exist between:
- Frontend and backend APIs
- Backend services and the LLM gateway
- LLM gateway and external model providers
- RAG pipeline and vector database

## Why Contract Testing Matters

```
Without contract testing:

Service A (Consumer)        Service B (Provider)
┌─────────────────┐        ┌─────────────────┐
│ Expects:        │        │ Returns:        │
│ {               │        │ {               │
│   "rate": 0.065 │  ???   │   "rate": 0.065 │
│   "disclaimer": │        │   "disclaimer": │
│     true        │        │     "yes"       │  <-- Type mismatch!
│ }               │        │ }               │
└─────────────────┘        └─────────────────┘

Service A expects boolean, Service B returns string.
Integration test passes. Production breaks.
```

## Pact for Consumer-Driven Contracts

Pact is the most widely used contract testing framework. It follows this workflow:

1. **Consumer writes a test** that defines its expectations
2. **Pact generates a contract** (JSON file) from the test
3. **Provider verifies** against the contract
4. **Contract is shared** via a Pact Broker

### Consumer Test (Python)

```python
from pact import Consumer, Provider
import pytest

@pytest.fixture
def pact():
    return Consumer("loan-advisor-api").has_pact_with(
        Provider("llm-gateway"),
        pact_dir="./pacts",
    )

def test_llm_gateway_contract(pact):
    """Define the contract the Loan Advisor API expects from LLM Gateway."""

    # Define the expected interaction
    (
        pact
        .given("a valid mortgage advice request exists")
        .upon_receiving("a request for mortgage advice")
        .with_request("POST", "/api/v1/mortgage-advice",
            body={
                "salary": 95000,
                "loan_amount": 450000,
                "term_years": 30,
            },
            headers={"Content-Type": "application/json"},
        )
        .will_respond_with(200,
            body={
                "recommendation": pact.like("Based on your profile..."),
                "monthly_payment": pact.EachLike(2844.52),
                "disclaimer": pact.EachLike("This is not financial advice"),
                "risk_level": pact.Term("low|medium|high", "low"),
            },
            headers={"Content-Type": "application/json"},
        )
    )

    with pact:
        # Make the actual call during test
        response = requests.post(
            f"{mock_server.uri}/api/v1/mortgage-advice",
            json={"salary": 95000, "loan_amount": 450000, "term_years": 30},
        )
        assert response.status_code == 200
```

This generates a Pact contract file:

```json
{
  "consumer": {"name": "loan-advisor-api"},
  "provider": {"name": "llm-gateway"},
  "interactions": [
    {
      "description": "a request for mortgage advice",
      "providerState": "a valid mortgage advice request exists",
      "request": {
        "method": "POST",
        "path": "/api/v1/mortgage-advice",
        "body": {
          "salary": 95000,
          "loan_amount": 450000,
          "term_years": 30
        }
      },
      "response": {
        "status": 200,
        "body": {
          "recommendation": "Based on your profile...",
          "monthly_payment": 2844.52,
          "disclaimer": "This is not financial advice",
          "risk_level": "low"
        }
      }
    }
  ]
}
```

### Provider Verification

```python
from pact import Verifier
from fastapi import FastAPI
import pytest

app = FastAPI()

@app.post("/api/v1/mortgage-advice")
async def mortgage_advice(request: dict):
    # Provider implementation
    return {
        "recommendation": "Based on your profile, you qualify for...",
        "monthly_payment": 2844.52,
        "disclaimer": "This is not financial advice. Please consult...",
        "risk_level": "low",
    }

@pytest.fixture
def verifier():
    return Verifier(
        provider="llm-gateway",
        provider_app_url="http://localhost:8000",
    )

def test_provider_verification(verifier):
    """Verify provider satisfies all consumer contracts."""
    verifier.verify_pacts(
        "./pacts/loan-advisor-api-llm-gateway.json",
    )
```

## API Schema Contracts with Pydantic

For Python services, Pydantic models serve as runtime contracts:

```python
from pydantic import BaseModel, Field, validator

class MortgageAdviceRequest(BaseModel):
    """Contract: what the consumer must send."""
    salary: float = Field(..., gt=0, description="Annual gross income")
    loan_amount: float = Field(..., gt=0, description="Requested loan amount")
    term_years: int = Field(..., ge=1, le=50, description="Loan term")
    credit_score: int = Field(..., ge=300, le=850, description="Credit score")

class MortgageAdviceResponse(BaseModel):
    """Contract: what the provider must return."""
    recommendation: str = Field(..., min_length=50)
    monthly_payment: float = Field(..., gt=0)
    disclaimer: str = Field(..., min_length=20)
    risk_level: str = Field(..., pattern="^(low|medium|high)$")
    dti_ratio: float = Field(..., ge=0, le=1)

    @validator("disclaimer")
    def disclaimer_must_contain_required_text(cls, v):
        required_terms = ["not financial advice", "consult", "professional"]
        if not any(term in v.lower() for term in required_terms):
            raise ValueError("Disclaimer must contain advisory language")
        return v
```

## LLM Output Schema Contracts

Define expected output schemas for LLM responses:

```python
from pydantic import BaseModel
from typing import Optional

class RAGResponse(BaseModel):
    """Contract for RAG pipeline responses."""
    answer: str = Field(..., min_length=10, description="The AI-generated answer")
    source_documents: list[dict] = Field(
        ..., min_items=1, description="Source documents used"
    )
    confidence: float = Field(..., ge=0, le=1, description="Confidence score")
    disclaimer_present: bool = Field(
        ..., description="Whether disclaimer is included"
    )
    model_used: str = Field(..., description="Which model generated this")
    tokens_used: int = Field(..., gt=0, description="Total tokens consumed")

    @validator("source_documents")
    def sources_must_have_required_fields(cls, docs):
        for doc in docs:
            required = ["content", "source", "similarity"]
            missing = [k for k in required if k not in doc]
            if missing:
                raise ValueError(f"Source document missing: {missing}")
        return docs
```

## Contract Testing in CI/CD

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  consumer-contracts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pytest tests/contracts/consumer/
      - name: Publish contracts to Pact Broker
        run: |
          pact-broker publish ./pacts \
            --consumer-app-version ${{ github.sha }} \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }}

  provider-verification:
    needs: consumer-contracts
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verify provider against contracts
        run: |
          pact-verifier \
            --provider-base-url http://localhost:8000 \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }}
```

## Common Contract Testing Mistakes

1. **Contracts only test happy path**: Include error responses in contracts (400, 500, etc.).

2. **No version tracking**: Contracts must be versioned. When a consumer changes its expectations, the contract version should change.

3. **Ignoring the LLM contract**: The LLM gateway's output contract is as important as any API contract. Define and test it.

4. **Contracts not enforced at runtime**: A contract verified in CI but not at runtime allows schema drift between deployments.

5. **Too-specific contracts**: Using `pact.like("exact string")` when `pact.like("any string")` is sufficient creates brittle contracts.
