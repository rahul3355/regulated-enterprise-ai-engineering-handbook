# Skill: Distributed System Debugging

## Core Principles

1. **Follow the Request** — In a distributed system, a single user request flows through multiple services. Trace IDs are your compass. Without them, you're blind.
2. **Assume Failure Everywhere** — Every network call, every database query, every cache lookup can fail. Design for failure, then debug from the assumption that something did.
3. **Correlate, Don't Isolate** — A bug in service A may be caused by service B returning bad data, which was caused by service C having a stale cache. Debug the chain, not individual services.
4. **Time Is Relative** — Clock skew between services makes ordering events difficult. Use logical clocks (Lamport timestamps) or rely on trace IDs for ordering.
5. **Start with the Symptom, Work Backward** — User reports a slow query. Start from the API response, trace through each hop, find the slow component. Don't start from the database and work forward.

## Mental Models

### The Distributed Request Flow
```
User ──▶ API Gateway ──▶ Auth Service ──▶ RAG Service ──▶ Vector DB
                              │                  │
                              │                  ├──▶ Embedding API
                              │                  │
                              │                  └──▶ Document Store
                              │
                              └──▶ Rate Limiter
                                        │
                                        └──▶ Redis Cache

Each arrow is a network call that can fail, timeout, or return bad data.
```

### The Debugging Decision Tree
```
Symptom: Request Failed
        │
        ▼
┌───────────────────────────┐
│ Do you have a trace ID?   │
└──────────┬────────────────┘
           │
     ┌─────┴──────┐
     Yes          No
     │            │
     ▼            ▼
┌──────────┐  ┌──────────────┐
│Follow the│  │Add trace ID  │
│trace across│ │support first,│
│all services│ │reproduce     │
└─────┬────┘  └──────────────┘
      │
      ▼
┌───────────────────────────┐
│ Which service failed?     │
│ Check its logs + metrics  │
└──────────┬────────────────┘
           │
     ┌─────┼─────┬──────────┐
     ▼     ▼     ▼          ▼
  Network  Config  Resource  Code
  Issue    Issue   Issue    Bug
  │        │       │        │
  ▼        ▼       ▼        ▼
 Check   Check    Check    Check
 DNS,    env vars, CPU,    error
 TLS,    secrets, memory,  logs,
 firewall DB conn, disk    stack
         status   I/O      trace
```

### The Distributed Debugging Checklist
```
□ Request has a unique trace ID (propagated across all services)
□ Each service logs the trace ID on entry and exit
□ Each service logs request duration
□ Each service logs downstream call duration
□ Centralized log aggregation (Splunk, Loki, ELK)
□ Distributed tracing system (Jaeger, Zipkin, OpenTelemetry)
□ Service dependency map documented and current
□ Circuit breaker status visible for all downstream dependencies
□ Rate limiter status visible (throttled vs. allowed)
□ Health check endpoints for each service
□ Service version visible in logs and health endpoints
□ Clock synchronization across all nodes (NTP)
```

## Step-by-Step Approach

### 1. Set Up Distributed Tracing with OpenTelemetry

```python
# Python service with OpenTelemetry
from opentelemetry import trace, context
from opentelemetry.trace import format_trace_id, format_span_id
from opentelemetry.propagate import inject
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
import httpx
import logging

# Configure tracer
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer("rag-service")

# Export to Jaeger/collector
exporter = OTLPSpanExporter(endpoint="otel-collector.observability.svc:4317")
span_processor = BatchSpanProcessor(exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Configure logging to include trace IDs
class TraceIdFilter(logging.Filter):
    def filter(self, record):
        span = trace.get_current_span()
        record.trace_id = format_trace_id(span.get_span_context().trace_id)
        record.span_id = format_span_id(span.get_span_context().span_id)
        return True

handler = logging.StreamHandler()
handler.addFilter(TraceIdFilter())
logging.basicConfig(handlers=[handler], format='%(trace_id)s %(span_id)s %(message)s')

# Usage in a request handler
@app.post("/api/chat")
async def chat(request: ChatRequest):
    with tracer.start_as_current_span("chat_request") as span:
        span.set_attribute("user_id", request.user_id)
        span.set_attribute("message_length", len(request.message))

        # Step 1: Validate input
        with tracer.start_as_current_span("validate_input"):
            validated = await validate_request(request)

        # Step 2: Retrieve documents (RAG)
        with tracer.start_as_current_span("retrieve_documents") as retrieval_span:
            docs = await retrieve_documents(validated.message)
            retrieval_span.set_attribute("documents_retrieved", len(docs))

        # Step 3: Call LLM (downstream service)
        with tracer.start_as_current_span("call_llm") as llm_span:
            # Propagate trace context to downstream call
            headers = {}
            inject(headers)  # Adds traceparent header
            response = await httpx.AsyncClient().post(
                "http://model-gateway.inference.svc/v1/chat",
                json={"messages": build_messages(validated.message, docs)},
                headers=headers,
            )
            llm_span.set_attribute("llm_response_time_ms", response.elapsed.total_seconds() * 1000)

        # Step 4: Apply guardrails
        with tracer.start_as_current_span("apply_guardrails"):
            result = await guardrail_pipeline(response.json())

        span.set_attribute("total_duration_ms", time.time() * 1000)
        return result
```

