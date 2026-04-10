# Backend Interview Questions (50+)

## Foundational Questions

### Q1: Explain the difference between REST and GraphQL. When would you use each?

**Strong Answer**: "REST uses multiple endpoints with fixed response structures -- each resource has its own URL, and you get a predetermined set of fields. GraphQL has a single endpoint where clients specify exactly what data they need in the query. REST is simpler to implement, cache, and monitor -- I'd use it for public APIs, webhook receivers, and when the response structure is stable. GraphQL excels when clients have diverse data needs (mobile vs. web vs. desktop), when you want to avoid over-fetching/under-fetching, and when the frontend team needs flexibility to change data requirements without backend changes. In banking, I typically use REST for external APIs (easier to version, document, and secure) and GraphQL for internal dashboards where multiple teams consume different subsets of data."

### Q2: How does database indexing work and when would you add an index?

**Strong Answer**: "A database index is a data structure (typically a B-tree) that speeds up lookups on specific columns at the cost of additional storage and slower writes. I add an index when: (1) A column is frequently used in WHERE clauses, JOIN conditions, or ORDER BY. (2) Query performance analysis (EXPLAIN ANALYZE) shows a sequential scan on a large table. (3) The column has high cardinality (many distinct values) -- indexes on low-cardinality columns (boolean, status enums) are less effective. I avoid indexing: columns that are frequently updated (each update must also update the index), very small tables (sequential scan is faster), or columns used only in SELECT (not WHERE). In PostgreSQL, I also consider partial indexes (index only rows matching a condition) and covering indexes (include all columns needed by the query to avoid table lookups)."

### Q3: What is the N+1 query problem and how do you solve it?

**Strong Answer**: "The N+1 problem occurs when you execute one query to fetch a list of records, then N additional queries to fetch related records for each item. For example: fetch 100 users (1 query), then for each user, fetch their orders (100 queries). Solutions: (1) **Eager loading**: Use JOIN or subquery to fetch all related records in one query. In SQLAlchemy: `session.query(User).options(joinedload(User.orders))`. (2) **Batch loading**: Collect all foreign keys from the first query and fetch all related records with an IN clause. (3) **DataLoader pattern**: Deduplicate and batch requests in GraphQL. The key is recognizing the pattern: if you see a query inside a loop, it's likely N+1."

### Q4: Explain the CAP theorem and its implications for distributed systems.

**Strong Answer**: "CAP theorem states that a distributed system can guarantee at most two of three properties: Consistency (all nodes see the same data at the same time), Availability (every request gets a response, even if some nodes are down), and Partition Tolerance (the system continues operating despite network failures between nodes). In practice, partition tolerance is mandatory for distributed systems, so the real choice is between Consistency and Availability. For banking: we prioritize Consistency over Availability for financial transactions (CP systems like PostgreSQL) -- it's better to return an error than to show incorrect account balances. For read-heavy, non-critical systems (dashboards, analytics), we may prioritize Availability (AP systems like Cassandra)."

### Q5: How do you handle database migrations in production?

**Strong Answer**: "Safe migration strategy: (1) **Backward-compatible changes first**: Add new columns as nullable, create new tables, add new indexes concurrently. (2) **Deploy code that works with both old and new schema**. (3) **Backfill data** if needed (in batches to avoid locking). (4) **Switch to new schema**: Update code to use new columns/tables. (5) **Remove old schema** in a subsequent migration. Key principles: every migration must be reversible (have a down migration), test migrations on a production-like dataset first, run schema changes during low-traffic periods, and monitor for lock contention. For large tables, use `CREATE INDEX CONCURRENTLY` in PostgreSQL to avoid blocking reads/writes."

## Intermediate Questions

### Q6: How would you design a rate limiter for an API?

**Strong Answer**: "I'd use a token bucket or sliding window algorithm stored in Redis. Token bucket: each user has a bucket with N tokens. Each request consumes one token. Tokens refill at a fixed rate. If the bucket is empty, the request is rejected. In Redis: store the bucket state as a hash with `tokens` and `last_refill` fields. Use Lua scripts for atomic check-and-decrement to avoid race conditions. For distributed rate limiting, Redis provides the centralized state. I'd also implement: per-endpoint rate limits (different limits for different API calls), burst allowance (temporary higher limits), and graceful degradation (queue requests instead of rejecting when near the limit). Response headers should include `X-RateLimit-Remaining` and `X-RateLimit-Reset`."

