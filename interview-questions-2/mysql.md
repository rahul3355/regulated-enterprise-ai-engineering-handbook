# MySQL Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | MySQL (Fundamentals, InnoDB, Indexing, Replication, Query Optimization, vs PostgreSQL) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | databases/ (Postgres files for comparison context) + general MySQL knowledge |
| Citi Relevance | MySQL is listed alongside Postgres in Citi's job requirements. Many legacy banking systems run on MySQL. Understanding the trade-offs between MySQL and PostgreSQL is essential for platform engineering decisions. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

### Q1: 🔵 Explain the MySQL client-server architecture. How does a connection flow from client to query execution?

**Strong Answer:**

MySQL follows a client-server architecture with a clear separation between the connection layer and the storage engine layer. This two-layer design is one of MySQL's most distinctive architectural decisions and enables its pluggable storage engine capability.

When a client connects to MySQL, it first hits the **Connection Manager** (layer 1). The server authenticates the client using credentials stored in the `mysql.user` system table. MySQL can authenticate via native password, SHA-256, caching_sha2_password (the default since 8.0), or external plugins like PAM/LDAP — critical for enterprise environments like Citi that integrate with Active Directory.

After authentication, the connection enters the **Thread Pool** (or one-thread-per-connection model depending on configuration). Each connection gets its own thread in the default model. For high-concurrency banking applications with thousands of connections, the Enterprise Edition thread pool or a connection pooler like ProxySQL is essential to avoid thread-creation overhead.

The query then passes through the **Parser**, which generates a parse tree, then the **Preprocessor** checks table/column existence and permissions, followed by the **Optimizer** which evaluates execution plans using cost-based optimization (CBO). The optimizer uses index statistics (from `ANALYZE TABLE`) to estimate costs and pick the cheapest plan.

Finally, the optimized query is passed to the **Storage Engine API** (layer 2). This is where MySQL's pluggable architecture shines — the server layer doesn't care whether InnoDB, MyISAM, or another engine handles the data. InnoDB is the default and the only engine that supports transactions, row-level locking, and foreign keys, making it the only viable choice for banking workloads.

```sql
-- Connection flow visualization:
Client → MySQL Port (3306) → Authentication → Thread Assignment
       → Parser → Preprocessor → Optimizer → Storage Engine API → InnoDB
       → Result Cache (if enabled) → Client

-- View active connections
SHOW PROCESSLIST;
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep';

-- Check thread pool status (Enterprise Edition)
SHOW STATUS LIKE 'threadpool%';

-- Max connections configuration
SHOW VARIABLES LIKE 'max_connections';
-- Default: 151. For banking, typically set to 500-2000 with ProxySQL.
```

The key interview insight: MySQL's server layer handles connections, parsing, optimization, and caching, while storage engines handle data storage, indexing, transactions, and recovery. This separation allows mixing engines per table (e.g., InnoDB for transactional tables, MyISAM for read-only reference data), though in practice, modern banking systems use InnoDB exclusively.

**Key Points to Hit:**
- [ ] Two-layer architecture: server layer (connections, SQL processing) + storage engine layer (data, transactions)
- [ ] Pluggable storage engine architecture — InnoDB is default since 5.5
- [ ] Authentication mechanisms (caching_sha2_password default in 8.0)
- [ ] Thread-per-connection vs thread pool models
- [ ] Query processing pipeline: Parser → Preprocessor → Optimizer → Execution
- [ ] Cost-based optimizer uses index statistics for plan selection

**Follow-Up Questions:**
1. What happens when you exceed `max_connections`? How do you handle this in production?
2. How does MySQL 8.0's document store differ from the traditional relational model?
3. Why can't you use connection pooling libraries the same way in MySQL as in PostgreSQL?

**Source:** banking-genai-engineering-academy/databases/postgres-fundamentals.md (for architecture comparison)

---

### Q2: 🔵 Compare InnoDB and MyISAM. Why is InnoDB the only acceptable choice for banking systems?

**Strong Answer:**

This is a foundational question that tests whether a candidate understands the practical implications of storage engine differences. While MyISAM was MySQL's default engine before version 5.5, it is fundamentally unsuitable for any system requiring data integrity — which describes every banking application.

**InnoDB** is a transactional, ACID-compliant storage engine with row-level locking, foreign key support, crash recovery, and MVCC (Multi-Version Concurrency Control). It uses a clustered index structure where the primary key determines physical data layout, and it maintains a buffer pool for caching both data pages and index pages.

**MyISAM** uses table-level locking (a single write locks the entire table), has no transaction support, no foreign keys, no crash recovery, and uses separate files for data (.MYD), indexes (.MYI), and table definitions (.frm). Its only advantages over InnoDB are slightly faster COUNT(*) operations (it stores row counts) and full-text search (though InnoDB added this in 5.6).

```sql
-- InnoDB features that matter for banking:

-- 1. Transactions with ACID guarantees
START TRANSACTION;
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 2;
COMMIT;  -- Atomic: either both succeed or both fail

-- 2. Foreign key constraints (referential integrity)
ALTER TABLE transactions
ADD CONSTRAINT fk_account
FOREIGN KEY (account_id) REFERENCES accounts(account_id)
ON DELETE RESTRICT ON UPDATE CASCADE;

-- 3. Row-level locking (high concurrency)
-- Multiple sessions can update different rows simultaneously
-- Session A: UPDATE accounts SET balance = 500 WHERE account_id = 1;  -- locks row 1
-- Session B: UPDATE accounts SET balance = 600 WHERE account_id = 2;  -- NOT blocked

-- 4. Crash recovery via redo/undo logs
-- After crash, InnoDB replays redo log and rolls back uncommitted transactions

-- MyISAM: table-level locking demonstration
-- Session A: UPDATE accounts SET balance = 500 WHERE account_id = 1;  -- locks ENTIRE table
-- Session B: UPDATE accounts SET balance = 600 WHERE account_id = 2;  -- BLOCKED (table locked)
-- Session C: SELECT COUNT(*) FROM accounts;                            -- BLOCKED too

-- Check engine per table
SELECT table_name, engine, table_rows, data_length, index_length
FROM information_schema.tables
WHERE table_schema = 'banking_db'
ORDER BY engine, table_name;

-- Convert MyISAM to InnoDB (if legacy tables exist)
ALTER TABLE legacy_table ENGINE = InnoDB;
```

For a banking system at Citi, the decision is non-negotiable:

1. **Data integrity**: Banking transactions must be atomic. If a debit succeeds but the corresponding credit fails (server crash between statements), MyISAM leaves the database in an inconsistent state. InnoDB's transaction log ensures recovery.

2. **Concurrent access**: MyISAM's table-level locking means a single write blocks all reads and writes. In a high-volume trading system processing thousands of transactions per second, this creates unacceptable latency.

3. **Foreign keys**: Referential integrity prevents orphaned transactions, payments without accounts, or audit records without a source transaction. MyISAM silently ignores FOREIGN KEY syntax.

4. **Crash recovery**: Banking systems must survive power failures without data loss. InnoDB's redo log and doublewrite buffer ensure committed transactions survive crashes.

**Key Points to Hit:**
- [ ] InnoDB supports ACID transactions; MyISAM does not
- [ ] Row-level locking (InnoDB) vs table-level locking (MyISAM)
- [ ] Foreign key enforcement in InnoDB
- [ ] Crash recovery via redo/undo logs in InnoDB
- [ ] MVCC for non-blocking reads in InnoDB
- [ ] MyISAM only advantage: fast COUNT(*), full-text (before 5.6)

**Follow-Up Questions:**
1. If you inherited a legacy MyISAM banking database, how would you migrate it?
2. Are there any valid use cases for MyISAM in a modern banking architecture?
3. How does InnoDB's crash recovery actually work at the storage level?

**Source:** banking-genai-engineering-academy/databases/postgres-fundamentals.md, databases/transactions.md

---

### Q3: 🔵 What is the InnoDB Buffer Pool, and why is it the single most important configuration parameter for MySQL performance?

**Strong Answer:**

The InnoDB Buffer Pool is MySQL's equivalent of PostgreSQL's `shared_buffers` — it's the primary memory area where InnoDB caches data pages and index pages read from disk. Think of it as InnoDB's working memory: every read operation first checks the buffer pool, and only performs a disk I/O if the page isn't cached (a "cold" read).

The buffer pool uses a **midpoint insertion strategy** with an LRU (Least Recently Used) algorithm divided into two sublists: the "new" sublist (pages recently accessed, ~37% of pool) and the "old" sublist (pages that might be evicted, ~63%). When a page is first read into the buffer pool, it's placed at the midpoint of the old list. If it's accessed again while in the old list, it gets promoted to the new list. This design prevents full table scans from polluting the cache with one-time-use pages.

```sql
-- Buffer pool configuration (my.cnf or SET GLOBAL)
-- For a dedicated 64GB MySQL server:
SET GLOBAL innodb_buffer_pool_size = 48 * 1024 * 1024 * 1024;  -- 48GB (75% of RAM)

-- General rule: allocate 50-80% of available RAM to buffer pool
-- Leave room for OS cache, connection buffers, and other processes

-- Buffer pool instances (reduces mutex contention on multi-core systems)
SET GLOBAL innodb_buffer_pool_instances = 8;
-- Default: 1 if pool < 1GB. For 48GB pool, use 8-16 instances.
-- Each instance has its own LRU list and free list.

-- Monitor buffer pool efficiency
SHOW ENGINE INNODB STATUS\G
-- Look for "BUFFER POOL AND MEMORY" section

-- Detailed buffer pool metrics
SELECT
    (SELECT variable_value FROM performance_schema.global_status
     WHERE variable_name = 'Innodb_buffer_pool_read_requests') AS logical_reads,
    (SELECT variable_value FROM performance_schema.global_status
     WHERE variable_name = 'Innodb_buffer_pool_reads') AS physical_reads,
    ROUND(
        (1 - (SELECT variable_value FROM performance_schema.global_status
              WHERE variable_name = 'Innodb_buffer_pool_reads') /
         NULLIF((SELECT variable_value FROM performance_schema.global_status
                 WHERE variable_name = 'Innodb_buffer_pool_read_requests'), 0))
        * 100, 2
    ) AS buffer_pool_hit_ratio_pct;

-- Target: >99% hit ratio for OLTP workloads
-- Below 95% indicates undersized buffer pool or inefficient queries

-- Buffer pool page states
SELECT
    pool_id,
    pool_size,
    free_buffers,
    database_pages,
    old_database_pages,
    modified_database_pages AS dirty_pages,
    hit_rate
FROM information_schema.innodb_buffer_pool_stats;
```

Why this is the most important parameter:

1. **I/O reduction**: A 99% hit ratio means 99 out of 100 page reads are served from memory. Disk I/O is ~100,000x slower than memory access. An undersized buffer pool is the #1 cause of MySQL performance degradation.

2. **Dynamic resizing**: Since MySQL 5.7, you can resize the buffer pool online without restart (`SET GLOBAL innodb_buffer_pool_size = ...`). MySQL 8.0 added `innodb_buffer_pool_chunk_size` for smoother resizing.

3. **Dirty page management**: Modified pages in the buffer pool ("dirty pages") are flushed to disk by background threads. The `innodb_max_dirty_pages_pct` variable controls when flushing kicks in. In banking systems with heavy write loads, tuning this prevents I/O spikes.

4. **Preloading**: You can warm the buffer pool at startup by saving/restoring the buffer pool state (`innodb_buffer_pool_dump_at_shutdown` and `innodb_buffer_pool_load_at_startup`), critical for production systems that need to reach full performance immediately after restart.

**Key Points to Hit:**
- [ ] Buffer pool caches data and index pages — primary performance determinant
- [ ] Midpoint insertion LRU algorithm prevents full-scan cache pollution
- [ ] Size recommendation: 50-80% of RAM on dedicated MySQL server
- [ ] Buffer pool instances reduce mutex contention on multi-core systems
- [ ] Target >99% hit ratio for OLTP; below 95% requires investigation
- [ ] Dynamic resizing available since MySQL 5.7
- [ ] Dirty page flushing controlled by background threads

**Follow-Up Questions:**
1. Your buffer pool hit ratio dropped from 99.5% to 92%. How do you diagnose?
2. What is the difference between `Innodb_buffer_pool_reads` and `Innodb_buffer_pool_read_requests`?
3. How does buffer pool preloading help after a server restart?

**Source:** banking-genai-engineering-academy/databases/postgres-performance.md (shared_buffers comparison)

---

### Q4: 🔵 Explain MySQL transactions. How do COMMIT and ROLLBACK work at the storage engine level?

**Strong Answer:**

In MySQL, transaction support is entirely an InnoDB feature — the server layer is transaction-unaware. When you issue `START TRANSACTION`, the server layer simply passes this command through to InnoDB, which manages everything: the transaction ID assignment, isolation level enforcement, lock management, undo log creation, and the eventual commit or rollback.

