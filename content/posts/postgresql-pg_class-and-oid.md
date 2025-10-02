---
title: "PostgreSQL pg_class & OID: The Complete Cheatsheet"
date: 2025-10-02T07:15:20Z
tags:
- postgresql
- pg_class
- oid
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Master PostgreSQL's pg_class system catalog and OID with this practical cheatsheet. Includes 12 real-world SQL queries, complete column reference, and performance tips for database introspection and optimization."
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
# PostgreSQL pg_class & OID: The Complete Cheatsheet

A practical guide to understanding and using PostgreSQL's system catalog.

---

## What is OID?

**OID (Object Identifier)** is PostgreSQL's internal row identifier used throughout system catalogs.

### Key Points
- Unique integer identifier for database objects
- Used internally by PostgreSQL to reference catalog entries
- NOT recommended for user tables (deprecated since PostgreSQL 12)
- Every system catalog row has an OID
- Fast lookups using OID instead of names

### OID Data Types
- `oid` - Standard 4-byte object identifier
- `regclass` - OID with table name casting
- `regproc` - OID for functions
- `regtype` - OID for data types
- `regnamespace` - OID for schemas

---

## pg_class Overview

The **master catalog** for all table-like objects in PostgreSQL.

### What's Stored Here?
- Tables
- Indexes
- Sequences
- Views (regular & materialized)
- Foreign tables
- Partitioned tables
- TOAST tables
- Composite types

---

## Essential Columns Reference

| Column | Type | Description | Example Values |
|--------|------|-------------|----------------|
| **oid** | oid | Unique identifier | 16384, 16385 |
| **relname** | name | Object name | users, users_pkey |
| **relnamespace** | oid | Schema OID | 2200 (public) |
| **reltype** | oid | Associated type OID | Links to pg_type |
| **relowner** | oid | Owner's role OID | Links to pg_authid |
| **relam** | oid | Access method | 2 (heap), 403 (btree) |
| **relkind** | char | Object type | r, i, S, v, m |
| **reltuples** | float4 | Est. row count | 150000.0 |
| **relpages** | int4 | Est. disk pages | 1234 |
| **relhasindex** | bool | Has indexes? | true/false |
| **relpersistence** | char | Persistence type | p, u, t |
| **relnatts** | int2 | Number of columns | 15 |
| **relchecks** | int2 | CHECK constraints | 2 |
| **relhasrules** | bool | Has rules? | true/false |
| **relhastriggers** | bool | Has triggers? | true/false |
| **relhassubclass** | bool | Has inheritance? | true/false |
| **relrowsecurity** | bool | Row security enabled? | true/false |
| **relforcerowsecurity** | bool | Force RLS? | true/false |
| **relispopulated** | bool | Materialized view populated? | true/false |
| **relreplident** | char | Replica identity | d, n, f, i |
| **relispartition** | bool | Is partition? | true/false |
| **reloptions** | text[] | Storage options | {fillfactor=70} |

### relkind Values

| Code | Meaning |
|------|---------|
| **r** | Ordinary table |
| **i** | Index |
| **S** | Sequence |
| **v** | View |
| **m** | Materialized view |
| **c** | Composite type |
| **t** | TOAST table |
| **f** | Foreign table |
| **p** | Partitioned table |
| **I** | Partitioned index |

### relpersistence Values

| Code | Meaning |
|------|---------|
| **p** | Permanent table |
| **u** | Unlogged table |
| **t** | Temporary table |

---

## Real-World Use Cases

### 1. Find All Tables in a Database

```sql
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relkind = 'r'
  AND relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public')
ORDER BY pg_total_relation_size(oid) DESC;
```

### 2. Get Table Statistics

```sql
SELECT 
    relname AS table_name,
    reltuples::bigint AS estimated_rows,
    relpages AS disk_pages,
    pg_size_pretty(pg_relation_size(oid)) AS table_size
FROM pg_class
WHERE relkind = 'r'
  AND relname = 'orders';
```

### 3. List All Indexes for a Table

```sql
SELECT 
    i.relname AS index_name,
    pg_size_pretty(pg_relation_size(i.oid)) AS index_size,
    idx.indisunique AS is_unique,
    idx.indisprimary AS is_primary
FROM pg_class t
JOIN pg_index idx ON t.oid = idx.indrelid
JOIN pg_class i ON i.oid = idx.indexrelid
WHERE t.relname = 'users';
```

### 4. Find Tables Without Primary Keys

```sql
SELECT 
    t.relname AS table_name,
    t.reltuples::bigint AS row_count
FROM pg_class t
WHERE t.relkind = 'r'
  AND t.relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public')
  AND NOT EXISTS (
      SELECT 1 
      FROM pg_index i 
      WHERE i.indrelid = t.oid 
        AND i.indisprimary
  );
```

### 5. Identify Bloated Tables

```sql
SELECT 
    relname AS table_name,
    n_dead_tup AS dead_tuples,
    n_live_tup AS live_tuples,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS bloat_percentage
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY bloat_percentage DESC;
```

### 6. Find All Sequences and Their Values

```sql
SELECT 
    c.relname AS sequence_name,
    pg_get_serial_sequence(t.schemaname || '.' || t.tablename, a.attname) AS associated_table
FROM pg_class c
LEFT JOIN pg_depend d ON d.objid = c.oid
LEFT JOIN pg_attrdef a ON a.oid = d.refobjid
LEFT JOIN pg_tables t ON t.tablename = a.adrelid::regclass::text
WHERE c.relkind = 'S';
```

### 7. Check Table Access Methods

