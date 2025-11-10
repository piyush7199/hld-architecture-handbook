# Uber Ride Matching - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A real-time ride-matching system like Uber or Lyft that connects riders with nearby drivers within seconds. The system handles millions of concurrent drivers continuously updating their GPS coordinates, supports sub-100ms proximity searches to find the nearest available drivers, provides accurate ETAs, handles trip lifecycle management (request → match → pickup → trip → payment), and ensures high availability despite network failures and connection drops.

**Core Challenge:** Efficiently index and query millions of moving objects (drivers) in two-dimensional space (latitude/longitude) with sub-second latency while handling 1M+ location updates per second.

**Key Characteristics:**
- **Real-time location updates** - Drivers report GPS coordinates every 4 seconds
- **Sub-100ms proximity search** - Find nearest available drivers within radius
- **Geo-spatial indexing** - Efficiently query millions of moving objects in 2D space
- **High write throughput** - 750K location updates/sec (peak)
- **Geographic sharding** - Scale horizontally by city/region
- **Trip lifecycle management** - Request → match → pickup → active → completed

---

## Requirements & Scale

### Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **Driver Location Updates** | Drivers report GPS coordinates every 4 seconds (heartbeat) | Must Have |
| **Proximity Search** | Find N nearest available drivers within radius for a rider | Must Have |
| **Real-Time ETA** | Calculate estimated time of arrival for matched drivers | Must Have |
| **Trip Matching** | Match rider with optimal driver (distance, rating, vehicle type) | Must Have |
| **Trip State Management** | Track trip lifecycle: requested → matched → pickup → active → completed | Must Have |
| **Driver Status** | Track driver availability: online, offline, on-trip, break | Must Have |
| **Surge Pricing** | Dynamic pricing based on supply/demand in geohash cells | Should Have |
| **Driver Routing** | Navigate driver to pickup location, then to destination | Should Have |
| **Multi-Ride Types** | Support UberX, UberXL, Uber Black (different vehicle types) | Nice to Have |

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Low Latency (Search)** | < 100 ms (p99) | Fast matching prevents rider abandonment |
| **High Availability** | 99.99% uptime | Service downtime = revenue loss |
| **Write Throughput** | 1M updates/sec | 1M drivers × 1 update/sec |
| **Read Throughput** | 50K searches/sec | Peak ride requests |
| **Geo-Query Accuracy** | 99.9% | Missed drivers = poor UX |
| **Scalability** | Global, multi-city | Horizontal scaling required |

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|-----------|-------------|--------|
| **Total Drivers (Global)** | 1 Million active drivers | - | 1M drivers |
| **Driver Location Updates** | 1 update per 4 seconds per driver | 1M / 4 = 250K writes/sec | 250K writes/sec (avg) |
| **Peak Updates** | 3× average during rush hour | 250K × 3 | 750K writes/sec (peak) |
| **Concurrent Riders** | 50,000 active riders searching | - | 50K riders |
| **Proximity Searches** | 1 search per rider request | 50K searches/sec | 50K reads/sec |
| **Total Throughput** | Writes + Reads | 750K + 50K | ~800K ops/sec (peak) |
| **Driver Data per Record** | lat, lng, status, timestamp, driver_id | 40 bytes/record | 40 bytes |
| **Total Memory (In-Memory Index)** | 1M drivers × 40 bytes | 1M × 40 | ~40 MB (fits in RAM) |

**Key Insights:**
- **Write-heavy workload:** 750K writes/sec vs 50K reads/sec (15:1 ratio)
- **In-memory feasible:** 40 MB index fits entirely in RAM (Redis)
- **Low latency critical:** Rider abandonment increases exponentially after 3 seconds
- **Global sharding needed:** Single datacenter can't serve all cities

---

## Key Components

| Component | Responsibility | Technology | Scalability |
|-----------|---------------|------------|-------------|
| **API Gateway** | Route requests to nearest datacenter | NGINX, Kong | Horizontal + GeoDNS |
| **Location Ingestion** | Validate GPS, publish to Kafka | Go, Java | Horizontal (stateless) |
| **Kafka Cluster** | Buffer 750K writes/sec, decouple producers/consumers | Apache Kafka | Horizontal (partitioned) |
| **Indexer Workers** | Consume from Kafka, update Redis geo-index | Go workers | Horizontal (100+ consumers) |
| **Redis Geo Cluster** | In-memory geohash index for proximity search | Redis with Geo commands | Horizontal (sharded by city) |
| **Driver State Store** | Store driver status, rating, vehicle type | Redis | Horizontal (sharded) |
| **Matching Service** | Find nearest drivers, calculate ETA | Go, Rust | Horizontal (stateless) |
| **Trip Management** | ACID transactions for trip lifecycle | PostgreSQL (sharded) | Horizontal (sharded by city) |
| **ETA Service** | Calculate travel time using road network | GraphHopper, OSRM | Horizontal |

