# Python Backend Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | Python Backend (FastAPI, Pydantic, SQLAlchemy, Celery, async, pytest) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Citi Relevance | Citi Assist backend uses Python with FastAPI for GenAI services. Async processing, Pydantic validation, SQLAlchemy ORM, and Celery task queues are core to the platform. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

## 🔵 Must-Know Questions (Q1-Q5)

### 🔵 Q1: How do you handle decimal precision in Python for banking financial calculations, and why is it critical?

**Strong Answer:**

In Python, `float` uses binary floating-point representation which introduces precision errors. For example, `0.1 + 0.2` evaluates to `0.30000000000000004`, not `0.3`. In a banking context with 500,000+ employees processing millions of transactions, even microscopic rounding errors compound into material financial discrepancies that would fail audit requirements.

The solution is to always use Python's `Decimal` type from the `decimal` module for all monetary calculations. `Decimal` stores numbers in base-10, exactly as humans write them:

```python
from decimal import Decimal, ROUND_HALF_UP

# WRONG — float loses precision
amount = 0.1 + 0.2  # 0.30000000000000004

# CORRECT — Decimal preserves precision
amount = Decimal('0.1') + Decimal('0.2')  # 0.3

# Banking rounding: round to 2 decimal places
result = amount.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
```

In Pydantic schemas, `Decimal` fields must also be handled carefully during JSON serialization. By default, Pydantic v2 will serialize `Decimal` as a number, which can still lose precision in JSON. The solution is to configure a custom JSON encoder that serializes `Decimal` to string:

```python
class PaymentResponse(BaseModel):
    amount: Decimal
    currency: str

    class Config:
        json_encoders = {
            Decimal: lambda v: str(v),
        }
```

In SQLAlchemy, balance columns should use `Numeric(19, 2)` — 19 total digits with 2 decimal places — which maps to `Decimal` in Python.

**Key Points to Hit:**
- [ ] `float` uses binary representation causing precision errors (`0.1 + 0.2 != 0.3`)
- [ ] `Decimal` uses base-10 arithmetic, preserving financial precision exactly
- [ ] Always construct `Decimal` from strings (`Decimal('0.1')`), not floats (`Decimal(0.1)`)
- [ ] Serialize `Decimal` as strings in JSON to avoid precision loss
- [ ] Use `Numeric(19, 2)` in SQLAlchemy for money columns

**Follow-Up Questions:**
1. What rounding modes does Python's `Decimal` support and which is standard in banking?
2. How does Pydantic v2 handle `Decimal` validation differently from v1?
3. If you need to call an external API that returns float amounts, how do you safely convert?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/README.md`, `banking-genai-engineering-academy/backend-engineering/python/pydantic.md`, `banking-genai-engineering-academy/backend-engineering/python/sqlalchemy.md`

---

### 🔵 Q2: Design a FastAPI endpoint that processes uploaded KYC documents using async patterns.

**Strong Answer:**

A KYC document analysis endpoint in a banking GenAI platform needs to handle file uploads, run OCR and AI analysis, and return structured results -- all without blocking the event loop. Here is the design:

```python
from fastapi import APIRouter, UploadFile, Depends
from pydantic import BaseModel

class DocumentAnalysisResult(BaseModel):
    document_type: str
    extracted_fields: dict[str, str]
    confidence_scores: dict[str, float]
    risk_assessment: str

router = APIRouter(prefix='/documents', tags=['documents'])

@router.post('/analyze', response_model=DocumentAnalysisResult)
async def analyze_document(
    file: UploadFile,
    analysis_type: str,
    ai_service: AIService = Depends(get_ai_service),
):
    """Analyze uploaded banking document (KYC, loan application, etc.)."""
    content = await file.read()

    # OCR + AI analysis — both are I/O-bound, use async
    text = await ocr_engine.process(content)
    result = await ai_service.analyze(
        text=text,
        prompt=f'Extract {analysis_type} fields from: {text}',
    )

    return DocumentAnalysisResult(
        document_type=analysis_type,
        extracted_fields=result.fields,
        confidence_scores=result.confidence,
        risk_assessment=result.risk_level,
    )
```

Key architectural decisions: The endpoint is `async def` because file reading, OCR processing, and LLM inference are all I/O-bound operations. Using `await` on each operation allows the event loop to serve other requests while waiting. The business logic lives in `AIService` (injected via dependency injection), not in the route handler -- routes handle HTTP concerns only. The response is a Pydantic model that enforces type safety and auto-generates OpenAPI documentation.

For production at scale, heavy document analysis should actually be offloaded to Celery for background processing, returning a task ID immediately and letting the client poll for results. The async endpoint above works for smaller documents but if the LLM call takes 30+ seconds, the client connection may timeout.

**Key Points to Hit:**
- [ ] Use `async def` for I/O-bound operations (file read, OCR, AI calls)
- [ ] Keep route handlers thin -- delegate to service layer
- [ ] Use Pydantic models for response validation and OpenAPI docs
- [ ] Use dependency injection (`Depends`) for service provisioning
- [ ] For long-running analysis, offload to Celery and return task ID

**Follow-Up Questions:**
1. How would you add rate limiting to this endpoint?
2. What happens if the AI service times out -- how do you handle it gracefully?
3. How would you stream LLM tokens back to the client instead of waiting for the full response?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/README.md`, `banking-genai-engineering-academy/backend-engineering/python/fastapi.md`, `banking-genai-engineering-academy/backend-engineering/python/async-python.md`

---

### 🔵 Q3: What is the GIL and how does it affect the design of a Python backend service for AI inference?

**Strong Answer:**

The GIL (Global Interpreter Lock) is a mutex in CPython that allows only one thread to execute Python bytecode at a time. This means that even on a multi-core machine, a single Python process cannot execute multiple Python threads in parallel. The GIL exists to protect Python's memory management (reference counting) from race conditions.

For banking backend services, the GIL has direct implications:

**I/O-bound tasks** (database queries, HTTP calls, file reads) are NOT affected by the GIL because the lock is released during I/O operations. This is why async FastAPI endpoints handling thousands of concurrent API calls work perfectly -- the GIL is released while waiting for network responses.

