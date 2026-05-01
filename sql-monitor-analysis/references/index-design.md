# Index Design Reference for SQL Tuning

## Core Principles

1. **An index is only as good as its leading column** — the optimizer can only use an index efficiently if the leading column(s) appear in the query's access predicates.
2. **Composite beats multiple singles** — a single composite index on (A, B, C) is usually better than three separate indexes when a query filters on all three.
3. **Covering indexes eliminate table access** — if the index contains all columns the query needs, Oracle never touches the table (INDEX-only scan).
4. **Stats must be current** — a perfect index won't be used if the optimizer's statistics show it's not selective.

---

## Index Column Ordering Rules

Order columns in a composite index by this priority:

1. **Equality predicates first** (`col = value`) — best selectivity per step
2. **Range predicates last** (`col > value`, `col BETWEEN`, `col LIKE 'x%'`) — can only use one range column per index
3. **Most selective column first** within equality predicates — reduces leaf blocks scanned fastest
4. **ORDER BY / GROUP BY columns** — add at the end to eliminate a sort operation

**Example:**
```sql
-- Query: WHERE status = 'OPEN' AND region = 'WEST' AND created_date > SYSDATE-30 ORDER BY created_date
-- Good index:
CREATE INDEX idx ON orders (status, region, created_date);
--            equality     equality  range+sort
```

---

## Common Query Patterns → Index Recipes

### Pattern 1: Simple equality lookup
```sql
WHERE customer_id = :b1
```
→ Index on `(customer_id)` — simple, effective
→ If query also needs `order_date`, `status`: add to index as covering columns

---

### Pattern 2: Equality + range
```sql
WHERE customer_id = :b1 AND order_date >= :b2 AND order_date < :b3
```
→ `(customer_id, order_date)` — equality first, range last
→ Covers both access predicates

---

### Pattern 3: Multiple equality columns
```sql
WHERE region = :b1 AND status = :b2 AND category = :b3
```
→ `(region, status, category)` or reorder by selectivity (most selective first)
→ Check column cardinality: high cardinality column should lead

---

### Pattern 4: Range-only (no equality)
```sql
WHERE created_date BETWEEN :b1 AND :b2
```
→ `(created_date)` — range scan
→ Only one range column can be used for access; additional range predicates become filters
→ Consider partitioning by date instead if range spans many partitions

---

### Pattern 5: ORDER BY satisfaction (eliminating sort)
```sql
WHERE customer_id = :b1 ORDER BY order_date DESC
```
→ `(customer_id, order_date DESC)` or `(customer_id, order_date)` with DESCENDING scan
→ Optimizer can avoid SORT ORDER BY if index delivers rows in right order
→ Verify in plan: SORT ORDER BY should disappear; INDEX RANGE SCAN (DESCENDING) should appear

---

### Pattern 6: GROUP BY aggregation
```sql
WHERE region = :b1 GROUP BY status
```
→ `(region, status)` — both columns in index eliminates sort for GROUP BY
→ Add aggregate column too for covering: `(region, status, order_amount)`

---

### Pattern 7: JOIN — inner side of Nested Loop
```sql
-- Outer table drives, inner table looked up per outer row
SELECT * FROM orders o JOIN order_items i ON i.order_id = o.order_id
-- If orders is outer, order_items needs:
```
→ `order_items(order_id)` — the FK column being joined on
→ If additional filter: `(order_id, status)` — equality on join key first, then filter
→ Missing FK index → full scan of inner table per outer row → catastrophic NL performance

---

### Pattern 8: EXISTS / IN subquery
```sql
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id AND o.status = 'OPEN')
```
→ Index on `orders(customer_id, status)` — join key + filter in composite

---

### Pattern 9: Covering index (avoid table access entirely)
```sql
SELECT order_id, order_date, status FROM orders WHERE customer_id = :b1
```
→ `(customer_id, order_id, order_date, status)` — all needed columns in index
→ Plan shows INDEX RANGE SCAN only, no TABLE ACCESS BY INDEX ROWID
→ Eliminates random I/O for each row — huge win if table access is bottleneck

---

### Pattern 10: Function in predicate
```sql
WHERE UPPER(last_name) = :b1
WHERE TRUNC(created_date) = :b2
WHERE TO_CHAR(order_id) = :b3
```
→ Regular index on `last_name` won't be used — function wraps the column
→ Create **function-based index**:
```sql
CREATE INDEX idx_upper_name ON customers (UPPER(last_name));
CREATE INDEX idx_trunc_date ON orders (TRUNC(created_date));
```
→ Or better: fix the query to avoid the function (store data pre-uppercased, use range for dates)

---

### Pattern 11: OR conditions
```sql
WHERE status = 'OPEN' OR region = 'WEST'
```
→ OR across different columns prevents single index use
→ Option A: Two separate indexes + INDEX COMBINE (bitmap OR) — works but less efficient
→ Option B: Rewrite as UNION ALL:
```sql
SELECT ... WHERE status = 'OPEN'
UNION ALL
SELECT ... WHERE region = 'WEST' AND status != 'OPEN'
```
→ Each branch can use its own index

