# Specialized NoSQL Databases

> **Core idea:** Specialized NoSQL databases are optimized around a
> particular **access pattern or workload**, rather than trying to be a
> general-purpose database for everything.

------------------------------------------------------------------------

## 1. Why Specialized Databases?

The classic NoSQL families solve different data-model problems:

-   **Key-Value** → lookup by key
-   **Document** → store/query flexible objects
-   **Wide-Column** → massive distributed workloads with predictable
    access patterns
-   **Graph** → relationships and traversals

Specialized databases go one step further:

> **Optimize the storage and indexing model for a very specific kind of
> query.**

The three important categories for HLD are:

``` text
Specialized NoSQL
│
├── Search
├── Time-Series
└── Vector
```

------------------------------------------------------------------------

# 2. Search Databases

## 2.1 What problem do they solve?

A search database answers:

> **"Find and rank the most relevant documents matching what the user is
> looking for."**

This is different from an ordinary database lookup.

Traditional database:

``` text
Get user where user_id = 123
```

Search:

``` text
"wireless headphones with good battery"
```

The user may not know the exact product, ID, or wording.

Search engines optimize for:

-   Full-text search
-   Relevance/ranking
-   Fuzzy matching
-   Typo tolerance
-   Autocomplete
-   Filtering
-   Faceting
-   Highlighting

------------------------------------------------------------------------

## 2.2 How does search work?

A major technique is the **inverted index**.

Conceptually:

``` text
Term          Documents
-------------------------
iphone        doc1, doc7, doc20
battery       doc2, doc7, doc15
wireless      doc1, doc2, doc7
```

For:

``` text
"wireless battery"
```

the engine can quickly find documents associated with those terms and
rank them.

------------------------------------------------------------------------

## 2.3 Examples

-   Elasticsearch
-   OpenSearch
-   Apache Solr

------------------------------------------------------------------------

## 2.4 Common use cases

### E-commerce

``` text
"running shoes"
```

Search by:

-   product name
-   description
-   brand
-   category
-   price
-   attributes

### Log search

``` text
Find ERROR logs
from service X
between 10:00 and 10:05
```

### Autocomplete

``` text
"amaz..."
      ↓
Amazon
Amazon Prime
Amazon Fresh
```

### Content search

Search across:

-   articles
-   documentation
-   support tickets
-   messages

------------------------------------------------------------------------

## 2.5 Search DB is usually not the source of truth

A common HLD architecture is:

``` text
                 ┌──────────────┐
                 │  Primary DB  │
                 │ Source Truth │
                 └──────┬───────┘
                        │
                   CDC / Events
                        │
                        ↓
                 ┌──────────────┐
                 │ Search Index │
                 └──────┬───────┘
                        │
                        ↓
                   Search API
```

The search database contains a **derived, read-optimized
representation**.

If the search index is lost, it can generally be rebuilt from the
primary data.

### Interview takeaway

> Use a search database when the dominant requirement is **full-text
> discovery and relevance ranking**, not transactional correctness.

------------------------------------------------------------------------

# 3. Time-Series Databases

## 3.1 What problem do they solve?

A time-series database answers:

> **"What happened to this measurement over time?"**

The data naturally contains a strong time dimension.

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

-   CPU utilization
-   Memory usage
-   Network traffic
-   Application latency
-   IoT sensor readings
-   Stock prices
-   Business metrics

------------------------------------------------------------------------

## 3.2 Typical queries

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

The workload is heavily centered around:

``` text
time range
+
aggregation
+
metrics
```

------------------------------------------------------------------------

## 3.3 Examples

-   InfluxDB
-   Prometheus
-   TimescaleDB
-   OpenTSDB

------------------------------------------------------------------------

## 3.4 Why not just use a normal database?

You can.

For moderate workloads, PostgreSQL or another relational database can
absolutely store timestamped data.

But at very high metric volumes, time-series systems can optimize for:

-   Time-range queries
-   Time-based partitioning
-   Compression
-   Retention policies
-   Downsampling
-   Aggregations

Example retention strategy:

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

This dramatically reduces long-term storage while preserving useful
historical information.

------------------------------------------------------------------------

## 3.5 Common HLD example

Monitoring platform:

``` text
Servers
   │
   ↓
Metrics Collector
   │
   ↓
Kafka / ingestion
   │
   ↓
Time-Series DB
   │
   ├── Dashboards
   └── Alerting
```

### Interview takeaway

> Use a time-series database when data is primarily **measurements
> indexed by time**, especially when time-range queries, aggregation,
> retention, and high ingestion rates dominate.

------------------------------------------------------------------------

# 4. Vector Databases

## 4.1 What problem do they solve?

A vector database answers:

> **"Find data that is similar to this data."**

This is different from exact or keyword matching.

For example:

``` text
"I forgot my password"
```

and:

``` text
"How can I recover my login credentials?"
```

may contain different words but have similar meaning.

An embedding model converts text into a numerical vector:

``` text
"I forgot my password"
        ↓
[0.12, -0.43, 0.87, 0.21, ...]
```

Another sentence becomes another vector:

``` text
"How can I recover my login credentials?"
        ↓
[0.10, -0.40, 0.84, 0.24, ...]
```

The vectors are mathematically close.

A vector database performs **similarity search** over these vectors.

------------------------------------------------------------------------

## 4.2 What is an embedding?

An embedding is a numerical representation of data that captures useful
semantic characteristics.

Conceptually:

``` text
Text / Image / Audio
        ↓
Embedding Model
        ↓
Vector
```

For text:

``` text
"How do I reset my password?"
        ↓
[0.13, -0.52, 0.72, ...]
```

The exact vector dimensions depend on the embedding model.

------------------------------------------------------------------------

## 4.3 Similarity search

Suppose the database contains:

``` text
Document A → password reset
Document B → Kubernetes configuration
Document C → vacation policy
Document D → password recovery
```

User asks:

``` text
"I forgot my login password"
```

Vector search might return:

``` text
1. Document D → password recovery
2. Document A → password reset
3. Document B → Kubernetes
```

because the first two are semantically closest.

------------------------------------------------------------------------

## 4.4 Examples

-   Pinecone
-   Milvus
-   Weaviate
-   Qdrant

Many traditional databases and search engines also support vector
search, so a separate vector database is not always necessary.

------------------------------------------------------------------------

# 5. Vector DB + RAG

This is particularly important for modern system design.

A common RAG architecture:

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
                  ┌──────────────┐
                  │  Vector DB   │
                  └──────┬───────┘
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
    │
    ↓
Chunking
    │
    ↓
Embedding Model
    │
    ↓
Vectors
    │
    ↓
Vector DB
```

At query time:

``` text
Question
   ↓
Embedding
   ↓
Vector similarity search
   ↓
Relevant chunks
   ↓
LLM
   ↓
Answer
```

### Interview takeaway

> Use vector search when the dominant requirement is **semantic
> similarity**, such as RAG, recommendations, image similarity, or
> duplicate detection.

------------------------------------------------------------------------

# 6. Search vs Vector Search

This is an important interview distinction.

  -------------------------------------------------------------------------------
                          Search                    Vector
  ----------------------- ------------------------- -----------------------------
  Matching                Keywords / structured     Semantic similarity
                          text                      

  Example                 `"wireless headphones"`   `"headphones for travel"`

  Understands meaning     Limited / depends on      Yes, via embeddings
                          search features           

  Ranking                 Text relevance + signals  Vector similarity

  Typical use             Product/log/content       RAG/recommendation/semantic
                          search                    search
  -------------------------------------------------------------------------------

In modern systems, you can also combine them:

``` text
                 User Query
                     │
            ┌────────┴────────┐
            ↓                 ↓
      Keyword Search     Vector Search
            │                 │
            └────────┬────────┘
                     ↓
              Hybrid Ranking
                     ↓
                  Results
```

This is called **hybrid search**.

------------------------------------------------------------------------

# 7. All Three Compared

  -----------------------------------------------------------------------
  Database                Dominant question       Key optimization
  ----------------------- ----------------------- -----------------------
  Search                  "What documents are     Inverted indexes +
                          relevant?"              ranking

  Time-Series             "What happened over     Time-range storage +
                          time?"                  aggregation

  Vector                  "What is similar?"      Approximate/vector
                                                  similarity search
  -----------------------------------------------------------------------

A useful memory trick:

``` text
Search     → FIND
Time-Series → TRACK
Vector     → SIMILARITY
```

------------------------------------------------------------------------

# 8. Where They Fit in a Real System

A large system can legitimately use several databases:

``` text
                     Application
                         │
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
   Primary DB          Redis          Search DB
   Truth/Txns          Cache          Text Search
        │
        │
        └───────────────┐
                        ↓
                    Event Bus
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
        Time-Series   Vector    Analytics
            DB          DB          DB
```

This is called:

## Polyglot Persistence

> **Use different storage technologies for different workloads instead
> of forcing one database to solve every problem.**

The important architectural question is not:

> "Which database is fastest?"

It is:

> **"What is the dominant access pattern and workload, and which storage
> model is naturally optimized for it?"**

------------------------------------------------------------------------

# 9. HLD Decision Cheat Sheet

``` text
Need simple lookup by ID?
        ↓
    Key-Value

Need flexible JSON-like objects?
        ↓
    Document

Need huge distributed scale with
predictable access patterns?
        ↓
    Wide-Column

Need relationship traversal?
        ↓
    Graph

Need full-text search / relevance?
        ↓
    Search

Need metrics over time?
        ↓
    Time-Series

Need semantic similarity?
        ↓
    Vector
```

------------------------------------------------------------------------

# 10. Important Interview Caveat

**Do not blindly introduce specialized databases.**

For example:

``` text
Small application
     ↓
PostgreSQL
     ↓
Everything works
```

You don't need:

``` text
PostgreSQL
+ Redis
+ Cassandra
+ Elasticsearch
+ Kafka
+ Vector DB
+ Neo4j
```

just because they are available.

Introduce a specialized database when the workload actually benefits
from its specialized capabilities.

### Strong HLD answer

> "I'd start with the simplest storage model that satisfies the
> requirements. If search becomes a dominant workload, I'd introduce a
> search index. If semantic retrieval is required, I'd add vector
> search. If we're ingesting massive time-oriented metrics, I'd consider
> a time-series store."

That's a much stronger answer than:

> "Elasticsearch is faster than PostgreSQL."

------------------------------------------------------------------------

# Quick Interview Summary

``` text
                 Specialized NoSQL

Search
  → Full-text discovery
  → Relevance/ranking
  → Logs, products, content

Time-Series
  → Time-oriented measurements
  → Aggregation + retention
  → Monitoring, IoT, metrics

Vector
  → Semantic similarity
  → Embeddings + nearest-neighbor search
  → RAG, recommendations, AI search
```

### The one sentence to remember

> **Specialized databases exist because different workloads have
> different access patterns, and a database can be heavily optimized
> when it knows what kind of question it needs to answer.**
