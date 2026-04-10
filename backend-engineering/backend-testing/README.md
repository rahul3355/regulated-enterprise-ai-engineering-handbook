# Backend Testing for Banking GenAI Platforms

## Why Testing Matters

In banking, bugs cause financial losses, regulatory violations, and reputation damage. A rounding error in interest calculation costs millions. A race condition in payment processing creates duplicate transfers. A broken KYC check allows money laundering. Comprehensive testing is not optional -- it is a regulatory requirement.

## Testing Pyramid

```
        ┌─────────────┐
        │    E2E      │  Few tests, high confidence, slow
        ├─────────────┤
        │ Integration │  Medium tests, service interactions
        ├─────────────┤
        │   Contract  │  API contract between services
        ├─────────────┤
        │    Unit     │  Many tests, fast, isolated
        └─────────────┘

Banking testing distribution:
- Unit tests:        70% of tests, < 1s each
- Integration tests:  20% of tests, < 10s each
- Contract tests:     5% of tests, < 30s each
- E2E tests:          5% of tests, < 60s each
```

## Unit Testing

### Test Structure (Arrange-Act-Assert)

```python
import pytest
from decimal import Decimal
from banking.accounts import Account, TransferService

class TestAccount:
    """Unit tests for Account domain model."""

    def test_deposit_increases_balance(self):
        # Arrange
        account = Account(id='ACC-001', balance=Decimal('1000.00'))

        # Act
        account.deposit(Decimal('500.00'))

        # Assert
        assert account.balance == Decimal('1500.00')

    def test_withdrawal_decreases_balance(self):
        # Arrange
        account = Account(id='ACC-001', balance=Decimal('1000.00'))

        # Act
        account.withdraw(Decimal('300.00'))

        # Assert
        assert account.balance == Decimal('700.00')

    def test_withdrawal_insufficient_funds_raises_error(self):
        # Arrange
        account = Account(id='ACC-001', balance=Decimal('100.00'))

        # Act & Assert
        with pytest.raises(InsufficientFundsError):
            account.withdraw(Decimal('200.00'))

    def test_account_closure_requires_zero_balance(self):
        # Arrange
        account = Account(id='ACC-001', balance=Decimal('50.00'))

        # Act & Assert
        with pytest.raises(AccountNotCloseableError):
            account.close()

        # Arrange - satisfy precondition
        account.withdraw(Decimal('50.00'))

        # Act
        account.close()

        # Assert
        assert account.status == 'closed'
```

### Testing Edge Cases

```python
class TestPaymentValidation:
    """Edge cases in payment validation."""

    @pytest.mark.parametrize('amount', [
        Decimal('0.00'),
        Decimal('-1.00'),
        Decimal('0.001'),  # Sub-cent amount
        Decimal('999999999.99'),  # Very large amount
    ])
    def test_invalid_amounts_rejected(self, amount):
        payment = Payment(amount=amount, currency='USD', to='ACC-002')
        with pytest.raises(InvalidPaymentError):
            validate_payment(payment)

    def test_payment_to_self_rejected(self):
        payment = Payment(
            amount=Decimal('100.00'),
            currency='USD',
            from_account='ACC-001',
            to_account='ACC-001',  # Same account
        )
        with pytest.raises(InvalidPaymentError, match='cannot pay self'):
            validate_payment(payment)

    def test_payment_with_unicode_reference_accepted(self):
        payment = Payment(
            amount=Decimal('100.00'),
            currency='USD',
            reference='Payment for \u00e9l\u00e8ctricity',
        )
        result = validate_payment(payment)
        assert result.is_valid
```

## Integration Testing

### Database Integration Tests

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def db_session():
    """Create isolated test database."""
    engine = create_engine('postgresql://test:test@localhost:5432/banking_test')

    # Create tables
    Base.metadata.create_all(engine)

    # Create session with transaction rollback
    Session = sessionmaker(bind=engine)
    session = Session()

    # Wrap in transaction and rollback after test
    transaction = engine.begin()
    session.begin()

    yield session

    # Cleanup
    session.rollback()
    transaction.rollback()
    Base.metadata.drop_all(engine)


