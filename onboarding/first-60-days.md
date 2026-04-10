# First 60 Days: Deepening and Contributing

> **Audience:** Senior Software Engineer, GenAI Platform Team (Days 31-60)
> **Purpose:** Transition from learning to contributing — production work, on-call, ownership
> **Prerequisites:** Completed `first-30-days.md` milestones

---

## Overview

Days 31-60 mark the transition from "new engineer learning the ropes" to "contributing team member who can be trusted with production systems." This period is characterized by:

1. **Deeper codebase knowledge** — understanding the why behind architectural decisions
2. **Production responsibility** — your first on-call shift, incident response participation
3. **Meaningful contributions** — features that matter, not just documentation updates
4. **Relationship deepening** — becoming a go-to person for at least one area

In a banking environment, this is also when your mistakes become more costly. You know enough to be dangerous but not enough to anticipate every edge case. The guardrails that exist are there to protect you as much as they protect the bank.

---

## Week 5-6: Deepening Technical Knowledge

### Understanding the Guardrails Engine

The guardrails service is the most critical component of our platform. It is the last line of defense between an AI model producing harmful output and a bank consumer receiving it.

**Architecture of the Guardrails Engine:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     Guardrails Engine                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Pre-flight  │  │   In-flight  │  │     Post-flight       │  │
│  │   Checks      │  │   Checks     │  │     Checks            │  │
│  │              │  │              │  │                       │  │
│  │ - Injection   │  │ - Rate       │  │ - Hallucination       │  │
│  │   Detection   │  │   Limiting   │  │   Detection           │  │
│  │ - PII Scan    │  │ - Timeout    │  │ - Factual Accuracy    │  │
│  │ - Toxicity    │  │   Monitoring │  │ - Response            │  │
│  │   Analysis    │  │              │  │   Completeness        │  │
│  │ - Policy      │  │              │  │ - Compliance          │  │
│  │   Compliance  │  │              │  │   Filtering           │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                      │              │
│         └─────────────────┼──────────────────────┘              │
│                           │                                     │
│                  ┌────────▼────────┐                            │
│                  │   Decision      │                            │
│                  │   Engine        │                            │
│                  │                 │                            │
│                  │ ALLOW / DENY /  │                            │
│                  │ REDACT / WARN   │                            │
│                  └────────┬────────┘                            │
│                           │                                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                    ┌───────▼───────┐
                    │    Audit      │
                    │    Logger     │
                    │               │
                    │ All decisions │
                    │ logged to     │
                    │ Splunk +      │
                    │ compliance DB │
                    └───────────────┘
```

**Code Walkthrough: The Guardrails Pipeline**

```python
# genai_platform/guardrails/pipeline.py
"""
The guardrails pipeline executes checks in a defined order.
Each check can return one of four actions:

ALLOW: Pass through without modification
DENY: Block the request/response entirely
REDACT: Remove specific content and continue
WARN: Allow but flag for human review

The pipeline uses the Chain of Responsibility pattern.
Each handler can short-circuit the chain.
"""

from dataclasses import dataclass
from enum import Enum
from typing import List, Optional
import structlog

logger = structlog.get_logger()

class GuardAction(Enum):
    ALLOW = "allow"
    DENY = "deny"
    REDACT = "redact"
    WARN = "warn"

@dataclass
class GuardResult:
    action: GuardAction
    confidence: float
    reason: str
    details: dict
    redacted_content: Optional[str] = None

