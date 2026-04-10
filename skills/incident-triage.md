# Skill: Incident Triage

## Core Principles

1. **Stabilize First, Understand Second** — During an incident, the goal is to restore service. Mitigate the impact (rollback, restart, scale up) before you investigate the root cause.
2. **Time Is the Enemy** — Every minute of outage costs money, trust, and productivity. Have runbooks ready, tooling accessible, and decision-making streamlined.
3. **Communication Is Half the Work** — An incident that nobody knows about is almost as bad as the incident itself. Update stakeholders regularly, even if the update is "we're still investigating."
4. **Blameless Postmortems Build Trust** — Incidents reveal system weaknesses, not individual failures. The goal of a postmortem is to improve the system, not to assign blame.
5. **Every Incident Is a Learning Opportunity** — The best incident response teams come out stronger. Document what went wrong, what went right, and what will change.

## Mental Models

### Incident Severity Classification
```
┌─────────────────────────────────────────────────────────────┐
│               Incident Severity Levels                       │
│                                                             │
│  SEV-1: Critical                                            │
│  ────────────                                               │
│  • Complete service outage                                  │
│  • Data breach or security incident                         │
│  • Regulatory compliance failure                            │
│  • Response: Immediate, all hands, 15-min updates           │
│  • Example: GenAI assistant down for all 100K users         │
│                                                             │
│  SEV-2: High                                                │
│  ─────────                                                  │
│  • Major feature broken, workaround available               │
│  • Significant performance degradation                      │
│  • Data integrity concern (not yet breached)                │
│  • Response: Page on-call, 30-min updates                   │
│  • Example: RAG retrieval returning 50% error rate          │
│                                                             │
│  SEV-3: Medium                                              │
│  ───────────                                                │
│  • Minor feature broken, no workaround                      │
│  • Intermittent failures affecting < 10% of users           │
│  • Non-critical monitoring gap                              │
│  • Response: Slack alert, investigate within 4 hours        │
│  • Example: Document embedding pipeline delayed by 1 hour   │
│                                                             │
│  SEV-4: Low                                                 │
│  ─────────                                                  │
│  • Cosmetic issue, minor bug                                │
│  • No user impact, internal-only issue                      │
│  • Response: Ticket, address in next sprint                 │
│  • Example: Typo in the chat interface                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Incident Response Flow
```
Detection ──▶ Triage ──▶ Mitigation ──▶ Resolution ──▶ Postmortem
    │            │            │              │              │
    ▼            ▼            ▼              ▼              ▼
  Alert      Classify      Stabilize     Confirm fix   Root cause
  (monitor   severity      (rollback,   (verify       analysis
  or user)   (SEV 1-4)    scale up)     metrics)      & actions
                │            │              │              │
                ▼            ▼              ▼              ▼
           Assemble       Communicate   Communicate    Document
           team           to stakeholders  to         and track
                          (30-min cadence) stakeholders  actions
```

### The Triage Decision Tree
```
Incident Reported
        │
        ▼
┌───────────────────────────┐
│ Is the service impacted?  │  ← Check dashboards, synthetic tests
└──────────┬────────────────┘
           │
     ┌─────┴──────┐
     Yes          No
     │            │
     ▼            ▼
┌──────────┐  ┌──────────────┐
│Classify  │  │False positive│
│severity  │  │or monitoring │
│(SEV 1-4) │  │issue         │
└─────┬────┘  └──────────────┘
      │
      ▼
┌───────────────────────────┐
│ What is the blast radius? │  ← How many users/services affected?
└──────────┬────────────────┘
           │
     ┌─────┼─────┬──────────┐
     ▼     ▼     ▼          ▼
  All    One   One      Data
  users  service region   integrity
     │     │      │          │
     ▼     ▼      ▼          ▼
  SEV-1  SEV-2  SEV-2/3   SEV-1
         │               (immediate)
         ▼
┌───────────────────────────┐
│ Is there a known fix?     │  ← Check runbooks, recent changes
└──────────┬────────────────┘
           │
     ┌─────┴──────┐
     Yes          No
     │            │
     ▼            ▼
┌──────────┐  ┌──────────────┐
│Execute   │  │Debug:         │
│mitigation│  │- Check recent │
│(rollback,│  │  changes      │
│ restart) │  │- Check traces │
└──────────┘  │- Check logs   │
              │- Check deps   │
              └──────────────┘
