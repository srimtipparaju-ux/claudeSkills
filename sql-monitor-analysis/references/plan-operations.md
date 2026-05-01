# Oracle Execution Plan Operations Reference

## Table of Contents
1. [Table Access Operations](#table-access)
2. [Index Operations](#index-operations)
3. [Join Operations](#join-operations)
4. [Sort & Aggregation](#sort-aggregation)
5. [Partition Operations](#partition-operations)
6. [Parallel Execution](#parallel-execution)
7. [Subquery Operations](#subquery-operations)
8. [Miscellaneous](#miscellaneous)

---

## Table Access Operations {#table-access}

### TABLE ACCESS FULL
- **What:** Reads every block in the table (full table scan)
- **When Oracle chooses it:** Returning > ~10-15% of rows, no usable index, small table, stale stats make selectivity appear low
- **Diagnose:** Check A-Rows vs table size. Is it returning most rows (intentional) or a tiny fraction (missing index)?
- **Fix:** Add index on filter column(s), fix statistics, use partition pruning

### TABLE ACCESS BY INDEX ROWID
- **What:** Fetches a single row from the table using a ROWID from an index lookup
- **When:** After an index probe, Oracle needs columns not in the index
- **Diagnose:** Check Buffers vs A-Rows. If Buffers >> A-Rows, the index isn't selective enough (many rows per block, or index fragmentation)
- **Fix:** Make index covering (add needed columns), or accept if ratio is reasonable

### TABLE ACCESS BY INDEX ROWID BATCHED
- **What:** Same as above but batches multiple ROWIDs before fetching — more efficient for bulk lookups
- **When:** Optimizer prefers this for slightly larger result sets than single ROWID
- **Note:** Generally better than non-batched; if this is slow, same diagnosis as ROWID above

### TABLE ACCESS INMEMORY FULL
- **What:** In-Memory Column Store scan — like FTS but from memory
- **When:** Table is populated in IM column store
- **Diagnose:** If slow despite IM, check if fully populated (`V$IM_SEGMENTS`), check column store filters

### TABLE ACCESS SAMPLE
- **What:** Reads a random sample of rows (from SAMPLE clause)
- **When:** Query uses SAMPLE(n) — intentional
- **Note:** Not a performance issue; row counts will be unpredictable

---

## Index Operations {#index-operations}

### INDEX UNIQUE SCAN
- **What:** Traverses B-tree to find exactly one entry (unique or primary key lookup)
- **When:** Equality predicate on unique/PK index
- **Performance:** Very fast — O(log n). Rarely a problem unless index is deeply unbalanced.
- **Fix needed:** Almost never; if slow, check index fragmentation (`ANALYZE INDEX ... VALIDATE STRUCTURE`)

### INDEX RANGE SCAN
- **What:** Traverses B-tree and scans a range of leaf blocks
- **When:** Range predicate (BETWEEN, >, <, LIKE 'val%') or equality on non-unique index
- **Diagnose:** Check A-Rows — if large, the range is wide and may not be selective. Check Buffers vs A-Rows.
- **Fix:** Composite index with more selective leading column, partition pruning

### INDEX RANGE SCAN DESCENDING
- **What:** Like range scan but traverses in reverse order
- **When:** ORDER BY DESC on indexed column, or MAX() optimization
- **Note:** Usually fine; can be forced with DESC index or DESCENDING hint

### INDEX SKIP SCAN
- **What:** Scans index by skipping the leading column (used when query has predicate on non-leading column)
- **When:** Leading column has very few distinct values; optimizer finds it cheaper than FTS
- **Diagnose:** Often indicates the index isn't well-suited. E-Rows vs A-Rows often off.
- **Fix:** Create a new index with the queried column as leading key

### INDEX FULL SCAN
- **What:** Reads all leaf blocks of the index in order (like FTS on index)
- **When:** Optimizer can satisfy ORDER BY or MIN/MAX from index order without sorting; or for small index driving a sort-merge join
- **Note:** Does single-block reads (sequential) vs INDEX FAST FULL SCAN. Can be slower for large indexes.

### INDEX FAST FULL SCAN
- **What:** Reads all index blocks using multiblock reads (like a table FTS but on the index)
- **When:** Query only needs columns in the index (covering), used for COUNT(*), aggregates
- **Note:** Generally efficient. If slow, index may be large and benefit from more selective query.

### INDEX JOIN
- **What:** Joins two indexes on the same table to retrieve needed columns without hitting the table
- **When:** Two partial indexes together cover all needed columns
- **Diagnose:** Rare. Can be slow if the hash join of the two indexes is expensive. Adding a composite index usually better.

### BITMAP INDEX operations (BITMAP AND/OR/MERGE/CONVERSION)
- **What:** Bitmap indexes used for low-cardinality columns; combined with AND/OR logic
- **When:** Data warehouses, OLAP; rarely in OLTP
- **Diagnose:** BITMAP CONVERSION TO ROWIDS can be slow for large result sets
- **Note:** Never use bitmap indexes on tables with concurrent DML (lock contention)

---

## Join Operations {#join-operations}

### NESTED LOOPS
- **What:** For each row in the outer (driving) input, probe the inner input once
- **Best for:** Small outer row count (< ~1,000), indexed inner table
- **True cost:** `outer_rows × inner_cost` — always multiply! Check `Starts` on inner child.
- **Red flag:** Starts on inner child > 10,000 — means 10K+ probes into inner side
- **Fix:** If outer is large, this join method is wrong → fix stats (optimizer chose NL because it underestimated outer rows), or hint USE_HASH

### NESTED LOOPS (with BATCHING)
- **What:** NL variant that batches ROWIDs before table access — more I/O efficient
- **Note:** Better than standard NL for slightly larger result sets; same diagnostic approach

### HASH JOIN
- **What:** Builds a hash table from the smaller input, probes with the larger input
- **Best for:** Large inputs where neither side is tiny (> ~1,000 rows on at least one side)
- **Build side:** Left child (first listed) — this becomes the hash table
- **Diagnose:** If build side is large → large hash table → possible PGA spill (check Temp Space). If probe side has many Buffers but few rows returned → probe side doing many lookups with poor selectivity.
- **Fix for spill:** Increase PGA_AGGREGATE_TARGET, or make build side smaller (fix upstream filter)

### HASH JOIN RIGHT OUTER / FULL OUTER
- **What:** Outer join variant of hash join
- **Note:** "RIGHT" means right input is the optional side. Extra overhead for outer join logic.

### HASH JOIN ANTI / SEMI
- **What:** Used for NOT IN / NOT EXISTS (ANTI) or EXISTS / IN (SEMI) subqueries
- **Note:** Usually efficient. ANTI join with NULL handling can behave unexpectedly — ensure NOT IN subquery columns are NOT NULL.

### MERGE JOIN
- **What:** Sorts both inputs then merges them in order
- **Best for:** Both inputs are already sorted (from index), or ORDER BY required anyway
- **Diagnose:** If SORT operations feed a MERGE JOIN, check for spill (Temp Space). If both inputs are pre-sorted, very efficient.
- **Fix for spill:** Indexes to pre-sort, or increase PGA

### MERGE JOIN CARTESIAN
- **What:** Cross join — every row from left matched with every row from right
- **Row count:** left_rows × right_rows — can explode quickly
- **Almost always a bug:** Check SQL for missing join predicate between these two tables
- **Intentional use:** Cross join for generating combinations — should be explicit in SQL
- **Fix:** Add the missing join condition, or rewrite query logic

### MERGE JOIN OUTER
- **What:** Outer join via sort-merge — less common than hash join outer
- **Note:** Same spill concerns as regular merge join

---

## Sort & Aggregation {#sort-aggregation}

### SORT ORDER BY
- **What:** Sorts rows to satisfy ORDER BY clause
- **Diagnose:** Check Temp Space — spill = slow. Check A-Rows going in — large sort = more memory needed.
- **Fix:** Index that satisfies ORDER BY (eliminates sort), increase PGA, fix upstream to reduce rows entering sort

### SORT GROUP BY
- **What:** Sorts data then applies GROUP BY aggregation (requires full sort before aggregation begins)
- **Note:** Alternative to HASH GROUP BY — used when order can be exploited or for certain aggregations
- **Fix for spill:** Same as SORT ORDER BY

### SORT JOIN
- **What:** Sort phase feeding a MERGE JOIN — see Merge Join above
- **Note:** Check Temp Space

### SORT UNIQUE
- **What:** Sorts and removes duplicates (used for DISTINCT, UNION, some subquery unnesting)
- **Diagnose:** Check Temp Space; check if DISTINCT is actually needed in the query

### HASH GROUP BY
- **What:** Hash-based aggregation — groups rows into a hash table (does not require full pre-sort)
- **Usually faster than SORT GROUP BY** for large aggregations
- **Diagnose:** Check Temp Space for spill
- **Fix for spill:** Increase PGA, pre-filter rows before grouping

### BUFFER SORT
- **What:** Materializes (caches) a result set in PGA to be re-scanned
- **When:** Inner side of a NL that has no index — Oracle re-executes the scan each outer iteration
- **Diagnose:** If on inner side of NL with large Starts, this is amplifying the inner cost enormously

### COUNT (STOPKEY)
- **What:** Stops fetching after N rows (for ROWNUM <= N or FETCH FIRST N ROWS)
- **Note:** Usually beneficial — early exit. Ensure the limiting is happening at the right level.

### WINDOW (SORT / BUFFER)
- **What:** Analytic function execution (ROW_NUMBER, RANK, LAG, LEAD, etc.)
- **Diagnose:** Check Temp Space on WINDOW SORT. Check if partitioning columns have indexes that could eliminate sort.

---

## Partition Operations {#partition-operations}

### PARTITION RANGE ALL
- **What:** Scanning ALL partitions — partition pruning not happening
- **Red flag:** If query has a predicate on the partition key, this means pruning failed
- **Fix:** Ensure WHERE clause uses the partition key with a literal or bind variable (not a function or column)

### PARTITION RANGE SINGLE
- **What:** Accessing exactly one partition — perfect pruning
- **Good:** This is the ideal outcome for partitioned tables

### PARTITION RANGE ITERATOR
- **What:** Accessing multiple but not all partitions — partial pruning
- **Diagnose:** How many partitions? Check `Pstart` and `Pstop` in plan. Acceptable if limited.

### PARTITION RANGE INLIST
- **What:** Accessing partitions corresponding to an IN list
- **Note:** Efficient for IN lists on partition key

### PARTITION HASH / LIST variants
- **What:** Same as above but for HASH or LIST partitioned tables

---

## Parallel Execution {#parallel-execution}

### PX COORDINATOR
- **What:** Query coordinator that dispatches and collects work from parallel slaves
- **When:** Top of parallel plan
- **Diagnose:** If query coordinator shows high time, check for serialization — slaves finished quickly but coordinator is bottlenecked

### PX SEND / PX RECEIVE
- **What:** Data redistribution between parallel slaves
- **PX SEND BROADCAST:** Small side broadcast to all slaves — efficient for small tables
- **PX SEND HASH:** Rows redistributed by hash value for joins — check for skew
- **PX SEND RANGE:** For range-based redistribution (ORDER BY in parallel)
- **Diagnose:** High time in PX operations = data redistribution bottleneck or slave skew

### PX BLOCK ITERATOR
- **What:** Splits table blocks across parallel slaves
- **Note:** Normal parallel FTS component — check if DOP matches expectation

### PX PARTITION RANGE ALL / ITERATOR
- **What:** Parallel partition scan
- **Note:** Check how many partitions vs DOP — if partitions < DOP, some slaves will be idle

---

## Subquery Operations {#subquery-operations}

### FILTER
- **What:** Applies a subquery as a filter condition — subquery re-executed for qualifying rows
- **Diagnose:** Check Starts on the subquery inside FILTER — if large, subquery runs once per outer row
- **Red flag:** FILTER with large Starts = correlated subquery performance killer
- **Fix:** Rewrite as JOIN (usually HASH JOIN SEMI/ANTI), or use WITH clause + join

### COUNT (subquery related)
- **What:** COUNT check for EXISTS subqueries — stops at first matching row
- **Note:** Usually efficient, but watch Starts if correlated

### HASH JOIN SEMI
- **What:** Efficient EXISTS / IN subquery execution via hash join
- **Good:** Much better than FILTER with correlated subquery

### MERGE JOIN SEMI
- **What:** Sort-merge variant of semi-join
- **Note:** Check for sort spill

### VIEW
- **What:** Inline view or WITH clause being resolved
- **Diagnose:** If VIEW has large Temp or Buffers, the subquery inside is the problem — dive into its children

### TEMP TABLE TRANSFORMATION
- **What:** WITH clause (CTE) materialized to a temp table
- **When:** Optimizer decides materialization is cheaper than repeated inline evaluation
- **Diagnose:** If WITH clause is referenced many times and results are large, materialization can help or hurt — check Temp Space

---

## Miscellaneous {#miscellaneous}

### TABLE QUEUE
- **What:** Used in distributed queries (database links) — fetching from remote database
- **Diagnose:** Network latency, remote DB performance; high row counts over dblink are slow

### REMOTE
- **What:** Operation executed on a remote database
- **Red flag:** Large row counts returned from REMOTE = pulling too much data over the link
- **Fix:** Push filters to remote side with views or `driving_site` hint; minimize rows returned over dblink

### CONNECT BY (WITH FILTERING / NO FILTERING / PUMP)
- **What:** Hierarchical query execution
- **Diagnose:** CONNECT BY can be very expensive on large hierarchies. Check A-Rows, depth of hierarchy.
- **Fix:** Use NOCYCLE if cycles possible, add WHERE clause to limit starting points, consider materialized hierarchy

### COLLECTION ITERATOR PICKLER FETCH
- **What:** Reading from a PL/SQL collection type passed to SQL
- **Note:** Usually fast for small collections; large collections slow

### FAST DUAL
- **What:** SELECT from DUAL — effectively a no-op row source
- **Note:** Zero cost; if appearing with UNION ALL, it's usually a row generator pattern

### INLIST ITERATOR
- **What:** Executes the child operation once per value in an IN list
- **Diagnose:** If IN list is large (100+ values) and child is expensive, this multiplies cost
- **Fix:** Use a temp table or bulk collect instead of large IN list; or index lookup if each value is selective

### BITMAP CONVERSION FROM ROWIDS / TO ROWIDS
- **What:** Converts between ROWID and bitmap representation
- **When:** Mixing bitmap and B-tree index operations
- **Note:** TO ROWIDS step before table access can produce large row counts; check Buffers after conversion

### RESULT CACHE
- **What:** SQL result cache storing query results
- **When:** RESULT_CACHE hint or automatic result caching
- **Note:** If hit (cached), very fast. If miss, normal execution + cache write overhead.
