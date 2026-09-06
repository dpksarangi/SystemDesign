# HLD & System Design Learning

A personal knowledge base for **High-Level Design (HLD), System Design,
distributed systems, and backend engineering concepts**.

The goal of this repository is not to collect definitions. It is to
build the ability to **reason about systems, make trade-offs, estimate
scale, and defend architectural decisions in interviews and real-world
engineering discussions.**

------------------------------------------------------------------------

## Repository Structure

``` text
HLD/
│
├── src/
│   └── main/
│       └── java/
│           └── com/
│               ├── hld/
│               │   ├── adclickaggregator/
│               │   │   ├── AdClickAggregator.drawio
│               │   │   ├── AdClickAggregator.svg
│               │   │   └── Ad_Click_Aggregator_Staff_System_Design_Notes.md
│               │   │
│               │   └── twiter/
│               │       ├── Twiter.drawio
│               │       ├── Twiter.png
│               │       └── Tweeter_Staff_System_Design_Notes.md
│               │
│               └── concepts.nosqldbs/
│
├── src/test/
│
├── pom.xml
└── README.md
```

> The repository currently uses a Java/Maven project structure, but the
> notes themselves are primarily Markdown and architecture diagrams.

------------------------------------------------------------------------

# What This Repository Contains

The repository is divided into two broad categories.

## 1. System Designs

Complete HLD exercises for large-scale systems.

Each system design should cover:

``` text
Requirements
      ↓
Capacity Estimation
      ↓
Back-of-the-Envelope Calculations
      ↓
APIs / Access Patterns
      ↓
High-Level Architecture
      ↓
Data Model
      ↓
Scaling
      ↓
Caching
      ↓
Consistency
      ↓
Failure Handling
      ↓
Trade-offs
      ↓
Interviewer Drill-downs
```

Examples:

-   Twitter / Tweeter
-   Ad Click Aggregator
-   Future system-design problems

Each design may contain:

-   `.md` --- design notes and reasoning
-   `.drawio` --- editable architecture diagram
-   `.svg` / `.png` --- rendered architecture diagram

------------------------------------------------------------------------

## 2. Distributed Systems & Backend Concepts

Concept-focused notes live separately from system designs.

Examples include:

-   CAP Theorem
-   PACELC
-   Consistency models
-   Availability
-   Partitioning
-   Replication
-   Sharding
-   Quorum
-   Leader election
-   Distributed locks
-   Consensus
-   Kafka
-   Cassandra
-   Redis
-   NoSQL databases
-   OLAP vs OLTP
-   Stream processing
-   Batch processing
-   Event-driven architecture
-   Distributed transactions
-   Idempotency
-   Exactly-once / at-least-once / at-most-once processing
-   Outbox pattern
-   CQRS
-   Event sourcing

The purpose of these notes is to understand the **building blocks** that
appear repeatedly inside system designs.

------------------------------------------------------------------------

# How I Approach System Design

The repository follows a **requirements-first** approach.

Instead of starting with:

> "Let's use Kafka, Cassandra, Redis and Kubernetes."

Start with:

> "What does the system need to do, how much traffic does it handle,
> what must be strongly consistent, and what can be asynchronous?"

Then choose technologies based on those requirements.

------------------------------------------------------------------------

## The Core Mental Model

For every system, ask:

``` text
1. What are the functional requirements?
             ↓
2. What are the non-functional requirements?
             ↓
3. What is the scale?
             ↓
4. What are the important access patterns?
             ↓
5. What is the source of truth?
             ↓
6. What data is derived?
             ↓
7. What can be asynchronous?
             ↓
8. What happens when a dependency fails?
             ↓
9. What becomes a bottleneck at 10× / 100× scale?
             ↓
10. What are the trade-offs?
```

------------------------------------------------------------------------

# Back-of-the-Envelope Calculations

**BOTE calculations are a first-class part of every system design.**

Before choosing infrastructure, estimate:

``` text
DAU / users
↓
requests per second
↓
peak requests per second
↓
storage per day
↓
storage per year
↓
network bandwidth
↓
database throughput
↓
cache size
↓
Kafka throughput
↓
partition count
↓
replication overhead
```

