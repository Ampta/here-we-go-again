# Section 5: SQL, CTEs, Window Functions & Database Optimization (Q115 – Q138)

---

### Q115: What is a CTE?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A Common Table Expression (CTE) is a temporary, named result set defined within the lexical execution scope of a single SQL statement (`SELECT`, `INSERT`, `UPDATE`, or `DELETE`) using the `WITH` clause.
* **Under the Hood:**
  * CTEs act as temporary virtual tables available only during the execution of that specific query.
  * In modern query optimizers (PostgreSQL 12+, MySQL 8+, Oracle), simple CTEs can be **inlined** directly into the query execution tree just like a subquery.
  * In PostgreSQL, you can use `WITH cte AS MATERIALIZED (...)` to force the database to materialize the CTE result into memory once and reuse it across multiple references, avoiding duplicate scans.
* **Advantages over Subqueries:** Dramatically improves readability, simplifies complex joins, supports self-referencing (Recursive CTEs), and can be referenced multiple times within the same query.

#### 2. Production Code Example
```sql
-- Finding top customers who spent more than the average order value
WITH CustomerOrderStats AS (
    SELECT 
        customer_id,
        COUNT(order_id) AS total_orders,
        SUM(total_amount) AS total_spend,
        AVG(total_amount) AS avg_order_amount
    FROM orders
    WHERE status = 'COMPLETED'
    GROUP BY customer_id
),
GlobalOrderStats AS (
    SELECT AVG(total_amount) AS overall_avg_spend
    FROM orders
    WHERE status = 'COMPLETED'
)
SELECT 
    c.customer_id,
    c.total_orders,
    c.total_spend
FROM CustomerOrderStats c
CROSS JOIN GlobalOrderStats g
WHERE c.avg_order_amount > g.overall_avg_spend
ORDER BY c.total_spend DESC;
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "A CTE (Common Table Expression) is a named temporary result set defined using the `WITH` keyword at the beginning of a query. It makes complex queries modular and easy to read by breaking them into logical steps, unlike deeply nested subqueries. They can be referenced multiple times and can also be recursive for hierarchical data."
* **In Our Project / Production Context:** "We use CTEs in our financial reporting queries to calculate intermediate tax deductions and promotional discounts before computing the final settlement amounts."
* **Pro-Tip & Interview Trap:** "Be aware of optimization differences: In older PostgreSQL versions (<12), CTEs were always materialized as optimization barriers, which sometimes prevented the planner from pushing down `WHERE` clauses. In modern databases, the optimizer inlines CTEs unless explicitly specified as `MATERIALIZED`."

---

### Q116: What is a Recursive CTE?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A CTE that references itself to iteratively traverse hierarchical or graph-structured data (e.g., organizational charts, category trees, bill-of-materials, breadcrumbs).
* **Anatomy of a Recursive CTE:**
  1. **Anchor Member:** The initial query that returns the base root result set (e.g., top-level CEO or root category).
  2. **`UNION ALL` Operator:** Joins the anchor query with the recursive member.
  3. **Recursive Member:** A query that joins the underlying table with the CTE itself to fetch the next hierarchical level.
  4. **Termination Condition:** The recursion stops automatically when the recursive member returns zero rows.

#### 2. Production Code Example
```sql
-- Traversing an Employee Management Hierarchy (Org Chart)
WITH RECURSIVE OrgHierarchy AS (
    -- 1. Anchor Member: Top-level CEO (manager_id is NULL)
    SELECT 
        employee_id, 
        name, 
        manager_id, 
        1 AS level,
        CAST(name AS VARCHAR(255)) AS management_chain
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- 2. Recursive Member: Join employees with previous hierarchy level
    SELECT 
        e.employee_id, 
        e.name, 
        e.manager_id, 
        o.level + 1,
        CAST(o.management_chain || ' -> ' || e.name AS VARCHAR(255))
    FROM employees e
    INNER JOIN OrgHierarchy o ON e.manager_id = o.employee_id
)
SELECT * FROM OrgHierarchy ORDER BY level, manager_id;
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "A Recursive CTE is a query that repeatedly references itself to traverse hierarchical or tree-structured data. It consists of an anchor member that seeds the initial root rows, followed by `UNION ALL`, and a recursive member that joins the CTE with the table until no more child rows are found."
* **In Our Project / Production Context:** "In our eCommerce catalog, categories have multi-level parent-child relationships (Electronics -> Computers -> Laptops). We use a recursive CTE to generate full category breadcrumb paths and fetch all descendant products."
* **Pro-Tip & Interview Trap:** "Infinite loop danger! If your data contains circular references (Employee A manages B, and B manages A), a recursive CTE will run infinitely until the database hits its recursion limit (e.g., `MAXRECURSION 100` in SQL Server). Always guard against cycles!"

---

### Q117: What is a Subquery?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A subquery (or inner query / nested query) is a `SELECT` statement embedded inside another outer SQL statement (`SELECT`, `INSERT`, `UPDATE`, or `DELETE`).
* **Types of Subqueries by Result Shape:**
  1. **Scalar Subquery:** Returns exactly **one row and one column** (a single value). Can be used anywhere an expression is expected (e.g., in `SELECT` or comparison operators `=, <, >`).
  2. **Row Subquery:** Returns a single row with multiple columns.
  3. **Table Subquery:** Returns multiple rows and multiple columns. Used in `FROM` clauses (inline views) or with `IN`, `EXISTS`, `ANY`, `ALL` operators.

