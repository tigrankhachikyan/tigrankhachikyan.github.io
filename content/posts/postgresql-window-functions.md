---
title: "PostgreSQL Window Functions Cheat Sheet"
date: 2025-07-29T11:30:30Z
tags:
- postgresql
- sql
- window_functions
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: A comprehensive reference with practical examples and expected results for PostgreSQL window functions.
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
  image: "<image path/url>"
  alt: "<alt text>"
  caption: "<text>"
  relative: false
  hidden: true

---

# 🧠 PostgreSQL Window Functions Cheat Sheet

Window functions perform calculations across rows related to the current row **without collapsing results** like `GROUP BY` does. They're essential for rankings, running totals, and comparative analysis.

---

## 📌 Basic Syntax

```sql
<function>(<expression>) OVER (
  [PARTITION BY <columns>]     -- Groups rows
  [ORDER BY <columns>]         -- Defines sequence
  [ROWS|RANGE BETWEEN ...]     -- Limits calculation frame
)
```

**Key Concepts:**
- **Partition**: Resets calculations for each group
- **Order**: Determines row sequence within partition  
- **Frame**: Defines which rows to include in calculation

---

## 🎯 Sample Data Setup

```sql
-- Employee table for examples
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  department VARCHAR(30),
  salary INTEGER,
  hire_date DATE
);

INSERT INTO employees VALUES
(1, 'Alice', 'Engineering', 90000, '2020-01-15'),
(2, 'Bob', 'Engineering', 85000, '2020-03-20'),
(3, 'Carol', 'Sales', 70000, '2020-06-10'),
(4, 'David', 'Engineering', 95000, '2021-01-05'),
(5, 'Eve', 'Sales', 75000, '2021-04-12'),
(6, 'Frank', 'Marketing', 65000, '2021-07-08');

-- Sales table for time-series examples
CREATE TABLE sales (
  id SERIAL PRIMARY KEY,
  sale_date DATE,
  amount DECIMAL(10,2),
  customer_id INTEGER
);

INSERT INTO sales VALUES
(1, '2023-01-01', 1000.00, 101),
(2, '2023-01-02', 1500.00, 102),
(3, '2023-01-03', 2000.00, 101),
(4, '2023-01-04', 1200.00, 103),
(5, '2023-01-05', 1800.00, 102);
```

---

## 🏆 Ranking Functions

### ROW_NUMBER() - Sequential numbering

```sql
-- Assign unique sequential numbers
SELECT name, department, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC) as overall_rank,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as dept_rank
FROM employees;
```

**Key Points:** 
- `ROW_NUMBER()` always gives unique numbers (1,2,3,4...)
- `PARTITION BY department` restarts numbering for each department
- `ORDER BY salary DESC` determines ranking order (highest first)

**Result:**
```
name  | department  | salary | overall_rank | dept_rank
------|-------------|--------|--------------|----------
David | Engineering | 95000  | 1            | 1
Alice | Engineering | 90000  | 2            | 2  
Bob   | Engineering | 85000  | 3            | 3
Eve   | Sales       | 75000  | 4            | 1
Carol | Sales       | 70000  | 5            | 2
Frank | Marketing   | 65000  | 6            | 1
```

### RANK() vs DENSE_RANK() - Handle ties differently

```sql
-- Test with tied salaries
UPDATE employees SET salary = 75000 WHERE name = 'Carol';

SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) as rank_with_gaps,
       DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank,
       ROW_NUMBER() OVER (ORDER BY salary DESC, name) as row_num
FROM employees;
```

**Key Points:**
- `RANK()` leaves gaps after ties (1,2,2,4)
- `DENSE_RANK()` continues without gaps (1,2,2,3)
- `ROW_NUMBER()` uses tiebreaker to ensure uniqueness

**Result:**
```
name  | salary | rank_with_gaps | dense_rank | row_num
------|--------|----------------|------------|--------
David | 95000  | 1              | 1          | 1
Alice | 90000  | 2              | 2          | 2
Bob   | 85000  | 3              | 3          | 3
Carol | 75000  | 4              | 4          | 4
Eve   | 75000  | 4              | 4          | 5  ← tied salary
Frank | 65000  | 6              | 5          | 6  ← gap vs no gap
```

### NTILE() - Split into buckets

```sql
-- Divide employees into 3 salary brackets
SELECT name, salary,
       NTILE(3) OVER (ORDER BY salary DESC) as salary_bracket
FROM employees;
```

