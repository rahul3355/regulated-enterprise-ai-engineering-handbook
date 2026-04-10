# Case Study: Token Cost Explosion — 400% Budget Overrun

## Executive Summary

Monthly LLM token costs exceeded budget by 400%, jumping from a budgeted £45,000/month to £227,000 in a single month. The root cause was a combination of uncontrolled prompt growth, a new feature that dramatically increased context window usage, missing cost monitoring, and an undocumented auto-scaling configuration for the LLM API. The incident triggered a SEV-3 cost anomaly investigation and resulted in comprehensive cost governance controls.

**Severity:** SEV-3 (Cost Anomaly)
**Duration:** 31 days (full billing cycle)
**Budget variance:** +400% (£45K budget vs. £227K actual)
**Root cause:** Uncontrolled prompt growth + new feature + missing cost monitoring
**Financial impact:** £182,000 overspend (one-time)

---

## Background and Context

### The System

Our GenAI platform served multiple applications:
1. **AssistBot**: Customer support RAG assistant (~40% of tokens)
2. **WealthAdvisor AI**: Financial advisory chatbot (~30% of tokens)
3. **ComplianceBot**: Internal compliance Q&A (~10% of tokens)
4. **SmartSearch**: Semantic search with LLM summarization (~20% of tokens)

### Budget Model

- Monthly budget: £45,000
- Allocation per application:
  - AssistBot: £18,000
  - WealthAdvisor AI: £13,500
  - ComplianceBot: £4,500
  - SmartSearch: £9,000
- Pricing: £0.03 per 1K input tokens, £0.09 per 1K output tokens (GPT-4)

### Cost Tracking (Insufficient)

What was tracked:
- Monthly invoice total from OpenAI/Azure
- Per-application API key usage (manual monthly review)

What was NOT tracked:
- Daily token consumption trends
- Per-endpoint token usage
- Prompt size distribution
- Context window utilization
- Cost per query/request
- Alerting on cost anomalies

---

## Timeline of Events

```mermaid
timeline
    title Token Cost Explosion Timeline
    section Month Prior
        Normal operations : Average monthly cost: £42,000<br/>Within budget of £45,000
        : Average query: ~3,200<br/>input tokens, ~400 output tokens
    section Week 1
        Day 1 : New feature launched:<br/>DocumentSummarizer<br/>(full document context in prompt)
        : DocumentSummarizer uses<br/>GPT-4 with 32K context<br/>for full document analysis
        : Average query jumps to<br/>~12,000 input tokens
        Day 3 : Cost per query increases<br/>3.7x, but no alert<br/>triggered (no monitoring)
        Day 5 : Prompt engineering team<br/>adds more context to<br/>AssistBot RAG (top_k 10 -> 20)
        Day 7 : Week 1 cost: £14,200<br/>(normalized: £56,800/month)<br/>No one checks
    section Week 2
        Day 8 : DocumentSummarizer<br/>goes viral internally,<br/>heavy usage by 200+ staff
        : Average daily token<br/>usage doubles again
        Day 10 : AssistBot system prompt<br/>expanded with new<br/>compliance guidelines (+800 tokens)
        Day 12 : Week 2 cumulative: £38,500<br/>Projected monthly: £110,000<br/>Still no alerting
    section Week 3
        Day 15 : WealthAdvisor AI adds<br/>portfolio analysis feature<br/>(sends full portfolio history per query)
        : Portfolio queries use<br/>~25,000 input tokens each
        Day 17 : Week 3 cumulative: £72,000<br/>Already exceeds monthly budget
        Day 19 : Finance team notices<br/>OpenAI invoice projection<br/>in weekly spend review
        Day 19 14:00 : Finance alerts engineering:<br/>"We're on track for £200K+<br/>this month"
    section Detection and Response
        Day 19 15:00 : SEV-3 declared<br/>(cost anomaly)
        Day 19 16:00 : Investigation begins:<br/>token usage by service<br/>analyzed
        Day 19 17:00 : Root causes identified:<br/>1. DocumentSummarizer<br/>2. AssistBot top_k increase<br/>3. WealthAdvisor portfolios<br/>4. System prompt growth
        Day 20 : Emergency cost controls:<br/>rate limits, context caps,<br/>model downgrade for non-critical
        Day 21 : DocumentSummarizer<br/>downgraded from GPT-4<br/>to GPT-3.5-Turbo
        Day 22 : AssistBot top_k<br/>reduced from 20 to 8<br/>with score threshold 0.80
        Day 25 : Cost monitoring<br/>dashboard deployed<br/>with daily alerting
    section Month End
        Final cost : £227,000<br/>(400% over budget)
        : £182,000 overspend<br/>absorbed from contingency<br/>budget
```

