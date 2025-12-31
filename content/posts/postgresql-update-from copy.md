---
title: "PostgreSQL `tstzrange` Cheatsheet"
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
description: "Comprehensive guide to PostgreSQL `tstzrange` type with examples and use cases."
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
# PostgreSQL `tstzrange` Cheatsheet

The `tstzrange` type represents a range of `timestamptz` values—useful for modeling time intervals like bookings, schedules, or validity periods.

## Creating Ranges

```sql
-- Inclusive lower, exclusive upper (default)
SELECT tstzrange('2024-01-01', '2024-01-31');

-- Specify bounds explicitly: [] inclusive, () exclusive
SELECT tstzrange('2024-01-01', '2024-01-31', '[]');  -- both inclusive
SELECT tstzrange('2024-01-01', '2024-01-31', '[)');  -- lower inclusive, upper exclusive
SELECT tstzrange('2024-01-01', '2024-01-31', '()');  -- both exclusive

-- Unbounded ranges
SELECT tstzrange('2024-01-01', NULL);   -- from date to infinity
SELECT tstzrange(NULL, '2024-01-01');   -- from -infinity to date
```

## Range Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `@>` | Contains element | `range @> '2024-01-15'::timestamptz` |
| `@>` | Contains range | `range1 @> range2` |
| `<@` | Element/range is contained by | `'2024-01-15'::timestamptz <@ range` |
| `&&` | Overlaps | `range1 && range2` |
| `-|-` | Adjacent to | `range1 -|- range2` |
| `<<` | Strictly left of | `range1 << range2` |
| `>>` | Strictly right of | `range1 >> range2` |

## Extracting Bounds

```sql
SELECT lower(r), upper(r) FROM my_table;
SELECT lower_inc(r), upper_inc(r);  -- check if bounds are inclusive
SELECT lower_inf(r), upper_inf(r);  -- check if bounds are infinite
SELECT isempty(r);                   -- check for empty range
```

## Range Functions

```sql
-- Intersection
SELECT tstzrange('2024-01-01', '2024-01-31') * tstzrange('2024-01-15', '2024-02-15');
-- Result: ["2024-01-15","2024-01-31")

-- Union (ranges must overlap or be adjacent)
SELECT tstzrange('2024-01-01', '2024-01-31') + tstzrange('2024-01-31', '2024-02-15');

-- Difference
SELECT tstzrange('2024-01-01', '2024-01-31') - tstzrange('2024-01-15', '2024-01-20');
```

## Exclusion Constraints

Prevent overlapping ranges in a table (requires `btree_gist` extension):

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE bookings (
    id serial PRIMARY KEY,
    room_id int,
    during tstzrange,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

## Indexing

```sql
-- GiST index for range queries
CREATE INDEX idx_bookings_during ON bookings USING gist (during);

-- Useful for: @>, <@, &&, <<, >>, -|-
```

## Common Patterns

```sql
-- Find all active records at a specific time
SELECT * FROM events WHERE during @> NOW();

-- Find overlapping reservations
SELECT * FROM bookings WHERE during && tstzrange('2024-03-01', '2024-03-07');

-- Check if a slot is available
SELECT NOT EXISTS (
    SELECT 1 FROM bookings 
    WHERE room_id = 101 
    AND during && tstzrange('2024-03-01 14:00', '2024-03-01 16:00')
);
```

## Tips

- Always use `tstzrange` over `tsrange` when timezone awareness matters
- The default `[)` bound style (inclusive start, exclusive end) avoids off-by-one issues and makes adjacent ranges easy to define
- Empty ranges are valid: `tstzrange('2024-01-01', '2024-01-01', '()')` produces `empty`
- Use `COALESCE(lower(r), '-infinity')` when you need to handle unbounded ranges in comparisons