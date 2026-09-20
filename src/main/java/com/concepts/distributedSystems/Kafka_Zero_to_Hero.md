# Apache Kafka — Zero to Hero

> Interview-ready Kafka notes: fundamentals → producers → partitions →
> consumers → delivery semantics → reliability → scaling → Kafka Streams
> → production patterns → troubleshooting.

## 1. What is Kafka?

Apache Kafka is a distributed event streaming platform.

Kafka provides: - Durable append-only logs - High-throughput writes and
reads - Horizontal scalability through partitions - Consumer groups for
parallel processing - Replay through retained offsets - Ordering within
a partition - Decoupling between producers and consumers

Kafka is not simply a traditional queue. Records are retained for a
configured retention period and consumers track offsets.

## 2. Core Terminology

### Topic

A logical stream of records. A topic is split into partitions.

### Partition

An ordered, append-only log. Kafka guarantees ordering within one
partition, not globally across partitions.

### Offset

A position of a record within a partition.

### Producer

Writes records to Kafka.

### Consumer

Reads records from Kafka.

### Consumer Group

A set of consumers cooperating to process a topic.

Within one consumer group: - One partition can have at most one active
consumer. - One consumer can own multiple partitions. - Different
consumer groups consume the same topic independently.

## 3. Kafka Architecture

``` text
                 Kafka Cluster
        +-----------------------------+
        | Broker 1                    |
        | Broker 2                    |
        | Broker 3                    |
        +-----------------------------+

Topic: orders

P0 -> Broker 1
P1 -> Broker 2
P2 -> Broker 3
P3 -> Broker 1
```

Partitions are replicated across brokers. A partition has one leader and
follower replicas. Modern Kafka deployments use KRaft rather than
ZooKeeper.

## 4. Why Partitions Exist

Partitions provide:

1.  Parallelism
2.  Horizontal scalability

``` text
Topic
 |
 +-- P0 -> Consumer 1
 +-- P1 -> Consumer 2
 +-- P2 -> Consumer 3
 +-- P3 -> Consumer 4
```

More partitions can increase producer and consumer parallelism, but also
increase metadata, resource usage, rebalance work, recovery time, and
operational complexity.

## 5. Producer → Partition Mapping

A producer can send a key:

``` json
{
  "key": "user-123",
  "event": "ORDER_CREATED"
}
```

Conceptually:

``` text
key
 |
 v
partitioner
 |
 v
partition N
```

The key idea is that the same key is routed consistently to the same
partition under stable partitioning configuration.

Examples:

``` text
user-123 -> P7
user-456 -> P2
user-123 -> P7
```

Choose the key based on the entity for which ordering matters:

``` text
orders       -> orderId
user-events  -> userId
payments     -> paymentId / bookingId
```

Do not automatically choose a field just because it is convenient.

## 6. Ordering

Kafka ordering is partition-level.

``` text
P0:
A -> B -> C -> D
```

A consumer reading P0 sees A, B, C, D in order.

But if:

``` text
P0: A B C
P1: X Y Z
```

Kafka does not guarantee a global ordering such as A X B Y C Z.

If events for an entity must be ordered:

``` text
same entity -> same partition
```

## 7. Hot Partitions

A hot partition occurs when a key creates highly skewed traffic.

Example:

``` text
key = celebrityId

Celebrity A -> P7 -> 80% of traffic
```

Kafka can have perfectly balanced partition ownership while one
partition is overloaded.

Possible approaches: - Better key distribution - Composite/sharded keys
where ordering permits - Admission/rate limiting - More partitions -
Separate topics for extreme workloads - Application-level aggregation

Do not randomly shard a key when strict ordering for that key is
required.

## 8. Consumer Groups

Example:

``` text
Topic = orders
Partitions = 6

Group G1:
C1 -> P0 P1
C2 -> P2 P3
C3 -> P4 P5
```

Another group can independently consume the same topic:

``` text
Group G2:
C4 -> P0 P1
C5 -> P2 P3
C6 -> P4 P5
```

This is how multiple downstream applications consume the same event
stream independently.

## 9. Consumers \> Partitions

This is legal.

Example:

``` text
100 partitions
150 consumers
```

Only 100 consumers can actively own partitions. Roughly 50 consumers are
idle.

This is usually wasteful but can happen during autoscaling or deployment
transitions.

Useful parallelism is bounded by the partition count.

## 10. Consumer → Partition → Pod → Thread

Suppose:

``` text
200 partitions
4 pods
Spring Kafka concurrency = 5
```

Approximately:

``` text
4 pods x 5 consumer instances = 20 consumers
```