### Q7: Explain the difference between optimistic and pessimistic locking.

**Strong Answer**: "Pessimistic locking assumes conflicts will happen and prevents them by acquiring a lock before modifying data: `SELECT ... FOR UPDATE` in SQL. This blocks other transactions from reading or modifying the row until the lock is released. It's safe but reduces concurrency. Optimistic locking assumes conflicts are rare and checks for them at commit time: each row has a version number or timestamp. When updating, you check `WHERE id = ? AND version = ?`. If no rows are updated, someone else modified the row concurrently, and you retry. Optimistic locking is better for read-heavy workloads where conflicts are rare. Pessimistic locking is better for write-heavy workloads or when conflicts would be very costly (e.g., double-charging a customer). In banking, I use pessimistic locking for account balance updates and optimistic locking for less critical data like user preferences."

### Q8: How do you design an idempotent API?

**Strong Answer**: "Idempotency means making the same request multiple times has the same effect as making it once. Key strategies: (1) **Idempotency keys**: Client sends a unique key with each request. Server stores the key and result. If the same key is received again, return the cached result instead of re-executing. (2) **HTTP semantics**: Use PUT for updates (inherently idempotent), POST for creates (not idempotent by default). (3) **Natural idempotency**: Design operations so that repeating them has no additional effect (e.g., `SET status = 'completed'` is idempotent; `ADD 10 TO balance` is not). (4) **Idempotency store**: Redis or database table storing idempotency keys with their results, with TTL. In banking, idempotency is critical for payment processing -- a retry due to network timeout should not result in a double charge."

### Q9: How do you handle long-running operations in an API?

**Strong Answer**: "For operations that take more than a few seconds, I use an async pattern: (1) Client POSTs to initiate the operation. (2) Server returns 202 Accepted with a job ID and status URL immediately. (3) Server processes the operation in a background worker (Celery, Kafka consumer, etc.). (4) Client polls the status URL or receives a webhook callback when complete. (5) Status endpoint returns: pending, in_progress, completed (with result), or failed (with error). For banking, this pattern is used for bulk data exports, report generation, and batch transaction processing. I also implement: job timeout (fail if not completed within SLA), job cancellation, progress reporting (e.g., 'processing 300 of 1000 records'), and retry logic for transient failures."

### Q10: Explain how you would implement full-text search without Elasticsearch.

**Strong Answer**: "PostgreSQL has built-in full-text search using `tsvector` and `tsquery`. I'd: (1) Create a `tsvector` column from the text content: `UPDATE docs SET search_vector = to_tsvector('english', content)`. (2) Create a GIN index on the tsvector column: `CREATE INDEX ON docs USING gin(search_vector)`. (3) Search using `@@`: `SELECT * FROM docs WHERE search_vector @@ plainto_tsquery('english', 'loan processing fee')`. For ranking, use `ts_rank()` to order results by relevance. For typo tolerance, I'd add trigram indexes with `pg_trgm`. This approach handles moderate-scale search well (up to ~1M documents). For larger scale or advanced features (faceting, highlighting, multi-language), I'd use Elasticsearch. The advantage of PostgreSQL full-text search is zero additional infrastructure."

## Advanced Questions

### Q11: How do you design a system that processes 1 million events per minute?

**Strong Answer**: "Architecture: (1) **Ingestion**: Kafka as the event buffer -- it handles high-throughput writes and provides ordered, durable storage. Multiple partitions for parallelism. (2) **Processing**: Stateless worker services consuming from Kafka topics. Scale horizontally by adding partitions and consumers. Use a framework like Kafka Streams or build custom consumers with offset management. (3) **Storage**: Write results to a time-series database (TimescaleDB) for recent data and S3/Parquet for long-term storage. (4) **Backpressure**: Monitor consumer lag. If lag grows, auto-scale workers. (5) **Error handling**: Dead letter queue for unprocessable events. Retry with exponential backoff for transient errors. (6) **Monitoring**: Track throughput, consumer lag, error rate, and processing latency. (7) **Exactly-once semantics**: Use Kafka's transactional API and idempotent writes to the database. Key design principle: make workers stateless and idempotent so they can be scaled and restarted freely."

