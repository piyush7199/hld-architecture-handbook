# 3.3.2 Design Uber/Lyft Ride Matching System

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design a real-time ride-matching system like Uber or Lyft that connects riders with nearby drivers within seconds. The
system must handle millions of concurrent drivers continuously updating their GPS coordinates, support sub-100ms
proximity searches to find the nearest available drivers, provide accurate ETAs, handle trip lifecycle management (
request → match → pickup → trip → payment), and ensure high availability despite network failures and connection drops.

**Core Challenge:** Efficiently index and query millions of moving objects (drivers) in two-dimensional space (
latitude/longitude) with sub-second latency while handling 1M+ location updates per second.

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement                 | Description                                                             | Priority     |
|-----------------------------|-------------------------------------------------------------------------|--------------|
| **Driver Location Updates** | Drivers report GPS coordinates every 4 seconds (heartbeat)              | Must Have    |
| **Proximity Search**        | Find N nearest available drivers within radius for a rider              | Must Have    |
| **Real-Time ETA**           | Calculate estimated time of arrival for matched drivers                 | Must Have    |
| **Trip Matching**           | Match rider with optimal driver (distance, rating, vehicle type)        | Must Have    |
| **Trip State Management**   | Track trip lifecycle: requested → matched → pickup → active → completed | Must Have    |
| **Driver Status**           | Track driver availability: online, offline, on-trip, break              | Must Have    |
| **Surge Pricing**           | Dynamic pricing based on supply/demand in geohash cells                 | Should Have  |
| **Driver Routing**          | Navigate driver to pickup location, then to destination                 | Should Have  |
| **Multi-Ride Types**        | Support UberX, UberXL, Uber Black (different vehicle types)             | Nice to Have |

### Non-Functional Requirements (NFRs)

| Requirement              | Target             | Rationale                                |
|--------------------------|--------------------|------------------------------------------|
| **Low Latency (Search)** | < 100 ms (p99)     | Fast matching prevents rider abandonment |
| **High Availability**    | 99.99% uptime      | Service downtime = revenue loss          |
| **Write Throughput**     | 1M updates/sec     | 1M drivers × 1 update/sec                |
| **Read Throughput**      | 50K searches/sec   | Peak ride requests                       |
| **Geo-Query Accuracy**   | 99.9%              | Missed drivers = poor UX                 |
| **Scalability**          | Global, multi-city | Horizontal scaling required              |

### Scale Estimation

| Metric                             | Assumption                             | Calculation                              | Result                 |
|------------------------------------|----------------------------------------|------------------------------------------|------------------------|
| **Total Drivers (Global)**         | 1 Million active drivers               | -                                        | 1M drivers             |
| **Driver Location Updates**        | 1 update per 4 seconds per driver      | $1\text{M} / 4 = 250\text{K}$ writes/sec | 250K writes/sec (avg)  |
| **Peak Updates**                   | 3× average during rush hour            | $250\text{K} \times 3$                   | 750K writes/sec (peak) |
| **Concurrent Riders**              | 50,000 active riders searching         | -                                        | 50K riders             |
| **Proximity Searches**             | 1 search per rider request             | $50\text{K}$ searches/sec                | 50K reads/sec          |
| **Total Throughput**               | Writes + Reads                         | $750\text{K} + 50\text{K}$               | ~800K ops/sec (peak)   |
| **Driver Data per Record**         | lat, lng, status, timestamp, driver_id | $40$ bytes/record                        | 40 bytes               |
| **Total Memory (In-Memory Index)** | 1M drivers × 40 bytes                  | $1\text{M} \times 40$                    | ~40 MB (fits in RAM)   |

**Key Insights:**

- **Write-heavy workload:** 750K writes/sec vs 50K reads/sec (15:1 ratio)
- **In-memory feasible:** 40 MB index fits entirely in RAM (Redis)
- **Low latency critical:** Rider abandonment increases exponentially after 3 seconds
- **Global sharding needed:** Single datacenter can't serve all cities

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### System Overview

