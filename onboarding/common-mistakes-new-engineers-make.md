# Common Mistakes New Engineers Make

> **Audience:** Engineers joining the GenAI Platform Team (Days 1-90)
> **Purpose:** Top 15 mistakes, real stories, and how to avoid them
> **Prerequisites:** Read this in Week 1. Revisit before each milestone (30/60/90 days).

---

## Why This Document Exists

Every mistake listed here actually happened. Every one had real consequences — from minor embarrassment to compliance incidents. They are documented not to shame anyone, but to ensure the pattern does not repeat.

**If you only remember one thing from this document:** When in doubt, ask. The cost of asking a "dumb" question is 5 minutes. The cost of a wrong assumption is measured in incidents.

---

## Mistake #1: Deploying Without Reading the Runbook

**What happened:** A new engineer deployed a change to production and was not sure how to verify it worked. They checked the Grafana dashboard but did not know which metrics to look at. The deployment had introduced a subtle bug in the guardrails pipeline that was not visible on the standard dashboard. The bug was discovered 6 hours later when a consumer reported that prompt injection detection was not working.

**Impact:** 6 hours of degraded guardrails protection. Incident ticket filed. Post-mortem required.

**How to avoid it:**

```
BEFORE DEPLOYING TO PRODUCTION:
1. Read the deployment runbook for the service you are deploying
2. Identify the key metrics to monitor post-deployment
3. Know the rollback procedure and have it ready
4. Have the runbook open during the deployment
5. If something looks wrong, follow the runbook's troubleshooting section

DEPLOYMENT RUNBOOK CHECKLIST:
[ ] I have read the runbook for this service
[ ] I know which Grafana dashboard to monitor
[ ] I know the normal values for key metrics (error rate, latency, throughput)
[ ] I know the rollback procedure and can execute it in < 5 minutes
[ ] I have the runbook URL open in my browser during deployment
```

**Severity:** HIGH — This can cause extended outages

---

## Mistake #2: Skipping the Full Test Suite

**What happened:** An engineer was in a hurry and ran only the unit tests before deploying. The integration tests would have caught a bug where the new guardrails check was not properly registered in the pipeline. The bug caused the check to be silently skipped in production.

**Impact:** A guardrails check was non-functional for 2 weeks before it was discovered during a routine audit of guardrails effectiveness.

**How to avoid it:**

```
ALWAYS run the full test suite before merging:
- make test-unit (fast, < 5 minutes)
- make test-integration (slower, < 15 minutes)
- make test-e2e (if available, against staging environment)

If a test is flaky (failing intermittently), file a ticket to fix it.
Do not skip it. Flaky tests are better than no tests.

The CI pipeline runs all tests automatically.
If CI passes, you can be confident the change is safe.
If you skip CI gates, you are flying blind.
```

**Severity:** HIGH — Can introduce undetected bugs

---

## Mistake #3: Hard-Coding Configuration

**What happened:** An engineer needed to add a new AI provider endpoint. Instead of adding it to the configuration service, they hard-coded the URL in the code. When the provider endpoint changed (it does every 90 days as part of certificate rotation), the hardcoded URL stopped working.

**Impact:** 4 hours of LLM routing failures until the hard-coded value was found and fixed.

**How to avoid it:**

```
NEVER hard-code:
- URLs (use the configuration service)
- API keys (use Vault)
- Thresholds (use configuration, even for "temporary" values)
- Feature flags (use LaunchDarkly)
- Environment-specific values (use environment configs)

ALWAYS use:
- ConfigLoader.get("service.key") for configuration
- Vault for secrets
- LaunchDarkly for feature flags
- Environment variables only for local development

The rule: If it changes between environments (dev/staging/prod),
it must be in configuration, not in code.
```

**Severity:** MEDIUM — Causes operational friction

---

## Mistake #4: Not Understanding the Guardrails Pipeline

**What happened:** An engineer added a new guardrails check but placed it at the end of the pipeline (after the expensive LLM call). The check was for prompt injection detection — which should run BEFORE the prompt is sent to the LLM. Because the check ran after the LLM call, malicious prompts were reaching the LLM before being detected and blocked.

**Impact:** Prompt injection protection was ineffective for 1 week. Discovered during a security review.

**How to avoid it:**

