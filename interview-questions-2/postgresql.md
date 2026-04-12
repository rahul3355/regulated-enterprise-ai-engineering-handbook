# PostgreSQL Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | PostgreSQL (Indexing, Transactions, Isolation, JSONB, pgvector, RLS, Partitioning, Replication, Query Tuning, Migrations) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | databases/ (key files: postgres-fundamentals, postgres-performance, pgvector, indexing, query-tuning, transactions, isolation-levels, deadlocks, jsonb, partitioning, row-level-security, banking-db-patterns, replication, migrations) |
| Citi Relevance | Postgres is Citi's primary database. pgvector is used for RAG embeddings. Row-Level Security enables multi-tenant isolation. Audit and compliance requirements are critical. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

### Q1: Explain how MVCC works in PostgreSQL and why it is critical for concurrency in a banking system.

**Strong Answer:**

MVCC (Multi-Version Concurrency Control) is PostgreSQL's mechanism for allowing multiple transactions to access the database concurrently without blocking each other. Instead of locking rows for reads, PostgreSQL maintains multiple versions of each tuple (row) in the heap.

Each tuple carries metadata in its header: `t_xmin` (the transaction ID that inserted it), `t_xmax` (the transaction ID that deleted or updated it, or 0 if still live), `t_cid` (command ID within the transaction), and `t_ctid` (physical location). When a transaction reads a row, PostgreSQL checks these fields against the transaction's snapshot to determine which version is visible.

Here is how it plays out in practice:

```sql
-- Time 1: Tx100 INSERTS row
-- Tuple v1: xmin=100, xmax=0

-- Time 2: Tx300 UPDATEs the row (balance=1500)
-- Tuple v1: xmin=100, xmax=300 (marked dead)
-- Tuple v2: xmin=300, xmax=0 (new version)

-- Tx200 that started before Tx300 still sees v1.
-- Tx400 that starts after Tx300 sees v2.
```

The critical benefit for banking is that **readers never block writers and writers never block readers**. In a high-throughput trading system, this means balance inquiries, regulatory reports, and analytics queries can run concurrently with deposit/withdrawal operations without contention.

However, MVCC has a cost: old tuple versions accumulate as "dead tuples." VACUUM cleans these up, marking space as reusable (but not returning it to the OS). Long-running transactions prevent VACUUM from cleaning up any tuples that the transaction might still need to see, causing table bloat. In banking, this means a 2-hour analytics query can block cleanup on millions of rows. The mitigation is to set `statement_timeout`, use dedicated read replicas for analytics, and tune autovacuum aggressively on high-churn tables.

**Key Points to Hit:**
- [ ] Each tuple has xmin/xmax metadata for version tracking
- [ ] Readers never block writers; writers never block readers
- [ ] Snapshot isolation determines which version a transaction sees
- [ ] VACUUM cleans dead tuples but does not return space to OS
- [ ] Long-running transactions prevent VACUUM, causing bloat

**Follow-Up Questions:**
1. How do you detect and resolve table bloat caused by long-running transactions?
2. What happens to dead tuples if autovacuum is disabled?
3. How does MVCC differ from traditional locking in MySQL/InnoDB?

**Source:** `databases/postgres-fundamentals.md`, `databases/isolation-levels.md`

---

### Q2: What are the ACID properties and how does PostgreSQL implement each one?

**Strong Answer:**

ACID stands for Atomicity, Consistency, Isolation, and Durability. These four properties guarantee reliable transaction processing, which is non-negotiable in banking systems where financial integrity depends on them.

**Atomicity** ensures a transaction is all-or-nothing. PostgreSQL implements this via Write-Ahead Logging (WAL). Before any data page is modified on disk, the change is first written to the WAL. If a crash occurs mid-transaction, recovery replays the WAL to redo committed changes and undo uncommitted ones. A `COMMIT` records the commit WAL record and flushes it to disk; a `ROLLBACK` simply discards the changes since they were never made visible (MVCC handles the visibility).

**Consistency** means the database moves from one valid state to another. PostgreSQL enforces structural consistency through constraints (PRIMARY KEY, FOREIGN KEY, CHECK, UNIQUE, NOT NULL) and rules. Application-level business rules (e.g., "balance cannot go negative") must be enforced by the application or via check constraints and triggers. In banking, double-entry bookkeeping is a consistency requirement — debits must always equal credits — typically enforced via stored procedures or application logic.

**Isolation** ensures concurrent transactions do not interfere with each other. PostgreSQL implements this through MVCC and supports three isolation levels: Read Committed (default), Repeatable Read, and Serializable. Each level provides different guarantees about what concurrent modifications are visible.

**Durability** guarantees that committed data survives crashes. Once a `COMMIT` returns success, the WAL record is on disk. PostgreSQL's `synchronous_commit` parameter controls the durability guarantee — when `on` (default), the server waits for WAL flush to disk before acknowledging the commit. Setting it to `off` improves write throughput but risks losing the last few transactions on crash (acceptable for bulk loads, never for financial transactions).

```sql
-- Durability check: synchronous_commit ensures WAL is on disk
SHOW synchronous_commit;  -- Should be 'on' for banking transactions

-- Atomicity: savepoints allow partial rollback within a transaction
BEGIN;
SAVEPOINT before_transfer;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- If something goes wrong:
ROLLBACK TO SAVEPOINT before_transfer;
COMMIT;
```

**Key Points to Hit:**
- [ ] Atomicity via WAL — all changes commit or none do
- [ ] Consistency via constraints, checks, and application logic
- [ ] Isolation via MVCC with three isolation levels
- [ ] Durability via WAL flush; synchronous_commit controls guarantee
- [ ] Banking requires all four — especially durability for financial transactions

**Follow-Up Questions:**
1. What is the impact of setting `synchronous_commit = off` on financial data?
2. How does WAL ensure both atomicity and durability?
3. Can you explain how savepoints provide partial atomicity within a transaction?

**Source:** `databases/transactions.md`, `databases/postgres-fundamentals.md`

---

### Q3: Describe the difference between B-Tree, GIN, GiST, BRIN, and HNSW indexes. When would you use each in a banking context?

**Strong Answer:**

PostgreSQL offers multiple index types, each optimized for different access patterns. Choosing the right one is critical for performance.

**B-Tree** is the default and most versatile index. It supports equality (`=`), range (`<`, `>`, `BETWEEN`), and prefix `LIKE` queries. In banking, B-Tree is used for foreign keys, account lookups, and date-range queries on transaction tables. Composite B-Tree indexes should order columns with equality columns first, then range columns:

```sql
-- Equality column first, range column last
CREATE INDEX idx_txn_account_time ON transactions (account_id, transaction_time DESC);
```

**GIN (Generalized Inverted Index)** indexes composite values like JSONB, arrays, and full-text search. In banking, GIN is essential for querying JSONB metadata (e.g., customer profiles, GenAI document metadata) and array containment queries. GIN indexes are larger and slower to write but enable powerful containment queries:

```sql
CREATE INDEX idx_doc_metadata ON banking_documents USING gin (metadata);
SELECT * FROM banking_documents WHERE metadata @> '{"doc_type": "product_info"}';
```

**GiST (Generalized Search Tree)** supports geometric data, range overlap queries, and trigram-based fuzzy text matching. In banking, GiST could be used for IP range lookups in access logs or fuzzy customer name matching for KYC processes:

```sql
-- Fuzzy name matching for KYC
CREATE INDEX idx_customer_name_trgm ON customers USING gin (last_name gin_trgm_ops);
SELECT * FROM customers WHERE last_name % 'Smiith' ORDER BY last_name <-> 'Smiith' LIMIT 10;
```

**BRIN (Block Range Index)** is a tiny index for naturally ordered, append-only data. It stores min/max values per block range (default 128 pages). For a 500M-row transaction table, a B-Tree might be 11 GB while a BRIN is only 5 MB. BRIN works when data is physically ordered by the indexed column (e.g., `transaction_time`), making it ideal for time-series banking data.

```sql
-- Tiny index for append-only time-series data
CREATE INDEX idx_txn_time_brin ON transactions USING brin (transaction_time);
```

**HNSW (Hierarchical Navigable Small World)** is used for vector similarity search via the pgvector extension. It builds a multi-layer graph for approximate nearest neighbor search on embeddings. In banking GenAI/RAG systems, HNSW indexes power semantic search over policy documents, compliance manuals, and customer-facing content:

```sql
CREATE INDEX idx_embeddings_hnsw ON document_chunks
USING hnsw (embedding vector_cosine_ops) WITH (m = 24, ef_construction = 128);
```

| Index | Best For | Banking Use Case | Size |
|-------|----------|-----------------|------|
| B-Tree | Equality, range | account_id lookups, date ranges | Medium |
| GIN | JSONB, arrays, full-text | JSONB metadata, tags | Large |
| GiST | Geometric, range overlap, trigram | IP ranges, fuzzy KYC matching | Medium |
| BRIN | Ordered, append-only | Time-series transactions | Tiny |
| HNSW | Vector similarity | RAG embedding retrieval | Large |

