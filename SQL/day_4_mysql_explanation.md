# 🚀 Day 4 Masterclass: Database Views, Temporary Tables, Advanced CTEs, Window Functions & Indexing Optimization

---

## Part 1: Detailed Breakdown of Your Class Notes

---

### 1. Database Views (`CREATE VIEW`, `ALTER VIEW`, Data Manipulation via Views)

#### Class Notes Code:
```sql
USE test;

SHOW TABLES;

SELECT * FROM test.change_view;
SELECT * FROM test.employee;

-- Updating underlying table data via View
UPDATE test.change_view
SET city = "mumbai"
WHERE name = "Alice smith";

-- Altering View Definition
ALTER VIEW change_view AS
SELECT id, name, department, city, salary
FROM employee
WHERE department = 'it';

ALTER VIEW change_view AS
SELECT id, name, department, city, salary, joining_date
FROM employee;

-- Inserting values through View
INSERT INTO change_view(id, name, department, city, salary, joining_date) VALUES
	(111, "Shivam Gupta", "IT", "Mumbai", 40000, "2026-02-23");

SELECT * FROM employee;
```

#### What Happens Under the Hood:
A **View** is a saved, named SQL `SELECT` query stored in the database dictionary as metadata. It acts as a "virtual table" without storing physical data pages of its own (unless materialized). 

When you query a view, MySQL uses one of two processing algorithms:
1. **`ALGORITHM = MERGE` (Default):** MySQL merges the outer query filters with the View's internal `SELECT` statement into a single optimized execution plan against the underlying base tables.
2. **`ALGORITHM = TEMPTABLE`:** MySQL executes the View's inner query, buffers the result into an in-memory temporary table, and then executes your outer query against that temporary table. (Read-only, non-updatable).

#### Updatable View Rules & Restrictions:
In your notes, executing `UPDATE change_view` and `INSERT INTO change_view` mutates rows in the underlying `employee` table. However, MySQL enforces strict rules for updatable views:

```
                      Is a View Updatable?
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
     ✅ YES (1-to-1 Mapping)              ❌ NO (Read-Only)
   • Simple SELECT on single table      • Contains GROUP BY or HAVING
   • Direct column projections          • Contains DISTINCT
   • Simple WHERE filters               • Contains AGGREGATE functions (SUM, AVG)
   • Primary Key included in INSERT     • Contains UNION / UNION ALL
                                        • Uses TEMPTABLE algorithm
```

#### ⚠️ Critical Security & Integrity Trap: `WITH CHECK OPTION`
Consider this scenario:
```sql
CREATE VIEW it_dept AS
SELECT id, name, department, salary FROM employee WHERE department = 'IT';

-- Dangerous Insert:
INSERT INTO it_dept (id, name, department, salary) 
VALUES (105, 'Rohan', 'HR', 55000); 
```
* **The Problem:** The insert succeeds, but Rohan belongs to `'HR'`. The row disappears from `it_dept` immediately after insertion!
* **The 3-Year Dev Solution:** Always append `WITH CHECK OPTION` to updatable views. MySQL will reject any `INSERT` or `UPDATE` that violates the View's `WHERE` clause:
```sql
ALTER VIEW change_view AS
SELECT id, name, department, city, salary
FROM employee
WHERE department = 'IT'
WITH CHECK OPTION; -- Throws SQL Error 1369 if department != 'IT'
```

---

### 2. Temporary Tables (`CREATE TEMPORARY TABLE`)

#### Class Notes Code:
```sql
-- Create Temporary Table from Query
CREATE TEMPORARY TABLE temp_emp AS 
SELECT id, name, city
FROM employee
WHERE department = 'IT';

SELECT * FROM temp_emp;

-- Recreating and Inserting into Temporary Table
DROP TABLE temp_emp;

CREATE TEMPORARY TABLE temp_emp AS 
SELECT id, name, city
FROM employee 
WHERE city = "Mumbai";

INSERT INTO temp_emp (id, name, city) VALUES 
	(112, "Vivek Waht!", "Mumbai");

UPDATE temp_emp 
SET city = "Bombay"
WHERE city = "Mumbai";

-- Batch Upsert / Syncing back to Production Table
INSERT INTO employee (id, name, city) 
SELECT id, name, city FROM temp_emp
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    city = VALUES(city);
```

#### Lifecycle, Scope & Session Isolation:
Temporary tables are isolated database structures created for a specific client session.