**CPU-bound tasks** (ML model inference, risk score calculations, document OCR) ARE blocked by the GIL. If you run ML inference in the same process as your FastAPI server, the event loop will block during prediction, causing all other requests to stall.

The solution is to offload CPU-bound work:

```python
# Option 1: Process pool (within same service)
from concurrent.futures import ProcessPoolExecutor

async def compute_risk_async(self, data: dict) -> float:
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as executor:
        result = await loop.run_in_executor(executor, compute_risk_score, data)
    return result

# Option 2: Celery worker (separate process/queue)
@celery_app.task
def analyze_document_task(document_id: str):
    result = ml_model.predict(load_document(document_id))
    return result
```

For a banking GenAI platform serving 500K+ employees, Option 2 (Celery) is the production choice because it provides retries, monitoring, horizontal scaling, and queue prioritization that a simple process pool cannot offer.

**Key Points to Hit:**
- [ ] GIL allows only one thread to execute Python bytecode at a time
- [ ] I/O-bound work is unaffected (GIL released during I/O waits)
- [ ] CPU-bound work (ML inference) blocks the event loop -- must be offloaded
- [ ] Use `ProcessPoolExecutor` with `run_in_executor` for in-process offloading
- [ ] Use Celery for production-grade background CPU-bound task processing

**Follow-Up Questions:**
1. Does the GIL exist in all Python implementations?
2. When would you choose `ProcessPoolExecutor` over Celery?
3. How does `uvicorn --workers 4` interact with the GIL?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/README.md`, `banking-genai-engineering-academy/backend-engineering/python/async-python.md`, `banking-genai-engineering-academy/backend-engineering/python/celery.md`

---

### 🔵 Q4: How do you structure a FastAPI project with 20+ endpoints across multiple business domains?

**Strong Answer:**

A production FastAPI service in a banking environment must follow a layered architecture that separates HTTP concerns, business logic, and data access. The recommended structure is:

```
banking-genai-service/
├── src/banking_api/
│   ├── main.py              # App entry point, middleware, lifespan
│   ├── config.py            # Pydantic Settings from environment variables
│   ├── dependencies.py      # Shared dependency providers
│   ├── middleware.py        # Correlation ID, security headers
│   ├── models/              # SQLAlchemy database models
│   ├── schemas/             # Pydantic request/response schemas
│   ├── api/v1/              # Route handlers (thin, HTTP-only logic)
│   ├── services/            # Business logic layer
│   ├── repositories/        # Data access layer (SQLAlchemy queries)
│   ├── core/                # Security, logging, exceptions
│   └── workers/             # Celery background tasks
├── tests/
├── alembic/                 # Database migrations
└── pyproject.toml
```

Key principles:

1. **API routes are thin** -- they parse requests, call services, and return responses. No business logic in routes.
2. **Service layer** contains business rules (e.g., "a payment cannot exceed the account balance"). Services depend on repositories, not directly on the database.
3. **Repository layer** handles all SQLAlchemy queries. This makes testing easy (mock the repo) and keeps SQL in one place.
4. **API versioning** via `/api/v1/` prefix allows breaking changes without disrupting existing consumers.
5. **Shared dependencies** are centralized in `dependencies.py` using FastAPI's `Depends()` for clean injection.

```python
# api/v1/payments.py — thin route handler
@router.post('/', response_model=PaymentResponse)
async def create_payment(
    payment: PaymentCreate,
    payment_service: PaymentService = Depends(get_payment_service),
):
    return await payment_service.create_payment(payment)

# services/payment_service.py — business logic
class PaymentService:
    def __init__(self, account_repo: AccountRepository, payment_repo: PaymentRepository):
        self.account_repo = account_repo
        self.payment_repo = payment_repo

    async def create_payment(self, data: PaymentCreate):
        account = await self.account_repo.get_by_id(data.from_account)
        if account.balance < data.amount:
            raise InsufficientFundsError(...)
        return await self.payment_repo.create(data)
```

**Key Points to Hit:**
- [ ] Layered architecture: routes → services → repositories → database
- [ ] Routes handle HTTP only; business logic in service layer
- [ ] Repository pattern isolates all database queries
- [ ] API versioning (`/api/v1/`) for backward compatibility
- [ ] Shared dependencies via `Depends()` in `dependencies.py`

**Follow-Up Questions:**
1. How do you handle cross-cutting concerns like audit logging across all layers?
2. When would you break a monolithic service into microservices?
3. How do you manage database transactions that span multiple services?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/fastapi.md`

---

### 🔵 Q5: How do you test a FastAPI endpoint that calls external services (AI APIs, payment gateways)?

**Strong Answer:**

Testing endpoints that depend on external services requires mocking those dependencies while exercising the full HTTP request/response cycle. The approach uses pytest fixtures, `httpx.AsyncClient`, and dependency overrides:

```python
# conftest.py — shared fixtures
@pytest.fixture
async def client(db_session):
    """Create async test client with test database."""
    async def override_get_db():
        yield db_session

    app.dependency_overrides['get_db_session'] = override_get_db

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url='http://test') as ac:
        yield ac

@pytest.fixture
def ai_service():
    """Create mocked AI service."""
    service = AsyncMock()
    return service

# test_api/test_documents.py
class TestDocumentAnalysis:
    @pytest.mark.asyncio
    async def test_analyze_kyc_document(self, client, ai_service):
        # Arrange: mock the AI response
        ai_service.analyze.return_value = {
            'document_type': 'passport',
            'fields': {'name': 'John Doe'},
            'confidence': {'name': 0.98},
        }
        app.dependency_overrides['get_ai_service'] = lambda: ai_service

        # Act: make real HTTP request
        response = await client.post('/api/v1/documents/analyze', json={
            'document': 'sample_kyc_data',
            'analysis_type': 'kyc',
        })

        # Assert: check response and that service was called
        assert response.status_code == 200
        data = response.json()
        assert data['document_type'] == 'passport'
        ai_service.analyze.assert_awaited_once()
```