**Key Points:**
- Divides rows into N approximately equal groups
- Higher buckets get extra rows if division isn't even
- Useful for percentile analysis and bucketing

**Result:**
```
name  | salary | salary_bracket | explanation
------|--------|----------------|------------
David | 95000  | 1              | Top bracket (high)
Alice | 90000  | 1              | Top bracket (high)
Bob   | 85000  | 2              | Middle bracket 
Eve   | 75000  | 2              | Middle bracket
Carol | 75000  | 3              | Bottom bracket (low)
Frank | 65000  | 3              | Bottom bracket (low)
```

---

## 📊 Aggregate Window Functions

### Running Totals

```sql
-- Running sum of sales
SELECT sale_date, amount,
       SUM(amount) OVER (ORDER BY sale_date) as running_total
FROM sales;
```

**Key Points:**
- Default frame: `RANGE UNBOUNDED PRECEDING` - includes all previous rows
- `ORDER BY` is crucial for meaningful running totals
- Each row shows cumulative total up to that point

**Result:**
```
sale_date  | amount  | running_total | explanation
-----------|---------|---------------|-------------
2023-01-01 | 1000.00 | 1000.00      | 1000
2023-01-02 | 1500.00 | 2500.00      | 1000 + 1500
2023-01-03 | 2000.00 | 4500.00      | 1000 + 1500 + 2000  
2023-01-04 | 1200.00 | 5700.00      | 1000 + 1500 + 2000 + 1200
2023-01-05 | 1800.00 | 7500.00      | All amounts summed
```

### Moving Averages

```sql
-- 3-day moving average
SELECT sale_date, amount,
       AVG(amount) OVER (
         ORDER BY sale_date 
         ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) as moving_avg_3day
FROM sales;
```

**Key Points:**
- `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` creates sliding window
- Window size changes for first few rows (1, then 2, then 3 rows)
- Useful for smoothing data trends

**Result:**
```
sale_date  | amount  | moving_avg_3day | calculation
-----------|---------|-----------------|-------------
2023-01-01 | 1000.00 | 1000.00        | 1000/1 (only current)
2023-01-02 | 1500.00 | 1250.00        | (1000+1500)/2
2023-01-03 | 2000.00 | 1500.00        | (1000+1500+2000)/3
2023-01-04 | 1200.00 | 1566.67        | (1500+2000+1200)/3
2023-01-05 | 1800.00 | 1666.67        | (2000+1200+1800)/3
```

### Percentage of Total

```sql
-- Each employee's salary as percentage of department total
SELECT name, department, salary,
       SUM(salary) OVER (PARTITION BY department) as dept_total,
       ROUND(100.0 * salary / SUM(salary) OVER (PARTITION BY department), 2) as pct_of_dept_total
FROM employees;
```

**Key Points:**
- `SUM() OVER (PARTITION BY department)` calculates department total for each row
- Division shows individual contribution to group
- Percentages within each department sum to 100%

**Result:**
```
name  | department  | salary | dept_total | pct_of_dept_total
------|-------------|--------|------------|------------------
Alice | Engineering | 90000  | 270000     | 33.33
Bob   | Engineering | 85000  | 270000     | 31.48  
David | Engineering | 95000  | 270000     | 35.19
Carol | Sales       | 75000  | 150000     | 50.00
Eve   | Sales       | 75000  | 150000     | 50.00
Frank | Marketing   | 65000  | 65000      | 100.00
```

---

## 🔄 Offset Functions (LAG/LEAD)

### Compare with Previous/Next Row

```sql
-- Compare current salary with previous hire
SELECT name, hire_date, salary,
       LAG(salary) OVER (ORDER BY hire_date) as prev_hire_salary,
       salary - LAG(salary) OVER (ORDER BY hire_date) as salary_diff,
       LEAD(salary) OVER (ORDER BY hire_date) as next_hire_salary
FROM employees
ORDER BY hire_date;
```

**Key Points:**
- `LAG()` looks at previous row, `LEAD()` looks at next row
- `ORDER BY hire_date` determines which row is "previous"
- First row has NULL for LAG, last row has NULL for LEAD

**Result:**
```
name  | hire_date  | salary | prev_hire_salary | salary_diff | next_hire_salary
------|------------|--------|------------------|-------------|------------------
Alice | 2020-01-15 | 90000  | NULL            | NULL        | 85000
Bob   | 2020-03-20 | 85000  | 90000           | -5000       | 75000
Carol | 2020-06-10 | 75000  | 85000           | -10000      | 95000  
David | 2021-01-05 | 95000  | 75000           | 20000       | 75000
Eve   | 2021-04-12 | 75000  | 95000           | -20000      | 65000
Frank | 2021-07-08 | 65000  | 75000           | -10000      | NULL
```

