# Row-Level Security (RLS) for Multi-Tenant Access Control

## Overview

Row-Level Security (RLS) enforces access control at the row level within the database, ensuring that users can only access data they are authorized to see. In multi-tenant banking systems, RLS prevents cross-tenant data leakage and simplifies application-level access control.

## RLS Fundamentals

```sql
-- Enable RLS on a table
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

-- Create a policy: Users can only see their own accounts
CREATE POLICY account_owner_policy ON accounts
    FOR ALL
    USING (customer_id = current_setting('app.current_customer_id')::BIGINT);

-- Set the customer context (from application)
SET app.current_customer_id = '1001';

-- Now queries automatically filter rows
SELECT * FROM accounts;  -- Only returns accounts for customer 1001

-- RLS applies to SELECT, INSERT, UPDATE, DELETE
-- Table owner and superusers bypass RLS (by default)
```

## Multi-Tenant Banking RLS

```sql
-- Multi-tenant banking: Separate customers by tenant (branch/entity)

-- Tenants table
CREATE TABLE tenants (
    tenant_id BIGINT PRIMARY KEY,
    tenant_name VARCHAR(100),
    schema_prefix VARCHAR(20)
);

-- Customer data with tenant isolation
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(tenant_id),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(200),
    customer_segment VARCHAR(20),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see customers in their tenant
CREATE POLICY tenant_isolation_policy ON customers
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

-- Role-based policies
CREATE ROLE tenant_user;
CREATE ROLE tenant_admin;

-- Regular users: Only see and modify their tenant's data
CREATE POLICY tenant_user_select ON customers
    FOR SELECT
    TO tenant_user
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

CREATE POLICY tenant_user_insert ON customers
    FOR INSERT
    TO tenant_user
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

CREATE POLICY tenant_user_update ON customers
    FOR UPDATE
    TO tenant_user
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

-- Admins: Can see all tenants but only modify their own
CREATE POLICY tenant_admin_select ON customers
    FOR SELECT
    TO tenant_admin
    USING (true);  -- See all

CREATE POLICY tenant_admin_modify ON customers
    FOR UPDATE
    TO tenant_admin
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::BIGINT);
```

## RLS with Application Integration

```python
"""
RLS context setup in application code.
Sets tenant/customer context at connection level.
"""
import psycopg2
from contextlib import contextmanager

class RLSDatabaseManager:
    """Manage database connections with RLS context."""
    
    def __init__(self, dsn: str):
        self.dsn = dsn
    
    @contextmanager
    def tenant_connection(self, tenant_id: int, user_role: str = 'tenant_user'):
        """Get a connection scoped to a specific tenant."""
        conn = psycopg2.connect(self.dsn)
        
        with conn.cursor() as cur:
            # Set tenant context
            cur.execute(
                "SET app.current_tenant_id = %s",
                (str(tenant_id),)
            )
            
            # Set role
            cur.execute(f"SET ROLE {user_role}")
            
            conn.commit()
        
        try:
            yield conn
        finally:
            with conn.cursor() as cur:
                cur.execute("RESET ROLE")
                cur.execute("RESET app.current_tenant_id")
            conn.close()

# Usage
db = RLSDatabaseManager("postgresql://user:pass@db-host/banking")

with db.tenant_connection(tenant_id=1, user_role='tenant_user') as conn:
    with conn.cursor() as cur:
        # RLS automatically filters to tenant 1
        cur.execute("SELECT * FROM customers")
        customers = cur.fetchall()
        # Only returns customers where tenant_id = 1
```

## RLS Performance Considerations

```sql
-- RLS adds a filter to every query
-- Ensure the filter column is indexed

-- Index on the RLS filter column
CREATE INDEX idx_customers_tenant ON customers (tenant_id);

-- Check if RLS is affecting query plans
EXPLAIN SELECT * FROM customers WHERE customer_id = 1001;
-- Look for "Filter: (tenant_id = current_setting(...))" in the plan

-- RLS with partitioning: RLS policy applied per partition
-- Can be efficient if tenant_id is the partition key

-- Disable RLS bypass for superuser (security hardening)
ALTER TABLE customers FORCE ROW LEVEL SECURITY;
-- Now even table owner is subject to RLS policies
```

## Cross-References

- **Multi-Tenant Patterns**: See [multi-tenant-db-patterns.md](multi-tenant-db-patterns.md) for architecture
- **RBAC**: See [rbac.md](../kubernetes-openshift/rbac.md) for role-based access in K8s

## Interview Questions

1. **What is Row-Level Security and when should you use it?**
2. **How do you implement multi-tenant isolation using RLS?**
3. **What is the performance impact of RLS? How do you mitigate it?**
4. **How do you handle RLS with application connection pooling (PgBouncer)?**
5. **Can RLS be bypassed? How do you prevent this?**
6. **How would you implement role-based RLS (different policies for different roles)?**

## Checklist: RLS Implementation

- [ ] RLS enabled on all multi-tenant tables
- [ ] Policies defined for SELECT, INSERT, UPDATE, DELETE
- [ ] Filter columns indexed for performance
- [ ] Context set at connection level (not per-query)
- [ ] Superuser bypass considered and controlled
- [ ] RLS policies tested with different roles
- [ ] Query plans verified for efficient RLS filtering
- [ ] Application code doesn't rely on RLS for business logic
- [ ] Audit logging captures RLS-filtered access
