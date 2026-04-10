# FastAPI Production Service Structure

## Why FastAPI

FastAPI is the modern Python web framework for building production APIs. It provides automatic request validation, OpenAPI documentation, async support, and high performance comparable to Node.js and Go. For banking GenAI platforms, FastAPI enables rapid development of type-safe APIs with built-in documentation.

## Production Service Layout

```
banking-genai-service/
├── pyproject.toml              # Dependencies and build config
├── Dockerfile                   # Container definition
├── docker-compose.yml           # Local development environment
├── .env.example                 # Environment variables template
├── alembic.ini                  # Database migration config
├── alembic/                     # Database migrations
│   ├── env.py
│   └── versions/
│       └── 001_initial.py
├── src/
│   └── banking_api/
│       ├── __init__.py
│       ├── main.py              # Application entry point
│       ├── config.py            # Settings management
│       ├── dependencies.py      # Shared dependencies
│       ├── middleware.py        # Custom middleware
│       ├── models/              # Database models (SQLAlchemy)
│       │   ├── __init__.py
│       │   ├── account.py
│       │   ├── payment.py
│       │   └── user.py
│       ├── schemas/             # Pydantic schemas
│       │   ├── __init__.py
│       │   ├── account.py
│       │   ├── payment.py
│       │   └── common.py
│       ├── api/                 # API routes
│       │   ├── __init__.py
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── accounts.py
│       │   │   ├── payments.py
│       │   │   └── router.py
│       │   └── dependencies.py
│       ├── services/            # Business logic
│       │   ├── __init__.py
│       │   ├── account_service.py
│       │   ├── payment_service.py
│       │   └── ai_service.py
│       ├── repositories/        # Data access layer
│       │   ├── __init__.py
│       │   ├── account_repo.py
│       │   └── payment_repo.py
│       ├── core/                # Core utilities
│       │   ├── __init__.py
│       │   ├── security.py
│       │   ├── logging.py
│       │   └── exceptions.py
│       └── workers/             # Background workers
│           ├── __init__.py
│           └── tasks.py
├── tests/
│   ├── conftest.py
│   ├── test_api/
│   ├── test_services/
│   └── test_repositories/
└── scripts/
    ├── create_db.py
    └── seed_data.py
```

## Application Entry Point

```python
# src/banking_api/main.py
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request, Response
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse

from banking_api.config import settings
from banking_api.api.v1.router import router as v1_router
from banking_api.core.logging import setup_logging
from banking_api.core.exceptions import BankingAPIError
from banking_api.middleware import CorrelationIdMiddleware

logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application startup and shutdown lifecycle."""
    # Startup
    logger.info('Starting Banking GenAI API')
    await setup_database()
    await setup_cache()
    yield
    # Shutdown
    logger.info('Shutting down Banking GenAI API')
    await teardown_database()
    await teardown_cache()


app = FastAPI(
    title='Banking GenAI Platform API',
    description='GenAI-powered banking services API',
    version='1.0.0',
    docs_url='/docs',
    redoc_url='/redoc',
    openapi_url='/openapi.json',
    lifespan=lifespan,
)

# Middleware
app.add_middleware(CorrelationIdMiddleware)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=['*'],
    allow_headers=['*'],
)

# Include routers
app.include_router(v1_router, prefix='/api')


@app.exception_handler(BankingAPIError)
async def banking_error_handler(request: Request, exc: BankingAPIError):
    """Handle domain-specific errors."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            'error': {
                'code': exc.error_code,
                'message': exc.message,
                'details': exc.details,
                'request_id': request.state.correlation_id,
            },
        },
    )


@app.exception_handler(Exception)
async def general_error_handler(request: Request, exc: Exception):
    """Catch-all for unexpected errors."""
    logger.exception('Unexpected error', extra={
        'correlation_id': getattr(request.state, 'correlation_id', None),
    })
    return JSONResponse(
        status_code=500,
        content={
            'error': {
                'code': 'INTERNAL_ERROR',
                'message': 'An unexpected error occurred',
                'request_id': getattr(request.state, 'correlation_id', None),
            },
        },
    )
```

## Configuration Management

