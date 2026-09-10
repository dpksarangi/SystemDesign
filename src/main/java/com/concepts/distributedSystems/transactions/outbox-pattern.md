# Transactional Outbox Pattern

> **Purpose:** Reliably publish events when a service changes its local database, without the database/message-broker dual-write problem.

---

# 1. What is the Outbox Pattern?

The **Transactional Outbox Pattern** stores an event in an **Outbox table in the same database transaction as the business data change**.

A separate relay later publishes that event to a message broker.

```text
Application
    ↓
DB Transaction
    ├── Business Data
    └── Outbox Event
              ↓
          Outbox Relay
              ↓
            Kafka
```

### Core idea

> **Make the database update and event creation atomic locally; publish the event asynchronously afterward.**

---

# 2. The Dual-Write Problem

Suppose an Order Service does:

```text
1. INSERT order
2. Publish OrderCreated to Kafka
```

These are two independent operations.

Failure:

```text
INSERT order ✓
      ↓
Kafka publish
      ↓
💥 Service crashes
```

Result:

```text
Database: Order exists
Kafka:    OrderCreated missing
```

The Saga may now be stuck because the next service never receives the event.

---

# 3. Why Not Publish First?

Reverse the order:

```text
Kafka publish ✓
      ↓
Database write
      ↓
💥 Service crashes
```

Now:

```text
Kafka:    OrderCreated exists
Database: Order does not exist
```

Consumers may process an event for data that was never committed.

Therefore:

> **The problem is not simply "which one comes first?" The problem is that DB and broker are separate transactional systems.**

---

# 4. Transactional Outbox Solution

Instead of publishing directly to Kafka:

```text
BEGIN TRANSACTION

INSERT order

INSERT outbox event

COMMIT
```

Example conceptual schema:

```text
orders
--------------------------------
id
customer_id
status
created_at

outbox
--------------------------------
event_id
aggregate_id
event_type
payload
created_at
published_at
```

Now:

```text
Order write ✓
Outbox write ✓
     ↓
   COMMIT
```

or:

```text
Order write ✗
Outbox write ✗
     ↓
  ROLLBACK
```

The application never has a committed business change without its corresponding outbox record.

---

# 5. Outbox Architecture

```text
                  Order Service
                       │
                       ↓
                ┌─────────────┐
                │  Order DB   │
                │             │
                │ orders      │
                │ outbox      │
                └──────┬──────┘
                       │
                       ↓
                 Outbox Relay
                       │
                       ↓
                     Kafka
                       │
            ┌──────────┼──────────┐
            ↓          ↓          ↓
        Payment    Inventory   Analytics
```

The relay is responsible for moving committed outbox records to the broker.

---

# 6. How Does the Relay Work?

Conceptually:

```text
SELECT *
FROM outbox
WHERE published_at IS NULL
ORDER BY created_at
LIMIT N;
```

Then:

```text
for each event:
    publish(event)
```

After successful publication:

```text
UPDATE outbox
SET published_at = ...
WHERE event_id = ...
```

The exact implementation can vary.

---

# 7. Polling Publisher

The simplest relay is a polling process.

```text
Outbox Table
     ↑
     │ poll every N ms
     │
Publisher
     ↓
Kafka
```

### Advantages

- Simple
- Easy to understand
- Works with many databases
- No special database-log infrastructure required

### Disadvantages

- Polling overhead
- Publication latency
- Need efficient indexes
- Concurrent publishers need coordination

Useful index:

```text
(status/published_at, created_at)
```

or an equivalent index appropriate for the DB/query pattern.

---

# 8. CDC-Based Outbox

Instead of polling the table, use **Change Data Capture**.

```text
Application
    ↓
DB Transaction
    ├── Business data
    └── Outbox
          ↓
      Transaction Log
          ↓
          CDC
          ↓
        Kafka
```

For example:

```text
PostgreSQL
    ↓
WAL
    ↓
Debezium / CDC connector
    ↓
Kafka
```

The application writes the outbox record atomically with the business transaction.

CDC reads the database change log and publishes the outbox event.

### Important

CDC is the **transport mechanism** here.

The Outbox pattern is the **atomic event-recording strategy**.

---

# 9. Outbox + Saga

This is one of the most important connections.

Suppose:

```text
Order Service
```

completes the first Saga step.

It needs:

```text
Order = CREATED
OrderCreated event
```

Do:

```text
BEGIN

INSERT order(status=CREATED)

INSERT outbox(
    event_type='OrderCreated',
    aggregate_id='order-123'
)

COMMIT
```

Then:

```text
Outbox
   ↓
CDC / Relay
   ↓
Kafka
   ↓
Payment Service
   ↓
next Saga step
```

Therefore:

> **Saga defines the distributed business workflow; Outbox makes the event publication from each local transaction reliable.**

---

# 10. Delivery Guarantees

Outbox usually provides **at-least-once publication**.

Consider:

```text
Relay
  ↓
Kafka ✓
  ↓
💥 relay crashes
```

The relay may crash before marking the outbox record as published.

After restart:

```text
same event
   ↓
Kafka again
```

Therefore consumers must tolerate duplicates.

---

# 11. Idempotent Consumers

Give every event a unique ID:

```text
event_id = abc123
```

Consumer can maintain processed event IDs:

```text
processed_events
----------------
event_id
processed_at
```

