# Data Retention, Archival, and Deletion

## Overview

Data retention policies ensure compliance with regulatory requirements while managing storage costs. In banking, some data must be kept for 7-10 years, while other data can be deleted sooner. This guide covers retention strategies, archival patterns, and safe deletion procedures for production banking systems.

## Retention Policy Framework

```yaml
# Data retention policies by data type
retention_policies:
  transactions:
    hot_storage: 90 days       # Primary database (fast queries)
    warm_storage: 2 years       # Archive database / lakehouse
    cold_storage: 7 years       # Object storage (S3 Glacier)
    deletion: after 7 years     # Permanent deletion with approval
  
  customer_records:
    hot_storage: 1 year
    warm_storage: 3 years
    cold_storage: 7 years after relationship termination
    deletion: 7 years after account closure + legal hold check
  
  audit_logs:
    hot_storage: 30 days
    warm_storage: 1 year
    cold_storage: 10 years
    deletion: after 10 years
  
  system_logs:
    hot_storage: 14 days
    warm_storage: 90 days
    cold_storage: 1 year
    deletion: after 1 year
  
  embeddings_and_chunks:
    hot_storage: 90 days
    warm_storage: 2 years
    deletion: when source document is deleted or expired
  
  temporary_staging_data:
    retention: 30 days
    deletion: automatic
```

## Archival Pipeline

```python
"""
Data archival pipeline: Move old data from hot storage to warm/cold storage.
Ensures data remains accessible for compliance while reducing primary DB size.
"""
import psycopg2
import boto3
import json
import logging
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)

class DataArchiver:
    """Archive old data from primary database to S3."""
    
    def __init__(self, source_db: dict, s3_bucket: str, batch_size: int = 10000):
        self.source_db = psycopg2.connect(**source_db)
        self.s3_client = boto3.client('s3')
        self.s3_bucket = s3_bucket
        self.batch_size = batch_size
    
    def archive_transactions(self, cutoff_date: str) -> dict:
        """Archive transactions older than cutoff_date to S3."""
        total_archived = 0
        batch_num = 0
        
        while True:
            # Fetch batch of old transactions
            with self.source_db.cursor() as cur:
                cur.execute("""
                    SELECT 
                        transaction_id, account_id, transaction_type,
                        amount, currency, transaction_time, balance_after,
                        reference, channel, status
                    FROM transactions
                    WHERE transaction_time < %s
                    ORDER BY transaction_time
                    LIMIT %s
                """, (cutoff_date, self.batch_size))
                
                rows = cur.fetchall()
                
                if not rows:
                    break
                
                # Convert to JSON lines
                columns = [desc[0] for desc in cur.description]
                records = [
                    dict(zip(columns, row)) for row in rows
                ]
                jsonl_data = '\n'.join(json.dumps(r, default=str) for r in records)
                
                # Upload to S3
                s3_key = (
                    f"archive/transactions/"
                    f"year={cutoff_date[:4]}/"
                    f"batch_{batch_num:05d}.jsonl"
                )
                
                self.s3_client.put_object(
                    Bucket=self.s3_bucket,
                    Key=s3_key,
                    Body=jsonl_data.encode(),
                    StorageClass='STANDARD_IA',  # Infrequent access
                    Metadata={
                        'archived_at': datetime.utcnow().isoformat(),
                        'record_count': str(len(records)),
                        'cutoff_date': cutoff_date,
                    }
                )
                
                # Mark as archived in source DB
                txn_ids = [r[0] for r in rows]
                cur.executemany("""
                    UPDATE transactions 
                    SET archived = true, archived_at = %s
                    WHERE transaction_id = %s
                """, [(datetime.utcnow(), txn_id) for txn_id in txn_ids])
                
                self.source_db.commit()
                
                total_archived += len(rows)
                batch_num += 1
                
                logger.info(
                    f"Archived batch {batch_num}: {len(rows)} records, "
                    f"total: {total_archived}"
                )
        
        return {
            'total_archived': total_archived,
            'batches': batch_num,
            'cutoff_date': cutoff_date,
            's3_location': f"s3://{self.s3_bucket}/archive/transactions/",
        }
    
    def delete_archived_data(self, cutoff_date: str) -> int:
        """Delete data that has been successfully archived."""
        with self.source_db.cursor() as cur:
            # Delete in batches to avoid long-running transactions
            total_deleted = 0
            
            while True:
                cur.execute("""
                    DELETE FROM transactions
                    WHERE transaction_id IN (
                        SELECT transaction_id
                        FROM transactions
                        WHERE archived = true
                          AND transaction_time < %s
                        LIMIT 10000
                    )
                """, (cutoff_date,))
                
                deleted = cur.rowcount
                self.source_db.commit()
                total_deleted += deleted
                
                if deleted == 0:
                    break
                
                logger.info(f"Deleted batch: {deleted} records")
        
        return total_deleted
```

## Safe Deletion Procedures

