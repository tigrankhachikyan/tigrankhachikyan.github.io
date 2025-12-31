---
title: "SQLite Exclusion Constraints — The Practical Cheatsheet (with PostgreSQL Parallels)"
date: 2025-12-31T6:05:10Z
tags:
- sqlite
- exclusion constraints
- postgresql
- sql
- database design
- constraints
- cheatsheet
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "SQLite does not support `EXCLUDE` constraints. This is intentional. This guide explains what you lose, what you gain, and how to build equivalent guarantees using SQLite's primitives."
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
# SQLite Exclusion Constraints — The Practical Cheatsheet

SQLite does not support `EXCLUDE` constraints. This cheatsheet shows how to enforce range uniqueness and overlap prevention using pure SQL.

---

## What is an Exclusion Constraint?

An exclusion constraint prevents rows from coexisting when a condition is true. Unlike `UNIQUE` (equality only), `EXCLUDE` can enforce range overlaps.

### PostgreSQL Native Syntax

```sql
CREATE TABLE room_bookings (
    id SERIAL PRIMARY KEY,
    room_id INTEGER NOT NULL,
    during TSTZRANGE NOT NULL,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

**Meaning:** No two rows can have the same `room_id` with overlapping `during` ranges.

---

## Quick Comparison

| Feature | PostgreSQL | SQLite |
|---------|------------|--------|
| `EXCLUDE` constraint | Yes | No |
| Range types (`tstzrange`) | Yes | No |
| GiST indexes | Yes | No |
| Partial UNIQUE index | Yes | Yes |
| Triggers | Yes | Yes |

---

## SQLite Patterns

### Pattern 1: UNIQUE Index (Equality Only)

```sql
-- SQLite & PostgreSQL (identical)
CREATE TABLE api_keys (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    key_hash TEXT NOT NULL,
    UNIQUE(user_id, key_hash)
);
```

**Use when:** Simple equality exclusion.

---

### Pattern 2: Partial UNIQUE Index (Conditional Equality)

```sql
-- One active session per user
CREATE TABLE sessions (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE UNIQUE INDEX idx_one_active 
ON sessions(user_id) 
WHERE is_active = 1;
```

**Use when:** Uniqueness applies only to a subset of rows.

**Common examples:**

```sql
-- Soft-delete safe uniqueness
CREATE UNIQUE INDEX idx_unique_email 
ON users(email) 
WHERE deleted_at IS NULL;

-- One primary address per user
CREATE UNIQUE INDEX idx_primary_addr 
ON addresses(user_id) 
WHERE is_primary = 1;
```

---

### Pattern 3: Trigger-Based Range Exclusion

This is the SQLite equivalent of PostgreSQL's `EXCLUDE ... WITH &&`.

#### PostgreSQL

```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    resource_id INTEGER NOT NULL,
    during TSTZRANGE NOT NULL,
    EXCLUDE USING gist (resource_id WITH =, during WITH &&)
);
```

#### SQLite Equivalent

```sql
CREATE TABLE reservations (
    id INTEGER PRIMARY KEY,
    resource_id INTEGER NOT NULL,
    start_ts TEXT NOT NULL,
    end_ts TEXT NOT NULL,
    CHECK (start_ts < end_ts)
);

CREATE INDEX idx_overlap ON reservations(resource_id, start_ts, end_ts);

-- Prevent overlapping inserts
CREATE TRIGGER trg_no_overlap_insert
BEFORE INSERT ON reservations
BEGIN
    SELECT RAISE(ABORT, 'Overlapping reservation')
    WHERE EXISTS (
        SELECT 1 FROM reservations
        WHERE resource_id = NEW.resource_id
          AND start_ts < NEW.end_ts
          AND end_ts > NEW.start_ts
    );
END;

-- Prevent overlapping updates
CREATE TRIGGER trg_no_overlap_update
BEFORE UPDATE ON reservations
BEGIN
    SELECT RAISE(ABORT, 'Overlapping reservation')
    WHERE EXISTS (
        SELECT 1 FROM reservations
        WHERE resource_id = NEW.resource_id
          AND id != OLD.id
          AND start_ts < NEW.end_ts
          AND end_ts > NEW.start_ts
    );
END;
```

---

## Range Overlap Logic

Two ranges `[A.start, A.end)` and `[B.start, B.end)` overlap when:

```
A.start < B.end AND A.end > B.start
```

Visual:

```
Case 1: Overlap
A: |-------|
B:     |-------|
   A.start < B.end ✓
   A.end > B.start ✓

Case 2: No overlap  
A: |-------|
B:            |-------|
   A.start < B.end ✓
   A.end > B.start ✗
