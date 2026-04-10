# Banking-Specific Database Patterns

## Overview

Banking systems have unique database requirements driven by regulatory requirements, financial accuracy, and audit mandates. This guide covers database patterns specific to banking: double-entry bookkeeping, audit trails, idempotent transactions, and regulatory data structures.

## Double-Entry Bookkeeping

```sql
-- Core banking principle: Every transaction has equal debit and credit
-- Total debits must always equal total credits

-- Journal entries table
CREATE TABLE journal_entries (
    entry_id BIGINT PRIMARY KEY,
    transaction_id VARCHAR(36) NOT NULL,
    account_id BIGINT NOT NULL,
    entry_type VARCHAR(10) CHECK (entry_type IN ('DEBIT', 'CREDIT')),
    amount DECIMAL(15, 2) NOT NULL CHECK (amount > 0),
    currency VARCHAR(3) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    posted_at TIMESTAMPTZ
);

-- Validate double-entry: debits = credits for each transaction
CREATE OR REPLACE FUNCTION validate_double_entry(p_transaction_id VARCHAR)
RETURNS BOOLEAN AS $$
DECLARE
    v_debits DECIMAL;
    v_credits DECIMAL;
BEGIN
    SELECT COALESCE(SUM(amount), 0) INTO v_debits
    FROM journal_entries
    WHERE transaction_id = p_transaction_id AND entry_type = 'DEBIT';
    
    SELECT COALESCE(SUM(amount), 0) INTO v_credits
    FROM journal_entries
    WHERE transaction_id = p_transaction_id AND entry_type = 'CREDIT';
    
    RETURN v_debits = v_credits AND v_debits > 0;
END;
$$ LANGUAGE plpgsql;

-- Transfer function with double-entry
CREATE OR REPLACE FUNCTION record_transfer(
    p_transaction_id VARCHAR,
    p_from_account BIGINT,
    p_to_account BIGINT,
    p_amount DECIMAL,
    p_currency VARCHAR,
    p_description TEXT
) RETURNS void AS $$
BEGIN
    -- Debit from source account
    INSERT INTO journal_entries (entry_id, transaction_id, account_id, entry_type, amount, currency, description)
    VALUES (nextval('journal_entry_seq'), p_transaction_id, p_from_account, 'DEBIT', p_amount, p_currency, p_description);
    
    -- Credit to destination account
    INSERT INTO journal_entries (entry_id, transaction_id, account_id, entry_type, amount, currency, description)
    VALUES (nextval('journal_entry_seq'), p_transaction_id, p_to_account, 'CREDIT', p_amount, p_currency, p_description);
    
    -- Validate
    IF NOT validate_double_entry(p_transaction_id) THEN
        RAISE EXCEPTION 'Double-entry validation failed for transaction %', p_transaction_id;
    END IF;
    
    -- Update account balances
    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_from_account;
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_to_account;
END;
$$ LANGUAGE plpgsql;

-- Daily balance check: Sum of all journal entries = account balance
CREATE OR REPLACE FUNCTION reconcile_account(p_account_id BIGINT, p_date DATE)
RETURNS TABLE(expected_balance DECIMAL, actual_balance DECIMAL, is_balanced BOOLEAN) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        (SELECT COALESCE(balance, 0) FROM accounts WHERE account_id = p_account_id) AS actual_balance,
        (
            SELECT COALESCE(SUM(CASE WHEN entry_type = 'CREDIT' THEN amount ELSE -amount END), 0)
            FROM journal_entries
            WHERE account_id = p_account_id AND posted_at::date <= p_date
        ) AS expected_balance,
        TRUE AS is_balanced;  -- Simplified
END;
$$ LANGUAGE plpgsql;
```

## Idempotent Transaction Processing

```sql
-- Prevent duplicate transaction processing
-- Use idempotency keys to ensure safe retries

CREATE TABLE idempotency_keys (
    key_hash VARCHAR(64) PRIMARY KEY,  -- SHA-256 hash of key
    transaction_id VARCHAR(36),
    status VARCHAR(20) DEFAULT 'PENDING',  -- PENDING, COMPLETED, FAILED
    response_data JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '24 hours'
);

-- Process payment with idempotency
CREATE OR REPLACE FUNCTION process_payment(
    p_idempotency_key TEXT,
    p_account_id BIGINT,
    p_amount DECIMAL,
    p_merchant_id VARCHAR
) RETURNS TABLE(status TEXT, transaction_id VARCHAR) AS $$
DECLARE
    v_key_hash VARCHAR;
    v_existing RECORD;
    v_txn_id VARCHAR;
BEGIN
    v_key_hash := encode(sha256(p_idempotency_key::bytea), 'hex');
    
    -- Check if already processed
    SELECT * INTO v_existing
    FROM idempotency_keys
    WHERE key_hash = v_key_hash
      AND expires_at > NOW();
    
    IF FOUND THEN
        IF v_existing.status = 'COMPLETED' THEN
            -- Return previous result
            RETURN QUERY SELECT 'DUPLICATE', v_existing.transaction_id;
            RETURN;
        ELSIF v_existing.status = 'PENDING' THEN
            -- Still processing
            RETURN QUERY SELECT 'IN_PROGRESS', v_existing.transaction_id;
            RETURN;
        END IF;
    END IF;
    
    -- Generate unique transaction ID
    v_txn_id := 'txn_' || gen_random_uuid();
    
    -- Mark as processing
    INSERT INTO idempotency_keys (key_hash, transaction_id, status)
    VALUES (v_key_hash, v_txn_id, 'PENDING');
    
    -- Process the payment
    -- (actual payment processing logic)
    INSERT INTO transactions (transaction_id, account_id, amount, merchant_id, status)
    VALUES (v_txn_id, p_account_id, p_amount, p_merchant_id, 'COMPLETED');
    
    -- Mark as completed
    UPDATE idempotency_keys 
    SET status = 'COMPLETED', response_data = '{"result": "success"}'
    WHERE key_hash = v_key_hash;
    
    RETURN QUERY SELECT 'COMPLETED', v_txn_id;
END;
$$ LANGUAGE plpgsql;
```

