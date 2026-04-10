# Distributed Tracing in Banking GenAI Systems

## What Is Distributed Tracing?

Distributed tracing follows a single request as it flows through all services in a system. Unlike logs (which are discrete events) and metrics (which are aggregates), traces show the complete path and timing of an individual request across service boundaries.

In a banking GenAI platform, a single customer query traverses multiple services:

```
Customer ──> API Gateway ──> Auth Service ──> RAG Pipeline ──> LLM Gateway ──> Response
                     │              │              │                │
                     │              │              ├──> Vector DB   │
                     │              │              └──> Doc Store   └──> External LLM
                     │              │
                     └──────────────┴─── Trace shows all of this
```

Without distributed tracing, each service generates its own logs. You see 15 separate log entries but cannot determine their order, which call was slow, or which service failed. With tracing, you see a single timeline showing exactly where time was spent.

## Trace Anatomy

A trace consists of:

1. **Trace**: The complete request lifecycle, identified by a 128-bit trace ID
2. **Span**: A single operation within a trace, identified by a 64-bit span ID
3. **Parent-Child Relationships**: Spans form a tree structure showing call hierarchy
4. **Span Attributes**: Key-value metadata attached to each span
5. **Span Events**: Timestamped annotations within a span
6. **Span Status**: OK, ERROR, or UNSET

### Trace Structure

```
Trace: 4bf92f3577b34da6a3ce929d0e0e4736
│
├─ Span 1: API Gateway - POST /api/v1/chat [2,847ms] OK
│  ├─ Span 2: Auth Service - validate_token [23ms] OK
│  ├─ Span 3: RAG Pipeline - retrieve_context [456ms] OK
│  │  ├─ Span 4: Embedding Service - generate_embedding [89ms] OK
│  │  ├─ Span 5: Vector DB - similarity_search [123ms] OK
│  │  └─ Span 6: Reranker - rank_documents [67ms] OK
│  ├─ Span 7: LLM Gateway - generate_response [2,234ms] OK
│  │  ├─ Span 8: Prompt Builder - assemble_prompt [12ms] OK
│  │  ├─ Span 9: Cache Check - lookup_cache [5ms] OK (miss)
│  │  └─ Span 10: External LLM - openai_completion [2,189ms] OK
│  └─ Span 11: Response Formatter - format_response [34ms] OK
└─ End Trace
```

This trace immediately reveals: 78% of time was spent in the external LLM call, 16% in RAG retrieval, and 6% in everything else.

## Context Propagation

The critical requirement for distributed tracing is that every service in the call chain must propagate the trace context. Without propagation, the trace is broken.

### W3C Trace Context Header

Use the W3C Trace Context standard for HTTP propagation:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-5bd639e71c2a4f89-01
             │  │                                │               │  │
             │  │                                │               │  └─ sampled (01=yes)
             │  │                                │               └─ current span ID
             │  │                                └─ trace ID
             └─ version (00)
```

### Manual Propagation (Python/FastAPI)

```python
from opentelemetry import trace, context
from opentelemetry.propagate import inject, extract
from opentelemetry.trace import format_trace_id, format_span_id
import httpx

tracer = trace.get_tracer(__name__)

@app.post("/api/v1/chat")
async def chat(request: ChatRequest, http_request: Request):
    # Extract trace context from incoming headers
    incoming_ctx = extract(http_request.headers)
    if incoming_ctx:
        context.attach(incoming_ctx)

    with tracer.start_as_current_span("chat_handler") as span:
        span.set_attribute("endpoint", "/api/v1/chat")
        span.set_attribute("user_tier", request.user_tier)

        # Call downstream service - inject trace context
        headers = {}
        inject(headers)
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "http://rag-pipeline/retrieve",
                json={"query": request.query},
                headers=headers  # Trace context travels in headers
            )

        return response.json()
```

### Banking-Specific Propagation Rules

1. **Always extract first, then start span**: If no incoming trace context exists, start a new trace. If it exists, join the existing trace.

2. **Include correlation ID in span attributes**: Store your correlation ID as a span attribute for cross-referencing with logs:

```python
span.set_attribute("correlation_id", correlation_id)
```

3. **Propagate through message queues**: When using Kafka or RabbitMQ for async processing, embed trace context in message headers:

```python
from opentelemetry.propagate import inject

# When producing a message
headers = {}
inject(headers)
producer.send("document-processing", value=message, headers=headers)

# When consuming a message
ctx = extract(message.headers)
if ctx:
    context.attach(ctx)