The test database fixture creates a session wrapped in a transaction that is rolled back after each test, ensuring test isolation. External services are replaced with `AsyncMock` via dependency overrides. For error scenarios, configure the mock to raise exceptions (timeout, connection error) and verify the endpoint returns appropriate HTTP status codes (503 for service unavailable).

**Key Points to Hit:**
- [ ] Use `httpx.AsyncClient` with `ASGITransport` for real async HTTP testing
- [ ] Override dependencies with `app.dependency_overrides` to inject mocks
- [ ] Use `AsyncMock` for async service methods
- [ ] Test database uses transaction rollback per test for isolation
- [ ] Test both success and failure paths (timeouts, connection errors)

**Follow-Up Questions:**
1. How do you test idempotency for a POST endpoint that creates payments?
2. What is the difference between unit tests and integration tests for FastAPI?
3. How do you measure and enforce test coverage thresholds?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/pytest.md`, `banking-genai-engineering-academy/backend-engineering/python/fastapi.md`

---

## 🟡 Medium Questions (Q6-Q15)

### 🟡 Q6: How does Pydantic validate a payment request against both format rules AND account balance?

**Strong Answer:**

Pydantic handles format validation at the schema level using field validators and model validators, but balance checking requires external data (the account balance from the database). The approach combines both:

```python
class PaymentCreate(BaseModel):
    from_account: str = Field(..., min_length=10, max_length=34)
    to_account: str = Field(..., min_length=10, max_length=34)
    amount: Decimal = Field(..., gt=0, le=Decimal('1000000.00'))
    currency: Currency

    @field_validator('amount')
    @classmethod
    def validate_amount_precision(cls, v: Decimal) -> Decimal:
        return v.quantize(Decimal('0.01'))

    @model_validator(mode='after')
    def validate_not_self_payment(self) -> 'PaymentCreate':
        if self.from_account == self.to_account:
            raise ValueError('Cannot pay to self')
        return self


# External context validation happens in the service layer
class PaymentContext:
    def validate_with_context(self, payment: PaymentCreate, account_balance: Decimal):
        if payment.amount > account_balance:
            raise ValueError(
                f'Insufficient funds: available {account_balance}, requested {payment.amount}'
            )
        daily_total = self.get_daily_total(payment.from_account)
        if daily_total + payment.amount > Decimal('50000'):
            raise ValueError('Daily transfer limit exceeded ($50,000)')
```

Format rules (IBAN format, amount precision, non-self-payment) are enforced by Pydantic at the API boundary. External context rules (sufficient balance, daily limits) are enforced in the service layer after fetching the account data. This separation is important because Pydantic should not have database dependencies -- it validates the shape of data, not the state of the world.

**Key Points to Hit:**
- [ ] Field validators (`@field_validator`) check individual field format
- [ ] Model validators (`@model_validator(mode='after')`) check cross-field rules
- [ ] External context validation (balance, daily limits) belongs in the service layer
- [ ] Pydantic should never depend on database or external services
- [ ] Two-phase validation: format at API boundary, context in business logic

**Follow-Up Questions:**
1. How would you implement a custom Pydantic type for IBAN validation?
2. What is the difference between `field_validator` and `model_validator`?
3. How do you handle partial updates (PATCH) where some fields are optional?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/pydantic.md`

---

### 🟡 Q7: Design a discriminated union in Pydantic for different payment methods (wire, SEPA, domestic).

**Strong Answer:**

Discriminated unions let Pydantic automatically select the correct schema based on a discriminator field. For payment methods:

```python
from typing import Annotated, Union, Literal
from pydantic import BaseModel, Field

class WireTransfer(BaseModel):
    type: Literal['wire']
    swift_code: str
    beneficiary_name: str
    beneficiary_account: str
    intermediary_bank: Optional[str] = None

class SEPATransfer(BaseModel):
    type: Literal['sepa']
    iban: str
    beneficiary_name: str
    bic: Optional[str] = None

class DomesticTransfer(BaseModel):
    type: Literal['domestic']
    account_number: str
    routing_number: str
    beneficiary_name: str

PaymentMethod = Annotated[
    Union[WireTransfer, SEPATransfer, DomesticTransfer],
    Field(discriminator='type'),
]

class PaymentCreate(BaseModel):
    amount: Decimal
    currency: str
    method: PaymentMethod
```

When a request arrives with `{"method": {"type": "sepa", "iban": "GB29NWBK60161331926819", ...}}`, Pydantic reads the `type` field and automatically validates against `SEPATransfer`. If the `iban` field is missing, the error message is specific to SEPA -- not a confusing union of all three schemas.

In FastAPI, this works seamlessly: the OpenAPI documentation shows all three payment method schemas with the discriminator field, and the API returns clear validation errors when the wrong fields are provided for a given type.

**Key Points to Hit:**
- [ ] Each variant has a `Literal` type field that serves as the discriminator
- [ ] `Annotated[Union[...], Field(discriminator='type')]` enables auto-selection
- [ ] FastAPI uses this to generate correct OpenAPI documentation
- [ ] Error messages are specific to the matched variant, not confusing union errors
- [ ] New payment methods are added by creating a new model and adding it to the Union

**Follow-Up Questions:**
1. What happens if the discriminator field is missing from the request?
2. Can you use a nested field as the discriminator?
3. How does this compare to using a single model with all optional fields?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/pydantic.md`

---

### 🟡 Q8: What is the difference between `asyncio.gather()` and `asyncio.as_completed()` and when would you use each?

**Strong Answer:**

Both run multiple coroutines concurrently, but they differ in how results are returned:

**`asyncio.gather()`** waits for ALL coroutines to complete and returns results in the same order as the input. It is "all or nothing" -- if any task raises an exception, `gather` re-raises it (unless `return_exceptions=True` is set).

```python
# All results come back at once, in input order
results = await asyncio.gather(
    fetch_account('ACC-001'),
    fetch_account('ACC-002'),
    fetch_account('ACC-003'),
)
# results[0] corresponds to ACC-001, results[1] to ACC-002, etc.
```

Use `gather` when you need ALL results before proceeding -- e.g., fetching all account details to display a dashboard.

**`asyncio.as_completed()`** returns an iterator that yields results as each coroutine finishes, regardless of input order. You process results immediately as they arrive.

```python
# Process results as they arrive
tasks = [fetch_account(acc_id) for acc_id in account_ids]
for coro in asyncio.as_completed(tasks):
    result = await coro
    process_result(result)  # Handle each one immediately
