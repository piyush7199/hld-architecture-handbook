# Yelp/Google Maps (Geo-Spatial Search) - Quick Overview

## Core Concept

A geo-spatial search system that enables users to find businesses, restaurants, and points of interest (POIs) within a specified radius of their location. It uses **Geohash encoding** to convert 2D geographic coordinates (lat/lng) into a searchable 1D index, combined with **Elasticsearch** for complex filtering and ranking.

**Key Challenge**: Query 50M POIs globally with <100ms latency, handle complex filtering (category, rating, hours), and manage hotspotting in densely populated areas.

---

## Requirements

### Functional
- **Proximity Search**: Find all POIs within N miles/km of a given location (lat/lng)
- **Point Lookup**: Retrieve detailed information for a specific POI by ID
- **Complex Filtering**: Support filters on category, rating, price range, hours, custom attributes
- **Geo-Indexing**: Store and index millions of POIs with latitude/longitude coordinates
- **Review Management**: Store and serve billions of user reviews with ratings
- **Ranking**: Sort results by relevance, distance, rating, or popularity
- **Auto-complete**: Suggest POI names as users type search queries
- **Map Rendering**: Serve map tiles and POI markers for visualization

### Non-Functional
- **Low Latency**: Search queries must return in <100ms (p95)
- **High Availability**: 99.9% uptime (map services are critical for navigation)
- **Scalability**: Handle 11,574 QPS peak read load
- **Data Durability**: Store POI and review data permanently with backups
- **Geographic Distribution**: Support global deployment with regional data centers
- **Consistency**: POI updates should be eventually consistent (acceptable)

---

## Components

### 1. **Geo-Search Service (Elasticsearch)**
- **Geo-Point Index**: Native `geo_point` field type with optimized indexing
- **Geohash Field**: String field for prefix matching
- **Complex Queries**: Combine geo filters with category, rating, hours filters
- **Inverted Index**: Fast text search on POI names, descriptions
- **Sharding**: Shard by Geohash prefix (3 characters) for geographic distribution

### 2. **POI Database (PostgreSQL)**
- **ACID Guarantees**: Ensure POI updates are consistent
- **Rich Schema**: Support complex relationships (POI → categories, hours, attributes)
- **Full-Text Search**: Built-in text search for POI names
- **Sharding**: Shard by Geohash prefix (64 shards)

### 3. **Review Storage (MongoDB)**
- **Document Store**: Flexible schema for reviews (text, rating, photos, helpful votes)
- **High Write Throughput**: 100k+ writes/sec per node
- **Horizontal Scaling**: Shard by POI ID hash
- **Aggregation Pipeline**: Calculate average rating, review count

### 4. **Cache Layer (Redis)**
- **POI Cache**: Hash (`poi:{poi_id}`) - POI metadata (1 hour TTL)
- **Search Results Cache**: Sorted Set (`search:{geohash}:{category}`) - Top 100 results (5 minutes TTL)
- **Ranked Results Cache**: Pre-computed top results for popular regions

### 5. **CDN (Cloudflare)**
- **Map Tiles**: Serve map tiles from edge (200+ PoPs)
- **Static Assets**: POI images, icons
- **Cache Headers**: `Cache-Control: public, max-age=604800, immutable`

### 6. **Search Service API**
- **REST API**: `/search?lat=37.7749&lng=-122.4194&radius=5km&category=restaurant`
- **Query Processing**: Geohash encoding, Elasticsearch query, ranking
- **Response**: Top 50 POIs with metadata (name, rating, distance, etc.)

---

## Architecture Flow

### Proximity Search Flow

```
1. Client → Search Service: GET /search?lat=37.7749&lng=-122.4194&radius=5km&category=restaurant
2. Search Service → Geohash: Encode lat/lng to 7-char Geohash (9q8yy9m)
3. Search Service → Geohash: Get 8 adjacent cells (3×3 grid)
4. Search Service → Elasticsearch: Query POIs in target Geohash cells
   - Filter: geohash prefix matches target cells
   - Filter: geo_distance within 5km
   - Filter: category = restaurant
5. Elasticsearch → Search Service: Return matching POIs
6. Search Service → Ranking: Calculate score (distance + rating + popularity)
7. Search Service → PostgreSQL: Fetch POI metadata (if not in cache)
8. Search Service → MongoDB: Fetch recent reviews (if needed)
9. Search Service → Client: Return top 50 POIs
```

