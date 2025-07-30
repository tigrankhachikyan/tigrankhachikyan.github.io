---
title: "PostgreSQL SELECT FOR UPDATE: The Complete Cheatsheet"
date: 2025-07-29T12:30:30Z
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
description: "Master row-level locking in PostgreSQL to prevent race conditions and ensure data consistency in concurrent applications."
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
# PostgreSQL SELECT FOR UPDATE: The Complete Cheatsheet

Master row-level locking in PostgreSQL to prevent race conditions and ensure data consistency in concurrent applications.

## What is SELECT FOR UPDATE?

`SELECT FOR UPDATE` places exclusive row-level locks on selected rows, preventing other transactions from modifying them until your transaction completes. Think of it as "reserving" rows for your exclusive use.

## Basic Syntax

```sql
SELECT columns FROM table WHERE conditions FOR UPDATE [options];
```

## Lock Types Comparison

| Lock Type | Syntax | Allows Reads | Allows Writes | Use Case |
|-----------|--------|--------------|---------------|----------|
| FOR UPDATE | `SELECT ... FOR UPDATE` | ❌ Other transactions wait | ❌ Blocked | Exclusive modification |
| FOR SHARE | `SELECT ... FOR SHARE` | ✅ Concurrent reads allowed | ❌ Blocked | Read consistency |
| FOR NO KEY UPDATE | `SELECT ... FOR NO KEY UPDATE` | ❌ Other transactions wait | ⚠️ Some updates allowed | Update non-key columns |
| FOR KEY SHARE | `SELECT ... FOR KEY SHARE` | ✅ Concurrent reads allowed | ⚠️ Key updates blocked | Foreign key references |

## Lock Behavior Options

### NOWAIT
Returns error immediately if rows are already locked:
```sql
SELECT * FROM products WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "products"
```

### SKIP LOCKED
Skips already-locked rows and continues:
```sql
SELECT * FROM queue_jobs 
WHERE status = 'pending' 
FOR UPDATE SKIP LOCKED 
LIMIT 5;
```

## Real-World Use Cases & Examples

### 1. Banking & Financial Transactions

**Problem**: Prevent overdrafts and race conditions in account transfers.

```sql
-- Safe money transfer between accounts
BEGIN;

-- Lock both accounts to prevent concurrent modifications
SELECT balance FROM accounts WHERE id IN (123, 456) FOR UPDATE;

-- Check sufficient funds
SELECT balance FROM accounts WHERE id = 123; -- Source account

-- Perform the transfer
UPDATE accounts SET balance = balance - 500 WHERE id = 123;
UPDATE accounts SET balance = balance + 500 WHERE id = 456;

COMMIT;
```

### 2. Inventory Management

**Problem**: Prevent overselling products when multiple customers order simultaneously.

```sql
-- Reserve inventory for an order
BEGIN;

-- Lock the product row
SELECT stock_quantity FROM products WHERE id = 789 FOR UPDATE;

-- Check availability and reserve
UPDATE products 
SET stock_quantity = stock_quantity - 3 
WHERE id = 789 AND stock_quantity >= 3;

-- If UPDATE affected 0 rows, rollback (insufficient stock)
COMMIT;
```

### 3. Job Queue Processing

**Problem**: Multiple workers competing for the same background jobs.

```sql
-- Worker claims next available job
BEGIN;

-- Get and lock the next pending job
SELECT id, payload FROM job_queue 
WHERE status = 'pending' 
ORDER BY created_at 
FOR UPDATE SKIP LOCKED 
LIMIT 1;

-- Mark as processing
UPDATE job_queue 
SET status = 'processing', worker_id = 'worker-123' 
WHERE id = :job_id;

COMMIT;
```

### 4. Sequence Generation

**Problem**: Generate custom sequence numbers without gaps.

```sql
-- Generate next invoice number
BEGIN;

SELECT last_invoice_number FROM invoice_sequences 
WHERE year = 2024 FOR UPDATE;

UPDATE invoice_sequences 
SET last_invoice_number = last_invoice_number + 1 
WHERE year = 2024;

-- Use the new number for invoice creation
INSERT INTO invoices (invoice_number, ...) 
VALUES ('INV-2024-001234', ...);

COMMIT;
```

### 5. Seat Reservation System

**Problem**: Prevent double-booking of seats or resources.

