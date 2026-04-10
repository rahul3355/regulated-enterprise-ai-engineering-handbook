# GenAI-Specific Incidents

## Overview

GenAI systems introduce unique incident patterns that do not exist in traditional software systems. This document catalogs these patterns, provides playbooks for each, and maps them to standard incident management procedures.

---

## Incident Catalog

### 1. Prompt Injection

**Description**: Malicious input overrides system instructions, causing the LLM to follow attacker-defined behavior.

**Detection**:
- Prompt injection pattern matching on input
- Anomalous LLM output (system prompt content in response, unexpected tool calls)
- User reports of unusual AI behavior

**Severity**: SEV-1 (if successful), SEV-2 (if detected and blocked)

**Playbook:**

```
1. CONFIRM: Did the injection succeed?
   - Check LLM output for signs of following malicious instructions
   - Check if system prompt was leaked
   - Check if unauthorized tool calls were executed

2. CONTAIN:
   - If successful: Disable the affected feature immediately
   - Block the malicious input pattern
   - Preserve evidence (input, output, session logs)

3. ASSESS SCOPE:
   - How many queries were affected?
   - Was any data exfiltrated?
   - Were any unauthorized actions taken?

4. MITIGATE:
   - Harden system prompt with explicit "do not follow" directives
   - Add input sanitization for the injection pattern
   - Add output filtering to block system prompt content

5. INVESTIGATE:
   - Full log analysis of affected sessions
   - Determine attack vector (direct input, uploaded document, etc.)
   - Identify any data exposure or unauthorized actions

6. REMEDIATE:
   - Fix the vulnerability (prompt design, input sanitization, output filtering)
   - Rotate any exposed credentials
   - Notify affected parties (regulators, customers if data exposed)

7. PREVENT:
   - Add injection pattern to detection rules
   - Red team test for similar injection vectors
   - Update prompt injection defense documentation
```

### 2. Model Degradation

**Description**: Model quality declines over time, producing incorrect or low-confidence outputs.

**Detection**:
- Model quality monitoring (accuracy, PSI drift)
- Human correction rate increase
- Customer complaints about incorrect answers

**Severity**: SEV-2 (if significant accuracy drop), SEV-3 (if minor degradation)

**Playbook:**

```
1. CONFIRM: Is the model actually degraded?
   - Check accuracy metrics against baseline
   - Check confidence scores distribution
   - Check human correction rate
   - Compare recent predictions to labeled test set

2. IDENTIFY CAUSE:
   - Data drift? (input distribution changed)
   - Model regression? (bad fine-tuning)
   - Preprocessing change? (tokenization, embedding update)
   - Context retrieval issue? (vector DB problem)

3. MITIGATE:
   - Rollback to previous model version if available
   - Increase confidence threshold to reject uncertain predictions
   - Enable human-in-the-loop review for low-confidence predictions
   - Switch to fallback model (rules-based or simpler model)

4. REMEDIATE:
   - If data drift: Retrain model with current data distribution
   - If model regression: Debug fine-tuning pipeline, retrain
   - If preprocessing: Revert preprocessing change
   - If retrieval: Fix vector DB or retrieval pipeline

5. PREVENT:
   - Implement continuous model quality monitoring
   - Add drift detection alerts
   - Regular model evaluation against holdout test set
   - Automated retraining pipeline with human corrections
```

### 3. Vector Database Outage

**Description**: Vector database is unavailable or degraded, affecting RAG retrieval.

**Detection**:
- Vector DB unreachable alert
- Retrieval latency spike
- Retrieval score drop
- RAG service errors

**Severity**: SEV-1 (complete outage), SEV-2 (degraded performance)

**Playbook:**

