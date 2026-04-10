# Cost Optimization

Managing token costs at enterprise scale is critical for GenAI platform sustainability. This guide covers strategies for reducing, tracking, and optimizing GenAI costs in a global banking environment.

## Cost Structure

### Model Pricing Comparison (per 1M tokens)

| Model | Input | Output | Cache Read | Cache Write | Context Window |
|-------|-------|--------|-----------|------------|----------------|
| GPT-4o | $2.50 | $10.00 | $1.25 | $3.75 | 128K |
| GPT-4o-mini | $0.15 | $0.60 | $0.075 | $0.225 | 128K |
| GPT-4 Turbo | $10.00 | $30.00 | $5.00 | $15.00 | 128K |
| Claude 3.5 Sonnet | $3.00 | $15.00 | — | — | 200K |
| Claude 3.5 Haiku | $0.80 | $4.00 | — | — | 200K |
| Gemini 1.5 Pro | $3.50 | $10.50 | — | — | 1M |
| Gemini 1.5 Flash | $0.075 | $0.30 | — | — | 1M |
| Llama 3 70B (self-hosted) | ~$0.50* | ~$0.50* | — | — | 8K-128K |

*Self-hosted cost varies by infrastructure. Estimate based on GPU compute cost.

### Understanding Cost Drivers

```python
def calculate_request_cost(
    input_tokens: int,
    output_tokens: int,
    model: str,
    cached: bool = False,
) -> float:
    """Calculate cost of a single request."""
    pricing = {
        "gpt-4o": {"input": 2.50, "output": 10.00, "cache_read": 1.25, "cache_write": 3.75},
        "gpt-4o-mini": {"input": 0.15, "output": 0.60, "cache_read": 0.075, "cache_write": 0.225},
        "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
        "claude-3-5-haiku": {"input": 0.80, "output": 4.00},
        "gemini-1.5-pro": {"input": 3.50, "output": 10.50},
    }

    rates = pricing[model]

    if cached:
        input_cost = input_tokens / 1_000_000 * rates.get("cache_read", rates["input"])
    else:
        input_cost = input_tokens / 1_000_000 * rates["input"]

    output_cost = output_tokens / 1_000_000 * rates["output"]

    return input_cost + output_cost
```

## Cost Reduction Strategies

### Strategy 1: Model Routing

```python
# Route to cheapest appropriate model based on task complexity

MODEL_ROUTING_RULES = {
    "simple_classification": {
        "primary": "gpt-4o-mini",      # $0.15/M input
        "fallback": "claude-3-5-haiku",  # $0.80/M input
        "quality_threshold": 0.95,      # If accuracy < 95%, upgrade
    },
    "document_summarization": {
        "primary": "gpt-4o",
        "fallback": "gemini-1.5-flash",
        "quality_threshold": 0.90,
    },
    "compliance_analysis": {
        "primary": "claude-3-5-sonnet",  # Better reasoning
        "fallback": "gpt-4o",
        "quality_threshold": 0.95,
    },
    "creative_content": {
        "primary": "gpt-4o",
        "fallback": "claude-3-5-sonnet",
        "quality_threshold": None,  # Subjective
    },
    "code_generation": {
        "primary": "claude-3-5-sonnet",
        "fallback": "gpt-4o",
        "quality_threshold": None,
    },
}

# Potential savings: 50-80% by routing simple tasks to smaller models
```

### Strategy 2: Prompt Optimization

```python
class PromptCostOptimizer:
    """Optimize prompts to minimize token usage."""

    def __init__(self):
        self.token_counter = TokenCounter()

    def optimize_prompt(self, prompt: str) -> str:
        """Remove unnecessary tokens from prompt."""
        original_tokens = self.token_counter.count(prompt)

        # Remove redundant instructions
        optimized = self._remove_redundancies(prompt)

        # Compress verbose text
        optimized = self._compress_text(optimized)

        # Remove unnecessary formatting
        optimized = self._clean_formatting(optimized)

        new_tokens = self.token_counter.count(optimized)
        savings = (original_tokens - new_tokens) / original_tokens * 100

        return {
            "original": prompt,
            "optimized": optimized,
            "original_tokens": original_tokens,
            "optimized_tokens": new_tokens,
            "savings_percent": savings,
        }

    def _remove_redundancies(self, text: str) -> str:
        """Remove redundant instructions."""
        # Common redundancies in system prompts
        redundancies = [
            # Saying the same thing multiple ways
            ("You should be professional and courteous. "
             "Always maintain a professional and courteous demeanor.",
             "Be professional and courteous."),
            ("Do not make up information. Do not hallucinate. "
             "Do not fabricate facts. Only use provided information.",
             "Only use provided information. Do not fabricate facts."),
        ]
        for redundant, replacement in redundancies:
            text = text.replace(redundant, replacement)
        return text
```

### Strategy 3: Response Caching