**Starting a transaction:** InnoDB assigns a unique **transaction ID** (a monotonically increasing 64-bit integer). It also takes a **consistent read snapshot** based on the isolation level. In Read Committed, a new snapshot is taken per statement. In Repeatable Read (InnoDB's default), the snapshot is taken once at transaction start. This snapshot determines which row versions are visible — the core of InnoDB's MVCC implementation.

**During a transaction:** Every `UPDATE` or `DELETE` operation writes the **old version** of the row to the **undo log** before modifying the page in the buffer pool. The new version in the data page records the current transaction ID. This dual recording is what enables both rollback (undo the changes using the undo log) and consistent reads (other transactions see the old version from their snapshot).

**COMMIT:** This is a two-phase process internally:
1. InnoDB writes a **commit record** to the **redo log** (a sequential write to the transaction log file, `ib_logfile0`/`ib_logfile1`). This is the durability guarantee — the redo log is flushed to disk before COMMIT returns.
2. InnoDB marks the transaction as committed in the **transaction system table**. The modified pages in the buffer pool remain dirty and are flushed to disk asynchronously by background threads.

```sql
-- Transaction lifecycle in InnoDB:

START TRANSACTION;  -- InnoDB assigns tx_id, takes snapshot

-- Each write creates undo log entries
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 1;
-- Undo log stores: account_id=1, balance=OLD_VALUE, tx_id=current

UPDATE accounts SET balance = balance + 1000 WHERE account_id = 2;

-- COMMIT: Two-phase internal process
COMMIT;
-- Phase 1: Write commit record to redo log (sequential disk write)
-- Phase 2: Mark transaction committed in transaction system table
-- Modified pages in buffer pool flushed asynchronously later

-- ROLLBACK: Use undo log to reverse changes
START TRANSACTION;
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 1;
ROLLBACK;  -- InnoDB reads undo log and restores original values

-- Savepoints for partial rollback
START TRANSACTION;
UPDATE accounts SET balance = 500 WHERE account_id = 1;
SAVEPOINT after_first_update;
UPDATE accounts SET balance = 600 WHERE account_id = 2;
ROLLBACK TO SAVEPOINT after_first_update;  -- Only second update undone
COMMIT;  -- First update is committed

-- Check active transactions
SELECT
    trx_id,
    trx_state,
    trx_started,
    trx_requested_lock_id,
    trx_wait_started,
    trx_weight,
    trx_mysql_thread_id,
    trx_query
FROM information_schema.innodb_trx;

-- InnoDB log files configuration
SHOW VARIABLES LIKE 'innodb_log_file%';
-- innodb_log_file_size: Size of each redo log file (default 48MB in 8.0)
-- innodb_log_files_in_group: Number of log files (default 2)
-- innodb_log_buffer_size: In-memory log buffer (default 16MB)
```

**ROLLBACK:** InnoDB reads the undo log entries created during the transaction and applies them in reverse order. The undo logs themselves are stored in the **undo tablespace** (separate from the data tablespace in modern InnoDB), and they are also protected by the redo log. This means even a ROLLBACK is crash-safe — if the server crashes mid-rollback, recovery will complete the rollback on restart.

For banking, the critical insight is that InnoDB's durability guarantee depends on the redo log being flushed to disk. The `innodb_flush_log_at_trx_commit` variable controls this:
- `1` (default): Flush to disk every commit — full ACID, ~2x write overhead
- `0`: Flush once per second — risk of losing 1 second of transactions
- `2`: Write to OS cache every commit, flush to disk once per second — risk on OS crash only

In a regulated banking environment, this must always be `1`. The only exception is during bulk data loads where temporary relaxation is acceptable.

**Key Points to Hit:**
- [ ] Transaction support is InnoDB-specific, not a server-layer feature
- [ ] Undo log stores old row versions for rollback and MVCC
- [ ] Redo log ensures durability — commit record flushed to disk before COMMIT returns
- [ ] Modified pages flushed to disk asynchronously after commit
- [ ] `innodb_flush_log_at_trx_commit = 1` is required for full ACID compliance
- [ ] Savepoints enable partial rollback within a transaction

**Follow-Up Questions:**
1. What is the performance impact of `innodb_flush_log_at_trx_commit = 1` vs `2`?
2. How does InnoDB handle a crash during a long-running ROLLBACK?
3. Why doesn't MyISAM support transactions?

**Source:** banking-genai-engineering-academy/databases/transactions.md, databases/postgres-fundamentals.md

---

### Q5: 🔵 Describe the B-tree index structure in MySQL/InnoDB. How does InnoDB use indexes differently from PostgreSQL?

**Strong Answer:**

MySQL/InnoDB uses B+tree indexes (not plain B-trees), which are the backbone of query performance. A B+tree is a self-balancing tree where all data resides in the leaf nodes, and internal nodes serve as a routing structure. The leaf nodes are linked together via doubly-linked lists, enabling efficient range scans.

In InnoDB, there are two types of B+tree indexes with fundamentally different structures:

**Clustered Index (Primary Key Index):** The table data IS the clustered index. InnoDB stores actual row data in the leaf nodes of the primary key B+tree. This means the primary key determines the physical storage order of rows. Every InnoDB table has exactly one clustered index — if you don't define a primary key, InnoDB creates a hidden 6-byte row ID.

**Secondary Indexes:** These are separate B+trees where leaf nodes contain the indexed column values plus the **primary key value** (not a physical row pointer). To fetch a full row via a secondary index, InnoDB performs a "bookmark lookup" — it finds the primary key in the secondary index, then looks up the row in the clustered index.

```sql
-- Clustered index: data is stored in PK order
CREATE TABLE accounts (
    account_id BIGINT PRIMARY KEY,  -- This IS the clustered index
    customer_id BIGINT,
    balance DECIMAL(15, 2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Data pages are physically ordered by account_id
-- Range scan on PK is extremely fast (sequential page reads)

-- Secondary index: contains (indexed_column + primary_key)
CREATE INDEX idx_accounts_customer ON accounts(customer_id);
-- Leaf nodes store: (customer_id_value, account_id)
-- To fetch full row: look up customer_id → get account_id → look up in clustered index

-- Covering index: avoids bookmark lookup
CREATE INDEX idx_covering ON accounts(customer_id, status, balance);
-- Query can be satisfied entirely from index:
SELECT customer_id, status, balance FROM accounts WHERE customer_id = 12345;
-- No clustered index lookup needed — "index-only scan"

-- Compare with PostgreSQL:
-- PostgreSQL: Heap table (data separate from indexes)
-- - Primary key index points to heap location (CTID)
-- - All indexes are "secondary" — none contain the full row
-- - Index-only scans use visibility map + INCLUDE clause

-- MySQL/InnoDB: Clustered index (data IS the primary key index)
-- - Primary key index contains full row data
-- - Secondary indexes include PK value for bookmark lookup
-- - No separate heap table

-- EXPLAIN shows index usage
EXPLAIN SELECT * FROM accounts WHERE customer_id = 12345;
-- type: ref, key: idx_accounts_customer, rows: estimated rows examined

-- Check index statistics
SHOW INDEX FROM accounts;
-- Cardinality: approximate number of distinct values (from ANALYZE TABLE)
```

The critical difference from PostgreSQL:

| Aspect | MySQL/InnoDB | PostgreSQL |
|--------|-------------|------------|
| Data storage | Clustered (in PK index) | Heap (separate from indexes) |
| Primary key | Is the clustered index | Is a unique index on heap |
| Secondary index leaf | (indexed_cols, PK_value) | (indexed_cols, heap_CTID) |
| Row lookup via secondary | Bookmark lookup (2 index traversals) | Direct heap fetch (1 index + 1 heap) |
| Index-only scan | Covering index (include all needed cols) | INCLUDE clause + visibility map |
| Table organization | Physically ordered by PK | Insertion order (no inherent order) |

This has real implications for schema design. In InnoDB, choosing the right primary key is critical because it determines: (a) how efficiently range queries execute, (b) the size of every secondary index (since PK is duplicated), and (c) write amplification on insert (inserting in random order causes page splits).

**Key Points to Hit:**
- [ ] InnoDB uses B+tree with data in leaf nodes
- [ ] Clustered index = table data stored in PK order (only one per table)
- [ ] Secondary index leaf = (indexed_cols, PK_value) — requires bookmark lookup
- [ ] Covering indexes eliminate bookmark lookups for index-only scans
- [ ] PostgreSQL uses heap storage; InnoDB uses clustered storage
- [ ] Primary key choice affects all secondary index sizes in InnoDB

**Follow-Up Questions:**
1. What happens if you use a UUID as a primary key in InnoDB? How do you mitigate?
2. How does the "bookmark lookup" affect query performance on wide tables?
3. Why does InnoDB create a hidden 6-byte row ID if no primary key is defined?

**Source:** banking-genai-engineering-academy/databases/indexing.md

---

### Q6: 🟡 Explain InnoDB's MVCC implementation. How do undo logs enable consistent reads without blocking writers?

**Strong Answer:**

InnoDB's MVCC (Multi-Version Concurrency Control) is the mechanism that allows readers and writers to operate concurrently without blocking each other. This is fundamental to MySQL's ability to handle high-throughput banking workloads where analytics queries run simultaneously with transaction processing.

The core concept: instead of locking rows for reads, InnoDB maintains **multiple versions** of each row. When a transaction reads data, it sees a **consistent snapshot** — a view of the database as it existed at a specific point in time. Writers create new versions without disturbing existing readers.

**How versions are stored:** Each row in an InnoDB page contains hidden columns that MVCC relies on:
- `DB_TRX_ID` (6 bytes): Transaction ID of the last transaction that inserted or updated this row
- `DB_ROLL_PTR` (7 bytes): Roll pointer pointing to the undo log record containing the previous version
- `DB_ROW_ID` (6 bytes): Hidden row ID (if no explicit primary key)

When a transaction updates a row, InnoDB:
1. Copies the current row version to the **undo log** (in the undo tablespace)
2. Updates the row in the buffer pool with the new values and new `DB_TRX_ID`
3. Sets the `DB_ROLL_PTR` to point to the undo log record

The undo log forms a **version chain** — each undo record points to the previous version, creating a linked list of all historical versions of a row.

```sql
-- MVCC version chain visualization:
--
-- Row in data page:
┌─────────────┬──────────────┬──────────┐
│ balance=500 │ DB_TRX_ID=300│ roll_ptr─┼──→ Undo log record v2
└─────────────┴──────────────┴──────────┘

-- Undo log (undo tablespace):
┌─────────────────────────────────────────────┐
│ v2: balance=500, trx_id=300, roll_ptr ──┐  │
└─────────────────────────────────────────┘  │
                                             ▼
                             ┌─────────────────────────────────────────────┐
                             │ v1: balance=1000, trx_id=200, roll_ptr ─┐  │
                             └─────────────────────────────────────────┘  │
                                                                         ▼
                                                         ┌─────────────────────────────────────────────┐
                                                         │ v0: balance=800, trx_id=100, roll_ptr=NULL  │
                                                         └─────────────────────────────────────────────┘

-- Transaction visibility:
-- Tx100 (snapshot at time 100): sees v0 (balance=800)
-- Tx200 (snapshot at time 200): sees v1 (balance=1000)
-- Tx300 (snapshot at time 300): sees v2 (balance=500)
-- Tx400 (snapshot at time 400): sees v2 (balance=500) — current version

-- Read View: determines which versions are visible
-- Contains: list of active transaction IDs at snapshot time

-- Check undo log status
SHOW ENGINE INNODB STATUS\G
-- Look for "TRANSACTIONS" section showing undo log segments

-- Monitor undo tablespace growth
SELECT
    file_name,
    tablespace_name,
    ROUND(file_size / 1024 / 1024, 2) AS size_mb
FROM information_schema.files
WHERE file_name LIKE '%undo%';

-- Long-running transactions prevent undo log purge
SELECT
    trx_id,
    trx_state,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_seconds
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

**Read Views:** When a transaction starts (in Repeatable Read) or a statement starts (in Read Committed), InnoDB creates a Read View — a snapshot of all currently active transaction IDs. When reading a row version, InnoDB walks the undo log chain and checks: "Was this version created by a transaction that was already committed when my Read View was taken?" If yes, the version is visible. If no, InnoDB follows the roll pointer to the next older version.

**Purge Thread:** Old undo log records that are no longer needed by any active transaction are cleaned up by the **purge thread**. If a transaction runs for hours (e.g., a long analytics query), the purge thread cannot remove versions created after that transaction's snapshot, causing the undo tablespace to grow. This is a common production issue in banking systems where mixed OLTP and analytics workloads share the same server.

The key difference from PostgreSQL: PostgreSQL stores old row versions in the main heap table itself (requiring VACUUM to clean them), while InnoDB stores them in a separate undo tablespace. This means InnoDB doesn't suffer from table bloat due to MVCC, but it does need to manage undo tablespace size.

**Key Points to Hit:**
- [ ] Each row has hidden columns: DB_TRX_ID, DB_ROLL_PTR, DB_ROW_ID
- [ ] Undo log stores old versions as a linked chain
- [ ] Read View determines which versions are visible to a transaction
- [ ] Purge thread cleans up obsolete undo log records
- [ ] Long-running transactions block undo log purge, causing tablespace growth
- [ ] InnoDB stores versions in undo tablespace; PostgreSQL stores in heap table
- [ ] Repeatable Read: snapshot at transaction start; Read Committed: per-statement

**Follow-Up Questions:**
1. A long-running analytics query is causing the undo tablespace to grow to 50GB. How do you fix?
2. How does MVCC relate to InnoDB's isolation levels?
3. What is the performance cost of maintaining undo log chains under heavy write load?

**Source:** banking-genai-engineering-academy/databases/postgres-fundamentals.md, databases/isolation-levels.md

---

### Q7: 🟡 How does composite index column ordering affect query performance? Explain index selectivity and cardinality.

**Strong Answer:**

Composite index column ordering is one of the most impactful yet commonly misunderstood aspects of MySQL performance. The order of columns in a composite B+tree index determines which queries can use the index and how efficiently.

**Leftmost Prefix Rule:** A composite index on `(A, B, C)` can be used for queries filtering on:
- `(A)` — uses first column
- `(A, B)` — uses first two columns
- `(A, B, C)` — uses all three columns
- `(A, C)` — uses A, then filesorts for C (B is skipped)
- `(B)` or `(C)` or `(B, C)` — cannot use the index at all

The rule is strict: the optimizer can only use a contiguous leftmost prefix of the index. If the WHERE clause skips a column in the middle, the index is used only up to that gap.

**Column ordering strategy:** The optimal order follows these rules:
1. **Equality columns first** — columns used with `=` or `IN` predicates
2. **Range columns last** — columns used with `<`, `>`, `BETWEEN`, `LIKE 'prefix%'`
3. **Most selective first** (within equality columns) — higher cardinality columns narrow down the search space faster

```sql
-- Banking scenario: transaction lookup by account, date, and status
CREATE TABLE transactions (
    txn_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_id BIGINT NOT NULL,
    txn_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL,
    amount DECIMAL(15, 2),
    description TEXT,
    INDEX idx_account (account_id),
    INDEX idx_date (txn_date)
);

-- Common query pattern:
SELECT * FROM transactions
WHERE account_id = 12345
  AND txn_date >= '2025-01-01'
  AND status = 'COMPLETED'
ORDER BY txn_date DESC;

-- Option A: INDEX (account_id, txn_date, status)
-- Uses account_id (equality), then txn_date (range) — status cannot use index after range
-- Good for this specific query

-- Option B: INDEX (account_id, status, txn_date)
-- Uses account_id (equality), status (equality), then txn_date (range)
-- Better: more columns used with equality, range column last
-- Also supports: WHERE account_id = ? AND status = ? (without date)

-- Option C: INDEX (status, account_id, txn_date)
-- Worse: status has low selectivity (only a few distinct values)
-- Would scan many rows before narrowing by account_id

-- Best choice: INDEX (account_id, status, txn_date)
-- Rationale:
-- 1. account_id: high cardinality (millions of distinct values), equality match
-- 2. status: equality match (narrows further)
-- 3. txn_date: range match + ORDER BY (must be last due to range rule)

CREATE INDEX idx_txn_optimal ON transactions(account_id, status, txn_date DESC);

-- Verify with EXPLAIN
EXPLAIN SELECT * FROM transactions
WHERE account_id = 12345 AND txn_date >= '2025-01-01' AND status = 'COMPLETED'
ORDER BY txn_date DESC\G
-- Expected: type=ref, key=idx_txn_optimal, Using index condition

-- Index selectivity: ratio of distinct values to total rows
SELECT
    COUNT(DISTINCT account_id) / COUNT(*) AS account_selectivity,
    COUNT(DISTINCT status) / COUNT(*) AS status_selectivity,
    COUNT(DISTINCT txn_date) / COUNT(*) AS date_selectivity,
    COUNT(DISTINCT (account_id, status, txn_date)) / COUNT(*) AS composite_selectivity
FROM transactions;
-- Higher selectivity = better index discrimination

-- Cardinality estimates (from ANALYZE TABLE)
SHOW INDEX FROM transactions;
-- Cardinality column shows estimated distinct values

-- Update statistics after large data changes
ANALYZE TABLE transactions;
```

**Index Selectivity:** Selectivity measures how well an index can narrow down results. A unique index (like a primary key) has selectivity of 1.0 — each value identifies exactly one row. An index on a boolean column has selectivity of 0.5 — worst case for discrimination.

In banking contexts, `account_id` typically has very high cardinality (millions of unique values), making it an excellent first column. `status` has low cardinality (PENDING, COMPLETED, FAILED, REVERSED), making it a poor first column but a good second column when combined with a high-selectivity equality predicate.

**Practical guideline for composite indexes:**
- Put the most discriminating equality column first
- Add remaining equality columns in descending cardinality order
- Put range columns (BETWEEN, <, >, LIKE prefix%) last
- If multiple range columns exist, pick the one used in ORDER BY or the most selective one

**Key Points to Hit:**
- [ ] Leftmost prefix rule — index can only be used from the left contiguously
- [ ] Equality columns before range columns in composite index
- [ ] Higher cardinality/selectivity columns first (within equality columns)
- [ ] Range column breaks index usage for subsequent columns
- [ ] ANALYZE TABLE updates cardinality statistics for optimizer
- [ ] EXPLAIN verifies actual index usage, not just theoretical

**Follow-Up Questions:**
1. A query filters on `account_id` and `txn_date` but the index is `(account_id, status, txn_date)`. Will it use the index?
2. How does the optimizer handle a composite index when only the second and third columns are in the WHERE clause?
3. When would you intentionally put a low-cardinality column first in a composite index?

**Source:** banking-genai-engineering-academy/databases/indexing.md

---

### Q8: 🟡 Explain InnoDB's locking mechanisms: shared locks, exclusive locks, intention locks, and next-key locking.

**Strong Answer:**

InnoDB's locking system is multi-layered and far more sophisticated than simple "row locks." Understanding these locks is essential for debugging deadlocks, lock contention, and unexpected blocking in banking applications.

**Row-Level Locks (Record Locks):** These are the most granular locks in InnoDB.
- **Shared Lock (S Lock):** Acquired by `SELECT ... LOCK IN SHARE MODE`. Multiple transactions can hold S locks on the same row. S locks are compatible with each other but incompatible with X locks.
- **Exclusive Lock (X Lock):** Acquired by `SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`, and `INSERT`. Only one X lock can exist on a row. X locks are incompatible with both S and X locks.

**Intention Locks (Table-Level):** These are metadata locks that signal a transaction's intent to acquire row-level locks. They prevent DDL operations from conflicting with active transactions.
- **Intention Shared (IS):** "I plan to acquire S locks on some rows."
- **Intention Exclusive (IX):** "I plan to acquire X locks on some rows."

Intention locks are why `SELECT ... FOR UPDATE` blocks `ALTER TABLE` but doesn't block another `SELECT ... FOR UPDATE` on a different row — the IX locks are compatible with each other but incompatible with table-level X locks.

**Next-Key Locks (Gap + Record):** This is InnoDB's mechanism for preventing phantom reads under Repeatable Read isolation. A next-key lock combines:
- A **record lock** on the index entry itself
- A **gap lock** on the range before the index entry

```sql
-- Next-key locking example:
-- Assume accounts table with account_id: 100, 200, 300, 500

-- Transaction A (Repeatable Read):
BEGIN;
SELECT * FROM accounts WHERE account_id = 200 FOR UPDATE;
-- Acquires next-key lock on (200, 300) — locks record 200 AND gap before 300

-- Transaction B tries:
INSERT INTO accounts (account_id, balance) VALUES (250, 1000);
-- BLOCKED! Gap lock prevents inserting into (200, 300) range

INSERT INTO accounts (account_id, balance) VALUES (150, 1000);
-- Also BLOCKED! Previous next-key lock on (100, 200) range

INSERT INTO accounts (account_id, balance) VALUES (600, 1000);
-- ALLOWED — outside all next-key lock ranges

-- Gap-only locks (from non-existent row lookup):
SELECT * FROM accounts WHERE account_id = 250 FOR UPDATE;
-- No row 250 exists, so InnoDB places a gap lock on (200, 300)
-- Prevents any insert of account_id between 200 and 300

-- Check current locks
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_query AS blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx r ON w.requesting_trx_id = r.trx_id
JOIN information_schema.innodb_trx b ON w.blocking_trx_id = b.trx_id;

-- Lock information from performance schema
SELECT
    object_schema,
    object_name,
    index_name,
    lock_type,
    lock_mode,
    lock_status,
    lock_data
FROM performance_schema.data_locks;
```

**Lock Compatibility Matrix:**

| Requested \ Held | IS | IX | S | X |
|---|---|---|---|---|
| IS | ✓ | ✓ | ✓ | ✗ |
| IX | ✓ | ✓ | ✗ | ✗ |
| S | ✓ | ✗ | ✓ | ✗ |
| X | ✗ | ✗ | ✗ | ✗ |

**Phantom Read Prevention:** Under Repeatable Read, next-key locks prevent other transactions from inserting rows that would appear in the current transaction's range query results. This is stricter than the SQL standard — InnoDB's Repeatable Read actually prevents phantom reads through next-key locking, not just non-repeatable reads.

**Record vs. Gap Lock Optimization:** Under Read Committed isolation, InnoDB disables gap locking and next-key locking — it only locks the specific rows being modified. This reduces lock contention significantly for high-throughput systems but allows phantom reads. In banking, this trade-off is acceptable for high-volume operational queries where exact snapshot consistency isn't required.

**Key Points to Hit:**
- [ ] Shared (S) locks for reads; Exclusive (X) locks for writes
- [ ] Intention locks (IS/IX) are table-level metadata signaling row-level intent
- [ ] Next-key locks = record lock + gap lock (prevents phantom reads)
- [ ] Gap locks prevent inserts into ranges, not just updates to existing rows
- [ ] Read Committed disables gap locks; Repeatable Read enables them
- [ ] Deadlock detection: InnoDB builds a wait-for graph and aborts the transaction with the least work to rollback

**Follow-Up Questions:**
1. Two transactions are deadlocking. How do you determine the root cause from `SHOW ENGINE INNODB STATUS`?
2. Why does `SELECT * FROM table WHERE id = 999 FOR UPDATE` (non-existent row) block inserts?
3. How does `innodb_table_locks` affect LOCK TABLES vs. row-level locks?

**Source:** banking-genai-engineering-academy/databases/isolation-levels.md, databases/deadlocks.md

---

### Q9: 🟡 How do you analyze and optimize a slow MySQL query? Walk through EXPLAIN output and the slow query log.

**Strong Answer:**

Query optimization in MySQL is a systematic process that starts with identification, moves to analysis, and ends with remediation. The two primary tools are the **slow query log** for identification and **EXPLAIN** for analysis.

**Step 1: Enable and monitor the slow query log.** The slow query log captures queries that exceed a configurable execution time threshold. In banking environments, this is essential for catching performance regressions before they impact customer-facing services.

```sql
-- Enable slow query log (my.cnf or runtime)
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow-query.log';
SET GLOBAL long_query_time = 1;  -- Log queries taking > 1 second
SET GLOBAL log_queries_not_using_indexes = 'ON';  -- Also log full table scans

-- Analyze slow query log with mysqldumpslow
-- Command line:
-- mysqldumpslow -s t -t 20 /var/log/mysql/slow-query.log
-- -s t: sort by total time, -t 20: show top 20

-- Or use pt-query-digest (Percona Toolkit, more powerful)
-- pt-query-digest /var/log/mysql/slow-query.log
-- Groups similar queries, shows percentiles, identifies worst offenders

-- MySQL 8.0: Performance Schema alternative
-- No file I/O overhead, queryable via SQL
SELECT
    DIGEST_TEXT AS query_template,
    COUNT_STAR AS executions,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_ms,
    ROUND(MAX_TIMER_WAIT / 1000000000, 2) AS max_ms,
    ROUND(SUM_TIMER_WAIT / 1000000000, 2) AS total_seconds,
    SUM_ROWS_EXAMINED AS total_rows_examined,
    SUM_ROWS_SENT AS total_rows_sent,
    SUM_NO_INDEX_USED AS full_table_scans
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0  -- Only queries doing full table scans
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

**Step 2: Analyze with EXPLAIN.** EXPLAIN shows the optimizer's execution plan without running the query. The key columns to interpret:

```sql
EXPLAIN SELECT
    a.account_id,
    a.customer_id,
    SUM(t.amount) AS total_deposits
FROM accounts a
JOIN transactions t ON a.account_id = t.account_id
WHERE a.status = 'ACTIVE'
  AND t.txn_date >= '2025-01-01'
  AND t.status = 'DEPOSIT'
GROUP BY a.account_id, a.customer_id
HAVING SUM(t.amount) > 10000\G

-- Critical EXPLAIN columns:
-- type: Access method (best to worst):
--   system > const > eq_ref > ref > range > index > ALL
--   "ALL" = full table scan (bad); "ref" = index lookup (good)
--
-- key: Which index is actually used (NULL = no index)
--
-- rows: Estimated rows examined (lower is better)
--
-- Extra: Additional info:
--   "Using where" — filtering after index lookup
--   "Using index" — covering index (no table lookup needed!)
--   "Using filesort" — sorting in memory/disk (potential issue)
--   "Using temporary" — temporary table for GROUP BY (potential issue)
--   "Using index condition" — Index Condition Pushdown (ICP)
--
-- filtered: Estimated % of rows that pass the WHERE filter
```

**Step 3: Optimize based on findings.** Common optimization patterns:

```sql
-- Problem: type=ALL (full table scan) on transactions
-- Solution: Add appropriate index
CREATE INDEX idx_txn_account_date_status ON transactions(account_id, status, txn_date);

-- Problem: Using filesort on large result set
-- Solution: Index covers ORDER BY columns
-- If query has ORDER BY txn_date DESC and index ends with txn_date, filesort eliminated

-- Problem: Using temporary for GROUP BY
-- Solution: Covering index that includes GROUP BY columns
-- Or reduce result set with more selective WHERE clauses

-- EXPLAIN FORMAT=JSON (more detailed)
EXPLAIN FORMAT=JSON
SELECT * FROM transactions WHERE account_id = 12345 AND status = 'COMPLETED'\G
-- Shows: cost estimates, index conditions, ICP usage, materialization decisions

-- ANALYZE TABLE ensures optimizer has current statistics
ANALYZE TABLE transactions;
-- Stale statistics cause the optimizer to choose suboptimal plans

-- Force index (use as last resort — usually indicates stale statistics)
SELECT * FROM transactions
FORCE INDEX (idx_txn_account_date_status)
WHERE account_id = 12345 AND status = 'COMPLETED';
```

**Systematic optimization checklist:**
1. Ensure `ANALYZE TABLE` has been run (stale stats = bad plans)
2. Check if appropriate indexes exist (EXPLAIN shows `key: NULL`)
3. Verify query structure avoids anti-patterns (SELECT *, function on indexed columns)
4. Consider covering indexes to eliminate table lookups
5. Check if query can be rewritten (subquery → JOIN, UNION → conditional aggregation)
6. Evaluate if query is running on the right isolation level
7. Consider query result caching at the application level

**Key Points to Hit:**
- [ ] Slow query log identifies queries exceeding time threshold
- [ ] Performance Schema provides query statistics without file I/O
- [ ] EXPLAIN `type` column: ALL (worst) → const (best)
- [ ] "Using filesort" and "Using temporary" indicate potential bottlenecks
- [ ] "Using index" means covering index — no table lookup needed
- [ ] EXPLAIN FORMAT=JSON shows optimizer cost estimates
- [ ] Stale statistics (missing ANALYZE TABLE) cause suboptimal plans

**Follow-Up Questions:**
1. EXPLAIN shows `type: ref` but the query is still slow. What's the next step?
2. How does Index Condition Pushdown (ICP) improve performance?
3. Your query uses `Using temporary` and `Using filesort`. How do you eliminate both?

**Source:** banking-genai-engineering-academy/databases/postgres-performance.md, databases/query-tuning.md, databases/indexing.md

---

### Q10: 🟡 Describe MySQL replication types: asynchronous, semi-synchronous, and Group Replication. When do you use each?

**Strong Answer:**

MySQL offers multiple replication architectures, each with different trade-offs between performance, consistency, and operational complexity. Choosing the right one is critical for banking systems where data loss is unacceptable but latency requirements vary by workload.

**Asynchronous Replication (Default):** The primary writes changes to its binary log and acknowledges the client immediately. Replicas pull from the binary log at their own pace. The primary has no knowledge of whether replicas have received or applied the changes.

```
Client → Primary (write) → COMMIT acknowledged immediately
              ↓
        Binary log
              ↓ (pull, async)
        Replica 1     Replica 2
        (may lag)     (may lag)
```

```sql
-- Asynchronous replication setup
-- Primary:
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'secure_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

SHOW MASTER STATUS;
-- Returns: File (binlog name) and Position (offset)

-- Replica:
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='primary-host',
    SOURCE_USER='repl_user',
    SOURCE_PASSWORD='secure_password',
    SOURCE_LOG_FILE='binlog.000001',
    SOURCE_LOG_POS=154;

START REPLICA;
SHOW REPLICA STATUS\G
-- Key fields:
--   Replica_IO_Running: Yes/No (connected to primary?)
--   Replica_SQL_Running: Yes/No (applying events?)
--   Seconds_Behind_Source: replication lag in seconds
--   Last_IO_Error / Last_SQL_Error: error messages
```

**Semi-Synchronous Replication:** The primary waits until at least one replica has received and acknowledged the write (written to its relay log) before acknowledging the client. This guarantees that at least one replica has the data, protecting against primary data loss. It does NOT guarantee the replica has applied the changes — only received them.

```sql
-- Enable semi-synchronous (both primary and replicas need plugins)
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';

SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_timeout = 1000;  -- Fall back to async after 1s

-- On replica:
SET GLOBAL rpl_semi_sync_replica_enabled = 1;

-- Monitor
SHOW STATUS LIKE 'Rpl_semi_sync%';
-- Rpl_semi_sync_source_yes_no: how many commits were semi-sync
-- Rpl_semi_sync_source_timeouts: how many fell back to async
```

**Group Replication (MySQL 8.0+):** This is MySQL's built-in high-availability solution based on distributed consensus (Paxos variant). It provides multi-primary or single-primary mode with automatic failover. Writes are coordinated through a consensus protocol — a write is only committed when a majority of the group agrees.

```sql
-- Group Replication setup (simplified)
-- All nodes must have same schema, same configuration
-- Requires: server_id unique, GTIDs enabled, binary log, slave_updates

-- Single-primary mode (recommended for banking):
-- One primary handles all writes; replicas are read-only
-- Automatic failover if primary fails

-- Multi-primary mode:
-- All nodes accept writes
-- Conflicts detected via certification-based conflict detection
-- Higher risk of conflicts (deadlock equivalents)

-- Check group status
SELECT * FROM performance_schema.replication_group_members;
-- MEMBER_STATE: ONLINE, RECOVERING, UNREACHABLE, ERROR, OFFLINE
-- MEMBER_ROLE: PRIMARY, SECONDARY

SELECT * FROM performance_schema.replication_group_member_stats;
-- Shows: conflicts detected, transactions checked, certification info
```

**When to use each:**

| Replication Type | Data Loss Risk | Write Latency | Use Case |
|---|---|---|---|
| Asynchronous | Yes (window of data loss) | None | Read scaling, analytics, DR standby |
| Semi-Synchronous | Minimal (relay log received) | +network RTT | Production with moderate HA requirements |
| Group Replication | None (majority consensus) | +consensus overhead | Mission-critical banking with zero RPO |

For Citi's banking workloads, the recommended architecture is:
- **Group Replication** (single-primary) for the core transaction database — ensures zero data loss with automatic failover
- **Semi-synchronous replication** to a remote data center — provides DR protection with bounded latency impact
- **Asynchronous replicas** for analytics and reporting — no write latency impact

**Key Points to Hit:**
- [ ] Async replication: primary doesn't wait for replicas — risk of data loss on primary failure
- [ ] Semi-sync: waits for at least one replica to receive write — protects against primary loss
- [ ] Group Replication: consensus-based (Paxos), majority commits — zero data loss
- [ ] Single-primary vs multi-primary modes in Group Replication
- [ ] GTID-based replication simplifies failover (no manual binlog position tracking)
- [ ] Banking recommendation: Group Replication for HA + async replicas for read scaling

**Follow-Up Questions:**
1. Semi-sync replication is timing out frequently and falling back to async. What's the root cause?
2. How does Group Replication handle split-brain scenarios?
3. A replica is 4 hours behind the primary. How do you bring it back without rebuilding?

**Source:** banking-genai-engineering-academy/databases/replication.md, databases/high-availability.md

---

### Q11: 🟡 Explain GTID-based replication. How does it simplify failover compared to traditional file-position replication?

**Strong Answer:**

GTID (Global Transaction ID) replication, introduced in MySQL 5.6 and enhanced in 5.7/8.0, is a fundamental improvement to MySQL replication that replaces the traditional binlog filename + position method with globally unique transaction identifiers.

**What is a GTID?** Each transaction committed on the primary is assigned a unique GTID in the format `source_uuid:transaction_id`. For example, `3E11FA47-71CA-11E1-9E33-C80AA9429562:23` means this is transaction #23 from the server with that UUID. The GTID is written to the binary log and travels with the transaction to all replicas.

**The key innovation:** Instead of telling a replica "start reading from binlog.000042 at position 1547," you tell it "execute all transactions after GTID X." The replica automatically finds where that GTID exists in its binary logs and continues from there.

```sql
-- Enable GTIDs (must be set before starting replication)
-- my.cnf on all servers:
-- gtid_mode = ON
-- enforce_gtid_consistency = ON
-- log_bin = binlog
-- log_slave_updates = ON  -- Required for GTID with chain replication

-- Create GTID-based replication
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='primary-host',
    SOURCE_USER='repl_user',
    SOURCE_PASSWORD='secure_password',
    SOURCE_AUTO_POSITION = 1;  -- THIS is the GTID magic
    -- No SOURCE_LOG_FILE or SOURCE_LOG_POS needed!

START REPLICA;

-- GTID execution state
SELECT @@gtid_executed;
-- e.g., "3E11FA47-71CA-11E1-9E33-C80AA9429562:1-1000"
-- Means: transactions 1 through 1000 from this server UUID have been executed

-- GTID purged (transactions removed from binary logs but tracked)
SELECT @@gtid_purged;

-- Monitor GTID replication
SHOW REPLICA STATUS\G
-- Retrieved_Gtid_Set: GTIDs received from primary
-- Executed_Gtid_Set: GTIDs actually applied
-- Gap between these = pending transactions

-- Skip a specific GTID (for error recovery)
SET SESSION gtid_next = '3E11FA47-71CA-11E1-9E33-C80AA9429562:1001';
BEGIN; COMMIT;
SET SESSION gtid_next = 'AUTOMATIC';
```

**Failover comparison — traditional vs GTID:**

**Traditional (file-position):**
```
1. Find primary's last binlog position: SHOW MASTER STATUS → binlog.000042:1547
2. Find the replica that has the most up-to-date position
3. Calculate which binlog file and position the replica has applied
4. Point other replicas to the new primary with CHANGE SOURCE TO
   MASTER_LOG_FILE='binlog.000043', MASTER_LOG_POS=1547
5. If the chosen replica was behind, you've lost transactions
6. If any replica executed transactions that the old primary didn't
   replicate (from delayed relay log application), you have divergence
```

**GTID-based:**
```
1. Find the replica with the most advanced gtid_executed
2. Promote it to primary (e.g., via Orchestrator or MySQL Shell)
3. Point other replicas: CHANGE MASTER TO SOURCE_AUTO_POSITION=1
4. Replicas automatically find their position and continue
5. GTID guarantees no duplicate execution and no missing transactions
6. The process is deterministic and scriptable
```

**GTID constraints (important for banking):**
1. **Non-transactional storage engines (MyISAM) are forbidden** with `enforce_gtid_consistency = ON`. This is actually a good constraint for banking since MyISAM shouldn't be used anyway.
2. **`CREATE TABLE ... SELECT` is forbidden** with GTIDs because it mixes DDL and DML in a way that can't be safely replicated.
3. **`CREATE TEMPORARY TABLE` is allowed** but not inside transactions.

For production banking failover, tools like **MySQL Orchestrator** or **MySQL InnoDB Cluster** (MySQL Shell with Group Replication) automate GTID-based failover with health checks, topology management, and automatic replica reconfiguration.

**Key Points to Hit:**
- [ ] GTID format: source_uuid:transaction_id (globally unique per transaction)
- [ ] `SOURCE_AUTO_POSITION = 1` eliminates manual binlog position tracking
- [ ] Failover is deterministic — replicas find their position automatically
- [ ] No risk of duplicate or missing transactions during failover
- [ ] `enforce_gtid_consistency` restricts certain operations for replication safety
- [ ] MySQL Orchestrator / InnoDB Cluster automate GTID-based failover

**Follow-Up Questions:**
1. A replica's `gtid_executed` is behind the primary by 500 GTIDs. Can you skip them?
2. How do GTIDs work with multi-source replication (one replica, multiple primaries)?
3. What happens when binary logs are purged but a replica hasn't caught up?

**Source:** banking-genai-engineering-academy/databases/replication.md, databases/high-availability.md

---

### Q12: 🟡 What is the MySQL slow query log, and how do you use it effectively in a production banking environment?

**Strong Answer:**

The slow query log is MySQL's primary diagnostic tool for identifying performance bottlenecks. It captures queries that exceed a configurable execution time threshold, along with optional metadata like lock time, rows examined, and rows sent. In a banking environment where SLA compliance is measured in milliseconds, the slow query log is essential for maintaining performance guarantees.

```sql
-- Production configuration (my.cnf)
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 0.5  -- 500ms threshold (adjust per SLA)
log_queries_not_using_indexes = 1  -- Catch full table scans even if fast
log_throttle_queries_not_using_indexes = 10  -- Rate limit: max 10/min
log_output = FILE  -- Or TABLE for queryable slow_log table
min_examined_row_limit = 1000  -- Skip queries examining few rows
```

The configuration choices matter significantly:

- `long_query_time = 0.5` (500ms) is appropriate for OLTP banking APIs where the SLA is typically <100ms. Setting it too low (e.g., 10ms) creates excessive log volume; too high (e.g., 5s) means you miss gradual degradation.
- `log_queries_not_using_indexes = 1` is critical because a full table scan on a small table might finish in 1ms but becomes catastrophic as the table grows. This catches the problem early.
- `log_throttle_queries_not_using_indexes` prevents log flooding when a misconfigured query without an index runs frequently.

```bash
# Analyze slow query log with mysqldumpslow
# Sort by total time, top 20 queries
mysqldumpslow -s t -t 20 /var/log/mysql/slow-query.log

# Sort by average time (finds worst individual queries)
mysqldumpslow -s at -t 20 /var/log/mysql/slow-query.log

# Sort by count (finds most frequently slow queries)
mysqldumpslow -s c -t 20 /var/log/mysql/slow-query.log

# Percona Toolkit's pt-query-digest (far more detailed)
pt-query-digest /var/log/mysql/slow-query.log \
  --report-format=profile \
  --limit=20 \
  --since=24h  # Only last 24 hours

# Output shows:
# - Query fingerprint (normalized, parameters replaced with ?)
# - Percentile response times (P50, P95, P99)
# - Total and average execution time
# - Rows examined vs rows sent ratio (indicates inefficient queries)
# - Index usage statistics
```

**Production banking practices:**

1. **Automated alerting:** Parse the slow query log (or Performance Schema) every 5 minutes. Alert when the count of slow queries exceeds a threshold or when a previously-fast query suddenly appears in the log.

2. **Performance Schema as alternative (MySQL 5.7+):** The `performance_schema.events_statements_summary_by_digest` table provides aggregated query statistics without file I/O overhead. This is preferred for high-throughput production systems.

```sql
-- Find the top 10 worst queries by total execution time
SELECT
    SCHEMA_NAME,
    DIGEST_TEXT AS query_pattern,
    COUNT_STAR AS execution_count,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_ms,
    ROUND(MAX_TIMER_WAIT / 1000000000, 2) AS max_ms,
    ROUND(SUM_TIMER_WAIT / 1000000000, 2) AS total_seconds,
    SUM_ROWS_EXAMINED AS total_rows_examined,
    SUM_ROWS_SENT AS total_rows_sent,
    SUM_NO_INDEX_USED AS no_index_used_count,
    SUM_NO_GOOD_INDEX_USED AS no_good_index_used_count,
    FIRST_SEEN,
    LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'banking_db'
  AND COUNT_STAR > 100  -- Only frequently executed queries
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Find queries with high variance (sometimes fast, sometimes slow)
SELECT
    DIGEST_TEXT,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_ms,
    ROUND(MAX_TIMER_WAIT / 1000000000, 2) AS max_ms,
    ROUND(STDDEV_TIMER_WAIT / 1000000000, 2) AS stddev_ms,
    ROUND(MAX_TIMER_WAIT / NULLIF(AVG_TIMER_WAIT, 0), 1) AS max_avg_ratio
FROM performance_schema.events_statements_summary_by_digest
ORDER BY max_avg_ratio DESC
LIMIT 10;
-- High max/avg ratio indicates intermittent performance issues
-- Common causes: lock waits, buffer pool eviction, table stats staleness
```

3. **Query fingerprinting and trending:** Store query digest statistics in a time-series database (Prometheus/Grafana) to track query performance trends over time. A gradual increase in avg execution time often indicates table growth without corresponding index optimization.

4. **Regulatory compliance:** In banking, the slow query log should be retained for audit purposes. Configure log rotation with compression and ship logs to a centralized logging system (ELK stack, Splunk) for long-term analysis.

**Key Points to Hit:**
- [ ] `long_query_time` sets threshold; `log_queries_not_using_indexes` catches full table scans
- [ ] `mysqldumpslow` and `pt-query-digest` aggregate and rank slow queries
- [ ] Performance Schema provides queryable stats without file I/O
- [ ] High max/avg ratio indicates intermittent issues (lock waits, cache misses)
- [ ] Rows examined vs rows sent ratio reveals query efficiency
- [ ] Production: automated alerting, trend analysis, log retention for compliance

**Follow-Up Questions:**
1. The slow query log is growing at 2GB/hour. How do you manage it without losing data?
2. A query appears in the slow log intermittently but EXPLAIN shows it uses an index. What's happening?
3. How do you correlate slow queries with specific application endpoints?

**Source:** banking-genai-engineering-academy/databases/postgres-performance.md, databases/query-tuning.md

---

### Q13: 🟡 Compare MySQL and PostgreSQL for a banking workload. When would you choose MySQL over PostgreSQL?

**Strong Answer:**

This is perhaps the most practical architecture question for a Citi GenAI role, as the team uses both databases. The answer requires understanding not just technical differences but also operational, ecosystem, and organizational factors that influence database selection in a regulated banking environment.

**Architecture differences that matter:**

| Aspect | MySQL/InnoDB | PostgreSQL |
|--------|-------------|------------|
| Data storage | Clustered (data in PK index) | Heap (separate from indexes) |
| MVCC | Undo log in separate tablespace | Old versions in heap table (needs VACUUM) |
| Replication | Binlog-based + Group Replication | Streaming + Logical replication |
| JSON support | JSON type (stored as LONGBLOB internally) | JSONB (binary, indexed, operators) |
| Full-text search | Built-in (basic) | Excellent with GIN + tsvector |
| Connection model | Thread-per-connection (requires pooler) | Process-per-connection (requires PgBouncer) |
| Extensions | Limited plugin architecture | Rich extension system (PostGIS, pgvector) |
| Window functions | Available since 8.0 | Available since 8.4 (2008) |
| CTEs | Available since 8.0 | Available since 8.4 (2008) |

**When MySQL is the better choice for banking:**

1. **Read-heavy workloads with simple query patterns:** MySQL's query optimizer, while less sophisticated than PostgreSQL's, is extremely fast for simple lookups and joins. If the workload is primarily `SELECT ... WHERE indexed_column = ?`, MySQL often outperforms PostgreSQL with lower operational overhead.

2. **Existing MySQL ecosystem:** If the team already has MySQL DBAs, tooling (Percona, ProxySQL, Orchestrator), and monitoring in place, switching to PostgreSQL introduces operational risk. Banking environments have high switching costs.

3. **MySQL Group Replication needs:** For multi-primary or automatic failover requirements, MySQL Group Replication is more mature and easier to operate than PostgreSQL's equivalent (Patroni + etcd).

4. **Simpler operational model:** MySQL has fewer configuration knobs to tune. The default configuration is closer to production-ready, reducing the operational expertise required.

5. **Embedding and application-level simplicity:** MySQL's connector ecosystem (especially for Java/Spring, which Citi uses extensively) is excellent. The MySQL JDBC driver is more battle-tested in enterprise environments than PostgreSQL's.

**When PostgreSQL is the better choice:**

1. **Complex analytical queries:** PostgreSQL's optimizer handles complex joins, CTEs, window functions, and subqueries far better than MySQL. For risk modeling, fraud detection, and regulatory reporting, PostgreSQL is superior.

2. **JSONB and document storage:** PostgreSQL's JSONB with GIN indexes is the best JSON implementation of any relational database. If the banking application needs flexible schema for GenAI prompt templates, document processing, or semi-structured audit data, PostgreSQL wins decisively.

3. **Geospatial requirements:** PostGIS is unmatched. For location-based banking services (branch locator, ATM finder with distance calculations), PostgreSQL is the only choice.

4. **Advanced indexing needs:** GIN, GiST, BRIN, and HNSW indexes enable query patterns that MySQL cannot efficiently support (array containment, vector similarity, range overlap).

5. **Regulatory compliance features:** PostgreSQL's row-level security, column-level privileges, and audit logging via extensions (`pgaudit`) provide more granular compliance controls than MySQL's role-based access.

**Migration considerations:**

```sql
-- Common migration challenges from MySQL to PostgreSQL:
-- 1. AUTO_INCREMENT → SERIAL / IDENTITY
-- 2. IFNULL() → COALESCE()
-- 3. NOW() → CURRENT_TIMESTAMP
-- 4. LIMIT offset, count → LIMIT count OFFSET offset
-- 5. backtick identifiers → double-quote or no quotes
-- 6. ENUM types → PostgreSQL ENUM or CHECK constraints
-- 7. TINYINT(1) → BOOLEAN
-- 8. DATETIME → TIMESTAMP WITH TIME ZONE

-- Example: Banking table migration
-- MySQL:
CREATE TABLE transactions (
    txn_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    status ENUM('PENDING','COMPLETED','FAILED') DEFAULT 'PENDING',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- PostgreSQL equivalent:
CREATE TABLE transactions (
    txn_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    account_id BIGINT NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING'
        CHECK (status IN ('PENDING', 'COMPLETED', 'FAILED')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

For Citi's GenAI role, the practical answer is: **use PostgreSQL for the GenAI platform** (vector search with pgvector, complex JSON processing for prompt templates, advanced analytical queries) and **maintain MySQL for existing core banking services** where migration cost exceeds the benefits. The integration layer between them should use a message queue (Kafka) for change data capture.

**Key Points to Hit:**
- [ ] MySQL excels at simple read-heavy OLTP with fast single-table lookups
- [ ] PostgreSQL excels at complex queries, JSONB, advanced indexing, analytics
- [ ] Clustered vs heap storage affects query patterns and index design
- [ ] MVCC implementation differs: undo tablespace (MySQL) vs heap bloat (PostgreSQL)
- [ ] Group Replication (MySQL) vs Patroni (PostgreSQL) for HA
- [ ] Migration requires syntax, type, and behavior changes — not just data transfer
- [ ] Decision factors: existing expertise, query complexity, feature requirements

**Follow-Up Questions:**
1. A MySQL query runs in 10ms but the equivalent PostgreSQL query takes 50ms. How do you investigate?
2. How do you replicate data between MySQL and PostgreSQL for a migration?
3. When would you run both MySQL and PostgreSQL in the same banking architecture?

**Source:** banking-genai-engineering-academy/databases/postgres-fundamentals.md, databases/indexing.md, databases/jsonb.md, databases/replication.md

---

### Q14: 🟡 Explain deadlock detection and resolution in InnoDB. How do you prevent deadlocks in banking applications?

**Strong Answer:**

A deadlock occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency. InnoDB has a built-in deadlock detector that runs periodically (or on each lock wait when `innodb_deadlock_detect = ON`, the default).

**How deadlock detection works:** InnoDB constructs a **wait-for graph** where nodes are transactions and edges represent "transaction A is waiting for a lock held by transaction B." When the graph contains a cycle, InnoDB has detected a deadlock. It then chooses a **victim transaction** to rollback — the one with the least "work" to undo (fewest row modifications). The victim receives error 1213: `Deadlock found when trying to get lock; try restarting transaction`.

```sql
-- View deadlock information
SHOW ENGINE INNODB STATUS\G
-- Look for "LATEST DETECTED DEADLOCK" section
-- Shows:
--   - Both transactions involved
--   - What locks each holds
--   - What locks each is waiting for
--   - The chosen victim

-- Example deadlock scenario:
-- Transaction A:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;  -- Locks row 1 (X lock)
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;  -- Waits for row 2 (X lock held by B)

-- Transaction B:
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE account_id = 2;   -- Locks row 2 (X lock)
UPDATE accounts SET balance = balance + 50 WHERE account_id = 1;   -- Waits for row 1 (X lock held by A)
-- → DEADLOCK! A waits for row 2 (held by B), B waits for row 1 (held by A)

-- Deadlock metrics from performance schema
SELECT
    deadlocks
FROM sys.innodb_lock_waits;

-- Monitor deadlock frequency
SHOW STATUS LIKE 'Innodb_deadlocks';
-- Cumulative count since server start

-- Disable deadlock detection (NOT recommended — use only for debugging)
SET GLOBAL innodb_deadlock_detect = OFF;
-- Then configure innodb_lock_wait_timeout instead
-- Default: 50 seconds — transaction waits this long before timeout
```

**Prevention strategies for banking applications:**

1. **Consistent lock ordering:** The single most effective prevention technique. If all transactions acquire locks in the same order (e.g., always lock account_id in ascending order), circular dependencies cannot occur.

```python
# Python example: consistent lock ordering
def transfer_funds(from_account: int, to_account: int, amount: float):
    """Always lock accounts in ascending order to prevent deadlocks."""
    first = min(from_account, to_account)
    second = max(from_account, to_account)

    with transaction.atomic():
        # Lock in consistent order regardless of transfer direction
        cursor.execute("SELECT balance FROM accounts WHERE account_id = %s FOR UPDATE", (first,))
        cursor.execute("SELECT balance FROM accounts WHERE account_id = %s FOR UPDATE", (second,))
        # Now perform the transfer — no deadlock possible
        ...
```

2. **Keep transactions short:** The longer a transaction holds locks, the higher the probability of conflict. In banking, this means:
   - Do validation before starting the transaction
   - Avoid user interaction within a transaction
   - Batch operations to reduce transaction count
   - Set `innodb_lock_wait_timeout` appropriately (5-10 seconds for OLTP, not the default 50)

3. **Use appropriate isolation level:** Under Read Committed, gap locks are disabled, reducing the surface area for deadlocks. If phantom reads are acceptable for the specific operation, this is a practical trade-off.

4. **Retry logic:** Even with perfect prevention, deadlocks can occur from concurrent access patterns. The application must handle error 1213 with exponential backoff retry.

```python
import time
from mysql.connector import errors

def execute_with_deadlock_retry(query, params, max_retries=3):
    """Execute query with automatic deadlock retry."""
    for attempt in range(max_retries):
        try:
            cursor.execute(query, params)
            return cursor.fetchall()
        except errors.DatabaseError as e:
            if e.errno == 1213 and attempt < max_retries - 1:
                wait_time = 0.1 * (2 ** attempt)  # Exponential backoff: 0.1s, 0.2s, 0.4s
                time.sleep(wait_time)
                continue
            raise
```

5. **Application-level queuing:** For high-contention operations (e.g., end-of-day batch processing for a specific account), serialize access at the application level using a message queue (Kafka, RabbitMQ) rather than relying on database locks.

**Key Points to Hit:**
- [ ] Deadlock detection uses wait-for graph cycle detection
- [ ] Victim selection: transaction with least undo work is rolled back
- [ ] Consistent lock ordering prevents circular dependencies
- [ ] Short transactions reduce lock hold time and deadlock probability
- [ ] Error 1213 requires application-level retry with exponential backoff
- [ ] Read Committed reduces deadlock surface area (no gap locks)
- [ ] `innodb_lock_wait_timeout` provides a safety net (default 50s)

**Follow-Up Questions:**
1. Deadlocks are increasing after a deployment but no code changed the query logic. What could cause this?
2. How does `innodb_deadlock_detect = OFF` change the behavior? What must you configure instead?
3. In Group Replication, how do deadlocks differ from single-node InnoDB?

**Source:** banking-genai-engineering-academy/databases/deadlocks.md, databases/isolation-levels.md, databases/transactions.md

---

### Q15: 🟡 How does InnoDB handle crash recovery? Walk through the redo log and doublewrite buffer mechanisms.

**Strong Answer:**

InnoDB's crash recovery is what makes it suitable for banking workloads where data integrity is non-negotiable. The mechanism relies on two components: the **redo log** (for forward recovery) and the **doublewrite buffer** (for corruption protection).

**The Write Path:**

When a transaction commits, InnoDB does NOT immediately write modified pages to the data files. Instead, it follows this sequence:

1. **Modify buffer pool pages:** The row changes are made to pages in the buffer pool. These pages become "dirty."
2. **Write to redo log buffer:** The logical description of what changed (the redo record) is written to the in-memory redo log buffer.
3. **Flush redo log to disk:** On COMMIT, the redo log buffer is flushed to the redo log files (`ib_logfile0`, `ib_logfile1`) on disk. This is a sequential write, which is fast.
4. **Acknowledge COMMIT:** Once the redo log is on disk, the commit is acknowledged to the client.
5. **Asynchronous page flush:** Dirty pages are flushed to the data files (`*.ibd`) later by background threads (the page cleaner threads).

```
Write path:
Client issues UPDATE → Buffer pool page modified (dirty)
                      → Redo log record written to redo buffer
                      → On COMMIT: redo buffer flushed to disk (sequential write)
                      → COMMIT acknowledged to client
                      → Later: dirty pages flushed to .ibd files (random write, async)
```

**Crash Recovery Process:**

When InnoDB starts after a crash, it goes through three phases:

**Phase 1 — Analysis:** InnoDB reads the redo log from the last checkpoint to determine what work needs to be done. It identifies the LSN (Log Sequence Number) of the last checkpoint and the LSN of the last committed transaction.

**Phase 2 — Redo (Forward Recovery):** InnoDB replays all redo log records from the checkpoint forward. This brings all pages in the buffer pool to their state at the time of the crash, including both committed and uncommitted changes.

**Phase 3 — Undo (Rollback):** InnoDB uses the undo log to roll back any transactions that were not committed at the time of the crash. This restores the database to a consistent state containing only committed transactions.

```sql
-- Redo log configuration
SHOW VARIABLES LIKE 'innodb_log_file%';
-- innodb_log_file_size: Size of each redo log file (default 48MB in 8.0)
-- Larger = more efficient for write-heavy workloads (less frequent checkpoint)
-- But longer recovery time after crash

-- innodb_log_files_in_group: Number of files in the redo log group (default 2)
-- Circular: when file 0 fills, writing continues to file 1, then wraps

SHOW VARIABLES LIKE 'innodb_log_buffer_size';
-- Default: 16MB. For heavy write workloads, increase to 64-256MB.
-- Larger buffer = fewer flushes = better write throughput

SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
-- 1 (default): Flush to disk every COMMIT — full durability
-- 2: Write to OS cache, flush to disk once/sec — loses up to 1s on OS crash
-- 0: Flush to disk once/sec — loses up to 1s on MySQL crash

-- Checkpoint information
SHOW ENGINE INNODB STATUS\G
-- Look for "Log sequence number" and "Last checkpoint at"
-- Difference = amount of redo to replay on recovery

-- Recovery simulation (don't run in production!)
-- Force crash recovery by killing mysqld -9
-- On restart, InnoDB automatically runs recovery:
-- [Note] InnoDB: Starting crash recovery.
-- [Note] InnoDB: Doing recovery: scanned up to log sequence number XXX
-- [Note] InnoDB: 1 transaction(s) which must be rolled back
-- [Note] InnoDB: Starting an apply batch of log records...
-- [Note] InnoDB: Progress in percent: 10 20 30 ... 100
-- [Note] InnoDB: Rollback of non-committed transactions completed
```

**Doublewrite Buffer:** This is a safety mechanism for partial page writes. When InnoDB flushes a dirty page to disk, it first writes the page to the doublewrite buffer (a reserved area in the system tablespace), then writes it to the actual data file. If a crash occurs during the write to the data file (e.g., power loss mid-write), the page in the data file may be partially written — a "torn page." On recovery, InnoDB detects the torn page (via page checksums) and restores it from the doublewrite buffer copy, then replays the redo log.

```sql
-- Doublewrite buffer configuration
SHOW VARIABLES LIKE 'innodb_doublewrite';
-- ON (default): Enabled — protects against partial page writes
-- OFF: Disabled — higher write performance but risk of corruption
-- 2: MySQL 8.0.20+ "doublewrite batch" mode — improved performance

-- NEVER disable doublewrite buffer in production, especially banking
-- The only valid reason to disable is benchmarking where you want to measure
-- raw write throughput without durability guarantees

-- Page checksums (detects corruption)
SHOW VARIABLES LIKE 'innodb_checksum_algorithm';
-- crc32 (default): Fast and reliable
-- strict_crc32: Also checks that algorithm is crc32
-- innodb: Older algorithm
-- none: No checksums (dangerous)
```

**Banking-specific considerations:**

1. `innodb_flush_log_at_trx_commit = 1` is mandatory for regulated banking systems. Any other setting means you can lose committed transactions, which is unacceptable for financial records.

2. The redo log size should be sized for your peak write throughput. If the redo log fills up before the next checkpoint, InnoDB must flush dirty pages synchronously, causing performance spikes. Monitor `Innodb_os_log_fsyncs` — if it's very high, increase `innodb_log_file_size`.

3. In a multi-data-center setup, the redo log provides the basis for replication. The binary log (which is based on the redo log) is what gets streamed to replicas.

**Key Points to Hit:**
- [ ] Redo log: sequential write on COMMIT, ensures durability before page flush
- [ ] Crash recovery phases: Analysis → Redo (forward) → Undo (rollback)
- [ ] Doublewrite buffer: protects against torn pages (partial writes)
- [ ] `innodb_flush_log_at_trx_commit = 1` is required for full ACID
- [ ] Larger redo log files improve write throughput but increase recovery time
- [ ] Page checksums detect corruption and trigger doublewrite recovery
- [ ] Dirty pages flushed asynchronously — committed data may not be in .ibd files yet

**Follow-Up Questions:**
1. The redo log is full and InnoDB has stopped accepting writes. What's the root cause?
2. How does the doublewrite buffer impact write performance? Quantify the overhead.
3. If you lose the redo log files but keep the data files, can you recover?

**Source:** banking-genai-engineering-academy/databases/postgres-fundamentals.md (WAL comparison), databases/transactions.md, databases/backups.md

---

### Q16: 🔴 Explain InnoDB Cluster, MySQL Shell, and automated failover. How does this compare to PostgreSQL's Patroni?

**Strong Answer:**

InnoDB Cluster is MySQL's integrated high-availability solution that combines **Group Replication** (data replication with consensus), **MySQL Shell** (administration interface), and **MySQL Router** (connection routing) into a single managed package. It provides automated failover, self-healing, and topology management without third-party tools.

**Architecture:**

```
                    ┌─────────────────┐
                    │  MySQL Router   │
                    │  (connection     │
                    │   routing)       │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
    │   Primary      │ │ Secondary │ │ Secondary │
    │   (read-write) │ │ (read-    │ │ (read-    │
    │                │ │  only)    │ │  only)    │
    │  Node 1        │ │ Node 2    │ │ Node 3    │
    └────────┬───────┘ └─────┬─────┘ └─────┬─────┘
             │               │              │
             └───────────────┼──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │    Group Replication        │
              │    (Paxos consensus)        │
              │    ┌─────────────────┐      │
              │    │ Consensus Layer │      │
              │    │ XCom Protocol   │      │
              │    └─────────────────┘      │
              └─────────────────────────────┘
```

**Creating a cluster with MySQL Shell:**

```javascript
// MySQL Shell (JavaScript mode) — cluster setup

// 1. Configure instances (must be run on each node)
dba.configureInstance('root@node1:3306', {clusterAdmin: "'admin'@'%'", clusterAdminPassword: "secure"});
dba.configureInstance('root@node2:3306', {clusterAdmin: "'admin'@'%'", clusterAdminPassword: "secure"});
dba.configureInstance('root@node3:3306', {clusterAdmin: "'admin'@'%'", clusterAdminPassword: "secure"});

// 2. Create the cluster (on the primary node)
var cluster = dba.createCluster('banking_cluster', {adoptFromGR: true});

// 3. Add replicas
cluster.addInstance('root@node2:3306');
cluster.addInstance('root@node3:3306');

// 4. Check status
cluster.status();
// Shows: topology, member states, replication lag, recovery status

// 5. Handle a failed node
cluster.removeInstance('root@node2:3306');  // Permanently remove
cluster.rejoinInstance('root@node2:3306');  // Re-add after repair

// 6. Switch primary (manual failover)
cluster.setPrimaryInstance('root@node2:3306');

// 7. Dissolve cluster (emergency)
cluster.dissolve({force: true});
```

**MySQL Router** sits between applications and the cluster, automatically routing write queries to the primary and read queries to secondaries. When failover occurs, Router detects the new primary within seconds and updates its routing table — applications reconnect transparently.

```ini
# MySQL Router configuration (mysqlrouter.conf)
[routing:banking_rw]
bind_address = 0.0.0.0
bind_port = 6446
destinations = metadata-cache://banking_cluster/?role=PRIMARY
routing_strategy = first-available

[routing:banking_ro]
bind_address = 0.0.0.0
bind_port = 6447
destinations = metadata-cache://banking_cluster/?role=SECONDARY
routing_strategy = round-robin-with-fallback
```

**Automated failover flow:**
1. Primary node becomes unreachable (detected by Group Replication's XCom protocol)
2. Remaining members elect a new primary through distributed consensus
3. MySQL Router detects the topology change via metadata cache
4. Router updates its routing table — new connections go to the new primary
5. Existing connections to the old primary receive an error and reconnect
6. When the old primary recovers, it rejoins as a secondary (automatic state transfer)

**Comparison with PostgreSQL's Patroni:**

| Aspect | MySQL InnoDB Cluster | PostgreSQL Patroni |
|--------|---------------------|-------------------|
| Consensus | Built-in (XCom/Paxos) | External (etcd, Consul, ZooKeeper) |
| Administration | MySQL Shell (integrated) | REST API + patronictl CLI |
| Failover detection | Group Replication quorum | Leader key in DCS expiry |
| Connection routing | MySQL Router (bundled) | PgBouncer + HAProxy (external) |
| Data synchronization | Group Replication (certification-based) | Streaming replication (WAL-based) |
| Split-brain protection | Quorum-based (majority required) | DCS leader election |
| Setup complexity | Lower (bundled solution) | Higher (multiple components) |
| Multi-primary support | Yes (with conflict detection) | No (single-primary only) |
| Monitoring | Performance Schema + Shell | Patroni REST API |

**Banking architecture recommendation:**

For Citi's GenAI platform, the choice depends on the database:
- **MySQL workloads:** InnoDB Cluster provides the simplest path to HA with bundled tooling. Single-primary mode is recommended for banking to avoid certification conflicts.
- **PostgreSQL workloads:** Patroni + etcd + PgBouncer is the industry standard. More complex to set up but more flexible and widely adopted in the PostgreSQL ecosystem.

**Key Points to Hit:**
- [ ] InnoDB Cluster = Group Replication + MySQL Shell + MySQL Router
- [ ] XCom (Paxos variant) provides consensus for automatic failover
- [ ] MySQL Router automatically detects primary changes and updates routing
- [ ] Single-primary mode recommended for banking (avoids write conflicts)
- [ ] Patroni uses external DCS (etcd); InnoDB Cluster has built-in consensus
- [ ] Both provide automated failover, but with different failure detection mechanisms
- [ ] Connection routing is bundled in MySQL (Router) but external in PostgreSQL

**Follow-Up Questions:**
1. A 3-node InnoDB Cluster loses 2 nodes simultaneously. What happens? Can the remaining node accept writes?
2. How does split-brain protection work in InnoDB Cluster vs Patroni?
3. What is the recovery process when a failed node rejoins the cluster?

**Source:** banking-genai-engineering-academy/databases/high-availability.md, databases/replication.md

---

### Q17: 🔴 How do you optimize MySQL for write-heavy banking workloads? Cover buffer pool, log files, flush behavior, and I/O patterns.

**Strong Answer:**

Write-heavy banking workloads (payment processing, trade settlement, real-time fraud detection) stress MySQL's write path differently than read-heavy workloads. The optimization focus shifts from cache hit ratios to write throughput, log management, and I/O scheduling.

**1. Redo Log Sizing:**

The redo log is the bottleneck for write throughput. When the redo log fills up, InnoDB must pause accepting writes and flush dirty pages to make room — this causes the dreaded "redo log stall."

```sql
-- Redo log configuration for write-heavy workloads
-- For a system processing 10K writes/sec with avg 1KB per write:
-- Peak write throughput: ~10MB/sec
-- Minimum redo log capacity: 10MB/sec * 60sec = 600MB (1 minute of writes)

SET GLOBAL innodb_log_file_size = 2 * 1024 * 1024 * 1024;  -- 2GB per file
SET GLOBAL innodb_log_files_in_group = 2;  -- Total capacity: 4GB
-- MySQL 8.0: Can be resized online
-- ALTER INSTANCE SET INNODB REDO_LOG CAPACITY 4G;

SET GLOBAL innodb_log_buffer_size = 64 * 1024 * 1024;  -- 64MB (default 16MB)
-- Larger buffer = fewer fsync calls to redo log

-- Monitor redo log usage
SHOW ENGINE INNODB STATUS\G
-- "Log sequence number" - "Last checkpoint at" = dirty data pending flush
-- If this difference approaches innodb_log_file_size * innodb_log_files_in_group,
-- you're at risk of a stall
```

**2. Dirty Page Flushing:**

InnoDB's page cleaner threads flush dirty pages from the buffer pool to disk. The flushing behavior is controlled by several parameters:

```sql
-- Control when flushing starts
SET GLOBAL innodb_max_dirty_pages_pct = 75;
-- Start flushing when 75% of buffer pool pages are dirty
-- Default: 90 (wait until 90% dirty). For write-heavy, lower to 70-75
-- to spread flushing more evenly and avoid I/O spikes

SET GLOBAL innodb_max_dirty_pages_pct_lwm = 10;
-- "Low water mark": pre-flush to keep dirty pages below this percentage
-- Default: 0 (disabled). Set to 10 for smoother I/O

SET GLOBAL innodb_io_capacity = 2000;
-- Tell InnoDB your disk's IOPS capacity
-- SSD: 2000-10000, NVMe: 10000-40000, HDD: 200
-- InnoDB uses this to throttle background I/O

SET GLOBAL innodb_io_capacity_max = 4000;
-- Maximum IOPS for urgent flushing
-- Should be 2x innodb_io_capacity

-- Monitor flushing
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_dirty';  -- Current dirty pages
SHOW STATUS LIKE 'Innodb_buffer_pool_bytes_dirty';   -- Bytes of dirty data
SHOW STATUS LIKE 'Innodb_data_fsyncs';               -- Fsync operations
```

**3. Flush Behavior:**

```sql
-- The durability knob
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
-- BANKING: MUST BE 1. No exceptions.
-- 2 = write to OS cache (risk on OS crash)
-- 0 = flush once/sec (risk on MySQL crash)

-- Alternative: Group commits for write batching
SET GLOBAL sync_binlog = 1;
-- Sync binary log to disk every commit (replication safety)
-- For extreme write workloads, can set to 0 or N (risk of losing N transactions)

-- Doublewrite buffer — keep enabled
SET GLOBAL innodb_doublewrite = 1;
-- MySQL 8.0.20+: innodb_doublewrite = 2 (batch mode, better performance)
-- NEVER set to 0 in banking
```

**4. I/O Pattern Optimization:**

```sql
-- Use O_DIRECT to bypass OS cache (prevents double-buffering)
-- InnoDB manages its own cache (buffer pool), OS cache is redundant
SET GLOBAL innodb_flush_method = 'O_DIRECT';
-- On Windows: this is the default behavior
-- On Linux: O_DIRECT or O_DIRECT_NO_FSYNC

-- Separate data and log files on different disks
-- Data files (*.ibd): Random I/O workload
-- Log files (ib_logfile*): Sequential I/O workload
-- Putting them on the same disk causes I/O contention

-- Use fast storage for redo log
-- NVMe for redo log files (sequential write performance is critical)
-- SSD for data files (random I/O performance)

-- Tablespaces: Use file-per-table for banking
SET GLOBAL innodb_file_per_table = 1;
-- Each table gets its own .ibd file
-- Benefits: per-table optimization, faster DROP TABLE,
--           individual table backup/restore, space reclamation
```

**5. Write-Specific Query Patterns:**

```sql
-- Batch inserts instead of single-row inserts
-- Bad: 1000 individual INSERT statements (1000 round trips, 1000 commits)
-- Good: Multi-row INSERT
INSERT INTO transactions (account_id, amount, status, txn_time)
VALUES
    (1, 100, 'COMPLETED', NOW()),
    (2, 200, 'COMPLETED', NOW()),
    ...  -- Batch of 100-1000 rows
    (1000, 50, 'COMPLETED', NOW());

-- Disable auto-commit for batch operations
SET autocommit = 0;
-- ... perform batch operations ...
COMMIT;

-- For bulk loads (end-of-day processing):
SET SESSION innodb_flush_log_at_trx_commit = 2;  -- TEMPORARILY
SET SESSION sync_binlog = 0;
-- ... load data ...
SET SESSION innodb_flush_log_at_trx_commit = 1;  -- Restore
SET SESSION sync_binlog = 1;

-- Drop and recreate secondary indexes before massive bulk loads
ALTER TABLE transactions DROP INDEX idx_txn_account;
-- ... load data ...
ALTER TABLE transactions ADD INDEX idx_txn_account(account_id);
-- Much faster than maintaining index during load
```

**Monitoring write health:**

```sql
-- Key metrics for write-heavy monitoring
SELECT
    (SELECT variable_value FROM performance_schema.global_status
     WHERE variable_name = 'Innodb_os_log_fsyncs') AS redo_log_fsyncs,
    (SELECT variable_value FROM performance_schema.global_status
     WHERE variable_name = 'Innodb_data_fsyncs') AS data_file_fsyncs,
    (SELECT variable_value FROM performance_schema.global_status
     WHERE variable_name = 'Innodb_buffer_pool_pages_dirty') AS dirty_pages,
    (SELECT variable_value FROM performance_schema.global_status
     WHERE variable_name = 'Innodb_dblwr_pages_written') AS doublewrite_pages;

-- Alerting thresholds:
-- Dirty pages > 75% of buffer pool → flush pressure increasing
-- Redo log fsyncs > 1000/sec → log file may be too small
-- Doublewrite pages growing rapidly → high write throughput
```

**Key Points to Hit:**
- [ ] Redo log must be sized for peak write throughput (60+ seconds of writes)
- [ ] `innodb_max_dirty_pages_pct` controls when flushing starts (lower = smoother I/O)
- [ ] `innodb_io_capacity` must match actual disk IOPS for proper throttling
- [ ] `innodb_flush_log_at_trx_commit = 1` is mandatory for banking durability
- [ ] Separate redo log and data files on different disks for I/O isolation
- [ ] Batch inserts with explicit transactions reduce write overhead
- [ ] O_DIRECT bypasses OS cache — InnoDB manages its own caching
- [ ] Doublewrite buffer must stay enabled — partial write protection

**Follow-Up Questions:**
1. Your redo log is filling up every 30 seconds causing write stalls. How do you size it correctly?
2. What is the impact of increasing `innodb_log_buffer_size` on memory usage?
3. How do you safely perform a bulk data load without risking data integrity?

**Source:** banking-genai-engineering-academy/databases/postgres-performance.md, databases/transactions.md, databases/postgres-fundamentals.md

---

### Q18: 🔴 Explain the MySQL optimizer's behavior with subqueries, derived tables, and materialization. How does this differ from PostgreSQL?

**Strong Answer:**

The MySQL optimizer has historically been the weakest part of MySQL compared to PostgreSQL, though MySQL 8.0 made significant improvements. Understanding these differences is critical when writing complex queries that must perform well on both platforms.

**Subquery Execution — The Historical Problem:**

Before MySQL 8.0, subqueries in the WHERE clause were often executed using a **dependent subquery** strategy — the subquery was re-executed for every row of the outer query, resulting in O(n*m) complexity. PostgreSQL would typically rewrite these as joins automatically.

```sql
-- Problematic pattern in MySQL 5.7:
SELECT * FROM accounts
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE segment = 'PREMIUM'
);
-- MySQL 5.7: Executes subquery for EACH row in accounts (DEPENDENT SUBQUERY)
-- PostgreSQL: Rewrites as a semi-join automatically

-- MySQL 8.0 improvement: Subquery materialization
-- The optimizer can now materialize (execute once, store in temp table)
-- and use the materialized result for lookups

EXPLAIN FORMAT=JSON
SELECT * FROM accounts
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE segment = 'PREMIUM'
)\G
-- MySQL 8.0: May show "materialized_from_subquery" strategy
-- MySQL 5.7: Shows "DEPENDENT SUBQUERY" — very bad
```

**Derived Tables:**

```sql
-- Derived table (subquery in FROM clause)
SELECT a.account_id, a.balance, sub.premium_txn_count
FROM accounts a
JOIN (
    SELECT account_id, COUNT(*) AS premium_txn_count
    FROM transactions
    WHERE status = 'COMPLETED'
    GROUP BY account_id
) sub ON a.account_id = sub.account_id
WHERE a.status = 'ACTIVE';

