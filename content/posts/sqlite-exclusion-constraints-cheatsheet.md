---
title: "SQLite Exclusion Constraints — The Practical Cheatsheet (with PostgreSQL Parallels)"
date: 2025-12-17T6:05:10Z
tags:
- postgresql
- sqlite
- cloudflare d1
- sql
- count
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
# SQLite Exclusion Constraints — The Practical Cheatsheet (with PostgreSQL Parallels)

SQLite does not support `EXCLUDE` constraints. This is intentional. This guide explains what you lose, what you gain, and how to build equivalent guarantees using SQLite's primitives.

---

## 1. What is an Exclusion Constraint?

An exclusion constraint prevents rows from existing simultaneously when a specified condition evaluates to true. Unlike `UNIQUE` (which only handles equality), exclusion constraints can enforce arbitrary predicates—most commonly range overlaps.

### PostgreSQL Example

```sql
CREATE TABLE room_bookings (
    id SERIAL PRIMARY KEY,
    room_id INTEGER NOT NULL,
    booked_during TSTZRANGE NOT NULL,
    EXCLUDE USING gist (room_id WITH =, booked_during WITH &&)
);
```

This constraint guarantees: **no two rows can have the same `room_id` with overlapping `booked_during` ranges**.

### Plain English

Think of `EXCLUDE` as a generalized uniqueness rule:

| Constraint Type | What It Enforces |
|-----------------|------------------|
| `UNIQUE(a, b)` | No two rows where `a1 = a2 AND b1 = b2` |
| `EXCLUDE ... WITH =, WITH &&` | No two rows where `a1 = a2 AND b1 && b2` (overlaps) |

The `WITH` clause specifies operators. `=` means equality. `&&` means range overlap. You can use any operator that has a GiST-compatible strategy.

---

## 2. SQLite vs PostgreSQL Philosophy

SQLite is deliberately minimal. It prioritizes simplicity, portability, and zero-configuration deployment over advanced constraint types.

### Feature Comparison

| Feature | PostgreSQL | SQLite |
|---------|------------|--------|
| `EXCLUDE` constraint | Yes | No |
| GiST / GIN indexes | Yes | No |
| Range types (`tstzrange`, etc.) | Yes | No |
| Partial indexes | Yes | Yes |
| `UNIQUE` constraints | Yes | Yes |
| Triggers | Yes | Yes |
| Deferred constraints | Yes | Limited |
| CHECK constraints | Yes | Yes |

### Why SQLite Avoids EXCLUDE

1. **No extensible index types** — GiST is required for efficient overlap checks; SQLite only has B-tree
2. **Philosophy** — SQLite avoids features that require complex query planner integration
3. **Use case alignment** — Most SQLite deployments don't need range exclusion
4. **Workarounds exist** — Triggers and partial indexes cover 90% of real needs

---

## 3. SQLite Exclusion Patterns

### Pattern 1: UNIQUE Index (Equality Exclusion)

**When to use:** You need standard uniqueness—no two rows with identical values in specified columns.

**What it enforces:** `NOT EXISTS (SELECT 1 FROM t WHERE t.a = new.a AND t.b = new.b)`

#### SQLite Example

```sql
CREATE TABLE api_keys (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    key_hash TEXT NOT NULL,
    UNIQUE(user_id, key_hash)
);
```

#### PostgreSQL Equivalent

```sql
-- Identical syntax
CREATE TABLE api_keys (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    key_hash TEXT NOT NULL,
    UNIQUE(user_id, key_hash)
);
```

| Pros | Cons |
|------|------|
| Native, fast, zero overhead | Only supports equality |
| Index-backed enforcement | Cannot express overlap, range, or custom predicates |
| Standard SQL |  |

---

### Pattern 2: Partial UNIQUE Index (Conditional Exclusion)

**When to use:** Uniqueness should only apply to a subset of rows (e.g., only "active" records).

**What it enforces:** Uniqueness within rows matching a `WHERE` clause.

#### SQLite Example

```sql
-- Only one active session per user
CREATE UNIQUE INDEX idx_one_active_session
ON sessions(user_id)
WHERE is_active = 1;
```

#### PostgreSQL Equivalent

```sql
-- Identical syntax
CREATE UNIQUE INDEX idx_one_active_session
ON sessions(user_id)
WHERE is_active = true;
```

