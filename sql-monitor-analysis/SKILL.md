---
name: sql-monitor-analysis
description: >
  Use this skill whenever a user uploads or mentions an Oracle SQL Monitoring Report
  and wants it analyzed, tuned, or diagnosed. Triggers include: any mention of
  'SQL monitoring report', 'SQL monitor', 'real-time SQL monitoring', 'DBMS_SQLTUNE',
  'execution plan analysis', 'SQL plan', 'slow SQL', 'explain plan', or when the user
  uploads an .html or .txt file that contains an Oracle execution plan or SQL monitoring
  output. Also trigger when the user pastes an execution plan inline and asks what is
  wrong with it, why a query is slow, or how to tune it. Always use this skill for
  any execution plan or SQL monitoring analysis — the structured approach here ensures
  nothing is missed.
---

# SQL Monitoring Report Analysis Skill

## What this skill does

Analyzes Oracle SQL Monitoring Reports (generated via `DBMS_SQLTUNE.REPORT_SQL_MONITOR`)
or any pasted execution plan to:

1. Parse and reconstruct the full execution plan tree
2. Identify exactly where time and work are being spent, step by step
3. Diagnose root causes (bad row estimates, full scans, missing indexes, join issues, etc.)
4. Produce operation-level findings with severity ratings
5. Recommend specific, actionable fixes
6. Generate a downloadable Word report

---

## Step 1: Read the Input

The report is at `/mnt/user-data/uploads/<filename>` or pasted inline.

### HTML format (most common — Oracle default)
```python
from html.parser import HTMLParser

class TextExtractor(HTMLParser):
    def __init__(self):
        super().__init__()
        self.parts = []
        self.skip = False
    def handle_starttag(self, tag, attrs):
        if tag in ('script', 'style'): self.skip = True
    def handle_endtag(self, tag):
        if tag in ('script', 'style'): self.skip = False
    def handle_data(self, data):
        if not self.skip and data.strip():
            self.parts.append(data.strip())

with open('/mnt/user-data/uploads/<file>', 'r', errors='replace') as f:
    html = f.read()
parser = TextExtractor()
parser.feed(html)
text = '\n'.join(parser.parts)
print(text[:8000])
```

### Plain text format
```bash
wc -c /mnt/user-data/uploads/<file>
cat /mnt/user-data/uploads/<file>   # if under 100KB
head -300 /mnt/user-data/uploads/<file>  # if larger
```

### Inline paste
If the user pasted the plan directly in the chat, work from that text — no file read needed.

---

## Step 2: Extract Report Header Metrics

Before looking at the plan, capture the top-level SQL metrics. These give critical context for interpreting the plan.

| Field | Where to find it | Why it matters |
|---|---|---|
| SQL ID | Header | Identifies the statement |
| SQL Text | Full text section | Need to read joins, predicates, hints |
| Status | Header (DONE / EXECUTING / etc.) | Was it killed? Still running? |
| Duration | Header | Total wall-clock elapsed time |
| DB Time | Header | CPU + wait time (excludes idle) |
| Executions | Header | 1 run vs 1000 changes everything |
| Fetch Calls | Header | Row-by-row vs bulk fetch |
| Buffer Gets | Header | Logical I/O total |
| Physical Reads | Header | I/O from disk |
| Rows Processed | Header | Total rows returned to client |
| Parallel Degree | Header | Is PX being used? |
| Parse Time | Header | High parse = missing cursor reuse |

**Flag immediately:**
- Duration vs DB Time: if Duration >> DB Time, query is waiting (network, client, lock)
- Executions > 1: all plan stats are cumulative — divide by executions for per-run numbers
- Physical Reads >> 0: significant I/O; check buffer cache size and scan operations below

---

## Step 3: Parse the Execution Plan Tree

The plan is a table with these key columns — extract ALL of them:

| Column | Meaning |
|---|---|
| Id | Step number (indented = child of parent above) |
| Operation | What Oracle is doing at this step |
| Name | Object (table/index) being accessed |
| Rows (E-Rows) | Estimated rows Oracle expected |
| Rows (A-Rows) | Actual rows produced (gold — what really happened) |
| Starts | How many times this step was executed |
| E-Time | Estimated time for this step |
| A-Time | Actual cumulative time at this step |
| Buffers | Logical I/O at this step |
| Reads | Physical I/O at this step |
| Temp Space | Spill to disk (sorts/hash joins) |
| Cost | Optimizer cost estimate |
| Predicate Information | Filter and access predicates (section below plan) |

