# Database Terms

> Essential database terminology for GenAI platform engineers. Each term includes definition, banking/GenAI example, related concepts, and common misunderstandings.

## Glossary

### 1. ACID
**Definition:** Properties that guarantee database transactions are processed reliably: Atomicity (all or nothing), Consistency (valid state), Isolation (concurrent transactions don't interfere), Durability (persisted after commit).
**Example:** When a GenAI query is saved to the database, ACID ensures that the query, response, and audit log are all saved together — or none of them are, preventing partial records.
**Related Concepts:** Transaction, isolation level, CAP theorem
**Common Misunderstanding:** ACID is not a single setting — it's a set of properties. Some databases support ACID, others don't (eventual consistency databases).

### 2. Normalization
**Definition:** Organizing database tables to reduce data redundancy and improve data integrity by splitting data into related tables.
**Example:** Instead of storing user details in every query row, the queries table references a users table via foreign key — user details are stored once and referenced.
**Related Concepts:** Denormalization, foreign key, join, schema design
**Common Misunderstanding:** Normalization is not always better. Over-normalization leads to excessive joins, which hurt query performance.

### 3. Denormalization
**Definition:** Intentionally duplicating data across tables to reduce joins and improve read performance at the cost of storage and write complexity.
**Example:** The GenAI analytics table stores the department name directly (duplicated from the users table) to avoid joins in reporting queries.
**Related Concepts:** Normalization, materialized view, read optimization
**Common Misunderstanding:** Denormalization is a conscious trade-off, not a mistake. It's the right choice when read performance matters more than storage efficiency.

### 4. Index
**Definition:** A data structure that improves the speed of data retrieval operations on a database table at the cost of additional writes and storage.
**Example:** An index on `queries(created_at, user_id)` makes the "recent queries by user" query fast — from scanning 10M rows to finding the relevant rows in milliseconds.
**Related Concepts:** B-tree, GIN, BRIN, composite index, covering index
**Common Misunderstanding:** Indexes speed up reads but slow down writes (every INSERT/UPDATE must also update indexes). Don't index everything.

### 5. EXPLAIN ANALYZE
**Definition:** A SQL command that shows how the database engine executes a query, including the actual time spent at each step.
**Example:** `EXPLAIN ANALYZE SELECT * FROM queries WHERE department = 'engineering' AND created_at > NOW() - INTERVAL '7 days';` reveals a sequential scan on 10M rows — indicating a missing index.
**Related Concepts:** Query plan, sequential scan, index scan, cost estimation
**Common Misunderstanding:** EXPLAIN (without ANALYZE) shows the estimated plan. EXPLAIN ANALYZE actually runs the query and shows real execution times.

### 6. Query Plan
**Definition:** The sequence of operations the database engine uses to execute a SQL query, including scan types, join methods, and sort operations.
**Example:** The query plan for a GenAI analytics query shows: Sequential Scan on queries → Hash Join with users → Sort by count → Limit 10. The sequential scan on 10M rows is the bottleneck.
**Related Concepts:** EXPLAIN ANALYZE, optimizer, index, statistics
**Common Misunderstanding:** Query plans can change over time as data grows and statistics are updated. A query that's fast today may be slow tomorrow.

### 7. Connection Pool
**Definition:** A cache of database connections maintained so that connections can be reused when future requests to the database are required.
**Example:** The GenAI service uses PgBouncer to maintain a pool of 20 database connections. Instead of opening a new connection for each request (10-50ms overhead), it reuses an existing one (<1ms).
**Related Concepts:** PgBouncer, connection limit, connection leak
**Common Misunderstanding:** Connection pools don't solve all performance problems. If all connections in the pool are busy, new requests must wait.

### 8. N+1 Query Problem
**Definition:** An anti-pattern where an application makes one query to fetch a list of items, then N additional queries (one per item) to fetch related data.
**Example:** The GenAI dashboard fetches 100 users (1 query), then makes 100 separate queries to get each user's query count. Total: 101 queries instead of 1 query with a JOIN or subquery.
**Related Concepts:** JOIN, eager loading, subquery, ORM
**Common Misunderstanding:** The N+1 problem is often caused by ORMs that lazy-load relationships by default. Use eager loading (`.joinedload()` in SQLAlchemy) to fix it.

### 9. Migration
**Definition:** A version-controlled script that changes the database schema (add/modify/drop tables, columns, indexes) in a reversible way.
**Example:** A migration adds the `embedding` column (vector type) to the `policy_documents` table and creates the IVFFlat index. The reverse migration drops the index and column.
**Related Concepts:** Alembic, Flyway, backward compatibility, zero-downtime migration
**Common Misunderstanding:** Migrations are not just about schema changes. They can also include data migrations (transforming existing data to a new format).

### 10. Zero-Downtime Migration
**Definition:** A database schema change that can be applied without taking the application offline.
**Example:** To add a new column: (1) Add the column as nullable, (2) Deploy code that writes to both old and new columns, (3) Backfill existing rows, (4) Make the column NOT NULL, (5) Remove writes to the old column.
**Related Concepts:** Expand-contract pattern, backward compatibility, blue-green deployment
**Common Misunderstanding:** Zero-downtime migrations take multiple deployment cycles. They're slower than simple migrations but necessary for 24/7 services.

### 11. CTE (Common Table Expression)
**Definition:** A named temporary result set that exists within the scope of a single query, created using the `WITH` clause.
**Example:** `WITH daily_queries AS (SELECT DATE(created_at) as day, COUNT(*) as cnt FROM queries GROUP BY DATE(created_at)) SELECT day, cnt, AVG(cnt) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg FROM daily_queries;`
**Related Concepts:** Subquery, window function, recursive CTE
**Common Misunderstanding:** CTEs are not always performance-neutral. In PostgreSQL 12+, the optimizer can inline CTEs. In older versions, CTEs act as optimization barriers (always materialized).

### 12. Window Function
**Definition:** A function that performs a calculation across a set of table rows related to the current row, without collapsing them into a single output row.
**Example:** `RANK() OVER (PARTITION BY department ORDER BY query_count DESC)` ranks users within each department by their query count.
**Related Concepts:** PARTITION BY, ORDER BY, aggregate function, CTE
**Common Misunderstanding:** Window functions don't reduce the number of rows. They add computed columns to existing rows. Use GROUP BY if you need to reduce rows.

### 13. Materialized View
**Definition:** A pre-computed result set stored physically in the database, which can be queried like a regular table but must be refreshed periodically.
**Example:** A materialized view stores daily model performance metrics (query count, p50/p95/p99 latency) — refreshed hourly. Dashboard queries hit the materialized view instead of computing metrics from 10M rows.
**Related Concepts:** View, index, refresh, denormalization
**Common Misunderstanding:** Materialized views are not automatically updated. They must be explicitly refreshed (`REFRESH MATERIALIZED VIEW`), and concurrent refreshes require `CONCURRENTLY` to avoid blocking.

### 14. Partitioning
**Definition:** Splitting a large table into smaller, more manageable pieces while maintaining the appearance of a single table.
**Example:** The `queries` table is partitioned by month (`PARTITION BY RANGE (created_at)`). Each month's data is a separate partition, making date-range queries scan only relevant partitions.
**Related Concepts:** Range partitioning, list partitioning, hash partitioning, index
**Common Misunderstanding:** Partitioning is not a substitute for indexing. A partitioned table still needs indexes on each partition. But partitioning reduces the amount of data scanned.

### 15. Replication
**Definition:** Copying data from a primary database to one or more replica databases for read scaling, high availability, or disaster recovery.
**Example:** The GenAI platform uses PostgreSQL streaming replication: one primary for writes, two replicas for read queries (analytics dashboards, reporting).
**Related Concepts:** Primary/replica, lag, failover, read replica
**Common Misunderstanding:** Replication is asynchronous by default, meaning replicas may be slightly behind the primary. Reads from replicas may return stale data (replication lag).

### 16. CAP Theorem
**Definition:** In a distributed system, you can only guarantee two of three properties: Consistency (all nodes see the same data), Availability (every request gets a response), Partition tolerance (system works despite network failures).
**Example:** PostgreSQL prioritizes Consistency and Partition tolerance (CP). Cassandra prioritizes Availability and Partition tolerance (AP). The GenAI platform uses PostgreSQL for audit logs (consistency-critical) and Redis for caching (availability-critical).
**Related Concepts:** PACELC, eventual consistency, strong consistency
**Common Misunderstanding:** CAP doesn't mean you choose two globally. Modern systems offer tunable consistency — strong for some operations, eventual for others.

### 17. OLTP vs. OLAP
**Definition:** OLTP (Online Transaction Processing) handles many small, fast transactions. OLAP (Online Analytical Processing) handles fewer, complex analytical queries.
**Example:** The GenAI chat service uses OLTP (PostgreSQL) for storing individual queries. The analytics team uses OLAP (a separate reporting database) for aggregate trend analysis.
**Related Concepts:** Data warehouse, ETL, column store, row store
**Common Misunderstanding:** Running OLAP queries on an OLTP database degrades transaction performance. Separate the workloads.

### 18. ORM (Object-Relational Mapping)
**Definition:** A programming technique that converts data between incompatible type systems (object-oriented code and relational database tables).
**Example:** SQLAlchemy maps Python classes to PostgreSQL tables. `User.query.filter_by(department='engineering').all()` generates and executes the corresponding SQL.
**Related Concepts:** SQLAlchemy, Prisma, N+1 problem, raw SQL
**Common Misunderstanding:** ORMs are convenient but can generate inefficient SQL. Always review the generated SQL (`.echo=True` in SQLAlchemy) and use raw SQL when the ORM can't express efficient queries.

### 19. Deadlock
**Definition:** A situation where two or more transactions are waiting for each other to release locks, creating a circular dependency that cannot be resolved.
**Example:** Transaction A locks row 1 and tries to lock row 2. Transaction B locks row 2 and tries to lock row 1. Neither can proceed. PostgreSQL detects this and aborts one transaction.
**Related Concepts:** Lock, isolation level, retry
**Common Misunderstanding:** Deadlocks are not bugs — they're a natural consequence of concurrent transactions. The fix is to ensure transactions access resources in a consistent order.

### 20. pgvector
**Definition:** A PostgreSQL extension that stores and queries vector embeddings, enabling similarity search within a regular PostgreSQL database.
**Example:** The GenAI RAG pipeline stores document embeddings (384-dimensional vectors) in a pgvector column and uses cosine distance for similarity search: `SELECT * FROM documents ORDER BY embedding <=> query_embedding LIMIT 5;`
**Related Concepts:** Vector database, embedding, cosine similarity, IVFFlat, HNSW
**Common Misunderstanding:** pgvector is not a standalone vector database — it's a PostgreSQL extension. It shares the same infrastructure, backup, and security model as the rest of PostgreSQL.

### 21. IVFFlat vs. HNSW Index
**Definition:** Two indexing methods for vector similarity search in pgvector. IVFFlat uses inverted file clustering (faster build, good recall). HNSW uses graph-based search (better accuracy, slower build).
**Example:** The GenAI platform uses IVFFlat for 2M documents (faster index build, adequate recall). If scaling to 50M documents, HNSW would provide better recall at the cost of longer index build time.
**Related Concepts:** Approximate nearest neighbor, recall, index build time
**Common Misunderstanding:** IVFFlat and HNSW provide approximate results, not exact. Some nearest neighbors may be missed. The `lists` (IVFFlat) or `m`/`ef_construction` (HNSW) parameters control the accuracy-speed trade-off.

### 22. Transaction Isolation Level
**Definition:** Controls how concurrent transactions interact with each other, balancing consistency against performance.
**Example:** The GenAI audit logging uses `READ COMMITTED` (default) — each statement sees the latest committed data. The financial reporting query uses `SERIALIZABLE` to ensure a consistent snapshot across the entire report.
**Related Concepts:** Read uncommitted, read committed, repeatable read, serializable, snapshot
**Common Misunderstanding:** Higher isolation levels provide more consistency but reduce concurrency. Use the lowest level that meets your consistency requirements.

### 23. Foreign Key
**Definition:** A constraint that ensures a value in one table matches a value in another table, maintaining referential integrity.
**Example:** The `queries.user_id` column has a foreign key constraint referencing `users.user_id`. This prevents queries from being associated with non-existent users.
**Related Concepts:** Referential integrity, cascade, ON DELETE, normalization
**Common Misunderstanding:** Foreign keys add overhead to INSERT and UPDATE operations (the referenced value must be checked). For high-throughput systems, some teams enforce referential integrity in application code instead.

### 24. Vacuum
**Definition:** A PostgreSQL maintenance operation that reclaims storage occupied by dead tuples (rows deleted or updated by previous transactions).
**Example:** Auto-vacuum runs automatically on the `queries` table, reclaiming space from deleted/updated rows. If auto-vacuum falls behind, the table bloats and query performance degrades.
**Related Concepts:** Auto-vacuum, table bloat, dead tuples, ANALYZE
**Common Misunderstanding:** VACUUM does not return space to the operating system — it makes space available for reuse within the table. VACUUM FULL does return space but requires an exclusive lock.

### 25. Connection Limit
**Definition:** The maximum number of simultaneous connections a database server will accept.
**Example:** PostgreSQL's default `max_connections` is 100. The GenAI platform has 4 service pods × 20 connections each = 80 connections. Adding analytics connections risks hitting the limit, causing "too many clients" errors.
**Related Concepts:** Connection pool, PgBouncer, max_connections, superuser_reserved_connections
**Common Misunderstanding:** Increasing `max_connections` is not a solution to connection exhaustion — each connection consumes memory. Use connection pooling (PgBouncer) instead of increasing the limit indefinitely.