When:

```text
abc123
```

arrives:

```text
if already processed:
    ignore
else:
    process
    record event_id
```

This turns:

```text
At-least-once delivery
```

into reliable business processing through:

```text
At-least-once
+
Idempotency
```

---

# 12. Ordering

Outbox records can have business ordering requirements.

Example:

```text
OrderCreated
OrderPaid
OrderCancelled
```

For a particular aggregate:

```text
order-123
```

we generally want:

```text
Created → Paid → Cancelled
```

Common techniques:

### Partition by aggregate ID

```text
Kafka key = order_id
```

All events for that order go to the same partition.

### Sequence/version

Include:

```text
aggregate_id
sequence_number
```

Example:

```text
order-123, sequence=1
order-123, sequence=2
order-123, sequence=3
```

Consumers can reject/defer unexpected versions.

### Important

> **Outbox does not automatically guarantee global event ordering.**

---

# 13. Outbox Cleanup

Outbox tables can grow indefinitely.

After events are safely published and retained for the required operational window:

```text
Published events
       ↓
Retention period
       ↓
Archive / delete
```

Possible approaches:

- Time-based deletion
- Partitioned outbox table
- Archival
- Separate retention policy

Be careful not to delete records before the system's recovery/replay requirements are satisfied.

---

# 14. Scaling the Outbox

At high throughput:

```text
Outbox
   ↓
Publisher 1
Publisher 2
Publisher 3
Publisher 4
```

Now publishers can race for the same rows.

Use database mechanisms such as:

- row locking
- `SKIP LOCKED` where supported
- leasing/claiming
- partitioning
- sharding
- multiple outbox partitions

The goal is:

```text
No duplicate work where avoidable
+
No single publisher bottleneck
+
Preserve required per-aggregate ordering
```

Duplicates should still be assumed possible.

---

# 15. Poison Events / Failed Publication

A particular event may repeatedly fail publication.

```text
Event A
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
Still failing
```

Don't allow one bad record to block the entire pipeline forever.

Possible strategy:

```text
Retry with backoff
      ↓
Maximum retries
      ↓
Failed/Dead-letter state
      ↓
Alert
      ↓
Manual repair / replay
```

Exactly how this works depends on the relay and broker architecture.

---

# 16. Outbox Does Not Solve Everything

Outbox solves:

```text
Business DB update
        +
Event creation
```

being atomic.

It does **not** automatically solve:

- Duplicate delivery
- Consumer failures
- Business compensation
- Global transactions
- Global ordering
- Consumer-side concurrency
- Reconciliation
- Schema evolution

Think of Outbox as one building block.

---

# 17. Outbox vs Direct Publishing

### Direct

```text
DB
 ↓
Kafka
```

Risk:

```text
DB ✓
Kafka ✗
```

### Outbox

```text
DB transaction
 ├── business data
 └── outbox
       ↓
    relay/CDC
       ↓
     Kafka
```

Much safer because the event record is committed with the business change.

---

# 18. Outbox vs CDC

These are related but different concepts.

### Outbox

A **pattern**:

```text
Business transaction
       +
Outbox event
```

### CDC

A **change-capture mechanism**:

```text
Database change log
       ↓
CDC
       ↓
Consumers / Kafka
```

They can be combined:

```text
Transactional Outbox
        ↓
       CDC
        ↓
      Kafka
```

### Key distinction

> **Outbox decides what business event should exist. CDC provides a reliable way to observe and transport the database change.**

---

# 19. Why Not Just CDC the Business Tables?

You could capture:

```text
orders UPDATE
```

directly.

But internal database changes don't always map cleanly to domain events.

Example:

```text
DB:
status = PAID
```

The desired business event might be:

```text
OrderPaymentCompleted
```

The Outbox lets the application explicitly define:

```text
event_type
payload
aggregate_id
```

rather than exposing internal database structure as the event contract.

---

# 20. Common Interview Traps

### Trap 1

> Outbox gives exactly-once delivery.

**Wrong.**

Usually design for at-least-once + idempotency.

### Trap 2

> Outbox removes all duplicates.

**Wrong.**

Relay crashes can produce duplicate publication.

### Trap 3

> CDC and Outbox are the same thing.

**Wrong.**

Outbox is a pattern; CDC is a change-capture mechanism.

### Trap 4

> Outbox means polling.

**Wrong.**

Polling and CDC are two common ways to relay outbox events.

### Trap 5

> Outbox makes the entire distributed transaction atomic.

**Wrong.**

It makes the **local DB change + event record** atomic.

The overall Saga remains distributed.

---

# 21. Staff-Level Mental Model

When designing Outbox, ask:

```text
1. What dual-write problem are we solving?
2. What belongs in the local DB transaction?
3. What does the outbox schema look like?
4. Polling or CDC?
5. How do we handle relay crashes?
6. What delivery guarantee do we provide?
7. How do consumers handle duplicates?
8. What ordering is required?
9. How do we scale publishers?
10. How do we handle poison events?
11. How long do we retain outbox records?
12. How do we replay failed events?
13. How do we evolve event schemas?
14. How does this integrate with Saga?
```

---

# 22. One-Line Summary

> **Transactional Outbox atomically records a business change and the event describing that change in the same local database transaction, then asynchronously relays the event to the broker—typically with at-least-once delivery and idempotent consumers.**