Reconstruct the tree mentally: indentation = parent-child. A JOIN has two children (left input, right input). The root (Id=0 or SELECT STATEMENT) is the final output.

---

## Step 4: Step-by-Step Analysis

For **every step** in the plan, evaluate the following. Flag issues and record them.

### 4A — Time Distribution

Identify which steps account for the most A-Time (actual elapsed time).

For each step, compute:
- **Step % of total**: `step_atime / root_atime * 100`
- Any step > 15% of total time is a **primary hotspot** — lead your analysis here

The A-Time column is **cumulative** (includes children). To find time spent *within* a step only:
```
Step own time = A-Time(step) - sum(A-Time(direct children))
```
A step with large own time and small children time = work is happening HERE, not downstream.

---

### 4B — Row Estimation Accuracy (Most Important Diagnostic)

For every step, compare E-Rows vs A-Rows:

| Ratio (A/E) | Diagnosis |
|---|---|
| 0.5x – 2x | Good estimate |
| 2x – 10x | Minor misestimate — monitor |
| 10x – 100x | **Significant misestimate** — likely stale/missing stats or skewed data |
| > 100x or E=1, A=millions | **Critical** — optimizer making decisions on completely wrong assumptions |
| A=0 when E>0 | Plan branch that produces nothing — wasted work or predicate issue |

**Why this matters:** Every join method, join order, and operation choice is based on E-Rows. Wrong E-Rows cascade — a bad estimate at step 3 corrupts every step above it.

**Root causes of bad estimates:**
- Stale or missing table/column statistics → `DBMS_STATS.GATHER_TABLE_STATS`
- Skewed data without histograms → gather with `METHOD_OPT => 'FOR ALL COLUMNS SIZE AUTO'`
- Correlated predicates (WHERE a=1 AND b=2 where a and b correlate) → use extended statistics: `DBMS_STATS.CREATE_EXTENDED_STATS`
- Functions in WHERE clause (`WHERE UPPER(name) = 'X'`) → stats don't apply → use function-based index
- Bind variable peeking with skewed data → consider `BIND_AWARE` or SQL Plan Baselines

---

### 4C — Access Path Analysis

For each TABLE ACCESS or INDEX operation, evaluate:

**TABLE ACCESS FULL**
- Always flag if the table has > ~10,000 rows and this is in a hotspot step
- Check: is there a usable index for the filter predicates?
- Look at the SQL text: what columns are in the WHERE clause for this table?
- If the WHERE column has an existing index but FTS is chosen: check if selectivity is low (returning > ~10-15% of rows), in which case FTS may be correct
- If selectivity is high (returning < 5% of rows): missing index, wrong index, or stale stats causing optimizer to underestimate selectivity

**TABLE ACCESS BY INDEX ROWID / ROWID BATCHED**
- Generally good — but check Buffers vs A-Rows
- If Buffers >> A-Rows (e.g., 50,000 buffers for 100 rows): index is not selective enough, many random I/Os per row
- If A-Rows is large (> 10,000) with ROWID access: consider covering index to avoid table access entirely

**INDEX RANGE SCAN**
- Good for selective queries
- Check: is the leading column of the index in the access predicate? (see Predicate Information section)
- If the predicate uses a non-leading column only → index skip scan or no index use — may need composite index
- Large A-Rows from a range scan driving a loop → selectivity problem; composite index with better leading column may help

**INDEX FULL SCAN / INDEX FAST FULL SCAN**
- INDEX FAST FULL SCAN: like a table scan but on the index — used for COUNT, aggregates. Often fine.
- INDEX FULL SCAN: ordered scan. Check if this is driving a sort-merge join or ORDER BY satisfaction.

**TABLE ACCESS INMEMORY FULL**
- In-Memory option being used — check if A-Time is still high → may not be populated or query not benefiting

---

### 4D — Join Analysis

Read the SQL text carefully to understand what is being joined and how.

**Nested Loops (NL)**
- Best for: small driving row set (outer input A-Rows should be small — under ~1,000 ideally)
- Problem: outer input returns many rows → inner side executes once PER outer row → `Starts` on inner child = A-Rows of outer
- Diagnosis: check `Starts` on the inner child — if Starts is large (e.g., 500,000), NL is doing 500K index lookups
- Fix: if outer is large, NL is wrong join method → bad stats causing it, or force Hash Join

