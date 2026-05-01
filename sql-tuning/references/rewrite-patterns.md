# SQL Rewrite Patterns Reference

## Pattern 1: Scalar Subquery → LEFT JOIN + GROUP BY

### Before
```sql
SELECT
    c.customer_id,
    c.customer_name,
    (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count,
    (SELECT MAX(order_date) FROM orders o WHERE o.customer_id = c.customer_id) AS last_order,
    (SELECT SUM(amount) FROM orders o WHERE o.customer_id = c.customer_id) AS total_spent
FROM customers c;
```
**Problem:** Three separate subqueries, each scanning orders once per customer.

### After
```sql
SELECT
    c.customer_id,
    c.customer_name,
    COALESCE(o.order_count, 0)  AS order_count,
    o.last_order,
    COALESCE(o.total_spent, 0)  AS total_spent
FROM customers c
LEFT JOIN (
    SELECT
        customer_id,
        COUNT(*)          AS order_count,
        MAX(order_date)   AS last_order,
        SUM(amount)       AS total_spent
    FROM orders
    GROUP BY customer_id
) o ON o.customer_id = c.customer_id;
```
**Gain:** Orders table scanned once instead of 3× customer count.

---

## Pattern 2: Correlated EXISTS → Anti-Join

### Before
```sql
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```
This is already good. But a common equivalent anti-pattern:

```sql
-- Bad: NOT IN with nullable column
SELECT * FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);
-- If any order has NULL customer_id → returns ZERO rows!
```

### After (always safe)
```sql
SELECT c.*
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.customer_id IS NULL;
-- OR use NOT EXISTS (also correct and often faster)
```

---

## Pattern 3: DISTINCT on Join → EXISTS

### Before
```sql
SELECT DISTINCT c.customer_id, c.customer_name, c.email
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE o.status = 'OPEN';
```
**Problem:** Join multiplies rows (one per open order), then DISTINCT de-duplicates with an expensive sort.

### After
```sql
SELECT c.customer_id, c.customer_name, c.email
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id AND o.status = 'OPEN'
);
```
**Gain:** No row multiplication, no sort. Index on `orders(customer_id, status)` makes this very fast.

---

## Pattern 4: OR → UNION ALL

### Before
```sql
SELECT * FROM orders
WHERE status = 'OPEN'
   OR region = 'WEST';
```
**Problem:** OR across two different columns prevents index use on either.

### After
```sql
SELECT * FROM orders WHERE status = 'OPEN'
UNION ALL
SELECT * FROM orders WHERE region = 'WEST' AND status != 'OPEN';
-- Note: second branch excludes rows already in first to avoid duplicates
-- If duplicates are acceptable (rare): just UNION ALL both without the exclusion
```
Each branch can now use its own index.

---

## Pattern 5: Aggregation Before Join (Reduce-then-Join)

### Before
```sql
SELECT d.dept_name, e.first_name, e.last_name, SUM(o.amount) AS dept_total
FROM departments d
JOIN employees e ON e.dept_id = d.dept_id
JOIN orders o ON o.sales_rep_id = e.emp_id
GROUP BY d.dept_name, e.first_name, e.last_name;
```
**Problem:** Joins all orders to all employees before aggregating — large intermediate result.

### After
```sql
SELECT d.dept_name, e.first_name, e.last_name, rep_sales.total
FROM departments d
JOIN employees e ON e.dept_id = d.dept_id
JOIN (
    SELECT sales_rep_id, SUM(amount) AS total
    FROM orders
    GROUP BY sales_rep_id
) rep_sales ON rep_sales.sales_rep_id = e.emp_id;
```
**Gain:** Aggregation reduces orders to one row per sales rep before joining to employees.

---

## Pattern 6: Row-by-Row → Set-Based UPDATE

### Before (PL/SQL anti-pattern visible in SQL context)
```sql
-- Conceptual: N separate UPDATE statements, one per order
UPDATE orders SET status = 'EXPIRED' WHERE order_id = 1001;
UPDATE orders SET status = 'EXPIRED' WHERE order_id = 1002;
UPDATE orders SET status = 'EXPIRED' WHERE order_id = 1003;
-- ... × 50,000 rows
```

### After
```sql
-- Single set-based UPDATE
UPDATE orders
SET status = 'EXPIRED', updated_at = SYSDATE
WHERE status = 'PENDING'
  AND order_date < ADD_MONTHS(SYSDATE, -6);
```

### After (updating from another table)
```sql
-- Use MERGE instead of correlated UPDATE
MERGE INTO orders tgt
USING (
    SELECT order_id, new_status, updated_by
    FROM order_status_changes
    WHERE processed = 'N'
) src ON src.order_id = tgt.order_id
WHEN MATCHED THEN
    UPDATE SET tgt.status = src.new_status,
               tgt.last_modified_by = src.updated_by,
               tgt.updated_at = SYSDATE;
```

---

## Pattern 7: Pagination — ROWNUM → OFFSET FETCH

### Before (Oracle pre-12c, still common — and often wrong)
```sql
-- BUG: ROWNUM applied before ORDER BY in inner query
SELECT * FROM (
    SELECT o.*, ROWNUM rn
    FROM orders o
    WHERE status = 'OPEN'
    ORDER BY order_date DESC   -- ORDER BY happens after ROWNUM assignment!
)
WHERE rn BETWEEN 11 AND 20;
```