```

## Span Attributes for Banking GenAI

Attach domain-relevant attributes to spans for meaningful querying:

### HTTP Spans

```python
span.set_attribute("http.method", "POST")
span.set_attribute("http.url", "/api/v1/mortgage-advice")
span.set_attribute("http.status_code", 200)
span.set_attribute("http.request_content_length", 1245)
span.set_attribute("http.response_content_length", 3892)
span.set_attribute("net.peer.name", "loan-advisor-api")
span.set_attribute("net.peer.port", 8443)
```

### LLM Call Spans

```python
span.set_attribute("gen_ai.system", "openai")
span.set_attribute("gen_ai.request.model", "gpt-4-turbo")
span.set_attribute("gen_ai.request.temperature", 0.1)
span.set_attribute("gen_ai.request.max_tokens", 2048)
span.set_attribute("gen_ai.response.finish_reason", "stop")
span.set_attribute("gen_ai.usage.input_tokens", 1245)
span.set_attribute("gen_ai.usage.output_tokens", 456)
span.set_attribute("gen_ai.operation.name", "chat")
span.set_attribute("banking.product_line", "mortgage")
span.set_attribute("banking.compliance.review_required", True)
```

### Database Spans

```python
span.set_attribute("db.system", "postgresql")
span.set_attribute("db.operation", "SELECT")
span.set_attribute("db.statement", "SELECT * FROM customer_profiles WHERE id = ?")
span.set_attribute("db.row_count", 1)
span.set_attribute("db.connection_pool.available", 8)
```

## Querying Traces

Traces enable investigative queries that are impossible with logs or metrics alone:

### Find Slow Requests

Query: "Show me all traces where the LLM call took more than 5 seconds"

```
service:llm-gateway AND span:openai_completion AND duration_ms:>5000
```

### Trace Error Chains

Query: "Show me the full trace for all requests that resulted in a 503"

```
http.status_code:503 | show full trace tree
```

### Cross-Service Latency

Query: "What percentage of total request time is spent in RAG retrieval?"

```
span:retrieve_context.duration_ms / root_span.duration_ms
| avg by endpoint | sort desc
```

## Sampling

In high-traffic banking systems, storing every trace is expensive. Use sampling strategies:

### Head-Based Sampling

Decide at trace start whether to sample:

```python
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased

# Sample 10% of all traces
sampler = TraceIdRatioBased(0.1)
```

### Tail-Based Sampling

Decide after the trace completes:

```python
from opentelemetry.contrib.sampler import TailSamplingSampler

# Always sample errors, sample 1% of successes
sampler = TailSamplingSampler(
    rules=[
        Rule(condition="status == ERROR", sample_rate=1.0),
        Rule(condition="duration_ms > 5000", sample_rate=1.0),
        Rule(default=True, sample_rate=0.01),
    ]
)
```

**Banking recommendation**: Always sample errors. Sample 100% of traces containing regulatory-relevant operations (loan decisions, compliance checks). Sample 1-5% of normal traffic.

## Trace Visualization

```
Service Map (auto-generated from traces):

                    ┌──────────────┐
                    │   Client     │
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │ API Gateway  │
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
     ┌──────┴──────┐ ┌─────┴─────┐ ┌─────┴─────┐
     │Auth Service │ │RAG Pipeline│ │LLM Gateway│
     └─────────────┘ └─────┬──────┘ └─────┬──────┘
                           │              │
                  ┌────────┴──────┐  ┌────┴────────┐
                  │               │  │             │
           ┌──────┴──────┐ ┌─────┴──┴─┐ ┌────────┴────────┐
           │  Vector DB  │ │Doc Store │ │ External LLM    │
           └─────────────┘ └──────────┘ │ (OpenAI/Anthropic)│
                                        └─────────────────┘
```

Service maps are automatically generated from trace data. They show you the topology of your system and highlight high-latency or error-prone connections.

## Integration with Logs and Metrics

Traces do not replace logs or metrics -- they complement them:

```
┌───────────────────────────────────────────────────────┐
│                  Observability Triad                   │
│                                                       │
│  LOGS          METRICS          TRACES                │
│  "What"        "How much"       "How it flowed"       │
│                                                       │
│  correlation_id ─────────────────> trace_id            │
│  trace_id ──────────────────────> trace_id             │
│  span_id ───────────────────────> span_id              │
│                                                       │
│  From a slow metric alert, jump to representative     │
│  traces. From a trace span, jump to correlated logs.  │
└───────────────────────────────────────────────────────┘
```

In your observability platform, clicking on a span in a trace view should show the corresponding log entries filtered by `trace_id` and `span_id`.

## Common Mistakes

1. **Not propagating context across service boundaries**: The #1 cause of broken traces. Use `inject/extract` at every HTTP call boundary.

2. **Creating spans for trivial operations**: Do not create a span for every function call. Create spans for operations that cross service boundaries or are meaningfully slow.

3. **Missing span attributes**: A span without attributes is just a timing. Always add relevant attributes.

4. **Not sampling errors**: If you sample at 1%, you will miss rare but critical errors. Always sample 100% of errors.

5. **Using trace ID as correlation ID**: While you CAN use the trace ID as your correlation ID, many banking systems prefer a shorter, human-readable correlation ID for support interactions. Map between them.
