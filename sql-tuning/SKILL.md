---
name: sql-tuning
description: >
  Use this skill whenever a user pastes SQL text and wants it reviewed, tuned,
  or analyzed for performance problems. Triggers include: any mention of 'tune this
  SQL', 'review this query', 'what's wrong with this SQL', 'optimize this query',
  'SQL performance', 'slow query', 'rewrite this SQL', 'is this SQL efficient',
  'check my joins', 'why is this query slow', or when the user pastes a block of
  SQL (SELECT, INSERT, UPDATE, DELETE, MERGE, WITH) and asks for feedback, review,
  or optimization. Also trigger if the user pastes SQL alongside an execution plan
  or EXPLAIN output and asks for help. Works with Oracle SQL, but principles apply
  to any RDBMS (Postgres, MySQL, SQL Server). Always use this skill — SQL review
  requires a systematic scan across multiple problem categories to avoid missing
  issues that look innocent in isolation.
---

# SQL Tuning Skill

## What this skill does

Analyzes plain SQL text — no file needed, just the query — to:
1. Parse and understand the query structure (tables, joins, filters, aggregations)
2. Scan for construction problems: bad joins, missing predicates, implicit conversions
3. Identify anti-patterns: SELECT *, functions on indexed columns, correlated subqueries
4. Flag optimizer traps: literal values, bind variable issues, statistics dependencies
5. Optionally analyze a provided execution plan alongside the SQL
6. Rewrite problematic sections with concrete improved SQL
7. Produce a downloadable Word report with annotated findings

---

## Step 1: Accept and Parse the Input

The user will paste SQL inline. Accept either:
- **Raw SQL text** — analyze statically
- **SQL + execution plan** — analyze both together (richer diagnosis)
- **SQL + table/index descriptions** — user may describe schema; use that context

If the user has not provided schema context (column data types, indexes, table sizes),
note where assumptions are made and flag them with: *"Assumption: if col is indexed..."*

Parse the SQL mentally into its structural components:
```
Query type:     SELECT / DML / DDL / WITH (CTE)
Tables:         list all tables/views referenced
Join types:     INNER / LEFT / RIGHT / CROSS / implicit comma
Join columns:   what columns connect each pair of tables
Filter columns: all columns in WHERE clause
Projections:    columns in SELECT list
Aggregations:   GROUP BY, HAVING, window functions
Subqueries:     inline views, EXISTS, IN, scalar subqueries
Hints:          any /*+ ... */ optimizer hints present
```

---

## Step 2: Structural Validity Check

Before performance analysis, verify the SQL is logically correct.

### Join completeness
Every pair of tables in the FROM clause must have a join condition. Flag:
- **Missing join predicate** → Cartesian product (most dangerous SQL bug)
  ```sql
  -- BUG: orders and customers have no join condition
  SELECT * FROM orders o, customers c, products p
  WHERE p.product_id = o.product_id
  -- customers will multiply every row × all customers
  ```
- **Comma-syntax joins** (old style) with WHERE clause joins mixed → easy to miss a predicate
  ```sql
  -- Risky: mix of comma and explicit JOIN makes missing predicates hard to see
  SELECT * FROM a, b JOIN c ON c.id = b.id WHERE a.id = b.id
  -- Rewrite as explicit JOIN chain for clarity and safety
  ```

### Filter sanity
- WHERE clause references correct table aliases
- HAVING clause used for aggregate filters, not WHERE (or vice versa)
  ```sql
  -- Wrong: filter on aggregate in WHERE
  WHERE SUM(amount) > 1000  -- illegal
  -- Correct:
  HAVING SUM(amount) > 1000
  ```
- Date/number comparisons use correct data types (string compared to DATE → no index use)

### Subquery correlation
- Correlated subquery references outer table correctly
- NOT IN subquery — does the subquery column have NOT NULL constraint? If nullable, result may be empty set.

---

## Step 3: SELECT Clause Analysis

