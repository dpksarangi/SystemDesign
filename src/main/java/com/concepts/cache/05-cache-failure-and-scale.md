# Cache Failure and Scale

## 1. Cache Stampede / Thundering Herd

A popular key expires:

``` text
1M requests
     |
     v
Redis MISS
     |
     v
1M DB requests
```

### Mitigation

-   Single-flight/request coalescing
-   Distributed lock
-   Refresh-ahead
-   TTL jitter
-   Request rate limiting
-   Stale-while-revalidate where acceptable

------------------------------------------------------------------------

## 2. Cache Penetration

Requests target data that does not exist.

``` text
GET fake:123
    |
    v
Cache MISS
    |
    v
DB MISS
```

Repeated millions of times can overload DB.

### Mitigation

-   Negative caching
-   Bloom filters
-   Request validation
-   Rate limiting

Example:

``` text
fake:123 -> NOT_FOUND
TTL = 30 sec
```

------------------------------------------------------------------------

## 3. Cache Avalanche

Many keys expire simultaneously.

``` text
10M keys
TTL = 1 hour

1:00 PM
  |
  +--> 10M expirations
  |
  +--> DB spike
```

### Mitigation

-   TTL jitter
-   Staggered TTLs
-   Refresh-ahead
-   Cache warming
-   Rate limiting
-   Multiple cache tiers

------------------------------------------------------------------------

## 4. Hot Key

A small number of keys receive disproportionate traffic.

``` text
1M requests
     |
     v
product:iphone
     |
     v
one Redis shard
```

A large Redis cluster does not automatically solve this.

### Mitigation

For read-heavy data:

-   L1/local cache
-   safe replication of read-only data
-   request coalescing
-   CDN

For mutable/correctness-critical data:

-   admission control
-   serialization
-   safe bucketing/sharding where business semantics permit
-   waiting room

Do not blindly replicate a mutable inventory counter.

------------------------------------------------------------------------

## 5. Cache Failure

### Normal Cache-Aside

Redis fails:

``` text
Redis unavailable
      |
      v
DB fallback
```

Only if DB has enough capacity.

### High-traffic system

Failing open to DB can turn:

``` text
Redis outage
```

into:

``` text
DB outage
```

Therefore:

-   rate limit fallback
-   circuit breaker
-   stale reads where acceptable
-   controlled degradation
-   fail closed for correctness-critical paths

------------------------------------------------------------------------

## 6. Cache Replica Lag

``` text
Primary = V10
Replica = V9
```

A read from replica can return stale data.

For ordinary catalog reads this may be acceptable.

For inventory authorization:

``` text
DO NOT:
read stale replica
   |
   v
approve reservation
```

Use an authoritative/atomic mutation path.

------------------------------------------------------------------------

## 7. Cache Eviction

Redis may evict keys depending on policy.

Possible policies include:

-   no eviction
-   all-keys LRU
-   volatile LRU
-   LFU variants

### Important question

What happens if an evicted key is:

``` text
inventory reservation
```

versus:

``` text
product description
```

For a normal cache, rebuilding is fine.

For stateful inventory, eviction can mean **business-state loss**.

Therefore reservation/inventory state requires a different durability
model from ordinary cache data.

------------------------------------------------------------------------

## 8. Cache Warming

Before a flash sale:

``` text
DB
 |
 v
warm-up
 |
 v
Redis
```

Load:

-   product metadata
-   pricing
-   campaign configuration
-   inventory state where appropriate

But warming must not create a second authority.

------------------------------------------------------------------------

## 9. Redis Recovery / Rebuild

For rebuildable cache:

``` text
DB
 |
 v
Redis rebuild
```

For stateful Redis:

``` text
Durable state/events
       |
       v
reconstruct state
       |
       v
Redis
```

This distinction is critical.

------------------------------------------------------------------------

## 10. Capacity Planning

Estimate:

``` text
QPS
Memory
Network bandwidth
Hot-key concentration
Replication factor
Peak-to-average ratio
```

Example:

``` text
Peak = 1M req/sec

If 80% targets one product:
800K req/sec → one logical key
```

Cluster size alone doesn't solve this.

------------------------------------------------------------------------

## 11. Fail-Open vs Fail-Closed

### Fail-open

If cache is unavailable:

``` text
continue using DB
```

Good for low-risk reads.

### Fail-closed

If authoritative cache/state is unavailable:

``` text
reject/defer request
```

Good for:

-   inventory reservation
-   payment authorization
-   security decisions

The correct choice depends on whether stale data can cause an incorrect
business action.

------------------------------------------------------------------------

## 12. Reconciliation

For systems where cache has stateful responsibilities, reconciliation is
a safety net.

``` text
Durable source/events
        |
        v
Expected state
        |
        +---- compare ----> Redis
```

But reconciliation should not be used as the primary mechanism for every
state transition.

------------------------------------------------------------------------

## 13. Failure Matrix

  ------------------------------------------------------------------------
  Failure                 Normal cache            Stateful inventory
  ----------------------- ----------------------- ------------------------
  Cache unavailable       DB fallback             Controlled/fail-closed

  Cache stale             Usually acceptable      Dangerous for
                          briefly                 authorization

  Key evicted             Rebuild                 Potential state loss

  Replica lag             Often acceptable        Dangerous

  Stampede                Protect DB              Protect state engine

  Rebuild                 Read DB                 Replay durable
                                                  state/events
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## 14. Interview Questions

### Q: Redis is down. Why not always use DB?

Because the DB may receive the entire traffic spike and fail too. A
cache outage should not automatically become a database outage.

### Q: How do you solve a hot key?

First identify whether it is read-only or mutable. Use
L1/CDN/replication for read-heavy data; use admission control and
serialization/bucketing strategies for mutable hot state.

### Q: Is reconciliation enough for inventory?

No. Correctness must be enforced at the state transition. Reconciliation
detects and repairs unexpected divergence.

### Q: What is the biggest cache failure mistake?

Treating every cache as disposable. A normal cache can be rebuilt; a
cache that participates in business-state transitions requires
durability and recovery design.