**Key Points to Hit:**
- [ ] B-Tree is default — use for equality and range queries
- [ ] GIN for JSONB, arrays, full-text search
- [ ] BRIN for naturally ordered append-only time-series (tiny footprint)
- [ ] HNSW for vector similarity (pgvector/RAG)
- [ ] GiST for geometric, range overlap, and trigram matching

**Follow-Up Questions:**
1. How do you decide between a composite B-Tree index and separate single-column indexes?
2. A GIN index on JSONB has grown to 10 GB — is this normal and how do you optimize it?
3. When would BRIN be ineffective even on time-ordered data?

**Source:** `databases/indexing.md`, `databases/pgvector.md`

---

### Q4: What is Write-Ahead Logging (WAL) and how does it ensure durability and crash recovery?

**Strong Answer:**

Write-Ahead Logging (WAL) is PostgreSQL's fundamental mechanism for ensuring durability and enabling crash recovery. The principle is simple: **before any change is written to the data files (heap), it must first be recorded in the WAL.**

The write flow works as follows:
1. A backend process modifies a page in the shared buffer pool (in memory).
2. A WAL record describing the change is written to the WAL buffer.
3. On `COMMIT`, the WAL buffer is flushed to disk (if `synchronous_commit = on`).
4. Dirty data pages are written to disk later by the background writer or checkpointer — this is asynchronous.

The recovery flow after a crash:
1. PostgreSQL reads the last checkpoint from disk.
2. It replays all WAL records from that checkpoint forward.
3. All committed transactions are recovered (their changes are reapplied).
4. Uncommitted transactions are rolled back (their changes are discarded).

This means PostgreSQL never loses committed data, even if the power goes out mid-transaction. The WAL files live in `$PGDATA/pg_wal/` and are managed automatically.

```sql
-- WAL configuration for banking (durability-critical)
-- wal_level = replica       -- Enables streaming replication
-- synchronous_commit = on   -- Default; ensures WAL flush before commit ack
-- max_wal_size = 1GB        -- Max WAL before forcing checkpoint
-- min_wal_size = 80MB       -- Minimum WAL to retain

-- Check current WAL position
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
```

WAL also enables streaming replication (standby servers replay WAL from primary) and Point-in-Time Recovery (PITR) via WAL archiving. In banking, PITR is essential for regulatory compliance — you must be able to reconstruct the database state at any historical moment for audit purposes.

Setting `synchronous_commit = off` trades durability for performance: the server acknowledges commits before WAL is on disk, risking loss of the last few transactions on crash. This is acceptable for bulk ETL loads but **never** for financial transaction processing.

**Key Points to Hit:**
- [ ] WAL records written before data pages modified
- [ ] Crash recovery replays WAL from last checkpoint
- [ ] synchronous_commit = on ensures WAL on disk before commit ack
- [ ] WAL enables streaming replication and PITR
- [ ] Banking requires PITR for regulatory audit trails

**Follow-Up Questions:**
1. What happens if the WAL directory fills up on disk?
2. How does `wal_level = logical` differ from `wal_level = replica`?
3. Can you explain the relationship between checkpoints and WAL?

**Source:** `databases/postgres-fundamentals.md`, `databases/replication.md`

---

### Q5: What is the default isolation level in PostgreSQL, and why was it chosen?

**Strong Answer:**

The default isolation level in PostgreSQL is **Read Committed**. This was chosen as the default because it provides the best balance between data consistency and concurrency for general OLTP workloads.

Under Read Committed, each statement within a transaction sees a snapshot of the database as of the **start of that statement** (not the start of the transaction). This means two SELECT statements within the same transaction can return different results if another transaction commits changes between them.

```sql
-- Read Committed behavior:
-- Transaction A                          Transaction B
BEGIN;                                    BEGIN;
SELECT balance FROM accounts              -- Sees balance = 1000
WHERE account_id = 1;
                                        UPDATE accounts
                                        SET balance = 1500
                                        WHERE account_id = 1;
                                        COMMIT;

SELECT balance FROM accounts              -- Sees balance = 1500 (new snapshot!)
WHERE account_id = 1;
COMMIT;
```

For banking, this has important implications. A read-modify-write pattern like "read balance, check if sufficient, then deduct" is unsafe under Read Committed because the balance could change between the read and the write. The solution is to use explicit row locking:

```sql
BEGIN;
SELECT balance FROM accounts
WHERE account_id = 1
FOR UPDATE;  -- Exclusive lock on the row

-- Now no other transaction can modify this row until we commit
UPDATE accounts SET balance = balance - 800 WHERE account_id = 1;
COMMIT;
```

PostgreSQL chose Read Committed as the default because:
1. **Best concurrency**: No serialization failures or retry logic needed.
2. **Matches developer expectations**: Most developers expect to see the latest committed data.
3. **Lower contention**: No need to retry transactions due to serialization conflicts.

For critical financial operations where stale reads are unacceptable, you would use Repeatable Read (consistent snapshot for the entire transaction) or Serializable (strictest guarantee, with automatic conflict detection and aborts).

**Key Points to Hit:**
- [ ] Default is Read Committed
- [ ] Each statement gets its own snapshot (not transaction-start snapshot)
- [ ] Chosen for best concurrency and lowest contention
- [ ] Read-modify-write patterns need explicit FOR UPDATE locking
- [ ] Banking should use higher isolation for critical operations

**Follow-Up Questions:**
1. When would you use Serializable isolation in a banking application?
2. What is the difference between PostgreSQL's Repeatable Read and the SQL standard?
3. How does FOR UPDATE NOWAIT differ from regular FOR UPDATE?

**Source:** `databases/isolation-levels.md`, `databases/transactions.md`

---

### Q6: Your application is experiencing deadlocks during concurrent account transfers. How do you diagnose, prevent, and resolve them?

**Strong Answer:**

A deadlock occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency. PostgreSQL automatically detects deadlocks (controlled by `deadlock_timeout`, default 1 second) and resolves them by aborting one transaction with an error.

**Diagnosis:**

Deadlocks are logged to the PostgreSQL server log with details about the processes and locks involved. You can also monitor lock waits in real-time:

```sql
-- Monitor current lock waits
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.query AS blocked_statement,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.pid != blocked_locks.pid
    AND NOT blocked_locks.granted
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid;
```

**Prevention — Consistent Lock Ordering:**

The classic banking deadlock scenario: Tx1 locks Account A then tries to lock Account B, while Tx2 locks Account B then tries to lock Account A. The fix is to always lock rows in a deterministic order:

```sql
CREATE OR REPLACE FUNCTION safe_transfer(
    from_account BIGINT, to_account BIGINT, amount DECIMAL
) RETURNS void AS $$
DECLARE
    v_first BIGINT;
    v_second BIGINT;
BEGIN
    -- Always lock lower account_id first
    IF from_account < to_account THEN
        v_first := from_account;
        v_second := to_account;
    ELSE
        v_first := to_account;
        v_second := from_account;
    END IF;

    -- Lock in order — no circular wait possible
    PERFORM 1 FROM accounts WHERE account_id = v_first FOR UPDATE;
    PERFORM 1 FROM accounts WHERE account_id = v_second FOR UPDATE;

    UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
    INSERT INTO transfer_log (from_account, to_account, amount, created_at)
    VALUES (from_account, to_account, amount, NOW());
END;
$$ LANGUAGE plpgsql;
```

**Additional Strategies:**

1. **NOWAIT**: Fail immediately if a row is locked instead of waiting:
   ```sql
   SELECT * FROM accounts WHERE account_id = 1 FOR UPDATE NOWAIT;
   ```

2. **SKIP LOCKED**: Skip locked rows, useful for queue processing:
   ```sql
   SELECT * FROM pending_payments ORDER BY created_at LIMIT 10 FOR UPDATE SKIP LOCKED;
   ```

3. **Retry logic**: Implement exponential backoff retry for deadlock errors at the application level:
   ```python
   @retry_on_deadlock(max_retries=3, base_delay=0.1)
   def execute_transfer(conn, from_acct, to_acct, amount):
       # ... transfer logic
   ```

4. **Keep transactions short**: The shorter a transaction holds locks, the less chance of deadlock.

5. **Reduce `deadlock_timeout`**: Setting it to 500ms speeds up detection:
   ```sql
   ALTER SYSTEM SET deadlock_timeout = '500ms';
   SELECT pg_reload_conf();
   ```

**Key Points to Hit:**
- [ ] Deadlock = circular lock dependency; PostgreSQL auto-detects and aborts one
- [ ] Consistent lock ordering (always lock by account_id ascending) prevents deadlocks
- [ ] NOWAIT and SKIP LOCKED are alternatives for specific use cases
- [ ] Application-level retry with exponential backoff handles residual deadlocks
- [ ] Keep transactions short to minimize lock duration

**Follow-Up Questions:**
1. How does `deadlock_timeout` affect detection performance?
2. Can SKIP LOCKED be used for the transfer problem? Why or why not?
3. What monitoring would you set up to track deadlock frequency in production?

**Source:** `databases/deadlocks.md`, `databases/isolation-levels.md`

---

### Q7: How do you read and interpret an `EXPLAIN ANALYZE` output? Walk through identifying and fixing a slow query.

