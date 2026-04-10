# Multi-Tenant Database Design

## Overview

Multi-tenant database design enables serving multiple customers (tenants) from a shared database infrastructure while maintaining data isolation, security, and performance. In banking, this applies to BaaS (Banking-as-a-Service) platforms, white-label banking solutions, and internal multi-entity architectures.

## Multi-Tenant Strategies

```mermaid
graph TB
    subgraph Strategy 1: Shared Database, Shared Schema
    S1[All tenants in same tables]
    S1A[tenant_id column on every table]
    S1B[RLS for isolation]
    end
    
    subgraph Strategy 2: Shared Database, Separate Schema
    S2[Separate schema per tenant]
    S2A[schema_1.customers]
    S2B[schema_2.customers]
    end
    
    subgraph Strategy 3: Separate Database
    S3[Separate DB instance per tenant]
    S3A[Tenant 1: DB instance 1]
    S3B[Tenant 2: DB instance 2]
    end
```

| Strategy | Isolation | Cost | Complexity | Best For |
|----------|-----------|------|------------|----------|
| Shared Schema | Row-level | Lowest | Medium | Many small tenants |
| Separate Schema | Schema-level | Low | High | Medium tenants with custom needs |
| Separate Database | Instance-level | Highest | Highest | Enterprise tenants, compliance |

## Shared Schema with RLS (Recommended)

```sql
-- Tenants table
CREATE TABLE tenants (
    tenant_id BIGINT PRIMARY KEY,
    tenant_code VARCHAR(20) UNIQUE NOT NULL,
    tenant_name VARCHAR(100),
    plan VARCHAR(20) DEFAULT 'standard',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    is_active BOOLEAN DEFAULT true
);

-- All tables include tenant_id
CREATE TABLE customers (
    customer_id BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(tenant_id),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(200),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (customer_id, tenant_id)
);

CREATE TABLE accounts (
    account_id BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(tenant_id),
    customer_id BIGINT NOT NULL,
    account_number VARCHAR(20) NOT NULL,
    account_type VARCHAR(20),
    balance DECIMAL(15, 2) DEFAULT 0,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    PRIMARY KEY (account_id, tenant_id),
    FOREIGN KEY (customer_id, tenant_id) REFERENCES customers(customer_id, tenant_id)
);

-- Enable Row-Level Security
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

-- Policies
CREATE POLICY tenant_isolation_customers ON customers
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::BIGINT);

CREATE POLICY tenant_isolation_accounts ON accounts
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::BIGINT);

-- Indexes include tenant_id for efficient filtering
CREATE INDEX idx_customers_tenant ON customers (tenant_id);
CREATE INDEX idx_accounts_tenant ON accounts (tenant_id);
CREATE INDEX idx_accounts_tenant_customer ON accounts (tenant_id, customer_id);
```

## Tenant-Aware Application Code

```python
"""Multi-tenant database access with automatic tenant scoping."""
import psycopg2
from contextlib import contextmanager
from typing import Generator

class TenantDatabaseManager:
    """Manage multi-tenant database connections."""
    
    def __init__(self, dsn: str):
        self.dsn = dsn
    
    @contextmanager
    def tenant_connection(self, tenant_id: int) -> Generator:
        """Get connection scoped to a specific tenant."""
        conn = psycopg2.connect(self.dsn)
        
        with conn.cursor() as cur:
            # Set tenant context (enforced by RLS)
            cur.execute("SET app.tenant_id = %s", (str(tenant_id),))
            conn.commit()
        
        try:
            yield conn
        finally:
            with conn.cursor() as cur:
                cur.execute("RESET app.tenant_id")
            conn.close()
    
    def execute_tenant_query(self, tenant_id: int, query: str, params=None):
        """Execute a query scoped to a tenant."""
        with self.tenant_connection(tenant_id) as conn:
            with conn.cursor() as cur:
                cur.execute(query, params)
                if cur.description:
                    return cur.fetchall()
                conn.commit()
                return None

# Usage
db = TenantDatabaseManager("postgresql://user:pass@db-host/banking")

# Tenant 1 can only see their own data
with db.tenant_connection(tenant_id=1) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM customers")
        # Automatically filtered by RLS to tenant 1 only
        customers = cur.fetchall()
```

