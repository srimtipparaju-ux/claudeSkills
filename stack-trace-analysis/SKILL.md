---
name: stack-trace-analysis
description: >
  Use this skill whenever a user pastes or uploads a stack trace, exception, crash
  report, or error log and wants it diagnosed. Triggers include: any mention of
  'stack trace', 'exception', 'error log', 'crash', 'NullPointerException', 'NPE',
  'StackOverflowError', 'ClassNotFoundException', 'NoSuchMethodError', 'caused by',
  'at com.', 'at java.', 'Traceback (most recent call last)', 'segfault', 'core dump',
  or when the user pastes a block of text starting with an exception class name and
  indented 'at' lines. Also trigger for Python tracebacks, Node.js error stacks,
  .NET stack traces, or any language's exception output. Always use this skill —
  it ensures the root cause (innermost 'Caused by') is found and not just the
  surface exception.
---

# Stack Trace Analysis Skill

## What this skill does

Analyzes stack traces from any language/runtime to:
1. Identify the root cause exception (not just the surface error)
2. Pinpoint the exact failing line in application code
3. Explain what the error means in plain language
4. Diagnose why it happened based on the call chain context
5. Provide specific, actionable fixes
6. Detect patterns in repeated errors (if multiple traces provided)

---

## Step 1: Read the Input

Stack traces are almost always pasted inline. If a file is uploaded:
```bash
cat /mnt/user-data/uploads/<file>        # if small
head -200 /mnt/user-data/uploads/<file>  # if large log file
grep -n "Exception\|Error\|Caused by\|Traceback" /mnt/user-data/uploads/<file> | head -50
```

If the file is a large log with many errors, extract distinct stack traces:
```bash
# Find all exception starting points
grep -n "^[A-Za-z].*Exception\|^[A-Za-z].*Error:" /mnt/user-data/uploads/<file> | head -30
```

---

## Step 2: Identify the Language and Runtime

| Indicator | Language/Runtime |
|---|---|
| `at com.example.Class.method(File.java:123)` | Java / JVM |
| `at Object.<anonymous> (file.js:45:12)` | Node.js / JavaScript |
| `File "script.py", line 45, in <module>` | Python |
| `at System.String.Format (...)` | .NET / C# |
| `#0 0x00007f... in function_name()` | C/C++ (GDB) |
| `goroutine N [running]:` | Go |
| `thread 'main' panicked at 'message'` | Rust |
| `RubyError (message): file.rb:45` | Ruby |
| `PHP Fatal error: ... in /path/file.php on line 45` | PHP |

---

## Step 3: Parse the Exception Chain

Most JVM and .NET exceptions wrap each other in a chain. Always find the **innermost** "Caused by" — that is the root cause.

```
java.lang.RuntimeException: Failed to process order          ← surface (symptom)
    at com.example.OrderService.process(OrderService.java:89)
    at com.example.OrderController.submitOrder(OrderController.java:45)
Caused by: java.sql.SQLException: Connection timeout          ← real cause
    at oracle.jdbc.driver.T4CConnection.logon(T4CConnection.java:740)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:213)
    ... 8 more
Caused by: java.net.SocketTimeoutException: connect timed out  ← root cause
    at java.net.PlainSocketImpl.socketConnect(Native Method)
```

**Always start analysis from the innermost Caused by, not the top.**

In this example: "Failed to process order" is a symptom. "Connection timeout" is the mechanism. "SocketTimeoutException: connect timed out" is the root — the database host is unreachable.

---

## Step 4: Pinpoint Application Code Lines

In the stack trace, distinguish:
- **Application frames**: `at com.example.*`, `at org.yourcompany.*` — YOUR code, where the fix lives
- **Framework frames**: `at org.springframework.*`, `at com.zaxxer.*`, `at org.hibernate.*` — library code, usually not the bug location but gives context
- **JDK frames**: `at java.util.*`, `at java.net.*`, `at sun.*` — stdlib, tells you WHAT failed
- **Native frames**: `Native Method` — OS-level call

