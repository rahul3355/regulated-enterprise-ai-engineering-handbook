# Architecture Decision Record (ADR) Template

> Capture important architectural decisions, their context, and consequences for future reference.

## Template

```markdown
# ADR-[Number]: [Title]

| Field | Value |
|-------|-------|
| **Status** | Proposed / Accepted / Deprecated / Superseded |
| **Date** | [YYYY-MM-DD] |
| **Author** | [Name] |
| **Reviewers** | [Names] |
| **Related** | [Links to design docs, RFCs, other ADRs] |

## Context

[What is the issue that we're addressing? What forces are at play?
Describe the problem, constraints, and requirements. 1-3 paragraphs.]

## Decision

[What is the change that we're proposing and/or doing?
Be specific and clear. 1-2 paragraphs.]

## Consequences

[What becomes easier or more difficult to do because of this decision?
Be honest about trade-offs.]

### Positive
- [...]

### Negative
- [...]

### Risks
- [...]

## Alternatives Considered

| Alternative | Pros | Cons | Why Not |
|-------------|------|------|---------|
| [...] | [...] | [...] | [...] |
```

## Example: Filled ADR

```markdown
# ADR-142: Use pgvector Over Pinecone for Embeddings Storage

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-02-15 |
| **Author** | James Wu, GenAI Platform |
| **Reviewers** | Sarah Chen, Priya Patel, Alex Rodriguez |
| **Related** | RFC-041 (RAG Pipeline Redesign), ADR-138 (Postgres as Primary) |

## Context

Our RAG pipeline needs a vector database for storing and retrieving
document embeddings. We evaluated pgvector (PostgreSQL extension)
and Pinecone (managed vector DB service).

Forces:
- We need cosine similarity search on 384-dimensional embeddings
- Dataset: ~2M documents, growing to 10M
- Latency requirement: p95 < 100ms for retrieval
- Operational constraint: Prefer to minimize new infrastructure
- Budget constraint: No incremental SaaS costs without strong justification
- Security requirement: Data must not leave our VPC

## Decision

We will use pgvector as our vector database, running on our existing
PostgreSQL cluster. We will NOT adopt Pinecone at this time.

Implementation:
- Add pgvector extension to our PostgreSQL 15 cluster
- Create IVFFlat index on the embeddings column
- Use the existing Postgres connection pool for queries
- Monitor query performance; reassess at 5M documents

## Consequences

### Positive
- Zero incremental infrastructure cost (uses existing Postgres)
- No new operational burden (Postgres is already managed by DBA team)
- Data stays within our VPC (security requirement met)
- Simpler architecture (one database instead of two)
- Unified backup/restore strategy

### Negative
- pgvector performance may not scale to 10M+ documents as well as
  dedicated vector databases
- IVFFlat index requires periodic rebuilds (not incremental)
- Fewer advanced features (no hybrid search out of the box)
- Postgres cluster resource contention (vector queries compete with
  transactional queries)

### Risks
- If we hit performance limits at scale, migration to a dedicated
  vector DB will require re-indexing all documents (estimated 3 days)
- pgvector is relatively new (v0.5.0). Community support is growing
  but less mature than Pinecone.

## Alternatives Considered

| Alternative | Pros | Cons | Why Not |
|-------------|------|------|---------|
| **Pinecone** | Better scalability, managed service, advanced features | $48K/yr cost, data leaves VPC, new vendor | Cost and security concerns outweigh benefits at current scale |
| **Milvus** | Open source, designed for scale, advanced indexing | New infrastructure to manage, operational complexity | We don't have the operational capacity for another database |
| **FAISS** | Fast, flexible, no new infrastructure | In-memory only (persistence challenges), no built-in access control | Lack of persistence and access control makes it unsuitable |
```

## ADR Best Practices

### Write ADRs When
- Choosing between significant alternatives (technology, architecture, patterns)
- Making a decision that will be expensive to reverse
- Setting a precedent that other teams will follow
- The rationale might not be obvious to future engineers

### Don't Write ADRs For
- Routine technology choices with obvious answers
- Decisions that will be revisited in weeks
- Implementation details covered in design docs

### Keep ADRs Alive
- Review ADRs quarterly — are they still valid?
- Supersede ADRs when decisions change (don't delete them)
- Link ADRs in design docs and onboarding materials
