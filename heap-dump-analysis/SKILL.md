---
name: heap-dump-analysis
description: >
  Use this skill whenever a user uploads or references a Java heap dump and wants it
  analyzed for memory issues. Triggers include: any mention of 'heap dump', 'hprof',
  'OutOfMemoryError', 'OOM', 'memory leak', 'heap space', 'GC overhead', 'MAT',
  'Eclipse Memory Analyzer', 'jmap', 'retained heap', 'shallow heap', 'dominator tree',
  or when the user uploads a .hprof file or a heap analysis report (HTML/txt from MAT
  or VisualVM). Also trigger when the user asks why a Java application is consuming
  too much memory, crashing with OOM, or experiencing frequent garbage collection.
  Always use this skill — heap analysis has a specific methodology that prevents
  common misdiagnoses.
---

# Heap Dump Analysis Skill

## What this skill does

Analyzes Java heap dump files or heap analysis reports to:
1. Identify the largest memory consumers (by retained heap)
2. Find memory leaks — objects accumulating without being collected
3. Diagnose the root cause of OutOfMemoryErrors
4. Identify GC pressure patterns and object lifecycle problems
5. Recommend fixes with code-level specificity
6. Produce a downloadable Word report

---

## Step 1: Read the Input

Heap dumps come in several forms:

### `.hprof` file (raw heap dump)
These are binary and can be gigabytes. Cannot be read directly — user must provide an analysis report from MAT, VisualVM, JMC, or similar.

If raw `.hprof` is uploaded, tell the user:
> "This is a raw heap dump binary. To analyze it, open it in Eclipse Memory Analyzer (MAT — free download) or VisualVM, then export the 'Leak Suspects Report' or 'Dominator Tree' as HTML/txt and share that here."
>
> Provide these MAT steps:
> 1. File → Open Heap Dump → select .hprof
> 2. Run: Reports → Leak Suspects
> 3. Also run: Window → Heap Dump Details → Dominator Tree
> 4. Export both as HTML or copy the text output

### HTML/TXT report from MAT or VisualVM
```bash
wc -c /mnt/user-data/uploads/<file>
# If HTML, extract text
python3 -c "
from html.parser import HTMLParser
class E(HTMLParser):
    def __init__(self): super().__init__(); self.t=[]
    def handle_data(self,d):
        if d.strip(): self.t.append(d.strip())
e=E()
e.feed(open('/mnt/user-data/uploads/<file>','r',errors='replace').read())
print('\n'.join(e.t))
" | head -500
```

### Inline pasted output (OOM stack trace + some stats)
Work directly from the pasted text.

---

## Step 2: Extract Top-Level Heap Metrics

From the report header, capture:

| Metric | Meaning |
|---|---|
| Total heap size | Max configured (`-Xmx`) |
| Used heap at dump time | How full the heap was |
| % used | Used / total — > 90% at dump = OOM likely |
| GC roots count | Entry points keeping objects alive |
| Total objects | Total live object count |
| Dump trigger | OOM? Manual? GC event? |
| JVM version / GC algorithm | Context for analysis |

---

## Step 3: Dominator Tree Analysis

The dominator tree shows which objects "own" the most memory — an object A dominates object B if every path from GC roots to B goes through A. The retained heap of A = everything that would be freed if A were collected.

For the top 20 entries in the dominator tree:

| Object / Class | Shallow Heap | Retained Heap | % of Total | Owner/Path |
|---|---|---|---|---|
| `byte[]` | 32 | 1.4GB | 68% | Held by `byte[] cache in com.example.ImageCache` |

**Key concepts:**
- **Shallow heap**: memory of the object itself (not what it references)
- **Retained heap**: total memory freed if this object is GC'd (includes all reachable objects only via this node)
- **Leak indicator**: any single object or object tree holding > 20% of heap is suspicious

