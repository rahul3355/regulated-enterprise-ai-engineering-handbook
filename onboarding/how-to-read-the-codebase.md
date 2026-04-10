# How to Read the Codebase

> **Audience:** Engineers joining the GenAI Platform Team
> **Purpose:** Navigate a large, multi-repository codebase efficiently
> **Prerequisites:** Access to all repositories (granted during Week 1)

---

## The Repository Landscape

Our platform spans multiple repositories, each with a specific purpose. Understanding this structure prevents you from getting lost.

### Repository Map

```
banking-genai-platform/
├── genai-platform-core/              # Core orchestrator and business logic
│   ├── src/genai_platform/
│   │   ├── orchestrator/             # Request handling, RAG pipeline
│   │   ├── guardrails/               # Pre/post-flight checks
│   │   ├── embedding/                # Embedding generation client
│   │   ├── rag/                      # Document retrieval and reranking
│   │   ├── llm/                      # Provider routing and generation
│   │   ├── audit/                    # Audit logging
│   │   └── config/                   # Configuration management
│   ├── tests/
│   ├── config/                       # Environment-specific configs
│   ├── Dockerfile
│   └── Makefile
│
├── genai-platform-api-gateway/       # Kong gateway configuration
│   ├── kong/
│   │   ├── plugins/                  # Custom Lua plugins
│   │   └── templates/                # Kong config templates
│   ├── config/
│   │   └── consumers/                # Per-consumer YAML configs
│   └── Dockerfile
│
├── genai-platform-guardrails/       # Guardrails service (separate deploy)
│   ├── src/
│   │   ├── checks/                   # Individual guardrail checks
│   │   ├── models/                   # ML model wrappers
│   │   └── pipeline.py               # Check execution pipeline
│   ├── tests/
│   ├── models/                       # Model artifacts
│   └── Dockerfile
│
├── genai-platform-embedding/        # Embedding service
│   ├── src/
│   │   ├── embedders/                # Model-specific embedders
│   │   └── router.py                 # Model selection logic
│   └── Dockerfile
│
├── genai-platform-document-indexer/ # Document ingestion pipeline
│   ├── src/
│   │   ├── ingest/                   # Document ingestion
│   │   ├── chunk/                    # Text chunking strategies
│   │   ├── embed/                    # Batch embedding
│   │   └── store/                    # Vector DB operations
│   └── Dockerfile
│
├── genai-platform-audit/            # Audit and observability
│   ├── src/
│   │   ├── logger/                   # Audit logging
│   │   ├── metrics/                  # Custom metrics
│   │   └── exporter/                 # Splunk/Grafana exporters
│   └── Dockerfile
│
├── genai-platform-cost-tracker/     # Cost monitoring
│   ├── src/
│   │   ├── tracker.py                # Real-time token tracking
│   │   ├── budget/                   # Budget management
│   │   └── reporter/                 # Cost reports and alerts
│   └── Dockerfile
│
├── genai-platform-infra/            # Infrastructure as Code
│   ├── terraform/                    # Cloud resource provisioning
│   ├── helm/                         # Kubernetes Helm charts
│   ├── argocd/                       # ArgoCD application definitions
│   └── scripts/                      # Operational scripts
│
├── genai-platform-client-sdk/       # Consumer client library
│   ├── python/                       # Python SDK (pip install)
│   ├── java/                         # Java SDK (Maven)
│   └── docs/                         # SDK documentation
│
└── genai-platform-docs/             # Internal documentation
    ├── architecture/                 # Architecture docs
    ├── runbooks/                     # Operational runbooks
    └── onboarding/                   # This directory
```

### Why So Many Repos?

**Historical reason:** The platform started as a monolith. As it grew, services were split out for:
- Independent deployment (guardrails can be updated without redeploying the orchestrator)
- Team ownership (different teams own different services)
- Security boundaries (guardrails code requires security review, other code does not)

**Current thinking:** We are evaluating a monorepo approach for 2025, but the migration is complex. For now, learn to navigate the multi-repo structure.

---

## Navigation Strategies

### 1. Start with the Entry Points

Every service has a clear entry point. Find it first:

```bash
# In genai-platform-core, the entry point is:
src/genai_platform/orchestrator/main.py

# In genai-platform-guardrails, the entry point is:
src/genai_platform/guardrails/api.py

# In genai-platform-document-indexer, the entry point is:
src/genai_platform/indexer/pipeline.py
```

