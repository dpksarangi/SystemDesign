# CRDTs --- Conflict-Free Replicated Data Types

## 1. What Is a CRDT?

A CRDT is a data structure designed so that replicas can accept updates
independently and later merge them deterministically without requiring
coordination for every update.

Core idea:

``` text
Independent writes
       |
       v
Deterministic merge
       |
       v
Replica convergence
```

CRDTs are especially useful for:

-   multi-writer systems
-   offline-first applications
-   collaborative editing
-   distributed counters
-   replicated sets/maps

------------------------------------------------------------------------

## 2. Why Do We Need CRDTs?

Consider:

``` text
US DB       India DB
  |             |
 +1 like       +1 like
```

Both start with:

``` text
10 likes
```

Naive last-write-wins can produce:

``` text
US     -> 11
India  -> 11
Final  -> 11
```

One increment was lost.

A CRDT can represent the independent contributions so both updates
survive.

------------------------------------------------------------------------

## 3. Important Merge Properties

A CRDT merge operation is typically designed to have properties such as:

### Commutative

``` text
merge(A,B) = merge(B,A)
```

Order does not matter.

### Associative

``` text
merge(merge(A,B),C)
=
merge(A,merge(B,C))
```

Grouping does not matter.

### Idempotent

``` text
merge(A,A) = A
```

Duplicate delivery does not change the state.

These properties make distributed retries, reordering, and duplicate
messages much safer.

------------------------------------------------------------------------

## 4. G-Counter

A grow-only counter supports increments.

Each replica maintains its own component:

``` text
{
  US: 5,
  India: 5
}
```

US increments:

``` text
{
  US: 6,
  India: 5
}
```

India independently increments:

``` text
{
  US: 5,
  India: 6
}
```

Merge by taking per-replica MAX:

``` text
US     -> max(6,5) = 6
India  -> max(5,6) = 6
```

Total:

``` text
6 + 6 = 12
```

No update is lost.

------------------------------------------------------------------------

## 5. PN-Counter

To support both increment and decrement, maintain two grow-only
counters:

``` text
P = increments
N = decrements

value = P - N
```

Example:

``` text
P:
{
  US: 10,
  India: 8
}

N:
{
  US: 2,
  India: 1
}
```

Value:

``` text
(10 + 8) - (2 + 1) = 15
```

Both P and N can be merged using the G-Counter approach.

------------------------------------------------------------------------

## 6. G-Set

A grow-only set supports additions:

``` text
Replica A = {A}
Replica B = {B}
```

Merge:

``` text
{A} UNION {B}
=
{A,B}
```

Union is:

-   commutative
-   associative
-   idempotent

The limitation is obvious:

> You cannot remove an element.

------------------------------------------------------------------------

## 7. 2P-Set

Maintain two sets:

``` text
Added
Removed
```

Membership:

``` text
element exists if:
element ∈ Added
AND
element ∉ Removed
```

The limitation is that once an element enters `Removed`, it cannot be
added again under the basic 2P-Set semantics.

------------------------------------------------------------------------

## 8. OR-Set

An Observed-Remove Set gives each addition a unique identity.

Instead of:

``` text
A
```

we track something like:

``` text
A#123
A#456
```

A remove operation removes the additions that the replica has observed.

If another replica concurrently creates a new addition that the remover
has not observed, that concurrent addition can survive.

This provides more useful add/remove semantics for concurrent
operations.

------------------------------------------------------------------------

## 9. CRDT vs Last-Write-Wins

### Last-Write-Wins

``` text
US     -> X
India  -> Y

latest timestamp wins
```

Simple, but a concurrent update may be discarded.

### CRDT

The data structure represents concurrent operations in a way that
permits deterministic merging.

Trade-off:

``` text
CRDT
+
more metadata
+
more implementation complexity
```

------------------------------------------------------------------------

## 10. CRDT Does Not Remove Business Conflicts

CRDTs do not decide what the business meaning of concurrent operations
should be.

Example:

``` text
User A -> REMOVE X
User B -> ADD X
```

The application still needs semantics for that situation.

CRDT provides:

> A deterministic convergence mechanism.

It does not provide:

> Universal business correctness.

------------------------------------------------------------------------

## 11. CRDT vs CDC

These solve different problems.

### CDC

``` text
DB
 |
 v
Change Data Capture
 |
 v
Kafka / stream
 |
 v
Consumers
```

CDC answers:

> How do I capture and propagate database changes?

### CRDT

``` text
Replica A
    \
     -> merge -> converged state
    /
Replica B
```

CRDT answers:

> How do independently modified replicated states merge?

They can be used together.

------------------------------------------------------------------------

## 12. When to Use CRDTs

Good candidates:

-   offline-first applications
-   collaborative applications
-   multi-region multi-writer state
-   counters where increments/decrements can be modeled safely
-   replicated sets/maps
-   systems where coordination would significantly hurt
    latency/availability

Avoid introducing CRDTs merely because a system is distributed.

Ask:

> **Do I actually need coordination-free multi-writer semantics?**

If not, a simpler model may be better.

------------------------------------------------------------------------

## 13. Interview Answer

> "I'd consider a CRDT when I need independently writable replicas or
> offline operation and want updates to converge without central
> coordination. The data structure is designed so concurrent updates can
> be merged deterministically despite reordering or duplicate delivery.
> The trade-off is additional metadata and complexity, and the CRDT
> semantics need to match the business operation."

------------------------------------------------------------------------

## 14. Quick Revision

``` text
CRDT

Independent updates
        |
        v
Mergeable state
        |
        v
Deterministic convergence
```

Remember:

**G-Counter → PN-Counter → G-Set → 2P-Set → OR-Set**

And the core idea:

> Don't replicate only the current value when concurrent updates matter.
> Represent state/operations so independent changes can be merged
> safely.