### Growth Rate Calculation

```sql
-- Month-over-month growth
SELECT sale_date, amount,
       LAG(amount) OVER (ORDER BY sale_date) as prev_amount,
       ROUND(
         100.0 * (amount - LAG(amount) OVER (ORDER BY sale_date)) / 
         NULLIF(LAG(amount) OVER (ORDER BY sale_date), 0), 2
       ) as growth_rate_pct
FROM sales;
```

**Key Points:**
- `NULLIF()` prevents division by zero errors
- Growth rate formula: (current - previous) / previous * 100
- First row always has NULL growth rate

**Result:**
```
sale_date  | amount  | prev_amount | growth_rate_pct | explanation
-----------|---------|-------------|-----------------|-------------
2023-01-01 | 1000.00 | NULL       | NULL            | No previous day
2023-01-02 | 1500.00 | 1000.00    | 50.00           | 50% increase
2023-01-03 | 2000.00 | 1500.00    | 33.33           | 33% increase  
2023-01-04 | 1200.00 | 2000.00    | -40.00          | 40% decrease
2023-01-05 | 1800.00 | 1200.00    | 50.00           | 50% increase
```

### Fill Missing Values

```sql
-- Forward fill using LAG with COALESCE
SELECT sale_date, 
       CASE WHEN sale_date = '2023-01-03' THEN NULL ELSE amount END as amount_with_gap,
       COALESCE(
         CASE WHEN sale_date = '2023-01-03' THEN NULL ELSE amount END,
         LAG(amount) OVER (ORDER BY sale_date)
       ) as filled_amount
FROM sales;
```

**Key Points:**
- `COALESCE()` returns first non-NULL value
- Forward fill uses previous valid value for missing data
- Common technique for handling sparse time series

**Result:**
```
sale_date  | amount_with_gap | filled_amount | explanation
-----------|-----------------|---------------|-------------
2023-01-01 | 1000.00        | 1000.00       | Original value
2023-01-02 | 1500.00        | 1500.00       | Original value
2023-01-03 | NULL           | 1500.00       | Filled from previous
2023-01-04 | 1200.00        | 1200.00       | Original value
2023-01-05 | 1800.00        | 1800.00       | Original value
```

---

## 🎯 Value Functions (FIRST_VALUE/LAST_VALUE)

### Compare to Benchmarks

```sql
-- Compare each salary to highest in department
SELECT name, department, salary,
       FIRST_VALUE(salary) OVER (
         PARTITION BY department 
         ORDER BY salary DESC
       ) as dept_max_salary,
       salary - FIRST_VALUE(salary) OVER (
         PARTITION BY department 
         ORDER BY salary DESC  
       ) as gap_from_max
FROM employees;
```

**Key Points:**
- `FIRST_VALUE()` with `ORDER BY salary DESC` gets the maximum
- `PARTITION BY department` finds max within each department
- Gap calculation shows how far each employee is from top performer

**Result:**
```
name  | department  | salary | dept_max_salary | gap_from_max
------|-------------|--------|-----------------|-------------
David | Engineering | 95000  | 95000          | 0         ← top performer
Alice | Engineering | 90000  | 95000          | -5000     ← $5k below max
Bob   | Engineering | 85000  | 95000          | -10000    ← $10k below max
Eve   | Sales       | 75000  | 75000          | 0         ← top performer  
Carol | Sales       | 75000  | 75000          | 0         ← tied for top
Frank | Marketing   | 65000  | 65000          | 0         ← only employee
```

### Track Changes Over Time

```sql
-- Show first and latest value for each customer
SELECT customer_id, sale_date, amount,
       FIRST_VALUE(amount) OVER (
         PARTITION BY customer_id 
         ORDER BY sale_date
       ) as first_purchase,
       LAST_VALUE(amount) OVER (
         PARTITION BY customer_id 
         ORDER BY sale_date
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) as latest_purchase
FROM sales
ORDER BY customer_id, sale_date;
```

**Key Points:**
- `FIRST_VALUE()` gets first purchase per customer
- `LAST_VALUE()` needs explicit frame to see all rows in partition
- Without explicit frame, `LAST_VALUE()` only sees current row

