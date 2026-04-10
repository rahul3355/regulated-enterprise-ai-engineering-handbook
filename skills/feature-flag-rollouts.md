# Skill: Feature Flag Rollouts

## Core Principles

1. **Flags Are Temporary Code Paths** — Every feature flag adds complexity and a potential bug path. Flags must have an owner, an expiration date, and a removal plan.
2. **Decouple Deployment from Release** — Deploy code behind a flag, test in production, then enable for users. This is the single most important benefit of feature flags.
3. **Control Blast Radius** — Use flags to limit the impact of failures. Roll out to 1% of users first, monitor, then expand.
4. **Flags Must Be Observable** — Every flag evaluation is a data point. Track who has the flag, how often it's evaluated, and what the outcome was.
5. **Never Ship Dead Code** — Remove flags immediately after the feature is fully rolled out. Stale flags are technical debt that causes production incidents.

## Mental Models

### Feature Flag Types and Use Cases
```
┌─────────────────────────────────────────────────────────────────┐
│                  Feature Flag Taxonomy                          │
│                                                                 │
│  Release Flags                Experiment Flags                   │
│  ──────────────                ──────────────                    │
│  Purpose: Deploy safely       Purpose: A/B test                 │
│  Lifetime: Days-weeks         Lifetime: Weeks-months             │
│  Target: Internal → % → all   Target: Random user segments       │
│  Example: New RAG pipeline    Example: New prompt template       │
│                                                                 │
│  Operational Flags            Permission Flags                   │
│  ──────────────                ──────────────                    │
│  Purpose: Kill switch         Purpose: Access control            │
│  Lifetime: Permanent          Lifetime: Role-based               │
│  Target: All users            Target: Specific roles/groups      │
│  Example: Disable model API   Example: Admin-only features       │
│           during outage                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The Rollout Strategy
```
Phase 1: Internal Testing (Day 1-2)
├── Flag enabled for: engineering team only
├── Monitor: error rate, latency, logs
└── Criteria to proceed: Zero P0/P1 bugs, stable error rate

Phase 2: Dogfood (Day 3-5)
├── Flag enabled for: 5% internal users (volunteers)
├── Monitor: user feedback, error rate, cost
└── Criteria to proceed: Positive feedback, no data issues

Phase 3: Canary (Day 6-10)
├── Flag enabled for: 1% → 5% → 10% of production users
├── Monitor: SLO impact, cost, user behavior
├── Each stage: 24 hours minimum
└── Criteria to proceed: No SLO degradation, cost within budget

Phase 4: Gradual Rollout (Day 11-20)
├── Flag enabled for: 25% → 50% → 75% → 100%
├── Monitor: Same as canary + user adoption
├── Each stage: 24-48 hours
└── Criteria to proceed: Stable metrics, no incidents

Phase 5: Full Release + Flag Removal (Day 21-30)
├── Flag enabled for: 100%
├── Remove flag from code within 2 weeks
├── Remove flag from flag management system
└── Confirm: no references to flag remain
```

### The Feature Flag Checklist
```
□ Flag has a clear name (e.g., "enable_new_rag_pipeline")
□ Flag has an owner (engineer responsible for rollout)
□ Flag has an expiration date (max 30 days from creation)
□ Flag has rollback criteria defined before rollout
□ Flag evaluation is logged (who, when, what value)
□ Flag has a kill switch (can disable instantly)
□ Flag is tested in both ON and OFF states
□ Flag cleanup is tracked as a task (not forgotten)
□ Rollout percentage changes are logged and alertable
□ Flag management system is highly available (degraded to default value)
□ Flag evaluation is fast (< 1ms, cached)
□ Sensitive flags require approval to change (production changes audited)
```

## Step-by-Step Approach

### 1. Feature Flag System (Python)

```python
import os
import time
import hashlib
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

class FlagState(Enum):
    DISABLED = "disabled"
    ENABLED = "enabled"
    PERCENTAGE = "percentage"
    TARGETED = "targeted"