**Strong Answer:**

`EXPLAIN ANALYZE` is the primary tool for understanding how PostgreSQL executes a query. It shows both the planner's estimated cost and the actual runtime statistics.

```
Index Scan using idx_txn_account_time on transactions
  (cost=0.56..234.56 rows=1234 width=128)
  (actual time=0.045..2.345 rows=1234 loops=1)
  Index Cond: (account_id = 12345 AND transaction_time >= '2025-01-01')
  Buffers: shared hit=456 read=12
Planning Time: 0.234 ms
Execution Time: 2.567 ms
```

Key metrics:
- **cost=0.56..234.56**: Startup cost (0.56) and total cost (234.56) in arbitrary planner units. Lower is better, but only comparable between alternative plans for the same query.
- **rows=1234**: Planner's estimate vs actual rows returned. Large gaps indicate stale statistics (run `ANALYZE`).
- **actual time=0.045..2.345**: Startup and total time in milliseconds.
- **Buffers: shared hit=456 read=12**: Pages found in cache (hit) vs read from disk (read). High "read" means cold cache or insufficient `shared_buffers`.
- **loops=1**: How many times this node executed. Multiply rows x loops for total rows processed.

**Common problems and fixes:**

**Problem 1: Sequential Scan on large table**
```
Seq Scan on transactions (actual time=1.234..567.890 rows=1234 loops=1)
  Filter: (account_id = 12345)
  Rows Removed by Filter: 49998766
```
Fix: Create an index on the filter column.

**Problem 2: Hash join spilling to disk**
```
Hash  (actual time=12.345..12.345 rows=50000 loops=1)
  Batches: 4  Memory Usage: 25600kB  Disk: 204800kB
```
Fix: Increase `work_mem` for this session so the hash fits in memory:
```sql
SET work_mem = '256MB';
```

**Problem 3: Sort spilling to disk**
```
Sort Method: external merge  Disk: 12345kB
```
Fix: Either increase `work_mem` or add an index on the ORDER BY column to avoid sorting entirely.

**Banking example optimization:**
```sql
-- Original: 45 seconds (Seq Scan on 500M row transactions table)
EXPLAIN ANALYZE
SELECT a.account_number, c.customer_name,
       SUM(t.amount) AS monthly_total, COUNT(*) AS txn_count
FROM transactions t
JOIN accounts a ON t.account_id = a.account_id
JOIN customers c ON a.customer_id = c.customer_id
WHERE t.transaction_time >= '2025-01-01' AND t.transaction_time < '2025-02-01'
  AND a.status = 'ACTIVE'
GROUP BY a.account_number, c.customer_name
ORDER BY monthly_total DESC;

-- Fixes:
-- 1. Covering index to avoid heap fetches
CREATE INDEX idx_txn_time_covering ON transactions (transaction_time, account_id)
    INCLUDE (amount);

-- 2. Index on accounts status for early filtering
CREATE INDEX idx_accounts_status ON accounts (status, account_id)
    INCLUDE (account_number, customer_id);

-- 3. CTE to materialize filtered accounts first
WITH active_accounts AS MATERIALIZED (
    SELECT account_id, account_number, customer_id
    FROM accounts WHERE status = 'ACTIVE'
)
SELECT aa.account_number, c.customer_name,
       SUM(t.amount), COUNT(*)
FROM active_accounts aa
JOIN transactions t ON aa.account_id = t.account_id
    AND t.transaction_time >= '2025-01-01' AND t.transaction_time < '2025-02-01'
JOIN customers c ON aa.customer_id = c.customer_id
GROUP BY aa.account_number, c.customer_name
ORDER BY monthly_total DESC;

-- Result: 0.5 seconds (90x improvement)
```

**Key Points to Hit:**
- [ ] Read cost (planner estimate) vs actual time (runtime)
- [ ] Compare estimated rows vs actual rows — gaps mean stale stats
- [ ] Buffers hit vs read reveals cache effectiveness
- [ ] Seq Scan on large table = missing index
- [ ] Hash/Sort spilling to disk = increase work_mem or add index

**Follow-Up Questions:**
1. How do you use `pg_stat_statements` to find slow queries across your entire application?
2. When does the planner choose Nested Loop vs Hash Join vs Merge Join?
3. What does `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` give you that plain `EXPLAIN` does not?

**Source:** `databases/query-tuning.md`, `databases/postgres-performance.md`

---

### Q8: How do you use JSONB in PostgreSQL for semi-structured data in a banking context? What are the trade-offs vs relational columns?

**Strong Answer:**

JSONB stores binary-decomposed JSON data, enabling document-style storage within PostgreSQL while retaining ACID guarantees. Unlike JSON (which stores raw text), JSONB supports indexing, querying, and manipulation, but discards duplicate keys and whitespace.

In banking, JSONB is ideal for:

1. **Customer preferences** — frequently changing, user-specific, no fixed schema:
   ```sql
   ALTER TABLE customers ADD COLUMN preferences JSONB DEFAULT '{}';
   ```

2. **Product configurations** — varying fields by product type (savings, loans, credit cards):
   ```sql
   CREATE TABLE banking_products (
       product_id BIGINT PRIMARY KEY,
       product_type VARCHAR(50),
       base_config JSONB,
       terms JSONB,
       fees JSONB
   );
   ```

3. **Event payloads** — audit trails, CDC events, webhook payloads:
   ```sql
   CREATE TABLE customer_events (
       event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       customer_id BIGINT,
       event_type VARCHAR(50),
       payload JSONB,
       created_at TIMESTAMPTZ DEFAULT NOW()
   );
   ```

4. **GenAI document metadata** — flexible metadata for RAG filtering:
   ```sql
   CREATE TABLE genai_documents (
       document_id UUID PRIMARY KEY,
       content TEXT,
       metadata JSONB,
       embedding vector(1536)
   );
   ```

**Querying JSONB:**
```sql
-- Extract value (-> returns JSONB, ->> returns text)
SELECT profile->'risk_profile'->>'category' AS risk_category
FROM customer_profiles WHERE customer_id = 1;

-- Containment query (uses GIN index)
SELECT * FROM customer_profiles
WHERE profile @> '{"preferences": {"theme": "dark"}}';

-- Key existence check
SELECT * FROM customer_profiles WHERE profile ? 'risk_profile';

-- Array containment
SELECT * FROM customer_profiles
WHERE profile->'metadata'->'tags' @> '["premium"]';
```

**Indexing JSONB:**
```sql
-- GIN index on entire document (supports @>, ?, ?&, ?|)
CREATE INDEX idx_profile_gin ON customer_profiles USING gin (profile);

-- Expression index on specific key (faster for targeted queries)
CREATE INDEX idx_profile_risk_score
ON customer_profiles ((profile->'risk_profile'->>'score'));

-- Partial index for specific conditions
CREATE INDEX idx_premium_customers ON customer_profiles (customer_id)
WHERE profile @> '{"metadata": {"tags": ["premium"]}';
```

**Trade-offs — JSONB vs Relational Columns:**

| Aspect | JSONB | Relational Columns |
|--------|-------|-------------------|
| Schema flexibility | No fixed schema; evolves freely | Fixed schema; migrations required |
| Query performance | GIN index adequate for containment | B-Tree index faster for equality/range |
| Data integrity | No type constraints at DB level | Strong typing, constraints, FKs |
| Storage | Larger overhead per document | Compact, column-level compression |
| Aggregation | Harder to aggregate across keys | Trivial with GROUP BY, window functions |

**The golden rule for banking**: Keep core transactional data (account balances, transaction amounts, customer IDs) in relational columns with strong typing and constraints. Use JSONB for semi-structured, frequently changing data where schema flexibility outweighs the loss of relational guarantees.

**Key Points to Hit:**
- [ ] JSONB is binary, indexed, supports containment queries via GIN
- [ ] Ideal for preferences, product configs, event payloads, GenAI metadata
- [ ] Core financial data stays in relational columns with constraints
- [ ] Expression indexes for frequently queried specific keys
- [ ] No DB-level schema enforcement; validation at application layer

**Follow-Up Questions:**
1. How do you update a single key in a JSONB document without rewriting the entire value?
2. When should you migrate JSONB data to relational columns?
3. What are the write performance implications of a GIN index on JSONB?

**Source:** `databases/jsonb.md`, `databases/indexing.md`

---

### Q9: How does pgvector enable vector similarity search in PostgreSQL? Compare HNSW vs IVFFlat indexes for a RAG system.

**Strong Answer:**

pgvector is a PostgreSQL extension that adds a `vector` data type and index methods for approximate nearest neighbor (ANN) search on embeddings. This makes Postgres a viable vector store for RAG (Retrieval-Augmented Generation) systems, eliminating the need for a separate vector database.

**Setup:**
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_chunks (
    chunk_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),  -- Matches text-embedding-3-small dimensions
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**HNSW (Hierarchical Navigable Small World):**

HNSW builds a multi-layer graph structure where each node connects to its nearest neighbors. Search traverses from the top layer (few connections, coarse) down to the bottom layer (many connections, fine-grained).