### Q12: How would you implement a distributed cache invalidation strategy?

**Strong Answer**: "Cache invalidation is famously one of the hardest problems in computer science. Strategies: (1) **TTL-based**: Set short TTLs (5-15 minutes) so stale data expires automatically. Simple but means data can be stale for up to the TTL. (2) **Write-through**: Update cache when writing to the database. Keeps cache fresh but adds latency to writes. (3) **Cache-aside**: Check cache first; if miss, read from DB and populate cache. Most common pattern. (4) **Event-driven invalidation**: When data changes, publish an event that triggers cache invalidation. (5) **Versioned keys**: Include a version number in the cache key. When data changes, increment the version -- old cache entries become unreachable. For a banking system, I'd combine TTL (safety net) with event-driven invalidation (freshness). Critical data (account balances) gets shorter TTLs or write-through caching. Non-critical data (user preferences) gets longer TTLs."

### Q13: How do you handle schema evolution in a microservices architecture?

**Strong Answer**: "Key principles: (1) **Never break backward compatibility**: Add fields, don't remove or rename. Make new fields optional/nullable. (2) **API versioning**: Use URL versioning (`/api/v2/users`) or header-based versioning for breaking changes. (3) **Contract testing**: Use Pact or similar to ensure provider and consumer contracts are compatible. (4) **Database per service**: Each service owns its database schema. Cross-service data sharing via APIs, not direct database access. (5) **Expand/contract pattern**: For schema changes that affect multiple services: first expand (add new fields), deploy consumers that can read both old and new, deploy producers that write new format, then contract (remove old fields). (6) **Event schema evolution**: Use Schema Registry (Confluent) with backward-compatible schema changes for event-driven communication. In banking, schema changes require thorough testing and sometimes regulatory notification, so the expand/contract pattern is essential."

### Q14: Design a URL shortener at scale.

**Strong Answer**: "Core requirements: shorten URLs, redirect to original, track click analytics. Design: (1) **Encoding**: Base62 encoding of a unique ID. For 7-character codes: 62^7 = 3.5 trillion possible URLs. (2) **ID generation**: Use a distributed ID generator (Snowflake algorithm) or a database sequence with range allocation (each server gets a range of 1M IDs). (3) **Storage**: Key-value store (Redis for hot URLs, PostgreSQL for persistence). Schema: `{short_code, original_url, created_at, user_id, expires_at}`. (4) **Redirect**: 301 (permanent) or 302 (temporary) HTTP redirect. 302 is better for analytics (every request hits the server). (5) **Caching**: Redis cache for popular URLs. (6) **Analytics**: Async write to click tracking table via message queue. (7) **Scale**: Read-heavy workload (100:1 read:write ratio). CDN for redirect responses, read replicas for analytics queries. (8) **Custom URLs**: Allow users to choose custom short codes with collision checking."

### Q15: How do you debug a production memory leak in a Python application?

**Strong Answer**: "Systematic approach: (1) **Confirm the leak**: Monitor RSS memory over time with Prometheus/Grafana. A steady upward trend confirms a leak. (2) **Take heap snapshots**: Use `tracemalloc` (built-in Python) or `objgraph` to capture heap state at two points in time. (3) **Compare snapshots**: Identify objects that are growing between snapshots. (4) **Common culprits in Python**: Unbounded caches (dicts that grow forever), unclosed database connections, circular references preventing garbage collection, module-level variables accumulating data, logging handlers holding references. (5) **Fix**: Add size limits to caches, use context managers for resources, break circular references with `weakref`. (6) **Prevent**: Add memory usage alerts, use memory profiling in staging, code review for common leak patterns. In production, I'd also implement automatic restart when memory exceeds a threshold as a safety net."

## Banking-Specific Backend Questions

### Q16: How would you design an API for transferring money between accounts?