```sql
-- NEVER delete banking data without:
-- 1. Confirming retention period has expired
-- 2. Checking for legal holds
-- 3. Verifying data has been archived
-- 4. Getting appropriate approvals

-- Check for legal holds before deletion
CREATE TABLE legal_holds (
    hold_id BIGINT PRIMARY KEY,
    case_number VARCHAR(50),
    hold_type VARCHAR(20),  -- LITIGATION, REGULATORY, AUDIT
    data_scope JSONB,  -- Which data is held
    start_date DATE,
    end_date DATE,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_by VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Deletion function with checks
CREATE OR REPLACE FUNCTION safe_delete_transactions(
    p_cutoff_date DATE,
    p_user_id VARCHAR(50)
) RETURNS TABLE(status TEXT, count BIGINT) AS $$
DECLARE
    v_hold_count INT;
    v_delete_count BIGINT;
BEGIN
    -- Check for active legal holds affecting this data
    SELECT COUNT(*) INTO v_hold_count
    FROM legal_holds
    WHERE status = 'ACTIVE'
      AND (data_scope->>'data_type') = 'transactions'
      AND (data_scope->>'end_date')::DATE >= p_cutoff_date;
    
    IF v_hold_count > 0 THEN
        RETURN QUERY 
        SELECT 'BLOCKED_BY_LEGAL_HOLD'::TEXT, v_hold_count::BIGINT;
        RETURN;
    END IF;
    
    -- Check data has been archived
    SELECT COUNT(*) INTO v_delete_count
    FROM transactions
    WHERE transaction_time < p_cutoff_date
      AND archived = false;
    
    IF v_delete_count > 0 THEN
        RETURN QUERY 
        SELECT 'NOT_ARCHIVED'::TEXT, v_delete_count;
        RETURN;
    END IF;
    
    -- Safe to delete
    DELETE FROM transactions
    WHERE transaction_time < p_cutoff_date
      AND archived = true;
    
    GET DIAGNOSTICS v_delete_count = ROW_COUNT;
    
    -- Log the deletion
    INSERT INTO data_deletion_log (
        table_name, cutoff_date, deleted_count, deleted_by, deleted_at
    ) VALUES (
        'transactions', p_cutoff_date, v_delete_count, p_user_id, NOW()
    );
    
    RETURN QUERY SELECT 'DELETED'::TEXT, v_delete_count;
END;
$$ LANGUAGE plpgsql;
```

## GDPR Data Deletion

```sql
-- GDPR "Right to be Forgotten": Anonymize rather than delete
-- Banking records must be retained for regulatory compliance,
-- but personal identifiers must be removed upon request

CREATE OR REPLACE FUNCTION anonymize_customer_data(
    p_customer_id BIGINT,
    p_request_id VARCHAR(50)
) RETURNS void AS $$
BEGIN
    -- Anonymize customer PII
    UPDATE customers SET
        first_name = '[REDACTED-GDPR-' || p_request_id || ']',
        last_name = '[REDACTED-GDPR-' || p_request_id || ']',
        date_of_birth = NULL,
        national_id_hash = NULL,
        email = NULL,
        phone = NULL
    WHERE customer_id = p_customer_id;
    
    -- Anonymize addresses
    UPDATE customer_addresses SET
        street_address = '[REDACTED]',
        city = '[REDACTED]',
        postal_code = '[REDACTED]'
    WHERE customer_id = p_customer_id;
    
    -- Keep transaction records (regulatory requirement)
    -- but remove customer linkage
    -- (transactions still needed for audit/AML)
    
    -- Log the anonymization
    INSERT INTO gdpr_deletion_log (
        customer_id, request_id, anonymized_at, anonymized_by
    ) VALUES (
        p_customer_id, p_request_id, NOW(), current_user
    );
END;
$$ LANGUAGE plpgsql;
```

## Cross-References

- **Data Governance**: See [data-governance.md](data-governance.md) for compliance framework
- **Data Partitioning**: See [data-partitioning.md](data-partitioning.md) for partition management
- **Audit Logging**: See [audit-logging.md](audit-logging.md) for deletion audit trails

## Interview Questions

1. **How do you balance regulatory retention requirements with storage cost management?**
2. **Design an archival pipeline for 5 years of banking transactions.**
3. **What checks must you perform before deleting banking data?**
4. **How does GDPR "right to be forgotten" conflict with banking retention requirements?**
5. **Your primary database is running out of disk space due to old data. What do you do?**
6. **How do you verify that archived data can be restored when needed?**

## Checklist: Retention and Deletion

- [ ] Retention policies documented and approved by compliance
- [ ] Legal hold system in place to block deletion of held data
- [ ] Archival pipeline tested and running on schedule
- [ ] Archived data verified for completeness and accessibility
- [ ] Deletion procedures include approval workflow
- [ ] All deletions logged in audit trail
- [ ] GDPR anonymization procedure defined
- [ ] Regular tests of archive restoration
- [ ] Storage costs monitored and projected
- [ ] Automated enforcement of retention policies