@dataclass
class FeatureFlag:
    name: str
    state: FlagState
    percentage: float = 0.0  # For percentage rollout
    targeted_users: set[str] = field(default_factory=set)
    targeted_groups: set[str] = field(default_factory=set)
    default_value: bool = False
    expiration_date: str | None = None
    owner: str = ""
    description: str = ""

class FeatureFlagService:
    """Production feature flag service with deterministic percentage rollout."""

    def __init__(self):
        self._flags: dict[str, FeatureFlag] = {}
        self._evaluation_log: list[dict] = []

    def register(self, flag: FeatureFlag) -> None:
        self._flags[flag.name] = flag

    def is_enabled(self, flag_name: str, user_id: str | None = None,
                   user_groups: set[str] | None = None) -> bool:
        flag = self._flags.get(flag_name)
        if flag is None:
            return False  # Unknown flag = disabled (safe default)

        # Check expiration
        if flag.expiration_date and time.time() > _parse_timestamp(flag.expiration_date):
            # Log stale flag usage — this is a bug
            self._log_evaluation(flag_name, user_id, False, "expired")
            return flag.default_value

        # Evaluate based on state
        if flag.state == FlagState.DISABLED:
            result = False
        elif flag.state == FlagState.ENABLED:
            result = True
        elif flag.state == FlagState.TARGETED:
            result = self._is_targeted(flag, user_id, user_groups)
        elif flag.state == FlagState.PERCENTAGE:
            result = self._is_in_percentage(flag, user_id)
        else:
            result = flag.default_value

        self._log_evaluation(flag_name, user_id, result, str(flag.state))
        return result

    def _is_targeted(self, flag: FeatureFlag, user_id: str | None,
                     user_groups: set[str] | None) -> bool:
        if user_id and user_id in flag.targeted_users:
            return True
        if user_groups and flag.targeted_groups & user_groups:
            return True
        return False

    def _is_in_percentage(self, flag: FeatureFlag, user_id: str | None) -> bool:
        if not user_id:
            return False
        # Deterministic: same user always gets same result
        hash_value = int(hashlib.md5(f"{flag.name}:{user_id}".encode()).hexdigest(), 16)
        return (hash_value % 100) < flag.percentage

    def _log_evaluation(self, flag_name: str, user_id: str | None,
                        result: bool, reason: str) -> None:
        self._evaluation_log.append({
            "flag": flag_name,
            "user_id": user_id,
            "result": result,
            "reason": reason,
            "timestamp": time.time(),
        })

    @property
    def evaluation_count(self) -> int:
        return len(self._evaluation_log)

# Usage
flags = FeatureFlagService()
flags.register(FeatureFlag(
    name="enable_new_rag_retrieval",
    state=FlagState.PERCENTAGE,
    percentage=10,
    owner="team-genai-platform",
    description="New hybrid search retrieval algorithm",
    expiration_date="2025-06-01",
))

# In the application code:
if flags.is_enabled("enable_new_rag_retrieval", user_id="emp-12345"):
    results = new_retrieval_strategy(query)
else:
    results = old_retrieval_strategy(query)
```

### 2. Rollout with Unleash (Enterprise Feature Flag Platform)

```python
# Integration with Unleash (commonly used in banking environments)
from UnleashClient import UnleashClient

client = UnleashClient(
    url="https://unleash.bank.internal/api",
    app_name="rag-service",
    environment="production",
    custom_headers={"Authorization": "unleash-api-token"},
)

client.initialize_client()

# Usage in application
from UnleashClient.strategies import Strategy

class PercentageRollout(Strategy):
    """Custom strategy for gradual percentage rollout."""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

    def is_enabled(self, context=None, parameters=None):
        if context is None:
            return False
        user_id = context.get("userId")
        if not user_id:
            return False
        percentage = float(parameters.get("percentage", 0))
        hash_value = int(hashlib.md5(f"{user_id}".encode()).hexdigest(), 16)
        return (hash_value % 100) < percentage

