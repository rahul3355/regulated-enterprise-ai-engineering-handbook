# Infrastructure Cost Management

## Overview

This document covers cost monitoring, optimization strategies, budget management, and FinOps practices for the banking GenAI platform. Cost management is a continuous engineering discipline, not a monthly finance exercise.

---

## Cost Breakdown

### Typical GenAI Platform Cost Distribution

```
┌──────────────────────────────────────────────────┐
│           Monthly Cost Breakdown                  │
│                                                   │
│  LLM API Calls:        £18,000  ████████  42%    │
│  GPU Compute:          £15,000  ███████   35%    │
│  Application Compute:   £5,000  ██       12%    │
│  Database & Storage:    £3,000  █         7%    │
│  Networking (CDN, LB):  £1,500  █         4%    │
│  Monitoring & Tools:    £1,000  █         2%    │
│  Other:                   £500  █         1%    │
│  ─────────────────────────────────               │
│  Total:                 £44,000  100%            │
└──────────────────────────────────────────────────┘
```

---

## Cost Monitoring

### Real-Time Cost Dashboards

```
Cost Dashboard
==============

Today's Spend: £1,420 (projected monthly: £42,600)
This Month:    £18,500 (42% of budget, 52% of elapsed time)

By Service:
  LLM API (Azure OpenAI):  £8,200  ████████  44%
  GPU Compute (AWS):       £6,100  ██████    33%
  AKS (Azure):             £2,100  ██        11%
  Database (PostgreSQL):   £1,100  █         6%
  Vector DB (Pinecone):     £500  █         3%
  CDN (CloudFront):         £300  █         2%
  Other:                    £200  █         1%

By Environment:
  Production:    £16,500  89%
  Staging:       £1,500    8%
  Development:     £500    3%

Cost Per Query:
  AssistBot:      £0.012/query
  WealthAdvisor:  £0.045/query
  ComplianceBot:  £0.008/query
  SmartSearch:    £0.005/query

Alerts:
  ✓ All services within budget
  ⚠ AssistBot daily cost trending 15% above forecast
```

### Cost Alerting

| Alert | Threshold | Severity | Action |
|-------|----------|----------|--------|
| **Daily cost > 150% of forecast** | Any service | SEV-3 | Investigate cost driver |
| **Daily cost > 200% of forecast** | Any service | SEV-2 | Apply cost controls |
| **Monthly projected > budget** | Total platform | SEV-2 | Executive review |
| **GPU utilization < 30%** | Any GPU instance | SEV-4 | Right-size or consolidate |
| **Unused resources detected** | Any resource | SEV-4 | Review and terminate |

---

## Cost Optimization Strategies

### 1. LLM API Cost Optimization

| Strategy | Savings | Implementation Effort |
|----------|---------|---------------------|
| **Prompt optimization** (reduce token count) | 20-40% | Medium |
| **Context caching** (reuse retrieved context) | 10-20% | Medium |
| **Model routing** (use cheaper models for simple tasks) | 30-50% | High |
| **Response caching** (cache common queries) | 15-30% | Low |
| **Committed spend discount** | 10-25% | Low |
| **Temperature/top_p tuning** (shorter responses) | 5-10% | Low |

#### Prompt Optimization Example

```python
# BEFORE: Verbose prompt (~800 tokens)
system_prompt = """
You are a helpful banking assistant named BankAssist. You work for a major
UK bank and you are designed to help customers with their banking needs.

Your responsibilities include:
1. Answering questions about account balances and transactions
2. Providing information about our products (mortgages, loans, credit cards)
3. Helping customers with common tasks (transferring money, paying bills)
4. Escalating complex issues to human agents when necessary

Please be polite, professional, and helpful at all times. If you are unsure
about something, say so rather than making up information. Always comply
with banking regulations and do not provide financial advice that could
put the customer at risk.

When answering questions about accounts, always verify the customer's
identity first. If the customer has not been authenticated, ask them to
log in before providing account-specific information.
"""

# AFTER: Optimized prompt (~300 tokens)
system_prompt = """
You are BankAssist, a UK banking customer service AI.

Rules:
- Verify identity before account-specific info
- No financial advice — information only
- Escalate complex issues to human agents
- Be concise and accurate
- If uncertain, say so
"""
# Savings: 500 tokens per query × 1M queries/month = 500M input tokens
# Cost savings: 500M × £0.01/1K = £5,000/month
```

