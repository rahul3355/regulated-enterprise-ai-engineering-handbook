# Incident Debugging Playbooks for Banking GenAI

## Purpose

Playbooks provide step-by-step guidance for diagnosing and resolving common incidents. During an incident, cognitive load is high. Playbooks reduce decision fatigue by providing pre-defined diagnostic paths.

## How to Use Playbooks

1. **Identify the symptom**: What are users experiencing?
2. **Follow the diagnostic steps**: Execute checks in order.
3. **Identify the root cause**: Match symptoms to known causes.
4. **Apply the remediation**: Execute the fix.
5. **Verify resolution**: Confirm the issue is resolved.
6. **Document**: Create an incident report.

---

## Playbook 1: High API Latency

### Symptom
Users report slow responses. p95 latency above SLO threshold (3s).

### Diagnostic Steps

```
Step 1: Check which component is slow
  -> Open Latency Breakdown dashboard
  -> Identify the component with increased latency

  If model_inference is slow:
    -> Go to Step 2

  If retrieval is slow:
    -> Go to Step 3

  If embedding is slow:
    -> Go to Step 4

  If all components are slow:
    -> Check infrastructure (CPU, memory, network)
    -> Go to Step 5
```

### Step 2: Model Inference Slow

```
Possible causes:
A. LLM provider degradation
B. Increased request complexity
C. Provider rate limiting

Diagnostic:
1. Check LLM provider status page
   $ curl https://status.openai.com
   $ curl https://status.anthropic.com

2. Check provider latency in dashboard
   -> Compare to 24h and 7d baseline
   -> If > 50% above baseline, provider-side issue

3. Check average prompt token count
   -> Query: avg(tokens_per_request{direction="input"})
   -> If increased significantly, request complexity increased

4. Check for rate limiting
   -> Check: rate(dependency_errors_total{error_type="rate_limit"})
   -> If rate limited, requests are being throttled

Remediation:
- If provider degradation: Switch to backup provider
  $ export LLM_PRIMARY_PROVIDER="anthropic"
  $ export LLM_FALLBACK_PROVIDER="openai"

- If request complexity: Review recent prompt template changes
  $ git log --oneline -10 -- src/prompts/

- If rate limiting: Enable request queuing
  $ kubectl scale deployment llm-gateway --replicas=6
```

### Step 3: Retrieval Slow

```
Possible causes:
A. Vector DB overload
B. Index fragmentation
C. Increased document volume

Diagnostic:
1. Check vector DB connection pool
   $ psql -h vector-db -c "SELECT count(*) FROM pg_stat_activity;"

2. Check vector DB query latency
   -> Query: histogram_quantile(0.95, rate(db_query_duration_seconds_bucket{table="embeddings"}))

3. Check collection size
   -> Query: vector_db_collection_size{collection="banking-policies"}
   -> If collection has grown significantly, index may need rebuild

Remediation:
- If connection pool exhausted:
  $ psql -h vector-db -c "ALTER SYSTEM SET max_connections = 200;"
  $ psql -h vector-db -c "SELECT pg_reload_conf();"

- If index fragmented:
  $ python scripts/rebuild_vector_index.py --collection banking-policies

- If collection too large:
  -> Review document retention policy
  -> Archive documents older than regulatory minimum
```

### Step 4: Embedding Slow

```
Diagnostic:
1. Check embedding service latency
   -> Query: histogram_quantile(0.95, rate(embedding_generation_seconds_bucket))

2. Check embedding model version
   -> Verify correct model is deployed

Remediation:
- If embedding service is a separate deployment:
  $ kubectl rollout restart deployment embedding-service

- If using external embedding API:
  -> Check provider status
  -> Consider switching to local embedding model
```

### Step 5: Infrastructure Check

```
Diagnostic:
1. Check CPU utilization
   $ kubectl top pods -l app=loan-advisor-api

2. Check memory utilization
   $ kubectl top pods -l app=loan-advisor-api --containers

3. Check network throughput
   $ kubectl exec -it <pod> -- iftop

4. Check disk I/O
   $ kubectl exec -it <pod> -- iostat

Remediation:
- If CPU throttling:
  $ kubectl patch deployment loan-advisor-api -p '{"spec":{"template":{"spec":{"containers":[{"name":"loan-advisor","resources":{"limits":{"cpu":"2000m"}}}]}}}}'

- If memory pressure:
  $ kubectl top pods -l app=loan-advisor-api
  -> Check for memory leaks
  -> Consider restarting pods one at a time
  $ kubectl rollout restart deployment loan-advisor-api
```

---

## Playbook 2: Increased Error Rate

### Symptom
Error rate above SLO threshold (0.1%).

### Diagnostic Steps

```
Step 1: Categorize errors
  -> Check error breakdown dashboard
  -> Group errors by status code and error type

  If 5xx errors:
    -> Server-side errors, go to Step 2

  If 4xx errors increased:
    -> Client-side or validation errors, go to Step 3

  If specific error type spike:
    -> Go to Step 4
```

### Step 2: 5xx Errors

```
Diagnostic:
1. Check error logs for stack traces
   $ grep -i "level.*ERROR" logs | jq -r '.error_type' | sort | uniq -c | sort -rn

2. Check recent deployments
   $ kubectl get events --sort-by='.lastTimestamp' | grep -i deploy

3. Check dependency health
   -> Open Dependencies dashboard
   -> Look for unhealthy dependencies

4. Check database connectivity
   $ psql -h postgres-db -c "SELECT 1;"

Remediation:
- If recent deployment caused errors:
  $ kubectl rollout undo deployment/loan-advisor-api

- If dependency unhealthy:
  -> Follow dependency-specific playbook

- If database connectivity issue:
  $ kubectl exec -it <pod> -- nc -zv postgres-db 5432
  -> Check network policies
  -> Check connection pool
```

