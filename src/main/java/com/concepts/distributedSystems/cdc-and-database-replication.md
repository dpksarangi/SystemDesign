# Database Replication & CDC

## 1. Database Replication

Replication means maintaining copies of data across multiple nodes.

Typical architecture:

``` text
              Application
                   |
                   v
             Primary / Leader
              /          \
             v            v
         Replica A     Replica B
```

Replication can be:

-   synchronous
-   asynchronous
-   single-leader
-   multi-leader
-   leaderless

The exact model depends on the database.

------------------------------------------------------------------------

## 2. Native Database Replication

Modern databases commonly provide replication as a built-in capability.

Examples of underlying mechanisms include:

-   PostgreSQL WAL
-   MySQL binary log
-   MongoDB oplog
-   database-specific distributed replication protocols

You normally do **not** write an application service just to copy every
database write from the primary to its replicas.

The database's replication mechanism handles that.

Conceptually:

``` text
Primary
  |
  | transaction/replication log
  v
Replica
```

------------------------------------------------------------------------

## 3. Synchronous Replication

The writer waits for required replicas before acknowledging.

``` text
Client
  |
  v
Primary
  |
  | sync
  v
Replica
  |
  v
ACK
```

### Benefits

-   stronger durability/consistency
-   smaller replication lag

### Costs

-   higher latency
-   availability can be affected when required replicas are unavailable
-   cross-region synchronization can be expensive

------------------------------------------------------------------------

## 4. Asynchronous Replication

The primary can acknowledge before replicas catch up.

``` text
Client
  |
  v
Primary
  |
  +---- ACK
  |
  +---- async ----> Replica
```

### Benefits

-   lower write latency
-   better tolerance for replica lag
-   useful for geo-distributed systems

### Costs

-   stale reads
-   replication lag
-   possible data loss depending on acknowledgement/durability semantics
    and failure model

------------------------------------------------------------------------

## 5. Replication Lag

If:

``` text
Primary → version 1050
Replica → version 1030
```

lag exists.

A read from the replica can return stale state.

Replication lag can be caused by:

-   network delays
-   replica overload
-   slow disk
-   large transactions
-   downstream backpressure
-   failures/retries

A robust design must decide what to do when lag becomes too high.

------------------------------------------------------------------------

## 6. Transaction Logs

Databases commonly maintain a durable record of changes.

Examples:

``` text
PostgreSQL -> WAL
MySQL     -> binlog
MongoDB   -> oplog
```

Conceptually:

``` text
Application
    |
    v
Database
    |
    v
Durable change log
```

The database can use this log for recovery and/or replication.

------------------------------------------------------------------------

# CDC --- Change Data Capture

## 7. What Is CDC?

CDC stands for **Change Data Capture**.

It means:

> Capture changes happening in a database and expose them to downstream
> systems.

Example:

``` text
Application
    |
    v
Database
    |
    | transaction log
    v
CDC
    |
    v
Kafka / event stream
    |
    +----> Search
    +----> Analytics
    +----> Cache invalidation
    +----> Data warehouse
```

------------------------------------------------------------------------

## 8. How CDC Works

Suppose:

``` sql
UPDATE users
SET name = 'DK'
WHERE id = 1;
```

The database records the transaction in its log.

A CDC process reads that log and turns the change into an event
conceptually like:

``` json
{
  "operation": "UPDATE",
  "table": "users",
  "key": 1,
  "after": {
    "name": "DK"
  }
}
```

The event can then be published to an event stream.

A popular open-source CDC technology is Debezium, but the important
concept is the pattern, not the product.

------------------------------------------------------------------------

## 9. Why Use CDC Instead of Application Events?

A tempting design is:

``` text
Application
  |
  +----> DB
  |
  +----> Kafka
```

This creates a **dual-write problem**.

Failure scenario:

``` text
1. DB update succeeds
2. Kafka publish fails
```

Now the database changed but downstream systems never learned about it.

The reverse can also happen.

With CDC:

``` text
Application
    |
    v
Database
    |
    v
Durable transaction log
    |
    v
CDC
    |
    v
Kafka
```

The database remains the source of truth for the change.

------------------------------------------------------------------------

## 10. CDC vs Database Replication

They are related but different.

### Database replication

Primary purpose:

> Keep another database copy synchronized.

``` text
DB A
 |
 v
DB B
```

Usually implemented by database-native mechanisms.

### CDC

Primary purpose:

> Make database changes available to downstream systems.

``` text
DB
 |
 v
CDC
 |
 v
Kafka
 |
 +--> Search
 +--> Analytics
 +--> Cache
```

CDC can be used as part of a replication pipeline, but CDC is broader
than replica synchronization.

------------------------------------------------------------------------

## 11. CDC and Eventual Consistency

CDC can be one mechanism for building an eventually consistent system:

``` text
DB A
 |
CDC
 |
Kafka
 |
Consumer
 |
DB B
```

But CDC itself does not define the consistency model.

The resulting consistency depends on:

-   event ordering
-   delivery semantics
-   consumer behavior
-   retry handling
-   replication lag
-   conflict resolution

So:

> **Consistency model = guarantee**

> **CDC = change propagation mechanism**

------------------------------------------------------------------------

## 12. Important Failure Cases

### Consumer is down

Events should remain available in the durable stream so the consumer can
catch up.

### Consumer is slow

Lag increases.

``` text
Produced: 1050
Consumed: 1000
Lag: 50
```

### Duplicate delivery

Consumers should ideally be idempotent.

### Out-of-order processing

If ordering matters, the system needs an appropriate partitioning/order
strategy.

### Schema changes

CDC pipelines need a strategy for schema evolution.

------------------------------------------------------------------------

## 13. HLD Decision Guide

Ask:

### Do I only need database replicas?

Prefer native database replication when it satisfies the requirement.

### Do multiple systems need to react to database changes?

Consider:

``` text
DB -> CDC -> Kafka -> Consumers
```

### Do I need custom transformation or cross-system replication?

Use downstream consumers/services after CDC.

------------------------------------------------------------------------

## 14. Interview Questions

-   What is database replication?
-   Synchronous vs asynchronous replication?
-   What causes replication lag?
-   How do replicas know what they missed?
-   What is WAL/binlog/oplog?
-   What is CDC?
-   CDC vs database replication?
-   Why not publish Kafka events directly from the application?
-   What is the dual-write problem?
-   How do you handle duplicate CDC events?
-   What happens if a CDC consumer falls behind?
-   Can CDC guarantee consistency by itself?

------------------------------------------------------------------------

## 15. Quick Revision

``` text
Database Replication:

DB
 |
 | native replication
 v
Replica
```

``` text
CDC:

DB
 |
 | transaction/change log
 v
CDC
 |
 v
Kafka / Stream
 |
 +--> Search
 +--> Analytics
 +--> Cache
```

Remember:

> **Replication keeps copies synchronized.**

> **CDC exposes database changes to other systems.**
