# System Design: AI Model Gateway with Routing, Caching, and Monitoring

## Problem Statement

Design a centralized AI model gateway that serves all AI-consuming applications within the bank. The gateway should provide unified access to multiple LLM providers (OpenAI, Anthropic, Google, open-source models), intelligent routing based on cost and quality, request/response caching, usage monitoring, and cost management. The bank currently has 15+ teams independently calling AI APIs with no coordination, resulting in uncontrolled costs and inconsistent quality.

## Requirements

### Functional Requirements
1. Unified API for accessing multiple LLM providers (OpenAI, Anthropic, Google, self-hosted)
2. Intelligent request routing based on model capability, cost, and availability
3. Semantic caching of LLM responses
4. Per-team/per-project usage tracking and budget management
5. Rate limiting and quota management
6. Request/response logging for audit and billing
7. Model fallback (if primary provider fails, try secondary)
8. Prompt template management and versioning
9. PII detection and redaction before external API calls
10. Real-time cost dashboard

### Non-Functional Requirements
1. Gateway latency overhead: < 50ms (excluding LLM call time)
2. Availability: 99.99% (critical infrastructure)
3. Support 50+ consuming applications, 2M+ API calls/day
4. All external API calls must be logged
5. Budget enforcement: hard stop when team exceeds budget
6. Multi-region deployment for disaster recovery
7. Zero-downtime model version updates

## Architecture

```mermaid
flowchart TB
    subgraph Consumers
        APP1[Chatbot App]
        APP2[Analytics App]
        APP3[Compliance App]
        APP4[Dev Tools]
    end
    
    subgraph API Gateway
        LB[Load Balancer]
        GATEWAY[Gateway Service]
        AUTH[API Key Auth]
        RATE[Rate Limiter]
    end
    
    subgraph Gateway Core
        ROUTER[Model Router]
        CACHE[Response Cache]
        PII[PII Redactor]
        RETRY[Retry/Fallback]
    end
    
    sub observability
        TRACKER[Usage Tracker]
        COST[Cost Calculator]
        MONITOR[Health Monitor]
        ALERT[Alerting]
    end
    
    subgraph Storage
        REDIS[(Redis Cache)]
        PG[(PostgreSQL - Logs, Budgets)]
        PROM[(Prometheus - Metrics)]
    end
    
    subgraph LLM Providers
        OPENAI[OpenAI API]
        ANTHROPIC[Anthropic API]
        GOOGLE[Google AI API]
        SELFHOSTED[Self-Hosted Models]
    end
    
    APP1 --> LB
    APP2 --> LB
    APP3 --> LB
    APP4 --> LB
    
    LB --> GATEWAY
    GATEWAY --> AUTH
    AUTH --> RATE
    RATE --> PII
    PII --> CACHE
    CACHE --> ROUTER
    ROUTER --> RETRY
    
    RETRY --> OPENAI
    RETRY --> ANTHROPIC
    RETRY --> GOOGLE
    RETRY --> SELFHOSTED
    
    GATEWAY --> TRACKER
    TRACKER --> COST
    TRACKER --> MONITOR
    MONITOR --> ALERT
    
    CACHE --> REDIS
    TRACKER --> PG
    MONITOR --> PROM
```

## Detailed Design

### 1. Model Router

