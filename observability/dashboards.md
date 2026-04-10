# Dashboard Design for Banking Observability

## Philosophy

Dashboards are communication tools, not data dumps. A good dashboard answers a specific question for a specific audience in under 10 seconds. A bad dashboard shows everything and helps no one.

In banking, dashboards serve three distinct audiences:

1. **Engineers on-call**: Need to diagnose and resolve incidents quickly
2. **Engineering managers**: Need to track reliability trends and SLO compliance
3. **Executives and auditors**: Need business-level health and compliance status

Each audience requires different dashboards with different granularity.

## Dashboard Hierarchy

```
┌─────────────────────────────────────────────────────┐
│                 Dashboard Pyramid                   │
│                                                     │
│              ┌──────────────┐                       │
│              │  Executive   │  Business health      │
│              │  Summary     │  SLO compliance       │
│              └──────────────┘                       │
│           ┌──────────────────────┐                  │
│           │  Service Overview    │  Per-service SLOs │
│           │  Dashboards          │  Key metrics      │
│           └──────────────────────┘                  │
│      ┌────────────────────────────────────┐         │
│      │  Debug/Troubleshooting             │  Deep   │
│      │  Dashboards                        │  detail │
│      └────────────────────────────────────┘         │
│   ┌──────────────────────────────────────────┐      │
│   │  Component-Specific                      │  GPU, │
│   │  Dashboards                              │  DB,  │
│   └──────────────────────────────────────────┘  Cache│
└─────────────────────────────────────────────────────┘
```

## Panel Types and When to Use Them

### Time Series

Use for metrics that change over time: latency, request rate, error rate, resource usage.

```
┌─────────────────────────────────────────────────┐
│  HTTP Request Rate (req/s)                       │
│                                                 │
│  500 ┤    ╱╲        ╱╲                          │
│      │   ╱  ╲      ╱  ╲       ╱╲               │
│  250 ┤  ╱    ╲    ╱    ╲     ╱  ╲    ╱╲        │
│      │ ╱      ╲  ╱      ╲   ╱    ╲  ╱  ╲       │
│    0 ┼──────────┴───────────┴──────┴─┴────┴───  │
│      00:00    04:00    08:00   12:00   16:00    │
└─────────────────────────────────────────────────┘
```

**Best practices**:
- Include SLO targets as horizontal lines
- Use separate panels for different metric types (do not mix latency and error rate)
- Set y-axis to start at zero unless showing small variations around a target
- Use consistent colors: green for success, red for errors, blue for latency

### Stat (Single Value)

Use for the most important current value: current error rate, current SLO remaining budget, active user count.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Current     │  │  SLO Budget  │  │  Active      │
│  Error Rate  │  │  Remaining   │  │  Sessions    │
│              │  │              │  │              │
│    0.12%     │  │    94.2%     │  │    2,847     │
│  (target:    │  │  (monthly    │  │  (peak:      │
│   < 0.5%)    │  │   budget)    │  │   3,200)     │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Best practices**:
- Include the target/threshold for context
- Show trend indicator (up/down arrow)
- Color code: green if healthy, yellow if warning, red if critical
- Include a sparkline showing the recent trend

### Heatmap

Use for understanding metric distributions over time: latency percentiles, response sizes.

```
┌─────────────────────────────────────────────────┐
│  Response Time Distribution                       │
│                                                 │
│  10s ┤                                          │
│      │                                          │
│   1s ┤  ████                                    │
│      │  ████████████████████████████            │
│ 100ms┤  ██████████████████████████████████████  │
│      │  ██████████████████████████████████████  │
│  10ms┤  ██████████████████████████████████████  │
│      └────────────────────────────────────────── │
│       00:00    04:00    08:00   12:00   16:00   │
└─────────────────────────────────────────────────┘
```

**Best practices**:
- Use logarithmic scale for latency
- Show SLO thresholds as overlay lines
- Useful for identifying tail latency patterns

### Table

Use for detailed lists: top slow endpoints, services by error rate, cost by model.

```
┌──────────────────────────────────────────────────────────────┐
│  Top 10 Endpoints by p99 Latency                             │
├──────────────────────┬──────────┬──────────┬─────────────────┤
│  Endpoint            │  p50     │  p99     │  Error Rate     │
├──────────────────────┼──────────┼──────────┼─────────────────┤
│  /mortgage-advice    │  2.1s    │  8.4s    │  0.3%           │
│  /document-analysis  │  1.8s    │  7.2s    │  0.5%           │
│  /chat               │  0.8s    │  3.1s    │  0.1%           │
│  /account-summary    │  0.3s    │  1.2s    │  0.02%          │
└──────────────────────┴──────────┴──────────┴─────────────────┘
```

### Gauge

Use for percentage-based metrics with clear targets: error budget remaining, disk usage, connection pool utilization.

```
┌──────────────────────────┐
│  Error Budget Remaining   │
│                          │
│      ┌──────────┐        │
│    ╱╱            ╲╲      │
│   │    94.2%     │       │
│    ╲╲            ╱╱      │
│      └──────────┘        │
│   0% ──── 50% ──── 100%  │
│   [████████████████░░░]  │
└──────────────────────────┘
```

## Essential Dashboards for Banking GenAI

### 1. Executive Summary Dashboard

**Audience**: VPs, Directors, Compliance Officers
**Refresh interval**: 5 minutes

