# Mitigation Strategies

## Overview

Mitigation is the process of reducing or eliminating the impact of an incident. This document covers common mitigation patterns, decision frameworks for selecting mitigation actions, and genai-specific mitigation strategies.

---

## Mitigation Decision Framework

When choosing a mitigation action, the IC should evaluate:

### 1. Speed
How quickly can this action be executed?
- **Immediate** (< 5 minutes): Feature flag toggle, traffic shift
- **Fast** (5-15 minutes): Rollback, scale up
- **Moderate** (15-60 minutes): Hotfix deployment, configuration change
- **Slow** (> 60 minutes): Architecture change, data repair

### 2. Risk
What is the risk of the action causing additional problems?
- **Low**: Reverting to a known-good state
- **Medium**: Scaling resources (may not address root cause)
- **High**: Deploying new code during an incident

### 3. Reversibility
Can the action be easily undone if it does not work?
- **Fully reversible**: Feature flag, traffic routing
- **Partially reversible**: Rollback (may lose new data)
- **Irreversible**: Data deletion, infrastructure destruction

### 4. Effectiveness
How likely is this action to resolve or reduce the impact?
- **Definitive**: Addresses the root cause
- **Partial**: Reduces impact but does not resolve
- **Containment**: Stops the bleeding but does not fix

### Decision Matrix

| Strategy | Speed | Risk | Reversibility | When to Use |
|----------|-------|------|---------------|-------------|
| Rollback | Fast | Low | Fully reversible | Bad deployment suspected |
| Scale Up | Fast | Low | Fully reversible | Resource exhaustion |
| Failover | Fast | Medium | Reversible | Dependency failure |
| Feature Disable | Immediate | Low | Fully reversible | Feature-specific issue |
| Traffic Shift | Immediate | Low | Fully reversible | Regional or version-specific issue |
| Hotfix | Moderate | High | Reversible | Known code fix required |
| Degrade Service | Immediate | Medium | Reversible | Partial dependency failure |
| Data Repair | Slow | High | May be irreversible | Data corruption |

---

## Common Mitigation Patterns

### 1. Rollback

**When**: A recent deployment is suspected as the cause.

**Procedure:**
```bash
# Kubernetes rollback
kubectl rollout undo deployment/<name> -n <namespace>

# Verify rollback
kubectl rollout status deployment/<name> -n <namespace>

# Check health
curl https://<service>/health
```

**Considerations:**
- Rollback to the last known-good version
- Check if database migrations are reversible
- Verify that the rollback resolves the issue
- Monitor for 10 minutes after rollback to confirm

**When NOT to rollback:**
- The root cause is not the deployment (e.g., infrastructure failure, external dependency)
- Data migrations make rollback unsafe
- The "known-good" version also has the bug (bug was present for multiple versions)

### 2. Scale Up

**When**: Resource exhaustion (CPU, memory, connections) is causing degradation.

**Procedure:**
```bash
# Scale deployment replicas
kubectl scale deployment/<name> --replicas=<n> -n <namespace>

# Scale HPA target
kubectl patch hpa/<name> -n <namespace> -p '{"spec":{"maxReplicas":20}}'

# Verify scaling
kubectl get pods -n <namespace> -w
```

**Considerations:**
- Check cluster capacity before scaling (are there available nodes?)
- Scaling addresses symptoms, not root cause
- Cost impact of additional resources
- Scale back down after resolution

### 3. Failover

**When**: A dependency (database, vector DB, LLM API) is unavailable.

**Procedure:**
```python
# LLM provider failover
async def generate_with_failover(request):
    try:
        return await primary_llm.generate(request, timeout=30)
    except (TimeoutError, ConnectionError):
        log_failover_event("primary", "backup")
        return await backup_llm.generate(request)
```

**Considerations:**
- Test failover regularly (chaos engineering)
- Failover may have different capabilities or quality
- Data consistency between primary and backup
- Automatic vs. manual failover decision

### 4. Feature Disable

**When**: A specific feature is causing or contributing to the incident.

**Procedure:**
```python
# Feature flag toggle
if feature_flags.is_enabled("rag_retrieval"):
    context = await retrieve_context(query)
else:
    context = None  # RAG disabled, LLM answers from training data
```

**Considerations:**
- Fastest mitigation option (instant via feature flag)
- May degrade service quality
- Communicate impact to customers
- Easy to re-enable after root cause is fixed

### 5. Traffic Shift

**When**: A specific version, region, or customer segment is affected.

