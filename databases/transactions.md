# Transactions and ACID Properties

## Overview

ACID (Atomicity, Consistency, Isolation, Durability) guarantees ensure reliable financial transactions in PostgreSQL. This guide covers transaction internals, ACID properties, savepoints, and banking-specific transaction patterns.

## ACID Properties

```
Atomicity: All or nothing
  - Transaction either fully commits or fully rolls back
  - No partial updates visible to other transactions
  - Implemented via WAL (Write-Ahead Logging)

Consistency: Database moves from valid state to valid state
  - All constraints, triggers, and rules enforced
  - Application logic must maintain business rules
  - Database enforces structural consistency (FKs, checks)

Isolation: Transactions don't interfere with each other
  - Each transaction sees a consistent snapshot
  - Concurrent transactions appear to run sequentially
  - Implemented via MVCC (Multi-Version Concurrency Control)

Durability: Committed data survives crashes
  - WAL flushed to disk before commit acknowledged
  - Crash recovery replays WAL to restore state
  - synchronous_commit controls durability guarantee
```

## Transaction Internals

```sql
-- Every transaction gets a unique Transaction ID (XID)
-- XIDs are 32-bit integers (wrap around after ~4 billion)

-- Start a transaction
BEGIN;

-- Each statement gets a Command ID (CID) within the transaction
INSERT INTO accounts (account_id, balance) VALUES (1, 1000);  -- CID = 0
UPDATE accounts SET balance = 1500 WHERE account_id = 1;      -- CID = 1
DELETE FROM accounts WHERE account_id = 2;                    -- CID = 2

-- Transaction metadata stored in tuple headers:
-- t_xmin: XID of inserting transaction
-- t_xmax: XID of deleting/updating transaction (0 if not deleted)
-- t_cid: Command ID within the transaction

-- Commit: makes all changes visible
COMMIT;

-- Rollback: undoes all changes
ROLLBACK;
```

## Savepoints

```sql
-- Savepoints allow partial rollback within a transaction
BEGIN;

INSERT INTO accounts (account_id, balance) VALUES (1, 1000);
INSERT INTO accounts (account_id, balance) VALUES (2, 2000);

SAVEPOINT before_third_account;

INSERT INTO accounts (account_id, balance) VALUES (3, 3000);
-- Oops, this should be 300
ROLLBACK TO SAVEPOINT before_third_account;

INSERT INTO accounts (account_id, balance) VALUES (3, 300);

COMMIT;

-- Banking pattern: transfer with savepoint
CREATE OR REPLACE FUNCTION transfer_with_savepoint(
    from_account BIGINT,
    to_account BIGINT,
    amount DECIMAL
) RETURNS void AS $$
BEGIN
    -- Validate
    IF amount <= 0 THEN
        RAISE EXCEPTION 'Transfer amount must be positive';
    END IF;
    
    BEGIN
        -- Try the transfer
        UPDATE accounts SET balance = balance - amount 
        WHERE account_id = from_account AND balance >= amount;
        
        IF NOT FOUND THEN
            RAISE EXCEPTION 'Insufficient funds';
        END IF;
        
        SAVEPOINT transfer_debit;
        
        UPDATE accounts SET balance = balance + amount 
        WHERE account_id = to_account;
        
        -- Log the transfer
        INSERT INTO transfers (from_account, to_account, amount, created_at)
        VALUES (from_account, to_account, amount, NOW());
        
        COMMIT;
        
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
    END;
END;
$$ LANGUAGE plpgsql;
```

## Two-Phase Commit (Distributed Transactions)

```sql
-- Two-phase commit for distributed transactions
-- Rarely used in modern banking (prefer saga pattern)

-- Phase 1: Prepare
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
PREPARE TRANSACTION 'transfer_1_to_2';

-- Phase 2: Commit (on same or different connection)
COMMIT PREPARED 'transfer_1_to_2';

-- Or rollback
ROLLBACK PREPARED 'transfer_1_to_2';

-- WARNING: Prepared transactions hold locks until committed/rolled back
-- Check for lingering prepared transactions
SELECT * FROM pg_prepared_xacts;

-- In modern banking: Use saga pattern with compensation instead
```

## Saga Pattern for Banking

```python
"""
Saga pattern: Distributed transaction with compensating actions.
Preferred over two-phase commit in modern distributed systems.
"""
from typing import Callable, List
from dataclasses import dataclass

@dataclass
class SagaStep:
    action: Callable
    compensation: Callable
    description: str

class SagaOrchestrator:
    """Execute distributed transaction with rollback capability."""
    
    def __init__(self, steps: List[SagaStep]):
        self.steps = steps
        self.completed_steps = []
    
    def execute(self, context: dict) -> dict:
        """Execute all steps, compensating on failure."""
        try:
            for step in self.steps:
                step.action(context)
                self.completed_steps.append(step)
            return context
        except Exception as e:
            # Compensate in reverse order
            for step in reversed(self.completed_steps):
                try:
                    step.compensation(context)
                except Exception as comp_error:
                    # Log compensation failure - needs manual intervention
                    logger.critical(
                        f"Compensation failed for {step.description}: {comp_error}"
                    )
            raise

# Banking transfer saga
def transfer_saga(from_account: int, to_account: int, amount: float):
    context = {
        'from_account': from_account,
        'to_account': to_account,
        'amount': amount,
    }
    
    saga = SagaOrchestrator([
        SagaStep(
            action=lambda ctx: debit_account(ctx['from_account'], ctx['amount']),
            compensation=lambda ctx: credit_account(ctx['from_account'], ctx['amount']),
            description=f"Debit account {ctx['from_account']}",
        ),
        SagaStep(
            action=lambda ctx: credit_account(ctx['to_account'], ctx['amount']),
            compensation=lambda ctx: debit_account(ctx['to_account'], ctx['amount']),
            description=f"Credit account {ctx['to_account']}",
        ),
        SagaStep(
            action=lambda ctx: log_transfer(ctx['from_account'], ctx['to_account'], ctx['amount']),
            compensation=lambda ctx: void_transfer_log(ctx['transfer_id']),
            description="Log transfer",
        ),
    ])
    
    return saga.execute(context)
```

## Cross-References

- **Isolation Levels**: See [isolation-levels.md](isolation-levels.md) for isolation behavior
- **Deadlocks**: See [deadlocks.md](deadlocks.md) for deadlock handling
- **Migrations**: See [migrations.md](migrations.md) for safe schema changes

## Interview Questions

1. **Explain ACID properties. How does PostgreSQL implement each one?**
2. **What happens when you COMMIT a transaction? What about ROLLBACK?**
3. **How do savepoints work? When would you use them?**
4. **What is the difference between two-phase commit and the saga pattern?**
5. **A transfer debited the source account but failed to credit the destination. How do you recover?**
6. **What is the impact of long-running transactions on PostgreSQL performance?**

## Checklist: Transaction Best Practices

- [ ] Transactions kept short (under 1 second for OLTP)
- [ ] Savepoints used for complex multi-step operations
- [ ] Explicit BEGIN/COMMIT (not autocommit for multi-statement)
- [ ] Error handling with proper ROLLBACK on failure
- [ ] Saga pattern used for distributed transactions
- [ ] Idle-in-transaction timeout configured
- [ ] Long-running transactions monitored and alerted
- [ ] No user interaction within database transactions
- [ ] Compensation actions tested for all saga steps
- [ ] Transaction retry logic for transient failures
