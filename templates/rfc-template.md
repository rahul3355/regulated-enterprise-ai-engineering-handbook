# RFC Template

> Use this template for lightweight proposals that gather feedback before significant work begins.

## Template

```markdown
# RFC-[Number]: [Title]

| Field | Value |
|-------|-------|
| **Author** | [Name, team] |
| **Status** | Draft / In Discussion / Accepted / Declined / Superseded |
| **Created** | [YYYY-MM-DD] |
| **Updated** | [YYYY-MM-DD] |
| **Discussion Link** | [GitHub/Confluence link] |
| **Decision Deadline** | [YYYY-MM-DD] |

## Summary

[One paragraph: What are you proposing and why? A busy reader should
understand the essence from this paragraph alone.]

## Problem

[What problem are you trying to solve? Who experiences this problem?
How do they currently work around it? Include data if available.]

## Proposed Change

[What are you proposing? Be specific but concise.
Include diagrams if they help. 1-2 pages maximum.]

### How It Works

[Brief description of the mechanism/design.]

### Impact

- **Who is affected:** [Teams, systems, users]
- **What changes:** [Behavior, interfaces, processes]
- **What stays the same:** [What this does NOT change]

## Alternatives Considered

[What other approaches did you consider? Why are you not proposing them?]

### Alternative 1: [Name]
- **Pros:** [...]
- **Cons:** [...]
- **Why not chosen:** [...]

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| [...] | Low/Med/High | Low/Med/High | [...] |

## Migration Plan

[If this changes existing behavior, how do you transition?]

## Open Questions

- [ ] [Question that needs resolution]
- [ ] [Question assigned to person/team]

## Decision Criteria

[How will we know if this RFC is a good idea? What metrics or signals
will indicate success?]

## Timeline

[If accepted, when would this happen? Key milestones?]
```

## Example: Filled RFC

```markdown
# RFC-047: Implement Prompt Response Caching for Common Queries

| Field | Value |
|-------|-------|
| **Author** | Sarah Chen, GenAI Platform |
| **Status** | Accepted |
| **Created** | 2026-03-15 |
| **Updated** | 2026-03-20 |
| **Discussion Link** | github.com/org/repo/discussions/47 |
| **Decision Deadline** | 2026-03-22 |

## Summary

We propose implementing a Redis-based cache for frequently-asked
questions to the GenAI assistant, reducing LLM API costs by an
estimated 15-25% and improving p50 latency from 1.2s to 50ms for
cached responses.

## Problem

Our GenAI assistant processes ~50K queries per day. Analysis of
query logs shows that 22% of queries are semantically similar to
queries we've already answered (e.g., "What's the expense report
policy?" asked by hundreds of employees).

Each uncached query costs $0.003-0.015 in LLM API fees and takes
800ms-2s. Currently, there is no caching — every query calls the
LLM API, even identical repetitions.

## Proposed Change

Implement a semantic similarity-based cache using Redis:

1. On incoming query, generate embedding (using existing model)
2. Query Redis for cached responses with cosine similarity > 0.95
3. If cache hit, return cached response (5ms vs. 1.2s)
4. If cache miss, call LLM API, store response in Redis with 7-day TTL

### How It Works

Cache key: Embedding vector (quantized to 256 dimensions)
Cache value: Response text + metadata (model version, timestamp)
TTL: 7 days (responses may become stale as policies change)

### Impact

- **Who is affected:** All assistant users (transparent improvement)
- **What changes:** Response path includes cache lookup (+5-10ms for miss)
- **What stays the same:** LLM API call for uncached queries, audit logging

## Alternatives Considered

### Exact-match cache (hash-based)
- **Pros:** Simpler implementation, no embedding overhead
- **Cons:** Only catches identical queries (estimated 8% hit rate vs. 22%)
- **Why not chosen:** Semantic similarity catches 3x more cacheable queries

### LLM-based deduplication
- **Pros:** More accurate similarity detection
- **Cons:** Adds LLM call for every query (defeats the purpose)
- **Why not chosen:** Cost and latency overhead

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Stale cached responses if policies change | Medium | Medium | 7-day TTL; manual invalidation for urgent updates |
| Incorrect response for sensitive queries | Low | High | High similarity threshold (0.95); log all cache hits |
| Redis operational overhead | Low | Low | Use existing Redis cluster; monitored by platform team |

## Migration Plan

1. Deploy cache alongside existing flow (no behavior change)
2. Enable in "shadow mode" (check cache but always call LLM, compare)
3. After 1 week validation, enable for 10% of traffic
4. Gradually increase to 100% over 2 weeks
5. Roll back to full LLM calls if any issues

## Open Questions

- [ ] Should we implement manual cache invalidation for policy updates?
- [x] What similarity threshold? → 0.95 (decided after discussion)

## Decision Criteria

- Cache hit rate > 15% after 30 days
- No increase in user-reported incorrect responses
- P50 latency improvement for cached queries to < 100ms

## Timeline

- Week of Mar 25: Implementation
- Week of Apr 1: Shadow mode testing
- Week of Apr 8: Production rollout (10% → 100%)
- Week of Apr 15: Review metrics and adjust
```

## When to Use an RFC vs. a Design Doc

| Use RFC When... | Use Design Doc When... |
|-----------------|----------------------|
| Exploring an idea | Committed to building |
| Seeking team input | Need cross-team alignment |
| Deciding whether to proceed | Deciding how to implement |
| 1-3 pages is sufficient | Need 5-15 pages of detail |
| Discussion leads to go/no-go | Discussion leads to approved design |
