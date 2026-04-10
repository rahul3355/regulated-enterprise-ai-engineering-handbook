# Pytest Testing Patterns

## Why Pytest

Pytest is Python's most powerful testing framework. It provides fixtures for dependency injection, parameterized tests, plugins for async testing, and rich assertion rewriting. For banking services, pytest ensures comprehensive test coverage with minimal boilerplate.

## Test Structure

```
tests/
├── conftest.py              # Shared fixtures
├── factories.py             # Test data factories
├── test_api/
│   ├── test_accounts.py
│   └── test_payments.py
├── test_services/
│   ├── test_account_service.py
│   └── test_payment_service.py
├── test_repositories/
│   └── test_account_repo.py
└── test_integration/
    └── test_payment_flow.py
```

## Fixtures

```python
# conftest.py
import pytest
from decimal import Decimal
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

from banking_api.main import app
from banking_api.models import Base
from banking_api.models.account import Account, AccountStatus


@pytest.fixture(scope='session')
def event_loop():
    """Create a single event loop for the session."""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope='session')
async def test_engine():
    """Create test database engine."""
    engine = create_async_engine(
        'postgresql+asyncpg://test:test@localhost:5432/banking_test',
        echo=False,
    )

    # Create tables
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    # Cleanup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture
async def db_session(test_engine):
    """Create a database session with automatic rollback."""
    async with test_engine.connect() as conn:
        async with conn.begin() as transaction:
            session = AsyncSession(conn)

            yield session

            # Rollback after test (clean state for next test)
            await session.close()
            await transaction.rollback()


@pytest.fixture
async def client(db_session):
    """Create async test client with test database."""
    # Override database dependency
    async def override_get_db():
        yield db_session

    app.dependency_overrides['get_db_session'] = override_get_db

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url='http://test') as ac:
        yield ac


@pytest.fixture
def account_factory(db_session):
    """Factory for creating test accounts."""
    async def _create(
        customer_id: str = None,
        balance: Decimal = Decimal('1000.00'),
        status: AccountStatus = AccountStatus.ACTIVE,
    ):
        account = Account(
            customer_id=customer_id or str(uuid.uuid4()),
            account_number=f'TEST{uuid.uuid4().hex[:10]}',
            account_type='checking',
            currency='USD',
            balance=balance,
            status=status,
        )
        db_session.add(account)
        await db_session.flush()
        return account

    return _create
```

## Basic Tests

```python
# test_api/test_accounts.py
import pytest
from decimal import Decimal


class TestAccountAPI:
    """Test account API endpoints."""

    async def test_get_account_success(self, client, account_factory):
        """Test successful account retrieval."""
        # Arrange
        account = await account_factory()

        # Act
        response = await client.get(f'/api/v1/accounts/{account.id}')

        # Assert
        assert response.status_code == 200
        data = response.json()
        assert data['id'] == str(account.id)
        assert data['balance'] == '1000.00'
        assert data['currency'] == 'USD'
        assert data['status'] == 'active'

    async def test_get_account_not_found(self, client):
        """Test account not found returns 404."""
        response = await client.get('/api/v1/accounts/nonexistent-id')
        assert response.status_code == 404
        data = response.json()
        assert data['error']['code'] == 'NOT_FOUND'

    async def test_create_account_validation_error(self, client):
        """Test invalid account data returns 422."""
        response = await client.post('/api/v1/accounts', json={
            'customer_id': 'invalid',  # Missing required fields
        })
        assert response.status_code == 422
```

## Parameterized Tests

```python
class TestPaymentValidation:
    """Test payment validation with various inputs."""

    @pytest.mark.parametrize('amount,expected_status', [
        (Decimal('0.01'), 200),
        (Decimal('100.00'), 200),
        (Decimal('999999.99'), 200),
        (Decimal('0.00'), 422),       # Zero not allowed
        (Decimal('-1.00'), 422),      # Negative not allowed
        (Decimal('1000000.01'), 422), # Exceeds max
    ])
    async def test_payment_amount_validation(
        self, client, account_factory, amount, expected_status,
    ):
        """Test payment amount validation."""
        account = await account_factory()

        response = await client.post('/api/v1/payments', json={
            'from_account': str(account.id),
            'to_account': 'ACC-002',
            'amount': str(amount),
            'currency': 'USD',
        })

        assert response.status_code == expected_status


    @pytest.mark.parametrize('currency', ['USD', 'EUR', 'GBP', 'JPY'])
    async def test_supported_currencies(self, client, currency):
        """Test all supported currencies are accepted."""
        response = await client.post('/api/v1/payments', json={
            'from_account': 'ACC-001',
            'to_account': 'ACC-002',
            'amount': '100.00',
            'currency': currency,
        })
        assert response.status_code in [200, 201, 422]  # Depends on auth
```

## Async Tests