### Cost Breakdown by Driver

| Driver | Token Impact | Cost Impact | Percentage |
|--------|-------------|-------------|------------|
| DocumentSummarizer (GPT-4, 32K context) | +4.2B tokens | +£126,000 | 55% |
| AssistBot top_k increase (10 -> 20) | +1.1B tokens | +£33,000 | 15% |
| WealthAdvisor portfolio analysis | +1.8B tokens | +£54,000 | 24% |
| System prompt growth (+800 tokens) | +0.3B tokens | +£9,000 | 4% |
| Normal baseline | ~1.4B tokens | +£42,000 | 18% |
| **Total** | **~8.8B tokens** | **£227,000** | **100%** |

---

## Root Cause Analysis

### Technical Root Causes

1. **Unbounded Context Growth**
   - DocumentSummarizer sent entire documents (up to 32K tokens) to GPT-4
   - No summarization or chunking before LLM processing
   - No maximum context length enforcement

2. **Retrieval Parameter Creep**
   - AssistBot top_k increased from 10 to 20 without cost analysis
   - Each additional document added ~512 tokens of context
   - No threshold tuning based on cost-benefit analysis

3. **System Prompt Bloat**
   - System prompt grew from 1,200 to 2,000 tokens over 6 months
   - Every regulatory update added more text
   - System prompt is sent with EVERY query (multiplied by millions of queries)

4. **No Cost Monitoring**
   - No daily token consumption tracking
   - No alerting on cost anomalies
   - No per-endpoint cost attribution
   - Monthly invoice was the only cost signal

5. **Model Selection Without Cost Analysis**
   - DocumentSummarizer used GPT-4 (£0.03/1K input) instead of GPT-3.5-Turbo (£0.003/1K input)
   - 10x cost difference for a task that did not require GPT-4 level reasoning
   - No cost-benefit analysis of model selection

### Organizational Root Causes

1. **No Cost Ownership**
   - Engineering owned functionality, finance owned budgets
   - Nobody owned the intersection: cost-efficient engineering
   - No FinOps function or cost-aware engineering practices

2. **Feature Launch Without Cost Review**
   - DocumentSummarizer launched without a cost impact assessment
   - No "cost budget" was allocated to the feature
   - Product manager was not aware of per-query cost implications

3. **Missing FinOps Culture**
   - Engineers had no visibility into the cost impact of their decisions
   - Token usage was not part of the engineering metrics dashboard
   - No cost reviews during sprint retrospectives

4. **Budget Process Failure**
   - Budget was set annually with no monthly tracking
   - Finance reviewed costs monthly but engineering did not
   - The gap between finance and engineering meant cost anomalies went unnoticed for weeks

---

## What Went Wrong Technically

### The Cost Multiplier Effect

```
System prompt (2,000 tokens) × 2M queries/month = 4B tokens = £120,000
Retrieved context (20 × 512 = 10,240 tokens) × 2M queries = 20.5B tokens = £615,000
DocumentSummarizer (32,000 tokens) × 5K documents = 160M tokens = £4,800,000 (at GPT-4 rates)
```

### The Missing Cost Monitor

