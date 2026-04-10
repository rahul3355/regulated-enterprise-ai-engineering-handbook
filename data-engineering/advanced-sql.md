# Advanced SQL for Banking Data Engineering

## Overview

Building on SQL fundamentals, advanced SQL techniques enable complex analytical queries, hierarchical data processing, and recursive computations that are essential for banking data pipelines. This guide covers Common Table Expressions (CTEs), recursive queries, analytical patterns, and query composition strategies used in production banking systems.

## Common Table Expressions (CTEs)

CTEs improve query readability by breaking complex logic into named subqueries. They also allow referencing the same subquery multiple times without duplication.

### Basic CTE Usage

```sql
-- Without CTE: Nested subqueries (hard to read)
SELECT 
    customer_id,
    total_spend,
    avg_spend
FROM (
    SELECT 
        customer_id,
        SUM(amount) AS total_spend
    FROM (
        SELECT * FROM transactions 
        WHERE transaction_time >= CURRENT_DATE - INTERVAL '30 days'
    ) recent
    GROUP BY customer_id
) totals
JOIN (
    SELECT 
        customer_id,
        AVG(amount) AS avg_spend
    FROM transactions
    WHERE transaction_time >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY customer_id
) averages USING (customer_id);

-- With CTE: Clear, modular, maintainable
WITH recent_transactions AS (
    SELECT *
    FROM transactions
    WHERE transaction_time >= CURRENT_DATE - INTERVAL '30 days'
),
customer_totals AS (
    SELECT 
        customer_id,
        SUM(amount) AS total_spend,
        COUNT(*) AS txn_count
    FROM recent_transactions
    GROUP BY customer_id
),
customer_averages AS (
    SELECT 
        customer_id,
        AVG(amount) AS avg_spend
    FROM recent_transactions
    GROUP BY customer_id
)
SELECT 
    ct.customer_id,
    ct.total_spend,
    ct.txn_count,
    ca.avg_spend
FROM customer_totals ct
JOIN customer_averages ca USING (customer_id)
ORDER BY ct.total_spend DESC;
```

### CTEs for Pipeline Transformations

```sql
-- dbt-style CTE pattern for banking data transformation
WITH source_transactions AS (
    SELECT * FROM raw_banking_transactions
),
filtered_valid AS (
    SELECT *
    FROM source_transactions
    WHERE status != 'REVERSED'
      AND amount IS NOT NULL
      AND amount > 0
),
enriched_with_customer AS (
    SELECT 
        t.*,
        c.customer_segment,
        c.risk_rating,
        c.account_opened_date
    FROM filtered_valid t
    JOIN dim_customers c ON t.customer_id = c.customer_id
),
calculated_metrics AS (
    SELECT 
        *,
        CASE 
            WHEN amount > 10000 THEN 'HIGH_VALUE'
            WHEN amount > 1000 THEN 'MEDIUM_VALUE'
            ELSE 'LOW_VALUE'
        END AS transaction_tier,
        CASE 
            WHEN risk_rating = 'HIGH' AND amount > 5000 THEN 'FLAGGED'
            ELSE 'CLEAR'
        END AS compliance_flag
    FROM enriched_with_customer
),
final_output AS (
    SELECT 
        transaction_id,
        customer_id,
        customer_segment,
        transaction_type,
        amount,
        currency,
        transaction_time,
        merchant_name,
        transaction_tier,
        compliance_flag,
        CURRENT_TIMESTAMP AS processed_at,
        'transactions_daily_v2' AS pipeline_version
    FROM calculated_metrics
)
SELECT * FROM final_output;
```

## Recursive CTEs

Recursive CTEs enable queries on hierarchical and graph data structures. They are essential for organizational hierarchies, account trees, and transaction chains.

### Organizational Hierarchy

```sql
-- Employee reporting chain
WITH RECURSIVE org_hierarchy AS (
    -- Anchor: Start with top-level managers
    SELECT 
        employee_id,
        manager_id,
        first_name,
        last_name,
        title,
        1 AS level,
        ARRAY[first_name || ' ' || last_name] AS reporting_chain
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: Add direct reports
    SELECT 
        e.employee_id,
        e.manager_id,
        e.first_name,
        e.last_name,
        e.title,
        oh.level + 1,
        oh.reporting_chain || ARRAY[e.first_name || ' ' || e.last_name]
    FROM employees e
    INNER JOIN org_hierarchy oh ON e.manager_id = oh.employee_id
)
SELECT 
    employee_id,
    first_name || ' ' || last_name AS employee_name,
    title,
    level AS depth_in_organization,
    array_length(reporting_chain, 1) AS chain_length,
    reporting_chain
FROM org_hierarchy
ORDER BY level, employee_id;
```

### Account Ownership Chain