---

## Geo-Spatial Indexing Strategy

**Challenge:** Transform 2D space (latitude, longitude) into a searchable data structure.

**Solution: Geohash Encoding**

Geohash converts a lat/lng pair into a short alphanumeric string. Nearby locations share common prefixes, enabling:
- Single-dimensional index lookups (fast)
- Proximity search by prefix matching
- Efficient range queries

**Geohash Example:**
```
San Francisco: lat=37.7749, lng=-122.4194 → geohash = "9q8yy"

Precision levels:
- 3 chars: 9q8 → ~156 km × 156 km
- 5 chars: 9q8yy → ~4.9 km × 4.9 km
- 7 chars: 9q8yywe → ~153 m × 153 m
```

**Why Geohash over H3?**
- ✅ **Simplicity:** String prefix matching (vs complex hexagonal math)
- ✅ **Redis Support:** Native GEOADD, GEORADIUS commands
- ⚠️ **Boundary Issues:** Edge cases at cell boundaries (H3 solves this)
- ✅ **Adoption:** Widely supported (vs Uber-specific H3)

**Decision:** Use **Geohash** for MVP, migrate to **H3** for production at Uber scale.

---

## Redis Geo Commands

Redis provides native geospatial commands that internally use Geohash.

**Key Operations:**

1. **Add Driver Location:**
   ```
   GEOADD drivers:sf -122.4194 37.7749 "driver:123"
   ```
   - Adds driver to index with coordinates
   - Internally converts to geohash and stores in sorted set
   - Time complexity: O(log N)

2. **Find Nearby Drivers:**
   ```
   GEORADIUS drivers:sf -122.4194 37.7749 5 km WITHDIST COUNT 10
   ```
   - Returns 10 nearest drivers within 5 km radius
   - Sorted by distance
   - Time complexity: O(N log N) where N = drivers in search area

3. **Update Driver Location:**
   ```
   GEOADD drivers:sf -122.4200 37.7755 "driver:123"
   ```
   - Overwrites previous location (upsert behavior)
   - No need to delete old entry

**Data Structure:**
```
Key: drivers:{city_code}
Type: Sorted Set (internally)
Members: driver:{driver_id}
Score: Geohash integer (52-bit)
```

---

## Location Ingestion Pipeline

**Why Kafka?**

**Problem:** 750K location updates/sec overwhelm any synchronous database write path.

**Solution:** Kafka acts as a buffer, absorbing bursts and decoupling producers (drivers) from consumers (indexers).

**Flow:**
1. **Driver App:** POST /location → API Gateway
2. **Location Service:** Validate coordinates, publish to Kafka topic "location-updates"
3. **Kafka:** Persist message, replicate to 3 brokers
4. **Indexer Workers:** 100 consumers pull from Kafka, update Redis
5. **Redis:** GEOADD command updates driver position

**Why This Over Synchronous Writes?**

| Approach | Latency (Driver POV) | Database Load | Fault Tolerance |
|----------|---------------------|---------------|-----------------|
| **Synchronous (Direct Redis)** | 50-100ms (waits for DB) | 750K writes/sec directly | Single point of failure ❌ |
| **Async (Kafka Buffer)** | 5-10ms (fire-and-forget) ✅ | Workers consume at sustainable rate ✅ | Kafka replication ✅ |

**Trade-off:** Slight delay (200-500ms) between driver movement and index update (acceptable for 4-second update intervals).

---

## Matching Service

**Challenge:** Find optimal driver for a rider request within 100ms.

**Algorithm:**

1. **Proximity Search:**
   ```
   GEORADIUS drivers:sf {rider_lat} {rider_lng} 5km WITHDIST COUNT 20
   ```
   - Returns 20 nearest drivers within 5 km
   - Sorted by distance