# Usage
is_enabled = client.is_enabled(
    "enable_new_embedding_model",
    context={"userId": "emp-12345", "environment": "production"},
)
```

### 3. Feature Flag in TypeScript (Next.js Frontend)

```typescript
// lib/featureFlags.ts
import { cookies } from 'next/headers';

export interface FeatureFlag {
  name: string;
  enabled: boolean;
  variant?: string;
  payload?: Record<string, unknown>;
}

// Server-side flag evaluation
export async function evaluateFlags(
  userId: string,
  userGroups: string[]
): Promise<FeatureFlag[]> {
  const cookieStore = await cookies();
  const flagOverrides = cookieStore.get('feature_flags');
  const parsedOverrides = overrides ? JSON.parse(flagOverrides.value) : {};

  // Fetch flags from internal flag service
  const response = await fetch('https://flags.bank.internal/api/v1/evaluate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId,
      userGroups,
      environment: process.env.NODE_ENV,
      overrides: parsedOverrides,
    }),
    // Cache for 60 seconds — flags don't change that frequently
    next: { revalidate: 60 },
  });

  return response.json();
}

// Client-side hook
export function useFeatureFlag(flagName: string): boolean {
  const flags = useContext(FeatureFlagContext);
  return flags.find(f => f.name === flagName)?.enabled ?? false;
}

// Usage in a React component
function ChatInterface({ userId }: { userId: string }) {
  const showNewUI = useFeatureFlag('enable_new_chat_ui');

  if (showNewUI) {
    return <NewChatInterface />;
  }
  return <LegacyChatInterface />;
}
```

### 4. Kill Switch Implementation

```python
from functools import wraps
from fastapi import HTTPException

class KillSwitch:
    """Operational kill switch for critical services."""

    def __init__(self, flag_service: FeatureFlagService):
        self._flag_service = flag_service

    def protect(self, flag_name: str, fallback_message: str = "Service temporarily unavailable"):
        """Decorator that blocks requests if kill switch is active."""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                if not self._flag_service.is_enabled(flag_name):
                    raise HTTPException(
                        status_code=503,
                        detail=fallback_message,
                    )
                return await func(*args, **kwargs)
            return wrapper
        return decorator

# Usage
kill_switch = KillSwitch(flags)

@app.post("/api/chat")
@kill_switch.protect(
    "rag_service_available",
    fallback_message="The AI assistant is temporarily unavailable. Please try again later."
)
async def chat(request: ChatRequest):
    # This endpoint is blocked if the kill switch is off
    return await generate_response(request)

# To activate the kill switch (e.g., during an incident):
# flags._flags["rag_service_available"].state = FlagState.DISABLED
# All /api/chat requests will immediately return 503
```

### 5. Flag Rollout with Observability

```python
from prometheus_client import Counter, Histogram

# Metrics
FLAG_EVALUATION_TOTAL = Counter(
    'feature_flag_evaluation_total',
    'Total feature flag evaluationsations',
    ['flag_name', 'result', 'reason']
)

FLAG_ROLLOUT_PERCENTAGE = Gauge(
    'feature_flag_rollout_percentage',
    'Current rollout percentage for percentage-based flags',
    ['flag_name']
)

# Instrumented flag service
class InstrumentedFlagService(FeatureFlagService):
    def is_enabled(self, flag_name: str, user_id: str | None = None,
                   user_groups: set[str] | None = None) -> bool:
        result = super().is_enabled(flag_name, user_id, user_groups)
        flag = self._flags.get(flag_name)
        reason = str(flag.state) if flag else "unknown"

        FLAG_EVALUATION_TOTAL.labels(
            flag_name=flag_name,
            result=str(result),
            reason=reason,
        ).inc()

        if flag and flag.state == FlagState.PERCENTAGE:
            FLAG_ROLLOUT_PERCENTAGE.labels(flag_name=flag_name).set(flag.percentage)

        return result