```python
class ModelRouter:
    """Route requests to the optimal LLM provider."""
    
    def __init__(self, config, cost_tracker, health_monitor):
        self.config = config  # Model registry and routing rules
        self.cost_tracker = cost_tracker
        self.health = health_monitor
    
    def select_model(self, request: GatewayRequest) -> ModelConfig:
        """Select the best model for this request."""
        
        # Check if request specifies a model
        if request.model:
            return self.config.get_model(request.model)
        
        # Intelligent routing based on request characteristics
        candidates = self._get_candidate_models(request)
        
        # Score each candidate
        scored = []
        for model in candidates:
            score = self._score_model(model, request)
            scored.append((score, model))
        
        # Pick highest scoring available model
        scored.sort(reverse=True, key=lambda x: x[0])
        
        for score, model in scored:
            if self.health.is_healthy(model.provider):
                # Check budget
                if self.cost_tracker.within_budget(request.team_id, model):
                    return model
        
        # Fallback to cheapest available model
        return self._get_cheapest_available_model()
    
    def _score_model(self, model: ModelConfig, request: GatewayRequest) -> float:
        """Score a model for this request."""
        
        score = 0.0
        
        # Capability match (0-40 points)
        if model.max_tokens >= request.expected_output_tokens:
            score += 20
        if model.supports_streaming and request.streaming:
            score += 10
        if model.context_window >= request.total_tokens:
            score += 10
        
        # Quality (0-30 points)
        quality_scores = {"gpt-4o": 30, "claude-sonnet": 28, "gpt-4o-mini": 20, "llama-3-70b": 18}
        score += quality_scores.get(model.name, 10)
        
        # Cost efficiency (0-20 points) - inverse of cost
        cost_per_1m = model.input_cost_per_1m + model.output_cost_per_1m
        score += max(0, 20 - (cost_per_1m / 5))  # Lower cost = higher score
        
        # Latency (0-10 points)
        avg_latency = self.health.get_avg_latency(model.provider)
        score += max(0, 10 - (avg_latency / 500))  # Lower latency = higher score
        
        return score
```

### 2. Semantic Response Cache

```python
class SemanticCache:
    """Cache LLM responses using semantic similarity."""
    
    def __init__(self, redis_client, embedding_model, similarity_threshold=0.95):
        self.redis = redis_client
        self.embedder = embedding_model
        self.threshold = similarity_threshold
        self.ttl = 86400  # 24 hours
    
    def get(self, request: GatewayRequest) -> str | None:
        """Find a cached response for this request."""
        
        # Exact match first
        exact_key = self._exact_cache_key(request)
        exact = self.redis.get(exact_key)
        if exact:
            return json.loads(exact)["response"]
        
        # Semantic match
        query_embedding = self.embedder.encode(request.prompt)
        
        # Search cached embeddings
        cached_items = self.redis.hgetall("cache_embeddings")
        
        best_match = None
        best_score = 0
        
        for cache_key, cached_embedding_json in cached_items.items():
            cached_embedding = json.loads(cached_embedding_json)
            similarity = cosine_similarity(query_embedding, cached_embedding)
            
            if similarity > best_score and similarity >= self.threshold:
                best_score = similarity
                best_match = cache_key
        
        if best_match:
            cached_response = self.redis.get(f"cache_response:{best_match}")
            if cached_response:
                return json.loads(cached_response)["response"]
        
        return None
    
    def set(self, request: GatewayRequest, response: str):
        """Cache a response."""
        
        exact_key = self._exact_cache_key(request)
        embedding = self.embedder.encode(request.prompt)
        cache_id = hashlib.sha256(request.prompt.encode()).hexdigest()[:12]
        
        self.redis.setex(
            exact_key, self.ttl,
            json.dumps({"response": response, "model": request.model, "cached_at": datetime.utcnow().isoformat()})
        )
        
        self.redis.hset("cache_embeddings", f"cache_embedding:{cache_id}", json.dumps(embedding.tolist()))
        self.redis.setex(f"cache_response:{cache_id}", self.ttl,
                        json.dumps({"response": response, "prompt_hash": cache_id}))
    
    def _exact_cache_key(self, request: GatewayRequest) -> str:
        """Generate exact match cache key."""
        content = f"{request.model}:{request.prompt}:{request.temperature}:{request.max_tokens}"
        return f"cache:exact:{hashlib.sha256(content.encode()).hexdigest()}"
```

### 3. Budget Management

