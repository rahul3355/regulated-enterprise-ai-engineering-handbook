# Design Review Exercise

> Review an architecture design document for a GenAI feature — find gaps, ask hard questions, and provide constructive feedback.

## Problem Statement

You are a senior engineer reviewing a design document for a new feature: **GenAI Document Summarization Service**. The service will automatically summarize long policy documents for the internal knowledge assistant.

Read the design document excerpt below and provide a thorough review covering architecture, security, operations, and banking context.

## Design Document Under Review

```markdown
# Design Doc: Document Summarization Service

## Problem
Employees often need to understand long policy documents (20-100 pages).
Currently, they must read the entire document to find relevant information.
We want to auto-generate summaries that employees can read first.

## Proposed Solution
A microservice that:
1. Monitors the document store for new/updated documents
2. When a document changes, sends it to the LLM for summarization
3. Stores the summary alongside the document
4. The RAG pipeline includes the summary in retrieval results

## Architecture
```
Document Store → Event → Summarization Service → LLM API → Summary Store
                                                          ↓
                                                    RAG Pipeline
```

## Technology
- Python FastAPI service
- Triggered by webhook from document management system
- Uses GPT-4 Turbo with 128K context window
- Summaries stored in PostgreSQL (same DB as RAG)
- No caching (summaries are regenerated on document change)

## Data Flow
1. Document uploaded → Webhook to /api/summarize
2. Service fetches document from document store
3. Service sends full document to LLM with prompt:
   "Summarize this document in 3-5 paragraphs."
4. Service stores summary in PostgreSQL
5. RAG pipeline picks up the summary on next indexing

## Security
- Service authenticates to document store with service account
- LLM API call uses existing API key
- Summaries stored in same database as documents

## Scaling
- One webhook per document change
- If many documents change at once, they queue up
- Service processes one document at a time (single-threaded)

## Monitoring
- Log each summarization event
- Track processing time per document
```

## Your Review

Provide feedback in these categories:

### 1. Architecture & Design

Questions to consider:
- Is the event-driven approach appropriate?
- What happens if the summarization fails?
- How do you handle documents longer than the LLM's context window?
- Is single-threaded processing sufficient?
- What happens to the queue during a bulk document update?

### 2. Security

Questions to consider:
- What classification levels can documents have? Should all documents be summarized?
- Does sending full documents to the LLM API create data exposure risk?
- Should summaries inherit the document's access control?
- What if a document contains PII?

### 3. Reliability

