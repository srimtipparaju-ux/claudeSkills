# AWR Common Problems Reference

## Wait Event Quick-Reference

### I/O Wait Events

| Wait Event | Root Cause | Investigation Steps | Recommendations |
|---|---|---|---|
| `db file sequential read` | Single-block I/O: index scans, undo reads | Check top SQL by physical reads; check avg wait time vs storage baseline | Tune SQL plans, cache hot indexes in KEEP pool, upgrade storage, check for missing indexes |
| `db file scattered read` | Multi-block I/O: full table/index scans | Identify SQL doing FTS via V$SQL / AWR Top SQL | Add indexes, enable partition pruning, increase `db_file_multiblock_read_count` |
| `direct path read` | PGA sort/hash spill to TEMP, parallel query | Check PGA advisory, temp usage | Increase PGA_AGGREGATE_TARGET, tune sorts/hash joins, reduce parallel degree |
| `direct path write` | Bulk insert, CTAS, parallel DML | Normal for ETL; check if unexpected | Usually acceptable; ensure TEMP tablespace has room |
| `direct path read temp` | Sort/hash running out of PGA, using temp | Check `one-pass` and `multi-pass` rows in PGA advisory | Increase PGA target, tune queries with sorts/hashes |

### Redo / Commit Waits

| Wait Event | Root Cause | Investigation Steps | Recommendations |
|---|---|---|---|
| `log file sync` | COMMIT waiting for LGWR to flush redo | Check `log file parallel write` avg wait, I/O on redo logs | Move redo logs to faster storage (SSD), reduce commit frequency (batch commits), check for sync writes forced by OS |
| `log file parallel write` | LGWR writing to redo log files | Check redo log device I/O latency | Separate redo logs to dedicated fast disks, consider Oracle Redo Log buffering |
| `log buffer space` | Redo buffer full | Check redo log switches/hr, buffer size | Increase `log_buffer`, add redo log groups/members |

### Concurrency / Locking

| Wait Event | Root Cause | Investigation Steps | Recommendations |
|---|---|---|---|
| `enq: TX - row lock contention` | Row-level locking between sessions | Check `v$lock`, identify blocking session and SQL | Fix application locking order, reduce transaction duration, add retry logic |
| `enq: TM - contention` | DML on table with no FK index | Check for FK constraints without indexes | Add indexes on all FK columns |
| `enq: HW - contention` | High water mark extension on segment | Check object growing rapidly | Pre-allocate extents (`ALLOCATE EXTENT`), use `ASSM` (Automatic Segment Space Management) |
| `enq: ST - contention` | Space management in dictionary-managed tablespace | Check tablespace type | Convert to locally managed tablespace |
| `buffer busy waits` | Multiple sessions competing for same buffer | Check `X$BH` for hot blocks; check segment header | Use ASSM, increase freelists, reduce row chaining, partition hot tables |
| `read by other session` | Many sessions reading same block into cache | Check top SQL doing same full scans | Use result cache, add indexes, reduce concurrent identical queries |

### Parsing / Library Cache

| Wait Event | Root Cause | Investigation Steps | Recommendations |
|---|---|---|---|
| `library cache lock` | DDL being run while others execute the object | Check who holds the lock, look for DDL in trace | Schedule DDL during off-hours, use DBMS_STATS with `no_invalidate` |
| `library cache pin` | Hard parse storm or DDL conflict | Check hard parses/sec in load profile | Use bind variables, set `cursor_sharing=FORCE` as workaround, fix application |
| `cursor: pin S wait on X` | Many sessions hard-parsing same SQL simultaneously | Check hard parses; check literal SQL | Enforce bind variables, review `cursor_sharing` parameter |
| `latch: shared pool` | Shared pool latch contention | Check library hit %, hard parse rate | Use bind variables, increase shared pool, enable AMM |
| `latch: library cache` | Library cache latch contention | Check hard parse rate | Use bind variables, enable cursor sharing |

### Network / Client

| Wait Event | Root Cause | Investigation Steps | Recommendations |
|---|---|---|---|
| `SQL*Net message from client` | Idle time waiting for client to send next call | High this is normal; only a problem if dominating | Reduce round-trips (array fetch, bulk bind), check think time |
| `SQL*Net more data from client` | Client sending large bind data / fetching in small rows | Check array fetch size | Increase `arraysize` / `prefetchrows` in client config |
| `SQL*Net message to client` | Sending large result sets to client | Check rows/exec for top SQL | Reduce result set size, pagination, check network bandwidth |

### Resource Manager

