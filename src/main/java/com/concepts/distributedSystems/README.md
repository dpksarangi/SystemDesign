# Distributed Systems Concepts

Interview-oriented notes supporting the HLD design repository.

## Notes

1.  [PACELC](pacelc.md) --- consistency/latency trade-offs during
    partitions and normal operation.
2.  [Consistency Models](consistency-models.md) --- eventual,
    read-your-writes, monotonic reads, session, causal, and strong
    consistency.
3.  [CRDTs](crdts.md) --- coordination-free conflict resolution and
    convergent replicated data structures.
4.  [Leader Election & Raft](leader-election-and-raft.md) --- quorum,
    terms, elections, replicated logs, failover, and split brain.
5.  [Database Replication & CDC](cdc-and-database-replication.md) ---
    native replication, replication lag, WAL/binlog/oplog, and change
    data capture.

## Intended use

These notes are deliberately HLD/interview focused. They emphasize:

-   mental models
-   architecture impact
-   trade-offs
-   implementation mechanisms
-   failure scenarios
-   Staff-level interview questions

Detailed implementation of individual database products or consensus
algorithms should be added only when it becomes useful for a design.
