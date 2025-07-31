---
title: "ACID Properties in RDBMS: SQLite vs PostgreSQL"
date: 2025-07-29T11:39:30Z
tags:
- postgresql
- sql
- sqlite
- rdbms
- database
- acid
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Explore the ACID properties of RDBMS with a detailed comparison of SQLite and PostgreSQL. Understand Atomicity, Consistency, Isolation, and Durability in practical scenarios."
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
# RDBMS ACID Properties: Technical Guide with SQLite vs PostgreSQL

ACID properties form the foundation of reliable database systems, ensuring data integrity through precise transaction management mechanisms.

## **A - Atomicity**

**Technical Definition**: 
Atomicity guarantees that a transaction is an indivisible unit of work. The transaction either commits entirely or aborts completely, with no partial state changes persisting in the database.

**Implementation Mechanisms**:
- **Transaction Logs**: Record all modifications before applying them
- **Shadow Paging**: Maintain duplicate pages during transaction execution
- **Write-Ahead Logging (WAL)**: Log changes before modifying data pages

**Example**:
```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A001';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'A002';
INSERT INTO transactions (from_account, to_account, amount, timestamp) 
VALUES ('A001', 'A002', 100, CURRENT_TIMESTAMP);
COMMIT;
```

### **SQLite vs PostgreSQL - Atomicity Implementation**

| Aspect | SQLite | PostgreSQL |
|--------|--------|------------|
| **Transaction Model** | File-level locking with rollback journal | MVCC with WAL |
| **Recovery Mechanism** | Journal replay on crash | WAL replay with undo/redo logs |
| **Savepoint Support** | SAVEPOINT/RELEASE/ROLLBACK TO | Full nested transaction support |
| **Concurrent Transactions** | Serialized execution | Parallel execution with isolation |

**SQLite Implementation**:
```sql
-- SQLite uses database-level locking
BEGIN IMMEDIATE TRANSACTION;  -- Acquires reserved lock
UPDATE inventory SET stock = stock - 5 WHERE product_id = 101;
UPDATE order_details SET quantity = 5 WHERE order_id = 1001;
-- If any operation fails, rollback journal restores original state
COMMIT;  -- Releases locks and deletes journal
```

**PostgreSQL Implementation**:
```sql
-- PostgreSQL uses MVCC for concurrent atomicity
BEGIN;
SAVEPOINT inventory_update;
UPDATE inventory SET stock = stock - 5 WHERE product_id = 101;
-- Complex logic with potential rollback
IF (SELECT stock FROM inventory WHERE product_id = 101) < 0 THEN
    ROLLBACK TO SAVEPOINT inventory_update;
    RAISE EXCEPTION 'Insufficient inventory';
END IF;
RELEASE SAVEPOINT inventory_update;
UPDATE order_details SET quantity = 5 WHERE order_id = 1001;
COMMIT;
```

**Key Technical Differences**:
- **SQLite**: Single-writer model ensures atomicity through exclusive locking
- **PostgreSQL**: MVCC allows concurrent writers while maintaining atomicity per transaction

## **C - Consistency**

**Technical Definition**:
Consistency ensures that any transaction brings the database from one valid state to another, maintaining all defined integrity constraints, triggers, and business rules.

**Constraint Types**:
- **Entity Constraints**: Primary keys, unique constraints
- **Referential Constraints**: Foreign keys, cascade operations
- **Domain Constraints**: Data types, check constraints, not null
- **User-Defined Constraints**: Custom business logic via triggers/functions

### **SQLite vs PostgreSQL - Consistency Enforcement**

| Feature | SQLite | PostgreSQL |
|---------|--------|------------|
| **Foreign Key Support** | Optional (PRAGMA foreign_keys=ON) | Always enforced |
| **Check Constraints** | Basic support | Full expression support |
| **Triggers** | BEFORE/AFTER/INSTEAD OF | BEFORE/AFTER + statement/row level |
| **Custom Functions** | Limited C extensions | Rich PL/pgSQL, Python, etc. |
| **Deferred Constraints** | Not supported | DEFERRABLE constraints |

**SQLite Consistency Example**:
```sql
PRAGMA foreign_keys = ON;

CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_total DECIMAL(10,2) CHECK (order_total > 0),
    order_date DATE DEFAULT CURRENT_DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Trigger for business logic
CREATE TRIGGER update_customer_stats
AFTER INSERT ON orders
BEGIN
    UPDATE customers 
    SET total_orders = total_orders + 1,
        lifetime_value = lifetime_value + NEW.order_total
    WHERE customer_id = NEW.customer_id;
END;
```