```python
# See caching.md for full implementation
# Key cost savings from caching: 30-60% reduction for repeated queries

# Example: Common customer queries that are cacheable
CACHEABLE_PATTERNS = [
    "What are your branch opening hours?",
    "How do I reset my online banking password?",
    "What is the current mortgage rate?",
    "How do I report a lost card?",
    "What documents do I need to open an account?",
]

# If 30% of queries are cacheable, and cache hit rate is 60%:
# Effective cost reduction = 30% * 60% = 18% cost savings
```

### Strategy 4: Token Budget Management

```python
class TokenBudgetManager:
    """Manage token budgets per team, project, and application."""

    def __init__(self, db_client):
        self.db = db_client

    def set_budget(self, team: str, project: str, monthly_budget_usd: float):
        """Set monthly token budget."""
        month = datetime.now().strftime("%Y-%m")
        self.db.upsert("token_budgets", {
            "team": team,
            "project": project,
            "month": month,
            "budget_usd": monthly_budget_usd,
            "alerts_at": [0.5, 0.75, 0.9, 1.0],  # Alert thresholds
        })

    def check_and_alert(self, team: str, project: str) -> list[dict]:
        """Check current spend and generate alerts."""
        month = datetime.now().strftime("%Y-%m")
        budget = self.db.get("token_budgets", team=team, project=project, month=month)

        if not budget:
            return []

        current_spend = self.db.get_monthly_spend(team, project, month)
        utilization = current_spend / budget["budget_usd"]

        alerts = []
        for threshold in sorted(budget["alerts_at"]):
            if utilization >= threshold and not self._alert_sent(team, project, threshold):
                alerts.append({
                    "team": team,
                    "project": project,
                    "threshold": threshold,
                    "current_spend": current_spend,
                    "budget": budget["budget_usd"],
                    "utilization": utilization,
                    "projected_monthly": current_spend / (day_of_month / days_in_month),
                })
                self._mark_alert_sent(team, project, threshold)

        return alerts

    def enforce_budget(self, team: str, project: str) -> bool:
        """Return False if budget is exceeded (block requests)."""
        month = datetime.now().strftime("%Y-%m")
        budget = self.db.get("token_budgets", team=team, project=project, month=month)

        if not budget:
            return True  # No budget set, allow

        current_spend = self.db.get_monthly_spend(team, project, month)
        return current_spend < budget["budget_usd"]
```

### Strategy 5: Batch Processing

```python
# Process multiple requests in a single API call when possible

class BatchProcessor:
    """Batch similar requests to reduce API overhead."""

    def __init__(self, max_batch_size: int = 20, max_wait_seconds: float = 5.0):
        self.max_batch_size = max_batch_size
        self.max_wait = max_wait_seconds
        self.queue: list[dict] = []
        self.lock = asyncio.Lock()

    async def add_to_batch(self, request: dict) -> asyncio.Future:
        """Add request to batch queue."""
        future = asyncio.get_event_loop().create_future()

        async with self.lock:
            self.queue.append({"request": request, "future": future})

            if len(self.queue) >= self.max_batch_size:
                # Process immediately
                await self._process_batch()
            elif len(self.queue) == 1:
                # First item, schedule processing
                asyncio.get_event_loop().call_later(
                    self.max_wait,
                    lambda: asyncio.ensure_future(self._process_batch()),
                )

        return future

    async def _process_batch(self):
        """Process all queued requests in a single batch."""
        async with self.lock:
            batch = self.queue.copy()
            self.queue.clear()

        if not batch:
            return

        # Combine into single API call where possible
        # (e.g., batch embedding requests)
        texts = [item["request"]["text"] for item in batch]
        embeddings = await embedding_client.embed_batch(texts)

        for item, embedding in zip(batch, embeddings):
            item["future"].set_result(embedding)
```

## Cost Monitoring and Alerting

### Real-Time Cost Dashboard

```python
class CostDashboard:
    """Real-time cost monitoring dashboard."""

    async def get_current_costs(self, granularity: str = "hour") -> dict:
        """Get current cost metrics."""
        now = datetime.utcnow()

        costs = {
            "current_hour": await self._get_costs(now - timedelta(hours=1), now),
            "today": await self._get_costs(now.replace(hour=0, minute=0), now),
            "this_month": await self._get_costs(
                now.replace(day=1, hour=0, minute=0), now
            ),
            "by_model": await self._get_costs_by_model(now),
            "by_team": await self._get_costs_by_team(now),
            "by_application": await self._get_costs_by_application(now),
            "top_expensive_requests": await self._get_top_expensive_requests(now),
        }

        return costs

    async def get_forecast(self) -> dict:
        """Forecast end-of-month costs based on current trajectory."""
        month_start = datetime.utcnow().replace(day=1, hour=0, minute=0)
        days_elapsed = (datetime.utcnow() - month_start).days + 1
        days_in_month = calendar.monthrange(datetime.utcnow().year, datetime.utcnow().month)[1]

        current_month_spend = await self._get_month_to_date_spend()
        daily_average = current_month_spend / days_elapsed

        forecast = daily_average * days_in_month

        return {
            "month_to_date": current_month_spend,
            "daily_average": daily_average,
            "forecasted_monthly": forecast,
            "days_remaining": days_in_month - days_elapsed,
            "projected_overage": max(0, forecast - await self._get_monthly_budget()),
        }
```