**Result:**
```
customer_id | sale_date  | amount  | first_purchase | latest_purchase
------------|------------|---------|----------------|----------------
101         | 2023-01-01 | 1000.00 | 1000.00       | 2000.00
101         | 2023-01-03 | 2000.00 | 1000.00       | 2000.00  ← same customer
102         | 2023-01-02 | 1500.00 | 1500.00       | 1800.00
102         | 2023-01-05 | 1800.00 | 1500.00       | 1800.00  ← same customer
103         | 2023-01-04 | 1200.00 | 1200.00       | 1200.00  ← only purchase
```

### NTH_VALUE - Get specific position

```sql
-- Get 2nd highest salary per department
SELECT DISTINCT department,
       NTH_VALUE(salary, 1) OVER (
         PARTITION BY department 
         ORDER BY salary DESC
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) as highest_salary,
       NTH_VALUE(salary, 2) OVER (
         PARTITION BY department 
         ORDER BY salary DESC
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) as second_highest_salary
FROM employees;
```

**Key Points:**
- `NTH_VALUE(col, n)` gets the nth value in ordered partition
- Explicit frame required to see all rows
- Returns NULL if partition has fewer than N rows

**Result:**
```
department  | highest_salary | second_highest_salary
------------|----------------|----------------------
Engineering | 95000         | 90000
Sales       | 75000         | 75000      ← tied values
Marketing   | 65000         | NULL       ← only 1 employee
```

---

## 📈 Statistical Functions

### Percentile Analysis

```sql
-- Salary percentiles
SELECT name, salary,
       PERCENT_RANK() OVER (ORDER BY salary) as percentile_rank,
       CUME_DIST() OVER (ORDER BY salary) as cumulative_distribution,
       ROUND(100 * PERCENT_RANK() OVER (ORDER BY salary), 1) as percentile
FROM employees
ORDER BY salary DESC;
```

**Key Points:**
- `PERCENT_RANK()` returns 0-1 scale (0% to 100%)
- `CUME_DIST()` includes current row in calculation
- Both functions handle ties by averaging positions

**Result:**
```
name  | salary | percentile_rank | cumulative_distribution | percentile
------|--------|-----------------|-------------------------|----------
David | 95000  | 1.0            | 1.0                     | 100.0  ← highest
Alice | 90000  | 0.8            | 0.83                    | 80.0   
Bob   | 85000  | 0.6            | 0.67                    | 60.0
Eve   | 75000  | 0.3            | 0.5                     | 30.0   ← tied
Carol | 75000  | 0.3            | 0.5                     | 30.0   ← tied  
Frank | 65000  | 0.0            | 0.17                    | 0.0    ← lowest
```

### Quartile Analysis

```sql
-- Salary quartiles with statistics
SELECT name, salary,
       NTILE(4) OVER (ORDER BY salary) as quartile,
       ROUND(AVG(salary) OVER (), 0) as overall_avg,
       salary - ROUND(AVG(salary) OVER (), 0) as deviation_from_avg
FROM employees;
```

**Key Points:**
- `AVG() OVER ()` with empty window calculates overall average
- Each quartile contains approximately 25% of data
- Deviation shows how far each salary is from company average

**Result:**
```
name  | salary | quartile | overall_avg | deviation_from_avg
------|--------|----------|-------------|-------------------
Frank | 65000  | 1        | 78333      | -13333    ← bottom 25%
Carol | 75000  | 2        | 78333      | -3333     ← 2nd quartile
Eve   | 75000  | 2        | 78333      | -3333     ← 2nd quartile  
Bob   | 85000  | 3        | 78333      | 6667      ← 3rd quartile
Alice | 90000  | 4        | 78333      | 11667     ← top 25%
David | 95000  | 4        | 78333      | 16667     ← top 25%
```

---

## 🎨 Frame Clauses Deep Dive

### Frame Types Comparison

```sql
-- Different frame behaviors
SELECT sale_date, amount,
       SUM(amount) OVER (ORDER BY sale_date) as default_sum,
       SUM(amount) OVER (
         ORDER BY sale_date 
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) as rows_sum,
       SUM(amount) OVER (
         ORDER BY sale_date 
         ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
       ) as moving_sum_3,
       SUM(amount) OVER (
         ORDER BY sale_date 
         ROWS BETWEEN CURRENT ROW AND 2 FOLLOWING
       ) as future_sum
FROM sales;
```

**Key Points:**
- Default frame = `RANGE UNBOUNDED PRECEDING` (all previous + current)
- `ROWS` counts physical rows, `RANGE` considers value ranges
- Different frames create different calculation windows

