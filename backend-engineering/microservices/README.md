# Microservices for Banking GenAI Platforms

## Why Microservices

A global bank's GenAI platform cannot be a monolith. The document analysis service has different scaling needs than the payment processing service. Regulatory requirements mandate isolation between customer data and transaction data. Microservices enable independent deployment, technology diversity, and fault isolation that monoliths cannot provide.

## Service Boundaries

### Bounded Context Design

```
Banking Domain Bounded Contexts:

┌─────────────────────────────────────────────────┐
│ Customer Context                                  │
│ - Customer profiles, KYC, identity               │
│ Owns: customers, identities, kyc_records         │
├─────────────────────────────────────────────────┤
│ Account Context                                   │
│ - Accounts, balances, statements                 │
│ Owns: accounts, balances, statements             │
├─────────────────────────────────────────────────┤
│ Payment Context                                   │
│ - Transfers, direct debits, standing orders       │
│ Owns: payments, payment_schedules                │
├─────────────────────────────────────────────────┤
│ Risk Context                                      │
│ - Fraud detection, risk scoring, AML              │
│ Owns: risk_scores, fraud_alerts, aml_flags       │
├─────────────────────────────────────────────────┤
│ GenAI Context                                     │
│ - Document analysis, embeddings, models           │
│ Owns: ai_jobs, model_versions, embeddings        │
├─────────────────────────────────────────────────┤
│ Notification Context                              │
│ - Alerts, emails, SMS, push notifications         │
│ Owns: notifications, templates, preferences      │
└─────────────────────────────────────────────────┘
```

### Service Ownership Rules

1. **Single ownership**: Each table/collection owned by exactly one service
2. **Private data**: No direct database access across service boundaries
3. **API contract**: All inter-service communication via published APIs
4. **Independent deployable**: Each service deploys without coordinating others

## Inter-Service Communication

### Synchronous: REST/gRPC

```python
# Account service calls Customer service for customer data
# REST approach
class AccountService:
    def __init__(self):
        self.customer_client = CustomerServiceClient(
            base_url='http://customer-service:8080',
            timeout=5,
            retry=Retry(total=3, backoff_factor=0.1),
        )

    def get_account_with_customer(self, account_id: str) -> dict:
        account = self.db.get_account(account_id)

        # Synchronous call to customer service
        customer = self.customer_client.get_customer(account.customer_id)

        return {
            **account.to_dict(),
            'customer': {
                'name': customer.name,
                'risk_level': customer.risk_level,
            },
        }
```

### Asynchronous: Events

```python
# Event-driven approach: no direct coupling
# Customer service publishes event, Account service consumes

# Customer service (producer)
def update_customer_risk_level(customer_id: str, new_level: str):
    customer.risk_level = new_level
    customer.save()

    # Publish event
    event_bus.publish('customer.risk_level_changed', {
        'customer_id': customer_id,
        'new_level': new_level,
        'timestamp': datetime.utcnow().isoformat(),
    })

# Account service (consumer)
def on_customer_risk_changed(event: dict):
    """Update local customer risk cache."""
    customer_id = event['customer_id']
    new_level = event['new_level']

    # Update local read model
    CustomerCache.objects.update_or_create(
        customer_id=customer_id,
        defaults={'risk_level': new_level},
    )

    # Check accounts for this customer
    accounts = Account.objects.filter(customer_id=customer_id)
    for account in accounts:
        if new_level == 'HIGH':
            account.add_flag('high_risk_customer')
```

### Communication Pattern Selection

| Pattern | When to Use | Latency | Reliability |
|---------|-----------|---------|-------------|
| REST sync | Real-time queries | 10-100ms | Medium |
| gRPC sync | Low-latency internal | 1-10ms | Medium |
| Async events | State synchronization | 100ms-1s | High |
| Async commands | Business workflows | 100ms-1s | High |

## Distributed Data Management

### Database per Service

```
Customer Service -> customer_db (PostgreSQL)
Account Service  -> account_db (PostgreSQL)
Payment Service  -> payment_db (PostgreSQL)
Risk Service     -> risk_db (PostgreSQL + Elasticsearch)
GenAI Service    -> genai_db (MongoDB + Redis)

# No shared databases
# Each service owns its data schema
# Cross-service queries via API or events
```

### Saga Pattern for Distributed Transactions