```
                    ┌────────────────────────────────────┐
                    │      Driver App (Mobile)           │
                    │  GPS: lat=37.7749, lng=-122.4194   │
                    └─────────────┬──────────────────────┘
                                  │ HTTPS (every 4s)
                    ┌─────────────▼──────────────────────┐
                    │       API Gateway / Load Balancer  │
                    │     (Geo-aware routing by city)    │
                    └─────────────┬──────────────────────┘
                                  │
                    ┌─────────────▼──────────────────────┐
                    │  Location Ingestion Service        │
                    │  - Validate coordinates            │
                    │  - Publish to Kafka                │
                    └─────────────┬──────────────────────┘
                                  │
                    ┌─────────────▼──────────────────────┐
                    │        Kafka Cluster               │
                    │   (Buffer 750K updates/sec)        │
                    └─────┬────────────────┬─────────────┘
                          │                │
           ┌──────────────▼──┐      ┌─────▼──────────────┐
           │ Indexer Workers │      │ Driver State Worker│
           │ (100 consumers) │      │ (status, rating)   │
           └──────────┬──────┘      └─────┬──────────────┘
                      │                    │
           ┌──────────▼──────────────┐     │
           │  Redis Cluster (Geo)    │     │
           │  - Geohash Index        │     │
           │  - 20 shards by city    │     │
           │  - GEOADD commands      │     │
           └──────────┬──────────────┘     │
                      │                    │
                      │         ┌──────────▼──────────────┐
                      │         │  Redis (Driver State)   │
                      │         │  - available/busy       │
                      │         │  - rating, vehicle type │
                      │         └─────────────────────────┘
                      │
        ┌─────────────▼─────────────┐
        │   Rider App (Mobile)       │
        │   "Find me a ride"         │
        └─────────────┬───────────────┘
                      │ HTTPS
        ┌─────────────▼───────────────┐
        │  Matching Service            │
        │  - GEORADIUS query           │
        │  - Filter by availability    │
        │  - Calculate ETA             │
        │  - Return top 5 drivers      │
        └─────────────┬───────────────┘
                      │
        ┌─────────────▼───────────────┐
        │  Trip Management Service     │
        │  - PostgreSQL (ACID)         │
        │  - State: request → match    │
        │  - Payment processing        │
        └──────────────────────────────┘
```

### Key Components

| Component              | Responsibility                                       | Technology              | Scalability                  |
|------------------------|------------------------------------------------------|-------------------------|------------------------------|
| **API Gateway**        | Route requests to nearest datacenter                 | NGINX, Kong             | Horizontal + GeoDNS          |
| **Location Ingestion** | Validate GPS, publish to Kafka                       | Go, Java                | Horizontal (stateless)       |
| **Kafka Cluster**      | Buffer 750K writes/sec, decouple producers/consumers | Apache Kafka            | Horizontal (partitioned)     |
| **Indexer Workers**    | Consume from Kafka, update Redis geo-index           | Go workers              | Horizontal (100+ consumers)  |
| **Redis Geo Cluster**  | In-memory geohash index for proximity search         | Redis with Geo commands | Horizontal (sharded by city) |
| **Driver State Store** | Store driver status, rating, vehicle type            | Redis                   | Horizontal (sharded)         |
| **Matching Service**   | Find nearest drivers, calculate ETA                  | Go, Rust                | Horizontal (stateless)       |
| **Trip Management**    | ACID transactions for trip lifecycle                 | PostgreSQL (sharded)    | Horizontal (sharded by city) |
| **ETA Service**        | Calculate travel time using road network             | GraphHopper, OSRM       | Horizontal                   |

---

## 4. Detailed Component Design

### 4.1 Geo-Spatial Indexing Strategy

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