```sql
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 24, ef_construction = 128);

-- Query-time tuning
SET hnsw.ef_search = 100;  -- Default 40; higher = better recall, slower

-- Cosine similarity query
SELECT chunk_id, content, metadata,
       1 - (embedding <=> '[0.023, -0.045, ...]'::vector) AS similarity
FROM document_chunks
ORDER BY embedding <=> '[0.023, -0.045, ...]'::vector
LIMIT 5;
```

HNSW characteristics:
- **Best for**: High recall, moderate dataset size (< 10M vectors)
- **Build time**: Can be built on an empty table; builds incrementally
- **Query speed**: Fast with good recall; tunable via `ef_search`
- **Index size**: Larger than IVFFlat; ~30 minutes to build for 1M vectors
- **Parameters**: `m` (connections per layer, default 16), `ef_construction` (build quality, default 64)

**IVFFlat (Inverted File with Flat Quantization):**

IVFFlat partitions vectors into clusters (lists) and searches only the nearest clusters at query time.

```sql
-- Requires existing data to build lists
CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 1000);  -- sqrt(N) where N = number of vectors

SET ivfflat.probes = 10;  -- Default 1; higher = better recall, slower
```

IVFFlat characteristics:
- **Best for**: Very large datasets, faster build time
- **Build time**: Requires vectors already in the table
- **Query speed**: Faster than exact search, but lower recall than HNSW at same size
- **Parameters**: `lists` (number of clusters, sqrt(N)), `probes` (clusters to search)

**Comparison for banking RAG:**

| Aspect | HNSW | IVFFlat |
|--------|------|---------|
| Build on empty table | Yes | No (needs data) |
| Recall quality | Higher | Lower |
| Build speed | Slower | Faster |
| Query speed | Fast | Fast |
| Index size | Larger | Smaller |
| Best dataset size | < 10M vectors | > 10M vectors |
| Tuning complexity | Moderate (m, ef_construction, ef_search) | Moderate (lists, probes) |

**Distance operators:**
- `<=>` : Cosine distance (best for text embeddings like OpenAI)
- `<->` : Euclidean (L2) distance (when magnitude matters)
- `<#>` : Negative inner product (for normalized vectors, equivalent to cosine)

**Combined vector + keyword search for RAG:**
```sql
WITH vector_results AS (
    SELECT chunk_id, content, metadata,
           1 - (embedding <=> $1::vector) AS vector_score
    FROM document_chunks
    WHERE metadata @> $2::jsonb
    ORDER BY embedding <=> $1::vector
    LIMIT 20
),
keyword_results AS (
    SELECT chunk_id, content, metadata,
           ts_rank(to_tsvector('english', content), plainto_tsquery('english', $3)) AS keyword_score
    FROM document_chunks
    WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $3)
    LIMIT 20
)
SELECT COALESCE(v.chunk_id, k.chunk_id) AS chunk_id,
       COALESCE(v.content, k.content) AS content,
       COALESCE(v.vector_score, 0) + COALESCE(k.keyword_score, 0) AS combined_score
FROM vector_results v
FULL OUTER JOIN keyword_results k USING (chunk_id)
ORDER BY combined_score DESC
LIMIT 5;
```

**Key Points to Hit:**
- [ ] HNSW can build on empty tables; IVFFlat requires existing data
- [ ] HNSW gives higher recall; IVFFlat scales better for very large datasets
- [ ] Cosine distance (`<=>`) is best for text embeddings
- [ ] `ef_search` / `probes` control the speed/accuracy trade-off at query time
- [ ] Hybrid vector + keyword search improves RAG retrieval quality

**Follow-Up Questions:**
1. Your vector query on 5M documents is slow. What do you check?
2. How do you handle embedding model version changes (e.g., upgrading from ada-002 to text-embedding-3-small)?
3. Can you efficiently combine vector search with metadata filtering?

**Source:** `databases/pgvector.md`, `databases/indexing.md`

---

### Q10: What is Row-Level Security (RLS) and how would you implement multi-tenant data isolation for a banking platform?

**Strong Answer:**

Row-Level Security (RLS) enforces access control at the row level within the database itself, ensuring users can only access data they are authorized to see. In a multi-tenant banking platform, RLS prevents cross-tenant data leakage at the database level — a critical security requirement for regulated financial institutions.

**Basic RLS setup:**
```sql
-- Enable RLS on the table
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

-- Create a policy that filters rows based on tenant context
CREATE POLICY tenant_isolation_policy ON accounts
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

-- Set the tenant context from the application
SET app.current_tenant_id = '1';

-- Now all queries automatically filter to tenant 1
SELECT * FROM accounts;  -- Only returns accounts where tenant_id = 1
```

**Role-based policies for different access levels:**
```sql
CREATE ROLE tenant_user;
CREATE ROLE tenant_admin;

-- Regular users: see and modify only their tenant's data
CREATE POLICY tenant_user_select ON customers
    FOR SELECT TO tenant_user
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

CREATE POLICY tenant_user_modify ON customers
    FOR INSERT TO tenant_user
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

-- Admins: can see all tenants but only modify their own
CREATE POLICY tenant_admin_select ON customers
    FOR SELECT TO tenant_admin
    USING (true);  -- See all tenants

CREATE POLICY tenant_admin_modify ON customers
    FOR UPDATE TO tenant_admin
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::BIGINT);
```

**Application integration with connection pooling:**
```python
def get_tenant_connection(tenant_id: int):
    conn = psycopg2.connect(dsn)
    with conn.cursor() as cur:
        cur.execute("SET app.current_tenant_id = %s", (str(tenant_id),))
        cur.execute("SET ROLE tenant_user")
    return conn
```

**Critical RLS considerations for banking:**

1. **Index the filter column**: RLS adds a filter to every query. Without an index on `tenant_id`, every query does a sequential scan.
   ```sql
   CREATE INDEX idx_accounts_tenant ON accounts (tenant_id);
   ```

2. **Force RLS for superusers**: By default, table owners and superusers bypass RLS. In banking, you may want to force it:
   ```sql
   ALTER TABLE accounts FORCE ROW LEVEL SECURITY;
   ```

3. **Connection pooling compatibility**: With PgBouncer in transaction mode, the RLS context must be set at the start of each transaction (not once per connection). Use a connection wrapper that sets the context on checkout.

4. **Audit implications**: RLS policies should be documented, version-controlled, and reviewed as part of security audits. The policies themselves are part of your compliance evidence.

5. **RLS is not a substitute for application authorization**: RLS provides defense-in-depth. The application should still enforce its own access controls; RLS is the last line of defense.

**Key Points to Hit:**
- [ ] RLS enforces row-level access control at the database level
- [ ] Policies use `USING` (read filter) and `WITH CHECK` (write filter)
- [ ] Filter column must be indexed for performance
- [ ] Superusers bypass RLS by default; use `FORCE ROW LEVEL SECURITY`
- [ ] With PgBouncer transaction mode, context must be set per transaction

**Follow-Up Questions:**
1. What is the performance overhead of RLS? How do you measure it?
2. How does RLS interact with partitioning?
3. Can you implement RLS policies that combine tenant isolation with role-based access?

**Source:** `databases/row-level-security.md`, `databases/isolation-levels.md`

---

### Q11: How does table partitioning work in PostgreSQL? Explain partition pruning and when it does NOT kick in.

**Strong Answer:**

Table partitioning splits a large logical table into smaller physical pieces (partitions) while presenting a single logical table to applications. This is essential for banking tables with billions of rows — transaction history, audit logs, event streams.

**Declarative partitioning (range partitioning by date):**
```sql
CREATE TABLE transactions (
    transaction_id BIGINT,
    account_id BIGINT NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    transaction_time TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    PRIMARY KEY (transaction_id, transaction_time)  -- Partition key MUST be in PK
) PARTITION BY RANGE (transaction_time);

-- Create monthly partitions
CREATE TABLE transactions_2025_01
    PARTITION OF transactions FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE transactions_2025_02
    PARTITION OF transactions FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

**Partition Pruning:**

Partition pruning is the query planner's ability to skip scanning partitions that cannot possibly contain matching rows. When you query with a predicate on the partition key, the planner eliminates irrelevant partitions before execution.

```sql
-- This query only scans transactions_2025_01 — not all 36 monthly partitions
EXPLAIN SELECT * FROM transactions
WHERE transaction_time >= '2025-01-01' AND transaction_time < '2025-02-01';
```

Verify pruning is working:
```sql
SET enable_partition_pruning = off;
EXPLAIN SELECT * FROM transactions WHERE transaction_time >= '2025-01-01';
-- Shows all partitions scanned

