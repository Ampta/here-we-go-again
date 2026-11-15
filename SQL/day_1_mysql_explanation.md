# 🚀 Day 1 Masterclass: MySQL Fundamentals, Table Architecture & Production Operations

---

## Part 1: Detailed Breakdown of Your Class Notes

### 1. Database & Table Inspection Basics
```sql
SHOW DATABASES;
USE test;
SHOW TABLES;
DESC product; -- or DESCRIBE product;
```
* **What happens under the hood:** 
  MySQL queries the system catalog (`information_schema`).
* **3-Year Dev Insight:** In production environments with thousands of tables or read-replicas under heavy load, running commands like `SHOW TABLES` or scanning `information_schema` without indexes can cause temporary metadata locks. Always inspect schema structures on non-production or staging environments.

---

### 2. Table Creation & Data Type Selection (`CREATE TABLE`)

#### Class Notes Code:
```sql
CREATE TABLE employee ( 
    id INT(4) PRIMARY KEY, 
    emp_name VARCHAR(20), 
    emp_salary DECIMAL(20)
);
```

#### Technical Breakdown & Real-World Gotchas:

1. **`INT(4)` vs `INT` (Display Width vs Storage)**
   * **Junior Myth:** `INT(4)` limits the integer value to 4 digits (e.g., 9999).
   * **3-Year Dev Reality:** `INT` in MySQL *always* uses **4 bytes (32 bits)** of storage, regardless of the number inside the parentheses. The `(4)` is merely a *display width hint* for `ZEROFILL` (which is deprecated in MySQL 8.0). An `INT(4)` can still store up to `2,147,483,647`.

2. **`DECIMAL(precision, scale)` for Financial Data**
   * In your notes: `DECIMAL(20)` implies `DECIMAL(20, 0)` (0 decimal places).
   * In production (e.g., e-commerce, banking, crypto), ALWAYS specify both precision and scale:
     ```sql
     emp_salary DECIMAL(12, 2) -- Max: 9,999,999,999.99
     ```
   * **Why not `FLOAT` or `DOUBLE`?** Floating-point numbers use binary representation where numbers like `0.1 + 0.2 = 0.30000000000000004`. In financial apps, floating-point errors cause missing cents. `DECIMAL` stores exact base-10 numbers.

---

### 3. Schema Alteration & Constraint Addition (`ALTER TABLE`)

#### Class Notes Code & Bug Correction:
In line 21–23 of your notes:
```sql
-- ⚠️ NOTE BUG: You tried to add a UNIQUE constraint to 'email' before adding the 'email' column!
ALTER TABLE employee ADD CONSTRAINT email UNIQUE (email);
```
#### Production-Corrected Code:
```sql
-- Step 1: Add column first
ALTER TABLE employee ADD COLUMN email VARCHAR(100);

-- Step 2: Add unique constraint
ALTER TABLE employee ADD CONSTRAINT uq_employee_email UNIQUE (email);

-- Or in a single statement:
ALTER TABLE employee ADD COLUMN email VARCHAR(100) UNIQUE;
```

#### Altering Column Data Types & Names:
```sql
-- Add new column
ALTER TABLE product ADD COLUMN category VARCHAR(50);

-- Rename column (MySQL 8.0+)
ALTER TABLE product RENAME COLUMN product_price TO selling_price;

-- Modify column data type/scale
ALTER TABLE product MODIFY COLUMN selling_price DECIMAL(10, 5);
```

---

### 4. DDL Operations in High-Traffic Production Environments

Your notes ask: **"We have an account table where ID is INT. Now we have more users and want to change to BIGINT. What risks exist?"**

#### The 4 Critical Production Risks:

| Risk Factor | Production Impact | 3-Year Developer Solution |
| :--- | :--- | :--- |
| **1. Exclusive Metadata Locks (Table Locking)** | `ALTER TABLE` locks the table. All `INSERT`, `UPDATE`, and `DELETE` requests from your web app queue up, pool connections freeze, and your API returns HTTP 504 Timeouts. | Use **Online Schema Change** tools like `gh-ost` (GitHub's Online Schema Change tool) or `pt-online-schema-change` instead of raw `ALTER TABLE`. |
| **2. Index Rebuild Overhead** | Primary Keys are B-Tree indexes. Changing `INT` (4 bytes) to `BIGINT` (8 bytes) forces MySQL to rewrite every data page and secondary index on disk. | Perform migrations off-peak or use blue/green deployment strategy. |
| **3. Application / ORM Incompatibility** | Java/C# ORMs mapped `id` to 32-bit `Integer`. When MySQL starts returning 64-bit numbers, your backend throws `OverflowException` or `ClassCastException`. | Update application code & DTOs to 64-bit `Long` *before* executing the DB migration. |
| **4. Disk & Memory Bloat** | Every row's primary key and every foreign key referencing it increases by 4 bytes. In a table of 500 million rows, this adds gigabytes to RAM (`buffer_pool`) usage. | Monitor InnoDB Buffer Pool size before migrating. |

---

### 5. `DELETE` vs `TRUNCATE` vs `DROP`

Your notes highlight this comparison. Here is the deep architectural breakdown:

```
        DELETE FROM info;            vs            TRUNCATE TABLE info;
 ┌──────────────────────────────┐          ┌──────────────────────────────┐
 │ • Scans & deletes row-by-row │          │ • Deallocates data pages     │
 │ • Writes Undo/Redo Logs      │          │ • Drops & recreates metadata │
 │ • Triggers execution         │          │ • Triggers DO NOT fire       │
 │ • Keeps AUTO_INCREMENT value │          │ • Resets AUTO_INCREMENT to 1 │
 │ • Can use WHERE clause       │          │ • CANNOT use WHERE clause    │
 └──────────────────────────────┘          └──────────────────────────────┘
```

#### SQL Safe Updates Mode:
```sql
SET sql_safe_updates = 0; -- Allows UPDATE/DELETE without WHERE clause on key columns
```
* **Production Warning:** Safe updates mode is enabled in SQL clients (like MySQL Workbench) to prevent accidental wipes. **Never disable this in production scripts!**

---

### 6. Audit Trail Columns (`createdAt` & `updatedAt`)

#### Class Notes Code:
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    updatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP() ON UPDATE CURRENT_TIMESTAMP()
);
```

#### Why Every Enterprise Table MUST Have These Columns:
1. **Audit & Incident Analysis:** When a customer complains "my order status changed unexpectedly," `updatedAt` narrows down which microservice update job modified the row.
2. **ETL / Data Warehousing Sync:** Data engineering pipelines (e.g., Airflow, Debezium) query `SELECT * FROM users WHERE updatedAt >= NOW() - INTERVAL 1 HOUR` to incrementally sync fresh data into Snowflake/BigQuery.
3. **Optimistic Locking:** Prevents race conditions where two simultaneous API requests overwrite each other's edits.
4. **Legal & Compliance:** Standards like GDPR, SOC2, and HIPAA mandate timestamp tracking for data modifications.

---

### 7. Binary Logs (Binlog) & Disaster Recovery

```sql
SHOW VARIABLES LIKE 'binlog_format';
SET GLOBAL binlog_format = 'Statement'; -- (Note: Modern MySQL defaults to 'ROW')
SHOW BINARY LOGS;
SHOW BINLOG EVENTS IN 'SS-bin.000002';
```

#### What is the Binary Log?
The **Binlog** is an append-only log file on disk that records all DDL and DML operations that modify database structure or content.

#### Real-World Disaster Recovery Scenario:
Imagine a developer accidentally executes `DROP TABLE product;` at **2:00 PM**.
1. **Restore Full Backup:** Restore last night's 12:00 AM backup. (Data is now caught up to midnight).
2. **Point-In-Time Recovery (PITR):** Replay binlog events from 12:00 AM up to **1:59:59 PM** (stopping right before the fatal `DROP` command at 2:00 PM).
3. **Result:** Zero data loss!

---

### 8. Soft Delete vs Hard Delete

#### Class Notes Logic:
```sql
-- Hard Delete (Permanently removes data from disk page)
DELETE FROM employee WHERE id = 3;

-- Soft Delete (Data stays; hidden via application filter)
ALTER TABLE employee ADD COLUMN is_deleted TINYINT(1) DEFAULT 0;
-- Or prefer timestamp:
ALTER TABLE employee ADD COLUMN deleted_at TIMESTAMP NULL DEFAULT NULL;
```

#### Soft Delete Real-World Tradeoffs:

| Aspect | Hard Delete | Soft Delete |
| :--- | :--- | :--- |
| **Data Recovery** | Impossible without binlog restoration. | Instant (`UPDATE users SET deleted_at = NULL`). |
| **Query Complexity** | Simple (`SELECT * FROM users`). | Must append `WHERE deleted_at IS NULL` to *every* single query. |
| **Unique Constraints** | Standard (`UNIQUE(email)` works easily). | Tricky! If user deletes `alice@test.com` and tries to register again, `UNIQUE(email)` throws an error. |
| **Performance** | Tables remain compact. | Index bloat over time as deleted rows still occupy B-Tree pages. |

---

### 9. Database Transactions & ACID (Banking Example)

#### Class Notes Code & Correction:
```sql
START TRANSACTION;

