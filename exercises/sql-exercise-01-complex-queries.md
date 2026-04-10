# SQL Exercise 01: Complex Queries

> Master window functions, CTEs, and analytical queries — essential for banking data analysis and GenAI analytics.

## Problem Statement

You are a data engineer on the GenAI platform team. The analytics team needs a report on GenAI platform usage with the following metrics:

1. Monthly active users by department
2. 7-day moving average of daily queries
3. Top 3 most active users per department each month
4. Month-over-month growth rate in query volume
5. P50 and P95 response latency by model
6. Users whose usage dropped more than 50% month-over-month (potential churn)

**Banking Context:** The bank needs to track GenAI platform adoption, identify power users, detect declining usage, and monitor performance across models. This data informs capacity planning, budget allocation, and user engagement strategies.

## Schema

```sql
CREATE TABLE users (
    user_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    department VARCHAR(100) NOT NULL,
    role VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE queries (
    query_id UUID PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL REFERENCES users(user_id),
    query_text TEXT NOT NULL,
    response_text TEXT NOT NULL,
    model VARCHAR(50) NOT NULL,
    tokens_prompt INT NOT NULL,
    tokens_completion INT NOT NULL,
    latency_ms INT NOT NULL,
    status VARCHAR(20) NOT NULL,  -- 'success', 'error', 'blocked'
    created_at TIMESTAMP NOT NULL
);
```

## Constraints

- Use PostgreSQL syntax
- Use CTEs (Common Table Expressions) for query organization
- Use window functions for running totals, rankings, and growth rates
- Use `PERCENTILE_CONT` for latency percentiles
- All queries should handle the case where departments have no queries in a period

## Tasks

### Task 1: Monthly Active Users by Department

```sql
-- For each department and month, count distinct active users
-- Include months where a department had zero users (generate series)
-- Order by department, month
```

### Task 2: 7-Day Moving Average of Daily Queries

```sql
-- For each day, calculate the average number of queries
-- over the preceding 7 days (including current day)
-- Use a window function with ROWS BETWEEN
```

### Task 3: Top 3 Users Per Department Per Month

```sql
-- Rank users within each department by query count per month
-- Return only the top 3 per department
-- Use DENSE_RANK() to handle ties
```

### Task 4: Month-over-Month Growth Rate

```sql
-- For each month, calculate total query volume
-- Calculate the percentage change from the previous month
-- Use LAG() to get the previous month's value
```

### Task 5: Latency Percentiles by Model

```sql
-- Calculate P50 and P95 latency for each model
-- Use PERCENTILE_CONT(0.5) and PERCENTILE_CONT(0.95)
-- Only include successful queries (status = 'success')
```

### Task 6: Usage Drop Detection

```sql
-- Find users whose monthly query count dropped > 50%
-- compared to their previous month
-- Include their department and the drop percentage
```

## Hints

### Hint 1: Generating a Date Series

```sql
-- Generate all months in a range
SELECT generate_series(
    '2025-01-01'::date,
    '2026-03-01'::date,
    '1 month'::interval
) AS month;
```

### Hint 2: Window Function for Moving Average

```sql
AVG(daily_count) OVER (
    ORDER BY query_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
) AS moving_avg_7d
```

### Hint 3: Ranking Within Groups

```sql
DENSE_RANK() OVER (
    PARTITION BY department, month
    ORDER BY query_count DESC
) AS dept_rank
```

### Hint 4: Month-over-Month Growth

```sql
LAG(total_queries) OVER (ORDER BY month) AS prev_month_queries,
ROUND(
    100.0 * (total_queries - LAG(total_queries) OVER (ORDER BY month))
    / NULLIF(LAG(total_queries) OVER (ORDER BY month), 0),
    1
) AS growth_pct
```

## Example Solutions

