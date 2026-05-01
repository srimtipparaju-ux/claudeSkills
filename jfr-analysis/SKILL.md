---
name: jfr-analysis
description: >
  Use this skill whenever a user uploads or references a Java Flight Recorder (JFR)
  recording and wants it analyzed for performance issues. Triggers include: any mention
  of 'JFR', 'Java Flight Recorder', 'jfr file', '.jfr', 'JMC', 'Java Mission Control',
  'flight recording', 'continuous recording', 'jcmd JFR', 'profiling recording', or when
  the user uploads a .jfr file or a JFR report export. Also trigger when the user asks
  about CPU profiling, method-level hotspots, GC analysis, lock profiling, I/O profiling,
  memory allocation profiling in Java, or wants to know what a Java application is
  spending time on at the method level. Always use this skill — JFR data has many
  dimensions and requires a structured analysis order to avoid missing the real bottleneck.
---

# JFR (Java Flight Recorder) Analysis Skill

## What this skill does

Analyzes Java Flight Recorder data — the gold standard for low-overhead JVM profiling — to:
1. Identify CPU hotspots at the method level (flame graph / call tree analysis)
2. Analyze GC events: frequency, pause times, allocation pressure
3. Detect lock contention and blocked time
4. Profile I/O events (file, socket, database)
5. Analyze memory allocation hotspots (who is allocating the most)
6. Identify JIT compilation issues and deoptimizations
7. Produce a downloadable report with findings and tuning recommendations

---

## Step 1: Read the Input

### Raw `.jfr` file
Binary format — cannot be read directly. Ask the user to export a report:

> "A .jfr file is a binary recording. To analyze it, open it in Java Mission Control (JMC — free from oracle.com or adoptium.net), then:
> 1. File → Open File → select .jfr
> 2. Go to each tab: Method Profiling, GC, Lock Instances, I/O, Allocation
> 3. Screenshot or export key tables as text/CSV
> 4. Share the exported data here
>
> Alternatively, use the JFR CLI to dump event summaries:
> ```bash
> jfr summary recording.jfr
> jfr print --events CPUSample,GCPhasePause,JavaMonitorBlocked recording.jfr > jfr_events.txt
> ```"

### Exported report / text output from JMC
```bash
wc -c /mnt/user-data/uploads/<file>
head -200 /mnt/user-data/uploads/<file>
cat /mnt/user-data/uploads/<file>   # if manageable size
```

### `jfr print` text output
Parse the structured event output:
```bash
grep "CPUSample\|GCPhasePause\|JavaMonitorBlocked\|SocketRead\|FileRead" /mnt/user-data/uploads/<file> | head -200
```

---

## Step 2: Recording Metadata

Extract from the report header:
- **Recording duration**: how long was the JFR running?
- **Recording type**: Continuous (low overhead) vs Profiling (higher fidelity)?
- **JVM version and flags**
- **Event types captured**: which categories are in this recording?
- **Start/end time**: correlate with incident window if known

---

## Step 3: CPU Profiling Analysis (Method Hotspots)

This is usually the most valuable JFR section.

JFR captures CPU samples (`CPUSample` events) typically at 10ms intervals — each sample captures the call stack of all RUNNABLE threads.

### Reading a flame graph or call tree
Aggregated CPU samples show the methods consuming the most CPU time:

```
Method                              Samples    % Self   % Total
com.example.ProductService.search      4,820     23%      45%
  com.example.db.QueryBuilder.build    2,100     18%      22%
    java.util.regex.Matcher.find         890     12%      12%
```

- **% Self (exclusive)**: CPU time spent IN this method (not its callees) — the actual hotspot
- **% Total (inclusive)**: CPU time in this method + everything it calls — shows which code path is hot
- High self% = the work is HERE
- High total% but low self% = this method calls something expensive

### What to look for:

**Regex in hot path**
```
java.util.regex.Pattern.matcher → high self%
```
→ Regex being compiled every call, or complex pattern on large input.
→ Fix: pre-compile patterns as static final `Pattern.compile(...)`. Avoid `String.matches()` in loops.

