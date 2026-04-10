# SQL Exercises (10) with Solutions

## Exercise 1: Find Duplicate Accounts

**Problem**: Find customers who have multiple accounts.

```sql
-- Schema: accounts(customer_id, account_id, account_type, balance, created_at)

SELECT customer_id, COUNT(*) as account_count
FROM accounts
GROUP BY customer_id
HAVING COUNT(*) > 1
ORDER BY account_count DESC;
```

## Exercise 2: Top 3 Customers by Total Balance

**Problem**: Find the top 3 customers by their total balance across all accounts.

```sql
-- Schema: customers(customer_id, name, email)
-- Schema: accounts(customer_id, account_id, account_type, balance)

SELECT c.customer_id, c.name, SUM(a.balance) as total_balance
FROM customers c
JOIN accounts a ON c.customer_id = a.customer_id
GROUP BY c.customer_id, c.name
ORDER BY total_balance DESC
LIMIT 3;
```

## Exercise 3: Monthly Transaction Summary

**Problem**: Generate a monthly summary of transaction count and total amount.

```sql
-- Schema: transactions(transaction_id, account_id, amount, type, created_at)

SELECT 
    DATE_TRUNC('month', created_at) as month,
    type,
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount
FROM transactions
WHERE created_at >= '2024-01-01'
GROUP BY DATE_TRUNC('month', created_at), type
ORDER BY month, type;
```

## Exercise 4: Accounts with No Transactions in 90 Days

**Problem**: Find accounts that have had no transactions in the last 90 days.

```sql
SELECT a.account_id, a.customer_id, a.balance
FROM accounts a
LEFT JOIN transactions t ON a.account_id = t.account_id 
    AND t.created_at >= CURRENT_DATE - INTERVAL '90 days'
WHERE t.transaction_id IS NULL;
```

## Exercise 5: Running Total of Daily Deposits

**Problem**: Calculate the running total of daily deposits.

```sql
SELECT 
    DATE(created_at) as transaction_date,
    SUM(amount) as daily_deposits,
    SUM(SUM(amount)) OVER (ORDER BY DATE(created_at)) as running_total
FROM transactions
WHERE type = 'deposit'
GROUP BY DATE(created_at)
ORDER BY transaction_date;
```

## Exercise 6: Customers Who Opened Accounts This Month

**Problem**: Find customers who opened their first account this month.

```sql
SELECT c.customer_id, c.name, c.email, a.account_id, a.account_type
FROM customers c
JOIN accounts a ON c.customer_id = a.customer_id
WHERE a.created_at >= DATE_TRUNC('month', CURRENT_DATE)
  AND c.customer_id NOT IN (
      SELECT customer_id FROM accounts 
      WHERE created_at < DATE_TRUNC('month', CURRENT_DATE)
  );
```

## Exercise 7: Average Time Between Transactions

**Problem**: Calculate the average time (in days) between consecutive transactions for each account.

```sql
WITH transaction_lags AS (
    SELECT 
        account_id,
        created_at,
        LAG(created_at) OVER (PARTITION BY account_id ORDER BY created_at) as prev_transaction
    FROM transactions
)
SELECT 
    account_id,
    AVG(EXTRACT(EPOCH FROM (created_at - prev_transaction)) / 86400) as avg_days_between
FROM transaction_lags
WHERE prev_transaction IS NOT NULL
GROUP BY account_id
HAVING COUNT(*) > 1;
```

## Exercise 8: Revenue by Product Line

**Problem**: Calculate revenue by product line, including products with zero revenue.

```sql
-- Schema: products(product_id, product_name, product_line)
-- Schema: sales(sale_id, product_id, amount, sale_date)

SELECT 
    p.product_line,
    COUNT(DISTINCT p.product_id) as product_count,
    COALESCE(SUM(s.amount), 0) as total_revenue,
    COUNT(s.sale_id) as sale_count
FROM products p
LEFT JOIN sales s ON p.product_id = s.product_id
GROUP BY p.product_line
ORDER BY total_revenue DESC;
```

## Exercise 9: Detect Unusual Transactions

**Problem**: Flag transactions that are more than 3 standard deviations above the account's average transaction amount.

```sql
WITH account_stats AS (
    SELECT 
        account_id,
        AVG(ABS(amount)) as avg_amount,
        STDDEV(ABS(amount)) as stddev_amount
    FROM transactions
    GROUP BY account_id
)
SELECT 
    t.transaction_id,
    t.account_id,
    t.amount,
    s.avg_amount,
    s.stddev_amount,
    ROUND((ABS(t.amount) - s.avg_amount) / NULLIF(s.stddev_amount, 0), 2) as z_score
FROM transactions t
JOIN account_stats s ON t.account_id = s.account_id
WHERE s.stddev_amount > 0
  AND (ABS(t.amount) - s.avg_amount) / s.stddev_amount > 3
ORDER BY z_score DESC;
```

## Exercise 10: Hierarchical Employee Report

**Problem**: Display the employee hierarchy with their level in the organization.

```sql
-- Schema: employees(employee_id, name, manager_id, department)

WITH RECURSIVE employee_hierarchy AS (
    -- Base case: employees with no manager (top level)
    SELECT 
        employee_id, name, manager_id, department,
        1 as level,
        name as reporting_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees reporting to someone
    SELECT 
        e.employee_id, e.name, e.manager_id, e.department,
        eh.level + 1,
        eh.reporting_path || ' -> ' || e.name
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT 
    employee_id,
    name,
    department,
    level,
    reporting_path
FROM employee_hierarchy
ORDER BY level, name;
```
