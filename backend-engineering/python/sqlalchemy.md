# SQLAlchemy ORM Patterns

## Why SQLAlchemy

SQLAlchemy is Python's most mature and flexible ORM. In banking, it provides type-safe database access, connection pooling, migration support, and query optimization. Its async support (SQLAlchemy 2.0) integrates with FastAPI and asyncio for high-performance database operations.

## Production Setup

```python
# database.py
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import MetaData

# Naming convention for constraints
naming_convention = {
    'ix': 'ix_%(column_0_label)s',
    'uq': 'uq_%(table_name)s_%(column_0_name)s',
    'ck': 'ck_%(table_name)s_`%(constraint_name)s`',
    'fk': 'fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s',
    'pk': 'pk_%(table_name)s',
}

metadata = MetaData(naming_convention=naming_convention)


class Base(DeclarativeBase):
    metadata = metadata


# Engine configuration
engine = create_async_engine(
    'postgresql+asyncpg://user:pass@localhost:5432/banking',
    pool_size=20,           # Persistent connections
    max_overflow=10,        # Temporary overflow
    pool_timeout=30,        # Wait for connection
    pool_recycle=1800,      # Recycle after 30 min
    pool_pre_ping=True,     # Health check before use
    echo=False,             # SQL logging (enable for debugging)
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

## Model Definitions

```python
# models/account.py
from sqlalchemy import (
    Column,
    String,
    Numeric,
    DateTime,
    Enum,
    Index,
    CheckConstraint,
)
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime, timezone
import uuid
from decimal import Decimal
from enum import Enum as PyEnum


class AccountStatus(PyEnum):
    PENDING = 'pending'
    ACTIVE = 'active'
    SUSPENDED = 'suspended'
    CLOSED = 'closed'


class Account(Base):
    __tablename__ = 'accounts'

    id: Mapped[str] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
    )
    customer_id: Mapped[str] = mapped_column(
        UUID(as_uuid=True),
        nullable=False,
        index=True,
    )
    account_number: Mapped[str] = mapped_column(
        String(34),
        unique=True,
        nullable=False,
        index=True,
    )
    account_type: Mapped[str] = mapped_column(
        String(20),
        nullable=False,
    )
    currency: Mapped[str] = mapped_column(
        String(3),
        nullable=False,
    )
    balance: Mapped[Decimal] = mapped_column(
        Numeric(19, 2),  # 19 digits, 2 decimal places
        nullable=False,
        default=Decimal('0.00'),
    )
    status: Mapped[AccountStatus] = mapped_column(
        Enum(AccountStatus),
        nullable=False,
        default=AccountStatus.PENDING,
    )
    version: Mapped[int] = mapped_column(
        nullable=False,
        default=0,
        server_default='0',
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(timezone.utc),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

    # Relationships
    transactions: Mapped[list['Transaction']] = relationship(
        back_populates='account',
        lazy='selectin',  # Eager loading
    )

    # Constraints
    __table_args__ = (
        CheckConstraint('balance >= -10000', name='ck_accounts_min_balance'),
        Index('idx_accounts_customer_status', 'customer_id', 'status'),
    )

    def deposit(self, amount: Decimal):
        if amount <= 0:
            raise ValueError('Deposit amount must be positive')
        self.balance += amount
        self.version += 1

    def withdraw(self, amount: Decimal):
        if amount <= 0:
            raise ValueError('Withdrawal amount must be positive')
        if self.balance - amount < Decimal('-10000'):
            raise InsufficientFundsError(
                f'Balance {self.balance} insufficient for withdrawal {amount}'
            )
        self.balance -= amount
        self.version += 1
```

## Repository Pattern

```python
# repositories/account_repo.py
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload


class AccountRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, account_id: str) -> Account | None:
        """Fetch account by ID with eager-loaded transactions."""
        stmt = (
            select(Account)
            .options(selectinload(Account.transactions))
            .where(Account.id == account_id)
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_by_account_number(self, account_number: str) -> Account | None:
        """Fetch account by account number."""
        stmt = select(Account).where(Account.account_number == account_number)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, data: dict) -> Account:
        """Create new account."""
        account = Account(**data)
        self.session.add(account)
        await self.session.flush()
        await self.session.refresh(account)
        return account

    async def update_balance(
        self,
        account_id: str,
        amount: Decimal,
        expected_version: int,
    ) -> Account:
        """Update balance with optimistic locking."""
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
            account.deposit(amount)
        else:
            account.withdraw(abs(amount))

        await self.session.flush()
        return account

    async def list_by_customer(
        self,
        customer_id: str,
        limit: int = 50,
        cursor: str | None = None,
    ) -> list[Account]:
        """List accounts for customer with cursor pagination."""
        stmt = select(Account).where(Account.customer_id == customer_id)

        if cursor:
            stmt = stmt.where(Account.id > cursor)

        stmt = stmt.order_by(Account.id).limit(limit)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

