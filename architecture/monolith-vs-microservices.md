# Monolith vs Microservices in Banking GenAI Systems

## Overview

The monolith vs microservices decision shapes the entire engineering organization's structure, deployment velocity, and operational complexity. For banking GenAI systems, this decision carries additional weight because of regulatory requirements for auditability, the need for rapid model iteration, and the computational characteristics of AI workloads.

There is no universal answer -- the right architecture depends on team size, deployment frequency, regulatory constraints, and the specific GenAI use case.

---

## Decision Framework

```mermaid
flowchart TD
    A[Start] --> B{Team size?}
    B -->|< 10 engineers| C[Start with modular monolith]
    B -->|> 10 engineers| D{Domain complexity?}
    D -->|Single domain| C
    D -->|Multiple domains| E[Consider microservices]

    C --> F{Deployment frequency?}
    F -->|Daily| G{Scaling needs?}
    F -->|Weekly/Monthly| C
    G -->|GPU vs CPU different| E
    G -->|Uniform| C

    E --> H[Define service boundaries]
    H --> I[Implement with clear contracts]
```

---

## Modular Monolith

A modular monolith is a single deployable unit with clear internal module boundaries. It is the recommended starting point for most banking GenAI teams.

### When to Choose a Modular Monolith

- Team size under 10 engineers
- Single GenAI use case (e.g., one RAG-powered banking assistant)
- Deployment frequency under daily
- Shared compute requirements (CPU and GPU co-located)
- Regulatory simplicity (single audit boundary)

### Architecture

```mermaid
graph TD
    A[API Gateway] --> B[Modular Monolith]
    B --> C[RAG Module]
    B --> D[Embedding Module]
    B --> E[Chat Module]
    B --> F[Document Ingestion]
    B --> G[Authorization Module]
    B --> H[Audit Logging Module]
    C --> I[(Vector DB)]
    D --> I
    F --> I
    B --> J[(PostgreSQL)]
    B --> K[(Redis Cache)]
    B --> L[LLM Provider API]

    style B fill:#4CAF50
```

```python
# app/__init__.py -- Modular monolith structure
"""
Banking GenAI Platform -- Modular Monolith

Internal modules communicate via function calls, not HTTP.
This keeps the architecture simple while maintaining clean boundaries.
"""
from app.rag import RAGService
from app.embeddings import EmbeddingService
from app.chat import ChatService
from app.documents import DocumentService
from app.auth import AuthService
from app.audit import AuditService

class BankingGenAIApplication:
    """
    The application orchestrates all internal services.
    Each service is a Python class with clear interfaces.
    """
    def __init__(self, config: dict):
        self.auth = AuthService(config["auth"])
        self.audit = AuditService(config["audit"])
        self.embeddings = EmbeddingService(config["embeddings"])
        self.rag = RAGService(config["rag"], self.embeddings)
        self.chat = ChatService(config["chat"], self.rag)
        self.documents = DocumentService(config["documents"], self.embeddings)

    async def query(self, user_request: dict) -> dict:
        """Handle a RAG query with full audit logging."""
        # Authorization
        self.auth.verify_token(user_request["token"])

        # Audit log
        audit_id = self.audit.log_request(user_request)

        # Process query
        result = await self.rag.query(
            query=user_request["query"],
            customer_id=user_request["customer_id"],
        )

        # Audit log response
        self.audit.log_response(audit_id, result)

        return result
```

### Advantages for Banking GenAI

| Aspect | Benefit |
|---|---|
| **Auditability** | Single codebase, single audit boundary, simpler compliance |
| **Development speed** | No network latency between modules, easy debugging |
| **Deployment** | One artifact, one deployment pipeline, simple rollback |
| **Testing** | Integration tests run in-process, faster CI |
| **Cost** | Single process, no inter-service communication overhead |

### When to Extract from Monolith

| Signal | Extract To |
|---|---|
| Embedding service needs dedicated GPU | Embedding microservice on GPU nodes |
| Document ingestion is slow and blocks queries | Async document service with message queue |
| Different team owns the chat product | Separate chat service with own deployment |
| Regulatory requires isolation of audit data | Separate audit service with independent database |

---

## Microservices Architecture

### When to Choose Microservices

- Multiple independent GenAI products (RAG assistant, fraud detection, document summarization)
- Different scaling requirements (GPU for inference, CPU for document processing)
- Multiple teams owning different domains
- Different technology stacks per domain
- Regulatory requires data isolation between services

### Banking GenAI Microservices Topology

