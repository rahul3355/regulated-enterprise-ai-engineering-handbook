# Integration Testing for Banking GenAI

## What Is Integration Testing?

Integration tests verify that multiple components work together correctly. Unlike unit tests (which isolate individual functions), integration tests exercise the boundaries between components: service-to-database, service-to-service, service-to-external-API.

## When to Use Integration Tests

```
┌─────────────────────────────────────────────────────────────┐
│  TEST TYPE vs WHAT IT VERIFIES                               │
├──────────────────────────┬──────────────────────────────────┤
│  Unit Test               │  Single function behaves correctly│
│  Integration Test        │  Components interact correctly   │
│  E2E Test               │  Complete user flow works         │
│  LLM Evaluation         │  AI output quality is acceptable  │
└──────────────────────────┴──────────────────────────────────┘
```

Use integration tests for:
- Database queries with real schema
- RAG pipeline (embedding + retrieval + reranking)
- API endpoint handlers with real request/response
- LLM gateway with mock provider
- Authentication flows with test identity provider

## Test Containers

Use Docker containers to run real dependencies in tests:

```python
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from sqlalchemy import create_engine

@pytest.fixture(scope="session")
def postgres_container():
    """Start a PostgreSQL container for testing."""
    with PostgresContainer("postgres:16") as postgres:
        yield postgres.get_connection_url()

@pytest.fixture
def db_engine(postgres_container):
    """Create database engine and run migrations."""
    engine = create_engine(postgres_container)
    run_migrations(engine)  # Apply schema
    yield engine
    engine.dispose()

def test_mortgage_application_stored_and_retrieved(db_engine):
    """Verify mortgage application can be stored and retrieved."""
    repo = MortgageRepository(db_engine)

    application = MortgageApplication(
        applicant_income=95_000,
        loan_amount=450_000,
        property_value=500_000,
        credit_score=720,
    )

    saved_id = repo.save(application)
    retrieved = repo.get_by_id(saved_id)

    assert retrieved.applicant_income == application.applicant_income
    assert retrieved.loan_amount == application.loan_amount
```

## Testing the RAG Pipeline

```python
@pytest.fixture
def vector_db_container():
    """Start a pgvector container for testing."""
    with PostgresContainer("pgvector/pgvector:pg16") as pg:
        url = pg.get_connection_url()
        engine = create_engine(url)
        setup_pgvector(engine)  # Create vector extension and tables
        yield engine

@pytest.fixture
def embedding_service_mock():
    """Mock embedding service with deterministic output."""
    with patch('src.embedding.OpenAIEmbedding') as mock:
        mock.return_value.encode.return_value = [0.1] * 3072  # Fixed vector
        yield mock

def test_rag_retrieval_finds_relevant_documents(vector_db_container, embedding_service_mock):
    """Verify RAG retrieval returns relevant documents."""
    # Insert test documents
    doc_store = DocumentStore(vector_db_container)
    doc_store.add(Document(
        content="The current 30-year fixed mortgage rate is 6.5% APR.",
        metadata={"source": "rate_sheet", "date": "2025-03-01"}
    ))
    doc_store.add(Document(
        content="Minimum credit score for conventional mortgage is 620.",
        metadata={"source": "policy", "section": "eligibility"}
    ))

    # Query
    rag = RAGPipeline(
        embedder=embedding_service_mock,
        vector_store=vector_db_container,
        top_k=5,
    )

    results = rag.retrieve("What is the current mortgage rate?")

    # The rate sheet document should be in top results
    assert any("6.5%" in doc.content for doc in results)
    assert len(results) >= 1

def test_rag_retrieval_returns_empty_for_irrelevant_query(vector_db_container, embedding_service_mock):
    """Verify RAG returns empty results for unrelated queries."""
    rag = RAGPipeline(
        embedder=embedding_service_mock,
        vector_store=vector_db_container,
        top_k=5,
        similarity_threshold=0.7,
    )

    results = rag.retrieve("What is the capital of France?")

    # Should return no results (no banking docs about France)
    assert len(results) == 0 or all(
        doc.similarity < 0.7 for doc in results
    )
```

## Testing the LLM Gateway

