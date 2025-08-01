---
title: "PostgreSQL `UPDATE ... FROM` Syntax & Usage Cheat Sheet"
date: 2025-08-01T05:30:30Z
tags:
- postgresql
- sql
- update_from
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Comprehensive guide to PostgreSQL UPDATE ... FROM syntax with examples and use cases."
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
# PostgreSQL `UPDATE ... FROM` Syntax & Usage Cheat Sheet

## Basic Syntax

```sql
UPDATE target_table
SET column1 = value1, 
    column2 = value2
FROM source_table
WHERE target_table.id = source_table.id;

-- Or with explicit JOIN syntax
UPDATE target_table
SET column1 = value1, 
    column2 = value2
FROM source_table
JOIN target_table ON target_table.id = source_table.id;
```

## Core Components

- **Target Table**: The table being updated
- **SET Clause**: Columns to update with new values
- **FROM Clause**: Additional tables for joining data
- **WHERE Clause**: Join condition and optional filters

## Use Case 1: Update from Another Table

**Scenario**: Update employee salaries based on department budget adjustments

```sql
-- Setup tables
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    salary DECIMAL(10,2)
);

CREATE TABLE department_adjustments (
    department_id INT,
    adjustment_percentage DECIMAL(5,2)
);

-- Sample data
INSERT INTO employees VALUES 
    (1, 'John Doe', 1, 50000),
    (2, 'Jane Smith', 1, 55000),
    (3, 'Bob Johnson', 2, 60000);

INSERT INTO department_adjustments VALUES 
    (1, 10.5),  -- 10.5% increase
    (2, -5.0);  -- 5% decrease

-- Update query
UPDATE employees
SET salary = salary * (1 + adj.adjustment_percentage / 100)
FROM department_adjustments adj
WHERE employees.department_id = adj.department_id;
```

**Result:**
```
 id |    name     | department_id |  salary  
----+-------------+---------------+----------
  1 | John Doe    |             1 | 55250.00
  2 | Jane Smith  |             1 | 60775.00
  3 | Bob Johnson |             2 | 57000.00
```

## Use Case 2: Multiple Table Joins

**Scenario**: Update product prices based on category and supplier information

```sql
-- Setup tables
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    category_id INT,
    supplier_id INT,
    price DECIMAL(10,2)
);

CREATE TABLE categories (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    markup_factor DECIMAL(3,2)
);

CREATE TABLE suppliers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    discount_rate DECIMAL(5,2)
);

-- Sample data
INSERT INTO products VALUES 
    (1, 'Laptop', 1, 1, 1000),
    (2, 'Mouse', 1, 2, 25),
    (3, 'Chair', 2, 1, 200);

INSERT INTO categories VALUES 
    (1, 'Electronics', 1.20),
    (2, 'Furniture', 1.15);

INSERT INTO suppliers VALUES 
    (1, 'TechCorp', 5.0),
    (2, 'GadgetInc', 8.0);

-- Update with multiple joins
UPDATE products
SET price = price * c.markup_factor * (1 - s.discount_rate / 100)
FROM categories c
JOIN suppliers s ON products.supplier_id = s.id
WHERE products.category_id = c.id;
```

**Result:**
```
 id |  name  | category_id | supplier_id |  price  
----+--------+-------------+-------------+---------
  1 | Laptop |           1 |           1 | 1140.00
  2 | Mouse  |           1 |           2 |   27.60
  3 | Chair  |           2 |           1 |  218.50
```

## Use Case 3: Conditional Updates with Subqueries

**Scenario**: Update customer status based on recent order activity

```sql
-- Setup tables
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    status VARCHAR(20) DEFAULT 'Active',
    last_updated DATE
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    amount DECIMAL(10,2)
);

-- Sample data
INSERT INTO customers VALUES 
    (1, 'Alice Brown', 'Active', '2024-01-01'),
    (2, 'Charlie Davis', 'Active', '2024-01-01'),
    (3, 'Eve Wilson', 'Active', '2024-01-01');

INSERT INTO orders VALUES 
    (1, 1, '2024-07-15', 500),
    (2, 1, '2024-07-20', 300),
    (3, 2, '2024-01-10', 150);

-- Update customer status based on recent orders
UPDATE customers
SET status = CASE 
    WHEN order_stats.total_amount > 500 THEN 'Premium'
    WHEN order_stats.recent_orders > 0 THEN 'Active'
    ELSE 'Inactive'
END,
last_updated = CURRENT_DATE
FROM (
    SELECT 
        customer_id,
        COUNT(*) as recent_orders,
        SUM(amount) as total_amount
    FROM orders
    WHERE order_date >= '2024-06-01'
    GROUP BY customer_id
) order_stats
WHERE customers.id = order_stats.customer_id;
```