```
1. CONFIRM: Is the vector DB actually down?
   - Test connectivity from application
   - Check vector DB health endpoint
   - Check managed provider status page (if applicable)

2. ACTIVATE FALLBACK:
   - Switch to keyword search (Elasticsearch)
   - Or: LLM-only mode (no retrieval context)
   - Or: Cached responses for common queries

3. ASSESS IMPACT:
   - What percentage of queries are affected?
   - Is fallback providing acceptable quality?
   - Are customers seeing degraded responses?

4. INVESTIGATE:
   - If self-hosted: Check pod status, resource utilization
   - If managed: Check provider incident updates
   - Check for recent maintenance or configuration changes

5. RESTORE:
   - Restart vector DB if self-hosted and recoverable
   - Wait for provider recovery if managed
   - Verify data integrity after recovery

6. PREVENT:
   - Implement multi-provider vector DB strategy
   - Add automatic failover to backup vector DB
   - Improve monitoring and early warning
   - Regular chaos engineering tests
```

### 4. Token Cost Explosion

**Description**: LLM API token costs exceed budget significantly.

**Detection**:
- Token cost anomaly alert
- Daily cost exceeding forecast by > 150%
- Per-service cost exceeding allocation

**Severity**: SEV-3 (100-200% over budget), SEV-2 (> 200% over budget)

**Playbook:**

```
1. CONFIRM: Is the cost increase real?
   - Check token usage by service
   - Check traffic volume (more users = more cost)
   - Check prompt size (larger context = more cost)
   - Check model pricing (did model change?)

2. IDENTIFY DRIVER:
   - Which service is driving the increase?
   - Is it from more queries, larger prompts, or more expensive model?
   - Was a new feature deployed that increases context?

3. MITIGATE:
   - Apply rate limits to the offending service
   - Reduce context window size
   - Downgrade model (GPT-4 -> GPT-3.5-Turbo) for non-critical tasks
   - Disable expensive features temporarily

4. REMEDIATE:
   - Fix the configuration or feature causing the increase
   - Implement per-service cost budgets with automatic enforcement
   - Add cost per query monitoring dashboard

5. PREVENT:
   - Cost review for all new GenAI features
   - Automated cost alerts per service
   - Context size limits at the API gateway level
   - Regular cost optimization reviews
```

### 5. PII Leakage Through AI Output

**Description**: LLM output contains personally identifiable information that should not be visible.

**Detection**:
- PII pattern detection on LLM output
- Customer reports of seeing another customer's data
- Audit log analysis revealing cross-tenant data access

**Severity**: SEV-1 (confirmed cross-customer PII exposure)

**Playbook:**

```
1. CONFIRM: What PII was exposed and to whom?
   - Identify the PII type and sensitivity
   - Determine which customer's data was exposed
   - Determine who received the exposed data
   - Check if it's same-customer or cross-customer exposure

2. CONTAIN:
   - If cross-customer: Take the affected service offline immediately
   - Block the specific retrieval or generation path causing the leak
   - Preserve all evidence (logs, responses, session data)

3. ASSESS SCOPE:
   - How many queries returned PII?
   - How many customers' data was exposed?
   - How long has the leakage been occurring?
   - Was any PII stored or cached by recipients?

4. INVESTIGATE ROOT CAUSE:
   - Check tenant isolation in retrieval layer
   - Check system prompt for data exposure
   - Check training data for PII contamination
   - Check for prompt injection causing data exfiltration

5. NOTIFY:
   - Legal and compliance teams
   - DPO (Data Protection Officer)
   - Regulators (ICO within 72 hours per GDPR)
   - Affected customers

6. REMEDIATE:
   - Fix the root cause (tenant isolation, output filtering, etc.)
   - Implement PII detection on all LLM output
   - Add PII redaction pipeline
   - Audit all GenAI systems for similar vulnerabilities

7. PREVENT:
   - Regular PII scanning of training data and vector DB
   - Mandatory output PII filtering for all GenAI services
   - Quarterly red team testing for data leakage
   - Strict tenant isolation at every layer
```

### 6. GPU Resource Exhaustion

**Description**: GPU resources are fully consumed, causing model serving pods to crash or degrade.

**Detection**:
- GPU memory utilization > 95%
- Model serving pods OOM crashing
- Inference latency spike
- DCGM exporter alerts