### 2. Trace Context Propagation in Go (Downstream Service)

```go
// Go service receiving propagated trace context
package main

import (
    "context"
    "net/http"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

var tracer = otel.Tracer("model-gateway")

func chatHandler(w http.ResponseWriter, r *http.Request) {
    // Extract trace context from incoming headers
    ctx := otel.GetTextMapPropagator().Extract(r.Context(), propagation.HeaderCarrier(r.Header))

    // Create a new span in the existing trace
    ctx, span := tracer.Start(ctx, "chat_completion")
    defer span.End()

    span.SetAttributes(
        attribute.String("model", "gpt-4o"),
        attribute.Int("prompt_tokens", promptTokens),
        attribute.Int("completion_tokens", completionTokens),
    )

    // Process the request...
    response, err := processChat(ctx, request)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    writeJSON(w, response)
}

func main() {
    // Set up global propagator
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},  // W3C Trace Context
        propagation.Baggage{},       // W3C Baggage
    ))

    // Wrap handler with OpenTelemetry middleware
    handler := otelhttp.NewHandler(http.HandlerFunc(chatHandler), "/v1/chat")
    http.Handle("/v1/chat", handler)
    http.ListenAndServe(":8080", nil)
}
```

### 3. Debugging with Jaeger UI

```bash
# Start Jaeger for local debugging
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.53

# Access Jaeger UI: http://localhost:16686

# Search for traces:
# 1. Select service: rag-service
# 2. Select operation: POST /api/chat
# 3. Filter by: duration > 2s (find slow requests)
# 4. Look at the trace waterfall view

# In the trace view, you'll see:
# chat_request [===========================] 2340ms
#   ├─ validate_input [===] 12ms
#   ├─ retrieve_documents [=======] 450ms
#   │   ├─ generate_embedding [==] 120ms
#   │   └─ vector_db_query [===] 280ms
#   ├─ call_llm [===============] 1800ms  <-- THE SLOW ONE
#   └─ apply_guardrails [=] 78ms
```

### 4. Debugging a Timeout Cascade

```
Scenario: Users report the chat API is returning 504 Gateway Timeout.

Step 1: Check the API Gateway logs
────────────────────────────────────────
2025-01-15T14:30:00Z trace_id=abc123 method=POST path=/api/chat status=504
  duration=30001ms upstream_timeout

The API Gateway has a 30-second timeout. The upstream (RAG service) didn't respond in time.

Step 2: Check the RAG service logs for the same trace
────────────────────────────────────────
2025-01-15T14:29:32Z trace_id=abc123 span=chat_request
  → POST /api/chat user_id=emp-12345
2025-01-15T14:29:32Z trace_id=abc123 span=validate_input duration=15ms
2025-01-15T14:29:32Z trace_id=abc123 span=retrieve_documents duration=480ms
2025-01-15T14:29:33Z trace_id=abc123 span=call_llm
  → POST http://model-gateway.inference.svc/v1/chat
  → timeout=25000ms
  ... [no response for 25 seconds] ...
2025-01-15T14:29:58Z trace_id=abc123 span=call_llm ERROR
  → context deadline exceeded (timeout after 25s)

The RAG service has a 25-second timeout for the LLM call. It timed out.

Step 3: Check the model gateway logs
────────────────────────────────────────
2025-01-15T14:29:33Z trace_id=abc123 span=chat_completion
  → POST /v1/chat model=gpt-4o
2025-01-15T14:29:33Z trace_id=abc123 span=queue_wait
  → queue_depth=450 wait_time=18000ms  <-- Queue is backed up!
2025-01-15T14:29:51Z trace_id=abc123 span=model_inference
  → duration=6000ms  (model itself is fine)

The model inference took 6 seconds, but the request waited 18 seconds in queue.
Total: 18s (queue) + 6s (inference) = 24s, which is under the 25s timeout.
But combined with the RAG service overhead, it exceeds the API Gateway's 30s.

Root cause: Model gateway queue depth is too high.
Fix: Scale up model gateway replicas, or reduce timeout to fail fast.
```