### Correct pre-12c pattern (three levels needed)
```sql
SELECT * FROM (
    SELECT o.*, ROWNUM rn
    FROM (
        SELECT * FROM orders
        WHERE status = 'OPEN'
        ORDER BY order_date DESC    -- sort in innermost
    ) o
    WHERE ROWNUM <= 20              -- cap in middle
)
WHERE rn > 10;                      -- offset in outer
```

### Best: Oracle 12c+ SQL standard
```sql
SELECT *
FROM orders
WHERE status = 'OPEN'
ORDER BY order_date DESC
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;
```

### Best for deep pagination (any volume)
```sql
-- Keyset pagination: no OFFSET — uses index efficiently regardless of page depth
SELECT *
FROM orders
WHERE status = 'OPEN'
  AND (order_date, order_id) < (:last_date, :last_id)  -- from previous page
ORDER BY order_date DESC, order_id DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## Pattern 8: SARGable Date Range

### Before
```sql
WHERE TRUNC(order_date) = TRUNC(SYSDATE)           -- today
WHERE TRUNC(order_date) BETWEEN :start AND :end     -- range
WHERE TO_CHAR(order_date, 'YYYY-MM') = '2024-01'   -- month
WHERE EXTRACT(YEAR FROM order_date) = 2024          -- year
```

### After (all index-friendly)
```sql
-- Today:
WHERE order_date >= TRUNC(SYSDATE)
  AND order_date <  TRUNC(SYSDATE) + 1

-- Date range:
WHERE order_date >= TRUNC(:start)
  AND order_date <  TRUNC(:end) + 1

-- Month of January 2024:
WHERE order_date >= DATE '2024-01-01'
  AND order_date <  DATE '2024-02-01'

-- Year 2024:
WHERE order_date >= DATE '2024-01-01'
  AND order_date <  DATE '2025-01-01'
```

---

## Pattern 9: Fan-Out Join Fix (Multi-Child Aggregation)

### Before (wrong aggregation due to fan-out)
```sql
SELECT o.order_id,
       SUM(i.unit_price * i.quantity)  AS item_total,  -- WRONG if multiple events per order
       COUNT(e.event_id)               AS event_count
FROM orders o
JOIN order_items i  ON i.order_id = o.order_id    -- 1:many
JOIN order_events e ON e.order_id = o.order_id    -- 1:many (second child)
GROUP BY o.order_id;
-- item rows multiplied by event rows per order — SUM is wrong!
```

### After (aggregate each child independently)
```sql
SELECT
    o.order_id,
    COALESCE(items.item_total, 0)    AS item_total,
    COALESCE(events.event_count, 0)  AS event_count
FROM orders o
LEFT JOIN (
    SELECT order_id, SUM(unit_price * quantity) AS item_total
    FROM order_items
    GROUP BY order_id
) items  ON items.order_id = o.order_id
LEFT JOIN (
    SELECT order_id, COUNT(*) AS event_count
    FROM order_events
    GROUP BY order_id
) events ON events.order_id = o.order_id;
```

---

## Pattern 10: Moving Filters Into CTEs / Subqueries Early

### Before (filter at outer level, full join computed first)
```sql
WITH all_orders AS (
    SELECT o.*, c.customer_name, c.region
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
)
SELECT * FROM all_orders
WHERE region = 'WEST' AND status = 'OPEN';
```

### After (push filter into CTE to reduce join volume)
```sql
WITH open_west_orders AS (
    SELECT o.*, c.customer_name
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
                     AND c.region = 'WEST'    -- join condition
    WHERE o.status = 'OPEN'                   -- filters orders early
)
SELECT * FROM open_west_orders;
```
Filtering inside the CTE reduces the rows that the join must process.

---

## Pattern 11: CASE WHEN for Conditional Aggregation (Pivot)

### Before (multiple passes)
```sql
SELECT 'OPEN'   AS status, COUNT(*) FROM orders WHERE status = 'OPEN'
UNION ALL
SELECT 'CLOSED' AS status, COUNT(*) FROM orders WHERE status = 'CLOSED'
UNION ALL
SELECT 'ALL'    AS status, COUNT(*) FROM orders;
-- 3 full scans of orders
```

### After (single pass)
```sql
SELECT
    COUNT(*)                                         AS total,
    COUNT(CASE WHEN status = 'OPEN'   THEN 1 END)   AS open_count,
    COUNT(CASE WHEN status = 'CLOSED' THEN 1 END)   AS closed_count,
    SUM(CASE WHEN status = 'OPEN'   THEN amount END) AS open_amount,
    SUM(CASE WHEN status = 'CLOSED' THEN amount END) AS closed_amount
FROM orders;
-- 1 scan only
```

---

## Pattern 12: Bind Variables vs Literals

### Before (hard parse every execution)
```sql
SELECT * FROM orders WHERE customer_id = 12345;
SELECT * FROM orders WHERE customer_id = 67890;
-- Each is a different SQL → separate hard parse, separate cursor
```

### After (soft parse, cursor reuse)
```sql
SELECT * FROM orders WHERE customer_id = :customer_id;
-- One cursor, parameterized — hard parsed once, soft parsed for every execution
```

**In application code:**
```java
// Bad: string concatenation = literals = hard parse storm
String sql = "SELECT * FROM orders WHERE customer_id = " + customerId;

// Good: bind variable
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM orders WHERE customer_id = ?");
ps.setInt(1, customerId);
```
