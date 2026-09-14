# Scalable API — High-Level Design

## 1. Problem

Design a REST API that can support growing traffic while remaining maintainable, predictable, and easy to operate.

The API should support:

* CRUD operations
* Searching and filtering
* Pagination
* Authentication and authorization
* Validation
* Consistent error handling
* Caching where appropriate
* Background processing for long-running operations
* Monitoring and troubleshooting

The design is intentionally generic and does not represent any specific production system.

---

## 2. Requirements

### Functional Requirements

The API should allow clients to:

* Create resources
* Retrieve resources
* Update resources
* Delete resources
* Search and filter resources
* Retrieve related information
* Start asynchronous operations

### Non-Functional Requirements

The API should provide:

* Predictable response times
* Horizontal scalability
* Secure access
* Database efficiency
* Consistent API responses
* Good observability
* Safe handling of large datasets

---

## 3. High-Level Architecture

```text
                         Client
                           |
                           v
                    Load Balancer
                           |
              +------------+------------+
              |            |            |
              v            v            v
          Rails API    Rails API    Rails API
              |            |            |
              +------------+------------+
                           |
             +-------------+-------------+
             |                           |
             v                           v
        PostgreSQL                     Redis
             |                           |
             |                           |
             v                           v
       Persistent Data              Cache / Jobs
                                         |
                                         v
                                      Workers
                                         |
                                         v
                                  External Services
```

The API servers remain as stateless as practical so additional instances can be added when traffic increases.

---

## 4. API Request Flow

A typical request flows through:

```text
Client
  |
  v
Load Balancer
  |
  v
Rails Application
  |
  +-- Authentication
  |
  +-- Authorization
  |
  +-- Parameter Validation
  |
  +-- Business Logic
  |
  +-- Database / Cache
  |
  v
Response
```

The API should reject invalid requests as early as possible.

---

## 5. Resource-Oriented API

A typical resource API could look like:

```text
GET    /api/v1/resources
GET    /api/v1/resources/:id

POST   /api/v1/resources

PATCH  /api/v1/resources/:id

DELETE /api/v1/resources/:id
```

The API should use predictable resource names and HTTP methods.

For example:

```text
GET /api/v1/users
```

rather than action-oriented URLs such as:

```text
GET /api/v1/getUsers
```

For operations that represent actions rather than standard CRUD, an explicit action endpoint may be appropriate.

---

## 6. API Versioning

API versions provide a way to evolve the contract without unexpectedly breaking existing clients.

Example:

```text
/api/v1/resources
/api/v2/resources
```

A version should be introduced when there is a meaningful contract change that existing clients cannot safely handle.

Versioning should not be used as a substitute for good API design.

---

## 7. Authentication

A request normally passes through an authentication layer before reaching application logic.

```text
Request
  |
  v
Authentication
  |
  +---- Invalid ---> 401
  |
  v
Authenticated User
  |
  v
Authorization
  |
  +---- Not Allowed ---> 403
  |
  v
Controller
```

Authentication answers:

> Who is making the request?

Authorization answers:

> Is this user allowed to perform this operation?

These should remain separate concerns.

---

## 8. Request Validation

Input should be validated before performing business operations.

Example:

```text
POST /api/v1/resources

{
  "name": "",
  "type": "invalid"
}
```

The API should return a predictable validation response rather than allowing invalid data to reach deeper application layers.

A typical response could be:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "Invalid request",
    "fields": {
      "name": ["can't be blank"],
      "type": ["is not supported"]
    }
  }
}
```

The exact response structure should be standardized across the API.

---

## 9. Pagination

Returning every record from a large table is not scalable.

Instead:

```text
GET /api/v1/resources?page=1&per_page=50
```

The API can return:

```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "has_next": true
  }
}
```

Pagination reduces:

* Database work
* Network payload size
* Application memory usage
* Client-side processing

For very large or frequently changing datasets, cursor-based pagination can be considered instead of offset-based pagination.

---

## 10. Filtering

Filtering should be explicit and predictable.

Example:

```text
GET /api/v1/resources?status=active
```

Multiple filters can be combined:

```text
GET /api/v1/resources?status=active&type=server
```

The API should avoid allowing arbitrary database fields to become filter parameters without validation.

Allowed filters should be defined intentionally.

---

## 11. Sorting

Clients may need sorted results:

```text
GET /api/v1/resources?sort=created_at
```

For descending order:

```text
GET /api/v1/resources?sort=-created_at
```

The application should maintain an allowlist of sortable fields.

This prevents clients from requesting expensive or unintended database operations.

---

## 12. Database Access

The API should avoid unnecessary database queries.

For example, an N+1 problem can occur when:

```ruby
resources.each do |resource|
  puts resource.owner.name