```sql
-- Reserve seats for a booking
BEGIN;

-- Check and lock available seats
SELECT seat_number FROM seats 
WHERE event_id = 100 
  AND seat_number IN ('A1', 'A2', 'A3')
  AND status = 'available'
FOR UPDATE NOWAIT;

-- Reserve the seats
UPDATE seats 
SET status = 'reserved', booking_id = 12345 
WHERE event_id = 100 
  AND seat_number IN ('A1', 'A2', 'A3');

COMMIT;
```

### 6. Rate Limiting

**Problem**: Implement per-user rate limiting with atomic counter updates.

```sql
-- Check and update rate limit counter
BEGIN;

SELECT request_count, window_start FROM rate_limits 
WHERE user_id = 'user123' 
  AND window_start = date_trunc('hour', now())
FOR UPDATE;

-- Increment counter or create new window
INSERT INTO rate_limits (user_id, window_start, request_count) 
VALUES ('user123', date_trunc('hour', now()), 1)
ON CONFLICT (user_id, window_start) 
DO UPDATE SET request_count = rate_limits.request_count + 1;

COMMIT;
```

## Performance Considerations

### When to Use Each Option

```sql
-- High contention: Use SKIP LOCKED for queue processing
SELECT * FROM tasks WHERE status = 'pending' 
FOR UPDATE SKIP LOCKED LIMIT 1;

-- Low contention but need immediate feedback: Use NOWAIT
SELECT * FROM exclusive_resource WHERE id = 1 
FOR UPDATE NOWAIT;

-- Default: Standard FOR UPDATE with timeout handling
SELECT * FROM accounts WHERE id = 123 FOR UPDATE;
```

### Avoiding Deadlocks

```sql
-- BAD: Different lock order in different transactions
-- Transaction 1: Lock account 123, then 456
-- Transaction 2: Lock account 456, then 123

-- GOOD: Always lock in consistent order
SELECT * FROM accounts 
WHERE id IN (123, 456) 
ORDER BY id  -- Consistent ordering prevents deadlocks
FOR UPDATE;
```

## Common Patterns

### Pessimistic Locking Pattern
```sql
BEGIN;
SELECT * FROM resource WHERE id = ? FOR UPDATE;
-- Check business rules
-- Perform updates
COMMIT;
```

### Optimistic Locking Alternative
```sql
-- Using version numbers instead of locks
UPDATE resource 
SET data = ?, version = version + 1 
WHERE id = ? AND version = ?;
-- Check if exactly 1 row was updated
```

### Conditional Processing
```sql
-- Process items only if conditions are met
WITH locked_items AS (
  SELECT id FROM processing_queue 
  WHERE status = 'ready' 
    AND retry_count < 3
  FOR UPDATE SKIP LOCKED
  LIMIT 10
)
UPDATE processing_queue 
SET status = 'processing' 
WHERE id IN (SELECT id FROM locked_items);
```

## Error Handling

```sql
-- Handle lock timeouts gracefully
BEGIN;

SELECT * FROM resource WHERE id = 1 FOR UPDATE;
-- Handle: ERROR: canceling statement due to lock timeout

-- Or use NOWAIT for immediate feedback
SELECT * FROM resource WHERE id = 1 FOR UPDATE NOWAIT;
-- Handle: ERROR: could not obtain lock on row
```

## Key Takeaways

✅ **Always use within transactions** - Locks are released on COMMIT/ROLLBACK  
✅ **Order your locks consistently** - Prevents deadlocks  
✅ **Use SKIP LOCKED for queues** - Better concurrency for job processing  
✅ **Consider NOWAIT for user-facing operations** - Avoid hanging requests  
✅ **Lock only what you need** - Minimize lock scope and duration  

❌ **Don't hold locks too long** - Impacts system performance  
❌ **Don't forget error handling** - Lock acquisition can fail  
❌ **Don't mix with application-level locks** - Can cause unexpected behavior

## Quick Reference

```sql
-- Basic exclusive lock
SELECT * FROM table WHERE id = 1 FOR UPDATE;

-- Non-blocking lock attempt  
SELECT * FROM table WHERE id = 1 FOR UPDATE NOWAIT;

-- Skip locked rows
SELECT * FROM table WHERE status = 'pending' FOR UPDATE SKIP LOCKED;

-- Shared lock (allows concurrent reads)
SELECT * FROM table WHERE id = 1 FOR SHARE;

-- Lock specific columns only
SELECT * FROM table WHERE id = 1 FOR UPDATE OF table;

-- Multiple tables
SELECT * FROM orders o JOIN items i ON o.id = i.order_id 
WHERE o.id = 1 FOR UPDATE OF o, i;
```

Master these patterns and you'll handle concurrent data access like a pro! 🚀