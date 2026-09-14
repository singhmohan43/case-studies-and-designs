# Rails Concurrency & Worker Design

## 1. Problem

A Rails application may need to handle many requests concurrently while keeping CPU, memory, and database usage under control.

As traffic grows, simply adding more workers or threads does not always improve performance.

The application needs to balance:

* Request concurrency
* CPU usage
* Memory usage
* Database connections
* External service limits
* Application server capacity

This design explains the relationship between Rails processes, threads, workers, and database connection pools.

The examples are generalized and do not represent any specific production system.

---

## 2. Process vs Thread

A process is an independent running instance of the application.

A thread is an execution path inside a process.

Conceptually:

```text
Process
 |
 +-- Thread 1
 +-- Thread 2
 +-- Thread 3
 +-- Thread 4
```

Multiple processes can also run:

```text
Application
 |
 +-- Process 1
 |     +-- Thread 1
 |     +-- Thread 2
 |
 +-- Process 2
 |     +-- Thread 1
 |     +-- Thread 2
 |
 +-- Process 3
       +-- Thread 1
       +-- Thread 2
```

Processes provide stronger isolation.

Threads allow multiple requests to be handled within the same process, depending on the Ruby runtime and application server configuration.

---

## 3. Why Concurrency Matters

Suppose requests spend significant time waiting for I/O:

```text
Request
   |
   v
Database
   |
   | waiting
   |
   v
Response
```

A concurrent application can use that waiting time to process another request.

```text
Thread 1 → Waiting for DB
Thread 2 → Processing request
Thread 3 → Calling external API
Thread 4 → Processing request
```

This can improve overall throughput.

However, concurrency also increases pressure on shared resources.

---

## 4. High-Level Architecture

```text
                         Load Balancer
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
          Rails App       Rails App       Rails App
          Process         Process         Process
              |               |               |
          +---+---+       +---+---+       +---+---+
          |   |   |       |   |   |       |   |   |
          v   v   v       v   v   v       v   v   v
        T1  T2  T3      T1  T2  T3      T1  T2  T3
          \   |   /       \   |   /       \   |   /
           \  |  /         \  |  /         \  |  /
            v | v            v | v            v | v
             Connection Pool / DB
                         |
                         v
                    PostgreSQL
```

The important relationship is:

```text
Processes
    +
Threads
    +
Database Connection Pool
    +
Available CPU/Memory
```

These values need to be considered together.

---

## 5. Database Connection Pool

Rails uses a database connection pool.

A simplified example:

```text
Rails Process
 |
 +-- Thread 1 ---> Connection 1
 +-- Thread 2 ---> Connection 2
 +-- Thread 3 ---> Connection 3
 +-- Thread 4 ---> Connection 4
```

If all threads need database access simultaneously, enough connections must be available.

If they are not:

```text
Thread
  |
  v
Request DB Connection
  |
  v
No Connection Available
  |
  v
Wait
```

This can appear as application latency even when CPU usage is low.

---

## 6. Connection Pool and Concurrency

Suppose an application has:

```text
3 processes
4 threads per process
```

The theoretical maximum concurrent threads could be:

```text
3 × 4 = 12
```

If many of those threads access PostgreSQL concurrently, the database connection capacity must be considered accordingly.

The important point is that application concurrency and database capacity must be designed together.

---

## 7. Increasing Workers Is Not Always the Solution

Suppose an endpoint is slow.

A common reaction is:

```text
More traffic
   |
   v
Add more workers
```

But if PostgreSQL is already saturated:

```text
More Workers
     |
     v
More DB Connections
     |
     v
More DB Load
     |
     v
Database Becomes Slower
```

The correct approach is to identify the bottleneck first.

Possible bottlenecks include:

* CPU
* Memory
* Database
* External API
* Redis
* Network
* Lock contention

---

## 8. CPU-Bound vs I/O-Bound Work

The type of workload matters.

### CPU-Bound

Examples:

* Heavy computation
* Large data transformation
* Complex serialization
* Image processing

Conceptually:

```text
Request
   |
   v
CPU
   |
   v
CPU
   |
   v
Response
```

Adding concurrency does not automatically improve CPU-bound work.

The CPU itself may become the bottleneck.

---

### I/O-Bound

Examples:

* Database queries
* External APIs
* Network operations
* Storage operations

Conceptually:

```text
Thread 1 → Waiting for DB
Thread 2 → Processing
Thread 3 → Waiting for API
Thread 4 → Processing
```

Concurrency can be more useful when work spends significant time waiting for I/O.

---

## 9. Memory Considerations

