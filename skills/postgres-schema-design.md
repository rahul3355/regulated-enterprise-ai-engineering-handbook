# Skill: PostgreSQL Schema Design

## Core Principles

1. **Model the Domain, Not the Framework** — Schema should reflect banking business concepts (accounts, transactions, customers), not ORM convenience.
2. **Constraints Are Your Friend** — Use `NOT NULL`, `CHECK`, `UNIQUE`, and foreign keys. The database is the last line of data integrity defense.
3. **Design for Auditability** — Every banking table needs audit columns: `created_at`, `created_by`, `updated_at`, `updated_by`. Regulators will ask for this.
4. **Plan for Growth** — Partition tables that will grow beyond millions of rows (transactions, audit logs, GenAI conversation logs).
5. **Separate Read and Write Concerns** — Normalize for writes, denormalize selectively for reads with materialized views.

## Mental Models

### The Banking Schema Hierarchy
```
┌─────────────────────────────────────────────────┐
│  Banking Database Schema Design Hierarchy        │
├─────────────────────────────────────────────────┤
│                                                  │
│  Level 1: Core Banking                           │
│  ├── customers                                   │
│  ├── accounts                                    │
│  ├── transactions                                │
│  └── balances                                    │
│                                                  │
│  Level 2: GenAI Platform                         │
│  ├── conversations                               │
│  ├── messages                                    │
│  ├── embeddings (pgvector)                       │
│  └── document_sources                            │
│                                                  │
│  Level 3: Audit & Compliance                     │
│  ├── audit_log                                   │
│  ├── access_log                                  │
│  ├── consent_records                             │
│  └── data_retention_schedule                     │
│                                                  │
│  Level 4: Platform                               │
│  ├── feature_flags                               │
│  ├── migrations                                  │
│  └── system_config                               │
│                                                  │
└─────────────────────────────────────────────────┘
```

### The Schema Design Checklist
```
□ All tables have a primary key (prefer BIGSERIAL or UUIDv7)
□ Foreign keys defined for all relationships (never orphan records)
□ NOT NULL on all required fields
□ CHECK constraints for business rules (e.g., amount > 0)
□ UNIQUE constraints where business requires uniqueness
□ Audit columns on all tables (created_at, updated_at, etc.)
□ Indexes on frequently queried columns
□ Indexes on foreign key columns
□ Partitioning strategy for large tables
□ Row-level security for multi-tenant data access
□ Data types match actual storage needs (don't use TEXT for everything)
□ Monetary values use DECIMAL(19,4), never FLOAT
□ Timestamps use TIMESTAMPTZ (always timezone-aware)
□ JSONB for flexible metadata, not for core fields
□ Migrations are reversible (always write DOWN scripts)
```

### Common Indexing Patterns
```
-- B-tree: default, good for equality and range queries
CREATE INDEX idx_transactions_account ON transactions(account_id);

-- Partial index: only index active records
CREATE INDEX idx_active_accounts ON accounts(customer_id)
  WHERE status = 'active';

-- GIN index: for JSONB columns
CREATE INDEX idx_documents_metadata ON documents USING GIN(metadata);

-- BRIN index: for time-series data (much smaller than B-tree)
CREATE INDEX idx_txn_created ON transactions USING BRIN(created_at);

-- Composite index: for multi-column queries
CREATE INDEX idx_txn_account_date ON transactions(account_id, created_at DESC);
```

## Step-by-Step Approach

### 1. Define Core Banking Entities

```sql
-- Customers table
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_number VARCHAR(20) UNIQUE NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    phone           VARCHAR(20),
    date_of_birth   DATE NOT NULL,
    nationality     CHAR(2) NOT NULL,
    risk_rating     VARCHAR(20) NOT NULL CHECK (risk_rating IN ('low', 'medium', 'high', 'critical')),
    kyc_status      VARCHAR(20) NOT NULL DEFAULT 'pending'
                      CHECK (kyc_status IN ('pending', 'verified', 'rejected', 'expired')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(100) NOT NULL DEFAULT 'system',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by      VARCHAR(100) NOT NULL DEFAULT 'system',
    metadata        JSONB NOT NULL DEFAULT '{}'
);

-- Accounts table
CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number  VARCHAR(34) UNIQUE NOT NULL, -- IBAN format
    customer_id     UUID NOT NULL REFERENCES customers(id),
    account_type    VARCHAR(20) NOT NULL
                      CHECK (account_type IN ('checking', 'savings', 'credit', 'investment')),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                      CHECK (status IN ('open', 'frozen', 'closed', 'suspended')),
    opened_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(100) NOT NULL DEFAULT 'system',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by      VARCHAR(100) NOT NULL DEFAULT 'system'
);

-- Note: Monetary amounts always use DECIMAL, never FLOAT
-- DECIMAL(19,4) handles up to $10 trillion with sub-penny precision
```

