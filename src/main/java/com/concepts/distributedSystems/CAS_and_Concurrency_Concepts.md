# CAS and Concurrency Control

## 1. CAS — Compare-And-Swap / Compare-And-Set

CAS atomically performs:

```text
if currentValue == expectedValue:
    currentValue = newValue
    return SUCCESS
else:
    return FAILURE
```

Example:

```java
AtomicInteger counter = new AtomicInteger(10);

counter.compareAndSet(10, 11);
```

If the value is still `10`, it becomes `11`. If another thread changed it first, the operation fails.

### Mental model

> **"Change this only if nobody changed it since I observed it."**

CAS is an **optimistic concurrency primitive**.

---

# 2. CAS vs Common Concurrency Mechanisms

| Mechanism | Core idea | Blocks? | Best suited for | Main drawback |
|---|---|---:|---|---|
| **CAS** | Change only if expected value/state is unchanged | No | Short atomic state transitions | Retries/contention |
| **Optimistic Locking** | Read version, update only if version is unchanged | No | DB entities with concurrent updates | Failed updates need retry/reconciliation |
| **Pessimistic Locking** | Acquire lock before modifying data | Yes | Strongly serialized critical sections | Blocking, deadlocks, lock contention |
| **Atomic Transaction** | Multiple operations succeed/fail as one unit | Usually transaction-dependent | Maintaining multi-step invariants | More expensive / lower concurrency |
| **Distributed Lock** | One owner at a time across processes/nodes | Usually yes/logically exclusive | Cross-process coordination | Failure/expiry/lease complexity |
| **`synchronized` / Mutex** | One thread enters critical section at a time | Yes | In-process critical sections | Blocking and contention |
| **`volatile`** | Provides visibility/order, not compound-operation atomicity | No | Simple shared state / flags | `x++` is still not atomic |

---

# 3. CAS vs Optimistic Locking

These are closely related but not identical.

### CAS

Usually a low-level atomic primitive:

```text
expected = 10
CAS(10 → 11)
```

### Optimistic locking

A higher-level concurrency pattern, commonly using a version:

```text
Read:
value = X
version = 5

Update:
WHERE version = 5
SET value = Y,
    version = 6
```

If another writer changed version 5 → 6, the update fails.

### Relationship

> **CAS is a primitive; optimistic locking is a concurrency-control pattern that can use CAS-like conditional updates.**

In a database, a conditional update such as:

```sql
UPDATE driver
SET status = 'RESERVED'
WHERE driver_id = 123
  AND status = 'AVAILABLE';
```

has CAS-like semantics.

---

# 4. CAS vs Pessimistic Locking

### CAS — optimistic

```text
Read expected state
       ↓
Try atomic update
       ↓
Success / Failure
       ↓
Retry if needed
```

Nobody is blocked waiting for a lock.

### Pessimistic lock

```text
Acquire lock
       ↓
Modify
       ↓
Release lock
```

Other writers wait.

### Mental model

**CAS:**

> "I'll try. If someone beat me, I'll retry."

**Pessimistic lock:**

> "Nobody else can touch this until I'm done."

---

# 5. CAS vs Atomic Transaction

These solve different problems.

### CAS

Protects a **single atomic state transition**:

```text
AVAILABLE → RESERVED
```

### Transaction

Guarantees a group of operations behaves as one unit:

```text
Debit Account A
Credit Account B
Create Ledger Entry
```

Either the transaction commits or rolls back.

So:

> **CAS gives atomic conditional state change; a transaction provides atomicity across multiple operations.**

A transaction may internally use locks, MVCC, optimistic concurrency, etc.

They are not competing alternatives in every situation.

---

# 6. CAS vs Distributed Lock

CAS is usually ideal when the invariant can be represented as an atomic state transition.

Example:

```text
Driver:
AVAILABLE → RESERVED
```

A distributed lock is useful when multiple operations across resources/processes must be coordinated:

```text
Acquire distributed lock
       ↓
Read multiple resources
       ↓
Perform coordinated work
       ↓
Release lock
```