### 5. Debugging a Circuit Breaker Trip

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30000, name="model_gateway")
async def call_model_gateway(messages: list[dict]) -> dict:
    """Calls the model gateway with circuit breaker protection."""
    async with httpx.AsyncClient(timeout=25.0) as client:
        response = await client.post(
            "http://model-gateway.inference.svc/v1/chat",
            json={"messages": messages},
        )
        response.raise_for_status()
        return response.json()

# When the circuit breaker trips:
# 1. After 5 consecutive failures, the circuit opens
# 2. All subsequent calls fail immediately (no downstream call)
# 3. After 30 seconds, the circuit half-opens (allows one test call)
# 4. If the test call succeeds, the circuit closes
# 5. If the test call fails, the circuit opens again

# Monitoring circuit breaker state
@app.get("/health/circuit-breakers")
async def get_circuit_breaker_status():
    return {
        "model_gateway": {
            "state": circuit.states.model_gateway.name,  # 'closed', 'open', 'half-open'
            "failure_count": circuit.failures.model_gateway,
            "last_failure": circuit.last_failure.model_gateway,
        }
    }
```

### 6. Debugging with Service Dependency Map

```python
# Generate a service dependency report
import json
from collections import defaultdict

# Parse logs to build a dependency map
def build_dependency_map(logs: list[dict]) -> dict:
    dependencies = defaultdict(lambda: defaultdict(int))

    for log in logs:
        if log.get("span_kind") == "client":
            caller = log.get("service_name")
            callee = log.get("peer.service")
            if caller and callee:
                dependencies[caller][callee] += 1

    return dict(dependencies)

# Example output
dependency_map = {
    "rag-service": {
        "vector-db": 15000,
        "model-gateway": 14500,
        "redis-cache": 12000,
        "auth-service": 15000,
    },
    "model-gateway": {
        "openai-api": 14500,
        "rate-limiter": 14500,
    },
    "vector-db": {
        "postgres": 15000,
        "embedding-api": 3000,  # Only for new documents
    },
}
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| No trace IDs in logs | Can't correlate events across services | Propagate trace ID through all services |
| Inconsistent timestamp formats | Can't order events across services | Use ISO 8601 with timezone, or epoch milliseconds |
| Clock skew between nodes | Events appear out of order | Sync clocks with NTP, use trace IDs for ordering |
| Missing timeout on downstream calls | Requests hang indefinitely | Set timeout on every external call |
| No circuit breaker | Cascading failures take down all services | Implement circuit breakers for all downstream calls |
| Not logging request duration | Can't identify slow services | Log duration at every service entry/exit |
| Swallowing errors in middleware | Errors disappear without a trace | Log all errors with full context |
| Retry storms | One failed service gets hammered by retries from all callers | Use exponential backoff with jitter, limit retry count |
| Debugging in production without care | Debugging changes make the problem worse | Use read-only debugging (logs, traces) first |
| Assuming the network is reliable | Intermittent failures are treated as "flukes" | Treat all network failures as real bugs |

## Banking-Specific Concerns

1. **Cross-Cluster Communication** — Banking services often span multiple clusters (production, DR, inference). Debug by checking network policies, service mesh (Istio) configuration, and cross-cluster DNS resolution.
2. **Data Sovereignty** — Some services must not communicate across geographic regions. A "timeout" might actually be a firewall rule blocking cross-region traffic.
3. **Compliance Logging** — Every inter-service communication involving customer data must be logged for audit. Missing logs are a compliance finding.
4. **Legacy System Integration** — GenAI services may call mainframe or legacy systems via APIs. These systems have different reliability characteristics. Implement circuit breakers and timeouts specifically for legacy integrations.
5. **Incident Response Coordination** — A distributed failure may involve 5+ teams. Use a shared incident channel and a single incident commander. Each team debugs their service in parallel.

## GenAI-Specific Concerns

