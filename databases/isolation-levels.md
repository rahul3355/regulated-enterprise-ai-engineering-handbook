# Transaction Isolation Levels

## Overview

Transaction isolation levels control how concurrent transactions interact with each other. PostgreSQL supports three of the four SQL standard isolation levels, each with different guarantees and performance characteristics. Understanding isolation is critical for banking systems where data consistency is paramount.

## Isolation Levels in PostgreSQL

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Banking Use Case |
|----------------|-----------|-------------------|--------------|-----------------|
| Read Committed | Impossible | Possible | Possible | Default, general OLTP |
| Repeatable Read | Impossible | Impossible | Impossible* | Financial calculations |
| Serializable | Impossible | Impossible | Impossible | Critical financial operations |

*PostgreSQL's Repeatable Read also prevents phantom reads (stricter than SQL standard).

## Read Committed (Default)

```sql
-- Read Committed is PostgreSQL's default isolation level
-- Each statement sees a snapshot as of the start of that statement

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Example: Two concurrent transactions
-- Transaction A                          Transaction B
BEGIN;                                    BEGIN;
SELECT balance FROM accounts              -- Sees balance = 1000
WHERE account_id = 1;                     
                                        UPDATE accounts 
                                        SET balance = 1500 
                                        WHERE account_id = 1;
                                        COMMIT;
                                        
SELECT balance FROM accounts              -- Sees balance = 1500 (NEW snapshot!)
WHERE account_id = 1;                     
COMMIT;

-- Implication: Within the same transaction, different statements
-- may see different data if other transactions commit in between.
```

### Banking Impact with Read Committed

```sql
-- Problem: Balance check may be stale by time of update
-- Transaction A                          Transaction B
BEGIN;                                    BEGIN;
SELECT balance FROM accounts              -- Sees balance = 1000
WHERE account_id = 1;                     
-- Application checks: 1000 >= 800? Yes! 
                                        UPDATE accounts 
                                        SET balance = 200  -- Withdrew 800
                                        WHERE account_id = 1;
                                        COMMIT;

UPDATE accounts SET balance = 200         -- Deducts 800 from 1000
WHERE account_id = 1;                     -- But balance was already 200!
-- Now balance = 200 (should have been -600, but constraint prevents)
COMMIT;

-- Solution: Use SELECT FOR LOCK or higher isolation level
BEGIN;
SELECT balance FROM accounts 
WHERE account_id = 1 
FOR UPDATE;  -- Locks the row

UPDATE accounts SET balance = balance - 800
WHERE account_id = 1;
COMMIT;
```

## Repeatable Read

```sql
-- Repeatable Read: Snapshot taken at transaction start
-- All statements in the transaction see the same data

SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Transaction A                          Transaction B
BEGIN;                                    BEGIN;
SELECT balance FROM accounts              -- Sees balance = 1000
WHERE account_id = 1;                     
                                        UPDATE accounts 
                                        SET balance = 1500 
                                        WHERE account_id = 1;
                                        COMMIT;
                                        
SELECT balance FROM accounts              -- Still sees balance = 1000
WHERE account_id = 1;                     -- (same snapshot as start)
COMMIT;

-- Serialization failure: If both transactions try to UPDATE the same row,
-- the second will fail with "could not serialize access"
-- Application must retry the transaction.
```

### Banking Use Case: End-of-Day Balance Calculation

```sql
-- Repeatable Read ensures consistent aggregation
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;

-- All these queries see the same snapshot
SELECT 
    account_type,
    COUNT(*) AS account_count,
    SUM(balance) AS total_balance,
    AVG(balance) AS avg_balance
FROM accounts
WHERE status = 'ACTIVE'
GROUP BY account_type;

-- These numbers are internally consistent
-- No risk of seeing accounts that changed mid-calculation

COMMIT;
```

## Serializable