**Pattern:** Look for `main.py`, `api.py`, `app.py`, or `__main__.py`. These are the starting points.

### 2. Follow the Request Flow

The best way to understand a service is to trace a request from entry to exit.

```bash
# Start with the request handler
src/genai_platform/orchestrator/handlers/request_handler.py

# Follow the imports
from genai_platform.guardrails.client import GuardrailsClient
from genai_platform.embedding.client import EmbeddingClient
from genai_platform.rag.retriever import DocumentRetriever

# Each of these imports is either:
# - In the same repo (look in src/genai_platform/)
# - In a different repo (listed above)
# - An external library (check requirements.txt)
```

### 3. Use the Dependency Graph

```bash
# Generate a dependency graph for the core repo
make deps-graph

# This outputs:
# target/dependency-graph.png
# 
# The graph shows:
# - Internal module dependencies (solid lines)
# - External library dependencies (dashed lines)
# - Circular dependencies (red lines — these are bugs)
```

**Common circular dependency to watch for:**

```
orchestrator -> guardrails -> audit -> orchestrator

This happens because the audit logger needs request context from the orchestrator,
but the orchestrator needs guardrails, and guardrails needs audit logging.

Solution: The audit logger uses a message queue pattern. Guardrails publishes
audit events to a queue, and the audit service consumes them asynchronously.
See: src/genai_platform/audit/queue.py
```

### 4. Read the Tests

Tests are documentation that cannot lie (well, they can, but it is harder).

```bash
# Find tests related to a specific module
# If you are reading: src/genai_platform/guardrails/pipeline.py
# Look at: tests/genai_platform/guardrails/test_pipeline.py

# The test file shows you:
# - What the expected behavior is
# - What edge cases exist
# - How to construct objects for testing
# - What mocks are needed
```

**Test organization:**

```
tests/
├── unit/                    # Fast tests (< 100ms each), no external deps
│   └── guardrails/
│       ├── test_pipeline.py
│       └── test_checks/
│           ├── test_injection_detection.py
│           └── test_pii_scan.py
├── integration/             # Slower tests, uses test containers
│   └── test_rag_pipeline.py # Full RAG pipeline with mock vector DB
├── e2e/                     # End-to-end tests against staging
│   └── test_chat_endpoint.py
└── fixtures/                # Shared test data
    ├── sample_prompts.json
    ├── sample_documents.json
    └── malicious_prompts.json
```

---

## Understanding Our Patterns

### Pattern 1: Dependency Injection via Configuration

We do not use a DI framework. Instead, dependencies are wired through configuration:

```python
# src/genai_platform/orchestrator/factory.py
"""
Service factory. Creates all dependencies for the orchestrator.

This is the composition root — where the object graph is assembled.
All dependencies are explicit, making the architecture visible.
"""

from genai_platform.guardrails.client import GuardrailsClient
from genai_platform.embedding.client import EmbeddingClient
from genai_platform.rag.retriever import DocumentRetriever
from genai_platform.llm.router import LLMRouter
from genai_platform.audit.logger import AuditLogger
from genai_platform.orchestrator.handlers import RequestHandler

def create_request_handler(config: dict) -> RequestHandler:
    """
    Create a fully configured RequestHandler.
    
    This function is the single entry point for creating handlers.
    In tests, you can create handlers with mock dependencies by
    calling the individual factory functions.
    """
    guardrails_client = GuardrailsClient(
        base_url=config["guardrails"]["url"],
        api_key=config["guardrails"]["api_key"],
        timeout=config["guardrails"]["timeout"],
    )
    
    embedding_client = EmbeddingClient(
        base_url=config["embedding"]["url"],
        model=config["embedding"]["model"],
        api_key=config["embedding"]["api_key"],
    )
    
    retriever = DocumentRetriever(
        vector_db_url=config["vector_db"]["url"],
        vector_db_key=config["vector_db"]["api_key"],
        index=config["vector_db"]["index"],
        reranker_model=config["reranker"]["model"],
    )
    
    llm_router = LLMRouter(
        providers=config["llm"]["providers"],
        circuit_breaker_config=config["llm"]["circuit_breaker"],
        cost_config=config["llm"]["cost"],
    )
    
    audit_logger = AuditLogger(
        splunk_url=config["audit"]["splunk_url"],
        splunk_token=config["audit"]["splunk_token"],
        compliance_db_url=config["audit"]["compliance_db_url"],
        retention_days=config["audit"]["retention_days"],
    )
    
    return RequestHandler(
        guardrails_client=guardrails_client,
        embedding_client=embedding_client,
        retriever=retriever,
        llm_router=llm_router,
        audit_logger=audit_logger,
    )
```