**Result:**
```
sale_date  | amount  | default_sum | rows_sum | moving_sum_3 | future_sum
-----------|---------|-------------|----------|--------------|------------
2023-01-01 | 1000.00 | 1000.00    | 1000.00  | 2500.00     | 4500.00
2023-01-02 | 1500.00 | 2500.00    | 2500.00  | 4500.00     | 5000.00  
2023-01-03 | 2000.00 | 4500.00    | 4500.00  | 4700.00     | 5000.00
2023-01-04 | 1200.00 | 5700.00    | 5700.00  | 5000.00     | 3000.00
2023-01-05 | 1800.00 | 7500.00    | 7500.00  | 3000.00     | 1800.00
```

### Common Frame Patterns

```sql
-- Most useful frame patterns  
SELECT sale_date, amount,
       SUM(amount) OVER (ORDER BY sale_date) as running_total,
       AVG(amount) OVER (
         ORDER BY sale_date 
         ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) as last_3_avg,
       AVG(amount) OVER (
         ORDER BY sale_date 
         ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
       ) as centered_avg,
       COUNT(*) OVER (
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) as total_rows
FROM sales;
```

**Key Points:**
- Running total: accumulates all previous values
- Last N average: sliding window for trend analysis  
- Centered average: smooths data using surrounding values
- Total rows: entire dataset size available to each row

**Result:**
```
sale_date  | amount  | running_total | last_3_avg | centered_avg | total_rows
-----------|---------|---------------|-------------|--------------|------------
2023-01-01 | 1000.00 | 1000.00      | 1000.00    | 1250.00     | 5
2023-01-02 | 1500.00 | 2500.00      | 1250.00    | 1500.00     | 5
2023-01-03 | 2000.00 | 4500.00      | 1500.00    | 1566.67     | 5  
2023-01-04 | 1200.00 | 5700.00      | 1566.67    | 1666.67     | 5
2023-01-05 | 1800.00 | 7500.00      | 1666.67    | 1500.00     | 5
```

---

## 💼 Real-World Business Scenarios

### 1. Top N per Category

```sql
-- Top 2 earners per department
WITH ranked_employees AS (
  SELECT name, department, salary,
         ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
  FROM employees
)
SELECT name, department, salary, rn as dept_rank
FROM ranked_employees 
WHERE rn <= 2
ORDER BY department, rn;
```

**Key Points:**
- CTE first ranks all employees within departments
- Filter in outer query gets top N per group
- `ROW_NUMBER()` ensures unique ranking even with ties

**Result:**
```
name  | department  | salary | dept_rank | explanation
------|-------------|--------|-----------|-------------
David | Engineering | 95000  | 1         | Top Engineering
Alice | Engineering | 90000  | 2         | 2nd Engineering
Frank | Marketing   | 65000  | 1         | Only Marketing (top by default)
Eve   | Sales       | 75000  | 1         | Top Sales (tied)
Carol | Sales       | 75000  | 2         | 2nd Sales (tied)
```

### 2. Customer Segmentation

```sql
-- Segment customers by purchase behavior
WITH customer_stats AS (
  SELECT customer_id,
         COUNT(*) as purchase_count,
         SUM(amount) as total_spent,
         AVG(amount) as avg_order_value
  FROM sales
  GROUP BY customer_id
)
SELECT customer_id, purchase_count, total_spent, avg_order_value,
       NTILE(3) OVER (ORDER BY total_spent DESC) as value_tier,
       CASE 
         WHEN NTILE(3) OVER (ORDER BY total_spent DESC) = 1 THEN 'High Value'
         WHEN NTILE(3) OVER (ORDER BY total_spent DESC) = 2 THEN 'Medium Value'  
         ELSE 'Low Value'
       END as customer_segment
FROM customer_stats
ORDER BY total_spent DESC;
```

**Key Points:**
- First CTE aggregates customer behavior metrics
- `NTILE(3)` divides customers into equal-sized value tiers
- Business logic converts tiers into meaningful segments

**Result:**
```
customer_id | purchase_count | total_spent | avg_order_value | value_tier | customer_segment
------------|----------------|-------------|-----------------|------------|------------------
102         | 2              | 3300.00    | 1650.00        | 1          | High Value
101         | 2              | 3000.00    | 1500.00        | 1          | High Value  
103         | 1              | 1200.00    | 1200.00        | 2          | Medium Value
```

### 3. Cohort Analysis