```python
# Payment transfer spans Account and Payment services
# Cannot use single database transaction

class PaymentTransferSaga:
    def __init__(self):
        self.account_client = AccountServiceClient()
        self.payment_client = PaymentServiceClient()

    def execute_transfer(self, from_account: str, to_account: str, amount: Decimal):
        """Execute transfer using saga pattern."""
        saga_id = generate_uuid()
        compensations = []

        try:
            # Step 1: Debit source account
            debit_result = self.account_client.debit(from_account, amount)
            compensations.append(lambda: self.account_client.credit(from_account, amount))

            # Step 2: Credit destination account
            credit_result = self.account_client.credit(to_account, amount)
            compensations.append(lambda: self.account_client.debit(to_account, amount))

            # Step 3: Create payment record
            payment_result = self.payment_client.create_payment({
                'from': from_account,
                'to': to_account,
                'amount': amount,
            })

            return {'status': 'success', 'payment_id': payment_result['id']}

        except Exception as e:
            # Compensate: undo completed steps in reverse order
            for compensation in reversed(compensations):
                try:
                    compensation()
                except Exception as comp_error:
                    # Compensation failed - manual intervention required
                    alert_oncall(f'Compensation failed for saga {saga_id}')
                    raise

            raise TransferFailedError(f'Transfer failed: {e}')
```

### Eventual Consistency

```python
# Account service maintains customer risk level as local cache
# Updated asynchronously via events

class CustomerRiskCache:
    """Local cache of customer risk level from Customer service."""

    def on_customer_risk_changed(self, event: dict):
        """Update local cache when event arrives."""
        customer_id = event['customer_id']
        new_risk = event['new_level']

        # Upsert into local cache
        self.db.execute(
            """
            INSERT INTO customer_risk_cache (customer_id, risk_level, updated_at)
            VALUES (?, ?, NOW())
            ON CONFLICT (customer_id)
            DO UPDATE SET risk_level = ?, updated_at = NOW()
            """,
            (customer_id, new_risk, new_risk),
        )

    def get_risk_level(self, customer_id: str) -> str:
        """Read from local cache (eventually consistent)."""
        result = self.db.execute(
            "SELECT risk_level FROM customer_risk_cache WHERE customer_id = ?",
            (customer_id,),
        )
        return result[0]['risk_level'] if result else 'UNKNOWN'
```

## Service Discovery and API Gateway

### Service Registry

```python
# Services register on startup
class ServiceRegistry:
    def register(self, service_name: str, host: str, port: int):
        """Register service instance."""
        consul_client.kv.put(
            f'services/{service_name}/{host}:{port}',
            json.dumps({
                'host': host,
                'port': port,
                'registered_at': datetime.utcnow().isoformat(),
            }),
        )

    def deregister(self, service_name: str, host: str, port: int):
        """Remove service instance on shutdown."""
        consul_client.kv.delete(f'services/{service_name}/{host}:{port}')

    def get_service_instances(self, service_name: str) -> list[dict]:
        """Get all instances of a service."""
        _, nodes = consul_client.kv.get(f'services/{service_name}/', recurse=True)
        return [json.loads(node['Value']) for node in nodes if node['Value']]
```

### API Gateway Pattern

```python
# Gateway routes requests to appropriate service
# Handles authentication, rate limiting, logging

routes = {
    # Customer service routes
    'GET /v1/customers/{id}': 'customer-service',
    'POST /v1/customers': 'customer-service',

    # Account service routes
    'GET /v1/accounts/{id}': 'account-service',
    'GET /v1/accounts/{id}/transactions': 'account-service',

    # Payment service routes
    'POST /v1/payments': 'payment-service',
    'GET /v1/payments/{id}': 'payment-service',

    # GenAI service routes
    'POST /v1/ai/chat': 'genai-service',
    'POST /v1/ai/analyze': 'genai-service',
}

# Gateway middleware pipeline
middleware = [
    AuthenticationMiddleware,      # Validate JWT token
    RateLimitMiddleware,           # Enforce rate limits
    LoggingMiddleware,             # Structured request logging
    CorrelationIdMiddleware,       # Propagate request ID
    RoutingMiddleware,             # Route to backend service
]
```

## GenAI Service Integration

### Model Service Architecture