**Procedure:**
```yaml
# Istio traffic shift
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: assistbot
spec:
  http:
    - route:
        - destination:
            host: assistbot
            subset: v2.3.1  # Shift traffic to previous version
          weight: 100
        - destination:
            host: assistbot
            subset: v2.4.0
          weight: 0
```

**Considerations:**
- Gradual shift (not all at once) to monitor impact
- Session affinity may be required
- Monitor the receiving version for overload

### 6. Service Degradation

**When**: A non-critical dependency is unavailable, but the core service can function without it.

**Procedure:**
```python
# Degrade RAG to LLM-only
async def answer_query(query):
    try:
        context = await retrieve_from_vector_db(query, timeout=5)
        # Full RAG: context + LLM
        return await llm.generate(prompt=query, context=context)
    except TimeoutError:
        # Degraded: LLM only (no retrieval)
        return await llm.generate(prompt=query, context=None)
```

**Considerations:**
- Clearly communicate degraded functionality
- Monitor quality of degraded responses
- Set customer expectations

---

## GenAI-Specific Mitigation Strategies

### Prompt Injection Mitigation

| Action | Speed | Effectiveness | Description |
|--------|-------|---------------|-------------|
| Input sanitization | Immediate | Partial | Block known injection patterns |
| System prompt hardening | Fast | Partial | Add explicit "do not follow" directives |
| Output filtering | Immediate | High | Block responses containing system prompt or tool calls |
| Feature disable | Immediate | Definitive | Disable document upload/analysis |
| Model switch | Moderate | Partial | Switch to a more robust model |

### Model Degradation Mitigation

| Action | Speed | Effectiveness | Description |
|--------|-------|---------------|-------------|
| Rollback to previous model | Fast | Definitive | Revert to last known-good model version |
| Confidence threshold increase | Immediate | Partial | Reject low-confidence predictions, escalate to human |
| Retrieval tuning | Fast | Partial | Adjust top_k, similarity threshold |
| Fallback to rules-based | Moderate | Partial | Use deterministic rules for common intents |

### Vector DB Outage Mitigation

| Action | Speed | Effectiveness | Description |
|--------|-------|---------------|-------------|
| Keyword search fallback | Immediate | Partial | Use Elasticsearch instead of vector search |
| Cached responses | Immediate | Partial | Serve cached responses for common queries |
| LLM-only mode | Immediate | Partial | Answer without retrieval context |
| Failover to backup VDB | Fast | Definitive | Switch to secondary vector database |

### Token Cost Explosion Mitigation

| Action | Speed | Effectiveness | Description |
|--------|-------|---------------|-------------|
| Rate limiting | Immediate | High | Cap requests per user/minute |
| Context truncation | Immediate | Partial | Cap prompt size |
| Model downgrade | Fast | High | Switch from GPT-4 to GPT-3.5-Turbo |
| Feature disable | Immediate | Definitive | Disable expensive features |

### PII Leakage Mitigation

| Action | Speed | Effectiveness | Description |
|--------|-------|---------------|-------------|
| Output filtering | Immediate | High | Block responses containing PII patterns |
| Service disable | Immediate | Definitive | Take affected service offline |
| Access control fix | Moderate | Definitive | Fix tenant isolation in retrieval |
| Model switch | Moderate | Partial | Switch to a model with better PII awareness |

---

## Mitigation Runbook Template

```markdown
# Mitigation Runbook: [Action Name]

## When to Use
[Conditions that trigger this mitigation]

## Prerequisites
- [Access/permissions needed]
- [Dependencies]

## Steps
1. [Step 1] - [Command/procedure]
2. [Step 2] - [Command/procedure]
3. [Step 3] - [Command/procedure]

## Verification
- [How to confirm the mitigation worked]
- [Metrics to monitor]
- [Time to wait before assessing]

## Rollback (if needed)
- [How to undo this mitigation]

## Risks
- [What could go wrong]
- [Impact of rollback failure]

## Communication
- [Who to notify when this action is taken]
- [Customer-facing impact to communicate]
```

---

## Post-Mitigation Monitoring

After implementing a mitigation:

1. **Monitor for 10-15 minutes**: Confirm the mitigation had the expected effect
2. **Watch for side effects**: Did the mitigation cause new problems?
3. **Verify resolution criteria**: Are the original symptoms resolved?
4. **Plan next steps**: Is the mitigation a permanent fix or a temporary workaround?
5. **Communicate**: Update stakeholders on mitigation status

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-command.md](incident-command.md) -- Incident commander decision-making
- [triage-playbooks.md](triage-playbooks.md) -- Triage patterns
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
- [war-room-management.md](war-room-management.md) -- War room procedures