**PostgreSQL Consistency Example**:
```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(customer_id),
    order_total DECIMAL(10,2) CHECK (order_total > 0),
    order_date DATE DEFAULT CURRENT_DATE,
    status order_status_enum DEFAULT 'pending',
    -- Complex constraint
    CONSTRAINT valid_rush_order CHECK (
        (rush_delivery = false) OR 
        (rush_delivery = true AND order_total >= 50.00)
    )
);

-- Advanced trigger with PL/pgSQL
CREATE OR REPLACE FUNCTION update_customer_metrics()
RETURNS TRIGGER AS $$
BEGIN
    -- Complex business logic
    UPDATE customer_analytics 
    SET total_orders = total_orders + 1,
        avg_order_value = (
            SELECT AVG(order_total) 
            FROM orders 
            WHERE customer_id = NEW.customer_id
        ),
        last_order_date = NEW.order_date
    WHERE customer_id = NEW.customer_id;
    
    -- Conditional logic
    IF NEW.order_total > 1000 THEN
        INSERT INTO high_value_customers (customer_id, order_id)
        VALUES (NEW.customer_id, NEW.order_id);
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER customer_metrics_trigger
    AFTER INSERT ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_customer_metrics();
```

## **I - Isolation**

**Technical Definition**:
Isolation ensures that concurrent execution of transactions leaves the database in the same state as if transactions were executed sequentially.

**Isolation Phenomena**:
- **Dirty Read**: Reading uncommitted changes from other transactions
- **Non-repeatable Read**: Getting different values when reading the same data twice
- **Phantom Read**: Different sets of rows returned for the same query
- **Serialization Anomaly**: Results inconsistent with any serial execution

**Standard Isolation Levels**:
1. **READ UNCOMMITTED**: No isolation guarantees
2. **READ COMMITTED**: Prevents dirty reads
3. **REPEATABLE READ**: Prevents dirty and non-repeatable reads  
4. **SERIALIZABLE**: Prevents all phenomena

### **SQLite vs PostgreSQL - Isolation Implementation**

| Aspect | SQLite | PostgreSQL |
|--------|--------|------------|
| **Concurrency Control** | Reader-writer locks | MVCC (Snapshot Isolation) |
| **Available Isolation Levels** | SERIALIZABLE only | All four standard levels |
| **Read Consistency** | Point-in-time snapshot | MVCC snapshots |
| **Lock Granularity** | Database-level | Row-level with MVCC |
| **Deadlock Handling** | Not applicable (single writer) | Detection and resolution |

**SQLite Isolation Behavior**:
```sql
-- SQLite WAL mode allows concurrent readers
BEGIN IMMEDIATE;  -- Acquires write lock immediately
SELECT COUNT(*) FROM orders WHERE status = 'pending';
-- Other transactions can read but not write
UPDATE orders SET status = 'processing' WHERE order_id = 1001;
-- All other write transactions are blocked
COMMIT;  -- Releases write lock
```

**PostgreSQL Isolation Control**:
```sql
-- Fine-grained isolation control
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;

-- Establish snapshot
SELECT balance FROM accounts WHERE account_id = 'A001';  -- Reads: $1000

-- Concurrent transaction modifies the same account
-- This transaction still sees $1000 due to MVCC

UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A001';
-- Uses the snapshot value ($1000) for the update

COMMIT;  -- May fail if conflicting updates occurred
```

**Advanced PostgreSQL Isolation Example**:
```sql
-- Serializable isolation with conflict detection
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Transaction 1
SELECT SUM(quantity) FROM inventory WHERE category = 'electronics';
INSERT INTO daily_reports (category, total_qty, report_date)
VALUES ('electronics', (SELECT SUM(quantity) FROM inventory WHERE category = 'electronics'), CURRENT_DATE);

-- If another serializable transaction modifies electronics inventory concurrently,
-- PostgreSQL will detect the serialization conflict and abort one transaction
COMMIT;  -- May raise serialization_failure error
```

## **D - Durability**

**Technical Definition**:
Durability guarantees that once a transaction commits, its changes are permanently persisted and will survive any subsequent system failure, including crashes, power outages, or hardware failures.

**Core Durability Mechanisms**:

**1. Write-Ahead Logging (WAL)**:
- Transaction changes are logged to stable storage **before** being applied to data files
- Ensures recoverability by maintaining a complete record of all modifications
- Log records contain both old values (undo information) and new values (redo information)