class GuardrailsPipeline:
    """
    Executes guardrail checks in order.
    
    Order matters because:
    1. Cheap checks run first (fail fast)
    2. Expensive ML model calls run later
    3. Compliance checks always run last (require full context)
    
    IMPORTANT: Never reorder checks without Security team approval.
    The current order is approved by the CISO office (ref: SEC-2024-0156).
    """
    
    def __init__(self, checks: List[GuardrailCheck]):
        self.checks = checks
        self.audit_logger = AuditLogger()
    
    async def execute(self, content: str, context: dict) -> GuardResult:
        """
        Execute all checks in order.
        
        If any check returns DENY, the pipeline short-circuits immediately.
        REDACT results modify the content for subsequent checks.
        WARN results are collected but do not block.
        
        Args:
            content: The prompt or response to check
            context: Metadata about the request (consumer, tier, etc.)
        
        Returns:
            The most restrictive result from all checks
        """
        results: List[GuardResult] = []
        current_content = content
        
        for check in self.checks:
            result = await check.evaluate(current_content, context)
            results.append(result)
            
            # Log every check execution
            logger.info(
                "guardrail_check_executed",
                check_name=check.name,
                action=result.action.value,
                confidence=result.confidence,
                request_id=context.get("request_id"),
            )
            
            # Short-circuit on DENY
            if result.action == GuardAction.DENY:
                self.audit_logger.log_decision(
                    content=content,
                    results=results,
                    final_action=GuardAction.DENY,
                    context=context,
                )
                return result
            
            # Apply redaction to content for subsequent checks
            if result.action == GuardAction.REDACT:
                current_content = result.redacted_content or current_content
        
        # Determine final action: most restrictive wins
        final_action = self._resolve_action(results)
        
        self.audit_logger.log_decision(
            content=content,
            results=results,
            final_action=final_action,
            context=context,
        )
        
        return GuardResult(
            action=final_action,
            confidence=max(r.confidence for r in results),
            reason=self._summarize_results(results),
            details={"all_results": [r.__dict__ for r in results]},
        )
    
    def _resolve_action(self, results: List[GuardResult]) -> GuardAction:
        """
        Resolve the most restrictive action from all results.
        
        Priority: DENY > REDACT > WARN > ALLOW
        """
        action_priority = {
            GuardAction.DENY: 4,
            GuardAction.REDACT: 3,
            GuardAction.WARN: 2,
            GuardAction.ALLOW: 1,
        }
        
        most_restrictive = max(results, key=lambda r: action_priority[r.action])
        return most_restrictive.action
```

**Exercise: Add a New Guardrail Check**

Your task: Implement a `FinancialAdviceDisclaimer` check that detects when the model provides specific financial advice (e.g., "you should invest in X") and automatically appends a disclaimer.

```python
# genai_platform/guardrails/checks/financial_advice.py
"""
Financial Advice Disclaimer Check

Detects when the model output contains specific financial advice
and appends a regulatory disclaimer.

Regulatory requirement: FCA Handbook COBS 4.4.1
All AI-generated financial content must include:
"This information is generated by AI and does not constitute 
financial advice. Please consult a qualified financial advisor."

Approved by: Compliance team (ref: COMP-2024-0892)
"""

import re
from typing import List
from ..pipeline import GuardrailCheck, GuardResult, GuardAction

FINANCIAL_ADVICE_PATTERNS = [
    r"(?i)you\s+(should|ought to|need to)\s+(invest|buy|sell|hold)",
    r"(?i)(I\s+)?recommend\s+(investing|buying|selling)",
    r"(?i)this\s+is\s+a\s+(good|bad)\s+(investment|time to|opportunity)",
    r"(?i)your\s+(best|optimal)\s+(strategy|approach|option)\s+is",
    r"(?i)\bguaranteed\s+(returns?|profit|income)",
]

DISCLAIMER = (
    "\n\n---\n"
    "*This information is generated by artificial intelligence and does not "
    "constitute financial advice. Past performance is not indicative of future "
    "results. Please consult a qualified financial advisor before making "
    "investment decisions.*"
)

class FinancialAdviceDisclaimerCheck(GuardrailCheck):
    name = "financial_advice_disclaimer"
    
    async def evaluate(self, content: str, context: dict) -> GuardResult:
        # Check if any financial advice patterns are present
        detected_patterns: List[str] = []
        
        for pattern in FINANCIAL_ADVICE_PATTERNS:
            if re.search(pattern, content):
                detected_patterns.append(pattern)
        
        if not detected_patterns:
            return GuardResult(
                action=GuardAction.ALLOW,
                confidence=0.0,
                reason="No financial advice patterns detected",
                details={},
            )
        
        # Append disclaimer
        redacted_content = content + DISCLAIMER
        
        return GuardResult(
            action=GuardAction.REDACT,
            confidence=len(detected_patterns) / len(FINANCIAL_ADVICE_PATTERNS),
            reason=f"Financial advice patterns detected: {len(detected_patterns)}",
            details={
                "patterns_detected": detected_patterns,
                "disclaimer_appended": True,
            },
            redacted_content=redacted_content,
        )
```

### Understanding the RAG Pipeline

Our platform uses Retrieval-Augmented Generation (RAG) for most consumer use cases. Understanding how retrieval works is critical because retrieval quality directly impacts output quality.

**The RAG Pipeline in Detail:**

```
User Query: "What are the current mortgage rates for first-time buyers?"