-- In MySQL 8.0:
-- The optimizer may materialize the derived table once, or
-- merge it into the outer query if possible (derived_merge optimization)

-- Force materialization (if optimizer chooses poorly):
SELECT /*+ NO_DERIVED_MERGE(sub) */ ...
-- Or disable globally:
SET optimizer_switch = 'derived_merge=off';

-- In PostgreSQL:
-- The planner almost always merges derived tables into the outer query,
-- treating them as equivalent to JOINs. Derived tables are rarely materialized
-- unless the planner determines it's cheaper (e.g., for CTEs with side effects).
```

**CTEs (Common Table Expressions):**

```sql
-- MySQL 8.0+ supports CTEs (PostgreSQL has had them since 8.4)
WITH premium_customers AS (
    SELECT customer_id FROM customers WHERE segment = 'PREMIUM'
),
active_accounts AS (
    SELECT account_id, customer_id, balance
    FROM accounts WHERE status = 'ACTIVE'
)
SELECT aa.*, pc.customer_id
FROM active_accounts aa
JOIN premium_customers pc ON aa.customer_id = pc.customer_id;

-- MySQL: CTE is materialized by default (execution materialized)
-- Can be overridden: WITH RECURSIVE cte AS (...) or WITH cte AS (/*+ MATERIALIZE */ ...)

