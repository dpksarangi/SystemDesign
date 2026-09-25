# Geo Search & Location Indexing

## 1. Problem

Many systems need to answer:

> Find objects/users within a geographical area or nearest to a given location.

Examples:

- Find nearby Uber drivers
- Find nearby Swiggy delivery partners
- Find restaurants within 3 km
- Find nearby hospitals
- Find stores near a customer
- Find available cabs in a region
- Geofencing
- Find the nearest warehouse

A location is typically represented as:

```text
latitude  = 12.9716
longitude = 77.5946
```

The challenge is efficiently answering:

```text
Find all objects within X km of (lat, lon)
```

at large scale.

---

# 2. Why Can't We Just Store Latitude / Longitude?

A naive database query would look conceptually like:

```sql
WHERE distance(latitude, longitude, user_lat, user_lon) < 2 km
```

This becomes expensive at large scale because the database may need to examine a very large number of records.

The key idea is:

> **Use a spatial index to reduce the search space before calculating exact distance.**

There are two major approaches:

```text
1. Native spatial indexing
   ├── PostGIS
   ├── MongoDB 2dsphere
   ├── Elasticsearch/OpenSearch
   └── Redis GEO

2. Spatial cell indexing
   ├── Geohash
   ├── H3
   └── S2
```

---

# 3. Spatial Cell Indexing

Instead of storing only:

```text
latitude + longitude
```

convert the location into a **spatial cell ID**.

```text
lat/lon
   |
   v
Spatial encoding
   |
   +--> Geohash
   +--> H3
   +--> S2
```

The Earth is divided into cells.

```text
+---------+---------+---------+
|         |         |         |
|   C1    |   C2    |   C3    |
|         |         |         |
+---------+---------+---------+
|         |         |         |
|   C4    |   C5    |   C6    |
|         |   *     |         |
+---------+---------+---------+
|         |         |         |
|   C7    |   C8    |   C9    |
|         |         |         |
+---------+---------+---------+

              *
           Customer
```

The customer belongs to C5.

Instead of searching the entire database:

```text
Search entire world
```

we search:

```text
C5 + neighboring cells
```

and then calculate exact distances.

---

# 4. Geohash

Geohash converts latitude/longitude into a string representing a geographical region.

```text
lat/lon
   |
   v
Geohash
   |
   v
"tdr1..."
```

Nearby locations generally share a geohash prefix.

Example:

```text
tdr1abc
tdr1abd
tdr1abe
```

The common prefix represents a common geographical region.

### Important limitation

Geohash cells are rectangular and have boundary problems.

Two very close points can fall into different cells:

```text
+-------------+-------------+
|             |             |
|             |     *       |
|             |             |
|             |-------------|
|             |     *       |
|             |             |
+-------------+-------------+

Two nearby objects
may have different geohashes.
```

Therefore, nearby-cell searches are required.

---

# 5. H3

H3 is a hierarchical geospatial indexing system that divides the Earth into **hexagonal cells**.

```text
       __
    __/  \__
   /        \
   \        /
    \__  __/
       \/
```

A location is converted into an H3 cell ID:

```text
lat/lon
   |
   v
H3 cell
```

H3 supports multiple resolutions:

```text
World
  |
  +-- Large cells
        |
        +-- Smaller cells
              |
              +-- Smaller cells
                    |
                    +-- ...
```

This allows the application to choose a resolution appropriate for the use case.

H3 is particularly useful when building a distributed geo-index:

```text
H3 cell → objects
```

Example:

```text
cell_abc → driver1, driver2, driver3
cell_def → driver4, driver5
```

---

# 6. S2

S2 is another hierarchical spatial indexing system.

Conceptually:

```text
Earth
  |
  v
Large S2 cells
  |
  v
Smaller cells
  |
  v
Smaller cells
```

S2 is based on mapping the Earth's surface onto a hierarchy of cells.

The important HLD concept is:

> H3, Geohash and S2 are different ways of converting geographic coordinates into searchable spatial regions.

---

# 7. Nearby Search Using Spatial Cells

Suppose a customer wants a driver within 2 km.

### Step 1 — Convert customer location

```text
Customer lat/lon
       |
       v
    H3 cell
```

### Step 2 — Find neighboring cells

```text
       C1
    C2 C3 C4
       C5
    C6 C7 C8
```

Depending on the search radius and cell resolution, determine which cells need to be searched.

### Step 3 — Query the index

```text
C5
C1
C2
C3
...
```

### Step 4 — Retrieve candidates

