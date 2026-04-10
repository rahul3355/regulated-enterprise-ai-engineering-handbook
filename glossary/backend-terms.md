# Backend Engineering Terms

> Essential backend engineering terminology for GenAI platform engineers. Each term includes definition, banking/GenAI example, related concepts, and common misunderstandings.

## Glossary

### 1. API (Application Programming Interface)
**Definition:** A contract that defines how software components communicate, specifying endpoints, request/response formats, and behaviors.
**Example:** The GenAI platform exposes a REST API: `POST /v1/chat` accepts a query and returns an AI-generated response with citations.
**Related Concepts:** REST, gRPC, GraphQL, OpenAPI, contract
**Common Misunderstanding:** An API is not just a URL — it's the complete contract including methods, parameters, responses, error codes, and behavior guarantees.

### 2. REST (Representational State Transfer)
**Definition:** An architectural style for designing networked applications using standard HTTP methods, stateless interactions, and resource-based URLs.
**Example:** `GET /v1/users/{id}` retrieves a user, `POST /v1/queries` creates a new query, `DELETE /v1/cache/{key}` clears a cache entry.
**Related Concepts:** HTTP, resource, idempotency, stateless, HATEOAS
**Common Misunderstanding:** REST is not "any HTTP API." True REST uses resources (nouns), not actions (verbs), and follows specific constraints.

### 3. Idempotency
**Definition:** An operation that produces the same result regardless of how many times it's executed.
**Example:** `DELETE /v1/cache/{key}` is idempotent — deleting the same key twice returns the same result. `POST /v1/queries` is NOT idempotent — each call creates a new query.
**Related Concepts:** Safe methods, retry, exactly-once, at-least-once
**Common Misunderstanding:** Idempotency is about the result, not the response. An idempotent operation may return different status codes (200 vs 204) but the system state is the same.

### 4. Microservice
**Definition:** A small, independently deployable service that owns a specific business capability and communicates with other services via APIs.
**Example:** The GenAI platform consists of separate microservices: chat service, retriever service, audit service, and guardrails service — each independently deployable.
**Related Concepts:** Monolith, service mesh, API gateway, distributed system
**Common Misunderstanding:** Microservices are not inherently better than monoliths. They add operational complexity. Start with a monolith and split when needed.

### 5. API Gateway
**Definition:** A single entry point that routes, authenticates, rate-limits, and monitors requests to multiple backend services.
**Example:** All GenAI API traffic goes through the API gateway, which handles authentication, rate limiting per user, request logging, and routing to the appropriate backend service.
**Related Concepts:** Load balancer, reverse proxy, service mesh, ingress
**Common Misunderstanding:** An API gateway is not just a load balancer. It handles cross-cutting concerns (auth, rate limiting, transformation) that load balancers don't.

### 6. Rate Limiting
**Definition:** Controlling the number of requests a client can make within a time window to prevent abuse and ensure fair resource usage.
**Example:** Each employee is limited to 100 GenAI queries per minute. Requests beyond this limit receive a 429 response with a `Retry-After` header.
**Related Concepts:** Throttling, quota, sliding window, token bucket, leaky bucket
**Common Misunderstanding:** Rate limiting is not just about preventing abuse. It also protects backend services from overload and controls costs (especially for LLM API calls).

### 7. Circuit Breaker
**Definition:** A pattern that prevents cascading failures by stopping calls to a failing service after a threshold of failures, and periodically testing for recovery.
**Example:** When the LLM API starts returning 503 errors, the circuit breaker opens after 5 consecutive failures, immediately returning an error to callers without waiting for timeouts. It half-opens after 30 seconds to test recovery.
**Related Concepts:** Timeout, retry, fallback, bulkhead, resilience
**Common Misunderstanding:** A circuit breaker is not a retry. It's the opposite — it STOPS retries to protect both the caller and the failing service.

### 8. Timeout
**Definition:** The maximum time a service will wait for a response before considering the request failed.
**Example:** The GenAI chat service sets a 30-second timeout for LLM API calls. If the LLM doesn't respond within 30 seconds, the request fails with a timeout error.
**Related Concepts:** Deadline, retry, circuit breaker, SLA, SLO
**Common Misunderstanding:** Timeouts should be set per-operation, not globally. A database query timeout (100ms) should be different from an LLM API timeout (30s).

