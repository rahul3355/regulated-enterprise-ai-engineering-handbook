# Runbook Template

> Use this template to document operational procedures for production services. An on-call engineer at 3 AM should be able to diagnose and resolve common issues using this runbook alone.

## Template

```markdown
# Runbook: [Service Name]

| Field | Value |
|-------|-------|
| **Service** | [Name] |
| **Team** | [Team name] |
| **On-Call Rotation** | [Rotation name / schedule] |
| **Last Updated** | [YYYY-MM-DD] |
| **Owner** | [Name] |
| **Dashboard** | [Grafana link] |
| **Repository** | [GitHub/GitLab link] |

## What Is This Service?

[2-3 sentence description of what the service does and who depends on it.]

## Architecture

[Brief architecture description with diagram. Link to full design doc if applicable.]

## Dependencies

| Dependency | Type | Critical? | Fallback |
|------------|------|-----------|----------|
| [Service/DB/API] | [Database/API/Cache/etc.] | Yes/No | [What happens if it's down] |

## Key Metrics

| Metric | Normal Range | Alert Threshold | Dashboard |
|--------|-------------|----------------|-----------|
| Error rate | < 0.1% | > 1% | [Link] |
| p95 Latency | < 1000ms | > 2000ms | [Link] |
| Throughput | 50-200 req/min | < 10 req/min | [Link] |
| Memory usage | < 70% | > 85% | [Link] |
| [Business metric] | [Range] | [Threshold] | [Link] |

## Alerts and Troubleshooting

### Alert: [Alert Name]

**Severity:** P0/P1/P2
**Description:** [What this alert means]
**Impact:** [What users experience]

#### Diagnosis

```bash
# Step 1: Check service health
kubectl get pods -n [namespace] -l app=[service]

# Step 2: Check recent logs
kubectl logs -n [namespace] -l app=[service] --tail=100

# Step 3: Check [specific metric]
# [Grafana link or command]
```

#### Resolution

**If [condition A]:**
```bash
# Fix steps
[Command or action]
```

**If [condition B]:**
```bash
# Fix steps
[Command or action]
```

**If unable to resolve within [X] minutes:**
1. Escalate to [team/person]
2. [Escalation contact details]

---

### Alert: [Next Alert Name]

[same structure]

## Common Issues

### Issue: [Brief Description]

**Symptoms:** [What the on-call engineer sees]
**Root Cause:** [What causes this]
**Resolution:**
```bash
# Step-by-step fix
[commands/actions]
```
**Prevention:** [How to prevent recurrence]

---

### Issue: [Next Issue]

[same structure]

## Maintenance Procedures

### Deploying a New Version

```bash
# Steps to deploy
[commands]
```

### Rolling Back

```bash
# Steps to rollback
[commands]
```

### Database Migration

```bash
# Steps to run migration
[commands]
```

### Scaling Up/Down

```bash
# Steps to change replica count
[commands]
```

### Rotating Secrets

```bash
# Steps to rotate [secret name]
[commands]
```

## Contact Information

| Role | Contact | Escalation After |
|------|---------|-----------------|
| Primary on-call | [PagerDuty/Opsgenie] | Immediate |
| Secondary on-call | [PagerDuty/Opsgenie] | 15 minutes |
| Tech lead | [Name, phone] | 30 minutes |
| Engineering manager | [Name, phone] | 1 hour |
| External dependency support | [Team, contact] | As needed |

## Known Issues

| Issue | Impact | Workaround | Fix Planned |
|-------|--------|-----------|-------------|
| [...] | [...] | [...] | [Version/date] |

## Change Log

| Date | Change | Author |
|------|--------|--------|
| YYYY-MM-DD | [What changed] | [Name] |
```

## Example: Filled Runbook Excerpt

```markdown
## Alerts and Troubleshooting

### Alert: High Error Rate (> 1%)

**Severity:** P1
**Description:** More than 1% of requests are returning 5xx errors.
**Impact:** Users see "Service temporarily unavailable" errors.

#### Diagnosis

```bash
# Step 1: Check which pods are failing
kubectl get pods -n genai -l app=genai-chat

# Step 2: Check error logs
kubectl logs -n genai -l app=genai-chat --tail=200 | grep "ERROR"

# Step 3: Check dependency health
kubectl exec -n genai deployment/genai-chat -- \
  curl -s http://genai-vector-db:5432
kubectl exec -n genai deployment/genai-chat -- \
  curl -s http://genai-redis:6379
```

#### Resolution

**If LLM API is returning errors:**
```bash
# Check LLM API status
curl https://status.openai.com

# If provider is down, check if fallback is available
kubectl get configmap genai-config -n genai -o yaml | grep fallback

# Enable fallback
kubectl set env deployment/genai-chat \
  FALLBACK_PROVIDER=anthropic -n genai
```

**If database connection errors:**
```bash
# Check database pod
kubectl get pods -n genai -l app=genai-vector-db

# If pod is crash-looping, check logs
kubectl logs -n genai -l app=genai-vector-db --tail=50

# Restart database pod
kubectl rollout restart statefulset/genai-vector-db -n genai
```

**If unable to resolve within 30 minutes:**
1. Declare incident in #incidents channel
2. Page the tech lead
3. Consider rolling back the last deployment
```
