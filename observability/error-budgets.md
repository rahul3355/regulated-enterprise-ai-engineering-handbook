# Error Budgets in Banking GenAI

## What Is an Error Budget?

An error budget quantifies how much unreliability is acceptable. It is the flip side of an SLO:

```
SLO: 99.9% availability over 30 days
Error Budget: 0.1% unavailability allowed over 30 days

For 15 million requests in 30 days:
  Allowed failures = 15,000,000 x 0.001 = 15,000 failures
```

The error budget is not a target to hit. It is a limit to stay within. Spending the entire budget means the service is too unreliable for its users.

## Why Error Budgets Matter

Without error budgets, reliability discussions become subjective:

**Without error budgets**:
- Product: "Ship the feature, we need it for the quarter."
- Engineering: "The system is not stable enough for new features."
- Result: Subjective argument, usually won by whoever has more authority.

**With error budgets**:
- "The loan advisor API has 82% of its monthly error budget remaining. We can ship the feature."
- "The RAG pipeline has 0% error budget remaining. We must invest in reliability before shipping new features."
- Result: Objective decision based on data.

## Error Budget Policy

### Policy Levels

```
┌──────────────────────────────────────────────────────────────┐
│  ERROR BUDGET POLICY                                         │
├─────────────────────┬────────────────────────────────────────┤
│  Budget Remaining   │  Action                                │
├─────────────────────┼────────────────────────────────────────┤
│  > 50%              │  Normal operations. Ship features.     │
│  25-50%             │  Monitor closely. Consider reliability │
│                     │  improvements alongside features.      │
│  10-25%             │  Reliability focus. New features       │
│                     │  require reliability justification.    │
│  < 10%              │  Reliability freeze. Only bug fixes,   │
│                     │  reliability work, and critical fixes. │
│  0% (exhausted)     │  Mandatory reliability sprint. No new  │
│                     │  features until budget is restored.    │
└─────────────────────┴────────────────────────────────────────┘
```

### Banking-Specific Considerations

In banking, error budget policies have additional constraints:

1. **Compliance SLOs have zero budget**: A 100% SLO (e.g., "all AI decisions must be logged") means any failure is an immediate severity-1 incident. There is no budget to spend.

2. **Regulatory deadlines override budgets**: If a regulatory deadline requires a feature launch, the team launches it. But they document the decision, communicate the risk to stakeholders, and prioritize reliability paydown immediately after.

3. **Customer impact is asymmetric**: In banking, one incorrect AI recommendation can cause more harm than 1,000 slow responses. Error budgets should weight failures by severity.

## Budget Consumption Tracking

### Monthly Budget Tracking Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  ERROR BUDGET: Loan Advisor API - March 2025                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Availability SLO: 99.9%                                     │
│  Budget: 0.1% = 15,000 allowed failures                      │
│                                                              │
│  Failures so far: 2,847                                      │
│  Budget remaining: 81.0%                                     │
│  Burn rate: 0.8x (healthy)                                   │
│  Projected end-of-month: 92% remaining                       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Budget: [████████████████████████████░░░░░░░░] 81%    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Recent consumption:                                         │
│  - Mar 12: -1,234 (vector DB outage, 45 minutes)            │
│  - Mar 14: -567 (LLM provider degradation, 12 minutes)      │
│  - Mar 15: -234 (deployment rollout, expected)              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Budget Consumption by Category

Track what is consuming the budget:

```
Error Budget Consumption - March 2025
  Infrastructure failures:    45% (vector DB, network, compute)
  Deployment issues:          20% (bad deploys, config errors)
  Provider failures:          15% (LLM API errors, rate limits)
  Quality issues:             12% (hallucinations, bad responses)
  Capacity limits:             5% (rate limiting, queuing)
  Unknown:                     3% (uncategorized)
```

This breakdown drives investment decisions. If 45% of budget consumption is infrastructure, invest in infrastructure resilience.

## Decision-Making with Error Budgets

### Feature Launch Decision

