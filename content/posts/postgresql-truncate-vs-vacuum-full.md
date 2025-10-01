---
title: "PostgreSQL: Reclaiming Space - TRUNCATE vs VACUUM FULL"
date: 2025-09-30T6:05:10Z
tags:
- postgresql
- sql
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: A practical cheatsheet comparing PostgreSQL's TRUNCATE and VACUUM FULL commands. Learn how each reclaims disk space, understand their locking behavior, and discover which approach fits your use case—whether you need to delete all data instantly or compress an existing table.
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

# PostgreSQL: Reclaiming Space - TRUNCATE vs VACUUM FULL

## The Problem
Large tables accumulate dead tuples and bloat over time, wasting disk space. Two commands can reclaim this space: `TRUNCATE` and `VACUUM FULL`.

## TRUNCATE: Fast but Destructive

**What it does:**
- Removes **ALL data** from the table instantly
- Resets auto-increment sequences
- Removes all rows, indexes, and constraints remain intact
- Releases disk space immediately back to the OS

**Syntax:**
```sql
TRUNCATE TABLE table_name;
TRUNCATE TABLE table_name RESTART IDENTITY;  -- Reset sequences
TRUNCATE TABLE table_name CASCADE;            -- Truncate related tables
```

**Key characteristics:**
- ⚡ **Extremely fast** - doesn't scan the table
- 🔒 **Requires ACCESS EXCLUSIVE lock** (blocks all operations)
- 🗑️ **Data loss** - all rows are permanently deleted
- 💾 **Immediate space reclaim** - disk space freed instantly
- ⏮️ **Not MVCC-safe** - cannot be rolled back easily

**Use when:**
- You don't need the data anymore
- Clearing staging/temporary tables
- Resetting test environments

## VACUUM FULL: Slow but Safe

**What it does:**
- Compacts the table by rewriting it entirely
- Removes dead tuples and reclaims space
- **Preserves all data** - only removes bloat
- Returns freed space to the OS

**Syntax:**
```sql
VACUUM FULL table_name;
VACUUM FULL VERBOSE table_name;  -- Show progress
```

**Key characteristics:**
- 🐌 **Very slow** - rewrites entire table
- 🔒 **Requires ACCESS EXCLUSIVE lock** (blocks reads and writes)
- ✅ **Data preserved** - only removes dead space
- 💾 **Space reclaimed** - but needs 2x space during operation
- ⏱️ **Long downtime** on large tables

**Use when:**
- Table has significant bloat but you need the data
- During maintenance windows
- After large DELETE/UPDATE operations

## Quick Comparison

| Feature | TRUNCATE | VACUUM FULL |
|---------|----------|-------------|
| **Speed** | Instant | Very slow |
| **Data** | Deleted | Preserved |
| **Lock** | ACCESS EXCLUSIVE | ACCESS EXCLUSIVE |
| **Space freed** | Immediate | After completion |
| **Disk space needed** | None extra | 2x table size |
| **Use case** | Don't need data | Need data, remove bloat |

## Understanding ACCESS EXCLUSIVE Locks

Both `TRUNCATE` and `VACUUM FULL` require an **ACCESS EXCLUSIVE lock** - the most restrictive lock in PostgreSQL.

**What this means:**
- 🚫 **No reads allowed** - even `SELECT` queries are blocked
- 🚫 **No writes allowed** - `INSERT`, `UPDATE`, `DELETE` are blocked
- 🚫 **No concurrent operations** - indexes, constraints, everything waits
- ⏳ **Lock wait time** - command waits if table is currently in use
- 🔄 **Query queuing** - all other queries queue behind the lock

**Real-world impact:**
```sql
-- Session 1: Long-running query
SELECT * FROM orders WHERE date > '2024-01-01';  -- Takes 30 seconds

-- Session 2: Tries to truncate
TRUNCATE TABLE orders;  -- Waits for Session 1 to finish

-- Session 3: Simple query
SELECT COUNT(*) FROM orders;  -- Also waits! Blocked by Session 2's lock request
```

**Lock behavior differences:**

| Operation | Lock Type | Blocks Reads? | Blocks Writes? |
|-----------|-----------|---------------|----------------|
| `SELECT` | ACCESS SHARE | ❌ No | ❌ No |
| `UPDATE/DELETE` | ROW EXCLUSIVE | ❌ No | ⚠️ Conflicting rows |
| `VACUUM` (regular) | SHARE UPDATE EXCLUSIVE | ❌ No | ❌ No |
| `VACUUM FULL` | ACCESS EXCLUSIVE | ✅ Yes | ✅ Yes |
| `TRUNCATE` | ACCESS EXCLUSIVE | ✅ Yes | ✅ Yes |

**Avoiding lock issues:**

```sql
-- Set a lock timeout to avoid waiting forever
SET lock_timeout = '5s';
VACUUM FULL orders;
-- If can't acquire lock in 5 seconds, it fails instead of waiting

-- Check for blocking queries before maintenance
SELECT pid, query, state, wait_event_type
FROM pg_stat_activity
WHERE datname = 'your_database' 
  AND state = 'active';

-- Kill blocking query if necessary (use carefully!)
SELECT pg_terminate_backend(pid);
```

## Important Notes

⚠️ **Lock duration matters:**
- `TRUNCATE`: Lock held for milliseconds (very fast)
- `VACUUM FULL`: Lock held for minutes to hours (depends on table size)

⚠️ **VACUUM FULL is rarely recommended** - consider alternatives:
- Regular `VACUUM` (only SHARE UPDATE EXCLUSIVE lock - non-blocking)
- `pg_repack` (online table rewrite, minimal locking)
- Partitioning strategies

⚠️ **TRUNCATE removes:**
- All rows
- Resets sequences (with RESTART IDENTITY)
- Triggers TRUNCATE triggers (if defined)

## Best Practices

**For development/testing:**
```sql
TRUNCATE TABLE staging_data;
```

**For production bloat:**
```sql
-- During maintenance window
VACUUM FULL VERBOSE production_table;

-- Or better: use pg_repack (no exclusive lock)
pg_repack -t production_table -d database_name
```

**Check table bloat before deciding:**
```sql
SELECT 
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS bloat
FROM pg_tables
WHERE tablename = 'your_table';
```

## TL;DR

- **TRUNCATE** = Delete everything, get space back instantly (use for temp data)
- **VACUUM FULL** = Keep data, remove bloat slowly (use during maintenance)
- **Both lock the table** - plan for downtime
- **Consider alternatives** like `pg_repack` for production systems