1. QUERY PROCESSING
   - Normalize query (lowercase, remove punctuation)
   - Detect intent (information_query)
   - Extract entities:
     - product_type: mortgage
     - customer_segment: first_time_buyer
     - region: UK (from consumer context)
   
2. EMBEDDING GENERATION
   - Convert query to vector using embedding model
   - Model: bank-internal/multilingual-embedding-v3 (768 dimensions)
   - Latency budget: 50ms
   
3. VECTOR DATABASE QUERY
   - Search Pinecone/Milvus index: banking-products-uk-2024
   - Top-K: 10 documents
   - Similarity threshold: 0.75 cosine similarity
   - Filter by metadata:
     - content_type: product_specification
     - status: active
     - effective_date <= today
     - expiry_date > today (CRITICAL: never return expired content)
   
4. RERANKING
   - Cross-encoder reranker scores the top 10 results
   - Model: bank-internal/cross-encoder-reranker-v2
   - Select top 3 for context window
   - Latency budget: 30ms
   
5. CONTEXT ASSEMBLY
   - Build prompt template:
     ```
     You are a banking product assistant for a UK retail bank.
     
     Relevant context:
     {retrieved_documents}
     
     Current date: {current_date}
     Customer segment: first_time_buyer
     Region: UK
     
     Question: {original_query}
     
     Guidelines:
     - Only use information from the provided context
     - If the answer is not in the context, say so clearly
     - Include the effective date of any rates mentioned
     - Add disclaimer that rates are subject to change
     ```
   
6. LLM GENERATION
   - Model: Azure OpenAI GPT-4o (deployment: banking-assistant-prod)
   - Temperature: 0.3 (low temperature for factual responses)
   - Max tokens: 500
   - Stop sequences: ["\n\n---", "Disclaimer:"]
   
7. RESPONSE VALIDATION
   - Guardrails check (as described above)
   - Rate extraction verification (does the rate match the source?)
   - Date validation (is the rate still effective?)
```

**Common RAG Failures and How to Debug Them:**

| Symptom | Likely Cause | Debug Approach |
|---------|-------------|----------------|
| Model says "I do not have that information" | No documents retrieved above threshold | Check vector DB query, verify documents are indexed |
| Model provides outdated rates | Stale documents in index | Check document `expiry_date` field, verify update pipeline |
| Model hallucinates a product | Retrieved documents are not relevant enough | Lower similarity threshold, improve reranker |
| Response is inconsistent with source | Prompt template does not enforce grounding | Add stronger grounding instructions to template |
| High latency in retrieval | Embedding model or vector DB is slow | Check model deployment health, index fragmentation |

**Debug Exercise:**

```bash
# Query the RAG pipeline with debug output
curl -X POST https://dev-api-gateway.platform.bank/internal/debug/rag-pipeline \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the current ISA transfer rates?",
    "consumer": "wealth-management-chat",
    "debug": true
  }'

# Response includes:
# - Retrieved documents with similarity scores
# - Reranking scores
# - Final context sent to LLM
# - Generated response with source citations
```

---

## Week 7: First On-Call Shift

### Preparation

Before your first on-call shift, ensure you have completed:

- [ ] Read all runbooks in `incident-management/runbooks/`
- [ ] Completed the on-call training course (internal LMS: "GENAI-ONCALL-101")
- [ ] Shadowed at least 2 full on-call shifts
- [ ] Set up alerting on your phone (PagerDuty/OpsGenie app installed)
- [ ] Know the escalation matrix (who to call when you are stuck)
- [ ] Have access to all production systems (Grafana, Splunk, ArgoCD)
- [ ] Reviewed the last 30 days of incidents and their resolutions

**Your First Shift Setup:**

```
On-Call Kit Checklist:
├── Hardware
│   ├── Company phone (charged, PagerDuty app installed)
│   ├── Laptop (charged, VPN configured)
│   └── Backup phone (personal phone with alerting configured)
├── Access
│   ├── Grafana dashboards bookmarked
│   ├── Splunk queries saved
│   ├── ArgoCD access verified
│   └── Slack on-call channels joined
├── Runbooks
│   ├── High error rate → runbook-error-rate.md
│   ├── High latency → runbook-latency.md
│   ├── LLM provider outage → runbook-provider-outage.md
│   ├── Guardrails failure → runbook-guardrails-failure.md
│   └── PII leak → runbook-PII-leak.md (CRITICAL)
└── Contacts
    ├── Team lead (first escalation, 15 min)
    ├── Engineering manager (second escalation, 30 min)
    ├── SRE lead (infrastructure issues)
    ├── Security team (compliance incidents)
    └── Vendor support (AI provider issues)
