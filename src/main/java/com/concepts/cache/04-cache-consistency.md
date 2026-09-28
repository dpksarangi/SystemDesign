# Cache Consistency

## 1. The Core Question

When DB and cache contain different values:

``` text
DB    = NEW
Cache = OLD
```

what is the system allowed to return?

Caching strategy and consistency strategy are separate decisions.

------------------------------------------------------------------------

## 2. TTL-Based Consistency

``` text
SET key value EX 300
```

Data may remain stale until TTL expires.

### Advantages

-   Very simple
-   Automatic cleanup
-   Limits maximum staleness approximately by TTL

### Problems

-   Data can be stale
-   Short TTL increases DB/cache load
-   Long TTL increases staleness

Use when approximate freshness is acceptable.

------------------------------------------------------------------------

## 3. Explicit Invalidation

Typical pattern:

``` text
UPDATE DB
   |
   v
DELETE Redis key
```

Next read reloads it.

### Advantage

Usually fresher than TTL-only.

### Failure

``` text
DB update succeeds
Redis DELETE fails
```

Now stale data remains.

Therefore production systems often combine explicit invalidation with
TTL as a safety net.

------------------------------------------------------------------------

## 4. Event-Driven Invalidation

``` text
DB change
   |
   v
Event
   |
   v
Kafka
   |
   +--> Service A cache
   +--> Service B cache
   +--> Service C cache
```

Advantages:

-   Decouples producers and consumers
-   Works well with many application instances
-   Supports distributed cache invalidation

Problems:

-   Events may be delayed
-   Consumers need idempotency
-   Ordering may matter
-   Temporary staleness is expected

------------------------------------------------------------------------

## 5. Versioning

Store:

``` text
value
version
```

Example:

``` text
DB:
value = V5
version = 5

Redis:
value = V4
version = 4
```

A consumer can reject older versions.

This is useful for preventing stale updates from overwriting newer
state.

------------------------------------------------------------------------

## 6. Strong-ish Consistency

Absolute global linearizability is expensive and uncommon for ordinary
caches.

A practical design may define a consistency boundary such as:

> After a successful write, subsequent reads through the same service
> must observe the new value.

Techniques:

-   write-through
-   synchronous invalidation
-   version checks
-   sticky/session routing where appropriate

Always define exactly what "consistent" means.

------------------------------------------------------------------------

## 7. Eventual Consistency

``` text
DB update
   |
   v
event
   |
   v
cache update
```

There is a period:

``` text
DB = NEW
Cache = OLD
```

Eventually:

``` text
DB = NEW
Cache = NEW
```

Good for:

-   recommendations
-   product descriptions
-   counters where approximate freshness is acceptable
-   search indexes
-   analytics

Dangerous for:

-   financial authorization
-   seat ownership
-   scarce inventory decisions

------------------------------------------------------------------------

## 8. Read-After-Write

User performs:

``` text
PUT profile
```

Then immediately:

``` text
GET profile
```

If GET returns the old value, the user experiences inconsistency.

Possible solutions:

-   update/invalidate cache synchronously
-   read from primary after write
-   version/session token
-   temporarily bypass stale cache

------------------------------------------------------------------------

## 9. Monotonic Reads

A user should not observe:

``` text
version 5
   ↓
version 4
```

This can happen when requests hit replicas with different lag.

Solutions:

-   version checks
-   session stickiness
-   primary reads for critical paths
-   client/session version tracking

------------------------------------------------------------------------

## 10. Cache + DB Race Conditions

### Race 1: stale value reinserted

``` text
T1: read old value from DB
T2: DB updated to new value
T2: invalidate cache
T1: writes old value to cache
```

Now:

``` text
Cache = OLD
DB = NEW
```

Solutions:

-   locking/single-flight
-   versioned cache writes
-   DB version validation
-   update-before-invalidate patterns depending on workload

------------------------------------------------------------------------

## 11. Cache-DB Consistency Is Not One Universal Setting

Ask:

1.  What is the source of truth?
2.  Can stale reads be tolerated?
3.  Maximum acceptable staleness?
4.  Can cache be rebuilt?
5.  Can updates be lost?
6.  Does ordering matter?
7.  Is the data correctness-critical?

Then choose the mechanism.

------------------------------------------------------------------------

## 12. Inventory Example

Normal catalog:

``` text
DB = authoritative
Redis = cache
```

Temporary mismatch:

``` text
DB price = 999
Redis price = 1099
```

may be undesirable but recoverable.

Inventory reservation is different:

``` text
Total = 100

Redis reserve 3
Available = 97
```

Redis is executing a correctness-critical state transition.

Therefore:

> **Do not treat inventory Redis as a normal read cache.**

The inventory design requires atomic state transitions, reservation
identity, idempotency, expiration handling and durable recovery.

------------------------------------------------------------------------

## 13. Interview Questions

### Q: Is TTL enough for consistency?

No. TTL limits stale lifetime but does not prevent stale reads within
that window.

### Q: Why use TTL if we already invalidate?

TTL is a safety net for failed invalidations and unexpected paths.

### Q: When is eventual consistency acceptable?

When temporary stale data does not cause an incorrect business decision.

### Q: Can eventual consistency work for inventory?

It can for non-authoritative display/read paths, but the actual
reservation/authorization decision must use a strongly coordinated state
transition.
