# NoSQL Databases

> **HLD interview notes:** NoSQL is not a single database technology. It
> is a family of data models designed to solve workloads where a
> traditional relational model may not be the best fit.

------------------------------------------------------------------------

# 1. Why NoSQL?

## 1.1 What is NoSQL?

NoSQL broadly refers to databases that use data models other than the
traditional relational/table model as their primary abstraction.

The major classic models are:

``` text
NoSQL
│
├── Key-Value
├── Document
├── Wide-Column
└── Graph
```

Modern systems also include specialized databases:

``` text
Specialized
│
├── Search
├── Time-Series
└── Vector
```

The important idea is:

> **NoSQL is about choosing a data model optimized for a workload, not
> simply about avoiding SQL.**

------------------------------------------------------------------------

## 1.2 Why did NoSQL emerge?

Relational databases are excellent at:

-   Transactions
-   Joins
-   Referential integrity
-   Rich queries
-   Strong consistency
-   Structured data

But large distributed applications can have requirements such as:

``` text
Very high traffic
Very large datasets
Horizontal scaling
High availability
Multi-region distribution
Very high write throughput
Flexible data models
Predictable access patterns
```

NoSQL databases were designed to address some of these requirements
using different data models and distributed architectures.

------------------------------------------------------------------------

## 1.3 NoSQL does NOT mean:

### "No schema"

Incorrect.

NoSQL databases may have flexible schemas, application-enforced schemas,
or query-driven schemas.

### "No transactions"

Incorrect.

Transaction capabilities vary by database.

### "Always eventually consistent"

Incorrect.

Consistency depends on the database and configuration.

### "Always faster than SQL"

Incorrect.

Performance depends on workload, data model, queries, indexes, hardware,
distribution, and consistency requirements.

------------------------------------------------------------------------

# 2. Key-Value Databases

## 2.1 Mental Model

Think of a distributed HashMap:

``` text
Key                  Value
--------------------------------
user:123              {...}
session:abc            {...}
cart:456               {...}
config:payment         {...}
```

The fundamental operation is:

``` text
GET key
SET key value
DELETE key
```

The database primarily knows about the **key**.

------------------------------------------------------------------------

## 2.2 Data Model

``` text
              Key
               │
               ↓
             Value
```

The value may be:

-   String
-   Number
-   JSON
-   Binary data
-   List
-   Set
-   Other structured data

depending on the database.

------------------------------------------------------------------------

## 2.3 Characteristics

### Fast key-based access

``` text
GET user:123
```

is the natural operation.

### Horizontal scaling

Keys can be distributed across nodes:

``` text
Hash(key)
   │
   ├── Node A
   ├── Node B
   └── Node C
```

### Flexible values

The stored value can often evolve without a rigid relational schema.

### Simple access pattern

The simplicity is both a strength and a limitation.

------------------------------------------------------------------------

## 2.4 Use Cases

Excellent for:

``` text
Caching
Session storage
Shopping carts
Feature flags
Rate limiting
Counters
Idempotency keys
User preferences
Simple metadata
```

Example:

``` text
session:user123
        ↓
{
  userId: 123,
  expiresAt: ...
}
```

------------------------------------------------------------------------

## 2.5 Limitations

Key-value stores are a poor fit when you need arbitrary queries such as:

``` text
Find users where
age > 30
AND
city = Bengaluru
```

or:

``` text
Find all products
matching several attributes
```

They are also not naturally designed for complex relationships.

------------------------------------------------------------------------

## 2.6 Examples

### Redis

Commonly used for:

-   Caching
-   Sessions
-   Counters
-   Rate limiting
-   Distributed locks
-   Real-time data structures

### DynamoDB

A managed distributed database supporting key-value and
document-oriented access patterns.

Useful for:

-   High-scale application data
-   Serverless applications
-   User data
-   Orders
-   Metadata
-   High-throughput workloads

------------------------------------------------------------------------

# 3. Document Databases

## 3.1 Mental Model

