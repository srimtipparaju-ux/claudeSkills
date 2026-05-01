---
name: thread-dump-analysis
description: >
  Use this skill whenever a user uploads or pastes a Java thread dump and wants it
  analyzed, diagnosed, or explained. Triggers include: any mention of 'thread dump',
  'jstack', 'thread state', 'deadlock', 'blocked threads', 'thread contention',
  'WAITING', 'BLOCKED', 'hung application', 'high CPU Java', 'unresponsive server',
  or when the user uploads a .txt or .log file that contains Java thread stack traces
  with thread state headers like "java.lang.Thread.State:". Also trigger when the
  user asks why a Java application is hung, slow, or using 100% CPU. Always use this
  skill — thread dump analysis requires a structured approach to find patterns across
  hundreds of threads that are easy to miss when reading linearly.
---

# Thread Dump Analysis Skill

## What this skill does

Analyzes Java thread dumps (from jstack, JVisualVM, JMC, kill -3, or any JVM) to:
1. Parse and categorize all threads by state
2. Identify deadlocks, lock contention, and blocked threads
3. Find CPU-consuming threads and hot code paths
4. Detect thread pool exhaustion and starvation
5. Identify resource leaks and long-running operations
6. Produce a downloadable Word report with findings and fixes

---

## Step 1: Read the Input

Thread dumps are typically `.txt` or `.log` files, or pasted inline.

```bash
wc -l /mnt/user-data/uploads/<file>
head -100 /mnt/user-data/uploads/<file>
```

If file is large (> 500KB), read in chunks — thread dumps from production can be massive:
```bash
# Count threads
grep -c '"' /mnt/user-data/uploads/<file>   # approx thread count (each thread starts with quoted name)
# Read full file
cat /mnt/user-data/uploads/<file>
```

**If multiple dumps were provided** (best practice is 3 dumps 10s apart): analyze each separately, then compare to find threads that are consistently blocked vs transiently waiting.

---

## Step 2: Parse Thread Inventory

For each thread, extract:
- **Thread name** — e.g., `"http-nio-8080-exec-5"`, `"GC task thread#0"`
- **Thread ID** — `tid=0x00007f...`
- **Native ID** — `nid=0x1a2b` (maps to OS PID for CPU profiling)
- **State** — `java.lang.Thread.State: RUNNABLE / BLOCKED / WAITING / TIMED_WAITING`
- **Stack trace** — full call stack
- **Lock info** — `waiting to lock <0x...>`, `locked <0x...>`

Build a summary count:
```
Total threads: N
  RUNNABLE:        N
  BLOCKED:         N   ← contention
  WAITING:         N   ← parked/idle
  TIMED_WAITING:   N   ← sleeping/timed-wait
  TERMINATED:      N
```

---

## Step 3: Deadlock Detection

Look for the `Found one Java-level deadlock:` section — JVM detects these automatically.

If not present, manually check: find all threads in BLOCKED state and trace their lock chains:
1. Thread A is `waiting to lock <0xABC>` — who holds `<0xABC>`?
2. Find the thread that has `locked <0xABC>` — is IT waiting for a lock that A holds?
3. If yes → deadlock. Map the full cycle.

```
Deadlock pattern:
Thread A → waiting for lock X (held by Thread B)
Thread B → waiting for lock Y (held by Thread A)
→ Circular dependency = deadlock = application hang
```

**Flag immediately as Critical if any deadlock found.**

Recommendations for deadlocks:
- Enforce consistent lock acquisition order across all code paths
- Use `java.util.concurrent` lock classes with `tryLock(timeout)` to detect and recover
- Use `ReentrantLock` with timeout instead of `synchronized` blocks
- Consider lock-free data structures (`ConcurrentHashMap`, `AtomicReference`)
- Add deadlock detection monitoring: `ThreadMXBean.findDeadlockedThreads()`

---

## Step 4: Lock Contention Analysis (BLOCKED threads)

Group all BLOCKED threads by the lock object address they're waiting for:

```
Lock <0x000000076b3a2f40> — held by Thread "db-pool-1"
  Waiting (BLOCKED): 23 threads
    "http-nio-8080-exec-1" — waiting 45s
    "http-nio-8080-exec-2" — waiting 38s
    ...
```