```sql
SELECT 
    c.relname AS table_name,
    am.amname AS access_method
FROM pg_class c
JOIN pg_am am ON c.relam = am.oid
WHERE c.relkind = 'r'
  AND c.relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public');
```

### 8. List Materialized Views and Refresh Status

```sql
SELECT 
    relname AS matview_name,
    relispopulated AS is_populated,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relkind = 'm'
ORDER BY relname;
```

### 9. Find Partitioned Tables and Their Partitions

```sql
-- Parent tables
SELECT relname AS partitioned_table
FROM pg_class
WHERE relkind = 'p';

-- Get partition info
SELECT 
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_get_expr(child.relpartbound, child.oid) AS partition_expression
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relkind = 'p';
```

### 10. Identify Unused Indexes

```sql
SELECT 
    i.relname AS index_name,
    t.relname AS table_name,
    pg_size_pretty(pg_relation_size(i.oid)) AS index_size,
    s.idx_scan AS number_of_scans
FROM pg_class i
JOIN pg_index idx ON i.oid = idx.indexrelid
JOIN pg_class t ON t.oid = idx.indrelid
LEFT JOIN pg_stat_user_indexes s ON i.relname = s.indexrelname
WHERE i.relkind = 'i'
  AND NOT idx.indisprimary
  AND s.idx_scan < 50
ORDER BY pg_relation_size(i.oid) DESC;
```

### 11. Monitor Table Sizes Across Schemas

```sql
SELECT 
    n.nspname AS schema_name,
    c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_relation_size(c.oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(c.oid) - pg_relation_size(c.oid)) AS indexes_size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 20;
```

### 12. Using OID for Fast Lookups

```sql
-- Convert table name to OID
SELECT 'users'::regclass::oid;
-- Result: 16384

-- Use OID directly (faster than name lookup)
SELECT * FROM pg_class WHERE oid = 16384;

-- Get table name from OID
SELECT 16384::regclass;
-- Result: users
```

---

## Performance Tips

### When to Use pg_class

✅ **Good for:**
- Database introspection and monitoring
- Building admin tools
- Understanding database structure
- Capacity planning queries
- Finding optimization opportunities

❌ **Avoid for:**
- High-frequency queries in application code
- Real-time data access patterns
- Using reltuples for exact counts (use COUNT(*) instead)

### Optimization Tricks

```sql
-- Use OID instead of joining on names
SELECT * 
FROM pg_class 
WHERE oid = 'my_table'::regclass;

-- Cache namespace OIDs
WITH schema_oid AS (
    SELECT oid FROM pg_namespace WHERE nspname = 'public'
)
SELECT relname 
FROM pg_class 
WHERE relnamespace = (SELECT oid FROM schema_oid);

-- Combine with pg_stat for statistics
SELECT 
    c.relname,
    s.n_live_tup,
    s.n_dead_tup
FROM pg_class c
JOIN pg_stat_user_tables s ON c.oid = s.relid
WHERE c.relkind = 'r';
```

---

## Common Joins

### Join with pg_namespace (Schemas)

```sql
SELECT 
    n.nspname AS schema_name,
    c.relname AS table_name
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid;
```

### Join with pg_attribute (Columns)

```sql
SELECT 
    c.relname AS table_name,
    a.attname AS column_name,
    a.attnum AS column_position
FROM pg_class c
JOIN pg_attribute a ON a.attrelid = c.oid
WHERE c.relname = 'users'
  AND a.attnum > 0
  AND NOT a.attisdropped;
```

### Join with pg_index (Index Info)

```sql
SELECT 
    t.relname AS table_name,
    i.relname AS index_name,
    idx.indisunique AS is_unique
FROM pg_class t
JOIN pg_index idx ON t.oid = idx.indrelid
JOIN pg_class i ON i.oid = idx.indexrelid;
```

### Join with pg_stat_user_tables (Statistics)

```sql
SELECT 
    c.relname,
    s.seq_scan,
    s.idx_scan,
    s.n_tup_ins,
    s.n_tup_upd,
    s.n_tup_del
FROM pg_class c
JOIN pg_stat_user_tables s ON c.oid = s.relid;
```

---

## Quick Reference Commands

```sql
-- Get your table's OID
SELECT oid FROM pg_class WHERE relname = 'your_table';

-- Or use the shortcut
SELECT 'your_table'::regclass::oid;

-- Convert OID back to name
SELECT oid::regclass FROM pg_class WHERE oid = 16384;

-- List all relation types in your database
SELECT relkind, COUNT(*) 
FROM pg_class 
GROUP BY relkind;

-- Find system vs user tables
SELECT 
    CASE 
        WHEN relnamespace = 11 THEN 'System'
        ELSE 'User'
    END AS table_type,
    COUNT(*)
FROM pg_class
WHERE relkind = 'r'
GROUP BY table_type;
```

---

## Key Takeaways

- **pg_class** is the central registry for all database objects
- **OID** is PostgreSQL's internal way to uniquely identify objects
- Use **relkind** to filter by object type (tables, indexes, sequences, etc.)
- **reltuples** and **relpages** are estimates, not exact counts
- Combine pg_class with other system catalogs for powerful introspection
- Always filter by **relnamespace** when working with specific schemas
- Use **regclass** for easy OID ↔ name conversions

---

## Further Reading

- Official docs: `pg_class` system catalog
- Related catalogs: `pg_namespace`, `pg_attribute`, `pg_index`, `pg_constraint`
- Statistics views: `pg_stat_user_tables`, `pg_stat_user_indexes`
- Size functions: `pg_relation_size()`, `pg_total_relation_size()`, `pg_size_pretty()`