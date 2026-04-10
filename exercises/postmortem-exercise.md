# Postmortem Exercise

> Write a blameless postmortem from incident data — the most important learning practice in engineering.

## Problem Statement

An incident occurred where the GenAI compliance assistant returned incorrect regulatory information to loan officers for 47 minutes. Using the incident data below, write a complete blameless postmortem.

## Incident Data

### Timeline

```
09:15 — A compliance policy document (POL-2847 "Capital Requirements 2026")
         was updated by the compliance team. The new version increased the
         corporate loan capital requirement from 8% to 12%.

09:16 — The updated document was uploaded to the document management system.

09:17 — The document management system sent a webhook to the RAG pipeline
         to re-index the updated document. The webhook failed with a 500
         error due to a misconfigured API endpoint.

09:17 — The webhook failure was logged but no alert was generated.
         The retry mechanism was not configured for this webhook.

09:20 — A loan officer asked the GenAI assistant: "What is the capital
         requirement for a corporate loan under Basel III?"

09:20 — The assistant retrieved the OLD version of POL-2847 from the
         vector database (the re-index failed, so the new version was
         never embedded).

09:20 — The assistant responded: "The capital requirement for corporate
         loans under Basel III is 8%." (INCORRECT — should be 12%)

09:22 — The loan officer processed a $50M corporate loan using the 8%
         capital requirement. This was $2M less capital than required.

09:30 — Another loan officer asked the same question and received the
         same incorrect answer.

09:45 — A compliance analyst noticed that the capital requirement in the
         GenAI assistant (8%) didn't match the updated policy document (12%).

09:47 — The compliance analyst filed an incident report.

09:50 — On-call engineer acknowledged and confirmed the index was stale.

09:55 — On-call triggered a manual re-index of POL-2847.

10:00 — Re-index completed. New version is now searchable.

10:02 — On-call verified: assistant now returns the correct 12% answer.

10:05 — Incident declared resolved. Total duration: 47 minutes (from first
         incorrect answer to resolution).

10:15 — Compliance team assessed impact: 2 loan officers received incorrect
         information, 1 loan processed with wrong capital requirement.

10:30 — Compliance team manually corrected the loan's capital requirement
         and notified the loan officer's manager.
```

### System State at Time of Incident

```
RAG Pipeline:
- Document indexer: Monitors document management system via webhook
- Webhook endpoint: /api/v1/index (had been changed to /api/v2/index in
  a deployment 2 weeks ago, but the document management system was not
  updated to use the new endpoint)
- Retry mechanism: Not configured for webhook failures
- Alerting: No alert on webhook failures
- Index staleness monitoring: No monitoring for how old indexed documents are

GenAI Assistant:
- No mechanism to detect that retrieved documents are outdated
- No confidence score on responses
- No "this information is based on documents last updated on X" message
- No flag when the assistant cannot find recent documents on a topic
```

### Impact

```
- 2 employees received incorrect compliance information
- 1 loan ($50M) processed with incorrect capital requirement
  (shortfall: $2M in required capital)
- The error was caught and corrected within 1 hour
- No regulatory filing was made with the incorrect information
- No customer impact (internal process only)
- Reputational risk: moderate (internal confidence in AI assistant affected)
```

## Expected Deliverables

Write a complete postmortem using the template from `templates/postmortem-template.md`. Your postmortem must include:

1. **Executive summary** — What happened, impact, duration
2. **Timeline** — Chronological events
3. **Root cause** — The chain of conditions that allowed this
4. **Contributing factors** — What made it possible, what made it worse
5. **Impact assessment** — Who was affected and how
6. **Action items** — Specific, assigned, time-bound
7. **Lessons learned** — What we now know that we didn't before

## Evaluation Criteria

| Criterion | What Good Looks Like |
|-----------|---------------------|
| **Blamelessness** | No individuals blamed; system focus |
| **Root cause depth** | Not "webhook failed" but "why did the webhook fail, why wasn't it caught, why was there no fallback" |
| **Action quality** | Specific, testable, assigned actions — not "be more careful" |
| **Impact honesty** | Clear about what happened without minimizing or exaggerating |
| **Lessons learned** | Genuine insights, not platitudes |

## Extensions

1. **Present the postmortem:** Deliver a 10-minute postmortem review to a simulated audience. Handle questions and pushback.

2. **Identify additional actions:** Beyond the obvious fixes, what systemic changes would prevent this class of incident?

3. **Write the executive summary:** Condense the postmortem into a 1-page executive summary for VP-level stakeholders.

4. **Design the prevention system:** Build a technical system that would have prevented this incident automatically.

5. **Analyze a real incident:** If you have access to past incidents in your organization, write a postmortem for one using this format.

## Interview Relevance

Postmortem writing is assessed in senior engineering interviews:

| Skill | Why It Matters |
|-------|---------------|
| Blameless analysis | Cultural maturity |
| Root cause depth | Systems thinking |
| Action item quality | Follow-through capability |
| Communication clarity | Can you explain complex incidents simply? |
| Learning orientation | Growth from failures |

**Follow-up questions:**
- "What's the difference between root cause and contributing factor?"
- "How do you write a postmortem when management wants to blame someone?"
- "What makes a good action item vs. a bad one?"
- "How do you ensure postmortem actions actually get completed?"