SET enable_partition_pruning = on;
EXPLAIN SELECT * FROM transactions WHERE transaction_time >= '2025-01-01';
-- Shows only relevant partitions scanned
```

**When partition pruning does NOT kick in:**

1. **Partition key not in WHERE clause**: A query filtering only on `account_id` will scan all partitions because the planner cannot determine which partition(s) contain the data.

2. **Function wraps the partition key**:
   ```sql
   -- NO pruning: DATE() function hides the partition key from planner
   WHERE DATE(transaction_time) = '2025-01-15';

   -- YES pruning: direct comparison
   WHERE transaction_time >= '2025-01-15' AND transaction_time < '2025-01-16';
   ```

3. **Implicit type conversion**:
   ```sql
   -- May not prune if types don't match exactly
   WHERE transaction_time = '2025-01-15';
   ```

4. **Complex expressions the planner cannot simplify**: Expressions involving partition key with non-sargable operations prevent pruning.

**Partitioning strategies in banking:**

- **Range** (by date): Transaction history, audit logs — enables efficient data lifecycle management (drop old partitions for retention).
- **List** (by category): Accounts by type (checking, savings, loan) — enables per-type optimization.
- **Hash** (by customer_id): Event streams — enables parallel processing with even distribution.

```sql
-- List partitioning by account type
CREATE TABLE accounts (account_id BIGINT, account_type VARCHAR(20) NOT NULL, ...)
    PARTITION BY LIST (account_type);
CREATE TABLE accounts_checking PARTITION OF accounts FOR VALUES IN ('CHECKING');
CREATE TABLE accounts_savings PARTITION OF accounts FOR VALUES IN ('SAVINGS');

-- Hash partitioning for parallel processing
CREATE TABLE customer_events (event_id BIGINT, customer_id BIGINT NOT NULL, ...)
    PARTITION BY HASH (customer_id);
CREATE TABLE customer_events_p0 PARTITION OF customer_events
    FOR VALUES WITH (MODULUS 16, REMAINDER 0);
```

**Key Points to Hit:**
- [ ] Partition key must be part of the primary key
- [ ] Partition pruning skips irrelevant partitions — huge performance win
- [ ] Pruning fails when partition key is wrapped in a function or not in WHERE
- [ ] Range partitioning by date is most common for banking time-series data
- [ ] Old partitions can be detached and dropped for data retention

**Follow-Up Questions:**
1. How do you migrate a 500M row non-partitioned table to a partitioned table with zero downtime?
2. What happens to indexes when you partition a table?
3. How do you handle the default partition growing too large?

**Source:** `databases/partitioning.md`, `databases/indexing.md`

---

### Q12: What are the key memory configuration parameters in PostgreSQL and how do you size them for a production banking server?

**Strong Answer:**

PostgreSQL memory configuration is the foundation of performance tuning. The parameters must be sized based on available RAM and workload characteristics (OLTP vs analytics).

For a **64GB production banking server**, the recommended configuration is:

```yaml
# 1. shared_buffers: PostgreSQL's buffer pool
# Rule: 25% of RAM
# This is the primary cache for data pages
shared_buffers = 16GB

# 2. effective_cache_size: Planner's estimate of total available cache
# Rule: 75% of RAM (shared_buffers + OS page cache)
# Does NOT allocate memory — it's a hint to the planner
effective_cache_size = 48GB

# 3. work_mem: Per-operation memory for sorts, hashes, joins
# WARNING: multiplied by operations x connections
# OLTP: 64MB-256MB; Analytics: 256MB-1GB
work_mem = 256MB

# 4. maintenance_work_mem: For VACUUM, CREATE INDEX, ALTER TABLE
# Rule: 1-2GB for large databases
maintenance_work_mem = 2GB

# 5. wal_buffers: WAL write buffer
# Auto-tuned to 1/32 of shared_buffers (up to 16MB)
wal_buffers = 64MB

# 6. huge_pages: Use OS huge pages for shared_buffers
huge_pages = try
```

**Critical understanding:** `work_mem` is **per operation, per connection**. If a query has 3 sort operations and 2 hash joins, it can use up to `5 × work_mem` per connection. With 200 connections, that's potentially `200 × 5 × 256MB = 250GB` — far more than available RAM. The key is that not all operations run simultaneously, but you must set it carefully.

**Per-role overrides for different workloads:**
```sql
-- Analytics queries need more memory
ALTER ROLE analytics_user SET work_mem = '512MB';
ALTER ROLE analytics_user SET statement_timeout = '300000';

-- API requests need low latency
ALTER ROLE api_user SET statement_timeout = '5000';

-- ETL jobs benefit from higher work_mem
ALTER ROLE etl_user SET work_mem = '512MB';
```

**Connection management:** Do not increase `max_connections` beyond 200-300. Instead, use PgBouncer for connection pooling:

```ini
# PgBouncer: transaction-level pooling
pool_mode = transaction
max_client_conn = 10000     # App connections
default_pool_size = 50      # Actual DB connections
```

**Monitoring memory effectiveness:**
```sql
-- Check cache hit ratio (should be > 99% for OLTP)
SELECT
    sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS ratio
FROM pg_statio_user_tables;

-- Find queries spilling to disk (work_mem too low)
SELECT query, temp_files, temp_bytes
FROM pg_stat_user_tables;  -- Via pg_stat_statements
```

**Key Points to Hit:**
- [ ] shared_buffers = 25% RAM; effective_cache_size = 75% RAM
- [ ] work_mem is per-operation per-connection — can multiply quickly
- [ ] maintenance_work_mem speeds up VACUUM, CREATE INDEX
- [ ] Use PgBouncer instead of increasing max_connections
- [ ] Per-role overrides tailor memory to workload type

**Follow-Up Questions:**
1. Your server has a 90% cache hit ratio. What does this mean and what do you do?
2. A hash join is spilling to disk. How do you fix it without changing global work_mem?
3. Why is `effective_cache_size` not actually allocating memory?

**Source:** `databases/postgres-performance.md`, `databases/query-tuning.md`

---

### Q13: Explain the difference between streaming replication and logical replication in PostgreSQL. When would you use each in a banking architecture?

**Strong Answer:**

PostgreSQL offers two fundamentally different replication mechanisms, each serving distinct purposes in a banking architecture.

**Streaming Replication (Physical Replication):**

Streaming replication creates an exact byte-for-byte copy of the entire PostgreSQL cluster. The primary sends WAL records to standby servers, which replay them in lockstep.

```sql
-- Primary configuration
-- wal_level = replica
-- max_wal_senders = 10

-- Create replication slot
SELECT pg_create_physical_replication_slot('standby_1');

-- Check replication status
SELECT client_addr, state, sent_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
       sync_state
FROM pg_stat_replication;
```

Key characteristics:
- **Scope**: Entire cluster (all databases, all tables)
- **Latency**: Sub-second with synchronous replication
- **Standby usage**: Read-only (hot standby)
- **Failover**: Standby can be promoted to primary
- **Use case**: High availability, disaster recovery, read scaling

In banking, streaming replication powers:
- **Read replicas** for analytics, reporting, and BI dashboards
- **Disaster recovery** standby in a different availability zone
- **Synchronous replication** for zero-data-loss critical systems

```sql
-- Synchronous replication (zero data loss)
-- postgresql.conf: synchronous_standby_names = 'standby_1'
```

**Logical Replication:**

Logical replication replicates specific tables at the row level, allowing different schemas between publisher and subscriber. It uses a decoding plugin to translate WAL records into logical change events.

```sql
-- Primary: Create publication
CREATE PUBLICATION banking_publication FOR TABLE
    customers, accounts, transactions;

-- Subscriber: Create subscription
CREATE SUBSCRIPTION banking_subscription
    CONNECTION 'host=primary-ip dbname=banking user=replicator password=secure'
    PUBLICATION banking_publication
    WITH (copy_data = true, create_slot = true);
```

Key characteristics:
- **Scope**: Specific tables (selective replication)
- **Latency**: Near real-time, but slightly higher than streaming
- **Schema**: Publisher and subscriber can have different schemas
- **Use case**: CDC pipelines, zero-downtime migrations, heterogeneous targets

In banking, logical replication powers:
- **CDC to analytics**: Replicate core banking tables to a separate analytics database with a star schema
- **Zero-downtime migrations**: Replicate from PostgreSQL 14 to PostgreSQL 16, then cutover
- **Selective data sharing**: Replicate only customer and account tables to a downstream risk system

**Comparison:**

| Aspect | Streaming Replication | Logical Replication |
|--------|----------------------|---------------------|
| Scope | Entire cluster | Specific tables |
| Schema | Must be identical | Can differ |
| Read writes | Standby is read-only | Subscriber is fully writable |
| WAL level | `replica` | `logical` |
| Use case | HA, DR, read scaling | CDC, migration, selective sync |
| Failover | Standby promotion | Not applicable |

**Monitoring replication lag:**
```sql
-- Streaming replication lag
SELECT EXTRACT(EPOCH FROM replay_lag) AS lag_seconds
FROM pg_stat_replication;