Questions to consider:
- What happens if the webhook is lost (document change but no summarization)?
- What happens if the LLM API is down during summarization?
- How do you detect stale summaries (document changed but summary wasn't updated)?
- What's the retry strategy for failed summarizations?

### 4. Quality

Questions to consider:
- How do you evaluate summary quality?
- What happens if the summary is inaccurate or misses critical information?
- Should there be a human-in-the-loop review for certain document types?
- How do you handle documents with complex formatting (tables, diagrams)?

### 5. Banking Context

Questions to consider:
- Should compliance documents be summarized differently from HR policies?
- Are there documents that should NEVER be summarized (legally sensitive)?
- How do you handle version control — what if the document changes between summary generation and retrieval?
- Does the summary itself need an audit trail?

## Example Review

```markdown
# Design Review: Document Summarization Service
# Reviewer: [Your Name]
# Date: [Date]

## Overall Assessment
🟡 Conditional Approval — Good concept, needs revisions in several areas
before implementation can begin.

## Critical Issues (Must Fix Before Implementation)

### 1. No Error Handling or Retry Logic
**Severity:** Critical
**Issue:** The design has no error handling. If the LLM API fails, the
summarization is silently lost. Documents will exist without summaries
and nobody will know.

**Recommendation:**
- Add retry logic with exponential backoff (3 retries, 1min/5min/15min)
- Dead letter queue for documents that fail after all retries
- Alert on DLQ size > 0
- Status tracking: pending → processing → completed / failed

### 2. No Access Control on Summaries
**Severity:** Critical
**Issue:** Summaries are stored in PostgreSQL but the design doesn't
mention access control. If a Confidential document is summarized, the
summary must inherit the same access restrictions.

**Recommendation:**
- Summaries must have the same classification level as the source document
- Summaries must have the same access control list as the source document
- RAG pipeline must filter summaries by user access, same as documents

### 3. No Document Length Handling
**Severity:** High
**Issue:** Documents can be 100+ pages. Even GPT-4 Turbo's 128K context
window has limits. The design doesn't address how to handle documents
that exceed the context window.

**Recommendation:**
- Implement chunking for long documents
- Summarize each chunk, then summarize the summaries (map-reduce pattern)
- Track which chunks were summarized to ensure full coverage

## Significant Issues (Should Fix Before Implementation)

### 4. No Quality Evaluation
**Severity:** High
**Issue:** There's no mechanism to evaluate summary quality. Bad summaries
are worse than no summaries because they give false confidence.

**Recommendation:**
- Build an evaluation dataset of documents with human-written summaries
- Compare AI summaries against human summaries using ROUGE/BERTScore
- For compliance documents, require human review before summary is published

### 5. Single-Threaded Processing
**Severity:** Medium
**Issue:** If 100 documents are updated simultaneously (e.g., annual policy
refresh), they'll queue up. With ~30 seconds per summary, that's 50 minutes
of backlog.

**Recommendation:**
- Use a message queue (Kafka/SQS) for summarization jobs
- Process multiple documents in parallel (configurable concurrency)
- Add priority: compliance documents > HR > IT > general

### 6. No Staleness Detection
**Severity:** Medium
**Issue:** If the webhook fails, the summary becomes stale. There's no
mechanism to detect or fix this.

**Recommendation:**
- Store document hash alongside summary
- Periodic job: compare document hash with summary's document hash
- If they differ, re-trigger summarization
- Alert on stale summaries older than 24 hours

## Minor Issues (Can Fix During Implementation)

### 7. Monitoring Gaps
**Severity:** Low
**Issue:** Basic logging is mentioned but no alerting or dashboards.

**Recommendation:**
- Dashboard: summaries generated/day, failure rate, avg processing time
- Alert: failure rate > 5%, DLQ size > 0, processing time > 2x average

### 8. Prompt Design Missing
**Severity:** Low
**Issue:** The prompt "Summarize this document in 3-5 paragraphs" is too
generic. Different document types need different summary styles.

**Recommendation:**
- Customize prompt by document category (compliance, HR, IT, finance)
- Include document metadata in the prompt (title, version, effective date)
- Ask the LLM to flag any compliance-relevant points

## Open Questions
1. Should we summarize existing documents (backfill) or only new ones?
2. What's the budget for summarization costs? (10K documents × $0.05 = $500)
3. Do we need multi-language summarization for our 40-country workforce?

## Recommendation
Proceed with implementation after addressing Critical and Significant issues.
I'm happy to discuss these findings in the design review meeting.
```

## Extensions

1. **Review a real design doc:** Find a design doc from your organization (or create one) and review it using this format.

2. **Security-focused review:** Re-review the design doc focusing only on security concerns. Use the STRIDE framework.

3. **Write the missing sections:** Complete the design doc by writing the missing sections: Alternatives Considered, Trade-offs, Migration Plan, Timeline.

4. **Present the review:** Deliver the review findings in a simulated design review meeting. Handle pushback from the author.

5. **Review rubric:** Create a rubric for evaluating the quality of design reviews. What makes a review thorough vs. superficial?

## Interview Relevance

Design review is tested in senior and staff engineering interviews:

| Skill | Why It Matters |
|-------|---------------|
| Finding gaps | Can you spot issues the author missed? |
| Constructive feedback | Can you critique without being destructive? |
| Prioritization | Can you distinguish critical from minor? |
| Banking context | Do you consider compliance and security? |
| Systems thinking | Do you consider edge cases and failure modes? |

**Follow-up questions:**
- "What's the most important thing to look for in a design doc?"
- "How do you review a design doc in an area you're not expert in?"
- "What do you do if the author pushes back on your feedback?"
- "How do you balance thoroughness with review speed?"