Every process consumes memory.

For example:

```text
Process 1 → Memory
Process 2 → Memory
Process 3 → Memory
```

Increasing the number of processes increases the overall memory requirement.

Threads generally share the process memory space, but they still introduce additional execution and stack-related overhead.

Therefore worker sizing should consider:

```text
Total Memory
    |
    v
Application Processes
    |
    v
Per-Process Memory
```

Memory usage should be measured rather than estimated only from configuration.

---

## 10. Worker Sizing

There is no universal number of workers or threads that works for every Rails application.

A practical approach is:

```text
Measure Current System
        |
        v
Identify Bottleneck
        |
        v
Change Worker/Thread Configuration
        |
        v
Load Test
        |
        v
Measure Again
        |
        v
Choose Stable Configuration
```

Metrics to observe include:

* Request latency
* Throughput
* CPU
* Memory
* Database connections
* Database latency
* Error rate
* Queue latency

---

## 11. Application Server

Rails applications can run behind application servers such as Passenger or other Rack-compatible servers.

The application server is responsible for managing application processes and/or threads and receiving requests from the web server.

A simplified flow:

```text
Client
  |
  v
Web Server / Load Balancer
  |
  v
Application Server
  |
  +---- Process
  |       +-- Thread
  |       +-- Thread
  |
  +---- Process
          +-- Thread
          +-- Thread
```

The exact concurrency model depends on the server and its configuration.

---

## 12. Request Isolation

Processes provide stronger isolation than threads.

If one process crashes:

```text
Process 1 → Failed

Process 2 → Running
Process 3 → Running
```

The other processes can continue serving requests.

This is one reason multiple application processes are commonly used in production.

---

## 13. Shared State

With multiple processes, local memory should not be treated as shared application state.

For example:

```text
Process 1
  |
  +-- Local Memory

Process 2
  |
  +-- Different Local Memory
```

If application state needs to be shared, use an appropriate external system:

```text
Process 1 ----+
              |
Process 2 ----+----> Redis / PostgreSQL
              |
Process 3 ----+
```

This is especially important when applications are horizontally scaled.

---

## 14. Graceful Deployment

A production deployment should avoid unnecessarily terminating active requests.

A simplified deployment flow:

```text
New Version
    |
    v
Start New Application Processes
    |
    v
Health Check
    |
    v
Send Traffic
    |
    v
Drain Old Processes
    |
    v
Stop Old Processes
```

Graceful shutdown allows existing requests to complete where supported by the deployment setup.

---

## 15. Long-Running Requests

Long-running HTTP requests can consume worker capacity.

For example:

```text
Worker
  |
  v
Long Request
  |
  |----------------------|
                         |
                     Still Running
```

If many requests behave this way:

```text
All Workers Busy
       |
       v
New Requests Wait
```

Possible solutions include:

* Move work to background jobs
* Optimize slow database queries
* Optimize external API calls
* Introduce caching
* Stream responses when appropriate

The correct solution depends on why the request is slow.

---

## 16. Background Workers

Web request workers and background job workers should be treated as separate workloads.

```text
                    Application
                         |
              +----------+----------+
              |                     |
              v                     v
          Web Requests         Background Jobs
              |                     |
              v                     v
          API Workers          Job Workers
```

This prevents a large background workload from unnecessarily consuming all capacity needed for user-facing requests.

---

## 17. Database Bottleneck

A common production situation is:

```text
Application
    |
    v
Many Threads
    |
    v
Many DB Queries
    |
    v
PostgreSQL
    |
    v
High DB Utilization
```

Before increasing application concurrency, investigate:

* Slow queries
* Missing indexes
* N+1 queries
* Lock contention
* Large transactions
* Connection saturation
* Inefficient queries

Increasing Rails concurrency cannot fix an inefficient database workload.

---

## 18. Connection Exhaustion

One possible failure scenario:

```text
Application
    |
    v
Too Many Concurrent Requests
    |
    v
Connection Pool Exhausted
    |
    v
Threads Wait
    |
    v
Request Latency Increases
```

Symptoms may include:

* Requests waiting for database connections
* Increased response time
* Timeout errors
* Increasing request queue
* Database connection saturation

The investigation should look at both the application and database sides.

---

## 19. Memory Growth

Another common production problem is increasing memory usage.

```text
Time
 |
 |      /
 |     /
 |    /
 |   /
 |__/
 +------------------> 
```

Possible causes include:

* Large objects retained in memory
* Unexpected caching
* Large query results
* Memory leaks
* Excessive object allocation
* Long-running processes retaining state

A useful investigation approach is:

```text
Memory Increase
      |
      v
Measure Process Memory
      |
      v
Check Allocation / Object Growth
      |
      v
Identify Retained Objects
      |
      v
Fix Cause
      |
      v
Measure Again
```

---

## 20. Ruby Garbage Collection

Ruby automatically manages memory using garbage collection.

Conceptually:

```text
Application
    |
    v
Create Objects
    |
    v
Objects Become Unused
    |
    v
Garbage Collector
    |
    v
Memory Reclaimed
```

Garbage collection itself consumes CPU time.

If an application creates excessive temporary objects:

```text
More Allocations
      |
      v
More GC Work
      |
      v
Higher CPU
      |
      v
Lower Throughput
```

Therefore Ruby performance work may involve reducing unnecessary object allocation rather than simply increasing server resources.

---

## 21. Performance Investigation

When concurrency or worker performance becomes an issue:

```text
Performance Problem
       |
       v
Check Request Latency
       |
       v
Check CPU
       |
       v
Check Memory
       |
       v
Check DB Connections
       |
       v
Check DB Query Time
       |
       v
Check External Dependencies
       |
       v
Identify Bottleneck
```

Tools and techniques may include:

* Application logs
* Request metrics
* Database query logs
* Ruby profiling tools
* Memory profiling
* APM tools
* PostgreSQL `EXPLAIN ANALYZE`

The specific tool is less important than having measurable evidence.

---

## 22. Load Testing

Before changing worker configuration in production, load testing can help understand system behavior.

A simple test can evaluate:

```text
Requests
   |
   v
10 concurrent users
   |
   v
20 concurrent users
   |
   v
50 concurrent users
   |
   v
Increasing Load
```

Measure:

* Average latency
* Percentile latency
* Throughput
* Error rate
* CPU
* Memory
* Database load

The goal is to find where the system starts degrading.

---

## 23. Example Bottleneck Analysis

Suppose request latency increases.

Initial observation:

```text
CPU       → Normal
Memory    → Normal
DB        → High
Latency   → High
```

The likely investigation path becomes:

```text
High DB Usage
     |
     v
Check Slow Queries
     |
     v
Check Query Plan
     |
     v
Check Indexes
     |
     v
Check N+1
     |
     v
Optimize Query
     |
     v
Measure Again
```

Adding more Rails workers would not necessarily solve the underlying problem.

---

## 24. Design Trade-offs

### More Processes

**Pros**

* Better isolation
* Can use multiple CPU cores
* Process failure is isolated

**Cons**

* Higher memory usage
* More database connections

---

### More Threads

**Pros**

* Can improve I/O concurrency
* Lower process overhead

**Cons**

* More shared-resource contention
* Requires appropriate connection-pool sizing
* Thread safety must be considered

---

### More Database Connections

**Pros**

* More concurrent database work

**Cons**

* More database resource usage
* Can overwhelm PostgreSQL
* Does not solve slow queries

---

### More Application Instances

**Pros**

* Higher capacity
* Better availability

**Cons**

* More infrastructure
* More database connections
* Shared dependencies may become bottlenecks

---

## 25. Recommended Design Approach

I would approach worker and concurrency configuration like this:

```text
Understand Workload
       |
       v
Classify CPU vs I/O
       |
       v
Measure Current Performance
       |
       v
Check DB / External Dependencies
       |
       v
Choose Process / Thread Model
       |
       v
Configure Connection Pool
       |
       v
Load Test
       |
       v
Monitor Resource Usage
       |
       v
Tune Gradually
```

Avoid changing several variables simultaneously because it becomes difficult to identify which change affected the system.

---

## 26. Key Design Principles

* Concurrency should be based on workload, not arbitrary numbers.
* Processes and threads have different trade-offs.
* Database connections must be considered when increasing concurrency.
* More workers do not automatically mean more performance.
* Identify the bottleneck before scaling.
* CPU-bound and I/O-bound workloads behave differently.
* Keep long-running work out of synchronous requests when possible.
* Monitor memory as well as CPU.
* Measure Ruby allocation and garbage-collection behavior when investigating memory or CPU issues.
* Treat web requests and background jobs as separate workloads.
* Load-test configuration changes where practical.
* Tune gradually and measure the result.

## Summary

Rails concurrency is a system-level problem rather than simply a worker configuration problem.

A reliable production configuration needs to balance:

**Processes → Threads → Database Connections → CPU → Memory → External Dependencies**

The objective is not to maximize concurrency.

The objective is to find the level of concurrency where the application provides good throughput and latency without overwhelming its dependencies.