**2. Force-Write Policy**:
- Log records must be physically written to disk before transaction commit
- Uses synchronous I/O operations (fsync, fdatasync) to guarantee disk persistence
- Creates ordering constraint: log-write → disk-sync → commit-response

**3. Recovery Process**:
- **Redo Phase**: Replay committed transactions from the log to restore changes
- **Undo Phase**: Roll back uncommitted transactions found in the log
- **Checkpointing**: Periodic background process that flushes dirty pages to reduce recovery time

**4. Storage Layer Guarantees**:
- Modern databases must handle write reordering by disk controllers and OS caches
- Requires barrier/flush operations to ensure write ordering
- Critical for maintaining log-before-data invariant

### **SQLite vs PostgreSQL - Durability Implementation**

| Aspect | SQLite | PostgreSQL |
|--------|--------|------------|
| **Logging Mechanism** | Rollback journal or WAL mode | Always WAL-based |
| **Crash Recovery** | Automatic on next connection | Automatic on startup |
| **Sync Granularity** | Database-level control | Per-transaction control |
| **Recovery Time** | Fast (single file) | Variable (depends on WAL size) |
| **Durability Modes** | 4 synchronous levels | Fine-grained commit control |

**SQLite Durability Deep Dive**:
```sql
-- SQLite's durability model
PRAGMA journal_mode = WAL;        -- Enable Write-Ahead Logging
PRAGMA synchronous = FULL;        -- Wait for OS to confirm disk write

BEGIN IMMEDIATE;
INSERT INTO orders (customer_id, amount) VALUES (123, 99.99);
-- At this point:
-- 1. Change is written to WAL file
-- 2. WAL is synced to disk (synchronous=FULL)
-- 3. Only then does COMMIT return
COMMIT;  -- Durability guaranteed: change survives any crash after this point
```

**Key SQLite Durability Features**:
- **Atomic Commit**: All changes in WAL are applied atomically during checkpoint
- **Crash Recovery**: Uncompleted transactions automatically rolled back on next open
- **WAL Persistence**: WAL file preserved until checkpoint ensures data file consistency

**PostgreSQL Durability Architecture**:
```sql
-- PostgreSQL's multi-layered durability
BEGIN;
INSERT INTO financial_transactions (account_id, amount, type) 
VALUES (12345, -500.00, 'withdrawal');

-- Before COMMIT returns:
-- 1. Transaction record written to WAL buffer
-- 2. WAL buffer flushed to WAL file on disk
-- 3. fsync() ensures WAL is physically on disk
-- 4. COMMIT response sent to client
COMMIT;  -- Transaction is now durable
```

**PostgreSQL Durability Controls**:
```sql
-- Synchronous commit levels
SET synchronous_commit = on;        -- Wait for WAL flush (default)
SET synchronous_commit = off;       -- Async commit (faster, slight risk)
SET synchronous_commit = local;     -- Wait for local WAL flush only
SET synchronous_commit = remote_write; -- Wait for replica WAL write

-- Example of controlled durability
BEGIN;
-- Critical financial transaction
SET LOCAL synchronous_commit = on;  -- Ensure absolute durability
UPDATE accounts SET balance = balance - 1000 WHERE id = 123;
COMMIT;  -- Will not return until WAL is synced to disk
```

**Advanced Durability Scenarios**:

**Crash Recovery Example**:
```sql
-- Scenario: System crashes during transaction
BEGIN;
INSERT INTO log_entries (message, timestamp) VALUES ('Process started', now());
UPDATE counters SET value = value + 1 WHERE name = 'active_processes';
-- ** SYSTEM CRASH OCCURS HERE **

-- On restart, database automatically:
-- 1. Scans WAL for uncommitted transactions
-- 2. Rolls back the incomplete transaction
-- 3. Database returns to consistent state before transaction began
```

**Durability vs Performance Trade-offs**:
```sql
-- High durability (slower)
PRAGMA synchronous = FULL;     -- SQLite: Wait for complete disk sync
SET synchronous_commit = on;   -- PostgreSQL: Wait for WAL sync

-- Balanced approach (faster, still safe)
PRAGMA synchronous = NORMAL;   -- SQLite: Sync at critical moments
SET synchronous_commit = on;   -- PostgreSQL: Standard durability

-- Performance optimized (risk of data loss)
PRAGMA synchronous = OFF;      -- SQLite: Rely on OS for syncing
SET synchronous_commit = off;  -- PostgreSQL: Async commit
```