```sql
-- Monthly cohort retention
WITH first_purchase AS (
  SELECT customer_id,
         DATE_TRUNC('month', MIN(sale_date)) as cohort_month
  FROM sales
  GROUP BY customer_id
),
monthly_activity AS (
  SELECT s.customer_id,
         fp.cohort_month,
         DATE_TRUNC('month', s.sale_date) as activity_month
  FROM sales s
  JOIN first_purchase fp ON s.customer_id = fp.customer_id
)
SELECT cohort_month,
       activity_month,
       COUNT(DISTINCT customer_id) as active_customers,
       FIRST_VALUE(COUNT(DISTINCT customer_id)) OVER (
         PARTITION BY cohort_month 
         ORDER BY activity_month
       ) as cohort_size,
       ROUND(100.0 * COUNT(DISTINCT customer_id) / 
         FIRST_VALUE(COUNT(DISTINCT customer_id)) OVER (
           PARTITION BY cohort_month 
           ORDER BY activity_month
         ), 2) as retention_rate
FROM monthly_activity
GROUP BY cohort_month, activity_month
ORDER BY cohort_month, activity_month;
```

**Key Points:**
- First CTE identifies when each customer first purchased (cohort)
- Second CTE tracks monthly activity per cohort
- `FIRST_VALUE()` gets initial cohort size for retention calculation

**Result:**
```
cohort_month | activity_month | active_customers | cohort_size | retention_rate
-------------|----------------|------------------|-------------|----------------
2023-01-01   | 2023-01-01    | 3               | 3           | 100.00
```

### 4. Gap and Island Detection

```sql
-- Find consecutive sales periods
WITH sales_with_gaps AS (
  SELECT sale_date,
         LAG(sale_date) OVER (ORDER BY sale_date) as prev_date,
         sale_date - LAG(sale_date) OVER (ORDER BY sale_date) as gap_days,
         SUM(CASE WHEN sale_date - LAG(sale_date) OVER (ORDER BY sale_date) > INTERVAL '1 day' 
                  OR LAG(sale_date) OVER (ORDER BY sale_date) IS NULL
                 THEN 1 ELSE 0 END) OVER (ORDER BY sale_date) as island_id
  FROM sales
)
SELECT island_id,
       MIN(sale_date) as period_start,
       MAX(sale_date) as period_end,
       COUNT(*) as consecutive_days,
       ARRAY_AGG(sale_date ORDER BY sale_date) as dates_in_period
FROM sales_with_gaps
GROUP BY island_id
ORDER BY period_start;
```

**Key Points:**
- `LAG()` finds gaps between consecutive dates
- Running sum creates island_id when gaps > 1 day found
- Groups consecutive periods together for analysis

**Result:**
```
island_id | period_start | period_end | consecutive_days | dates_in_period
----------|--------------|------------|------------------|------------------
1         | 2023-01-01  | 2023-01-05 | 5                | {2023-01-01,2023-01-02,2023-01-03,2023-01-04,2023-01-05}
```

### 5. Running Calculations with Resets

```sql
-- Running total that resets each month
SELECT sale_date, amount,
       DATE_TRUNC('month', sale_date) as month,
       SUM(amount) OVER (
         PARTITION BY DATE_TRUNC('month', sale_date) 
         ORDER BY sale_date
       ) as monthly_running_total,
       ROW_NUMBER() OVER (
         PARTITION BY DATE_TRUNC('month', sale_date) 
         ORDER BY sale_date
       ) as day_of_month,
       SUM(amount) OVER (ORDER BY sale_date) as overall_running_total
FROM sales;
```

**Key Points:**
- `PARTITION BY month` resets calculations for each month
- Shows difference between monthly vs overall running totals
- `ROW_NUMBER()` counts days within each month

**Result:**
```
sale_date  | amount  | month      | monthly_running_total | day_of_month | overall_running_total
-----------|---------|------------|----------------------|--------------|----------------------
2023-01-01 | 1000.00 | 2023-01-01 | 1000.00             | 1            | 1000.00
2023-01-02 | 1500.00 | 2023-01-01 | 2500.00             | 2            | 2500.00
2023-01-03 | 2000.00 | 2023-01-01 | 4500.00             | 3            | 4500.00
2023-01-04 | 1200.00 | 2023-01-01 | 5700.00             | 4            | 5700.00
2023-01-05 | 1800.00 | 2023-01-01 | 7500.00             | 5            | 7500.00
```

---

## ⚡ Performance Optimization