### POI Lookup Flow

```
1. Client → Search Service: GET /pois/{poi_id}
2. Search Service → Redis: HGETALL poi:{poi_id} (cache check)
3. If cache miss:
   a. Search Service → PostgreSQL: SELECT * FROM pois WHERE poi_id = {poi_id}
   b. Search Service → Redis: HSET poi:{poi_id} {metadata} EX 3600
4. Search Service → MongoDB: Fetch reviews (last 10 reviews)
5. Search Service → Client: Return POI details + reviews
```

---

## Key Design Decisions

### 1. **Geohash Encoding** (vs H3/PostGIS R-Tree)

**Why Geohash?**
- **Simplicity**: String prefix matching (easy to implement)
- **Elasticsearch Support**: Native `geo_point` field type
- **Sharding**: Easy prefix-based sharding (first 3 chars)
- **Widely Adopted**: Industry standard

**Trade-off**: Edge cases at cell boundaries vs perfect hexagonal tiling (H3)

**Geohash Precision**:
- **3 chars**: ~156 km × 156 km (city-level)
- **5 chars**: ~4.9 km × 4.9 km (neighborhood)
- **7 chars**: ~153 m × 153 m (POI-level) ← **Chosen for POI search**
- **9 chars**: ~4.8 m × 4.8 m (building-level)

### 2. **Elasticsearch for Geo-Search** (vs PostgreSQL PostGIS)

**Why Elasticsearch?**
- **Native Geo Support**: Built-in `geo_point` field type with optimized indexing
- **Complex Queries**: Combine geo filters with category, rating, hours filters
- **Inverted Index**: Fast text search on POI names, descriptions
- **Horizontal Scaling**: Shard by Geohash prefix for geographic distribution
- **Real-time Updates**: Near real-time index updates (<1 second)

**Trade-off**: Eventual consistency vs strong consistency (PostgreSQL)

### 3. **PostgreSQL for POI Metadata** (vs Cassandra/MongoDB)

**Why PostgreSQL?**
- **ACID Guarantees**: Ensure POI updates are consistent
- **Rich Schema**: Support complex relationships (POI → categories, hours, attributes)
- **Full-Text Search**: Built-in text search for POI names
- **Mature Ecosystem**: Widely used, well-documented

**Trade-off**: Harder to scale (need sharding) vs horizontal scaling (Cassandra)

### 4. **MongoDB for Reviews** (vs PostgreSQL/Cassandra)

**Why MongoDB?**
- **Document Store**: Flexible schema for reviews (text, rating, photos, helpful votes)
- **High Write Throughput**: 100k+ writes/sec per node
- **Horizontal Scaling**: Shard by POI ID hash
- **Aggregation Pipeline**: Calculate average rating, review count

**Trade-off**: Eventual consistency vs strong consistency

### 5. **Multi-Layer Caching**

**L1: POI Cache** (Redis Hash): POI metadata (1 hour TTL, 70% hit rate)
**L2: Search Results Cache** (Redis Sorted Set): Top 100 results (5 minutes TTL, 60% hit rate)
**L3: Ranked Results Cache** (Redis): Pre-computed top results for popular regions (5 minutes TTL, 80% hit rate)

**Trade-off**: Memory cost vs query latency

### 6. **Cross-Boundary Query Handling**

**Problem**: Search at edge of two Geohash cells may miss POIs

**Solution**: Query center cell + 8 adjacent cells (3×3 grid)

**Algorithm**:
1. Calculate center Geohash: `geohash_center`
2. Calculate 8 neighbors: `geohash_neighbors = get_neighbors(geohash_center)`
3. Query Elasticsearch: `geohash IN [geohash_center, ...geohash_neighbors]`
4. Filter by exact distance: Keep only POIs within radius R

**Trade-off**: 9x more cells to query vs missing POIs at boundaries