| Wait Event | Root Cause | Investigation Steps | Recommendations |
|---|---|---|---|
| `resmgr:cpu quantum` | Sessions throttled by Oracle Resource Manager | Check DBRM plan in effect | Review consumer group assignments, adjust CPU limits, or disable DBRM if not intentional |
| `resmgr: become active` | Session waiting to become active under DBRM | Check active session limits in plan | Increase session limit or reorganize consumer groups |

---

## Instance Efficiency — What Low Values Mean

| Metric | If Low → Means | Fix |
|---|---|---|
| Buffer Hit % | Many physical reads — buffer cache too small OR full table scans | Increase `db_cache_size`, add indexes, check SQL |
| Library Hit % | Shared pool too small OR too many hard parses | Increase `shared_pool_size`, use bind variables |
| Execute to Parse % | High parse-to-execute ratio — not reusing cursors | Use bind variables, enable cursor sharing, check `session_cached_cursors` |
| Parse CPU to Parse Elapsed % | Parse phase is waiting (latches) rather than running | Use bind variables to reduce hard parses |
| Latch Hit % | Latch contention — high concurrency on internal structures | Reduce hard parses, check specific latch in `V$LATCH` |
| Non-Parse CPU % | Most CPU is non-parse — good if high | N/A |
| In-memory Sort % | Sorts spilling to disk | Increase PGA target |
| Soft Parse % | Proportion of soft parses out of total — should be > 95% | Same as Execute to Parse — use bind variables |

---

## Memory Sizing Rules of Thumb

### Buffer Cache
- Advisory shows 10%+ improvement with more memory → undersized
- Buffer Hit % < 95% and I/O wait events are high → undersized or full scans
- Use KEEP pool for frequently-accessed small tables/indexes

### Shared Pool
- Library Hit % < 98% → shared pool too small or literal SQL abuse
- Frequent ORA-04031 errors → shared pool fragmentation
- Increase `shared_pool_size`; use `ALTER SYSTEM FLUSH SHARED_POOL` only as emergency

### PGA
- `%PGA targets honoured` < 90% → PGA_AGGREGATE_TARGET too small
- Multi-pass operations > 0% → critical; seriously undersized
- One-pass operations < 5% of total → good
- Check advisory: if adding PGA reduces one/multi-pass dramatically → increase target

---

## SQL Problem Patterns

| Pattern | Diagnosis | Fix |
|---|---|---|
| Gets/exec > 100,000 | Full table scan or very large index range scan | Check plan with DBMS_XPLAN, add index or partition pruning |
| Rows/exec << Rows in table, but Gets/exec very high | Inefficient plan fetching too many rows then filtering | Fix predicate, ensure statistics are current |
| Hard parses > 100/sec | Literal SQL without bind variables | Force bind variables in app or set `cursor_sharing=FORCE` |
| Elapsed/exec > 1s for OLTP SQL | Bad plan, missing index, or lock waits | Check plan, check wait breakdown in AWR SQL detail |
| CPU% of elapsed < 20% | Query is I/O or wait bound | Look at wait events in AWR SQL stats for that SQL ID |
| CPU% of elapsed > 90% | CPU-intensive — sorts, hash joins, no I/O | Check plan for Cartesian products, missing indexes, bad cardinality estimates |
| Row count between executions varies wildly | Bind variable peeking + skewed data | Use `BIND_AWARE` hints, histograms, or cardinality hints |

---

## Diagnostic SQL Snippets (run on database, not in AWR)

```sql
-- Top segments by buffer busy waits
SELECT owner, segment_name, segment_type, buffer_busy_waits
FROM V$SEGMENT_STATISTICS
WHERE statistic_name = 'buffer busy waits'
ORDER BY buffer_busy_waits DESC FETCH FIRST 10 ROWS ONLY;

-- Sessions currently blocking others
SELECT s.sid, s.serial#, s.username, s.sql_id, s.event,
       s.blocking_session, s.seconds_in_wait
FROM V$SESSION s
WHERE s.blocking_session IS NOT NULL
ORDER BY s.seconds_in_wait DESC;

-- Hard parse offenders
SELECT sql_text, parse_calls, executions, round(parse_calls/greatest(executions,1)*100,1) pct
FROM V$SQL
WHERE parse_calls > 100
ORDER BY parse_calls DESC FETCH FIRST 20 ROWS ONLY;

-- PGA usage check
SELECT name, value/1024/1024 mb
FROM V$PGASTAT
WHERE name IN ('aggregate PGA target parameter','aggregate PGA auto target',
               'total PGA inuse','total PGA allocated',
               'over allocation count');
```
