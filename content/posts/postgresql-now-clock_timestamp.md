---
title: "PostgreSQL's NOW() vs clock_timestamp(): A Deep Dive into Time and MVCC"
date: 2026-03-03T02:30:30Z
tags:
- postgresql
- sql
- time_functions
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
# PostgreSQL's NOW() vs clock_timestamp(): A Deep Dive into Time and MVCC

If you've ever written a long-running transaction in PostgreSQL and wondered why all your timestamps look identical, you've stumbled into one of the more subtle aspects of how PostgreSQL handles time. The difference between `NOW()` and `clock_timestamp()` isn't just a minor implementation detail—it's fundamentally tied to how PostgreSQL implements Multi-Version Concurrency Control (MVCC).

## The Core Difference

Let's start with a simple demonstration:

```sql
BEGIN;
SELECT NOW(), pg_sleep(2), NOW();
--  now           |  pg_sleep  |  now
-- 2024-03-15 10:00:00 |          | 2024-03-15 10:00:00

SELECT clock_timestamp(), pg_sleep(2), clock_timestamp();
--  clock_timestamp     |  pg_sleep  |  clock_timestamp
-- 2024-03-15 10:00:02 |            | 2024-03-15 10:00:04
COMMIT;
```

`NOW()` (and its SQL-standard alias `CURRENT_TIMESTAMP`) returns the same value throughout the entire transaction. `clock_timestamp()`, on the other hand, returns the actual wall-clock time at the moment it's called.

## Why Does NOW() Behave This Way?

This isn't a quirk—it's a deliberate design decision rooted in PostgreSQL's MVCC architecture.

MVCC allows multiple transactions to see consistent snapshots of the database without blocking each other. When your transaction starts, PostgreSQL essentially takes a "photograph" of the database state at that moment. Every query within your transaction sees this same snapshot, regardless of what other transactions are doing.

The transaction timestamp (`NOW()`) is part of this snapshot. By keeping it constant, PostgreSQL ensures that:

1. **Deterministic behavior**: Running the same query twice in a transaction produces identical results
2. **Consistency**: All rows inserted or updated within a transaction share the same logical timestamp
3. **Repeatability**: The transaction's view of "now" doesn't shift as it executes

Think of it this way: your transaction exists in a frozen moment of time. Everything that happens within it logically occurs at that single instant.

## The MVCC Connection

PostgreSQL tracks row visibility using two system columns: `xmin` (the transaction ID that created the row) and `xmax` (the transaction ID that deleted or updated it). When deciding if a row is visible to your transaction, PostgreSQL checks whether these transaction IDs fall within your snapshot.

The transaction timestamp complements this mechanism. Consider an audit table:

```sql
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    action TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

BEGIN;
INSERT INTO audit_log (action) VALUES ('step 1');
-- ... complex operations taking 30 seconds ...
INSERT INTO audit_log (action) VALUES ('step 2');
COMMIT;
```

Both rows get the same `created_at` value. This might seem wrong from a wall-clock perspective, but it's semantically correct: both operations belong to the same atomic transaction. They either both happened or neither did. Giving them the same timestamp reflects this atomicity.

## When clock_timestamp() Makes Sense

Sometimes you genuinely need wall-clock time:

**Performance logging:**
```sql
DO $$
DECLARE
    start_time TIMESTAMPTZ := clock_timestamp();
BEGIN
    -- expensive operation
    PERFORM pg_sleep(5);
    RAISE NOTICE 'Operation took %', clock_timestamp() - start_time;
END $$;
```

**Progress tracking in long batches:**
```sql
INSERT INTO processing_log (batch_id, item_id, processed_at)
SELECT 
    batch_id,
    item_id,
    clock_timestamp()  -- actual time each row was processed
FROM items_to_process;
```

**Debugging transaction timing:**
```sql
BEGIN;
SELECT clock_timestamp();  -- when we started
-- ... work ...
SELECT clock_timestamp();  -- how long the work took
COMMIT;
```

## The Full Family of Time Functions

PostgreSQL offers several time functions, each with specific use cases:

| Function | Behavior | Use Case |
|----------|----------|----------|
| `NOW()` / `CURRENT_TIMESTAMP` | Transaction start time | Default timestamps, audit trails |
| `TRANSACTION_TIMESTAMP()` | Same as NOW() | Explicit about intent |
| `STATEMENT_TIMESTAMP()` | Start of current statement | Per-statement timing |
| `clock_timestamp()` | Actual wall-clock time | Performance measurement, real-time logging |

`STATEMENT_TIMESTAMP()` is particularly interesting—it changes between statements but stays constant within a single statement. This can be useful for batch operations where you want each statement's rows to share a timestamp.

## Practical Implications

**Bulk inserts behave consistently:**
```sql
INSERT INTO events (name, occurred_at)
SELECT name, NOW()
FROM generate_series(1, 1000000);
-- All million rows get the same timestamp
```

**But clock_timestamp() creates a timeline:**
```sql
INSERT INTO events (name, occurred_at)
SELECT name, clock_timestamp()
FROM generate_series(1, 1000000);
-- Each row gets a slightly different timestamp
```

**Watch out for index usage:**
```sql
-- This might not use an index on occurred_at efficiently
SELECT * FROM events 
WHERE occurred_at > NOW() - INTERVAL '1 hour';
-- Because NOW() is evaluated once at transaction start
```

## A Common Gotcha

Developers sometimes write code like this:

```sql
BEGIN;
UPDATE orders SET status = 'processing', updated_at = NOW() WHERE id = 1;
-- ... API calls, external systems, 30 seconds pass ...
UPDATE orders SET status = 'complete', updated_at = NOW() WHERE id = 1;
COMMIT;
```

Both updates write the same `updated_at` value. If you need to track the actual time each status change occurred, use `clock_timestamp()`. But consider whether that's really what you want—if the transaction rolls back, do those intermediate timestamps have meaning?

## Conclusion

The distinction between `NOW()` and `clock_timestamp()` reflects a deeper truth about PostgreSQL's transaction model. `NOW()` gives you logical time—the moment your transaction's snapshot was taken. `clock_timestamp()` gives you physical time—what the server's clock actually reads.

Most of the time, `NOW()` is what you want. It plays nicely with MVCC semantics, ensures consistency within transactions, and matches the mental model of transactions as atomic operations. Reach for `clock_timestamp()` when you specifically need to measure elapsed time or track the physical progression of operations.

Understanding this distinction helps you write PostgreSQL code that works *with* the database's concurrency model rather than against it.