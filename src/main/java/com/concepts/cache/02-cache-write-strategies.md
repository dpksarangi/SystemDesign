# Cache Write Strategies

## 1. Mental Model

Write strategies answer:

> **When data changes, which layer is written first and when does the DB
> become durable?**

The main patterns:

-   Write-Through
-   Write-Behind / Write-Back
-   Write-Around

------------------------------------------------------------------------

## 2. Write-Through

Application writes through the cache, and the underlying DB is
synchronously updated.

``` text
Application
     |
     v
   Cache
     |
     v
    DB
     |
     v
 success
```

The write is normally considered successful only after the durable store
succeeds.

### Example

``` text
SET user:123 = NEW
        |
        v
UPDATE DB
        |
        v
return success
```

### Advantages

-   Cache is generally fresh after successful writes
-   Simple read path
-   Less stale-cache risk

### Problems

-   DB remains on the write critical path
-   Higher write latency
-   Does not remove DB write pressure
-   Failure handling must coordinate cache and DB

### Use when

-   Writes are moderate
-   Fresh cache matters
-   Synchronous persistence is acceptable

------------------------------------------------------------------------

## 3. Write-Behind / Write-Back

Application writes to cache first. DB persistence happens
asynchronously.

``` text
Application
     |
     v
   Redis
     |
     | async
     v
    DB
```

Example:

``` text
Initial:
Redis = 100
DB    = 100

Write:
Redis = 110
DB    = 100

Later:
DB = 110
```

### Advantages

-   Very low write latency
-   High write throughput
-   Can batch/coalesce writes
-   Absorbs traffic spikes

### Problems

#### Data loss

``` text
Redis updated
    |
    X
Redis crashes before DB persistence
```

#### Temporary inconsistency

``` text
Redis != DB
```

#### Ordering

Multiple updates must reach DB in the correct logical order.

#### Recovery

Need durable event/logging, persistence, replication, replay, or
reconciliation.

### Use when

-   Very high write throughput matters
-   Eventual persistence is acceptable
-   A reliable recovery mechanism exists

------------------------------------------------------------------------

## 4. Write-Around

Writes bypass the cache.

``` text
Application
     |
     +------> DB

Cache remains untouched
```

Next read populates cache:

``` text
Read
 |
 v
Cache MISS
 |
 v
DB
 |
 v
Cache
```

### Why?

Suppose millions of records are written but only a small percentage are
read.

Populating cache on every write wastes cache capacity.

### Use when

-   Write-heavy workload
-   Low read-after-write probability
-   Cache should contain only hot data

------------------------------------------------------------------------

## 5. Write-Through vs Write-Behind

                       Write-Through   Write-Behind
  -------------------- --------------- ---------------------
  App latency          Higher          Lower
  DB write             Synchronous     Async
  DB pressure          Full            Reduced
  Consistency          Stronger        Eventual
  Failure complexity   Lower           Higher
  Throughput           Moderate        Very high potential

------------------------------------------------------------------------

## 6. The Critical Ownership Question

Always ask:

> **Is Redis a cache or part of the state machine?**

Normal cache:

``` text
DB = authority
Redis = derived copy
```

Write-behind:

``` text
Redis = temporary dirty state
DB = durable authority
```

High-throughput inventory:

``` text
DB = durable state
Redis = hot-path reservation/state-transition engine
```

That last case is **not simply a caching pattern**.

------------------------------------------------------------------------

## 7. Common Failure Scenarios

### Cache write succeeds, DB write fails

With write-through:

``` text
Redis = NEW
DB = OLD
```

Need rollback, retry, or invalidate depending on semantics.

### Redis succeeds, process crashes before enqueueing persistence

This is the classic write-behind durability problem.

Solution:

-   durable event/log
-   transactional/outbox-style mechanism where appropriate
-   Redis persistence
-   replay/reconciliation

### Writes arrive out of order

Example:

``` text
A: balance = 90
B: balance = 80
```

If B reaches DB first and A later, final state may incorrectly become
90.

Use:

-   sequence/version
-   partitioning by entity
-   conditional updates
-   idempotency

------------------------------------------------------------------------

## 8. Interview Questions

### Q: Why not use Write-Through everywhere?

Because DB remains in the critical path and high write traffic still
reaches DB.

### Q: When would Write-Behind be dangerous?

When losing or reordering an update is unacceptable and there is no
durable recovery mechanism.

### Q: Is Write-Behind the same as eventual consistency?

Not exactly. Write-behind commonly produces eventual consistency, but
eventual consistency is a broader consistency property.

### Q: How do you protect Write-Behind from cache failure?

Use durable queues/logs, persistence, replication, replay and
reconciliation depending on the required durability.

### Q: Why is our inventory Redis design different?

Because Redis is not merely a dirty cache. It performs atomic
reserve/release operations and owns the short-lived hot-path reservation
state.
