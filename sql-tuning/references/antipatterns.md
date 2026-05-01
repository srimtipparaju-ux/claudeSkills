# SQL Anti-Patterns Reference

## Category 1: Non-SARGable Predicates (Index Killers)

### Functions on indexed columns

| Anti-pattern | Why bad | Fix |
|---|---|---|
| `WHERE UPPER(name) = 'SMITH'` | Function wraps column → no index range scan | Normalize data to uppercase at insert, OR create FBI: `CREATE INDEX ON t (UPPER(name))` |
| `WHERE TRUNC(order_date) = TRUNC(SYSDATE)` | Converts every date before comparing | `WHERE order_date >= TRUNC(SYSDATE) AND order_date < TRUNC(SYSDATE)+1` |
| `WHERE TO_CHAR(id) = '12345'` | Converts NUMBER to VARCHAR2 | `WHERE id = 12345` (use correct type) |
| `WHERE SUBSTR(code,1,3) = 'ABC'` | Substring prevents range scan | `WHERE code LIKE 'ABC%'` (if prefix known) |
| `WHERE NVL(status,'X') = 'OPEN'` | NVL wraps column | `WHERE status = 'OPEN' OR status IS NULL` |
| `WHERE COALESCE(col, 0) > 100` | Same as NVL | `WHERE col > 100 OR (col IS NULL AND 0 > 100)` |
| `WHERE DECODE(type,'A',1,'B',2) = 1` | Function prevents index | `WHERE type = 'A'` (rewrite logic) |

### Arithmetic on indexed columns

| Anti-pattern | Fix |
|---|---|
| `WHERE salary * 12 > 60000` | `WHERE salary > 5000` |
| `WHERE order_date - 30 > :start` | `WHERE order_date > :start + 30` |
| `WHERE amount / rate > 100` | `WHERE amount > 100 * rate` |
| `WHERE MONTHS_BETWEEN(SYSDATE, hire_date) > 24` | `WHERE hire_date < ADD_MONTHS(SYSDATE, -24)` |

### Implicit type conversions

| Anti-pattern | Problem | Fix |
|---|---|---|
| `WHERE num_col = '123'` | String → NUMBER conversion on col | `WHERE num_col = 123` |
| `WHERE date_col = '2024-01-01'` | String → DATE conversion; NLS-dependent | `WHERE date_col = DATE '2024-01-01'` |
| `WHERE char_col = 123` | NUMBER → VARCHAR2 on column | `WHERE char_col = '123'` |
| JOIN on mismatched types | Conversion on join column | Fix schema or cast the constant, not the column |

---

## Category 2: Join Problems

### Cartesian Products

```sql
-- Missing join predicate (comma syntax easy to miss)
SELECT * FROM orders o, customers c, products p
WHERE o.product_id = p.product_id
-- MISSING: customer join → every order × every customer

-- Explicit but still missing
SELECT * FROM orders o
CROSS JOIN customers c       -- intentional cross join = probably wrong
WHERE o.customer_id = 1
```

**Detection rule:** N tables in FROM → must have at least N-1 join conditions between them.

### Outer join nullified by WHERE filter

```sql
-- LEFT JOIN but WHERE on right table = acts like INNER JOIN
SELECT o.order_id, c.name
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id
WHERE c.country = 'US'    -- ← nullifies the LEFT JOIN

-- Fix A: move filter to JOIN condition (keeps non-matching orders)
LEFT JOIN customers c ON c.id = o.customer_id AND c.country = 'US'

-- Fix B: if INNER JOIN is actually wanted, say so explicitly
INNER JOIN customers c ON c.id = o.customer_id AND c.country = 'US'
```

### Wrong join type

| Situation | Wrong | Right |
|---|---|---|
| Need all orders even without customer | `INNER JOIN customers` | `LEFT JOIN customers` |
| Need only orders WITH a customer | `LEFT JOIN customers` + filtering | `INNER JOIN customers` |
| Need all customers and all orders | No standard join | `FULL OUTER JOIN` |

### Joining on expression instead of column

```sql
-- Prevents index on either side
ON TO_DATE(a.date_str) = b.event_date
ON TRIM(a.code) = TRIM(b.code)

-- Fix: store data in correct format; join on raw columns
```

### Fan-out joins (unintended row multiplication)

```sql
-- orders 1:many order_items 1:many order_events
-- Result: one order row × items × events = huge Cartesian-like explosion
SELECT o.order_id, SUM(i.amount), COUNT(e.event_id)
FROM orders o
JOIN order_items i ON i.order_id = o.order_id
JOIN order_events e ON e.order_id = o.order_id
GROUP BY o.order_id
-- SUM(i.amount) is wrong — items are duplicated for each event!

-- Fix: aggregate each child separately, then join
SELECT o.order_id, i.total_amount, e.event_count
FROM orders o
JOIN (SELECT order_id, SUM(amount) AS total_amount FROM order_items GROUP BY order_id) i
    ON i.order_id = o.order_id
JOIN (SELECT order_id, COUNT(*) AS event_count FROM order_events GROUP BY order_id) e
    ON e.order_id = o.order_id
```