**The most important application frame** is the one closest to the root cause exception. That's where to look for the fix.

---

## Step 5: Exception Dictionary

### Java / JVM

**NullPointerException (NPE)**
```
java.lang.NullPointerException: Cannot invoke "String.length()" because "str" is null
    at com.example.TextUtil.process(TextUtil.java:23)
```
- An object was `null` when a method was called on it or a field was accessed
- Java 14+ helpful NPE messages tell you exactly which variable was null
- **Diagnosis**: find line 23 in TextUtil.java — what variable is null there? Trace back where it comes from.
- **Fixes**: add null check, use `Optional<T>`, fix upstream code that should return non-null, use `Objects.requireNonNull()` to fail fast

**ClassNotFoundException / NoClassDefFoundError**
```
java.lang.ClassNotFoundException: org.postgresql.Driver
```
- A class is missing from the classpath at runtime
- **ClassNotFoundException**: class not found when explicitly loaded (e.g., `Class.forName()`)
- **NoClassDefFoundError**: class was present at compile time but missing at runtime
- **Fixes**: add missing dependency to `pom.xml`/`build.gradle`, check dependency scope (not `provided` when needed), check fat JAR assembly

**NoSuchMethodError / NoSuchMethodException**
```
java.lang.NoSuchMethodError: com.example.api.Client.getData(Ljava/lang/String;)Ljava/util/List;
```
- Method exists at compile time but not at runtime — usually a JAR version conflict
- **Diagnosis**: multiple versions of the same library on classpath; one version compiled against, different version loaded
- **Fixes**: check dependency tree (`mvn dependency:tree`), enforce consistent versions, exclude conflicting transitive dependencies

**StackOverflowError**
```
java.lang.StackOverflowError
    at com.example.TreeNode.traverse(TreeNode.java:15)
    at com.example.TreeNode.traverse(TreeNode.java:18)
    at com.example.TreeNode.traverse(TreeNode.java:18)  [repeated 1024 times]
```
- Infinite (or too deep) recursion
- **Diagnosis**: find the repeating frames — that's the recursive method. What's the base case? Is it missing or unreachable?
- **Fixes**: add/fix base case, convert recursion to iteration with explicit stack, increase stack size (`-Xss`) as last resort

**OutOfMemoryError: Java heap space**
```
java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:3210)
    at java.util.ArrayList.grow(ArrayList.java:265)
    at com.example.ReportService.generateReport(ReportService.java:156)
```
- Heap is full. The stack frames show what was being allocated when OOM hit.
- **Diagnosis**: check what generateReport is doing at line 156 — loading too much data? Unbounded list?
- **Fixes**: add pagination, stream large datasets, fix memory leak (see heap dump analysis), increase `-Xmx` as temporary measure

**OutOfMemoryError: Metaspace**
```
java.lang.OutOfMemoryError: Metaspace
```
- Class metadata space full. Usually from dynamic class generation or classloader leak.
- **Fixes**: check for Groovy/CGLib/ByteBuddy dynamic class generation, fix classloader leaks in hot-deploy, increase `-XX:MaxMetaspaceSize`

**ConcurrentModificationException**
```
java.util.ConcurrentModificationException
    at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:911)
    at com.example.OrderProcessor.process(OrderProcessor.java:67)
```
- Collection modified while being iterated
- **Fixes**: use `Iterator.remove()` instead of `Collection.remove()` during iteration, collect items to remove then remove after iteration, use `CopyOnWriteArrayList` for concurrent access, use streams with `filter()`

**ClassCastException**
```
java.lang.ClassCastException: class java.lang.String cannot be cast to class java.lang.Integer
    at com.example.ConfigLoader.getIntValue(ConfigLoader.java:34)
```
- Object cast to incompatible type. Generics erasure or raw types often the cause.
- **Fixes**: check what is actually stored at the source, add type check before cast (`instanceof`), fix the storage code to store correct type

**IllegalArgumentException / IllegalStateException**
- Argument is invalid for the method: fix the calling code to validate before calling
- State is wrong for the operation: fix the order of operations or add state check