## Query Optimization

```python
# N+1 Problem: Loading related objects one at a time
async def get_accounts_bad(customer_ids: list[str]) -> list[Account]:
    """N+1 queries: 1 for accounts + N for customer details."""
    accounts = await session.execute(
        select(Account).where(Account.customer_id.in_(customer_ids))
    )
    result = []
    for account in accounts.scalars():
        # Each iteration triggers a query!
        customer = await session.execute(
            select(Customer).where(Customer.id == account.customer_id)
        )
        result.append({
            'account': account,
            'customer': customer.scalar_one(),
        })
    return result


# Solution: Eager loading with joinedload/selectinload
async def get_accounts_good(customer_ids: list[str]) -> list[Account]:
    """2 queries: 1 for accounts + 1 for customers."""
    stmt = (
        select(Account)
        .options(selectinload(Account.customer))  # Single query for all customers
        .where(Account.customer_id.in_(customer_ids))
    )
    result = await session.execute(stmt)
    return list(result.scalars().unique().all())
```

## Bulk Operations

```python
async def bulk_insert_accounts(accounts_data: list[dict]):
    """Bulk insert with single query."""
    await session.execute(
        Account.__table__.insert(),
        accounts_data,
    )
    await session.commit()


async def bulk_update_balances(updates: list[dict]):
    """Bulk update with case statement."""
    from sqlalchemy import case

    account_ids = [u['id'] for u in updates]

    # Single UPDATE with CASE
    await session.execute(
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
    await session.commit()
```

## Migrations with Alembic

```python
# alembic/versions/001_create_accounts.py
"""Create accounts table.

Revision ID: 001
Revises:
Create Date: 2026-04-10
"""
from alembic import op
import sqlalchemy as sa

revision = '001'
down_revision = None
branch_labels = None
depends_on = None


def upgrade():
    op.create_table(
        'accounts',
        sa.Column('id', sa.UUID(), primary_key=True),
        sa.Column('customer_id', sa.UUID(), nullable=False, index=True),
        sa.Column('account_number', sa.String(34), unique=True, nullable=False),
        sa.Column('account_type', sa.String(20), nullable=False),
        sa.Column('currency', sa.String(3), nullable=False),
        sa.Column('balance', sa.Numeric(19, 2), nullable=False, server_default='0'),
        sa.Column('status', sa.Enum('pending', 'active', 'suspended', 'closed'), nullable=False),
        sa.Column('version', sa.Integer(), nullable=False, server_default='0'),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False),
        sa.Column('updated_at', sa.DateTime(timezone=True), nullable=False),
        sa.CheckConstraint('balance >= -10000', name='ck_accounts_min_balance'),
        sa.Index('idx_accounts_customer_status', 'customer_id', 'status'),
    )


def downgrade():
    op.drop_table('accounts')
```

## Common Mistakes

1. **N+1 queries**: Not using eager loading for relationships
2. **No connection pooling**: Creating new connection per request
3. **expire_on_commit=True**: Accessing attributes after commit triggers lazy load
4. **Not using optimistic locking**: Concurrent updates lose data
5. **Large transactions**: Holding transactions open too long
6. **Ignoring indexes**: Slow queries on unindexed columns
7. **Using sync SQLAlchemy in async code**: Blocks event loop

## Interview Questions

1. How do you prevent N+1 queries when fetching accounts with their transactions?
2. Design optimistic locking for concurrent balance updates.
3. How do you structure a bulk update for 10,000 account balances?
4. What is the difference between `selectinload` and `joinedload`?
5. How do you handle database migrations in a zero-downtime deployment?

## Cross-References

- `./fastapi.md` - Async database session dependency
- `./async-python.md` - Async SQLAlchemy patterns
- `../concurrency/README.md` - Optimistic vs pessimistic locking
- `../backend-testing/README.md` - Testing with test databases
