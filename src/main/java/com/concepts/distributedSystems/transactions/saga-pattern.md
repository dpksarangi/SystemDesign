# Saga Pattern

> **Purpose:** Understand how to manage a long-running business transaction across multiple independent services without using a distributed ACID transaction.

---

## 1. What is the Saga Pattern?

A **Saga** is a sequence of **local transactions**, where each service commits its own transaction independently.

If a later step fails, previously completed steps are reversed through **compensating transactions**.

Example:

```text
Create Order
    ↓
Process Payment
    ↓
Reserve Inventory
    ↓
Create Shipment
```

If inventory reservation fails:

```text
Cancel Payment
    ↓
Cancel Order
```

### Core idea

> **Saga provides business-level consistency across services using local transactions + compensation.**

It does **not** provide ACID atomicity across services.

---

## 2. Why Do We Need Saga?

Suppose an order workflow spans:

```text
Order Service
Payment Service
Inventory Service
Shipping Service
```

A traditional transaction cannot easily span all four independent databases.

A distributed transaction using 2PC can provide stronger atomicity, but introduces coordination, blocking, availability and operational complexity.

Saga instead allows:

```text
Service A → local commit
Service B → local commit
Service C → local commit
```

and handles failures through compensation.

---

## 3. Saga vs 2PC

| | Saga | 2PC |
|---|---|---|
| Transaction model | Business transaction | Distributed transaction |
| Atomicity | Eventual/business consistency | Stronger atomicity |
| Locking | Usually no global locks | Coordinator/participants |
| Failure handling | Compensation | Rollback/commit protocol |
| Availability | Generally better | Can block |
| Long-running workflows | Good | Poor fit |
| Complexity | Business workflow complexity | Infrastructure/protocol complexity |

### Interview line

> **Use Saga when the business process can tolerate intermediate states and has meaningful compensating actions.**

---

# 4. Saga Execution Models

## 4.1 Choreography

There is no central coordinator.

Services react to events.

```text
Order Service
    │
    │ OrderCreated
    ↓
Payment Service
    │
    │ PaymentCompleted
    ↓
Inventory Service
    │
    │ InventoryReserved
    ↓
Shipping Service
```

### Failure

```text
InventoryFailed
      ↓
Payment Service
      ↓
RefundPayment
      ↓
Order Service
      ↓
CancelOrder
```

### Advantages

- No central coordinator
- Services remain autonomous
- Natural fit for event-driven systems
- Can scale well

### Problems

- Workflow becomes difficult to visualize
- Harder to understand ownership
- Event dependencies can become tangled
- Failure handling becomes distributed

### Event-spaghetti warning

```text
A → B
A → C
B → D
C → E
D → F
E → B
```

As workflows grow, choreography can become difficult to reason about.

---

# 5.2 Orchestration

A central **Saga Orchestrator** controls the workflow.

```text
             Saga Orchestrator
              /      |       \
             ↓       ↓        ↓
          Payment Inventory Shipping
```

The orchestrator knows:

```text
1. Charge payment
2. Reserve inventory
3. Create shipment
```

If step 2 fails:

```text
InventoryFailed
       ↓
Orchestrator
       ↓
Refund Payment
       ↓
Cancel Order
```

### Advantages

- Centralized workflow visibility
- Easier failure handling
- Explicit state machine
- Easier to add timeouts/retries
- Better for complex workflows

### Problems

- Orchestrator becomes an important component
- Can become overly centralized if poorly designed
- Requires durable Saga state

---

# 6. Saga State Machine

A complex Saga should be modeled as a state machine.

Example:

```text
CREATED
   ↓
PAYMENT_PENDING
   ↓
PAYMENT_COMPLETED
   ↓
INVENTORY_PENDING
   ↓
INVENTORY_RESERVED
   ↓
SHIPPING_PENDING
   ↓
COMPLETED
```

Failure transitions:

```text
INVENTORY_FAILED
      ↓
REFUND_PAYMENT
      ↓
CANCEL_ORDER
      ↓
FAILED
```

Persist important state:

```text
saga_id
business_id / order_id
current_state
completed_steps
pending_step
retry_count
created_at
updated_at
```

For long-running Sagas, this state must be **durable**.

---

# 7. Compensation

Compensation is not a database rollback.

If:

```text
ChargePayment
```

succeeds, compensation might be:

```text
RefundPayment
```

If:

```text
ReserveInventory
```

succeeds:

```text
ReleaseInventory
```

### Important distinction

> **A compensating transaction is a new business transaction, not an undo operation.**

Real-world side effects may not be perfectly reversible.

For example:

```text
SendEmail
```

cannot truly be rolled back.

---

# 8. Compensation Can Fail

Never assume compensation succeeds.

```text
Payment succeeded
Inventory failed
        ↓
Refund payment
        ↓
       FAIL
```

The system needs:

```text
Retry
  ↓
Retry
  ↓
Retry
  ↓
DLQ / Failed State
  ↓
Alert + Reconciliation / Manual Repair
```

A production Saga therefore needs a **recovery strategy**, not just compensation logic.

---

# 9. Idempotency

Distributed messaging often provides **at-least-once delivery**.

Therefore:

```text
PaymentCompleted
PaymentCompleted
```

may be received twice.

Every Saga operation should have an idempotency key such as:

```text
saga_id
operation_id
event_id
```

Example:

```text
RefundPayment(order123, operationId=abc)
```

If `abc` has already been processed, the retry becomes a no-op.

### Key principle

> **At-least-once delivery + idempotent consumers is a common production design.**

---

# 10. Event Ordering

Ordering is **not a fundamental property of Saga**. It is a messaging/application concern that Saga may depend on.

We usually need:

```text
PaymentCompleted
       ↓
InventoryReserved
       ↓
OrderConfirmed
```

not global ordering across every Saga.

### Kafka approach

Use:

```text
key = saga_id / aggregate_id
```

so events for one Saga go to the same partition.

```text
Partition 3
---------------------------
Order A event 1
Order A event 2
Order A event 3

Partition 7
---------------------------
Order B event 1
Order B event 2
Order B event 3
```

This provides ordering **within the partition**, while allowing different Sagas to execute concurrently.

### Important distinction

> **Message ordering ≠ processing ordering ≠ state-update ordering.**

Consumers can still process messages incorrectly if multiple processing threads or asynchronous work are involved.

### Application-level protection

Use:

- sequence numbers
- aggregate versions
- optimistic concurrency
- state validation
- buffering/retry of future events

Example:

```text
Expected sequence = 2
Received sequence = 3
        ↓
Do not process yet
        ↓
Wait/retry for sequence 2
```

---

# 11. Isolation and Concurrent Sagas

Saga does not provide traditional cross-service transaction isolation.

Example:

```text
Inventory = 1

Saga A → reserve item
Saga B → reserve item
```

Both could race.

Possible solutions:

- Optimistic locking
- Conditional updates
- Version numbers
- Pessimistic locking where appropriate
- Inventory reservations
- Semantic locks

### Key principle

> **Saga handles cross-service workflow consistency; each service still needs local concurrency control.**

---

# 12. Timeouts and Stuck Sagas

A Saga can become stuck:

```text
PaymentCompleted
       ↓
Inventory request
       ↓
No response
       ↓
Saga remains IN_PROGRESS
```

Use:

```text
Timeout
  ↓
Retry / compensate
  ↓
Fail / escalate
```

For long-running workflows, monitor:

```text
IN_PROGRESS duration
```

and detect Sagas that exceed expected execution time.

---

# 13. Reconciliation

Even a well-designed Saga can encounter unexpected states.

Example:

```text
Saga state:
Payment = FAILED

Payment database:
Payment = SUCCESS
```

A reconciliation process periodically checks important business invariants.

```text
Reconciliation Job
       ↓
Compare system states
       ↓
Detect inconsistency
       ↓
Repair / alert
```

This is especially important for financial or inventory systems.

---

# 14. Observability

A Saga may cross many services and asynchronous messages.

Propagate:

```text
saga_id
correlation_id
trace_id
```

