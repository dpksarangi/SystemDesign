# Leader Election & Raft

## 1. Why Do We Need a Leader?

In a replicated system:

``` text
        Leader
        /  |  \
       v   v   v
      R1  R2  R3
```

The leader can act as the authoritative coordinator for writes and
operation ordering.

Without coordination, multiple nodes may independently accept
conflicting writes.

------------------------------------------------------------------------

## 2. Leader Failure

Initially:

``` text
A = Leader
B = Follower
C = Follower
```

If A fails:

``` text
A ❌
```

the cluster needs a new leader.

That process is **leader election**.

A common election mechanism uses:

-   heartbeats
-   timeouts
-   voting
-   quorum/majority
-   generation numbers/terms

------------------------------------------------------------------------

## 3. Why Failure Detection Is Hard

If B cannot reach A:

``` text
B --------X-------- A
```

B cannot know with certainty whether:

-   A crashed
-   the network is broken
-   A is alive but temporarily unreachable

Therefore distributed systems use **failure suspicion**, commonly
through heartbeat timeouts.

------------------------------------------------------------------------

## 4. Raft

Raft is a consensus algorithm.

It is not a database.

Raft nodes can be:

``` text
Follower
Candidate
Leader
```

The protocol helps the cluster agree on:

-   who the leader is
-   which log entries are committed
-   the order of operations

------------------------------------------------------------------------

## 5. Terms

A term is an election generation.

``` text
Term 7 -> Leader A
Term 8 -> Leader B
Term 9 -> Leader C
```

A higher term supersedes an older term.

This helps nodes reject stale messages and stale leadership.

------------------------------------------------------------------------

## 6. Leader Election

Suppose:

``` text
A = Leader
B,C,D,E = Followers
```

A fails.

B stops receiving heartbeats and becomes a candidate.

``` text
B:
Term = 11
State = Candidate
```

B requests votes from other nodes.

If B obtains a majority:

``` text
B + C + D = 3 / 5
```

B becomes leader.

------------------------------------------------------------------------

## 7. Quorum

For N nodes:

``` text
majority = floor(N/2) + 1
```

Examples:

    Nodes   Majority
  ------- ----------
        3          2
        5          3
        7          4

Why majority?

Any two majorities must overlap.

For five nodes:

``` text
Majority 1 = A B C
Majority 2 = C D E

Overlap = C
```

This overlap is fundamental to safe consensus.

------------------------------------------------------------------------

## 8. Network Partition

Five nodes:

``` text
A | B C D E
```

If the right side has four nodes, it has a majority and can elect a
leader.

A alone cannot.

This prevents a minority partition from safely establishing a competing
consensus decision.

This is closely related to the CAP availability/consistency trade-off.

------------------------------------------------------------------------

## 9. Split Brain

Split brain occurs when two partitions incorrectly operate as
independent authorities.

Example:

``` text
A = old leader

Network partition

A       |       B C D E
```

If both sides accepted independent authoritative writes, they could
diverge.

Consensus + quorum + terms help prevent this.

------------------------------------------------------------------------

## 10. Fencing and Terms

A particularly important failure case:

``` text
Old Leader A
```

becomes unreachable but is still alive.

Meanwhile:

``` text
B = New Leader
```

B operates under a newer term/epoch.

When A learns:

``` text
current_term > A's term
```

it must step down.

This is a form of fencing through **generation/epoch information**.

More generally, systems can use fencing tokens so stale leaders cannot
continue performing privileged operations.

------------------------------------------------------------------------

## 11. Replicated Log

Raft maintains an ordered log.

Example:

``` text
Index   Term   Command

1       10     SET X=100
2       10     SET Y=200
3       11     SET X=150
4       11     SET Y=300
```

The log represents the ordered history of operations.

Conceptually:

``` text
Replicated Log
      |
      v
State Machine
      |
      v
Database State
```

------------------------------------------------------------------------

## 12. Log Replication

Leader receives:

``` text
SET X=500
```

It appends the operation to its log and replicates it to followers.

``` text
Leader
 / | \
v  v  v
R1 R2 R3
```

Once sufficient replicas have the entry and the protocol's commit
conditions are satisfied, the entry becomes committed and is applied to
the state machine.

------------------------------------------------------------------------

## 13. Uncommitted vs Committed

Suppose:

``` text
5 nodes

Leader + 1 follower = 2 copies
```

There is no majority.

If the leader dies, that entry may be discarded/reconciled.

A committed entry has stronger protection because it has achieved the
protocol's required agreement.

Therefore:

> **Leader acknowledgement and committed consensus are not automatically
> the same thing.**

Always clarify acknowledgement semantics.

------------------------------------------------------------------------

## 14. Raft Concepts to Know for HLD

You do not usually need to implement Raft in a system-design interview.

You should understand:

-   leader
-   follower
-   candidate
-   term
-   election timeout
-   heartbeat
-   vote
-   quorum
-   log
-   commit index
-   leader failover
-   split brain
-   fencing/epochs

------------------------------------------------------------------------

## 15. Connection to Databases

A database can use a consensus protocol to coordinate:

``` text
Who is the leader?
Which writes are committed?
What is the order of operations?
What happens after failure?
```

This is different from simply asking:

> "How are changes replicated?"

Replication and leader election are related but distinct concerns.

------------------------------------------------------------------------

## 16. Interview Questions

-   Why do we need leader election?
-   How does a node know the leader failed?
-   Why do we need quorum?
-   What happens during a network partition?
-   What is split brain?
-   What is a Raft term?
-   What happens when the old leader comes back?
-   Why can an uncommitted log entry be discarded?
-   What is the difference between leader and replica?
-   How does quorum affect availability?
-   How does leader election affect failover latency?

------------------------------------------------------------------------

## 17. Quick Revision

``` text
Leader Election
      |
      v
Who is in charge?
      |
      v
Term + Vote + Quorum
      |
      v
New Leader
      |
      v
Replicated Log
      |
      v
Commit
      |
      v
State Machine
```

The key idea:

> **Consensus is about getting distributed nodes to agree on authority
> and an ordered history despite failures.**