end
```

causes a separate query for each owner.

Instead, related data can be loaded appropriately:

```ruby
Resource.includes(:owner)
```

The exact loading strategy should depend on the query and required response.

The goal is not simply to reduce the number of queries, but to make the overall database workload efficient.

---

## 13. Query Optimization

When an API endpoint becomes slow, investigate the actual database query.

Useful techniques include:

```sql
EXPLAIN
```

and:

```sql
EXPLAIN ANALYZE
```

Areas to investigate include:

* Missing indexes
* Poor query plans
* Large table scans
* Unnecessary joins
* Sorting large datasets
* Returning unnecessary columns
* N+1 queries

Indexes should be based on real query patterns.

---

## 14. Caching

Some API responses may be expensive to calculate but do not change frequently.

A cache can reduce repeated database work:

```text
Request
   |
   v
Check Redis
   |
   +---- Hit ----> Return Cached Response
   |
   +---- Miss
          |
          v
       Database
          |
          v
       Build Response
          |
          v
       Store Cache
          |
          v
       Return Response
```

Caching should be used carefully.

Important questions include:

* How long can data be stale?
* When should the cache expire?
* What happens after an update?
* How large can the cache become?

---

## 15. Cache Invalidation

A common strategy is to invalidate related cache entries when data changes.

```text
Update Resource
      |
      v
Save Database Record
      |
      v
Invalidate Cache
      |
      v
Next Request
      |
      v
Load Fresh Data
```

The cache should never become the only source of truth for important persistent data.

---

## 16. Asynchronous Operations

Some API operations should not remain synchronous.

For example:

```text
POST /api/v1/reports
```

If generating the report takes significant time:

```text
Client
  |
  v
POST /reports
  |
  v
Create Report Request
  |
  v
Enqueue Job
  |
  v
Return 202 Accepted
```

The client can then check the status:

```text
GET /api/v1/reports/:id
```

Example lifecycle:

```text
Pending
   |
   v
Processing
   |
   v
Completed
```

or:

```text
Pending
   |
   v
Processing
   |
   v
Failed
```

This keeps long-running work away from the normal request path.

---

## 17. Rate Limiting

An API may need protection against excessive traffic.

```text
Client
   |
   v
Rate Limiter
   |
   +---- Limit exceeded ---> 429
   |
   v
Rails API
```

Rate limiting can be applied based on:

* User
* API key
* Client
* IP
* Endpoint

The appropriate strategy depends on the application's security and traffic requirements.

---

## 18. Idempotency

Some API operations may be retried by clients because of network failures.

For example:

```text
Client
  |
  | POST request
  v
API
  |
  v
Operation succeeds
  |
  X
Response lost
  |
  v
Client retries
```

Without protection, the operation could happen twice.

For operations where duplicates are dangerous, an idempotency key can be used:

```text
POST /api/v1/payments
Idempotency-Key: abc123
```

The server can associate the key with the result of the operation.

A repeated request using the same key can return the existing result rather than executing the operation again.

---

## 19. Error Handling

The API should have consistent error responses.

Common categories:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Entity
429 Too Many Requests
500 Internal Server Error
```

Example:

```json
{
  "error": {
    "code": "resource_not_found",
    "message": "Resource not found"
  }
}
```

Internal implementation details should not be exposed to clients.

---

## 20. Database Transactions

Operations involving multiple related changes should use transactions when consistency requires them.

Example:

```text
Create Order
     |
     +-- Create Order Record
     |
     +-- Create Order Items
     |
     +-- Update Related Data
     |
     v
Commit
```

If an important step fails:

```text
Transaction
     |
     +-- Operation A ✓
     +-- Operation B ✓
     +-- Operation C ✗
     |
     v
```