```

Use `as_completed` when you want to process results incrementally -- e.g., streaming processed documents to a client as each AI analysis finishes, rather than waiting for all 100 documents.

For banking error handling with `gather`, always use `return_exceptions=True` so one failed account fetch does not cancel all others:

```python
results = await asyncio.gather(*tasks, return_exceptions=True)
for i, result in enumerate(results):
    if isinstance(result, Exception):
        logger.error(f'Account {ids[i]} failed: {result}')
    else:
        process(result)
```

**Key Points to Hit:**
- [ ] `gather()` returns all results at once in input order; `as_completed()` yields as each finishes
- [ ] `gather()` raises on first exception unless `return_exceptions=True`
- [ ] Use `gather()` when all results are needed before proceeding
- [ ] Use `as_completed()` for incremental/streaming processing
- [ ] Always use `return_exceptions=True` with `gather()` in production

**Follow-Up Questions:**
1. How does `asyncio.wait()` differ from both?
2. How do you implement a timeout for `asyncio.gather()`?
3. What happens to pending tasks if you cancel a `gather()`?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/async-python.md`

---

### 🟡 Q9: How do you prevent N+1 queries in SQLAlchemy when fetching accounts with their transactions?

**Strong Answer:**

The N+1 problem occurs when you fetch a list of parent objects, then issue a separate query for each parent's related children. For accounts with transactions:

```python
# BAD: N+1 queries
accounts = await session.execute(select(Account))
for account in accounts.scalars():
    # Each access to account.transactions triggers a new query!
    for txn in account.transactions:
        process(txn)
# Result: 1 query for accounts + N queries for transactions
```

The solution is eager loading. SQLAlchemy offers two strategies:

**`selectinload`** issues a second SELECT with an IN clause for all related objects:

```python
# GOOD: 2 queries total
stmt = (
    select(Account)
    .options(selectinload(Account.transactions))
    .where(Account.customer_id.in_(customer_ids))
)
result = await session.execute(stmt)
accounts = result.scalars().unique().all()
```

Query 1: `SELECT * FROM accounts WHERE customer_id IN (...)`
Query 2: `SELECT * FROM transactions WHERE account_id IN (...)`

**`joinedload`** uses a LEFT OUTER JOIN to fetch everything in one query. This is better when the relationship is 1:1 or the child set is small.

```python
stmt = (
    select(Account)
    .options(joinedload(Account.customer))  # 1:1 relationship
    .where(Account.id.in_(account_ids))
)
```

For 1:many relationships (account → many transactions), `selectinload` is almost always better because JOINs multiply rows (an account with 100 transactions produces 100 identical account rows), wasting memory and network bandwidth.

Always call `.unique()` on the result when using eager loading to deduplicate rows produced by JOINs.

**Key Points to Hit:**
- [ ] N+1 occurs when accessing relationships triggers individual queries per parent
- [ ] `selectinload` issues a second IN-clause query -- best for 1:many
- [ ] `joinedload` uses LEFT OUTER JOIN -- best for 1:1 or small child sets
- [ ] Always use `.unique()` after eager loading to deduplicate JOIN rows
- [ ] Configure `lazy='selectin'` on relationships for default eager loading

**Follow-Up Questions:**
1. What is the performance trade-off between `selectinload` and `joinedload`?
2. How do you handle 3-level deep eager loading (account → transactions → categories)?
3. What does `lazy='selectin'` on a relationship definition do?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/sqlalchemy.md`

---

### 🟡 Q10: Design a Celery retry strategy for an external payment gateway with rate limiting.

**Strong Answer:**

External payment gateways impose rate limits (e.g., 100 requests/minute) and occasionally return 5xx errors. The retry strategy must handle both transient failures and rate limits intelligently:

```python
@celery_app.task(bind=True, max_retries=5)
def call_payment_gateway(self, payment_id: str):
    try:
        response = requests.post(
            'https://gateway.external.com/payments',
            json={'payment_id': payment_id},
            timeout=30,
        )
        response.raise_for_status()
        return response.json()

    except requests.exceptions.Timeout:
        # Transient timeout — exponential backoff with jitter
        retry_delay = min(60 * (2 ** self.request.retries), 3600)
        import random
        retry_delay = retry_delay * random.uniform(0.5, 1.5)
        raise self.retry(countdown=retry_delay)

    except requests.exceptions.HTTPError as e:
        status_code = e.response.status_code
        if status_code >= 500:
            # Server error — retry with backoff
            raise self.retry(countdown=120)
        elif status_code == 429:
            # Rate limited — respect Retry-After header
            retry_after = int(e.response.headers.get('Retry-After', 60))
            raise self.retry(countdown=retry_after)
        else:
            # 4xx client error — don't retry (permanent)
            raise
```

Key design decisions:
- **`max_retries=5`** prevents infinite retry loops
- **Exponential backoff with jitter** avoids thundering herd when many tasks retry simultaneously
- **Respect `Retry-After` header** for 429 responses -- the gateway tells us exactly when to retry
- **Distinguish transient vs permanent errors** -- 5xx retries, 4xx (except 429) fails immediately
- **Cap max delay at 1 hour** to avoid tasks lingering indefinitely

For critical payment tasks that exhaust all retries, route to a dead letter queue for manual investigation:

```python
except Exception as exc:
    if self.request.retries >= self.max_retries:
        DeadLetterQueue.objects.create(
            task_name=self.name, task_id=self.request.id,
            args=payment_id, error=str(exc),
        )
        alert_oncall_team(f'Task {self.name} exhausted retries')
    raise self.retry(exc=exc)