```
GUARDRAILS PIPELINE ORDER:

Pre-flight (BEFORE LLM call):
1. Input validation (format, length, encoding)
2. Prompt injection detection (MUST run before LLM sees the prompt)
3. PII detection and redaction
4. Toxicity analysis
5. Policy compliance check

Post-flight (AFTER LLM call):
1. Hallucination detection (requires comparing output to retrieved context)
2. Factual accuracy verification
3. Response completeness check
4. Compliance filtering
5. Financial advice disclaimer (if applicable)

NEVER reorder guardrails checks without Security team approval.
The order is approved by the CISO office and is based on:
- Cost (cheap checks first, fail fast)
- Security (injection detection before LLM exposure)
- Correctness (hallucination detection requires output to analyze)
```

**Severity:** CRITICAL — Security vulnerability

---

## Mistake #5: Not Escalating During an Incident

**What happened:** A new engineer was on their first solo on-call shift. An alert fired at 2 AM for high error rates. They tried to diagnose the problem themselves for 45 minutes before escalating. The issue was a downstream database connection pool exhaustion that the SRE team could have fixed in 5 minutes.

**Impact:** 40 minutes of unnecessary service degradation. Consumers experienced errors.

**How to avoid it:**

```
ESCALATION RULES:

- If you do not know the cause within 10 minutes → ESCALATE
- If the runbook does not cover the situation → ESCALATE
- If the issue involves infrastructure you do not understand → ESCALATE
- If the incident is P1/P2 → ESCALATE IMMEDIATELY (do not diagnose alone)
- If you are the newest engineer on the team → ESCALATE EARLIER, not later

The 15-minute rule: If you have not made progress in 15 minutes, you MUST escalate.
This is not a suggestion. It is a policy.

Escalation is not a failure. It is the correct procedure.
The on-call escalation matrix exists for this reason.
Your backup contact expects to be called. That is their job.
```

**Severity:** HIGH — Extends incident duration unnecessarily

---

## Mistake #6: Committing Secrets or Sensitive Data

**What happened:** An engineer committed a `.env` file containing a development API key to the repository. The pre-commit hook caught it, but the engineer used `--no-verify` to bypass the hook. The key was detected by the automated secrets scanner and rotated, but the commit history still contained the secret.

**Impact:** Key rotation required. All services using that key needed to be updated. Audit finding: "Secrets management process not followed."

**How to avoid it:**

```
NEVER:
- Commit secrets in any form (even "test" or "development" keys)
- Use --no-verify to bypass pre-commit hooks
- Share API keys in Slack, email, or any unsecured channel
- Put secrets in Docker images
- Print secrets in logs (even temporarily)

ALWAYS:
- Use Vault for all secrets
- Use environment variables loaded from Vault at runtime
- Use the bank's secrets management CLI: vault-secret get <path>
- Report any accidental secret commit IMMEDIATELY (do not try to fix it silently)
- Rotate any secret that may have been exposed

If you accidentally commit a secret:
1. Do NOT push the commit
2. Reset the commit: git reset HEAD~1
3. Report it to the Security team
4. Request key rotation
5. The secret must be considered COMPROMISED
```

**Severity:** CRITICAL — Security incident, audit finding

---

## Mistake #7: Not Testing with Realistic Data

**What happened:** An engineer tested the RAG pipeline with hand-crafted test queries that were clean and well-formed. In production, consumers submitted messy, ambiguous queries with typos and mixed languages. The retrieval quality was much worse than testing suggested.

**Impact:** Consumer complaints about poor response quality. Retrieval precision dropped from 92% (tested) to 67% (production).

**How to avoid it:**

```
TEST WITH REALISTIC DATA:
- Use anonymized production data for testing (available in the test data store)
- Include edge cases: empty queries, very long queries, mixed-language queries
- Test with malicious inputs (see tests/fixtures/malicious_prompts.json)
- Test with PII-containing inputs (verify redaction works)
- Test with queries that have no relevant documents (verify graceful handling)

TEST DATA SOURCES:
- tests/fixtures/sample_prompts.json (curated test cases)
- tests/fixtures/malicious_prompts.json (adversarial test cases)
- Internal test data store (anonymized production data, refreshed weekly)
- Consumer-provided test suites (ask consumer teams for their test scenarios)

Before deploying changes to the RAG pipeline:
1. Run the full test suite with realistic data
2. Deploy to staging and run e2e tests
3. Compare staging metrics with production baseline
4. If retrieval quality drops, do not deploy
```