```python
# src/banking_api/config.py
from pydantic_settings import BaseSettings
from pydantic import field_validator
from functools import lru_cache


class Settings(BaseSettings):
    """Application settings loaded from environment variables."""

    # Application
    app_name: str = 'banking-genai-api'
    environment: str = 'development'
    debug: bool = False

    # Database
    database_url: str
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = 'redis://localhost:6379/0'

    # Security
    jwt_secret_key: str
    jwt_algorithm: str = 'HS256'
    allowed_origins: list[str] = []

    # AI Service
    openai_api_key: str
    openai_model: str = 'gpt-4'
    ai_timeout: int = 60

    # Rate limiting
    rate_limit_per_minute: int = 1000

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
    """Cached settings singleton."""
    return Settings()


settings = get_settings()
```

## Request Validation with Pydantic

```python
# src/banking_api/schemas/payment.py
from pydantic import BaseModel, Field, field_validator
from decimal import Decimal
from datetime import datetime
from typing import Optional


class PaymentCreate(BaseModel):
    """Schema for creating a payment."""

    to_account: str = Field(
        ...,
        min_length=10,
        max_length=34,
        description='Recipient account number (IBAN format)',
    )
    amount: Decimal = Field(
        ...,
        gt=0,
        le=1_000_000,
        description='Payment amount (max $1M)',
    )
    currency: str = Field(
        ...,
        pattern='^[A-Z]{3}$',
        description='ISO 4217 currency code',
    )
    reference: Optional[str] = Field(
        None,
        max_length=140,
        description='Payment reference',
    )
    execution_date: Optional[datetime] = Field(
        None,
        description='Scheduled execution date (default: immediate)',
    )

    @field_validator('amount')
    @classmethod
    def validate_amount_precision(cls, v: Decimal) -> Decimal:
        """Ensure amount has at most 2 decimal places."""
        if v.as_tuple().exponent < -2:
            raise ValueError('Amount cannot have more than 2 decimal places')
        return v.quantize(Decimal('0.01'))

    @field_validator('to_account')
    @classmethod
    def validate_iban_format(cls, v: str) -> str:
        """Validate IBAN format."""
        if not v[0:2].isalpha():
            raise ValueError('IBAN must start with 2-letter country code')
        return v.upper()


class PaymentResponse(BaseModel):
    """Schema for payment response."""

    id: str
    status: str
    amount: Decimal
    currency: str
    to_account: str
    reference: Optional[str]
    created_at: datetime
    executed_at: Optional[datetime]

    class Config:
        from_attributes = True
```

## Dependency Injection

```python
# src/banking_api/dependencies.py
from fastapi import Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession

from banking_api.core.database import get_db_session
from banking_api.repositories.account_repo import AccountRepository
from banking_api.services.account_service import AccountService
from banking_api.services.ai_service import AIService


def get_account_service(
    db: AsyncSession = Depends(get_db_session),
) -> AccountService:
    """Provide AccountService with dependencies."""
    repo = AccountRepository(db)
    return AccountService(account_repo=repo)


def get_ai_service() -> AIService:
    """Provide AIService with dependencies."""
    return AIService()
```

## API Route Implementation

```python
# src/banking_api/api/v1/accounts.py
from fastapi import APIRouter, Depends, Query
from typing import Optional

from banking_api.schemas.account import AccountResponse, AccountList
from banking_api.dependencies import get_account_service
from banking_api.services.account_service import AccountService

router = APIRouter(prefix='/accounts', tags=['accounts'])


@router.get('/{account_id}', response_model=AccountResponse)
async def get_account(
    account_id: str,
    account_service: AccountService = Depends(get_account_service),
):
    """Retrieve account by ID."""
    return await account_service.get_account(account_id)


@router.get('/{account_id}/transactions', response_model=AccountList)
async def get_account_transactions(
    account_id: str,
    limit: int = Query(default=50, le=100),
    cursor: Optional[str] = None,
    account_service: AccountService = Depends(get_account_service),
):
    """Get account transactions with cursor-based pagination."""
    return await account_service.get_transactions(
        account_id,
        limit=limit,
        cursor=cursor,
    )


@router.get('/{account_id}/statements')
async def generate_statement(
    account_id: str,
    month: str = Query(pattern=r'^\d{4}-\d{2}$'),
    account_service: AccountService = Depends(get_account_service),
):
    """Generate monthly account statement."""
    return await account_service.generate_statement(account_id, month)
```

## Structured Logging