```sql
-- Serializable: Strictest isolation
-- Ensures transactions could have run sequentially
-- Conflicts detected and aborted automatically

SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Transaction A                          Transaction B
BEGIN;                                    BEGIN;
SELECT SUM(balance)                       SELECT SUM(balance)
FROM accounts;                            FROM accounts;
                                        -- Both read same data
                                        
UPDATE accounts SET balance = balance     UPDATE accounts 
+ 100 WHERE account_id = 1;               SET balance = balance 
                                          + 200 WHERE account_id = 2;
COMMIT;                                   
                                        -- CONFLICT: B's write depends
                                        -- on data A also wrote to
                                        COMMIT;  -- FAILS with serialization error
                                        
-- Application must retry Transaction B

-- Banking use case: Critical operations where consistency is non-negotiable
-- Account closure, interest calculation, regulatory reporting
```

## SELECT ... FOR UPDATE and Row Locking

```sql
-- Explicit row locking for critical operations

-- FOR UPDATE: Exclusive lock (blocks all other locks)
SELECT * FROM accounts 
WHERE account_id = 1 
FOR UPDATE;

-- FOR NO KEY UPDATE: Locks row but allows other FOR KEY SHARE
SELECT * FROM accounts 
WHERE account_id = 1 
FOR NO KEY UPDATE;

-- FOR SHARE: Share lock (allows other FOR SHARE, blocks FOR UPDATE)
SELECT * FROM accounts 
WHERE account_id = 1 
FOR SHARE;

-- FOR KEY SHARE: Weakest lock (only blocks FOR UPDATE)
SELECT * FROM accounts 
WHERE account_id = 1 
FOR KEY SHARE;

-- NOWAIT: Fail immediately if row is locked
SELECT * FROM accounts 
WHERE account_id = 1 
FOR UPDATE NOWAIT;

-- SKIP LOCKED: Skip locked rows (useful for queue processing)
SELECT * FROM pending_transfers 
ORDER BY created_at 
LIMIT 10 
FOR UPDATE SKIP LOCKED;

-- WAIT N: Wait N seconds before failing
SELECT * FROM accounts 
WHERE account_id = 1 
FOR UPDATE WAIT 5;
```

## Choosing the Right Isolation Level

```
Read Committed (default):
- Use for: General OLTP, most API requests
- Pros: Best concurrency, no retries needed
- Cons: Must handle concurrent modification explicitly

Repeatable Read:
- Use for: Reports, aggregations, multi-step reads
- Pros: Consistent snapshot, no non-repeatable reads
- Cons: Serialization failures require retry logic

Serializable:
- Use for: Critical financial operations, account closure
- Pros: Strongest consistency guarantee
- Cons: Highest contention, frequent serialization failures

Rule of thumb:
- Start with Read Committed + explicit locking (FOR UPDATE)
- Use Repeatable Read for consistent reads
- Use Serializable only when correctness depends on it
```

## Cross-References

- **Transactions**: See [transactions.md](transactions.md) for ACID properties
- **Deadlocks**: See [deadlocks.md](deadlocks.md) for lock conflicts
- **Row-Level Security**: See [row-level-security.md](row-level-security.md) for access control

## Interview Questions

1. **What is the default isolation level in PostgreSQL? Why was it chosen?**
2. **How does Repeatable Read differ between PostgreSQL and the SQL standard?**
3. **When would you use Serializable isolation in a banking application?**
4. **What is the difference between FOR UPDATE and FOR SHARE?**
5. **Your Serializable transaction keeps failing with serialization errors. What do you do?**
6. **How does MVCC enable PostgreSQL's isolation levels?**

## Checklist: Isolation Level Selection

- [ ] Default Read Committed used for most OLTP operations
- [ ] Explicit row locking (FOR UPDATE) for read-modify-write patterns
- [ ] Repeatable Read used for consistent aggregations
- [ ] Serializable used only for critical financial operations
- [ ] Retry logic implemented for serialization failures
- [ ] lock_timeout configured to prevent indefinite waits
- [ ] NOWAIT or SKIP LOCKED used for queue processing
- [ ] Isolation level documented for critical operations
- [ ] Tested behavior under concurrent access