For each contention hotspot:
- Identify the lock holder — what is it doing? (read its stack trace)
- How long has the holder held the lock?
- Is the holder itself blocked on something else? (lock chain)
- What is the contended code path? (find the `synchronized` block or method)

**Severity by blocked thread count:**
- 🔴 Critical: > 10 threads blocked on same lock, or any deadlock
- 🟠 High: 5–10 threads blocked, or lock holder doing slow I/O while holding lock
- 🟡 Medium: 2–5 threads blocked
- 🟢 Low: occasional transient blocking

**Common root causes:**
| Symptom | Root Cause | Fix |
|---|---|---|
| Many threads blocked on DB connection pool | Pool exhausted or queries too slow | Increase pool size, tune slow queries, add connection timeout |
| Blocked on `synchronized` cache | Single-lock cache under high concurrency | Replace with `ConcurrentHashMap` or read-write lock |
| Blocked on file/stream write | Synchronous I/O in shared resource | Async I/O, separate write thread, buffered writer |
| Lock holder doing HTTP call while locked | Network call inside synchronized block | Move network call outside the lock |
| Blocked on class loading | Parallel class loading contention | Enable parallel class loading (`-XX:+UnsyncloadClass`) |

---

## Step 5: CPU Analysis (RUNNABLE threads)

High CPU on a Java process = too many RUNNABLE threads doing real work, or tight loops.

For each RUNNABLE thread, read the top of its stack trace — group by common stack frames:

**Patterns indicating high CPU work:**
- Tight loops: thread stack shows the same few frames repeatedly across multiple dumps
- GC-related: `VM Thread`, `GC task thread` — indicates GC pressure (cross-reference with heap)
- Regex/String ops: `java.util.regex`, `String.matches()` — expensive if called in tight loops
- JSON/XML parsing: large payload parsing in request threads
- Sorting/searching large collections
- Hash computation on large objects
- Reflection-heavy frameworks in hot paths

**Finding the CPU-consuming thread by PID:**
```bash
# On Linux, map nid (hex) to thread ID
# nid=0x1a2b → decimal = 6699 → match with: top -H -p <java_pid>
printf '%d\n' 0x1a2b
```

If the user provides a `top -H` output alongside the dump, match the high-CPU thread PIDs to nids in the dump to pinpoint exactly which thread/code is burning CPU.

---

## Step 6: Thread Pool Analysis

Identify thread pools (threads with numbered suffixes sharing a name prefix):
- `http-nio-8080-exec-1` through `http-nio-8080-exec-200` → Tomcat thread pool
- `pool-1-thread-1` through `pool-1-thread-N` → ExecutorService pool
- `ForkJoinPool.commonPool-worker-N` → Fork/Join pool

For each pool, compute:
```
Pool: http-nio-8080-exec (200 threads)
  RUNNABLE:      185  ← near exhaustion — check max pool size
  BLOCKED:        12
  WAITING:         3  ← idle, waiting for work
  TIMED_WAITING:   0
```

**Thread pool exhaustion** (most threads RUNNABLE/BLOCKED, very few WAITING):
- 🔴 Critical: < 5% of pool threads idle → new requests will queue or be rejected
- Root cause: requests taking too long (slow DB, slow downstream API, lock contention)
- Fix: increase pool size (short term), fix slow operations (real fix), add circuit breakers

**Starvation pattern**: all threads WAITING/TIMED_WAITING → pool is idle but application appears slow:
- Usually means work is being submitted but not to this pool
- Or deadlock upstream is preventing work from reaching the pool

---

## Step 7: Common Stack Pattern Detection

Scan all stack traces for these high-value patterns:

### Pattern: Slow DB query blocking request threads
```
"http-nio-8080-exec-N" RUNNABLE
  java.net.SocketInputStream.read(...)
  oracle.jdbc.driver.T4CSocketInputStreamWrapper.read(...)
  oracle.jdbc.driver.T4CMAREngineNIO.prepareForUnmarshalling(...)
```
→ Request threads are waiting on database I/O. Check slow query log, AWR.

### Pattern: HTTP client calls in request threads (outbound)
```
"http-nio-8080-exec-N" RUNNABLE  
  java.net.SocketInputStream.socketRead0(...)
  sun.net.www.http.HttpClient.parseHTTP(...)
  java.net.HttpURLConnection.getResponseCode(...)
```
→ Threads blocked on outbound HTTP. Check downstream service latency. Add timeouts.