```python
# src/banking_api/core/logging.py
import logging
import json
import sys
from datetime import datetime


class JSONFormatter(logging.Formatter):
    """Format log records as JSON for structured logging."""

    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'message': record.getMessage(),
            'logger': record.name,
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno,
        }

        # Add extra fields
        if hasattr(record, 'correlation_id'):
            log_entry['correlation_id'] = record.correlation_id
        if hasattr(record, 'user_id'):
            log_entry['user_id'] = record.user_id
        if record.exc_info and record.exc_info[0]:
            log_entry['exception'] = self.formatException(record.exc_info)

        return json.dumps(log_entry)


def setup_logging(environment: str = 'development'):
    """Configure application-wide logging."""
    handler = logging.StreamHandler(sys.stdout)

    if environment == 'production':
        handler.setFormatter(JSONFormatter())
    else:
        handler.setFormatter(logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        ))

    root_logger = logging.getLogger()
    root_logger.addHandler(handler)
    root_logger.setLevel(logging.INFO)
```

## Security Middleware

```python
# src/banking_api/middleware.py
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import uuid


class CorrelationIdMiddleware(BaseHTTPMiddleware):
    """Add correlation ID to every request for distributed tracing."""

    async def dispatch(self, request: Request, call_next):
        # Get correlation ID from header or generate new one
        correlation_id = (
            request.headers.get('X-Correlation-ID')
            or str(uuid.uuid4())
        )
        request.state.correlation_id = correlation_id

        response = await call_next(request)

        # Add correlation ID to response
        response.headers['X-Correlation-ID'] = correlation_id

        return response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to all responses."""

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        response.headers['X-Content-Type-Options'] = 'nosniff'
        response.headers['X-Frame-Options'] = 'DENY'
        response.headers['X-XSS-Protection'] = '1; mode=block'
        response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
        response.headers['Content-Security-Policy'] = "default-src 'none'"

        return response
```

## Health Checks

```python
# src/banking_api/api/v1/health.py
from fastapi import APIRouter
from sqlalchemy import text

from banking_api.core.database import engine
from banking_api.core.redis import redis_client

router = APIRouter()


@router.get('/health')
async def health_check():
    """Basic health check."""
    return {'status': 'healthy'}


@router.get('/health/ready')
async def readiness_check():
    """Readiness check: verify all dependencies are available."""
    checks = {}

    # Database check
    try:
        async with engine.connect() as conn:
            await conn.execute(text('SELECT 1'))
        checks['database'] = 'ok'
    except Exception as e:
        checks['database'] = f'error: {e}'

    # Redis check
    try:
        await redis_client.ping()
        checks['redis'] = 'ok'
    except Exception as e:
        checks['redis'] = f'error: {e}'

    status = 'ready' if all(v == 'ok' for v in checks.values()) else 'not_ready'
    return {'status': status, 'checks': checks}


@router.get('/health/live')
async def liveness_check():
    """Liveness check: is the process running?"""
    return {'status': 'alive'}
```

## Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim AS base

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    POETRY_VERSION=1.7.1 \
    POETRY_HOME="/opt/poetry" \
    POETRY_VIRTUALENVS_CREATE=false

# Install Poetry
RUN pip install "poetry==$POETRY_VERSION"

WORKDIR /app

# Install dependencies first (leverage Docker cache)
COPY pyproject.toml poetry.lock ./
RUN poetry install --no-interaction --no-ansi --only main

# Copy application code
COPY src/ ./src/

# Create non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Run with uvicorn
CMD ["uvicorn", "banking_api.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

## Common Mistakes

1. **Putting business logic in routes**: Routes should only handle HTTP concerns
2. **No lifespan management**: Resources not cleaned up on shutdown
3. **Sync database calls in async routes**: Blocks the event loop
4. **Missing request validation**: Relying on manual validation instead of Pydantic
5. **No correlation ID**: Cannot trace requests across services
6. **Hardcoded configuration**: Not using environment variables
7. **No health checks**: Kubernetes cannot determine pod readiness

## Interview Questions

1. How do you structure a FastAPI project with 20+ endpoints and multiple domains?
2. Design a middleware pipeline with authentication, logging, and rate limiting.
3. How do you handle graceful shutdown of async database connections?
4. What is the difference between dependency injection in FastAPI vs Spring?
5. How do you implement cursor-based pagination in a FastAPI endpoint?

## Cross-References

- `./pydantic.md` - Data validation with Pydantic
- `./async-python.md` - Async database sessions and HTTP clients
- `./sqlalchemy.md` - SQLAlchemy async ORM patterns
- `../api-design/README.md` - REST API design principles
- `../microservices/README.md` - Microservice architecture
