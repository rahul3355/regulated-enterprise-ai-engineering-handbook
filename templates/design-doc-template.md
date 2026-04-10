# Design Document Template

## Instructions

Fill in this template when designing a non-trivial feature or system. Remove sections that do not apply. Add sections as needed for your specific case.

---

# Design Doc: [Feature Name]

| Field | Value |
|-------|-------|
| **Author** | [Name] |
| **Reviewers** | [Names] |
| **Status** | Draft / In Review / Approved / Superseded |
| **Created** | YYYY-MM-DD |
| **Last Updated** | YYYY-MM-DD |
| **Target Release** | [Quarter / Sprint] |

## Problem Statement

**What problem are we solving?**

Describe the problem from the user's perspective. Include data or evidence that the problem exists and is worth solving.

**Banking context**: How does this problem affect customers, compliance, or business operations?

**Current state**: How is this problem handled today? (Manually? Not at all? With a workaround?)

**Impact**: What happens if we do NOT solve this problem?

## Goals and Non-Goals

### Goals

- [Specific, measurable goal 1]
- [Specific, measurable goal 2]

### Non-Goals

- [What is explicitly out of scope for this design]
- [What will be addressed in a future iteration]

## Proposed Solution

**High-level description**: Summarize the solution in 2-3 sentences.

**Architecture overview**: Include a diagram showing how new components fit into the existing system.

```
[Diagram: ASCII, Mermaid, or link to drawing tool]
```

### Key Components

| Component | Responsibility | Technology |
|-----------|---------------|------------|
| [Name] | [What it does] | [Language, framework, infrastructure] |
| [Name] | [What it does] | [Language, framework, infrastructure] |

### Data Flow

Describe how data flows through the system. Include:
- What data enters the system and from where
- How data is transformed
- Where data is stored
- What data leaves the system and where it goes

### API Design

```
POST /api/v1/[resource]
Request:  { ... }
Response: { ... }

GET /api/v1/[resource]/{id}
Response: { ... }
```

Include full request/response schemas (can link to OpenAPI spec).

### Data Model

```sql
CREATE TABLE [table_name] (
    id UUID PRIMARY KEY,
    -- columns
);
```

Or document NoSQL schema if applicable.

## Alternatives Considered

### Alternative 1: [Name]

**Description**: What is this alternative?

**Pros**:
- ...

**Cons**:
- ...

**Why rejected**: ...

### Alternative 2: [Name]

[Same structure as above]

## Technical Details

### LLM Integration (if applicable)

- **Model selection**: Which model(s) will be used and why?
- **Prompt design**: How will prompts be structured? (Link to prompt templates)
- **Response handling**: How will LLM outputs be parsed and validated?
- **Fallback strategy**: What happens when the LLM provider fails?
- **Cost estimate**: Expected tokens per request and monthly cost

### Security

- **Authentication**: How are users authenticated?
- **Authorization**: What can each user role do?
- **Data protection**: How is sensitive data handled?
- **PII handling**: Where does PII enter the system and how is it protected?
- **Threat model**: Link to threat model document if one exists

### Compliance (REQUIRED for banking)

- **Regulatory requirements**: Which regulations apply? (GLBA, ECOA, TILA, etc.)
- **Audit trail**: How are actions logged for compliance?
- **Data retention**: How long is data retained?
- **Disclaimer requirements**: What disclaimers must be shown to users?
- **Compliance review status**: Has compliance reviewed sign off?

### Observability

- **Logging**: What key events are logged? (Ensure no PII)
- **Metrics**: What metrics are emitted?
- **Alerting**: What alerts are configured?
- **Dashboards**: What dashboards are needed?

### Error Handling

| Error Scenario | Detection | Response | Recovery |
|---------------|-----------|----------|----------|
| LLM provider timeout | Response time > 10s | Return cached response or error | Retry with backoff |
| Vector DB unavailable | Connection error | Return error with degraded mode | Retry, alert on-call |

### Performance

- **Expected load**: Requests per second, data volume
- **Latency targets**: p50, p95, p99 targets
- **Capacity planning**: How many instances, what size?
- **Bottleneck analysis**: Where do you expect bottlenecks?

## Implementation Plan

### Phase 1: [Name] (Week 1-2)

- [ ] Task 1
- [ ] Task 2

### Phase 2: [Name] (Week 3-4)

- [ ] Task 1
- [ ] Task 2

### Rollout Strategy

1. **Internal testing**: Deploy to staging, test with internal users
2. **Beta**: Deploy to 5% of production traffic
3. **General availability**: Deploy to 100% of traffic

### Rollback Plan

If the feature causes issues:
1. **Immediate**: Toggle feature flag off (0 minute impact)
2. **Short-term**: Revert deployment (5-15 minutes)
3. **Long-term**: Rollback data migrations if any (plan required)

## Cost Estimate

| Resource | Monthly Cost | Notes |
|----------|-------------|-------|
| LLM API calls | $X,XXX | Based on estimated tokens/query |
| Infrastructure | $X,XXX | Compute, storage, networking |
| **Total** | **$X,XXX** | |

## Open Questions

- [ ] Question 1 -- Owner -- Due date
- [ ] Question 2 -- Owner -- Due date

## Appendix

### Glossary

Define any domain-specific terms used in this document.

### References

- Link to related design documents
- Link to ADRs
- Link to research or benchmarks
- Link to competitor analysis if relevant
