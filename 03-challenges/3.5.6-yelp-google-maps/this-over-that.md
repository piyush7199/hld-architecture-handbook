# Yelp/Google Maps - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions for the Yelp/Google Maps geo-spatial
search system, explaining why specific technologies and approaches were chosen over alternatives.

---

## Table of Contents

1. [Geohash vs H3 Hexagonal Indexing](#1-geohash-vs-h3-hexagonal-indexing)
2. [Elasticsearch vs PostgreSQL PostGIS vs Redis Geo](#2-elasticsearch-vs-postgresql-postgis-vs-redis-geo)
3. [MongoDB vs PostgreSQL for Review Storage](#3-mongodb-vs-postgresql-for-review-storage)
4. [Geohash Prefix Sharding vs Hash-based Sharding](#4-geohash-prefix-sharding-vs-hash-based-sharding)
5. [Denormalization vs Real-time Joins](#5-denormalization-vs-real-time-joins)
6. [Caching Strategy: Redis vs Elasticsearch Cache](#6-caching-strategy-redis-vs-elasticsearch-cache)
7. [Adjacent Cell Querying vs Single Cell Query](#7-adjacent-cell-querying-vs-single-cell-query)
8. [Synchronous vs Asynchronous POI Updates](#8-synchronous-vs-asynchronous-poi-updates)

---

## 1. Geohash vs H3 Hexagonal Indexing

### The Problem

We need to transform 2D geographic coordinates (latitude, longitude) into a searchable 1D index that enables fast
proximity queries for millions of POIs. The indexing scheme must support:

- **Fast queries:** Sub-100ms proximity search for POIs within N km radius
- **Scalability:** Support for 50 million POIs globally
- **Accuracy:** No missed POIs within search radius
- **Sharding:** Enable geographic distribution of data

### Options Considered

| Factor                    | Geohash                                            | H3 (Uber's Hexagonal)                              | PostGIS R-Tree                 |
|---------------------------|----------------------------------------------------|----------------------------------------------------|--------------------------------|
| **Simplicity**            | ✅ String prefix matching, easy to understand       | ❌ Complex hexagonal math, steeper learning curve   | ❌ Complex SQL queries          |
| **Elasticsearch Support** | ✅ Native geo_point with geohash                    | ⚠️ Custom implementation required                  | ❌ Database-only                |
| **Boundary Accuracy**     | ⚠️ Edge cases at cell boundaries (10-15% error)    | ✅ Hexagons tile perfectly, minimal boundary issues | ✅ No boundaries                |
| **Cell Shape**            | ❌ Rectangular (variable aspect ratio by latitude)  | ✅ Hexagonal (uniform distance to neighbors)        | ✅ Optimal for spatial queries  |
| **Query Performance**     | ✅ O(log N) for index lookup                        | ✅ O(log N) similar performance                     | ⚠️ Slightly slower (10-50ms)   |
| **Storage Efficiency**    | ✅ Compact string representation (6-7 chars)        | ⚠️ Larger integers (64-bit)                        | ⚠️ Database-dependent          |
| **Adoption**              | ✅ Widely supported (Elasticsearch, MongoDB, Redis) | ⚠️ Uber-specific, limited tooling                  | ✅ Industry standard            |
| **Sharding**              | ✅ Easy (prefix-based sharding)                     | ⚠️ Custom sharding logic                           | ❌ Database-dependent           |
| **Development Cost**      | ✅ Low (use existing libraries)                     | ❌ High (custom indexing layer)                     | ✅ Moderate (PostGIS extension) |

### Decision Made

**Use Geohash for POI search system.**

### Rationale

1. **Elasticsearch Native Support:** Elasticsearch's `geo_point` field type internally uses Geohash encoding. This
   means:
    - Zero custom code needed for indexing
    - Optimized query performance out of the box
    - Proven reliability at scale (used by Yelp, Foursquare)

2. **Sharding Simplicity:** Geohash prefixes make geographic sharding straightforward:
   ```
   Shard 1: Geohash prefix "9q8" (San Francisco area)
   Shard 2: Geohash prefix "drm" (London area)
   ```
   This ensures all POIs in a geographic region are on the same shard (locality).

3. **Boundary Problem is Solvable:** When a query is near a cell boundary, we query the primary cell + 8 adjacent
   cells (9 total). This increases query latency from 50ms to 80ms, but only affects 15% of queries. The average latency
   remains acceptable at 60ms.

4. **Development Speed:** Using Geohash reduces development time by 3-6 months compared to building custom H3 indexing.
   Time-to-market is critical.

5. **Proven at Scale:** Yelp uses Geohash with Elasticsearch for their POI search. They handle millions of searches per
   day with sub-100ms latency.

### Implementation Details

**Geohash Precision Selection:**

- Use 7-character geohash (~153m × 153m cells) for POI-level accuracy
- Query radius: 5km (covers ~49 cells in worst case)
- Adjacent cell queries: Check 8 neighbors for boundary cases

**Code:**

```
function index_poi(poi_id, lat, lng):
  geohash = encode_geohash(lat, lng, precision=7)
  
  elasticsearch.index({
    "poi_id": poi_id,
    "location": {"lat": lat, "lon": lng},
    "geohash": geohash,
    ...
  })
```

*See [pseudocode.md::encode_geohash()](pseudocode.md) for full implementation.*

### Trade-offs Accepted

| What We Gain                                        | What We Sacrifice                                       |
|-----------------------------------------------------|---------------------------------------------------------|
| ✅ Fast development (no custom indexing)             | ❌ 10-15% of queries require 9-cell lookup (slower)      |
| ✅ Elasticsearch native support (proven reliability) | ❌ Cell boundaries cause 5-10% accuracy loss             |
| ✅ Easy to debug and monitor                         | ❌ Rectangular cells waste computation at high latitudes |
| ✅ Simple sharding strategy                          | ❌ Eventually may need migration to H3 at extreme scale  |

### When to Reconsider

**Switch to H3 when:**

1. **Scale threshold:** >100 million POIs globally (boundary overhead becomes costly)
2. **Accuracy requirement:** Need <2% miss rate (e.g., for autonomous navigation)
3. **Performance degradation:** Boundary queries exceed 25% (latency suffers)
4. **Cost justification:** Boundary query overhead costs more than $1M/year in infrastructure

**Indicators:**

- Elasticsearch query latency p99 > 150ms
- > 25% of queries require 9-cell lookup
- Customer complaints about "missing POIs" despite being within radius

---

## 2. Elasticsearch vs PostgreSQL PostGIS vs Redis Geo

### The Problem

We need a search engine that can handle:

- **11,574 QPS reads:** Proximity search queries
- **Complex filtering:** Category + rating + hours + custom attributes
- **Sub-100ms latency:** Search results must be extremely fast
- **50M POIs:** Store and query millions of POIs efficiently

### Options Considered

| Factor                     | Elasticsearch                                         | PostgreSQL + PostGIS                      | Redis Geo                              |
|----------------------------|-------------------------------------------------------|-------------------------------------------|----------------------------------------|
| **Read Latency**           | ✅ 20-50ms (p95)                                       | ⚠️ 50-100ms (disk I/O)                    | ✅ <5ms (in-memory)                     |
| **Write Throughput**       | ✅ 10K writes/sec per node                             | ❌ 1K writes/sec per node                  | ✅ 250K writes/sec per node             |
| **Complex Queries**        | ✅ Multi-dimensional filters (category, rating, hours) | ✅ SQL with multiple WHERE clauses         | ❌ Only proximity search                |
| **Full-Text Search**       | ✅ Native inverted index                               | ⚠️ PostgreSQL full-text search            | ❌ Not supported                        |
| **Geo-Spatial Queries**    | ✅ geo_distance, geo_bounding_box                      | ✅ ST_DWithin, ST_Distance                 | ✅ GEORADIUS                            |
| **Filtering**              | ✅ bool query with must/should/must_not                | ✅ SQL WHERE clauses                       | ❌ No filtering                         |
| **Scalability**            | ✅ Horizontal sharding (shard by Geohash prefix)       | ⚠️ Vertical scaling + read replicas       | ✅ Horizontal sharding (hash slots)     |
| **Data Durability**        | ✅ Index persisted to disk                             | ✅ ACID, WAL (zero data loss)              | ⚠️ RDB snapshots (potential data loss) |
| **Memory Usage**           | ⚠️ Moderate (disk + RAM cache)                        | ✅ Low (mostly disk)                       | ❌ High (all data in RAM)               |
| **Cost**                   | ⚠️ Moderate (15 nodes for 50M POIs)                   | ✅ Low (disk storage)                      | ❌ Expensive (all data in RAM)          |
| **Operational Complexity** | ⚠️ Requires cluster management                        | ⚠️ Requires connection pooling, vacuuming | ⚠️ Requires Redis Cluster + Sentinel   |

### Decision Made

**Use Elasticsearch as primary search engine for POI queries.**

### Rationale

1. **Complex Query Requirements:** Users need to filter by:
    - Location (within 5km radius)
    - Category (restaurant, hotel, etc.)
    - Rating (>4.0 stars)
    - Hours (open now)
    - Price range ($, $$, $$$)

   Elasticsearch's `bool` query handles all these filters in a single query:
   ```json
   {
     "query": {
       "bool": {
         "must": [
           {"geo_distance": {"distance": "5km", "location": {...}}},
           {"term": {"category": "restaurant"}},
           {"range": {"rating": {"gte": 4.0}}}
         ]
       }
     }
   }
   ```

   Redis Geo only supports proximity search (GEORADIUS), requiring application-level filtering (slow).

2. **Full-Text Search:** POI names need text search (e.g., "Golden Gate Pizza"). Elasticsearch's inverted index provides
   fast text search, while Redis has no text search capability.

3. **Horizontal Scaling:** Elasticsearch can shard by Geohash prefix for geographic distribution:
   ```
   Shard 1: Geohash prefix "9q8" (San Francisco)
   Shard 2: Geohash prefix "drm" (London)
   ```
   This ensures queries for a specific region hit only relevant shards.

4. **Proven at Scale:** Yelp uses Elasticsearch for POI search. They handle millions of searches per day with sub-100ms
   latency.

5. **Latency Acceptable:** 20-50ms latency is acceptable for search queries (vs. <5ms for Redis, but Redis can't handle
   complex filtering).

### Why NOT PostgreSQL PostGIS?

- **Slower queries:** 50-100ms latency (disk I/O) vs. 20-50ms for Elasticsearch
- **Write bottleneck:** 1K writes/sec max per node (would need 50+ nodes for 50K writes/sec)
- **Complex SQL:** Queries become complex with multiple filters:
  ```sql
  SELECT * FROM pois
  WHERE ST_DWithin(location, ST_MakePoint(lng, lat), 5000)
    AND category = 'restaurant'
    AND rating >= 4.0
    AND (hours->>'monday'->>'open') <= CURRENT_TIME
  ```
  Elasticsearch's query DSL is more readable and maintainable.

### Why NOT Redis Geo?

- **No filtering:** GEORADIUS only returns POIs within radius, no category/rating filters
- **No full-text search:** Can't search POI names
- **Memory constraints:** 50M POIs × 50 bytes = 2.5GB minimum (expensive)
- **Application-level filtering:** Would need to:
    1. Query Redis for proximity (fast)
    2. Filter results by category/rating in application (slow)
    3. This defeats the purpose of using Redis for speed

### Implementation Details

**Elasticsearch Cluster Setup:**

- 15 nodes (3 nodes × 5 shards)
- Shard by Geohash prefix (first 3 characters)
- Replication: 1 replica per shard (30 total shards)
- Index settings:
  ```
  number_of_shards: 5
  number_of_replicas: 1
  refresh_interval: 1s (near real-time)
  ```

**Query Pattern:**

```
1. Calculate Geohash for search center
2. Query Elasticsearch with geo_distance filter
3. Apply additional filters (category, rating, hours)
4. Sort by distance + rating
5. Return top 20 results
```

### Trade-offs Accepted

| What We Gain                                 | What We Sacrifice                                |
|----------------------------------------------|--------------------------------------------------|
| ✅ Complex querying (category, rating, hours) | ❌ Higher latency (20-50ms vs. <5ms for Redis)    |
| ✅ Full-text search on POI names              | ❌ More infrastructure (15 nodes vs. 3 for Redis) |
| ✅ Horizontal scaling (shard by Geohash)      | ❌ Operational complexity (cluster management)    |
| ✅ Proven at scale (Yelp uses it)             | ❌ Higher cost (disk + RAM vs. RAM only)          |

### When to Reconsider

**Consider Redis Geo when:**

- Only proximity search needed (no filtering)
- Latency must be <10ms (real-time applications)
- Memory cost acceptable (smaller dataset)

**Consider PostgreSQL PostGIS when:**

- ACID guarantees required (financial data)
- Single database preference (simpler architecture)
- Can accept 50-100ms latency

---

## 3. MongoDB vs PostgreSQL for Review Storage

### The Problem

We need to store billions of reviews (10TB over 5 years) with:

- **High write throughput:** 1M+ reviews per day
- **Simple access pattern:** Always query by POI ID (get all reviews for a POI)
- **Unstructured data:** Reviews have varying fields (text, photos, ratings)
- **Scalability:** Horizontal sharding by POI ID

### Options Considered

| Factor                     | MongoDB (Document Store)                  | PostgreSQL (Relational)                     |
|----------------------------|-------------------------------------------|---------------------------------------------|
| **Write Throughput**       | ✅ 50K writes/sec per shard                | ⚠️ 10K writes/sec per node                  |
| **Schema Flexibility**     | ✅ Dynamic schema (varying fields)         | ❌ Fixed schema (requires migrations)        |
| **Access Pattern**         | ✅ Perfect for "get all reviews by POI ID" | ⚠️ Requires JOIN or denormalization         |
| **Horizontal Sharding**    | ✅ Native sharding by POI ID               | ⚠️ Requires custom sharding logic           |
| **Data Model**             | ✅ Document model (reviews are documents)  | ❌ Relational model (reviews are rows)       |
| **Query Complexity**       | ✅ Simple find({"poi_id": "poi_123"})      | ⚠️ SELECT * FROM reviews WHERE poi_id = ... |
| **ACID Guarantees**        | ⚠️ Limited transactions (4.0+)            | ✅ Full ACID                                 |
| **Operational Complexity** | ⚠️ Requires sharded cluster management    | ⚠️ Requires connection pooling, vacuuming   |
| **Cost**                   | ⚠️ Moderate (32 shards for 1B reviews)    | ✅ Lower (fewer nodes needed)                |
| **Ecosystem**              | ✅ Mature (used by Yelp, Foursquare)       | ✅ Mature (industry standard)                |

### Decision Made

**Use MongoDB for review storage with sharding by POI ID.**

### Rationale

1. **Document Model Fit:** Reviews are naturally documents:
   ```json
   {
     "review_id": "rev_123",
     "poi_id": "poi_456",
     "user_id": "user_789",
     "rating": 5,
     "text": "Great pizza!",
     "photos": ["photo1.jpg", "photo2.jpg"],
     "created_at": "2024-01-15T10:00:00Z"
   }
   ```
   MongoDB's document model fits this perfectly, while PostgreSQL would require multiple tables (reviews, review_photos,
   etc.).

2. **Simple Access Pattern:** All queries are "get all reviews for POI X":
   ```javascript
   db.reviews.find({"poi_id": "poi_456"}).sort({"created_at": -1})
   ```
   This is a simple key-value lookup, perfect for MongoDB's sharding by `poi_id`.

3. **High Write Throughput:** 1M reviews/day = 12 writes/sec average, but bursts can be 1000+ writes/sec. MongoDB's 50K
   writes/sec per shard easily handles this.

4. **Schema Flexibility:** Reviews have varying fields (some have photos, some don't). MongoDB's dynamic schema handles
   this without migrations.

5. **Proven at Scale:** Yelp uses MongoDB for review storage. They handle billions of reviews with horizontal sharding.

### Why NOT PostgreSQL?

- **Schema rigidity:** Reviews have varying fields (photos, tags, etc.). PostgreSQL requires schema migrations for new
  fields.
- **JOIN overhead:** If reviews and photos are in separate tables, queries require JOINs (slower).
- **Sharding complexity:** PostgreSQL sharding requires custom logic (e.g., pg_shard, Citus). MongoDB has native
  sharding.

### Implementation Details

**MongoDB Sharding Setup:**

- 32 shards (3-node replica set per shard = 96 total nodes)
- Shard key: `poi_id` (ensures all reviews for a POI are on same shard)
- Replication: 3-node replica set per shard (high availability)

**Collection Structure:**

```javascript
db.reviews.createIndex({"poi_id": 1, "created_at": -1})
```

**Query Pattern:**

```
1. User requests reviews for POI "poi_123"
2. MongoDB routes to shard containing "poi_123"
3. Query shard: db.reviews.find({"poi_id": "poi_123"})
4. Return sorted by created_at (most recent first)
```

### Trade-offs Accepted

| What We Gain                                | What We Sacrifice                             |
|---------------------------------------------|-----------------------------------------------|
| ✅ Simple document model (natural fit)       | ❌ Less ACID guarantees (eventual consistency) |
| ✅ High write throughput (50K/sec per shard) | ❌ Operational complexity (96 nodes)           |
| ✅ Schema flexibility (varying fields)       | ❌ Higher cost (more nodes)                    |
| ✅ Native sharding (by POI ID)               | ❌ Less mature ecosystem for analytics         |

### When to Reconsider

**Consider PostgreSQL when:**

- ACID guarantees required (financial reviews)
- Complex queries needed (JOINs, aggregations)
- Lower operational complexity preferred

---

## 4. Geohash Prefix Sharding vs Hash-based Sharding

### The Problem

We need to shard 50 million POIs across multiple database nodes to handle:

- **Geographic locality:** Queries for a region should hit few shards
- **Load distribution:** Even distribution of data and queries
- **Hotspot management:** Handle densely populated areas (NYC, Tokyo) without overloading shards

### Options Considered

| Factor                  | Geohash Prefix Sharding                        | Hash-based Sharding (by POI ID)       |
|-------------------------|------------------------------------------------|---------------------------------------|
| **Geographic Locality** | ✅ All POIs in region on same shard             | ❌ POIs in same region scattered       |
| **Query Efficiency**    | ✅ Single shard query for region                | ❌ Multi-shard query (fanout)          |
| **Load Distribution**   | ⚠️ Uneven (NYC has more POIs than rural areas) | ✅ Even (hash distributes uniformly)   |
| **Hotspot Management**  | ❌ Dense areas overload shards                  | ✅ Even distribution prevents hotspots |
| **Sharding Logic**      | ✅ Simple (first 3 chars of Geohash)            | ✅ Simple (hash(poi_id) % num_shards)  |
| **Rebalancing**         | ⚠️ Complex (POIs move when region grows)       | ✅ Simple (consistent hashing)         |
| **Query Latency**       | ✅ Low (single shard)                           | ❌ Higher (fanout to all shards)       |

### Decision Made

**Use Geohash prefix sharding with hierarchical sub-sharding for hotspots.**

### Rationale

1. **Query Efficiency:** 90% of queries are for a specific region (e.g., "restaurants near me in San Francisco").
   Geohash prefix sharding ensures all POIs in San Francisco (prefix "9q8") are on the same shard, enabling single-shard
   queries (fast).

2. **Hotspot Mitigation:** Densely populated areas (NYC, Tokyo) are sub-sharded by category or rating:
   ```
   Shard "9q8" (San Francisco): 1M POIs → Too many!
   Solution: Sub-shard by category
   - Shard "9q8:restaurant": 400K POIs
   - Shard "9q8:hotel": 200K POIs
   - Shard "9q8:other": 400K POIs
   ```

3. **Locality Benefits:**
    - Cache locality: Popular regions cached together
    - Network efficiency: Fewer network hops
    - Operational simplicity: Can move entire regions to different data centers

4. **Proven at Scale:** Google Maps uses geographic sharding. They handle billions of POIs with regional shards.

### Why NOT Hash-based Sharding?

- **Query fanout:** A query for "restaurants in San Francisco" would fanout to all 64 shards (slow, 64× network
  overhead).
- **No locality:** POIs in the same region are scattered, making caching inefficient.
- **Complex queries:** Can't optimize for geographic queries (e.g., "POIs along this route").

### Implementation Details

**Sharding Strategy:**

```
Level 1: Geohash prefix (first 3 chars)
  - "9q8" → Shard 1 (San Francisco)
  - "drm" → Shard 2 (London)
  
Level 2: Sub-sharding for hotspots (if POI count > 500K)
  - "9q8" has 1M POIs → Sub-shard by category
  - "9q8:restaurant" → Shard 1a
  - "9q8:hotel" → Shard 1b
```

**Query Routing:**

```
1. Calculate Geohash prefix for search center
2. Route query to shard(s) with matching prefix
3. If sub-sharded, query relevant sub-shards only
```

### Trade-offs Accepted

| What We Gain                             | What We Sacrifice                     |
|------------------------------------------|---------------------------------------|
| ✅ Single-shard queries (fast)            | ❌ Uneven load distribution (hotspots) |
| ✅ Geographic locality (cache efficiency) | ❌ Complex rebalancing (POIs move)     |
| ✅ Operational simplicity (region-based)  | ❌ Sub-sharding needed for hotspots    |
| ✅ Network efficiency (fewer hops)        | ❌ Manual hotspot management           |

### When to Reconsider

**Consider hash-based sharding when:**

- Queries are not geographic (e.g., "top-rated restaurants globally")
- Even load distribution is critical (no hotspots acceptable)
- Simple rebalancing preferred

---

## 5. Denormalization vs Real-time Joins

### The Problem

Search queries need data from multiple sources:

- **Elasticsearch:** POI location, category, rating (from reviews)
- **PostgreSQL:** POI metadata (address, phone, hours)
- **MongoDB:** Review details (text, photos)

We need to decide: denormalize review data into Elasticsearch or join at query time?

### Options Considered

| Factor                     | Denormalization (Store in Elasticsearch) | Real-time Joins (Query Multiple DBs) |
|----------------------------|------------------------------------------|--------------------------------------|
| **Query Latency**          | ✅ Fast (single query to Elasticsearch)   | ❌ Slow (N+1 queries: ES + MongoDB)   |
| **Data Freshness**         | ⚠️ Stale (updated every 5 minutes)       | ✅ Real-time (always latest)          |
| **Storage Cost**           | ⚠️ Higher (duplicate data in ES)         | ✅ Lower (single source of truth)     |
| **Write Complexity**       | ⚠️ Complex (update ES when review added) | ✅ Simple (write to MongoDB only)     |
| **Consistency**            | ⚠️ Eventual consistency                  | ✅ Strong consistency                 |
| **Scalability**            | ✅ Single query (no fanout)               | ❌ Multi-DB queries (fanout)          |
| **Operational Complexity** | ⚠️ Requires sync pipeline                | ✅ Simple (direct queries)            |

### Decision Made

**Denormalize review aggregates (avg_rating, review_count) into Elasticsearch, keep full reviews in MongoDB.**

### Rationale

1. **Latency Critical:** Search queries must be <100ms. Real-time joins would require:
    - Query Elasticsearch for POIs (50ms)
    - Query MongoDB for review aggregates (30ms per POI)
    - Total: 50ms + (30ms × 20 POIs) = 650ms (too slow!)

2. **Review Aggregates Sufficient:** For search results, we only need:
    - Average rating (4.5 stars)
    - Review count (1,250 reviews)
    - Full review text not needed in search results

3. **Update Frequency Acceptable:** Review aggregates change slowly (new review every few minutes). Updating
   Elasticsearch every 5 minutes is acceptable (stale by 5 minutes is fine for search).

4. **Proven Pattern:** Yelp denormalizes review aggregates into Elasticsearch. They handle millions of searches with
   sub-100ms latency.

### Why NOT Real-time Joins?

- **Latency:** N+1 queries (1 to Elasticsearch + N to MongoDB) would increase latency to 500ms+ (unacceptable).
- **Scalability:** 20 POIs × 30ms MongoDB query = 600ms overhead (bottleneck).

### Implementation Details

**Denormalization Pipeline:**

```
1. Review added to MongoDB
2. Update review aggregates in PostgreSQL (avg_rating, review_count)
3. Every 5 minutes: Batch update Elasticsearch with aggregates
4. Elasticsearch query returns POIs with up-to-date aggregates
```

**Elasticsearch Document Structure:**

```json
{
  "poi_id": "poi_123",
  "name": "Golden Gate Pizza",
  "location": {
    ...
  },
  "avg_rating": 4.5,
  // Denormalized from MongoDB
  "review_count": 1250,
  // Denormalized from MongoDB
  "category": "restaurant"
}
```

**Query Flow:**

```
1. User searches: "restaurants near me, rating > 4.0"
2. Elasticsearch query (single query, <100ms)
3. Returns POIs with avg_rating already included
4. No MongoDB query needed for search results
```

### Trade-offs Accepted

| What We Gain                    | What We Sacrifice                 |
|---------------------------------|-----------------------------------|
| ✅ Fast queries (<100ms)         | ❌ Data stale by 5 minutes         |
| ✅ Single query (no fanout)      | ❌ Higher storage (duplicate data) |
| ✅ Scalable (no N+1 queries)     | ⚠️ Complex sync pipeline          |
| ✅ Proven pattern (Yelp uses it) | ⚠️ Eventual consistency           |

### When to Reconsider

**Consider real-time joins when:**

- Data freshness critical (real-time requirements)
- Storage cost is a concern (large dataset)
- Can accept higher latency (500ms+ acceptable)

---

## 6. Caching Strategy: Redis vs Elasticsearch Cache

### The Problem

We need to cache search results to reduce load on Elasticsearch and improve latency. Should we use Redis cache or rely
on Elasticsearch's internal cache?

### Options Considered

| Factor                     | Redis Cache (Application-level)      | Elasticsearch Cache (Internal)          |
|----------------------------|--------------------------------------|-----------------------------------------|
| **Cache Control**          | ✅ Full control (TTL, invalidation)   | ⚠️ Limited control (OS cache)           |
| **Cache Hit Rate**         | ✅ 80% (popular queries cached)       | ⚠️ 50% (OS-level, less predictable)     |
| **Latency**                | ✅ <5ms (in-memory)                   | ⚠️ 10-20ms (OS cache)                   |
| **Cache Granularity**      | ✅ Query-level (cache full results)   | ⚠️ Segment-level (cache index segments) |
| **Operational Complexity** | ⚠️ Requires Redis cluster            | ✅ No additional infrastructure          |
| **Cost**                   | ⚠️ Additional infrastructure (Redis) | ✅ No additional cost                    |
| **Invalidation**           | ✅ Explicit (cache keys, TTL)         | ⚠️ Implicit (OS eviction)               |

### Decision Made

**Use Redis cache for popular queries (80% cache hit rate target).**

### Rationale

1. **Query-level Caching:** Popular queries (e.g., "restaurants in NYC, rating > 4.0") are cached as complete results in
   Redis. This provides:
    - <5ms latency (vs. 50ms for Elasticsearch)
    - 80% cache hit rate (reduces Elasticsearch load by 80%)

2. **Cache Control:** We can explicitly invalidate cache when:
    - New POI added (invalidate region cache)
    - Review added (invalidate POI cache)
    - POI updated (invalidate POI cache)

3. **Cost-Benefit:** Redis cluster costs $5K/month but saves $20K/month in Elasticsearch infrastructure (80% reduction
   in queries).

4. **Proven Pattern:** Yelp uses Redis cache for popular queries. They achieve 80% cache hit rate.

### Why NOT Elasticsearch Cache Only?

- **Unpredictable:** OS-level cache is less predictable (cache hit rate varies).
- **Limited control:** Can't explicitly invalidate cache (must wait for TTL).
- **Granularity:** OS cache is segment-level, not query-level (less efficient).

### Implementation Details

**Cache Key Strategy:**

```
cache_key = f"search:{geohash_prefix}:{category}:{rating_min}:{limit}"
Example: "search:9q8:restaurant:4.0:20"
```

**Cache TTL:**

- Popular queries: 5 minutes
- Rare queries: 1 minute
- POI updates: Invalidate immediately

**Cache Invalidation:**

```
1. New POI added → Invalidate region cache: "search:9q8:*"
2. Review added → Invalidate POI cache: "poi:poi_123"
3. POI updated → Invalidate POI + region cache
```

### Trade-offs Accepted

| What We Gain                     | What We Sacrifice                         |
|----------------------------------|-------------------------------------------|
| ✅ Fast queries (<5ms for cached) | ❌ Additional infrastructure (Redis)       |
| ✅ High cache hit rate (80%)      | ❌ Cache invalidation complexity           |
| ✅ Query-level control            | ❌ Storage cost (cache popular queries)    |
| ✅ Proven pattern (Yelp uses it)  | ⚠️ Operational complexity (Redis cluster) |

### When to Reconsider

**Consider Elasticsearch cache only when:**

- Cost is critical (no budget for Redis)
- Cache hit rate acceptable (50% is fine)
- Operational simplicity preferred

---

## 7. Adjacent Cell Querying vs Single Cell Query

### The Problem

When searching for POIs within a radius, Geohash cell boundaries can cause missed POIs. Should we query only the center
cell or also query adjacent cells?

### Options Considered

| Factor                 | Single Cell Query                     | Adjacent Cell Query (9 cells)            |
|------------------------|---------------------------------------|------------------------------------------|
| **Query Latency**      | ✅ Fast (single cell, 50ms)            | ⚠️ Slower (9 cells, 80ms)                |
| **Accuracy**           | ❌ Misses 10-15% of POIs at boundaries | ✅ No missed POIs (100% accuracy)         |
| **Query Complexity**   | ✅ Simple (single Geohash)             | ⚠️ Complex (center + 8 neighbors)        |
| **Elasticsearch Load** | ✅ Low (1 query per search)            | ⚠️ Higher (9 queries or 1 query with OR) |
| **False Positives**    | ✅ None                                | ⚠️ Some (POIs outside radius)            |

### Decision Made

**Query center cell + 8 adjacent cells (3×3 grid) for boundary accuracy.**

### Rationale

1. **Accuracy Critical:** Missing 10-15% of POIs is unacceptable for a search system. Users expect to see all POIs
   within radius.

2. **Latency Acceptable:** 80ms for 9-cell query is still within <100ms target (acceptable trade-off for accuracy).

3. **Filtering Handles False Positives:** Elasticsearch's `geo_distance` filter removes POIs outside radius:
   ```json
   {
     "query": {
       "bool": {
         "must": [
           {"terms": {"geohash": ["9q8yy9m", "9q8yy9j", ...]}},  // 9 cells
           {"geo_distance": {"distance": "5km", "location": {...}}}  // Filter by exact distance
         ]
       }
     }
   }
   ```

4. **Proven Pattern:** Google Maps queries adjacent cells for boundary accuracy. They achieve 100% accuracy for radius
   queries.

### Why NOT Single Cell Query?

- **Accuracy:** Missing 10-15% of POIs is unacceptable (users complain about "missing results").
- **User Experience:** Users expect to see all POIs within radius (not 85% of them).

### Implementation Details

**Adjacent Cell Calculation:**

```
1. Calculate center Geohash: geohash_center = encode_geohash(lat, lng, 7)
2. Calculate 8 neighbors: neighbors = get_adjacent_geohashes(geohash_center)
3. Query Elasticsearch: geohash IN [geohash_center, ...neighbors]
4. Filter by exact distance: geo_distance filter removes false positives
```

*See [pseudocode.md::get_adjacent_geohashes()](pseudocode.md) for implementation.*

### Trade-offs Accepted

| What We Gain                      | What We Sacrifice                     |
|-----------------------------------|---------------------------------------|
| ✅ 100% accuracy (no missed POIs)  | ❌ Higher latency (80ms vs. 50ms)      |
| ✅ User satisfaction (all results) | ❌ More complex queries (9 cells)      |
| ✅ Proven pattern (Google Maps)    | ⚠️ Slightly higher Elasticsearch load |

### When to Reconsider

**Consider single cell query when:**

- Latency must be <50ms (real-time applications)
- 10-15% miss rate acceptable
- Simpler queries preferred

---

## 8. Synchronous vs Asynchronous POI Updates

### The Problem

When a POI is updated (new hours, address change), should we update Elasticsearch synchronously (blocking) or
asynchronously (non-blocking)?

### Options Considered

| Factor                     | Synchronous Updates                    | Asynchronous Updates (Kafka)           |
|----------------------------|----------------------------------------|----------------------------------------|
| **Data Consistency**       | ✅ Strong (immediate consistency)       | ⚠️ Eventual (updated within seconds)   |
| **Write Latency**          | ⚠️ Higher (waits for ES update, 50ms)  | ✅ Lower (returns immediately, 5ms)     |
| **Write Throughput**       | ⚠️ Limited by ES write speed (10K/sec) | ✅ High (Kafka handles 100K/sec)        |
| **Failure Handling**       | ❌ User waits for ES failure            | ✅ Kafka buffers, retries automatically |
| **Operational Complexity** | ✅ Simple (direct update)               | ⚠️ Complex (Kafka + consumer)          |
| **Backpressure**           | ❌ ES overload blocks writes            | ✅ Kafka buffers, handles backpressure  |

### Decision Made

**Use asynchronous updates via Kafka for POI updates (non-critical path).**

### Rationale

1. **Non-Critical Path:** POI updates are not time-sensitive (users can wait a few seconds for changes to appear in
   search). This is different from search queries (must be <100ms).

2. **Write Throughput:** POI updates can burst (1K updates/sec during business hours). Kafka buffers these bursts,
   preventing Elasticsearch overload.

3. **Failure Resilience:** If Elasticsearch is down, Kafka buffers updates and retries automatically. Users don't see
   errors.

4. **Proven Pattern:** Yelp uses Kafka for POI updates. They handle bursts without impacting search performance.

### Why NOT Synchronous Updates?

- **Latency:** 50ms write latency is unacceptable for API responses (users expect <10ms).
- **Backpressure:** If Elasticsearch is overloaded, synchronous writes block all POI updates (bad user experience).
- **Failure Handling:** ES failures cause user-facing errors (poor UX).

### Implementation Details

**Update Flow:**

```
1. User updates POI via API
2. Update PostgreSQL (synchronous, 10ms)
3. Publish event to Kafka: {"poi_id": "poi_123", "action": "update"}
4. Return success to user (5ms total)
5. Kafka consumer updates Elasticsearch (async, within 5 seconds)
```

**Kafka Topic:**

```
Topic: poi-updates
Partition: hash(poi_id) % 10 (ensures order per POI)
Replication: 3 (high availability)
```

**Consumer:**

```
1. Consume event from Kafka
2. Fetch POI data from PostgreSQL
3. Update Elasticsearch document
4. Commit offset (at-least-once delivery)
```

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice                      |
|--------------------------------------|----------------------------------------|
| ✅ Fast API responses (<10ms)         | ❌ Eventual consistency (5 seconds)     |
| ✅ High write throughput (100K/sec)   | ⚠️ Complex pipeline (Kafka + consumer) |
| ✅ Failure resilience (Kafka buffers) | ⚠️ Operational complexity              |
| ✅ Proven pattern (Yelp uses it)      | ⚠️ Debugging harder (async flow)       |

### When to Reconsider

**Consider synchronous updates when:**

- Data consistency critical (real-time requirements)
- Write throughput low (<1K/sec)
- Operational simplicity preferred

---

## Summary Table

| Decision              | Chosen          | Alternative         | Key Trade-off                       |
|-----------------------|-----------------|---------------------|-------------------------------------|
| **Geo-Indexing**      | Geohash         | H3 Hexagonal        | Simplicity vs. Boundary Accuracy    |
| **Search Engine**     | Elasticsearch   | PostgreSQL PostGIS  | Complex Queries vs. ACID Guarantees |
| **Review Storage**    | MongoDB         | PostgreSQL          | Document Model vs. ACID             |
| **Sharding**          | Geohash Prefix  | Hash-based          | Locality vs. Even Distribution      |
| **Data Model**        | Denormalization | Real-time Joins     | Latency vs. Data Freshness          |
| **Caching**           | Redis           | Elasticsearch Cache | Control vs. Simplicity              |
| **Boundary Handling** | Adjacent Cells  | Single Cell         | Accuracy vs. Latency                |
| **Updates**           | Asynchronous    | Synchronous         | Throughput vs. Consistency          |

---

## Conclusion

The Yelp/Google Maps system prioritizes **query performance** and **scalability** over **strong consistency** and *
*operational simplicity**. Key decisions (Elasticsearch, Geohash, denormalization, async updates) are all optimized for
fast search queries (<100ms) at scale (50M POIs, 11K QPS).

These trade-offs are acceptable because:

- Search is the critical path (users expect fast results)
- Data freshness can be eventual (5 seconds is acceptable)
- Operational complexity is manageable with proper tooling