2. **Filter by Availability:**
   - Query Driver State Store (Redis): `MGET driver:123:status driver:456:status ...`
   - Remove busy/offline drivers

3. **Calculate ETA:**
   - For each available driver, call ETA Service
   - ETA Service uses road network graph (GraphHopper, OSRM)
   - Returns travel time considering traffic

4. **Rank Drivers:**
   - Score = 0.7 × distance + 0.2 × ETA + 0.1 × driver_rating
   - Return top 5 drivers to rider

5. **Send Ride Requests:**
   - Send push notifications to top 5 drivers simultaneously
   - First to accept gets the trip

**Optimization: Smart Radius Expansion**

If < 5 drivers found within 5 km:
- Expand search to 10 km
- If still < 5, expand to 20 km
- Maximum radius: 50 km (prevent excessive searches)

---

## Sharding Strategy

**Problem:** 1M drivers globally can't fit in single Redis instance.

**Solution: Geographic Sharding**

**Approach:**
- Shard by city/region (e.g., `drivers:sf`, `drivers:nyc`, `drivers:london`)
- API Gateway routes requests to correct shard based on rider location
- Each city has dedicated Redis cluster (3-5 nodes)

**Sharding Logic:**
```
1. Rider location: lat=37.7749, lng=-122.4194
2. Reverse geocode to city: "san_francisco"
3. Route to Redis shard: drivers:sf
```

**Benefits:**
- ✅ **Isolation:** SF traffic doesn't impact NYC
- ✅ **Scalability:** Add new cities independently
- ✅ **Latency:** Data co-located with users

**Challenge: Cross-City Trips**
- Rider near city boundary might need drivers from adjacent city
- Solution: Query both shards, merge results

---

## Driver State Management

**Challenge:** Geo-index only stores location, not status (available/busy).

**Solution: Separate State Store**

**Data Model (Redis):**
```
Key: driver:{driver_id}:state
Value: {
    "status": "available",  // available, on_trip, offline
    "vehicle_type": "sedan",
    "rating": 4.8,
    "current_trip_id": null,
    "last_update": timestamp
}
TTL: 300 seconds (5 minutes)
```

**Operations:**
- On login: `SET driver:123:state {...} EX 300`
- Heartbeat: `EXPIRE driver:123:state 300` (refresh TTL)
- On trip start: Update status to "on_trip"
- On trip end: Update status to "available"

**Why Separate Store?**
- Geo-index optimized for location queries (not status)
- Status changes less frequent than location updates
- Allows independent scaling

---

## Bottlenecks & Solutions

### Bottleneck 1: Write Contention on Redis

**Problem:**
- 750K writes/sec concentrated in peak hours
- Single Redis instance: ~100K ops/sec max
- Need 8-10 Redis instances just for writes

**Solutions:**

1. **Geographic Sharding:**
   - Split by city: `drivers:sf`, `drivers:nyc`, etc.
   - Each city: 20K-50K drivers → manageable
   - SF: 50K drivers × 0.25 writes/sec = 12.5K writes/sec ✅

2. **Write Batching:**
   - Indexer workers batch 100 GEOADD commands
   - Single pipeline: `GEOADD drivers:sf {100 lat/lng pairs}`
   - Reduces network roundtrips by 100×

3. **Read Replicas:**
   - Master: Handles writes
   - 3 Read replicas: Handle GEORADIUS queries
   - Replication lag: <100ms (acceptable)

### Bottleneck 2: Geohash Boundary Issues

**Problem:**
- Geohash cells have sharp boundaries
- Driver 10 meters away but in different cell = not found

**Example:**
```
Rider at:    geohash = 9q8yy (SF downtown)
Driver A at: geohash = 9q8yy (same cell, 500m away) ✅ Found
Driver B at: geohash = 9q8yw (adjacent cell, 50m away) ❌ Missed!
```

**Solutions:**

1. **Check Adjacent Cells:**
   - Query 9 cells: center + 8 neighbors
   - 9 GEORADIUS queries → merge results
   - Increases latency: 10ms → 90ms ⚠️

2. **Larger Search Radius:**
   - Instead of 5 km, search 7 km
   - Covers boundary cases with margin
   - More results to filter (trade-off)

3. **H3 Hexagonal Grid:**
   - Hexagons have no sharp corners
   - Better neighbor coverage
   - Requires custom implementation