**Severity:** MEDIUM — Quality degradation

---

## Mistake #8: Ignoring Feature Flags

**What happened:** An engineer deployed a new guardrails check behind a feature flag but forgot to actually add the feature flag check in the code. The check was always active. When the check had a bug, it could not be disabled without a code deployment.

**Impact:** 3-day deployment cycle to fix the bug (CAB approval required). During this time, the buggy check was causing false positives.

**How to avoid it:**

```
FEATURE FLAG CHECKLIST:
[ ] Flag created in LaunchDarkly (all environments)
[ ] Flag defaults to OFF (safe default)
[ ] Code checks the flag before executing new behavior
[ ] Flag state is logged when evaluated
[ ] Flag can be turned off without code deployment
[ ] Rollback plan includes flag deactivation
[ ] Flag has an expiration date (prevents stale flags)

CORRECT PATTERN:
flags = FeatureFlagClient()
if flags.is_enabled("new-guardrails-check", context):
    result = await new_guardrails_check.evaluate(content, context)
else:
    result = GuardResult(action=GuardAction.ALLOW, ...)

INCORRECT PATTERN:
# Flag exists in LaunchDarkly but is not checked in code
result = await new_guardrails_check.evaluate(content, context)  # Always runs!
```

**Severity:** MEDIUM — Reduces deployment flexibility

---

## Mistake #9: Not Understanding Data Residency Requirements

**What happened:** An engineer configured a new AI provider without checking data residency requirements. The provider processed requests in the US East region. Some consumer data (UK retail banking) is prohibited from leaving the UK under internal data handling policies.

**Impact:** Compliance violation. All affected requests had to be identified and reported. Data Protection Officer notified. Policy review required.

**How to avoid it:**

```
DATA RESIDENCY RULES:

UK retail banking data:
- Must remain in UK regions (UK South, UK West)
- Cannot be processed outside the UK
- Azure OpenAI UK South is approved
- Azure OpenAI US regions are NOT approved

International banking data:
- Check the data handling policy for each jurisdiction
- Some data must remain in the country of origin
- Some data can be processed in approved regions

Internal data:
- Must remain on-premises (internal LLM)
- Cannot be sent to external providers

BEFORE configuring a new AI provider:
1. Check the data residency requirements for all affected consumers
2. Verify the provider's data processing locations
3. Get Data Protection Officer approval
4. Document the data flow in the provider configuration
5. Add data residency checks to the LLM router
```

**Severity:** CRITICAL — Regulatory violation

---

## Mistake #10: Assuming the Runbook Is Always Right

**What happened:** An engineer followed a runbook exactly during an incident. The runbook was 18 months old and referenced a service that had been decommissioned. Following the runbook wasted 20 minutes during a P1 incident.

**Impact:** Incident duration extended by 20 minutes. Runbook trust was damaged (engineers started ignoring all runbooks).

**How to avoid it:**

```
RUNBOOK REALITY:

Runbooks are living documents. They can become outdated.
When using a runbook:
1. Check the "Last Updated" date. If > 6 months old, be skeptical.
2. Verify commands and URLs before running them.
3. If something in the runbook does not match reality, note it.

After an incident:
1. Review every runbook you used.
2. Update any outdated information.
3. File tickets for runbooks you could not verify.
4. If a runbook is completely wrong, mark it as DEPRECATED.

When you notice a runbook issue (even outside incidents):
1. Fix it if you can (it takes 5 minutes)
2. File a ticket if you cannot fix it now
3. Tell the team ("Hey, the runbook for X is outdated")

Runbooks are everyone's responsibility. The team that maintains good runbooks
responds to incidents faster and with less stress.
```

**Severity:** MEDIUM — Wastes time during incidents

---

## Mistake #11: Not Considering Cost Impact

**What happened:** An engineer improved the RAG retrieval quality by increasing the top-K parameter from 10 to 50. This improved retrieval quality by 8% but increased embedding costs by 5x (5x more documents embedded per query). The monthly cost increased by $15,000.

**Impact:** Budget overrun. FinOps escalation. Change had to be reverted. Engineer was embarrassed.

**How to avoid it:**