```python
@pytest.fixture
def mock_llm_provider():
    """Mock LLM provider with controlled responses."""
    server = MockLLMServer(
        responses={
            "default": "This is a default test response.",
            "mortgage_rates": json.dumps({
                "recommendation": "Based on current rates, a 30-year fixed at 6.5% is available.",
                "disclaimer": "This is for informational purposes only.",
                "rate": 0.065,
            }),
            "hallucination_test": json.dumps({
                "claim": "The mortgage rate is 2.5%",  # Intentionally wrong
                "source": "unknown",
            }),
        }
    )
    with server:
        yield server.endpoint

def test_llm_gateway_returns_parsed_response(mock_llm_provider):
    """Verify LLM gateway calls provider and parses response."""
    gateway = LLMGateway(
        provider_url=mock_llm_provider,
        model="gpt-4-turbo",
    )

    response = gateway.call(
        prompt="What are the current mortgage rates?",
        response_schema=MortgageAdviceSchema,
    )

    assert response.parsed["rate"] == pytest.approx(0.065)
    assert "disclaimer" in response.parsed
    assert response.tokens_prompt > 0
    assert response.tokens_completion > 0

def test_llm_gateway_handles_provider_error():
    """Verify LLM gateway handles provider failures gracefully."""
    gateway = LLMGateway(
        provider_url="http://nonexistent-provider:9999",
        model="gpt-4-turbo",
        timeout=5,
    )

    with pytest.raises(ProviderError):
        gateway.call(prompt="Test")
```

## Testing API Endpoints

```python
from fastapi.testclient import TestClient
from src.app import app
from src.database import get_db

@pytest.fixture
def client(db_engine):
    """Create test client with test database."""
    def override_get_db():
        db = SessionLocal(bind=db_engine)
        try:
            yield db
        finally:
            db.close()

    app.dependency_overrides[get_db] = override_get_db

    with TestClient(app) as test_client:
        yield test_client

    app.dependency_overrides.clear()

def test_mortgage_advice_endpoint(client):
    """Test the mortgage advice API endpoint."""
    response = client.post(
        "/api/v1/mortgage-advice",
        json={
            "salary": 95_000,
            "loan_amount": 450_000,
            "term_years": 30,
            "credit_score": 720,
        },
        headers={"Authorization": "Bearer test-token"},
    )

    assert response.status_code == 200
    data = response.json()
    assert "recommendation" in data
    assert "disclaimer" in data
    assert "monthly_payment" in data
    assert data["monthly_payment"] == pytest.approx(2844.52, rel=0.01)

def test_mortgage_advice_requires_auth(client):
    """Unauthenticated requests should be rejected."""
    response = client.post(
        "/api/v1/mortgage-advice",
        json={"salary": 95_000, "loan_amount": 450_000},
    )
    assert response.status_code in [401, 403]

def test_mortgage_advice_validates_input(client):
    """Invalid input should return 400."""
    response = client.post(
        "/api/v1/mortgage-advice",
        json={
            "salary": -1000,  # Invalid: negative salary
            "loan_amount": 450_000,
        },
        headers={"Authorization": "Bearer test-token"},
    )
    assert response.status_code == 422  # Validation error
```

## Integration Test CI Configuration

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  integration-tests:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements-test.txt

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379/0
        run: |
          pytest tests/integration/ \
            --cov=src \
            --cov-report=xml \
            -v \
            --tb=short
```

## Common Integration Testing Mistakes

1. **Using production databases**: Never run tests against production. Use test containers or a dedicated test database.

2. **Shared test data between tests**: Tests that depend on each other's data are flaky. Each test should set up its own data.

3. **Not cleaning up**: Test containers are automatically cleaned up. But if you use a shared test database, you must clean up after each test.

4. **Testing everything through the API**: Integration tests should exercise component boundaries, not every possible API combination. Use unit tests for fine-grained behavior.

5. **Slow tests**: Integration tests are slower than unit tests. Keep them focused. A CI pipeline with 500 integration tests should complete in under 10 minutes.

6. **No environment parity**: Test containers should use the same database version as production. Testing against Postgres 14 when production runs Postgres 16 misses real issues.
