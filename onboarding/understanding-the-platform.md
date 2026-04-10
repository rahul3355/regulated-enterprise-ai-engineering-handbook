# Understanding the GenAI Platform

> **Audience:** All engineers joining the GenAI Platform Team
> **Purpose:** High-level and detailed architecture of the platform, how services interact
> **Prerequisites:** None — this is Day 1 reading

---

## Platform Mission

The GenAI Platform enables banking applications across the organization to safely and effectively use Large Language Models (LLMs) and related AI technologies. We provide:

1. **Abstraction** — Consumers do not need to know which AI provider is behind the scenes
2. **Guardrails** — Every request and response is validated for security, compliance, and quality
3. **Observability** — Full visibility into usage, cost, performance, and quality
4. **Governance** — Audit trails, policy enforcement, and regulatory compliance

**Our consumers include:**

| Consumer | Team | Use Case | AI Provider | Daily Requests |
|----------|------|----------|-------------|----------------|
| Wealth Management Chat | Wealth Digital | Customer Q&A on portfolios | Azure OpenAI GPT-4o | 15,000 |
| Investment Research Bot | Investment Analytics | Summarize market reports | Azure OpenAI GPT-4o + internal LLM | 8,000 |
| Compliance Assistant | Risk & Compliance | Policy document Q&A | Internal LLM (Llama 3 70B) | 3,000 |
| Code Review Assistant | Engineering | PR summarization and suggestions | Azure OpenAI GPT-4o | 2,000 |
| HR Policy Chatbot | People Operations | Employee handbook Q&A | Azure OpenAI GPT-3.5-turbo | 5,000 |
| Fraud Detection Analyst | Financial Crime | Transaction pattern analysis | Internal LLM (fine-tuned) | 10,000 |
| KYC Document Processor | Onboarding | Extract data from ID documents | Azure OpenAI Vision + internal OCR | 20,000 |

**Platform Scale:**