**Why this matters:** When reading code, you can always trace back to `factory.py` to understand how objects are created and what their dependencies are.

### Pattern 2: Structured Logging with Context

All logging uses `structlog` with automatic context injection:

```python
import structlog

logger = structlog.get_logger()

# Good: Include context as structured data
logger.info(
    "guardrail_check_completed",
    check_name="injection_detection",
    action="allow",
    confidence=0.12,
    request_id="req-12345",
    consumer_id="wealth-chat",
    duration_ms=45,
)

# Bad: Do not do this
logger.info(f"Guardrail check completed for request {request_id}")

# Why structured logging matters:
# 1. Splunk queries can filter on any field
#    | json | search request_id="req-12345" | table check_name, action, confidence
# 2. Grafana dashboards can create panels from any field
# 3. Alert conditions can be based on specific field values
# 4. Post-incident analysis requires structured data
```

**Log correlation across services:**

```
Every request gets a unique request_id at the API gateway.
This ID is propagated through all services via HTTP headers:
X-Request-ID: req-12345

In Splunk, you can trace a single request across all services:
index=genai_platform request_id="req-12345"

This shows:
1. API Gateway received the request (timestamp, consumer, IP)
2. Orchestrator started processing
3. Guardrails pre-flight check (result, confidence)
4. Embedding service called (model, latency)
5. Vector DB query (documents retrieved, similarity scores)
6. LLM provider called (provider, token usage, latency)
7. Guardrails post-flight check (result, confidence)
8. Response returned to consumer
9. Audit log entry created
```

### Pattern 3: Feature Flags

All behavioral changes are behind feature flags:

```python
# src/genai_platform/config/feature_flags.py
"""
Feature flags control behavioral changes without code deployment.

Flag types:
- Release flags: New features, temporary until rollout complete
- Experiment flags: A/B tests, temporary until experiment concludes
- Operational flags: Runtime controls, permanent

Flag management:
- Flags are stored in LaunchDarkly (bank enterprise account)
- Flag changes require CAB approval for production
- Flag state is cached in Redis (5-minute TTL)
- Flag changes are logged to Splunk
"""

from typing import Optional
import launchdarkly_server
import redis

class FeatureFlagClient:
    def __init__(self, ld_client: launchdarkly_server.LDClient, redis_client: redis.Redis):
        self.ld = ld_client
        self.redis = redis_client
        self.cache_ttl = 300  # 5 minutes
    
    def is_enabled(
        self,
        flag_key: str,
        context: dict,
        default: bool = False,
    ) -> bool:
        """
        Check if a feature flag is enabled.
        
        Uses Redis cache to avoid LD API call on every request.
        Cache key includes the context for user-specific flags.
        """
        cache_key = f"flag:{flag_key}:{hash(str(context))}"
        cached = self.redis.get(cache_key)
        
        if cached is not None:
            return cached == "true"
        
        # LD context requires a 'key' field
        ld_context = {
            "kind": "multi",
            "consumer": {"key": context.get("consumer_id", "unknown")},
            "environment": {"key": context.get("environment", "prod")},
        }
        
        result = self.ld.variation(flag_key, ld_context, default)
        
        self.redis.setex(cache_key, self.cache_ttl, str(result).lower())
        
        return result

# Usage:
# flags = FeatureFlagClient(ld_client, redis_client)
# if flags.is_enabled("new-reranker-model", {"consumer_id": consumer_id}):
#     docs = await self.reranker_v2.rerank(query, docs)
# else:
#     docs = await self.reranker_v1.rerank(query, docs)
```

**Feature flag lifecycle:**

```
1. Developer creates flag in LaunchDarkly (dev environment)
2. Flag defaults to OFF in all environments
3. Code is deployed behind the flag (flag check in code)
4. Flag is turned ON for dev environment (testing)
5. Flag is turned ON for staging environment (QA)
6. CAB approves flag activation for production
7. Flag is turned ON for 10% of consumers (canary)
8. Flag is turned ON for 50% of consumers
9. Flag is turned ON for 100% of consumers
10. After 2 weeks with no issues, flag is removed from code
11. Flag is archived in LaunchDarkly
```

