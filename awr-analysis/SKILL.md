---
name: awr-analysis
description: >
  Use this skill whenever a user uploads or mentions an Oracle AWR (Automatic Workload Repository)
  report and wants it analyzed, reviewed, or diagnosed. Triggers include: any mention of 'AWR',
  'awr report', 'Oracle performance report', 'workload repository', 'statspack', or when the user
  uploads an .html or .txt file that looks like an Oracle database report. Also trigger when the
  user asks about Oracle database performance problems, slow SQL, wait events, high CPU, I/O issues,
  or memory pressure and has an attached file. Always use this skill — do not try to analyze AWR
  reports without it, as the structured approach here ensures consistent, thorough diagnosis.
---

# AWR Report Analysis Skill

## What this skill does

Analyzes Oracle AWR (Automatic Workload Repository) reports — in HTML or plain-text format — to:
1. Extract key performance metrics
2. Identify performance problems across SQL, wait events, memory, and I/O
3. Assign severity ratings (Critical / High / Medium / Low)
4. Provide actionable recommendations
5. Produce a downloadable Word document report

---

## Step 1: Read the AWR File

The AWR file is at `/mnt/user-data/uploads/<filename>`. Determine the format:

### HTML format
```python
import subprocess
result = subprocess.run(['wc', '-c', '/mnt/user-data/uploads/<file>'], capture_output=True, text=True)
# Then extract text from HTML
from html.parser import HTMLParser

class AWRTextExtractor(HTMLParser):
    def __init__(self):
        super().__init__()
        self.text_parts = []
        self.skip = False
    def handle_starttag(self, tag, attrs):
        if tag in ('script', 'style'):
            self.skip = True
    def handle_endtag(self, tag):
        if tag in ('script', 'style'):
            self.skip = False
    def handle_data(self, data):
        if not self.skip and data.strip():
            self.text_parts.append(data.strip())

with open('/mnt/user-data/uploads/<file>', 'r', errors='replace') as f:
    html = f.read()

parser = AWRTextExtractor()
parser.feed(html)
awr_text = '\n'.join(parser.text_parts)
print(awr_text[:5000])  # Preview
```

### Plain text format
```bash
wc -c /mnt/user-data/uploads/<file>
head -200 /mnt/user-data/uploads/<file>
```
If under 500KB: read the full file. If larger, read in sections (head, grep for sections, tail).

---

## Step 2: Extract Key Sections

Parse the following sections from the AWR text. Each section has characteristic headers:

| Section | Header patterns to look for |
|---|---|
| Report summary | "DB Name", "Snap Id", "Begin Snap", "End Snap", "Elapsed" |
| Load profile | "Load Profile", "Redo size", "Logical reads", "Block changes" |
| Instance efficiency | "Instance Efficiency", "Buffer Hit", "Library Hit", "Execute to Parse" |
| Top wait events | "Top 10 Foreground Events", "Wait Events", "Event", "Waits", "Time(s)" |
| Top SQL by elapsed time | "SQL ordered by Elapsed Time", "Elapsed Time", "Executions", "SQL Id" |
| Top SQL by CPU | "SQL ordered by CPU Time" |
| Top SQL by I/O | "SQL ordered by User I/O Wait Time", "SQL ordered by Gets", "SQL ordered by Reads" |
| Memory stats | "Memory Statistics", "SGA", "PGA", "Buffer Cache", "Shared Pool" |
| I/O stats | "Tablespace IO Stats", "File IO Stats", "Physical Reads", "Physical Writes" |
| Advisory | "Buffer Cache Advisory", "Shared Pool Advisory", "PGA Memory Advisory" |

Extract each section as its own text block for analysis.

---

## Step 3: Analyze Each Domain

### 3A — Report Header / Snapshot Info
Extract:
- DB Name, Instance, Version
- Begin/End Snap time, Elapsed minutes
- Sessions, Avg Active Sessions (AAS)
- DB Time, CPU Time

**Flag:** AAS > number of CPUs → database is overloaded.

---

### 3B — Load Profile Analysis
Key ratios to compute and evaluate:

| Metric | Concern threshold |
|---|---|
| Redo size / sec | > 10 MB/s indicates heavy DML |
| Logical reads / sec | Baseline dependent; spikes indicate full scans |
| Hard parses / sec | > 100/s → bind variables missing |
| Soft parses / sec | Very high with low hard parses → parse overhead |
| Executes / sec | Useful for context on SQL volume |
| User calls / sec | High without corresponding DB Time may indicate idle waits |

---

### 3C — Instance Efficiency
Check these ratios (values near 100% are good):

| Metric | Warning threshold | Critical threshold |
|---|---|---|
| Buffer Hit % | < 95% | < 90% |
| Library Hit % | < 95% | < 90% |
| Execute to Parse % | < 80% | < 60% |
| Parse CPU to Parse Elapsed % | < 50% | — (indicates latch contention) |
| Latch Hit % | < 99% | < 98% |

---

### 3D — Wait Events Analysis
The most diagnostic section. For each top wait event:

**CPU / Scheduler waits:**
- `CPU time` — normal, should be proportional
- `resmgr:cpu quantum` → Resource Manager throttling

**I/O waits (physical):**
- `db file sequential read` → single-block reads (index lookups, slow storage)
- `db file scattered read` → multi-block reads (full table scans)
- `direct path read/write` → PGA/temp I/O
- `log file sync` → redo log / commit latency
- `log file parallel write` → redo disk bottleneck

**Concurrency waits:**
- `buffer busy waits` → hot blocks, segment header contention
- `read by other session` → many sessions reading same block
- `enq: TX - row lock contention` → application locking issue
- `enq: HW - contention` → HWM (high water mark) contention
- `library cache lock/pin` → DDL during peak, hard parse storms
- `cursor: pin S wait on X` → parsing contention

**Network / client:**
- `SQL*Net message from client` → idle/client think time (usually OK)
- `SQL*Net more data from/to client` → network throughput issue

Flag any event consuming > 5% of total DB Time.

---

### 3E — Top SQL Analysis
For each SQL statement in the top lists:

1. **Elapsed time**: Total and per-execution — high per-exec indicates a bad plan
2. **CPU time**: CPU% of elapsed — low % suggests I/O or wait-bound
3. **Executions**: Very high exec count + low per-exec → aggregate impact
4. **Rows processed**: Rows/exec — large values suggest missing WHERE predicates
5. **Buffer gets**: Gets/exec — high values → inefficient plan (full scans)
6. **Physical reads**: Reads/exec — I/O intensive query

**Pattern recognition:**

| Pattern | Likely cause |
|---|---|
| High gets/exec, low rows | Full table scan, missing index |
| High physical reads | Poor buffer cache hit, large scan, no partition pruning |
| High CPU per exec | Inefficient sort/hash/filter operation |
| Many executions, simple SQL | Missing bind variables (literals causing hard parses) |
| Single exec, huge elapsed | One-off batch job or problematic plan change |

---

### 3F — Memory Analysis

**SGA components:**
- Buffer Cache — check advisory: if adding memory shows > 10% improvement, cache is undersized
- Shared Pool — Library Cache hit < 95% or frequent ORA-04031 → undersized
- Large Pool — check if parallel query or RMAN is used

**PGA:**
- `%PGA targets honoured` — if < 90%, PGA_AGGREGATE_TARGET too small
- Optimal/One-pass/Multi-pass sorts/hash joins ratio — multi-pass > 0% is a problem

---

### 3G — I/O Analysis
From Tablespace and File I/O sections:

- Identify tablespaces with highest read/write counts
- Check Av Rd (ms) — average read time > 20ms is concerning, > 50ms is critical
- Identify if TEMP tablespace is doing heavy I/O (PGA spill)
- Check if UNDO is doing heavy read (long-running queries causing undo reads)

---

## Step 4: Build the Findings List

For each problem found, create a finding object:

```python
finding = {
    "severity": "Critical|High|Medium|Low",
    "category": "SQL Performance|Wait Events|Memory|I/O|Parsing|Locking",
    "title": "Short descriptive title",
    "evidence": "Specific numbers from the AWR (e.g., 'db file sequential read: 45% of DB Time, avg wait 28ms')",
    "impact": "What this is causing (e.g., 'Queries waiting on I/O, users experiencing slow response')",
    "recommendations": [
        "Specific action 1",
        "Specific action 2",
    ]
}
```

**Severity guide:**
- **Critical**: DB is failing or severely degraded; immediate action needed
- **High**: Significant performance impact; fix within days
- **Medium**: Inefficiency present; fix during next maintenance window
- **Low**: Best practice gap; fix opportunistically

---

## Step 5: Conversational Summary

Deliver a structured inline summary before the document:

```
## AWR Analysis Summary
**Database:** <name> | **Version:** <ver> | **Period:** <start> – <end> (<N> min)

### Overall Health: [Critical / Degraded / Fair / Good]
<2–3 sentence executive summary>

### Top Findings
1. 🔴 [Critical] <Title> — <one-line evidence>
2. 🟠 [High] <Title> — <one-line evidence>
3. 🟡 [Medium] <Title> — <one-line evidence>
...

### Key Metrics
| Metric | Value | Status |
|---|---|---|
| Avg Active Sessions | X.X | 🔴/🟡/🟢 |
| Buffer Hit % | XX% | ... |
| Library Hit % | XX% | ... |
| Top Wait Event | <name> (XX% DB Time) | ... |
| Hard Parses/sec | X | ... |
```

Use emoji severity indicators: 🔴 Critical, 🟠 High, 🟡 Medium, 🟢 Good/Info.

---

## Step 6: Generate Downloadable Report

Read the docx skill for full details on document creation:
`/mnt/skills/public/docx/SKILL.md`

Install and create the report:

```bash
npm install -g docx 2>/dev/null | tail -1
```

Create `/home/claude/awr_report.js` with this structure:

```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell,
        HeadingLevel, AlignmentType, BorderStyle, WidthType, ShadingType,
        LevelFormat } = require('docx');
const fs = require('fs');

// Color palette
const COLORS = {
  critical: "C00000",  // Dark red
  high:     "E26B0A",  // Orange
  medium:   "BF9000",  // Gold
  low:      "375623",  // Dark green
  header:   "1F3864",  // Navy
  subhead:  "2E75B6",  // Blue
  rowAlt:   "EBF3FB",  // Light blue
  white:    "FFFFFF",
};

// Severity badge helper
function severityRun(severity) {
  const colorMap = { Critical: COLORS.critical, High: COLORS.high,
                     Medium: COLORS.medium, Low: COLORS.low };
  return new TextRun({
    text: ` ${severity.toUpperCase()} `,
    bold: true, color: COLORS.white,
    highlight: undefined,
    shading: { fill: colorMap[severity] || COLORS.low }
  });
}

// Section heading helper
function heading1(text) {
  return new Paragraph({
    heading: HeadingLevel.HEADING_1,
    children: [new TextRun({ text, font: "Arial", size: 32, bold: true, color: COLORS.header })]
  });
}

function heading2(text) {
  return new Paragraph({
    heading: HeadingLevel.HEADING_2,
    children: [new TextRun({ text, font: "Arial", size: 26, bold: true, color: COLORS.subhead })]
  });
}

// Standard table helper
function makeTable(headers, rows, colWidths) {
  const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
  const borders = { top: border, bottom: border, left: border, right: border };
  const totalWidth = colWidths.reduce((a, b) => a + b, 0);

  return new Table({
    width: { size: totalWidth, type: WidthType.DXA },
    columnWidths: colWidths,
    rows: [
      // Header row
      new TableRow({
        tableHeader: true,
        children: headers.map((h, i) => new TableCell({
          borders,
          width: { size: colWidths[i], type: WidthType.DXA },
          shading: { fill: COLORS.header, type: ShadingType.CLEAR },
          margins: { top: 80, bottom: 80, left: 120, right: 120 },
          children: [new Paragraph({ children: [new TextRun({ text: h, bold: true, color: COLORS.white, font: "Arial", size: 20 })] })]
        }))
      }),
      // Data rows
      ...rows.map((row, ri) => new TableRow({
        children: row.map((cell, ci) => new TableCell({
          borders,
          width: { size: colWidths[ci], type: WidthType.DXA },
          shading: { fill: ri % 2 === 0 ? COLORS.white : COLORS.rowAlt, type: ShadingType.CLEAR },
          margins: { top: 60, bottom: 60, left: 120, right: 120 },
          children: [new Paragraph({ children: [new TextRun({ text: String(cell), font: "Arial", size: 18 })] })]
        }))
      }))
    ]
  });
}

// ── POPULATE THESE FROM YOUR ANALYSIS ──────────────────────────────────────
const reportMeta = {
  dbName: "ORCL",
  instance: "orcl1",
  version: "19.0.0.0.0",
  beginSnap: "2024-01-15 08:00",
  endSnap:   "2024-01-15 10:00",
  elapsedMin: 120,
  overallHealth: "Degraded",  // Critical / Degraded / Fair / Good
};

const keyMetrics = [
  ["Avg Active Sessions (AAS)", "8.4", "🔴 High (> CPU count of 4)"],
  ["DB Time", "1,008 min", ""],
  ["CPU Time", "420 min", "42% of DB Time"],
  ["Buffer Hit %", "94.2%", "🟡 Below 95% threshold"],
  ["Library Hit %", "98.1%", "🟢 Good"],
  ["Execute to Parse %", "72%", "🟡 Below 80%"],
  ["Hard Parses/sec", "142", "🔴 High — missing bind variables"],
  ["Top Wait Event", "db file sequential read", "38% of DB Time"],
];

const findings = [
  {
    severity: "Critical",
    category: "Wait Events / I/O",
    title: "db file sequential read consuming 38% of DB Time",
    evidence: "47,832 waits, avg 28ms, total 1,341s out of 3,520s DB Time",
    impact: "Index-range scans hitting slow storage; all sessions waiting on I/O",
    recommendations: [
      "Identify hot index blocks using DBA_HIST_SEG_STAT and consider caching in KEEP pool",
      "Review storage I/O latency — 28ms average read is above 20ms threshold",
      "Check whether ASM or underlying LUNs are experiencing contention",
      "Consider Automatic Indexing or review execution plans for top SQL by physical reads",
    ]
  },
  // Add more findings here...
];

const topSql = [
  // ["SQL ID", "Elapsed (s)", "Exec", "Elapsed/Exec (s)", "CPU%", "Gets/Exec", "Issue"]
  ["a1b2c3d4e5f6g", "820", "1", "820", "22%", "1,842,000", "Full table scan — missing index"],
];
// ────────────────────────────────────────────────────────────────────────────

const children = [
  // Cover / Header
  new Paragraph({
    alignment: AlignmentType.CENTER,
    spacing: { before: 480, after: 240 },
    children: [new TextRun({ text: "Oracle AWR Performance Analysis", font: "Arial", size: 56, bold: true, color: COLORS.header })]
  }),
  new Paragraph({
    alignment: AlignmentType.CENTER,
    spacing: { after: 120 },
    children: [new TextRun({ text: `Database: ${reportMeta.dbName} | Instance: ${reportMeta.instance}`, font: "Arial", size: 24, color: "444444" })]
  }),
  new Paragraph({
    alignment: AlignmentType.CENTER,
    spacing: { after: 120 },
    children: [new TextRun({ text: `Oracle ${reportMeta.version}`, font: "Arial", size: 22, color: "666666" })]
  }),
  new Paragraph({
    alignment: AlignmentType.CENTER,
    spacing: { after: 120 },
    children: [new TextRun({ text: `Snapshot Period: ${reportMeta.beginSnap} – ${reportMeta.endSnap} (${reportMeta.elapsedMin} min)`, font: "Arial", size: 22, color: "666666" })]
  }),
  new Paragraph({
    alignment: AlignmentType.CENTER,
    spacing: { after: 480 },
    children: [new TextRun({ text: `Overall Health: ${reportMeta.overallHealth}`, font: "Arial", size: 28, bold: true, color: COLORS.high })]
  }),

  // Key Metrics
  heading1("Key Performance Metrics"),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),
  makeTable(
    ["Metric", "Value", "Assessment"],
    keyMetrics,
    [3600, 2400, 3360]
  ),
  new Paragraph({ children: [new TextRun({ text: "", size: 16 })] }),

  // Findings
  heading1("Performance Findings"),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),

  ...findings.flatMap((f, i) => [
    new Paragraph({
      spacing: { before: 240, after: 60 },
      children: [
        new TextRun({ text: `Finding ${i + 1}: `, font: "Arial", size: 24, bold: true }),
        severityRun(f.severity),
        new TextRun({ text: `  ${f.title}`, font: "Arial", size: 24, bold: true }),
      ]
    }),
    makeTable(
      ["Field", "Detail"],
      [
        ["Category", f.category],
        ["Evidence", f.evidence],
        ["Impact", f.impact],
      ],
      [2400, 6960]
    ),
    new Paragraph({ spacing: { before: 120, after: 40 }, children: [new TextRun({ text: "Recommendations:", font: "Arial", size: 22, bold: true })] }),
    ...f.recommendations.map(r => new Paragraph({
      spacing: { after: 40 },
      indent: { left: 360 },
      children: [new TextRun({ text: `• ${r}`, font: "Arial", size: 20 })]
    })),
    new Paragraph({ children: [new TextRun({ text: "", size: 16 })] }),
  ]),

  // Top SQL
  heading1("Top SQL Statements"),
  new Paragraph({ children: [new TextRun({ text: "", size: 12 })] }),
  makeTable(
    ["SQL ID", "Elapsed (s)", "Exec", "Elapsed/Exec", "CPU%", "Gets/Exec", "Diagnosis"],
    topSql,
    [1440, 1200, 800, 1440, 800, 1200, 2480]
  ),
  new Paragraph({ children: [new TextRun({ text: "", size: 16 })] }),

  // Footer note
  new Paragraph({
    spacing: { before: 480 },
    border: { top: { style: BorderStyle.SINGLE, size: 4, color: COLORS.subhead, space: 1 } },
    children: [new TextRun({ text: "Report generated by AWR Analysis Skill. Findings are based on snapshot data only — validate against application behavior and change history before making changes in production.", font: "Arial", size: 16, italics: true, color: "666666" })]
  }),
];

const doc = new Document({
  styles: {
    default: { document: { run: { font: "Arial", size: 22 } } },
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
    properties: {
      page: {
        size: { width: 12240, height: 15840 },
        margin: { top: 1080, right: 1080, bottom: 1080, left: 1080 }
      }
    },
    children
  }]
});

Packer.toBuffer(doc).then(buf => {
  fs.writeFileSync('/home/claude/awr_analysis_report.docx', buf);
  console.log('Report written: /home/claude/awr_analysis_report.docx');
});
```

Run it:
```bash
node /home/claude/awr_report.js
cp /home/claude/awr_analysis_report.docx /mnt/user-data/outputs/awr_analysis_report.docx
```

---

## Step 7: Validate and Present

```bash
# Validate the docx
python /mnt/skills/public/docx/scripts/office/validate.py /home/claude/awr_analysis_report.docx
```

Then use `present_files` to share `/mnt/user-data/outputs/awr_analysis_report.docx` with the user.

---

## Reference: Common AWR Problems Quick-Reference

See `references/common-problems.md` for a comprehensive table of wait events, their root causes, and recommended fixes. Read it when you encounter an unfamiliar wait event or need deeper diagnostic guidance.

---

## Important Notes

- **Always show the inline summary first** before generating the document — the user needs immediate answers
- **Cite specific numbers** from the AWR — never make vague statements like "I/O is slow"
- **Prioritize by DB Time %** — a wait event with 50 occurrences but 40% DB Time matters far more than one with 50,000 occurrences at 0.1%
- **Be conservative with recommendations** — suggest investigation steps before prescribing fixes for production systems
- **Note the snapshot duration** — a 10-minute snapshot during a batch job is very different from a 2-hour OLTP window