class TestAccountRepository:
    """Integration tests with real database."""

    def test_create_and_retrieve_account(self, db_session):
        repo = AccountRepository(db_session)
        account = repo.create(
            customer_id='CUST-001',
            account_type='checking',
            currency='USD',
        )

        retrieved = repo.get_by_id(account.id)
        assert retrieved.customer_id == 'CUST-001'
        assert retrieved.account_type == 'checking'

    def test_concurrent_balance_updates(self, db_session):
        """Test concurrent updates don't lose money."""
        repo = AccountRepository(db_session)
        account = repo.create(customer_id='CUST-001', balance=Decimal('1000.00'))

        # Simulate concurrent updates
        def deposit():
            for _ in range(100):
                account.deposit(Decimal('10.00'))
                db_session.commit()

        threads = [threading.Thread(target=deposit) for _ in range(5)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()

        # Verify no money lost
        final_account = repo.get_by_id(account.id)
        assert final_account.balance == Decimal('6000.00')  # 1000 + 5 * 100 * 10
```

### External Service Integration Tests

```python
import pytest
from unittest.mock import patch, MagicMock

class TestExternalPaymentGateway:
    """Integration tests with mocked external service."""

    @pytest.fixture
    def gateway(self):
        return ExternalPaymentGateway(
            api_key='test-key',
            base_url='https://sandbox.gateway.com',
        )

    @patch('requests.post')
    def test_successful_payment(self, mock_post, gateway):
        mock_post.return_value = MagicMock(
            status_code=200,
            json=MagicMock(return_value={
                'payment_id': 'EXT-PAY-001',
                'status': 'accepted',
            }),
        )

        result = gateway.initiate_payment(
            amount=Decimal('100.00'),
            currency='USD',
            recipient='GB29NWBK60161331926819',
        )

        assert result.status == 'accepted'
        assert result.payment_id == 'EXT-PAY-001'

    @patch('requests.post')
    def test_timeout_handling(self, mock_post, gateway):
        mock_post.side_effect = requests.exceptions.Timeout()

        with pytest.raises(PaymentGatewayTimeoutError):
            gateway.initiate_payment(
                amount=Decimal('100.00'),
                currency='USD',
                recipient='GB29NWBK60161331926819',
            )
```

## Contract Testing

### Pact Contract Tests

```python
import pytest
from pact import Consumer, Provider

# Define contract
pact = Consumer('AccountService').has_pact_with(Provider('CustomerService'))
pact.start_service()

class TestCustomerServiceContract:
    """Contract test between Account and Customer services."""

    def test_get_customer_by_id(self):
        # Define expected interaction
        (pact
         .given('customer CUST-001 exists')
         .upon_receiving('a request for customer CUST-001')
         .with_request('GET', '/v1/customers/CUST-001')
         .will_respond_with(200, body={
             'id': 'CUST-001',
             'name': 'John Doe',
             'risk_level': 'LOW',
             'kyc_status': 'verified',
         }))

        # Make actual request to mock provider
        response = requests.get(f'{pact.uri}/v1/customers/CUST-001')

        assert response.status_code == 200
        data = response.json()
        assert data['id'] == 'CUST-001'
        assert data['risk_level'] in ['LOW', 'MEDIUM', 'HIGH']

    def tearDown(self):
        pact.stop_service()
```

### OpenAPI Schema Validation

```python
from openapi_spec_validator import validate_spec
import yaml

class TestAPISchema:
    """Validate API responses match OpenAPI schema."""

    @pytest.fixture
    def schema(self):
        with open('api/openapi.yaml') as f:
            return yaml.safe_load(f)

    def test_get_account_response_matches_schema(self, schema):
        """Validate actual response against schema."""
        account = get_account_from_api()

        # Validate against schema
        account_schema = schema['components']['schemas']['Account']
        validate_response_against_schema(account, account_schema)

    def test_error_response_matches_schema(self, schema):
        """Validate error responses match error schema."""
        error_response = call_api_for_nonexistent_account()

        error_schema = schema['components']['schemas']['Error']
        validate_response_against_schema(error_response, error_schema)
        assert 'code' in error_response['error']
        assert 'message' in error_response['error']
```

## Property-Based Testing

### Hypothesis for Banking Logic

```python
from hypothesis import given, strategies as st
from decimal import Decimal
import hypothesis.strategies as st

class TestTransferInvariants:
    """Property-based tests for transfer logic."""

    @given(
        from_balance=st.decimals(min_value='0', max_value='1000000', places=2),
        to_balance=st.decimals(min_value='0', max_value='1000000', places=2),
        amount=st.decimals(min_value='0.01', max_value='1000000', places=2),
    )
    def test_total_balance_preserved(self, from_balance, to_balance, amount):
        """Total balance before and after transfer must be equal."""
        if amount > from_balance:
            with pytest.raises(InsufficientFundsError):
                execute_transfer(from_balance, to_balance, amount)
        else:
            new_from, new_to = execute_transfer(from_balance, to_balance, amount)
            assert new_from + new_to == from_balance + to_balance

    @given(
        amount=st.decimals(min_value='0.01', max_value='1000000', places=2),
    )
    def test_transfer_amount_exact(self, amount):
        """Transferred amount must be exactly as requested."""
        from_balance = Decimal('10000.00')
        to_balance = Decimal('5000.00')

        new_from, new_to = execute_transfer(from_balance, to_balance, amount)

        assert from_balance - new_from == amount
        assert new_to - to_balance == amount
```

## Testing GenAI Components

### Testing AI Service Integration

```python
class TestGenAIService:
    """Tests for GenAI service integration."""

    @pytest.fixture
    def ai_service(self):
        return GenAIService(api_key='test-key')

    @pytest.mark.asyncio
    async def test_document_analysis(self, ai_service):
        """Test document analysis returns expected structure."""
        result = await ai_service.analyze_document(
            document_url='s3://test-docs/kyc-form.pdf',
            analysis_type='kyc_extraction',
        )

        assert 'extracted_fields' in result
        assert 'customer_name' in result['extracted_fields']
        assert 'confidence_scores' in result
        assert all(0 <= v <= 1 for v in result['confidence_scores'].values())

    @pytest.mark.asyncio
    async def test_ai_timeout_handling(self, ai_service):
        """Test graceful handling of AI service timeout."""
        with pytest.raises(AIInferenceTimeoutError):
            await ai_service.analyze_document(
                document_url='s3://test-docs/large-document.pdf',
                timeout=5,  # Very short timeout
            )

    def test_embedding_dimension_consistency(self, ai_service):
        """All embeddings must have same dimension."""
        texts = ['Hello world', 'Banking regulations', 'Risk assessment']
        embeddings = ai_service.generate_embeddings(texts)

        assert len(embeddings) == len(texts)
        dimensions = [len(emb) for emb in embeddings]
        assert len(set(dimensions)) == 1  # All same dimension
```

## Test Data Management

### Test Data Factories

```python
import factory

class AccountFactory(factory.Factory):
    class Meta:
        model = Account

    id = factory.Sequence(lambda n: f'ACC-{n:04d}')
    customer_id = factory.Sequence(lambda n: f'CUST-{n:04d}')
    account_type = 'checking'
    currency = 'USD'
    balance = Decimal('1000.00')
    status = 'active'

class PaymentFactory(factory.Factory):
    class Meta:
        model = Payment

    id = factory.Sequence(lambda n: f'PAY-{n:06d}')
    from_account = factory.SubFactory(AccountFactory)
    to_account = factory.SubFactory(AccountFactory)
    amount = Decimal('100.00')
    currency = 'USD'
    status = 'pending'
    reference = factory.Faker('sentence')

# Usage
account = AccountFactory(balance=Decimal('5000.00'))
payment = PaymentFactory(amount=Decimal('250.00'), from_account=account)
```

## Common Mistakes

1. **Testing implementation, not behavior**: Tests break on refactoring
2. **No isolation**: Tests depend on each other's state
3. **Flaky tests**: Tests fail intermittently due to timing
4. **Over-mocking**: Mocking everything tests nothing real
5. **No property-based tests**: Missing edge cases that hypothesis finds
6. **Testing against production data**: PII leakage, test data corruption
7. **Ignoring test performance**: Test suite takes 30+ minutes

## Interview Questions

1. How do you test a payment transfer that spans multiple services?
2. Design tests for a KYC verification service that calls external APIs.
3. How do you ensure test data doesn't contain production PII?
4. What properties would you test for an interest calculation function?
5. How do you test distributed transactions (saga pattern)?

## Cross-References

- [[api-design/README.md]] - API validation and error handling
- [[resilience-patterns/README.md]] - Testing circuit breaker and retry behavior
- [[microservices/README.md]] - Contract testing between services
- `../python/pytest.md` - Python-specific testing patterns
- `../go/testing.md` - Go table-driven tests
- `../java/testing.md` - JUnit 5 and Mockito patterns