```

### 6. A/B Test Analysis for Flag Variants

```python
from scipy import stats
import numpy as np

def analyze_ab_test(
    control_conversions: int, control_total: int,
    variant_conversions: int, variant_total: int,
    alpha: float = 0.05
) -> dict:
    """
    Analyze A/B test results using statistical significance testing.

    Returns whether the variant is significantly different from control.
    """
    control_rate = control_conversions / control_total
    variant_rate = variant_conversions / variant_total

    # Chi-squared test
    contingency = np.array([
        [control_conversions, control_total - control_conversions],
        [variant_conversions, variant_total - variant_conversions],
    ])
    chi2, p_value, dof, expected = stats.chi2_contingency(contingency)

    # Effect size (relative lift)
    lift = (variant_rate - control_rate) / control_rate

    return {
        "control_rate": control_rate,
        "variant_rate": variant_rate,
        "lift": lift,
        "p_value": p_value,
        "is_significant": p_value < alpha,
        "confidence_level": 1 - alpha,
        "recommendation": (
            "Roll out variant" if p_value < alpha and lift > 0
            else "Keep control" if p_value < alpha and lift < 0
            else "Inconclusive — run longer"
        ),
    }

# Usage: comparing old vs new RAG retrieval
result = analyze_ab_test(
    control_conversions=450,  # Users who found useful results (old)
    control_total=1000,        # Total users (old)
    variant_conversions=520,   # Users who found useful results (new)
    variant_total=1000,        # Total users (new)
)
# {'control_rate': 0.45, 'variant_rate': 0.52, 'lift': 0.155,
#  'p_value': 0.002, 'is_significant': True,
#  'recommendation': 'Roll out variant'}
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Flags without expiration dates | Stale flags accumulate, code becomes unmaintainable | Every flag must have an owner and expiration |
| No flag evaluation logging | Can't debug why a user got a specific experience | Log every evaluation with user context |
| Flag service outage blocks deployment | Can't deploy if flag management system is down | Default to safe value (flag OFF) on service failure |
| Too many flags active simultaneously | Exponential code path combinations, testing nightmare | Limit active flags per service (max 5-10) |
| Not testing the OFF path | Bug only appears when flag is disabled, catches team off-guard | Test both ON and OFF paths equally |
| Changing rollout percentage too fast | No time to detect regressions at each stage | Minimum 24 hours at each percentage stage |
| Flag changes without monitoring | Can't see the impact of enabling a flag | Define metrics before changing rollout percentage |
| Frontend-only flags for backend features | Inconsistent experience, security bypass possible | Evaluate flags server-side for security-critical features |
| Not removing flags after full rollout | Dead code, confusion, technical debt | Flag removal is part of the done definition |
| Storing flags in environment variables | Can't change without redeploying, no audit trail | Use a flag management system (Unleash, LaunchDarkly) |

## Banking-Specific Concerns

1. **Change Approval** — Enabling a flag in production is a configuration change. In many banks, this requires a change request. Automate CR creation with flag change details.
2. **Audit Trail** — Every flag evaluation and change must be logged. Who changed the flag, when, from what value to what value, and why. This is subject to audit.
3. **Compliance Flags** — Some flags control compliance-related behavior (e.g., PII redaction). These flags must never be disabled without compliance team approval. Implement as permission flags with dual control.
4. **Regional Rollouts** — Banking data may be region-specific. A flag rollout must respect data residency. Users in region A should not be affected by a flag rollout targeting region B.
5. **Regulatory Freeze** — During regulatory reporting periods (e.g., quarter-end, Basel III reporting), all feature flag changes may be frozen. Plan rollout schedules accordingly.
6. **Emergency Kill Switch** — Banking operations require the ability to instantly disable any feature. The kill switch must be highly available and work even if the flag management system is down.