The numbers do not need to be exact.

The objective is to establish the **order of magnitude**.

For example:

``` text
10K events/sec
× 500 bytes/event
≈ 5 MB/sec

5 MB/sec
× 86,400 sec/day
≈ 432 GB/day
```

Then ask:

> Does this change my architecture?

If not, don't over-engineer.

------------------------------------------------------------------------

# HLD vs LLD

This repository intentionally focuses on **HLD first**.

### HLD

Focus on:

-   requirements
-   major components
-   data flow
-   storage choices
-   scaling strategy
-   consistency
-   availability
-   failure modes
-   trade-offs

Example:

``` text
Click Processor
      ↓
Kafka
      ↓
Flink
      ↓
OLAP
```

### LLD

Only drill down when the interviewer asks:

``` text
Kafka
 → partition count
 → producer acknowledgements
 → retry configuration

Flink
 → state backend
 → checkpointing
 → watermark implementation

Outbox
 → schema
 → WAL
 → retry mechanics
```

**Staff-level design is not about showing every implementation detail.**

It is about knowing **where the detail matters and when to go deeper.**

------------------------------------------------------------------------

# Staff-Level Design Principles

## 1. Start simple

Don't introduce a distributed system component unless there is a reason.

## 2. Design around access patterns

The database should follow the queries.

``` text
Know the query
      ↓
Design the data model
      ↓
Choose the storage
```

## 3. Separate canonical and derived data

For example:

``` text
Canonical events
      ↓
Kafka / Object Storage
      ↓
Derived processing
      ↓
OLAP / Search / Cache
```

If derived data is lost, it should ideally be reconstructable.

## 4. Make asynchronous work asynchronous

If the user doesn't need the result immediately:

``` text
request
  ↓
durable event
  ↓
async processing
```

## 5. Design for failure

For every major dependency ask:

``` text
What happens if it goes down?
```

Not:

> "It won't go down."

## 6. Identify pathological cases

Normal traffic rarely breaks distributed systems.

Look for:

-   celebrity users
-   hot keys
-   viral content
-   huge partitions
-   traffic spikes
-   slow consumers
-   retry storms
-   duplicate events
-   late events

## 7. Explain trade-offs

Avoid:

> "Kafka is better."

Prefer:

> "Kafka gives us durable, partitioned event processing, but adds
> operational complexity. Given the throughput and replay requirement,
> that complexity is justified."

------------------------------------------------------------------------

# Reliability Checklist

For every design, consider:

``` text
□ Replication
□ Retries
□ Idempotency
□ Timeouts
□ Backpressure
□ Dead-letter / failed-event handling
□ Consumer lag
□ Data recovery
□ Replay
□ Disaster recovery
□ Graceful degradation
□ Eventual consistency
```

Always distinguish:

``` text
Data loss
vs
Data delay
vs
Duplicate data
vs
Stale data
```

These are different failure modes.

------------------------------------------------------------------------

# Consistency Checklist

Ask:

``` text
What must be strongly consistent?
What can be eventually consistent?
What happens during a partition?
What is the source of truth?
Can derived data be rebuilt?
```

Do not simply say:

> "We use eventual consistency."

Explain **where** and **why**.

------------------------------------------------------------------------

# Common Technology Mental Models

These are intentionally simplified interview mental models.

``` text
Kafka
→ durable distributed event log / event backbone

Cassandra
→ predictable high-scale key-based access

Redis
→ low-latency cache / ephemeral state

Object Storage
→ durable, cheap historical data

Spark
→ large-scale batch processing / recomputation

Flink
→ low-latency stateful stream processing

OLAP
→ analytical query serving

Elasticsearch
→ search / text-oriented retrieval

SQL
→ transactional relational workloads

Graph DB
→ relationship traversal
```

The technology should follow the problem, not the other way around.

------------------------------------------------------------------------

# Interview Preparation Workflow

For a new system-design question:

### Step 1 --- Clarify requirements

Don't design before understanding the problem.

### Step 2 --- Estimate scale

Write down rough numbers.

``` text
Users:
RPS:
Peak RPS:
Data/event size:
Storage/day:
Query QPS:
Latency:
```