```python
import pytest
import pytest_asyncio


@pytest_asyncio.fixture
async def ai_service():
    """Create AI service mock."""
    service = MockAIService()
    yield service


class TestDocumentAnalysis:
    """Test AI document analysis."""

    @pytest.mark.asyncio
    async def test_analyze_kyc_document(self, client, ai_service):
        """Test KYC document analysis."""
        # Mock AI response
        ai_service.mock_response({
            'document_type': 'passport',
            'extracted_fields': {
                'name': 'John Doe',
                'nationality': 'US',
                'document_number': '123456789',
            },
            'confidence_scores': {
                'name': 0.98,
                'nationality': 0.99,
            },
        })

        # Upload document
        with open('tests/fixtures/sample_kyc.pdf', 'rb') as f:
            response = await client.post(
                '/api/v1/documents/analyze',
                files={'file': ('sample_kyc.pdf', f, 'application/pdf')},
                data={'analysis_type': 'kyc_extraction'},
            )

        assert response.status_code == 200
        data = response.json()
        assert data['document_type'] == 'passport'
        assert data['extracted_fields']['name'] == 'John Doe'

    @pytest.mark.asyncio
    async def test_analyze_document_ai_timeout(self, client, ai_service):
        """Test graceful handling of AI service timeout."""
        ai_service.mock_timeout()

        with open('tests/fixtures/sample_doc.pdf', 'rb') as f:
            response = await client.post(
                '/api/v1/documents/analyze',
                files={'file': ('sample.pdf', f, 'application/pdf')},
                data={'analysis_type': 'kyc'},
            )

        assert response.status_code == 503  # Service unavailable
```

## Testing with Mocks

```python
from unittest.mock import AsyncMock, patch, MagicMock


class TestPaymentService:
    """Test payment service with mocked dependencies."""

    @pytest.fixture
    def account_repo(self):
        return AsyncMock()

    @pytest.fixture
    def notification_service(self):
        return AsyncMock()

    @pytest.fixture
    def payment_service(self, account_repo, notification_service):
        return PaymentService(account_repo, notification_service)

    @pytest.mark.asyncio
    async def test_transfer_success(
        self, payment_service, account_repo, notification_service,
    ):
        """Test successful transfer."""
        # Arrange
        from_account = Account(balance=Decimal('1000.00'))
        to_account = Account(balance=Decimal('500.00'))

        account_repo.get_by_id.side_effect = [from_account, to_account]

        # Act
        result = await payment_service.transfer(
            from_account='ACC-001',
            to_account='ACC-002',
            amount=Decimal('200.00'),
        )

        # Assert
        assert result['status'] == 'completed'
        assert from_account.balance == Decimal('800.00')
        assert to_account.balance == Decimal('700.00')

        # Verify notifications sent
        notification_service.send.assert_any_call(
            account_id='ACC-001',
            type='debit',
            amount=Decimal('200.00'),
        )

    @pytest.mark.asyncio
    async def test_transfer_insufficient_funds(
        self, payment_service, account_repo,
    ):
        """Test transfer with insufficient funds."""
        # Arrange
        from_account = Account(balance=Decimal('100.00'))
        account_repo.get_by_id.return_value = from_account

        # Act & Assert
        with pytest.raises(InsufficientFundsError):
            await payment_service.transfer(
                from_account='ACC-001',
                to_account='ACC-002',
                amount=Decimal('200.00'),
            )
```

## Property-Based Testing with Hypothesis

```python
from hypothesis import given, strategies as st
from decimal import Decimal
import pytest


class TestInterestCalculation:
    """Property-based tests for interest calculation."""

    @given(
        principal=st.decimals(
            min_value='0.01',
            max_value='10000000',
            places=2,
        ),
        rate=st.decimals(
            min_value='0.001',
            max_value='0.20',
            places=4,
        ),
        days=st.integers(min_value=1, max_value=365),
    )
    def test_interest_is_non_negative(self, principal, rate, days):
        """Interest calculation always produces non-negative result."""
        interest = calculate_interest(principal, rate, days)
        assert interest >= 0

    @given(
        principal=st.decimals(min_value='1', max_value='1000000', places=2),
        rate=st.decimals(min_value='0.01', max_value='0.10', places=4),
    )
    def test_higher_rate_produces_more_interest(self, principal, rate):
        """Higher interest rate produces more interest."""
        interest_low = calculate_interest(principal, rate, 365)
        interest_high = calculate_interest(principal, rate * 2, 365)
        assert interest_high > interest_low
```

## Test Coverage

```bash
# Run tests with coverage
pytest tests/ \
    --cov=banking_api \
    --cov-report=html \
    --cov-report=term-missing \
    --cov-fail-under=90

# Coverage configuration in pyproject.toml
[tool.coverage.run]
source = ['banking_api']
omit = ['*/tests/*', '*/migrations/*']

[tool.coverage.report]
fail_under = 90
show_missing = true
exclude_lines = [
    'pragma: no cover',
    'if TYPE_CHECKING:',
    'if __name__ == .__main__:',
]
```

## Common Mistakes

1. **Shared test database**: Tests interfere with each other
2. **No fixtures**: Repeating setup code in every test
3. **Testing implementation**: Tests break on refactoring
4. **Ignoring async tests**: Using sync test runner for async code
5. **No parameterized tests**: Writing nearly identical tests manually
6. **Missing property-based tests**: Edge cases found only by hypothesis
7. **Slow tests**: Test suite takes 30+ minutes, developers skip running

## Interview Questions

1. How do you test a FastAPI endpoint that depends on external services?
2. Design fixtures for testing a payment flow across multiple services.
3. How do you test idempotency for a POST endpoint?
4. What properties would you test for a compound interest calculation?
5. How do you ensure test isolation when tests share a database?

## Cross-References

- `./fastapi.md` - FastAPI testing with TestClient
- `./sqlalchemy.md` - Testing with test databases
- `../backend-testing/README.md` - Overall testing strategy
- `../api-design/README.md` - Testing API contracts
