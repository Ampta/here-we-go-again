# 🚀 Day 5 Masterclass: Composite Keys, Advanced B+Tree Index Types, Slow Query Diagnostics & Stored Procedures Architecture

---

## Part 1: Detailed Breakdown of Your Class Notes

---

### 1. Composite Primary Keys

#### Class Notes Code:
```sql
CREATE TABLE order_items (
    order_id INT(4),
    product_name VARCHAR(20),
    quantity INT(4),
    price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_name)
);

INSERT INTO order_items (order_id, product_name, quantity, price) VALUES
(1001, 'Laptop', 1, 899.99),
(1001, 'Mouse', 2, 25.50),     -- Same order_id (1001), different product
(1002, 'Mouse', 1, 25.50),     -- Different order_id, same product
(1002, 'Keyboard', 1, 45.00),
(1003, 'Headphones', 3, 79.99);

-- Duplicate Key Error Test:
-- Inserting (1001, 'Laptop') again will throw Error 1062 (23000): Duplicate entry '1001-Laptop' for key 'PRIMARY'
```

#### What Happens Under the Hood:
A **Composite Primary Key** combines two or more columns to form a unique identifier for each row in a table.

```
                         Composite Key: (order_id, product_name)
                         ┌──────────┬──────────────┬──────────┐
                         │ order_id │ product_name │ Unique?  │
                         ├──────────┼──────────────┼──────────┤
                         │ 1001     │ Laptop       │ ✅ Row 1 │
                         │ 1001     │ Mouse        │ ✅ Row 2 │
                         │ 1002     │ Mouse        │ ✅ Row 3 │
                         │ 1001     │ Laptop       │ ❌ Error │
                         └──────────┴──────────────┴──────────┘
```

#### 3-Year Dev Use Cases:
1. **Junction / Pivot Tables (Many-to-Many Relationships):** In e-commerce or ERP systems, linking `orders` and `products` requires a junction table `order_items`. Using `(order_id, product_id)` as a composite primary key guarantees that a product cannot be added twice as separate line items in the same order.
2. **Multi-Tenant Partitioned Architecture:** In SaaS platforms, composite keys like `(tenant_id, user_id)` ensure tenant data isolation at the index level.

---

### 2. Deep-Dive into Database Indexing & B+Tree Architecture

#### Class Notes Overview:
```sql
-- Unique Index Creation
CREATE UNIQUE INDEX idx_hero_email ON heros(email);

-- Composite Index Creation
CREATE INDEX idx_dept_salary ON heros(department, salary);

-- Single Column Index
CREATE INDEX idx_salary ON heros(salary);

-- Inspection & Deletion
SHOW INDEX FROM heros;
DROP INDEX idx_salary ON heros;
```

#### Storage Engine Index Implementations Breakdown:

```
                               Index Types Matrix
 ┌───────────┬───────────────────────────────┬──────────────────────────────────────┐
 │ Index Type│ Underlying Data Structure     │ Best Used For                        │
 ├───────────┼───────────────────────────────┼──────────────────────────────────────┤
 │ B+Tree    │ Balanced Tree + Linked Leaves │ `=`, `>`, `<`, `BETWEEN`, `ORDER BY`  │
 │ Hash      │ In-Memory Hash Table          │ Exact Equality (`=`) - O(1) Lookup   │
 │ Fulltext  │ Inverted Text Index           │ Word & Phrase Search (`MATCH/AGAINST`)│
 │ Spatial   │ R-Tree (Spatial Bounding)     │ GIS Coordinates (`ST_Contains`, GPS) │
 └───────────┴───────────────────────────────┴──────────────────────────────────────┘
```

#### Why MySQL Uses B+Trees Instead of Binary Search Trees (BST):
1. **High Fan-Out (Lower Tree Height):** Binary trees split into 2 branches per node. A dataset of 10,000,000 rows requires a BST height of $\approx 24$ levels (24 separate disk reads!). B+Trees have a fan-out of hundreds of keys per 16KB page node, keeping tree height at just 3 to 4 levels!
2. **Sequential Leaf Page Scans:** In a B+Tree, all data pointers sit in the bottom leaf nodes, and leaf nodes form a **doubly linked list**. Performing a range query (`WHERE salary BETWEEN 50000 AND 90000`) traverses to the starting key once and scans adjacent leaf pages sequentially without re-traversing tree branches.