```

**Key Points to Hit:**
- [ ] Set `max_retries` to prevent infinite retry loops
- [ ] Use exponential backoff with jitter to avoid thundering herd
- [ ] Respect `Retry-After` header for 429 rate limit responses
- [ ] Retry 5xx (transient), fail immediately on 4xx (permanent, except 429)
- [ ] Route exhausted retries to a dead letter queue with alerting

**Follow-Up Questions:**
1. How do you configure different queues for different task priorities?
2. What does `task_acks_late = True` do and why is it critical for payments?
3. How do you monitor Celery task health in production?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/celery.md`

---

### 🟡 Q11: How do you handle a CPU-intensive ML inference task inside an async FastAPI endpoint?

**Strong Answer:**

Running CPU-bound ML inference directly in an async FastAPI endpoint blocks the event loop, stalling all other requests. There are two production-ready approaches:

**Approach 1: ProcessPoolExecutor within the same process**

```python
from concurrent.futures import ProcessPoolExecutor

def compute_risk_score(data: dict) -> float:
    """CPU-intensive — runs in separate process."""
    return model.predict(data)

async def predict_risk_endpoint(data: dict):
    """Async endpoint — offloads to process pool."""
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as executor:
        result = await loop.run_in_executor(executor, compute_risk_score, data)
    return result
```

This works for moderate loads because `run_in_executor` schedules the function in a separate OS process, bypassing the GIL. The event loop remains unblocked.

**Approach 2: Celery task queue (production choice for banking)**

```python
# In the FastAPI endpoint
@router.post('/risk/predict')
async def predict_risk(data: RiskRequest):
    task = compute_risk_task.delay(data.model_dump())
    return {'task_id': task.id, 'status': 'queued'}

# In Celery worker
@celery_app.task
def compute_risk_task(data: dict) -> dict:
    score = ml_model.predict(data)
    return {'risk_score': score}
```

For a banking GenAI platform, Approach 2 is the correct choice because:
- ML inference can take seconds; clients should not hold HTTP connections open
- Celery provides retries if the model crashes
- Separate worker processes can be GPU-enabled without affecting the API server
- Queue prioritization ensures critical risk checks run before batch scoring
- Monitoring via Flower provides visibility into inference latency

**Key Points to Hit:**
- [ ] CPU-bound tasks block the async event loop -- must be offloaded
- [ ] `ProcessPoolExecutor` + `run_in_executor` works for moderate loads
- [ ] Celery is the production choice: retries, monitoring, GPU workers, queue priority
- [ ] Return task ID immediately; client polls for result (async pattern)
- [ ] Separate AI inference workers from general workers for resource isolation

**Follow-Up Questions:**
1. How many worker processes should the ProcessPoolExecutor have?
2. What happens if the ML model takes 60 seconds to respond?
3. How do you handle model version updates without downtime?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/async-python.md`, `banking-genai-engineering-academy/backend-engineering/python/celery.md`

---

### 🟡 Q12: How do you configure and use Pydantic Settings for environment-based configuration in a banking service?

**Strong Answer:**

`pydantic-settings` provides type-safe configuration loaded from environment variables with validation at startup:

```python
from pydantic_settings import BaseSettings
from pydantic import field_validator
from functools import lru_cache

class Settings(BaseSettings):
    app_name: str = 'banking-genai-api'
    environment: str = 'development'
    debug: bool = False

    database_url: str
    database_pool_size: int = 20
    database_max_overflow: int = 10

    redis_url: str = 'redis://localhost:6379/0'

    jwt_secret_key: str
    jwt_algorithm: str = 'HS256'
    allowed_origins: list[str] = []

    openai_api_key: str
    openai_model: str = 'gpt-4'
    ai_timeout: int = 60

    @field_validator('database_url')
    @classmethod
    def validate_database_url(cls, v: str) -> str:
        if not v.startswith(('postgresql://', 'postgresql+asyncpg://')):
            raise ValueError('Database URL must start with postgresql://')
        return v

    @field_validator('jwt_secret_key')
    @classmethod
    def validate_jwt_secret(cls, v: str) -> str:
        if len(v) < 32:
            raise ValueError('JWT secret must be at least 32 characters')
        return v

    class Config:
        env_file = '.env'
        env_file_encoding = 'utf-8'
        case_sensitive = False

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

Key design decisions:
- **No defaults for secrets** (`database_url`, `jwt_secret_key`, `openai_api_key`) -- the service crashes at startup if these are missing, failing fast rather than running with insecure defaults
- **Field validators** enforce format rules (URL scheme, minimum key length)
- **`@lru_cache()`** creates a singleton -- settings are loaded once and reused
- **Sensible defaults** for non-sensitive values (`pool_size=20`, `debug=False`)
- **`.env` file support** for local development, overridden by actual environment variables in production

**Key Points to Hit:**
- [ ] `pydantic-settings` loads from environment variables with type coercion
- [ ] No defaults for secrets -- fail fast at startup if missing
- [ ] Field validators enforce format (URL scheme, key length)
- [ ] `@lru_cache()` provides a cached singleton
- [ ] `.env` file for development; real env vars in production/Kubernetes

**Follow-Up Questions:**
1. How do you handle different settings for dev/staging/production?
2. What happens if a required environment variable is missing?
3. How do you reload settings at runtime without restarting the service?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/fastapi.md`, `banking-genai-engineering-academy/backend-engineering/python/packaging.md`

---

### 🟡 Q13: How do you use Poetry to ensure reproducible builds across development, staging, and production?

**Strong Answer:**

Poetry ensures reproducibility through three mechanisms:

**1. `pyproject.toml`** declares dependencies with version constraints using caret syntax:

```toml
[tool.poetry.dependencies]
python = "^3.11"          # >=3.11, <4.0
fastapi = "^0.110.0"     # >=0.110.0, <1.0.0
pydantic = "^2.6.0"      # >=2.6.0, <3.0.0
```

**2. `poetry.lock`** pins the exact resolved versions of every dependency (including transitive dependencies). This file MUST be committed to version control. When CI runs `poetry install`, it reads `poetry.lock` and installs the exact same versions everywhere.

**3. Separation of dev and production dependencies** ensures production images do not include testing tools:

```toml
[tool.poetry.group.dev.dependencies]
pytest = "^8.0.0"
ruff = "^0.2.0"
mypy = "^1.8.0"
```

Build process:
```bash
# Development
poetry install              # Installs main + dev dependencies from lock

