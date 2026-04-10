# Pydantic Data Validation

## Why Pydantic

Pydantic provides runtime type validation using Python type hints. In banking, data validation is critical: malformed payment amounts, invalid account numbers, and missing KYC fields cause financial losses and regulatory violations. Pydantic catches these errors at the API boundary before they reach business logic.

## Core Validation Patterns

### Basic Models

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from decimal import Decimal
from datetime import datetime
from typing import Optional, Literal
from enum import Enum


class Currency(str, Enum):
    USD = 'USD'
    EUR = 'EUR'
    GBP = 'GBP'
    JPY = 'JPY'


class PaymentStatus(str, Enum):
    PENDING = 'pending'
    PROCESSING = 'processing'
    COMPLETED = 'completed'
    FAILED = 'failed'
    REVERSED = 'reversed'


class PaymentCreate(BaseModel):
    """Schema for creating a payment."""

    from_account: str = Field(
        ...,
        min_length=10,
        max_length=34,
        description='Source account number',
    )
    to_account: str = Field(
        ...,
        min_length=10,
        max_length=34,
        description='Destination account number',
    )
    amount: Decimal = Field(
        ...,
        gt=0,
        le=Decimal('1000000.00'),
        description='Payment amount (max 1M)',
    )
    currency: Currency = Field(
        ...,
        description='ISO 4217 currency code',
    )
    reference: Optional[str] = Field(
        None,
        max_length=140,
        description='Payment reference',
    )
    execution_date: Optional[datetime] = Field(
        None,
        description='Scheduled execution (default: immediate)',
    )
    priority: Literal['normal', 'urgent', 'critical'] = Field(
        'normal',
        description='Payment processing priority',
    )
```

### Custom Validators

```python
class PaymentCreate(BaseModel):
    # ... fields as above ...

    @field_validator('from_account', 'to_account')
    @classmethod
    def validate_account_format(cls, v: str) -> str:
        """Validate account number format."""
        v = v.strip().upper()
        if not v[0:2].isalpha():
            raise ValueError('Account must start with 2-letter country code')
        if not v[2:4].isdigit():
            raise ValueError('Account must have 2-digit check number')
        return v

    @field_validator('amount')
    @classmethod
    def validate_amount_precision(cls, v: Decimal) -> Decimal:
        """Round to 2 decimal places."""
        return v.quantize(Decimal('0.01'))

    @model_validator(mode='after')
    def validate_not_self_payment(self) -> 'PaymentCreate':
        """Prevent payments to self."""
        if self.from_account == self.to_account:
            raise ValueError('Source and destination accounts must differ')
        return self

    @model_validator(mode='after')
    def validate_execution_date(self) -> 'PaymentCreate':
        """Execution date must be in the future."""
        if self.execution_date and self.execution_date < datetime.utcnow():
            raise ValueError('Execution date must be in the future')
        return self
```

### Nested Models

```python
class Address(BaseModel):
    street: str = Field(..., max_length=100)
    city: str = Field(..., max_length=50)
    state: str = Field(..., max_length=50)
    postal_code: str = Field(..., pattern=r'^\d{5}(-\d{4})?$')
    country: str = Field(..., pattern=r'^[A-Z]{2}$')


class CustomerKYC(BaseModel):
    """Nested KYC information for customer."""

    full_name: str = Field(..., min_length=2, max_length=100)
    date_of_birth: datetime = Field(...)
    nationality: str = Field(..., pattern=r'^[A-Z]{2}$')
    id_document_type: Literal['passport', 'national_id', 'driving_license']
    id_document_number: str = Field(..., min_length=5, max_length=20)
    address: Address
    risk_category: Literal['low', 'medium', 'high'] = 'low'
    politically_exposed: bool = False

    @field_validator('date_of_birth')
    @classmethod
    def validate_age(cls, v: datetime) -> datetime:
        """Customer must be at least 18 years old."""
        age = (datetime.utcnow() - v).days / 365.25
        if age < 18:
            raise ValueError('Customer must be at least 18 years old')
        return v


class AccountCreate(BaseModel):
    """Account creation with nested KYC."""

    customer: CustomerKYC
    account_type: Literal['checking', 'savings', 'business']
    currency: Currency = Currency.USD
    initial_deposit: Decimal = Field(
        Decimal('0.00'),
        ge=0,
        description='Initial deposit amount',
    )
```

### Generic Models

```python
from typing import Generic, TypeVar, Optional

T = TypeVar('T')


class APIResponse(BaseModel, Generic[T]):
    """Generic API response wrapper."""

    data: T
    meta: Optional[dict] = None


class PaginatedResponse(BaseModel, Generic[T]):
    """Paginated response wrapper."""

    items: list[T]
    pagination: dict = Field(..., description='Pagination metadata')


class ErrorResponse(BaseModel):
    """Standardized error response."""

    error: dict = Field(..., description='Error details')


# Usage
class AccountResponse(BaseModel):
    id: str
    name: str
    balance: Decimal
    currency: str


# Returns PaginatedResponse[AccountResponse]
async def list_accounts(
    limit: int = 50,
    cursor: Optional[str] = None,
) -> PaginatedResponse[AccountResponse]:
    accounts, next_cursor = await repo.list(limit=limit, cursor=cursor)
    return PaginatedResponse(
        items=[AccountResponse(**a) for a in accounts],
        pagination={
            'next_cursor': next_cursor,
            'has_more': next_cursor is not None,
            'limit': limit,
        },
    )