```
 Client Session A (App Server 1)        Client Session B (App Server 2)
┌────────────────────────────────┐     ┌────────────────────────────────┐
│ CREATE TEMPORARY TABLE temp... │     │ SELECT * FROM temp_emp;        │
│                                │     │                                │
│ ✅ Reads/Writes temp_emp       │     │ ❌ Error: Table 'test.temp_emp'│
│    Data buffered in session    │     │    does not exist              │
└────────────────────────────────┘     └────────────────────────────────┘
```

1. **Session Isolation:** `temp_emp` is visible ONLY to the database connection that executed `CREATE TEMPORARY TABLE`. Other concurrent web app threads cannot see or interfere with it.
2. **Auto-Destruction:** When the database connection closes or drops, MySQL automatically drops all temporary tables and frees memory pages.
3. **Name Shadowing:** A temporary table can share the exact name of a permanent production table (`CREATE TEMPORARY TABLE employee ...`). During that session, all queries to `employee` hit the temporary table, masking the physical table!

#### Enterprise UPSERT Pattern (`ON DUPLICATE KEY UPDATE`):
Your notes show a crucial enterprise ETL pattern:
1. Extract and transform high-volume raw data inside an isolated `TEMPORARY TABLE`.
2. Perform batch manipulation (`UPDATE temp_emp SET city = 'Bombay'`).
3. Atomically sync changes back into the primary production table using `INSERT ... ON DUPLICATE KEY UPDATE`. If `id` exists, it updates `name` and `city`; if not, it inserts a new row.

#### Architectural Comparison Matrix:
| Feature | View | Temporary Table | CTE (`WITH ...`) | InnoDB Table |
| :--- | :--- | :--- | :--- | :--- |
| **Data Storage** | None (Virtual query) | Memory / Temp disk | Statement-scoped memory | Permanent Disk (B-Tree) |
| **Scope** | Database-wide | Connection session | Single SQL statement | Database-wide |
| **Lifetime** | Permanent | Dies on disconnect | Dies after query | Permanent |
| **Indexing** | Uses base table indexes | Can add explicit indexes | Cannot index | Custom indexes |
| **Use Case** | Security & query abstraction | Multi-step ETL batch jobs | Complex query readability | Core application entities |

---

### 3. Bulk Operations, Schema Alterations & String Matching (`TRUNCATE`, `ALTER TABLE`, `LIKE`)

#### Class Notes Code:
```sql
SELECT * FROM users;

TRUNCATE TABLE users;

-- Multi-column Schema Alteration
ALTER TABLE users
ADD address VARCHAR(20),
ADD phone VARCHAR(15),
ADD age INT;

DESC users;

-- Bulk Data Insertion (50 rows)
INSERT INTO users (user_id, username, createdAt, updatedAt, address, phone, age) VALUES
(1, 'johndoe', NOW(), NOW(), '123 Main St', '555-0101', 28),
...
(50, 'amber_n', NOW(), NOW(), '104 Cypress St', '555-0150', 32);

-- Wildcard Searching
SELECT * FROM users
WHERE phone LIKE '%9';
```

#### 1. Multi-Column `ALTER TABLE` Performance Optimization:
Instead of running three separate DDL queries (`ALTER TABLE users ADD address...`, `ALTER TABLE users ADD phone...`), combining them into a single `ALTER TABLE` statement ensures MySQL rebuilds the underlying InnoDB data table **only once**, saving substantial disk I/O and lock duration.

#### 2. The Wildcard Indexing Trap (`LIKE '%9'` vs `LIKE '9%'`):
In your notes, `SELECT * FROM users WHERE phone LIKE '%9'` searches for phone numbers ending in `9`.

```
  LIKE '555%' (Leading Prefix)                LIKE '%9' (Leading Wildcard)
 ┌──────────────────────────────┐          ┌──────────────────────────────┐
 │ B-Tree Index Range Scan      │          │ FULL TABLE SCAN              │
 │ • Jumps directly to '555'    │          │ • Must read EVERY single     │
 │ • Traverses contiguous leaves│          │   row from disk              │
 │ • Execution: O(log N)        │          │ • Execution: O(N)            │
 └──────────────────────────────┘          └──────────────────────────────┘
```

* **3-Year Dev Rule:** Leading wildcards (`%string`) **cannot** use B-Tree indexes! In a table of 10,000,000 users, `LIKE '%9'` causes a massive disk-bound Full Table Scan.
* **Production Fixes:**
  - Use `FULLTEXT` indexing (`MATCH(phone) AGAINST('9')`).
  - Create a Generated Column with reversed string values (`REVERSE(phone)`) and build a B-Tree index on the reversed column to search `LIKE '9%'`.