#### 2. Production Code Example
```sql
-- 1. Scalar Subquery in SELECT clause
SELECT 
    name, 
    salary,
    (SELECT AVG(salary) FROM employees) AS company_avg_salary
FROM employees;

-- 2. Table Subquery in WHERE clause with IN
SELECT name, department_id 
FROM employees
WHERE department_id IN (
    SELECT department_id 
    FROM departments 
    WHERE location = 'New York'
);
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "A subquery is a nested query enclosed in parentheses inside an outer SQL statement. It can be a scalar subquery returning a single value, a row subquery, or a table subquery used inside `FROM` clauses or with operators like `IN` and `EXISTS`."
* **In Our Project / Production Context:** "We use scalar subqueries to append real-time aggregated metrics—like total lifetime orders—onto individual customer rows in reporting dashboards."
* **Pro-Tip & Interview Trap:** "If a scalar subquery returns more than one row at runtime, the query will crash with a runtime error: `Subquery returned more than 1 row`."

---

### Q118: What is the difference between correlated and non-correlated subqueries?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | Non-Correlated Subquery | Correlated Subquery |
| :--- | :--- | :--- |
| **Outer Query Dependency** | **Independent:** Does not reference any columns from the outer query. | **Dependent:** References one or more columns from the outer query table. |
| **Execution Frequency** | Executed **only once**; the result is reused for all outer rows. | Executed **repeatedly for every single row** evaluated by the outer query ($N$ times!). |
| **Performance Impact** | Fast ($O(M + N)$); easily cached or converted into a hash join. | Slower ($O(N 	imes M)$); can cause severe CPU bottlenecks on large tables. |
| **Standalone Execution** | Can be copied, pasted, and executed independently in SQL client. | **Cannot run independently** because it references outer table aliases. |

#### 2. Production Code Example
```sql
-- 1. Non-Correlated: Inner query runs ONCE
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees); -- Standalone query

-- 2. Correlated: Inner query runs once PER outer employee row!
SELECT e.name, e.department_id, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(sub.salary) 
    FROM employees sub 
    WHERE sub.department_id = e.department_id -- References outer query e.department_id!
);
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "A non-correlated subquery is independent of the outer query, executes only once, and can be run on its own. A correlated subquery references columns from the outer query and executes repeatedly for every row processed by the outer query, which can cause severe performance issues on large datasets. We often rewrite correlated subqueries using joins or window functions."
* **In Our Project / Production Context:** "We had a slow query finding employees with above-average salaries in their department that used a correlated subquery and took 8 seconds. We refactored it using `AVG(salary) OVER(PARTITION BY department_id)`, reducing execution time to 120ms."
* **Pro-Tip & Interview Trap:** "Whenever you see a correlated subquery in a code review, look for opportunities to rewrite it using a Window Function (`AVG() OVER(...)`) or an inner join with a CTE."

---

### Q119: What is ROW_NUMBER()?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A window function that assigns a unique, contiguous, sequential integer starting at 1 to each row within a designated partition of a query result set.
* **Under the Hood:**
  * Must be paired with an `OVER (ORDER BY ...)` clause.
  * Even if two rows have identical sorting values (ties), `ROW_NUMBER()` **never assigns duplicate numbers**—it arbitrates ties non-deterministically unless tie-breaker columns are specified.
  * Computed during the windowing phase of query execution (after `WHERE` and `GROUP BY`, but before `LIMIT`).

#### 2. Production Code Example
```sql
SELECT 
    employee_id,
    department_id,
    salary,
    -- Assigns a unique sequential number within each department ordered by salary descending
    ROW_NUMBER() OVER (
        PARTITION BY department_id 
        ORDER BY salary DESC
    ) AS row_num
FROM employees;
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`ROW_NUMBER()` is a window function that assigns a unique sequential integer (1, 2, 3...) to each row within a partition ordered by a specified column. It never produces ties or gaps in numbering, making it the industry standard for deduplication and pagination."
* **In Our Project / Production Context:** "We use `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY updated_at DESC)` to identify and delete duplicate user audit records while keeping the latest entry."
* **Pro-Tip & Interview Trap:** "Because `ROW_NUMBER()` is evaluated in the window phase, you cannot use it directly in a `WHERE` clause (`WHERE ROW_NUMBER() = 1` is invalid syntax). You must wrap it in a CTE or derived table!"

---

### Q120: What is RANK()?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A window function that assigns a rank to each row within a partition based on an `ORDER BY` clause, **allowing ties and leaving gaps in the ranking sequence**.
* **Tie Handling Mechanism:**
  * If two rows share the same value, they receive the **same rank**.
  * The next rank assigned will **skip** numbers equal to the number of tied rows.
  * Example: If two rows tie for rank 1, both receive rank 1, and the next row receives rank 3 (sequence: 1, 1, 3, 4).

#### 2. Production Code Example
```sql
SELECT 
    name,
    score,
    -- Rank with gaps on ties:
    RANK() OVER (ORDER BY score DESC) AS player_rank
FROM tournament_scores;
```
**Sample Output:**
| name | score | player_rank |
| :--- | :--- | :--- |
| Alice | 100 | **1** |
| Bob | 100 | **1** |
| Charlie | 90 | **3** (Rank 2 was skipped!) |
| David | 80 | **4** |

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`RANK()` is a window function that assigns ranks based on ordering. When values are tied, rows receive the identical rank, but it skips subsequent ranks, leaving gaps in the sequence (e.g., 1, 1, 3). It reflects traditional sports tournament rankings where tied positions skip next ranks."
* **In Our Project / Production Context:** "We use `RANK()` in our sales leaderboard service where co-winners for first place share the #1 rank, and the next seller is ranked #3."
* **Pro-Tip & Interview Trap:** "Contrast `RANK()` with `DENSE_RANK()`: `RANK()` leaves gaps after ties; `DENSE_RANK()` never leaves gaps."

---

### Q121: What is DENSE_RANK()?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A window function that assigns ranks to rows within a partition based on an `ORDER BY` clause, **allowing ties but NEVER leaving gaps in the ranking sequence**.
* **Tie Handling Mechanism:**
  * If two rows share identical values, they receive the **same rank**.
  * The very next distinct value receives the **immediate consecutive integer** without skipping.
  * Example: If two rows tie for rank 1, both get rank 1, and the next row gets rank 2 (sequence: 1, 1, 2, 3).
* **Ideal Use Case:** Finding the Nth-highest or Nth-lowest distinct values (e.g., finding the 2nd highest salary).

#### 2. Production Code Example
```sql
SELECT 
    name,
    salary,
    -- Dense rank: No gaps after ties!
    DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;