### Cost Anomaly Detection

```python
class CostAnomalyDetector:
    """Detect unusual spikes in token costs."""

    def __init__(self, stats_client):
        self.stats = stats_client

    def check_for_anomalies(self) -> list[dict]:
        """Check for cost anomalies."""
        anomalies = []

        # Check hourly costs against historical average
        current_hour_cost = self.stats.get_current_hour_cost()
        historical_avg = self.stats.get_historical_avg_hour(
            day_of_week=datetime.utcnow().weekday(),
            hour=datetime.utcnow().hour,
        )

        if current_hour_cost > historical_avg * 3:  # 3x normal
            anomalies.append({
                "type": "hourly_spike",
                "current": current_hour_cost,
                "expected": historical_avg,
                "multiplier": current_hour_cost / historical_avg,
                "severity": "HIGH",
            })

        # Check daily run rate
        daily_run_rate = self.stats.get_daily_run_rate()
        monthly_budget = self.stats.get_monthly_budget()
        daily_budget = monthly_budget / 30

        if daily_run_rate > daily_budget * 1.5:
            anomalies.append({
                "type": "daily_budget_exceeded",
                "current_run_rate": daily_run_rate,
                "daily_budget": daily_budget,
                "severity": "CRITICAL",
            })

        return anomalies
```

## Common Mistakes and Anti-Patterns

### Anti-Pattern 1: No Cost Visibility

```python
# WRONG: Single API key shared across all teams
# No way to attribute costs to specific teams or applications

# RIGHT: Per-team API keys with cost attribution
TEAM_API_KEYS = {
    "customer_service": "sk-cs-...",
    "compliance": "sk-comp-...",
    "engineering": "sk-eng-...",
}

# Track costs per key
for key_name, api_key in TEAM_API_KEYS.items():
    monthly_cost = calculate_monthly_cost(api_key)
    logger.info(f"Team {key_name} monthly cost: ${monthly_cost:.2f}")
```

### Anti-Pattern 2: Using GPT-4 for Everything

```python
# WRONG: Using GPT-4 ($10/$30 per 1M) for simple classification
response = gpt4.classify_transaction(transaction)  # Overkill

# RIGHT: Use appropriate model
task_complexity = assess_complexity(transaction)
if task_complexity == "simple":
    response = gpt4o_mini.classify_transaction(transaction)  # $0.15/$0.60
elif task_complexity == "medium":
    response = gpt4o.classify_transaction(transaction)  # $2.50/$10
else:
    response = claude_sonnet.classify_transaction(transaction)  # $3/$15

# Savings: 80-95% for simple tasks
```

### Anti-Pattern 3: Not Caching Embeddings

```python
# WRONG: Re-embedding the same documents repeatedly
# Every time a user searches, you embed their query
# If 100 users search the same thing, you pay 100x

# RIGHT: Cache embeddings for common queries
# "What are your branch hours?" — embed once, cache forever

embedding_cache = {}
def get_cached_embedding(text: str) -> list[float]:
    # Normalize text for cache lookup
    normalized = text.lower().strip()

    if normalized in embedding_cache:
        return embedding_cache[normalized]

    embedding = embedding_client.embed(text)
    embedding_cache[normalized] = embedding
    return embedding

# Savings: 40-60% reduction in embedding costs
```

### Anti-Pattern 4: Unbounded Context

```python
# WRONG: Always sending full conversation history
# Context grows with every turn, costs increase over time

messages = [
    {"role": "system", "content": system_prompt},  # 500 tokens
    {"role": "user", "content": "Hello"},  # 1 token
    {"role": "assistant", "content": "Hi! How can I help?"},  # 10 tokens
    # ... 20 more turns ...
    # Total: 5,000+ tokens and growing

# RIGHT: Summarize old turns, keep recent context
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "system", "content": "Previous conversation summary: User asked about mortgage rates..."},
    # Keep only last 5 turns in full
    {"role": "user", "content": "What about home insurance?"},
]
# Total: 1,500 tokens — stable regardless of conversation length
```

## Interview Questions

1. How would you reduce GenAI costs by 50% without impacting quality?
2. A team's monthly GenAI costs jumped from $5K to $50K. How do you investigate?
3. What is the ROI calculation for fine-tuning vs. using API models?
4. How do you allocate GenAI costs across teams in a shared platform?
5. Design a cost monitoring and alerting system for enterprise GenAI usage.

## Cross-References

- [tokenization.md](./tokenization.md) — Token counting and optimization
- [model-routing.md](./model-routing.md) — Routing to cost-appropriate models
- [caching.md](./caching.md) — Caching to reduce API calls
- [multi-model-architecture.md](./multi-model-architecture.md) — Multi-provider cost optimization
- [cost-optimization.md](./cost-optimization.md) — This file
- [../observability/](../observability/) — Cost monitoring and alerting