For each large dominator:
1. What class is it? Is it a collection (Map, List, byte[]) or a domain object?
2. Who holds a reference to it? (follow GC root path)
3. Should it be this large? Is it a cache, session store, queue?
4. Is it growing? (compare across multiple dumps if available)

---

## Step 4: Class Histogram Analysis

The class histogram shows total instance count and shallow bytes per class.

Look for:
- Classes with unexpectedly high instance counts (e.g., 5 million `String` objects)
- Large `byte[]` or `char[]` arrays — usually holding data payloads
- Large numbers of domain objects (e.g., 2M `Order` objects in memory)
- Wrapper/duplicate objects (many small `Integer`, `Long` — consider primitive arrays)

**High instance count patterns:**

| Class | Likely cause |
|---|---|
| `byte[]` / `char[]` | Large strings, cached data, network buffers |
| `String` | String interning overuse, large request/response data |
| `HashMap$Entry` / `LinkedHashMap$Entry` | Large maps / caches not bounded |
| `Object[]` | Large lists, unbounded queues |
| Domain objects (e.g., `Order`, `User`) | ORM L1/L2 cache, session data, unbounded result sets |
| `Class` | Classloader leak — classes not unloaded |
| `java.lang.ref.Finalizer` | Finalizer queue backup |

---

## Step 5: Leak Suspects Analysis

MAT's Leak Suspects report highlights the most likely leaks. For each suspect:

1. **Accumulation point**: what object is holding the leak? (usually a collection or cache)
2. **Reference chain**: GC root → ... → collection → leaked objects
3. **Why not GC'd**: what is keeping the accumulation point alive?

### Common leak patterns:

#### Cache without eviction
```
com.example.cache.LocalCache$data (HashMap)
  → 2,847,293 entries
  → Each entry holds Customer object + associated data
  → No expiry, no size limit
```
**Fix:** Use bounded cache (Caffeine with `maximumSize` + `expireAfterWrite`). Never use unbounded `HashMap` as cache.

#### Static collection accumulating over time
```
com.example.registry.ServiceRegistry$instances (static ArrayList)
  → 4,200,000 ServiceProxy objects
  → Static reference keeps alive forever
```
**Fix:** Remove from list when no longer needed. Use `WeakReference` if lifecycle is unclear.

#### HTTP Session / JPA EntityManager not closed
```
org.springframework.orm.jpa.EntityManagerHolder
  → EntityManager holding 100,000 managed entities
  → Session scope but EntityManager not closed
```
**Fix:** Use `@Transactional` properly to scope EntityManager. Clear L1 cache with `entityManager.clear()` in batch operations.

#### Classloader leak (common in hot-deploy / webapp reload)
```
sun.misc.Launcher$AppClassLoader
  → holds references to 12,450 Class objects
  → Previous webapp deployment not GC'd
```
**Cause:** Thread locals, static references, or driver registrations keeping old classloader alive.
**Fix:** Deregister JDBC drivers on webapp shutdown. Clear thread locals. Use weak references for classloader-scoped objects.

#### ThreadLocal leak
```
java.lang.ThreadLocal$ThreadLocalMap$Entry
  → many entries with key=null (cleared) but value still referenced
  → Values: com.example.RequestContext objects (1.2MB each)
```
**Fix:** Always call `threadLocal.remove()` in finally block after use. Use `InheritableThreadLocal` carefully.

#### Event listener / callback not unregistered
```
com.example.EventBus$listeners (CopyOnWriteArrayList)
  → 280,000 listener objects
  → Listeners registered but never removed
  → Each listener holds reference to a View/Controller
```
**Fix:** Unregister listeners in component lifecycle destroy/close methods.

#### Unbounded queue
```
java.util.concurrent.LinkedBlockingQueue
  → 890,000 Runnable tasks
  → Producer faster than consumer
```
**Fix:** Use bounded `ArrayBlockingQueue`. Implement backpressure. Monitor queue depth.

---

