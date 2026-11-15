# 🚀 Day 2 Masterclass: User Security (DCL), Advanced Transactions, Filtering, Functions & Query Optimization

---

## Part 1: Comprehensive Explanation of Day 2 Notes

---

### 1. User Management & Data Control Language (DCL)

MySQL uses a combination of **Username** and **Host Specifier** (`'user'@'host'`) to authenticate users.

```sql
-- Create a user restricted to local machine socket only
CREATE USER 'app_admin'@'localhost' IDENTIFIED BY 'SecurePass123!';

-- Create a user accessible from ANY remote host ('%')
CREATE USER 'report_user'@'%' IDENTIFIED BY 'StrongReportPass!';

-- Create a user restricted to a specific production server IP
CREATE USER 'api_service'@'192.168.1.50' IDENTIFIED BY 'ApiSecretKey!';
```

#### Understanding Host Specifiers:
* `'user'@'localhost'`: User can **only** log in directly from the physical server host where MySQL is running (via Unix socket).
* `'user'@'%'`: User can log in from **any IP address** on the network/internet.
* `'user'@'192.168.1.%'`: User can log in from any server inside the `192.168.1.x` subnet.

#### Granting & Revoking Privileges:
```sql
-- Grant read-only access to a specific table
GRANT SELECT ON test.users TO 'report_user'@'%';

-- Grant read and write access
GRANT SELECT, INSERT, UPDATE ON test.users TO 'report_user'@'%';

-- Check active privileges for a user
SHOW GRANTS FOR 'report_user'@'%';

-- Remove specific permission
REVOKE INSERT ON test.users FROM 'report_user'@'%';

-- Drop user permanently
DROP USER 'report_user'@'%';
```

#### ⚠️ Security Danger: `WITH GRANT OPTION` & Superuser Access
```sql
-- DANGER: Gives full admin power AND allows user to grant privileges to others!
GRANT ALL PRIVILEGES ON *.* TO 'hola'@'%' WITH GRANT OPTION;
```
* **3-Year Dev Insight:** Principle of Least Privilege (PoLP). Application microservices should **never** connect as `root` or with `ALL PRIVILEGES`. If an SQL Injection vulnerability occurs, an attacker with `ALL PRIVILEGES` can drop databases, create rogue admin users, or read system files via `LOAD DATA INFILE`.

---

### 2. Role-Based Access Control (RBAC)

Instead of granting permissions directly to individual users, modern enterprise databases assign permissions to **Roles**, then assign roles to users.

```sql
-- Step 1: Create Roles
CREATE ROLE 'reporting_role';
CREATE ROLE 'developer_role';

-- Step 2: Grant Permissions to Roles
GRANT SELECT ON test.* TO 'reporting_role';
GRANT SELECT, INSERT, UPDATE, DELETE ON test.* TO 'developer_role';

-- Step 3: Assign Roles to Users
GRANT 'reporting_role' TO 'report_user'@'%';
GRANT 'developer_role' TO 'dev_alice'@'%';
```

#### ⚠️ The MySQL Role Activation Trap:
In MySQL 8.0, when you grant a role to a user, **the role is INACTIVE by default when they log in!**
To fix this globally so users automatically get their granted roles upon login:
```sql
-- Make all granted roles active upon user login
SET GLOBAL activate_all_roles_on_login = ON;

-- Or set default role for a specific user:
SET DEFAULT ROLE ALL TO 'report_user'@'%';
```

---

### 3. Advanced Transactions & Savepoints (`SAVEPOINT`)

A `SAVEPOINT` creates a marker within an active transaction. This allows you to perform a **partial rollback** to the savepoint without cancelling the entire transaction.

```sql
SET autocommit = 0; -- Turn off automatic commit for explicit control
START TRANSACTION;

INSERT INTO users (user_id, username) VALUES (6, 'second');
SAVEPOINT sp1; -- Bookmark 1

INSERT INTO users (user_id, username) VALUES (9, 'third');
SAVEPOINT sp2; -- Bookmark 2

INSERT INTO users (user_id, username) VALUES (10, 'fourth');
SAVEPOINT sp3; -- Bookmark 3

-- Rollback ONLY the work done after sp2 (removes 'fourth', keeps 'second' and 'third')
ROLLBACK TO SAVEPOINT sp2;

-- Finalize and save 'second' and 'third' to disk permanently
COMMIT;
```