```python
# WHAT SHOULD HAVE EXISTED:

class TokenCostMonitor:
    def __init__(self, daily_budget: float, alert_threshold: float = 0.8):
        self.daily_budget = daily_budget
        self.alert_threshold = alert_threshold
        self.daily_usage = defaultdict(int)  # service -> tokens

    def record_usage(self, service: str, input_tokens: int, output_tokens: int):
        cost = self._calculate_cost(input_tokens, output_tokens)
        self.daily_usage[service] += cost

    def check_budget(self):
        total = sum(self.daily_usage.values())
        if total > self.daily_budget * self.alert_threshold:
            self._alert(f"Daily token cost at {total/self.daily_budget:.0%} of budget")

        # Per-service breakdown
        for service, cost in self.daily_usage.items():
            service_budget = self._get_service_budget(service)
            if cost > service_budget * self.alert_threshold:
                self._alert(f"{service} at {cost/service_budget:.0%} of its budget")

    def _calculate_cost(self, input_tokens: int, output_tokens: int) -> float:
        # GPT-4 pricing
        return (input_tokens / 1000) * 0.03 + (output_tokens / 1000) * 0.09

# This monitor did not exist.
```

### The Unbounded Context

```python
# DocumentSummarizer (BEFORE - no cost control)
async def summarize_document(document_id: str) -> str:
    doc = await load_document(document_id)
    # Sends ENTIRE document to GPT-4 - could be 32K tokens!
    response = await llm.chat(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "Summarize this document."},
            {"role": "user", "content": doc.full_text}  # UNBOUNDED
        ]
    )
    return response.choices[0].message.content

# DocumentSummarizer (AFTER - with cost controls)
async def summarize_document(document_id: str) -> str:
    doc = await load_document(document_id)

    # Cap context at 8K tokens
    context = truncate_to_tokens(doc.full_text, max_tokens=8000)

    # Use GPT-3.5-Turbo for summarization (10x cheaper)
    response = await llm.chat(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": "Summarize this document in 3-5 bullet points."},
            {"role": "user", "content": context}
        ],
        max_tokens=500,  # Cap output too
    )
    return response.choices[0].message.content
```

---

## What Went Wrong Organizationally

1. **No FinOps Function**: The organization had no FinOps practice. Cost optimization was reactive, not proactive.

2. **Engineering-Finance Disconnect**: Engineers built features without cost visibility. Finance tracked costs without understanding the engineering drivers.

3. **Feature Velocity Over Cost Awareness**: Features were shipped based on user value without considering cost efficiency.

4. **No Cost Review Process**: New features underwent security and performance review but not cost review.

---

## Immediate Response and Mitigation

### First 48 Hours

1. **Cost Controls Implemented**
   - Rate limits on DocumentSummarizer (100 documents/hour per user)
   - Context cap at 8K tokens for summarization
   - AssistBot top_k reduced from 20 to 8
   - WealthAdvisor portfolio history limited to last 12 months

2. **Model Downgrades**
   - DocumentSummarizer: GPT-4 -> GPT-3.5-Turbo (10x cost reduction)
   - SmartSearch summaries: GPT-4 -> GPT-3.5-Turbo
   - AssistBot and WealthAdvisor remained on GPT-4 (quality-critical)

3. **System Prompt Audit**
   - System prompt reviewed and reduced from 2,000 to 1,400 tokens
   - Redundant compliance language removed (now covered by RAG context)
   - Frozen change process for system prompt (requires approval to modify)

### Week 1-2

1. **Cost Monitoring Dashboard**: Real-time token cost tracking with daily alerting
2. **Per-Service Budgets**: Budget allocation per service with individual alerting
3. **Cost Per Query Metric**: Engineering dashboard now shows cost per query
4. **FinOps Working Group**: Cross-functional team (engineering, finance, product) meets weekly

### Cost Projection After Controls

| Service | Before (monthly) | After (monthly) | Savings |
|---------|-----------------|-----------------|---------|
| DocumentSummarizer | £126,000 | £12,600 | -90% |
| AssistBot | £51,000 | £28,000 | -45% |
| WealthAdvisor AI | £67,500 | £22,000 | -67% |
| SmartSearch | £9,000 | £4,500 | -50% |
| ComplianceBot | £4,500 | £3,500 | -22% |
| **Total** | **£227,000** | **£70,600** | **-69%** |