### Indexing Strategy

```sql
-- Essential indexes for window functions
CREATE INDEX idx_employees_dept_salary ON employees (department, salary DESC);
CREATE INDEX idx_sales_date ON sales (sale_date);
CREATE INDEX idx_sales_customer_date ON sales (customer_id, sale_date);

-- Test performance impact
EXPLAIN (ANALYZE, BUFFERS) 
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;
```

**Key Points:**
- Index columns used in `PARTITION BY` and `ORDER BY` clauses
- Composite indexes support multiple window function operations
- `EXPLAIN ANALYZE` shows actual performance impact

**Example EXPLAIN output:**
```
WindowAgg  (cost=1.11..1.19 rows=6 width=42) (actual time=0.020..0.025 rows=6)
  ->  Sort  (cost=1.11..1.13 rows=6 width=42) (actual time=0.015..0.016 rows=6)
        Sort Key: department, salary DESC
        ->  Index Scan using idx_employees_dept_salary on employees
```

### Query Optimization Tips

```sql
-- ❌ Inefficient: Multiple window functions with different ordering
SELECT sale_date,
       SUM(amount) OVER (ORDER BY sale_date) as sum1,
       AVG(amount) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as avg1
FROM sales;

-- ✅ Better: Consistent window definitions using WINDOW clause
SELECT sale_date,
       SUM(amount) OVER w as running_sum,
       AVG(amount) OVER (w ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg
FROM sales
WINDOW w AS (ORDER BY sale_date);

-- ✅ Best: Use CTEs to limit data first
WITH recent_sales AS (
  SELECT * FROM sales 
  WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT sale_date,
       SUM(amount) OVER (ORDER BY sale_date) as running_sum
FROM recent_sales;
```

**Key Points:**
- `WINDOW` clause allows reusing window definitions
- Filter data early to reduce window function workload
- Consistent ordering avoids multiple sort operations

---

## 🚨 Common Pitfalls & Solutions

### 1. LAST_VALUE Gotcha

```sql
-- ❌ Wrong: Only sees current row due to default frame
SELECT name, salary,
       LAST_VALUE(salary) OVER (ORDER BY salary) as wrong_max
FROM employees;

-- ✅ Correct: Explicit frame to see all rows  
SELECT name, salary,
       LAST_VALUE(salary) OVER (
         ORDER BY salary 
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) as correct_max
FROM employees;
```

**Key Points:**
- Default frame stops at current row for `LAST_VALUE()`
- Explicit frame `UNBOUNDED FOLLOWING` includes all rows
- This is the most common window function mistake

**Wrong Result vs Correct Result:**
```
-- Wrong (default frame)
name  | salary | wrong_max
------|--------|----------
Frank | 65000  | 65000    ← only sees itself
Carol | 75000  | 75000    ← only sees up to itself
Eve   | 75000  | 75000    ← only sees up to itself

-- Correct (explicit frame)  
name  | salary | correct_max
------|--------|------------
Frank | 65000  | 95000    ← sees all rows
Carol | 75000  | 95000    ← sees all rows  
Eve   | 75000  | 95000    ← sees all rows
```

### 2. NULL Handling

```sql
-- Test with NULL values
UPDATE sales SET amount = NULL WHERE id = 3;

-- Handle NULLs in window functions
SELECT sale_date, amount,
       LAG(amount) OVER (ORDER BY sale_date) as prev_amount,
       COALESCE(LAG(amount) OVER (ORDER BY sale_date), 0) as prev_amount_filled,
       AVG(amount) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as avg_ignore_null
FROM sales;
```

**Key Points:**
- Window functions naturally ignore NULLs in aggregations
- Use `COALESCE()` to provide default values for LAG/LEAD
- NULLs don't break calculations but may need handling for business logic

**Result:**
```
sale_date  | amount  | prev_amount | prev_amount_filled | avg_ignore_null
-----------|---------|-------------|-------------------|----------------
2023-01-01 | 1000.00 | NULL       | 0                 | 1000.00
2023-01-02 | 1500.00 | 1000.00    | 1000.00          | 1250.00
2023-01-03 | NULL    | 1500.00    | 1500.00          | 1250.00  ← ignores NULL
2023-01-04 | 1200.00 | NULL       | 0                 | 1350.00  ← avg of 3 values
2023-01-05 | 1800.00 | 1200.00    | 1200.00          | 1500.00
```

### 3. Tie Handling in Rankings