---

## Category 3: Subquery Anti-Patterns

### Correlated subquery in SELECT

```sql
-- Runs once per outer row
SELECT c.customer_id,
       (SELECT MAX(order_date) FROM orders o WHERE o.customer_id = c.customer_id) AS last_order
FROM customers c

-- Fix: JOIN with aggregated subquery
SELECT c.customer_id, o.last_order
FROM customers c
LEFT JOIN (SELECT customer_id, MAX(order_date) AS last_order FROM orders GROUP BY customer_id) o
    ON o.customer_id = c.customer_id
```

### NOT IN with nullable subquery

```sql
-- Returns ZERO rows if any product_id in discontinued is NULL!
WHERE p.id NOT IN (SELECT product_id FROM discontinued)

-- Always safe alternative:
WHERE NOT EXISTS (SELECT 1 FROM discontinued d WHERE d.product_id = p.id)
```

### IN vs EXISTS for large subqueries

```sql
-- IN: entire subquery evaluated, result set held in memory
WHERE id IN (SELECT id FROM large_table WHERE complex_condition)

-- EXISTS: stops at first match — much better for large subqueries
WHERE EXISTS (SELECT 1 FROM large_table l WHERE l.id = outer.id AND l.complex_condition)
```

Rule of thumb:
- Subquery returns small result (< 1,000 rows): IN is fine
- Subquery is large or correlated: use EXISTS
- Checking non-existence: always use NOT EXISTS, never NOT IN

### Unnecessary subquery (can be a direct join)

```sql
-- Redundant subquery wrapper
SELECT * FROM (SELECT * FROM orders WHERE status = 'OPEN') open_orders
WHERE customer_id = :b1

-- Direct:
SELECT * FROM orders WHERE status = 'OPEN' AND customer_id = :b1
```

---

## Category 4: Aggregation Problems

### Filtering after aggregation (should be before)

```sql
-- Wrong: all rows aggregated, then filtered
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
HAVING customer_id IN (SELECT id FROM vip_customers)  -- should be WHERE

-- Right: filter first (reduces rows before aggregation)
SELECT customer_id, SUM(amount)
FROM orders
WHERE customer_id IN (SELECT id FROM vip_customers)
GROUP BY customer_id
```

**Rule:** HAVING is only for conditions on aggregate functions (SUM, COUNT, AVG...). Everything else goes in WHERE.

### Joining before aggregating (row multiplication)

```sql
-- Joins first, then aggregates — SUM may double-count if join causes fan-out
SELECT d.dept_name, SUM(e.salary)
FROM departments d
JOIN employees e ON e.dept_id = d.dept_id
JOIN employee_bonuses b ON b.emp_id = e.emp_id  -- one employee, multiple bonuses
GROUP BY d.dept_name
-- salary is multiplied by bonus count!

-- Fix: aggregate before joining
SELECT d.dept_name, emp.total_salary
FROM departments d
JOIN (SELECT dept_id, SUM(salary) AS total_salary FROM employees GROUP BY dept_id) emp
    ON emp.dept_id = d.dept_id
```

### GROUP BY all non-aggregate columns

```sql
-- Unnecessary: grouping by PK already makes each row unique
SELECT customer_id, customer_name, email, phone, address, COUNT(order_id)
FROM customers c JOIN orders o ON o.customer_id = c.customer_id
GROUP BY customer_id, customer_name, email, phone, address

-- Better: group by PK only, join details after
SELECT c.customer_id, c.customer_name, c.email, c.phone, c.address, agg.order_count
FROM customers c
JOIN (SELECT customer_id, COUNT(*) AS order_count FROM orders GROUP BY customer_id) agg
    ON agg.customer_id = c.customer_id
```

---

## Category 5: SELECT Clause Problems

### SELECT * — five problems

1. Fetches unneeded columns (network + memory overhead)
2. Prevents covering index usage
3. Breaks if schema changes (new columns appear)
4. Causes ambiguous column names after joins
5. Makes query intent impossible to understand

**Always list specific columns.**

### DISTINCT as a join band-aid

```sql
-- DISTINCT hiding that the join multiplies rows
SELECT DISTINCT c.customer_id, c.name
FROM customers c JOIN orders o ON o.customer_id = c.customer_id

-- Better: check existence without joining
SELECT customer_id, name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id)
```

### Scalar subquery in SELECT

Already covered above — always replace with LEFT JOIN + GROUP BY.