### Pattern 4: Circuit Breakers

All external service calls are protected by circuit breakers:

```python
# src/genai_platform/llm/circuit_breaker.py
"""
Circuit breaker pattern for LLM provider calls.

States:
- CLOSED: Normal operation, requests flow through
- OPEN: Provider is failing, requests are rejected immediately
- HALF_OPEN: Testing if provider has recovered

Transitions:
CLOSED -> OPEN: When failure count exceeds threshold
OPEN -> HALF_OPEN: After cooldown period
HALF_OPEN -> CLOSED: When test request succeeds
HALF_OPEN -> OPEN: When test request fails

Configuration (per provider):
- failure_threshold: 5 consecutive failures
- cooldown_period: 60 seconds
- test_request_timeout: 10 seconds
"""

from enum import Enum
from datetime import datetime, timedelta
import structlog

logger = structlog.get_logger()

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(
        self,
        provider: str,
        failure_threshold: int = 5,
        cooldown_period: int = 60,
    ):
        self.provider = provider
        self.failure_threshold = failure_threshold
        self.cooldown_period = cooldown_period
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time: Optional[datetime] = None
    
    def can_execute(self) -> bool:
        """Check if a request can be executed."""
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            # Check if cooldown has expired
            if self._cooldown_expired():
                self.state = CircuitState.HALF_OPEN
                logger.info(
                    "circuit_breaker_half_open",
                    provider=self.provider,
                )
                return True
            return False
        
        # HALF_OPEN: allow one test request
        return True
    
    def record_success(self):
        """Record a successful request."""
        if self.state == CircuitState.HALF_OPEN:
            logger.info(
                "circuit_breaker_closed",
                provider=self.provider,
                message="Provider recovered",
            )
            self.state = CircuitState.CLOSED
            self.failure_count = 0
        
        if self.state == CircuitState.CLOSED:
            self.failure_count = 0
    
    def record_failure(self):
        """Record a failed request."""
        self.failure_count += 1
        self.last_failure_time = datetime.utcnow()
        
        if self.state == CircuitState.HALF_OPEN:
            # Test request failed, go back to OPEN
            logger.warning(
                "circuit_breaker_reopened",
                provider=self.provider,
                message="Provider not yet recovered",
            )
            self.state = CircuitState.OPEN
            return
        
        if self.state == CircuitState.CLOSED:
            if self.failure_count >= self.failure_threshold:
                logger.error(
                    "circuit_breaker_open",
                    provider=self.provider,
                    failure_count=self.failure_count,
                    threshold=self.failure_threshold,
                )
                self.state = CircuitState.OPEN
```

---

## Finding What You Need

### Quick Reference: Where to Look

| What You Need | Where to Find It |
|---------------|-----------------|
| How a request is handled | `genai-platform-core/src/genai_platform/orchestrator/handlers/` |
| Guardrails check implementation | `genai-platform-guardrails/src/genai_platform/guardrails/checks/` |
| Consumer configuration | `genai-platform-api-gateway/config/consumers/` |
| Feature flags | `genai-platform-core/src/genai_platform/config/feature_flags.py` |
| LLM provider routing | `genai-platform-core/src/genai_platform/llm/router.py` |
| RAG retrieval logic | `genai-platform-core/src/genai_platform/rag/retriever.py` |
| Document indexing pipeline | `genai-platform-document-indexer/src/genai_platform/indexer/` |
| Audit logging | `genai-platform-audit/src/genai_platform/audit/logger.py` |
| Cost tracking | `genai-platform-cost-tracker/src/genai_platform/cost_tracker/` |
| Infrastructure (K8s) | `genai-platform-infra/helm/` |
| Deployment definitions | `genai-platform-infra/argocd/` |
| Terraform (cloud resources) | `genai-platform-infra/terraform/` |
| API contract | `genai-platform-core/openapi.yaml` |
| Client SDK | `genai-platform-client-sdk/python/` |
| Test fixtures | `genai-platform-*/tests/fixtures/` |
| Runbooks | `genai-platform-docs/runbooks/` |
| Architecture decisions | `genai-platform-docs/architecture/adrs/` |

