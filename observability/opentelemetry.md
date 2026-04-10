# OpenTelemetry Setup for Banking GenAI Systems

## Overview

OpenTelemetry (OTel) is the industry-standard framework for collecting and exporting telemetry data (traces, metrics, logs). It is vendor-neutral, meaning you can switch observability backends without changing your instrumentation code.

In banking, vendor neutrality is important because:
- Regulatory requirements may mandate specific data residency (on-prem vs. cloud)
- Procurement processes may change observability vendors
- Multi-cloud strategies require consistent observability across providers

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Application                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              OpenTelemetry SDK                        │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │   │
│  │  │ Tracing  │  │ Metrics  │  │ Logging (logs)   │   │   │
│  │  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │   │
│  │       │              │                │             │   │
│  │       └──────────────┴────────────────┘             │   │
│  │                          │                           │   │
│  │                    Exporters                         │   │
│  └──────────────────────────┬───────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────┘
                              │ (OTLP Protocol)
                              │
                    ┌─────────┴─────────┐
                    │  OTel Collector   │
                    │  (sidecar or      │
                    │   daemonset)      │
                    └─────────┬─────────┘
                              │
                ┌─────────────┼─────────────┐
                │             │             │
         ┌──────┴──────┐ ┌───┴────┐ ┌──────┴──────┐
         │   Traces    │ │Metrics │ │    Logs     │
         │   Backend   │ │Backend │ │   Backend   │
         │  (Jaeger/   │ │(Prom/  │ │ (Elastic/   │
         │   Tempo)    │ │Cortex) │ │  Loki)      │
         └─────────────┘ └────────┘ └─────────────┘
```

## Python SDK Setup

### Installation

```bash
pip install opentelemetry-api \
            opentelemetry-sdk \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-fastapi \
            opentelemetry-instrumentation-httpx \
            opentelemetry-instrumentation-sqlalchemy \
            opentelemetry-instrumentation-redis \
            opentelemetry-exporter-prometheus
```

### Basic Configuration

```python
# otel_setup.py
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
import os

def setup_opentelemetry(service_name: str, version: str):
    """Configure OpenTelemetry SDK for the banking GenAI platform."""

    # Resource identifies this service in telemetry
    resource = Resource.create({
        "service.name": service_name,
        "service.version": version,
        "deployment.environment": os.getenv("ENVIRONMENT", "production"),
        "cloud.region": os.getenv("AWS_REGION", "us-east-1"),
    })

    # --- Tracing ---
    tracer_provider = TracerProvider(resource=resource)

    # Export traces to OTel Collector via OTLP
    otlp_exporter = OTLPSpanExporter(
        endpoint=os.getenv("OTEL_COLLECTOR_ENDPOINT", "localhost:4317"),
        insecure=True  # Set to False in production with TLS
    )
    span_processor = BatchSpanProcessor(otlp_exporter)
    tracer_provider.add_span_processor(span_processor)
    trace.set_tracer_provider(tracer_provider)

    # --- Metrics ---
    metric_exporter = OTLPMetricExporter(
        endpoint=os.getenv("OTEL_COLLECTOR_ENDPOINT", "localhost:4317"),
        insecure=True
    )
    meter_provider = MeterProvider(resource=resource)
    metrics.set_meter_provider(meter_provider)

    # --- Auto-instrumentation ---
    FastAPIInstrumentor.instrument_app(app)
    HTTPXClientInstrumentor().instrument()
    # Add other instrumentations as needed

    return trace.get_tracer(service_name)
```

### FastAPI Integration

```python
from fastapi import FastAPI
from otel_setup import setup_opentelemetry

app = FastAPI(title="Loan Advisor API")

tracer = setup_opentelemetry(
    service_name="loan-advisor-api",
    version="2.4.1"
)

@app.post("/api/v1/mortgage-advice")
async def mortgage_advice(request: MortgageRequest):
    with tracer.start_as_current_span("mortgage_advice") as span:
        span.set_attribute("banking.product", "mortgage")
        span.set_attribute("banking.loan_amount", request.loan_amount)

        # Your business logic here
        advice = generate_advice(request)

        span.set_attribute("banking.advice_generated", True)
        return advice
```

### Custom Instrumentation

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

class RAGPipeline:
    def retrieve(self, query: str) -> list[Document]:
        with tracer.start_as_current_span("rag_retrieve") as span:
            span.set_attribute("query_length", len(query))

            # Embedding
            with tracer.start_as_current_span("rag_embedding") as embedding_span:
                embedding = self.embedder.encode(query)
                embedding_span.set_attribute("embedding_dimension", len(embedding))

            # Vector search
            with tracer.start_as_current_span("rag_vector_search") as search_span:
                results = self.vector_db.search(embedding, top_k=10)
                search_span.set_attribute("results_found", len(results))

            # Reranking
            with tracer.start_as_current_span("rag_reranking") as rerank_span:
                reranked = self.reranker.rerank(results, query)
                rerank_span.set_attribute("reranked_count", len(reranked))

            span.set_attribute("final_documents", len(reranked))
            return reranked
```