```

### Realistic On-Call Scenario

**03:14 AM — Alert Fires**

```
Alert: [CRITICAL] GuardrailsInjectionDetectionRateDrop
Severity: P1
Description: Prompt injection detection rate has dropped to 78% (threshold: 95%)
             over the last 15 minutes.
             Affected consumers: all
             Runbook: runbook-guardrails-detection-rate-drop.md
```

**Your Response (following the runbook):**

```
03:14 — Acknowledge alert, join the incident bridge
03:15 — Check Grafana dashboard for guardrails metrics
        - Detection rate: 78% (dropped from 96% at 02:45)
        - False positive rate: 2% (normal)
        - Request volume: steady (no traffic spike)
        - Model latency: normal
03:17 — Check recent deployments
        - Deployment PROD-2024-1204 at 02:30 (guardrails model update)
        - Model version changed from injection-scanner-v2.3 to v2.4
03:19 — Hypothesis: New model version has regression
        - Check model evaluation report for v2.4
        - Report shows: 94.2% accuracy on test set (below 95% threshold)
        - Red flag: Model was deployed despite failing accuracy gate
03:21 — Decision: Roll back to v2.3
        - Execute rollback via ArgoCD
        - kubectl rollout undo deployment/genai-platform-guardrails -n prod
03:25 — Rollback complete
03:27 — Verify detection rate returning to normal
        - Detection rate: 94% and climbing
03:30 — Detection rate at 95.5%, incident resolved
03:35 — Post-incident actions:
        - Create incident ticket INC-2024-1205
        - Block model v2.4 from redeployment
        - Add pre-deployment accuracy gate enforcement
        - Notify Security team
        - Schedule post-mortem for next business day
```

**Post-Mortem Excerpt:**

```
INCIDENT POST-MORTEM: INC-2024-1205
Guardrails Model Regression — Injection Detection Rate Drop

Root Cause:
The model deployment pipeline allowed v2.4 to deploy despite accuracy 
falling below the 95% threshold. The threshold was configured as a 
WARNING rather than a BLOCKING gate.

Contributing Factors:
1. The accuracy threshold was changed from BLOCKING to WARNING in 
   PR-8842 (approved by contractor who did not understand the policy)
2. The deployment happened at 02:30 AM — no human reviewed the warning
3. Automated deployment does not currently enforce accuracy gates

Remediation:
1. [IMMEDIATE] Revert accuracy gate to BLOCKING (completed)
2. [SHORT-TERM] Add deployment freeze if any guardrails metric is 
   below threshold (target: Sprint 24)
3. [LONG-TERM] Implement canary analysis for guardrails model updates 
   before full rollout (target: Sprint 26)
4. [PROCESS] Require Security team sign-off for any guardrails 
   threshold changes (policy update submitted to CAB)
```

### On-Call Survival Tips

**What Experienced On-Call Engineers Do:**

1. **Triage, do not investigate.** Your first job is to restore service, not find root cause. Roll back if a recent deployment is suspicious.
2. **Use the runbook, do not improvise.** Runbooks are tested procedures. Deviating from them during an incident is how things get worse.
3. **Communicate early, communicate often.** Post in the incident channel every 10 minutes, even if the update is "still investigating."
4. **Escalate before you need to.** If you are stuck for 10 minutes, escalate. It is better to wake someone up than to lose 30 minutes.
5. **Document everything.** Your incident timeline becomes the audit trail. Write it as if a regulator will read it (because they might).

**What New On-Call Engineers Do Wrong:**

1. **Try to fix the problem instead of restoring service.** Root cause analysis happens after the incident, not during.
2. **Wait too long to escalate.** The 15-minute escalation rule exists for a reason. Use it.
3. **Assume the runbook is wrong.** The runbook is probably right. Your intuition is probably wrong (at first).
4. **Forget to communicate.** Silence during an incident causes more panic than the incident itself.
5. **Skip documentation.** The post-mortem will be painful without good notes.

---

## Week 8: Production Incident Response

### Participating in Your First Live Incident

You will eventually join a live incident as a responder. Your role during the incident:

1. **Scribe** — Document the timeline, decisions, and actions taken
2. **Investigator** — Follow the runbook and report findings
3. **Communicator** — Update stakeholders on progress

**Incident Response Framework:**

```
INCIDENT LIFECYCLE