## GenAI-Specific Concerns

1. **Prompt Template Flags** — Different prompt templates can produce dramatically different outputs. A/B test prompt templates with a feature flag and measure response quality (not just engagement).
2. **Model Version Flags** — When switching between model versions (e.g., GPT-4 to GPT-4o), use a flag to control the rollout. Different models have different cost, latency, and quality profiles.
3. **Guardrail Configuration Flags** — Guardrail sensitivity (PII detection threshold, toxicity threshold) should be flag-controlled so they can be tuned in production without redeployment.
4. **RAG Pipeline Flags** — When testing a new retrieval strategy (hybrid search, reranking, new embedding model), use a flag to compare the old and new approaches on a subset of traffic.
5. **Token Budget Flags** — Control per-user token budgets with flags. If costs spike, reduce the token budget for non-critical users without redeploying.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Active flag count | > 10 per service | Excessive complexity, testing burden |
| Stale flags (past expiration) | > 0 | Technical debt, potential bugs |
| Flag evaluation latency | > 1ms | Degrading request performance |
| Flag service availability | < 99.99% | Flag outages block rollout changes |
| Rollout percentage change rate | > 2 changes per day | Too aggressive, insufficient monitoring |
| Flag-related incidents | > 0 per quarter | Flag misconfiguration causing outages |
| Code coverage of both flag paths | < 100% | Untested code path is a risk |
| Flag evaluation error rate | > 0.1% | Flag system itself is broken |

## Interview Questions

1. How do you decide whether to use a feature flag vs. a direct deployment?
2. Explain the difference between a release flag and an experiment flag. When would you use each?
3. How would you implement a deterministic percentage rollout (same user always gets the same result)?
4. What happens when your feature flag management system goes down? How do you design for this?
5. How do you ensure feature flags are removed after the feature is fully rolled out?
6. Design a feature flag system for gradually rolling out a new LLM model in production.
7. What metrics would you monitor during a feature flag rollout?
8. How do you handle the case where a flag controls a database schema change that is not backward-compatible?

## Hands-On Exercise

### Exercise: Design a Feature Flag Rollout for a New GenAI Model

**Problem:** Your team has integrated a new LLM (GPT-4o) that is 30% faster and 15% cheaper than the current model (GPT-4). You need to roll it out to production with zero risk of degradation in response quality.

**Constraints:**
- The new model may produce different outputs for the same prompts
- You must measure response quality (grounding, helpfulness, toxicity) during rollout
- The rollout must be reversible instantly at any stage
- You have 100,000 active users
- The banking compliance team requires a full audit trail of all flag changes
- The cost savings must be tracked and reported

**Expected Output:**
- Feature flag configuration with rollout stages
- A/B test design with success criteria
- Rollback criteria and automated kill switch
- Monitoring dashboard definition (what metrics to track at each stage)
- Audit trail logging implementation
- Flag removal plan

**Hints:**
- Use a percentage rollout with 24-hour minimum at each stage
- Define quality metrics (grounding score, toxicity rate) before starting
- Use deterministic hashing so the same user always gets the same model during the test
- Implement a kill switch that can instantly revert to the old model

**Extension:**
- Design an automated rollout system that advances percentage based on metric thresholds (if error rate < 0.1% and grounding > 0.8 for 24h, advance from 10% to 25%)
- Implement a flag conflict detection system that warns when two active flags affect the same code path
- Build a flag cost analysis dashboard showing the cost of maintaining each active flag

---

**Related files:**
- `cicd-devops/deployment-strategies.md` — Deployment patterns
- `genai-platforms/model-selection.md` — Model comparison and selection
- `observability/feature-flag-monitoring.md` — Monitoring flag impact
- `skills/ci-cd-pipeline-design.md` — CI/CD pipeline design
- `engineering-culture/feature-flag-culture.md` — Flag management practices
- `genai-platforms/llm-ops.md` — LLM operations and model management