**Hash Join**
- Best for: large inputs where neither side is tiny
- Check: which side is the build input (left child = hash table built in memory)?
- If build side A-Rows is very large → large hash table → possible PGA spill (check Temp Space column)
- If probe side has Buffers >> A-Rows → probe side doing many lookups with few hits

**Sort Merge Join**
- Check Temp Space on SORT operations feeding it — spill to disk = slow
- Why chosen over Hash Join? Usually when inputs are already sorted (ORDER BY / index order)

**Cartesian Join (MERGE JOIN CARTESIAN)**
- Almost always a bug — check if a join condition is missing in the SQL
- Small cartesian can be intentional (cross join); large cartesian = likely missing predicate

**Join Order Issues**
- Optimizer drives from smallest estimated result to largest
- If a large table appears as the outer/driving input in a NL, check the E-Rows on it — likely a misestimate made it look small
- Read the SQL: are all FK/join columns indexed? Missing index on the inner side of a NL = full scan per outer row

---

### 4E — Predicate Information Analysis

The Predicate Information section (below the plan) is critical — it shows the actual filter and access conditions per step.

For each step in the hotspot area, find its predicates:

**Access Predicate** — used to navigate the index. Good — this is index being used efficiently.

**Filter Predicate** — applied AFTER rows are fetched. This means Oracle fetched rows and then discarded them.

Red flags:
- Filter predicate on a TABLE ACCESS FULL → the filter is not using an index at all; adding an index on that column would convert this to an access predicate
- Filter predicate with a function: `TO_DATE(col, 'YYYY-MM-DD')` or `UPPER(col)` → function prevents index use; recommend function-based index or rewrite query predicate
- Filter predicate on indexed column that should be access: column may not be leading key in composite index → add column as leading key or create new index
- Large row reduction from filter (E-Rows >> A-Rows after filter step): filter is very selective but not indexed → strong index candidate

**Check join predicates:**
- Are FK columns on the inner table of a NL indexed? If not, every NL probe does a full scan
- Are join columns in the predicate info using the right data types? Implicit type conversion (`WHERE num_col = '123'`) prevents index use

---

### 4F — Parallel Execution Analysis (if PX present)

If the plan contains PX COORDINATOR, PX SEND, PX RECEIVE operations:

- Check DOP (degree of parallelism) — is it what was requested?
- Check skew: if one PX slave finishes much later than others (visible in Real-Time SQL Monitor HTML), data distribution is skewed
- PX SEND BROADCAST: small table broadcast to all slaves — usually good
- PX SEND HASH / RANGE: data redistributed by hash/range — necessary for joins; watch for skew
- If parallel query is slower than expected: check if slaves are waiting on I/O (disk bottleneck), or if DOP was downgraded by Resource Manager

---

### 4G — Sort and Hash Operations

**SORT ORDER BY / SORT GROUP BY / SORT JOIN**
- Check Temp Space column — any value here means spill to disk
- Check A-Rows going into the sort vs PGA available
- Fix: increase PGA_AGGREGATE_TARGET, add ORDER BY-satisfying index, rewrite to avoid sort

**HASH GROUP BY / HASH JOIN**
- Same — check Temp Space
- HASH GROUP BY vs SORT GROUP BY: optimizer chooses; HASH is usually faster but uses more memory

---

### 4H — SQL Text Review

Read the full SQL text and flag these patterns:

| Pattern | Issue | Recommendation |
|---|---|---|
| `SELECT *` | Fetches unneeded columns; prevents index-only access | Select only needed columns; enables covering indexes |
| `WHERE UPPER(col) = :b` | Function prevents index use | Create function-based index or normalize data |
| `WHERE col LIKE '%value%'` | Leading wildcard prevents index range scan | Consider full-text index or rewrite |
| `WHERE TO_DATE(col) = :d` | Implicit conversion on indexed column | Store as DATE; use range predicate |
| Missing join predicate | Accidental Cartesian product | Add join condition |
| `WHERE col != value` | Negative predicates rarely use indexes | Rewrite as range if possible |
| `OR` conditions spanning indexed columns | Prevents index use | Rewrite as UNION ALL or use index combine |
| Subquery in SELECT list executed per row | Scalar subquery — check Starts on subquery plan | Rewrite as JOIN |
| Large IN list (100+ values) | May degrade to full scan | Use temp table or bulk join |
| `NOT IN` with nullable column | Returns no rows if subquery has NULL | Use `NOT EXISTS` instead |
| Nested scalar subquery in WHERE | Each outer row triggers full subquery execution | Rewrite as JOIN or lateral |

