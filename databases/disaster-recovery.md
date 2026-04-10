# Disaster Recovery for Banking Databases

## Overview

Disaster recovery (DR) ensures that banking systems can recover from catastrophic failures (datacenter outage, hardware failure, ransomware) within defined RTO (Recovery Time Objective) and RPO (Recovery Point Objective) targets. This guide covers DR strategies, architectures, and testing procedures for production banking databases.

## RTO and RPO

```
RPO (Recovery Point Objective): Maximum acceptable data loss
  - How far back can we recover?
  - Determined by backup/replication frequency
  - Banking: Typically 0-15 minutes for core systems

RTO (Recovery Time Objective): Maximum acceptable downtime
  - How fast can we be operational again?
  - Determined by failover automation
  - Banking: Typically 5-30 minutes for core systems

Recovery Tier:
  Tier 1 (Critical): RPO < 1 min, RTO < 5 min   (synchronous replication, auto-failover)
  Tier 2 (Important): RPO < 15 min, RTO < 30 min (async replication, manual failover)
  Tier 3 (Standard): RPO < 1 hour, RTO < 4 hours (daily backups, manual restore)
  Tier 4 (Archive): RPO < 24 hours, RTO < 24 hours (cold backups)
```

## DR Architecture

```mermaid
graph TB
    subgraph Primary Region (us-east-1)
    P[(Primary DB)]
    S1[Sync Standby - Same AZ]
    S2[Async Standby - Different AZ]
    end
    
    subgraph DR Region (us-west-2)
    DR[(Async Standby - DR)]
    WAL[WAL Archive - S3]
    end
    
    P -->|Sync| S1
    P -->|Async| S2
    P -->|Async| DR
    P -->|Archive| WAL
    S2 -->|Async| DR
```

## Failover Procedures

```bash
# Manual failover: Promote standby to primary

# Check current replication status
psql -h primary -c "SELECT * FROM pg_stat_replication;"

# On standby server:
# Method 1: Using pg_ctl (PostgreSQL < 12)
pg_ctl promote -D /var/lib/postgresql/15/data

# Method 2: Using pg_promote() function (PostgreSQL 12+)
psql -h standby -c "SELECT pg_promote();"

# Method 3: Using Patroni (recommended)
patronictl -c /etc/patroni/patroni.yml switchover --master primary --candidate standby

# After promotion, update application connection strings
# Or use DNS/VIP to point to new primary

# Verify new primary
psql -h new-primary -c "SELECT pg_is_in_recovery();"  -- Should return false
psql -h new-primary -c "SELECT pg_last_xact_replay_timestamp();"  -- Should be NULL
```

## Cross-Region Replication for DR

```sql
-- DR standby in different region (async replication)
-- postgresql.conf on DR standby:
-- primary_conninfo = 'host=primary-region-db port=5432 user=replicator password=xxx application_name=dr_standby'
-- primary_slot_name = 'dr_standby_slot'
-- restore_command = 'aws s3 cp s3://banking-dr-backups/wal/%f %p'
-- recovery_target_timeline = 'latest'

-- Monitor cross-region replication lag
SELECT 
    slot_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes,
    active
FROM pg_replication_slots
WHERE slot_name = 'dr_standby_slot';

-- If lag is unacceptable, consider:
-- 1. Increasing wal_keep_size
-- 2. Improving network bandwidth
-- 3. Using compression (archive_command with gzip)
```

## DR Testing

```sql
-- Regular DR testing procedure (quarterly minimum)

-- 1. Verify standby is current
SELECT 
    pg_last_wal_receive_lsn() AS receive_lsn,
    pg_last_wal_replay_lsn() AS replay_lsn,
    pg_last_xact_replay_timestamp() AS last_replay,
    EXTRACT(EPOCH FROM NOW() - pg_last_xact_replay_timestamp()) AS lag_seconds;

-- 2. Run data consistency checks
-- Compare row counts between primary and standby
-- Primary:
SELECT table_name, 
       (xpath('/row/cnt/text()', xml_count))[1]::text::int AS row_count
FROM (
    SELECT table_name, 
           query_to_xml(format('SELECT count(*) as cnt FROM %I', table_name), 
                        false, true, '') AS xml_count
    FROM information_schema.tables 
    WHERE table_schema = 'public'
) t;

-- 3. Test failover in staging environment
-- Clone standby to staging, promote, run application tests

-- 4. Document results and update DR runbook
```

## Ransomware Recovery

```bash
# In case of ransomware or data corruption:

# 1. Isolate the affected systems
#    - Disconnect from network
#    - Do NOT shut down (may need memory forensics)

# 2. Assess the damage
#    - Determine when the attack started
#    - Identify affected databases and tables

# 3. Restore from clean backup
#    - Use backup from BEFORE the attack
#    - Restore to new, clean infrastructure

# PITR to point before attack
restore_command = 'aws s3 cp s3://banking-backups/wal/%f %p'
recovery_target_time = '2025-01-10 03:00:00 UTC'  -- Before attack started
recovery_target_action = 'promote'

# 4. Verify data integrity
#    - Compare with known-good data
#    - Run data quality checks

# 5. Rotate ALL credentials
#    - Database passwords
#    - API keys
#    - SSL certificates
```

## Cross-References

- **Backups**: See [backups.md](backups.md) for backup strategies
- **High Availability**: See [high-availability.md](high-availability.md) for HA setups
- **Replication**: See [replication.md](replication.md) for replication

## Interview Questions

1. **What is the difference between RTO and RPO? How do you achieve each?**
2. **Design a DR strategy for a banking database with RPO < 5 minutes and RTO < 15 minutes.**
3. **How often should you test DR procedures? What does a DR test involve?**
4. **Your DR standby is 30 minutes behind primary. Is this acceptable?**
5. **How do you recover from ransomware that encrypted your database?**
6. **What is the difference between failover and switchover?**

## Checklist: Disaster Recovery

- [ ] RTO and RPO defined for each database tier
- [ ] DR standby in separate region configured and monitored
- [ ] Failover procedure documented and tested quarterly
- [ ] WAL archiving to off-site storage enabled
- [ ] Cross-region replication lag monitored and alerted
- [ ] DNS/VIP failover mechanism in place
- [ ] Application connection failover tested
- [ ] DR runbook updated and accessible
- [ ] Recovery credentials stored securely (not on database servers)
- [ ] Regular DR test results documented
- [ ] Post-DR testing procedure defined