```sql
-- Different behaviors with salary ties
UPDATE employees SET salary = 85000 WHERE name IN ('Alice', 'Bob');

SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) as rank_gaps,        
       DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank, 
       ROW_NUMBER() OVER (ORDER BY salary DESC, name) as row_num
FROM employees
ORDER BY salary DESC, name;
```

**Key Points:**
- Choose ranking function based on business requirements
- `ROW_NUMBER()` needs tiebreaker for deterministic results
- Understand how each function handles ties differently

**Result:**
```
name  | salary | rank_gaps | dense_rank | row_num | explanation
------|--------|-----------|------------|---------|-------------
David | 95000  | 1         | 1          | 1       | Clear winner
Alice | 85000  | 2         | 2          | 2       | Tied for 2nd
Bob   | 85000  | 2         | 2          | 3       | Tied for 2nd
Eve   | 75000  | 4         | 3          | 4       | Rank jumps to 4
Carol | 75000  | 4         | 3          | 5       | Dense rank stays 3
Frank | 65000  | 6         | 4          | 6       | Rank jumps to 6
```

---

## 🔗 Advanced Combinations

### Multiple Window Functions

```sql
-- Comprehensive employee analysis
SELECT name, department, salary, hire_date,
       -- Rankings within department
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dept_salary_rank,
       ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 1) as company_percentile,
       
       -- Salary comparisons
       salary - AVG(salary) OVER (PARTITION BY department) as dept_salary_diff,
       ROUND(salary * 100.0 / FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC), 1) as pct_of_dept_max,
       
       -- Temporal analysis  
       EXTRACT(DAYS FROM hire_date - LAG(hire_date) OVER (ORDER BY hire_date)) as days_since_last_hire,
       COUNT(*) OVER (PARTITION BY department) as dept_size,
       
       -- Advanced calculations
       CASE WHEN salary > AVG(salary) OVER (PARTITION BY department) THEN 'Above Avg' ELSE 'Below Avg' END as dept_performance
FROM employees
ORDER BY department, salary DESC;
```

**Key Points:**
- Multiple window functions can reference different partitions/orders
- Combine rankings, aggregations, and comparisons in single query
- Mix window functions with regular expressions for complex analysis

**Result:**
```
name  | department  | salary | dept_salary_rank | company_percentile | dept_salary_diff | pct_of_dept_max | days_since_last_hire | dept_size | dept_performance
------|-------------|--------|------------------|-------------------|------------------|-----------------|---------------------|-----------|------------------
David | Engineering | 95000  | 1                | 100.0             | 5000             | 100.0           | 300                 | 3         | Above Avg
Alice | Engineering | 85000  | 2                | 60.0              | -5000            | 89.5            | 64                  | 3         | Below Avg
Bob   | Engineering | 85000  | 2                | 60.0              | -5000            | 89.5            | 81                  | 3         | Below Avg
```

---

## 📚 Quick Reference Table

| **Use Case** | **Function** | **Pattern** | **Example Output** |
|-------------|-------------|-------------|-------------------|
| Sequential numbering | `ROW_NUMBER()` | `ROW_NUMBER() OVER (ORDER BY col)` | 1,2,3,4,5,6 |
| Top N per group | `ROW_NUMBER()` | `ROW_NUMBER() OVER (PARTITION BY group ORDER BY col)` | 1,2,3,1,2,1 |
| Running total | `SUM()` | `SUM(col) OVER (ORDER BY date)` | 100,300,600,1000 |
| Moving average | `AVG()` | `AVG(col) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` | 100,150,200,233 |
| Compare to previous | `LAG()` | `LAG(col, 1) OVER (ORDER BY date)` | NULL,100,200,300 |
| Percentage of total | `SUM()` | `col / SUM(col) OVER (PARTITION BY group) * 100` | 33.3,33.3,33.3 |
| First/last value | `FIRST_VALUE()` | `FIRST_VALUE(col) OVER (PARTITION BY group ORDER BY date)` | 100,100,100 |
| Percentile rank | `PERCENT_RANK()` | `PERCENT_RANK() OVER (ORDER BY col)` | 0.0,0.2,0.4,0.6,0.8,1.0 |
| Quartiles | `NTILE()` | `NTILE(4) OVER (ORDER BY col)` | 1,1,2,2,3,3,4,4 |

---

## 📖 Resources

- [PostgreSQL Window Functions Documentation](https://www.postgresql.org/docs/current/functions-window.html)
- [PostgreSQL Query Performance](https://www.postgresql.org/docs/current/performance-tips.html)