- Total daily requests: ~63,000
- Average latency (P50): 340ms (excluding LLM call)
- Average latency (P99): 2,800ms (dominated by LLM call)
- Monthly token usage: ~2.4 billion
- Monthly AI provider cost: ~$480,000
- Platform availability SLO: 99.95%

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CONSUMER LAYER                                     │
│                                                                             │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │ Wealth Mgmt │ │Investment    │ │ Compliance   │ │ 6+ other consumers   │ │
│  │ Chat        │ │ Research Bot │ │ Assistant    │ │                       │ │
│  └──────┬──────┘ └──────┬───────┘ └──────┬───────┘ └──────────┬───────────┘ │
│         │               │                │                     │             │
│         └───────────────┴────────────────┴─────────────────────┘             │
│                                     │                                         │
│                              mTLS + API Key                                   │
│                                     │                                         │
├─────────────────────────────────────┼─────────────────────────────────────────┤
│                           PLATFORM LAYER                                      │
│                                                                             │
│  ┌──────────────────────────────────▼──────────────────────────────────┐    │
│  │                      API GATEWAY                                     │    │
│  │                                                                      │    │
│  │  - TLS termination                                                   │    │
│  │  - API key validation (against Vault)                                │    │
│  │  - Rate limiting (token bucket per consumer)                         │    │
│  │  - Request/response logging (PII-redacted to Splunk)                 │    │
│  │  - Request routing to Orchestrator                                   │    │
│  │  - Health check endpoints                                            │    │
│  │                                                                      │    │
│  │  Technology: Kong Gateway + custom Lua plugins                       │    │
│  │  Deployment: Kubernetes (3 replicas, pod anti-affinity)              │    │
│  │  Latency budget: 10ms                                                │    │
│  └──────────────────────────────────┬──────────────────────────────────┘    │
│                                     │                                        │
│  ┌──────────────────────────────────▼──────────────────────────────────┐    │
│  │                      ORCHESTRATOR                                    │    │
│  │                                                                      │    │
│  │  - Request processing and intent detection                           │    │
│  │  - RAG pipeline management                                           │    │
│  │  - Provider selection and routing                                    │    │
│  │  - Response assembly                                                 │    │
│  │  - Circuit breaker management for downstream services                │    │
│  │  - Request/response correlation (distributed tracing)                │    │
│  │                                                                      │    │
│  │  Technology: Python (FastAPI), asyncio                               │    │
│  │  Deployment: Kubernetes (5 replicas, HPA based on CPU + queue depth)│    │
│  │  Latency budget: 50ms (excluding RAG and LLM calls)                  │    │
│  └───────────┬──────────────┬──────────────┬──────────────────────────┘    │
│              │              │              │                                │
│     ┌────────▼────┐  ┌─────▼──────┐  ┌───▼────────────┐                   │
│     │ GUARDRAILS  │  │  EMBEDDING │  │  LLM PROVIDER  │                   │
│     │ SERVICE     │  │  SERVICE   │  │  ROUTER        │                   │
│     │             │  │            │  │                │                   │
│     │ Pre-flight: │  │ Query ->   │  │ Selects which  │                   │
│     │ - Injection │  │ Vector     │  │ AI provider to │                   │
│     │   detection │  │ Embedding  │  │ use based on:  │                   │
│     │ - PII scan  │  │            │  │ - Consumer     │                   │
│     │ - Toxicity  │  │ Vector DB: │  │   preference   │                   │
│     │             │  │ - Pinecone │  │ - Cost         │                   │
│     │ Post-flight:│  │ - Milvus   │  │ - Latency      │                   │
│     │ - Halluc.   │  │ (internal) │  │ - Availability │                   │
│     │   detection │  │            │  │ - Capability   │                   │
│     │ - Factual   │  │ Indexes:   │  │                │                   │
│     │   accuracy  │  │ - banking- │  │ Providers:     │                   │
│     │ - Compliance│  │   products │  │ - Azure OpenAI │                   │
│     │   filtering │  │ - policies │  │ - Internal     │                   │
│     │             │  │ -合规      │  │   (Llama 3)    │                   │
│     │ Model:      │  │ - market-  │  │ - (future)     │                   │
│     │ Custom      │  │   data     │  │   AWS Bedrock  │                   │
│     │ classifier  │  │            │  │                │                   │
│     │ + rules     │  │            │  │                │                   │
│     └─────────────┘  └────────────┘  └────────────────┘                    │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    SUPPORTING SERVICES                                 │  │
│  │                                                                       │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐    │  │
│  │  │ Audit       │  │ Cost         │  │ Document                  │    │  │
│  │  │ Logger      │  │ Tracker      │  │ Indexer                   │    │  │
│  │  │             │  │              │  │                           │    │  │
│  │  │ All guard-  │  │ Real-time    │  │ Ingests documents from    │    │  │
│  │  │ rails       │  │ token usage  │  │ content teams, chunks,    │    │  │
│  │  │ decisions   │  │ per consumer │  │ embeds, and stores in    │    │  │
│  │  │ logged to   │  │ and platform │  │ vector DB. Handles        │    │  │
│  │  │ Splunk +    │  │ level.       │  │ versioning, freshness,   │    │  │
│  │  │ compliance  │  │ Alerts on    │  │ and metadata extraction. │    │  │
│  │  │ database.   │  │ budget       │  │                           │    │  │
│  │  │ 365-day     │  │ thresholds.  │  │ Pipeline:                 │    │  │
│  │  │ retention.  │  │              │  │ S3 -> Tika -> Chunker -> │    │  │
│  │  │             │  │ Integrates   │  │ Embedder -> Vector DB    │    │  │
│  │  │             │  │ with FinOps  │  │                           │    │  │
│  │  │             │  │ dashboards.  │  │                           │    │  │
│  │  └─────────────┘  └──────────────┘  └──────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                     INFRASTRUCTURE LAYER                                    │
│                                                                             │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────────────┐ │
│  │ Kubernetes      │  │ Service Mesh     │  │ Observability Stack         │ │
│  │ (OpenShift 4.14)│  │ (Istio 1.20)     │  │                             │ │
│  │                 │  │                  │  │ Grafana (metrics)           │ │
│  │ 3 clusters:     │  │ mTLS between     │  │ Splunk (logs)              │ │
│  │ - dev           │  │ all services     │  │ Jaeger (tracing)           │ │
│  │ - staging       │  │ Traffic          │  │ PagerDuty (alerting)       │ │
│  │ - prod          │  │ policies         │  │                             │ │
│  │                 │  │ Rate limiting    │  │ Dashboards:                 │ │
│  │ Node pools:     │  │ Telemetry        │  │ - Platform overview        │ │
│  │ - general       │  │                  │  │ - LLM provider health      │ │
│  │ - GPU (for      │  │                  │  │ - Guardrails metrics       │ │
│  │   internal LLM) │  │                  │  │ - Cost tracking            │ │
│  │                 │  │                  │  │ - Consumer breakdown       │ │
│  │ Autoscaling:    │  │                  │  │ - Infrastructure health    │ │
│  │ HPA + Karpenter │  │                  │  │                             │ │
│  └─────────────────┘  └──────────────────┘  └────────────────────────────┘ │
│                                                                             │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────────────┐ │
│  │ Secrets Mgmt    │  │ Data Storage     │  │ CI/CD                        │ │
│  │                 │  │                  │  │                              │ │
│  │ HashiCorp Vault │  │ PostgreSQL 15    │  │ Jenkins (build + test)      │ │
│  │ - API keys      │  │ (consumer config,│  │ ArgoCD (GitOps deployment)  │ │
│  │ - DB creds      │  │  budget config)  │  │ SonarQube (code quality)    │ │
│  │ - Certificates  │  │                  │  │ Trivy (container scanning)  │ │
│  │                 │  │ Redis 7          │  │ OWASP Dependency Check      │ │
│  │ Rotation:       │  │ (caching, rate   │  │ Custom compliance gates    │ │
│  │ - API keys: 90d │  │  limiting, token │  │                             │ │
│  │ - DB creds: 30d │  │  budget tracking)│  │ Pipeline: see               │ │
│  │ - Certs: auto   │  │                  │  │ understanding-the-deployment│ │
│  │                 │  │ Elasticsearch    │  │ -pipeline.md                │ │
│  │                 │  │ (log aggregation │  │                              │ │
│  │                 │  │  + search)       │  │                              │ │
│  │                 │  │                  │  │                              │ │
│  │                 │  │ S3/MinIO         │  │                              │ │
│  │                 │  │ (document store, │  │                              │ │
│  │                 │  │  model artifacts)│  │                              │ │
│  └─────────────────┘  └──────────────────┘  └────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Service Deep Dive: The Orchestrator