## Step 6: GC Root Path Analysis

For any suspected leak, trace the path from GC root to the leaked object:

```
GC Root: Thread "main" → static field
  → com.example.App.cache (HashMap, retained: 1.2GB)
    → HashMap$Entry[0]
      → key: "customer:12345" (String)
      → value: com.example.CustomerProfile (2.4KB)
        → List<Order> (450 Orders, avg 500 bytes each = 225KB)
          → ...
```

The GC root type tells you WHY it can't be collected:
- **Static field**: lives forever unless class is unloaded
- **Thread**: lives as long as the thread (thread locals especially)
- **JNI reference**: native code holding reference
- **System class**: class loaded by bootstrap classloader

---

## Step 7: OOM Error Analysis

If an OOM error is provided, classify it:

| OOM Message | Meaning | Fix |
|---|---|---|
| `Java heap space` | Heap full — objects can't be allocated | Find leak or increase -Xmx |
| `GC overhead limit exceeded` | > 98% time in GC, < 2% productive work | Severe leak or heap too small |
| `Metaspace` | Class metadata space full | Class loader leak, too many dynamically generated classes |
| `Direct buffer memory` | Off-heap NIO buffers full | Check `ByteBuffer.allocateDirect()` usage, increase `-XX:MaxDirectMemorySize` |
| `Unable to create new native thread` | OS can't create thread | Too many threads, increase OS limits (`ulimit -u`) |
| `request {} bytes for {}. Out of swap space?` | OS-level OOM | System-level memory pressure, Java heap + native competing |
| `Compressed class space` | Compressed class pointer space full | Use `-XX:CompressedClassSpaceSize` to increase |

---

## Step 8: Findings and Recommendations

```
Finding N — [Severity] — [Category]
Object/Class:   <what is leaking>
Size:           <retained heap in MB/GB, % of total>
Root cause:     <what is holding the reference and why>
Growth rate:    <if multiple dumps: X MB/min>
Impact:         <OOM risk, GC pressure, latency>
Recommendations:
  1. Immediate: <what to do right now>
  2. Code fix: <specific code change>
  3. Monitoring: <what metric to watch>
```

---

## Step 9: Inline Summary

```
## Heap Dump Analysis

**JVM:** <version> | **Heap configured:** Xmx=<N>GB | **Used at dump:** <N>GB (<N>%) | **Trigger:** <OOM/manual>

### Overall Assessment: [Critical — OOM Imminent / High GC Pressure / Memory Leak Detected / Healthy]
<2–3 sentence summary>

### Heap Composition
| Category | Size | % of Heap |
|---|---|---|
| Top object type | X GB | X% |
| ... | | |

### Top Memory Consumers (Dominator Tree)
| Rank | Object | Retained Heap | % | Suspected Role |
|---|---|---|---|---|

### Findings
1. 🔴 [Critical] <title> — <evidence>
```

---

## Step 10: Generate Word Report

Read `/mnt/skills/public/docx/SKILL.md`.

Report sections: Cover, Heap Metrics, Dominator Tree (table), Class Histogram highlights, Leak Suspects (one section per), GC Root Paths, OOM Classification, Findings with recommendations, Monitoring setup.

```bash
node /home/claude/heap_report.js
cp /home/claude/heap_report.docx /mnt/user-data/outputs/heap_report.docx
```

---

## Key principles

- **Retained heap is the real number** — shallow heap alone is misleading; a 48-byte HashMap object can dominate 2GB
- **Follow the root** — fixing the leak means breaking the reference chain to the GC root, not just clearing the collection
- **Unbounded collections are almost always the culprit** — caches, queues, session stores, listener lists
- **One dump is a snapshot; two dumps show growth** — if user can provide two dumps 5 minutes apart, calculate growth rate to estimate time to OOM
- **Metaspace OOM is different** — it's about class loading, not heap objects; usually a hot-deploy or codegen problem