```text
Driver A
Driver B
Driver C
Driver D
```

### Step 5 — Calculate exact distance

Spatial cells only provide **candidate filtering**.

We still calculate:

```text
distance(customer, driver)
```

using the actual latitude/longitude.

### Step 6 — Filter and rank

```text
distance <= 2 km
```

Then:

```text
sort by distance
```

Final flow:

```text
Customer location
       |
       v
H3 / Geohash
       |
       v
Current + neighboring cells
       |
       v
Candidate objects
       |
       v
Exact distance calculation
       |
       v
Filter
       |
       v
Sort / Rank
```

---

# 8. Cassandra + H3

Cassandra does not provide the same rich spatial query capabilities as PostGIS.

Instead, we design the access pattern around the spatial cell.

Example:

```sql
CREATE TABLE drivers_by_geo_cell (
    geo_cell text,
    driver_id uuid,
    latitude double,
    longitude double,
    status text,
    updated_at timestamp,
    PRIMARY KEY (geo_cell, driver_id)
);
```

Conceptually:

```text
geo_cell     drivers
----------------------------
C1           D101, D102
C2           D103
C3           D104, D105
C4           D106
```

Query flow:

```text
customer
   |
   v
calculate H3
   |
   v
find neighboring cells
   |
   v
query Cassandra
   |
   v
exact distance
```

### Why this works well with Cassandra

Cassandra is designed around:

> **Partition key → efficiently retrieve data**

So we turn:

```text
geo_cell
```

into the partitioning/access key.

---

# 9. Partition Size Problem

Spatial indexing introduces an important tradeoff.

### Cells too large

```text
+-------------------+
|                   |
|  100,000 drivers  |
|                   |
+-------------------+
```

One partition becomes too large.

### Cells too small

```text
+--+--+--+--+--+--+
|  |  |  |  |  |  |
+--+--+--+--+--+--+
|  |  |  |  |  |  |
+--+--+--+--+--+--+
```

A search may need to query many cells.

Therefore:

> **Choose spatial resolution based on object density, search radius and expected query volume.**

---

# 10. PostGIS

PostGIS is a geospatial extension for PostgreSQL.

It adds:

- Geographic data types
- Spatial indexes
- Distance calculations
- Polygon operations
- Spatial relationships
- Geofencing
- Nearest-neighbor queries

Example:

```sql
location geography(Point, 4326)
```

Then:

```sql
SELECT *
FROM drivers
WHERE ST_DWithin(
    location,
    ST_MakePoint(77.5946, 12.9716)::geography,
    2000
);
```

Meaning:

> Find drivers within 2,000 meters.

PostGIS is useful when we want **rich spatial querying** rather than building the spatial indexing logic ourselves.

---

# 11. Redis GEO

Redis provides native geospatial operations.

Example:

```text
GEOADD drivers 77.5946 12.9716 driver123
```

Then:

```text
GEOSEARCH drivers
  FROMLONLAT 77.5946 12.9716
  BYRADIUS 2 km
```

This makes Redis attractive for:

> **Very fast, real-time proximity searches.**

Typical architecture:

```text
Driver App
    |
    | GPS updates
    v
Location Service
    |
    v
Redis GEO
    |
    v
Nearby drivers
```

Redis is generally treated as a **hot/current-state store**, not necessarily the durable source of all location history.

---

# 12. Elasticsearch / OpenSearch

Elasticsearch/OpenSearch supports geographic fields such as:

```text
geo_point
```

and geo-distance queries.

Useful when geo search needs to be combined with other search/filtering requirements:

```text
Find restaurants within 3 km
AND
open now
AND
rating > 4
AND
cuisine = Indian
```

Architecture:

```text
Application
    |
    v
Elasticsearch/OpenSearch
    |
    +--> geo filter
    +--> text search
    +--> business filters
```

---

# 13. MongoDB 2dsphere

MongoDB supports geospatial indexing through the `2dsphere` index.

It can support operations such as:

```text
$near
$geoWithin
```

Useful when MongoDB is already the application's primary datastore.

---

# 14. Comparison

| Solution | Main Idea | Strength |
|---|---|---|
| **PostGIS** | Native spatial database | Rich geo queries |
| **Redis GEO** | In-memory geo index | Very fast real-time lookup |
| **Elasticsearch/OpenSearch** | Search + geo index | Geo + filtering/search |
| **MongoDB 2dsphere** | Native Mongo geo index | Simple application integration |
| **Cassandra + H3** | Cell-based distributed index | Large distributed workloads |
| **Geohash + KV/DB** | Cell-based lookup | Simple and flexible |
| **S2 + KV/DB** | Hierarchical spatial cells | Large-scale spatial indexing |
| **H3 + Redis** | Spatial cells + in-memory store | Fast real-time geo lookup |