Prefer the smallest coordination mechanism that protects the invariant.

Don't introduce a distributed lock merely because multiple services exist.

---

# 7. Uber Example

Requirement:

> A driver must not be assigned to two active rides.

Two matching workers see:

```text
Driver 123 = AVAILABLE
```

Both try:

```text
AVAILABLE → RESERVED
```

Using an atomic conditional update:

```text
Worker A → SUCCESS
Worker B → FAILURE
```

The invariant is preserved.

Then:

```text
RESERVED
   ↓
notify driver
   ↓
ACCEPT
   ↓
ASSIGNED
```

Use a short TTL for the reservation so a crashed worker or non-responsive driver does not permanently hold the driver.

### Important

Do **not** hold a pessimistic lock while waiting for the driver to accept.

Reserve atomically, release the lock/transaction immediately, then wait asynchronously.

---

# 8. CAS and Java `Atomic*`

Java's `AtomicInteger`, `AtomicLong`, `AtomicReference`, etc. expose atomic operations commonly implemented using hardware atomic primitives such as CAS.

Example:

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();

count.compareAndSet(0, 1);
```

Think:

```text
AtomicInteger
      |
      +── CAS
      +── atomic increment
      +── atomic update
```

`volatile` is different:

```java
volatile int count;

count++;
```

`volatile` provides visibility/order guarantees, but `count++` is still a read-modify-write sequence and is not atomic.

---

# 9. CAS Retry / CAS Loop

CAS failure does not necessarily mean failure of the business operation.

Example:

```text
Thread A: read 10
Thread B: read 10

Thread A: CAS(10 → 11) → SUCCESS
Thread B: CAS(10 → 11) → FAILURE

Thread B: read 11
Thread B: CAS(11 → 12) → SUCCESS
```

This is a **CAS retry loop**.

Tradeoff:

> CAS avoids blocking, but under extreme contention repeated retries can consume CPU.

---

# 10. ABA Problem

CAS usually checks:

```text
A → C
```

But another thread could do:

```text
A → B → A
```

CAS sees `A` again and may incorrectly assume nothing changed.

Use a version/stamp when this matters:

```text
A, version 1
A → B, version 2
B → A, version 3

CAS(A, version 1 → C) → FAIL
```

Java provides `AtomicStampedReference` for this type of scenario.

---

# 11. Interview Cheat Sheet

### "Is CAS optimistic locking?"

> CAS is an optimistic concurrency primitive. Optimistic locking is a broader concurrency-control pattern, often implemented using a version/conditional update.

### "CAS vs pessimistic lock?"

> CAS doesn't block; it attempts an atomic conditional update and retries on conflict. Pessimistic locking serializes access by making competing operations wait.

### "CAS vs transaction?"

> CAS protects an atomic state transition. A transaction makes multiple operations commit or roll back as one unit. They can be used together.

### "CAS vs distributed lock?"

> CAS is preferable for a simple atomic state transition. A distributed lock is useful when coordination spans multiple operations/resources and cannot be expressed as one atomic conditional update.

### "CAS vs volatile?"

> `volatile` provides visibility and ordering guarantees; CAS provides an atomic compare-and-update operation. `volatile x++` is not atomic.

---

# 12. The Big Picture

```text
                 Concurrency Control
                         |
       +-----------------+------------------+
       |                 |                  |
   Optimistic        Pessimistic        Transaction
       |                 |                  |
       |                 |          Multiple operations
       |                 |          as one atomic unit
       |
   CAS / Version
   Conditional Update
       |
       +-------------------------+
                                 |
                       "Did state change?"
                                 |
                         +-------+-------+
                         |               |
                       YES              NO
                         |               |
                      Retry           Commit
```

### One-line memory aid

> **CAS = "I won't overwrite your change."**  
> **Optimistic lock = "I'll update only if my version is still current."**  
> **Pessimistic lock = "I'll stop you while I work."**  
> **Transaction = "These operations succeed or fail together."**