```
┌──────────────────────────────────────────────────────────────┐
│  GENAI PLATFORM - EXECUTIVE HEALTH                           │
├─────────────┬─────────────┬──────────────┬───────────────────┤
│ Total       │ Platform    │ SLO          │ Customer          │
│ Queries     │ Uptime      │ Compliance   │ Satisfaction      │
│ Today       │             │              │                   │
│             │             │              │                   │
│  12,847     │  99.97%     │  4/4 met     │  4.2/5.0         │
│  (+8% vs   │  (target:   │  (target:    │  (target: >3.5)  │
│   yesterday)│   99.9%)    │   all)       │                   │
└─────────────┴─────────────┴──────────────┴───────────────────┘

┌──────────────────────────────────┬──────────────────────────┐
│  Query Volume (last 7 days)      │  Cost This Month         │
│  [Time series chart]             │  $12,450 of $15,000 budget│
│                                  │  [Gauge: 83%]            │
└──────────────────────────────────┴──────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Compliance Status                                           │
│  All AI decisions logged: YES (100%)                         │
│  PII in prompts detected: 0 (last 30 days)                   │
│  Model version audit: PASS                                   │
│  Data retention compliance: PASS (7-year archive verified)   │
└──────────────────────────────────────────────────────────────┘
```

### 2. Service SLO Dashboard

**Audience**: Engineering managers, SREs
**Refresh interval**: 1 minute

```
┌──────────────────────────────────────────────────────────────┐
│  LOAN ADVISOR API - SLO STATUS                                │
├────────────────────┬──────────┬──────────┬───────────────────┤
│  SLO               │  Target  │  Actual  │  Budget Remaining │
├────────────────────┼──────────┼──────────┼───────────────────┤
│  Availability      │  99.9%   │  99.95%  │  97.2%            │
│  Latency (p95)     │  < 3s    │  2.1s    │  94.8%            │
│  Correctness       │  99.5%   │  99.7%   │  98.1%            │
│  Freshness         │  < 1h    │  12min   │  99.9%            │
└────────────────────┴──────────┴──────────┴───────────────────┘

┌──────────────────────────────────┬──────────────────────────┐
│  Error Rate (last 24h)           │  Latency Distribution    │
│  [Time series with SLO line]     │  [Heatmap]               │
└──────────────────────────────────┴──────────────────────────┘
```

### 3. LLM Platform Dashboard

**Audience**: ML engineers, Platform engineers
**Refresh interval**: 30 seconds

```
┌──────────────────────────────────────────────────────────────┐
│  LLM PLATFORM - OPERATIONAL VIEW                              │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  Calls/min   │  Avg Latency │  Error Rate  │  Token Rate     │
│              │              │              │  (tokens/min)   │
│  1,247       │  2.3s        │  0.18%       │  245,678        │
│  (vs 1,100   │  (vs 1.9s    │  (vs 0.12%   │  (vs 210k       │
│   yesterday) │   yesterday) │   yesterday) │   yesterday)    │
└──────────────┴──────────────┴──────────────┴─────────────────┘

┌──────────────────────────────────┬──────────────────────────┐
│  Model Usage Breakdown           │  Token Cost by Model     │
│  gpt-4: 45%                      │  gpt-4: $8,234           │
│  gpt-3.5-turbo: 30%              │  gpt-3.5: $2,156         │
│  claude-3: 15%                   │  claude-3: $1,890        │
│  custom-model: 10%               │  custom: $170            │
└──────────────────────────────────┴──────────────────────────┘

┌──────────────────────────────────┬──────────────────────────┐
│  Cache Hit Rate                  │  Queue Depth             │
│  [Stat with sparkline]           │  [Time series]           │
│  23% (target: >20%)             │                          │
└──────────────────────────────────┴──────────────────────────┘
```

## Dashboard Design Principles

### 1. One Dashboard, One Question

Each dashboard should answer one primary question. The "Loan Advisor API SLO Dashboard" answers "Are we meeting our SLOs?" The "LLM Platform Operational Dashboard" answers "Is the LLM platform healthy?"

Do not mix executive summary data with debug-level details.

### 2. Top-to-Bottom Information Hierarchy

Place the most important information at the top. Engineers read dashboards top-to-bottom during incidents.

```
Row 1: Current status (Stat panels) - "Is there a problem?"
Row 2: Trends (Time series) - "How did we get here?"
Row 3: Breakdown (Tables, pie charts) - "What is affected?"
Row 4: Details (Heatmaps, logs) - "Why is it happening?"
```

### 3. Use Templates and Variables

Create one dashboard template that works for all services using Grafana variables:

```
Variables:
  $service - dropdown: loan-advisor-api, rag-pipeline, llm-gateway
  $environment - dropdown: production, staging
  $region - dropdown: us-east-1, eu-west-1

Panel query:
  rate(http_requests_total{service="$service", environment="$environment"}[5m])
```

### 4. Set Appropriate Refresh Intervals

```
Executive dashboards: 5 minutes
SLO dashboards: 1 minute
Debug dashboards: 15-30 seconds
Real-time incident dashboards: 5 seconds
```

Faster refresh rates increase load on your observability backend. Do not set all dashboards to 5-second refresh.

### 5. Include Context, Not Just Data

Every panel should answer "Is this good or bad?"

Bad: `Error Rate: 0.18%`
Good: `Error Rate: 0.18% (target: < 0.5%) [OK]`

Include SLO targets, thresholds, and previous-period comparisons.

## Anti-Patterns

1. **The Christmas Tree Dashboard**: Everything flashing different colors. If everything is alert-worthy, nothing is actionable.

2. **The Infinite Scroll Dashboard**: 50+ panels in a single dashboard. Split by concern.

3. **The Untitled Panel**: A graph with no title, no legend, and no units. An engineer should understand any panel in 5 seconds.

4. **The Raw Metrics Dashboard**: Showing raw counter values instead of rates. A counter that always goes up tells you nothing.

5. **No One Owns This Dashboard**: Every dashboard should have an identified owner who maintains it and removes stale panels.

6. **Dashboards That No One Looks At**: Audit dashboard usage quarterly. Remove or consolidate dashboards with no views.