```
Decision Framework: Launch New RAG Feature

1. Check error budget status:
   Availability SLO: 97% remaining (HEALTHY)
   Latency SLO: 45% remaining (CONCERNING)
   Quality SLO: 88% remaining (HEALTHY)

2. Assess feature risk:
   - Changes retrieval algorithm: MEDIUM risk to latency
   - Adds reranking step: MEDIUM risk to latency
   - No changes to error paths: LOW risk to availability

3. Decision:
   - Availability and quality budgets are healthy -> APPROVED
   - Latency budget is concerning -> MITIGATION REQUIRED
   - Mitigation: Feature flag with automatic rollback if
     p95 latency increases by > 10%

4. Rollout plan:
   - 5% traffic for 24 hours, monitor latency
   - 25% traffic for 24 hours
   - 100% traffic if latency SLO still healthy
```

### Reliability Investment Decision

```
Decision Framework: Invest in Reliability

Trigger: Latency SLO error budget at 0% for 2 consecutive weeks

Options analysis:
  Option A: Optimize vector DB queries
    - Estimated effort: 2 sprints
    - Expected improvement: 40% latency reduction
    - Cost: Delays 2 features

  Option B: Add caching layer
    - Estimated effort: 3 sprints
    - Expected improvement: 60% latency reduction for cacheable queries
    - Cost: Higher infrastructure cost, cache invalidation complexity

  Option C: Upgrade vector DB hardware
    - Estimated effort: 1 sprint (procurement: 4 weeks)
    - Expected improvement: 30% latency reduction
    - Cost: $15,000/month additional infrastructure

Decision: Option A + Option C
  - Short-term: Optimize queries (2 sprints)
  - Long-term: Upgrade hardware (procurement started)
  - Features delayed: Mortgage calculator integration (reprioritized to next quarter)
```

## Error Budget Reporting

### Weekly Report

```
## Error Budget Weekly Report - Loan Advisor API
## Week of 2025-03-10

### Budget Status
| SLO | Monthly Budget | Used | Remaining | Trend |
|-----|---------------|------|-----------|-------|
| Availability (99.9%) | 15,000 | 2,847 | 81.0% | Stable |
| Latency p95<3s (99%) | 150,000 | 148,200 | 1.2% | CRITICAL |
| Quality >95% (95%) | 750,000 | 112,500 | 85.0% | Improving |

### Actions Required
- LATENCY BUDGET CRITICAL: Team to dedicate 50% of next sprint to latency
- Investigate: p95 latency increased 15% after v2.4.0 deployment
- Root cause: Vector DB index fragmentation

### Incidents This Week
- 2025-03-12: Vector DB fragmentation (45 min, -1,234 budget)
- 2025-03-14: LLM provider rate limiting (8 min, -234 budget)
```

## Error Budget Reset

Error budgets reset at the end of each measurement window (typically 30 days). Important behaviors:

1. **Budget does not roll over**: Unused budget is lost at reset. You cannot "save" budget from a good month to spend in a bad month.

2. **Budget does not carry debt**: A breached SLO does not mean you start the next month in debt. The new month starts fresh.

3. **Trends matter more than point-in-time**: A service that exhausts its budget every month needs architectural changes. A service that occasionally breaches is operating at the right edge of reliability.

## Common Mistakes

1. **Treating budget as a target**: "We have 80% budget remaining, let us use it!" No. The budget is a limit, not a resource to consume.

2. **No policy for budget exhaustion**: An exhausted budget without a defined policy is just a number. Define what happens at each budget level.

3. **Ignoring the quality dimension**: A 99.9% availability SLO with 5% hallucination rate is not acceptable in banking. Quality budgets are essential.

4. **Monthly reporting only**: Check budget status weekly. Monthly checks mean you discover budget exhaustion too late to act.

5. **Blaming teams for budget consumption**: The error budget is a tool for objective decision-making, not a performance metric for teams.

6. **No link to business outcomes**: Connect error budget status to business impact. "0% latency budget means 150 customers per day experience slow responses, resulting in an estimated 5% drop in feature adoption."
