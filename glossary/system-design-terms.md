# System Design Terms

> Essential system design terminology for GenAI platform engineers. Each term includes definition, banking/GenAI example, related concepts, and common misunderstandings.

## Glossary

### 1. Scalability
**Definition:** A system's ability to handle increased load by adding resources, without degrading performance.
**Example:** The GenAI chat service scales horizontally — when query volume increases from 50 to 500 per minute, Kubernetes adds more pods to maintain response times.
**Related Concepts:** Horizontal scaling, vertical scaling, elasticity, load balancing
**Common Misunderstanding:** Scalability is not the same as performance. A system can be fast but not scalable (works well for 100 users, breaks at 1000), or scalable but slow (handles millions of users, each with 2s latency).

### 2. Horizontal Scaling (Scale Out)
**Definition:** Adding more machines/nodes to a system to increase capacity.
**Example:** The GenAI platform scales from 4 to 12 chat service pods to handle increased query volume. Each pod handles a portion of the load.
**Related Concepts:** Vertical scaling, load balancing, statelessness, partitioning
**Common Misunderstanding:** Horizontal scaling only works for stateless services or properly partitioned data. You can't just add more database nodes without sharding.

### 3. Vertical Scaling (Scale Up)
**Definition:** Adding more resources (CPU, memory) to an existing machine to increase capacity.
**Example:** Increasing the GenAI retriever pod's memory from 512Mi to 2Gi so it can load a larger embedding model.
**Related Concepts:** Horizontal scaling, resource limits, single point of failure
**Common Misunderstanding:** Vertical scaling has a hard ceiling (the largest available machine). Horizontal scaling theoretically has no ceiling.

### 4. Availability
**Definition:** The percentage of time a system is operational and accessible to users.
**Example:** The GenAI platform targets 99.9% availability — meaning no more than 43 minutes of downtime per month.
**Related Concepts:** SLO, SLA, uptime, redundancy, failover
**Common Misunderstanding:** 99.9% sounds like "almost always up" but still allows 8.76 hours of downtime per year. Each additional "9" dramatically reduces allowed downtime.

### 5. Latency
**Definition:** The time it takes for a request to receive a response, typically measured in milliseconds.
**Example:** The GenAI chat API has p50 latency of 800ms, p95 of 1.2s, and p99 of 3.0s — meaning 95% of responses arrive within 1.2 seconds.
**Related Concepts:** Throughput, p50/p95/p99, tail latency, time to first token
**Common Misunderstanding:** Average latency is misleading. The p99 (tail latency) matters more because it represents the worst experience users have.

### 6. Throughput
**Definition:** The number of operations a system can handle per unit of time (requests per second, queries per minute).
**Example:** The GenAI platform handles 200 queries per minute at peak, with each query taking ~1 second to process, requiring ~4 concurrent processing slots.
**Related Concepts:** Latency, concurrency, Little's Law, capacity planning
**Common Misunderstanding:** Throughput and latency are related but distinct. You can have high throughput with high latency (processing many slow requests) or low throughput with low latency (processing few fast requests).

### 7. Consistency
**Definition:** In distributed systems, whether all nodes see the same data at the same time.
**Example:** The GenAI audit log uses strong consistency — once an event is written, all readers see it immediately. The response cache uses eventual consistency — a cached response may be stale for up to 7 days.
**Related Concepts:** CAP theorem, eventual consistency, strong consistency, replication
**Common Misunderstanding:** Consistency is not binary. There's a spectrum from strong (all nodes agree instantly) to eventual (nodes converge over time) and many levels in between.

### 8. Partitioning (Sharding)
**Definition:** Splitting data across multiple database instances based on a key, so each instance handles a subset of the data.
**Example:** The GenAI audit logs are partitioned by month — each month's data is on a separate partition, making date-range queries faster and allowing old partitions to be archived.
**Related Concepts:** Sharding key, cross-shard queries, rebalancing, range partitioning
**Common Misunderstanding:** Sharding adds operational complexity. Queries that span shards are slower and more complex. Choose a sharding key that minimizes cross-shard queries.

