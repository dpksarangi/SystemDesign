# Consistency Models

## 1. Why Consistency Models Matter

In a distributed system, multiple replicas may temporarily contain
different versions of data.

The question is not simply:

> "Is the database consistent?"

Instead ask:

> **What consistency guarantee does this operation need?**

Different operations can require different guarantees.

------------------------------------------------------------------------

## 2. Eventual Consistency

### Definition

If updates stop, replicas will eventually converge to the same state.

``` text
T0: A = new     B = old     C = old
T1: A = new     B = new     C = old
T2: A = new     B = new     C = new
```

### Typical implementation

``` text
DB
 |
 | async replication
 v
Replicas
```

Changes may be propagated through:

-   native database replication
-   WAL/binlog/oplog
-   CDC
-   message streams

### What it allows

A read can temporarily return stale data.

### Use cases

-   social feeds
-   analytics
-   search indexes
-   recommendations
-   counters

### Main trade-off

``` text
Latency       ↓
Availability  ↑
Immediate consistency ↓
```

------------------------------------------------------------------------

## 3. Read-Your-Writes

### Definition

After a client's write has been acknowledged, subsequent reads by that
client must reflect that write.

``` text
WRITE X
   |
   v
READ
   |
   v
must see X
```

This does **not** mean every user sees X immediately.

### Implementation options

#### Sticky routing

Route the session to the same replica/leader.

#### Leader reads

Route reads to the leader after writes.

#### Version/token based routing

Write returns:

``` text
commit_version = 1050
```

Subsequent read requires:

``` text
replica_version >= 1050
```

### Important point

RYW can be provided on top of asynchronous replication. It does not
automatically require global strong consistency.

------------------------------------------------------------------------

## 4. Monotonic Reads

### Definition

Once a client has observed version V, future reads must not return a
version older than V.

``` text
10 -> 12 -> 15 -> 15 -> 18
```

Allowed.

``` text
10 -> 12 -> 9
```

Violation.

### Implementation

Track:

``` text
last_seen_version
```

Route reads only to replicas satisfying:

``` text
replica_version >= last_seen_version
```

If no replica qualifies:

-   wait for a replica to catch up
-   read from a leader
-   retry/fail, depending on requirements

### Difference from RYW

RYW protects:

> My writes.

Monotonic reads protect:

> My observed progress through the data.

------------------------------------------------------------------------

## 5. Session Consistency

Session consistency is best thought of as a collection of guarantees
applied to a client's session.

Common session guarantees include:

1.  Read-your-writes
2.  Monotonic reads
3.  Monotonic writes
4.  Writes-follow-reads

Rather than treating "session consistency" as one universal
implementation, specify the guarantees you actually provide.

Example:

``` text
Session
  |
  +-- last_write_version
  |
  +-- last_read_version
```

A subsequent read can require:

``` text
replica_version >= max(last_write_version, last_read_version)
```

------------------------------------------------------------------------

## 6. Causal Consistency

### Definition

Causally related operations must be observed in an order that respects
their causal relationship.

Example:

``` text
Alice posts P1
      |
      v
Bob reads P1
      |
      v
Bob replies P2
```

There is a causal relationship:

``` text
P1 -> P2
```

A replica should not expose P2 while hiding the causal dependency P1.

### How can it be implemented?

Systems can track dependencies using mechanisms such as:

-   logical timestamps
-   Lamport clocks
-   vector clocks
-   version vectors
-   causal metadata

The exact implementation depends on the storage system.

### Important distinction

Causal consistency does not require every unrelated operation to have a
global order.

That is why it can be cheaper than strong global consistency.

------------------------------------------------------------------------

## 7. Strong Consistency

"Strong consistency" is often used loosely. In an interview, clarify the
exact guarantee required.

A common distributed-systems interpretation is that reads observe the
latest completed writes according to a strong global
ordering/serialization.

Conceptually:

``` text
WRITE X
   |
   | committed
   v
READ
   |
   v
must reflect X
```

Strong consistency commonly requires more coordination.

Possible mechanisms include:

-   leader-based replication
-   quorum
-   consensus
-   synchronous replication
-   linearizable reads/writes, depending on the system

### Cost

``` text
Consistency ↑
Coordination ↑
Latency ↑
Availability during some partitions ↓
```

------------------------------------------------------------------------

## 8. Comparison

  -----------------------------------------------------------------------
  Model                   Main guarantee          Typical cost
  ----------------------- ----------------------- -----------------------
  Eventual                Replicas eventually     Low
                          converge                

  Read-your-writes        Client sees own writes  Low--medium

  Monotonic reads         Client never reads      Low--medium
                          older state than        
                          previously seen         

  Session guarantees      Defined consistency     Medium
                          guarantees within a     
                          session                 

  Causal                  Causal ordering         Medium
                          preserved               

  Strong                  Strong global           Highest coordination
                          ordering/visibility     
  -----------------------------------------------------------------------

These are not simply six mutually exclusive database modes. Several
guarantees can be composed.

------------------------------------------------------------------------

## 9. Choosing a Model

Ask:

``` text
Is stale data acceptable?
        |
       Yes
        |
Can the user's own writes be stale?
        |
       No -> RYW
        |
Do reads need to move only forward?
        |
       No -> Eventual may be enough
        |
Do causal relationships matter?
        |
       Yes -> Causal
```

For critical financial state:

``` text
Strong consistency
```

For social feeds:

``` text
Eventual + selected session guarantees
```

------------------------------------------------------------------------

## 10. HLD Impact

A consistency decision can change:

-   database topology
-   read/write routing
-   replication mode
-   quorum
-   acknowledgement strategy
-   client/session metadata
-   conflict resolution
-   failover behavior
-   latency

Consistency should therefore be treated as an **architecture decision**,
not merely a database property.

------------------------------------------------------------------------

## 11. Interview Questions

-   Explain eventual consistency.
-   Difference between eventual consistency and strong consistency?
-   How would you implement read-your-writes?
-   How do monotonic reads differ from read-your-writes?
-   Can you provide RYW while retaining asynchronous replication?
-   How would you preserve causal ordering?
-   What happens when no replica is sufficiently caught up?
-   Which consistency model would you choose for a social feed?
-   Which would you choose for a payment balance?