### 2. GPU Compute Cost Optimization

| Strategy | Savings | Implementation Effort |
|----------|---------|---------------------|
| **Reserved instances** (1-year commitment) | 40-60% | Low |
| **Spot instances** (for training/batch) | 60-90% | Medium |
| **Right-sizing** (match GPU to model size) | 20-40% | Medium |
| **GPU sharing** (MIG, multi-tenant) | 30-50% | High |
| **Auto-scaling** (scale to zero for dev) | 40-60% | Medium |
| **Inference optimization** (vLLM, TensorRT) | 30-50% throughput increase | High |

#### GPU Utilization Monitoring

```python
# GPU utilization tracking
def monitor_gpu_utilization():
    """Track GPU utilization and alert on underutilization."""
    for node in gpu_nodes:
        utilization = get_gpu_utilization(node)
        memory_usage = get_gpu_memory_usage(node)

        if utilization < 30:
            alert(f"GPU {node} utilization low: {utilization}%")

        if memory_usage < 40:
            alert(f"GPU {node} memory usage low: {memory_usage}%")

        # Recommendation engine
        if utilization < 30 and memory_usage < 40:
            recommend(f"Consider downsizing GPU on {node}")
        elif utilization > 80:
            recommend(f"Consider adding GPU capacity on {node}")
```

### 3. Storage Cost Optimization

| Strategy | Savings | Implementation Effort |
|----------|---------|---------------------|
| **Lifecycle policies** (hot → cool → cold) | 40-70% on storage | Low |
| **Compression** | 50-80% on data size | Low |
| **Deduplication** | 30-60% | Medium |
| **Delete unused data** | 10-30% | Medium |
| **Intelligent tiering** | 20-40% | Low |

### 4. Networking Cost Optimization

| Strategy | Savings | Implementation Effort |
|----------|---------|---------------------|
| **VPC endpoints** (avoid NAT egress) | 20-40% on data transfer | Low |
| **CDN caching** | 30-50% on origin bandwidth | Medium |
| **Cross-region transfer reduction** | 50-80% on cross-region | High |
| **Private connectivity** (Direct Connect) | 30-50% on data transfer | Medium |

---

## Budget Management

### Budget Structure

```yaml
# Monthly budget allocation
budget:
  total: £45,000
  allocation:
    llm_api:
      budget: £18,000
      alert_threshold: 80%  # £14,400
      hard_limit: 120%       # £21,600
    gpu_compute:
      budget: £15,000
      alert_threshold: 80%
      hard_limit: 120%
    application_compute:
      budget: £5,000
      alert_threshold: 80%
      hard_limit: 120%
    database_storage:
      budget: £3,000
      alert_threshold: 80%
      hard_limit: 120%
    networking:
      budget: £1,500
      alert_threshold: 80%
      hard_limit: 120%
    contingency:
      budget: £2,500
      usage_approval: engineering_lead
```

### Budget Alerts and Actions

| Budget Level | Alert | Action |
|-------------|-------|--------|
| **80% of monthly budget** | Warning alert to engineering team | Review cost trends, identify drivers |
| **100% of monthly budget** | Alert to engineering lead | Evaluate necessity of additional spend |
| **120% of monthly budget** | SEV-3 incident declared | Apply cost controls, investigate root cause |
| **150% of monthly budget** | SEV-2 incident | Mandatory cost reduction, executive review |

### Budget Forecasting

