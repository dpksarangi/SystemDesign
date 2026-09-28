# Cache Access Strategies

## 1. Mental Model

Cache access strategies answer:

> **How does the application get data into and out of the cache?**

Assume the DB is the source of truth.

``` text
Application
     |
     v
   Cache
     |
     v
    DB
```

The three core patterns are:

-   Cache-Aside / Lazy Loading
-   Read-Through
-   Refresh-Ahead

------------------------------------------------------------------------

## 2. Cache-Aside / Lazy Loading

The application owns cache population.

### Read

``` text
Client
  |
  v
Application
  |
  v
Redis
  |
  +---- HIT ----> return
  |
  +---- MISS
          |
          v
         DB
          |
          v
       Redis SET
          |
          v
        return
```

### Write

Common approach:

``` text
Application
   |
   +----> DB update
   |
   +----> invalidate Redis
```

On the next read, the cache is repopulated.

### Example

``` text
Redis: user:123 -> old profile

UPDATE DB
DELETE user:123 from Redis

Next GET:
Redis MISS
  -> DB
  -> Redis SET
```

### Advantages

-   Simple and widely applicable
-   Only frequently accessed data enters cache
-   Cache can be rebuilt from DB
-   Cache failure does not necessarily lose data
-   Application has explicit control

### Problems

#### Stale cache

``` text
DB    = NEW
Redis = OLD
```

if invalidation fails.

#### Cache stampede

A hot key expires and thousands of requests all hit DB.

#### Race conditions

Concurrent reads/writes can cause an old value to be reinserted into
cache.

### Best use cases

-   Product/catalog data
-   User profiles
-   Configuration
-   Read-heavy APIs
-   Data where occasional staleness is acceptable

------------------------------------------------------------------------

## 3. Read-Through

The application asks the cache for data. The cache is responsible for
loading the DB on a miss.

``` text
Application
     |
     v
   Cache
     |
     +---- HIT ----> return
     |
     +---- MISS
            |
            v
        Cache Loader
            |
            v
           DB
            |
            v
          Cache
            |
            v
          return
```

### Difference from Cache-Aside

                            Cache-Aside   Read-Through
  ------------------------- ------------- ------------------
  Cache miss logic          Application   Cache layer
  DB loading                Application   Cache/provider
  Application awareness     High          Lower
  Operational abstraction   Simple        More centralized

Read-through is often provided by a caching library/platform rather than
manually implemented.

### Advantages

-   Centralized cache loading
-   Less repeated cache-loading code
-   Consistent behavior across applications

### Problems

-   Tighter coupling between cache and data source
-   Failure behavior needs careful design
-   Not every cache technology provides native read-through

------------------------------------------------------------------------

## 4. Refresh-Ahead

Refresh the cache **before** expiration so users don't encounter a cache
miss.

``` text
TTL nearly expired
       |
       v
Background refresh
       |
       v
      DB
       |
       v
   Redis SET
```

Example:

``` text
TTL = 10 minutes

09:00 cache populated
09:08 refresh begins
09:09 new value available
09:10 old TTL would have expired
```

### Advantages

-   Avoids latency spikes caused by cache misses
-   Useful for extremely hot keys
-   Good when access patterns are predictable

### Problems

-   Can refresh data nobody requests
-   Requires refresh scheduling
-   Still needs stampede protection
-   More background traffic

------------------------------------------------------------------------

## 5. Comparison

  Strategy        Who loads cache?         Typical use
  --------------- ------------------------ ---------------------------
  Cache-Aside     Application              General-purpose
  Read-Through    Cache                    Centralized data access
  Refresh-Ahead   Background/cache layer   Very hot predictable data

### Decision Rule

Use **Cache-Aside** as the default unless there is a strong reason for
another model.

Use **Read-Through** when cache loading should be abstracted away.

Use **Refresh-Ahead** when cache misses for hot keys are expensive and
access patterns are predictable.

------------------------------------------------------------------------

## 6. Interview Questions

### Q: Why is Cache-Aside so common?

Because it keeps DB ownership clear, is simple to operate, and allows
the cache to be treated as a rebuildable derived layer.

### Q: What happens if Redis is down?

For ordinary Cache-Aside data, fall back to DB if the DB can absorb the
traffic. For hot systems, protect DB with rate limiting, request
coalescing, or controlled degradation.

### Q: What if 100K requests miss the same key?

Use single-flight/request coalescing, locking, TTL jitter, or
refresh-ahead so only a small number of requests load the DB.

### Q: Is Cache-Aside a consistency model?

No. It is an access/population strategy. Consistency is a separate
design decision.