# Production (Docker build)
poetry install --only main  # Only production dependencies
```

In CI/CD:
```yaml
- name: Install dependencies
  run: poetry install --with dev  # Full install for testing

- name: Run tests
  run: poetry run pytest --cov=banking_api

- name: Build Docker
  run: |
    # Multi-stage build: install deps first (cached), then copy code
    docker build -t banking-api:latest .
```

The Dockerfile uses multi-stage builds to keep the final image small: dependencies are installed in one stage, then copied to a clean production image without build tools.

**Key Points to Hit:**
- [ ] `poetry.lock` pins exact versions -- must be committed to version control
- [ ] Caret syntax (`^1.0.0`) allows compatible updates while preventing breaking changes
- [ ] `--only main` excludes dev dependencies in production
- [ ] Multi-stage Docker builds keep production images small
- [ ] CI installs full dependencies for testing; production installs only main

**Follow-Up Questions:**
1. What is the difference between `^1.0.0` and `~1.0.0` version constraints?
2. How do you update a single dependency without changing everything else?
3. What happens if two team members have different Python minor versions?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/packaging.md`

---

### 🟡 Q14: How do you ensure test isolation when tests share a database in pytest?

**Strong Answer:**

The key technique is wrapping each test in a database transaction that is rolled back after the test completes, rather than using separate databases or manual cleanup:

```python
# conftest.py
@pytest.fixture
async def db_session(test_engine):
    """Database session with automatic rollback."""
    async with test_engine.connect() as conn:
        async with conn.begin() as transaction:
            session = AsyncSession(conn)
            yield session
            # After test: rollback cleans up all changes
            await session.close()
            await transaction.rollback()
```

How it works:
1. `test_engine` connects to a dedicated test database (created once per session)
2. Each test gets its own `AsyncSession` wrapped in a transaction
3. The test can INSERT, UPDATE, DELETE freely
4. After the test finishes, `transaction.rollback()` undoes everything
5. The next test starts with a clean database

This is better than truncating tables between tests because:
- Rollback is instant (no DELETE queries)
- Cannot forget cleanup -- it is automatic in the fixture
- Works correctly with foreign keys and cascades

For the test engine itself, tables are created once at session start and dropped at session end:

```python
@pytest.fixture(scope='session')
async def test_engine():
    engine = create_async_engine('postgresql+asyncpg://test:test@localhost:5432/banking_test')
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()
```

**Key Points to Hit:**
- [ ] Each test runs in a transaction that is rolled back after completion
- [ ] Test database schema is created once per session, not per test
- [ ] Rollback is instant and handles cascading deletes automatically
- [ ] Use a dedicated test database, never the development or production database
- [ ] Fixtures handle setup/teardown -- tests should not manage database state

**Follow-Up Questions:**
1. How do you test database migrations with pytest?
2. What happens if a test explicitly commits inside the transaction?
3. How do you handle tests that need to test the commit behavior itself?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/pytest.md`, `banking-genai-engineering-academy/backend-engineering/python/sqlalchemy.md`

---

### 🟡 Q15: How do you stream LLM tokens to a client using FastAPI and async generators?

**Strong Answer:**

Streaming LLM responses provides a better user experience because the client sees tokens appearing in real time rather than waiting 30+ seconds for the full response. The implementation uses async generators with FastAPI's `StreamingResponse`:

```python
async def stream_llm_response(prompt: str):
    """Stream LLM tokens as an async generator."""
    async for chunk in openai_client.chat.completions.create(
        model='gpt-4',
        messages=[{'role': 'user', 'content': prompt}],
        stream=True,
    ):
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

@app.get('/v1/ai/chat/stream')
async def chat_stream(prompt: str):
    async def generate():
        async for token in stream_llm_response(prompt):
            yield f'data: {json.dumps({"token": token})}\n\n'
        yield 'data: [DONE]\n\n'

    return StreamingResponse(generate(), media_type='text/event-stream')
```

Key design decisions:
- **Async generator** (`async for ... yield`) produces tokens one at a time as they arrive from the LLM
- **Server-Sent Events (SSE)** format (`data: {...}\n\n`) allows the browser to parse individual tokens
- **`StreamingResponse`** with `text/event-stream` media type keeps the HTTP connection open and flushes each yield immediately
- **`[DONE]` sentinel** signals the end of the stream to the client

For a banking chatbot answering questions about account policies, this means a 200-word response starts displaying within 1-2 seconds rather than requiring the user to wait for the full generation.

**Key Points to Hit:**
- [ ] Async generators (`async for ... yield`) produce tokens incrementally
- [ ] Server-Sent Events format (`data: ...\n\n`) for client-side parsing
- [ ] `StreamingResponse` with `text/event-stream` keeps connection open
- [ ] `[DONE]` sentinel marks stream completion
- [ ] Dramatically improves perceived latency for long LLM responses

**Follow-Up Questions:**
1. How do you handle errors that occur mid-stream?
2. How do you add authentication to a streaming endpoint?
3. What happens if the client disconnects before the stream completes?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/async-python.md`, `banking-genai-engineering-academy/backend-engineering/python/fastapi.md`

---

## 🔴 Advanced Questions (Q16-Q20)

### 🔴 Q16: Design an async system that processes 1,000 payments concurrently with a limit of 50 concurrent connections.

**Strong Answer:**

Processing 1,000 payments concurrently without overwhelming the payment gateway requires a semaphore-based concurrency limiter:

```python
async def process_payments_with_limit(payments: list[dict], max_concurrent: int = 50):
    """Process payments with concurrency limit."""
    semaphore = asyncio.Semaphore(max_concurrent)
    results = []

    async def process_one(payment: dict):
        async with semaphore:
            return await execute_payment(payment)

    tasks = [process_one(p) for p in payments]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # Separate successes from failures
    successes = []
    failures = []
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            failures.append({'payment': payments[i], 'error': result})
        else:
            successes.append(result)

    return {'successes': successes, 'failures': failures}
```