### 9. Load Balancer
**Definition:** A component that distributes incoming requests across multiple backend servers to prevent any single server from being overwhelmed.
**Example:** The API Gateway acts as a load balancer, distributing GenAI chat requests across 4 chat service pods using round-robin distribution.
**Related Concepts:** Round-robin, least connections, health check, sticky sessions
**Common Misunderstanding:** Load balancers distribute load but don't guarantee even distribution. If requests have variable processing times, some servers may be busier than others.

### 10. CDN (Content Delivery Network)
**Definition:** A network of geographically distributed servers that cache and serve static content from locations close to users.
**Example:** The GenAI platform's static assets (JavaScript bundles, CSS, images) are served through a CDN, reducing load times for users in different regions from 500ms to 50ms.
**Related Concepts:** Edge caching, origin server, TTL, cache invalidation
**Common Misunderstanding:** CDNs are for static content, not dynamic API responses. You can't cache GenAI responses in a CDN because they're user-specific and dynamic.

### 11. Message Queue
**Definition:** A system that stores messages for asynchronous processing, decoupling producers from consumers.
**Example:** GenAI audit events are published to Kafka. Multiple consumers process them independently: one writes to the hot database, one archives to cold storage, one streams to security monitoring.
**Related Concepts:** Pub/sub, Kafka, RabbitMQ, consumer group, dead letter queue
**Common Misunderstanding:** Message queues don't guarantee ordering across partitions. If ordering matters, use a single partition or a partition key.

### 12. Eventual Consistency
**Definition:** A consistency model where updates will propagate to all nodes eventually, but reads may return stale data in the interim.
**Example:** The GenAI response cache is eventually consistent. When a policy document is updated, cached responses may serve stale data until the cache TTL expires or the cache is manually invalidated.
**Related Concepts:** Strong consistency, replication lag, CAP theorem, TTL
**Common Misunderstanding:** Eventual consistency doesn't mean "inconsistent." It means "temporarily inconsistent, but converges to consistent."

### 13. Idempotency
**Definition:** An operation that produces the same result no matter how many times it's executed.
**Example:** `DELETE /cache/{key}` is idempotent — deleting the same key twice has the same effect as deleting it once. This allows safe retries without side effects.
**Related Concepts:** Retry, safe methods, exactly-once, at-least-once
**Common Misunderstanding:** Idempotency is about the system state, not the response. A GET and a PUT can both be idempotent even though they return different data.

### 14. Backpressure
**Definition:** A mechanism where a system communicates its capacity limits to upstream components, signaling them to slow down.
**Example:** When the GenAI audit consumer can't keep up with the Kafka queue depth, it signals the producer to reduce the publishing rate, preventing memory exhaustion.
**Related Concepts:** Flow control, rate limiting, queue depth, overflow handling
**Common Misunderstanding:** Backpressure is reactive (responding to overload), while rate limiting is proactive (preventing overload).

### 15. Graceful Degradation
**Definition:** A system's ability to continue operating with reduced functionality when some components fail, rather than failing completely.
**Example:** When the LLM API is unavailable, the GenAI assistant serves cached responses for common queries and directs users to the manual policy portal for other queries.
**Related Concepts:** Fallback, circuit breaker, resilience, failover
**Common Misunderstanding:** Graceful degradation must be designed and tested. It doesn't happen automatically — you need explicit fallback paths for each dependency.

### 16. Rate Limiting
**Definition:** Controlling the rate of requests from a client to prevent abuse and ensure fair resource usage.
**Example:** Each GenAI user is limited to 100 queries per minute. Requests beyond this receive HTTP 429 with a `Retry-After` header.
**Related Concepts:** Token bucket, sliding window, throttling, quota
**Common Misunderstanding:** Rate limiting is typically per-client. Global rate limiting (total requests across all clients) protects the system but doesn't prevent one user from monopolizing resources.

### 17. Circuit Breaker
**Definition:** A pattern that stops sending requests to a failing service after consecutive failures, and periodically tests for recovery.
**Example:** After 5 consecutive LLM API failures, the circuit breaker opens. Subsequent requests immediately fail with a friendly error instead of waiting for timeouts. After 30 seconds, the breaker half-opens to test recovery.
**Related Concepts:** Timeout, retry, fallback, bulkhead
**Common Misunderstanding:** A circuit breaker is not a timeout. It actively prevents calls to failing services, while a timeout just limits how long each call waits.