The orchestrator is the brain of the platform. It coordinates all other services to fulfill a consumer request.

### Request Lifecycle

```python
# genai_platform/orchestrator/handlers/request_handler.py
"""
Main request handler in the orchestrator.

This is the most critical code path in the platform.
Every consumer request flows through here.

Performance budget:
- Total orchestrator overhead: < 100ms (excluding LLM call)
- P99 overhead: < 200ms

SLO impact:
- If orchestrator adds > 200ms, platform SLO is at risk
- Profile this code path quarterly
"""

import asyncio
from datetime import datetime
from typing import Optional
import structlog
from opentelemetry import trace

from genai_platform.guardrails.client import GuardrailsClient
from genai_platform.embedding.client import EmbeddingClient
from genai_platform.rag.retriever import DocumentRetriever
from genai_platform.llm.router import LLMRouter
from genai_platform.audit.logger import AuditLogger

logger = structlog.get_logger()
tracer = trace.get_tracer("orchestrator")

class RequestHandler:
    def __init__(
        self,
        guardrails_client: GuardrailsClient,
        embedding_client: EmbeddingClient,
        retriever: DocumentRetriever,
        llm_router: LLMRouter,
        audit_logger: AuditLogger,
    ):
        self.guardrails = guardrails_client
        self.embedding = embedding_client
        self.retriever = retriever
        self.llm_router = llm_router
        self.audit = audit_logger
    
    @tracer.start_as_current_span("handle_request")
    async def handle(
        self,
        consumer_id: str,
        prompt: str,
        context: dict,
    ) -> dict:
        """
        Handle a consumer request through the full RAG pipeline.
        
        Steps:
        1. Pre-flight guardrails check (injection, PII, toxicity)
        2. Generate embedding for the query
        3. Retrieve relevant documents from vector DB
        4. Rerank and select top documents
        5. Construct the prompt with context
        6. Route to appropriate LLM provider
        7. Generate response
        8. Post-flight guardrails check (hallucination, compliance)
        9. Log to audit trail
        10. Return response
        
        Args:
            consumer_id: The consumer making the request
            prompt: The user's prompt
            context: Request metadata (user_id, session_id, etc.)
        
        Returns:
            Response dict with content, metadata, and audit info
        """
        request_id = context.get("request_id", "unknown")
        start_time = datetime.utcnow()
        
        # Step 1: Pre-flight guardrails
        logger.info("starting_preflight_guardrails", request_id=request_id)
        preflight_result = await self.guardrails.check_prompt(
            content=prompt,
            check_type="preflight",
            consumer_id=consumer_id,
        )
        
        if preflight_result.action == "DENY":
            self.audit.log_decision(
                request_id=request_id,
                action="denied_preflight",
                reason=preflight_result.reason,
                consumer_id=consumer_id,
            )
            return {
                "status": "denied",
                "reason": preflight_result.reason,
                "request_id": request_id,
            }
        
        if preflight_result.action == "REDACT":
            prompt = preflight_result.redacted_content
        
        # Step 2-4: RAG retrieval
        logger.info("starting_rag_retrieval", request_id=request_id)
        embedding = await self.embedding.generate(prompt)
        
        retrieved_docs = await self.retriever.retrieve(
            query_embedding=embedding,
            consumer_id=consumer_id,
            top_k=10,
            similarity_threshold=0.75,
        )
        
        reranked_docs = await self.retriever.rerank(
            query=prompt,
            documents=retrieved_docs,
            top_k=3,
        )
        
        # Step 5: Construct prompt
        constructed_prompt = self._build_prompt(
            original_prompt=prompt,
            retrieved_docs=reranked_docs,
            consumer_id=consumer_id,
        )
        
        # Step 6-7: LLM generation
        logger.info("starting_llm_generation", request_id=request_id)
        provider = self.llm_router.select_provider(consumer_id)
        
        llm_response = await self.llm_router.generate(
            provider=provider,
            prompt=constructed_prompt,
            consumer_id=consumer_id,
            request_id=request_id,
        )
        
        # Step 8: Post-flight guardrails
        logger.info("starting_postflight_guardrails", request_id=request_id)
        postflight_result = await self.guardrails.check_response(
            content=llm_response.content,
            retrieved_docs=reranked_docs,
            consumer_id=consumer_id,
        )
        
        final_content = llm_response.content
        if postflight_result.action == "REDACT":
            final_content = postflight_result.redacted_content
        
        if postflight_result.action == "DENY":
            # This should be rare — the model produced something unacceptable
            self.audit.log_decision(
                request_id=request_id,
                action="denied_postflight",
                reason=postflight_result.reason,
                consumer_id=consumer_id,
            )
            return {
                "status": "denied",
                "reason": "Response failed validation: " + postflight_result.reason,
                "request_id": request_id,
            }
        
        # Step 9: Audit logging
        processing_time = (datetime.utcnow() - start_time).total_seconds()
        
        self.audit.log_completion(
            request_id=request_id,
            consumer_id=consumer_id,
            provider=provider,
            processing_time_ms=processing_time * 1000,
            token_usage=llm_response.token_usage,
            retrieved_documents=len(reranked_docs),
            guardrails={
                "preflight": preflight_result.action,
                "postflight": postflight_result.action,
            },
        )
        
        # Step 10: Return response
        return {
            "status": "success",
            "content": final_content,
            "metadata": {
                "request_id": request_id,
                "provider": provider,
                "processing_time_ms": processing_time * 1000,
                "token_usage": llm_response.token_usage,
                "retrieved_documents": len(reranked_docs),
                "sources": [
                    {"title": doc.title, "id": doc.id}
                    for doc in reranked_docs
                ],
            },
        }
    
    def _build_prompt(
        self,
        original_prompt: str,
        retrieved_docs: list,
        consumer_id: str,
    ) -> str:
        """
        Construct the final prompt with retrieved context.
        
        Template varies by consumer_id to match their requirements.
        Templates are stored in config/templates/{consumer_id}.yaml
        """
        # Template loading from config
        # ... (implementation in template_engine.py)
        pass
```