### 2. Design the Transactions Table (with Partitioning)

```sql
-- Transactions: this will be our largest table, so partition by month
CREATE TABLE transactions (
    id              UUID NOT NULL,
    account_id      UUID NOT NULL,
    transaction_type VARCHAR(30) NOT NULL
                      CHECK (transaction_type IN ('deposit', 'withdrawal', 'transfer', 'fee', 'interest', 'adjustment')),
    amount          DECIMAL(19,4) NOT NULL CHECK (amount > 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('credit', 'debit')),
    description     VARCHAR(500),
    reference       VARCHAR(100),
    balance_after   DECIMAL(19,4) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'posted'
                      CHECK (status IN ('pending', 'posted', 'reversed', 'failed')),
    posted_at       TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(100) NOT NULL DEFAULT 'system',
    metadata        JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (id, posted_at)  -- Must include partition key in PK
) PARTITION BY RANGE (posted_at);

-- Create monthly partitions
CREATE TABLE transactions_2025_01 PARTITION OF transactions
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE transactions_2025_02 PARTITION OF transactions
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Indexes on partitioned tables
CREATE INDEX idx_txn_account ON transactions(account_id, posted_at DESC);
CREATE INDEX idx_txn_reference ON transactions(reference) WHERE reference IS NOT NULL;

-- Automate partition creation (run via pg_cron)
CREATE OR REPLACE FUNCTION create_next_partition() RETURNS void AS $$
DECLARE
    next_month TEXT;
    start_date TEXT;
    end_date TEXT;
BEGIN
    next_month := TO_CHAR(NOW() + INTERVAL '2 months', 'YYYY_MM');
    start_date := TO_CHAR(DATE_TRUNC('month', NOW() + INTERVAL '2 months'), 'YYYY-MM-DD');
    end_date := TO_CHAR(DATE_TRUNC('month', NOW() + INTERVAL '3 months'), 'YYYY-MM-DD');

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS transactions_%s PARTITION OF transactions
         FOR VALUES FROM (%L) TO (%L)',
        next_month, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;
```

### 3. Design GenAI Platform Tables with pgvector

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Documents for RAG retrieval
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_uri      VARCHAR(2000) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    content         TEXT NOT NULL,
    content_hash    VARCHAR(64) NOT NULL, -- SHA-256 for deduplication
    document_type   VARCHAR(50) NOT NULL,
    classification  VARCHAR(20) NOT NULL
                      CHECK (classification IN ('public', 'internal', 'confidential', 'restricted')),
    department      VARCHAR(100),
    chunk_index     INTEGER NOT NULL DEFAULT 0,
    total_chunks    INTEGER NOT NULL DEFAULT 1,
    embedding       vector(1536),  -- OpenAI text-embedding-3-small dimension
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Vector similarity search index
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Conversations for audit and analytics
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         VARCHAR(100) NOT NULL,
    tenant_id       VARCHAR(100) NOT NULL, -- department/business unit
    model_name      VARCHAR(100) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    prompt_template VARCHAR(200) NOT NULL,
    prompt_template_version VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'completed'
                      CHECK (status IN ('completed', 'failed', 'blocked', 'timeout')),
    total_tokens    INTEGER NOT NULL DEFAULT 0,
    prompt_tokens   INTEGER NOT NULL DEFAULT 0,
    completion_tokens INTEGER NOT NULL DEFAULT 0,
    cost_usd        DECIMAL(10,6) NOT NULL DEFAULT 0,
    duration_ms     INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Individual messages within a conversation
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    role            VARCHAR(20) NOT NULL CHECK (role IN ('system', 'user', 'assistant', 'tool')),
    content         TEXT NOT NULL,
    token_count     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index for querying conversation history
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at);
```

### 4. Add Row-Level Security for Multi-Tenant Access

```sql
-- Enable RLS on documents
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see documents matching their clearance level
CREATE POLICY document_access_policy ON documents
    USING (
        classification <= current_setting('app.user_clearance', true)::VARCHAR(20)
        AND department = current_setting('app.user_department', true)
    );