| Pros | Cons |
|------|------|
| Declarative, index-backed | Still equality-only |
| Replaces many `EXCLUDE` use cases | Predicate must be deterministic |
| Well-optimized in SQLite |  |

---

### Pattern 3: Trigger-Based Exclusion (Range / Overlap)

**When to use:** You need to prevent overlapping ranges, or any condition beyond equality.

**What it enforces:** Any arbitrary predicate you can express in SQL.

#### SQLite Example

```sql
CREATE TABLE room_bookings (
    id INTEGER PRIMARY KEY,
    room_id INTEGER NOT NULL,
    start_time TEXT NOT NULL,  -- ISO8601
    end_time TEXT NOT NULL
);

CREATE INDEX idx_bookings_room_time ON room_bookings(room_id, start_time, end_time);

CREATE TRIGGER trg_no_overlap_insert
BEFORE INSERT ON room_bookings
BEGIN
    SELECT RAISE(ABORT, 'Overlapping booking exists')
    WHERE EXISTS (
        SELECT 1 FROM room_bookings
        WHERE room_id = NEW.room_id
          AND start_time < NEW.end_time
          AND end_time > NEW.start_time
    );
END;

CREATE TRIGGER trg_no_overlap_update
BEFORE UPDATE ON room_bookings
BEGIN
    SELECT RAISE(ABORT, 'Overlapping booking exists')
    WHERE EXISTS (
        SELECT 1 FROM room_bookings
        WHERE room_id = NEW.room_id
          AND id != NEW.id
          AND start_time < NEW.end_time
          AND end_time > NEW.start_time
    );
END;
```

#### PostgreSQL Equivalent

```sql
CREATE TABLE room_bookings (
    id SERIAL PRIMARY KEY,
    room_id INTEGER NOT NULL,
    booked_during TSTZRANGE NOT NULL,
    EXCLUDE USING gist (room_id WITH =, booked_during WITH &&)
);
```

| Pros | Cons |
|------|------|
| Full flexibility | Manual implementation |
| Works for any predicate | Two triggers needed (INSERT + UPDATE) |
| Explicit, auditable logic | Performance depends on index design |

---

### Pattern 4: Application-Level Exclusion (Transaction-Based)

**When to use:** When database-level enforcement is impractical, or you need cross-table/cross-database checks.

**What it enforces:** Whatever your application code validates.

#### SQLite Example (Python)

```python
def create_booking(conn, room_id, start, end):
    with conn:
        # Acquire exclusive lock
        conn.execute("BEGIN EXCLUSIVE")
        
        conflict = conn.execute("""
            SELECT 1 FROM room_bookings
            WHERE room_id = ?
              AND start_time < ?
              AND end_time > ?
            LIMIT 1
        """, (room_id, end, start)).fetchone()
        
        if conflict:
            raise ValueError("Overlapping booking")
        
        conn.execute("""
            INSERT INTO room_bookings (room_id, start_time, end_time)
            VALUES (?, ?, ?)
        """, (room_id, start, end))
        
        conn.commit()
```

| Pros | Cons |
|------|------|
| Maximum flexibility | Constraint lives outside database |
| Works across databases | Easy to bypass |
| Can integrate with business logic | Requires careful transaction handling |

---

## 4. Partial UNIQUE Indexes in SQLite

Partial unique indexes are SQLite's most underutilized feature. They solve a large class of problems that would otherwise require `EXCLUDE` in PostgreSQL.

### Syntax

```sql
CREATE UNIQUE INDEX index_name ON table(columns) WHERE condition;
```

### Why They Replace Many EXCLUDE Constraints

Most "exclusion" requirements in practice are:
- Uniqueness among "active" records
- Uniqueness among non-deleted records  
- Uniqueness for a specific status/type

These are all expressible as partial uniqueness.

### Real-World Examples

#### One Active Session Per User

```sql
CREATE TABLE sessions (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    token TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL
);

-- Only one active session allowed per user
CREATE UNIQUE INDEX idx_single_active_session
ON sessions(user_id)
WHERE is_active = 1;
```

This allows unlimited inactive sessions but enforces exactly one active.

#### Soft Deletes with Uniqueness