**Key Technical Insights**:
- **Storage Durability**: SSDs and modern storage controllers have internal caches that require explicit flush commands
- **Battery-Backed Controllers**: Hardware RAID controllers with battery backup can safely cache writes
- **Network Storage**: Durability guarantees must account for network partitions and storage system failures
- **Replication**: True durability in production often requires synchronous replication to multiple nodes

## **Performance and Trade-offs Analysis**

### **SQLite Performance Characteristics**:
```sql
-- Optimizations for SQLite
PRAGMA journal_mode = WAL;        -- Better concurrency
PRAGMA temp_store = MEMORY;       -- Faster temporary operations  
PRAGMA mmap_size = 268435456;     -- Memory mapping for large files
PRAGMA cache_size = -64000;       -- Larger cache

-- Batch operations for better performance
BEGIN;
-- Insert multiple records in single transaction
INSERT INTO logs (timestamp, level, message) VALUES 
    ('2024-01-15 10:00:00', 'INFO', 'System startup'),
    ('2024-01-15 10:01:00', 'INFO', 'Database connected');
COMMIT;
```

### **PostgreSQL Performance Tuning**:
```sql
-- Connection-level settings
SET work_mem = '256MB';               -- Sort/hash operations
SET maintenance_work_mem = '512MB';   -- VACUUM, CREATE INDEX
SET effective_cache_size = '4GB';     -- OS cache estimate
SET random_page_cost = 1.1;           -- SSD optimization

-- Advanced transaction control
BEGIN;
SET LOCAL enable_seqscan = false;     -- Force index usage
SET LOCAL statement_timeout = '30s';  -- Prevent long-running queries

-- Bulk operations with reduced durability
SET LOCAL synchronous_commit = off;   -- Async commit for performance
INSERT INTO staging_table SELECT * FROM large_source_table;
COMMIT;
```

## **Enterprise Considerations**

### **High Availability Scenarios**:

**SQLite Limitations**:
- Single point of failure (file-based)
- No built-in replication
- Manual backup strategies required

**PostgreSQL Capabilities**:
```sql
-- Streaming replication setup
-- On primary server
SELECT pg_create_physical_replication_slot('replica1');

-- On replica server  
-- primary_conninfo = 'host=primary port=5432 user=replicator'
-- primary_slot_name = 'replica1'

-- Monitoring replication lag
SELECT client_addr, 
       state, 
       pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### **ACID Compliance in Distributed Systems**:

**PostgreSQL with Foreign Data Wrappers**:
```sql
-- Two-phase commit for distributed transactions
BEGIN;
-- Prepare transaction for commit
PREPARE TRANSACTION 'distributed_txn_001';

-- On coordinator
COMMIT PREPARED 'distributed_txn_001';
-- Or if any participant fails
ROLLBACK PREPARED 'distributed_txn_001';
```

## **Best Practices Summary**

### **SQLite Best Practices**:
- Enable WAL mode for better concurrency
- Use IMMEDIATE transactions for write operations
- Implement application-level backup strategies
- Monitor database file size and vacuum regularly
- Use connection pooling in multi-threaded applications

### **PostgreSQL Best Practices**:
- Choose appropriate isolation levels based on application needs
- Monitor and tune checkpoint and WAL settings
- Implement comprehensive backup and PITR strategies
- Use connection pooling (pgBouncer, pgPool-II)
- Regular VACUUM and ANALYZE operations
- Monitor replication lag in high-availability setups

## **When to Choose Each Database**

### **Choose SQLite When**:
- Embedded applications or mobile apps
- Single-user or low-concurrency scenarios
- Simple deployment requirements
- Development and testing environments
- Applications requiring zero-configuration databases

### **Choose PostgreSQL When**:
- Multi-user web applications
- High concurrency requirements
- Complex querying and analytical workloads
- Enterprise applications requiring high availability
- Applications needing advanced SQL features and extensibility

Both SQLite and PostgreSQL provide full ACID compliance, but with different architectural approaches optimized for their respective use cases. SQLite excels in embedded scenarios with its simplicity and reliability, while PostgreSQL provides enterprise-grade features for complex, concurrent environments.

---

*This guide provides a comprehensive technical overview of ACID properties with practical comparisons between SQLite and PostgreSQL implementations. Understanding these concepts is crucial for designing robust database systems that maintain data integrity under various operational conditions.*