---

### 4. Enterprise Schema Design (`CREATE TABLE products`)

#### Class Notes Code:
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- Efficient ID
    sku CHAR(12) NOT NULL UNIQUE,                -- Fixed-length SKU string
    title VARCHAR(255) NOT NULL,                 -- Variable-length text
    price DECIMAL(10,2) NOT NULL,                -- Exact currency storage
    stock_quantity INT UNSIGNED DEFAULT 0,       -- Positive whole numbers
    description MEDIUMTEXT,                      -- Long product copy
    metadata JSON,                               -- Varying product specs (NoSQL inside SQL)
    is_active TINYINT(1) DEFAULT 1,              -- Boolean toggle flag
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Masterclass Breakdown of Data Type Engineering:

1. **`INT UNSIGNED`:**
   * Standard signed `INT` ranges from `-2,147,483,648` to `+2,147,483,647`.
   * Since IDs and stock quantities can never be negative, `UNSIGNED` shifts the range to `0` to `4,294,967,295`, doubling your capacity for free without increasing memory footprint (4 bytes)!

2. **`CHAR(12)` vs `VARCHAR(255)`:**
   * **`CHAR(12)` (Fixed-Length):** Always allocates exactly 12 bytes on disk. Perfect for SKUs, UUIDs, airport codes, or SHA-256 hashes. Fast B-Tree indexing because record offsets are static.
   * **`VARCHAR(255)` (Variable-Length):** Stores 1 byte of length metadata + actual string bytes. Ideal for product titles where length varies from 5 to 200 characters.

3. **`JSON` Column Type (NoSQL within Relational DB):**
   * MySQL stores `JSON` in an optimized binary format for rapid key lookup without parsing strings.
   * **Indexing JSON Fields:** You cannot directly add a standard B-Tree index on a raw `JSON` column. You must use **Virtual Generated Columns**:
     ```sql
     -- Extracting brand from JSON metadata and indexing it
     ALTER TABLE products 
     ADD COLUMN brand VARCHAR(50) AS (metadata->>'$.brand') STORED,
     ADD INDEX idx_brand (brand);
     ```

4. **`TINYINT(1)` vs `BOOLEAN`:**
   * In MySQL, `BOOLEAN` is an alias for `TINYINT(1)`. It uses 1 byte of storage (0 = `FALSE`, 1 = `TRUE`).

---

### 5. Database Normalization & Denormalization

#### What is Normalization?
Normalization is the process of organizing database attributes to minimize data redundancy and prevent **Insert, Update, and Delete Anomalies**.

```
 Unnormalized Table (Data Redundancy)         3NF Normalized Architecture
┌──────────────────────────────────────┐     ┌──────────────┐    ┌──────────────┐
│ OrderID | CustomerName | ItemName    │ ──> │ Customers    │    │ Orders       │
│ 101     │ Alice        │ Chocolate   │     │ CustID | Name│    │ OrdID| CustID│
│ 102     │ Alice        │ Vanilla     │     └──────────────┘    └──────────────┘
└──────────────────────────────────────┘
```

#### Normal Forms Breakdown:
* **1NF (First Normal Form):** Atomic values per cell (no comma-separated tags `1,2,3`), unique row identification via Primary Key.
* **2NF (Second Normal Form):** Meets 1NF + Removes **Partial Dependencies** (all non-key columns depend on the *entire* primary key, relevant for composite keys).
* **3NF (Third Normal Form):** Meets 2NF + Removes **Transitive Dependencies** (non-key attributes must NOT depend on other non-key attributes). Example: Store `zip_code` in `users`, but store `city` and `state` in a separate `zip_codes` table.
* **BCNF (Boyce-Codd Normal Form):** Advanced 3NF handling edge cases where multiple overlapping candidate keys exist.

#### Denormalization (Strategic Enterprise Performance Engineering):
* **Why Denormalize?** High-scale e-commerce systems (like Amazon or Flipkart) handle 95% READ queries and 5% WRITE queries. Joining 8 normalized tables on every homepage request causes database bottlenecking.
* **Denormalization Strategy:** Pre-aggregate data!
  - Store `total_orders_count` directly on the `users` table.
  - Store `avg_rating` directly on the `products` table.
  - Update pre-aggregated counts asynchronously via background message queues (Kafka/RabbitMQ) or MySQL Triggers.

