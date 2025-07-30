---
title: "PostgreSQL Cheat Sheet: generate_series()"
date: 2025-07-29T11:30:30Z
tags:
- postgresql
- sql
- generate_series
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: A quick reference for using PostgreSQL's `generate_series()` function.
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
# PostgreSQL Cheat Sheet: `generate_series()`

PostgreSQL's `generate_series()` is a hidden gem for developers and analysts who work with sequences, time series data, or just need quick dummy rows. Here's a cheat sheet to get you up and running fast.

---

## 📌 Syntax & Parameter Types

```sql
-- Integer series
generate_series(start INTEGER, stop INTEGER [, step INTEGER])

-- Bigint series
generate_series(start BIGINT, stop BIGINT [, step BIGINT])

-- Timestamp series
generate_series(start TIMESTAMP, stop TIMESTAMP, step INTERVAL)

-- Date series
generate_series(start DATE, stop DATE, step INTERVAL)
```

### Notes:

* `step` is **optional** for integers and defaults to `1`.
* For timestamps and dates, `step` is **required**.

---

## 🔢 Integer Examples

### ✅ 1. Numbers from 1 to 10

```sql
SELECT * FROM generate_series(1, 10);
```

**Result:**

| generate\_series |
| ---------------- |
| 1                |
| 2                |
| 3                |
| ...              |
| 10               |

### 🔁 2. Countdown from 10 to 1

```sql
SELECT * FROM generate_series(10, 1, -1);
```

**Result:**

| generate\_series |
| ---------------- |
| 10               |
| 9                |
| 8                |
| ...              |
| 1                |

### 🔢 3. Even numbers only

```sql
SELECT * FROM generate_series(0, 10, 2);
```

**Result:**

| generate\_series |
| ---------------- |
| 0                |
| 2                |
| 4                |
| 6                |
| 8                |
| 10               |

---

## 🕒 Timestamp & Date Examples

### 📅 4. Daily series in July 2025

```sql
SELECT * FROM generate_series(
  '2025-07-01'::date,
  '2025-07-05'::date,
  '1 day'
);
```

**Result:**

| generate\_series |
| ---------------- |
| 2025-07-01       |
| 2025-07-02       |
| 2025-07-03       |
| 2025-07-04       |
| 2025-07-05       |

### ⏰ 5. Hourly range in one day

```sql
SELECT * FROM generate_series(
  '2025-07-30 00:00',
  '2025-07-30 03:00',
  '1 hour'
);
```

**Result:**

| generate\_series    |
| ------------------- |
| 2025-07-30 00:00:00 |
| 2025-07-30 01:00:00 |
| 2025-07-30 02:00:00 |
| 2025-07-30 03:00:00 |

---

## ⚙️ Common Use Cases

### 📈 6. Fill missing dates in a time series

```sql
SELECT gs.day, COALESCE(s.value, 0) AS value
FROM generate_series('2025-07-01'::date, '2025-07-07'::date, '1 day') AS gs(day)
LEFT JOIN sales s ON s.sale_date = gs.day;
```

### ✖️ 7. Cartesian product generator

```sql
SELECT a, b
FROM generate_series(1, 3) AS a,
     generate_series(1, 3) AS b;
```

**Result:**

| a   | b   |
| --- | --- |
| 1   | 1   |
| 1   | 2   |
| 1   | 3   |
| 2   | 1   |
| 2   | 2   |
| ... | ... |

### 🔗 8. Join to pad missing times in log data

```sql
WITH minutes AS (
  SELECT generate_series(
    '2025-07-30 00:00',
    '2025-07-30 01:00',
    '1 minute'
  ) AS ts
)
SELECT ts, COUNT(logs.id) AS log_count
FROM minutes
LEFT JOIN logs ON date_trunc('minute', logs.logged_at) = ts
GROUP BY ts
ORDER BY ts;
```

This query generates one row per minute and joins it to a log table, filling in time intervals with zero logs.

---

## 🨠 Tips & Tricks

### 🔤 9. Aliasing output

```sql
SELECT i FROM generate_series(1, 5) AS i;
```