---

### 3. Slow Query Diagnostics & Execution Plan Profiling

#### Class Notes Code:
```sql
-- Enable Slow Query Log globally
SET GLOBAL slow_query_log = 'ON';
SHOW VARIABLES LIKE 'slow_query_log';

-- Set threshold to 2 seconds
SET GLOBAL long_query_time = 2;
```

#### Causes of Slow Queries in Production:
1. **Missing Indexes:** Searching on unindexed `WHERE`, `JOIN`, or `ORDER BY` columns forces MySQL into a disk-bound Full Table Scan (`type: ALL`).
2. **`SELECT *` Anti-Pattern:** Fetching unused `MEDIUMTEXT` or `JSON` columns increases network buffer overhead and prevents MySQL from utilizing **Covering Indexes** (`Using index`).
3. **Function Wrappers on Indexed Columns:** `WHERE UPPER(email) = 'USER@TEST.COM'` prevents index usage.
4. **Deep Subquery Nesting:** Forces MySQL to materialize unindexed intermediate derived tables.

#### Decoding `EXPLAIN` Access Types (Best to Worst Hierarchy):

```
  [BEST]                                                                        [WORST]
  System ──> Const ──> Eq_ref ──> Ref ──> Fulltext ──> Range ──> Index ──> ALL
```

| Join Type | Meaning | Production Performance |
| :--- | :--- | :--- |
| `system` / `const` | Single matching row returned from Primary Key / Unique Index. | ⚡ Lightning Fast ($O(1)$) |
| `eq_ref` | One row read from indexed Primary Key for each row combination in a `JOIN`. | ⚡ Excellent ($O(\log N)$) |
| `ref` | Non-unique index lookup matching multiple rows. | ✅ Fast |
| `range` | Index range scan using operators like `>`, `<`, `BETWEEN`, `IN`, `LIKE 'prefix%'`. | ✅ Good |
| `index` | Scans the entire B+Tree index sequentially (Covering index scan). | ⚠️ Slow for large tables |
| `ALL` | **Full Table Scan!** Reads every physical data page from disk. | ❌ Catastrophic |

---

### 4. Stored Procedures Architecture (`CREATE PROCEDURE`)

#### Class Notes Code:
```sql
-- Procedure with IN parameters:
DELIMITER //
CREATE PROCEDURE get_heroes_by_dept (IN dept_name VARCHAR(20))
BEGIN 
    SELECT * FROM heros 
    WHERE department = dept_name;
END //
DELIMITER ;

CALL get_heroes_by_dept('Finance');

-- Procedure with OUT parameters:
DELIMITER //
CREATE PROCEDURE count_heroes_by_dept (IN dept_name VARCHAR(20), OUT total_count INT)
BEGIN
    SELECT COUNT(*) INTO total_count 
    FROM heros 
    WHERE department = dept_name;
END //
DELIMITER ;

CALL count_heroes_by_dept('Finance', @total_count);
SELECT @total_count;

-- Procedure with Conditional IF/ELSE Logic:
DELIMITER //
CREATE PROCEDURE calculate_hero_bonus(IN hero_id INT, OUT new_salary DECIMAL(10,2))
BEGIN 
    DECLARE current_sal DECIMAL(10,2);
    
    SELECT salary INTO current_sal
    FROM heros
    WHERE id = hero_id;
    
    IF current_sal >= 80000.00 THEN 
        SET new_salary = current_sal + (current_sal * 0.20); -- 20% bonus for high earners
    ELSE 
        SET new_salary = current_sal + (current_sal * 0.10); -- 10% bonus
    END IF;
END //
DELIMITER ;

CALL calculate_hero_bonus(31, @new_salary);
SELECT @new_salary;
```

#### Why Stored Procedures are Used in Enterprise Architectures:
1. **Pre-Compiled Execution Plans:** MySQL compiles and parses the procedure once. Subsequent `CALL` invocations execute instantly without parsing SQL text overhead.
2. **Network Traffic Reduction:** Instead of sending multi-page SQL queries across the network from an API server to the database server, the backend sends a lightweight 1-line `CALL proc_name(args)` instruction.
3. **Encapsulation & Security:** Granting execution privileges on stored procedures allows users to modify data without exposing direct write access to base tables.

---

### 5. Enterprise Exception Handling in Stored Procedures

