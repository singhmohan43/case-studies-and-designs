# Background Job System — High-Level Design

## 1. Problem

Design a background processing system for operations that should not block a user's HTTP request.

Typical examples include:

* Sending emails or notifications
* Processing uploaded files
* Generating reports
* Synchronizing data from external systems
* Processing large datasets
* Performing scheduled maintenance tasks

The system should be reliable, retryable, observable, and able to process increasing workloads.

---

## 2. Goals

The system should provide:

* Asynchronous processing
* Reliable job execution
* Automatic retries
* Failure handling
* Idempotent processing
* Controlled concurrency
* Monitoring
* Horizontal scalability

The design should also avoid putting unnecessary load on the Rails application or database.

---

## 3. High-Level Architecture

```text
                         Client
                           |
                           v
                    Rails Application
                           |
                           |
                     Enqueue Job
                           |
                           v
                         Redis
                           |
                 +---------+---------+
                 |                   |
                 v                   v
             Worker 1             Worker 2
                 |                   |
                 +---------+---------+
                           |
                           v
                  Application / Services
                    /       |       \
                   v        v        v
               Database  External   Storage
                          APIs
```

The Rails application accepts the request and creates a background job.

Workers consume jobs independently and perform the actual processing.

---

## 4. Why Background Processing?

Consider an operation that takes several seconds:

```text
Without Background Job

User
 |
 v
Rails Request
 |
 v
Call External Service
 |
 v
Process Data
 |
 v
Save Result
 |
 v
Response
```

The user has to wait for the complete operation.

With background processing:

```text
User
 |
 v
Rails Request
 |
 v
Create Job
 |
 v
Return Response
       |
       |
       v
     Redis
       |
       v
     Worker
       |
       v
Process Operation
```

The HTTP request finishes quickly while the expensive operation runs asynchronously.

---

## 5. Job Lifecycle

A typical job lifecycle is:

```text
Created
   |
   v
Queued
   |
   v
Picked by Worker
   |
   v
Processing
   |
   +------ Success ------> Completed
   |
   +------ Failure ------> Retry
                              |
                              v
                           Queued
                              |
                              v
                         Process Again
```

If a job repeatedly fails, it should eventually move to a failed/dead state rather than retry indefinitely.

---

## 6. Job Structure

A job should have a clear responsibility.

Example:

```ruby
class GenerateReportJob
  include Sidekiq::Job

  def perform(report_id)
    report = Report.find(report_id)

    ReportGenerator.new(report).generate
  end
end
```

The job should coordinate the operation rather than contain a large amount of business logic.

The actual processing can live in a service or domain object.

```text
Job
 |
 v
Service
 |
 +---- Database
 |
 +---- External API
 |
 +---- Storage
```

This makes the business logic easier to test independently from the job framework.

---

## 7. Passing Job Arguments

Prefer passing identifiers rather than large objects.

For example:

```ruby
GenerateReportJob.perform_async(report.id)
```

Instead of passing a large serialized object.

The worker can load the current record when processing starts.

This keeps job payloads small and reduces serialization problems.

---

## 8. Idempotency

A job may execute more than once.

For example:

```text
Job
 |
 v
Worker
 |
 v
External API succeeds
 |
 X
Worker crashes before marking job complete
 |
 v
Job retried
```

The operation may now run again.

Therefore important jobs should be designed to be **idempotent**.

For example, instead of blindly creating a record:

```ruby
Notification.create(...)
```

the application may use a unique business key or database constraint so that the same logical operation cannot create duplicates.

```text
Job
 |
 v
Check operation state
 |
 +---- Already completed ---> Stop
 |
 +---- Not completed -------> Process
                              |
                              v
                           Mark Done
```

Idempotency is especially important when dealing with payments, notifications, external APIs, and data synchronization.

---

## 9. Retry Strategy

Not every error should be treated the same way.

### Temporary failures

Examples:

* Network timeout
* External service unavailable
* Temporary database connection issue

These can usually be retried.

```text
Failure
  |
  v
Retry
  |
  v
Retry
  |
  v
Success
```

### Permanent failures

Examples:

* Invalid input
* Missing required record
* Invalid configuration

Retrying may not solve these problems.

The system should record the failure and make it available for investigation.

---

## 10. Exponential Backoff

Repeated retries should not happen immediately.

A simple strategy is:

```text
Attempt 1 → wait
Attempt 2 → wait longer
Attempt 3 → wait even longer
Attempt 4 → move to failed/dead state
```

This prevents a temporary dependency failure from creating a large burst of traffic.

---

## 11. Queue Design

Different types of jobs may have different priorities.

For example:

```text
High Priority
    |
    +-- User notifications
    +-- Important workflows

Normal Priority
    |
    +-- Data processing
    +-- Synchronization

Low Priority
    |
    +-- Reports
    +-- Maintenance
```

Separate queues allow workers to process important jobs without being blocked by large amounts of lower-priority work.

---

## 12. Concurrency

Workers can process multiple jobs concurrently.

```text
                  Queue
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
     Worker 1    Worker 2    Worker 3
        |           |           |
        v           v           v
      Job A       Job B       Job C
```

Increasing concurrency can improve throughput, but it also increases resource usage.

Important limits include:

* CPU
* Memory
* Database connections
* External API rate limits

More workers do not automatically mean better performance.

---

## 13. Database Connection Pool

One important relationship is:

```text
Worker Concurrency
        |
        v
Database Connections
```

If workers need database connections, the ActiveRecord connection pool must be sized appropriately.

For example:

```text
Worker Process
    |
    +-- Thread 1 ---> DB connection
    +-- Thread 2 ---> DB connection
    +-- Thread 3 ---> DB connection
    +-- Thread 4 ---> DB connection
```

If concurrency is increased without considering the database connection pool, workers may wait for available connections.

The database itself can also become the bottleneck.

---

## 14. External API Rate Limits

Background workers may call external services.

If many workers process jobs simultaneously:

```text
Workers
  |
  +----+
  |    |
  v    v
External API
```

the application may exceed the API's rate limit.

Possible controls include:

* Concurrency limits
* Rate limiting
* Queue separation
* Exponential backoff
* Request batching

The goal is to increase throughput without overwhelming dependencies.

---

## 15. Large Batch Processing

Large datasets should generally not be loaded into memory at once.

Instead of:

```ruby
records = Record.all

records.each do |record|
  process(record)
end
```

use batching:

```ruby
Record.find_each do |record|
  process(record)
end
```

Conceptually:

```text
Database
   |
   +---- Batch 1 ---> Process
   |
   +---- Batch 2 ---> Process
   |
   +---- Batch 3 ---> Process
   |
   +---- ...
```

This keeps memory usage more predictable.

---

## 16. Scheduling

Some jobs need to run periodically.

Examples:

* Synchronization
* Cleanup
* Report generation
* Data aggregation

A scheduler can create jobs:

```text
Scheduler
    |
    v
Create Job
    |
    v
Queue
    |
    v
Worker
    |
    v
Process
```

Scheduling should be separated from actual job execution where practical.

The scheduler decides **when** something should run.

The worker decides **how** it should run.

---

## 17. Failure Handling

A production system should make failed jobs visible.

```text
Job
 |
 v
Worker
 |
 X
Failure
 |
 +---- Retryable ---> Retry Queue
 |
 +---- Permanent ---> Failed/Dead Queue
                         |
                         v
                    Investigation
```

Useful information to capture includes:

* Job type
* Job ID
* Input identifier
* Error message
* Retry count
* Timestamp
* Duration
* Relevant correlation/request ID

Sensitive data should not be included in logs.

---

## 18. Monitoring

Important metrics include:

### Queue Metrics

* Queue depth
* Oldest job age
* Jobs processed
* Jobs failed
* Retry count

### Worker Metrics

* Worker utilization
* Processing time
* CPU usage
* Memory usage
* Worker crashes

### Application Metrics

* Database connection usage
* External API failures
* Request latency
* Error rates

A useful operational signal is not just the number of queued jobs, but **how long jobs have been waiting**.

---

## 19. Scaling

If queue depth continuously increases:

```text
Incoming Jobs
      |
      v
    Queue
      |
      v
Workers cannot keep up
```

we need to increase processing capacity.

Possible approaches:

```text
                 Queue
                   |
        +----------+----------+
        |          |          |
        v          v          v
     Worker      Worker      Worker
```

Workers can be scaled horizontally.

However, scaling workers should be done together with checking:

* Database capacity
* Redis capacity
* External API limits
* CPU
* Memory
* Connection pools

Otherwise the bottleneck simply moves to another component.

---

## 20. Observability Flow

A useful production troubleshooting flow is:

```text
Queue Backlog
     |
     v
Check Queue Age
     |
     v
Check Worker Health
     |
     v
Check Job Duration
     |
     v
Check Database / External APIs
     |
     v
Identify Bottleneck
     |
     v
Apply Fix
     |
     v
Monitor Result
```

---

## 21. Design Trade-offs

### More Workers

**Pros**

* Higher throughput
* More parallel processing

**Cons**

* More memory usage
* More database connections
* More load on dependencies

### More Retries

**Pros**

* Better recovery from temporary failures

**Cons**

* Can increase load during an outage
* Can delay detection of permanent failures

### More Queues

**Pros**

* Better priority control
* Better workload isolation

**Cons**

* More operational complexity

### More Caching

**Pros**

* Lower database load
* Faster repeated operations

**Cons**

* Cache invalidation complexity
* Stale data risk

---

## 22. Security Considerations

Job payloads should contain only the information required to perform the operation.

Avoid placing:

* Passwords
* Authentication tokens
* Sensitive personal information
* Large objects

directly into job payloads or logs.

Access to job monitoring and administrative interfaces should also be restricted.

---

## 23. Testing Strategy

Background jobs should be tested for:

### Successful execution

```text
Input → Job → Expected Result
```

### Retry behavior

```text
Temporary Failure → Retry → Success
```

### Permanent failure

```text
Invalid Input → Failure → No Unnecessary Retries
```

### Idempotency

```text
Same Job
   |
   +---- First Execution → Result
   |
   +---- Second Execution → No Duplicate Effect
```

### Large datasets

Verify that batch processing does not unnecessarily increase application memory.

---

## 24. Final Architecture

```text
                         Rails Application
                               |
                               v
                         Job Producer
                               |
                               v
                         +-----------+
                         |   Redis   |
                         +-----------+
                               |
                +--------------+--------------+
                |              |              |
                v              v              v
             Worker 1       Worker 2       Worker 3
                |              |              |
                +--------------+--------------+
                               |
                 +-------------+-------------+
                 |             |             |
                 v             v             v
```