**Reflection overhead**
```
sun.reflect.GeneratedMethodAccessor.invoke → high self%
java.lang.reflect.Method.invoke
```
→ Reflection in hot path. Often from serialization, DI frameworks, or method invocation.
→ Fix: cache `Method` objects, use interface dispatch, generate bytecode.

**String operations**
```
java.lang.String.substring / concat / format → high self%
```
→ Excessive String creation in loops.
→ Fix: use `StringBuilder`, avoid `String.format` in tight loops.

**Collection copying / sorting**
```
java.util.Collections.sort / java.util.Arrays.sort → high self%
java.util.ArrayList.<init> → many new ArrayList() in loops
```
→ Sorting large collections repeatedly, or creating many short-lived lists.
→ Fix: sort once and cache, use pre-sized collections, use streaming.

**JSON/XML serialization**
```
com.fasterxml.jackson.databind.ObjectMapper.writeValue → high self%
```
→ ObjectMapper called many times, or serializing very large objects.
→ Fix: reuse ObjectMapper (it's thread-safe), use streaming API for large payloads.

**Cryptography**
```
sun.security.provider.SHA.implCompress → high self%
javax.crypto.Cipher.doFinal
```
→ Crypto in hot path. Random number generation can also be slow.
→ Fix: cache SecureRandom instance, use hardware crypto acceleration, cache derived keys.

---

## Step 4: GC Analysis

JFR captures detailed GC events — far richer than GC logs alone.

### Key GC events to extract:
- `GCPhasePause` — stop-the-world pause events: duration and type
- `GarbageCollection` — full GC info: cause, duration, heap before/after
- `GCHeapSummary` — heap size at each GC
- `ObjectAllocationInNewTLAB` / `ObjectAllocationOutsideTLAB` — allocation hotspots
- `PromotionFailed` — object couldn't be promoted from young to old gen

### GC pause analysis:

| GC Pause Pattern | Diagnosis |
|---|---|
| Frequent young GC (< 1s each) | High allocation rate — short-lived objects, minor GC keeping up |
| Long young GC (> 500ms) | Young gen too small, or survivor space pressure |
| Frequent Old Gen GC | Objects promoted too fast, old gen too small, memory leak |
| Full GC (stop-the-world) | Old gen near full — critical. Check heap dump for leak. |
| Concurrent GC cycles not completing before full GC | GC not keeping up with allocation rate — increase heap or reduce allocation |

Compute:
- **GC overhead**: total GC pause time / recording duration × 100 — > 5% is high, > 10% is critical
- **Allocation rate**: bytes allocated per second (from `ObjectAllocationInNewTLAB` events)
- **Promotion rate**: bytes promoted to old gen per second

### Allocation hotspots (if captured):

JFR can show which methods are allocating the most memory:
```
Method                                    Allocated     % of Total
com.example.api.ResponseBuilder.build()    2.4 GB/min    42%
  new byte[] in ObjectOutputStream.write    1.8 GB/min    31%
```

**Common allocation culprits:**
- `byte[]` creation in serialization — fix: reuse buffers, use streaming
- `String` creation — fix: StringBuilder, intern pool, avoid String.format in loops
- Boxing primitives — fix: use primitive collections (Eclipse Collections, FastUtil)
- Short-lived domain objects — fix: object pooling, reuse patterns
- Large arrays allocated per request — fix: pooled allocators

---

## Step 5: Lock Contention Analysis

JFR captures `JavaMonitorBlocked` and `JavaMonitorWait` events — these are much more precise than thread dumps because they include actual wait durations.

For each lock contention event:
- **Monitor class**: what object was locked?
- **Duration**: how long was the thread blocked?
- **Blocked thread**: which thread was waiting?
- **Stack trace**: what code path hit the lock?

Aggregate by monitor class:
```
Monitor: com.example.cache.LocalCache
  Total blocked time: 142s
  Events: 48,203
  Avg wait: 2.9ms
  Max wait: 8,430ms
  Top blocking stack: LocalCache.get() ← LocalCache.computeIfAbsent()
```

Any monitor with total blocked time > 5% of recording duration is a significant bottleneck.

---

## Step 6: I/O Analysis

### Socket I/O (`SocketRead`, `SocketWrite` events)
```
Remote address: db-server:5432
  Total read time: 38s
  Events: 12,040
  Avg: 3.2ms  Max: 4,200ms (4.2s!)
  Called from: com.example.dao.OrderDAO.findAll()
```

High max socket read times → downstream service latency spikes.
Many small reads → chatty protocol, consider batching.

### File I/O (`FileRead`, `FileWrite` events)
Large file reads in request path → move to async I/O or cache.
Frequent small writes → use `BufferedWriter`, batch writes.

### Database (if JDBC events captured)
Some JFR configurations capture JDBC events:
```
DatabaseQuery events:
  SELECT * FROM orders WHERE customer_id=? → avg 280ms (1,204 executions)
```

---

## Step 7: JIT Compilation and Deoptimization

### `Deoptimization` events
JIT can "deoptimize" a method when its assumptions are violated:
```
Deoptimization: com.example.OrderProcessor.process(Order)
  Reason: null_check
  Action: reinterpret
```
- Frequent deoptimizations = JIT spending time recompiling, performance cliff
- `null_check` → add null guards so JIT can optimize confidently
- `class_check` → polymorphic call site — reduce number of concrete types at call site
- `unstable_if` → branch prediction failing — restructure conditional logic

### `CompilerQueueStats` — compilation queue depth
If compilation queue is backed up → JIT can't keep up with hot methods → code running interpreted.
→ Increase JIT compiler threads: `-XX:CICompilerCount=N`

---

## Step 8: Findings and Recommendations

```
Finding N — [Severity] — [Category: CPU / GC / Lock / I/O / JIT]
Evidence:      <specific JFR metrics — sample counts, times, sizes>
Hot method:    <class.method() if applicable>
Root cause:    <what is actually happening>
Impact:        <CPU %, GC overhead %, latency ms>
Recommendations:
  1. Code fix: <specific change>
  2. JVM flag: <if applicable>
  3. Verification: <how to confirm fix worked in next JFR>
```

---

## Step 9: Inline Summary

```
## JFR Analysis

**Recording:** <duration> | **Type:** Profiling/Continuous | **JVM:** <version>

### Top CPU Hotspots
| Method | Self% | Total% | Issue |
|---|---|---|---|

### GC Summary
| Metric | Value | Assessment |
|---|---|---|
| GC overhead | X% | 🔴/🟡/🟢 |
| Young GC frequency | N/min | |
| Max pause | Xms | |
| Allocation rate | X MB/s | |

### Lock Contention
| Monitor | Total Blocked Time | Events | Avg Wait |
|---|---|---|---|

### Findings
1. 🔴 [Critical] ...
```

---

## Step 10: Generate Word Report

Read `/mnt/skills/public/docx/SKILL.md`.

Report sections: Cover, Recording Metadata, CPU Hotspots (flame graph summary table), GC Analysis, Lock Contention, I/O Analysis, JIT Analysis, Findings, JVM Tuning Flags Recommended.

```bash
node /home/claude/jfr_report.js
cp /home/claude/jfr_report.docx /mnt/user-data/outputs/jfr_report.docx
```

---

## Reference files

- `references/jvm-flags.md` — Common JVM performance flags with explanations, when to use, and safe defaults

---

## Key principles

- **Self% in CPU samples is the real hotspot** — don't be distracted by total% of framework code; find where YOUR code burns CPU
- **GC overhead > 5% means fix memory first** — CPU tuning is pointless if GC is stealing 20% of time
- **Lock contention duration > count** — 10 locks for 100ms each > 10,000 locks for 1ms each
- **Allocation rate drives GC** — if you can halve allocations, GC overhead roughly halves
- **JFR is low overhead (< 2%)** — recommend it run continuously in production with `jcmd <pid> JFR.start duration=1h filename=recording.jfr`