Kafka assigns the 200 partitions across those consumers, roughly 10
partitions per consumer.

Mental model:

``` text
key
 |
 v
partition
 |
 v
consumer-group assignment
 |
 v
consumer
 |
 v
pod
 |
 v
Spring Kafka listener/container thread
```

The key determines partition placement. The consumer group determines
partition ownership.

## 11. Consumer Failure and Rebalancing

Suppose:

``` text
C1 -> P0 P1 P2
C2 -> P3 P4 P5
C3 -> P6 P7 P8
```

C2 dies.

Kafka detects the membership change and rebalances. Partitions may move
to remaining consumers.

Rebalances can temporarily affect throughput and latency.

Important concepts: - Cooperative rebalancing - Static membership -
Session timeout - Poll interval

## 12. Consumer Polling

Conceptually:

``` text
while running:
    records = poll()
    process(records)
    commit()
```

Consumers must continue polling. If processing takes too long without
polling, Kafka can consider the consumer unhealthy.

Important settings: - `max.poll.interval.ms` - `max.poll.records` -
`session.timeout.ms` - `heartbeat.interval.ms`

## 13. Offset Management

Suppose:

``` text
P0:
0 1 2 3 4 5 6
```

If the consumer has committed through offset 3 and crashes before safely
processing offset 4, it can resume from the committed position.

Depending on commit timing, records may be processed more than once.
Design consumers accordingly.

## 14. At-Most-Once

``` text
poll
  |
commit offset
  |
process
```

If the application crashes after commit but before processing, the
record can be lost.

## 15. At-Least-Once

``` text
poll
  |
process
  |
commit offset
```

If the application crashes after processing but before commit, the
record can be processed again.

At-least-once plus idempotent processing is a common production pattern.

## 16. Exactly-Once

Kafka supports exactly-once semantics for specific Kafka transactional
workflows.

But:

``` text
Kafka -> external DB
Kafka -> REST API
Kafka -> payment provider
```

does not automatically become exactly-once.

External side effects still need appropriate: - Idempotency -
Deduplication - Transactions where applicable - Reconciliation

## 17. Idempotent Consumers

Suppose:

``` text
PaymentSuccess(B123)
PaymentSuccess(B123)
```

The consumer should safely process the business effect once.

Common approach:

``` text
processed_events
----------------
eventId
bookingId
```

Combine deduplication and business-state changes in one database
transaction where possible.

## 18. Producer Reliability

Important producer settings include:

### `acks=0`

No broker acknowledgement. Fast but weak durability.

### `acks=1`

Leader acknowledgement.

### `acks=all`

Wait for the required in-sync replicas.

For important business events, `acks=all` is commonly appropriate,
depending on the system’s durability requirements.

## 19. Producer Idempotence

A producer can retry after a timeout:

``` text
send
 |
timeout
 |
retry
```

Without idempotence, retries can create duplicate records.

Kafka producer idempotence helps prevent duplicates caused by producer
retries within its supported semantics.

## 20. Replication and ISR

Suppose:

``` text
Replication Factor = 3

P0:
Leader   -> Broker 1
Follower -> Broker 2
Follower -> Broker 3
```

ISR means In-Sync Replicas.

If Broker 3 falls sufficiently behind:

``` text
ISR:
Broker 1
Broker 2
```

A common durability configuration for important data might be:

``` text
RF = 3
min.insync.replicas = 2
producer acks = all
```

This is not a universal answer; capacity and failure requirements
determine the correct configuration.

## 21. Leader Failure

If the leader fails:

``` text
P0:
Leader   -> B1 💥
Follower -> B2
Follower -> B3
```

Kafka can elect an eligible replica as the new leader:

``` text
P0:
Leader   -> B2
Follower -> B3
```

Clients refresh metadata and continue.

## 22. Retention and Replay

Kafka records are normally retained according to policy rather than
deleted immediately after consumption.

Examples: - `retention.ms` - `retention.bytes`

Therefore:

``` text
consumer crashes
      |
      v
records remain
      |
      v
consumer restarts
      |
      v
resume from offset
```

Retention enables replay and recovery.

## 23. Consumer Lag

Conceptually:

``` text
latest offset = 1000
committed offset = 850

lag ≈ 150
```

Lag is one of the most important Kafka operational metrics.

Also monitor: - Processing latency - Throughput - Rebalances - Poll
latency - Commit failures - Broker health - Under-replicated
partitions - Error rate

Lag is not identical to end-to-end application latency.

## 24. Uneven Consumer Lag

Kafka balances partition ownership, not message volume.

