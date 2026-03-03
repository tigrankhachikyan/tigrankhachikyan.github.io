---
title: "PostgreSQL's Hidden Time Bombs: XID Wraparound and the Visibility Map"
date: 2026-03-03T02:30:30Z
tags:
- postgresql
- sql
- xid
- visibility_map
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Comprehensive guide to PostgreSQL XID wraparound and the visibility map with examples and use cases."
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
# PostgreSQL's Hidden Time Bombs: XID Wraparound and the Visibility Map

Every PostgreSQL database is running a race against time. Not metaphorically—literally. Deep in the engine, a 32-bit counter ticks upward with every transaction, and when it runs out of numbers, bad things happen. Meanwhile, a separate mechanism called the visibility map quietly determines whether your queries run fast or grind through unnecessary I/O.

Both of these MVCC internals are easy to ignore until they cause problems. Let's fix that.

---

## Part 1: Transaction ID Wraparound

### The Counter That Never Stops

PostgreSQL assigns every transaction a unique 32-bit identifier called an XID. This number does several critical jobs:

- Determines which rows a transaction can see
- Gets stamped on rows via the `xmin` and `xmax` system columns
- Enables MVCC's snapshot isolation

A 32-bit integer gives you about 4.2 billion possible values. That sounds like a lot until you realize that a busy database can burn through millions of XIDs per day. At 1,000 transactions per second, you'd exhaust the counter in about 49 days.

### The Wraparound Problem

PostgreSQL uses modular arithmetic to compare XIDs. From any given XID's perspective:

- The ~2 billion XIDs "ahead" of it are in the future (not yet visible)
- The ~2 billion XIDs "behind" it are in the past (potentially visible)

Here's the problem: if a very old row still references an ancient XID, and the counter wraps around, that row's XID could suddenly appear to be in the *future*. The row would vanish from queries. Data corruption without a single bit changing on disk.

```
XIDs:  0 -------- 2 billion -------- 4 billion -----> wraps to 0
                     ^
            "past" | | "future"
                   current XID
```

If old XIDs aren't cleaned up, eventually the "past" catches up with the "future," and PostgreSQL can't tell which is which.

### PostgreSQL's Safety Mechanism

To prevent this catastrophe, PostgreSQL enforces a hard limit. When you get within about 10 million XIDs of the oldest unfrozen row's XID, the database enters "wraparound protection mode":

```
WARNING: database "mydb" must be vacuumed within 10000000 transactions
HINT: To avoid a database shutdown, execute a database-wide VACUUM.
```

If you ignore this and push further, PostgreSQL does the only safe thing—it refuses all new writes:

```
ERROR: database is not accepting commands to avoid wraparound data loss
```

At this point, you must run a manual vacuum in single-user mode. Production is down. Pagers are firing. It's 3 AM.

### Freezing: The Solution

The fix is called "freezing." When PostgreSQL vacuums a row and determines it's old enough that all possible transactions can see it, it marks the row as "frozen." Frozen rows are effectively immortal—their visibility no longer depends on XID comparisons.

```sql
-- Check the age of your oldest unfrozen XIDs
SELECT datname, age(datfrozenxid) as xid_age,
       current_setting('autovacuum_freeze_max_age') as freeze_threshold
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

Healthy output looks like this:
```
 datname  | xid_age  | freeze_threshold
----------+----------+------------------
 mydb     | 150000000| 200000000
 postgres | 150000123| 200000000
```

Dangerous output looks like this:
```
 datname  | xid_age   | freeze_threshold
----------+-----------+------------------
 mydb     | 1950000000| 200000000       -- DANGER!
```

### Why Wraparound Still Catches Teams

**Long-running transactions**: A transaction that stays open for hours (or days—it happens) holds back the oldest XID horizon. Vacuum can't freeze anything newer than what that transaction might need to see.

```sql
-- Find long-running transactions
SELECT pid, now() - xact_start as duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;
```

**Abandoned replication slots**: Logical replication slots hold back XIDs similarly. An unused slot can silently prevent freezing.

```sql
-- Check replication slot lag
SELECT slot_name, age(xmin) as xmin_age, age(catalog_xmin) as catalog_age
FROM pg_replication_slots;
```

**Massive tables with insufficient vacuum resources**: Autovacuum might not keep up with large tables that see heavy updates. The default settings are conservative.

**Disabled autovacuum**: Some teams disable autovacuum "for performance" without understanding the consequences. Don't do this.

### Monitoring and Prevention

Track XID age as a core metric:

```sql
-- Per-table XID age (find what needs freezing)
SELECT schemaname, relname, 
       age(relfrozenxid) as xid_age,
       pg_size_pretty(pg_total_relation_size(oid)) as size
FROM pg_stat_user_tables
JOIN pg_class ON relname = pg_stat_user_tables.relname
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

Set up alerts when XID age exceeds safe thresholds—say, 500 million. That gives you runway to investigate before the warnings start.

Tune autovacuum for large tables:

```sql
-- More aggressive freezing for a specific table
ALTER TABLE big_table SET (
    autovacuum_freeze_max_age = 100000000,
    autovacuum_freeze_table_age = 50000000
);
```

---

## Part 2: The Visibility Map

### What the Visibility Map Does

The visibility map is a compact data structure—two bits per heap page—that answers a simple question: "Are all tuples on this page visible to everyone?"

If yes, PostgreSQL can skip that page entirely during certain operations. If no, it must examine the page to check tuple visibility.

Those two bits per page encode:

1. **All-visible**: Every tuple on this page is visible to all current and future transactions
2. **All-frozen**: Every tuple is both visible and frozen (won't need future freezing)

For a 1 GB table with 8 KB pages, the visibility map is only about 16 KB. It's tiny but powerful.

### Index-Only Scans: The Performance Payoff

Consider this query:

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);
SELECT customer_id FROM orders WHERE customer_id = 12345;
```

Without the visibility map, PostgreSQL must:
1. Find matching entries in the index
2. Visit the heap (table) to verify each row is visible to the current transaction
3. Return the data

That heap visit is called a "heap fetch," and it's expensive—random I/O to potentially scattered pages.

With an up-to-date visibility map:
1. Find matching entries in the index
2. Check the visibility map: is this page all-visible?
3. If yes, skip the heap entirely and return data from the index
4. If no, do the heap fetch as usual

This is an **index-only scan**, and it can be dramatically faster.

```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT customer_id FROM orders WHERE customer_id = 12345;

-- Good (index-only scan, minimal heap fetches):
Index Only Scan using idx_orders_customer on orders
  Heap Fetches: 0
  Buffers: shared hit=3

-- Bad (forced to check heap constantly):
Index Only Scan using idx_orders_customer on orders
  Heap Fetches: 15420
  Buffers: shared hit=15423
```

### When the Visibility Map Goes Stale

The visibility map is updated by VACUUM, not by regular DML. This means:

**After INSERT**: New rows aren't in the visibility map. The pages containing them are marked not-all-visible.

**After UPDATE**: The old row version is marked dead, the new one is inserted. Both the old page and the new page (if different) become not-all-visible.

**After DELETE**: The page is marked not-all-visible until vacuum cleans up.

Until VACUUM runs, those pages force heap fetches even for index-only scans.

### The Silent Performance Killer

Here's a scenario that catches teams:

1. You have a large table with an index that supports fast index-only scans
2. A batch job updates 20% of the rows nightly
3. Autovacuum doesn't finish before morning traffic hits
4. Queries that ran in 5ms now take 500ms because every index-only scan degrades to a regular index scan with heap fetches
5. Nobody changed any queries or indexes—performance just tanked

The visibility map doesn't lie:

```sql
SELECT 
    relname,
    n_live_tup,
    n_dead_tup,
    n_mod_since_analyze,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

High `n_dead_tup` or large `n_mod_since_analyze` combined with an old `last_vacuum` timestamp means your visibility map is stale.

### Measuring Visibility Map Coverage

You can inspect the visibility map directly:

```sql
CREATE EXTENSION pg_visibility;

-- Summary for a table
SELECT * FROM pg_visibility_map_summary('orders');
--  all_visible | all_frozen
-- -------------+------------
--        45231 |      42100

-- What percentage of pages are all-visible?
SELECT 
    all_visible::float / (all_visible + all_frozen + 
        (SELECT relpages FROM pg_class WHERE relname = 'orders') - all_visible) * 100 
    as visibility_pct
FROM pg_visibility_map_summary('orders');
```

For tables where you rely on index-only scans, you want visibility coverage above 90%.

### Strategies for Keeping the Visibility Map Fresh

**Tune autovacuum thresholds for critical tables:**

```sql
-- Vacuum more aggressively (default is 20% of table modified)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- vacuum after 5% modified
    autovacuum_vacuum_threshold = 1000
);
```

**Schedule manual vacuums after batch operations:**

```sql
-- After a big batch update
VACUUM orders;
-- Or just update visibility map without full vacuum
VACUUM (INDEX_CLEANUP OFF, TRUNCATE OFF) orders;
```

**Monitor heap fetches in index-only scans:**

```sql
SELECT 
    schemaname, relname, indexrelname,
    idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_tup_fetch > idx_tup_read * 0.1  -- more than 10% heap fetches
ORDER BY idx_tup_fetch DESC;
```

**Use covering indexes strategically:**

```sql
-- Include columns needed by queries to avoid heap access entirely
CREATE INDEX idx_orders_covering ON orders(customer_id) 
    INCLUDE (order_date, total);
    
-- Now this query can be index-only even if visibility map is stale
SELECT customer_id, order_date, total FROM orders WHERE customer_id = 12345;
```

---

## The Connection Between XID Wraparound and Visibility Map

These two mechanisms interact in an important way.

The "all-frozen" bit in the visibility map tells VACUUM it can skip that page entirely during anti-wraparound freezing. If a page is marked all-frozen, VACUUM knows every tuple on it has already been frozen—no need to check.

This means:
- Aggressive freezing improves visibility map coverage (more all-frozen pages)
- Better visibility map coverage makes future vacuum runs faster (skip frozen pages)
- It's a virtuous cycle when healthy, a vicious cycle when neglected

When wraparound pressure builds and VACUUM must freeze aggressively, it will be faster if the visibility map is well-maintained. When the visibility map is neglected, emergency freezing takes longer, extending the outage.

---

## Key Takeaways

**For XID Wraparound:**
- Monitor `age(datfrozenxid)` as a critical metric
- Kill long-running transactions and clean up unused replication slots
- Tune autovacuum for your workload—defaults are conservative
- Treat wraparound warnings as emergencies, not annoyances

**For Visibility Map:**
- Understand that index-only scan performance depends on vacuum freshness
- Monitor heap fetches—they reveal stale visibility maps
- Vacuum after batch operations that modify significant portions of tables
- Use `pg_visibility` extension to inspect coverage

Both of these MVCC internals reward proactive monitoring and maintenance. The teams that understand them run fast, reliable databases. The teams that don't eventually get paged at 3 AM.