UPDATE accounts 
SET balance = balance + 10000 
WHERE account_number = 1;

UPDATE accounts 
SET balance = balance - 10000 
WHERE account_number = 2;

-- ⚠️ MISSING IN NOTES: You must explicitly COMMIT or ROLLBACK!
COMMIT;
```

#### Real-World Failure Scenario (Why Atomicity Matters):
If the server crashes between updating `account_number = 1` and `account_number = 2`:
* **Without Transaction:** Account 1 got +10,000, but Account 2 never lost 10,000. Money was printed out of thin air!
* **With Transaction:** On system restart, InnoDB inspects the **Undo Log** and automatically **rolls back** Account 1's update, restoring exact database consistency.

---

## Part 2: Missing Topics to Bridge to a 3-Year Developer Level

To truly level up, here are 5 critical database concepts missing from Day 1 notes that mid-level engineers use daily:

### 1. Storage Engine Architecture (InnoDB vs MyISAM)
* **MyISAM:** Legacy engine. No transactions, no foreign keys, uses **Table-Level Locking** (one write blocks all reads across the entire table).
* **InnoDB (MySQL Default):** Supports ACID transactions, row-level locking, and crash recovery. Always ensure your tables use `ENGINE=InnoDB`.

---

### 2. Primary Key Strategy: `AUTO_INCREMENT` vs `UUIDv7`
* **`AUTO_INCREMENT` Issues:**
  1. Vulnerable to enumeration attacks (e.g., hackers browsing `example.com/api/user/1`, `/user/2`).
  2. Difficult in distributed multi-region databases (ID collision between servers).
* **`UUIDv4` Issues:** Random string generated by app causes severe B-Tree index fragmentation.
* **Modern Solution (`UUIDv7`):** Time-ordered 128-bit UUIDs. Gives unique global IDs without fragmenting B-Tree indexes on disk.

---

### 3. Three-Valued Logic & `NULL` Handling Traps
In SQL, boolean evaluation has 3 states: `TRUE`, `FALSE`, and `UNKNOWN` (`NULL`).

```sql
-- ❌ MISTAKE: This returns ZERO rows even if balance is NULL!
SELECT * FROM accounts WHERE balance = NULL; 

-- ✅ CORRECT:
SELECT * FROM accounts WHERE balance IS NULL;
```
* **Senior Tip:** `NULL != NULL` evaluates to `UNKNOWN`, not `TRUE` or `FALSE`. Always define columns as `NOT NULL DEFAULT ...` unless missing data is an explicit domain concept.

---

### 4. Index Locality & Clustered Primary Keys
In InnoDB, tables are stored **as** a B-Tree clustered index organized by the Primary Key.
* Rows with primary keys 1, 2, 3 sit physically adjacent on disk pages.
* Selecting by Primary Key is an $O(\log N)$ direct page lookup.
* Secondary indexes store the Primary Key as their pointer, causing a two-step lookup (index lookup $\rightarrow$ primary key lookup) called **Index Bookmark Lookup**.

---

### 5. Foreign Key Cascades & Locking Overhead
```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);
```
* **Production Warning:** `ON DELETE CASCADE` can lock thousands of child rows during a parent deletion, leading to catastrophic database deadlocks. Mid-level developers often enforce referential integrity in application service layers or execute batch updates to delete child records cleanly.

---

## 💡 Summary Cheat Sheet for Day 1

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                              DAY 1 SUMMARY                             │
 ├──────────────────┬─────────────────────────────────────────────────────┤
 │ DDL Commands     │ CREATE, ALTER, DROP, TRUNCATE (Schema level)       │
 │ DML Commands     │ INSERT, UPDATE, DELETE, SELECT (Row data level)     │
 │ Data Types       │ Use BIGINT for PKs, DECIMAL(P,S) for money          │
 │ TRUNCATE vs      │ TRUNCATE = DDL page drop (Resets AUTO_INC, fast)   │
 │ DELETE           │ DELETE = DML row-by-row (Scans, slow, keeps counter)│
 │ Audit Fields     │ createdAt & updatedAt mandatory for production      │
 │ Safety           │ Always scope UPDATE/DELETE with WHERE clause        │
 │ Transactions     │ START TRANSACTION -> queries -> COMMIT / ROLLBACK   │
 └──────────────────┴─────────────────────────────────────────────────────┘
```