**SQLException**
```
java.sql.SQLException: ORA-01017: invalid username/password
java.sql.SQLException: ORA-00942: table or view does not exist
java.sql.SQLIntegrityConstraintViolationException: ORA-00001: unique constraint violated
```
- Database-level error wrapped in Java exception
- **Diagnosis**: the ORA- / SQL error code tells you exactly what happened (see Oracle error codes)
- Common: connection pool exhausted, wrong credentials, missing table (wrong schema/env), constraint violation (duplicate data)

**HibernateException / JPA LazyInitializationException**
```
org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role:
  com.example.Order.items, could not initialize proxy - no Session
```
- Accessing a lazy-loaded association after the JPA session/EntityManager is closed
- **Fixes**: use `EAGER` fetch (careful of N+1), use JOIN FETCH in query, open session in view pattern (anti-pattern), ensure access is within `@Transactional` boundary

**Spring / Framework Exceptions**
```
org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'com.example.Service'
```
- Missing Spring bean: not annotated, wrong package scan, conditional not met
- **Fixes**: add `@Service`/`@Component`, fix `@ComponentScan` base package, check `@Conditional` or `@Profile`

```
org.springframework.web.HttpMediaTypeNotSupportedException: Content type 'application/x-www-form-urlencoded' not supported
```
- Controller expects JSON but received form data (or vice versa)
- **Fixes**: add correct `Content-Type: application/json` header in client, or add `consumes` attribute to `@RequestMapping`

---

### Python Tracebacks

```
Traceback (most recent call last):
  File "app.py", line 45, in process_order
    result = db.query(sql, params)
  File "db.py", line 23, in query
    cursor.execute(sql, params)
psycopg2.OperationalError: could not connect to server: Connection refused
```
- Read bottom-up: last frame is where the error occurred, first frame is the entry point
- Root cause is the bottom exception type + message

Common Python exceptions:
| Exception | Meaning | Fix |
|---|---|---|
| `AttributeError: 'NoneType' object has no attribute 'x'` | Object is None when attribute accessed | Check for None before access, fix upstream that returns None |
| `KeyError: 'field_name'` | Dict key doesn't exist | Use `.get('key', default)` or check with `in` |
| `IndexError: list index out of range` | List access beyond bounds | Check length before indexing |
| `TypeError: unsupported operand type(s)` | Wrong types in operation | Check types; string + int, None + string, etc. |
| `ImportError: No module named 'X'` | Package not installed or wrong env | `pip install X`, check virtual env |
| `RecursionError: maximum recursion depth exceeded` | Infinite recursion | Same as Java StackOverflow |
| `JSONDecodeError` | Invalid JSON string | Validate JSON input, add try/except |
| `UnicodeDecodeError` | Wrong encoding | Use `errors='replace'` or `errors='ignore'`, fix encoding |

---

### Node.js / JavaScript

```
TypeError: Cannot read properties of undefined (reading 'id')
    at processOrder (/app/order.js:45:23)
    at async OrderController.submit (/app/controller.js:23:5)
```
- Read top-down: first frame is where error occurred
- Async/await stacks: look for the async function that rejected

Common Node.js errors:
| Error | Fix |
|---|---|
| `Cannot read properties of undefined/null` | Optional chaining `?.`, null check |
| `UnhandledPromiseRejectionWarning` | Add `.catch()` or `try/catch` in async |
| `ECONNREFUSED` | Target service not running, wrong port |
| `ETIMEDOUT` | Network timeout — add timeout option |
| `ENOENT: no such file or directory` | File path wrong or file doesn't exist |
| `MODULE_NOT_FOUND` | npm package not installed or import path wrong |
| `Maximum call stack size exceeded` | Infinite recursion |

---

### .NET / C#

```
System.NullReferenceException: Object reference not set to an instance of an object.
   at OrderService.ProcessOrder(Order order) in /app/OrderService.cs:line 45
```
- Same pattern as Java NPE
- Enable nullable reference types (`#nullable enable`) to catch at compile time