#### 🛠️ Real-World E-Commerce Use Case for Savepoints:
During an order checkout:
1. Deduct stock from inventory.
2. Create order record (`SAVEPOINT payment_attempt`).
3. Attempt payment gateway charge.
4. **If payment fails:** `ROLLBACK TO SAVEPOINT payment_attempt` (keeps order draft, lets user retry payment) instead of rolling back the entire checkout session!

---

### 4. Database Core Fundamentals & ACID Architecture

#### What is ACID?

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                            ACID PROPERTIES                              │
 ├────────────────┬────────────────────────────────────────────────────────┤
 │ Atomicity      │ All statements in a transaction succeed, or ALL fail. │
 │                │ (Handled by InnoDB Undo Logs)                          │
 ├────────────────┼────────────────────────────────────────────────────────┤
 │ Consistency    │ Database moves from one valid state to another valid   │
 │                │ state, satisfying all constraints & foreign keys.      │
 ├────────────────┼────────────────────────────────────────────────────────┤
 │ Isolation      │ Concurrent transactions cannot see each other's        │
 │                │ uncommitted changes (handled by MVCC & Locks).         │
 ├────────────────┼────────────────────────────────────────────────────────┤
 │ Durability     │ Once COMMITTED, data survives power loss / crashes.    │
 │                │ (Handled by InnoDB Redo Logs / WAL).                   │
 └────────────────┴────────────────────────────────────────────────────────┘
```

#### DBMS vs RDBMS:
* **DBMS (Database Management System):** File-based or key-value storage (e.g., MS Access, XML files). Data is flat, lacks relational constraints, and typically lacks row-level locking.
* **RDBMS (Relational Database Management System):** Tabular data structure with Primary Keys, Foreign Keys, ACID compliance, and SQL standard support (e.g., MySQL, PostgreSQL, Oracle).

---

### 5. Filtering & Subquery Optimization: `IN` vs `EXISTS`

Your notes highlight: **"In vs Exists: when to use when?"**

#### Technical Difference:

```sql
-- Query A: Using IN
SELECT * FROM employee 
WHERE department_id IN (SELECT id FROM department WHERE location = 'New York');

-- Query B: Using EXISTS
SELECT * FROM employee e 
WHERE EXISTS (
    SELECT 1 FROM department d 
    WHERE d.id = e.department_id AND d.location = 'New York'
);
```

| Feature | `IN` Keyword | `EXISTS` Keyword |
| :--- | :--- | :--- |
| **Execution Mechanics** | Evaluates the subquery first, builds a list of values in memory, then matches outer rows. | Evaluates as a boolean test for **existence**. Stops scanning as soon as the **first matching row** is found (short-circuit). |
| **Best Used When...** | The subquery result set is **small** (e.g., 5–50 items). | The outer table is large and matching against a **correlated subquery** with an indexed key. |
| **`NULL` Behavior Trap** | **DANGEROUS with NOT IN!** (See below). | Handles `NULL` cleanly. |

#### ⚠️ The Fatal `NOT IN` with `NULL` Trap:
If a subquery returns even a single `NULL` value, `NOT IN` returns **ZERO ROWS** for the entire query!

```sql
-- If subquery returns (1, 2, NULL):
SELECT * FROM users WHERE user_id NOT IN (1, 2, NULL);

-- Why? SQL expands this to:
-- user_id != 1 AND user_id != 2 AND user_id != NULL
-- Since (user_id != NULL) evaluates to UNKNOWN, the entire WHERE clause becomes UNKNOWN!
```
* **3-Year Dev Rule:** Always use `NOT EXISTS` or `LEFT JOIN ... WHERE right.id IS NULL` instead of `NOT IN` when subqueries might contain `NULL` values.

---

### 6. Pattern Matching (`LIKE`) & Index Locality (SARGability)

```sql
-- Case 1: Trailing Wildcard (SARGable -> Fast Index Range Scan)
SELECT * FROM employee WHERE name LIKE 'Alice%';

-- Case 2: Leading Wildcard (NON-SARGable -> Slow Full Table Scan)
SELECT * FROM employee WHERE name LIKE '%Smith';