---

## Category 6: WHERE Clause Patterns

### OR across different indexed columns

```sql
WHERE status = 'OPEN' OR region = 'WEST'
-- One index cannot serve both; likely full scan

-- Fix: UNION ALL (each branch can use its own index)
SELECT * FROM orders WHERE status = 'OPEN'
UNION ALL
SELECT * FROM orders WHERE region = 'WEST' AND status != 'OPEN'
```

### BETWEEN on dates (use half-open range)

```sql
-- Ambiguous with timestamps: does '2024-01-31' include 23:59:59?
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'

-- Precise half-open range (always correct)
WHERE order_date >= DATE '2024-01-01' AND order_date < DATE '2024-02-01'
```

### NULL handling

```sql
-- These do NOT catch NULLs:
WHERE col != 'VALUE'       -- NULLs excluded
WHERE col NOT IN (1,2,3)   -- NULLs excluded

-- To include NULLs:
WHERE col != 'VALUE' OR col IS NULL
WHERE col IS NULL OR col NOT IN (1,2,3)
```

---

## Category 7: DML Anti-Patterns

### Row-by-row processing (RBAR — Row By Agonizing Row)

```sql
-- PL/SQL loop updating one row at a time
FOR rec IN (SELECT order_id FROM orders WHERE status = 'PENDING') LOOP
    UPDATE orders SET status = 'PROCESSED', updated_at = SYSDATE WHERE order_id = rec.order_id;
END LOOP;

-- Fix: single set-based UPDATE
UPDATE orders SET status = 'PROCESSED', updated_at = SYSDATE WHERE status = 'PENDING';
```

### DELETE when TRUNCATE is faster

```sql
-- Slow: generates undo for every row
DELETE FROM staging_table;

-- Fast: no row-level undo, auto-commit (use only when appropriate)
TRUNCATE TABLE staging_table;
```

### INSERT SELECT without filter

```sql
-- Copies entire source table — usually unintentional
INSERT INTO archive SELECT * FROM orders;

-- Add appropriate filter
INSERT INTO archive SELECT * FROM orders WHERE order_date < ADD_MONTHS(SYSDATE, -24);
```

### UPDATE with correlated subquery (use MERGE)

```sql
-- Slow: subquery executes per row being updated
UPDATE orders o
SET status = (SELECT new_status FROM status_updates u WHERE u.order_id = o.order_id)
WHERE EXISTS (SELECT 1 FROM status_updates u WHERE u.order_id = o.order_id);

-- Fast: MERGE
MERGE INTO orders o
USING status_updates u ON u.order_id = o.order_id
WHEN MATCHED THEN UPDATE SET o.status = u.new_status;
```

---

## Category 8: Oracle-Specific Issues

### ROWNUM pagination (pre-12c style — prone to bugs)

```sql
-- Wrong order of operations: ROWNUM assigned BEFORE ORDER BY
SELECT * FROM (
    SELECT o.*, ROWNUM rn FROM orders o ORDER BY order_date DESC
) WHERE rn BETWEEN 11 AND 20;
-- ROWNUM is applied to unsorted rows, then ORDER BY sorts — order of rn is wrong!

-- Correct: sort in inner query, apply ROWNUM in middle, filter in outer
SELECT * FROM (
    SELECT o.*, ROWNUM rn FROM (
        SELECT * FROM orders ORDER BY order_date DESC
    ) o WHERE ROWNUM <= 20
) WHERE rn > 10;

-- Best (Oracle 12c+): use SQL standard syntax
SELECT * FROM orders ORDER BY order_date DESC
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;
```

### Hint misuse

```sql
-- Forcing an index that may not be best
/*+ INDEX(o idx_orders_status) */

-- Hints should be last resort; fix stats/SQL first
-- If hint needed, document why and version it
```

### NVL vs COALESCE vs CASE

```sql
NVL(a, b)          -- evaluates both a and b always (b even if a is not null)
COALESCE(a, b)     -- evaluates lazily — stops at first non-null (better)
NVL2(a, b, c)      -- if a not null return b, else c

-- COALESCE is ANSI standard and preferred
```

### DECODE vs CASE

```sql
DECODE(status, 'OPEN', 1, 'CLOSED', 2, 0)   -- Oracle-specific, limited
CASE status WHEN 'OPEN' THEN 1 WHEN 'CLOSED' THEN 2 ELSE 0 END  -- ANSI, preferred
```

### Implicit vs explicit date conversion

```sql
-- Dangerous: NLS-dependent, breaks in different environments
WHERE hire_date = '01-JAN-24'

-- Safe: always explicit
WHERE hire_date = DATE '2024-01-01'
WHERE hire_date = TO_DATE('2024-01-01', 'YYYY-MM-DD')
```