---

### Pattern 12: IN list
```sql
WHERE status IN ('OPEN', 'PENDING', 'PROCESSING')
```
→ Index on `(status)` works — optimizer uses INLIST ITERATOR (one range scan per value)
→ If IN list has 100+ values, consider joining to a temp table instead

---

### Pattern 13: NOT IN / NOT EXISTS
```sql
WHERE customer_id NOT IN (SELECT customer_id FROM blacklist)
```
→ Index on `blacklist(customer_id)` helps the anti-join
→ If blacklist column is nullable: NOT IN returns no rows when subquery has any NULL — use NOT EXISTS instead

---

### Pattern 14: LIKE with prefix
```sql
WHERE last_name LIKE 'SMITH%'   -- leading literal
WHERE last_name LIKE '%MITH'    -- leading wildcard -- index won't help
```
→ Leading literal: range scan on `(last_name)` — works well
→ Leading wildcard: no B-tree index helps → use Oracle Text index (`CONTAINS`) or accept FTS

---

## Index Maintenance Diagnostics

### Check if an index is being used
```sql
-- After running the query, check V$SQL_PLAN
SELECT * FROM V$SQL_PLAN
WHERE sql_id = '<your_sql_id>'
AND object_name = '<index_name>';
```

### Check index selectivity
```sql
-- High ratio = selective = index is valuable
SELECT index_name,
       num_rows,
       distinct_keys,
       ROUND(num_rows / NULLIF(distinct_keys, 0), 1) AS rows_per_key
FROM dba_indexes
WHERE table_name = 'ORDERS'
ORDER BY rows_per_key;
```

### Check for stale statistics
```sql
SELECT table_name, last_analyzed, num_rows, stale_stats
FROM dba_tab_statistics
WHERE table_name IN ('ORDERS', 'CUSTOMERS')
ORDER BY last_analyzed;
```

### Gather statistics (most common fix)
```sql
-- Single table
EXEC DBMS_STATS.GATHER_TABLE_STATS(
  ownname     => 'SCHEMA_NAME',
  tabname     => 'TABLE_NAME',
  cascade     => TRUE,                          -- includes indexes
  method_opt  => 'FOR ALL COLUMNS SIZE AUTO',   -- histograms for skewed data
  degree      => 4                              -- parallel
);

-- Check histograms exist for skewed columns
SELECT column_name, histogram, num_distinct, num_nulls
FROM dba_tab_col_statistics
WHERE table_name = 'ORDERS'
ORDER BY num_distinct DESC;
```

### Extended statistics for correlated columns
```sql
-- When WHERE a=1 AND b=2 has much better selectivity than a alone or b alone
-- Oracle can't know this without extended stats
EXEC DBMS_STATS.CREATE_EXTENDED_STATS('SCHEMA','TABLE','(COL_A, COL_B)');
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA','TABLE',method_opt=>'FOR ALL COLUMNS SIZE AUTO');
```

### Find missing indexes on FK columns (inner side of NL)
```sql
SELECT c.table_name, c.constraint_name, cc.column_name,
       'MISSING INDEX' AS status
FROM dba_constraints c
JOIN dba_cons_columns cc ON c.constraint_name = cc.constraint_name
  AND c.owner = cc.owner
WHERE c.constraint_type = 'R'
  AND c.owner = 'SCHEMA_NAME'
  AND NOT EXISTS (
    SELECT 1 FROM dba_ind_columns ic
    WHERE ic.table_name = c.table_name
      AND ic.owner = c.owner
      AND ic.column_name = cc.column_name
      AND ic.column_position = 1
  )
ORDER BY c.table_name;
```

---

## When NOT to Add an Index

- **Table is tiny** (< 1,000 rows): FTS is often faster due to multi-block reads
- **Column has very low cardinality** (e.g., M/F, Y/N with even distribution): index rarely selective enough; consider bitmap index for DW
- **Table has heavy DML**: every INSERT/UPDATE/DELETE maintains all indexes — too many indexes slow writes
- **Query already returns > 15% of rows**: FTS with multi-block reads often wins; index + random ROWID access is slower
- **Column already has a usable index**: check if an existing composite index with this as leading key already covers the use case

---

## Validating an Index Fixed the Problem

After adding an index, verify in SQL Monitor:

1. TABLE ACCESS FULL → replaced by INDEX RANGE SCAN + TABLE ACCESS BY INDEX ROWID
2. A-Time for that step drops significantly
3. Buffers for that step drops (fewer logical I/Os)
4. E-Rows ≈ A-Rows (if stats were also refreshed)
5. Join method may change (NL → Hash Join or vice versa) as optimizer recalculates costs