-- Logical replication lag
SELECT srrelid::regclass AS table_name, srsubstate AS state
FROM pg_subscription_rel;
```

**Key Points to Hit:**
- [ ] Streaming = full cluster copy, identical schema, read-only standby
- [ ] Logical = specific tables, different schemas allowed, writable subscriber
- [ ] Streaming for HA/DR; Logical for CDC/migrations
- [ ] WAL level must be `replica` (streaming) or `logical` (both)
- [ ] Replication lag monitoring is critical for banking SLAs

**Follow-Up Questions:**
1. What happens to the primary when a synchronous standby goes down?
2. Your replication slot is holding gigabytes of WAL. What caused this?
3. How do you use logical replication for zero-downtime major version upgrades?

**Source:** `databases/replication.md`, `databases/postgres-fundamentals.md`

---

### Q14: How do you deploy database migrations with zero downtime in a production banking system?

**Strong Answer:**

Zero-downtime database migrations in banking require the **Expand-Migrate-Contract** pattern, which ensures backward compatibility across multiple application versions running simultaneously during a rolling deployment.

**The Three-Phase Pattern:**

**Phase 1: EXPAND** — Add new structures without removing old ones.
```sql
-- Scenario: Split customer "full_name" into "first_name" and "last_name"
ALTER TABLE customers
    ADD COLUMN first_name VARCHAR(100),
    ADD COLUMN last_name VARCHAR(100);
-- Deploy application v2 that writes to BOTH old and new columns
```

**Phase 2: MIGRATE** — Backfill data from old to new structures.
```sql
-- Backfill in batches to avoid long table locks
UPDATE customers
SET first_name = SPLIT_PART(full_name, ' ', 1),
    last_name = SUBSTRING(full_name FROM POSITION(' ' IN full_name) + 1)
WHERE first_name IS NULL
LIMIT 10000;
-- Repeat until all rows migrated
-- Deploy application v2 that reads from NEW columns
```

**Phase 3: CONTRACT** — Remove old structures.
```sql
-- Only after confirming all reads use new columns
ALTER TABLE customers DROP COLUMN full_name;
-- Deploy application v3 that no longer references old column
```

**Common migration patterns:**

**Adding a column with a default value:**
```sql
-- BAD: Locks table while updating every row with default
ALTER TABLE transactions ADD COLUMN channel VARCHAR(20) DEFAULT 'ONLINE';

-- GOOD: Add without default, backfill, then set default
ALTER TABLE transactions ADD COLUMN channel VARCHAR(20);
-- Backfill in batches (see Phase 2 above)
ALTER TABLE transactions ALTER COLUMN channel SET DEFAULT 'ONLINE';
ALTER TABLE transactions ALTER COLUMN channel SET NOT NULL;
```

**Creating indexes on large tables:**
```sql
-- Use CONCURRENTLY to avoid blocking writes
CREATE INDEX CONCURRENTLY idx_txn_channel ON transactions (channel);
```

**Flyway migration files:**
```sql
-- V20250115_001__add_transaction_channel.sql
ALTER TABLE transactions ADD COLUMN channel VARCHAR(20);
CREATE INDEX idx_txn_channel ON transactions (channel) WHERE channel IS NULL;

-- V20250115_002__backfill_transaction_channel.sql
-- Batched backfill (executed separately)