### SELECT * anti-pattern
```sql
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id
```
**Problems:**
- Fetches all columns from all joined tables — far more data than needed
- Prevents covering index usage (optimizer can't use index-only scan)
- Breaks when table schema changes (new columns appear unexpectedly)
- In ORM/application code, often causes lazy-load storms on mapped entities

**Fix:** List only the columns the application actually uses.

### Duplicate column names
After a join, `SELECT *` may return two columns both named `id`, `name`, etc.
Flag if the query uses `SELECT *` across a join — the column naming will be ambiguous.

### Scalar subquery in SELECT (correlated)
```sql
SELECT o.order_id,
       (SELECT COUNT(*) FROM order_items i WHERE i.order_id = o.order_id) AS item_count
FROM orders o
```
**Problem:** The subquery executes once per row in `orders`. If orders has 1M rows, the subquery runs 1M times.
**Fix:** Replace with a JOIN + aggregation:
```sql
SELECT o.order_id, COUNT(i.item_id) AS item_count
FROM orders o
LEFT JOIN order_items i ON i.order_id = o.order_id
GROUP BY o.order_id
```

---

## Step 4: FROM / JOIN Clause Analysis

### Implicit cross join (Cartesian product)
```sql
-- Dangerous: no join condition between orders and regions
SELECT o.*, r.region_name
FROM orders o, customers c, regions r
WHERE c.customer_id = o.customer_id
-- Missing: r join condition → every order × every region
```
**Flag immediately as Critical.**

### Outer join to filtering WHERE clause (nullifies the outer join)
```sql
SELECT o.*, c.customer_name
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id
WHERE c.status = 'ACTIVE'   -- ← this turns LEFT JOIN into INNER JOIN
```
**Problem:** The WHERE filter on the outer-joined table eliminates NULLs, making the LEFT JOIN behave like an INNER JOIN — probably not intended.
**Fix:** Move the filter into the JOIN condition:
```sql
LEFT JOIN customers c ON c.customer_id = o.customer_id AND c.status = 'ACTIVE'
```
Or acknowledge it's intentional and use INNER JOIN explicitly.

### Non-SARGable join conditions
```sql
JOIN customers c ON TO_CHAR(c.customer_id) = o.customer_ref
```
**Problem:** Function on join column prevents index use on `customer_id`. Every join probe scans the table.
**Fix:** Fix the data type mismatch at the source, or store `customer_ref` as the same type as `customer_id`.

### Joining on low-cardinality columns without additional predicates
```sql
JOIN order_status s ON s.status_code = o.status
```
If `status_code` has only 3 distinct values, the join will return many rows per order.
Flag if no additional filter narrows the result — may be intentional or may indicate a missing predicate.

### View joining to view
Note if the SQL joins complex views to other views — the optimizer must unnest both, potentially creating a very complex plan. Suggest checking whether the views can be replaced with direct table joins.

---

## Step 5: WHERE Clause Analysis (Critical Section)

This is where most performance problems originate. Scan every predicate.

### 5A — Non-SARGable predicates (index killers)

SARGable = Search ARGument-able = can use an index range scan.

**Function on indexed column:**
```sql
WHERE UPPER(customer_name) = 'SMITH'        -- index on customer_name not used
WHERE TRUNC(order_date) = TRUNC(SYSDATE)    -- index on order_date not used
WHERE TO_NUMBER(order_ref) = 12345          -- index on order_ref not used
WHERE SUBSTR(account_no, 1, 4) = 'ACC1'    -- index on account_no not used
```
**Fix per case:**
```sql
WHERE customer_name = 'Smith'               -- normalize data, or:
-- function-based index: CREATE INDEX ON customers (UPPER(customer_name))
WHERE order_date >= TRUNC(SYSDATE) AND order_date < TRUNC(SYSDATE) + 1
WHERE order_ref = '12345'                   -- fix type mismatch at source
WHERE account_no LIKE 'ACC1%'              -- leading literal = SARGable
```

**Arithmetic on indexed column:**
```sql
WHERE salary + bonus > 50000               -- index on salary not used
WHERE order_date - 30 > :start_date        -- index on order_date not used
```
**Fix:** Move arithmetic to the other side:
```sql
WHERE salary > 50000 - bonus
WHERE order_date > :start_date + 30
```

**Implicit type conversion:**
```sql
WHERE employee_id = '12345'    -- employee_id is NUMBER; string forces conversion
WHERE order_date = '2024-01-15' -- date column compared to string literal
```
The column gets implicitly converted, preventing index use.
**Fix:** Use correct literal types: `WHERE employee_id = 12345`, `WHERE order_date = DATE '2024-01-15'`

**Negation predicates:**
```sql
WHERE status != 'CLOSED'
WHERE status NOT IN ('CLOSED', 'CANCELLED')
WHERE NOT EXISTS (...)
```
Indexes rarely help with negation — optimizer usually does a full scan.
**Fix where possible:** Rewrite as positive range (`WHERE status IN ('OPEN','PENDING')` if the domain is known and small).

**Leading wildcard LIKE:**
```sql
WHERE description LIKE '%keyword%'    -- full scan — no index help
WHERE product_code LIKE '%ABC'        -- same
```
**Fix:** Oracle Text index (`CONTAINS(description, 'keyword') > 0`), or application-level search. If prefix is known, `LIKE 'ABC%'` is SARGable.

**OR across different columns:**
```sql
WHERE status = 'OPEN' OR region = 'WEST'
```
Prevents single-index use. Fix: `UNION ALL` of two indexed queries.

### 5B — Missing predicates (returning too much data)

Look for queries that join large tables without filtering either side:
```sql
SELECT * FROM orders o JOIN order_items i ON i.order_id = o.order_id
-- No WHERE clause — returns entire orders table joined to all items
```
Flag any query touching tables > ~10K rows with no filter predicate on that table.

### 5C — Redundant predicates
```sql
WHERE status IN ('OPEN', 'OPEN', 'PENDING')  -- duplicate
WHERE amount > 0 AND amount > 100            -- first predicate redundant
WHERE 1=1 AND customer_id = :b1             -- 1=1 harmless but noise
```

### 5D — NULL handling issues
```sql
WHERE column != 'VALUE'         -- NULLs are NOT returned (NULL != anything = NULL)
WHERE column NOT IN (1, 2, NULL) -- Returns no rows! NULL in NOT IN list = empty result
```
**Fix:** `WHERE (column != 'VALUE' OR column IS NULL)` and never include NULL in NOT IN lists.

### 5E — Date range patterns
```sql
-- Avoid: BETWEEN on timestamps can miss/include boundary rows ambiguously
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'

-- Better: explicit half-open range, works correctly with timestamps
WHERE order_date >= DATE '2024-01-01' AND order_date < DATE '2024-02-01'
```

---

## Step 6: Subquery Analysis

### IN vs EXISTS
```sql
-- Potentially slow if subquery returns large result set:
WHERE customer_id IN (SELECT customer_id FROM vip_customers WHERE tier = 'GOLD')

-- Better for large subqueries — stops at first match:
WHERE EXISTS (SELECT 1 FROM vip_customers v WHERE v.customer_id = o.customer_id AND v.tier = 'GOLD')
```
Use `EXISTS` when the subquery is large or correlated; use `IN` when the subquery is small and uncorrelated.

### NOT IN with nullable column
```sql
WHERE product_id NOT IN (SELECT product_id FROM discontinued_products)
-- If discontinued_products.product_id has ANY NULL rows → returns ZERO rows
```
**Always use NOT EXISTS instead:**
```sql
WHERE NOT EXISTS (SELECT 1 FROM discontinued_products d WHERE d.product_id = p.product_id)
```

### Correlated subquery in WHERE (executed per outer row)
```sql
WHERE (SELECT MAX(order_date) FROM orders o WHERE o.customer_id = c.customer_id) > SYSDATE - 90
```
Runs once per customer — if 1M customers, runs 1M times.
**Fix:** Rewrite as JOIN with pre-aggregated subquery:
```sql
JOIN (SELECT customer_id, MAX(order_date) AS last_order FROM orders GROUP BY customer_id) lo
  ON lo.customer_id = c.customer_id AND lo.last_order > SYSDATE - 90
```

### Unnecessary DISTINCT caused by bad join
```sql
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c JOIN orders o ON o.customer_id = c.customer_id
```
DISTINCT here is used to de-duplicate rows caused by the one-to-many join.
**Better approach:** use EXISTS to check if orders exist without joining and multiplying:
```sql
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id)
```

### Scalar subquery vs JOIN
```sql
-- Slow: runs subquery per row
SELECT p.product_name,
       (SELECT category_name FROM categories WHERE category_id = p.category_id) AS cat
FROM products p

-- Fast: single JOIN
SELECT p.product_name, cat.category_name
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
```

---

## Step 7: GROUP BY / HAVING / Aggregation Analysis

### Grouping by non-key columns (over-aggregation)
```sql
GROUP BY customer_id, customer_name, customer_email, customer_phone, customer_address
```
If `customer_id` is the PK, all other columns are functionally dependent — grouping by them is unnecessary overhead.
**Fix:** `GROUP BY customer_id` only, join customer details after aggregation.

### HAVING instead of WHERE for non-aggregate filters
```sql
-- Slow: filter applied AFTER grouping millions of rows
GROUP BY customer_id
HAVING customer_id IN (SELECT customer_id FROM vip_customers)

-- Fast: filter BEFORE grouping
WHERE customer_id IN (SELECT customer_id FROM vip_customers)
GROUP BY customer_id
```

### Aggregating before joining (reduce rows first)
```sql
-- Slow: join first, then aggregate
SELECT c.customer_name, SUM(o.amount)
FROM customers c JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_name

-- Better: aggregate orders first, then join to customer for name
SELECT c.customer_name, agg.total
FROM customers c
JOIN (SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id) agg
  ON agg.customer_id = c.customer_id
```
Especially useful when the join has a large orders table.

### COUNT(*) vs COUNT(column)
```sql
COUNT(*)           -- counts all rows including NULLs — usually what you want
COUNT(column)      -- counts non-NULL values of column — use when NULLs matter
COUNT(DISTINCT column) -- expensive on large datasets; needs sort or hash
```
Flag `COUNT(DISTINCT)` on large tables — recommend ensuring it's necessary.

---

## Step 8: CTE / WITH Clause Analysis

```sql
WITH monthly_totals AS (
    SELECT customer_id, TRUNC(order_date, 'MM') AS month, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id, TRUNC(order_date, 'MM')
),
ranked AS (
    SELECT *, RANK() OVER (PARTITION BY customer_id ORDER BY total DESC) AS rnk
    FROM monthly_totals
)
SELECT * FROM ranked WHERE rnk = 1
```

Check:
- **CTE referenced multiple times** → may be materialized (computed once and stored) or inlined (computed multiple times). In Oracle, use `/*+ MATERIALIZE */` or `/*+ INLINE */` hints to control.
- **CTE doing aggregation before a join** → good pattern (reduces rows early)
- **CTE that could be a simple subquery** → extra complexity for no benefit
- **Recursive CTE** → flag for potential infinite loop if termination condition is wrong

---

## Step 9: ORDER BY / PAGINATION Analysis

### ORDER BY on non-indexed column
```sql
SELECT * FROM orders WHERE customer_id = :b1 ORDER BY order_date DESC
```
If there's no index on `(customer_id, order_date)`, Oracle must fetch all matching rows then sort.
**Fix:** Composite index `(customer_id, order_date DESC)` eliminates the sort.

### Pagination anti-patterns
```sql
-- Oracle old-style (wrong — wrong for ties, poor performance)
SELECT * FROM (SELECT o.*, ROWNUM rn FROM orders o WHERE ROWNUM <= 100) WHERE rn > 90

-- Oracle correct OFFSET/FETCH (12c+)
SELECT * FROM orders ORDER BY order_id OFFSET 90 ROWS FETCH NEXT 10 ROWS ONLY

-- Deep pagination problem (any DB): OFFSET 900000 scans 900,010 rows to return 10
-- Fix: keyset pagination
WHERE order_id > :last_seen_id ORDER BY order_id FETCH FIRST 10 ROWS ONLY
```

### ORDER BY in subquery / CTE (usually useless)
```sql
-- The ORDER BY here is meaningless — optimizer may ignore it
WITH data AS (SELECT * FROM orders ORDER BY order_date)
SELECT * FROM data WHERE customer_id = :b1
```
ORDER BY only has guaranteed meaning at the outermost SELECT.

---

## Step 10: DML-Specific Analysis (INSERT / UPDATE / DELETE / MERGE)

### UPDATE / DELETE without WHERE clause
```sql
UPDATE orders SET status = 'PROCESSED'   -- updates EVERY row
DELETE FROM temp_staging                  -- deletes EVERY row (use TRUNCATE instead)
```
**Flag as Critical if no WHERE clause on a DML statement.**

### Row-by-row DML in what should be set-based
```sql
-- Anti-pattern visible in SQL: updating with scalar subquery per row
UPDATE orders o SET status = (SELECT new_status FROM status_map m WHERE m.order_id = o.order_id)
-- Better: MERGE or UPDATE with JOIN
MERGE INTO orders o
USING status_map m ON m.order_id = o.order_id
WHEN MATCHED THEN UPDATE SET o.status = m.new_status
```

### MERGE without DELETE clause when full sync needed
If user says they want to sync a table, check if the MERGE handles all three cases (insert, update, delete for rows no longer in source).

### INSERT with SELECT * from large table without filter
```sql
INSERT INTO archive_orders SELECT * FROM orders   -- copies entire table
```
Flag if no WHERE clause — likely unintentional or needs pagination.

### Implicit commit risk in DML
Note if DML does not appear to be wrapped in a transaction boundary — for Oracle, remind that DDL auto-commits but DML does not require explicit commit.

---

## Step 11: Execution Plan Analysis (Optional — if provided)

If the user also provides an execution plan (from EXPLAIN PLAN, SQL*Plus AUTOTRACE, or SQL Monitor), cross-reference each SQL finding with the plan:

1. Map each suspicious WHERE predicate → is it showing as ACCESS or FILTER in the plan?
2. Map each join → what join method is chosen? Does it match expected?
3. Check E-Rows vs A-Rows for each table access → is the optimizer's estimate accurate?
4. Check for TABLE ACCESS FULL on large tables correlated with missing SARGable predicate found above
5. Cross-reference with `sql-monitor-analysis` skill for deeper plan analysis if needed

For full execution plan analysis, invoke the `sql-monitor-analysis` skill.

---

## Step 12: Index Recommendations

Based on the WHERE clause and JOIN analysis, suggest specific indexes:

For each recommendation:
```sql
-- Finding: full scan on ORDERS due to filter on STATUS (non-leading column)
-- Current filter: WHERE o.status = 'OPEN' AND o.customer_id = :b1
-- Recommendation:
CREATE INDEX idx_orders_status_customer
    ON orders (status, customer_id, order_date)
    TABLESPACE INDX;
-- Rationale: equality on status first (low cardinality but filters well combined),
--            customer_id narrows further, order_date covers ORDER BY → no sort needed

-- Verify after creation:
SELECT index_name, status FROM user_indexes WHERE table_name = 'ORDERS';
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDERS', cascade => TRUE);
```

Also suggest statistics refresh if:
- Query has very selective predicates that the optimizer might misestimate
- Table has been recently loaded with large amounts of data
- Columns in WHERE have skewed distributions (e.g., 90% of orders have status='CLOSED')

---

## Step 13: Rewritten SQL

For every significant finding, provide a corrected version of that section.

For major rewrites, provide the full corrected query with inline comments:

```sql
-- REWRITTEN: replaced scalar subquery with JOIN, removed SELECT *,
--            fixed implicit type conversion, added SARGable date range

SELECT
    o.order_id,
    o.order_date,
    o.status,
    o.total_amount,
    c.customer_name,
    c.email,
    cat.category_name,
    item_totals.line_count        -- was: scalar subquery per row
FROM orders o
JOIN customers c
    ON c.customer_id = o.customer_id  -- was: implicit conversion TO_CHAR(c.id)
JOIN categories cat
    ON cat.category_id = o.category_id
JOIN (
    SELECT order_id, COUNT(*) AS line_count
    FROM order_items
    GROUP BY order_id
) item_totals
    ON item_totals.order_id = o.order_id
WHERE o.order_date >= DATE '2024-01-01'    -- was: TRUNC(order_date) = ... (non-SARGable)
  AND o.order_date <  DATE '2024-02-01'
  AND o.status IN ('OPEN', 'PENDING')      -- was: status != 'CLOSED' (negation)
  AND c.status = 'ACTIVE'                  -- moved from HAVING to WHERE
ORDER BY o.order_date DESC
FETCH FIRST 100 ROWS ONLY;                 -- added explicit limit
```

---

## Step 14: Findings and Inline Summary

### Severity guide:
- 🔴 **Critical**: Cartesian join, DML without WHERE, NOT IN with nullable column (returns 0 rows), query guaranteed to return wrong results
- 🟠 **High**: Non-SARGable predicate on high-cardinality column, correlated subquery in SELECT/WHERE, SELECT * on large join, implicit type conversion
- 🟡 **Medium**: Scalar subquery replaceable with JOIN, missing covering index, DISTINCT hiding a join problem, GROUP BY over-specification
- 🟢 **Low**: Style/readability issue, minor optimization opportunity, best practice gap

### Output format:

```
## SQL Tuning Analysis

**Query type:** SELECT with 3 tables, 2 CTEs, 1 subquery
**Lines:** N | **Complexity:** Low / Medium / High / Very High

### Overall Assessment: [Correct but inefficient / Has bugs / Critical issues]
<2–3 sentence summary>

### Findings
| # | Severity | Category | Location | Issue | Fix |
|---|---|---|---|---|---|
| 1 | 🔴 Critical | Join | FROM clause | Missing join predicate on REGIONS → Cartesian product | Add join condition |
| 2 | 🟠 High | WHERE | Line 8 | UPPER(customer_name) prevents index use | Function-based index or normalize data |
| 3 | 🟠 High | Subquery | SELECT list | Scalar subquery runs per row | Rewrite as LEFT JOIN + GROUP BY |
| 4 | 🟡 Medium | SELECT | Line 1 | SELECT * on 3-table join | List specific columns |
| 5 | 🟢 Low | ORDER BY | Line 15 | No index covers ORDER BY column | Composite index recommendation |

### Index Recommendations
<DDL for each recommended index>

### Rewritten Query
<full corrected SQL>
```

---

## Step 15: Generate Word Report

Read `/mnt/skills/public/docx/SKILL.md`.

Report sections:
1. Cover — query summary, overall verdict
2. Original SQL (formatted, with line numbers)
3. Findings table — all issues severity-ranked
4. Finding detail — one section per finding with evidence + fix
5. Index Recommendations — DDL ready to execute
6. Rewritten SQL — full corrected query with inline comments
7. Statistics recommendations

```bash
node /home/claude/sql_tuning_report.js
cp /home/claude/sql_tuning_report.docx /mnt/user-data/outputs/sql_tuning_report.docx
```

---

## Reference files

- `references/antipatterns.md` — Comprehensive SQL anti-pattern catalog with examples and fixes across Oracle, Postgres, MySQL, SQL Server
- `references/rewrite-patterns.md` — Common query rewrite patterns: scalar subquery → JOIN, IN → EXISTS, row-by-row → set-based, pagination fixes

---

## Key principles

- **A Cartesian join is always a bug** — never leave it unflagged, even if the result "looks right" at small data volumes
- **Non-SARGable predicates are the most common performance killer** — every function in a WHERE clause is a red flag
- **NOT IN with nullable columns returns zero rows** — this is a correctness bug, not just a performance issue
- **DISTINCT is often a symptom** — when you see DISTINCT on a join result, ask what duplication it's hiding
- **Rewrite, don't just flag** — always provide the corrected SQL, not just a description of the problem
- **Context matters** — a full table scan on a 100-row table is fine; on a 100M-row table it's critical
- **If schema is unknown, state assumptions** — "assuming ORDER_DATE is indexed" makes the recommendation actionable once the user confirms