---

## Key Design Decisions

### 1. Why We Use a Provider Router

We do not hard-code consumers to a specific AI provider. The LLM router decides which provider to use based on:

- **Consumer configuration** — Some consumers specify preferred providers
- **Cost** — Cheaper providers are preferred when quality requirements allow
- **Latency** — If a provider is degraded, traffic shifts to alternatives
- **Capability** — Some models support vision, others do not
- **Availability** — Circuit breaker state affects routing

```python
# genai_platform/llm/router.py — simplified provider selection logic

def select_provider(self, consumer_id: str) -> str:
    """
    Select the best provider for this request.
    
    Decision order:
    1. If consumer has a forced provider, use it
    2. If consumer has preferred providers, try them in order
    3. Otherwise, select based on cost and availability
    
    Circuit breaker state is checked before selection.
    """
    consumer_config = self.config.get_consumer(consumer_id)
    
    # Forced provider (emergency override)
    if consumer_config.forced_provider:
        return consumer_config.forced_provider
    
    # Preferred providers (try in order)
    for provider in consumer_config.preferred_providers:
        if self.circuit_breaker.is_available(provider):
            return provider
    
    # Fallback: select by cost and availability
    available = [
        p for p in self.all_providers
        if self.circuit_breaker.is_available(p)
    ]
    
    if not available:
        raise ProviderUnavailableError(
            f"All providers are unavailable (circuit breakers open)"
        )
    
    return min(available, key=lambda p: self.cost_per_token[p])
```

### 2. Why We Use Internal LLMs Alongside Azure OpenAI