**Severity**: SEV-2 (partial degradation), SEV-1 (complete model serving outage)

**Playbook:**

```
1. CONFIRM: Is it GPU resource exhaustion?
   - Check nvidia-smi on affected nodes
   - Check DCGM metrics: GPU memory, utilization
   - Check pod crash logs for OOM errors

2. IDENTIFY THE CONSUMER:
   - Is it a training job on inference GPUs?
   - Is it a memory leak in the inference server?
   - Is it an unusually large batch of requests?

3. MITIGATE:
   - If training job: Kill the training pod
   - If inference server: Restart the server
   - If traffic spike: Scale up or rate limit
   - Rebalance pods across GPU nodes

4. RESTORE:
   - Verify inference pods are running
   - Check inference latency returning to normal
   - Monitor for recurrence

5. PREVENT:
   - Separate training and inference node pools
   - Configure MIG for GPU partitioning
   - Set pod priority: inference > training
   - GPU memory monitoring and alerting
   - Runbook for GPU troubleshooting
```

### 7. Hallucinated Output

**Description**: LLM generates factually incorrect or fabricated information that is presented as fact.

**Detection**:
- Human fact-checking of AI output
- Customer reports of incorrect information
- Automated fact-checking pipeline (if available)
- Confidence score anomaly (low confidence but presented as certain)

**Severity**: SEV-2 (if advice is actionable and wrong), SEV-3 (if informational only)

**Playbook:**

```
1. CONFIRM: Is the output actually hallucinated?
   - Fact-check against source documents
   - Check if retrieval context supports the answer
   - Check model confidence score

2. ASSESS IMPACT:
   - How many users received the hallucinated output?
   - Was the output actionable (did users act on it)?
   - If financial advice: Was the advice incorrect?
   - If compliance advice: Was it wrong?

3. MITIGATE:
   - Add confidence threshold: reject low-confidence outputs
   - Add fact-checking layer: verify output against source context
   - Add "I'm not sure" behavior: model should express uncertainty
   - Enable human review for high-stakes topics

4. REMEDIATE:
   - Improve RAG retrieval quality (better context = less hallucination)
   - Fine-tune model on factuality
   - Add explicit "do not fabricate" system prompt instructions
   - Implement grounding checks

5. PREVENT:
   - Regular factuality evaluation of model output
   - Retrieval quality monitoring
   - User feedback loop for incorrect answers
   - Hallucination rate tracking as a quality metric
```

### 8. Embedding Pipeline Failure

**Description**: New documents are not being embedded and indexed, causing stale retrieval results.

**Detection**:
- Embedding queue not processing
- Vector DB index not updating
- Retrieval returning outdated results
- Embedding service errors

**Severity**: SEV-3 (if brief), SEV-2 (if prolonged, causing stale knowledge)

**Playbook:**

```
1. CONFIRM: Is the embedding pipeline broken?
   - Check embedding queue depth and processing rate
   - Check embedding service health
   - Check vector DB indexing status
   - Check for recent document updates in the index

2. IDENTIFY CAUSE:
   - Embedding API unavailable or rate-limited?
   - Embedding pipeline code bug?
   - Vector DB write errors?
   - Queue consumer crashed?

3. MITIGATE:
   - Restart the embedding pipeline
   - Manually process backlog of documents
   - If embedding API down: switch to backup provider

4. RESTORE:
   - Verify documents are being embedded and indexed
   - Verify retrieval includes newly indexed documents
   - Monitor pipeline throughput

5. PREVENT:
   - Embedding pipeline health monitoring
   - Alert on queue backlog
   - Regular index freshness checks
   - Embedding pipeline redundancy
```

### 9. System Prompt Drift

**Description**: The system prompt used by the LLM changes unintentionally over time.

**Detection**:
- System prompt hash change without approval
- Behavioral change in LLM output
- Configuration management alert

**Severity**: SEV-2 (if behavioral change is significant)

**Playbook:**