```

### The Incident Triage Checklist
```
□ Acknowledge the alert within SLA (SEV-1: 5 min, SEV-2: 15 min)
□ Classify severity based on impact, not symptoms
□ Assemble the incident team (Incident Commander, Tech Lead, Comms)
□ Open incident channel (#incident-YYYY-MM-DD-service-name)
□ Check dashboards: what's the blast radius?
□ Check recent changes: deployments, config changes, feature flags
□ Check dependent services: are they healthy?
□ Form a hypothesis: what changed and why did it break?
□ Test the hypothesis: rollback, restart, or apply fix
□ Verify the fix: metrics return to normal, synthetic tests pass
□ Communicate resolution to stakeholders
□ Create postmortem within 48 hours
□ Track action items from postmortem
```

## Step-by-Step Approach

### 1. Incident Detection and Initial Response

```
ALERT: SEV-2 — RAG Service Error Rate Elevated
─────────────────────────────────────────────────
Triggered by: Prometheus alert
Alert name: SLO_RagService_HighBurnRate_Ticket
Condition: Error rate > 6x SLO budget burn rate over 30 minutes
Time: 2025-01-15 14:30 UTC

Step 1: Acknowledge the alert (within 15 minutes for SEV-2)
────────────────────────────────────────────────────────────────
[14:32] @oncall-engineer acknowledges the alert in PagerDuty
[14:32] PagerDuty notifies the #genai-platform-oncall Slack channel

Step 2: Initial assessment — check the dashboard
────────────────────────────────────────────────────────────────
[14:33] Open Grafana dashboard for "RAG Service — Overview"
[14:33] Observations:
  - Error rate: 8% (normal: < 0.1%) ← SIGNIFICANT
  - Latency p99: 4.5s (normal: 1.2s) ← DEGRADED
  - Request rate: stable at 500 req/s ← No traffic spike
  - Active users: normal ← Not all users affected

[14:34] Check the error breakdown:
  - 502 Bad Gateway: 6% of requests ← PRIMARY ERROR
  - 504 Gateway Timeout: 2% of requests ← SECONDARY
  - 500 Internal Server Error: < 1%

[14:35] Check which endpoints are affected:
  - POST /api/chat: 15% error rate ← AFFECTED
  - GET /api/documents: 2% error rate ← MINOR
  - GET /health: 0% error rate ← HEALTHY

Step 3: Classify severity
────────────────────────────────────────────────────────────────
- Service is degraded, not fully down
- Core feature (chat) is significantly impacted (15% errors)
- Workaround: users can retry, but experience is poor
- Classification: SEV-2 (major feature degraded)

Step 4: Open incident channel and assemble team
────────────────────────────────────────────────────────────────
[14:36] Create #incident-2025-01-15-rag-errors
[14:36] Invite: @oncall-engineer (IC), @backend-lead (TL), @comms-lead
[14:37] Initial update in channel:
  "SEV-2 incident: RAG service error rate at 8% (normal < 0.1%).
   Primary error: 502 Bad Gateway on POST /api/chat.
   Investigating root cause. Next update in 30 minutes."
```

### 2. Debug the Incident — Follow the Evidence

```
Step 5: Check recent changes
────────────────────────────────────────────────────────────────
[14:38] Check deployment history:
  - Last deployment: 2025-01-15 12:00 UTC — rag-service v2.3.0 → v2.3.1
  - Change: Updated embedding model, increased memory limits
  - ArgoCD sync status: Synced, Healthy

[14:39] Check config changes:
  - DATABASE_URL changed from pg-primary to pg-replica-01
  - This is a read replica, not the primary

[14:40] Check feature flags:
  - enable_new_embedding_model: 50% rollout (started 12:00 UTC)

Step 6: Form a hypothesis
────────────────────────────────────────────────────────────────
Hypothesis: The read replica (pg-replica-01) is lagging behind the primary,
causing the RAG service to fail when it tries to read recently embedded
documents that haven't replicated yet.

Supporting evidence:
- Error started after deployment at 12:00 UTC
- Config change switched to read replica
- Errors are 502 (upstream connection issues) and 504 (timeout)
- Error rate correlates with embedding pipeline activity

Step 7: Test the hypothesis
────────────────────────────────────────────────────────────────
[14:42] Check replica lag:
  $ psql -h pg-replica-01 -c "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;"
  Result: lag_seconds = 45.2  ← 45 seconds behind!

[14:43] Check the RAG service error logs:
  $ splunk search 'service=rag-service status=502 | head 20'
  Result: "connection refused: host=pg-replica-01 port=5432"
  → The replica is rejecting connections, not just lagging

[14:44] Check the replica's status:
  $ kubectl get pods -n genai-db
  pg-replica-01-0  0/1  CrashLoopBackOff  ← THE ROOT CAUSE

[14:45] Check the replica pod logs:
  $ kubectl logs pg-replica-01-0 -n genai-db --previous
  Result: "FATAL: could not connect to primary: connection timeout"
  → The replica lost connection to the primary and crashed

Step 8: Mitigate — restore service
────────────────────────────────────────────────────────────────
Option A: Switch back to primary (fastest fix)
  $ oc set env deployment/rag-service DATABASE_URL=postgresql://...@pg-primary... -n genai-platform
  $ oc rollout status deployment/rag-service -n genai-platform

Option B: Restart the replica (may take 5-10 minutes to sync)
  $ kubectl delete pod pg-replica-01-0 -n genai-db

Decision: Option A — switch to primary immediately.
The replica can be fixed after service is restored.

[14:48] Execute Option A: Switch DATABASE_URL back to primary
[14:50] Deployment rolling out (maxUnavailable=0, maxSurge=1)
[14:53] All pods running v2.3.1 with primary database connection

Step 9: Verify the fix
────────────────────────────────────────────────────────────────
[14:54] Check Grafana:
  - Error rate: dropping from 8% → 2% → 0.1% ✓
  - Latency p99: 4.5s → 1.5s → 1.2s ✓
  - 502 errors: 6% → 0% ✓

[14:55] Run synthetic tests:
  $ pytest tests/synthetic/ -v
  All 5 tests passed ✓

[14:56] Confirm resolution:
  "Service recovered. Error rate returned to normal (< 0.1%).
   Root cause: read replica crashed, RAG service could not connect.
   Mitigation: switched back to primary database.
   Follow-up: investigate why replica crashed, add replica health monitoring."
```

### 3. Incident Communication Template

```markdown
# Incident Update Template

## Initial Notification
> **SEV-2 Incident**: RAG Service Error Rate Elevated
> **Impact**: Chat assistant returning errors for ~15% of requests
> **Started**: 2025-01-15 14:30 UTC
> **Detected by**: Prometheus alert (SLO burn rate)
> **Team investigating**: GenAI Platform
> **Next update**: 30 minutes

## Status Update (every 30 minutes)
> **Current status**: Investigating
> **What we know**: Error rate at 8%, primary error is 502 Bad Gateway
> **What we're doing**: Checking recent changes and database connectivity
> **Impact assessment**: ~15% of chat requests failing, no data loss
> **Next update**: 30 minutes

## Resolution Notification
> **Status**: Resolved
> **Duration**: 26 minutes (14:30 - 14:56 UTC)
> **Root cause**: Read replica crashed, RAG service couldn't connect to database
> **Mitigation**: Switched RAG service to connect to primary database
> **User impact**: ~15% of chat requests failed for 26 minutes
> **Data impact**: No data loss or corruption
> **Next steps**: Postmortem within 48 hours, replica health monitoring
```

### 4. Runbook: RAG Service High Error Rate

```markdown
# Runbook: RAG Service High Error Rate

## Symptoms
- Error rate > 1% on POST /api/chat
- 502 Bad Gateway or 504 Gateway Timeout errors
- SLO burn rate alert firing

## Immediate Actions

### 1. Assess Impact
```bash
# Check current error rate
kubectl exec -it prometheus-0 -n monitoring -- \
  curl -s 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=sum(rate(http_request_errors_total{service="rag-service"}[5m]))'

# Check affected endpoints
kubectl exec -it prometheus-0 -n monitoring -- \
  curl -s 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=sum(rate(http_request_errors_total{service="rag-service"}[5m])) by (endpoint)'
```

### 2. Check Recent Changes
```bash
# Check recent deployments
kubectl get events -n genai-platform --sort-by='.lastTimestamp' | tail -20

# Check ArgoCD sync status
argocd app get rag-service

# Check feature flag changes
curl https://flags.bank.internal/api/flags/recent-changes
```

### 3. Check Dependencies
```bash
# Database connectivity
kubectl exec -it rag-service-pod -n genai-platform -- \
  python -c "import psycopg2; psycopg2.connect('DATABASE_URL')"

# Model gateway connectivity
kubectl exec -it rag-service-pod -n genai-platform -- \
  curl -f http://model-gateway.inference.svc:8080/health

# Redis connectivity
kubectl exec -it rag-service-pod -n genai-platform -- \
  redis-cli -h redis-cache.genai-platform.svc ping
```

### 4. Mitigation Options

#### Option A: Rollback last deployment
```bash
kubectl rollout undo deployment/rag-service -n genai-platform
kubectl rollout status deployment/rag-service -n genai-platform
```

#### Option B: Scale up replicas
```bash
kubectl scale deployment/rag-service --replicas=6 -n genai-platform
```

#### Option C: Enable maintenance mode
```bash
# Enable kill switch — return friendly error to users
kubectl patch configmap rag-service-config -n genai-platform \
  --patch '{"data":{"MAINTENANCE_MODE":"true"}}'
```

## Escalation Criteria
- If error rate > 50% → escalate to SEV-1
- If data integrity concern → involve database team immediately
- If security incident → involve security team immediately

## Post-Resolution
1. Verify metrics return to normal
2. Run synthetic test suite
3. Communicate resolution to stakeholders
4. Create postmortem within 48 hours
```

### 5. Blameless Postmortem Template

```markdown
# Postmortem: RAG Service Error Rate Incident

**Date:** 2025-01-15
**Incident Duration:** 26 minutes (14:30 - 14:56 UTC)
**Severity:** SEV-2
**Author:** @oncall-engineer
**Attendees:** @backend-lead, @platform-lead, @dba-lead

## Summary
The RAG service experienced elevated error rates (8%) for 26 minutes due to
a database read replica crash. The service was configured to use the replica
for read queries, and when the replica crashed, the RAG service could not
retrieve documents, resulting in 502 errors.

## Timeline
| Time (UTC) | Event |
|------------|-------|
| 12:00 | Deployment of rag-service v2.3.1, switched to read replica |
| 14:28 | Read replica crashed due to primary connection timeout |
| 14:30 | Prometheus alert fired (SLO burn rate) |
| 14:32 | On-call engineer acknowledged |
| 14:45 | Root cause identified (replica CrashLoopBackOff) |
| 14:48 | Mitigation applied (switched to primary) |
| 14:56 | Service fully recovered |

## Root Cause
The read replica lost its replication connection to the primary database
due to a transient network issue. The replica pod entered CrashLoopBackOff
because it couldn't reconnect automatically. The RAG service had no
fallback to the primary database when the replica was unavailable.

## Contributing Factors
1. **No fallback**: RAG service had no logic to fall back to primary when replica is down
2. **No replica monitoring**: No alert on replica lag or replica health
3. **Aggressive replica restart policy**: Kubernetes restarted the pod before the network recovered
4. **Deployment timing**: The config change to use the replica was deployed at the same time as the model change, making debugging harder

## What Went Well
- Alert fired within 2 minutes of SLO breach
- On-call engineer acknowledged within 2 minutes
- Runbook was followed correctly
- Rollback was fast (8 minutes from decision to recovery)

## What Went Wrong
- Took 15 minutes to identify root cause (checking wrong places first)
- No monitoring on replica health
- No fallback mechanism in application code
- Two changes deployed simultaneously (model + database config)

## Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Add replica health monitoring | @dba-lead | 2025-01-22 | High |
| Add fallback to primary in RAG service | @backend-lead | 2025-01-29 | High |
| Separate database config changes from feature deployments | @platform-lead | 2025-01-20 | Medium |
| Add runbook section for replica failure | @oncall-engineer | 2025-01-18 | Medium |
| Test replica failover in staging | @dba-lead | 2025-02-01 | Medium |

## Lessons Learned
1. **Never deploy infrastructure and application changes together** — they make debugging twice as hard.
2. **Replica health is as important as primary health** — we monitored the primary but not the replica.
3. **Fallback logic is essential** — any system that depends on a single replica needs a fallback.
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| No runbooks for common failures | Wasted time debugging known issues | Create and maintain runbooks |
| Blaming individuals in postmortems | People hide mistakes, no learning | Blameless culture, focus on systems |
| No incident commander | Confusion, conflicting actions | Designate IC for every SEV-1/SEV-2 |
| Ignoring the incident until escalation | Longer outage, more damage | Alert and respond within SLA |
| Not communicating during the incident | Stakeholders lose trust | Regular updates, even if no progress |
| Fixing without understanding | Same incident recurs | Understand root cause before fixing (unless emergency) |
| No postmortem follow-through | Same mistakes repeated | Track action items, review in team meetings |
| Alert fatigue (too many false positives) | Real alerts ignored | Tune alert thresholds, use SLO-based alerting |
| Not practicing incident response | Slow response during real incidents | Regular game days, chaos engineering |
| Postmortem written 2 weeks later | Details forgotten, actions don't get done | Write within 48 hours, review within 1 week |

## Banking-Specific Concerns

1. **Regulatory Reporting** — SEV-1 incidents affecting customer-facing services may require regulatory notification. Know the reporting requirements and timelines.
2. **Business Impact Quantification** — Banking leadership needs to understand incident impact in business terms: number of affected customers, transaction volume lost, reputational risk.
3. **Escalation to Executive Leadership** — SEV-1 incidents must be escalated to the CIO/CTO within 30 minutes. Have a predefined escalation path.
4. **Cross-Team Coordination** — Banking incidents often involve multiple teams (networking, database, application, security). The incident commander coordinates across teams.
5. **Audit of Incident Response** — Regulators may review incident response procedures. Document the timeline, decisions, and rationale thoroughly.

## GenAI-Specific Concerns

1. **Model Degradation Incidents** — An LLM provider may change model behavior without changing the version. This manifests as "quality degradation" rather than a traditional outage. Detect with evaluation tests.
2. **Guardrail False Positive Storms** — A guardrail update may start blocking legitimate requests at high rates. Monitor guardrail block rates as an incident signal.
3. **Token Budget Exhaustion** — If the daily token budget is exceeded, the service starts rejecting requests. Monitor token usage and alert at 80% of budget.
4. **Embedding Pipeline Backlog** — If the embedding pipeline falls behind, the RAG system serves stale data. This is a slow-building incident that may go unnoticed for hours.
5. **Prompt Template Errors** — A bad prompt template change can cause the LLM to produce nonsensical or harmful outputs. Monitor output quality metrics.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Time to detect (TTD) | > 5 minutes for SEV-1 | Detection gap |
| Time to acknowledge (TTA) | > 15 minutes for SEV-2 | On-call responsiveness |
| Time to mitigate (TTM) | > 1 hour for SEV-2 | Mitigation efficiency |
| Incident recurrence rate | Same incident > 1x per quarter | Action items not effective |
| Postmortem completion rate | < 100% within 48 hours | Process adherence |
| Action item completion rate | < 80% within due date | Follow-through |
| False positive alert rate | > 50% | Alert fatigue |
| SEV-1 incidents per quarter | Track trend | Overall reliability |
| User impact per incident | Track affected user count | Business impact |
| Cost per incident | Track financial impact | Investment justification |

## Interview Questions

1. Walk me through how you would respond to a SEV-1 incident where the GenAI assistant is completely down.
2. How do you decide whether to rollback or debug forward during an incident?
3. What makes a good runbook? Give an example.
4. How do you write a blameless postmortem? What should it include and what should it avoid?
5. What is the difference between time to detect, time to acknowledge, and time to mitigate?
6. How would you handle an incident where the root cause is unclear but the service is degraded?
7. What metrics would you track to measure incident response effectiveness?
8. How do you balance thorough debugging with the need to restore service quickly?

## Hands-On Exercise

### Exercise: Run an Incident Response Simulation

**Problem:** It's 3:00 PM on a Thursday. You receive a PagerDuty notification:

```
SEV-1: GenAI Assistant Unresponsive
Alert: Synthetic test failures on all endpoints
Service: rag-service
Impact: All users unable to use the GenAI assistant
Started: 2025-01-15 15:00 UTC
```

**Scenario:**
- The synthetic tests are failing with 502 errors on all endpoints
- The Grafana dashboard shows 100% error rate
- The last deployment was 2 hours ago (rag-service v2.3.1)
- The model gateway is reporting "connection refused" errors
- The OpenShift cluster shows 3 pods in CrashLoopBackOff
- There is a network policy change that was applied 30 minutes ago
- The compliance team is asking about impact for an ongoing audit

**Constraints:**
- You are the Incident Commander
- You have access to all monitoring tools (Grafana, Splunk, Jaeger)
- You have access to OpenShift and ArgoCD
- You must communicate updates every 15 minutes
- You must restore service as quickly as possible

**Expected Output:**
- Initial assessment and severity classification
- Incident channel setup and team assembly
- Debugging timeline showing your investigation steps
- Mitigation decision and execution
- Resolution verification and communication
- Draft postmortem with root cause, timeline, and action items

**Hints:**
- Start with the dashboards — what's the blast radius?
- Check recent changes — what changed in the last 2 hours?
- The network policy change is a strong signal — check if it's blocking traffic
- The fastest mitigation may be to rollback the network policy change
- Don't forget to communicate with the compliance team

**Extension:**
- Design a chaos engineering experiment to test this failure mode proactively
- Create a self-healing mechanism that detects and recovers from this failure automatically
- Write a runbook that would help the next on-call engineer handle this faster

---

**Related files:**
- `incident-management/incident-response.md` — Incident response process
- `incident-management/postmortems.md` — Postmortem guide
- `skills/observability-design.md` — Observability for incident detection
- `skills/kubernetes-debugging.md` — Kubernetes debugging during incidents
- `skills/distributed-system-debugging.md` — Cross-service debugging