---

## Step 5: Build the Findings List

For every issue found, produce a structured finding:

```
Finding N — [Severity] — [Category]
Step ID:       <plan step number(s)>
Operation:     <TABLE ACCESS FULL / NESTED LOOPS / etc.>
Object:        <table or index name>
Evidence:      <specific numbers — E-Rows, A-Rows, A-Time %, Starts, Buffers>
Root Cause:    <what is actually wrong>
Impact:        <what this is causing — time, I/O, CPU>
Recommendation:
  1. <Specific action — include index DDL, stats command, or query rewrite>
  2. <Follow-up action>
  3. <How to verify the fix worked>
```

**Severity guide:**
- 🔴 Critical: Step accounts for > 40% of total time, or row estimate is off by > 100x, or Cartesian join found
- 🟠 High: Step accounts for 15–40% of total time, or row estimate off by 10–100x, or NL with large Starts
- 🟡 Medium: Suboptimal access path but not dominant, minor misestimates, missing covering index opportunity
- 🟢 Low / Info: Best practice gap, minor inefficiency, worth noting but not urgent

---

## Step 6: Inline Conversational Summary

Always deliver this FIRST, before generating any document:

```
## SQL Monitoring Report Analysis

**SQL ID:** <id> | **Status:** <DONE/RUNNING> | **Duration:** <N>s | **Executions:** <N>
**Total DB Time:** <N>s | **Buffer Gets:** <N> | **Physical Reads:** <N>

### Overall Verdict: [Well-tuned / Needs Attention / Poorly Tuned / Critical Issues Found]
<2–3 sentence executive summary of what's happening and why>

### Hotspot Steps (where time is actually going)
| Step | Operation | Object | A-Time % | E-Rows | A-Rows | Finding |
|---|---|---|---|---|---|---|
| 5 | TABLE ACCESS FULL | ORDERS | 67% | 100 | 4,820,000 | 🔴 Missing index — full scan returning millions |
| 3 | NESTED LOOPS | — | 22% | 1 | 284,000 | 🟠 Outer loop ran 284K times — wrong join method |

### Findings Summary
1. 🔴 [Critical] <Title> — <one-line evidence>
2. 🟠 [High] <Title> — <one-line evidence>
3. 🟡 [Medium] <Title> — <one-line evidence>

### Quick Wins
List 1–3 highest-impact changes that are easy to implement.
```

---

## Step 7: Generate Downloadable Word Report

Read `/mnt/skills/public/docx/SKILL.md` for full document creation details.

```bash
npm install -g docx 2>/dev/null | tail -1
```

Create `/home/claude/sqlmon_report.js`:

```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell,
        HeadingLevel, AlignmentType, BorderStyle, WidthType, ShadingType,
        LevelFormat } = require('docx');
const fs = require('fs');

const COLORS = {
  critical: "C00000", high: "E26B0A", medium: "BF9000", low: "375623",
  info:     "2E75B6", header: "1F3864", subhead: "2E75B6",
  rowAlt:   "EBF3FB", white: "FFFFFF", lightGray: "F2F2F2",
};

const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };

function cell(text, width, opts = {}) {
  return new TableCell({
    borders,
    width: { size: width, type: WidthType.DXA },
    shading: { fill: opts.fill || COLORS.white, type: ShadingType.CLEAR },
    margins: { top: 60, bottom: 60, left: 120, right: 120 },
    children: [new Paragraph({
      children: [new TextRun({
        text: String(text),
        font: "Arial", size: opts.size || 18,
        bold: opts.bold || false,
        color: opts.color || "000000",
      })]
    })]
  });
}

function headerCell(text, width) {
  return cell(text, width, { fill: COLORS.header, bold: true, color: COLORS.white, size: 19 });
}

function severityCell(severity, width) {
  const colors = { Critical: COLORS.critical, High: COLORS.high, Medium: COLORS.medium, Low: COLORS.low };
  return cell(severity, width, { fill: colors[severity] || COLORS.low, bold: true, color: COLORS.white });
}

function h1(text) {
  return new Paragraph({
    heading: HeadingLevel.HEADING_1,
    children: [new TextRun({ text, font: "Arial", size: 32, bold: true, color: COLORS.header })]
  });
}
function h2(text) {
  return new Paragraph({
    heading: HeadingLevel.HEADING_2,
    children: [new TextRun({ text, font: "Arial", size: 26, bold: true, color: COLORS.subhead })]
  });
}
function para(text, opts = {}) {
  return new Paragraph({
    spacing: { after: opts.after || 60 },
    indent: opts.indent ? { left: opts.indent } : undefined,
    children: [new TextRun({ text, font: "Arial", size: opts.size || 20,
      bold: opts.bold || false, italics: opts.italic || false, color: opts.color || "000000" })]
  });
}
function bullet(text) {
  return new Paragraph({
    spacing: { after: 40 }, indent: { left: 360 },
    children: [new TextRun({ text: `• ${text}`, font: "Arial", size: 20 })]
  });
}

// ── POPULATE FROM YOUR ANALYSIS ──────────────────────────────────────────────
const meta = {
  sqlId:        "abc123def456g",
  status:       "DONE",
  duration:     "142s",
  dbTime:       "138s",
  executions:   1,
  bufferGets:   "48,293,000",
  physReads:    "12,840,000",
  rowsReturned: "1,284",
  verdict:      "Critical Issues Found",
  verdictColor: COLORS.critical,
  summary:      "The query is spending 67% of its time performing a full table scan on ORDERS due to a missing index on ORDER_STATUS. A nested loops join is executing 284,000 iterations because of a severely underestimated row count caused by stale statistics. Two indexes and a stats refresh should reduce execution time by an estimated 80–90%.",
};

// Hotspot plan steps
const hotspotSteps = [
  // [Id, Operation, Object, ATime%, ERows, ARows, Starts, Buffers, Finding]
  ["5", "TABLE ACCESS FULL", "ORDERS", "67%", "100", "4,820,000", "1", "41,200,000", "🔴 Missing index on ORDER_STATUS"],
  ["3", "NESTED LOOPS", "", "22%", "1", "284,000", "1", "6,100,000",  "🟠 Outer loop drove 284K NL iterations"],
  ["7", "SORT ORDER BY", "", "8%",  "284,000", "1,284", "1", "0",    "🟡 Sort spilled 240MB to TEMP"],
];

// Full plan (all steps)
const planSteps = [
  // [Id, Operation, Object, ERows, ARows, Starts, ATime, Buffers, Reads]
  ["0",  "SELECT STATEMENT",        "",          "",        "1,284",     "1",   "142s",  "48.2M", "12.8M"],
  ["1",  "SORT ORDER BY",           "",          "284,000", "1,284",     "1",   "11s",   "48.2M", "12.8M"],
  ["2",  "HASH JOIN",               "",          "284,000", "284,000",   "1",   "128s",  "48.2M", "12.8M"],
  ["3",  "NESTED LOOPS",            "",          "1",       "284,000",   "1",   "31s",   "6.1M",  "0"],
  ["4",  "TABLE ACCESS FULL",       "CUSTOMERS", "50,000",  "50,000",    "1",   "3s",    "12,000","0"],
  ["5",  "TABLE ACCESS FULL",       "ORDERS",    "100",     "4,820,000", "1",   "92s",   "41.2M", "12.8M"],
  ["6",  "TABLE ACCESS FULL",       "PRODUCTS",  "500",     "500",       "1",   "2s",    "900",   "0"],
  ["7",  "SORT ORDER BY",           "",          "284,000", "1,284",     "1",   "11s",   "0",     "240MB"],
];

// Findings
const findings = [
  {
    severity: "Critical",
    category: "Access Path",
    stepId: "5",
    operation: "TABLE ACCESS FULL",
    object: "ORDERS",
    evidence: "A-Time: 92s (67% of total). E-Rows: 100, A-Rows: 4,820,000 (48,200x misestimate). Buffers: 41.2M.",
    rootCause: "Full table scan on ORDERS (4.8M rows returned). No index on ORDER_STATUS column used in WHERE clause. Optimizer estimated only 100 rows due to stale statistics — actual selectivity is far lower than estimated.",
    impact: "Single step accounts for 67% of total query time. 41M logical I/Os performed. Every execution of this query scans the entire ORDERS table.",
    recommendations: [
      "Create composite index: CREATE INDEX idx_orders_status_cust ON ORDERS (ORDER_STATUS, CUSTOMER_ID, ORDER_DATE);",
      "Gather fresh statistics: EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA','ORDERS', method_opt=>'FOR ALL COLUMNS SIZE AUTO', cascade=>TRUE);",
      "After stats refresh, if FTS persists despite good selectivity, add hint /*+ INDEX(o idx_orders_status_cust) */ to confirm index fixes the plan, then investigate why optimizer rejects it.",
      "Verify fix: re-run query, check SQL Monitor — TABLE ACCESS FULL on ORDERS should be replaced by INDEX RANGE SCAN.",
    ]
  },
  {
    severity: "High",
    category: "Join Method / Row Estimation",
    stepId: "3",
    operation: "NESTED LOOPS",
    object: "—",
    evidence: "A-Rows: 284,000 outer rows (E-Rows: 1). Inner side (Step 5) executed 284,000 times. Each inner execution does a full scan of ORDERS.",
    rootCause: "Optimizer estimated 1 row from the driving input (CUSTOMERS full scan) due to stale stats. Chose NESTED LOOPS as a result. Actual outer input delivers 284,000 rows, causing 284,000 full scans of ORDERS.",
    impact: "Multiplies the Step 5 full scan impact by 284,000. After fixing Step 5 stats, optimizer should switch to HASH JOIN automatically.",
    recommendations: [
      "Fix root cause: gather statistics on CUSTOMERS table as well — the E-Rows=1 on Step 4 is suspicious.",
      "After stats refresh, optimizer should choose HASH JOIN for this volume of rows.",
      "If NL persists after stats fix, add hint: /*+ USE_HASH(c o) */ to force hash join and verify performance.",
      "Ensure CUSTOMER_ID is indexed on ORDERS table for when NL is used with smaller result sets.",
    ]
  },
  {
    severity: "Medium",
    category: "Sort / Memory",
    stepId: "7",
    operation: "SORT ORDER BY",
    object: "—",
    evidence: "Temp Space: 240MB. A-Rows in: 284,000. A-Rows out: 1,284.",
    rootCause: "Sort operation spilling 240MB to TEMP tablespace because PGA could not hold the 284K-row sort in memory. After the join fix (fewer rows entering the sort), this may resolve itself.",
    impact: "8% of query time. Temp I/O adds latency. After join fix reduces rows, this should shrink significantly.",
    recommendations: [
      "Fix the join issue first — if rows entering the sort drop from 284K to ~1,284, spill will disappear.",
      "If sort spill persists after other fixes: consider adding an index with ORDER BY columns to satisfy sort without operation.",
      "As fallback: increase PGA_AGGREGATE_TARGET or workarea_size_policy=AUTO to allow larger in-memory sorts.",
    ]
  },
];

// Quick wins
const quickWins = [
  "Gather statistics on ORDERS and CUSTOMERS tables — likely resolves the row estimation errors driving multiple issues.",
  "Create index on ORDERS(ORDER_STATUS, CUSTOMER_ID, ORDER_DATE) — eliminates the 67% time full scan.",
  "After both above: re-run and check SQL Monitor — expect Hash Join to replace Nested Loops automatically.",
];
// ─────────────────────────────────────────────────────────────────────────────

const children = [
  // Title
  new Paragraph({ alignment: AlignmentType.CENTER, spacing: { before: 240, after: 120 },
    children: [new TextRun({ text: "Oracle SQL Monitoring Report — Performance Analysis",
      font: "Arial", size: 52, bold: true, color: COLORS.header })] }),
  new Paragraph({ alignment: AlignmentType.CENTER, spacing: { after: 80 },
    children: [new TextRun({ text: `SQL ID: ${meta.sqlId}  |  Status: ${meta.status}  |  Duration: ${meta.duration}  |  Executions: ${meta.executions}`,
      font: "Arial", size: 22, color: "444444" })] }),
  new Paragraph({ alignment: AlignmentType.CENTER, spacing: { after: 400 },
    children: [new TextRun({ text: `Verdict: ${meta.verdict}`, font: "Arial", size: 28, bold: true, color: meta.verdictColor })] }),

  // Summary metrics table
  h1("Report Summary"),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),
  new Table({
    width: { size: 9360, type: WidthType.DXA }, columnWidths: [3120, 3120, 3120],
    rows: [
      new TableRow({ children: [headerCell("Metric", 3120), headerCell("Value", 3120), headerCell("Assessment", 3120)] }),
      new TableRow({ children: [cell("Duration", 3120, {fill: COLORS.lightGray, bold:true}), cell(meta.duration, 3120), cell("Total wall-clock time", 3120)] }),
      new TableRow({ children: [cell("DB Time", 3120, {fill: COLORS.lightGray, bold:true}), cell(meta.dbTime, 3120), cell("Active database work", 3120)] }),
      new TableRow({ children: [cell("Executions", 3120, {fill: COLORS.lightGray, bold:true}), cell(meta.executions, 3120), cell("Plan stats are per-run (not cumulative)", 3120)] }),
      new TableRow({ children: [cell("Buffer Gets", 3120, {fill: COLORS.lightGray, bold:true}), cell(meta.bufferGets, 3120), cell("Total logical I/O", 3120)] }),
      new TableRow({ children: [cell("Physical Reads", 3120, {fill: COLORS.lightGray, bold:true}), cell(meta.physReads, 3120), cell("Disk I/O — high indicates scan ops", 3120)] }),
      new TableRow({ children: [cell("Rows Returned", 3120, {fill: COLORS.lightGray, bold:true}), cell(meta.rowsReturned, 3120), cell("Final result set size", 3120)] }),
    ]
  }),
  new Paragraph({ spacing: { after: 200 }, children: [new TextRun({ text: "", size: 12 })] }),
  para(meta.summary, { italic: true, color: "333333" }),
  new Paragraph({ spacing: { after: 240 }, children: [new TextRun({ text: "", size: 12 })] }),

  // Hotspot table
  h1("Execution Plan Hotspots"),
  para("Steps ordered by actual time consumed. These are the primary targets for tuning.", { italic: true }),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),
  new Table({
    width: { size: 9360, type: WidthType.DXA },
    columnWidths: [480, 2200, 1400, 700, 900, 900, 700, 1080],
    rows: [
      new TableRow({ children: [
        headerCell("Id", 480), headerCell("Operation", 2200), headerCell("Object", 1400),
        headerCell("Time%", 700), headerCell("E-Rows", 900), headerCell("A-Rows", 900),
        headerCell("Starts", 700), headerCell("Finding", 1080),
      ]}),
      ...hotspotSteps.map((s, i) => new TableRow({ children: [
        cell(s[0], 480,  {fill: i%2?COLORS.rowAlt:COLORS.white}),
        cell(s[1], 2200, {fill: i%2?COLORS.rowAlt:COLORS.white}),
        cell(s[2], 1400, {fill: i%2?COLORS.rowAlt:COLORS.white}),
        cell(s[3], 700,  {fill: i%2?COLORS.rowAlt:COLORS.white, bold:true}),
        cell(s[4], 900,  {fill: i%2?COLORS.rowAlt:COLORS.white}),
        cell(s[5], 900,  {fill: i%2?COLORS.rowAlt:COLORS.white, bold:true}),
        cell(s[6], 700,  {fill: i%2?COLORS.rowAlt:COLORS.white}),
        cell(s[7], 1080, {fill: i%2?COLORS.rowAlt:COLORS.white}),
      ]})),
    ]
  }),
  new Paragraph({ spacing: { after: 240 }, children: [new TextRun({ text: "", size: 12 })] }),

  // Full plan table
  h1("Full Execution Plan"),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),
  new Table({
    width: { size: 9360, type: WidthType.DXA },
    columnWidths: [480, 2000, 1200, 800, 800, 640, 800, 820, 820],
    rows: [
      new TableRow({ children: [
        headerCell("Id",2000), headerCell("Operation",2000), headerCell("Object",1200),
        headerCell("E-Rows",800), headerCell("A-Rows",800), headerCell("Starts",640),
        headerCell("A-Time",800), headerCell("Buffers",820), headerCell("Reads",820),
      ]}),
      ...planSteps.map((s, i) => new TableRow({ children: s.map((v, ci) =>
        cell(v, [480,2000,1200,800,800,640,800,820,820][ci], {fill: i%2?COLORS.rowAlt:COLORS.white})
      )})),
    ]
  }),
  new Paragraph({ spacing: { after: 240 }, children: [new TextRun({ text: "", size: 12 })] }),

  // Findings
  h1("Performance Findings"),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),
  ...findings.flatMap((f, i) => [
    new Paragraph({ spacing: { before: 240, after: 80 }, children: [
      new TextRun({ text: `Finding ${i+1}  `, font: "Arial", size: 24, bold: true }),
      new TextRun({ text: ` ${f.severity.toUpperCase()} `, bold: true, color: COLORS.white,
        shading: { fill: {Critical:COLORS.critical,High:COLORS.high,Medium:COLORS.medium,Low:COLORS.low}[f.severity] } }),
      new TextRun({ text: `   ${f.category}`, font: "Arial", size: 22, color: COLORS.subhead }),
    ]}),
    new Table({
      width: { size: 9360, type: WidthType.DXA }, columnWidths: [1800, 7560],
      rows: [
        new TableRow({ children: [cell("Step / Operation", 1800, {fill: COLORS.lightGray, bold:true}), cell(`Step ${f.stepId} — ${f.operation} ${f.object}`, 7560)] }),
        new TableRow({ children: [cell("Evidence", 1800, {fill: COLORS.lightGray, bold:true}), cell(f.evidence, 7560)] }),
        new TableRow({ children: [cell("Root Cause", 1800, {fill: COLORS.lightGray, bold:true}), cell(f.rootCause, 7560)] }),
        new TableRow({ children: [cell("Impact", 1800, {fill: COLORS.lightGray, bold:true}), cell(f.impact, 7560)] }),
      ]
    }),
    para("Recommendations:", { bold: true, after: 40 }),
    ...f.recommendations.map((r, ri) => bullet(`${ri+1}. ${r}`)),
    new Paragraph({ spacing: { after: 200 }, children: [new TextRun({ text: "", size: 12 })] }),
  ]),

  // Quick wins
  h1("Quick Wins"),
  para("Highest-impact changes, easiest to implement first:"),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),
  ...quickWins.map((w, i) => new Paragraph({
    spacing: { after: 80 }, indent: { left: 360 },
    children: [
      new TextRun({ text: `${i+1}. `, font: "Arial", size: 20, bold: true, color: COLORS.subhead }),
      new TextRun({ text: w, font: "Arial", size: 20 }),
    ]
  })),

  new Paragraph({
    spacing: { before: 480 },
    border: { top: { style: BorderStyle.SINGLE, size: 4, color: COLORS.subhead, space: 1 } },
    children: [new TextRun({ text: "Analysis based on SQL Monitoring snapshot data. Validate recommendations in a non-production environment before applying to production. Re-run SQL Monitor after each change to confirm improvement.", font: "Arial", size: 16, italics: true, color: "666666" })]
  }),
];

const doc = new Document({
  styles: {
    default: { document: { run: { font: "Arial", size: 20 } } },
    paragraphStyles: [
      { id: "Heading1", name: "Heading 1", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 32, bold: true, font: "Arial", color: COLORS.header },
        paragraph: { spacing: { before: 360, after: 120 }, outlineLevel: 0 } },
      { id: "Heading2", name: "Heading 2", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 26, bold: true, font: "Arial", color: COLORS.subhead },
        paragraph: { spacing: { before: 240, after: 80 }, outlineLevel: 1 } },
    ]
  },
  sections: [{
    properties: { page: {
      size: { width: 12240, height: 15840 },
      margin: { top: 1080, right: 1080, bottom: 1080, left: 1080 }
    }},
    children
  }]
});

Packer.toBuffer(doc).then(buf => {
  fs.writeFileSync('/home/claude/sqlmon_analysis_report.docx', buf);
  console.log('Done: /home/claude/sqlmon_analysis_report.docx');
});
```