---

## Geo-Spatial Query Processing

### Proximity Search Algorithm

**Challenge**: Find all POIs within radius R of point (lat, lng)

**Solution: Geohash Range Query**

1. **Calculate Geohash Precision**: Based on radius R
   - R < 1km → 7-char Geohash (153m × 153m)
   - R < 5km → 6-char Geohash (610m × 610m)
   - R < 20km → 5-char Geohash (4.9km × 4.9km)

2. **Generate Geohash for Center Point**: `encode_geohash(lat, lng, precision)`

3. **Find Adjacent Cells**: Query 8 neighbors + center cell (3×3 grid)

4. **Query Elasticsearch**: Search POIs in target Geohash cells
   - Filter: `geohash` prefix matches target cells
   - Distance filter: `geo_distance` within radius R

5. **Post-Process**: Filter by exact distance (Geohash is approximate)

### Ranking Algorithm

**Multi-factor ranking score**:

```
score = w1 × distance_score + w2 × rating_score + w3 × popularity_score

where:
- distance_score = 1 / (1 + distance_km)
- rating_score = rating / 5.0
- popularity_score = log(review_count + 1) / log(max_reviews)
- weights: w1=0.4, w2=0.4, w3=0.2
```

**Why This Ranking?**
- **Distance**: Users prefer nearby POIs
- **Rating**: Quality matters
- **Popularity**: Social proof (review count)

---

## Bottlenecks & Solutions

### 1. **Hotspotting in Densely Populated Areas**

**Problem**: NYC and Tokyo have 10x more POIs than rural areas, overloading specific Geohash shards

**Solution**: Hierarchical Sharding
- **Primary Shard**: By Geohash prefix (3 chars)
- **Sub-shard**: By category or rating for high-traffic prefixes

**Example**:
```
Before: All NYC POIs in shard "dr5"
After:
  - dr5:restaurant → Shard 1A
  - dr5:hotel → Shard 1B
  - dr5:attraction → Shard 1C
```

**Result**: Distributes load across more machines, reduces query latency

### 2. **Cross-Boundary Query Latency**

**Problem**: Search at edge of two Geohash cells requires querying both cells, doubling query time

**Solution**:
1. **Optimize Geohash Precision**: Use appropriate precision based on radius
2. **Parallel Query Execution**: Query all 9 cells (center + 8 neighbors) in parallel
3. **Cache Adjacent Cells**: Pre-fetch adjacent cell data for popular regions

### 3. **Multi-DB Join Latency**

**Problem**: Final result requires joining Elasticsearch (POI metadata) with MongoDB (reviews)

**Solution**: Denormalization
- Store essential review data directly in Elasticsearch document:
  - `avg_rating`, `review_count`, `recent_reviews` (top 3 reviews)
- **Update Strategy**: Real-time update Elasticsearch when review added (async via Kafka)
- **Batch**: Re-calculate avg_rating every hour (background job)

**Trade-off**: Storage duplication (10-20% increase) vs query latency

### 4. **Ranking Calculation Cost**

**Problem**: CPU-heavy ranking calculations slow down search queries

**Solution**: Pre-computed Rankings
1. **Background Job**: Calculate top 100 POIs for popular regions every 5 minutes
2. **Cache Results**: Store in Redis with TTL=5 minutes
3. **Query Path**: Check cache first, fall back to real-time ranking if cache miss

**Cache Hit Rate**: 80-90% (most searches are for popular regions)

---

## Common Anti-Patterns

### ❌ **1. Scanning Entire Database**

**Problem**: Scans all 50M POIs (slow: 10+ seconds), no index usage

**Bad**:
```sql
SELECT * FROM pois
WHERE ABS(latitude - 37.7749) < 0.05 AND
      ABS(longitude - -122.4194) < 0.05;
```

**Good**: Use Geohash index (only queries relevant cells, fast: <100ms)

### ❌ **2. Ignoring Adjacent Cells**

**Problem**: Only query center cell, misses POIs at boundaries

**Bad**:
```python
geohash = encode_geohash(lat, lng, 7)
results = search_pois(geohash)  # Misses POIs at boundaries
```