```
COST IMPACT CHECKLIST:

Before deploying changes that affect AI provider usage:
[ ] Estimate the token/cost impact of the change
[ ] Check current token usage per consumer
[ ] Model the cost impact (new_usage = old_usage * multiplier)
[ ] Compare against the consumer's token budget
[ ] Compare against the platform's total budget
[ ] Discuss with FinOps if the impact is significant (> 5%)

COST MONITORING:
- Real-time cost dashboard: Grafana > GenAI Platform > Cost Tracking
- Daily cost reports sent to engineering team
- Monthly cost review with FinOps

If you are optimizing for quality:
- Measure the quality improvement
- Measure the cost increase
- Calculate the cost-per-quality-point improvement
- Decide if the trade-off is worth it

Example:
- Improvement: +8% retrieval precision
- Cost increase: +$15,000/month
- Cost per quality point: $15,000 / 8 = $1,875 per percentage point
- Is this worth it? Discuss with Product Owner and FinOps.
```

**Severity:** MEDIUM — Financial impact

---

## Mistake #12: Not Writing Structured Logs

**What happened:** An engineer added logging to help debug a new feature but used string formatting: `logger.info(f"Processing request for user {user_id}")`. When an incident occurred, the team could not query the logs effectively because the data was unstructured.

**Impact:** Incident investigation took 2x longer because logs could not be filtered or aggregated.

**How to avoid it:**

```
ALWAYS use structured logging:

BAD:
logger.info(f"Request {request_id} processed in {duration}ms for {consumer_id}")
# Cannot query: "show me all requests for consumer X"
# Cannot query: "show me all requests with duration > 1000ms"

GOOD:
logger.info(
    "request_processed",
    request_id=request_id,
    duration_ms=duration,
    consumer_id=consumer_id,
)
# Can query: consumer_id="wealth-chat" | stats avg(duration_ms)
# Can query: duration_ms > 1000 | table request_id, consumer_id

SPLUNK QUERY EXAMPLES:
# All requests for a specific consumer
index=genai_platform consumer_id="wealth-chat" | stats count by action

# Slow requests
index=genai_platform duration_ms > 1000 | table request_id, consumer_id, duration_ms

# Error rate by consumer
index=genai_platform action="error" | stats count by consumer_id

# Guardrails decisions
index=genai_platform source="guardrails" | stats count by action, check_name
```

**Severity:** LOW-MEDIUM — Impairs debugging

---

## Mistake #13: Not Communicating Deployment Plans

**What happened:** An engineer deployed a change that modified the API response format. They did not notify any consumers. The wealth management chat team discovered the change when their frontend broke in production.

**Impact:** 2 hours of broken chat experience. Relationship with consumer team damaged. CAB review required for process violation.

**How to avoid it:**

```
DEPLOYMENT COMMUNICATION CHECKLIST:

For changes that affect consumers:
[ ] Consumer teams notified at least 1 week before deployment
[ ] Change described in consumer-facing changelog
[ ] API versioning considered (breaking change = new version)
[ ] Backward compatibility maintained during transition period
[ ] Consumer teams given time to update their integration
[ ] Deployment scheduled at a time agreed with consumers
[ ] Consumer teams notified when deployment is complete

BREAKING CHANGE POLICY:
- Breaking changes require a new API version
- Old version must be supported for at least 3 months
- Consumers must be notified 3 months before deprecation
- Deprecation warnings in API responses
- Consumer migration support provided

Non-breaking changes (new fields, new optional parameters) can be deployed
without consumer notification but should still be in the changelog.
```

**Severity:** HIGH — Consumer impact, relationship damage

---

## Mistake #14: Not Understanding the Incident Severity Matrix

**What happened:** A new engineer classified a P1 incident (guardrails completely non-functional) as P3 because "the platform is still responding to requests." The incident was not escalated for 2 hours.

**Impact:** 2 hours of unprotected LLM access. Compliance violation. Regulatory notification may be required.

**How to avoid it:**

```
INCIDENT SEVERITY MATRIX:

P1 (Critical) — Immediate action required:
- Guardrails completely non-functional
- PII exposure or data breach
- Complete platform outage
- Security vulnerability being actively exploited
Response: Page entire team, immediate escalation to leadership

P2 (High) — Urgent action required:
- Guardrails degraded (detection rate < 80%)
- One or more consumers unable to use platform
- LLM provider outage affecting consumers
- Significant cost anomaly (> 3x normal usage)
Response: Page on-call, escalate within 15 minutes

P3 (Medium) — Action required during business hours:
- Guardrails false positive rate increased
- Platform performance degraded (P99 > 2x normal)
- One consumer experiencing intermittent issues
Response: On-call investigates, escalate if P1/P2 criteria met

P4 (Low) — Address in normal workflow:
- Minor bugs with workarounds
- Documentation errors
- Non-critical feature requests
Response: File ticket, prioritize in sprint planning

WHEN IN DOUBT, CLASSIFY HIGHER.
It is always better to over-escalate and stand down than to under-escalate.
```