---

## Long-Term Fixes and Systemic Changes

### Technical Fixes

1. **Token Cost Monitoring and Alerting**
   ```python
   # Production cost monitoring middleware
   class CostTrackingMiddleware:
       async def on_request_complete(self, request, response):
           input_tokens = response.usage.prompt_tokens
           output_tokens = response.usage.completion_tokens
           cost = self.pricing.calculate_cost(input_tokens, output_tokens)

           self.metrics.record(
               service=request.service_name,
               endpoint=request.endpoint,
               input_tokens=input_tokens,
               output_tokens=output_tokens,
               cost=cost,
           )

           # Check against daily budget
           if self.monitor.is_over_budget(request.service_name):
               self._alert(f"{request.service_name} exceeded daily token budget")
   ```

2. **Context Budget Enforcement**
   - Maximum context per query enforced at the API gateway level
   - Prompt size limits per service
   - Automatic rejection of queries exceeding context budget

3. **Model Selection Framework**
   - Decision matrix for model selection based on task complexity vs. cost
   - GPT-3.5-Turbo for summarization, classification, simple Q&A
   - GPT-4 for complex reasoning, financial advice, compliance
   - Regular cost-benefit reviews

4. **System Prompt Governance**
   - System prompt changes require approval from engineering lead and product owner
   - Monthly review of system prompt size and content
   - Token budget allocated for system prompt (max 1,500 tokens)

### Process Changes

1. **FinOps Practice**: Dedicated FinOps function with engineering and finance representation
2. **Cost Review for New Features**: All new GenAI features require cost impact assessment
3. **Monthly Cost Reviews**: Engineering and finance review token costs monthly
4. **Per-Service Budgets**: Each service has a token budget with individual alerting
5. **Quarterly Model Evaluation**: Regular evaluation of whether services can use cheaper models

### Cultural Changes

1. **Cost as an Engineering Metric**: Token cost is now a first-class engineering metric alongside latency and error rate
2. **Cost-Aware Design**: Engineers consider cost implications during design, not after deployment
3. **Transparency**: Cost dashboards visible to all engineers, not just management

---

## Lessons Learned

1. **Context Is Expensive**: Every token in the prompt costs money. Context must be budgeted like any other resource.

2. **Model Selection Matters**: Using GPT-4 when GPT-3.5-Turbo would suffice is a 10x cost difference.

3. **Small Changes Compound**: Increasing top_k from 10 to 20 doubles context costs. System prompt growth of 800 tokens costs £9,000/month at scale.

4. **Monitoring Must Include Cost**: Technical monitoring (latency, errors) without cost monitoring is incomplete.

5. **FinOps Is Essential**: A dedicated FinOps function bridges the gap between engineering and finance.

6. **Budget Reviews Should Be Frequent**: Monthly budget reviews are too slow for rapidly scaling GenAI usage. Weekly or daily monitoring is needed.

---

## Interview Questions Derived From This Case Study

1. **FinOps**: "How would you design a cost monitoring system for a GenAI platform? What metrics and alerts would you set up?"

2. **System Design**: "You need to build a document summarization feature. How do you balance quality, cost, and performance?"

3. **Architecture**: "How do you enforce cost budgets at the API level for a multi-service GenAI platform?"

4. **Decision Making**: "When would you choose GPT-4 over GPT-3.5-Turbo? What framework would you use to decide?"

5. **Operations**: "Token costs have increased 300% month-over-month. How do you investigate and remediate?"

6. **Process**: "What governance would you put in place to prevent uncontrolled token cost growth?"

---

## Cross-References

- See `../infrastructure/infrastructure-cost-management.md` for cost management practices
- See `../genai-platforms/token-economics.md` for token cost optimization
- See `../incident-management/incident-classification.md` for SEV level definitions
- See `../engineering-philosophy/cost-aware-engineering.md` for cost-aware engineering principles
- See `../architecture/prompt-optimization.md` for prompt size optimization techniques