```

### Validation with External Data

```python
class PaymentUpdate(BaseModel):
    """Schema for updating payment (partial update)."""

    status: Optional[PaymentStatus] = None
    reference: Optional[str] = None

    class Config:
        # Allow missing fields (partial update)
        pass


# Validate incoming request
def validate_payment_update(data: dict) -> PaymentUpdate:
    try:
        return PaymentUpdate.model_validate(data)
    except ValidationError as e:
        raise HTTPException(
            status_code=400,
            detail=e.errors(),
        )


# Validate with context
class PaymentContext:
    """External context for validation (e.g., account balance)."""

    def validate_with_context(self, payment: PaymentCreate, account_balance: Decimal):
        """Validate payment against account balance."""
        if payment.amount > account_balance:
            raise ValueError(
                f'Insufficient funds: available {account_balance}, '
                f'requested {payment.amount}'
            )

        # Check daily limit
        daily_total = self.get_daily_total(payment.from_account)
        if daily_total + payment.amount > Decimal('50000'):
            raise ValueError('Daily transfer limit exceeded ($50,000)')
```

## Banking-Specific Validators

### IBAN Validation

```python
def validate_iban(iban: str) -> bool:
    """Validate IBAN using modulo-97 algorithm."""
    # Rearrange IBAN
    rearranged = iban[4:] + iban[:4]
    # Convert letters to numbers
    numeric = ''.join(
        str(ord(c) - 55) if c.isalpha() else c
        for c in rearranged
    )
    # Modulo 97
    return int(numeric) % 97 == 1


class IBANField:
    """Pydantic validator for IBAN."""

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler):
        from pydantic_core import core_schema
        return core_schema.no_info_plain_validator_function(cls.validate)

    @classmethod
    def validate(cls, v: str) -> str:
        v = v.replace(' ', '').upper()
        if not validate_iban(v):
            raise ValueError('Invalid IBAN')
        return v
```

### SWIFT/BIC Validation

```python
class SWIFTCode:
    """Validate SWIFT/BIC code."""

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler):
        from pydantic_core import core_schema
        return core_schema.no_info_plain_validator_function(cls.validate)

    @classmethod
    def validate(cls, v: str) -> str:
        v = v.strip().upper()
        if not re.match(r'^[A-Z]{4}[A-Z]{2}[A-Z0-9]{2}([A-Z0-9]{3})?$', v):
            raise ValueError('Invalid SWIFT/BIC code')
        return v
```

## Discriminated Unions

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


# Discriminated union: type field determines which schema to use
PaymentMethod = Annotated[
    Union[WireTransfer, SEPATransfer, DomesticTransfer],
    Field(discriminator='type'),
]


class PaymentCreate(BaseModel):
    amount: Decimal
    currency: str
    method: PaymentMethod


# Usage: FastAPI automatically validates based on 'type' field
# {"amount": 100, "currency": "EUR", "method": {"type": "sepa", "iban": "...", ...}}
```

## Serialization

```python
class PaymentResponse(BaseModel):
    id: str
    amount: Decimal
    currency: str
    status: str
    created_at: datetime
    executed_at: Optional[datetime] = None

    class Config:
        # Convert Decimal to string for JSON serialization
        json_encoders = {
            Decimal: lambda v: str(v),
        }


# Serialize to JSON
payment = PaymentResponse(
    id='PAY-001',
    amount=Decimal('1000.00'),
    currency='USD',
    status='completed',
    created_at=datetime.utcnow(),
)

json_str = payment.model_dump_json()
# {"id":"PAY-001","amount":"1000.00","currency":"USD",...}

# Or as dict
data = payment.model_dump(mode='json')
# {'id': 'PAY-001', 'amount': '1000.00', ...}
```

## Performance

```python
# Pydantic v2 uses Rust core for 5-50x faster validation
from pydantic import TypeAdapter

# For validating many items of same type
payments = [
    {'amount': 100, 'currency': 'USD', ...},
    {'amount': 200, 'currency': 'EUR', ...},
    # ... thousands of records
]

# TypeAdapter for batch validation
adapter = TypeAdapter(list[PaymentCreate])
validated_payments = adapter.validate_python(payments)
```

## Common Mistakes

1. **No custom validators**: Only using built-in constraints, missing domain rules
2. **Mutable default values**: `items: list = []` instead of `items: list = field(default_factory=list)`
3. **Ignoring model validators**: Only using field validators, missing cross-field validation
4. **Not using discriminated unions**: Manual type checking instead of Pydantic's union types
5. **Serializing Decimal as float**: Losing precision in JSON output
6. **Over-validating**: Validating at multiple layers (API + service + repository)

## Interview Questions

1. How do you validate a payment request against both format rules and account balance?
2. Design a discriminated union for different payment methods (wire, SEPA, domestic).
3. How do you handle partial updates (PATCH) with Pydantic?
4. What is the difference between `field_validator` and `model_validator`?
5. How do you serialize Decimal to JSON without losing precision?

## Cross-References

- `./fastapi.md` - Using Pydantic schemas in FastAPI routes
- `../api-design/README.md` - API validation and error responses
- `../backend-testing/README.md` - Testing validation rules
- `./pydantic.md` - This file (data validation)