---

### 6. Common Table Expressions (CTE) vs Subqueries

#### Class Notes Code:
```sql
-- Finding 2nd Highest Salary via Subquery:
SELECT MAX(salary) AS second_highest
FROM employee
WHERE salary < (SELECT MAX(salary) FROM employee);

-- Top Salary per Department via CTE:
WITH rankedEmployees AS (
    SELECT name, salary, department, 
    ROW_NUMBER() OVER(PARTITION BY department ORDER BY salary DESC) AS dep
    FROM employee
)
SELECT name, salary, department	
FROM rankedEmployees
WHERE dep = 1;
```

#### Why CTEs Win in Modern Codebases:
1. **Readability:** CTEs write SQL logic top-to-bottom sequentially rather than nested inside-out subqueries.
2. **Reusability:** A single CTE defined in `WITH cte_name AS (...)` can be joined multiple times in the final `SELECT`.
3. **Recursive Execution:** CTEs support recursion (`WITH RECURSIVE`), essential for querying hierarchical data structures like org charts, category trees, and bill of materials (BOM).

---

### 7. Window Functions & Advanced Ranking (`ROW_NUMBER`, `RANK`, `DENSE_RANK`)

#### Class Notes Code:
```sql
-- RANK() skips ranks on ties (1, 2, 2, 4)
SELECT name, salary, RANK() OVER(ORDER BY salary DESC) AS ranking
FROM employee
ORDER BY salary DESC;

-- DENSE_RANK() does NOT skip ranks on ties (1, 2, 2, 3)
SELECT name, salary, DENSE_RANK() OVER(ORDER BY salary DESC) AS ranking
FROM employee
ORDER BY salary DESC;

-- Partition By (Highest paid salary department wise)
SELECT name, salary, department, 
	ROW_NUMBER() OVER(PARTITION BY department ORDER BY salary DESC) AS dep
FROM employee;
```

#### Visual Comparison Matrix for Ties:
Suppose three employees earn \$80,000:

| Employee | Department | Salary | `ROW_NUMBER()` | `RANK()` | `DENSE_RANK()` |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Shivam | IT | \$90,000 | 1 | 1 | 1 |
| Vivek | IT | \$80,000 | 2 | 2 | 2 |
| Alice | IT | \$80,000 | 3 | 2 | 2 |
| Brijesh | IT | \$70,000 | 4 | **4** (Skips 3) | **3** (No Skip) |

#### `GROUP BY` vs `WINDOW FUNCTIONS`:
```
        GROUP BY department                        WINDOW FUNCTION (PARTITION BY)
 ┌──────────────────────────────┐              ┌─────────────────────────────────────┐
 │ • Collapses 10 rows into 1   │              │ • Retains ALL 10 individual rows    │
 │ • Destroys non-aggregated    │              │ • Appends computed analytical column│
 │   detail columns             │              │   alongside individual row data     │
 └──────────────────────────────┘              └─────────────────────────────────────┘
```

---

### 8. Database Indexing & Query Optimization Mechanics

#### Class Notes Overview:
* An index is a specialized B-Tree data structure on disk that allows MySQL to locate rows in $O(\log N)$ time complexity instead of scanning all $N$ rows ($O(N)$).

```
                      B-Tree Index Structure
                             [ 50 ]                   <-- Root Page
                            /      \
                   [ 20 | 35 ]     [ 65 | 80 ]        <-- Branch Pages
                  /     |    \     /    |    \
              [10..] [25..] [40..] [60..] [75..] [90..] <-- Leaf Pages (Data Pointers)
```

#### Indexing Classifications:
1. **Clustered Index (InnoDB Primary Key):** The table *is* the index! Row data is stored directly in the leaf pages of the Primary Key B-Tree. (Only 1 Clustered Index per table).
2. **Non-Clustered / Secondary Index:** B-Tree built on secondary columns (e.g. `email`, `department`). Leaf nodes store the indexed column value + the **Clustered Primary Key pointer**.
3. **Index Types:**
   * **B-Tree (Default):** Supports `=`, `>`, `<`, `>=`, `<=`, `BETWEEN`, `LIKE 'prefix%'`.
   * **Hash Index (Memory Engine):** Extremely fast $O(1)$ exact match, but cannot perform range searches.
   * **Fulltext Index:** Built for natural language document search (`MATCH...AGAINST`).
   * **Spatial Index (R-Tree):** Designed for geographic coordinates (`POINT`, `POLYGON`).