Think:

> **An application object stored as a document.**

Example:

``` json
{
  "userId": 123,
  "name": "Deepak",
  "email": "deepak@example.com",
  "address": {
    "city": "Bengaluru",
    "country": "India"
  },
  "skills": [
    "Java",
    "Kubernetes",
    "System Design"
  ]
}
```

------------------------------------------------------------------------

## 3.2 Data Model

``` text
Collection
   │
   ├── Document
   ├── Document
   └── Document
```

Documents are commonly JSON/BSON-like.

Unlike a simple key-value store, the database understands the structure
of the document.

------------------------------------------------------------------------

## 3.3 Why Document Databases?

Suppose a user profile consists of:

``` text
User
 ├── identity
 ├── address
 ├── preferences
 └── skills
```

If the application usually reads this information together, storing it
as one document can provide natural data locality.

Instead of:

``` text
users
addresses
preferences
user_skills
```

we can often retrieve one document.

------------------------------------------------------------------------

## 3.4 Characteristics

### Flexible schema

Different documents can contain different fields.

### Richer queries

You can query fields inside the document.

Example:

``` text
address.city = "Bengaluru"
```

### Indexing

Fields inside documents can be indexed.

### Horizontal scaling

Documents can be distributed through sharding/partitioning.

------------------------------------------------------------------------

## 3.5 Use Cases

Good examples:

``` text
User profiles
Product catalogs
Content management
Articles
Orders
Metadata
Configuration
Mobile/backend applications
```

Especially useful when:

> **An entity naturally forms a self-contained document.**

------------------------------------------------------------------------

## 3.6 Limitations

Potential problems include:

``` text
Complex relationships
Many joins
Highly normalized relational data
Complex cross-document transactions
Heavy analytical workloads
```

Modern document databases can support transactions and sophisticated
queries, so these are trade-offs rather than absolute limitations.

------------------------------------------------------------------------

## 3.7 Example: MongoDB

A product might be represented as:

``` json
{
  "productId": "P123",
  "name": "Laptop",
  "brand": "Lenovo",
  "specifications": {
    "ram": "16GB",
    "storage": "1TB"
  },
  "variants": [...]
}
```

This representation maps naturally to application objects.

------------------------------------------------------------------------

# 4. Wide-Column Databases

## 4.1 Why This Category Matters

Wide-column databases are especially important for HLD interviews
because they are designed for:

``` text
Huge datasets
High write throughput
Horizontal scaling
High availability
Predictable access patterns
Distributed workloads
```

Examples:

-   Cassandra
-   ScyllaDB
-   HBase
-   Google Bigtable

------------------------------------------------------------------------

# 4.2 Mental Model

Do not think:

> "It's a SQL table distributed across machines."

Instead think:

> **"Data is organized into partitions based on the access pattern."**

Simplified example:

``` text
Partition Key     Data
--------------------------------
user_123          tweet_100
user_123          tweet_101
user_123          tweet_102

user_456          tweet_200
user_456          tweet_201
```

------------------------------------------------------------------------

# 4.3 Query-Driven Data Modeling

This is one of the most important Cassandra concepts.

Relational design often starts with:

``` text
What entities exist?
What relationships exist?
```

Cassandra-style design starts with:

``` text
What queries must the system support?
```

Example:

``` text
Q1:
Get tweets for a user.

Q2:
Get recent tweets for a user.

Q3:
Get tweets for a user in a time range.
```

Then design the table to support these queries efficiently.

------------------------------------------------------------------------

# 4.4 Partition Key

The partition key determines how data is distributed.

Conceptually:

``` text
hash(partition_key)
        ↓
partition placement
        ↓
Node
```

Example:

``` text
user_123 → Node A
user_456 → Node C
user_789 → Node B
```

A good partition key should help achieve:

``` text
Even distribution
+
Reasonable partition size
+
Efficient queries
```

------------------------------------------------------------------------

# 4.5 Hot Partitions