### 🪜 10. Use with `ROW_NUMBER()`

```sql
SELECT ROW_NUMBER() OVER (), *
FROM generate_series(10, 50, 10) AS val;
```

---

## 🔬 Advanced Patterns

### ⏱️ 11. Timestamps every 15 minutes

```sql
SELECT * FROM generate_series(
  '2025-07-30 00:00',
  '2025-07-30 00:45',
  '15 minutes'::interval
);
```

**Result:**

| generate\_series    |
| ------------------- |
| 2025-07-30 00:00:00 |
| 2025-07-30 00:15:00 |
| 2025-07-30 00:30:00 |
| 2025-07-30 00:45:00 |

### 🧵 12. Create a date/category grid

```sql
SELECT gs, cat
FROM generate_series('2025-07-01'::date, '2025-07-03'::date, '1 day') AS gs,
     unnest(ARRAY['A', 'B']) AS cat;
```

This query produces a **grid** of all combinations between each day and each category. Useful for cross-tab reporting or full combinatoric analysis.

**Result:**

| gs         | cat |
| ---------- | --- |
| 2025-07-01 | A   |
| 2025-07-01 | B   |
| 2025-07-02 | A   |
| 2025-07-02 | B   |
| 2025-07-03 | A   |
| 2025-07-03 | B   |

---

## ⚠️ Error Handling & Edge Cases

### 🚫 13. Empty series (backwards step)

```sql
-- This returns no rows
SELECT * FROM generate_series(1, 10, -1);
```

**Result:** Empty set (0 rows)

### 🔄 14. NULL handling

```sql
-- Any NULL parameter returns empty set
SELECT * FROM generate_series(1, NULL, 1);
SELECT * FROM generate_series(NULL, 10, 1);
```

**Result:** Empty set (0 rows)

### ⚡ 15. Large ranges consideration

```sql
-- Be careful with large ranges - this generates 10M rows!
SELECT COUNT(*) FROM generate_series(1, 10000000);
```

---

## 🔢 Data Type Variations

### 💰 16. Numeric/Decimal series

```sql
SELECT * FROM generate_series(1.0, 5.0, 0.5);
```

**Result:**

| generate_series |
| --------------- |
| 1.0             |
| 1.5             |
| 2.0             |
| 2.5             |
| 3.0             |
| 3.5             |
| 4.0             |
| 4.5             |
| 5.0             |

### 🌍 17. Timezone-aware timestamps

```sql
SELECT * FROM generate_series(
  '2025-07-30 00:00+00'::timestamptz,
  '2025-07-30 02:00+00'::timestamptz,
  '1 hour'::interval
);
```

---

## 📊 Complex Interval Examples

### 📅 18. Weekly series

```sql
SELECT * FROM generate_series(
  '2025-07-01'::date,
  '2025-07-31'::date,
  '1 week'::interval
);
```

### 📆 19. Monthly series

```sql
SELECT * FROM generate_series(
  '2025-01-01'::date,
  '2025-12-01'::date,
  '1 month'::interval
);
```

### 💼 20. Business days only (Monday-Friday)

```sql
SELECT gs::date AS business_day
FROM generate_series('2025-07-01'::date, '2025-07-31'::date, '1 day') AS gs
WHERE EXTRACT(dow FROM gs) BETWEEN 1 AND 5;
```

### 🔄 21. Every 3 weeks

```sql
SELECT * FROM generate_series(
  '2025-01-01'::date,
  '2025-06-01'::date,
  '3 weeks'::interval
);
```

---

## 🛠️ Practical Database Tasks

### 🧪 22. Generate test data

```sql
SELECT 
  'user_' || i AS username,
  'user' || i || '@example.com' AS email,
  (ARRAY['active', 'inactive', 'pending'])[1 + i % 3] AS status
FROM generate_series(1, 100) AS i;
```

### 📊 23. Split date ranges into chunks

```sql
-- Split a year into quarters
SELECT 
  gs AS quarter_start,
  gs + '3 months'::interval - '1 day'::interval AS quarter_end
FROM generate_series('2025-01-01', '2025-10-01', '3 months') AS gs;
```