```

---

## Real-World Examples

### Employee Shift Scheduling

```sql
-- PostgreSQL
CREATE TABLE shifts (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER NOT NULL,
    during TSTZRANGE NOT NULL,
    EXCLUDE USING gist (employee_id WITH =, during WITH &&)
);

-- SQLite
CREATE TABLE shifts (
    id INTEGER PRIMARY KEY,
    employee_id INTEGER NOT NULL,
    start_time TEXT NOT NULL,
    end_time TEXT NOT NULL
);

CREATE TRIGGER trg_shift_no_overlap
BEFORE INSERT ON shifts
BEGIN
    SELECT RAISE(ABORT, 'Shift overlaps existing')
    WHERE EXISTS (
        SELECT 1 FROM shifts
        WHERE employee_id = NEW.employee_id
          AND start_time < NEW.end_time
          AND end_time > NEW.start_time
    );
END;
```

### Subscription Periods

```sql
-- PostgreSQL
CREATE TABLE subscriptions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    plan_id INTEGER NOT NULL,
    active_during TSTZRANGE NOT NULL,
    EXCLUDE USING gist (user_id WITH =, plan_id WITH =, active_during WITH &&)
);

-- SQLite
CREATE TABLE subscriptions (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    plan_id INTEGER NOT NULL,
    starts_at TEXT NOT NULL,
    ends_at TEXT
);

CREATE TRIGGER trg_sub_no_overlap
BEFORE INSERT ON subscriptions
BEGIN
    SELECT RAISE(ABORT, 'Subscription period overlaps')
    WHERE EXISTS (
        SELECT 1 FROM subscriptions
        WHERE user_id = NEW.user_id
          AND plan_id = NEW.plan_id
          AND starts_at < COALESCE(NEW.ends_at, '9999-12-31')
          AND COALESCE(ends_at, '9999-12-31') > NEW.starts_at
    );
END;
```

### Meeting Room Bookings

```sql
-- PostgreSQL
CREATE TABLE bookings (
    id SERIAL PRIMARY KEY,
    room_id INTEGER NOT NULL,
    booked TSTZRANGE NOT NULL,
    EXCLUDE USING gist (room_id WITH =, booked WITH &&)
);

-- SQLite
CREATE TABLE bookings (
    id INTEGER PRIMARY KEY,
    room_id INTEGER NOT NULL,
    start_at TEXT NOT NULL,
    end_at TEXT NOT NULL
);

CREATE INDEX idx_booking_range ON bookings(room_id, start_at, end_at);

CREATE TRIGGER trg_booking_no_overlap
BEFORE INSERT ON bookings
BEGIN
    SELECT RAISE(ABORT, 'Room already booked')
    WHERE EXISTS (
        SELECT 1 FROM bookings
        WHERE room_id = NEW.room_id
          AND start_at < NEW.end_at
          AND end_at > NEW.start_at
    );
END;
```

---

## NULL Handling

SQLite treats `NULL != NULL` in UNIQUE constraints.

```sql
-- Multiple NULLs allowed (both databases)
CREATE TABLE t (val TEXT UNIQUE);
INSERT INTO t VALUES (NULL);
INSERT INTO t VALUES (NULL);  -- Succeeds

-- To treat NULL as unique value:
CREATE UNIQUE INDEX idx ON t(COALESCE(val, '__NULL__'));
```

---

## Performance Notes

| Method | Index Required | Complexity |
|--------|----------------|------------|
| PostgreSQL EXCLUDE | GiST (automatic) | O(log n) |
| SQLite Trigger | B-tree (manual) | O(log n)* |

*With proper index on `(partition_key, start, end)`.

---

## Quick Reference

```
┌────────────────────────────────────────────────────────┐
│ PostgreSQL EXCLUDE          │ SQLite Equivalent        │
├────────────────────────────────────────────────────────┤
│ EXCLUDE (a WITH =)          │ UNIQUE(a)                │
│ EXCLUDE (a WITH =)          │ UNIQUE INDEX ... WHERE   │
│   WHERE condition           │   condition              │
│ EXCLUDE (a WITH =,          │ BEFORE INSERT/UPDATE     │
│          r WITH &&)         │   trigger + index        │
└────────────────────────────────────────────────────────┘

Overlap condition: start_ts < NEW.end_ts AND end_ts > NEW.start_ts
```

---

## When to Use PostgreSQL Instead

- High write volume (GiST is faster than trigger scans)
- Complex range types (geometric, network ranges)
- Need declarative single-line constraints
- Deferred constraint checking required

For most SQLite use cases—embedded, edge, Cloudflare D1—triggers are sufficient.