```mermaid
graph TD
    A[API Gateway] --> B[Auth Service]
    A --> C[RAG Query Service]
    A --> D[Chat Service]
    A --> E[Document Service]
    A --> F[Analytics Service]

    C --> G[Embedding Service]
    C --> H[Vector DB]
    E --> G
    E --> H

    C --> I[LLM Provider Pool]
    D --> I
    E --> I

    B --> J[(Auth DB)]
    C --> K[(Query Cache)]
    E --> L[(Document Store)]
    F --> M[(Analytics DB)]

    C --> N[Message Broker]
    E --> N
    N --> F

    style C fill:#2196F3
    style G fill:#FF9800
    style I fill:#9C27B0
```

```yaml
# docker-compose.microservices.yaml
version: '3.8'
services:
  auth-service:
    image: banking-genai/auth-service:latest
    ports:
      - "8081:8080"
    environment:
      - DATABASE_URL=postgresql://auth:auth@auth-db:5432/auth
    depends_on:
      - auth-db

  rag-query-service:
    image: banking-genai/rag-service:latest
    ports:
      - "8082:8080"
    environment:
      - EMBEDDING_SERVICE_URL=http://embedding-service:8080
      - VECTOR_DB_URL=http://qdrant:6333
      - LLM_PROVIDER_URL=http://llm-router:8080
      - REDIS_URL=redis://redis:6379/1
    depends_on:
      - embedding-service
      - qdrant
      - llm-router

  embedding-service:
    image: banking-genai/embedding-service:latest
    ports:
      - "8083:8080"
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
    environment:
      - MODEL_NAME=text-embedding-3-large
      - BATCH_SIZE=32

  chat-service:
    image: banking-genai/chat-service:latest
    ports:
      - "8084:8080"
    environment:
      - RAG_SERVICE_URL=http://rag-query-service:8080
      - LLM_PROVIDER_URL=http://llm-router:8080

  document-service:
    image: banking-genai/document-service:latest
    ports:
      - "8085:8080"
    environment:
      - EMBEDDING_SERVICE_URL=http://embedding-service:8080
      - DOCUMENT_STORE_URL=http://minio:9000

  llm-router:
    image: banking-genai/llm-router:latest
    ports:
      - "8086:8080"
    environment:
      - PROVIDERS=openai,anthropic,self-hosted
      - LOAD_BALANCING=latency-weighted

  qdrant:
    image: qdrant/qdrant:v1.7.0
    ports:
      - "6333:6333"

  redis:
    image: redis:7-alpine

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
```

### Service Communication Patterns

```python
# services/rag_query_service/main.py
"""
RAG Query Service -- handles retrieval and generation.
Communicates with other services via HTTP (synchronous) and message broker (asynchronous).
"""
from fastapi import FastAPI, Depends
import httpx
from app.dependencies import get_config

app = FastAPI()

@app.post("/api/v1/query")
async def query(request: dict, config=Depends(get_config)):
    # Synchronous: call embedding service
    async with httpx.AsyncClient() as client:
        embedding_response = await client.post(
            f"{config.EMBEDDING_SERVICE_URL}/embed",
            json={"text": request["query"]},
            timeout=5.0,
        )
        query_embedding = embedding_response.json()["embedding"]

    # Synchronous: query vector DB
    vector_results = await query_vector_db(query_embedding, top_k=5)

    # Synchronous: call LLM
    async with httpx.AsyncClient() as client:
        llm_response = await client.post(
            f"{config.LLM_PROVIDER_URL}/completions",
            json={
                "prompt": build_rag_prompt(request["query"], vector_results),
                "max_tokens": 500,
            },
            timeout=30.0,
        )
        answer = llm_response.json()["text"]

    # Asynchronous: log query for analytics
    await publish_to_message_broker("rag_query_completed", {
        "query": request["query"],
        "response_time_ms": llm_response.elapsed.total_seconds() * 1000,
        "documents_retrieved": len(vector_results),
        "customer_id": request.get("customer_id"),
    })

    return {
        "answer": answer,
        "sources": [doc["metadata"] for doc in vector_results],
        "confidence": calculate_confidence(vector_results, llm_response),
    }
```

---

## Migration Strategy: Monolith to Microservices

```mermaid
flowchart LR
    A[Modular Monolith] --> B[Strangler Fig Pattern]
    B --> C[Extract Embedding Service]
    B --> D[Extract Document Service]
    B --> E[Extract Chat Service]
    C --> F[Full Microservices]
    D --> F
    E --> F
```

