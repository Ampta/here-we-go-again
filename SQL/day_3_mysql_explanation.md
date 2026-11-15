# 🚀 Day 3 Masterclass: Relational Joins, Subqueries, CTEs, Views, Temporary Tables & Window Functions

---

## Part 1: Detailed Breakdown of Day 3 Notes

---

### 1. The Hidden Cost of `ORDER BY` & Filesort Mechanics

Your notes state: **"Why ORDER BY is expensive if the dataset is huge: CPU, memory, temporary space."**

```sql
SELECT name, salary 
FROM employee
WHERE department = 'IT'
ORDER BY salary DESC 
LIMIT 10;
```

#### What Happens Under the Hood (MySQL Filesort):
1. **Index Lookup:** If there is an index on `(department, salary)`, MySQL uses the index order directly—**Zero Filesort required!**
2. **Filesort in RAM (`sort_buffer_size`):** If there is no index, MySQL scans all matching rows into memory (`sort_buffer_size`, default 256KB) and sorts them using QuickSort.
3. **Disk Spilling:** If the dataset exceeds `sort_buffer_size`, MySQL breaks the data into chunks, sorts them on disk in temporary files, and performs a multi-pass **External Merge Sort**.
4. **CPU Spikes:** External merge sorts consume massive CPU cycles and disk I/O.

* **3-Year Dev Insight:** To eliminate `ORDER BY` overhead on high-traffic endpoints, create a **Composite Index** matching your filtering and sorting columns:
  ```sql
  CREATE INDEX idx_dept_salary ON employee(department, salary DESC);
  ```

---

### 2. Relational Joins Masterclass

#### Class Notes Tables:
```sql
CREATE TABLE customers (
    cus_id INT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE orders (
    or_id INT PRIMARY KEY,
    cus_id INT,
    price DECIMAL(10, 2),
    name VARCHAR(50),
    FOREIGN KEY (cus_id) REFERENCES customers(cus_id)
);
```

#### ANSI JOIN Types Visualized:

```
      INNER JOIN                      LEFT JOIN                    RIGHT JOIN
    ┌───┐    ┌───┐                  ┌───┐    ┌───┐                ┌───┐    ┌───┐
   │    │████│   │                 │████│████│   │               │   │████│████│
   │    │████│   │                 │████│████│   │               │   │████│████│
    └───┘    └───┘                  └───┘    └───┘                └───┘    └───┘
  Matching rows in BOTH           ALL Left rows +                ALL Right rows +
       tables                        matching Right                matching Left
```

#### A. Implicit Join vs Explicit ANSI Join:
```sql
-- ❌ IMPLICIT JOIN (Legacy 1980s Syntax - Hard to read, prone to accidental Cartesian Products)
SELECT c.name, o.or_id, o.price
FROM customers c, orders o
WHERE c.cus_id = o.cus_id;

-- ✅ EXPLICIT ANSI INNER JOIN (Modern Production Standard)
SELECT c.name, o.or_id, o.price
FROM customers c
INNER JOIN orders o ON c.cus_id = o.cus_id;
```

#### B. Anti-Join Pattern (Finding Un-ordered Customers):
How do you find customers who have **never placed an order**?
```sql
-- LEFT JOIN Anti-Join Pattern
SELECT c.cus_id, c.name 
FROM customers c
LEFT JOIN orders o ON c.cus_id = o.cus_id
WHERE o.cus_id IS NULL; -- Filters out any customer that matched an order!
```
* **Performance Note:** In MySQL 8.0, the `LEFT JOIN ... WHERE right.id IS NULL` pattern is optimized identically to `NOT EXISTS`.

---

### 3. Subquery Types & MySQL Optimizer Myths

Your notes highlight: **"Many devs think JOIN is faster and subquery is slow, which is not true. MySQL optimizer transforms IN, EXISTS, subqueries into optimized joins internally."**

#### Subquery Placement Types:

```sql
-- 1. Scalar Subquery in SELECT clause (Evaluates single value per row)
SELECT name, salary, (SELECT AVG(salary) FROM employee) AS company_avg
FROM employee;

-- 2. Subquery in WHERE clause (Filtering based on calculation)
SELECT name, salary
FROM employee
WHERE salary > (SELECT AVG(salary) FROM employee WHERE department = 'IT');

-- 3. Derived Table / Subquery in FROM clause (Must have an alias!)
SELECT dept_summary.department, dept_summary.avg_sal
FROM (
    SELECT department, AVG(salary) AS avg_sal 
    FROM employee 
    GROUP BY department
) AS dept_summary
WHERE dept_summary.avg_sal > 60000;
```

#### 🛡️ Modern Semi-Join Transformations (MySQL 8.0):
Historically (MySQL 5.6 and earlier), subqueries like `WHERE id IN (SELECT ...)` executed as a nested loop for *every outer row*. 
In **MySQL 8.0**, the query optimizer automatically rewrites subqueries into **Semi-Joins**, **Materialized Subquery Tables**, or **FirstMatch** join strategies.

---

### 4. Common Table Expressions (CTEs - `WITH` Clause)

A **CTE** creates a named temporary result set that exists only within the scope of a single query statement. It replaces hard-to-read nested subqueries.

#### Class Notes Problem rewritten with CTE:
```sql
-- Clean, readable query using WITH clause
WITH IT_Department_Stats AS (
    SELECT AVG(salary) AS avg_it_salary
    FROM employee
    WHERE department = 'IT'
)
SELECT e.name, e.salary, s.avg_it_salary
FROM employee e
CROSS JOIN IT_Department_Stats s
WHERE e.salary > s.avg_it_salary;
```

---

### 5. MySQL Database Views (`CREATE VIEW`)

A **View** is a saved SQL query definition in the database metadata that acts as a virtual table.

```sql
-- Create a view for IT Department Employees
CREATE VIEW v_it_employees AS 
SELECT id, name, city, joining_date 
FROM employee
WHERE department = 'IT';

-- Query the view like a normal table
SELECT * FROM v_it_employees WHERE city = 'New York';
```

#### 3 Enterprise Use Cases for Views:
1. **Security & Column Masking:** Expose `v_employee_public` (hiding sensitive `salary` or `password` columns) to analytics teams.
2. **Query Simplification:** Hide complex 5-table `JOIN` statements behind a single clean view `v_customer_order_summary`.
3. **Backward Compatibility:** When refactoring database schema (renaming tables), keep old view names active so legacy application APIs don't crash.

---

### 6. Temporary Tables (`CREATE TEMPORARY TABLE`)

A **Temporary Table** is a session-private table visible ONLY to the current database connection. It is automatically destroyed when the connection closes.

```sql
-- Create a session-scoped temporary table
CREATE TEMPORARY TABLE temp_high_value_customers (
    customer_id INT PRIMARY KEY,
    total_spent DECIMAL(10, 2)
);

-- Populate temporary table
INSERT INTO temp_high_value_customers (customer_id, total_spent)
SELECT cus_id, SUM(price)
FROM orders
GROUP BY cus_id
HAVING SUM(price) > 500.00;

-- Query temporary table in current session
SELECT * FROM temp_high_value_customers;

-- Clean up (or wait for connection disconnect)
DROP TEMPORARY TABLE IF EXISTS temp_high_value_customers;
```

#### Temporary Tables vs Regular Tables vs Views:

| Feature | Regular Table | Database View | Temporary Table |
| :--- | :--- | :--- | :--- |
| **Data Storage** | Permanent on disk. | No data stored (Query definition only). | Private data per session (RAM or Disk temp). |
| **Visibility** | All connections. | All connections. | Current connection ONLY. |
| **Lifetime** | Until `DROP TABLE`. | Until `DROP VIEW`. | Auto-dropped on connection close. |

---

## Part 2: Missing Topics to Bridge to a 3-Year Developer Level

Here are 5 critical database topics missing from Day 3 notes that mid-level backend engineers use daily:

---

### 1. SQL Window Functions (`OVER (PARTITION BY ...)`)

Unlike `GROUP BY` (which collapses multiple rows into a single summary row), **Window Functions** compute aggregate values across a set of rows while **preserving individual row detail**!

```sql
SELECT 
    name, 
    department, 
    salary,
    -- Rank employees by salary WITHIN their department
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_salary_rank,
    
    -- Calculate department average next to individual salary
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM employee;
```