### Pattern: Lock held during I/O (dangerous)
```
"pool-thread-1" RUNNABLE
  locked <0x000000076b3a2f40>
  java.io.FileOutputStream.write(...)   ← doing file I/O while holding a lock
```
→ Any thread needing this lock will be BLOCKED for the duration of the file write.

### Pattern: Thread pool exhaustion via sleep/wait loops
```
"exec-thread-N" TIMED_WAITING
  java.lang.Thread.sleep(...)
  com.example.RetryLogic.waitAndRetry(...)
```
→ Threads occupying pool slots while sleeping. Use async retry (`ScheduledExecutorService`).

### Pattern: Infinite loop / tight spin
```
"exec-thread-N" RUNNABLE
  com.example.MessageQueue.poll(...)  ← same frame in every dump
  com.example.Worker.run(...)
```
→ Thread spinning on poll. Should use blocking queue with `take()` instead of busy poll.

### Pattern: Class loader deadlock
```
"Thread-1" BLOCKED
  waiting to lock <0x...> (java.lang.ClassLoader)
"Thread-2" BLOCKED  
  waiting to lock <0x...> (java.lang.ClassLoader)
```
→ Parallel class initialization deadlock. Fix circular static initializers.

---

## Step 8: Findings and Recommendations

For each finding:

```
Finding N — [Severity]
Category:       Deadlock / Lock Contention / CPU / Pool Exhaustion / etc.
Threads:        List of affected thread names/count
Evidence:       Specific lock addresses, stack frames, counts
Root Cause:     What is actually happening
Impact:         Effect on application (hung, slow, CPU spike, etc.)
Recommendations:
  1. Immediate action
  2. Code fix
  3. Monitoring/prevention
```

---

## Step 9: Inline Summary Output

```
## Thread Dump Analysis

**Dump time:** <timestamp if present>  |  **Total threads:** N  |  **Dumps analyzed:** N

### Overall Health: [Critical / Degraded / Fair / Good]
<2–3 sentence summary>

### Thread State Distribution
| State | Count | % | Assessment |
|---|---|---|---|
| RUNNABLE | N | N% | 🔴/🟡/🟢 |
| BLOCKED | N | N% | |
| WAITING | N | N% | |
| TIMED_WAITING | N | N% | |

### Deadlocks: [None Found / N DEADLOCK(S) DETECTED 🔴]

### Key Findings
1. 🔴 [Critical] <title> — <one-line evidence>
2. 🟠 [High] <title> — <one-line evidence>
...

### Top Contended Locks
| Lock Address | Holder Thread | Waiting Threads | Holder Doing |
|---|---|---|---|
```

---

## Step 10: Generate Word Report

Read `/mnt/skills/public/docx/SKILL.md` for document creation.

```bash
npm install -g docx 2>/dev/null | tail -1
```

Report sections:
1. Cover — dump timestamp, JVM info, overall health verdict
2. Thread State Summary — table + commentary
3. Deadlocks — full lock chain diagram (text-based)
4. Lock Contention — table of contended locks, waiting threads, holder analysis
5. CPU Analysis — top RUNNABLE threads with stack summaries
6. Thread Pool Status — per-pool utilization
7. Stack Pattern Findings — grouped by pattern type
8. Findings — severity-ranked with recommendations
9. Monitoring Setup — JMX/JMC monitoring commands to add

```bash
node /home/claude/threadump_report.js
cp /home/claude/threadump_report.docx /mnt/user-data/outputs/threadump_report.docx
```

---

## Reference files

- `references/thread-states.md` — Complete guide to Java thread states, what each means, and diagnostic approaches
- `references/common-patterns.md` — Library of common thread dump anti-patterns with stack signatures and fixes

---

## Key principles

- **BLOCKED is the most urgent state** — these threads are actively failing to make progress right now
- **Multiple dumps reveal motion** — a single dump is a snapshot; three dumps show whether threads are stuck or just slow
- **Lock holder is the root cause** — don't fix the waiting threads; fix what the lock holder is doing
- **Thread names are your map** — well-named threads (Tomcat, Hikari, etc.) immediately tell you which subsystem is impacted
- **RUNNABLE doesn't mean healthy** — RUNNABLE + same stack across dumps = busy loop or genuine CPU work