```sql
CREATE TABLE email_addresses (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    email TEXT NOT NULL,
    deleted_at TEXT  -- NULL means not deleted
);

-- Unique email only among non-deleted rows
CREATE UNIQUE INDEX idx_unique_active_email
ON email_addresses(email)
WHERE deleted_at IS NULL;
```

A user can delete an email and re-add the same one later.

#### Primary Flag Pattern

```sql
CREATE TABLE addresses (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    street TEXT NOT NULL,
    is_primary INTEGER NOT NULL DEFAULT 0
);

-- Only one primary address per user
CREATE UNIQUE INDEX idx_one_primary_address
ON addresses(user_id)
WHERE is_primary = 1;
```

Users can have many addresses, but only one marked primary.

---

## 5. Range / Overlap Exclusion

This is where SQLite requires the most work. PostgreSQL's `EXCLUDE USING gist` with `&&` handles this natively; SQLite needs triggers.

### Complete Trigger Example

```sql
CREATE TABLE reservations (
    id INTEGER PRIMARY KEY,
    resource_id INTEGER NOT NULL,
    start_ts TEXT NOT NULL,  -- ISO8601 timestamp
    end_ts TEXT NOT NULL,
    CONSTRAINT valid_range CHECK (start_ts < end_ts)
);

-- Critical: index for overlap queries
CREATE INDEX idx_res_overlap 
ON reservations(resource_id, end_ts, start_ts);

-- Prevent overlapping inserts
CREATE TRIGGER trg_reservation_no_overlap_insert
BEFORE INSERT ON reservations
BEGIN
    SELECT RAISE(ABORT, 'Reservation overlaps with existing')
    WHERE EXISTS (
        SELECT 1 FROM reservations
        WHERE resource_id = NEW.resource_id
          AND start_ts < NEW.end_ts
          AND end_ts > NEW.start_ts
    );
END;

-- Prevent overlapping updates
CREATE TRIGGER trg_reservation_no_overlap_update
BEFORE UPDATE OF resource_id, start_ts, end_ts ON reservations
BEGIN
    SELECT RAISE(ABORT, 'Reservation overlaps with existing')
    WHERE EXISTS (
        SELECT 1 FROM reservations
        WHERE resource_id = NEW.resource_id
          AND id != OLD.id
          AND start_ts < NEW.end_ts
          AND end_ts > NEW.start_ts
    );
END;
```

### Required Indexes

Without proper indexing, overlap checks become O(n) table scans.

| Index Strategy | Query Pattern | Notes |
|----------------|---------------|-------|
| `(resource_id, end_ts, start_ts)` | Allows range filtering | Best for overlap queries |
| `(resource_id, start_ts)` | Simple range scans | Less optimal |

### PostgreSQL Comparison

| Aspect | PostgreSQL | SQLite |
|--------|------------|--------|
| Declaration | Single `EXCLUDE` line | Two triggers + index |
| Index type | GiST (R-tree for ranges) | B-tree (less efficient) |
| Range types | Native `tstzrange` | Text/INTEGER simulation |
| Maintenance | Automatic | Manual trigger updates |
| Performance | O(log n) via GiST | O(log n) via B-tree (with good index) |

---

## 6. NULL Behavior Differences

SQLite and PostgreSQL differ in how `NULL` interacts with `UNIQUE` constraints.

### SQLite: NULL != NULL (for UNIQUE)

```sql
CREATE TABLE example (
    id INTEGER PRIMARY KEY,
    nullable_col TEXT,
    UNIQUE(nullable_col)
);

INSERT INTO example (nullable_col) VALUES (NULL);
INSERT INTO example (nullable_col) VALUES (NULL);  -- Succeeds!
```

Multiple `NULL` values are allowed because `NULL != NULL`.

### PostgreSQL: Same Behavior (by default)

PostgreSQL matches SQLite here. However, PostgreSQL 15+ offers `NULLS NOT DISTINCT`:

```sql
CREATE TABLE example (
    id SERIAL PRIMARY KEY,
    nullable_col TEXT,
    UNIQUE NULLS NOT DISTINCT (nullable_col)
);
```

### Implications for Exclusion Logic

| Scenario | SQLite Behavior | Design Solution |
|----------|-----------------|-----------------|
| UNIQUE on nullable column | Multiple NULLs allowed | Use `COALESCE` in partial index |
| Trigger-based exclusion | NULL comparisons return NULL (falsy) | Explicit `IS NOT NULL` checks |
| Partial index condition | `WHERE col IS NULL` is valid | Can be used strategically |