-- Policy: users can only see their own conversations
CREATE POLICY conversation_owner_policy ON conversations
    USING (user_id = current_setting('app.user_id', true));

-- Set context at connection start (done by connection pooler)
-- SET app.user_id = 'emp-12345';
-- SET app.user_clearance = 'confidential';
-- SET app.user_department = 'risk-management';
```

### 5. Create Audit Logging with Triggers

```sql
-- Audit log table
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_data        JSONB,
    new_data        JSONB,
    changed_by      VARCHAR(100) NOT NULL,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    request_id      VARCHAR(100)  -- For distributed tracing
);

-- Partition audit_log by month (it grows very fast)
-- CREATE TABLE audit_log PARTITION BY RANGE (changed_at);

-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_function() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, record_id, action, new_data, changed_by, request_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW), current_setting('app.user_id', true), current_setting('app.request_id', true));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_by, request_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_setting('app.user_id', true), current_setting('app.request_id', true));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_data, changed_by, request_id)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD), current_setting('app.user_id', true), current_setting('app.request_id', true));
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply to accounts table
CREATE TRIGGER accounts_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON accounts
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

### 6. Use Migrations (with a tool like Flyway or Alembic)

```python
# Alembic migration example: add index for RAG performance
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Add composite index for customer account lookup
    op.create_index(
        'idx_accounts_customer_status',
        'accounts',
        ['customer_id', 'status'],
        postgresql_where=sa.text("status = 'active'")
    )

    # Add GIN index for document metadata queries
    op.execute(
        "CREATE INDEX idx_documents_metadata_gin ON documents USING GIN(metadata)"
    )

def downgrade():
    op.drop_index('idx_documents_metadata_gin', 'documents')
    op.drop_index('idx_accounts_customer_status', 'accounts')
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Using `FLOAT` for monetary values | Rounding errors in financial calculations | Always use `DECIMAL(19,4)` |
| No foreign key constraints | Orphaned records, data inconsistency | Define FKs on all relationships |
| Missing indexes on FK columns | Slow JOINs, full table scans | Index all FK columns |
| Using `TIMESTAMP` instead of `TIMESTAMPTZ` | Timezone confusion, incorrect date queries | Always use `TIMESTAMPTZ` |
| Storing JSON for everything | No type safety, slow queries, no constraints | Use JSONB only for truly flexible metadata |
| No partitioning on large tables | Query performance degrades over time | Partition by time for time-series data |
| Soft deletes without indexes on `deleted_at` | Queries slow down as table grows | Partial index: `WHERE deleted_at IS NULL` |
| Ignoring connection pooling limits | Connection exhaustion under load | Use PgBouncer with transaction pooling |
| Not planning for data retention | Database grows without bound | Implement partition-based archival and drop |
| Over-normalization with too many JOINs | Read performance suffers | Use materialized views for complex read patterns |
| Missing `CHECK` constraints | Invalid data enters through application bugs | Enforce business rules at database level |
| No RLS in multi-tenant setup | Data leakage between tenants | Enable RLS with tenant-based policies |

## Banking-Specific Concerns

1. **Regulatory Data Retention** — Transaction data must be retained for 7+ years per banking regulations. Implement a data retention schedule that archives old partitions to cold storage (e.g., S3 Glacier) before dropping them.
2. **Audit Trail Requirements** — Every change to customer data, account status, or transaction must be logged with who made the change and when. Use triggers or application-level audit logging.
3. **Data Classification** — Schema must support data classification (PII, PCI, confidential). Use column-level encryption or separate schemas for different classification levels.
4. **Segregation of Duties** — Database access roles must be separated: read-only for analysts, read-write for applications, admin for DBAs only.
5. **PCI-DSS Compliance** — Card data must never be stored in plain text. Tokenize card numbers and store only tokens in Postgres.
6. **GDPR Right to Erasure** — Customer data must be deletable on request. Design soft-delete patterns and anonymization procedures.
7. **Reconciliation** — Balances must always reconcile with transaction history. Add a `CHECK` constraint or nightly reconciliation job.

## GenAI-Specific Concerns

1. **Embedding Versioning** — When you change embedding models, old embeddings become incompatible. Store `embedding_model` and `embedding_version` columns to track which model generated each vector.
2. **Vector Index Maintenance** — IVFFlat and HNSW indexes need periodic rebuilding. Schedule index maintenance during low-traffic windows.
3. **Conversation Data Sensitivity** — User prompts may contain PII or confidential information. Encrypt conversation content at rest using pgcrypto.
4. **Token Cost Tracking** — Track per-user, per-department token costs in the schema for chargeback and budgeting.
5. **Prompt Template Versioning** — Store prompt templates as separate tables with versions, so you can reproduce any conversation.
6. **Guardrail Audit Trail** — Log when guardrails block or modify prompts/responses. This is critical for compliance reviews.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Table bloat ratio | > 2x expected size | Dead tuples not being cleaned; run VACUUM |
| Index hit rate | < 95% | Queries doing sequential scans; add indexes |
| Cache hit rate | < 99% | Database reading from disk; increase shared_buffers |
| Long-running queries | > 30s | Possible lock contention or missing indexes |
| Lock wait time | > 5s | Concurrent transactions blocking each other |
| Replication lag | > 10s | Read replicas serving stale data |
| Connection pool utilization | > 80% | Running out of connections; scale PgBouncer |
| Partition size | > 10GB per partition | Partitioning granularity too coarse |
| Audit log growth | > 100GB/month | May need separate audit database |
| Deadlock count | > 0 per day | Application logic causing circular locks |

## Interview Questions

1. Why should you use `DECIMAL` instead of `FLOAT` for monetary values? Give an example of a rounding error that could occur.
2. How would you design a schema to store 100 million+ monthly transactions while keeping queries fast?
3. What is row-level security and when would you use it in a banking application?
4. How do you handle schema migrations in a zero-downtime deployment?
5. Explain the difference between `TIMESTAMP` and `TIMESTAMPTZ`. Why does it matter?
6. When would you use JSONB instead of normalized columns? When would you regret that choice?
7. How would you design an audit trail for customer data changes that satisfies regulatory requirements?
8. What strategies would you use to manage embedding versioning when upgrading your RAG model?

## Hands-On Exercise

### Exercise: Design the Schema for a GenAI-Powered Financial Assistant

**Problem:** You are building an internal GenAI assistant for a bank's risk management team. The assistant needs to:
- Search through regulatory documents, internal policies, and risk reports (RAG)
- Maintain conversation history for audit purposes
- Track which documents were cited in each response
- Enforce access control based on user clearance level
- Log all guardrail interventions (blocked prompts, filtered responses)

Design the PostgreSQL schema that supports this system.

**Constraints:**
- Must support 10,000+ documents with vector similarity search
- Must track conversation history for 50,000+ employees
- Must comply with audit requirements (who asked what, when, and what was returned)
- Must support document-level access control (users only see documents they're cleared for)
- Must track token costs per department for chargeback
- Must handle embedding model upgrades gracefully

**Expected Output:**
- SQL `CREATE TABLE` statements for all tables
- Index definitions for performance
- Row-level security policies
- Partitioning strategy for large tables
- Audit trigger definition
- Sample queries showing common access patterns

**Hints:**
- Use pgvector for embeddings (check `databases/vector-databases.md`)
- Look at the transactions table pattern for partitioning
- Use `CHECK` constraints for data validation
- Consider a separate schema for prompt templates

**Extension:**
- Write a migration script (Alembic or SQL) to add a new column to track embedding model version across all existing documents
- Design a materialized view that shows per-department token costs by month
- Implement a data retention policy that archives conversations older than 1 year

---

**Related files:**
- `databases/postgres-fundamentals.md` — Core PostgreSQL concepts
- `databases/vector-databases.md` — pgvector and embedding storage
- `security/data-encryption.md` — Column-level encryption patterns
- `data-engineering/data-retention.md` — Data lifecycle management
- `genai-platforms/rag-architecture.md` — RAG system design
- `regulations-and-compliance/gdpr.md` — GDPR data erasure requirements