### 18. Caching Strategy
**Definition:** The approach for storing and retrieving frequently-accessed data to reduce latency and backend load.
**Example:** The GenAI platform uses: Redis for query response caching (TTL: 7 days), browser cache for static assets (TTL: 1 year), and application-level cache for embeddings (TTL: until model changes).
**Related Concepts:** Cache invalidation, write-through, write-behind, cache stampede
**Common Misunderstanding:** There are only two hard things in computer science: cache invalidation and naming things. Caching introduces complexity — stale data, invalidation bugs, and consistency issues.

### 19. Single Point of Failure (SPOF)
**Definition:** A component whose failure would cause the entire system to fail.
**Example:** A single PostgreSQL instance for the GenAI audit database is a SPOF. If it goes down, all audit logging stops. The fix: add a replica with automatic failover.
**Related Concepts:** Redundancy, high availability, failover, disaster recovery
**Common Misunderstanding:** Redundancy eliminates SPOFs, but adds complexity. More components = more things that can fail. Design for failure at every level.

### 20. Distributed System
**Definition:** A system whose components are located on different networked computers that communicate and coordinate their actions by passing messages.
**Example:** The GenAI platform is a distributed system: API gateway, chat service, retriever service, vector database, LLM API, cache, and audit logger — all communicating over the network.
**Related Concepts:** Microservice, network partition, consensus, distributed tracing
**Common Misunderstanding:** Distributed systems are not "faster" — they're more complex. A single well-designed monolith can outperform a poorly-designed distributed system.

### 21. Consistent Hashing
**Definition:** A hashing technique that minimizes the number of keys that need to be remapped when the number of cache servers changes.
**Example:** The GenAI response cache uses consistent hashing to distribute queries across cache nodes. When a node is added or removed, only 1/N of the keys are remapped (vs. all keys with simple hashing).
**Related Concepts:** Hash ring, cache distribution, partitioning
**Common Misunderstanding:** Consistent hashing reduces cache churn during scaling but doesn't eliminate it. Some keys will always be remapped.

### 22. Leader Election
**Definition:** The process by which a distributed system selects one node as the coordinator/leader to handle coordination tasks.
**Example:** Among 3 GenAI indexer instances, one is elected as the leader to process document updates. If the leader fails, a new leader is elected from the remaining instances.
**Related Concepts:** Raft, Paxos, consensus, high availability
**Common Misunderstanding:** Leader election takes time (seconds). During the election, the system may be unable to process writes. Design for this window.

### 23. Two-Phase Commit (2PC)
**Definition:** A protocol for achieving atomic agreement across distributed systems, where all participants must agree to commit or all must abort.
**Example:** A cross-service transaction that updates both the user profile service and the audit service uses 2PC to ensure either both updates succeed or both fail.
**Related Concepts:** Distributed transaction, saga, atomic commit, consensus
**Common Misunderstanding:** PC blocks all participants until the coordinator decides, causing performance issues. The saga pattern is preferred in modern distributed systems.

### 24. Quorum
**Definition:** The minimum number of nodes that must agree on a value for it to be considered valid in a distributed system.
**Example:** In a 5-node GenAI database cluster, a write requires acknowledgment from at least 3 nodes (majority quorum) before being considered committed.
**Related Concepts:** Consensus, replication factor, CAP theorem, Raft
**Common Misunderstanding:** Quorum is not "all nodes." It's typically a majority (N/2 + 1). This allows the system to tolerate up to (N-1)/2 node failures.

### 25. Bloom Filter
**Definition:** A probabilistic data structure that tests whether an element is a member of a set — it can tell you "definitely not" or "probably yes" (with false positives).
**Example:** The GenAI cache uses a Bloom filter to quickly check if a query response is cached before querying Redis. If the filter says "no," it skips the Redis lookup entirely.
**Related Concepts:** Probabilistic data structure, false positive rate, cache optimization
**Common Misunderstanding:** Bloom filters can have false positives (saying an element is in the set when it's not) but never false negatives. They're useful for filtering out known misses.
