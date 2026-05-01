# Thread States & Common Patterns Reference

## Java Thread States

### RUNNABLE
Thread is executing OR ready to execute (scheduled by OS).
- Does NOT mean it's using CPU — could be waiting on I/O at native level
- Stack frames like `socketRead0`, `read`, `write` = I/O wait disguised as RUNNABLE
- Stack frames like `Object.wait` should NOT appear here (would be WAITING)
- If same stack across multiple dumps → stuck or busy loop

### BLOCKED
Thread tried to enter a `synchronized` block/method but another thread holds the monitor.
- Most urgent state — thread is doing NO useful work
- Every BLOCKED thread shows: `waiting to lock <address>`
- Find who holds that address — they are the root cause

### WAITING (indefinite)
Thread called `Object.wait()`, `LockSupport.park()`, `Thread.join()` with no timeout.
- Will stay here until explicitly notified/unparked/joined
- Common in: thread pools waiting for tasks, event loops, producer-consumer queues
- Usually healthy for idle threads; concerning if work threads are here unexpectedly

### TIMED_WAITING
Thread called a timed variant: `Thread.sleep(n)`, `Object.wait(n)`, `LockSupport.parkNanos(n)`
- Will wake up after timeout OR when notified
- Common in: retry logic, scheduled tasks, keepalive pings
- Concerning if request-handling threads are sleeping — wastes pool slots

### TERMINATED
Thread has finished. Should not appear in live dumps unless thread object is referenced.

---

## Common Stack Signatures and Diagnoses

### Database Wait (JDBC)
```
java.net.SocketInputStream.socketRead0(Native Method)
java.net.SocketInputStream.read(SocketInputStream.java)
oracle.jdbc.driver.T4CSocketInputStreamWrapper.read(...)
com.zaxxer.hikari.pool.ProxyPreparedStatement.executeQuery(...)
com.example.dao.OrderRepository.findByCustomer(...)
```
**Diagnosis:** Thread waiting on DB response. If many threads show this → slow queries or DB overload.
**Fix:** Tune slow queries, add indexes, increase connection pool if pool is the bottleneck.

### Connection Pool Wait (Hikari/C3P0/DBCP)
```
com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:213)
com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:128)
com.example.service.OrderService.processOrder(...)
```
**Diagnosis:** All connections in use; threads queuing for one. Pool exhausted.
**Fix:** Increase `maximumPoolSize`, fix slow queries holding connections too long, add `connectionTimeout`.

### Outbound HTTP (RestTemplate / HttpClient / OkHttp)
```
java.net.SocketInputStream.socketRead0(Native Method)
sun.net.www.http.HttpClient.parseHTTP(HttpClient.java:704)
java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:480)
org.springframework.http.client.SimpleClientHttpResponse.getRawStatusCode(...)
com.example.client.PaymentClient.charge(...)
```
**Diagnosis:** Thread blocked waiting for external HTTP response. No timeout = thread stuck indefinitely.
**Fix:** Set `connectTimeout` and `readTimeout` on all HTTP clients. Use async client (WebClient).

### Synchronized Cache Contention
```
"pool-thread-12" BLOCKED
  waiting to lock <0x000000076b3a2f40> (com.example.cache.LocalCache)
"pool-thread-1" RUNNABLE
  locked <0x000000076b3a2f40>
  com.example.cache.LocalCache.get(LocalCache.java:45)
```
**Diagnosis:** Single synchronized lock on a shared cache. Every thread serializes.
**Fix:** Replace with `ConcurrentHashMap`, `Caffeine`, or read-write lock (`ReadWriteLock`).

### GC Pressure (GC threads dominating)
```
"GC task thread#0 (ParallelGC)" RUNNABLE
"GC task thread#1 (ParallelGC)" RUNNABLE
...
"VM Thread" RUNNABLE  ← stop-the-world GC
```
**Diagnosis:** Excessive GC. Application threads may be paused.
**Fix:** Analyze heap dump, increase heap (`-Xmx`), tune GC algorithm, fix memory leaks.

### Deadlock
```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f... (object 0x..., a java.util.HashMap),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f... (object 0x..., a java.lang.String),
  which is held by "Thread-1"
```
**Fix:** Enforce single lock acquisition order. Use `tryLock()` with timeout. Use lock-free structures.

### Thread Sleeping in Request Handler
```
"http-nio-8080-exec-15" TIMED_WAITING
  java.lang.Thread.sleep(Native Method)
  com.example.service.RetryService.waitAndRetry(RetryService.java:67)
  com.example.service.PaymentService.processWithRetry(PaymentService.java:89)
```
**Diagnosis:** Request thread sleeping during retry — holds a Tomcat thread slot doing nothing.
**Fix:** Use `ScheduledExecutorService` for retry scheduling. Release thread during wait.

### Busy Poll / Spin Loop
```
"worker-thread-3" RUNNABLE  ← same in dump 1
"worker-thread-3" RUNNABLE  ← same in dump 2 (10s later)
  com.example.queue.MessagePoller.poll(MessagePoller.java:34)  ← same frame
  com.example.queue.MessagePoller.run(MessagePoller.java:22)
```
**Diagnosis:** Thread spinning on non-blocking poll. Burns CPU even when no work available.
**Fix:** Use `BlockingQueue.take()` or `LinkedBlockingQueue` — blocks until work arrives.

### ClassLoader Deadlock
```
"Thread-A" BLOCKED
  waiting to lock <0x...> (a java.lang.Class for com.example.A)
  com.example.B.<clinit>(B.java:10)  ← static initializer of B loading A
"Thread-B" BLOCKED
  waiting to lock <0x...> (a java.lang.Class for com.example.B)
  com.example.A.<clinit>(A.java:15)  ← static initializer of A loading B
```
**Diagnosis:** Circular static initialization. A's clinit triggers loading B, B's clinit triggers loading A.
**Fix:** Break circular dependency. Lazy initialization instead of static initializers.

### Thread Pool Exhaustion
```
All "http-nio-8080-exec-*" threads (200 total) are RUNNABLE or BLOCKED
Only 1-2 are WAITING (idle)
```
**Diagnosis:** Tomcat thread pool near exhaustion. New requests will queue.
**Fix:** Find what's making requests slow (DB, HTTP calls, locks). Increase pool as temporary relief.

### Finalization Queue Buildup
```
"Finalizer" RUNNABLE
  java.lang.ref.Finalizer.runFinalizer(Finalizer.java:101)
  java.lang.ref.Finalizer.access$100(Finalizer.java:32)
```
**Diagnosis:** Objects with finalizers are slow to be collected. If Finalizer thread is always RUNNABLE, finalizer queue is backed up.
**Fix:** Avoid finalizers. Use `try-with-resources`, `Cleaner` API, or explicit close().