**Uber's Solution:** H3 grid eliminates boundary issues.

### Bottleneck 3: ETA Service Overload

**Problem:**
- Matching service calls ETA service for 20 drivers per search
- 50K searches/sec × 20 = 1M ETA requests/sec
- GraphHopper can't handle 1M requests/sec

**Solutions:**

1. **Caching:**
   - Key: `eta:{start_geohash_5}:{end_geohash_5}:{time_bucket_15min}`
   - Hit rate: ~60% (common routes cached)
   - 1M requests → 400K cache misses (manageable)

2. **Pre-Computation:**
   - Pre-compute ETA grid for city at 15-minute intervals
   - Store in Redis: `eta_grid:sf:{time_bucket}`
   - Lookup: O(1) instead of routing calculation

3. **Approximation:**
   - Use straight-line distance × 1.3 (Manhattan distance factor)
   - Accurate within 20% for most urban areas
   - Fallback when ETA service overloaded

### Bottleneck 4: Cold Start Problem

**Problem:**
- Driver logs in → location not in index yet
- First update via Kafka → 200-500ms delay
- Rider searches immediately → driver not found

**Solutions:**

1. **Synchronous Initial Update:**
   - On login: Driver app → POST /location (synchronous)
   - API directly writes to Redis (bypass Kafka)
   - Subsequent updates → async via Kafka

2. **Pre-Warm Index:**
   - When driver goes online, mark "status = available" in Driver State
   - Use last known location from database
   - Update with fresh GPS within 4 seconds

3. **Hybrid Approach:**
   - Login: Sync write to Redis (100ms)
   - Background: Also publish to Kafka (for audit trail)
   - Best of both worlds

---

## Common Anti-Patterns to Avoid

### 1. Polling for Nearby Drivers

❌ **Bad:** Polling every 5 seconds
- Wastes resources (repeated searches)
- 5-second latency = poor UX
- Doesn't scale (50K riders × 1 search/5s = 10K searches/sec wasted)

✅ **Good:** Event-Driven
- Rider subscribes to WebSocket
- Matching service pushes driver matches in real-time
- Driver acceptance notification via push

### 2. Storing Full Trip History in Redis

❌ **Bad:** Store all locations in Redis
- Redis memory expensive ($1/GB/month)
- 1M drivers × 15 trips/day × 200 locations/trip = 3B locations/day
- 3B × 40 bytes = 120 GB/day = $120/day just for storage

✅ **Good:** Multi-Tier Storage
- **Redis:** Only current location (40 MB total)
- **Cassandra:** Historical trip data (cheap storage)
- **S3:** Raw GPS logs for analytics (pennies per GB)

### 3. No Driver State TTL

❌ **Bad:** No TTL on driver status
- Driver goes offline (app crash, network drop)
- Status remains "available" forever
- Matching service sends ride requests to ghost drivers

✅ **Good:** TTL with Heartbeat
```
SET driver:123:status "available" EX 300  // 5-minute TTL
Driver heartbeat every 60 seconds refreshes TTL
```
If no heartbeat → key expires → driver marked offline automatically

### 4. Single Global Redis Instance

❌ **Bad:** All drivers worldwide → single Redis instance
- Latency: US rider queries Redis in EU (150ms network)
- Bottleneck: 750K writes/sec overwhelms single instance
- Failure: Redis down = global outage

✅ **Good:** Geographic Sharding
- **Geographic Sharding:** `drivers:sf`, `drivers:nyc`, `drivers:london`
- **Regional Deployment:** Redis co-located with users
- **Isolation:** SF outage doesn't affect NYC

---

## Monitoring & Observability

### Key Metrics

| Metric | Target | Alert Threshold | Description |
|--------|--------|-----------------|-------------|
| **Matching Latency (P99)** | < 100 ms | > 200 ms | Time to find nearest drivers |
| **Location Update Latency** | < 500 ms | > 2 seconds | Kafka → Redis update time |
| **Kafka Lag** | < 1000 messages | > 10K messages | Indexer worker lag |
| **Redis Write QPS** | < 100K per shard | > 150K per shard | Write contention |
| **Geo-Query Accuracy** | > 99.9% | < 99% | Missed drivers in search |
| **ETA Service Latency** | < 200 ms | > 1 second | Routing calculation time |

### Dashboards