```
┌─────────────────────────────────────────┐
│ API Gateway                              │
└──────────┬──────────────────────────────┘
           │
    ┌──────┴──────┐
    │             │
┌───┴────┐  ┌────┴────┐
│ Chat   │  │ Document│
│ Service│  │ Analysis│
│        │  │ Service │
└───┬────┘  └────┬────┘
    │             │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │ Model       │
    │ Orchestration│
    │ Service     │
    └──────┬──────┘
           │
    ┌──────┴──────┬────────────┐
    │             │            │
┌───┴────┐  ┌────┴────┐  ┌───┴────┐
│ Model A│  │ Model B │  │ Model C│
│ (LLM)  │  │ (Embed) │  │ (Class)│
└────────┘  └─────────┘  └────────┘
```

### Model Deployment via Events

```python
# Model registry publishes deployment events
# All services that use the model subscribe and update

def on_model_deployed(event: dict):
    """Update local model client when new model is deployed."""
    model_id = event['model_id']
    model_version = event['version']
    endpoint = event['endpoint']

    # Update model routing configuration
    model_router.update_route(model_id, model_version, endpoint)

    # Warm up connection to new model endpoint
    model_client.warmup(endpoint)

    # Update cache with new model version
    cache.set(f'model:{model_id}:version', model_version)

    # Log deployment
    logger.info(
        'Model deployed',
        extra={
            'model_id': model_id,
            'version': model_version,
            'endpoint': endpoint,
        },
    )
```

## Resilience in Microservices

### Timeout Configuration

```python
# Timeout hierarchy for service calls
TIMEOUTS = {
    'customer-service': {
        'connect': 2,    # 2 seconds to establish connection
        'read': 5,       # 5 seconds for response
        'total': 10,     # 10 seconds end-to-end
    },
    'payment-service': {
        'connect': 2,
        'read': 10,      # Payments take longer
        'total': 30,
    },
    'genai-service': {
        'connect': 5,
        'read': 60,      # AI inference can be slow
        'total': 120,
    },
}
```

### Bulkhead Pattern

```python
# Isolate service calls to prevent cascading failures
from threading import Semaphore

class BulkheadClient:
    def __init__(self, max_concurrent: int = 10):
        self.semaphore = Semaphore(max_concurrent)

    def call(self, service_name: str, request: dict) -> dict:
        if not self.semaphore.acquire(blocking=False):
            raise ServiceOverloadedError(f'{service_name} bulkhead full')

        try:
            return self._make_request(service_name, request)
        finally:
            self.semaphore.release()
```

## Testing Strategies

### Contract Testing

```python
# Pact contract between Account and Customer services
import pact

class AccountCustomerContract:
    """Define expected interaction between services."""

    @pact.consumer('AccountService')
    @pact.provider('CustomerService')
    def test_get_customer(self, pact):
        (pact
         .given('customer exists')
         .upon_receiving('a request for customer details')
         .with_request('GET', '/v1/customers/CUST-001')
         .will_respond_with(200, body={
             'id': 'CUST-001',
             'name': 'John Doe',
             'risk_level': 'LOW',
         }))

        # Test the interaction
        customer = self.customer_client.get_customer('CUST-001')
        assert customer['name'] == 'John Doe'
        assert customer['risk_level'] == 'LOW'
```

## Common Mistakes

1. **Shared database**: Multiple services accessing same tables
2. **Distributed monolith**: All services must deploy together
3. **Chatty calls**: 10+ REST calls to fulfill one request
4. **No saga compensation**: Partial failures leave data inconsistent
5. **Ignoring network latency**: Assuming inter-service calls are fast
6. **Missing correlation IDs**: Cannot trace requests across services
7. **Tight version coupling**: Service A v2 only works with Service B v2

## Interview Questions

1. How do you maintain data consistency across services without distributed transactions?
2. Design a payment transfer that spans Account, Payment, and Notification services.
3. How do you handle schema evolution when Service A changes its API?
4. What happens when the Customer service is down but the Account service needs customer data?
5. How do you test a microservice in isolation when it depends on 5 other services?

## Cross-References

- [[distributed-systems/README.md]] - Distributed consistency patterns
- [[resilience-patterns/README.md]] - Circuit breaker, retry for service calls
- [[queues-and-events/README.md]] - Event-driven inter-service communication
- [[api-design/README.md]] - API gateway and routing
- `../python/fastapi.md` - FastAPI microservice implementation