```python
class BudgetManager:
    """Track and enforce per-team budgets."""
    
    def __init__(self, db):
        self.db = db
    
    def check_budget(self, team_id: str, model: ModelConfig, 
                     estimated_cost: float) -> tuple[bool, str]:
        """Check if a request is within budget."""
        
        budget = self.db.query("""
            SELECT monthly_budget, current_month_spend, hard_limit
            FROM team_budgets
            WHERE team_id = %s AND month = %s
        """, (team_id, datetime.utcnow().strftime("%Y-%m")))
        
        if not budget:
            return False, "No budget configured for this team"
        
        remaining = budget["monthly_budget"] - budget["current_month_spend"]
        
        if remaining <= 0:
            return False, f"Monthly budget exhausted. Spent ${budget['current_month_spend']:.2f} of ${budget['monthly_budget']:.2f}"
        
        if estimated_cost > remaining and budget["hard_limit"]:
            return False, f"Request would exceed budget. Remaining: ${remaining:.2f}, estimated cost: ${estimated_cost:.2f}"
        
        return True, f"Within budget. Remaining: ${remaining:.2f}"
    
    def record_usage(self, team_id: str, model: ModelConfig, 
                     input_tokens: int, output_tokens: int, cost: float):
        """Record usage for billing."""
        
        self.db.execute("""
            INSERT INTO usage_logs 
            (team_id, model, input_tokens, output_tokens, cost, timestamp)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, (team_id, model.name, input_tokens, output_tokens, cost, datetime.utcnow()))
        
        self.db.execute("""
            UPDATE team_budgets 
            SET current_month_spend = current_month_spend + %s
            WHERE team_id = %s AND month = %s
        """, (cost, team_id, datetime.utcnow().strftime("%Y-%m")))
        
        # Alert if approaching budget limit
        budget = self.get_budget(team_id)
        if budget["current_month_spend"] > budget["monthly_budget"] * 0.8:
            self._send_budget_alert(team_id, budget)
```

### 4. Fallback and Retry

```python
class FallbackHandler:
    """Handle provider failures with automatic fallback."""
    
    def __init__(self, model_config):
        self.fallback_chain = {
            "gpt-4o": ["claude-sonnet", "gpt-4o-mini"],
            "claude-sonnet": ["gpt-4o", "gpt-4o-mini"],
            "gpt-4o-mini": ["claude-haiku", "llama-3-70b"],
        }
    
    def execute_with_fallback(self, request: GatewayRequest) -> GatewayResponse:
        """Try primary model, fall back to alternatives on failure."""
        
        primary_model = request.model
        last_error = None
        
        for model_name in [primary_model] + self.fallback_chain.get(primary_model, []):
            try:
                provider = self.get_provider(model_name)
                response = provider.generate(
                    prompt=request.prompt,
                    model=model_name,
                    temperature=request.temperature,
                    max_tokens=request.max_tokens,
                    timeout=30  # seconds
                )
                
                # If we fell back, log the incident
                if model_name != primary_model:
                    log_fallback_incident(primary_model, model_name, last_error)
                
                return GatewayResponse(
                    text=response.text,
                    model=model_name,
                    input_tokens=response.usage.input_tokens,
                    output_tokens=response.usage.output_tokens,
                    fallback_used=model_name != primary_model
                )
                
            except (TimeoutError, ConnectionError) as e:
                last_error = e
                log_provider_error(model_name, e)
                continue
            except Exception as e:
                # Non-retriable errors (auth, rate limit)
                raise e
        
        # All models failed
        raise AllProvidersFailedError(f"All providers failed for request. Last error: {last_error}")
```

### 5. Monitoring and Cost Dashboard