Example:

``` text
12 partitions
4 consumers

C1 -> P0 P1 P2
C2 -> P3 P4 P5
C3 -> P6 P7 P8
C4 -> P9 P10 P11
```

If P0-P2 contain much more traffic, C1 can have much higher lag than
C2-C4.

This is often a partition-skew problem, not a consumer-assignment
problem.

## 25. Spring Kafka

Example:

``` java
@KafkaListener(
    topics = "orders",
    groupId = "order-service",
    concurrency = "5"
)
public void consume(OrderEvent event) {
    // business logic
}
```

Conceptually, one pod runs multiple consumer/container instances:

``` text
Pod
 |
 +-- Consumer 1
 +-- Consumer 2
 +-- Consumer 3
 +-- Consumer 4
 +-- Consumer 5
```

With 4 pods and concurrency 5:

``` text
4 x 5 = 20 consumer instances
```

Spring manages consumer lifecycle, polling, listener invocation,
commits, and rebalances.

## 26. Batch vs Record Consumption

Record listener:

``` text
one record -> listener
```

Batch listener:

``` text
poll -> batch -> listener(batch)
```

Batch processing can improve throughput when the downstream operation
benefits from bulk work, but it increases batch failure and latency
considerations.

## 27. Retry

Transient failures can use bounded retries with backoff:

``` text
1 sec
5 sec
30 sec
2 min
...
```

Do not retry everything forever.

Typical flow:

``` text
Main Topic
    |
    v
Retry
    |
    +-- transient failure -> retry
    |
    +-- repeated failure -> DLQ
```

Retry topics or framework-managed retry mechanisms can be used depending
on the application.

## 28. DLQ

A DLQ isolates records that cannot be processed after the defined retry
policy.

A useful DLQ record contains:

``` json
{
  "originalTopic": "orders",
  "originalPartition": 7,
  "originalOffset": 18382,
  "eventId": "E123",
  "error": "DB_TIMEOUT",
  "retryCount": 5
}
```

DLQ means:

> Automatic processing stopped; recovery requires controlled retry,
> investigation, or reconciliation.

## 29. Transactional Outbox

Problem:

``` text
DB transaction
      +
Kafka publish
```

If DB commit succeeds but Kafka publish fails, the systems disagree.

Outbox:

``` text
Application
    |
    +---- DB transaction ----+
    |                        |
    |                    Outbox table
    |                        |
    +---- business state     |
                             |
                             v
                      Outbox Publisher
                             |
                             v
                           Kafka
```

Business state and the outbox record are committed atomically in one DB
transaction.

## 30. Inbox / Deduplication

For Kafka → database processing:

``` text
Kafka event
    |
    v
Consumer
    |
    +-- check eventId
    |
    +-- update business state
    |
    +-- record processed event
```

Ideally these database operations are atomic.

## 31. Kafka vs Traditional Queue

Kafka: - Durable log - Replay - Multiple independent consumer groups -
Partition ordering - High throughput - Retention

Traditional queues: - Often optimized around work distribution -
Messages commonly disappear after acknowledgement - Replay may be less
central

Kafka is especially useful when one event must feed many independent
downstream applications.

## 32. Kafka vs Database

Kafka is not a replacement for a transactional database.

Common architecture:

``` text
PostgreSQL
    |
    | CDC / Outbox
    v
  Kafka
 /  |  /   |   Search Analytics Services
```

Kafka is excellent for event distribution and streaming. A transactional
database remains the source of truth when the domain requires
transactional state.

## 33. Kafka vs Redis Pub/Sub

Redis Pub/Sub: - Very low latency - Simple - Ephemeral - No durable
message history - Messages can be lost when subscribers are unavailable

Kafka: - Durable - Replayable - Partitioned - Consumer groups -
Retention

Use Redis Pub/Sub when transient broadcast semantics are acceptable. Use
Kafka when durable event processing matters.

## 34. Kafka Streams

Kafka Streams is a library for processing Kafka streams.

``` text
orders
   |
   v
Kafka Streams
   |
   +--> filter
   +--> map
   +--> join
   +--> aggregate
   |
   v
order-summary
```

Important concepts: - KStream - KTable - GlobalKTable - State stores -
Windowing - Joins - Exactly-once processing modes

Kafka is the platform; Kafka Streams is a processing library.

## 35. Kafka Connect

Kafka Connect integrates Kafka with external systems.

Examples:

``` text
PostgreSQL -> Kafka
Kafka -> Elasticsearch
Kafka -> S3
Kafka -> BigQuery
```

Source connectors bring data into Kafka. Sink connectors send data out.