### Designing Around NULL

```sql
-- If you want NULL to be treated as a unique value:
CREATE UNIQUE INDEX idx_unique_including_null
ON example(COALESCE(nullable_col, '__NULL_SENTINEL__'));

-- Or use partial index to exclude NULLs entirely:
CREATE UNIQUE INDEX idx_unique_non_null
ON example(nullable_col)
WHERE nullable_col IS NOT NULL;
```

---

## 7. Performance and Correctness Comparison

| Method | Write Performance | Read Impact | Correctness Guarantee | Maintenance Burden |
|--------|-------------------|-------------|----------------------|-------------------|
| UNIQUE index | Excellent | Index lookup on insert | Database-enforced | None |
| Partial UNIQUE index | Excellent | Index lookup (filtered) | Database-enforced | None |
| BEFORE trigger | Good | Depends on query | Database-enforced | Low |
| Application logic | Variable | None | Application-dependent | High |
| PostgreSQL EXCLUDE | Excellent | GiST lookup | Database-enforced | None |

### When Each Approach Fails

| Method | Failure Mode |
|--------|--------------|
| UNIQUE index | Cannot express non-equality |
| Partial UNIQUE index | Predicate must be simple, deterministic |
| Trigger | Missing UPDATE trigger; poor index design |
| Application logic | Transaction isolation bugs; bypass via raw SQL |

---

## 8. Migration Mental Model

When moving from PostgreSQL to SQLite, use this decision process:

### Decision Checklist

```
1. Is this an equality-only exclusion?
   ├─ Yes → Use UNIQUE index
   └─ No → Continue

2. Is this equality within a subset of rows?
   ├─ Yes → Use partial UNIQUE index
   └─ No → Continue

3. Is this range/overlap exclusion?
   ├─ Yes → Use trigger + appropriate index
   └─ No → Continue

4. Is this a complex multi-table constraint?
   ├─ Yes → Consider application-level enforcement
   └─ No → Re-evaluate requirements
```

### Translation Examples

| PostgreSQL | SQLite Equivalent |
|------------|-------------------|
| `UNIQUE(a, b)` | `UNIQUE(a, b)` — identical |
| `UNIQUE(a) WHERE deleted_at IS NULL` | `CREATE UNIQUE INDEX ... WHERE deleted_at IS NULL` — identical |
| `EXCLUDE USING gist (a WITH =, b WITH &&)` | Trigger checking `a = a AND ranges overlap` |
| `EXCLUDE USING gist (point WITH ~=)` | Trigger with proximity calculation |

---

## 9. Real-World SQLite Examples

### Webhook Deduplication

Prevent processing the same webhook event twice within a time window.

```sql
CREATE TABLE webhook_events (
    id INTEGER PRIMARY KEY,
    event_id TEXT NOT NULL,
    source TEXT NOT NULL,
    received_at TEXT NOT NULL DEFAULT (datetime('now')),
    processed INTEGER NOT NULL DEFAULT 0
);

-- Exact deduplication: same event_id + source
CREATE UNIQUE INDEX idx_webhook_dedup
ON webhook_events(source, event_id);

-- If you need time-windowed deduplication, use a trigger:
CREATE TRIGGER trg_webhook_time_dedup
BEFORE INSERT ON webhook_events
BEGIN
    SELECT RAISE(ABORT, 'Duplicate webhook within window')
    WHERE EXISTS (
        SELECT 1 FROM webhook_events
        WHERE source = NEW.source
          AND event_id = NEW.event_id
          AND received_at > datetime('now', '-1 hour')
    );
END;
```

### One Active Subscription Per User Per Product

```sql
CREATE TABLE subscriptions (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('active', 'cancelled', 'expired')),
    started_at TEXT NOT NULL,
    ended_at TEXT
);

-- Only one active subscription per user/product combination
CREATE UNIQUE INDEX idx_one_active_sub
ON subscriptions(user_id, product_id)
WHERE status = 'active';
```

### Soft-Delete-Safe Uniqueness

