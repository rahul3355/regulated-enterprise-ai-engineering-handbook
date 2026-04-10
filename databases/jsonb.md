# JSONB for Flexible Schema in Banking

## Overview

PostgreSQL's JSONB type provides document storage with indexing and querying capabilities, combining the flexibility of NoSQL with the ACID guarantees of a relational database. In banking, JSONB is ideal for storing semi-structured data like customer preferences, product configurations, and event payloads.

## JSONB Fundamentals

```sql
-- JSONB stores binary-decomposed JSON (unlike JSON which stores text)
-- JSONB: No duplicate keys, indexed, slightly slower inserts
-- JSON: Preserves formatting, faster inserts, no indexing

CREATE TABLE customer_profiles (
    customer_id BIGINT PRIMARY KEY,
    profile JSONB NOT NULL DEFAULT '{}',
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insert JSONB data
INSERT INTO customer_profiles (customer_id, profile) VALUES
(1, '{
    "preferences": {
        "language": "en",
        "notifications": {"email": true, "sms": false, "push": true},
        "theme": "dark"
    },
    "risk_profile": {
        "score": 720,
        "category": "moderate",
        "last_assessed": "2025-01-15"
    },
    "metadata": {
        "source": "mobile_app",
        "version": "2.1.0",
        "tags": ["premium", "digital_native"]
    }
}');
```

## Querying JSONB

```sql
-- Extract value (returns JSONB)
SELECT profile->'preferences'->>'language' AS language
FROM customer_profiles
WHERE customer_id = 1;

-- Extract as text (->> returns text, -> returns JSONB)
SELECT 
    profile->'risk_profile'->>'category' AS risk_category,
    (profile->'risk_profile'->>'score')::INT AS risk_score
FROM customer_profiles;

-- Check containment
SELECT * FROM customer_profiles
WHERE profile @> '{"preferences": {"theme": "dark"}}';

-- Check key existence
SELECT * FROM customer_profiles
WHERE profile ? 'risk_profile';

-- Check array containment
SELECT * FROM customer_profiles
WHERE profile->'metadata'->'tags' @> '["premium"]';

-- Nested array query: customers with push notifications enabled
SELECT * FROM customer_profiles
WHERE (profile->'preferences'->'notifications'->>'push')::BOOLEAN = true;

-- Unnest JSONB array
SELECT 
    customer_id,
    jsonb_array_elements_text(profile->'metadata'->'tags') AS tag
FROM customer_profiles;
```

## JSONB Indexing

```sql
-- GIN index on entire JSONB document (supports @>, ?, ?&, ?|)
CREATE INDEX idx_profile_gin ON customer_profiles USING gin (profile);

-- Query uses GIN index
SELECT * FROM customer_profiles
WHERE profile @> '{"risk_profile": {"category": "moderate"}}';

-- Expression index on specific key (faster for specific queries)
CREATE INDEX idx_profile_risk_score 
ON customer_profiles ((profile->'risk_profile'->>'score'));

-- Query uses expression index
SELECT * FROM customer_profiles
WHERE (profile->'risk_profile'->>'score')::INT > 700;

-- Partial index for specific conditions
CREATE INDEX idx_premium_customers 
ON customer_profiles (customer_id)
WHERE profile @> '{"metadata": {"tags": ["premium"]}}';
```

## Banking Use Cases

```sql
-- 1. Customer preferences (frequently changing, user-specific)
ALTER TABLE customers ADD COLUMN preferences JSONB DEFAULT '{}';

-- 2. Product configurations (varying fields by product type)
CREATE TABLE banking_products (
    product_id BIGINT PRIMARY KEY,
    product_type VARCHAR(50),
    base_config JSONB,
    terms JSONB,
    fees JSONB
);

INSERT INTO banking_products VALUES
(1, 'SAVINGS', 
 '{"min_balance": 100, "interest_calculation": "daily_compound"}'::jsonb,
 '{"min_term_days": 0, "max_withdrawals_monthly": null}'::jsonb,
 '{"monthly_fee": 0, "overdraft_fee": null}'::jsonb
),
(2, 'LOAN',
 '{"principal_max": 500000, "term_options": [12, 24, 36, 60]}'::jsonb,
 '{"min_credit_score": 650, "collateral_required": true}'::jsonb,
 '{"origination_fee_pct": 1.5, "late_fee": 35}'::jsonb
);

-- 3. Event payloads (audit trail, CDC events)
CREATE TABLE customer_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id BIGINT,
    event_type VARCHAR(50),
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_events_type ON customer_events (event_type);
CREATE INDEX idx_events_payload ON customer_events USING gin (payload);
CREATE INDEX idx_events_customer ON customer_events (customer_id, created_at DESC);

-- 4. GenAI document metadata
CREATE TABLE genai_documents (
    document_id UUID PRIMARY KEY,
    title VARCHAR(500),
    content TEXT,
    metadata JSONB,  -- Flexible metadata for RAG filtering
    embedding vector(1536)
);

CREATE INDEX idx_doc_metadata ON genai_documents USING gin (metadata);

-- Filter by metadata in RAG queries
SELECT document_id, content, embedding <=> $1 AS similarity
FROM genai_documents
WHERE metadata @> '{"doc_type": "product_info", "status": "published"}'
ORDER BY embedding <=> $1
LIMIT 5;
```

## Modifying JSONB

```sql
-- Update specific keys (without replacing entire document)
UPDATE customer_profiles SET
    profile = jsonb_set(profile, '{risk_profile,score}', '750'),
    updated_at = NOW()
WHERE customer_id = 1;

-- Add new key
UPDATE customer_profiles SET
    profile = profile || '{"verified": true}'::jsonb,
    updated_at = NOW()
WHERE customer_id = 1;

-- Remove key
UPDATE customer_profiles SET
    profile = profile - 'metadata',
    updated_at = NOW()
WHERE customer_id = 1;

-- Update nested value
UPDATE customer_profiles SET
    profile = jsonb_set(
        profile, 
        '{preferences,notifications,sms}', 
        'true'
    ),
    updated_at = NOW()
WHERE customer_id = 1;

-- Append to array
UPDATE customer_profiles SET
    profile = jsonb_set(
        profile,
        '{metadata,tags}',
        (profile->'metadata'->'tags') || '"verified"'::jsonb
    )
WHERE customer_id = 1;
```

## Cross-References

- **Schema Design**: See [schema-design.md](../data-engineering/schema-design.md) for when to use JSONB
- **Indexing**: See [indexing.md](indexing.md) for GIN index details

## Interview Questions

1. **When would you use JSONB instead of a normalized relational schema in banking?**
2. **How do you index JSONB columns? What operators are supported?**
3. **What is the difference between JSON and JSONB in PostgreSQL?**
4. **How do you update a single key in a JSONB document without replacing the entire value?**
5. **What are the performance trade-offs of storing data in JSONB?**
6. **How would you use JSONB for GenAI document metadata and RAG filtering?**

## Checklist: JSONB Usage

- [ ] JSONB used for semi-structured, frequently changing data
- [ ] Core transactional data kept in relational columns
- [ ] GIN index created for containment queries (@>)
- [ ] Expression indexes for frequently queried specific keys
- [ ] JSONB validated at application level (no schema enforcement in DB)
- [ ] Document size monitored (large JSONB impacts performance)
- [ ] JSONB used for GenAI metadata with RAG filtering
- [ ] Migration path from JSONB to structured columns if needed