**Result:**
```
 id |     name     |  status  | last_updated 
----+--------------+----------+--------------
  1 | Alice Brown  | Premium  | 2025-08-01
  2 | Charlie Davis| Active   | 2025-08-01
  3 | Eve Wilson   | Active   | 2024-01-01
```

## Use Case 4: Update with Aggregations

**Scenario**: Update department statistics based on employee data

```sql
-- Setup tables
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    employee_count INT DEFAULT 0,
    avg_salary DECIMAL(10,2) DEFAULT 0,
    total_salary_budget DECIMAL(12,2) DEFAULT 0
);

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    salary DECIMAL(10,2)
);

-- Sample data
INSERT INTO departments VALUES 
    (1, 'Engineering', 0, 0, 0),
    (2, 'Marketing', 0, 0, 0);

INSERT INTO employees VALUES 
    (1, 'John Doe', 1, 75000),
    (2, 'Jane Smith', 1, 80000),
    (3, 'Bob Johnson', 1, 70000),
    (4, 'Alice Brown', 2, 60000),
    (5, 'Charlie Davis', 2, 65000);

-- Update department statistics
UPDATE departments
SET employee_count = dept_stats.emp_count,
    avg_salary = dept_stats.avg_sal,
    total_salary_budget = dept_stats.total_sal
FROM (
    SELECT 
        department_id,
        COUNT(*) as emp_count,
        AVG(salary) as avg_sal,
        SUM(salary) as total_sal
    FROM employees
    GROUP BY department_id
) dept_stats
WHERE departments.id = dept_stats.department_id;
```

**Result:**
```
 id |    name     | employee_count |  avg_salary  | total_salary_budget 
----+-------------+----------------+--------------+---------------------
  1 | Engineering |              3 |     75000.00 |           225000.00
  2 | Marketing   |              2 |     62500.00 |           125000.00
```

## Use Case 5: Self-Join Updates

**Scenario**: Update employee records with manager information

```sql
-- Setup table
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INT,
    manager_name VARCHAR(100)
);

-- Sample data
INSERT INTO employees VALUES 
    (1, 'CEO Boss', NULL, NULL),
    (2, 'John Manager', 1, NULL),
    (3, 'Jane Developer', 2, NULL),
    (4, 'Bob Developer', 2, NULL),
    (5, 'Alice Lead', 1, NULL);

-- Update with manager names using self-join
UPDATE employees
SET manager_name = mgr.name
FROM employees mgr
WHERE employees.manager_id = mgr.id;
```

**Result:**
```
 id |     name      | manager_id | manager_name 
----+---------------+------------+--------------
  1 | CEO Boss      |       NULL | NULL
  2 | John Manager  |          1 | CEO Boss
  3 | Jane Developer|          2 | John Manager
  4 | Bob Developer |          2 | John Manager
  5 | Alice Lead    |          1 | CEO Boss
```

## Performance Tips

1. **Use indexes** on join columns for better performance
2. **Filter early** in the FROM clause when possible
3. **Consider EXPLAIN ANALYZE** to optimize complex updates
4. **Use CTEs** for complex subqueries to improve readability

## Common Patterns

### Pattern 1: Direct Column Update
```sql
UPDATE table1
SET column1 = table2.value
FROM table2
WHERE table1.id = table2.ref_id;
```

### Pattern 2: Calculated Update  
```sql
UPDATE table1
SET column1 = table1.column1 * table2.factor
FROM table2
WHERE table1.category = table2.category;
```

### Pattern 3: Conditional Update
```sql
UPDATE table1
SET status = CASE 
    WHEN table2.score > 80 THEN 'High'
    WHEN table2.score > 60 THEN 'Medium'
    ELSE 'Low'
END
FROM table2
WHERE table1.id = table2.ref_id;
```

### Pattern 4: Multiple Table Joins
```sql
UPDATE table1
SET column1 = table2.value * table3.multiplier
FROM table2
JOIN table3 ON table2.category_id = table3.id
WHERE table1.ref_id = table2.id;
```

## Key Benefits

- **Efficiency**: Single query instead of multiple UPDATE statements
- **Atomicity**: All updates happen in one transaction
- **Flexibility**: Complex joins and calculations in one operation
- **Performance**: Better than cursor-based or application-level updates