A poor partition key can create a hot partition.

Bad example:

``` text
partition_key = country
```

If one country receives a huge percentage of traffic:

``` text
India
  ↓
Huge traffic
  ↓
Hot partition
  ↓
Node bottleneck
```

High-cardinality keys such as `user_id` often distribute better, but
cardinality alone does not guarantee a good partition key.

Also consider:

-   Traffic distribution
-   Partition size
-   Growth
-   Query pattern
-   Time distribution

------------------------------------------------------------------------

# 4.6 Clustering Columns

Within a partition, clustering columns can organize rows.

Example:

``` text
Partition key = user_id
Clustering column = created_at
```

Conceptually:

``` text
user_123
   │
   ├── 10:00 tweet
   ├── 10:05 tweet
   ├── 10:10 tweet
   └── 10:15 tweet
```

This is useful for ordered and range queries within a partition.

------------------------------------------------------------------------

# 4.7 Denormalization

Cassandra commonly uses denormalization.

Suppose we need:

``` text
Q1:
Tweets by user

Q2:
Tweets by hashtag
```

We may create:

``` text
tweets_by_user

tweets_by_hashtag
```

The same tweet may be stored in both.

Why?

``` text
Extra storage
+
Extra write complexity
        ↓
Fast predictable reads
```

This is an intentional trade-off.

------------------------------------------------------------------------

# 4.8 Replication

Suppose:

``` text
Replication Factor = 3
```

A partition can be replicated across multiple nodes:

``` text
Partition
   │
   ├── Node A
   ├── Node B
   └── Node C
```

If one node fails, another replica may serve the request.

Replication is fundamental to availability and fault tolerance.

------------------------------------------------------------------------

# 4.9 Consistency

Distributed NoSQL databases may expose configurable consistency levels.

For example, Cassandra has consistency levels such as:

``` text
ONE
QUORUM
ALL
```

A simplified quorum relationship often discussed is:

``` text
R + W > RF
```

where:

``` text
R  = replicas required for a read
W  = replicas required for a write
RF = replication factor
```

Example:

``` text
RF = 3
R = 2
W = 2

2 + 2 > 3
```

There is replica overlap between the read and write operations.

Important:

> This should not be casually equated with application-level
> linearizability. The actual consistency guarantee depends on the
> database semantics, configuration, failure conditions, and workload.

------------------------------------------------------------------------

# 4.10 Cassandra Strengths

Excellent for:

``` text
Massive write volume
High availability
Horizontal scaling
Large datasets
Predictable queries
Distributed workloads
Multi-region workloads
```

Typical use cases:

``` text
Activity feeds
Messaging
IoT
Telemetry
Event history
User timelines
Large-scale application events
```

------------------------------------------------------------------------

# 4.11 Cassandra Limitations

Poor fit for:

``` text
Complex joins
Ad-hoc queries
Highly relational workloads
Frequent cross-partition queries
Complex multi-partition transactions
Unknown query patterns
```

The key Cassandra rule:

> **Design tables around the queries.**

------------------------------------------------------------------------

# 4.12 Example: Tweeter Timeline

Suppose:

``` text
Get the latest 20 tweets
for user 123
```

A possible model:

``` text
tweets_by_user

Partition:
    user_id

Clustering:
    created_at
```

Then:

``` text
user_123
 ├── tweet_102  ← newest
 ├── tweet_101
 ├── tweet_100
 └── ...
```

This is exactly the type of predictable access pattern where Cassandra
becomes attractive.

------------------------------------------------------------------------

# 5. Graph Databases

## 5.1 Mental Model

Graph databases model:

``` text
Nodes + Relationships
```

Example:

``` text
Deepak ──follows──> Alice
   │
   └──follows──> Bob

Alice ──follows──> Charlie
```

The relationship is first-class data.

------------------------------------------------------------------------

# 5.2 Data Model

### Node

Represents an entity.

``` text
User
Product
Company
Account
```

### Edge

Represents a relationship.