**Severity:** CRITICAL — Affects incident response

---

## Mistake #15: Trying to Be a Hero

**What happened:** An engineer noticed a complex bug in the RAG pipeline and decided to fix it alone rather than asking for help. They spent 3 days trying to understand the issue. When they finally asked for help, a senior engineer identified the root cause in 30 minutes (it was a known issue with a documented workaround).

**Impact:** 3 days of wasted effort. The bug affected consumers for 3 additional days because no one else knew about it.

**How to avoid it:**

```
THE HERO ANTI-PATTERN:

Symptoms:
- Struggling with a problem for > 4 hours without progress
- Not telling anyone about the problem
- Assuming you should be able to figure it out alone
- Feeling embarrassed to ask for help

The reality:
- Senior engineers ask for help regularly
- The person who wrote the code knows things you do not
- The runbook exists for a reason — use it
- Asking for help after 30 minutes of being stuck is the right move

The 30-minute rule:
If you have been stuck on the same problem for 30 minutes:
1. Write down what you have tried
2. Write down what you think the problem is
3. Ask someone for help (Slack, DM, or walk over)
4. Include your notes so they can help efficiently

What to say:
"Hey, I am working on [task] and I am stuck on [specific problem].
I have tried [things I tried]. I think the issue might be [hypothesis].
Can you take a look or point me in the right direction?"

What NOT to say:
"Nothing works. I give up." (No context, no effort shown)
"I have been working on this for 3 days." (Should have asked 2.5 days ago)
```

**Severity:** MEDIUM — Wastes time, delays fixes

---

## Summary: The Mistake Severity Table

| # | Mistake | Severity | Category |
|---|---------|----------|----------|
| 1 | Deploying without reading the runbook | HIGH | Operational |
| 2 | Skipping the full test suite | HIGH | Quality |
| 3 | Hard-coding configuration | MEDIUM | Code Quality |
| 4 | Not understanding guardrails pipeline order | CRITICAL | Security |
| 5 | Not escalating during an incident | HIGH | Operational |
| 6 | Committing secrets | CRITICAL | Security |
| 7 | Not testing with realistic data | MEDIUM | Quality |
| 8 | Ignoring feature flags | MEDIUM | Operational |
| 9 | Not understanding data residency | CRITICAL | Compliance |
| 10 | Assuming runbooks are always right | MEDIUM | Operational |
| 11 | Not considering cost impact | MEDIUM | Financial |
| 12 | Not writing structured logs | LOW-MEDIUM | Observability |
| 13 | Not communicating deployment plans | HIGH | Consumer Relations |
| 14 | Not understanding incident severity | CRITICAL | Incident Response |
| 15 | Trying to be a hero | MEDIUM | Team Dynamics |

---

## The Underlying Pattern

Every mistake on this list shares a common root cause:

**Assumption without verification.**

- Assuming the runbook is current → Verify it
- Assuming the tests pass → Run them
- Assuming the cost impact is small → Calculate it
- Assuming the consumer knows about the change → Tell them
- Assuming you can figure it out alone → Ask for help
- Assuming the configuration is correct → Check it
- Assuming the incident is not severe → Check the matrix

**The habit that prevents all of these:** Verify before you act. Ask before you assume. Check before you deploy.

---

## Further Reading

- `first-30-days.md` — Your first month (when mistakes #1, #6, #15 are most likely)
- `first-60-days.md` — Your second month (when mistakes #2, #5, #14 are most likely)
- `first-90-days.md` — Your third month (when mistakes #11, #13 are most likely)
- `understanding-the-deployment-pipeline.md` — Deployment process details
- `understanding-regulated-environments.md` — Compliance requirements
- `incident-management/` — Incident response procedures
- `how-to-earn-trust-on-the-team.md` — How mistake handling affects trust

---

*Last updated: April 2025 | Document owner: Engineering Enablement Team | Review cycle: Quarterly*
*Contributions welcome: If you make a mistake not on this list, add it. That is how the list stays useful.*
