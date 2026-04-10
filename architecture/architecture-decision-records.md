# Architecture Decision Records

## What Is an ADR?

An Architecture Decision Record (ADR) documents a significant architectural decision, the context surrounding it, the alternatives considered, and the consequences. ADRs create a searchable history of why the system is built the way it is.

In banking GenAI, ADRs are essential because:
- Regulators may ask why a specific architectural choice was made
- New team members need context for existing decisions
- Revisiting old decisions requires understanding the original tradeoffs

## ADR Format

```markdown
# ADR-NNN: Title

## Status
Proposed | Accepted | Deprecated | Superseded by ADR-NNN

## Context
What is the issue that we're seeing that needs addressing?
What are the forces at play?
What are the constraints?

## Decision
What is the change that we're proposing and/or doing?

## Consequences
What becomes easier or more difficult to do because of this change?
What are the risks?
What is the rollback plan?

## Alternatives Considered
### Alternative 1
Description. Why rejected?

### Alternative 2
Description. Why rejected?

## Compliance Impact
Does this decision affect regulatory compliance?
Which regulations are relevant?

## Review Date
When should this decision be reviewed? (typically 6-12 months)
```

## Example ADR

```markdown
# ADR-042: Use pgvector for Vector Database

## Status
Accepted

## Context
The RAG pipeline requires a vector database for similarity search over banking policy documents. We evaluated several options:

- We need to store ~500K document embeddings (3072 dimensions each)
- Queries must return results in < 200ms at p95
- Must support filtering by document metadata (product line, date)
- Must integrate with existing PostgreSQL infrastructure
- Must meet banking data residency requirements (data cannot leave US)

## Decision
We will use pgvector as our vector database, running on our existing PostgreSQL cluster.

## Consequences

### Positive
- No new infrastructure to manage (uses existing Postgres)
- Data residency requirements automatically satisfied
- Simpler operations (one database instead of two)
- ACID transactions for document updates
- Existing backup/DR procedures apply

### Negative
- pgvector is less mature than dedicated vector DBs (Milvus, Weaviate)
- Query performance degrades significantly above 1M vectors (we are at 500K)
- Limited indexing options compared to dedicated solutions
- Vector search competes with transactional workload for resources

### Risks
- If vector count exceeds 1M, we will need to migrate to a dedicated vector DB
- pgvector development pace is slower than dedicated solutions

### Rollback Plan
If performance is unacceptable, migrate to Milvus with a dual-write period.

## Alternatives Considered

### Pinecone (Managed Vector DB)
Pros: Fully managed, excellent performance, auto-scaling.
Cons: Data leaves our infrastructure (compliance risk), vendor lock-in,
      higher cost at scale ($0.04/vector/month = $20K/month for 500K vectors).

### Milvus (Self-hosted)
Pros: Mature, excellent performance, rich feature set.
Cons: New infrastructure to manage, separate backup/DR procedures,
      data residency requires careful configuration.

### Weaviate
Pros: Good performance, GraphQL interface, built-in reranking.
Cons: Less mature than Milvus, smaller community, same infrastructure concerns.

## Compliance Impact
GLBA: Using existing Postgres simplifies data governance (already approved).
SOX: No change to audit trail requirements.

## Review Date
2025-09-01 (or when vector count exceeds 750K)
```

## ADR Lifecycle

```
Proposed ──> Accepted ──> Superseded
    │            │              │
    │            │              └──> New ADR replaces old one
    │            │
    │            └──> Deprecated (decision reversed)
    │
    └──> Rejected (alternative chosen)
```

## ADR Log

Maintain a central log of all ADRs:

```
┌──────┬──────────────────────────────────────┬──────────┬────────────┐
│  #   │  Title                               │  Status  │  Date      │
├──────┼──────────────────────────────────────┼──────────┼────────────┤
│ 001  │ Use Python for backend services       │ Accepted │ 2024-01-15 │
│ 002  │ Use FastAPI over Django              │ Accepted │ 2024-01-20 │
│ 003  │ Event-driven architecture for audit  │ Accepted │ 2024-02-01 │
│ ...  │ ...                                   │ ...      │ ...        │
│ 042  │ Use pgvector for vector database     │ Accepted │ 2025-03-01 │
│ 043  │ Response caching strategy             │ Proposed │ 2025-03-10 │
└──────┴──────────────────────────────────────┴──────────┴────────────┘
```

## When to Write an ADR

Write an ADR when:
- The decision affects multiple teams or services
- The decision has compliance or regulatory implications
- The decision involves significant cost (infrastructure, licensing)
- The decision is hard to reverse (technology choice, data model)
- There are multiple reasonable alternatives with meaningful tradeoffs

Do NOT write an ADR when:
- The decision is localized to a single module
- The decision is obvious and uncontroversial
- The decision is easily reversed (configuration change)

## ADR Storage and Discovery

```
architecture/
├── decisions/
│   ├── 001-use-python-for-backend.md
│   ├── 002-use-fastapi-over-django.md
│   ├── 003-event-driven-audit.md
│   ├── ...
│   └── 042-use-pgvector.md
├── adr-log.md          # Central log
└── templates/
    └── adr-template.md
```

## Common ADR Mistakes

1. **Writing ADRs after the fact**: ADRs should capture the decision process, not retroactively justify a decision already made.

2. **No review date**: Decisions become stale. Every ADR should have a date when it is re-evaluated.

3. **Missing alternatives**: If you do not document what you considered and rejected, you cannot explain why the current choice is correct.

4. **Too detailed or too vague**: ADRs should be detailed enough to be useful but concise enough to be read. 1-2 pages is ideal.

5. **No compliance impact section**: In banking, architectural decisions often have compliance implications. Document them.

6. **ADRs not linked from code**: If developers cannot find relevant ADRs, they will make decisions without context. Link ADRs from README files and code comments.