---

## Part 2: Missing Topics to Bridge to a 3-Year Developer Level

---

### 1. Analyzing Execution Plans with `EXPLAIN ANALYZE`

To tune queries in production, mid-level developers never guess—they inspect execution plans using `EXPLAIN ANALYZE` (MySQL 8.0+):

```sql
EXPLAIN ANALYZE
SELECT e.name, e.salary, d.department_name
FROM employee e
JOIN department d ON e.department_id = d.id
WHERE e.salary > 50000;
```

#### Key `EXPLAIN` Columns to Monitor:
* `type`: Access type. **Goal:** Move from `ALL` (Full Table Scan) $\rightarrow$ `range` $\rightarrow$ `ref` $\rightarrow$ `eq_ref` / `const`.
* `possible_keys`: Indexes MySQL considered.
* `key`: The actual index chosen by MySQL Query Optimizer.
* `rows`: Estimated number of rows MySQL must inspect.
* `Extra`: 
  - `Using index`: **Best!** Covering index used (No Primary Key bookmark lookup required).
  - `Using where`: Filtered after reading table rows.
  - `Using temporary; Using filesort`: **Warning!** Disk/memory sorting overhead.

---

### 2. Composite Indexes & The Leftmost Prefix Rule

When creating multi-column indexes:
```sql
CREATE INDEX idx_dept_status_salary ON employee(department, status, salary);
```

#### The Leftmost Prefix Rule:
MySQL can ONLY use this composite index if your query filters match index columns starting from the **leftmost column**:

| Query Filter | Index Used? | Explanation |
| :--- | :--- | :--- |
| `WHERE department = 'IT'` | ✅ YES | Uses 1st column (`department`). |
| `WHERE department = 'IT' AND status = 'ACTIVE'` | ✅ YES | Uses 1st & 2nd columns. |
| `WHERE department = 'IT' AND salary > 50000` | ✅ YES | Uses 1st & 3rd columns. |
| `WHERE status = 'ACTIVE'` | ❌ NO | Skips leftmost column (`department`). Full table scan! |
| `WHERE salary > 50000` | ❌ NO | Skips leftmost column (`department`). Full table scan! |

---

### 3. Hidden Indexing Anti-Patterns (Implicit Conversion)

#### ⚠️ The Silent Index Invalidation Trap:
Suppose `phone` is defined as `VARCHAR(15)` and indexed (`idx_phone`).

```sql
-- ❌ CATASTROPHIC MISTAKE: Passing number instead of string!
SELECT * FROM users WHERE phone = 5550101;
```
* **Why it breaks:** MySQL implicitly converts every string in the column to a number (`CAST(phone AS UNSIGNED)`) to compare with `5550101`. Wrapping the indexed column in a type conversion function forces a **Full Table Scan**!
* **✅ Correct Query:**
```sql
SELECT * FROM users WHERE phone = '555-0101';
```

---

## Part 3: 💡 Summary Cheat Sheet for Day 4

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                              DAY 4 SUMMARY                             │
 ├──────────────────┬─────────────────────────────────────────────────────┤
 │ Database Views   │ Virtual tables (ALGORITHM=MERGE/TEMPTABLE). Updat-  │
 │                  │ able only with 1-to-1 row mappings. Use CHECK OPTION│
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Temporary Tables │ Session-isolated, auto-destroyed on disconnect.     │
 │                  │ Ideal for ETL batch cleansing & UPSERT syncs.       │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Schema Design    │ UNSIGNED for IDs, CHAR for fixed SKUs, DECIMAL for  │
 │                  │ money, JSON + Generated Virtual Columns for specs.  │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Wildcard Searching│ LIKE 'abc%' uses B-Tree index range scan (O(log N)).│
 │                  │ LIKE '%abc' forces Full Table Scan (O(N)).          │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Ranking Functions│ ROW_NUMBER() = 1,2,3,4 (Unique sequential)          │
 │                  │ RANK() = 1,2,2,4 (Skips ties)                       │
 │                  │ DENSE_RANK() = 1,2,2,3 (No skipping)                │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Indexing         │ Clustered (PK data pages) vs Secondary (PK pointer).│
 │                  │ Always obey Leftmost Prefix Rule on composite keys! │
 └──────────────────┴─────────────────────────────────────────────────────┘
```