### 9. Retry
**Definition:** Automatically re-attempting a failed operation, typically with exponential backoff to avoid overwhelming the failing service.
**Example:** If the vector database returns a transient error, the GenAI retriever retries with exponential backoff: 1s, 2s, 4s, 8s (max 4 retries).
**Related Concepts:** Exponential backoff, jitter, idempotency, circuit breaker
**Common Misunderstanding:** Retries should only be used for transient errors (network timeouts, 503s), not permanent errors (400, 401, 404). Retrying a 400 will never succeed.

### 10. Load Balancing
**Definition:** Distributing incoming requests across multiple service instances to ensure no single instance is overwhelmed.
**Example:** The API gateway distributes GenAI chat requests across 4 chat service pods using round-robin load balancing.
**Related Concepts:** Health check, sticky sessions, least connections, weighted routing
**Common Misunderstanding:** Load balancing distributes load, but doesn't guarantee even load. If requests have variable processing times, some instances may be busier than others.

### 11. Connection Pooling
**Definition:** Reusing existing database/network connections instead of creating new ones for each request, reducing overhead.
**Example:** The GenAI service maintains a pool of 20 PostgreSQL connections that are reused across requests, instead of opening a new connection for each query.
**Related Concepts:** Connection leak, max connections, keepalive
**Common Misunderstanding:** Connection pools are not unlimited. If the pool is exhausted, requests wait for an available connection, potentially causing cascading delays.

### 12. Middleware
**Definition:** Software that sits between the request and the handler, processing requests before they reach the handler and/or responses before they're returned.
**Example:** The GenAI chat API uses middleware for: authentication (validate API key), rate limiting (check quota), request logging (record metadata), and error handling (catch and format exceptions).
**Related Concepts:** Interceptor, filter, decorator, pipeline
**Common Misunderstanding:** Middleware runs in a specific order. The order matters — authentication should run before rate limiting (so you can rate limit per user, not per IP).

### 13. Async/Await
**Definition:** A programming pattern for writing non-blocking code that can handle multiple operations concurrently without dedicated threads for each.
**Example:** The GenAI chat endpoint uses `async def` to handle many concurrent requests without blocking on I/O operations (database queries, LLM API calls).
**Related Concepts:** Event loop, coroutine, concurrency, threading, multiprocessing
**Common Misunderstanding:** Async doesn't make code faster for CPU-bound operations. It helps with I/O-bound operations (network, database) where the code spends time waiting.

### 14. Event-Driven Architecture
**Definition:** A design pattern where services communicate through events (asynchronous messages) rather than direct synchronous calls.
**Example:** When a document is updated, the document management system publishes an event. The GenAI indexer consumes this event and updates the vector database — without the document system waiting for a response.
**Related Concepts:** Message queue, pub/sub, event bus, Kafka, async
**Common Misunderstanding:** Event-driven is not "fire and forget." Events still need to be processed, and failures need to be handled (dead letter queues, retries).

### 15. Message Queue
**Definition:** A system that stores messages (tasks, events) for asynchronous processing by consumers.
**Example:** GenAI audit events are published to a Kafka queue. Multiple consumers process them: one writes to the hot database, one archives to S3, one streams to the security monitoring system.
**Related Concepts:** Kafka, RabbitMQ, SQS, pub/sub, consumer group
**Common Misunderstanding:** Message queues don't guarantee exactly-once delivery. They typically guarantee at-least-once, meaning consumers must be idempotent.

### 16. Cache
**Definition:** A fast-access storage layer that stores frequently-used data to reduce latency and backend load.
**Example:** Frequently-asked GenAI queries are cached in Redis. When the same query comes in again, the cached response is returned in 5ms instead of calling the LLM API (1.2s).
**Related Concepts:** TTL, cache invalidation, cache stampede, write-through, write-behind
**Common Misunderstanding:** Caching introduces complexity: stale data, invalidation challenges, and consistency issues. Not every benefit outweighs the cost.

### 17. Graceful Degradation
**Definition:** A system's ability to continue operating with reduced functionality when some components fail, rather than failing completely.
**Example:** When the LLM API is down, the GenAI assistant returns cached responses for common queries and a helpful message directing users to the policy portal for other queries.
**Related Concepts:** Fallback, circuit breaker, resilience, SLO
**Common Misunderstanding:** Graceful degradation must be designed and tested. It doesn't happen automatically. You need fallback paths for every critical dependency.

