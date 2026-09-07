# PACELC

## 1. What is PACELC?

PACELC extends CAP theorem by describing the consistency trade-off not
only during a network partition, but also during normal operation.

**PACELC:**

> If there is a **Partition (P)**, choose between **Availability (A)**
> and **Consistency (C)**; **Else (E)**, choose between **Latency (L)**
> and **Consistency (C)**.

``` text
                  Network Partition?
                    /           \
                  YES             NO
                   |               |
                 A or C           L or C
```

The important addition is the **ELC** part: even when the network is
healthy, distributed systems often trade consistency for latency.

------------------------------------------------------------------------

## 2. Why CAP Is Not Enough

CAP tells us about the system during a partition:

``` text
P -> A or C
```

But most of the time the system is **not partitioned**.

Consider a multi-region database:

``` text
India DB <---- network ----> US DB
```

For a write, we can:

### Option A --- synchronous cross-region replication

``` text
Client
  |
  v
India
  |
  | synchronous
  v
US
  |
  v
ACK
```

This gives stronger consistency but adds cross-region latency.

### Option B --- asynchronous replication

``` text
Client
  |
  v
India
  |
  +---- ACK immediately
  |
  +---- async ----> US
```

This reduces latency but permits temporary inconsistency.

CAP does not describe this normal-operation choice very well.

PACELC does.

------------------------------------------------------------------------

## 3. PACELC Dimensions

### P --- Partition

A network partition means nodes cannot reliably communicate.

The system must choose between:

-   **Availability** --- continue accepting operations
-   **Consistency** --- refuse/limit operations until agreement can be
    restored

### E --- Else

When there is no partition, the system still has a trade-off.

### L --- Latency

To obtain stronger consistency across replicas, the system may need
coordination or synchronous replication.

### C --- Consistency

The system may wait for more replicas, quorum, or a global ordering
before acknowledging an operation.

------------------------------------------------------------------------

## 4. PACELC Is a Trade-off, Not a Binary Label

Do not think:

> "A system is either consistent or available."

Real systems often expose multiple consistency levels.

For example:

``` text
Eventual
Read-your-writes
Monotonic reads
Causal
Strong
```

A system can also provide different guarantees for different operations.

For a social platform:

``` text
Tweet propagation       -> eventual
User's own tweet        -> read-your-writes
Conversation ordering   -> causal
Payment balance         -> strong
```

The correct consistency model is therefore a **data and business
requirement decision**.

------------------------------------------------------------------------

## 5. How PACELC Changes an HLD

Suppose we have:

``` text
              Global Traffic
                 /      \
                v        v
             India      US
               |         |
              DB        DB
               \         /
                \       /
                 Network
```

### If we choose E -\> C

Writes may require cross-region coordination.

Result:

``` text
Consistency  ↑
Latency      ↑
Complexity   ↑
```

### If we choose E -\> L

Writes can be acknowledged locally and propagated asynchronously.

Result:

``` text
Latency      ↓
Availability ↑
Immediate global consistency ↓
```

This decision can change:

-   replication strategy
-   write acknowledgement
-   read routing
-   quorum requirements
-   failover behavior
-   conflict resolution
-   user-visible consistency

------------------------------------------------------------------------

## 6. Example: Tweeter

For a Twitter-like system:

### Tweets

Tweets are mostly immutable.

We can use asynchronous propagation:

``` text
User
 |
 v
Local Region
 |
 v
Tweet DB
 |
 +---- async ----> Other Regions
```

A follower may see a tweet slightly later.

This is generally acceptable.

### Timeline

Use eventual consistency plus session guarantees where useful.

### Counters

Likes, views, and retweets can often tolerate eventual aggregation. Some
counters can be modeled using CRDT-like techniques.

### Payments

If the platform has financial transactions:

``` text
Strong consistency
+
quorum/consensus
```

may be appropriate.

The key point:

> PACELC helps us justify why different parts of the same system can
> make different consistency choices.

------------------------------------------------------------------------

## 7. PACELC vs CAP

  Question                                       CAP                    PACELC
  ---------------------------------------------- ---------------------- -------------
  Partition scenario                             Yes                    Yes
  Availability vs consistency during partition   Yes                    Yes
  Normal operation                               Mostly not addressed   Yes
  Latency vs consistency                         Not the focus          Explicit
  Useful for multi-region design                 Partially              Very useful

CAP answers:

> What happens when the network is partitioned?

PACELC additionally asks:

> What trade-off are we making when everything is working normally?

------------------------------------------------------------------------

## 8. Common Interview Questions

### Q1. Why do we need PACELC if CAP already exists?

Because CAP focuses on behavior during partitions. PACELC additionally
describes the latency-vs-consistency trade-off during normal operation.

### Q2. Does PACELC replace CAP?

No. It extends the CAP discussion.

### Q3. Does eventual consistency automatically mean PA/EL?

No. The label depends on the system's actual failure and
normal-operation choices.

### Q4. Why does strong consistency increase latency?

Because replicas may need coordination before an operation can be
acknowledged or before a read can safely return.

### Q5. Where is PACELC most useful?

Especially in:

-   multi-region databases
-   globally distributed services
-   replicated storage
-   quorum-based systems
-   systems with configurable consistency levels

------------------------------------------------------------------------

## 9. Staff-Level Discussion Points

When designing a system, do not simply say:

> "We need eventual consistency."

Instead ask:

1.  Which data needs strong consistency?
2.  Which data can be stale?
3.  How stale is acceptable?
4.  Is the consistency requirement global or per-user/session?
5.  Can we acknowledge locally?
6.  Do we need cross-region synchronous replication?
7.  What happens during a partition?
8.  What happens when replicas lag?
9.  How are concurrent writes resolved?
10. What latency are we willing to pay for stronger guarantees?

This turns PACELC from a theoretical acronym into an architecture
decision framework.

------------------------------------------------------------------------

## 10. Quick Revision

``` text
PACELC

P -> A / C
E -> L / C
```

**Partition:** Availability vs Consistency

**Else:** Latency vs Consistency

The most important interview insight:

> CAP explains the failure-mode trade-off. PACELC forces us to also
> reason about the consistency-vs-latency trade-off in the normal
> operating path.