How the semaphore works: `asyncio.Semaphore(50)` creates a lock with 50 permits. Each `async with semaphore` acquires one permit. If all 50 are in use, the 51st coroutine waits until a permit is released. This ensures at most 50 concurrent payment API calls at any time.

For production banking, additional considerations apply:

```python
async def process_payments_production(payments: list[dict]):
    semaphore = asyncio.Semaphore(50)

    async def process_one_with_retry(payment: dict):
        async with semaphore:
            for attempt in range(3):
                try:
                    return await execute_payment(payment)
                except (ConnectionError, TimeoutError) as e:
                    if attempt == 2:
                        raise
                    await asyncio.sleep(2 ** attempt)  # Exponential backoff

    tasks = [process_one_with_retry(p) for p in payments]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

This combines concurrency limiting with per-payment retry logic. For truly critical payments that must not be lost, offload to Celery with `task_acks_late=True` and a dead letter queue instead.

**Key Points to Hit:**
- [ ] `asyncio.Semaphore(N)` limits concurrent operations to N
- [ ] Wrap each task with `async with semaphore` to acquire/release permits
- [ ] Use `return_exceptions=True` with `gather` to handle individual failures
- [ ] Add per-payment retry with exponential backoff for transient errors
- [ ] For payments that absolutely cannot be lost, use Celery with acks_late + DLQ

**Follow-Up Questions:**
1. What happens if 500 of the 1,000 payments fail?
2. How do you add rate limiting based on time (e.g., 50 per second, not just 50 concurrent)?
3. When would you choose Celery over asyncio for this scenario?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/async-python.md`, `banking-genai-engineering-academy/backend-engineering/python/celery.md`

---

### 🔴 Q17: Design optimistic locking in SQLAlchemy for concurrent balance updates.

**Strong Answer:**

Optimistic locking prevents lost updates when multiple transactions modify the same row concurrently. The pattern uses a `version` column that increments on each update:

```python
class Account(Base):
    __tablename__ = 'accounts'

    id: Mapped[str] = mapped_column(UUID(as_uuid=True), primary_key=True)
    balance: Mapped[Decimal] = mapped_column(Numeric(19, 2), nullable=False)
    version: Mapped[int] = mapped_column(nullable=False, default=0, server_default='0')

class AccountRepository:
    async def update_balance(
        self,
        account_id: str,
        amount: Decimal,
        expected_version: int,
    ) -> Account:
        """Update balance only if version matches."""
        stmt = (
            select(Account)
            .where(Account.id == account_id, Account.version == expected_version)
        )
        result = await self.session.execute(stmt)
        account = result.scalar_one_or_none()

        if account is None:
            raise ConcurrentModificationError(
                f'Account {account_id} not found or version mismatch'
            )

        if amount > 0:
            account.deposit(amount)  # Increments version
        else:
            account.withdraw(abs(amount))  # Increments version

        await self.session.flush()
        return account
```

The `deposit` and `withdraw` methods increment `version`:
```python
def deposit(self, amount: Decimal):
    self.balance += amount
    self.version += 1
```

The SQL generated is:
```sql
UPDATE accounts SET balance = 1200.00, version = 2
WHERE id = 'abc' AND version = 1;
```

If another transaction already updated this row (version is now 2), the WHERE clause matches zero rows, `scalar_one_or_none()` returns None, and we raise `ConcurrentModificationError`. The caller must retry: fetch the latest version and attempt again.

For banking, this is superior to pessimistic locking (`SELECT ... FOR UPDATE`) because it does not hold database locks during business logic execution, reducing contention in high-throughput systems.

**Key Points to Hit:**
- [ ] Version column incremented on each update
- [ ] UPDATE includes `WHERE version = expected_version` in the WHERE clause
- [ ] Zero rows affected = concurrent modification detected
- [ ] Caller retries with fresh data on conflict
- [ ] Superior to `SELECT FOR UPDATE` for reducing lock contention

**Follow-Up Questions:**
1. How do you implement retry logic for optimistic locking conflicts?
2. When would you choose pessimistic locking over optimistic locking?
3. How does this interact with Celery tasks that process the same account?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/sqlalchemy.md`

---

### 🔴 Q18: How do you handle graceful shutdown of async database connections and background tasks in FastAPI?

**Strong Answer:**

Graceful shutdown ensures that in-flight requests complete, database connections close cleanly, and background tasks finish before the process terminates. In a banking environment, improper shutdown can leave transactions half-committed or connections leaked.

FastAPI provides the `lifespan` context manager for startup/shutdown:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info('Starting Banking GenAI API')
    await setup_database()
    await setup_cache()

    yield  # Server runs here

    # Shutdown
    logger.info('Shutting down Banking GenAI API')
    await teardown_database()
    await teardown_cache()

app = FastAPI(lifespan=lifespan)
```

For database teardown:
```python
async def teardown_database():
    """Close all database connections gracefully."""
    await engine.dispose()  # Closes all pooled connections
```

For background tasks, track them and cancel on shutdown:
```python
background_tasks: set[asyncio.Task] = set()

async def spawn_background_task(coro):
    task = asyncio.create_task(coro)
    background_tasks.add(task)
    task.add_done_callback(background_tasks.discard)
    return task

async def teardown_background_tasks():
    for task in background_tasks:
        task.cancel()
    if background_tasks:
        await asyncio.gather(*background_tasks, return_exceptions=True)
```

The uvicorn server also needs configuration for graceful shutdown:
```bash
uvicorn banking_api.main:app \
    --host 0.0.0.0 \
    --port 8000 \
    --workers 4 \
    --timeout-graceful-shutdown 30
```

This gives 30 seconds for in-flight requests to complete before forced termination. Kubernetes sends SIGTERM, uvicorn stops accepting new requests, waits for existing requests, runs the lifespan shutdown, then exits.