**Strong Answer**: "Critical requirements: atomicity, idempotency, auditability. Design: (1) **Request**: POST `/transfers` with `{from_account, to_account, amount, reference, idempotency_key}`. (2) **Validation**: Check accounts exist, sufficient balance, within daily limits, not frozen. (3) **Transaction**: Use database transaction with `SELECT ... FOR UPDATE` on both accounts to prevent concurrent modifications. (4) **Double-entry bookkeeping**: Debit from source, credit to destination. Both must succeed or both must fail. (5) **Audit log**: Record the transfer in an append-only audit table before committing. (6) **Idempotency**: Check idempotency key; if seen before, return the previous result. (7) **Response**: Return transfer ID, new balances, and timestamp. (8) **Async notification**: Publish event for downstream systems (fraud detection, notification service, reconciliation). Error handling: insufficient funds (400), account not found (404), concurrent modification (409 with retry), system error (500 with idempotent retry)."

### Q17: How do you ensure data consistency across multiple banking services?

**Strong Answer**: "In a microservices banking architecture, I use the Saga pattern: (1) Each service manages its own database. (2) A cross-service operation is a sequence of local transactions, each with a compensating action. (3) If any step fails, execute compensating actions in reverse order. Example: opening a new account requires (a) create account record, (b) set up billing, (c) send welcome email. If step (b) fails, compensate by deleting the account from step (a). For operations that require strong consistency (transfers), I keep them in a single service with a single database. I also implement: event sourcing for audit trail, outbox pattern for reliable event publishing (write to database and message queue atomically), and periodic reconciliation jobs to detect and fix any inconsistencies."

### Q18: How would you handle a situation where a payment was debited but not credited?

**Strong Answer**: "This is a critical production incident. Steps: (1) **Investigate**: Check the transaction log, audit trail, and both account balances. Identify where the process failed. (2) **Contain**: If it's a systemic issue (affecting multiple transactions), pause the payment processing pipeline. (3) **Fix**: Manually credit the destination account with a journal entry that references the original transaction. This must be done by the operations team with proper authorization. (4) **Communicate**: Notify the affected customer and the operations team. (5) **Root cause**: Was it a network timeout, database deadlock, application bug? (6) **Prevent**: Implement idempotent payment processing so retries don't double-charge. Use the outbox pattern to ensure the debit event is always published. Add monitoring that detects unmatched debits/credits within a time window. Design the system so every debit has a corresponding credit entry (double-entry bookmaking)."

## Quick-Fire Questions

### Q19: What is the difference between TCP and UDP?
**Answer**: "TCP is connection-oriented, reliable, and ordered -- every packet is acknowledged and retransmitted if lost. UDP is connectionless, unreliable, and unordered -- fire and forget. TCP for APIs, databases, file transfers. UDP for real-time streaming, VoIP, gaming."

### Q20: What is a circuit breaker?
**Answer**: "A pattern that wraps calls to external services. If the service fails repeatedly, the circuit 'opens' and calls fail immediately without waiting for timeout. After a cooldown period, it 'half-opens' to test if the service recovered. Prevents cascading failures."

### Q21: What is the difference between horizontal and vertical scaling?
**Answer**: "Vertical scaling: add more resources (CPU, RAM) to a single machine. Limited by hardware ceiling. Horizontal scaling: add more machines. Theoretically unlimited but requires distributed system complexity."

### Q22: What is eventual consistency?
**Answer**: "A consistency model where updates propagate asynchronously across nodes. After a write, reads may temporarily return stale data, but eventually all nodes converge to the same state. Acceptable for non-critical data; not for account balances."

### Q23: What is the difference between authentication and authorization?
**Answer**: "Authentication verifies WHO you are (login, MFA). Authorization verifies WHAT you can do (permissions, roles). AuthN comes first, AuthZ follows."

### Q24: How does a load balancer work?
**Answer**: "Distributes incoming requests across multiple backend servers. Algorithms: round-robin, least connections, IP hash. Health checks remove unhealthy servers from the pool. Types: L4 (transport layer) and L7 (application layer, can inspect HTTP)."

### Q25: What is a dead letter queue?
**Answer**: "A queue where messages that can't be processed after multiple retries are sent for manual inspection. Prevents poison messages from blocking the main queue indefinitely."