### Step 3: 4xx Errors

```
Diagnostic:
1. Check if validation rules changed
   -> Review recent API schema changes

2. Check if client application changed
   -> Coordinate with frontend team

3. Check for malformed requests
   -> Sample recent 4xx requests and inspect
```

### Step 4: Specific Error Type Spike

```
Diagnostic:
1. Identify the error type
   -> Check error_type field in logs

2. Check if correlated with specific model, endpoint, or product line
   -> Filter error rate by label

Common error types and fixes:
- VectorDBTimeout: Increase timeout, rebuild index (see Playbook 1, Step 3)
- LLMProviderError: Check provider status, switch to fallback
- PromptTooLong: Reduce context documents, check prompt template
- AuthenticationError: Check SSO provider health, token validation
```

---

## Playbook 3: LLM Provider Outage

### Symptom
LLM provider returning errors or not responding.

### Diagnostic Steps

```
Step 1: Confirm outage
  1. Check provider status page
  2. Run synthetic check
     $ python scripts/check_llm_provider.py --provider openai
  3. Check error rate from provider
     -> Query: rate(dependency_errors_total{dependency="openai-api"})
```

### Step 2: Activate Failover

```
1. Switch to backup provider
   $ kubectl set env deployment/llm-gateway LLM_PRIMARY_PROVIDER=anthropic

2. Verify failover
   $ python scripts/check_llm_provider.py --provider anthropic

3. Monitor error rate after failover
   -> Should decrease within 2-5 minutes
```

### Step 3: If No Backup Available

```
1. Enable cached responses
   $ kubectl set env deployment/llm-gateway CACHE_ONLY_MODE=true

2. Show user-facing message
   -> "Our AI advisory system is temporarily using cached responses.
       For the most current information, please try again shortly."

3. Estimate time to recovery
   -> Monitor provider status page
   -> Update stakeholders every 15 minutes
```

---

## Playbook 4: PII Detected in Logs

### Symptom
Automated PII scan detects sensitive data in logs.

### Immediate Actions (within 5 minutes)

```
1. CONFIRM the detection
   $ grep -r "ssn\|credit_card\|account_number" /var/log/app/

2. STOP the log source
   -> Identify which service is logging PII
   -> Disable verbose logging for that component
   $ kubectl set env deployment/<service> LOG_LEVEL=WARN

3. QUARANTINE affected log files
   -> Move to secure, access-controlled storage
   $ mv /var/log/app/*.log /secure/quarantine/
```

### Remediation (within 1 hour)

```
4. ASSESS scope
   -> How long was PII being logged?
   -> Which log destinations received PII?
   -> What types of PII were exposed?

5. CLEAN UP
   -> Redact PII from all log destinations
   -> Purge from log aggregation system
   -> Verify no backups contain PII

6. FILE compliance incident
   -> Notify compliance team
   -> Document timeline and scope
   -> Begin regulatory notification if required
```

### Prevention

```
7. FIX the root cause
   -> Update redaction patterns
   -> Add PII detection to CI/CD
   -> Add automated PII scanning to log pipeline

8. REVIEW all services
   -> Audit logging across all services
   -> Test redaction with edge cases
   -> Update runbook with lessons learned
```

---

## Playbook 5: Hallucination Spike

### Symptom
Hallucination rate exceeds threshold (> 5%).

### Diagnostic Steps

```
Step 1: Identify scope
  -> Which model is affected?
  -> Which product line?
  -> When did it start?

Step 2: Check for recent changes
  1. Model version changes
     -> Check model governance log

  2. Knowledge base changes
     -> Check vector DB document update log

  3. Prompt template changes
     -> Check prompt template git history

Step 3: Analyze hallucinations
  1. Sample recent hallucinations
     -> Read 10-20 recent hallucination examples
     -> Identify pattern (wrong data, fabricated info, outdated info)

  2. Check source document freshness
     -> Are retrieved documents outdated?

Remediation:
- If model version change caused it:
  -> Rollback to previous model version
  -> File model quality incident

- If knowledge base outdated:
  -> Trigger knowledge base refresh
  -> Remove outdated documents

- If prompt template issue:
  -> Revert prompt template changes
  -> Review and fix prompt
```

---

## Playbook Quick Reference

```
┌──────────────────────────┬──────────────┬────────────────────┐
│  Incident                │  First Step  │  Runbook           │
├──────────────────────────┼──────────────┼────────────────────┤
│  High API latency        │  Check       │  Playbook 1        │
│                          │  latency     │                    │
│                          │  breakdown   │                    │
├──────────────────────────┼──────────────┼────────────────────┤
│  Increased error rate    │  Categorize  │  Playbook 2        │
│                          │  errors      │                    │
├──────────────────────────┼──────────────┼────────────────────┤
│  LLM provider outage     │  Confirm,    │  Playbook 3        │
│                          │  failover    │                    │
├──────────────────────────┼──────────────┼────────────────────┤
│  PII in logs             │  Confirm,    │  Playbook 4        │
│                          │  stop,       │                    │
│                          │  quarantine  │                    │
├──────────────────────────┼──────────────┼────────────────────┤
│  Hallucination spike     │  Identify    │  Playbook 5        │
│                          │  scope and   │                    │
│                          │  recent chg  │                    │
├──────────────────────────┼──────────────┼────────────────────┤
│  Database connection     │  Check pool  │  Infra runbook     │
│  pool exhausted          │  and restart │                    │
├──────────────────────────┼──────────────┼────────────────────┤
│  Kubernetes pod crash    │  Check pod   │  K8s runbook       │
│  loop                    │  logs and    │                    │
│                          │  events      │                    │
└──────────────────────────┴──────────────┴────────────────────┘
```