## Cross-Tenant Analytics (Admin View)

```sql
-- Admin role bypasses RLS for cross-tenant analytics
CREATE ROLE admin_user;

-- Admin can see all tenants
GRANT SELECT ON ALL TABLES IN SCHEMA public TO admin_user;

-- But cannot modify tenant data directly
-- Must use application layer for writes

-- Cross-tenant analytics
SELECT 
    t.tenant_code,
    t.plan,
    COUNT(DISTINCT c.customer_id) AS customer_count,
    COUNT(DISTINCT a.account_id) AS account_count,
    SUM(a.balance) AS total_balance
FROM tenants t
LEFT JOIN customers c ON t.tenant_id = c.tenant_id
LEFT JOIN accounts a ON t.tenant_id = a.tenant_id
WHERE t.is_active = true
GROUP BY t.tenant_code, t.plan
ORDER BY total_balance DESC;

-- Tenant usage analytics
SELECT 
    t.tenant_code,
    COUNT(DISTINCT c.customer_id) AS customers,
    COUNT(*) AS total_transactions,
    AVG(txn.amount) AS avg_transaction_amount,
    MAX(txn.transaction_time) AS last_activity
FROM tenants t
JOIN customers c ON t.tenant_id = c.tenant_id
JOIN accounts a ON c.customer_id = a.customer_id AND c.tenant_id = a.tenant_id
JOIN transactions txn ON a.account_id = txn.account_id AND a.tenant_id = txn.tenant_id
WHERE txn.transaction_time >= CURRENT_DATE - 30
GROUP BY t.tenant_code
ORDER BY total_transactions DESC;
```

## Tenant Onboarding

```sql
-- Automated tenant provisioning
CREATE OR REPLACE FUNCTION onboard_tenant(
    p_tenant_code VARCHAR(20),
    p_tenant_name VARCHAR(100),
    p_plan VARCHAR(20) DEFAULT 'standard'
) RETURNS BIGINT AS $$
DECLARE
    v_tenant_id BIGINT;
BEGIN
    -- Create tenant record
    INSERT INTO tenants (tenant_code, tenant_name, plan)
    VALUES (p_tenant_code, p_tenant_name, p_plan)
    RETURNING tenant_id INTO v_tenant_id;
    
    -- Create tenant-specific settings
    INSERT INTO tenant_settings (tenant_id, settings)
    VALUES (v_tenant_id, '{"notifications": true, "two_factor": true}');
    
    -- Create tenant API key
    INSERT INTO tenant_api_keys (tenant_id, api_key, created_at)
    VALUES (v_tenant_id, gen_random_uuid()::text, NOW());
    
    RETURN v_tenant_id;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT onboard_tenant('acme_bank', 'Acme Bank', 'enterprise');
```

## Cross-References

- **Row-Level Security**: See [row-level-security.md](row-level-security.md) for RLS details
- **Banking Patterns**: See [banking-db-patterns.md](banking-db-patterns.md) for specific patterns

## Interview Questions

1. **Compare the three multi-tenant strategies. When would you use each?**
2. **How does Row-Level Security help with multi-tenant isolation?**
3. **How do you handle schema migrations in a multi-tenant database?**
4. **What happens when one tenant's workload impacts other tenants (noisy neighbor)?**
5. **How do you implement per-tenant rate limiting at the database level?**
6. **How do you backup and restore a single tenant's data in a shared schema?**

## Checklist: Multi-Tenant Database

- [ ] Tenant isolation strategy defined and enforced (RLS or schema)
- [ ] All tables include tenant_id column
- [ ] Foreign keys include tenant_id for referential integrity
- [ ] Indexes include tenant_id for efficient filtering
- [ ] Application sets tenant context on every connection
- [ ] Admin access for cross-tenant analytics
- [ ] Tenant provisioning automated
- [ ] Per-tenant resource limits configured
- [ ] Noisy neighbor monitoring and alerting
- [ ] Tenant-specific backup/restore tested