```sql
-- Trace beneficial ownership through corporate account structures
WITH RECURSIVE ownership_chain AS (
    -- Anchor: Direct account holders
    SELECT 
        account_id,
        holder_id AS entity_id,
        holder_id AS ultimate_entity,
        ownership_percentage,
        1 AS depth,
        ARRAY[holder_id] AS ownership_path
    FROM account_holders
    WHERE ownership_percentage >= 25.0
    
    UNION ALL
    
    -- Recursive: Follow corporate ownership
    SELECT 
        oc.account_id,
        e.shareholder_id AS entity_id,
        oc.ultimate_entity,
        ROUND(oc.ownership_percentage * e.ownership_pct / 100.0, 4) AS ownership_percentage,
        oc.depth + 1,
        oc.ownership_path || ARRAY[e.shareholder_id]
    FROM ownership_chain oc
    JOIN entity_ownership e ON oc.entity_id = e.entity_id
    WHERE oc.depth < 10  -- Prevent infinite recursion
      AND oc.ownership_percentage * e.ownership_pct / 100.0 >= 10.0  -- Material threshold
)
SELECT DISTINCT
    account_id,
    ultimate_entity AS beneficial_owner_id,
    ownership_percentage,
    depth,
    ownership_path
FROM ownership_chain
WHERE depth = (
    SELECT MAX(depth) 
    FROM ownership_chain oc2 
    WHERE oc2.account_id = ownership_chain.account_id
);
```

### Transaction Chain Analysis (Money Flow)

```sql
-- Trace money flow through transaction chains (AML investigation)
WITH RECURSIVE money_flow AS (
    -- Anchor: Starting transaction (flagged by compliance)
    SELECT 
        transaction_id,
        source_account,
        destination_account,
        amount,
        transaction_time,
        1 AS hop_count,
        ARRAY[transaction_id] AS chain,
        amount AS original_amount
    FROM transactions
    WHERE compliance_flag = 'SUSPICIOUS'
      AND transaction_time >= CURRENT_DATE - INTERVAL '7 days'
    
    UNION ALL
    
    -- Recursive: Follow outgoing transactions from destination
    SELECT 
        t.transaction_id,
        t.source_account,
        t.destination_account,
        t.amount,
        t.transaction_time,
        mf.hop_count + 1,
        mf.chain || ARRAY[t.transaction_id],
        mf.original_amount
    FROM money_flow mf
    JOIN transactions t ON mf.destination_account = t.source_account
    WHERE t.transaction_time > mf.transaction_time
      AND t.transaction_time < mf.transaction_time + INTERVAL '24 hours'
      AND t.transaction_id != ALL(mf.chain)  -- Prevent cycles
      AND mf.hop_count < 10
)
SELECT 
    chain AS transaction_chain,
    hop_count AS chain_length,
    original_amount AS initial_amount,
    array_agg(DISTINCT destination_account) AS final_destinations
FROM money_flow
GROUP BY chain, hop_count, original_amount
HAVING hop_count >= 3  -- Chains with 3+ hops
ORDER BY hop_count DESC, original_amount DESC;
```

## Analytical Query Patterns

### Funnel Analysis

```sql
-- Customer onboarding funnel
WITH funnel_stages AS (
    SELECT 
        customer_id,
        MAX(CASE WHEN event = 'registration_started' THEN event_time END) AS stage_1,
        MAX(CASE WHEN event = 'identity_verified' THEN event_time END) AS stage_2,
        MAX(CASE WHEN event = 'documents_uploaded' THEN event_time END) AS stage_3,
        MAX(CASE WHEN event = 'account_funded' THEN event_time END) AS stage_4,
        MAX(CASE WHEN event = 'first_transaction' THEN event_time END) AS stage_5
    FROM customer_events
    WHERE event_time >= CURRENT_DATE - INTERVAL '90 days'
    GROUP BY customer_id
)
SELECT 
    'Registration Started' AS stage,
    COUNT(*) AS users,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct_of_total
FROM funnel_stages WHERE stage_1 IS NOT NULL
UNION ALL
SELECT 
    'Identity Verified' AS stage,
    COUNT(*) AS users,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM funnel_stages WHERE stage_1 IS NOT NULL), 1)
FROM funnel_stages WHERE stage_2 IS NOT NULL
UNION ALL
SELECT 
    'Documents Uploaded' AS stage,
    COUNT(*) AS users,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM funnel_stages WHERE stage_1 IS NOT NULL), 1)
FROM funnel_stages WHERE stage_3 IS NOT NULL
UNION ALL
SELECT 
    'Account Funded' AS stage,
    COUNT(*) AS users,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM funnel_stages WHERE stage_1 IS NOT NULL), 1)
FROM funnel_stages WHERE stage_4 IS NOT NULL
UNION ALL
SELECT 
    'First Transaction' AS stage,
    COUNT(*) AS users,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM funnel_stages WHERE stage_1 IS NOT NULL), 1)
FROM funnel_stages WHERE stage_5 IS NOT NULL;
```

### Cohort Analysis