### 18. Health Check
**Definition:** An endpoint or mechanism that reports whether a service is operational and ready to handle requests.
**Example:** The GenAI chat service exposes `/health` (basic: is the process running?) and `/healthz` (deep: can it connect to the database, vector DB, and Redis?).
**Related Concepts:** Readiness probe, liveness probe, startup probe
**Common Misunderstanding:** A health check that always returns 200 is useless. It should actually verify dependencies and return 503 when something is wrong.

### 19. Distributed Tracing
**Definition:** Tracking a request as it flows through multiple services, providing visibility into the entire request path and timing.
**Example:** A GenAI query is traced from the API gateway → chat service → retriever → vector DB → LLM API → guardrails → response. The trace shows latency at each step.
**Related Concepts:** OpenTelemetry, span, trace ID, Jaeger, observability
**Common Misunderstanding:** Distributed tracing is not logging. Logs tell you what happened at one point. Traces show you the flow across services.

### 20. Backpressure
**Definition:** A mechanism where a system signals upstream components to slow down when it cannot keep up with incoming load.
**Example:** When the GenAI audit consumer can't keep up with the Kafka queue depth, it signals the producer to slow down, preventing unbounded memory growth.
**Related Concepts:** Flow control, queue depth, overflow, shedding
**Common Misunderstanding:** Backpressure is not the same as rate limiting. Rate limiting prevents load from entering. Backpressure communicates capacity constraints when load is already in the system.

### 21. Bulkhead Pattern
**Definition:** Isolating resources so that a failure in one area doesn't consume resources and affect other areas.
**Example:** The GenAI platform uses separate connection pools for read queries and write queries. If read queries exhaust their pool, write queries (like audit logging) are unaffected.
**Related Concepts:** Isolation, circuit breaker, resource partitioning
**Common Misunderstanding:** Bulkheads reduce overall resource utilization efficiency (some resources sit idle in one partition while another is exhausted) in exchange for isolation.

### 22. Saga Pattern
**Definition:** A pattern for managing distributed transactions across multiple services using a sequence of local transactions with compensating actions for rollback.
**Example:** Creating a new GenAI user involves: (1) creating the user account, (2) provisioning API keys, (3) setting up audit logging. If step 3 fails, steps 1 and 2 are compensated (rolled back).
**Related Concepts:** Distributed transaction, two-phase commit, eventual consistency
**Common Misunderstanding:** Sagas are complex to implement correctly. For simple cases, a single-service transaction is preferable. Use sagas only when data spans multiple services.

### 23. Idempotency Key
**Definition:** A unique key provided by the client that the server uses to ensure an operation is executed only once, even if the client retries.
**Example:** The GenAI API accepts an `X-Idempotency-Key` header. If the client retries a query submission with the same key, the server returns the cached result instead of creating a duplicate.
**Related Concepts:** Idempotency, retry, deduplication
**Common Misunderstanding:** The idempotency key must be unique per intended operation. Reusing the same key for different requests causes data loss.

### 24. Service Discovery
**Definition:** The process by which services dynamically find the network locations of other services in a distributed system.
**Example:** The GenAI chat service discovers the retriever service's address through Kubernetes DNS (`genai-retriever.genai.svc.cluster.local`) rather than hardcoding an IP address.
**Related Concepts:** DNS, service registry, sidecar proxy, service mesh
**Common Misunderstanding:** In Kubernetes, service discovery is built-in (DNS). You don't need Consul or Eureka unless you're in a non-Kubernetes environment.

### 25. API Versioning
**Definition:** The practice of maintaining multiple versions of an API simultaneously to avoid breaking existing clients.
**Example:** The GenAI platform maintains both `/v1/chat` and `/v2/chat`. V2 adds streaming support and enhanced citations. V1 clients continue working until they migrate to v2.
**Related Concepts:** Backward compatibility, deprecation, breaking change, contract
**Common Misunderstanding:** API versioning in the URL path (`/v1/`, `/v2/`) is the simplest approach but not the only one. Header-based and content-negotiation versioning are alternatives.