## Audit Trail Pattern

```sql
-- Every data change creates an audit record
-- Trigger-based audit trail

CREATE TABLE audit_trail (
    audit_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    record_id BIGINT NOT NULL,
    action VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMPTZ DEFAULT NOW(),
    client_ip INET,
    session_id VARCHAR(100)
);

-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_fn() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_trail (table_name, record_id, action, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_trail (table_name, record_id, action, old_data, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_trail (table_name, record_id, action, old_data, changed_by)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD), current_user);
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Attach audit trigger to tables
CREATE TRIGGER audit_accounts
    AFTER INSERT OR UPDATE OR DELETE ON accounts
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_fn();

CREATE TRIGGER audit_customers
    AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_fn();

-- Query audit trail
SELECT 
    table_name,
    action,
    changed_by,
    changed_at,
    old_data->>'balance' AS old_balance,
    new_data->>'balance' AS new_balance
FROM audit_trail
WHERE table_name = 'accounts'
  AND record_id = 12345
ORDER BY changed_at DESC;
```

## Regulatory Hold Pattern

```sql
-- Prevent deletion of data under legal/regulatory hold

CREATE TABLE regulatory_holds (
    hold_id BIGINT PRIMARY KEY,
    hold_type VARCHAR(20) CHECK (hold_type IN ('LITIGATION', 'AUDIT', 'INVESTIGATION', 'COMPLIANCE')),
    case_number VARCHAR(50),
    description TEXT,
    data_scope JSONB,  -- Which tables/records are held
    start_date DATE NOT NULL,
    end_date DATE,  -- NULL = indefinite
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_by VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Check holds before deletion
CREATE OR REPLACE FUNCTION check_regulatory_hold() RETURNS TRIGGER AS $$
DECLARE
    v_hold_count INT;
BEGIN
    IF TG_OP = 'DELETE' THEN
        SELECT COUNT(*) INTO v_hold_count
        FROM regulatory_holds
        WHERE status = 'ACTIVE'
          AND data_scope->>'table_name' = TG_TABLE_NAME
          AND (
              (data_scope->>'record_id')::BIGINT = OLD.id
              OR data_scope->>'record_id' IS NULL  -- Hold on entire table
          );
        
        IF v_hold_count > 0 THEN
            RAISE EXCEPTION 'Cannot delete: record is under regulatory hold';
        END IF;
    END IF;
    
    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach to all critical tables
CREATE TRIGGER regulatory_hold_check
    BEFORE DELETE ON transactions
    FOR EACH ROW EXECUTE FUNCTION check_regulatory_hold();
```

## Cross-References

- **Data Governance**: See [data-governance.md](../data-engineering/data-governance.md) for compliance
- **Audit Logging**: See [audit-logging.md](../data-engineering/audit-logging.md) for logging
- **Transactions**: See [transactions.md](transactions.md) for ACID properties

## Interview Questions

1. **How does double-entry bookkeeping work in a database? Why is it important?**
2. **How do you implement idempotent payment processing?**
3. **What is the best way to implement an audit trail in PostgreSQL?**
4. **How do you prevent deletion of data under regulatory hold?**
5. **How do you reconcile account balances with journal entries?**
6. **What database patterns are unique to banking that you don't see in other industries?**

## Checklist: Banking Database Patterns

- [ ] Double-entry bookkeeping enforced for all financial transactions
- [ ] Daily reconciliation procedure automated
- [ ] Idempotency keys used for all external API calls
- [ ] Audit trail on all financial tables
- [ ] Regulatory hold system implemented
- [ ] Data retention policies enforced
- [ ] PII masking in non-production environments
- [ ] End-of-day batch procedures automated
- [ ] Account balance history tracked (SCD Type 2)
- [ ] Transaction sequence numbers for ordering