```
**Sample Output:**
| name | salary | salary_rank |
| :--- | :--- | :--- |
| Alice | $150,000 | **1** |
| Bob | $150,000 | **1** |
| Charlie | $120,000 | **2** (Immediate next integer!) |
| David | $100,000 | **3** |

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`DENSE_RANK()` is a window function that assigns ranks without gaps in the sequence. When rows have identical values, they receive the same rank, and the subsequent row receives the next consecutive integer (e.g., 1, 1, 2). It is the premier tool for finding the Nth-highest value in a table."
* **In Our Project / Production Context:** "Whenever interviewers ask for the 2nd or 3rd highest salary, `DENSE_RANK()` is the cleanest and most robust solution because it correctly handles ties without skipping ranks."
* **Pro-Tip & Interview Trap:** "If you used `RANK()` to find the 2nd highest salary and there was a tie for 1st place, `WHERE rank = 2` would return zero rows! `DENSE_RANK()` guarantees `rank = 2` exists."

---

### Q122: What is the difference between ROW_NUMBER(), RANK(), and DENSE_RANK()?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Window Function | Tie Handling | Leaves Gaps? | Output for Scores: [100, 100, 90, 80] |
| :--- | :--- | :--- | :--- |
| **`ROW_NUMBER()`** | No ties allowed; assigns distinct sequential integers. | **No gaps.** | **1, 2, 3, 4** |
| **`RANK()`** | Identical values get the **same rank**. | **Leaves gaps** equal to number of ties. | **1, 1, 3, 4** (Gap at 2) |
| **`DENSE_RANK()`** | Identical values get the **same rank**. | **No gaps;** consecutive integers. | **1, 1, 2, 3** |

#### 2. Production Code Example
```sql
SELECT 
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK()       OVER (ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rnk
FROM (VALUES 
    ('Alice', 10000),
    ('Bob', 10000),
    ('Charlie', 8000),
    ('David', 7000)
) AS Employees(name, salary);
```
**Result Matrix:**
| name | salary | row_num | rnk | dense_rnk |
| :--- | :--- | :--- | :--- | :--- |
| Alice | 10000 | 1 | 1 | 1 |
| Bob | 10000 | 2 | 1 | 1 |
| Charlie | 8000 | 3 | 3 | 2 |
| David | 7000 | 4 | 4 | 3 |

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`ROW_NUMBER()` assigns strictly unique integers with no ties (1, 2, 3, 4). `RANK()` assigns the same rank to ties and skips subsequent ranks, leaving gaps (1, 1, 3, 4). `DENSE_RANK()` assigns the same rank to ties but never skips ranks, producing a contiguous sequence (1, 1, 2, 3). In production, we use `ROW_NUMBER` for deduplication and `DENSE_RANK` for Nth-highest queries."
* **In Our Project / Production Context:** "This distinction is critical: in deduplicating customer profiles, we use `ROW_NUMBER` to pick exactly one row, whereas for leaderboard tiers, we use `DENSE_RANK`."
* **Pro-Tip & Interview Trap:** "Remember this quick memory trick: *Dense* means 'packed closely together'—hence, no gaps in numbers!"

---

### Q123: What is the purpose of PARTITION BY?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A clause used inside the `OVER (...)` window specification to divide the query result set into discrete partitions (subsets) to which the window function is applied independently.
* **Under the Hood:**
  * When `PARTITION BY department_id` is evaluated, the window function's state (counter, running sum, rank) **resets back to 1** for every new partition boundary.
  * **Difference from `GROUP BY`:** `GROUP BY` collapses multiple rows into a single summary row, discarding original individual row details. `PARTITION BY` calculates aggregations or ranks across groups **while preserving the identity of every single row in the output!**

#### 2. Production Code Example
```sql
SELECT 
    employee_id,
    department_id,
    salary,
    -- Running total salary reset per department:
    SUM(salary) OVER (
        PARTITION BY department_id 
        ORDER BY salary ASC
    ) AS running_dept_salary,
    -- Highest salary in the employee's department preserved on each row:
    MAX(salary) OVER (
        PARTITION BY department_id
    ) AS max_dept_salary
FROM employees;
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`PARTITION BY` divides the result set into logical groups so a window function calculates rankings or running totals independently within each group, resetting the calculation at partition boundaries. Unlike `GROUP BY`, which collapses rows into a single summary, `PARTITION BY` retains every individual row in the result set."
* **In Our Project / Production Context:** "We use `PARTITION BY tenant_id` in our multi-tenant SaaS reporting queries to calculate usage percentiles per customer without leaking data across tenants."
* **Pro-Tip & Interview Trap:** "If you omit `PARTITION BY` and only specify `OVER (ORDER BY ...)`, the entire table is treated as a single partition."

---

### Q124: How do you find duplicate records?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Approach 1: `GROUP BY` and `HAVING COUNT(*) > 1`:**
  * Groups rows by the business candidate key columns (e.g., `email` or `first_name, last_name`).
  * Filters groups where the occurrence count exceeds 1.
* **Approach 2: Window Function `ROW_NUMBER()`:**
  * Partitions by duplicate key columns.
  * Filters rows where `rn > 1`.
  * Superior when you need to inspect the actual full duplicate rows alongside their primary key IDs.

#### 2. Production Code Example
```sql
-- Method 1: Identifying duplicate emails and counts
SELECT email, COUNT(*) AS duplicate_count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Method 2: Viewing the entire duplicate rows with IDs
WITH RankedUsers AS (
    SELECT 
        id, 
        email, 
        created_at,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at ASC) AS rn
    FROM users
)
SELECT * FROM RankedUsers WHERE rn > 1;
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "To find duplicate records, we can either use `GROUP BY` on the duplicate key columns with `HAVING COUNT(*) > 1` to find duplicate values and counts, or use `ROW_NUMBER() OVER(PARTITION BY columns ORDER BY id)` inside a CTE to retrieve the actual duplicate rows where the row number is greater than 1."
* **In Our Project / Production Context:** "Before applying a unique index migration on our user contacts table, we ran a `ROW_NUMBER` partition query to identify and export all existing duplicate phone numbers for data cleansing."
* **Pro-Tip & Interview Trap:** "Be mindful of `NULL` values: in SQL, `NULL = NULL` evaluates to unknown, but `GROUP BY` groups `NULL`s together as a single bucket."

---

### Q125: How do you delete duplicate records while keeping one record?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Strategy:**
  1. Partition data by duplicate candidate keys (e.g., `email`).
  2. Order by a tie-breaker (e.g., `id ASC` to keep the oldest record, or `created_at DESC` to keep the newest record).
  3. Assign `ROW_NUMBER()`.
  4. Delete rows where `ROW_NUMBER() > 1`.
* **Database-Specific Syntax:**
  * **PostgreSQL / MySQL 8+ / Oracle:** Using a CTE with `DELETE FROM users WHERE id IN (...)`.
  * **Self-Join Method (Alternative):** `DELETE u1 FROM users u1 INNER JOIN users u2 WHERE u1.email = u2.email AND u1.id > u2.id`.

#### 2. Production Code Example
```sql
-- Production Standard: CTE with ROW_NUMBER() in PostgreSQL
WITH DuplicatesToDelete AS (
    SELECT id
    FROM (
        SELECT 
            id,
            ROW_NUMBER() OVER (
                PARTITION BY email 
                ORDER BY created_at ASC, id ASC -- Keep earliest record (rn = 1)
            ) AS rn
        FROM users
    ) sub
    WHERE sub.rn > 1
)
DELETE FROM users
WHERE id IN (SELECT id FROM DuplicatesToDelete);
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "To delete duplicate records while preserving one original, we use a CTE with `ROW_NUMBER() OVER(PARTITION BY duplicate_columns ORDER BY id ASC)` to assign sequential numbers, and then delete all rows where `rn > 1`. This safely deletes duplicates while preserving the lowest ID row."
* **In Our Project / Production Context:** "During a database migration cleanup in PostgreSQL, we wrapped this CTE deletion script in a transaction to purge duplicate customer records before adding a unique constraint."
* **Pro-Tip & Interview Trap:** "In MySQL, you cannot delete directly from a table that is being selected in a subquery without using an alias or derived table wrapper. Using a join or the CTE syntax avoids this restriction."

---

### Q126: How do you find the nth-highest salary?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Best Practice: `DENSE_RANK()`:**
  * Correctly accounts for ties (if two people make the highest salary, the next distinct salary is still #2).
* **Alternative: `LIMIT 1 OFFSET (N-1)`:**
  * Works well in MySQL/PostgreSQL for distinct values: `SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET (N-1)`.
  * Requires `DISTINCT` to avoid ties throwing off the offset count.

#### 2. Production Code Example
```sql
-- Generic & Robust: 2nd Highest Salary using DENSE_RANK()
WITH RankedSalaries AS (
    SELECT 
        employee_id,
        name,
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT name, salary
FROM RankedSalaries
WHERE rnk = 2; -- Change '2' to any N!

-- Alternative using LIMIT / OFFSET (PostgreSQL / MySQL)
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1; -- For 2nd highest: OFFSET is (N - 1)
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "The most robust way to find the Nth-highest salary is using `DENSE_RANK() OVER (ORDER BY salary DESC)` inside a CTE and filtering `WHERE rnk = N`. `DENSE_RANK` is essential because if multiple employees tie for first place, it ensures the next salary is correctly ranked as #2 without skipping. Alternatively, `SELECT DISTINCT salary ... LIMIT 1 OFFSET (N-1)` can be used in MySQL/Postgres."
* **In Our Project / Production Context:** "We use the `DENSE_RANK()` pattern in our performance evaluation engine to identify tier brackets across employee departments."
* **Pro-Tip & Interview Trap:** "Never use `ROW_NUMBER()` or raw `OFFSET` without `DISTINCT` for this question! If the top 3 employees have identical salaries of \$200k, `OFFSET 1` will return \$200k again instead of the true second-highest salary."

---

### Q127: How do you optimize a slow SQL query?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Structured Troubleshooting Methodology:**
  1. **Analyze Execution Plan:** Run `EXPLAIN ANALYZE` (Postgres) or `EXPLAIN` (MySQL) to identify bottlenecks:
     * Look for **Sequential / Full Table Scans** on large tables.
     * Check for high **Cost** operations and disk spills (`External Sort`).
  2. **Index Optimization:**
     * Create **Composite B-Tree Indexes** on columns in `WHERE`, `JOIN`, and `ORDER BY` clauses following the Leftmost Prefix rule.
     * Use **Covering Indexes** (`INCLUDE` clause in Postgres) so queries read directly from index pages without hitting table heap blocks.
  3. **Refactor Bad SQL Anti-Patterns:**
     * Eliminate `SELECT *`: fetch only necessary columns to reduce I/O and network transfer.
     * Avoid applying functions on indexed columns in `WHERE` clauses (e.g., `WHERE YEAR(created_at) = 2024` prevents index usage; rewrite as `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'`).
     * Replace leading wildcards (`LIKE '%term'`) which prevent index seeks with full-text search.
  4. **Update Table Statistics:** Run `ANALYZE table_name;` to update cost statistics for the query planner.

#### 2. Production Code Example
```sql
-- Anti-Pattern: Function on column disables index seek
EXPLAIN ANALYZE
SELECT id, total_amount FROM orders WHERE DATE(created_at) = '2024-05-01'; -- Seq Scan!

-- Optimized Pattern: SARGable query (Search Argument Able) uses Index Scan
EXPLAIN ANALYZE
SELECT id, total_amount 
FROM orders 
WHERE created_at >= '2024-05-01 00:00:00' 
  AND created_at <  '2024-05-02 00:00:00'; -- Index Scan on idx_orders_created_at!
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "To optimize a slow SQL query, I start by running `EXPLAIN ANALYZE` to identify whether the bottleneck is a full table scan, missing index, or nested loop join. I ensure queries are SARGable by removing functions from `WHERE` clauses, add composite or covering B-Tree indexes, eliminate `SELECT *`, and ensure table statistics are updated so the query planner makes optimal cost estimates."
* **In Our Project / Production Context:** "Our transaction history query was taking 4.2 seconds due to `WHERE UPPER(status) = 'PAID'`. Removing the `UPPER()` function and adding an index on `status` brought query execution time down to 18 milliseconds."
* **Pro-Tip & Interview Trap:** "Remember **SARGable** queries: A query is SARGable (Search Argument Able) when the database engine can take advantage of an index. Wrapping an indexed column in a function like `DATE(col)` or `col + 1` makes it non-SARGable."

---

### Q128: What is an execution plan?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** The compiled roadmap generated by the Database Query Optimizer outlining the exact physical operations the database engine will execute to fulfill an SQL statement.
* **Key Components Revealed in an Execution Plan:**
  1. **Access Methods:**
     * **Seq Scan / Full Table Scan:** Reading every block in the table heap from disk (slow for large tables).
     * **Index Scan:** Traverses B-Tree index to locate row pointers, then reads table heap.
     * **Index Only Scan:** Reads all required data directly from the index without touching the table heap (fastest!).
  2. **Join Algorithms:**
     * **Nested Loop Join:** Iterates through outer table and probes inner table; fast for small datasets.
     * **Hash Join:** Builds an in-memory hash table of smaller table and matches larger table; efficient for large unindexed joins.
     * **Merge Join:** Sorts both tables and merges them; efficient when both inputs are pre-sorted.
  3. **Cost vs Actual Rows:** `Cost=0.00..450.20 Rows=1000` (Cost is a synthetic unit of disk I/O and CPU work).

#### 2. Production Code Example
```sql
-- PostgreSQL Execution Plan Analysis
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.order_id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'SHIPPED';
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "An execution plan is the query optimizer's detailed execution roadmap showing the physical steps the engine will take, such as index scans versus table scans, join algorithms like Hash Join or Nested Loop, and cost estimates. We inspect it using `EXPLAIN ANALYZE` to pinpoint bottlenecks like missing indexes, disk spills, or incorrect row estimates."
* **In Our Project / Production Context:** "We use `EXPLAIN ANALYZE` in our staging database whenever a PR introduces a new query. Seeing a 'Seq Scan' on an orders table with 5 million rows immediately signals that a missing index must be added before deploying."
* **Pro-Tip & Interview Trap:** "`EXPLAIN` displays the planner's *estimate* without running the query. `EXPLAIN ANALYZE` actually *executes* the query and reports real runtime latency and buffer reads. Warning: running `EXPLAIN ANALYZE DELETE ...` will actually delete data!"

---

### Q129: What is a Deadlock?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Database Deadlock:** A situation where two or more concurrent database transactions hold exclusive locks on different rows and each transaction attempts to acquire a lock on the row held by the other, forming a cyclic dependency.
* **Under the Hood (Deadlock Detection):**
  * Modern relational databases run a background **Deadlock Detection Daemon** (evaluating wait-for graphs every `deadlock_timeout`, e.g., 1000ms).
  * When a cycle is detected, the database engine arbitrarily picks one transaction as the **deadlock victim**, aborts it, rolls it back, and throws a `DeadlockException` (SQLState `40P01` in Postgres), allowing the other transaction to proceed.

#### 2. Production Code Example
```sql
-- Transaction 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- Holds lock on ID 1
-- (Transaction 2 locks ID 2 simultaneously)
UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- WAITS for Transaction 2 lock!

-- Transaction 2:
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;  -- Holds lock on ID 2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;  -- WAITS for Transaction 1 lock!
-- >>> DEADLOCK DETECTED! Database aborts Transaction 2 as victim <<<
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "A database deadlock occurs when two concurrent transactions hold exclusive row locks and wait cyclically for each other's locks. The database engine automatically detects the cycle using a wait-for graph, terminates one transaction as the deadlock victim with an exception, and rolls it back. We prevent deadlocks by ensuring all transactions acquire locks in the exact same order and by keeping transactions short."
* **In Our Project / Production Context:** "In our funds transfer module, concurrent transfers between User A and User B caused occasional deadlocks. We eliminated them by sorting account IDs numerically (`id1 < id2`) before issuing `SELECT ... FOR UPDATE` row locks."
* **Pro-Tip & Interview Trap:** "Deadlocks cannot be 100% prevented in complex high-concurrency systems. In addition to lock ordering, applications must implement retry logic with exponential backoff and jitter when catching `CannotAcquireLockException`."

---

### Q130: What is the difference between COMMIT and ROLLBACK?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Operation | `COMMIT` | `ROLLBACK` |
| :--- | :--- | :--- |
| **Action** | **Permanently saves** all database mutations made during the current transaction. | **Undoes and cancels** all mutations made during the current transaction. |
| **Data Visibility** | Changes become permanent and **visible to all other concurrent transactions** (satisfying ACID Durability). | Changes are erased as if they never happened; original state remains. |
| **Lock Release** | Releases all acquired row/table locks and closes the transaction. | Releases all acquired row/table locks and closes the transaction. |
| **Write-Ahead Log (WAL)** | Flushes commit record to the WAL on disk to guarantee durability. | Writes abort record to WAL; data pages revert to pre-transaction state. |

#### 2. Production Code Example
```sql
BEGIN;

UPDATE inventory SET stock = stock - 1 WHERE sku = 'MACBOOK-PRO';
INSERT INTO order_audit (order_id, action) VALUES ('ORD-101', 'CREATED');

-- If payment succeeds:
COMMIT; -- Permanently written to database!

-- If payment fails:
-- ROLLBACK; -- Reverts inventory stock and cancels audit record insertion!
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`COMMIT` permanently writes all changes made within a transaction to disk, releases locks, and makes the updates visible to other transactions, satisfying ACID durability. `ROLLBACK` cancels all changes made in the transaction, releases locks, and restores the database to its pre-transaction state."
* **In Our Project / Production Context:** "In Spring Boot, `@Transactional` handles this automatically: if the method completes normally, Spring calls `connection.commit()`. If an unhandled `RuntimeException` is thrown, Spring immediately calls `connection.rollback()`."
* **Pro-Tip & Interview Trap:** "Mention **Savepoints**: You can create partial rollback checkpoints within a transaction using `SAVEPOINT sp1;` and `ROLLBACK TO SAVEPOINT sp1;`, allowing you to roll back a specific failed sub-operation without discarding the entire transaction."

---

### Q131: What are isolation levels?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **ANSI SQL Isolation Levels (from lowest to highest):**
  1. **READ UNCOMMITTED:** Transactions can read uncommitted data from other transactions. Suffers from **Dirty Reads**, **Non-Repeatable Reads**, and **Phantom Reads**.
  2. **READ COMMITTED (Default in Postgres, Oracle, SQL Server):** Prevents Dirty Reads. Transactions only read committed data. Suffers from **Non-Repeatable Reads** and **Phantom Reads**.
  3. **REPEATABLE READ (Default in MySQL InnoDB):** Prevents Dirty Reads and Non-Repeatable Reads. Guarantees that if a row is read twice in a transaction, its values will not change. Uses Multi-Version Concurrency Control (MVCC) snapshots.
  4. **SERIALIZABLE:** Highest isolation. Prevents all anomalies including **Phantom Reads**. Emulates strict serial transaction execution, but carries high lock contention and transaction abort rates.
* **The 3 Read Phenomena:**
  * **Dirty Read:** Reading data written by an uncommitted concurrent transaction that might later be rolled back.
  * **Non-Repeatable Read:** Reading a row, and upon re-reading the *same row*, finding its column values have changed because another transaction committed an update.
  * **Phantom Read:** Re-executing a range query (`WHERE age > 30`) and finding *new rows* have appeared because another transaction committed an insert.

#### 2. Production Code Example
```java
// Spring Boot Declarative Isolation Level
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void calculateQuarterlyDividends(Long portfolioId) {
    // Guarantees row values don't change between multiple reads in this transaction
    List<Stock> stocks = portfolioRepo.findStocks(portfolioId);
    // ...
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Isolation levels control the degree to which concurrent transactions are isolated from each other to prevent concurrency anomalies. The four levels are Read Uncommitted, Read Committed (the industry standard default), Repeatable Read, and Serializable. Higher isolation levels prevent dirty reads, non-repeatable reads, and phantom reads, but reduce concurrency and throughput."
* **In Our Project / Production Context:** "We use the default `READ_COMMITTED` for 98% of our transactional endpoints, but apply `REPEATABLE_READ` or pessimistic locking on inventory deductions to prevent race conditions during flash sales."
* **Pro-Tip & Interview Trap:** "PostgreSQL's `REPEATABLE READ` also prevents phantom reads due to its MVCC snapshot implementation, exceeding standard ANSI SQL specifications."

---

### Q132: What is a Composite Index?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** A single database index created across **two or more columns** of a table: `CREATE INDEX idx_user ON users (company_id, status, created_at);`.
* **Under the Hood (B-Tree Multi-Column Layout):**
  * The index is sorted first by the 1st column (`company_id`). For identical values of the 1st column, it is sorted by the 2nd column (`status`), and so on.
* **The Leftmost Prefix Rule:**
  * A composite index `(A, B, C)` can speed up queries filtering on:
    * `WHERE A = ?`
    * `WHERE A = ? AND B = ?`
    * `WHERE A = ? AND B = ? AND C = ?`
  * **It CANNOT be used efficiently** if column `A` is omitted (e.g., `WHERE B = ? AND C = ?` cannot perform an index seek on `(A, B, C)`).

#### 2. Production Code Example
```sql
-- Creating composite index
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);

-- 1. Uses the composite index efficiently (Leftmost column present):
SELECT * FROM orders WHERE customer_id = 101 AND status = 'COMPLETED';

-- 2. Uses the composite index efficiently (Leftmost column present):
SELECT * FROM orders WHERE customer_id = 101;

-- 3. CANNOT use index efficiently (Leftmost column omitted - requires Full Scan!):
SELECT * FROM orders WHERE status = 'COMPLETED';
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "A composite index is an index on multiple columns that stores data sorted hierarchically by the specified column order. It strictly obeys the Leftmost Prefix Rule: queries must filter on the leading columns of the index to utilize it for seeks. When designing a composite index, we place high-cardinality equality columns first, followed by range or sort columns."
* **In Our Project / Production Context:** "In our orders table, queries regularly filter `WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC`. Creating a composite index on `(tenant_id, status, created_at)` eliminated file sorts and sped up our merchant dashboard by 8x."
* **Pro-Tip & Interview Trap:** "Column ordering matters enormously! Always put **equality filter columns first**, and **range filter or sorting columns last** (Equality -> Sort -> Range rule)."

---

### Q133: What is the difference between EXISTS and IN?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | `IN` Operator | `EXISTS` Operator |
| :--- | :--- | :--- |
| **Evaluation Mechanism** | Evaluates the subquery, collects all matching values into an in-memory list, and checks if outer value matches. | **Boolean Short-Circuit:** Evaluates whether the subquery returns **at least one row**. Stops scanning immediately upon first match ($O(1)$ short-circuit). |
| **Performance with Large Subqueries** | Slower if subquery returns millions of rows (creates large in-memory value set). | **Extremely fast on large tables** because it halts on the first matching row. |
| **Handling `NULL` Values** | **Dangerous with `NOT IN`!** If subquery contains a single `NULL`, `NOT IN` returns zero rows! | **Null-safe:** `EXISTS` checks for row existence; `NULL` column values do not break it. |
| **Syntax Style** | `WHERE id IN (SELECT user_id FROM orders)` | `WHERE EXISTS (SELECT 1 FROM orders WHERE orders.user_id = users.id)` |

#### 2. Production Code Example
```sql
-- 1. IN: Compares against a set of values
SELECT name FROM customers 
WHERE id IN (SELECT customer_id FROM orders WHERE total_amount > 1000);

-- 2. EXISTS: Correlated boolean check (Short-circuits immediately)
SELECT c.name FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.id AND o.total_amount > 1000
);

-- 3. CRITICAL TRAP: NOT IN with NULL returns EMPTY SET!
-- If any order has customer_id IS NULL:
SELECT name FROM customers WHERE id NOT IN (SELECT customer_id FROM orders); -- RETURNS 0 ROWS!

-- SAFE: NOT EXISTS handles NULLs properly
SELECT c.name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id); -- ACCURATE!
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`IN` collects all matching values from a subquery and compares the outer value against the set, whereas `EXISTS` evaluates a boolean check and short-circuits the moment a single matching row is found. `EXISTS` is superior when the subquery returns large datasets, and crucially, `NOT EXISTS` is immune to the fatal `NULL` comparison trap that causes `NOT IN` to return zero rows."
* **In Our Project / Production Context:** "We refactored all customer segmentation queries from `NOT IN` to `NOT EXISTS` after a single `NULL` customer ID in a legacy table caused our inactive customer export script to return zero records."
* **Pro-Tip & Interview Trap:** "Always highlight the `NOT IN` with `NULL` trap: In SQL, `id NOT IN (1, 2, NULL)` evaluates to `UNKNOWN`, which results in `false` for every single row!"

---

### Q134: What is the difference between a Stored Procedure and a Function?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | Stored Procedure (`PROCEDURE`) | User-Defined Function (`FUNCTION`) |
| :--- | :--- | :--- |
| **Return Value** | May return **zero, one, or multiple values** (via `OUT` parameters). | **Must return exactly one value** (or a table in Table-Valued Functions). |
| **Transaction Control** | **Can manage transactions:** Can issue `COMMIT` and `ROLLBACK` statements internally. | **Cannot manage transactions:** Cannot execute `COMMIT` or `ROLLBACK`. |
| **Invocation in SQL** | Invoked using `CALL proc_name()` or `EXEC proc_name`. **Cannot be used inside a `SELECT` statement.** | **Can be called directly inside SQL queries** (e.g., `SELECT calculate_tax(salary) FROM emp`). |
| **Side Effects** | Designed to modify database state (DML: `INSERT`, `UPDATE`, `DELETE`). | Typically designed to calculate and return derived values (pure computation). |

#### 2. Production Code Example
```sql
-- 1. User-Defined Function (Deterministic calculation, called in SELECT)
CREATE OR REPLACE FUNCTION calculate_discount(amount NUMERIC) 
RETURNS NUMERIC AS $$
BEGIN
    RETURN amount * 0.10;
END;
$$ LANGUAGE plpgsql;

-- Usage in SELECT:
SELECT id, total_amount, calculate_discount(total_amount) AS discount FROM orders;

-- 2. Stored Procedure (Performs transactions, invoked via CALL)
CREATE OR REPLACE PROCEDURE process_eod_settlement() 
AS $$
BEGIN
    UPDATE accounts SET balance = balance + interest;
    COMMIT; -- Permitted inside Procedure!
END;
$$ LANGUAGE plpgsql;

-- Usage:
CALL process_eod_settlement();
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "A Stored Procedure is designed for procedural business workflows, can take `IN` and `OUT` parameters, can manage transactions with `COMMIT` and `ROLLBACK`, and is called using `CALL`. A Function must return a single value, cannot perform transaction management, and can be called directly within `SELECT` or `WHERE` clauses."
* **In Our Project / Production Context:** "In our banking database, end-of-day batch reconciliation is executed via a Stored Procedure because it requires manual transaction commits in batches of 1,000 rows. Currency conversions are implemented as Functions so they can be called directly inside reporting queries."
* **Pro-Tip & Interview Trap:** "Remember that user-defined functions called in `SELECT` queries can be a major performance hazard if they are non-deterministic, as they execute row-by-row on millions of rows."

---

### Q135: You need to call an API whenever the user enters a search keyword. How will you avoid calling the API for every keystroke?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **The Problem:** Calling an API on every keystroke (`keyup`) floods the backend with dozens of unnecessary HTTP requests, exhausts network bandwidth, and causes race conditions where an earlier slow search response overwrites a newer fast search response.
* **The Solution: Debouncing with RxJS:**
  1. **`debounceTime(300)`:** Waits for a specified pause in keystrokes (e.g., 300ms) before emitting the latest value. If the user types another letter before 300ms elapses, the timer resets.
  2. **`distinctUntilChanged()`:** Only emits if the current search string is different from the previous search string (avoids searching if user presses arrow keys or caps lock).
  3. **`switchMap()`:** Automatically **cancels in-flight pending HTTP requests** if a new search term arrives before the previous request completed!

#### 2. Production Code Example (Angular / RxJS)
```typescript
import { Component, OnInit } from '@angular/core';
import { FormControl } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

@Component({
  selector: 'app-product-search',
  template: `<input [formControl]="searchControl" placeholder="Search products..." />`
})
export class ProductSearchComponent implements OnInit {
  searchControl = new FormControl('');

  ngOnInit() {
    this.searchControl.valueChanges.pipe(
      debounceTime(300),              // 1. Wait for 300ms pause in typing
      distinctUntilChanged(),         // 2. Ignore if query hasn't changed
      switchMap(query =>              // 3. Cancel previous pending HTTP calls!
        query ? this.productService.search(query).pipe(
          catchError(() => of([]))    // Handle errors gracefully
        ) : of([])
      )
    ).subscribe(results => {
      this.searchResults = results;
    });
  }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "To prevent calling an API on every keystroke, we implement debouncing using RxJS operators on the input's `valueChanges` stream: `debounceTime(300)` waits for a 300ms pause in typing, `distinctUntilChanged()` ignores non-character keystrokes, and `switchMap()` cancels any in-flight pending HTTP requests if the user resumes typing, completely preventing race conditions."
* **In Our Project / Production Context:** "In our auto-complete product search bar, adding debouncing and `switchMap` reduced our search service API traffic by 75% and eliminated out-of-order search result rendering bugs."
* **Pro-Tip & Interview Trap:** "Always mention **`switchMap`**! Debouncing delays the call, but if request #1 takes 2 seconds and request #2 takes 200ms, request #1 might return last and overwrite results. `switchMap` cancels request #1, ensuring only the latest query's response is rendered."

---

### Q136: Your component is making multiple unnecessary API calls. How will you optimize it?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Common Root Causes & Optimizations:**
  1. **Multiple Subscriptions to Same Observable:** Subscribing to an HTTP Observable multiple times (e.g., using `async` pipe 3 times in template).
     * *Fix:* Use **`shareReplay(1)`** on the service Observable to multicast the cached response to all subscribers.
  2. **Default Change Detection Over-firing:** Angular re-renders components on any DOM/timer event.
     * *Fix:* Switch to **`ChangeDetectionStrategy.OnPush`**.
  3. **Loop Re-rendering:** `*ngFor` re-rendering entire DOM lists when items change.
     * *Fix:* Implement **`trackBy`** functions.
  4. **Component Lifecycle Subscriptions Leaking:** Unclosed subscriptions making calls after destruction.
     * *Fix:* Use `takeUntilDestroyed()` (Angular 16+) or clean up in `ngOnDestroy()`.

#### 2. Production Code Example (Angular Service Caching with shareReplay)
```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private userProfile$: Observable<UserProfile> | null = null;

  getUserProfile(): Observable<UserProfile> {
    if (!this.userProfile$) {
      // shareReplay(1) caches response and multicasts to all subscribers
      this.userProfile$ = this.http.get<UserProfile>('/api/v1/profile').pipe(
        shareReplay({ bufferSize: 1, refCount: true })
      );
    }
    return this.userProfile$;
  }

  clearCache() {
    this.userProfile$ = null;
  }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "To eliminate unnecessary API calls in Angular, we cache service HTTP calls using `shareReplay(1)` so multiple template subscribers share a single HTTP execution. We switch components to `ChangeDetectionStrategy.OnPush`, use `trackBy` in `*ngFor` loops, and ensure subscriptions are closed using `takeUntilDestroyed()` to prevent zombie background calls."
* **In Our Project / Production Context:** "Our navigation bar, sidebar, and profile header were each calling `/api/profile` independently. Wrapping the call in a shared service with `shareReplay(1)` consolidated three separate HTTP requests into one."
* **Pro-Tip & Interview Trap:** "Watch out for multiple `async` pipes in templates: `<p>{{ (user$ | async)?.name }}</p><span>{{ (user$ | async)?.email }}</span>` will issue **two separate HTTP requests** unless `shareReplay()` is used in the service or wrapped in an `<ng-container *ngIf="user$ | async as user">`!"

---

### Q137: You accidentally deleted some records. How can you recover them?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Recovery Strategies by Infrastructure Level:**
  1. **Active Transaction (Before Commit):** Issue an immediate `ROLLBACK;` to revert mutations.
  2. **Point-in-Time Recovery (PITR) via Write-Ahead Logs (WAL):**
     * Relational databases (PostgreSQL, AWS Aurora, MySQL) write every change to Write-Ahead Logs before applying them to data pages.
     * PITR restores a base backup snapshot and replays the WAL binary logs up to the exact timestamp (e.g., `2026-11-09 14:22:15`) just before the accidental `DELETE` occurred.
  3. **Flashback Query (Oracle / SQL Server Temporal Tables):**
     * Query data as it existed in the past using system undo tables: `SELECT * FROM users AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR);`.
  4. **Architectural Prevention (Soft Deletes):**
     * Never physically `DELETE` production data. Use a `is_deleted = true` column with Hibernate's `@SQLDelete` and `@SQLRestriction`.

#### 2. Production Code Example (Hibernate Soft Delete Pattern)
```java
@Entity
@Table(name = "customers")
@SQLDelete(sql = "UPDATE customers SET is_deleted = true, deleted_at = NOW() WHERE id = ?")
@SQLRestriction("is_deleted = false") // Automatically filters soft-deleted rows in queries!
public class Customer {
    @Id
    private Long id;
    private String name;

    private boolean isDeleted = false;
    private LocalDateTime deletedAt;
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "If an uncommitted transaction was running, we issue an immediate `ROLLBACK`. If committed in production, we use Point-in-Time Recovery (PITR) via database Write-Ahead Logs to restore a snapshot to the exact second prior to deletion. To architecturally prevent accidental data loss, we enforce the Soft Delete pattern using Hibernate `@SQLDelete` and `@SQLRestriction` so records are merely flagged rather than physically purged."
* **In Our Project / Production Context:** "In our customer database, we enforce soft deletes with an `is_deleted` column. Accidental deletions can be restored in seconds by running `UPDATE customers SET is_deleted = false WHERE id = ?` without needing database restore procedures."
* **Pro-Tip & Interview Trap:** "Never test raw `DELETE` queries on production! In our deployment guidelines, all manual DDL/DML migrations require running inside explicit `BEGIN; ... ROLLBACK;` blocks first to verify row counts affected before committing."

---

### Q138: A stored procedure is running slowly. How will you troubleshoot it?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Step-by-Step Troubleshooting Procedure:**
  1. **Identify the Slow Statement:** Break down the procedure and isolate which specific query inside the procedure is consuming the execution time using database profilers (`pg_stat_statements` or SQL Server Profiler).
  2. **Inspect Parameter Sniffing:** When a procedure compiles, the query optimizer creates a plan based on the parameters passed during the *very first execution*. If the first run used atypical parameters, the cached plan may be horribly unsuited for general parameters.
     * *Fix:* Recompile the procedure or use local variable decoupling: `DECLARE @local_id INT = @param_id;`.
  3. **Check for Table Scans and Missing Indexes:** Run `EXPLAIN ANALYZE` on the identified query.
  4. **Check for Row-by-Row Cursor Loops:** Replace slow `CURSOR` loops with vectorized set-based `UPDATE` or `INSERT` operations.
  5. **Inspect Lock Contention & Blocking:** Check if the procedure is waiting on row locks held by other concurrent transactions (`pg_locks`).
  6. **Update Table Statistics:** Run `ANALYZE` on the involved tables so cardinality estimates are accurate.

#### 2. Production Code Example
```sql
-- Troubleshooting via pg_stat_statements in PostgreSQL:
SELECT 
    query, 
    calls, 
    total_exec_time / calls AS avg_time_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;

-- Parameter Sniffing Fix in SQL Server:
-- Add OPTION (RECOMPILE) to problematic query within the procedure
SELECT * FROM orders 
WHERE customer_id = @customerId 
OPTION (RECOMPILE);
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "To troubleshoot a slow stored procedure, I isolate which individual query is slow using query profilers, analyze its execution plan with `EXPLAIN ANALYZE` for missing indexes or table scans, check for parameter sniffing issues, inspect lock contention from other concurrent transactions, and ensure cursors are refactored into set-based operations."
* **In Our Project / Production Context:** "Our nightly reconciliation stored procedure degraded from 5 minutes to 45 minutes. Profiling revealed a cursor loop updating balances one row at a time. Refactoring the cursor into a single set-based `UPDATE ... FROM` statement reduced execution time to under 90 seconds."
* **Pro-Tip & Interview Trap:** "Parameter sniffing is the most common reason a procedure that runs fast in a local SQL editor runs slowly in production! Local variable assignment inside the procedure forces the optimizer to generate a balanced average plan rather than sniffing the first parameter."