**Good**: Query center + 8 neighbors (3×3 grid)

### ❌ **3. No Caching**

**Problem**: Every search queries Elasticsearch (slow, expensive)

**Bad**:
```
GET /search → Query Elasticsearch → 200ms latency
```

**Good**: Multi-layer caching (Redis POI cache, search results cache, ranked results cache)

---

## Monitoring & Observability

### Key Metrics

- **Search Query Latency P95**: <100ms
- **POI Lookup Latency**: <50ms
- **Cache Hit Rates**: POI 70%, Search Results 60%, Ranked Results 80%
- **Elasticsearch Query Time**: <50ms (p95)
- **Database Replication Lag**: <100ms
- **CDN Hit Rate**: >90%

### Alerts

- **Search Query Latency P95** >200ms
- **Cache Hit Rate** <50% (POI cache)
- **Elasticsearch Query Time** >100ms
- **Database Replication Lag** >500ms
- **CDN Hit Rate** <80%

---

## Trade-offs Summary

| Decision | What We Gain | What We Sacrifice |
|----------|--------------|-------------------|
| **Geohash Encoding** | ✅ Simple, Elasticsearch native | ❌ Edge cases at boundaries |
| **Elasticsearch** | ✅ Complex queries, fast text search | ❌ Eventual consistency |
| **PostgreSQL POI Metadata** | ✅ ACID guarantees | ❌ Harder to scale (sharding) |
| **MongoDB Reviews** | ✅ Flexible schema, high write throughput | ❌ Eventual consistency |
| **Multi-Layer Caching** | ✅ Fast queries (<100ms) | ❌ Memory cost |
| **Cross-Boundary Queries** | ✅ No missed POIs | ❌ 9x more cells to query |
| **Denormalization** | ✅ Fast queries (no joins) | ❌ Storage duplication (10-20%) |

---

## Real-World Examples

### Google Maps
- **Architecture**: S2 Geometry Library (spherical geometry), Spanner (global distributed database), custom distributed search
- **Innovation**: S2 cells for precise geographic indexing, pre-computed map tiles
- **Scale**: 200M+ POIs globally, 1B+ searches per day, sub-100ms latency

### Yelp
- **Architecture**: Geohash, Elasticsearch, PostgreSQL (POI metadata), MongoDB (reviews), Redis caching
- **Innovation**: Hybrid search (Elasticsearch + PostgreSQL), aggressive caching
- **Scale**: 50M+ POIs, 100M+ reviews, 10K+ QPS peak

### Foursquare
- **Architecture**: H3 (hexagonal grid), PostgreSQL with PostGIS, Redis
- **Innovation**: H3 for hexagonal tiling (no boundary issues)
- **Scale**: 100M+ POIs, 10M+ check-ins/day

---

## Key Takeaways

1. **Geohash Encoding**: Convert 2D coordinates to 1D string index (7 chars for POI-level precision)
2. **Cross-Boundary Queries**: Query center + 8 neighbors (3×3 grid) to avoid missing POIs
3. **Elasticsearch for Geo-Search**: Native `geo_point` support, complex filtering, horizontal scaling
4. **Multi-Layer Caching**: POI cache (70% hit), search results cache (60% hit), ranked results cache (80% hit)
5. **Denormalization**: Store review data in Elasticsearch (avoid joins, 10-20% storage increase)
6. **Hierarchical Sharding**: Shard by Geohash prefix, sub-shard by category for hotspots
7. **Pre-computed Rankings**: Background job calculates top 100 POIs for popular regions (5-minute refresh)
8. **CDN Strategy**: Aggressive caching (7 days, immutable) for 90% hit rate

---

## Recommended Stack

- **Geo-Search**: Elasticsearch (geo_point, Geohash indexing)
- **POI Database**: PostgreSQL (sharded by Geohash prefix)
- **Review Storage**: MongoDB (sharded by POI ID)
- **Cache**: Redis Cluster (POI cache, search results cache)
- **CDN**: Cloudflare (map tiles, static assets)
- **Search Service**: Go/Java REST API
- **Monitoring**: Prometheus, Grafana, ELK Stack