**Key Points to Hit:**
- [ ] `lifespan` context manager handles startup and shutdown logic
- [ ] `engine.dispose()` closes all pooled database connections
- [ ] Track background tasks and cancel them during shutdown
- [ ] Configure `--timeout-graceful-shutdown` in uvicorn
- [ ] Kubernetes SIGTERM triggers the entire graceful shutdown chain

**Follow-Up Questions:**
1. How do you drain a pod in Kubernetes without dropping requests?
2. What happens to Celery workers during a rolling deployment?
3. How do you handle a scenario where a request is mid-transaction during shutdown?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/fastapi.md`, `banking-genai-engineering-academy/backend-engineering/python/async-python.md`

---

### 🔴 Q19: How do you handle a scenario where 10,000 Celery tasks are queued but only 10 workers are available?

**Strong Answer:**

This is a classic capacity and prioritization problem. The solution involves multiple layers:

**1. Queue segregation by priority:**

```python
celery_app.conf.task_routes = {
    'tasks.payment.*': {'queue': 'payments'},        # High priority
    'tasks.notifications.*': {'queue': 'notifications'},  # Medium
    'tasks.reports.*': {'queue': 'reports'},          # Low
}
```

Run more workers on the payments queue:
```bash
# Payment workers (high priority)
celery -A celery_config worker --queues=payments --concurrency=5

# Report workers (low priority)
celery -A celery_config worker --queues=reports --concurrency=2
```

**2. Worker capacity controls:**

```python
celery_app.conf.task_acks_late = True        # Don't lose tasks if worker crashes
celery_app.conf.worker_prefetch_multiplier = 1  # One task at a time per worker
celery_app.conf.task_soft_time_limit = 300   # 5 min soft limit
celery_app.conf.task_time_limit = 600        # 10 min hard limit
```

With `worker_prefetch_multiplier = 1`, each worker only holds one task at a time. Without this, a worker could prefetch 4 tasks (default multiplier), meaning 10 workers could hold 40 tasks, starving other queues.

**3. Monitoring and alerting:**

```python
@task_prerun.connect
def task_prerun_handler(task_id, task, *args, **kwargs):
    task._start_time = time.time()

@task_postrun.connect
def task_postrun_handler(task_id, task, *args, retval=None, **kwargs):
    duration = time.time() - getattr(task, '_start_time', time.time())
    TASK_DURATION.labels(task_name=task.name).observe(duration)
```

Monitor queue length via Flower or Prometheus. If the payments queue exceeds a threshold (e.g., 500 pending), alert oncall and consider auto-scaling workers.

**4. Dead letter queue for failed tasks** ensures that tasks that cannot be processed do not block the queue indefinitely:

```python
if self.request.retries >= self.max_retries:
    DeadLetterQueue.objects.create(...)
    alert_oncall_team(...)
```

**Key Points to Hit:**
- [ ] Route tasks to separate queues by priority (payments vs reports)
- [ ] Allocate more workers to high-priority queues
- [ ] Set `worker_prefetch_multiplier = 1` to prevent task hoarding
- [ ] Use `task_acks_late = True` so crashed workers do not lose tasks
- [ ] Monitor queue depth and alert/auto-scale when thresholds exceeded

**Follow-Up Questions:**
1. How do you dynamically add workers to a running Celery cluster?
2. What happens if the Redis broker runs out of memory with 100K pending tasks?
3. How do you cancel or reprioritize tasks that are already in the queue?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/celery.md`

---

### 🔴 Q20: How do you structure bulk database operations for 10,000 account balance updates using SQLAlchemy?

**Strong Answer:**

Updating 10,000 accounts one at a time with individual `session.commit()` calls is prohibitively slow. The correct approach uses bulk operations that execute as single SQL statements:

```python
async def bulk_update_balances(updates: list[dict]):
    """Bulk update balances with a single SQL statement using CASE."""
    from sqlalchemy import case

    account_ids = [u['id'] for u in updates]

    stmt = (
        Account.__table__.update()
        .where(Account.id.in_(account_ids))
        .values(
            balance=case(
                *(
                    (Account.id == u['id'], Account.balance + u['amount'])
                    for u in updates
                ),
            ),
            updated_at=datetime.now(timezone.utc),
        ),
    )

    await session.execute(stmt)
    await session.commit()
```

This generates a single SQL statement:
```sql
UPDATE accounts
SET balance = CASE
    WHEN id = 'acc-1' THEN balance + 100.00
    WHEN id = 'acc-2' THEN balance + 250.50
    WHEN id = 'acc-3' THEN balance - 50.00
    ...
END,
updated_at = NOW()
WHERE id IN ('acc-1', 'acc-2', 'acc-3', ...);
```

For bulk inserts, use `executemany`:
```python
async def bulk_insert_accounts(accounts_data: list[dict]):
    """Bulk insert with single query."""
    await session.execute(
        Account.__table__.insert(),
        accounts_data,  # List of dicts
    )
    await session.commit()
```

Important considerations for banking:
- **Wrap in a single transaction** so either all updates succeed or none do (atomicity)
- **Use `__table__.update()`** (Core API) instead of ORM for bulk operations -- ORM would instantiate 10,000 objects, triggering events and validation for each
- **Optimistic locking bypass**: bulk operations skip version checks. If concurrent updates are possible, use the `version` column in the WHERE clause or apply updates in a maintenance window
- **Chunk large operations**: for 100K+ updates, batch in chunks of 10,000 to avoid transaction log overflow

**Key Points to Hit:**
- [ ] Use SQLAlchemy Core (`__table__.update()`) for bulk, not ORM
- [ ] CASE statement updates all rows in a single SQL statement
- [ ] Single transaction ensures atomicity (all or nothing)
- [ ] ORM instantiation for 10K objects is prohibitively slow
- [ ] Chunk extremely large operations (100K+) to manage transaction log size

**Follow-Up Questions:**
1. How do you handle optimistic locking with bulk updates?
2. What is the maximum number of rows in a single CASE statement?
3. How do you track which rows were actually updated vs unchanged?

**Source:** `banking-genai-engineering-academy/backend-engineering/python/sqlalchemy.md`

---