Run it:
```bash
node /home/claude/sqlmon_report.js
cp /home/claude/sqlmon_analysis_report.docx /mnt/user-data/outputs/sqlmon_analysis_report.docx
```

---

## Step 8: Validate and Present

```bash
python /mnt/skills/public/docx/scripts/office/validate.py /home/claude/sqlmon_analysis_report.docx
```

Then use `present_files` to share the report.

---

## Reference files

- `references/plan-operations.md` — Full reference of every Oracle execution plan operation, what it means, and how to diagnose it. Read when encountering an unfamiliar operation.
- `references/index-design.md` — Index design patterns for common query shapes. Read when formulating index recommendations.

---

## Key principles

- **A-Time % is the compass** — always start with the step consuming the most actual time. Everything else is secondary.
- **E-Rows vs A-Rows is the most powerful diagnostic** — a large gap here is almost always the root cause of everything downstream.
- **Starts × inner cost = true cost of NL** — a step that looks cheap individually can be catastrophically expensive when multiplied by Starts.
- **Filter predicates are missed indexes** — every filter predicate that isn't an access predicate is a candidate for an index improvement.
- **Fix stats before adding indexes** — stale stats cause wrong join methods and wrong access paths. A stats refresh often fixes multiple issues at once and should be recommended first.
- **One change at a time** — recommend changes in priority order and advise the user to re-run SQL Monitor after each one to isolate impact.