## 36. Schema Evolution

Event schemas must evolve without unexpectedly breaking consumers.

Common technologies: - Avro - Protobuf - JSON Schema - Schema Registry

Important concepts: - Backward compatibility - Forward compatibility -
Full compatibility - Versioning - Optional fields

A producer should generally avoid breaking existing consumers.

## 37. Security

Production Kafka commonly uses:

``` text
Authentication
Authorization
Encryption
```

Examples: - TLS - SASL - ACLs - Encryption in transit - Encryption at
rest depending on infrastructure

Restrict who can produce and consume from which topics.

## 38. Scaling Kafka

Three different scaling questions:

### Producer scaling

More producers can write concurrently.

### Broker scaling

More brokers provide additional storage, network, and partition
capacity.

### Consumer scaling

More consumers provide more processing parallelism up to the partition
count.

Example:

``` text
100 partitions

10 consumers -> useful
50 consumers -> useful
100 consumers -> maximum partition-level parallelism
200 consumers -> 100 idle
```

## 39. Choosing Partition Count

Consider:

``` text
required throughput
/
throughput per partition
```

Then account for: - Consumer parallelism - Key distribution - Future
growth - Recovery/rebalance cost - Broker resources

Do not choose partitions only from today’s traffic.

## 40. Backpressure

Kafka can absorb bursts, but an unlimited backlog is not healthy.

Example:

``` text
Producer
   |
   v
Kafka
   |
   v
Consumer
   |
   v
Slow database
```

Monitor lag and protect downstream systems with admission control, rate
limiting, bounded concurrency, or other backpressure mechanisms.

## 41. Poison Messages

Bad pattern:

``` text
consume
  ↓
fail
  ↓
retry forever
```

Better:

``` text
consume
  ↓
bounded retry
  ↓
DLQ
  ↓
investigation/reconciliation
```

A poison message should not unnecessarily block healthy processing
unless strict ordering requirements make that unavoidable.

## 42. Rebalancing

Rebalances can be triggered by: - Consumer joins - Consumer leaves -
Consumer crashes - Partition changes - Group membership changes

During a rebalance, ownership can move:

``` text
P7
C1
 ↓
C2
```

Minimize unnecessary rebalances because they can affect throughput and
latency.

## 43. Static Membership

Static membership gives a consumer a stable group identity using
`group.instance.id`.

This can reduce unnecessary rebalances during controlled restarts or
transient failures.

## 44. Multi-Region Kafka

Possible models include:

### One Kafka cluster

``` text
Region A apps
      |
      v
   Kafka
      ^
      |
Region B apps
```

### Separate clusters

``` text
Kafka A <---- replication ----> Kafka B
```

Multi-region design must answer: - Active-active or active-passive? -
Where is the source of truth? - How is replication handled? - What
happens during network partition? - How are duplicate events handled? -
How are offsets handled? - Where should consumers process events?

Two regions does not automatically mean two consumer groups.

## 45. Production Monitoring

### Producer

- Request latency
- Error rate
- Record rate
- Bytes in
- Retries
- Record errors

### Broker

- CPU
- Memory
- Disk
- Network
- Request latency
- Under-replicated partitions
- Offline partitions
- ISR changes

### Consumer

- Consumer lag
- Records consumed
- Processing latency
- Rebalance count
- Poll latency
- Commit failures
- Consumer errors

### Application

- Business failures
- DB latency
- Downstream latency
- Retry count
- DLQ volume

## 46. Common Kafka Mistakes

### Mistake 1

“Kafka guarantees ordering.”

Correct: ordering is within a partition.

### Mistake 2

“More consumers always means more throughput.”

Correct: useful consumer parallelism is bounded by partitions.

### Mistake 3

“Kafka gives exactly-once for everything.”

Correct: exactly-once has scope; external side effects still need
idempotency/reconciliation.

### Mistake 4

“Kafka balances load evenly.”

Correct: Kafka balances partition ownership, not necessarily message
volume.

### Mistake 5

“DLQ solves failures.”

Correct: DLQ isolates failures; recovery still needs a strategy.

### Mistake 6

“Use any business ID as the key.”

Correct: choose the key based on ordering requirements and distribution.

### Mistake 7

“Increase partitions whenever lag increases.”

First determine whether the bottleneck is: - Hot partition - Consumer
processing - Downstream database - Network - Broker capacity - Poor key
distribution

# 47. Kafka Interview Mental Model

When designing Kafka, walk through:

``` text
1. What events exist?
2. What are the topics?
3. What is the partition key?
4. What ordering is required?
5. How many partitions?
6. How many consumer groups?
7. How many consumers?
8. What happens when a consumer dies?
9. What delivery semantics are required?
10. How are duplicates handled?
11. What happens when processing fails?
12. What is the retry strategy?
13. What is the DLQ strategy?
14. What is the retention period?
15. What is the replication factor?
16. What durability is required?
17. How are hot partitions handled?
18. How is lag monitored?
19. How does schema evolution work?
20. What happens during regional failure?
```

# 48. Staff-Level Kafka Checklist

Do not stop at:

> “Use Kafka.”

Explain:

``` text
                    Kafka
                      |
       +--------------+--------------+
       |              |              |
   Partitioning    Durability     Consumers
       |              |              |
      key          replication    groups
       |              |              |
   ordering         ISR          scaling
       |              |              |
   hot keys        failover      lag
                                      |
                                  retries
                                      |
                                    DLQ
                                      |
                               reconciliation
```

For every component, explain **why it exists**, the tradeoff, and the
failure mode.

# 49. Rules Worth Memorizing

``` text
Ordering = partition-level

Same key -> same partition
(assuming stable partitioning configuration)

1 partition -> max 1 active consumer
within one consumer group

1 consumer -> multiple partitions

Consumers > partitions -> some consumers idle

Equal partition count != equal traffic

At-least-once -> design for duplicates

Kafka retention -> replay is possible

Kafka is not your transactional database

Exactly-once Kafka processing != exactly-once external side effects
```

# 50. Interview Questions

## Fundamentals

1.  What problem does Kafka solve?
2.  Kafka vs a traditional message queue?
3.  What is a topic?
4.  What is a partition?
5.  What is an offset?
6.  What is a consumer group?
7.  Why does Kafka need partitions?
8.  How does Kafka provide ordering?
9.  What happens when a consumer joins a group?
10. What happens when a consumer crashes?

## Partitioning

11. How does Kafka choose a partition?
12. What happens when a producer supplies a key?
13. Why is key selection important?
14. What is a hot partition?
15. How would you fix a hot partition?
16. Can you increase partition count?
17. What are the tradeoffs of many partitions?
18. Can two consumers in the same group consume the same partition?
19. Can two different consumer groups consume the same partition?
20. What happens if consumers outnumber partitions?

## Consumers

21. How does consumer-group rebalancing work?
22. What is consumer lag?
23. Why can two consumers have different lag?
24. Does Kafka balance traffic or partitions?
25. What does Spring Kafka `concurrency` do?
26. How does a consumer poll records?
27. What happens if processing takes longer than `max.poll.interval.ms`?
28. What happens when a pod dies?
29. How do you minimize rebalances?
30. What is static consumer membership?

## Reliability

31. What are at-most-once, at-least-once, and exactly-once?
32. Why is at-least-once common?
33. How do you make a consumer idempotent?
34. What does Kafka producer idempotence solve?
35. What is `acks=all`?
36. What is ISR?
37. What does `min.insync.replicas` do?
38. What happens when a broker fails?
39. How does Kafka recover a partition leader?
40. What is the difference between replication and consumer groups?

## Failure Handling

41. How do you retry failed messages?
42. When should a message go to a DLQ?
43. How do you handle poison messages?
44. How do you preserve ordering while retrying?
45. How do you replay a DLQ safely?
46. What should a DLQ record contain?
47. How do you reconcile an unknown payment status?
48. What happens if the consumer crashes after the DB update but before
    committing the Kafka offset?

## Architecture

49. Kafka vs Redis Pub/Sub?
50. Kafka vs database?
51. What is the transactional outbox pattern?
52. What is the inbox/deduplication pattern?
53. When would you use Kafka Streams?
54. When would you use Kafka Connect?
55. How would you design Kafka for 1M events/sec?
56. How would you choose partition count?
57. How would you design Kafka across two regions?
58. How would you protect a downstream DB from a Kafka surge?
59. How would you detect and fix consumer lag?
60. How would you design for a hot key?

# 51. Final Mental Model

When someone says:

> “We have Kafka.”

Immediately ask:

``` text
What topic?
    ↓
What event?
    ↓
What key?
    ↓
How many partitions?
    ↓
What ordering?
    ↓
What consumer groups?
    ↓
How many consumers?
    ↓
What delivery semantics?
    ↓
What happens on duplicate?
    ↓
What happens on failure?
    ↓
Retry or DLQ?
    ↓
How do we recover?
    ↓
How do we monitor lag?
    ↓
What happens at scale?
```

That chain is the foundation of serious Kafka design discussions.
