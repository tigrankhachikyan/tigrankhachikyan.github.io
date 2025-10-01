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
- 💾 **Immediate space reclaim** - disk space freed after commit
- ⏮️ **Transactional** - can be rolled back within a transaction
- 🔄 **MVCC-compliant** - uses file swapping mechanism

**Use when:**
- You don't need the data anymore
- Clearing staging/temporary tables
- Resetting test environments

### How TRUNCATE Actually Works Under the Hood

Unlike what you might expect, **TRUNCATE doesn't immediately delete data from disk**. Instead, it uses PostgreSQL's MVCC (Multi-Version Concurrency Control) system with a clever file-swapping mechanism:

**Step 1: File Creation**
```sql
BEGIN;
TRUNCATE table1;
```

When you run TRUNCATE:
- PostgreSQL creates a **brand new, empty data file** with a new relfilenode (file identifier)
- The old file containing all your data remains completely untouched on disk
- Your transaction now points to the new empty file
- Other concurrent sessions still point to the old file with all the data

**On disk during uncommitted TRUNCATE:**
```
/data/base/16384/12345  <- Old file (still has all data, other sessions use this)
/data/base/16384/12346  <- New empty file (your transaction uses this)
```

**Step 2: Transaction Visibility**

Within your transaction, you immediately see an empty table:
```sql
BEGIN;
TRUNCATE table1;
SELECT * FROM table1;  -- Returns 0 rows (you see the new empty file)
-- But other sessions still see all the original data!
```

Meanwhile, parallel sessions see the original data:
```sql
-- Different session/connection
SELECT * FROM table1;  -- Still returns all rows from the old file
```

**This is how both versions coexist!** Your transaction references the new empty file, while other transactions reference the old full file. Both files exist on disk simultaneously.

**Step 3: After COMMIT**
```sql
COMMIT;
```

- The system catalog permanently updates to point to the new empty file
- The old data file is marked as "pending deletion"
- Other transactions started after this point will see the empty table

**Step 4: Cleanup Phase**

The old file is physically deleted from disk only when:
- No active transactions could possibly still need it
- PostgreSQL confirms no one references it anymore
- Cleanup happens via checkpoint process or during subsequent table modifications

**You can see this file swapping in action:**
```sql
-- Check current file identifier
SELECT relfilenode FROM pg_class WHERE relname = 'table1';
-- Returns: 12345

BEGIN;
TRUNCATE table1;

-- Check again (same session)
SELECT relfilenode FROM pg_class WHERE relname = 'table1';
-- Returns: 12346  (new file created!)

ROLLBACK;  -- Or COMMIT

-- After ROLLBACK: still 12345 (reverted to old file)
-- After COMMIT: 12346 becomes permanent
```

**Why this design?**

This file-swapping approach provides several critical benefits:

1. **Instant rollback**: Just discard the new empty file and keep the old one
2. **True MVCC compliance**: Other transactions see consistent data throughout
3. **No blocking reads**: Concurrent SELECT queries continue uninterrupted until commit
4. **Crash safety**: If PostgreSQL crashes, the old file is still intact
5. **Speed**: Swapping file pointers is instant, regardless of table size

**Contrast with DELETE:**
- `DELETE FROM table1;` marks every row as deleted within the same file
- Old row versions remain in the file taking up space
- Requires VACUUM to reclaim space
- Much slower for large tables (must scan every row)
- `TRUNCATE` just swaps files - no row scanning needed

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

### How VACUUM FULL Works

Unlike TRUNCATE's file swapping, VACUUM FULL:
- Creates a new file and copies live rows into it (compacting them)
- Rebuilds all indexes
- Swaps to the new file after copying is complete
- Deletes the old bloated file

This explains why VACUUM FULL is much slower - it must read and copy every live row, while TRUNCATE just creates an empty file.

## Quick Comparison

| Feature | TRUNCATE | VACUUM FULL |
|---------|----------|-------------|
| **Speed** | Instant | Very slow |
| **Data** | Deleted | Preserved |
| **Lock** | ACCESS EXCLUSIVE | ACCESS EXCLUSIVE |
| **Mechanism** | File swapping | File rewrite + copy |
| **Space freed** | After commit | After completion |
| **Disk space needed** | None extra | 2x table size |
| **Transactional** | Yes (can rollback) | No (can't rollback) |
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
- `TRUNCATE`: Lock held for milliseconds (file swap is instant)
- `VACUUM FULL`: Lock held for minutes to hours (depends on table size)

⚠️ **TRUNCATE is transactional:**
```sql
BEGIN;
TRUNCATE TABLE my_table;
-- You see empty table, others see old data
ROLLBACK;  -- Data comes back!
```

⚠️ **VACUUM FULL is rarely recommended** - consider alternatives:
- Regular `VACUUM` (only SHARE UPDATE EXCLUSIVE lock - non-blocking)
- `pg_repack` (online table rewrite, minimal locking)
- Partitioning strategies

⚠️ **TRUNCATE also affects:**
- All indexes (new empty index files created)
- TOAST tables for large columns (new empty TOAST files)
- Sequences (reset with RESTART IDENTITY)
- TRUNCATE triggers (if defined)

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

**Verify file swapping (educational):**
```sql
-- Before truncate
SELECT relfilenode FROM pg_class WHERE relname = 'my_table';

-- After truncate
SELECT relfilenode FROM pg_class WHERE relname = 'my_table';
-- Different number = new file was created!
```

## TL;DR

- **TRUNCATE** = Creates new empty file, swaps pointer instantly, old file cleaned up later (use for temp data)
- **VACUUM FULL** = Copies data to new compact file, slow but preserves data (use during maintenance)
- **Both lock the table** - plan for downtime
- **TRUNCATE is transactional** - happens within your session immediately, visible to others after COMMIT
- **Consider alternatives** like `pg_repack` for production systems