1. **External Model API Dependency** — Calls to OpenAI, Anthropic, or other external model APIs are the most common failure point. Monitor rate limits, latency, and error rates from these providers.
2. **Embedding Pipeline Latency** — Embedding large documents is slow and resource-intensive. If the embedding pipeline falls behind, the RAG system serves stale data. Monitor embedding queue depth and processing rate.
3. **Non-Deterministic Behavior** — LLMs may produce different outputs for the same input. This makes debugging "intermittent" issues harder. Log the exact prompt, model, and parameters used for each request.
4. **Token Budget Exhaustion** — If the daily token budget is exceeded, the model API starts rejecting requests. Monitor token usage and set up alerts at 80% of budget.
5. **Conversation Context Growth** — Long conversations accumulate context, increasing token usage and latency. Monitor conversation length and implement context window management (summarization, truncation).

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| End-to-end request latency (p99) | > 5s | User-perceived slowness |
| Inter-service call latency (p99) | > 1s | Downstream service degradation |
| Error rate per service | > 1% | Service health degradation |
| Circuit breaker trip count | > 0 per hour | Downstream service instability |
| Trace ID coverage | < 95% of requests | Blind spots in debugging |
| Retry rate | > 5% of requests | Downstream service flakiness |
| Timeout rate | > 1% of requests | Timeout configuration too aggressive or service slow |
| Queue depth (async services) | > 80% of max | Processing backlog |
| Cross-cluster latency | > 100ms | Network issues between clusters |
| Token usage rate | > 80% of daily budget | Approaching API rate limit |

## Interview Questions

1. A user reports that the GenAI assistant is returning 504 errors. Walk me through your debugging process.
2. How do you propagate trace IDs across multiple services in different languages (Python, Go, TypeScript)?
3. What is a circuit breaker and when would you use one in a GenAI platform?
4. How would you debug an issue where service A works fine in isolation but fails when called by service B?
5. What is the difference between a timeout and a circuit breaker? When does each help and when does each hurt?
6. How do you handle clock skew when debugging distributed systems?
7. Explain how retry storms happen and how to prevent them.
8. A GenAI service works for 99% of users but fails for 1%. How do you debug this?

## Hands-On Exercise

### Exercise: Debug a Cascading Failure in a GenAI Platform

**Problem:** At 2:30 PM, users start reporting that the GenAI assistant is returning errors and timing out. Within 15 minutes, the risk management team escalates this as a P1 incident. The platform consists of:

- API Gateway (HAProxy on OpenShift Routes)
- Auth Service (Go) — validates tokens
- RAG Service (Python/FastAPI) — orchestrates retrieval and LLM calls
- Model Gateway (Go) — routes to different LLM providers
- Vector DB (Postgres with pgvector) — document retrieval
- Embedding Pipeline (Python/Celery) — async document embedding
- Redis Cache — response caching

**Scenario:**
- The embedding pipeline was running a batch job to re-embed 50,000 documents with a new model
- The batch job consumed all available database connections
- The RAG service couldn't get database connections for retrieval queries
- The RAG service timed out waiting for the database
- Users saw 504 Gateway Timeout
- The API Gateway's connection pool to the RAG service filled up
- New requests couldn't connect to the API Gateway

**Constraints:**
- You have access to all service logs via Splunk
- You have access to Jaeger for distributed tracing
- You have access to OpenShift for checking pod status
- You have access to Postgres for checking connection status
- You cannot restart services without understanding the root cause

**Expected Output:**
- A timeline of events reconstructed from logs and traces
- Identification of the root cause (embedding batch job consuming all DB connections)
- Identification of the cascade (DB connection exhaustion → RAG timeout → API Gateway pool exhaustion)
- Immediate fix to restore service
- Long-term fix to prevent recurrence

**Hints:**
- Check the Postgres connection count: `SELECT count(*) FROM pg_stat_activity`
- Check which services are holding connections
- Look at the embedding pipeline logs to see if it's running a batch job
- Check the RAG service connection pool configuration
- Look at the API Gateway connection pool status

**Extension:**
- Design a connection pool management system that prevents this cascade
- Implement a circuit breaker between the RAG service and Vector DB
- Create a runbook for this specific failure mode
- Design a load-shedding mechanism that protects retrieval queries during batch processing

---

**Related files:**
- `observability/distributed-tracing.md` — OpenTelemetry and Jaeger
- `incident-management/incident-response.md` — Incident response process
- `skills/kubernetes-debugging.md` — Kubernetes debugging
- `skills/incident-triage.md` — Incident triage procedures
- `databases/postgres-connection-management.md` — Connection pooling
- `genai-platforms/rag-architecture.md` — RAG system design