1. DETECTION
   - Automated alert fires
   - On-call acknowledges within SLA (P1: 5 min, P2: 15 min, P3: 30 min)
   
2. TRIAGE
   - Assess severity (P1-P4)
   - Identify affected services and consumers
   - Determine if immediate action is needed (rollback, traffic shift)
   
3. MITIGATION
   - Execute runbook steps
   - If no runbook exists, follow general troubleshooting:
     a. Check recent deployments
     b. Check infrastructure health
     c. Check downstream dependencies
     d. Roll back if uncertain
   
4. RESOLUTION
   - Service restored
   - Verify all metrics are back to normal
   - Monitor for 30 minutes before declaring resolved
   
5. POST-INCIDENT
   - Create incident ticket within 24 hours
   - Schedule post-mortem within 48 hours (P1/P2) or 1 week (P3/P4)
   - Complete post-mortem document within 5 business days
   - Track remediation actions to completion
```

**Real Incident Example: Token Rate Limit Exhaustion**

```
INCIDENT: INC-2024-1189
SEVERITY: P2
DURATION: 47 minutes
IMPACT: 3 consumers experienced degraded response times

TIMELINE:
14:15 — Alert: TokenBudgetExceeded (80% threshold reached for Azure OpenAI)
14:16 — On-call acknowledges, begins investigation
14:20 — Investigation reveals:
        - One consumer (investment-research-bot) is using 60% of total tokens
        - This consumer had a 3x traffic spike due to a marketing campaign
        - Campaign was not communicated to the platform team
14:30 — Action: Apply emergency rate limit to investment-research-bot
        - Reduce from 200 req/min to 50 req/min
        - This protects other consumers but degrades the campaign experience
14:35 — Trade-off accepted: Better to degrade one consumer than all consumers
14:40 — Rate limit applied, other consumers returning to normal
15:00 — Token usage stabilizing, incident resolved
15:05 — Post-incident: Contact marketing team about campaign coordination
         Implement per-consumer token budgets to prevent future cross-consumer impact

LESSON:
Marketing campaigns cause traffic spikes. The platform team must be notified
before campaigns launch. This led to a new process: Marketing must file a
Capacity Planning Request (CPR) before any campaign that affects GenAI consumers.
```

---

## End of Month 2: Self-Assessment

### Technical Check

**Can you confidently:**

- [ ] Explain the guardrails pipeline and add a new check?
- [ ] Debug a RAG pipeline failure (retrieval, reranking, generation)?
- [ ] Roll back a deployment in production?
- [ ] Read and interpret Grafana dashboards for all platform services?
- [ ] Write Splunk queries to investigate an incident?
- [ ] Configure a new consumer on the API gateway?
- [ ] Deploy a change through the full CI/CD pipeline?
- [ ] Respond to the top 5 most common alerts without consulting the runbook?

### Process Check

**Have you:**

- [ ] Completed your first on-call shift (with backup)?
- [ ] Participated in at least one live incident?
- [ ] Written at least one post-mortem (or contributed to one)?
- [ ] Attended a CAB meeting (to understand the change approval process)?
- [ ] Filed a Capacity Planning Request (or observed one)?
- [ ] Reviewed PRs from other team members (not just had yours reviewed)?

### Trust Check

**Signs you are earning trust:**

- Other engineers are asking you to review their PRs
- Your PR reviews are cited as "helpful" or "thorough"
- You are being included in design discussions, not just implementation
- The on-call engineer escalates to you for certain types of issues
- Your opinions are sought in sprint planning and estimation
- Someone newer than you asks you for help

### Output Check

**What you should have produced by Day 60:**

- [ ] Minimum 15 merged PRs (mix of features, fixes, tests, docs)
- [ ] At least 1 PR that changed guardrails code (reviewed by Security)
- [ ] At least 1 production deployment you led end-to-end
- [ ] At least 1 runbook you updated or created
- [ ] At least 1 post-mortem you contributed to
- [ ] At least 3 PR reviews that received positive feedback

---

## What Comes Next

In the next 30 days (see `first-90-days.md`), you will:

- Own a component or service end-to-end
- Mentor a newer team member or intern
- Lead a project from design to deployment
- Be interview-ready to discuss your contributions
- Take on-call shifts without backup

**Remember:** By Day 60, you should be dangerous but not reckless. You know how things work well enough to change them, and you respect the guardrails enough not to bypass them.

---

*Last updated: April 2025 | Document owner: Engineering Enablement Team | Review cycle: Quarterly*