-- PostgreSQL: CTE was always a "optimization fence" (prevented pushdown) until PG 12
-- Since PG 12: CTEs are inlined by default if referenced once, materialized if referenced multiple times
-- Can force: WITH cte AS MATERIALIZED (...) or WITH cte AS NOT MATERIALIZED (...)

-- Key difference: MySQL 8.0 CTE materialization is always eager by default.
-- PostgreSQL 12+ is smart about when to materialize vs. inline.
-- This matters for banking queries with large intermediate results.
```

**Optimizer Hints (MySQL 8.0+):**

```sql
-- MySQL 8.0 introduced optimizer hints
SELECT /*+ JOIN_ORDER(accounts, transactions)
           INDEX(transactions idx_txn_account_date)
           NO_ICP(transactions) */
    a.account_id, SUM(t.amount)
FROM accounts a
JOIN transactions t ON a.account_id = t.account_id
WHERE a.status = 'ACTIVE'
  AND t.txn_date >= '2025-01-01'
GROUP BY a.account_id;

-- Available hints:
-- INDEX(table index_name): Force specific index
-- NO_INDEX(table index_name): Prevent specific index
-- JOIN_ORDER(t1, t2, t3): Force join order
-- MERGE(t): Force derived table merge
-- NO_MERGE(t): Force derived table materialization
-- MATERIALIZE(t): Force subquery materialization
-- NO_MATERIALIZE(t): Prevent materialization
-- BKA(t): Use Block Nested Loop
-- NO_BKA(t): Prevent Block Nested Loop