| Factor              | Geohash                                | H3 (Uber's Hexagonal Grid)         |
|---------------------|----------------------------------------|------------------------------------|
| **Simplicity**      | String prefix matching ✅               | Complex hexagonal math ❌           |
| **Redis Support**   | Native GEOADD, GEORADIUS ✅             | Custom implementation ❌            |
| **Boundary Issues** | Yes (edge cases at cell boundaries) ⚠️ | Better (hexagons tile perfectly) ✅ |
| **Adoption**        | Widely supported ✅                     | Uber-specific, less tooling ❌      |

**Decision:** Use **Geohash** for MVP, migrate to **H3** for production at Uber scale.

*See [this-over-that.md: Geohash vs H3](this-over-that.md) for detailed comparison.*

---

### 4.2 Redis Geo Commands

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

*See [pseudocode.md::update_driver_location()](pseudocode.md) for implementation.*

---

### 4.3 Location Ingestion Pipeline

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

| Approach                       | Latency (Driver POV)       | Database Load                         | Fault Tolerance           |
|--------------------------------|----------------------------|---------------------------------------|---------------------------|
| **Synchronous (Direct Redis)** | 50-100ms (waits for DB)    | 750K writes/sec directly              | Single point of failure ❌ |
| **Async (Kafka Buffer)**       | 5-10ms (fire-and-forget) ✅ | Workers consume at sustainable rate ✅ | Kafka replication ✅       |

**Trade-off:** Slight delay (200-500ms) between driver movement and index update (acceptable for 4-second update
intervals).

*See [sequence-diagrams.md: Location Update Flow](sequence-diagrams.md) for detailed flow.*

---

### 4.4 Matching Service

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

*See [pseudocode.md::find_nearest_drivers()](pseudocode.md) for implementation.*

---

### 4.5 Sharding Strategy

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

*See [this-over-that.md: Sharding Strategies](this-over-that.md) for alternatives.*

---

### 4.6 Driver State Management

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

1. **Set Driver Available:**
   ```
   HSET driver:123:state status "available" vehicle_type "sedan" rating 4.8
   EXPIRE driver:123:state 300
   ```

2. **Check Availability (Batch):**
   ```
   HMGET driver:123:state driver:456:state driver:789:state
   ```

3. **Update to Busy (on trip match):**
   ```
   HSET driver:123:state status "on_trip" current_trip_id "trip-abc-123"
   ```

**Why Separate from Geo-Index?**

| Aspect               | Combined (Single Redis Key)                | Separated (Two Keys)     |
|----------------------|--------------------------------------------|--------------------------|
| **Update Frequency** | Location: every 4s, Status: on trip change | Optimized per use case ✅ |
| **Query Pattern**    | Geo query + status filter mixed            | Clean separation ✅       |
| **TTL Management**   | Complex (different lifetimes)              | Independent TTLs ✅       |

---

### 4.7 Trip State Machine

**States:**

1. **REQUESTED:** Rider submitted request
2. **SEARCHING:** Matching service finding drivers
3. **MATCHED:** Driver accepted, navigating to pickup
4. **ARRIVED:** Driver at pickup location
5. **IN_PROGRESS:** Trip started
6. **COMPLETED:** Trip finished
7. **CANCELLED:** Trip cancelled by rider/driver

**State Transitions:**

```
REQUESTED → SEARCHING → MATCHED → ARRIVED → IN_PROGRESS → COMPLETED
     ↓          ↓          ↓          ↓           ↓
  CANCELLED  CANCELLED  CANCELLED  CANCELLED  COMPLETED
```

**Database: PostgreSQL (ACID)**

**Why PostgreSQL over NoSQL?**

Trip state involves:

- Payment processing (requires ACID)
- Refunds/cancellation fees (requires transactions)
- Audit trail (requires strong consistency)

**Schema:**

```sql
CREATE TABLE trips (
    trip_id UUID PRIMARY KEY,
    rider_id UUID NOT NULL,
    driver_id UUID,
    status VARCHAR(20) NOT NULL,
    pickup_lat DECIMAL(10, 8),
    pickup_lng DECIMAL(11, 8),
    dropoff_lat DECIMAL(10, 8),
    dropoff_lng DECIMAL(11, 8),
    estimated_fare DECIMAL(10, 2),
    actual_fare DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT NOW(),
    matched_at TIMESTAMP,
    completed_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(255)
);

CREATE INDEX idx_rider_trips ON trips(rider_id, created_at DESC);
CREATE INDEX idx_driver_trips ON trips(driver_id, created_at DESC);
```

*See [pseudocode.md::trip_state_machine()](pseudocode.md) for state transition logic.*

---

### 4.8 ETA Calculation

**Challenge:** Calculate accurate travel time considering:

- Road network (not straight-line distance)
- Traffic conditions
- One-way streets, turn restrictions

**Solution: GraphHopper / OSRM**

**GraphHopper:**

- Open-source routing engine
- Pre-processes OpenStreetMap (OSM) data into road network graph
- Dijkstra's algorithm with A* optimization
- Considers traffic via real-time APIs (Google Traffic, TomTom)

**API Call:**

```
GET /route?point=37.7749,-122.4194&point=37.7849,-122.4094&vehicle=car
→ {
    "distance": 2500 (meters),
    "time": 420 (seconds = 7 minutes),
    "path": [...coordinates...]
}
```

**Caching Strategy:**

- Cache common routes (e.g., airport → downtown)
- TTL: 15 minutes (traffic changes)
- Key: `route:{start_geohash}:{end_geohash}:{time_bucket}`

**Fallback (if GraphHopper fails):**

- Haversine distance × average city speed
- Example: 2 km / 30 km/h = 4 minutes

*See [pseudocode.md::calculate_eta()](pseudocode.md) for implementation.*

---

## 5. Why This Over That?

### Decision 1: Redis Geo vs PostgreSQL PostGIS

**Chosen:** Redis with GEOADD/GEORADIUS

**Why:**

- ✅ **Speed:** In-memory (sub-millisecond queries)
- ✅ **Throughput:** Handles 750K writes/sec easily
- ✅ **Native Commands:** GEORADIUS built-in
- ✅ **Simplicity:** No complex SQL spatial queries

**Alternatives:**

**PostgreSQL PostGIS:**

- ❌ **Slower:** Disk-based, 10-50ms queries
- ❌ **Write Bottleneck:** 10K writes/sec max (needs sharding)
- ✅ **ACID:** Better for persistent trip data
- **When to Use:** For historical trip data, not real-time indexing

**MongoDB with Geo Indexes:**

- ⚠️ **Moderate Speed:** 5-20ms queries
- ⚠️ **Write Throughput:** 50K writes/sec (better than Postgres, worse than Redis)
- ✅ **Document Model:** Good for complex driver profiles
- **When to Use:** If driver profiles are complex documents

*See [this-over-that.md: Redis vs PostGIS vs MongoDB](this-over-that.md)*

---

### Decision 2: Kafka vs Direct Database Writes

**Chosen:** Kafka Buffer

**Why:**

- ✅ **Burst Handling:** Absorbs 750K writes/sec spikes
- ✅ **Decoupling:** Drivers don't wait for database writes
- ✅ **Fault Tolerance:** Kafka replication prevents data loss
- ✅ **Replay:** Can reprocess location history

**Alternative: Synchronous Redis Writes:**

- ❌ **Latency:** Driver app waits 50-100ms for Redis ACK
- ❌ **Overload Risk:** Single Redis failure blocks all writes
- ✅ **Simpler:** No Kafka infrastructure
- **When to Use:** MVP with <10K drivers

---

### Decision 3: Geohash vs H3 Hexagonal Grid

**Chosen:** Geohash (for MVP)

**Why:**

- ✅ **Redis Native:** GEOADD supports Geohash internally
- ✅ **Standard:** Widely adopted, more libraries
- ✅ **Simple:** String prefix matching

**Alternative: H3 (Uber's Choice):**

- ✅ **Better Boundaries:** Hexagons tile perfectly (no edge artifacts)
- ✅ **Uniform Distance:** All neighbors equidistant
- ❌ **Custom Implementation:** Redis doesn't support H3 natively
- ❌ **Complexity:** Requires custom indexing logic
- **When to Use:** At Uber scale (millions of drivers)

**Uber's Actual Architecture:**

- Started with Geohash
- Migrated to H3 at 1M+ drivers for better accuracy
- Custom H3-optimized database (no Redis)

*See [this-over-that.md: Geohash vs H3 Deep Dive](this-over-that.md)*

---

## 6. Bottlenecks and Future Scaling

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

*See [pseudocode.md::batch_update_locations()](pseudocode.md)*

---

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

*See [hld-diagram.md: Boundary Problem Visualization](hld-diagram.md)*

---

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

*See [pseudocode.md::calculate_eta_with_cache()](pseudocode.md)*

---

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

## 7. Common Anti-Patterns

### ❌ Anti-Pattern 1: Polling for Nearby Drivers

**Bad Approach:**

```
while (not matched):
    drivers = search_nearby_drivers(rider_location)
    if drivers.length > 0:
        send_ride_request(drivers[0])
    sleep(5 seconds)
```

**Why Bad:**

- Wastes resources (repeated searches)
- 5-second latency = poor UX
- Doesn't scale (50K riders × 1 search/5s = 10K searches/sec wasted)

**✅ Best Practice:**

- **Event-Driven:** Rider subscribes to WebSocket
- Matching service pushes driver matches in real-time
- Driver acceptance notification via push

*See [sequence-diagrams.md: Event-Driven Matching](sequence-diagrams.md)*

---

### ❌ Anti-Pattern 2: Storing Full Trip History in Redis

**Bad Approach:**

```
GEOADD driver_history:123 {lat} {lng} {timestamp} → Store all locations
```

**Why Bad:**

- Redis memory expensive ($1/GB/month)
- 1M drivers × 15 trips/day × 200 locations/trip = 3B locations/day
- 3B × 40 bytes = 120 GB/day = $120/day just for storage

**✅ Best Practice:**

- **Redis:** Only current location (40 MB total)
- **Cassandra:** Historical trip data (cheap storage)
- **S3:** Raw GPS logs for analytics (pennies per GB)

---

### ❌ Anti-Pattern 3: No Driver State TTL

**Bad Approach:**

```
SET driver:123:status "available"  // No TTL
```

**Why Bad:**

- Driver goes offline (app crash, network drop)
- Status remains "available" forever
- Matching service sends ride requests to ghost drivers

**✅ Best Practice:**

```
SET driver:123:status "available" EX 300  // 5-minute TTL
Driver heartbeat every 60 seconds refreshes TTL
```

If no heartbeat → key expires → driver marked offline automatically

---

### ❌ Anti-Pattern 4: Single Global Redis Instance

**Bad Approach:**

```
All drivers worldwide → single Redis instance "drivers:global"
```

**Why Bad:**

- Latency: US rider queries Redis in EU (150ms network)
- Bottleneck: 750K writes/sec overwhelms single instance
- Failure: Redis down = global outage

**✅ Best Practice:**

- **Geographic Sharding:** `drivers:sf`, `drivers:nyc`, `drivers:london`
- **Regional Deployment:** Redis co-located with users
- **Isolation:** SF outage doesn't affect NYC

---

## 8. Alternative Approaches

### Alternative 1: Elasticsearch for Geo-Search

**Architecture:**

- Replace Redis with Elasticsearch
- Use `geo_distance` query
- Store driver documents with `geo_point` field

**Pros:**

- ✅ **Rich Queries:** Filter by rating, vehicle type, distance simultaneously
- ✅ **Full-Text Search:** Search drivers by name, license plate
- ✅ **Analytics:** Built-in aggregations (surge pricing zones)

**Cons:**

- ❌ **Slower Writes:** 10K writes/sec/node (vs Redis 100K)
- ❌ **Higher Latency:** 20-50ms queries (vs Redis <5ms)
- ❌ **Complex:** Requires cluster management, replication

**When to Use:** If you need rich filtering (e.g., "5-star drivers with SUV within 2 km").

---

### Alternative 2: DynamoDB with Geohashing

**Architecture:**

- Store drivers in DynamoDB
- Partition key: geohash (first 4 chars)
- Sort key: geohash (full) + driver_id

**Pros:**

- ✅ **Serverless:** No infrastructure management
- ✅ **Scalability:** Auto-scales to millions of writes/sec
- ✅ **Durability:** Multi-AZ replication

**Cons:**

- ❌ **No Native Geo:** Manual geohash querying (query 9 cells)
- ❌ **Cost:** $1.25 per million writes ($937/day for 750K writes/sec)
- ❌ **Latency:** 10-30ms (vs Redis <5ms)

**When to Use:** If using AWS ecosystem and willing to pay for serverless simplicity.

---

### Alternative 3: QuadTree In-Memory Structure

**Architecture:**

- Custom in-memory QuadTree
- Each node: Bounding box with driver list
- Recursively split when node exceeds 100 drivers

**Pros:**

- ✅ **Fast:** O(log N) insertion, O(k + log N) query
- ✅ **No External DB:** Pure in-memory
- ✅ **Fine-Tuned:** Optimize for specific use case

**Cons:**

- ❌ **Complex:** Implement tree rebalancing, concurrency
- ❌ **No Persistence:** Lost on server restart
- ❌ **Scaling:** Difficult to shard across nodes

**When to Use:** For embedded systems or specialized hardware (not distributed systems).

---

## 9. Monitoring and Observability

### Critical Metrics

| Metric                          | Target        | Alert Threshold |
|---------------------------------|---------------|-----------------|
| **Matching Latency (P99)**      | < 100ms       | > 200ms 🔴      |
| **Driver Location Update Rate** | 750K/sec peak | < 500K/sec 🟡   |
| **Redis Memory Usage**          | < 80%         | > 90% 🔴        |
| **Kafka Consumer Lag**          | < 1000 msgs   | > 10K msgs 🔴   |
| **Match Success Rate**          | > 95%         | < 90% 🔴        |
| **Driver Availability**         | 1M active     | < 800K 🟡       |
| **ETA Accuracy**                | ±20%          | ±40% 🟡         |

### Dashboards

1. **Real-Time Operations:**
    - Active drivers by city (map view)
    - Location updates/sec (line chart)
    - Matching requests/sec (line chart)
    - Average matching latency (histogram)

2. **Geo-Index Health:**
    - Redis memory usage (gauge)
    - GEOADD operations/sec (line chart)
    - GEORADIUS query latency (histogram)
    - Cache hit rate (gauge)

3. **Trip Funnel:**
    - Requests → Searches → Matches → Completions
    - Conversion rate at each stage
    - Drop-off analysis

### Alerts

**🔴 Critical (Page On-Call):**

- Matching service down (no ride requests processed)
- Redis cluster down (no geo-index)
- Kafka broker failure (location updates lost)
- Match success rate < 90% (insufficient drivers)

**🟡 Warning (Investigate Next Day):**

- ETA service latency > 500ms
- Redis memory > 90% (scale up)
- Kafka consumer lag > 10K (add consumers)

*See [hld-diagram.md: Monitoring Dashboard](hld-diagram.md)*

---

## 10. Trade-offs Summary

### What We Gained ✅

| Benefit             | Explanation                                    |
|---------------------|------------------------------------------------|
| **Low Latency**     | <100ms matching via in-memory Redis geo-index  |
| **High Throughput** | 750K location updates/sec via Kafka buffering  |
| **Scalability**     | Geographic sharding enables horizontal scaling |
| **Availability**    | Redis replication + Kafka replication = 99.99% |
| **Accuracy**        | Geohash enables precise proximity search       |

### What We Sacrificed ❌

| Trade-off                 | Impact                                                      |
|---------------------------|-------------------------------------------------------------|
| **Eventual Consistency**  | 200-500ms delay between driver movement and index update    |
| **Complexity**            | Kafka + Redis + PostgreSQL = 3 data stores                  |
| **Cost**                  | Redis RAM expensive for large fleets ($1/GB/month)          |
| **Boundary Issues**       | Geohash cells have sharp boundaries (missed nearby drivers) |
| **No Historical Queries** | Redis only stores current location, not trip history        |

---

## 11. Real-World Implementations

### Uber

**Architecture:**

- **Geo-Index:** Custom H3 hexagonal grid (not Geohash)
- **Database:** RocksDB (custom key-value store, not Redis)
- **Routing:** Internal "Otto" routing engine (not GraphHopper)
- **Scale:** 5M+ drivers, 100M+ riders, 15M trips/day

**Key Insights:**

- Migrated from Geohash to H3 for better boundary handling
- Custom database (RocksDB) for cost efficiency (Redis too expensive at scale)
- Multi-region deployment (20+ cities, 100+ datacenters)

**Why Custom Tech Stack?**

- Off-the-shelf Redis couldn't handle 5M drivers
- H3 provides 15% better matching accuracy than Geohash
- RocksDB: 10× cheaper than Redis for same storage

---

### Lyft

**Architecture:**

- **Geo-Index:** S2 geometry library (Google's spherical indexing)
- **Database:** DynamoDB + Redis hybrid
- **Routing:** Mapbox Directions API
- **Scale:** 2M drivers, 20M+ riders

**Key Insights:**

- Uses S2 cells (similar to H3, but spherical coordinates)
- DynamoDB for durability, Redis for speed
- Heavy use of AWS managed services (less custom infrastructure)

---

### Grab (Southeast Asia)

**Architecture:**

- **Geo-Index:** Redis Geo (Geohash)
- **Database:** MySQL (sharded by city)
- **Routing:** Open-source OSRM
- **Scale:** 9M drivers, 187M users (across 8 countries)

**Key Insights:**

- Stuck with Redis Geo (Geohash) due to cost constraints
- MySQL instead of Cassandra (team expertise)
- OSRM for routing (free, vs Google Maps $7/1000 requests)

---

## 12. Deep Dive: Advanced Topics

### 12.1 Surge Pricing

**Problem:** Too many riders, not enough drivers.

**Solution: Dynamic Pricing by Geohash Cell**

1. **Calculate Supply/Demand Ratio:**
   ```
   For each geohash cell (1 km × 1 km):
   available_drivers = COUNT(drivers with status=available in cell)
   pending_requests = COUNT(unmatched riders in cell)
   ratio = available_drivers / pending_requests
   ```

2. **Apply Surge Multiplier:**
   ```
   if ratio < 0.5:  surge = 2.0×  (high demand)
   elif ratio < 1.0: surge = 1.5×
   else: surge = 1.0× (no surge)
   ```

3. **Update Every 2 Minutes:**
    - Background job recalculates surge for all cells
    - Riders see surge pricing before requesting ride

**Data Storage:**

```
Key: surge:{city}:{geohash_5}
Value: {"multiplier": 1.5, "updated_at": timestamp}
TTL: 300 seconds (5 minutes)
```

---

### 12.2 Driver Repositioning

**Problem:** Drivers cluster in downtown, but demand is in suburbs.

**Solution: Incentivize Movement**

1. **Heat Map:** Show drivers where demand is high
2. **Destination Mode:** Driver sets preferred direction (e.g., "heading home")
3. **Bonuses:** "Drive to Airport, get $5 bonus for next trip"

**ML Model:**

- Predict demand by geohash cell for next 30 minutes
- Input: historical trip data, events (concerts, games), weather
- Output: "Probability of ride request in cell X at time T"

---

### 12.3 Multi-Pickup (UberPool)

**Problem:** Multiple riders, similar routes → share ride.

**Challenge:** Find riders with overlapping routes in real-time.

**Approach:**

1. **Rider A requests ride:** pickup A → dropoff A
2. **Check for compatible riders:**
    - Rider B: pickup B within 500m of A's route, dropoff B within 500m of A's dropoff
3. **Calculate detour:**
    - Original trip A: 10 minutes
    - With pickup B: 13 minutes (30% detour)
    - If detour < 30% → match

**Data Structure:**

```
Active pool requests:
[
  {rider_id: A, pickup: {lat, lng}, dropoff: {lat, lng}, route: [...]},
  {rider_id: B, pickup: {lat, lng}, dropoff: {lat, lng}, route: [...]},
]

For new rider C:
  - Check all active requests
  - Calculate route overlap
  - If overlap > 70% → suggest pool
```

---

## 13. Interview Discussion Points

### Q1: How Would You Handle 10× Traffic Growth?

**Answer:**

**From 1M → 10M drivers:**

1. **Redis Sharding:**
    - Current: 20 city shards (50K drivers each)
    - New: 200 city shards (50K drivers each)
    - Geographic + sub-city sharding (e.g., sf-north, sf-south)

2. **Kafka Partitioning:**
    - Current: 100 partitions
    - New: 1000 partitions
    - More consumers (1000 indexer workers)

3. **Regional Datacenters:**
    - Deploy Redis clusters in 50+ cities (vs current 20)
    - Reduce cross-region latency

4. **Cost:**
    - Redis: 40 MB → 400 MB (still fits in RAM, $5/month/shard)
    - Kafka: Add brokers (100 → 200 brokers)
    - Total infrastructure: $50K/month → $500K/month

---

### Q2: What If Redis Goes Down?

**Answer:**

**Failure Scenarios:**

1. **Single Node Failure:**
    - Redis Sentinel detects failure (heartbeat timeout)
    - Promotes replica to master (<5 seconds)
    - Impact: <5 seconds of degraded matching

2. **Entire Shard Failure (e.g., sf shard):**
    - Fallback: Query adjacent cities (oakland, san_jose)
    - Wider search radius (20 km instead of 5 km)
    - Temporary: "Finding drivers, please wait..."

3. **Complete Redis Cluster Failure:**
    - Rebuild index from Kafka replay (last 7 days retained)
    - Time to rebuild: 10 minutes for 1M drivers
    - Downtime: 10 minutes (worst case)

**Mitigation:**

- Multi-AZ Redis deployment (3 nodes across availability zones)
- Cross-region replication (replicate sf shard to nyc as backup)

---

### Q3: How to Prevent Driver Spoofing (Fake GPS)?

**Answer:**

**Attack:** Driver spoofs GPS to appear closer to rider.

**Detection:**

1. **Velocity Check:**
    - If driver moves 10 km in 1 second → physically impossible
    - Flag: `driver_behavior:suspicious_velocity`

2. **Accelerometer Data:**
    - Mobile app sends accelerometer readings
    - If GPS shows movement but accelerometer shows stillness → spoofing

3. **Cell Tower Triangulation:**
    - Cross-check GPS with cell tower location
    - If mismatch > 1 km → suspicious

4. **ML Model:**
    - Train on historical trip patterns
    - Detect anomalies (e.g., driver teleporting)

**Response:**

- Warning: "We detected unusual location activity"
- Suspension: Temporary account lock
- Permanent ban: Repeat offenders

---

### Q4: How to Handle Cross-Border Trips?

**Problem:** Rider in Detroit requests ride to Windsor, Canada.

**Challenges:**

- Different currency (USD → CAD)
- Different regulations (insurance, licensing)
- Driver may not have cross-border permit

**Solutions:**

1. **Block Cross-Border by Default:**
    - Geofence: Detect dropoff location in different country
    - Show error: "Cross-border trips not supported"

2. **Special Cross-Border Mode:**
    - Only drivers with permits can accept
    - Filter: `driver:123:permits = ["US", "CA"]`
    - Higher fare (cross-border surcharge)

3. **Hand-Off:**
    - Driver A (US) drives to border
    - Rider transfers to Driver B (CA) at border
    - Two separate trips, two payments

---

### Q5: How to Optimize for Electric Vehicles (EVs)?

**Problem:** EV range limited, need charging station routing.

**Solutions:**

1. **Battery-Aware Matching:**
    - Track driver battery level: `driver:123:battery = 40%`
    - Don't assign long trips (>50 km) to drivers with <50% battery

2. **Charging Station Routing:**
    - If driver battery <20%, suggest route to nearby charging station
    - ETA service considers charging time: "15 min drive + 30 min charge"

3. **Incentives:**
    - "Drive to charging station during off-peak, earn $10 bonus"
    - Prevents EV drivers from running out of charge mid-shift

---

## 14. References

### Related System Design Components

- **[2.1.11 Redis Deep Dive](../../02-components/2.1.11-redis-deep-dive.md)** - Redis Geo commands (GEOADD, GEORADIUS)
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)** - Message streaming, buffering
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - PostgreSQL sharding strategies
- **[2.2.2 Consistent Hashing](../../02-components/2.2.2-consistent-hashing.md)** - Load distribution
- **[2.0.3 Real-Time Communication](../../02-components/2.0.3-real-time-communication.md)** - WebSocket patterns for
  real-time updates

### Related Design Challenges

- **[3.3.1 Live Chat System](../3.3.1-live-chat-system/)** - Real-time location updates, WebSocket patterns
- **[3.2.2 Notification Service](../3.2.2-notification-service/)** - Push notifications to drivers
- **[3.5.6 Yelp/Google Maps](../3.5.6-yelp-google-maps/)** - Geo-spatial search patterns

### External Resources

- **Uber Engineering Blog:** [H3: Uber's Hexagonal Hierarchical Spatial Index](https://eng.uber.com/h3/) - H3 grid
  system
- **Redis Documentation:** [Geo Commands](https://redis.io/commands/geoadd/) - GEOADD, GEORADIUS
- **GraphHopper:** [Open-source routing engine](https://www.graphhopper.com/) - ETA calculation
- **H3 Library:** [H3 Documentation](https://h3geo.org/) - Hexagonal hierarchical spatial index
- **OSRM:** [Open Source Routing Machine](http://project-osrm.org/) - Routing engine alternative

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Chapters on geo-spatial indexing and real-time systems
- *Redis in Action* by Josiah Carlson - Redis Geo commands and patterns