---

# 15. Uber / Swiggy Example

For a ride-sharing system:

```text
Driver
   |
   | GPS updates
   v
Location Service
   |
   +--------------------+
   |                    |
   v                    v
Kafka              Hot Geo Index
                      |
                 Redis GEO /
                 H3 + Redis
                      |
                      v
                 Nearby Driver
                     Search
```

The location service receives frequent updates.

For a rider request:

```text
Rider
  |
  | pickup location
  v
Matching Service
  |
  v
Geo Index
  |
  v
Candidate drivers
  |
  v
Exact distance
  |
  v
Availability / ETA / vehicle type
  |
  v
Ranking
  |
  v
Selected driver
```

---

# 16. Why Not Cassandra for Every GPS Update?

Drivers can send location updates every few seconds.

At large scale:

```text
1 million drivers
×
1 update / 5 seconds
=
200,000 location updates/sec
```

Writing every update directly into a durable Cassandra table may be unnecessary if the primary requirement is:

> "Where is the driver right now?"

A better architecture may separate:

```text
Hot state
    ↓
Redis / specialized geo index

Durable events/history
    ↓
Kafka
    ↓
Cassandra / data lake
```

This is an important distinction:

> **Current location and location history have different storage requirements.**

---

# 17. Geofencing

Geo search is not only about "nearest".

Another common requirement is:

> "Did the driver enter this geographical region?"

```text
Airport
+-----------------------+
|                       |
|       Geofence        |
|          *            |
|                       |
+-----------------------+
```

Applications can detect:

```text
ENTERED region
EXITED region
INSIDE region
```

Use cases:

- Airport pickup zones
- Delivery zones
- Surge zones
- Store boundaries
- Restricted areas
- Service availability regions

PostGIS, H3, S2, Elasticsearch and other spatial systems can all participate in such designs depending on the exact requirement.

---

# 18. Important HLD Insight

Don't start with:

> "Which database supports geo?"

Start with:

> **"What type of geographic query do I need?"**

### A. Simple nearby lookup

```text
Redis GEO
```

### B. Rich spatial queries

```text
PostGIS
```

### C. Search + geo + filters

```text
Elasticsearch/OpenSearch
```

### D. Massive distributed cell-based indexing

```text
H3 / S2 / Geohash
        +
distributed datastore
```

### E. Real-time + large scale

A hybrid approach is often appropriate:

```text
                    Location Updates
                           |
                           v
                    Location Service
                           |
                     +-----+-----+
                     |           |
                     v           v
                   Kafka      Hot Geo
                     |         Index
                     |       Redis/H3
                     |
                     v
               Durable Storage
             Cassandra / Data Lake
```

---

# 19. Interview Questions

### Fundamentals

- How do you find nearby users?
- Why can't we simply query latitude/longitude?
- What is a geohash?
- What is H3?
- What is S2?
- Why do we need neighboring cells?
- Why calculate exact distance after spatial lookup?

### Database

- Why use PostGIS instead of Cassandra?
- Why use Redis GEO?
- When would you choose Elasticsearch geo queries?
- How would you partition Cassandra by geographic location?
- What happens if one geographic cell becomes extremely hot?
- How do you handle uneven geographic density?

### Scale

- What happens if there are 1M active drivers?
- Drivers send GPS updates every 5 seconds — how do you handle the write volume?
- Would you write every location update to Cassandra?
- How would you scale geo search horizontally?
- How do you handle a city center having 100x the driver density of a rural area?

### Consistency

- What happens if a driver's location is stale?
- How stale can a driver's location be before removing them from matching?
- What happens if Redis loses data?
- Where is the source of truth?
- Do we need strong consistency for driver location?

### Advanced

- How would you implement geofencing?
- How would you find the nearest N drivers?
- How would you dynamically change H3 resolution?
- How would you combine proximity with ETA?
- How would you prevent all requests from querying the same hot geographic cell?

---

# 20. One-Line Mental Model

```text
LAT/LON
   ↓
Spatial Index
   ↓
Reduce Search Space
   ↓
Candidate Objects
   ↓
Exact Distance
   ↓
Filter / Rank
   ↓
Result
```

**The most important HLD concept:**

> **Geo indexing is usually a candidate-generation problem first, and an exact-distance problem second.**