## OTel Collector Configuration

The OTel Collector receives telemetry from applications and exports it to backends. Deploy it as a sidecar (one per pod) or daemonset (one per node).

### collector-config.yaml

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000

  # Tail sampling for traces
  tail_sampling:
    policies:
      [
        {
          name: errors,
          type: status_code,
          status_code: { status_codes: [ERROR] }
        },
        {
          name: slow-requests,
          type: latency,
          latency: { threshold_ms: 5000 }
        },
        {
          name: random-sample,
          type: probabilistic,
          probabilistic: { sampling_percentage: 5 }
        }
      ]

  # Add banking-specific attributes
  attributes:
    actions:
      - key: banking.compliance_level
        value: "high"
        action: insert
      - key: banking.data_classification
        value: "confidential"
        action: insert

exporters:
  otlp/jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true

  prometheus:
    endpoint: 0.0.0.0:8889

  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, tail_sampling, attributes]
      exporters: [otlp/jaeger]

    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]

    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

## GenAI Semantic Conventions

OpenTelemetry has semantic conventions for GenAI operations. Use them:

```python
from opentelemetry.semconv_ai import SpanAttributes

# When making an LLM call
with tracer.start_as_current_span("llm_call") as span:
    span.set_attribute(SpanAttributes.LLM_SYSTEM, "openai")
    span.set_attribute(SpanAttributes.LLM_REQUEST_MODEL, "gpt-4-turbo")
    span.set_attribute(SpanAttributes.LLM_REQUEST_TEMPERATURE, 0.1)
    span.set_attribute(SpanAttributes.LLM_REQUEST_MAX_TOKENS, 2048)
    span.set_attribute(SpanAttributes.LLM_USAGE_PROMPT_TOKENS, 1245)
    span.set_attribute(SpanAttributes.LLM_USAGE_COMPLETION_TOKENS, 456)
    span.set_attribute(SpanAttributes.LLM_RESPONSE_FINISH_REASON, "stop")

    # Banking-specific extensions
    span.set_attribute("banking.product_line", "mortgage")
    span.set_attribute("banking.requires_compliance_review", True)
    span.set_attribute("banking.model_version", "gpt-4-turbo-2024-04-09")
```

## Environment Variables

Configure OTel via environment variables for easy deployment changes:

```bash
# Collector endpoint
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"

# Service identification
export OTEL_SERVICE_NAME="loan-advisor-api"
export OTEL_RESOURCE_ATTRIBUTES="service.version=2.4.1,deployment.environment=production"

# Sampling rate (for head sampling)
export OTEL_TRACES_SAMPLER="parentbased_traceidratio"
export OTEL_TRACES_SAMPLER_ARG="0.1"  # 10% sampling

# Propagators
export OTEL_PROPAGATORS="tracecontext,baggage"

# Log level for OTel SDK
export OTEL_LOG_LEVEL="info"
```

## Kubernetes Deployment

### Collector as DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.96.0
        args: ["--config=/etc/otel/config.yaml"]
        ports:
        - containerPort: 4317  # OTLP gRPC
        - containerPort: 4318  # OTLP HTTP
        - containerPort: 8889  # Prometheus metrics
        volumeMounts:
        - name: otel-config
          mountPath: /etc/otel
      volumes:
      - name: otel-config
        configMap:
          name: otel-collector-config
```

### Application Pod Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loan-advisor-api
spec:
  template:
    spec:
      containers:
      - name: loan-advisor
        image: loan-advisor:2.4.1
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://localhost:4317"  # Sidecar or node-local collector
        - name: OTEL_SERVICE_NAME
          value: "loan-advisor-api"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "service.version=2.4.1"
```

## Testing Your Setup

```python
# test_otel.py
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export.in_memory_span_exporter import InMemorySpanExporter
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from opentelemetry import trace

def test_span_creation():
    """Verify spans are created correctly."""
    exporter = InMemorySpanExporter()
    provider = TracerProvider()
    provider.add_span_processor(SimpleSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    tracer = trace.get_tracer("test")
    with tracer.start_as_current_span("test_span") as span:
        span.set_attribute("test.key", "test_value")

    spans = exporter.get_finished_spans()
    assert len(spans) == 1
    assert spans[0].name == "test_span"
    assert spans[0].attributes["test.key"] == "test_value"
```

## Common Mistakes

1. **Not setting resource attributes**: Without `service.name` and `service.version`, you cannot filter traces by service.

2. **Using SyncSpanProcessor in production**: Always use `BatchSpanProcessor` for async, batched export.

3. **Forgetting to instrument HTTP clients**: Auto-instrument the server but not the client means outgoing calls are invisible.

4. **Collector misconfiguration**: A missing processor or wrong exporter endpoint silently drops all telemetry.

5. **Not testing in staging**: OTel configuration errors often surface only under load. Test in staging with production-like traffic.

6. **Ignoring the cost of verbose spans**: Every attribute increases payload size. In high-traffic banking systems, this matters.