### 📅 24. Calendar table generation

```sql
CREATE TABLE calendar AS
SELECT 
  gs::date AS date,
  EXTRACT(year FROM gs) AS year,
  EXTRACT(month FROM gs) AS month,
  EXTRACT(day FROM gs) AS day,
  EXTRACT(dow FROM gs) AS day_of_week,
  TO_CHAR(gs, 'Day') AS day_name,
  TO_CHAR(gs, 'Month') AS month_name
FROM generate_series('2025-01-01'::date, '2025-12-31'::date, '1 day') AS gs;
```

### 🔢 25. Pagination helper

```sql
-- Generate page numbers for pagination
SELECT 
  i AS page_number,
  (i - 1) * 20 AS offset_value
FROM generate_series(1, CEIL(1000.0 / 20)) AS i;
```

---

## ⚡ Performance Tips

### 🎯 26. Limit large series

```sql
-- Use LIMIT for safety with large ranges
SELECT * FROM generate_series(1, 1000000) LIMIT 100;
```

### 📈 27. Index considerations

```sql
-- When joining to generated series, ensure proper indexes exist
SELECT gs.hour, COUNT(orders.id)
FROM generate_series(0, 23) AS gs(hour)
LEFT JOIN orders ON EXTRACT(hour FROM orders.created_at) = gs.hour
GROUP BY gs.hour;

-- Make sure orders.created_at has an index or use expression index:
-- CREATE INDEX idx_orders_hour ON orders (EXTRACT(hour FROM created_at));
```

### 🔄 28. Alternative approaches

```sql
-- For simple sequences, consider SEQUENCE objects
CREATE SEQUENCE IF NOT EXISTS seq_ids START 1;
SELECT nextval('seq_ids') FROM generate_series(1, 10);

-- For arrays, consider array constructors
SELECT unnest(ARRAY[1,2,3,4,5]) AS val;
```

---

## 🔗 Integration Examples

### 🪟 29. With window functions

```sql
SELECT 
  i,
  i * 2 AS doubled,
  ROW_NUMBER() OVER (ORDER BY i DESC) AS reverse_rank,
  LAG(i, 1) OVER (ORDER BY i) AS previous_value
FROM generate_series(1, 10) AS i;
```

### 📋 30. Subquery patterns

```sql
-- Use in EXISTS subqueries
SELECT user_id 
FROM users u
WHERE EXISTS (
  SELECT 1 FROM generate_series(1, 12) AS month
  WHERE NOT EXISTS (
    SELECT 1 FROM user_activity ua 
    WHERE ua.user_id = u.user_id 
    AND EXTRACT(month FROM ua.activity_date) = month
  )
);
```

### 📦 31. Array generation from series

```sql
-- Create arrays from series
SELECT ARRAY(SELECT generate_series(1, 10)) AS number_array;
SELECT ARRAY(SELECT gs FROM generate_series('2025-01-01', '2025-01-07', '1 day') AS gs) AS date_array;
```

### 🔧 32. Dynamic pivot tables

```sql
-- Generate columns dynamically
WITH months AS (
  SELECT generate_series(1, 12) AS month_num
)
SELECT 
  user_id,
  SUM(CASE WHEN EXTRACT(month FROM created_at) = 1 THEN amount END) AS jan,
  SUM(CASE WHEN EXTRACT(month FROM created_at) = 2 THEN amount END) AS feb,
  -- ... continue for all months
  SUM(amount) AS total
FROM orders
CROSS JOIN months
GROUP BY user_id;
```

---

## 📝 Notes

* Returns a **set** of rows, usable in `SELECT`, `JOIN`, `CTE`, etc.
* Supports **INTEGER**, **BIGINT**, **DATE**, and **TIMESTAMP** types.
* `step` must be an `INTERVAL` for DATE/TIMESTAMP inputs.
* Very handy for:

  * Time series gaps
  * Generating test data
  * Scheduling jobs
  * Building grids and ranges

---

## 📚 References
  *  https://www.postgresql.org/docs/current/functions-srf.html