-- V20250115_003__set_transaction_channel_default.sql
UPDATE transactions SET channel = 'ONLINE' WHERE channel IS NULL;
ALTER TABLE transactions ALTER COLUMN channel SET DEFAULT 'ONLINE';
ALTER TABLE transactions ALTER COLUMN channel SET NOT NULL;
```

**Critical rules for banking migrations:**

1. **Never drop columns or tables in the same migration as adding new ones** — this breaks backward compatibility.
2. **All long-running operations must be batched** — a single UPDATE on 500M rows will lock the table.
3. **Migration tracking table** records all applied migrations for auditability:
   ```sql
   CREATE TABLE schema_migrations (
       migration_id VARCHAR(100) PRIMARY KEY,
       applied_at TIMESTAMPTZ DEFAULT NOW(),
       execution_time_ms INT,
       status VARCHAR(20) DEFAULT 'APPLIED'
   );
   ```
4. **Every migration must have a rollback plan** — even if the rollback is "restore from backup."
5. **Feature flags** allow gradual rollout: deploy the migration, enable the feature for 1% of users, monitor, then expand.
6. **Test on production-like data volumes** — a migration that runs in 1 second on dev may take 30 minutes on prod.

**Key Points to Hit:**
- [ ] Expand-Migrate-Contract: add new, backfill, then remove old
- [ ] Batch all long-running operations to avoid table locks
- [ ] CREATE INDEX CONCURRENTLY avoids blocking writes
- [ ] Add columns without NOT NULL/default first, then backfill
- [ ] Every migration needs a rollback plan and tracking record

**Follow-Up Questions:**
1. Your migration locked a production table for 30 minutes. How do you prevent this?
2. How do you handle migrations when multiple application versions run simultaneously?
3. What happens in PostgreSQL 11+ when you `ALTER TABLE ADD COLUMN ... DEFAULT value`?

**Source:** `databases/migrations.md`, `databases/transactions.md`

---

### Q15: How do you implement an audit trail in PostgreSQL for a regulated banking environment?

**Strong Answer:**

In a regulated banking environment, every data change to financial tables must be auditable — who changed what, when, and from where. PostgreSQL supports audit trails through trigger-based logging, and this is a regulatory requirement (not optional).

**Trigger-based audit trail:**

```sql
-- Central audit table
CREATE TABLE audit_trail (
    audit_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    record_id BIGINT NOT NULL,
    action VARCHAR(10) NOT NULL,   -- INSERT, UPDATE, DELETE
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

-- Attach to all financial tables
CREATE TRIGGER audit_accounts
    AFTER INSERT OR UPDATE OR DELETE ON accounts
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_fn();

CREATE TRIGGER audit_transactions
    AFTER INSERT OR UPDATE OR DELETE ON transactions
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_fn();
```

**Querying the audit trail:**
```sql
SELECT table_name, action, changed_by, changed_at,
       old_data->>'balance' AS old_balance,
       new_data->>'balance' AS new_balance
FROM audit_trail
WHERE table_name = 'accounts' AND record_id = 12345
ORDER BY changed_at DESC;
```

**Additional banking audit requirements:**

1. **Regulatory hold**: Prevent deletion of data under legal hold:
   ```sql
   CREATE TABLE regulatory_holds (
       hold_id BIGINT PRIMARY KEY,
       hold_type VARCHAR(20) CHECK (hold_type IN ('LITIGATION', 'AUDIT', 'INVESTIGATION', 'COMPLIANCE')),
       data_scope JSONB,
       start_date DATE NOT NULL,
       end_date DATE,
       status VARCHAR(20) DEFAULT 'ACTIVE'
   );

   -- Trigger that blocks DELETE when regulatory hold is active
   CREATE TRIGGER regulatory_hold_check
       BEFORE DELETE ON transactions
       FOR EACH ROW EXECUTE FUNCTION check_regulatory_hold();
   ```

2. **Soft deletes**: Never actually DELETE financial rows — use a `deleted_at` or `status` column:
   ```sql
   ALTER TABLE accounts ADD COLUMN deleted_at TIMESTAMPTZ;
   -- "Delete" = UPDATE accounts SET deleted_at = NOW() WHERE account_id = 1;
   ```

3. **Audit columns on every table**: Standard pattern for banking tables:
   ```sql
   created_at TIMESTAMPTZ DEFAULT NOW(),
   created_by VARCHAR(100),
   updated_at TIMESTAMPTZ DEFAULT NOW(),
   updated_by VARCHAR(100),
   deleted_at TIMESTAMPTZ  -- Soft delete
   ```

4. **Partition the audit table**: The audit trail grows unboundedly. Partition it by date for efficient queries and retention:
   ```sql
   CREATE TABLE audit_trail (
       audit_id BIGINT GENERATED ALWAYS AS IDENTITY,
       table_name VARCHAR(100) NOT NULL,
       changed_at TIMESTAMPTZ NOT NULL,
       -- ... other columns
   ) PARTITION BY RANGE (changed_at);
   ```

**Key Points to Hit:**
- [ ] Trigger-based audit captures old and new data as JSONB
- [ ] Regulatory hold prevents deletion of data under legal/audit hold
- [ ] Soft deletes (deleted_at column) preferred over actual DELETEs
- [ ] Standard audit columns (created_by, updated_by, deleted_at) on all tables
- [ ] Audit table should be partitioned by date for retention management

**Follow-Up Questions:**
1. What is the performance impact of audit triggers on high-throughput tables?
2. How do you handle audit trail for partitioned tables?
3. What are the trade-offs between trigger-based auditing and application-level audit logging?

**Source:** `databases/banking-db-patterns.md`, `databases/partitioning.md`

---

### Q16: Your PostgreSQL table has 50% dead tuples and is experiencing severe bloat. What caused this, how do you diagnose it, and how do you fix it?

**Strong Answer:**

Dead tuples are old row versions left behind by UPDATE and DELETE operations. Under MVCC, PostgreSQL keeps these versions because other transactions might still need to see them. VACUUM cleans them up by marking their space as reusable. When dead tuples accumulate to 50%, it means VACUUM is not keeping up.

**Root causes:**

1. **Long-running transactions**: A single 2-hour analytics query prevents VACUUM from cleaning any tuples that the transaction might need. This is the most common cause.
2. **Disabled or poorly tuned autovacuum**: If `autovacuum_enabled = false` or thresholds are too high, cleanup does not run frequently enough.
3. **High update/delete churn**: Tables with frequent updates (e.g., account status changes) generate many dead tuples.

**Diagnosis:**
```sql
-- Check dead tuple ratio across tables
SELECT
    schemaname,
    relname AS table_name,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Find long-running transactions blocking VACUUM
SELECT
    pid,
    usename,
    state,
    query,
    EXTRACT(EPOCH FROM (now() - query_start)) AS duration_seconds
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration_seconds DESC;

-- Check transaction ID age (wraparound risk)
SELECT datname, age(datfrozenxid) AS xid_age,
       2147483647 - age(datfrozenxid) AS xids_remaining
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

**Immediate fix:**
```sql
-- Standard VACUUM: marks dead tuple space as reusable (no table lock)
VACUUM VERBOSE transactions;

-- VACUUM ANALYZE: also updates statistics for the query planner
VACUUM ANALYZE transactions;

-- VACUUM FULL: rewrites the entire table, returns space to OS
-- WARNING: Takes an exclusive lock on the table (blocks all access)
-- Only use during maintenance windows
VACUUM FULL transactions;

-- Alternative: pg_repack extension (online table reorganization)
-- Does NOT require exclusive lock
-- pg_repack -d banking_db -t transactions
```

**Permanent fix — tune autovacuum for high-churn tables:**
```sql
-- Aggressive autovacuum for high-churn tables
ALTER TABLE transactions SET (
    autovacuum_vacuum_threshold = 500,
    autovacuum_vacuum_scale_factor = 0.01,  -- Vacuum after 1% dead tuples
    autovacuum_analyze_threshold = 500,
    autovacuum_analyze_scale_factor = 0.01
);

-- For critical tables, set statement timeout to prevent runaway queries
SET statement_timeout = '30min';
```

**Prevention:**
- Monitor `n_dead_tup` with alerts at 10% threshold
- Set `idle_in_transaction_session_timeout = 300000` (5 min) to kill idle transactions
- Use dedicated read replicas for analytics queries
- Consider `pg_repack` for periodic online table maintenance

**Key Points to Hit:**
- [ ] Dead tuples caused by MVCC — old row versions from UPDATE/DELETE
- [ ] Long-running transactions are the #1 cause of VACUUM prevention
- [ ] VACUUM reclaims space internally; VACUUM FULL returns to OS (exclusive lock)
- [ ] pg_repack provides online reorganization without exclusive locks
- [ ] Tune autovacuum per-table with aggressive thresholds for high-churn tables

**Follow-Up Questions:**
1. What is the difference between VACUUM and VACUUM FULL?
2. How does transaction ID wraparound relate to VACUUM?
3. When would you use pg_repack instead of VACUUM FULL?

**Source:** `databases/postgres-fundamentals.md`, `databases/postgres-performance.md`

---

### Q17: Explain the saga pattern and how it compares to two-phase commit for distributed banking transactions.

**Strong Answer:**

In a distributed banking system (microservices), a single business operation may span multiple databases or services. The challenge is maintaining atomicity across these boundaries. Two approaches exist: two-phase commit (2PC) and the saga pattern.

**Two-Phase Commit (2PC):**

2PC uses a coordinator to ensure all participants either commit or roll back together.

```sql
-- Phase 1: Prepare (lock resources)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
PREPARE TRANSACTION 'transfer_1_to_2';

-- Phase 2: Commit (or rollback)
COMMIT PREPARED 'transfer_1_to_2';
-- Or: ROLLBACK PREPARED 'transfer_1_to_2';
```

**Problems with 2PC in banking:**
1. **Blocking**: Prepared transactions hold locks until explicitly committed or rolled back. If the coordinator crashes, locks persist.
2. **Scalability**: Performance degrades with each additional participant.
3. **Availability**: If any participant is down, the entire transaction blocks.
4. **Complexity**: Requires a distributed transaction coordinator (e.g., JTA).

**The Saga Pattern:**

A saga breaks a distributed transaction into a sequence of local transactions, each with a compensating action (undo) for rollback.

```python
@dataclass
class SagaStep:
    action: Callable
    compensation: Callable
    description: str

class SagaOrchestrator:
    def execute(self, context: dict) -> dict:
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
                except Exception:
                    logger.critical(f"Compensation failed for {step.description}")
            raise

# Banking transfer saga
saga = SagaOrchestrator([
    SagaStep(
        action=lambda ctx: debit_account(ctx['from_account'], ctx['amount']),
        compensation=lambda ctx: credit_account(ctx['from_account'], ctx['amount']),
        description="Debit source account",
    ),
    SagaStep(
        action=lambda ctx: credit_account(ctx['to_account'], ctx['amount']),
        compensation=lambda ctx: debit_account(ctx['to_account'], ctx['amount']),
        description="Credit destination account",
    ),
    SagaStep(
        action=lambda ctx: log_transfer(ctx['from_account'], ctx['to_account'], ctx['amount']),
        compensation=lambda ctx: void_transfer_log(ctx['transfer_id']),
        description="Log transfer",
    ),
])
```

**Comparison:**

| Aspect | Two-Phase Commit | Saga Pattern |
|--------|-----------------|--------------|
| Atomicity | Strong (all-or-nothing at DB level) | Eventual (compensating actions) |
| Locking | Holds locks across all participants | Short-lived local locks only |
| Failure handling | Coordinator decides abort/commit | Compensation actions reverse changes |
| Scalability | Poor with many participants | Scales well |
| Complexity | Requires distributed coordinator | Application-level orchestration |
| Modern preference | Rarely used | Preferred in microservices |

**Banking considerations with sagas:**
- **Compensation failures**: If a compensation action itself fails, manual intervention is required. Log these critically.
- **Idempotency**: Saga steps must be idempotent — safe to retry.
- **Visibility**: Between saga steps, the system is in an inconsistent state. Other services must handle this gracefully.
- **Audit**: Each saga step and compensation must be logged for regulatory compliance.

**Key Points to Hit:**
- [ ] 2PC holds distributed locks; saga uses compensating actions
- [ ] Saga pattern is preferred in modern microservice architectures
- [ ] Compensation failures require manual intervention — log critically
- [ ] Saga steps must be idempotent for safe retries
- [ ] Between saga steps, the system is temporarily inconsistent

**Follow-Up Questions:**
1. How do you handle a compensation action that itself fails?
2. Can you implement saga orchestration within PostgreSQL using stored procedures?
3. How do you ensure saga steps are idempotent?

**Source:** `databases/transactions.md`, `databases/banking-db-patterns.md`

---

### Q18: How would you implement idempotent payment processing in PostgreSQL to prevent duplicate charges?

**Strong Answer:**

In banking payment processing, network timeouts and retries are common. If a client retries a payment that was actually processed but the response was lost, the system must not charge the customer twice. Idempotency ensures that the same request processed multiple times produces the same result as processing it once.

**Implementation using idempotency keys:**

```sql
CREATE TABLE idempotency_keys (
    key_hash VARCHAR(64) PRIMARY KEY,  -- SHA-256 hash of the idempotency key
    transaction_id VARCHAR(36),
    status VARCHAR(20) DEFAULT 'PENDING',  -- PENDING, COMPLETED, FAILED
    response_data JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '24 hours'
);

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
            -- Return previous result — no duplicate processing
            RETURN QUERY SELECT 'DUPLICATE', v_existing.transaction_id;
            RETURN;
        ELSIF v_existing.status = 'PENDING' THEN
            -- Still being processed by another request
            RETURN QUERY SELECT 'IN_PROGRESS', v_existing.transaction_id;
            RETURN;
        END IF;
    END IF;

    -- Generate unique transaction ID
    v_txn_id := 'txn_' || gen_random_uuid();

    -- Mark as processing (atomic insert)
    INSERT INTO idempotency_keys (key_hash, transaction_id, status)
    VALUES (v_key_hash, v_txn_id, 'PENDING');

    -- Process the actual payment
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

**Key design decisions:**

1. **Hash the idempotency key**: Store SHA-256 hash instead of raw key to prevent exposure of sensitive data and ensure fixed-length primary keys.

2. **Three-state status**: `PENDING` (being processed), `COMPLETED` (success), `FAILED` (error). This handles concurrent retries gracefully.

3. **Expiration**: Keys expire after 24 hours to prevent unbounded table growth. The expiration window should match the client's retry window.

4. **Atomic check-and-insert**: The initial INSERT into `idempotency_keys` acts as a distributed lock. If two requests arrive simultaneously with the same key, only one succeeds (unique constraint violation), and the other must handle the error.

5. **Application-level handling**: The client generates a unique idempotency key per request (e.g., UUID) and sends it with every request including retries.

```python
def make_payment(account_id, amount, merchant, idempotency_key):
    """Client sends payment with idempotency key."""
    response = db.execute("""
        SELECT process_payment(%s, %s, %s, %s)
    """, (idempotency_key, account_id, amount, merchant))

    status, txn_id = response

    if status == 'DUPLICATE':
        return {"status": "already_processed", "transaction_id": txn_id}
    elif status == 'IN_PROGRESS':
        return {"status": "still_processing", "transaction_id": txn_id}
    else:
        return {"status": "success", "transaction_id": txn_id}
```

**Banking context**: Idempotency is required by PCI-DSS and payment network specifications (Visa, Mastercard). Every external API call in a banking system should use idempotency keys.

**Key Points to Hit:**
- [ ] Idempotency key table with hash, status, and expiration
- [ ] Three states: PENDING, COMPLETED, FAILED for concurrent retry handling
- [ ] Atomic INSERT acts as distributed lock
- [ ] Return previous result for duplicate requests
- [ ] Expiration prevents unbounded table growth

**Follow-Up Questions:**
1. What happens if two requests with the same idempotency key arrive simultaneously?
2. How do you handle idempotency for operations that are not naturally idempotent (like incrementing a counter)?
3. How long should idempotency keys be retained?

**Source:** `databases/banking-db-patterns.md`, `databases/transactions.md`

---

### Q19: How do you perform a zero-downtime migration from a non-partitioned 500M row table to a partitioned table in PostgreSQL?

**Strong Answer:**

Migrating a massive table to partitioning without downtime is one of the most complex database operations. The strategy uses logical replication to keep the old and new tables in sync during the migration.

**Step-by-step approach:**

**Step 1: Create the new partitioned table structure.**
```sql
CREATE TABLE transactions_new (
    transaction_id BIGINT,
    account_id BIGINT NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    transaction_time TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    PRIMARY KEY (transaction_id, transaction_time)
) PARTITION BY RANGE (transaction_time);

-- Create partitions for the data range
CREATE TABLE transactions_new_2024_01
    PARTITION OF transactions_new
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
-- ... create all needed partitions
CREATE TABLE transactions_new_default
    PARTITION OF transactions_new DEFAULT;
```

**Step 2: Use logical replication to copy and sync data.**
```sql
-- On the primary, create a publication for the old table
CREATE PUBLICATION txn_migration_pub FOR TABLE transactions;

-- On the same database (different table), subscribe
-- Or use a programmatic approach (pglogical, Debezium)
-- to continuously copy data from old to new table
```

In practice, for same-database migration, use a programmatic approach:
```python
# Continuously copy new/updated rows from old to new table
# While also handling the initial bulk copy in batches
def migrate_batch(start_id, end_id):
    conn.execute("""
        INSERT INTO transactions_new
        SELECT * FROM transactions
        WHERE transaction_id BETWEEN %s AND %s
        ON CONFLICT DO NOTHING
    """, (start_id, end_id))
```

**Step 3: Dual-write during transition.** Modify the application to write to BOTH the old and new tables. This ensures no data is lost during the migration window.

**Step 4: Backfill historical data in batches.** Copy old data to the new table in small batches (10K-50K rows) to avoid long locks and WAL bloat.

**Step 5: Validate data consistency.** Compare row counts, sums, and checksums between old and new tables.

**Step 6: Switch reads to the new table.** Deploy application that reads from `transactions_new`. Keep writing to both.

**Step 7: Drop the old table.** After a validation period (days or weeks), remove the old table and rename the new one:
```sql
BEGIN;
ALTER TABLE transactions RENAME TO transactions_old;
ALTER TABLE transactions_new RENAME TO transactions;
-- Update all dependent objects (views, functions, triggers)
COMMIT;
```

**Alternative: pg_partman extension**

The `pg_partman` extension can automate partition creation and management, reducing operational overhead.

**Critical considerations:**
- **Foreign keys**: Partitioning breaks traditional foreign key references. You must handle referential integrity at the application level or use deferred constraints.
- **Unique constraints**: Must include the partition key.
- **Indexes**: Must be recreated on each partition. Use `CREATE INDEX CONCURRENTLY`.
- **WAL growth**: Bulk inserts generate significant WAL. Monitor `pg_wal` disk usage.
- **Rollback plan**: Keep the old table until the migration is fully validated.

**Key Points to Hit:**
- [ ] Create partitioned table alongside the original
- [ ] Dual-write during transition to prevent data loss
- [ ] Backfill in small batches (10K-50K rows)
- [ ] Logical replication or programmatic sync for continuous data copy
- [ ] Validate consistency before cutover; keep old table for rollback

**Follow-Up Questions:**
1. How do you handle foreign key constraints when partitioning a table?
2. What happens to indexes during the migration?
3. How do you minimize WAL growth during bulk data copy?

**Source:** `databases/partitioning.md`, `databases/migrations.md`, `databases/replication.md`

---

### Q20: You need to build a production RAG system for banking documents. Design the complete pgvector storage, indexing, and retrieval architecture.

**Strong Answer:**

Building a production RAG system for banking documents requires careful consideration of embedding storage, index strategy, metadata filtering, retrieval quality, and operational concerns.

**Database schema:**
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_chunks (
    chunk_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),  -- OpenAI text-embedding-3-small
    metadata JSONB DEFAULT '{}',
    embedding_model VARCHAR(50),
    embedding_version VARCHAR(20),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(document_id, chunk_index)
);

-- Critical: Track embedding model/version for future migrations
-- metadata structure:
-- {
--   "doc_type": "product_info",
--   "source": "savings_account_terms.pdf",
--   "department": "retail_banking",
--   "language": "en",
--   "last_reviewed": "2025-01-15",
--   "regulatory_flag": true,
--   "region": "US"
-- }
```

**Indexing strategy:**
```sql
-- HNSW index for fast approximate similarity search
CREATE INDEX CONCURRENTLY idx_chunks_embedding ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 24, ef_construction = 128);

-- GIN index on metadata for efficient containment filtering
CREATE INDEX CONCURRENTLY idx_chunks_metadata ON document_chunks
USING gin (metadata);

-- Query-time tuning
SET hnsw.ef_search = 100;  -- Higher recall for banking accuracy
```

**RAG retrieval query with hybrid scoring:**
```sql
WITH vector_results AS (
    SELECT
        chunk_id, document_id, content, metadata,
        1 - (embedding <=> $1::vector) AS vector_score,
        1 AS source_weight
    FROM document_chunks
    WHERE metadata @> $2::jsonb  -- Filter by doc_type, regulatory_flag, etc.
    ORDER BY embedding <=> $1::vector
    LIMIT 20
),
keyword_results AS (
    SELECT
        chunk_id, document_id, content, metadata,
        ts_rank(to_tsvector('english', content), plainto_tsquery('english', $3)) AS keyword_score,
        0.5 AS source_weight
    FROM document_chunks
    WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $3)
    LIMIT 20
),
combined AS (
    SELECT
        COALESCE(v.chunk_id, k.chunk_id) AS chunk_id,
        COALESCE(v.content, k.content) AS content,
        COALESCE(v.metadata, k.metadata) AS metadata,
        COALESCE(v.vector_score, 0) * COALESCE(v.source_weight, 0) +
        COALESCE(k.keyword_score, 0) * COALESCE(k.source_weight, 0) AS combined_score
    FROM vector_results v
    FULL OUTER JOIN keyword_results k USING (chunk_id)
)
SELECT chunk_id, content, metadata, combined_score
FROM combined
ORDER BY combined_score DESC
LIMIT 5;
```

**Production architecture decisions:**

1. **Embedding model versioning**: Track `embedding_model` and `embedding_version` columns. When upgrading models, you can re-embed incrementally and compare retrieval quality before cutover.

2. **Metadata filtering**: The GIN index on JSONB enables filtering by document type, department, regulatory status, and region before vector search. This is critical in banking — a compliance query should only search compliance documents, not product marketing.

3. **Hybrid retrieval**: Combining vector similarity with keyword search (BM25 via `ts_rank`) improves recall for queries where exact term matching matters (e.g., regulatory codes, product names).

4. **Chunking strategy**: Banking documents require careful chunking — regulatory clauses, fee schedules, and terms must not be split across chunks. Use document-aware chunking (by section/paragraph boundaries).

5. **Scaling considerations**:
   - HNSW build time for 1M vectors: ~30 minutes
   - Index size: ~10-20% of raw embedding size
   - For > 10M vectors, consider IVFFlat or a dedicated vector database
   - Read replicas can serve vector queries for read-heavy RAG workloads

6. **Compliance**: All document chunks must be auditable. The `metadata` JSONB should include source document provenance, review dates, and approval status for regulatory traceability.

7. **Monitoring**:
   ```sql
   -- Monitor index size
   SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
   FROM pg_indexes WHERE tablename = 'document_chunks';

   -- Monitor embedding distribution
   SELECT embedding_model, embedding_version, COUNT(*)
   FROM document_chunks GROUP BY embedding_model, embedding_version;
   ```

**Key Points to Hit:**
- [ ] HNSW index for similarity search; GIN index for metadata filtering
- [ ] Track embedding model/version for future migration paths
- [ ] Hybrid vector + keyword search improves RAG retrieval quality
- [ ] Metadata filtering ensures regulatory/compliance document scoping
- [ ] Chunking strategy must respect document structure (clauses, sections)

**Follow-Up Questions:**
1. How do you re-embed all documents when upgrading the embedding model?
2. At what scale does pgvector become inadequate vs a dedicated vector database?
3. How do you ensure the retrieved chunks are from the latest approved version of a banking document?

**Source:** `databases/pgvector.md`, `databases/indexing.md`, `databases/jsonb.md`