| Exception | Fix |
|---|---|
| `NullReferenceException` | Null check, `?.` operator, `??` default |
| `InvalidOperationException: Sequence contains no elements` | LINQ `.First()` on empty — use `.FirstOrDefault()` |
| `ArgumentNullException` | Caller passed null — validate inputs |
| `StackOverflowException` | Infinite recursion |
| `OutOfMemoryException` | Memory leak or processing too large a dataset |
| `DbUpdateConcurrencyException` | EF Core optimistic concurrency conflict |

---

## Step 6: Multi-Error Log Analysis

If many stack traces are provided (log file), find patterns:

```python
from collections import Counter
import re

with open('/mnt/user-data/uploads/<file>') as f:
    content = f.read()

# Extract exception types
exceptions = re.findall(r'([\w\.]+(?:Exception|Error)): (.+)', content)
counter = Counter(f"{e[0]}: {e[1][:60]}" for e in exceptions)

print("Most frequent errors:")
for err, count in counter.most_common(10):
    print(f"  {count:4d}×  {err}")

# Extract timestamps if present
timestamps = re.findall(r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})', content)
if timestamps:
    print(f"\nFirst error: {timestamps[0]}")
    print(f"Last error:  {timestamps[-1]}")
    print(f"Total time span: {len(set(timestamps))} unique timestamps")
```

Look for:
- **Spike patterns**: errors concentrated in a time window → correlate with deployment, traffic spike, scheduled job
- **Cascading errors**: one root cause producing many different exceptions (e.g., DB connection failure → NPE in result handling → generic RuntimeException logged)
- **Intermittent vs persistent**: is the error every request, or occasional?

---

## Step 7: Findings and Recommendations

```
Finding N — [Severity] — [Exception Type]
Location:       <ClassName.method():line>
Root cause:     <what is actually failing and why>
Frequency:      <once / intermittent / every request>
User impact:    <request fails / data loss / service down / etc.>
Recommendations:
  1. Immediate fix: <specific code change>
  2. Prevention: <how to prevent recurrence>
  3. Monitoring: <what to alert on>
```

**Severity:**
- 🔴 Critical: Service down, data corruption, security issue, OOM
- 🟠 High: Requests failing for users, recurring error, resource leak
- 🟡 Medium: Handled exception with poor UX, occasional failure
- 🟢 Low: Logged but gracefully handled, or test environment only

---

## Step 8: Inline Summary

```
## Stack Trace Analysis

**Language/Runtime:** Java 17 / Spring Boot 3.1
**Exception count:** N (if log file) | **Time span:** <if timestamps present>

### Root Cause
<Exception type>: <message>
<Plain English explanation of what this means>

### Where it fails
<ClassName.method():line> — <what this code is trying to do>

### Why it happened
<Explanation of the cause chain from innermost Caused By upward>

### Findings
1. 🔴 [Critical] ...

### Fix
<Most important immediate action>
```

---

## Step 9: Generate Word Report

Read `/mnt/skills/public/docx/SKILL.md`.

Report sections: Cover, Exception Summary, Root Cause Analysis (full chain), Application Code Hotspot, Error Frequency Analysis (if log file), Findings, Code Fix Examples.

```bash
node /home/claude/stacktrace_report.js
cp /home/claude/stacktrace_report.docx /mnt/user-data/outputs/stacktrace_report.docx
```

---

## Key principles

- **Always find the innermost "Caused by"** — the top exception is almost always a wrapper; the real error is at the bottom of the chain
- **Find YOUR code in the stack** — the closest application frame to the root cause is where the fix lives; framework frames give context, not fixes
- **Exception message + type together tell the story** — `NullPointerException` alone tells you little; `NullPointerException: Cannot invoke "Order.getId()" because "order" is null at OrderService.java:45` tells you exactly what to fix
- **Frequency and timing matter** — a rare NPE on a specific input is different from an NPE on every request
- **Cascading exceptions hide the root** — when you see 5 different exception types in a log spike, find the first one and fix that; the rest are symptoms