```python
class MonitoringService:
    """Real-time monitoring of the AI gateway."""
    
    def get_dashboard(self, timeframe: str = "24h") -> dict:
        """Generate monitoring dashboard data."""
        
        return {
            "total_requests": self.get_request_count(timeframe),
            "total_cost": self.get_total_cost(timeframe),
            "cost_breakdown": self.get_cost_by_team(timeframe),
            "model_usage": self.get_usage_by_model(timeframe),
            "latency": {
                "p50": self.get_latency_percentile(50, timeframe),
                "p95": self.get_latency_percentile(95, timeframe),
                "p99": self.get_latency_percentile(99, timeframe),
            },
            "error_rate": self.get_error_rate(timeframe),
            "cache_hit_rate": self.get_cache_hit_rate(timeframe),
            "fallback_rate": self.get_fallback_rate(timeframe),
            "budget_status": self.get_budget_status(),
            "provider_health": self.get_provider_health(),
            "top_teams_by_spend": self.get_top_teams_by_spend(timeframe),
            "top_queries_by_cost": self.get_top_expensive_queries(timeframe),
        }
```

## Tradeoffs

### Gateway Technology: Custom vs. API Gateway Product

| Criteria | Custom Gateway | Kong/APISIX | Cloud API Gateway |
|---|---|---|---|
| **Flexibility** | Full control | Plugin-based | Limited |
| **LLM-specific features** | Built-in | Requires plugins | Not available |
| **Cost tracking** | Native | Requires setup | Requires setup |
| **Development effort** | High | Medium | Low |
| **Maintenance** | Our responsibility | Vendor + ours | Vendor |
| **Decision** | **SELECTED** | Rejected | Rejected |

**Rationale**: LLM gateway needs (semantic caching, model routing, cost tracking, PII redaction) are too specialized for generic API gateways. Building a custom gateway provides the exact features needed.

### Caching: Exact vs. Semantic

- **Exact match cache**: Simple, fast, but low hit rate (20-30%)
- **Semantic cache**: Higher hit rate (30-50%) but slower and more complex
- **Decision**: Both -- exact match first (fast path), then semantic (slower path)

## Bottlenecks and Scaling

| Component | Scale | Bottleneck | Solution |
|---|---|---|---|
| Gateway service | 5000 QPS per instance | CPU for PII detection | Horizontal scaling, load balancer |
| Redis cache | 200K ops/sec | Memory limit | Redis Cluster |
| PostgreSQL (logs) | 10K writes/sec | Write throughput | Partition by date, async writes |
| External API calls | Provider rate limits | OpenAI/Anthropic RPM | Multiple API keys, queue |

## Security

1. **API key management**: Per-team API keys with scoped permissions
2. **PII redaction**: All prompts scanned before external API calls
3. **Audit logging**: Every request/response logged with team ID
4. **Budget enforcement**: Hard stops prevent cost overruns
5. **Network isolation**: Gateway in private subnet, LLM providers accessed via NAT

## Interview Questions

### Q: How do you handle a situation where a team's budget is exhausted but they have a critical production need?

**Strong Answer**: "I implement a tiered budget system: (1) Soft limit at 80% -- send alerts to team lead. (2) Hard limit at 100% -- block non-critical requests but allow critical ones with override. (3) Emergency override: Team lead can authorize additional budget for a defined period with VP approval. The override is logged, audited, and triggers a budget review conversation. This balances cost control with operational flexibility. I also implement a 'priority queue' where critical production requests (identified by service level tags) bypass the budget check but are still tracked and reported."

### Q: How would you detect and prevent prompt injection attacks through the gateway?

**Strong Answer**: "Multiple layers: (1) Input validation -- detect common prompt injection patterns (system prompt overrides, role-play attempts). (2) PII redaction prevents data exfiltration. (3) Output validation -- check for responses that contain system instructions or unexpected content types. (4) Rate limiting per API key prevents rapid exploitation. (5) Anomaly detection -- flag teams or users with unusual prompt patterns (very long prompts, many system message attempts). (6) Log all prompts and responses for post-incident analysis. The key defense is defense-in-depth -- no single layer is sufficient."