``` text
FOLLOWS
BOUGHT
WORKS_AT
TRANSFERRED_TO
```

### Property

Metadata associated with a node or relationship.

Example:

``` text
FOLLOWS
{
    created_at: ...
}
```

------------------------------------------------------------------------

# 5.3 Graph Traversal

The key operation is traversal.

Example:

``` text
Deepak
  ↓ follows
Alice
  ↓ follows
Charlie
```

Question:

``` text
Who are Deepak's friends-of-friends?
```

This naturally maps to graph traversal.

------------------------------------------------------------------------

# 5.4 Use Cases

### Social networks

``` text
Who follows whom?
Mutual connections?
Friends of friends?
```

### Recommendation systems

``` text
User
 ↓ bought
Product
 ↓ belongs_to
Category
```

### Fraud detection

``` text
Account
   ↓ uses
Phone
   ↓ used_by
Account B
```

Relationship patterns can reveal suspicious behavior.

### Knowledge graphs

``` text
Person
 ↓ works_at
Company
 ↓ located_in
Country
```

------------------------------------------------------------------------

# 5.5 Examples

-   Neo4j
-   Amazon Neptune
-   JanusGraph

------------------------------------------------------------------------

# 5.6 Graph Database Strengths

Excellent when:

``` text
Relationships are first-class
+
Queries involve multi-hop traversal
```

The graph model can make these queries much more natural than repeatedly
joining relational tables.

------------------------------------------------------------------------

# 5.7 Graph Database Limitations

Potentially poor fit for:

``` text
Simple key lookups
Massive append-only workloads
Basic CRUD
Large analytical scans
Workloads where relationships are not important
```

Do not choose a graph database simply because the application has
relationships.

The question is:

> **Are relationship traversals a dominant workload?**

------------------------------------------------------------------------

# 6. Specialized NoSQL

Classic NoSQL models are not enough for every workload.

Some databases specialize around a particular query pattern.

``` text
Specialized
│
├── Search
├── Time-Series
└── Vector
```

------------------------------------------------------------------------

# 6.1 Search Databases

### Problem

Answer:

> **"Find and rank relevant documents."**

Typical requirements:

``` text
Full-text search
Relevance ranking
Fuzzy matching
Typo tolerance
Autocomplete
Filtering
Faceting
Highlighting
```

### Examples

``` text
Elasticsearch
OpenSearch
Solr
```

------------------------------------------------------------------------

## Inverted Index

A major search technique is an inverted index.

Conceptually:

``` text
Term          Documents
-------------------------
iphone        doc1, doc7
battery       doc2, doc7
wireless      doc1, doc2, doc7
```

For:

``` text
"wireless battery"
```

the engine can quickly identify candidate documents and rank them.

------------------------------------------------------------------------

## Use Cases

``` text
E-commerce search
Log search
Documentation search
Content search
Autocomplete
```

------------------------------------------------------------------------

## Important HLD Pattern

Search databases are often **derived indexes**, not the source of truth.

``` text
              Primary DB
             Source of Truth
                   │
                   │ CDC / Events
                   ↓
             Search Index
                   │
                   ↓
              Search API
```

If the index is lost, it can generally be rebuilt from the source data.

------------------------------------------------------------------------

# 6.2 Time-Series Databases

### Problem

Answer:

> **"What happened to this measurement over time?"**

Example:

``` text
Timestamp       CPU
--------------------
10:00:00        42%
10:00:01        43%
10:00:02        47%
10:00:03        51%
```

Typical data:

``` text
CPU
Memory
Network
Latency
IoT measurements
Stock prices
Business metrics
```

### Examples

``` text
InfluxDB
Prometheus
TimescaleDB
OpenTSDB
```

------------------------------------------------------------------------

## Typical Queries

``` text
CPU for server-123
between 10:00 and 11:00
```

or:

``` text
Average CPU
per 5 minutes
for the last 24 hours
```

These workloads are heavily oriented around:

``` text
Time range
+
Aggregation
+
Metrics
```

------------------------------------------------------------------------

## Why Specialized?

Time-series systems can optimize for:

-   Time-range queries
-   Time-based partitioning
-   Compression
-   Retention policies
-   Downsampling
-   Aggregation

Example:

``` text
Raw data
1-second resolution
     ↓
Keep 7 days

5-minute aggregates
     ↓
Keep 90 days

1-hour aggregates
     ↓
Keep 2 years
```

------------------------------------------------------------------------

# 6.3 Vector Databases

### Problem

Answer:

> **"Find data that is semantically similar to this data."**

Traditional search:

``` text
"wireless headphones"
```

tries to match words and relevance signals.

Vector search can answer:

``` text
"headphones for travel"
```

and find:

``` text
"wireless noise-cancelling headphones"
```

because their embeddings are semantically similar.

------------------------------------------------------------------------

## Embeddings

An embedding model converts data into a numerical vector:

``` text
Text / Image / Audio
        ↓
Embedding Model
        ↓
Vector
```

Example:

``` text
"How do I reset my password?"
        ↓
[0.12, -0.43, 0.87, ...]
```

A semantically similar sentence produces a nearby vector.

------------------------------------------------------------------------

## Examples

``` text
Pinecone
Milvus
Weaviate
Qdrant
```

Traditional databases and search engines can also support vector search,
so a separate vector database is not always required.

------------------------------------------------------------------------

## Use Cases

``` text
RAG
Semantic search
Recommendations
Image similarity
Duplicate detection
AI knowledge retrieval
```

------------------------------------------------------------------------

# 6.4 Vector Search + RAG

A common architecture:

``` text
                    User Question
                          │
                          ↓
                   Embedding Model
                          │
                          ↓
                     Query Vector
                          │
                          ↓
                    ┌───────────┐
                    │ Vector DB │
                    └─────┬─────┘
                          │
                  Similar Documents
                          │
                          ↓
                         LLM
                          │
                          ↓
                       Answer
```

During ingestion:

``` text
Documents
    ↓
Chunking
    ↓
Embedding Model
    ↓
Vectors
    ↓
Vector DB
```

------------------------------------------------------------------------

# 7. Search vs Vector Search

This is an important distinction.

  ------------------------------------------------------------------------------
                          Search                  Vector
  ----------------------- ----------------------- ------------------------------
  Primary goal            Find relevant           Find semantically similar data
                          text/documents          

  Matching                Keywords + text signals Vector similarity

  Typical input           Text query              Query embedding

  Example                 "wireless headphones"   "headphones for travel"

  Use cases               Product/log/content     RAG/recommendations/semantic
                          search                  search
  ------------------------------------------------------------------------------

Modern systems can combine both.

------------------------------------------------------------------------

# 7.1 Hybrid Search

``` text
                 User Query
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
    Keyword Search        Vector Search
          │                     │
          └──────────┬──────────┘
                     ↓
               Hybrid Ranking
                     ↓
                  Results
```

This can provide both:

``` text
Exact keyword matching
+
Semantic understanding
```

------------------------------------------------------------------------

# 8. Complete NoSQL Comparison

  ------------------------------------------------------------------------
  Model             Mental Model      Dominant Query    Typical Use Case
  ----------------- ----------------- ----------------- ------------------
  Key-Value         Distributed       Get by key        Cache/session
                    HashMap                             

  Document          JSON/object       Query             Profiles/catalog
                                      entity/document   

  Wide-Column       Partitioned       Known access      Massive-scale
                    distributed data  pattern           events/feed

  Graph             Nodes + edges     Relationship      Social/fraud
                                      traversal         

  Search            Search index      Find/rank text    Product/log search

  Time-Series       Measurements over Time range +      Metrics/IoT
                    time              aggregation       

  Vector            Embeddings        Similarity        RAG/semantic
                                                        search
  ------------------------------------------------------------------------

------------------------------------------------------------------------