### Step 3 --- Identify access patterns

Examples:

``` text
Get timeline(user)
Get followers(user)
Get metrics(adId, time range)
Search tweets(query)
```

### Step 4 --- Draw the simplest architecture

Start with:

``` text
Client
  ↓
API
  ↓
Service
  ↓
Storage
```

Then add complexity only when justified.

### Step 5 --- Attack the design

Ask:

``` text
What if traffic becomes 10×?

What if one key becomes extremely hot?

What if Kafka is down?

What if the database is down?

What if events are duplicated?

What if events arrive late?

What if a consumer dies?

What if derived data is corrupted?
```

### Step 6 --- Discuss trade-offs

Explain why you chose the design and what you sacrificed.

### Step 7 --- Drill down only when required

Move from:

``` text
HLD → component design → LLD
```

only as the interviewer asks.

------------------------------------------------------------------------

# Design Review Scorecard

Each system design can be evaluated across:

  Area             Question
  ---------------- ------------------------------------------------------
  Requirements     Did we identify the actual problem?
  Capacity         Did we estimate the scale?
  BOTE             Are the numbers internally consistent?
  Architecture     Does the architecture follow the requirements?
  Data model       Does it match access patterns?
  Scalability      What happens at 10× / 100×?
  Reliability      What happens when components fail?
  Consistency      Where is strong/eventual consistency appropriate?
  Performance      Can we meet latency requirements?
  Trade-offs       Can we explain why we chose this design?
  Staff depth      Did we identify pathological cases and future risks?
  HLD discipline   Did we avoid unnecessary LLD?

------------------------------------------------------------------------

# Notes Philosophy

These notes are meant to capture **reasoning**, not just answers.

A good note should ideally answer:

``` text
What problem does this solve?
Why do we need it?
How does it work at a high level?
What alternatives exist?
When should I use it?
What are the trade-offs?
What failure modes should I consider?
What interview questions can follow?
```

If a concept cannot be connected to a real system-design decision, it
probably needs more context.

------------------------------------------------------------------------

# Current Designs

## Ad Click Aggregator

Focus areas:

-   high-volume click ingestion
-   durable event capture
-   Kafka
-   Flink
-   Blob Storage
-   Spark reconciliation
-   OLAP
-   1-minute aggregation
-   idempotency
-   late events
-   hot keys
-   real-time vs batch processing

## Tweeter

Focus areas:

-   fan-out-on-write
-   fan-out-on-read
-   celebrity users
-   Cassandra data modeling
-   social graph
-   timeline storage
-   pagination
-   search
-   caching
-   media storage
-   Kafka
-   consistency and failure handling

------------------------------------------------------------------------

# The Goal

The ultimate goal of this repository is to move from:

> **"I know Kafka, Cassandra, Redis, Spark..."**

to:

> **"I understand the problem, I can estimate the scale, I can design
> the system, I know where it will fail, and I can explain why I made
> each trade-off."**

And eventually:

``` text
Requirements
      ↓
Reasoning
      ↓
Numbers
      ↓
Architecture
      ↓
Failure modes
      ↓
Trade-offs
      ↓
Staff-level design
```

------------------------------------------------------------------------

## Interview Mantra

> **Start with requirements.**
>
> **Estimate before choosing technology.**
>
> **Design around access patterns.**
>
> **Make the normal path fast.**
>
> **Identify the pathological case.**
>
> **Design for failure.**
>
> **Keep derived data rebuildable where possible.**
>
> **Explain trade-offs.**
>
> **Stay at HLD until the interviewer asks for LLD.**

------------------------------------------------------------------------

### Future Topics

Potential additions:

-   Rate Limiter
-   URL Shortener
-   Notification System
-   Distributed Job Scheduler
-   File Storage / Dropbox
-   Ride Sharing
-   Food Delivery
-   Payment System
-   Chat / Messaging
-   Video Streaming
-   Distributed Cache
-   Metrics / Monitoring Platform
-   News Feed
-   Search Autocomplete
-   Recommendation System
-   API Gateway
-   Distributed Lock Service
-   Workflow Engine