**Dashboard 1: Real-Time Health**
- Active drivers per city
- Location update rate (writes/sec)
- Matching success rate
- Kafka lag per consumer group

**Dashboard 2: Geo-Spatial Performance**
- GEORADIUS query latency (P50, P99, P999)
- Search radius distribution
- Driver density per geohash cell
- Boundary issue frequency

**Critical Alerts:**
- Matching latency > 200ms for > 1 minute → Check Redis load, ETA service
- Kafka lag > 10K messages → Scale indexer workers
- Redis write QPS > 150K per shard → Add more shards
- Geo-query accuracy < 99% → Check boundary issues, geohash precision

---

## Trade-offs Summary

| Decision | Choice | Alternative | Why Chosen | Trade-off |
|----------|--------|-------------|------------|-----------|
| **Geo-Index** | Redis Geo (Geohash) | Elasticsearch, H3 | Native commands, simple | Boundary issues |
| **Location Updates** | Kafka Buffer | Direct Redis | Handles 750K writes/sec | 200-500ms lag |
| **Sharding** | Geographic by City | Consistent Hashing | Isolation, co-location | Cross-city complexity |
| **State Store** | Separate Redis | Combined with Geo | Independent scaling | Two stores to manage |
| **ETA Service** | GraphHopper + Cache | Google Maps API | Cost-effective | Requires caching |

### What We Gained

✅ **Low Latency:** <100ms matching via in-memory Redis geo-index
✅ **High Throughput:** 750K location updates/sec via Kafka buffering
✅ **Scalability:** Geographic sharding enables horizontal scaling
✅ **Availability:** Redis replication + Kafka replication = 99.99%
✅ **Accuracy:** Geohash enables precise proximity search

### What We Sacrificed

❌ **Eventual Consistency:** 200-500ms delay between driver movement and index update
❌ **Complexity:** Kafka + Redis + PostgreSQL = 3 data stores
❌ **Cost:** Redis RAM expensive for large fleets ($1/GB/month)
❌ **Boundary Issues:** Geohash cells have sharp boundaries (missed nearby drivers)
❌ **No Historical Queries:** Redis only stores current location, not trip history

---

## Real-World Implementations

### Uber
- **Architecture:** Custom H3 hexagonal grid (not Geohash), RocksDB (not Redis)
- **Scale:** 5M+ drivers, 100M+ riders, 15M trips/day
- **Key Insight:** Migrated from Geohash to H3 for better boundary handling, custom database for cost efficiency

### Lyft
- **Architecture:** S2 geometry library (Google's spherical indexing), DynamoDB + Redis hybrid
- **Scale:** 2M drivers, 20M+ riders
- **Key Insight:** Uses S2 cells (similar to H3), heavy use of AWS managed services

### Grab (Southeast Asia)
- **Architecture:** Redis Geo (Geohash), MySQL (sharded by city)
- **Scale:** 9M drivers, 187M users (across 8 countries)
- **Key Insight:** Stuck with Redis Geo due to cost constraints, OSRM for routing (free)

---

## Key Takeaways

1. **Redis Geo (Geohash)** provides sub-100ms proximity search for millions of drivers
2. **Kafka buffers** 750K location updates/sec, decouples producers from consumers
3. **Geographic sharding** by city enables horizontal scaling and isolation
4. **Separate state store** (Redis) for driver availability vs location (geo-index)
5. **H3 hexagonal grid** solves Geohash boundary issues (Uber's approach)
6. **ETA caching** reduces routing service load by 60%+
7. **TTL-based state** automatically marks offline drivers (heartbeat pattern)
8. **Event-driven matching** via WebSocket (not polling)
9. **Multi-tier storage:** Redis (current), Cassandra (history), S3 (analytics)
10. **Cold start mitigation:** Synchronous initial update on driver login

---

## Recommended Stack

- **Geo-Index:** Redis Cluster with Geo commands (Geohash) or H3 for production
- **Message Queue:** Apache Kafka (buffers 750K writes/sec)
- **State Store:** Redis Cluster (driver availability, rating)
- **Trip Database:** PostgreSQL (sharded by city, ACID transactions)
- **ETA Service:** GraphHopper or OSRM (open-source routing)
- **API Gateway:** NGINX or Kong (geo-aware routing)
- **Monitoring:** Prometheus + Grafana (metrics, dashboards)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (location update, matching, trip lifecycle)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (geohash, matching, ETA calculation)