Useful metrics:

```text
Saga success rate
Saga failure rate
Compensation rate
Compensation failure rate
Saga duration
Stuck Sagas
Retry count
```

Without correlation IDs, debugging a distributed Saga becomes extremely difficult.

---

# 15. Saga + Outbox

Saga often relies on events to trigger the next step.

That creates the **database + message dual-write problem**.

```text
DB update
   +
Kafka publish
```

These are separate systems.

A crash can produce:

```text
DB ✓
Kafka ✗
```

The **Transactional Outbox Pattern** solves this by writing the business change and event record in the same local DB transaction.

```text
BEGIN

Update Order
Insert Outbox Event

COMMIT
```

Then:

```text
Outbox
   ↓
Relay / CDC
   ↓
Kafka
   ↓
Next Saga step
```

Outbox is not inherently required by Saga, but it is a very common production technique for reliable event-driven Sagas.

---

# 16. Saga + Kafka

A common architecture:

```text
             Saga Orchestrator
                    │
                    ↓
                 Kafka
              /    |    \
             ↓     ↓     ↓
        Payment Inventory Shipping
             │      │      │
             ↓      ↓      ↓
            DB     DB     DB
```

Important supporting concepts:

```text
Kafka
  ├── partitioning → per-Saga ordering
  ├── retries
  ├── at-least-once delivery
  └── durable event transport

Consumers
  ├── idempotency
  ├── state validation
  └── version/sequence checks
```

---

# 17. Exactly-Once Misconception

Saga + Kafka + Outbox does **not** automatically mean exactly-once business processing.

A realistic design is:

```text
At-least-once delivery
        +
Idempotency
        +
Deduplication
        +
Reconciliation
```

The goal is **correct business outcomes**, not magical exactly-once infrastructure.

---

# 18. When Should We Use Saga?

Good fit:

- Long-running workflows
- Multiple independent services
- Each step has a meaningful local transaction
- Business operations have compensating actions
- Eventual consistency is acceptable

Examples:

```text
Order → Payment → Inventory → Shipping

Travel booking
Loan processing
Insurance workflows
E-commerce checkout
```

---

# 19. When NOT to Use Saga

Don't introduce Saga simply because multiple tables/services exist.

Prefer a normal local transaction if the operation can remain within one service/database.

Saga may also be a poor fit when:

- Strict atomicity is mandatory
- Compensation is impossible
- Intermediate inconsistency is unacceptable
- The workflow is actually simple enough for one local transaction

---

# 20. Common Interview Traps

### Trap 1

> Saga = distributed transaction.

**Wrong.**

Saga is a way to coordinate distributed business transactions using local commits and compensation.

### Trap 2

> Compensation = rollback.

**Wrong.**

Compensation is a new business action.

### Trap 3

> Saga guarantees ordering.

**Wrong.**

Messaging/application mechanisms provide ordering.

### Trap 4

> Kafka gives exactly-once Saga execution.

**Wrong.**

Design for duplicates and idempotency.

### Trap 5

> Compensation always succeeds.

**Wrong.**

Compensation needs retries, escalation and reconciliation.

### Trap 6

> Saga solves concurrency.

**Wrong.**

Each service still needs appropriate local concurrency control.

---

# 21. Staff-Level Mental Model

When designing a Saga, ask:

```text
1. What is the business transaction?
2. What are the local transactions?
3. What is the Saga boundary?
4. Choreography or orchestration?
5. What is the state machine?
6. What compensates every successful step?
7. Can compensation itself fail?
8. Are operations idempotent?
9. What happens with duplicate events?
10. What ordering guarantees are required?
11. What happens when a Saga times out?
12. How do we detect stuck Sagas?
13. How do concurrent Sagas interact?
14. How do DB changes and events become atomic?
15. How do we reconcile inconsistent states?
16. How do we observe the entire Saga?
```

---

# 22. One-Line Summary

> **Saga = local transactions + business-level compensation + durable workflow coordination, with idempotency, ordering, retries, timeouts and reconciliation providing the production reliability around it.**