#### Class Notes Syntax:
```sql
DECLARE handler_action HANDLER FOR condition_value handler_statement;
```

#### Handling Errors & Custom Exceptions:
```sql
DELIMITER //
CREATE PROCEDURE safe_salary_update(IN hero_id INT, IN new_sal DECIMAL(10,2))
BEGIN
    -- Declare Exit Handler for duplicate key or SQL exceptions
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Rollback changes if an error occurs
        ROLLBACK;
        SELECT 'Transaction Failed: Invalid data or constraint violation' AS error_message;
    END;

    START TRANSACTION;
    
    -- Custom Validation Signal
    IF new_sal < 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Salary cannot be negative!';
    END IF;

    UPDATE heros SET salary = new_sal WHERE id = hero_id;
    
    COMMIT;
END //
DELIMITER ;
```

#### Standard Handler Actions & SQLSTATES:
* **`CONTINUE`:** Catches error, finishes current block, and resumes execution at the next statement.
* **`EXIT`:** Catches error and immediately terminates execution of the enclosing `BEGIN...END` block.
* **Common SQLSTATE Codes:**
  - `SQLEXCEPTION`: Generic catch-all for database errors.
  - `NOT FOUND` (`SQLSTATE '02000'`): Query returned zero rows (useful when fetching cursors).
  - `23000`: Constraint violation (Duplicate primary key or foreign key violation).
  - `45000`: User-defined custom exception raised via `SIGNAL`.

---

## Part 2: Missing Topics to Bridge to a 3-Year Developer Level

---

### 1. Database Cursors inside Stored Procedures

When you need to iterate row-by-row through a `SELECT` result set inside MySQL, use a **Cursor**:

```sql
DELIMITER //
CREATE PROCEDURE process_annual_appraisals()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE h_id INT;
    DECLARE h_sal DECIMAL(10,2);
    
    -- Step 1: Declare Cursor
    DECLARE hero_cursor CURSOR FOR 
        SELECT id, salary FROM heros;
        
    -- Step 2: Declare NOT FOUND handler to stop loop
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    OPEN hero_cursor;

    read_loop: LOOP
        FETCH hero_cursor INTO h_id, h_sal;
        IF done = 1 THEN
            LEAVE read_loop;
        END IF;

        -- Process each row individually
        IF h_sal < 60000 THEN
            UPDATE heros SET salary = salary + 5000 WHERE id = h_id;
        END IF;
    END LOOP;

    CLOSE hero_cursor;
END //
DELIMITER ;
```

---

### 2. Dynamic SQL Execution via Prepared Statements

To execute dynamic SQL strings constructed at runtime inside stored procedures:

```sql
DELIMITER //
CREATE PROCEDURE search_heroes_dynamic(IN search_col VARCHAR(50), IN search_val VARCHAR(50))
BEGIN
    SET @sql_query = CONCAT('SELECT * FROM heros WHERE ', search_col, ' = ?');
    
    -- Prepare, execute, and deallocate statement
    PREPARE stmt FROM @sql_query;
    SET @val = search_val;
    EXECUTE stmt USING @val;
    DEALLOCATE PREPARE stmt;
END //
DELIMITER ;
```

---

## Part 3: 💡 Summary Cheat Sheet for Day 5

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                              DAY 5 SUMMARY                             │
 ├──────────────────┬─────────────────────────────────────────────────────┤
 │ Composite PK     │ PRIMARY KEY (col1, col2) enforces combined multi-   │
 │                  │ column uniqueness. Essential for junction tables.   │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ B+Tree Indexing  │ High fan-out keeps tree depth at 3-4 levels. Leaf   │
 │                  │ nodes linked sequentially for range lookups.        │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Slow Query Log   │ SET GLOBAL slow_query_log='ON'. Use EXPLAIN to turn │
 │                  │ type 'ALL' (Full Table Scan) into 'ref' or 'range'. │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Stored Procedures│ Pre-compiled routines on DB server. Reduces network │
 │                  │ payload, accepts IN/OUT params, supports IF/CASE.   │
 ├──────────────────┼─────────────────────────────────────────────────────┤
 │ Exception Handling│ DECLARE EXIT HANDLER FOR SQLEXCEPTION. Use SIGNAL   │
 │                  │ SQLSTATE '45000' to throw custom validation errors. │
 └──────────────────┴─────────────────────────────────────────────────────┘
```