-- Case 3: Single Character Wildcard ('_')
SELECT * FROM employee WHERE name LIKE 'A_ice'; -- Matches 'Alice', 'Abice'
```

#### What is SARGability? (Search Argument Able)
A query is **SARGable** if the database engine can utilize a B-Tree index to speed up execution.
* `LIKE 'Alice%'` allows MySQL to jump directly to the `'A'` section of the B-Tree index.
* `LIKE '%Smith'` forces MySQL to inspect **every single row on disk** (Full Table Scan) because the index is ordered alphabetically from left to right.

---

### 7. MySQL Built-in Functions Masterclass

#### A. String Functions:
```sql
-- UPPER() & LOWER()
SELECT UPPER(name), LOWER(city) FROM employee;

-- LENGTH (bytes) vs CHAR_LENGTH (character count in UTF-8 multi-byte strings)
SELECT LENGTH('Café'), CHAR_LENGTH('Café'); -- LENGTH=5 (é is 2 bytes), CHAR_LENGTH=4

-- CONCAT() & CONCAT_WS() (Concatenate With Separator)
SELECT CONCAT(name, ' (', city, ')') FROM employee;
SELECT CONCAT_WS(', ', name, department, city) FROM employee; -- 'Alice Smith, IT, New York'

-- SUBSTRING(string, start_position, length)
SELECT SUBSTRING('ENGINEERING', 1, 6); -- Returns 'ENGINI'

-- TRIM() & REPLACE()
SELECT TRIM('  hello  '); -- 'hello'
SELECT REPLACE('product_old_v1', 'old', 'new'); -- 'product_new_v1'
```

#### B. Numeric & Aggregate Functions:
```sql
SELECT MAX(salary) AS highest_sal, MIN(salary) AS lowest_sal FROM employee;
SELECT SUM(salary) AS total_payroll, AVG(salary) AS avg_sal FROM employee;

-- ROUND(value, decimals), FLOOR(), CEIL()
SELECT ROUND(1234.567, 2); -- 1234.57
SELECT FLOOR(1234.99);     -- 1234
SELECT CEIL(1234.01);      -- 1235
```

#### C. Date & Time Functions:
```sql
-- Current Date & Timestamp
SELECT CURRENT_DATE(), CURRENT_TIMESTAMP();

-- Date Arithmetic using INTERVAL
SELECT CURRENT_DATE() + INTERVAL 7 DAY;
SELECT DATE_ADD(CURRENT_DATE(), INTERVAL 1 MONTH);
SELECT DATE_SUB(CURRENT_DATE(), INTERVAL 2 YEAR);

-- DATEDIFF(end_date, start_date) -> returns difference in DAYS
SELECT DATEDIFF('2026-12-31', '2026-01-01'); -- 364 days

-- TIMESTAMPDIFF(unit, start_datetime, end_datetime) -> Unit can be YEAR, MONTH, DAY, HOUR
SELECT TIMESTAMPDIFF(YEAR, '1995-05-15', CURRENT_DATE()) AS age_years;
SELECT TIMESTAMPDIFF(MONTH, joining_date, CURRENT_DATE()) AS tenure_months FROM employee;

-- DATE_FORMAT(date, format_string)
SELECT DATE_FORMAT(joining_date, '%d-%b-%Y') FROM employee; -- '15-Mar-2022'
```

---

### 8. Result Ordering & Pagination (`ORDER BY`, `LIMIT`, `OFFSET`)

```sql
-- Basic sorting and pagination
SELECT id, name, salary 
FROM employee 
ORDER BY salary DESC 
LIMIT 5 OFFSET 0; -- Top 5 highest paid employees (Page 1)

SELECT id, name, salary 
FROM employee 
ORDER BY salary DESC 
LIMIT 5 OFFSET 5; -- Next 5 employees (Page 2)
```

---

## Part 2: Missing Topics to Bridge to a 3-Year Developer Level

Here are 5 critical concepts missing from Day 2 notes that mid-level engineers master to write production-grade queries:

---

### 1. Transaction Isolation Levels & Concurrency Bugs

MySQL InnoDB supports 4 isolation levels to control concurrency vs consistency tradeoffs:

| Isolation Level | Dirty Read Risk | Non-Repeatable Read Risk | Phantom Read Risk | Performance |
| :--- | :--- | :--- | :--- | :--- |
| **Read Uncommitted** | ⚠️ Yes | ⚠️ Yes | ⚠️ Yes | Fastest |
| **Read Committed** | 🛡️ Prevented | ⚠️ Yes | ⚠️ Yes | High (PostgreSQL Default) |
| **Repeatable Read** | 🛡️ Prevented | 🛡️ Prevented | 🛡️ Prevented (in InnoDB via Gap Locks) | **MySQL Default** |
| **Serializable** | 🛡️ Prevented | 🛡️ Prevented | 🛡️ Prevented | Slowest (Locks everything) |

#### Definitions:
* **Dirty Read:** Reading uncommitted data from another transaction that later gets rolled back.
* **Non-Repeatable Read:** Querying the same row twice inside one transaction and getting **different values** because another transaction updated and committed it.
* **Phantom Read:** Querying a range of rows twice and finding **new rows inserted** by another committed transaction.

```sql
-- Check current session isolation level
SELECT @@transaction_isolation;