```sql
-- Task 1: Monthly Active Users by Department
WITH monthly_users AS (
    SELECT
        u.department,
        DATE_TRUNC('month', q.created_at) AS month,
        COUNT(DISTINCT q.user_id) AS active_users
    FROM users u
    JOIN queries q ON u.user_id = q.user_id
    GROUP BY u.department, DATE_TRUNC('month', q.created_at)
)
SELECT
    department,
    month,
    active_users,
    SUM(active_users) OVER (PARTITION BY department ORDER BY month) AS cumulative_users
FROM monthly_users
ORDER BY department, month;

-- Task 2: 7-Day Moving Average
WITH daily_counts AS (
    SELECT
        DATE(created_at) AS query_date,
        COUNT(*) AS daily_count
    FROM queries
    WHERE status = 'success'
    GROUP BY DATE(created_at)
)
SELECT
    query_date,
    daily_count,
    ROUND(AVG(daily_count) OVER (
        ORDER BY query_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 1) AS moving_avg_7d
FROM daily_counts
ORDER BY query_date;

-- Task 3: Top 3 Users Per Department Per Month
WITH user_monthly AS (
    SELECT
        u.department,
        u.name,
        DATE_TRUNC('month', q.created_at) AS month,
        COUNT(*) AS query_count
    FROM users u
    JOIN queries q ON u.user_id = q.user_id
    GROUP BY u.department, u.name, DATE_TRUNC('month', q.created_at)
),
ranked AS (
    SELECT
        department,
        name,
        month,
        query_count,
        DENSE_RANK() OVER (
            PARTITION BY department, month
            ORDER BY query_count DESC
        ) AS rk
    FROM user_monthly
)
SELECT department, name, month, query_count
FROM ranked
WHERE rk <= 3
ORDER BY department, month, rk;

-- Task 4: Month-over-Month Growth Rate
WITH monthly_totals AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        COUNT(*) AS total_queries
    FROM queries
    GROUP BY DATE_TRUNC('month', created_at)
)
SELECT
    month,
    total_queries,
    LAG(total_queries) OVER (ORDER BY month) AS prev_month_queries,
    ROUND(
        100.0 * (total_queries - LAG(total_queries) OVER (ORDER BY month))
        / NULLIF(LAG(total_queries) OVER (ORDER BY month), 0),
        1
    ) AS growth_pct
FROM monthly_totals
ORDER BY month;

-- Task 5: Latency Percentiles by Model
SELECT
    model,
    COUNT(*) AS total_queries,
    ROUND(AVG(latency_ms), 1) AS avg_latency_ms,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY latency_ms)::numeric, 1) AS p50_latency_ms,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms)::numeric, 1) AS p95_latency_ms
FROM queries
WHERE status = 'success'
GROUP BY model
ORDER BY avg_latency_ms DESC;

-- Task 6: Usage Drop Detection
WITH monthly_counts AS (
    SELECT
        user_id,
        DATE_TRUNC('month', created_at) AS month,
        COUNT(*) AS query_count
    FROM queries
    GROUP BY user_id, DATE_TRUNC('month', created_at)
),
with_prev AS (
    SELECT
        user_id,
        month,
        query_count,
        LAG(query_count) OVER (PARTITION BY user_id ORDER BY month) AS prev_count
    FROM monthly_counts
)
SELECT
    wp.user_id,
    u.name,
    u.department,
    wp.month,
    wp.query_count,
    wp.prev_count,
    ROUND(
        100.0 * (wp.prev_count - wp.query_count) / wp.prev_count,
        1
    ) AS drop_pct
FROM with_prev wp
JOIN users u ON wp.user_id = u.user_id
WHERE wp.prev_count IS NOT NULL
  AND 100.0 * (wp.prev_count - wp.query_count) / wp.prev_count > 50
ORDER BY drop_pct DESC;
```

## Extensions

1. **Add cost analysis:** If each token costs $0.00002 (prompt) and $0.00006 (completion), calculate monthly cost by department.

2. **Session analysis:** Group queries into sessions (queries within 30 minutes of each other by the same user). Calculate average session length and queries per session.

3. **Cohort analysis:** Group users by their first month of activity. Track retention — what percentage of each cohort is still active in subsequent months?

4. **Query complexity scoring:** Classify queries as simple (1-50 tokens), medium (51-200), or complex (200+). Track the distribution over time.

5. **Anomaly detection:** Using window functions, identify days where query volume is more than 2 standard deviations from the 30-day rolling average.

## Interview Relevance

SQL analytics is tested in almost every banking engineering interview:

| Skill | Why It Matters |
|-------|---------------|
| CTEs | Query organization, readability |
| Window functions | Running totals, rankings, growth rates |
| Percentile calculations | SLO reporting |
| Date manipulation | Time-series analytics |
| Self-joins / LAG | Period-over-period comparison |

**Follow-up questions:**
- "What's the difference between RANK(), DENSE_RANK(), and ROW_NUMBER()?"
- "When would you use a CTE vs. a subquery?"
- "How would you optimize the top-3-per-department query?"
- "What's the performance impact of window functions on large datasets?"