### Search Tips

```bash
# Find all usages of a function
rg "def check_prompt" --type py

# Find all imports of a module
rg "from genai_platform.guardrails" --type py

# Find TODO items assigned to nobody
rg "TODO(?!.*\()" --type py

# Find TODO items assigned to you
rg "TODO.*yourname" --type py

# Find recent changes to a file
git log -p --follow -- src/genai_platform/guardrails/pipeline.py | head -200

# Find who last changed a specific line
git blame src/genai_platform/guardrails/pipeline.py

# Find all PRs that touched a file
gh pr list --search "file:guardrails/pipeline.py"
```

### Understanding Git History

```bash
# See when a specific line was last changed
git log -p -S "PromptInjectionDetector" -- src/

# See the evolution of a file over time
git log --follow --oneline -- src/genai_platform/guardrails/pipeline.py

# Find the commit that introduced a bug (bisect)
git bisect start
git bisect bad HEAD          # Current version is broken
git bisect good v2.13.0      # Previous release was fine
# Git will checkout commits for you to test
# Run your test, then:
git bisect good   # or git bisect bad
# Repeat until git identifies the first bad commit
```

---

## Code Review: What We Look For

When your code is reviewed, reviewers will check:

### Must-Have (will block the PR)

- [ ] Correctness: The code does what it claims to do
- [ ] Security: No secrets exposed, input validated, output sanitized
- [ ] Compliance: Audit logging in place, data handling follows policy
- [ ] Tests: Unit tests for logic, integration tests for interactions
- [ ] Error handling: Failures are handled gracefully, not silently swallowed

### Should-Have (will be requested but may not block)

- [ ] Readability: Another engineer can understand the code in 6 months
- [ ] Consistency: Follows existing patterns in the codebase
- [ ] Performance: No obvious inefficiencies (O(n^2) where O(n) is possible)
- [ ] Observability: Structured logging at key decision points

### Nice-to-Have (suggestions, not blockers)

- [ ] Elegance: Could this be written more cleanly?
- [ ] Extensibility: Will this design support future changes?
- [ ] Documentation: Are complex parts explained?

---

## Anti-Patterns to Avoid

### 1. God Classes

```python
# BAD: This class does too much
class Orchestrator:
    def handle_request(self, ...):
        # 200 lines of code doing everything
    
# GOOD: Separate concerns
class RequestHandler:
    def handle(self, ...):
        # Coordinates other components
        pass

class PromptBuilder:
    def build(self, ...):
        # Constructs the prompt
        pass

class ResponseValidator:
    def validate(self, ...):
        # Validates the response
        pass
```

### 2. Silent Failures

```python
# BAD: Exception caught and ignored
try:
    await self.guardrails.check(prompt)
except Exception:
    pass  # Request proceeds without guardrails!

# GOOD: Exception logged and request denied
try:
    result = await self.guardrails.check(prompt)
except Exception as e:
    logger.error(
        "guardrails_check_failed",
        error=str(e),
        action="denying_request",
        request_id=request_id,
    )
    return {"status": "denied", "reason": "Guardrails service unavailable"}
```

### 3. Hard-Coded Configuration

```python
# BAD: Configuration in code
GUARDRAILS_URL = "https://guardrails.platform.bank:8082"
INJECTION_THRESHOLD = 0.85

# GOOD: Configuration from config service
config = ConfigLoader.load()
guardrails_url = config.get("guardrails.url")
injection_threshold = config.get("guardrails.injection_threshold")
```

### 4. Missing Audit Trail

```python
# BAD: Decision made without logging
if confidence > 0.85:
    return {"action": "deny"}

# GOOD: Every decision is logged
if confidence > 0.85:
    logger.info(
        "guardrails_decision",
        action="deny",
        confidence=confidence,
        request_id=request_id,
    )
    self.audit.log_decision(request_id, "deny", confidence)
    return {"action": "deny"}
```

---

## Further Reading

- `understanding-the-platform.md` — Platform architecture overview
- `understanding-the-deployment-pipeline.md` — How code reaches production
- `architecture/adrs/` — Architecture Decision Records
- `backend-engineering/code-style.md` — Coding standards and conventions

---

*Last updated: April 2025 | Document owner: Engineering Enablement Team | Review cycle: Quarterly*