# 9. How to Choose?

Database selection should be driven by several dimensions.

``` text
                Database Selection
                       │
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
   Data Model      Access Pattern      Scale
       │               │                │
       └───────────────┼────────────────┘
                       ↓
                 Requirements
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
     Consistency   Availability   Complexity
```

------------------------------------------------------------------------

# 9.1 Access Patterns

Start with:

> **What questions will the system ask the database?**

Examples:

``` text
GET user by ID
        → Key-Value

Find product by multiple attributes
        → Document

Get user's recent events
        → Wide-Column

Find friends of friends
        → Graph

Search "running shoes"
        → Search

Average CPU over the last hour
        → Time-Series

Find documents semantically similar to a question
        → Vector
```

------------------------------------------------------------------------

# 9.2 Scale

Estimate:

``` text
Reads/sec
Writes/sec
Storage
Data growth/day
Peak traffic
Partition size
```

A database choice that works for:

``` text
100 requests/sec
```

may not be appropriate for:

``` text
1 million requests/sec
```

------------------------------------------------------------------------

# 9.3 Consistency

Ask:

``` text
Does every read need the latest value?
```

Possible requirements:

``` text
Strong consistency
Eventual consistency
Read-your-writes
Causal consistency
```

The correct model depends on the business requirement.

Examples:

``` text
Payment balance
→ stronger consistency

Social feed
→ may tolerate eventual consistency

Search index
→ eventual consistency often acceptable

Analytics
→ eventual consistency usually acceptable
```

------------------------------------------------------------------------

# 9.4 Availability

Ask:

> **What should happen when a node/region is unavailable?**

Possible priorities:

``` text
Consistency
vs
Availability
```

This connects directly to:

-   CAP theorem
-   PACELC
-   Replication
-   Quorum
-   Failure handling

------------------------------------------------------------------------

# 9.5 Query Complexity

Ask:

``` text
Do we need joins?
Do we need arbitrary filtering?
Do we need aggregations?
Do we need relationship traversal?
Do we need full-text search?
Do we need semantic similarity?
```

The more specialized the query, the more useful a specialized database
can become.

------------------------------------------------------------------------

# 9.6 Operational Complexity

Always ask:

> **Do we actually need another database?**

For a small system:

``` text
PostgreSQL
    ↓
Everything works
```

may be better than:

``` text
PostgreSQL
+ Redis
+ Cassandra
+ Elasticsearch
+ Kafka
+ Neo4j
+ Vector DB
```

The latter creates:

-   More infrastructure
-   More monitoring
-   More failure modes
-   More operational knowledge
-   More data synchronization
-   More backup/recovery concerns

------------------------------------------------------------------------

# 10. Polyglot Persistence

Using multiple database technologies in one system is called:

> **Polyglot persistence**

Example:

``` text
                    Application
                         │
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
   PostgreSQL          Redis          Cassandra
   Transactions        Cache          Activity
        │
        │
        └───────────────┐
                        ↓
                  Elasticsearch
                      Search
```

And separately:

``` text
Metrics
   ↓
Prometheus / Time-Series DB
```

Modern AI systems may additionally use:

``` text
Vector DB
   ↓
Semantic retrieval
```

------------------------------------------------------------------------

# 10.1 Why Polyglot Persistence?

Because one database rarely provides the best solution for every
workload.

For example:

``` text
PostgreSQL
→ transactional source of truth

Redis
→ low-latency cache

Cassandra
→ massive-scale activity data

Elasticsearch
→ text search

Vector DB
→ semantic retrieval

Prometheus
→ infrastructure metrics
```

Each database is selected for a specific workload.

------------------------------------------------------------------------

# 11. Primary DB vs Derived Stores

A very important HLD distinction is:

``` text
Canonical data
       vs
Derived data
```

Example:

``` text
                Primary DB
              Source of Truth
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
       Search     Cache     Analytics
        Index
```

Derived stores may be rebuilt from canonical data.