-- Change isolation level for session
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

### 2. The High-Offset Pagination Performance Trap & Keyset Seek Solution

#### The Problem with `LIMIT 10 OFFSET 1000000`:
When you execute `LIMIT 10 OFFSET 1000000`:
* InnoDB **reads 1,000,010 rows off disk**, sorts them, throws away the first 1,000,000 rows, and returns only 10 rows!
* Page 1 takes 2ms. Page 10,000 takes **8 seconds!**

#### The 3-Year Dev Solution: Keyset / Seek Pagination
Instead of `OFFSET`, filter by the last seen Primary Key:

```sql
-- Page 1 (Fetch first 10 items)
SELECT id, name, salary 
FROM employee 
ORDER BY id ASC 
LIMIT 10;

-- Page 2 (Pass last seen ID from Page 1, e.g. last_id = 10)
SELECT id, name, salary 
FROM employee 
WHERE id > 10 
ORDER BY id ASC 
LIMIT 10;
```
* **Performance:** $O(\log N)$ instant B-Tree index jump regardless of whether you are on Page 1 or Page 1,000,000!

---

### 3. Function-Wrapped Columns Invalidate Indexes

```sql
-- ❌ BAD (Non-SARGable): Wrapping column in YEAR() function forces FULL TABLE SCAN!
SELECT * FROM employee WHERE YEAR(joining_date) = 2023;

-- ✅ GOOD (SARGable): Range condition uses B-Tree Index!
SELECT * FROM employee 
WHERE joining_date >= '2023-01-01' AND joining_date < '2024-01-01';
```

---

### 4. Grouping & Aggregations (`GROUP BY` vs `HAVING`)

```sql
-- Find departments with an AVERAGE salary greater than 70,000
SELECT department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employee
WHERE city != 'New York' -- WHERE filters individual rows BEFORE grouping
GROUP BY department
HAVING AVG(salary) > 70000; -- HAVING filters aggregated group results AFTER grouping
```
* **Rule:** Use `WHERE` to filter individual rows before grouping. Use `HAVING` to filter aggregated metrics calculated by `GROUP BY`.

---

### 5. Subquery Semi-Join Optimizations in MySQL 8.0

In modern MySQL (8.0+), the MySQL Query Optimizer rewrites `WHERE id IN (SELECT ...)` queries internally into **Semi-Joins** or **FirstMatch** algorithms. You can inspect how MySQL plans to execute your query using `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM employee WHERE department IN (SELECT department FROM employee WHERE salary > 80000);
```
Look for `type: range` or `type: ref` in the output instead of `type: ALL` (which indicates a full table scan).

---

## 💡 Summary Cheat Sheet for Day 2

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                              DAY 2 SUMMARY                             │
 ├──────────────────┬─────────────────────────────────────────────────────┤
 │ DCL Commands     │ CREATE USER, GRANT, REVOKE, DROP USER, RBAC Roles   │
 │ Role Trap        │ Must activate roles via activate_all_roles_on_login │
 │ Savepoints       │ SAVEPOINT sp1 -> ROLLBACK TO SAVEPOINT sp1          │
 │ ACID             │ Atomicity, Consistency, Isolation, Durability       │
 │ IN vs EXISTS     │ IN = small subqueries; EXISTS = correlated/short-ckt│
 │ NOT IN + NULL    │ DANGER! Single NULL in subquery breaks NOT IN       │
 │ SARGability      │ LIKE 'abc%' uses index; LIKE '%abc' forces table scan│
 │ Pagination       │ OFFSET is slow for deep pages; use Keyset (WHERE id)│
 └──────────────────┴─────────────────────────────────────────────────────┘
```