| Factor | Azure OpenAI | Internal LLM (Llama 3 70B) |
|--------|-------------|---------------------------|
| Cost per 1K tokens | $0.01 (input) / $0.03 (output) | $0.003 (compute amortized) |
| Latency (P50) | 800ms | 400ms (GPU-optimized) |
| Data residency | UK South region | On-premises, full control |
| Compliance | Microsoft DPA + bank addendum | Full internal control |
| Model capabilities | State-of-the-art | Good, improving |
| Fine-tuning | Limited | Full control |
| Use cases | General purpose, creative tasks | Compliance, internal data, high-volume |

**Routing strategy:** We prefer internal LLMs for high-volume, internal-data use cases (compliance, fraud detection) and Azure OpenAI for consumer-facing, quality-sensitive use cases (wealth management chat).

### 3. Why Guardrails Run Twice

Guardrails run both **before** (pre-flight) and **after** (post-flight) the LLM call because:

- **Pre-flight** protects the LLM from malicious prompts (injection, jailbreaks)
- **Post-flight** protects the consumer from harmful outputs (hallucinations, toxic content, compliance violations)

Some checks only make sense post-flight:
- Hallucination detection (requires comparing output to retrieved context)
- Factual accuracy verification
- Response completeness check
- Compliance filtering of generated content

---

## Common Consumer Integration Pattern

When a new team wants to use the platform, they follow this pattern:

```python
# Example: How the Wealth Management Chat team integrates with our platform

import requests
from typing import Optional

class GenAIPlatformClient:
    """
    Client for the GenAI Platform API.
    
    All consumers should use this client library (available at:
    pypi.bank.internal/genai-platform-client)
    
    Never call the platform API directly — the client handles:
    - Authentication (mTLS + API key)
    - Retry logic with exponential backoff
    - Request ID generation for tracing
    - Timeout management
    """
    
    def __init__(
        self,
        consumer_id: str,
        api_key: str,
        base_url: str = "https://api-gateway.platform.bank",
        timeout: float = 30.0,
    ):
        self.consumer_id = consumer_id
        self.api_key = api_key
        self.base_url = base_url
        self.timeout = timeout
        self.session = self._create_session()
    
    def _create_session(self) -> requests.Session:
        session = requests.Session()
        session.headers.update({
            "X-Consumer-ID": self.consumer_id,
            "X-API-Key": self.api_key,
            "Content-Type": "application/json",
        })
        session.verify = "/etc/ssl/certs/bank-ca.pem"  # mTLS
        return session
    
    def chat(
        self,
        prompt: str,
        user_id: str,
        session_id: str,
        temperature: float = 0.3,
        max_tokens: int = 500,
    ) -> dict:
        """
        Send a chat request to the platform.
        
        Returns:
            {
                "status": "success",
                "content": "response text",
                "metadata": {
                    "request_id": "...",
                    "provider": "azure-openai",
                    "processing_time_ms": 1200,
                    "token_usage": {"prompt": 450, "completion": 120},
                    "sources": [{"title": "...", "id": "..."}],
                }
            }
        """
        response = self.session.post(
            f"{self.base_url}/v1/chat",
            json={
                "prompt": prompt,
                "user_id": user_id,
                "session_id": session_id,
                "temperature": temperature,
                "max_tokens": max_tokens,
            },
            timeout=self.timeout,
        )
        response.raise_for_status()
        return response.json()
```

---

## Platform Roadmap (Q2-Q3 2025)

| Initiative | Description | Target Date | Status |
|-----------|-------------|-------------|--------|
| Multi-modal support | Image and document analysis via Azure OpenAI Vision | May 2025 | In development |
| Streaming responses | Server-sent events for long-running generation | June 2025 | Design phase |
| Fine-tuning pipeline | Internal LLM fine-tuning for domain-specific tasks | July 2025 | Planning |
| Agent framework | Multi-step task execution with tool use | August 2025 | Research |
| Cross-region DR | Platform failover to secondary region | September 2025 | Planning |
| Real-time fact-checking | Live verification against authoritative data sources | Q3 2025 | Research |

---

## Further Reading

- `architecture/` — Detailed architecture diagrams and ADRs
- `genai-platforms/` — AI provider configurations and integration guides
- `banking-domain/` — Domain-specific use cases and requirements
- `databases/` — Data storage strategies and schemas
- `infrastructure/` — Kubernetes and infrastructure configuration
- `incident-management/` — Runbooks and post-mortem templates
- `understanding-the-deployment-pipeline.md` — How code reaches production
- `understanding-regulated-environments.md` — Compliance requirements

---

*Last updated: April 2025 | Document owner: Platform Architecture Team | Review cycle: Monthly*