This is common with:

``` text
Search indexes
Caches
Materialized views
Analytics stores
Vector indexes
```

This distinction affects:

-   Consistency
-   Recovery
-   Data ownership
-   Failure handling

------------------------------------------------------------------------

# 12. Database Selection Examples

## Example 1: User Session

Requirement:

``` text
Given session ID,
return session data.
```

Choice:

``` text
Key-Value
```

Why?

``` text
Exact key lookup
+
Low latency
```

------------------------------------------------------------------------

## Example 2: Product Catalog

Requirement:

``` text
Product contains flexible attributes,
variants and metadata.
```

Choice:

``` text
Document
```

Why?

``` text
Natural object representation
+
Flexible schema
```

------------------------------------------------------------------------

## Example 3: Social Feed

Requirement:

``` text
Billions of events
High write throughput
Get recent events for user
```

Choice:

``` text
Wide-Column / Cassandra
```

Why?

``` text
Predictable partition-oriented queries
+
Horizontal scale
+
High write throughput
```

------------------------------------------------------------------------

## Example 4: Social Relationships

Requirement:

``` text
Friends of friends
Mutual connections
Relationship traversal
```

Choice:

``` text
Graph
```

Why?

``` text
Relationships are first-class
```

------------------------------------------------------------------------

## Example 5: Product Search

Requirement:

``` text
Search:
"running shoes"
with filters and ranking
```

Choice:

``` text
Search DB
```

Why?

``` text
Full-text search
+
Relevance
+
Filtering
```

------------------------------------------------------------------------

## Example 6: Infrastructure Monitoring

Requirement:

``` text
Store CPU/memory/latency
and query historical ranges.
```

Choice:

``` text
Time-Series DB
```

Why?

``` text
Time-based ingestion
+
Range queries
+
Aggregation
+
Retention
```

------------------------------------------------------------------------

## Example 7: Company Knowledge Assistant

Requirement:

``` text
Ask:
"What is our process for requesting access?"
```

Choice:

``` text
Vector Search
+
LLM
```

Why?

``` text
Semantic retrieval
+
RAG
```

------------------------------------------------------------------------

# 13. Interview Decision Tree

``` text
                       Start
                         │
                         ↓
                 What is the workload?
                         │
        ┌────────────────┼─────────────────┐
        ↓                ↓                 ↓
    Simple key       Flexible entity    Relationships
     lookup             query                │
        │                │                   ↓
        ↓                ↓                 Graph
    Key-Value        Document
                         │
                         ↓
                 Massive predictable
                   distributed load
                         │
                         ↓
                    Wide-Column
```

Specialized workloads:

``` text
Full-text discovery
        ↓
     Search

Measurements over time
        ↓
   Time-Series

Semantic similarity
        ↓
      Vector
```

------------------------------------------------------------------------

# 14. Interview Questions

## Q: Why Cassandra instead of PostgreSQL?

> "If the workload requires very high write throughput, horizontal
> scaling, high availability, and predictable query patterns, Cassandra
> can be a strong fit. I'd design partitions around the required access
> patterns. If the workload requires complex joins, transactions, or
> flexible ad-hoc queries, I'd prefer PostgreSQL."

------------------------------------------------------------------------

## Q: Why MongoDB instead of Cassandra?

> "MongoDB is a better fit when the data naturally forms flexible
> documents and we need richer document queries. Cassandra is more
> appropriate when the workload is primarily predictable,
> partition-oriented access at very large scale."

------------------------------------------------------------------------

## Q: Why not Redis for everything?

> "Redis is excellent for low-latency in-memory workloads, but it is not
> a universal database replacement. Durability, cost, data volume, query
> requirements, and consistency requirements need to be considered."

------------------------------------------------------------------------

## Q: Why use a graph database?

> "When relationships are first-class and the workload requires
> multi-hop traversal, a graph model can make those queries much more
> natural than repeatedly joining or traversing relational tables."

------------------------------------------------------------------------