```
1. CONFIRM: Did the system prompt actually change?
   - Compare current prompt to approved version
   - Check configuration change history
   - Check deployment history

2. ASSESS IMPACT:
   - What changed in the prompt?
   - How does it affect LLM behavior?
   - Are there safety or compliance implications?

3. MITIGATE:
   - Revert to approved system prompt version
   - Deploy the fix

4. INVESTIGATE:
   - How did the change happen?
   - Was it intentional (undocumented) or accidental?
   - Is there a process gap?

5. PREVENT:
   - System prompt versioning and approval workflow
   - Immutable system prompt storage
   - Hash verification at runtime
   - Change management for prompt modifications
```

### 10. LLM Provider Outage

**Description**: The external LLM API provider (Azure OpenAI, Anthropic) experiences an outage.

**Detection**:
- LLM API returning errors or timeouts
- Provider status page reports outage
- Multiple services affected simultaneously

**Severity**: SEV-1 (if no fallback), SEV-2 (if fallback available but degraded)

**Playbook:**

```
1. CONFIRM: Is the LLM provider down?
   - Check provider status page
   - Test API connectivity
   - Check if other teams/services are affected

2. ACTIVATE FALLBACK:
   - Switch to backup LLM provider
   - If no backup: activate degraded mode
   - Communicate service impact to customers

3. ASSESS IMPACT:
   - What services are affected?
   - What is the fallback capability?
   - What is the expected provider recovery time?

4. MONITOR PROVIDER RECOVERY:
   - Track provider status updates
   - Test API periodically
   - Switch back to primary when stable

5. PREVENT:
   - Multi-provider abstraction layer
   - Automatic failover between providers
   - Local model fallback for critical services
   - Regular provider failover testing
```

---

## GenAI Incident Severity Quick Reference

| Incident Type | SEV-1 | SEV-2 | SEV-3 |
|--------------|-------|-------|-------|
| Prompt injection | Successful exploitation | Detected and blocked | Pattern detected, no impact |
| Model degradation | Wrong financial advice | 10%+ accuracy drop | 5% accuracy drop |
| Vector DB outage | Complete outage | Degraded performance | Intermittent latency |
| Token cost explosion | > 400% over budget | 200-400% over budget | 100-200% over budget |
| PII leakage | Cross-customer exposure | Same-customer PII in output | PII detected, blocked |
| GPU exhaustion | Complete serving outage | Partial degradation | Intermittent latency |
| Hallucination | Wrong compliance advice | Wrong factual information | Minor inaccuracy |
| Embedding failure | > 24 hours stale index | > 4 hours stale index | Queue backlog |
| System prompt drift | Safety/compliance impact | Behavioral change | Minor wording change |
| LLM provider outage | No fallback available | Fallback degraded | Brief timeout |

---

## GenAI Incident Response Team

### Specialized Roles

In addition to standard incident management roles, GenAI incidents may require:

| Role | Responsibility | When Needed |
|------|---------------|-------------|
| **ML Engineer** | Model quality assessment, rollback, retraining | Model degradation, hallucination |
| **Vector DB Specialist** | Vector DB troubleshooting, recovery | Vector DB outage, retrieval issues |
| **Prompt Engineer** | Prompt analysis, injection investigation | Prompt injection, system prompt drift |
| **Data Protection Officer** | PII assessment, regulatory notification | PII leakage, data exposure |
| **GPU Specialist** | GPU resource debugging, MIG configuration | GPU exhaustion |
| **FinOps Analyst** | Token cost analysis, cost controls | Token cost explosion |

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-classification.md](incident-classification.md) -- Severity classification
- [triage-playbooks.md](triage-playbooks.md) -- Triage patterns
- [mitigation-strategies.md](mitigation-strategies.md) -- Mitigation strategies
- [regulatory-notification.md](regulatory-notification.md) -- Regulatory obligations
- [on-call-best-practices.md](on-call-best-practices.md) -- On-call requirements
- [game-days.md](game-days.md) -- Game day scenarios
- [banking-incident-requirements.md](banking-incident-requirements.md) -- Banking regulatory obligations
