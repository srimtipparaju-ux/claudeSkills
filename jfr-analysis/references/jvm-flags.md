# JVM Performance Flags Reference

## Heap & Memory

| Flag | Purpose | Recommendation |
|---|---|---|
| `-Xms<size>` | Initial heap size | Set equal to -Xmx to avoid heap resizing pauses |
| `-Xmx<size>` | Maximum heap size | Leave 25% of system RAM for OS/other processes |
| `-Xss<size>` | Thread stack size | Default 512k-1MB; reduce if running thousands of threads |
| `-XX:MetaspaceSize=<size>` | Initial Metaspace size | Set to expected class metadata size to avoid early GC |
| `-XX:MaxMetaspaceSize=<size>` | Metaspace cap | Set to prevent runaway class loading; default is unlimited |
| `-XX:MaxDirectMemorySize=<size>` | Off-heap NIO buffer limit | Set if using Netty/NIO heavily |
| `-XX:+AlwaysPreTouch` | Pre-allocate heap pages at startup | Eliminates page fault pauses during runtime; good for servers |

## GC Algorithm Selection (Java 11+)

| Flag | GC Algorithm | Best for |
|---|---|---|
| `-XX:+UseG1GC` | G1 (default Java 9+) | General purpose; good latency + throughput balance |
| `-XX:+UseZGC` | ZGC | Ultra-low latency (< 1ms pauses); Java 15+ for production |
| `-XX:+UseShenandoahGC` | Shenandoah | Low latency; RedHat OpenJDK |
| `-XX:+UseParallelGC` | Parallel GC | Maximum throughput; tolerates longer pauses (batch jobs) |
| `-XX:+UseSerialGC` | Serial GC | Single-core / very small heaps only |

## G1GC Tuning

| Flag | Purpose | Default | When to change |
|---|---|---|---|
| `-XX:MaxGCPauseMillis=<ms>` | Pause time target | 200ms | Lower for latency-sensitive (50–100ms) |
| `-XX:G1HeapRegionSize=<size>` | G1 region size (1-32MB, power of 2) | Auto | Increase if many humongous allocations (objects > region/2) |
| `-XX:G1NewSizePercent=<pct>` | Min young gen % | 5 | Increase if young GC too frequent |
| `-XX:G1MaxNewSizePercent=<pct>` | Max young gen % | 60 | Increase if allocation rate very high |
| `-XX:InitiatingHeapOccupancyPercent=<pct>` | Start concurrent GC when heap is X% full | 45 | Decrease if full GCs are occurring (start GC earlier) |
| `-XX:G1ReservePercent=<pct>` | Reserve heap % for promotion | 10 | Increase if promotion failures occur |
| `-XX:ConcGCThreads=<n>` | Concurrent GC threads | auto | Increase if concurrent GC not finishing fast enough |

## ZGC Tuning (Java 15+)

| Flag | Purpose |
|---|---|
| `-XX:SoftMaxHeapSize=<size>` | Soft target for ZGC to try to stay under |
| `-XX:ZCollectionInterval=<sec>` | Force GC interval even if not needed (keeps heap lean) |
| `-XX:ZUncommit` | Return unused heap to OS (default on) |

## JIT Compiler

| Flag | Purpose | Recommendation |
|---|---|---|
| `-XX:CICompilerCount=<n>` | JIT compiler thread count | Default: 2-4; increase for large apps with many hot methods |
| `-XX:+TieredCompilation` | Enable tiered compilation (C1+C2) | Default on; rarely disable |
| `-XX:CompileThreshold=<n>` | Invocations before JIT compiles | Default 10,000; lower to JIT sooner (trade startup for throughput) |
| `-XX:+PrintCompilation` | Log JIT compilation events | Debug only — very verbose |
| `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` | Log inlining decisions | Debug only |

## Thread Tuning

| Flag | Purpose | Recommendation |
|---|---|---|
| `-XX:+UseVirtualThreads` | Enable virtual threads (Java 21+) | Use for I/O-heavy workloads; eliminates thread-per-request limits |
| `-Djava.util.concurrent.ForkJoinPool.common.parallelism=<n>` | Common ForkJoin pool size | Default = CPU cores; reduce if competing with other thread pools |

## GC Logging (Essential for Production)

```bash
# Java 11+ unified logging
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=20m

# Include safepoint info (shows stop-the-world beyond GC)
-Xlog:gc*,safepoint:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=20m
```

## JFR Continuous Recording (Low Overhead — Safe for Production)

```bash
# Start JFR on application startup (add to JVM args)
-XX:StartFlightRecording=duration=0,filename=/var/log/app/recording.jfr,settings=profile,maxsize=500m,maxage=1h

# Or start at runtime via jcmd
jcmd <pid> JFR.start name=prod duration=1h filename=/tmp/recording.jfr settings=profile

# Dump current recording without stopping it
jcmd <pid> JFR.dump name=prod filename=/tmp/snapshot.jfr

# Stop recording
jcmd <pid> JFR.stop name=prod
```

## Diagnostic Flags (Add for Troubleshooting)

```bash
# OOM heap dump (essential — always add in production)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heapdumps/

# Exit JVM on OOM (let orchestration restart it)
-XX:+ExitOnOutOfMemoryError

# Log GC details (Java 11+)
-Xlog:gc*:file=/var/log/gc.log:time:filecount=5,filesize=20m

# Native memory tracking
-XX:NativeMemoryTracking=summary
# Then: jcmd <pid> VM.native_memory summary

# Show JVM version and flags on startup
-XX:+PrintCommandLineFlags
-XX:+PrintVMOptions
```

## Common JVM Tuning Starting Points

### Latency-sensitive service (API, microservice)
```bash
-Xms4g -Xmx4g
-XX:+UseZGC                          # or G1GC with MaxGCPauseMillis=50
-XX:MaxGCPauseMillis=50
-XX:+AlwaysPreTouch
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps/
-XX:StartFlightRecording=duration=0,filename=/jfr/recording.jfr,maxage=2h,maxsize=1g,settings=profile
```

### Throughput-focused service (batch, ETL)
```bash
-Xms8g -Xmx8g
-XX:+UseParallelGC
-XX:ParallelGCThreads=8
-XX:+AlwaysPreTouch
-XX:+HeapDumpOnOutOfMemoryError
```

### Memory-constrained container
```bash
-XX:+UseContainerSupport             # Respect cgroup memory limits (default Java 10+)
-XX:MaxRAMPercentage=75.0            # Use 75% of container RAM for heap
-XX:+UseZGC
-XX:+HeapDumpOnOutOfMemoryError
```