## Q: Is NoSQL always eventually consistent?

> "No. NoSQL is a broad category. Consistency guarantees depend on the
> specific database and configuration."

------------------------------------------------------------------------

## Q: Why use Elasticsearch if PostgreSQL already has the data?

> "PostgreSQL can support many search requirements, but if full-text
> search, relevance ranking, fuzzy matching, autocomplete, or
> large-scale search becomes a dominant workload, a dedicated search
> index can provide a better fit. PostgreSQL remains the source of
> truth."

------------------------------------------------------------------------

# 15. Common Mistakes

### Mistake 1

> "NoSQL is faster than SQL."

Better:

> Performance depends on the workload and data model.

------------------------------------------------------------------------

### Mistake 2

> "NoSQL has no schema."

Better:

> NoSQL often provides flexible or query-driven schemas, but deliberate
> data modeling is still required.

------------------------------------------------------------------------

### Mistake 3

> "Cassandra is just distributed SQL."

Better:

> Cassandra is designed around partitioning, replication, availability,
> and query-driven data modeling.

------------------------------------------------------------------------

### Mistake 4

> "Use Cassandra because the data is huge."

Better:

> First identify access patterns, write/read volume, partitioning
> strategy, consistency, and availability requirements.

------------------------------------------------------------------------

### Mistake 5

> "Use Elasticsearch as the primary database."

Usually incorrect.

A common architecture is:

``` text
Primary DB
   ↓
CDC / Events
   ↓
Search Index
```

------------------------------------------------------------------------

### Mistake 6

> "Use a graph DB because there are relationships."

Almost every real system has relationships.

The stronger reason is:

> **The workload is dominated by relationship traversal.**

------------------------------------------------------------------------

# 16. One-Page Cheat Sheet

``` text
╔══════════════════════════════════════════════╗
║                 NoSQL                        ║
╚══════════════════════════════════════════════╝

KEY-VALUE
──────────
"I know the key."

Examples:
Redis, DynamoDB

Use:
Cache, sessions, carts, counters


DOCUMENT
────────
"I know the entity/object."

Examples:
MongoDB, Couchbase

Use:
Profiles, catalogs, content, metadata


WIDE-COLUMN
───────────
"I know my access pattern."

Examples:
Cassandra, ScyllaDB, HBase, Bigtable

Use:
Massive distributed workloads,
events, feeds, telemetry


GRAPH
─────
"I care about relationships."

Examples:
Neo4j, Neptune

Use:
Social graph, fraud,
recommendations, knowledge graphs


SEARCH
──────
"I need to find and rank relevant text."

Examples:
Elasticsearch, OpenSearch

Use:
Product search, logs, content


TIME-SERIES
───────────
"I need to analyze measurements over time."

Examples:
Prometheus, InfluxDB, TimescaleDB

Use:
Metrics, monitoring, IoT


VECTOR
──────
"I need semantic similarity."

Examples:
Pinecone, Milvus, Weaviate, Qdrant

Use:
RAG, semantic search, recommendations
```

------------------------------------------------------------------------

# 17. Final Mental Model

Do not memorize databases as isolated technologies.

Memorize the question each model is designed to answer:

``` text
Key-Value
    ↓
"Give me the value for this key."

Document
    ↓
"Give me/query this flexible entity."

Wide-Column
    ↓
"Give me massive-scale data using
 my known access pattern."

Graph
    ↓
"How are these entities connected?"

Search
    ↓
"Find and rank relevant documents."

Time-Series
    ↓
"What happened to this metric over time?"

Vector
    ↓
"What data is semantically similar?"
```

The strongest HLD mindset is:

> **Start with requirements and access patterns. Then choose the data
> model. Then evaluate scale, consistency, availability, partitioning,
> and operational complexity.**

Do not start with:

> "I want to use Cassandra."

Start with:

> **"Here are the queries, traffic, data volume, consistency
> requirements, and failure expectations. Cassandra is a good fit
> because..."**
