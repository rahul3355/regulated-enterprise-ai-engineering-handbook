# Token Usage Tracking and Cost Monitoring

## Why Token Tracking Matters

LLM API calls are priced per token. In a banking GenAI platform serving thousands of customer queries daily, token costs can reach tens of thousands of dollars per month. Without per-query, per-service, and per-team token tracking, costs spiral without visibility.

## Token Usage Metrics

### Basic Tracking

```python
from prometheus_client import Counter, Histogram

# Total token consumption
token_usage_total = Counter(
    'token_usage_total',
    'Total tokens consumed',
    ['model', 'provider', 'direction', 'product_line', 'user_tier']
    # direction: 'input' (prompt) or 'output' (completion)
)

# Tokens per request
tokens_per_request = Histogram(
    'tokens_per_request',
    'Tokens per request',
    ['model', 'direction'],
    buckets=[10, 25, 50, 100, 250, 500, 1000, 2000, 5000, 10000, 20000]
)

def record_token_usage(
    model: str,
    provider: str,
    prompt_tokens: int,
    completion_tokens: int,
    product_line: str = None,
    user_tier: str = None
):
    """Record token usage for a single LLM call."""
    for direction, count in [('input', prompt_tokens), ('output', completion_tokens)]:
        token_usage_total.labels(
            model=model, provider=provider, direction=direction,
            product_line=product_line, user_tier=user_tier
        ).inc(count)
        tokens_per_request.labels(model=model, direction=direction).observe(count)
```

### Cost Calculation

```python
# Model pricing (update regularly as providers change prices)
MODEL_PRICING = {
    'gpt-4-turbo': {
        'provider': 'openai',
        'input_per_1k': 0.01,
        'output_per_1k': 0.03,
    },
    'gpt-3.5-turbo': {
        'provider': 'openai',
        'input_per_1k': 0.0005,
        'output_per_1k': 0.0015,
    },
    'claude-3-sonnet': {
        'provider': 'anthropic',
        'input_per_1k': 0.003,
        'output_per_1k': 0.015,
    },
    'claude-3-haiku': {
        'provider': 'anthropic',
        'input_per_1k': 0.00025,
        'output_per_1k': 0.00125,
    },
}

llm_cost_usd = Counter(
    'llm_cost_usd_total',
    'Total LLM API cost in USD',
    ['model', 'provider', 'product_line', 'user_tier']
)

def calculate_cost(model: str, prompt_tokens: int, completion_tokens: int) -> float:
    """Calculate cost for a single LLM call."""
    pricing = MODEL_PRICING.get(model)
    if not pricing:
        return 0.0

    cost = (
        (prompt_tokens / 1000) * pricing['input_per_1k'] +
        (completion_tokens / 1000) * pricing['output_per_1k']
    )
    return round(cost, 6)

def record_cost(model, prompt_tokens, completion_tokens, product_line=None, user_tier=None):
    """Calculate and record cost."""
    cost = calculate_cost(model, prompt_tokens, completion_tokens)
    provider = MODEL_PRICING[model]['provider']

    llm_cost_usd.labels(
        model=model, provider=provider,
        product_line=product_line, user_tier=user_tier
    ).inc(cost)

    return cost
```

## Budget Tracking

### Monthly Budget Configuration

```python
# Monthly budgets by team/product
BUDGETS = {
    'mortgage-advisor': {
        'monthly_usd': 5000,
        'alert_thresholds': [0.5, 0.75, 0.9, 1.0],
    },
    'investment-chat': {
        'monthly_usd': 3000,
        'alert_thresholds': [0.5, 0.75, 0.9, 1.0],
    },
    'credit-card-bot': {
        'monthly_usd': 2000,
        'alert_thresholds': [0.5, 0.75, 0.9, 1.0],
    },
    'general-assistant': {
        'monthly_usd': 4000,
        'alert_thresholds': [0.5, 0.75, 0.9, 1.0],
    },
    'total': {
        'monthly_usd': 15000,
        'alert_thresholds': [0.5, 0.75, 0.9, 1.0],
    },
}
```

### Budget Monitoring

```python
def get_current_month_cost(product_line: str) -> float:
    """Get total cost for current month."""
    # Query Prometheus for current month's cost
    query = f'sum(increase(llm_cost_usd_total{{product_line="{product_line}"}}[30d]))'
    return prometheus_query(query)

def check_budget_thresholds():
    """Check if any product line has exceeded budget thresholds."""
    import datetime
    day_of_month = datetime.datetime.now().day
    days_in_month = 30  # Simplified
    progress = day_of_month / days_in_month

    for product_line, budget_config in BUDGETS.items():
        current_cost = get_current_month_cost(product_line)
        budget = budget_config['monthly_usd']
        expected_progress_cost = budget * progress

        for threshold in budget_config['alert_thresholds']:
            threshold_cost = budget * threshold
            if current_cost >= threshold_cost and current_cost < threshold_cost + 10:
                # Just crossed this threshold
                send_budget_alert(product_line, threshold, current_cost, budget)
                break
```

## Cost by Customer Segment

Track cost per customer segment to understand ROI:

```python
# Cost per user tier
tier_costs = {
    'premium': {
        'monthly_cost': 4567.89,
        'queries': 12456,
        'avg_cost_per_query': 0.367,
        'revenue': 125000,
        'roi': 27.4  # revenue / cost
    },
    'standard': {
        'monthly_cost': 3456.78,
        'queries': 45678,
        'avg_cost_per_query': 0.076,
        'revenue': 45000,
        'roi': 13.0
    },
    'basic': {
        'monthly_cost': 2345.67,
        'queries': 23456,
        'avg_cost_per_query': 0.100,
        'revenue': 12000,
        'roi': 5.1
    },
}
```

## Cost Optimization Analysis

### Model Selection by Use Case

```
┌──────────────────────────────────────────────────────────────┐
│  COST vs QUALITY BY MODEL                                    │
├────────────────────┬───────────┬───────────┬─────────────────┤
│  Model             │  Cost/Q  │  Quality │  Best For       │
├────────────────────┼───────────┼───────────┼─────────────────┤
│  gpt-4-turbo       │  $0.042  │  0.94    │  Complex advice  │
│  gpt-3.5-turbo     │  $0.004  │  0.82    │  Simple queries  │
│  claude-3-sonnet   │  $0.038  │  0.93    │  Document analysis│
│  claude-3-haiku    │  $0.002  │  0.78    │  Classification  │
└────────────────────┴───────────┴───────────┴─────────────────┘

Strategy:
- Simple queries (greeting, FAQ): Use claude-3-haiku ($0.002/query)
- Mortgage advice: Use gpt-4-turbo ($0.042/query) - quality matters
- Document analysis: Use claude-3-sonnet ($0.038/query)
- Classification/routing: Use claude-3-haiku ($0.002/query)

Estimated monthly savings with optimal routing: 35%
```

### Caching Impact on Cost

```python
cache_cost_savings = Counter(
    'cache_cost_savings_usd_total',
    'Cost savings from caching (USD)',
    ['cache_type', 'model']
)

def record_cache_saving(model, prompt_tokens, completion_tokens):
    """Record the cost saved by a cache hit."""
    saved_cost = calculate_cost(model, prompt_tokens, completion_tokens)
    cache_cost_savings.labels(
        cache_type='llm_response', model=model
    ).inc(saved_cost)
```

## Cost Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  LLM COST TRACKING - MARCH 2025                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Monthly Budget: $15,000                                     │
│  Spent So Far: $12,450 (83%)                                 │
│  Daily Run Rate: $687                                        │
│  Projected Month-End: $21,297 (EXCEEDS BUDGET)               │
│  Days Remaining: 16                                          │
│  Remaining Budget: $2,550                                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Budget: [████████████████████░░░░░░░░░░░░] 83%        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Cost by Product Line:                                       │
│  Mortgage Advisor:    $5,234 (42%)  [Over budget trend]     │
│  Investment Chat:     $3,456 (28%)  [On track]              │
│  Credit Card Bot:     $2,100 (17%)  [Under budget]          │
│  General Assistant:   $1,660 (13%)  [On track]              │
│                                                              │
│  Cost by Model:                                              │
│  gpt-4-turbo:     $8,234 (66%)  - avg $0.042/query          │
│  claude-3-sonnet: $2,890 (23%)  - avg $0.038/query          │
│  gpt-3.5-turbo:   $1,156 (9%)   - avg $0.004/query          │
│  claude-3-haiku:  $  170 (2%)   - avg $0.002/query          │
│                                                              │
│  Cache Savings This Month: $3,450 (22% of potential cost)   │
└──────────────────────────────────────────────────────────────┘
```

## Budget Alerts

```yaml
- alert: LLMBudgetThresholdExceeded
  expr: |
    sum(increase(llm_cost_usd_total[30d]))
    > 15000 * 0.9
  for: 1h
  labels:
    severity: warning
    team: ml-platform
  annotations:
    summary: "LLM cost exceeded 90% of monthly budget"
    description: "Current spend: ${{ $value }} of $15,000 budget"

- alert: LLMSpikeInDailyCost
  expr: |
    sum(increase(llm_cost_usd_total[1h]))
    > 100
  for: 15m
  labels:
    severity: warning
    team: ml-platform
  annotations:
    summary: "LLM cost rate above $100/hour"
    description: "Current hourly cost rate: ${{ $value }}"
```

## Cost Per Query Optimization

```python
def analyze_cost_per_query():
    """Analyze cost per query by product line."""
    return {
        'mortgage_advisor': {
            'avg_cost_per_query': 0.042,
            'median_prompt_tokens': 1200,
            'median_completion_tokens': 450,
            'optimization_opportunities': [
                'Reduce context documents from 10 to 5 (save ~300 input tokens)',
                'Use gpt-3.5-turbo for follow-up questions (save 80%)',
                'Increase cache hit rate from 20% to 35%'
            ]
        }
    }
```

## Common Cost Tracking Mistakes

1. **Not tracking by product line**: If you only know total cost, you cannot identify which feature is overspending.

2. **Ignoring embedding costs**: Embedding models also cost money. At scale, embedding API costs are significant.

3. **No budget alerts**: Discovering you are over budget at month-end is too late. Alert at 50%, 75%, and 90%.

4. **Not tracking cache savings**: Cache hit rate directly translates to cost savings. Track it to justify caching investment.

5. **No cost-per-query metric**: Without knowing the average cost per query, you cannot evaluate the ROI of a GenAI feature.

6. **Forgetting streaming overhead**: Streaming responses use more tokens than non-streaming because the model generates more verbose output. Account for this in cost projections.