#### Essential Window Functions:
* `ROW_NUMBER()`: Assigns unique sequential integers (1, 2, 3...).
* `RANK()`: Ranks items (1, 2, 2, 4 — leaves gaps on ties).
* `DENSE_RANK()`: Ranks items (1, 2, 2, 3 — no gaps on ties).
* `LAG(col, offset)`: Accesses data from a **previous row** in the result set (great for calculating Month-over-Month growth!).
* `LEAD(col, offset)`: Accesses data from a **subsequent row**.

---

### 2. Recursive CTEs (Hierarchical Data Trees)

How do you query multi-level category trees (e.g. `Cakes` $\rightarrow$ `Custom Tiered` $\rightarrow$ `Wedding Tiered`) or employee manager hierarchies?

```sql
WITH RECURSIVE CategoryHierarchy AS (
    -- Anchor member: Select root category (parent_id IS NULL)
    SELECT category_id, category_name, parent_category_id, 1 AS depth
    FROM categories
    WHERE parent_category_id IS NULL
    
    UNION ALL
    
    -- Recursive member: Join child categories to parent category
    SELECT c.category_id, c.category_name, c.parent_category_id, ch.depth + 1
    FROM categories c
    INNER JOIN CategoryHierarchy ch ON c.parent_category_id = ch.category_id
)
SELECT * FROM CategoryHierarchy ORDER BY depth, category_id;
```

---

### 3. Join Execution Algorithms Under the Hood (Hash Join vs Nested Loop)

How does MySQL physically join two tables in memory?

1. **Index Nested Loop Join (NLJ):** For each row in Table A, MySQL uses an index to look up matching rows in Table B. Very fast when Table B has an index!
2. **Hash Join (MySQL 8.0.18+):** If there are NO indexes on the join columns, MySQL builds an in-memory Hash Table of the smaller table, then streams the larger table through it. Replaces the slow Block Nested-Loop algorithm!

---

### 4. View Performance Traps (`MERGE` vs `TEMPTABLE` Algorithm)

When querying a view, MySQL uses one of two execution algorithms:
* **`ALGORITHM=MERGE` (Fast):** MySQL merges the view query directly into your main SQL statement before optimizing. Indexes on underlying tables are fully preserved!
* **`ALGORITHM=TEMPTABLE` (Slow):** MySQL executes the view query first, materializes the result set into a temporary table, and then queries the temporary table (loses index optimization).
* **Rule:** Avoid `GROUP BY`, `DISTINCT`, `LIMIT`, or Aggregate functions inside views if you want them to be mergeable!

---

### 5. On-Disk Temporary Table Bottlenecks (`tmp_table_size`)

When executing complex `GROUP BY`, `DISTINCT`, or `UNION` queries, MySQL allocates memory up to `tmp_table_size` / `max_heap_table_size`.
* If the intermediate result exceeds memory limits, MySQL converts the temporary table into an **InnoDB On-Disk Temporary Table**.
* **Symptom:** Query suddenly takes 15 seconds instead of 50ms.
* **Fix:** Optimize query with composite indexes or increase memory parameters in `my.cnf`.

---

## 💡 Summary Cheat Sheet for Day 3

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                              DAY 3 SUMMARY                             │
 ├──────────────────┬─────────────────────────────────────────────────────┤
 │ ORDER BY Cost    │ Causes Filesort; fix with Composite Indexes         │
 │ ANSI JOINs       │ Prefer explicit INNER/LEFT JOIN over implicit comma │
 │ Anti-Join        │ LEFT JOIN ... WHERE right_id IS NULL                │
 │ Optimizer        │ MySQL 8.0 transforms IN/EXISTS into Semi-Joins      │
 │ CTE (WITH)       │ Modular, readable temporary query expressions       │
 │ Views            │ Saved virtual query definitions for security & logic│
 │ Temp Tables      │ Session-private tables auto-dropped on disconnect   │
 │ Window Funcs     │ DENSE_RANK(), LAG(), LEAD() OVER (PARTITION BY...)  │
 └──────────────────┴─────────────────────────────────────────────────────┘
```