-- PostgreSQL equivalent:
-- No direct optimizer hints (by design philosophy)
-- Use: SET enable_seqscan = off; (discouraged in production)
-- Or: Rewrite the query to guide the planner
-- Or: Use pg_hint_plan extension (third-party)
```

**Key optimizer differences:**

| Feature | MySQL 8.0 | PostgreSQL 15+ |
|--------|----------|----------------|
| Subquery optimization | Materialization + semi-join | Automatic subquery flattening |
| CTE default behavior | Materialized | Inlined (smart materialization) |
| Join order | Greedy + dynamic programming | Dynamic programming (GEQO for many tables) |
| Hash joins | Available since 8.0.20 | Available since 8.0 (2005) |
| Parallel query | Limited (8.0+) | Mature (9.6+) |
| Statistics | Single-value histograms, sampling | Multi-column stats, extended stats |
| Optimizer hints | Native (8.0+) | pg_hint_plan extension |

For banking queries involving complex aggregations, multi-table joins, and analytical processing, PostgreSQL's optimizer consistently produces better execution plans. MySQL requires more manual intervention (hints, query rewrites) to achieve equivalent performance.

**Key Points to Hit:**
- [ ] MySQL 5.7 had dependent subquery problem — fixed in 8.0 with materialization
- [ ] MySQL 8.0 CTEs are materialized by default; PG 12+ is smart about inlining
- [ ] Optimizer hints available in MySQL 8.0+; PostgreSQL discourages them
- [ ] PostgreSQL's dynamic programming join ordering is more sophisticated
- [ ] Hash joins in MySQL (8.0.20+) still less mature than PostgreSQL
- [ ] PostgreSQL parallel query is more advanced for analytical workloads
- [ ] MySQL often requires manual query rewriting for complex queries

**Follow-Up Questions:**
1. A 5-table join query runs 10x slower on MySQL than PostgreSQL. How do you debug the optimizer's decision?
2. When would you prefer a derived table over a CTE in MySQL 8.0?
3. How do extended statistics in PostgreSQL help the optimizer vs MySQL's single-column stats?

**Source:** banking-genai-engineering-academy/databases/postgres-performance.md, databases/query-tuning.md, databases/postgres-fundamentals.md

---

### Q19: 🔴 Design a MySQL schema for a high-volume transaction ledger. Cover normalization, JSON columns, partitioning, and indexing strategy.

**Strong Answer:**

Designing a schema for a banking transaction ledger requires balancing normalization (data integrity), denormalization (read performance), and operational concerns (partitioning, archiving). The decisions made at schema level determine the ceiling for query performance and the floor for data integrity.

**Core Schema — Normalized Design:**

```sql
-- Core tables: 3NF normalized for integrity
CREATE TABLE customers (
    customer_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_uuid CHAR(36) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    kyc_status ENUM('PENDING', 'VERIFIED', 'REJECTED', 'EXPIRED') NOT NULL DEFAULT 'PENDING',
    risk_rating ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL') NOT NULL DEFAULT 'LOW',
    customer_attributes JSON,  -- Flexible metadata (KYC documents, preferences)
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (customer_id),
    UNIQUE INDEX uk_customer_uuid (customer_uuid),
    UNIQUE INDEX uk_customer_email (email),
    INDEX idx_kyc_status (kyc_status)  -- For KYC processing queues
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE accounts (
    account_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    account_uuid CHAR(36) NOT NULL,
    customer_id BIGINT UNSIGNED NOT NULL,
    account_type ENUM('CHECKING', 'SAVINGS', 'CREDIT', 'INVESTMENT') NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    balance DECIMAL(18, 4) NOT NULL DEFAULT 0.0000,  -- 4 decimal places for precision
    available_balance DECIMAL(18, 4) NOT NULL DEFAULT 0.0000,
    status ENUM('ACTIVE', 'FROZEN', 'CLOSED', 'SUSPENDED') NOT NULL DEFAULT 'ACTIVE',
    opened_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    closed_at TIMESTAMP NULL,
    account_metadata JSON,  -- Account-specific settings, limits, features
    PRIMARY KEY (account_id),
    UNIQUE INDEX uk_account_uuid (account_uuid),
    INDEX idx_account_customer (customer_id, status),
    INDEX idx_account_type (account_type, status),
    CONSTRAINT fk_account_customer FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Transaction ledger: the critical table
CREATE TABLE transactions (
    txn_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    txn_uuid CHAR(36) NOT NULL,
    account_id BIGINT UNSIGNED NOT NULL,
    counterparty_account_id BIGINT UNSIGNED NULL,
    txn_type ENUM('DEPOSIT', 'WITHDRAWAL', 'TRANSFER', 'FEE', 'INTEREST', 'REVERSAL') NOT NULL,
    amount DECIMAL(18, 4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    balance_after DECIMAL(18, 4) NOT NULL,  -- Running balance for audit trail
    status ENUM('PENDING', 'COMPLETED', 'FAILED', 'REVERSED') NOT NULL DEFAULT 'PENDING',
    reference_number VARCHAR(50) NOT NULL,  -- External reference (wire, check, ACH)
    description VARCHAR(500),
    txn_details JSON,  -- Type-specific details (wire info, card metadata, etc.)
    txn_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL,
    created_by VARCHAR(100),  -- System or user who initiated
    audit_trail JSON,  -- Immutable audit log entries
    PRIMARY KEY (txn_id),
    UNIQUE INDEX uk_txn_uuid (txn_uuid),
    UNIQUE INDEX uk_reference_number (reference_number)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Partitioning by month for the transactions table
-- This is critical for a table that grows by millions of rows per month
ALTER TABLE transactions
PARTITION BY RANGE (UNIX_TIMESTAMP(txn_time)) (
    PARTITION p202501 VALUES LESS THAN (UNIX_TIMESTAMP('2025-02-01')),
    PARTITION p202502 VALUES LESS THAN (UNIX_TIMESTAMP('2025-03-01')),
    PARTITION p202503 VALUES LESS THAN (UNIX_TIMESTAMP('2025-04-01')),
    PARTITION p202504 VALUES LESS THAN (UNIX_TIMESTAMP('2025-05-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE  -- Catch-all for safety
);

-- Secondary indexes on partitioned table (must include partition key)
CREATE INDEX idx_txn_account_time ON transactions(account_id, txn_time);
CREATE INDEX idx_txn_status_time ON transactions(status, txn_time);
CREATE INDEX idx_txn_type_time ON transactions(txn_type, txn_time);
CREATE INDEX idx_txn_counterparty ON transactions(counterparty_account_id, txn_time);
-- Note: In MySQL, partitioned table indexes MUST include the partition key column
```

**Partitioning Strategy:**

Monthly partitioning serves multiple banking needs:
1. **Fast archival:** Drop old partitions instead of DELETE (instant, no undo log)
2. **Partition pruning:** Queries with `txn_time >= '2025-03-01'` only scan relevant partitions
3. **Independent maintenance:** Analyze/optimize one partition without affecting others
4. **Regulatory retention:** Keep required partitions, drop expired ones per compliance rules

```sql
-- Drop a partition (instant — no row-by-row delete)
ALTER TABLE transactions DROP PARTITION p202401;

-- Add future partitions (automate with a monthly cron job)
ALTER TABLE transactions REORGANIZE PARTITION p_future INTO (
    PARTITION p202505 VALUES LESS THAN (UNIX_TIMESTAMP('2025-06-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Verify partition pruning
EXPLAIN PARTITIONS
SELECT * FROM transactions
WHERE txn_time >= '2025-03-01' AND txn_time < '2025-04-01'\G
-- partitions: p202503 — only one partition scanned
```

**JSON Column Usage:**

MySQL's JSON type (stored as binary format, not text) is appropriate for:
- `customer_attributes`: KYC data, preferences, risk assessment results
- `account_metadata`: Account limits, features, product codes
- `txn_details`: Type-specific data (wire beneficiary, card terminal info, ACH trace)
- `audit_trail`: Immutable log of who changed what and when

```sql
-- JSON queries in MySQL 8.0
SELECT
    txn_uuid,
    txn_type,
    JSON_EXTRACT(txn_details, '$.wire.beneficiary_name') AS beneficiary,
    txn_details->>'$.ach.trace_number' AS trace_number  -- ->> unquotes
FROM transactions
WHERE txn_details->>'$.wire.swift_code' = 'CITIUS33';

-- Generated columns for JSON indexing
ALTER TABLE transactions
ADD COLUMN wire_beneficiary VARCHAR(200)
    GENERATED ALWAYS AS (txn_details->>'$.wire.beneficiary_name') VIRTUAL,
ADD INDEX idx_wire_beneficiary (wire_beneficiary);

-- This allows indexing JSON fields without full JSON indexes
```

**Denormalization for Read Performance:**

The `balance_after` column in the transactions table is a deliberate denormalization. While it could be computed by summing all previous transactions, storing it provides:
1. **Audit trail integrity:** The balance after each transaction is immutable and independently verifiable
2. **Fast balance queries:** No need to sum millions of historical transactions
3. **Reconciliation:** Compare running balance against account balance for integrity checks

**Key Points to Hit:**
- [ ] 3NF normalization for core entities (customers, accounts, transactions)
- [ ] JSON columns for flexible, type-specific metadata with generated column indexes
- [ ] Monthly partitioning on transactions table for archival and partition pruning
- [ ] `balance_after` as denormalized column for audit trail and fast balance queries
- [ ] DECIMAL(18,4) for currency — 4 decimal places, not float/double
- [ ] UUID columns alongside auto_increment for external reference and security
- [ ] Partitioned table indexes must include partition key in MySQL
- [ ] Running balance serves as independent verification of account integrity

**Follow-Up Questions:**
1. The transactions table has 2 billion rows. Partition pruning isn't helping a query. What's wrong?
2. How do you ensure the JSON audit_trail column is truly immutable (append-only)?
3. Should you use a separate table for transaction line items (ledger entries) vs a single-row-per-transaction model?

**Source:** banking-genai-engineering-academy/databases/partitioning.md, databases/jsonb.md, databases/indexing.md, databases/banking-db-patterns.md

---

### Q20: 🔴 You need to perform a zero-downtime major version upgrade (MySQL 5.7 → 8.0) for a 10TB banking database. Walk through the strategy.

**Strong Answer:**

A zero-downtime major version upgrade of a 10TB banking database is one of the most complex operations a database engineer can perform. It requires careful planning, extensive testing, and a rollback strategy. The recommended approach uses **logical replication** for the cutover, not in-place upgrade.

**Why logical replication, not in-place?**

An in-place upgrade (`mysqld --upgrade`) requires stopping the server, replacing binaries, and restarting. For a 10TB database, this means hours of downtime. Logical replication allows the upgrade to happen while the old server continues serving traffic, with a sub-second cutover.

**Phase 1: Preparation (Weeks 1-2)**

```sql
-- 1. Audit incompatible changes
-- MySQL 5.7 → 8.0 breaking changes:
-- - Reserved words: GROUPS, CUBE, ROWS, EXCEPT (check column/table names)
-- - Authentication: mysql_native_password → caching_sha2_password (default)
-- - utf8 → utf8mb4 (utf8 is alias for utf8mb3, deprecated)
-- - MyISAM system tables → InnoDB (automatic)
-- - Query cache removed entirely
-- - GROUP BY implicit ordering removed

-- Check for reserved word conflicts
SELECT table_name, column_name
FROM information_schema.columns
WHERE column_name IN ('groups', 'cube', 'rows', 'except')
  AND table_schema = 'banking_db';

-- Check authentication plugins
SELECT user, host, plugin FROM mysql.user;
-- Update mysql_native_password users to caching_sha2_password:
ALTER USER 'app_user'@'%' IDENTIFIED WITH caching_sha2_password BY 'new_password';

-- Check for utf8 (not utf8mb4) columns
SELECT table_name, column_name, character_set_name
FROM information_schema.columns
WHERE character_set_name = 'utf8' AND table_schema = 'banking_db';
-- Convert: ALTER TABLE t MODIFY col VARCHAR(255) CHARACTER SET utf8mb4;

-- Check for MyISAM tables (should be none in banking)
SELECT table_name, engine FROM information_schema.tables
WHERE engine = 'MyISAM' AND table_schema = 'banking_db';

-- Take full backup before starting
mysqldump --single-transaction --routines --triggers --events \
  --all-databases > pre_upgrade_backup.sql
```

**Phase 2: Build and Test (Weeks 3-6)**

1. **Provision MySQL 8.0 servers** with identical hardware and configuration (adjusted for 8.0-specific settings).
2. **Set up logical replication** from MySQL 5.7 (source) to MySQL 8.0 (target).

```sql
-- On MySQL 5.7 (source):
-- Note: MySQL 5.7 does NOT support native logical replication
-- Use one of these approaches:

-- Option A: MySQL Enterprise Edition (logical replication 5.7→8.0)
-- Requires MySQL Enterprise Subscription

-- Option B: pt-osc + binary log replay
-- Use Percona Toolkit to sync, then replay binlog

-- Option C: Third-party tools
-- gh-ost, Debezium CDC → Kafka → MySQL 8.0 consumer
-- Tungsten Replicator
-- AWS DMS (if on RDS)

-- Option D: Application-level dual-write
-- Deploy application that writes to BOTH 5.7 and 8.0
-- Read from 5.7, validate against 8.0
-- After validation, switch reads to 8.0

-- If both servers are 8.0 (for minor upgrades), native logical replication:
-- Source (8.0):
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'secure';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl_user'@'%';
CREATE PUBLICATION banking_pub FOR ALL TABLES;

-- Target (8.0):
CREATE SUBSCRIPTION banking_sub
    CONNECTION 'host=5.7-server user=repl_user password=secure dbname=banking_db'
    PUBLICATION banking_pub;
```

**Phase 3: Validation (Weeks 7-8)**

```sql
-- Row count validation
SELECT
    table_name,
    table_rows AS source_rows
FROM information_schema.tables
WHERE table_schema = 'banking_db'
ORDER BY table_rows DESC;

-- Compare with target (must match)

-- Checksum validation (for critical tables)
-- MySQL doesn't have built-in table checksums
-- Use:
-- pt-table-checksum (Percona Toolkit)
-- Or application-level reconciliation

-- Query performance comparison
-- Run identical query sets on both servers
-- Compare execution plans (EXPLAIN FORMAT=JSON)
-- Compare response times

-- Application testing
-- Point staging environment to MySQL 8.0
-- Run full regression test suite
-- Verify: query results, error handling, connection pooling
```

**Phase 4: Cutover (Maintenance Window — Target: <30 seconds)**

```
1. STOP writes to MySQL 5.7 (application maintenance mode)
2. Wait for replication to catch up (Seconds_Behind_Source = 0)
3. Verify final checksums match
4. Update DNS/connection strings to point to MySQL 8.0
5. START writes to MySQL 8.0
6. Verify application health checks pass
7. Keep MySQL 5.7 running as standby for 72 hours (rollback option)
```

**Rollback Strategy:**

If MySQL 8.0 exhibits issues after cutover:
1. Stop writes to MySQL 8.0
2. Set up reverse replication (8.0 → 5.7) during the validation phase
3. Point application back to MySQL 5.7
4. The downtime for rollback is the same as the cutover: <30 seconds

**Post-Cutover (Weeks 9-10)**

```sql
-- Enable MySQL 8.0 specific features
-- CTEs, window functions, invisible indexes, descending indexes, etc.

-- Update optimizer settings
SET PERSIST optimizer_switch = 'derived_merge=on';  -- MySQL 8.0 default

-- Review and update statistics
ANALYZE TABLE customers, accounts, transactions;

-- Monitor for 30 days
-- Watch for: query plan changes, new errors, performance regressions
-- MySQL 8.0's optimizer may choose different plans than 5.7
```

**Critical lessons from enterprise upgrades:**

1. **Test on production-scale data.** A 1GB test database will not reveal the same issues as 10TB. Create a production-like test environment with anonymized data.
2. **Plan for rollback from day one.** If you can't rollback, you're not doing zero-downtime — you're gambling.
3. **Application compatibility is harder than database compatibility.** The database upgrade is straightforward; the application changes to use new features (or adapt to removed features) are where most failures occur.
4. **Communicate the maintenance window.** Even with a <30 second cutover, banking applications may have dependent systems that need coordination.

**Key Points to Hit:**
- [ ] Logical replication (not in-place) enables near-zero downtime upgrade
- [ ] MySQL 5.7 → 8.0 has breaking changes: auth plugin, reserved words, query cache removal
- [ ] MySQL 5.7 lacks native logical replication — requires workarounds (Debezium, DMS, dual-write)
- [ ] Validation must include row counts, checksums, query plan comparison, and app testing
- [ ] Cutover: stop writes → sync → verify → switch DNS → start writes
- [ ] Reverse replication for rollback capability
- [ ] Test on production-scale data with anonymized copy
- [ ] Application compatibility is the primary risk, not database compatibility

**Follow-Up Questions:**
1. During the upgrade, you discover a query that works in 5.7 but fails in 8.0. How do you handle?
2. The cutover takes 5 minutes instead of 30 seconds. What's the impact on banking SLAs?
3. How would the strategy differ for an in-place upgrade vs blue-green deployment?

**Source:** banking-genai-engineering-academy/databases/migrations.md, databases/high-availability.md, databases/replication.md

---

## Summary Matrix

| # | Question | Difficulty | Key Topics |
|---|----------|-----------|------------|
| Q1 | Client-server architecture | 🔵 | Architecture, connection flow, server vs storage layers |
| Q2 | InnoDB vs MyISAM | 🔵 | Storage engines, transactions, locking |
| Q3 | Buffer pool | 🔵 | Memory, caching, performance tuning |
| Q4 | Transactions and COMMIT/ROLLBACK | 🔵 | Undo log, redo log, ACID, durability |
| Q5 | B-tree indexes vs PostgreSQL | 🔵 | Clustered index, secondary index, bookmark lookup |
| Q6 | MVCC and undo logs | 🟡 | Version chains, read views, purge thread |
| Q7 | Composite index ordering | 🟡 | Leftmost prefix, selectivity, cardinality |
| Q8 | Locking mechanisms | 🟡 | S/X locks, intention locks, next-key locks |
| Q9 | Query optimization with EXPLAIN | 🟡 | Slow query log, EXPLAIN, covering indexes |
| Q10 | Replication types | 🟡 | Async, semi-sync, Group Replication |
| Q11 | GTID-based replication | 🟡 | Global transaction IDs, automated failover |
| Q12 | Slow query log in production | 🟡 | Performance Schema, pt-query-digest, alerting |
| Q13 | MySQL vs PostgreSQL | 🟡 | Architecture, features, migration |
| Q14 | Deadlock prevention | 🟡 | Wait-for graph, lock ordering, retry logic |
| Q15 | Crash recovery | 🟡 | Redo log, doublewrite buffer, recovery phases |
| Q16 | InnoDB Cluster vs Patroni | 🔴 | HA, automated failover, consensus |
| Q17 | Write-heavy optimization | 🔴 | Redo log sizing, flush behavior, I/O patterns |
| Q18 | Optimizer behavior vs PostgreSQL | 🔴 | Subqueries, CTEs, materialization, hints |
| Q19 | Ledger schema design | 🔴 | Normalization, JSON, partitioning, indexing |
| Q20 | Zero-downtime 5.7 → 8.0 upgrade | 🔴 | Logical replication, cutover, rollback |
