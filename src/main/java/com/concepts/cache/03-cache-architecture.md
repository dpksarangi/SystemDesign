# Cache Architecture

## 1. Cache Location

Caching can exist at multiple layers:

``` text
User
  |
  v
CDN / Edge
  |
  v
Application
  |
  v
L1 Local Cache
  |
  v
L2 Distributed Cache
  |
  v
DB
```

Each layer trades consistency, latency, capacity and operational
complexity.

------------------------------------------------------------------------

## 2. L1 / Local Application Cache

Examples:

-   Caffeine
-   Guava Cache
-   in-process maps with eviction

``` text
Application Instance
       |
       +--> Local Cache
       |
       +--> Redis
```

### Advantages

-   Extremely low latency
-   No network hop
-   Reduces Redis traffic
-   Good for very hot read-only data

### Problems

Every application instance has its own copy.

``` text
Node A → value V1
Node B → value V2
```

Invalidation becomes difficult.

### Use for

-   Configuration
-   Static metadata
-   Very hot read-heavy data
-   Short-lived data

------------------------------------------------------------------------

## 3. L2 / Distributed Cache

Example:

``` text
Application
     |
     v
Redis Cluster
 /    |    \
N1   N2    N3
     |
     v
    DB
```

Advantages:

-   Shared across instances
-   Larger capacity
-   Centralized eviction
-   Horizontal scaling

Costs:

-   Network hop
-   Cluster operations
-   Hot keys
-   replication/consistency issues

Redis is a common L2 cache.

------------------------------------------------------------------------

## 4. Multi-Level Cache

``` text
Request
   |
   v
L1 Caffeine
   |
   +-- HIT --> response
   |
   +-- MISS
         |
         v
      Redis
         |
         +-- HIT --> populate L1
         |
         +-- MISS
               |
               v
              DB
               |
               v
          Redis + L1
```

### Why?

A hot key may generate huge Redis traffic.

L1 absorbs the hottest reads.

### Problem

Now invalidation must handle:

``` text
L1 on every node
      +
Redis
      +
DB
```

Event-driven invalidation is often useful.

------------------------------------------------------------------------

## 5. Redis Cluster / Partitioning

A distributed cache usually partitions keys:

``` text
hash(key)
   |
   +--> shard 1
   +--> shard 2
   +--> shard 3
```

This scales memory and throughput.

### But partitioning does not automatically solve hot keys.

If:

``` text
product:iphone
```

is extremely hot, all requests may target one shard.

This is especially important for inventory.

------------------------------------------------------------------------

## 6. Replication

``` text
Primary
 /     \
R1     R2
```

Useful for:

-   availability
-   read scaling
-   failover

But replicas may lag.

``` text
Primary = 97
Replica = 100
```

A correctness-critical read from a lagging replica can be dangerous.

------------------------------------------------------------------------

## 7. CDN / Edge Cache

``` text
User
 |
 v
Nearest CDN
 |
 v
Origin
```

Excellent for:

-   images
-   videos
-   JS/CSS
-   static pages
-   cacheable catalog data

Poor fit for rapidly changing correctness-critical inventory.

A CDN can safely show:

``` text
"iPhone available"
```

as a presentation hint, but the purchase path must perform an
authoritative availability/reservation operation.

------------------------------------------------------------------------

## 8. Cache Warming

Populate cache before traffic arrives.

``` text
DB
 |
 v
Warm-up job
 |
 v
Redis
```

Useful before:

-   flash sales
-   major events
-   known product launches

Avoids a massive initial cache-miss wave.

### Trade-off

Warm only predictable/high-value data; warming everything wastes memory
and DB resources.

------------------------------------------------------------------------

## 9. Cache Architecture Decision

  Requirement                     Good choice
  ------------------------------- -----------------------------
  Ultra-low latency local reads   L1
  Shared cache                    Redis/L2
  Static global content           CDN
  Very hot predictable keys       L1 + Redis
  Flash-sale preparation          Cache warming
  Correctness-critical mutation   Don't rely on cache replica

------------------------------------------------------------------------

## 10. Staff-Level Questions

### Q: Why use L1 if Redis is already fast?

To remove network latency and reduce Redis QPS for extremely hot reads.

### Q: What happens if L1 and Redis disagree?

Define invalidation/versioning rules. Never let an eventually stale L1
determine a correctness-critical mutation.

### Q: Does Redis Cluster solve a hot key?

No. A single key still maps to a single shard. Hot-key mitigation
requires application-level strategies such as request coalescing,
admission control, or safe key bucketing where semantics permit.

### Q: Can CDN cache inventory?

Only as a stale presentation/read optimization. Never use a stale CDN
value as the final authorization for a purchase.