```python
def forecast_monthly_cost(daily_costs: List[float], days_elapsed: int) -> Dict:
    """Forecast monthly cost based on daily spend trends."""
    current_spend = sum(daily_costs)
    daily_average = current_spend / days_elapsed
    projected_monthly = daily_average * 30

    # Trend analysis (is cost increasing or decreasing?)
    if len(daily_costs) >= 7:
        recent_avg = np.mean(daily_costs[-7:])
        previous_avg = np.mean(daily_costs[:-7])
        trend = (recent_avg - previous_avg) / previous_avg
    else:
        trend = 0

    return {
        "current_spend": current_spend,
        "daily_average": daily_average,
        "projected_monthly": projected_monthly,
        "trend": f"{'increasing' if trend > 0.05 else 'decreasing' if trend < -0.05 else 'stable'}",
        "trend_percentage": trend * 100,
    }
```

---

## FinOps Practices

### FinOps Team Structure

| Role | Responsibility | Time Commitment |
|------|---------------|----------------|
| **FinOps Lead** | Cost strategy, optimization roadmap | 20% |
| **Engineering Representatives** | Service-level cost ownership | 10% |
| **Finance Partner** | Budget management, forecasting | 20% |
| **Cloud Provider TAM** | Provider-side optimization advice | As needed |

### FinOps Cadence

| Meeting | Frequency | Attendees | Agenda |
|---------|-----------|-----------|--------|
| **Cost Review** | Weekly | FinOps Lead, Engineering Reps | Review spend, identify anomalies, action items |
| **Budget Review** | Monthly | Engineering Lead, Finance | Budget vs. actual, forecast, adjustments |
| **Optimization Review** | Quarterly | Engineering Leadership | Long-term cost strategy, major initiatives |

### Cost Allocation Tags

All resources must be tagged for cost allocation:

| Tag | Example | Purpose |
|-----|---------|---------|
| `environment` | `prod`, `staging`, `dev` | Cost by environment |
| `service` | `assistbot`, `wealth-advisor`, `compliance-bot` | Cost by service |
| `team` | `platform`, `inference`, `data` | Cost by team |
| `cost-center` | `genai-platform` | Finance allocation |
| `managed-by` | `terraform`, `manual` | Resource governance |

---

## Cost Optimization Roadmap

### Immediate Wins (Week 1-2)

- [ ] Review and terminate unused resources
- [ ] Enable storage lifecycle policies
- [ ] Apply reserved instances to baseline compute
- [ ] Optimize prompt token usage

### Short-Term (Month 1)

- [ ] Implement cost monitoring and alerting
- [ ] Deploy response caching for common queries
- [ ] Right-size GPU instances based on utilization
- [ ] Negotiate committed spend discount with LLM provider

### Medium-Term (Quarter 1)

- [ ] Build model routing layer (cheapest model for each task)
- [ ] Implement GPU auto-scaling based on queue depth
- [ ] Optimize embedding pipeline costs
- [ ] Establish FinOps review cadence

### Long-Term (Quarter 2-4)

- [ ] Self-host models for high-volume, low-complexity tasks
- [ ] Implement speculative decoding for faster inference
- [ ] Multi-cloud cost optimization (route to cheapest provider)
- [ ] Custom fine-tuned models replacing API calls where cost-effective

---

## Cost Per Unit Metrics

Track cost efficiency, not just total cost:

| Metric | Current | Target | Notes |
|--------|---------|--------|-------|
| **Cost per query (AssistBot)** | £0.012 | £0.008 | Prompt optimization, caching |
| **Cost per query (WealthAdvisor)** | £0.045 | £0.030 | Model routing, context reduction |
| **Cost per embedding** | £0.001 | £0.0005 | Batch embedding, self-hosted |
| **Cost per GPU inference hour** | £3.50 | £2.00 | Reserved instances, optimization |
| **Cost per customer per month** | £0.022 | £0.015 | Scale efficiency |

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [cloud-providers.md](cloud-providers.md) -- Cloud provider pricing
- [compute.md](compute.md) -- Compute cost optimization
- [storage.md](storage.md) -- Storage cost optimization
- [../genai-platforms/token-economics.md](../genai-platforms/token-economics.md) -- Token economics
- [../engineering-philosophy/cost-aware-engineering.md](../engineering-philosophy/cost-aware-engineering.md) -- Cost-aware engineering