```sql
CREATE TABLE organizations (
    id INTEGER PRIMARY KEY,
    slug TEXT NOT NULL,
    name TEXT NOT NULL,
    deleted_at TEXT  -- NULL = active
);

-- Slug must be unique among non-deleted orgs
CREATE UNIQUE INDEX idx_org_slug_active
ON organizations(slug)
WHERE deleted_at IS NULL;

-- Allows: org A with slug "acme" deleted, org B with slug "acme" active
```

### Non-Overlapping Employee Assignments

```sql
CREATE TABLE assignments (
    id INTEGER PRIMARY KEY,
    employee_id INTEGER NOT NULL,
    project_id INTEGER NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT,  -- NULL = ongoing
    CONSTRAINT valid_dates CHECK (end_date IS NULL OR start_date < end_date)
);

CREATE INDEX idx_assignment_overlap
ON assignments(employee_id, start_date, end_date);

CREATE TRIGGER trg_no_assignment_overlap_insert
BEFORE INSERT ON assignments
BEGIN
    SELECT RAISE(ABORT, 'Employee has overlapping assignment')
    WHERE EXISTS (
        SELECT 1 FROM assignments
        WHERE employee_id = NEW.employee_id
          AND (
            -- Existing ongoing overlaps any new range
            (end_date IS NULL AND (NEW.end_date IS NULL OR start_date < NEW.end_date))
            OR
            -- Existing bounded overlaps new range
            (end_date IS NOT NULL AND start_date < COALESCE(NEW.end_date, '9999-12-31') AND end_date > NEW.start_date)
          )
    );
END;

CREATE TRIGGER trg_no_assignment_overlap_update
BEFORE UPDATE OF employee_id, start_date, end_date ON assignments
BEGIN
    SELECT RAISE(ABORT, 'Employee has overlapping assignment')
    WHERE EXISTS (
        SELECT 1 FROM assignments
        WHERE employee_id = NEW.employee_id
          AND id != OLD.id
          AND (
            (end_date IS NULL AND (NEW.end_date IS NULL OR start_date < NEW.end_date))
            OR
            (end_date IS NOT NULL AND start_date < COALESCE(NEW.end_date, '9999-12-31') AND end_date > NEW.start_date)
          )
    );
END;
```

---

## 10. Final Takeaway

### SQLite's Primitives Are Sufficient When:

- Your exclusion is based on **equality** → `UNIQUE` index
- Your exclusion is **conditional equality** → Partial `UNIQUE` index
- Your exclusion involves **range overlap** and you're willing to write triggers
- You have **low to moderate write volume** (trigger overhead is acceptable)
- You're running in **embedded/edge environments** where SQLite is the only option

### PostgreSQL EXCLUDE Is Superior When:

- You need **GiST-backed range types** for true O(log n) overlap checks at scale
- You have **complex geometric or temporal exclusions** (nearest-neighbor, polygon overlap)
- You want **declarative, single-line constraint definitions**
- Your write volume is **high** and trigger overhead matters
- You need **deferred constraint checking** within transactions

### The Pragmatic View

SQLite's lack of `EXCLUDE` is rarely a blocker. For most applications:

1. Partial unique indexes handle 70% of cases
2. Simple triggers handle 25% more
3. The remaining 5% either need PostgreSQL or can tolerate application-level checks

Build with what SQLite gives you. When you outgrow it, you'll know.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    SQLite Exclusion Patterns                     │
├─────────────────────────────────────────────────────────────────┤
│ Equality exclusion     │ UNIQUE(col1, col2)                     │
│ Conditional equality   │ CREATE UNIQUE INDEX ... WHERE cond     │
│ Range overlap          │ BEFORE INSERT/UPDATE trigger           │
│ Complex predicates     │ Trigger or application logic           │
├─────────────────────────────────────────────────────────────────┤
│ NULL handling          │ NULL != NULL in UNIQUE                 │
│ Fix: exclude NULLs     │ ... WHERE col IS NOT NULL              │
│ Fix: treat NULL unique │ COALESCE(col, sentinel)                │
├─────────────────────────────────────────────────────────────────┤
│ Trigger template:                                                │
│   CREATE TRIGGER trg_name BEFORE INSERT ON tbl                  │
│   BEGIN                                                          │
│     SELECT RAISE(ABORT, 'msg') WHERE EXISTS (...);              │
│   END;                                                           │
└─────────────────────────────────────────────────────────────────┘
```