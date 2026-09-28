# High-Throughput Inventory Management

## 1. Why Inventory Is a Reusable HLD Primitive

Ticketing, e-commerce, food delivery, hotels, ride capacity and
appointment systems all contain some form of:

``` text
ADD
RESERVE
RELEASE
CONFIRM / CONSUME
CANCEL
ADJUST
```

The hard problem is not simply high traffic.

It is:

> **How do we process massive concurrent mutations while never
> allocating more inventory than exists?**

------------------------------------------------------------------------

## 2. Example: Flash Sale

Assume:

``` text
iPhone stock = 10,000
Users clicking BUY = 1,000,000
```

Do not send 1M requests directly to the inventory DB.

Use admission control/waiting room to reduce the mutation rate.

``` text
1M users
   |
   v
Admission / Waiting Room
   |
   v
Inventory Service
   |
   v
Redis atomic reservation
   |
   v
Order Saga
```

The waiting room solves traffic absorption.

The inventory layer solves correctness.

------------------------------------------------------------------------

## 3. Core Principle

Separate:

``` text
Traffic control
    ↓
Waiting Room / Queue
```

from:

``` text
Inventory correctness
    ↓
Atomic reserve/release
```

Kafka can smooth traffic, but Kafka alone does not prevent overselling.

------------------------------------------------------------------------

## 4. Redis + DB Model

A useful model for high-throughput inventory is:

``` text
DB
└── Durable inventory/order state

Redis
└── Hot-path reservation state
```

Do not think of Redis as merely a read cache.

It participates in the inventory state machine.

------------------------------------------------------------------------

## 5. Reserve Operation

Never do:

``` text
GET available
   |
   v
if available >= quantity
   |
   v
DECR
```

This has a race.

Instead:

``` text
ATOMIC:

if available >= quantity:
    available -= quantity
    create reservation
else:
    reject
```

Example:

``` text
Initial = 100

Reserve 5
100 -> 95

Reserve 4
95 -> 91
```

Return:

``` text
reservation_id
expires_at
```

------------------------------------------------------------------------

## 6. Reservation State Machine

``` text
             AVAILABLE
                 |
               RESERVE
                 |
                 v
             RESERVED
              /     \
             /       \
        PAYMENT       TTL/CANCEL
           |              |
           v              v
       CONFIRMED       AVAILABLE
```

Every transition must be idempotent.

------------------------------------------------------------------------

## 7. TTL Is for the Reservation

Do not think:

``` text
TTL expired
   |
   v
Redis magically increments inventory
```

Think:

``` text
Reservation R123
quantity = 3
expires_at = T+5m
status = RESERVED
```

When it expires:

``` text
RESERVED -> EXPIRED
```

and atomically:

``` text
available += 3
```

The expiration mechanism invokes the same inventory release operation
used by cancellation.

The important design principle:

> **TTL determines when a reservation becomes eligible for expiration;
> the inventory state transition performs the release.**

------------------------------------------------------------------------

## 8. Seat-Level Inventory

For ticketing:

``` text
A1
A2
A3
...
```

Maintain individual seat state plus aggregate counters.

Example:

``` text
Total = 100

Available = 90
Reserved  = 5
Sold      = 5
```

Invariant:

``` text
Total = Available + Reserved + Sold
```

Reserve 3 seats:

``` text
Available: 90 -> 87
Reserved:   5 -> 8
```

Confirm:

``` text
Reserved: 8 -> 5
Sold:     5 -> 8
```

Expire:

``` text
Reserved: 8 -> 5
Available:87 -> 90
```

The seat-level transition and aggregate counter update should happen
atomically in the Redis state transition.

Do not periodically count all seats to derive the normal counter.

------------------------------------------------------------------------

## 9. Saga Integration

Reservation happens first:

``` text
Redis
  |
  | reserve
  v
reservation_id
  |
  v
Order Saga
```

If Saga succeeds:

``` text
RESERVED -> CONFIRMED
```

If Saga fails/cancels:

``` text
RESERVED -> RELEASED
```

The release operation increments available inventory.

Example:

``` text
100
  |
reserve 3
  |
97
  |
Saga fails
  |
100
```

The Inventory Service should own these state transitions.

------------------------------------------------------------------------

## 10. DB Persistence

DB is the durable representation.

A reservation/confirmation can be persisted asynchronously where
business requirements allow:

``` text
Redis atomic state transition
        |
        v
durable event / persistence path
        |
        v
DB
```

The critical design question is:

> How do we guarantee that a successful Redis state transition is
> recoverable if Redis fails before DB persistence?

Possible mechanisms:

-   Redis persistence/replication
-   durable event/log
-   reservation record
-   idempotent persistence
-   replay
-   reconciliation

The exact mechanism depends on the required durability guarantee.

------------------------------------------------------------------------

## 11. Redis and DB Must Not Be Treated as Identical Counters

Bad mental model:

``` text
Redis = 97
DB    = 100

"Redis is stale!"
```

For stateful inventory, the values may represent different stages.

A better model:

``` text
DB = durable committed state
Redis = current hot-path reservation state
```

Define exactly what each number means.

If both store the same logical counter, synchronization becomes much
harder.

------------------------------------------------------------------------

## 12. Failure: Saga Fails

Example:

``` text
Initial:
Redis = 100
DB = 100

Reserve 3:
Redis = 97

Saga fails:
InventoryService.release(R123)

Redis = 100
```

DB does not need a compensating inventory decrement if the reservation
was never durably committed as sold/consumed.

This is why reservation and committed inventory should be modeled as
distinct states.

------------------------------------------------------------------------

## 13. Failure: Payment vs Expiration Race

At T+5 minutes:

``` text
Payment -> CONFIRMED
Expiry  -> EXPIRED
```

Both cannot win.

Use an atomic state transition:

``` text
if status == RESERVED:
    status = CONFIRMED
```

or:

``` text
if status == RESERVED
and now >= expires_at:
    status = EXPIRED
    release inventory
```

The first successful transition wins.

The second sees:

``` text
status != RESERVED
```

and becomes a no-op.

------------------------------------------------------------------------

## 14. Idempotency

Every mutation should have a unique operation/reservation ID.

Examples:

``` text
reserve(R123)
release(R123)
confirm(R123)
```

If release is called twice:

``` text
First:
RESERVED -> EXPIRED
inventory += 3

Second:
already EXPIRED
no-op
```

Without idempotency:

``` text
inventory += 3
inventory += 3
```

and inventory becomes corrupted.

------------------------------------------------------------------------

## 15. Hot Inventory

A cluster with many Redis nodes does not automatically solve:

``` text
1M requests
     |
     v
inventory:iphone
     |
     v
one Redis shard
```

The logical inventory key is hot.

Mitigation hierarchy:

``` text
1. Admission control
2. Limit mutation rate
3. Atomic Redis operation
4. Request coalescing where applicable
5. Inventory bucketing/sharding for suitable models
6. Serialized processing for extremely scarce inventory
```

Do not blindly shard a single scarce counter without defining how the
global invariant is preserved.

------------------------------------------------------------------------

## 16. What Each Component Owns

``` text
Waiting Room
  -> controls admission rate

Inventory Service
  -> owns inventory state transitions

Redis
  -> executes high-QPS atomic hot-path state

Order Saga
  -> coordinates order/payment workflow

DB
  -> durable business state

Durable event/log
  -> recovery and propagation

Reconciliation
  -> detects unexpected divergence
```

------------------------------------------------------------------------

## 17. Key Invariants

### No overselling

``` text
allocated <= total inventory
```

### Reservation conservation

``` text
total =
available
+ reserved
+ consumed/sold
```

### One terminal transition

``` text
RESERVED
   -> CONFIRMED
OR -> EXPIRED
OR -> CANCELLED
```

Never two.

### Idempotent operations

Repeated commands must not change the result after the first successful
transition.

------------------------------------------------------------------------

## 18. Staff-Level Design Questions

### Q: Why Redis instead of directly updating DB?

To move extremely high-frequency, low-latency reservation mutations away
from the DB hot path.

### Q: Why not just cache inventory?

Because reservation is a correctness-critical mutation. Redis is
participating in the state transition, not merely accelerating reads.

### Q: What if Redis goes down?

Do not blindly fail over to DB and create two competing authorities. Use
controlled degradation/fail-closed behavior and a defined
recovery/rebuild path.

### Q: What if Redis says reservation succeeded but the response is lost?

Client retries using an idempotency/reservation key. The service returns
the existing reservation rather than allocating another unit.

### Q: What if TTL and payment happen simultaneously?

Atomic compare-and-transition on reservation state. Exactly one terminal
transition wins.

### Q: Why not periodically recalculate available inventory from seats?

Because the counter is a materialized state updated atomically with
every transition. Periodic recounting is a reconciliation mechanism, not
the primary correctness mechanism.

### Q: What if the DB is temporarily behind Redis?

That can be acceptable if Redis owns the hot-path reservation state and
the durability/recovery contract explicitly supports eventual
persistence.

------------------------------------------------------------------------

## 19. Core Takeaway

For high-throughput inventory:

``` text
Massive traffic
      |
      v
Admission control
      |
      v
Atomic Redis reservation
      |
      v
Reservation TTL
      |
      +------ payment ------> CONFIRMED
      |
      +------ cancel -------> RELEASED
      |
      +------ expiry -------> RELEASED
      |
      v
Durable persistence
      |
      v
DB
```

The central Staff-level idea is:

> **Do not solve inventory by putting a counter in Redis. Model
> inventory as atomic state transitions---AVAILABLE → RESERVED →
> CONFIRMED/RELEASED---and make Redis the high-throughput execution
> layer while DB/durable events provide persistence and recovery.**