```sql
-- Monthly retention cohort analysis
WITH customer_cohorts AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', account_opened_date) AS cohort_month
    FROM customers
),
monthly_activity AS (
    SELECT 
        c.customer_id,
        c.cohort_month,
        DATE_TRUNC('month', t.transaction_time) AS activity_month,
        EXTRACT(MONTH FROM 
            AGE(DATE_TRUNC('month', t.transaction_time), c.cohort_month)
        )::INTEGER AS months_since_signup
    FROM customer_cohorts c
    JOIN transactions t ON c.customer_id = t.customer_id
),
cohort_retention AS (
    SELECT 
        cohort_month,
        months_since_signup,
        COUNT(DISTINCT customer_id) AS active_customers
    FROM monthly_activity
    GROUP BY cohort_month, months_since_signup
)
SELECT 
    cohort_month,
    months_since_signup,
    active_customers,
    FIRST_VALUE(active_customers) OVER (
        PARTITION BY cohort_month 
        ORDER BY months_since_signup
    ) AS cohort_size,
    ROUND(
        100.0 * active_customers / FIRST_VALUE(active_customers) OVER (
            PARTITION BY cohort_month 
            ORDER BY months_since_signup
        ), 1
    ) AS retention_rate
FROM cohort_retention
ORDER BY cohort_month, months_since_signup;
```

### Gap-and-Island Detection

```sql
-- Identify gaps in customer transaction activity (dormancy detection)
WITH daily_activity AS (
    SELECT 
        customer_id,
        DATE_TRUNC('day', transaction_time) AS activity_date,
        COUNT(*) AS txn_count,
        SUM(amount) AS daily_amount
    FROM transactions
    WHERE transaction_time >= CURRENT_DATE - INTERVAL '90 days'
    GROUP BY customer_id, DATE_TRUNC('day', transaction_time)
),
with_groups AS (
    SELECT 
        customer_id,
        activity_date,
        txn_count,
        daily_amount,
        activity_date - (ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY activity_date
        ))::INTEGER AS grp
    FROM daily_activity
),
islands AS (
    SELECT 
        customer_id,
        MIN(activity_date) AS period_start,
        MAX(activity_date) AS period_end,
        COUNT(*) AS active_days,
        SUM(txn_count) AS total_txns,
        SUM(daily_amount) AS total_amount
    FROM with_groups
    GROUP BY customer_id, grp
)
SELECT 
    customer_id,
    period_start,
    period_end,
    active_days,
    total_txns,
    total_amount,
    -- Calculate gap to next active period
    LEAD(period_start) OVER (
        PARTITION BY customer_id 
        ORDER BY period_start
    ) - period_end AS gap_days
FROM islands
WHERE gap_days > 7 OR gap_days IS NULL  -- Gaps larger than 7 days
ORDER BY customer_id, period_start;
```

## Performance Optimization Tips

### When to Use CTEs

```sql
-- Good: CTE referenced multiple times
WITH expensive_calculation AS (
    -- This runs ONCE and is cached
    SELECT customer_id, complex_function(data) AS result
    FROM large_table
)
SELECT * FROM expensive_calculation WHERE result > 100
UNION ALL
SELECT * FROM expensive_calculation WHERE result < -100;

-- Caution: CTE as optimization fence in older Postgres
-- In Postgres 12+, CTEs are inlined by default unless RECURSIVE
-- Use MATERIALIZED to force evaluation, or NOT MATERIALIZED to inline
WITH customer_data AS NOT MATERIALIZED (
    SELECT * FROM customers WHERE status = 'ACTIVE'
)
SELECT * FROM customer_data JOIN accounts USING (customer_id);
```

### Lateral Joins for Complex Correlated Queries

```sql
-- Get the top 3 transactions per customer
SELECT c.customer_id, c.first_name, t.*
FROM customers c
CROSS JOIN LATERAL (
    SELECT *
    FROM transactions
    WHERE customer_id = c.customer_id
    ORDER BY amount DESC
    LIMIT 3
) t
WHERE c.customer_segment = 'PREMIUM';
```

## Cross-References

- **SQL Fundamentals**: See [sql-fundamentals.md](sql-fundamentals.md) for basic SQL
- **Query Optimization**: See [query-optimization.md](query-optimization.md) for EXPLAIN plans
- **Banking Data Architecture**: See [banking-data-architecture.md](banking-data-architecture.md) for data models

## Interview Questions

1. **When does a CTE become a performance liability? How do you mitigate it?**
2. **Write a recursive CTE to find all managers above an employee in a 10-level hierarchy.**
3. **How would you detect circular money flows in a transaction graph?**
4. **Explain the difference between CTEs and temporary tables. When would you use each?**
5. **Design a query to calculate customer lifetime value using cohort-based retention.**
6. **How do you handle cycles in recursive CTEs? What happens if you don't?**

## Checklist: Advanced SQL Best Practices

- [ ] Use CTEs to break complex queries into readable, testable chunks
- [ ] Apply `NOT MATERIALIZED` hint when CTE should be inlined (Postgres 12+)
- [ ] Set depth limits in recursive CTEs to prevent infinite loops
- [ ] Use cycle detection (`array[column]`) in recursive queries
- [ ] Prefer lateral joins over correlated subqueries for top-N-per-group
- [ ] Validate recursive results against known anchor points
- [ ] Document the base case and recursive case in recursive CTEs
- [ ] Test recursive queries with edge cases (empty sets, single rows, cycles)