### Phase 1: Identify Extraction Candidates

```python
# migration/analysis.py
"""
Analyze monolith to identify services for extraction.
Look for: high resource usage, independent deployment needs, different scaling patterns.
"""
import psutil
from collections import defaultdict

def analyze_module_resource_usage() -> dict:
    """Identify modules that could benefit from independent scaling."""
    resource_usage = defaultdict(lambda: {"cpu": 0, "memory_mb": 0, "requests_per_sec": 0})

    # Analyze per-module metrics
    # In practice, this comes from APM (Datadog, New Relic)
    resource_usage["embedding"]["cpu"] = 75  # High CPU usage
    resource_usage["embedding"]["memory_mb"] = 4096  # GPU memory
    resource_usage["embedding"]["requests_per_sec"] = 200

    resource_usage["document_ingestion"]["cpu"] = 40
    resource_usage["document_ingestion"]["memory_mb"] = 2048
    resource_usage["document_ingestion"]["requests_per_sec"] = 50  # Low frequency, high cost

    resource_usage["rag_query"]["cpu"] = 20
    resource_usage["rag_query"]["memory_mb"] = 512
    resource_usage["rag_query"]["requests_per_sec"] = 500

    return dict(resource_usage)


def recommend_extraction(resource_usage: dict) -> list:
    """Recommend modules for extraction based on resource patterns."""
    recommendations = []

    for module, metrics in resource_usage.items():
        reasons = []

        if metrics["cpu"] > 60:
            reasons.append("High CPU usage -- benefits from dedicated scaling")
        if metrics["memory_mb"] > 2048:
            reasons.append("High memory usage -- GPU isolation benefit")
        if metrics["requests_per_sec"] < 100:
            reasons.append("Low frequency -- can run on separate schedule")

        if len(reasons) >= 2:
            recommendations.append({
                "module": module,
                "reasons": reasons,
                "priority": "high" if len(reasons) >= 3 else "medium",
            })

    return recommendations
```

### Phase 2: Implement Anti-Corruption Layer

```python
# migration/anti_corruption_layer.py
"""
Anti-corruption layer allows the monolith to communicate with extracted services
without changing the monolith's internal API.
"""
import httpx

class EmbeddingServiceClient:
    """
    Replaces the in-process embedding module with a service call.
    The monolith's internal API remains unchanged.
    """
    def __init__(self, service_url: str):
        self.service_url = service_url

    async def embed(self, text: str) -> list[float]:
        """
        Same interface as the old in-process embedding module.
        Internally calls the extracted microservice.
        """
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.service_url}/embed",
                json={"text": text},
                timeout=5.0,
            )
            return response.json()["embedding"]

    # Fallback to in-process if service is unavailable
    async def embed_with_fallback(self, text: str, fallback_module=None) -> list[float]:
        try:
            return await self.embed(text)
        except httpx.ConnectError:
            # Fall back to the old in-process module
            return fallback_module.embed(text)
```

---

## Interview Questions

1. **When would you choose a monolith for a banking GenAI system?**
   - When the team is small (under 10), the use case is单一 (one RAG assistant), deployment frequency is moderate, and regulatory requirements favor a single audit boundary. Start with a modular monolith and extract only when specific pain points emerge.

2. **What is the Strangler Fig pattern and how does it apply to GenAI systems?**
   - Gradually replace parts of the monolith with microservices by routing specific functionality to new services while keeping the monolith running. For GenAI, extract the embedding service first (clear boundary, dedicated GPU needs), then document ingestion (async, different scaling), then chat (independent product).

3. **How do microservices affect compliance and audit in banking?**
   - Microservices increase the audit surface: each service has its own deployment pipeline, access controls, and logging. However, they also enable better isolation (audit data in a separate service with strict access controls) and independent compliance certifications per service.

4. **What is the biggest operational cost of microservices in GenAI systems?**
   - Observability and debugging. When a query fails, you must trace through auth, embedding, vector retrieval, LLM routing, and response generation. Distributed tracing (OpenTelemetry) is mandatory. The cost of operating the observability infrastructure often exceeds the cost of the services themselves.

---

## Cross-References

- See [architecture/service-boundaries.md](./service-boundaries.md) for defining service boundaries
- See [architecture/api-gateway-design.md](./api-gateway-design.md) for gateway patterns
- See [kubernetes-openshift/](../kubernetes-openshift/) for deployment strategies
- See [engineering-culture/team-topologies.md](../engineering-culture/team-topologies.md) for team structure
