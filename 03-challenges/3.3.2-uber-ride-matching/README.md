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

## Problem Statement

Design a real-time ride-matching system like Uber or Lyft that connects riders with nearby drivers within seconds. The system must handle millions of concurrent drivers continuously updating their GPS coordinates, support sub-100ms proximity searches to find the nearest available drivers, provide accurate ETAs, handle trip lifecycle management (request → match → pickup → trip → payment), and ensure high availability despite network failures and connection drops.

**Core Challenge:** Efficiently index and query millions of moving objects (drivers) in two-dimensional space (latitude/longitude) with sub-second latency while handling 1M+ location updates per second.

---

## Requirements

### Functional Requirements

- **Driver Location Updates:** Drivers report GPS coordinates every 4 seconds (heartbeat)
- **Proximity Search:** Find N nearest available drivers within radius for a rider
- **Real-Time ETA:** Calculate estimated time of arrival for matched drivers
- **Trip Matching:** Match rider with optimal driver (distance, rating, vehicle type)
- **Trip State Management:** Track trip lifecycle: requested → matched → pickup → active → completed
- **Driver Status:** Track driver availability: online, offline, on-trip, break
- **Surge Pricing:** Dynamic pricing based on supply/demand in geohash cells
- **Driver Routing:** Navigate driver to pickup location, then to destination

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Latency (Search) | < 100 ms (p99) |
| Availability | 99.99% |
| Write Throughput | 750K updates/sec (peak) |
| Read Throughput | 50K searches/sec |
| Geo-Query Accuracy | 99.9% |

### Scale Estimation

| Metric | Value |
|--------|-------|
| Total Drivers | 1 Million active |
| Location Updates | 250K writes/sec (avg), 750K peak |
| Concurrent Riders | 50,000 searching |
| Proximity Searches | 50K reads/sec |
| Total Throughput | ~800K ops/sec (peak) |
| Memory (Geo-Index) | ~40 MB (in-memory) |

**Key Insight:** 40 MB index fits entirely in RAM → Redis perfect choice

---

## High-Level Architecture

> 📊 **See detailed architecture:** [HLD Diagrams](./hld-diagram.md)

### System Overview

```
Driver App (GPS) → API Gateway → Location Service
                                        ↓
                                    Kafka Buffer (750K writes/sec)
                                        ↓
                            ┌───────────┴───────────┐
                            ↓                       ↓
                    Indexer Workers          Driver State Worker
                            ↓                       ↓
                    Redis Geo Index          Redis State Store
                    (GEOADD/GEORADIUS)      (available/busy)
                            ↓
                    Matching Service
                    (Find nearest drivers)
                            ↓
                    Trip Management
                    (PostgreSQL ACID)
```

### Key Components

| Component | Technology | Responsibility |
|-----------|------------|----------------|
| **API Gateway** | NGINX, Kong | Geo-aware routing by city |
| **Location Ingestion** | Go, Java | Validate GPS, publish to Kafka |
| **Kafka** | Apache Kafka | Buffer 750K writes/sec |
| **Indexer Workers** | Go | Update Redis geo-index |
| **Redis Geo** | Redis w/ Geo commands | In-memory geohash index |
| **Matching Service** | Go, Rust | Find nearest drivers, ETA |
| **Trip Management** | PostgreSQL | ACID transactions |

---

## Key Design Decisions

### Decision 1: Redis Geo vs PostGIS

**Chosen:** Redis with GEOADD/GEORADIUS

**Why:**
- ✅ In-memory (sub-millisecond queries)
- ✅ Handles 750K writes/sec easily
- ✅ Native GEORADIUS command
- ✅ Simple setup (vs complex SQL)

**Alternatives:**
- ❌ PostgreSQL PostGIS: Too slow (10-50ms queries), write bottleneck
- ❌ MongoDB Geo: Moderate speed (5-20ms), worse than Redis

*See [this-over-that.md](this-over-that.md) for detailed analysis*

### Decision 2: Geohash Encoding

**Chosen:** Geohash (for MVP)

**Why:**
- ✅ Converts 2D space (lat/lng) → 1D string
- ✅ Nearby locations share prefix
- ✅ Redis native support
- ✅ Fast range queries

**How it Works:**
```
San Francisco: lat=37.7749, lng=-122.4194
→ geohash = "9q8yy"

Precision:
- 3 chars (9q8): ~156 km × 156 km
- 5 chars (9q8yy): ~4.9 km × 4.9 km
- 7 chars (9q8yywe): ~153 m × 153 m
```

**Alternative:** H3 Hexagonal Grid
- ✅ Better boundaries (no edge artifacts)
- ❌ Complex (custom implementation)
- Used by Uber at production scale

### Decision 3: Kafka Buffer

**Chosen:** Async location updates via Kafka

**Why:**
- ✅ Absorbs 750K writes/sec bursts
- ✅ Drivers don't wait for DB writes (5ms response)
- ✅ Fault tolerant (replication)
- ✅ Replayable (audit trail)

**Alternative:** Synchronous Redis writes
- ❌ 50-100ms latency (driver waits)
- ❌ Overload risk (single point of failure)

---

## Geo-Spatial Indexing

### Redis Geo Commands

**1. Add Driver Location:**
```
GEOADD drivers:sf -122.4194 37.7749 "driver:123"
```
- Time: O(log N)
- Internally converts to geohash

**2. Find Nearby Drivers:**
```
GEORADIUS drivers:sf -122.4194 37.7749 5 km WITHDIST COUNT 10
```
- Returns 10 nearest drivers within 5 km
- Sorted by distance
- Time: O(N log N) where N = drivers in area

**3. Update Location (Upsert):**
```
GEOADD drivers:sf -122.4200 37.7755 "driver:123"
```
- Overwrites previous location

**Data Structure:**
```
Key: drivers:{city_code}
Type: Sorted Set
Members: driver:{driver_id}
Score: Geohash integer (52-bit)
```

*See [pseudocode.md::update_driver_location()](pseudocode.md)*

---

## Location Ingestion Pipeline

### Flow

1. **Driver App** sends GPS every 4 seconds
2. **API Gateway** routes to nearest datacenter
3. **Location Service** validates coordinates
4. **Kafka** buffers 750K updates/sec
5. **Indexer Workers** (100 consumers) pull from Kafka
6. **Redis** updated via GEOADD

**Why Async?**

| Approach | Latency | Database Load | Fault Tolerance |
|----------|---------|---------------|-----------------|
| **Sync** | 50-100ms ❌ | 750K direct writes ❌ | Single point ❌ |
| **Async (Kafka)** | 5-10ms ✅ | Sustainable rate ✅ | Replicated ✅ |

**Trade-off:** 200-500ms delay between driver movement and index update (acceptable for 4-second intervals)

---

## Matching Service

### Algorithm

**1. Proximity Search:**
```
GEORADIUS drivers:sf {rider_lat} {rider_lng} 5km COUNT 20
→ [driver:123 (0.5 km), driver:456 (1.2 km), ...]
```

**2. Filter by Availability:**
```
MGET driver:123:status driver:456:status
→ ["available", "on_trip", "available", ...]
Keep only "available"
```

**3. Calculate ETA:**
- Call ETA Service (GraphHopper, OSRM)
- Considers road network, traffic
- Returns travel time (e.g., 7 minutes)

**4. Rank Drivers:**
```
score = 0.7 × distance + 0.2 × ETA + 0.1 × (5 - rating)
Return top 5 drivers
```

**5. Send Requests:**
- Push notifications to top 5 drivers
- First to accept gets trip

**Optimization: Smart Radius Expansion**
- < 5 drivers found in 5 km? → Expand to 10 km
- Still < 5? → Expand to 20 km
- Max radius: 50 km

*See [pseudocode.md::find_nearest_drivers()](pseudocode.md)*

---

## Sharding Strategy

**Problem:** 1M drivers can't fit in single Redis instance

**Solution: Geographic Sharding**

```
Shard by city:
- drivers:sf (San Francisco)
- drivers:nyc (New York)
- drivers:london (London)

API Gateway routes based on rider location:
Rider at lat=37.7749 → Reverse geocode → "sf" → Redis shard "drivers:sf"
```

**Benefits:**
- ✅ Isolation (SF traffic doesn't impact NYC)
- ✅ Scalability (add new cities independently)
- ✅ Low latency (data co-located)

**Challenge: Cross-City Boundaries**
- Solution: Query both adjacent shards, merge results

*See [this-over-that.md: Sharding Strategies](this-over-that.md)*

---

## Driver State Management

**Problem:** Geo-index only stores location, not status

**Solution: Separate State Store**

**Data Model:**
```
Key: driver:{driver_id}:state
Value: {
    "status": "available",  // available, on_trip, offline
    "vehicle_type": "sedan",
    "rating": 4.8,
    "current_trip_id": null
}
TTL: 300 seconds
```

**Operations:**

```
SET driver:123:state {...} EX 300
HMGET driver:123:state driver:456:state
```

**Why Separate?**

| Aspect | Combined | Separated |
|--------|----------|-----------|
| **Update Frequency** | Mixed (bad) | Optimized per use case ✅ |
| **Query Pattern** | Complex | Clean separation ✅ |
| **TTL** | Difficult | Independent ✅ |

---

## Trip State Machine

**States:**
```
REQUESTED → SEARCHING → MATCHED → ARRIVED → IN_PROGRESS → COMPLETED
     ↓          ↓          ↓          ↓           ↓
  CANCELLED  CANCELLED  CANCELLED  CANCELLED  COMPLETED
```

**Database: PostgreSQL (ACID)**

**Why PostgreSQL?**
- Payment processing requires ACID
- Refunds require transactions
- Audit trail requires strong consistency

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
    completed_at TIMESTAMP
);
```

*See [pseudocode.md::trip_state_machine()](pseudocode.md)*

---

## ETA Calculation

**Challenge:** Calculate accurate travel time (not straight-line)

**Solution: GraphHopper / OSRM**

- Open-source routing engine
- Pre-processes OpenStreetMap road network
- Considers traffic, one-way streets, turn restrictions

**API:**
```
GET /route?point=37.7749,-122.4194&point=37.7849,-122.4094
→ {
    "distance": 2500 meters,
    "time": 420 seconds (7 minutes)
}
```

**Caching:**
```
Key: eta:{start_geohash_5}:{end_geohash_5}:{time_bucket}
TTL: 15 minutes
Hit rate: ~60%
```

**Fallback:** Haversine distance × 1.3 (Manhattan factor)

*See [pseudocode.md::calculate_eta()](pseudocode.md)*

---

## Bottlenecks and Solutions

### Bottleneck 1: Redis Write Contention

**Problem:**
- 750K writes/sec overwhelms single Redis
- Need 8-10 instances

**Solutions:**

1. **Geographic Sharding:**
   - Split by city: drivers:sf, drivers:nyc
   - SF: 50K drivers × 0.25 writes/sec = 12.5K writes/sec ✅

2. **Write Batching:**
   - Batch 100 GEOADD commands
   - Single pipeline reduces network roundtrips

3. **Read Replicas:**
   - Master: writes
   - 3 replicas: reads
   - Replication lag: <100ms

*See [pseudocode.md::batch_update_locations()](pseudocode.md)*

---

### Bottleneck 2: Geohash Boundary Issues

**Problem:**
- Driver 10m away but in different geohash cell = missed

**Example:**
```
Rider:    geohash = 9q8yy (SF downtown)
Driver A: geohash = 9q8yy (same cell, 500m) ✅ Found
Driver B: geohash = 9q8yw (adjacent, 50m) ❌ Missed!
```

**Solutions:**

1. **Check Adjacent Cells:**
   - Query center + 8 neighbors (9 total)
   - Merge results
   - Cost: 10ms → 90ms

2. **Larger Search Radius:**
   - 5 km → 7 km (covers boundaries)

3. **H3 Hexagonal Grid:**
   - No sharp corners
   - Better neighbor coverage
   - Uber's solution

*See [hld-diagram.md: Boundary Problem](hld-diagram.md)*

---

### Bottleneck 3: ETA Service Overload

**Problem:**
- 50K searches/sec × 20 drivers = 1M ETA requests/sec
- GraphHopper can't handle 1M requests/sec

**Solutions:**

1. **Caching:**
   - Key: eta:{start}:{end}:{time_bucket}
   - Hit rate: 60%
   - 1M → 400K cache misses

2. **Pre-Computation:**
   - Pre-compute ETA grid for city
   - Lookup: O(1)

3. **Approximation:**
   - Straight-line × 1.3
   - Accurate within 20%

*See [pseudocode.md::calculate_eta_with_cache()](pseudocode.md)*

---

### Bottleneck 4: Cold Start

**Problem:**
- Driver logs in → not in index yet
- First Kafka update → 200-500ms delay
- Rider searches → driver not found

**Solutions:**

1. **Synchronous Initial Update:**
   - Login: Direct Redis write (bypass Kafka)
   - Subsequent: Async via Kafka

2. **Pre-Warm Index:**
   - Use last known location
   - Update with fresh GPS in 4 seconds

3. **Hybrid:**
   - Sync write (100ms response)
   - Also publish to Kafka (audit)

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Polling for Drivers

**Bad:**
```
while not matched:
    search()
    sleep(5 seconds)
```

**Why Bad:**
- Wastes resources
- 5-second latency
- Doesn't scale

**✅ Better:** Event-driven (WebSocket push)

---

### ❌ Anti-Pattern 2: Full History in Redis

**Bad:** Store all location history in Redis

**Why Bad:**
- Memory expensive
- 3B locations/day = 120 GB/day = $120/day

**✅ Better:**
- Redis: Current location only (40 MB)
- Cassandra: Historical data
- S3: Raw logs

---

### ❌ Anti-Pattern 3: No TTL on Driver State

**Bad:**
```
SET driver:123:status "available"  # No TTL
```

**Why Bad:**
- Driver crashes → remains "available" forever
- Ghost drivers in system

**✅ Better:**
```
SET driver:123:status "available" EX 300
Heartbeat every 60s refreshes
```

---

### ❌ Anti-Pattern 4: Single Global Redis

**Bad:** All drivers worldwide → one Redis

**Why Bad:**
- High latency (cross-region queries)
- Bottleneck (750K writes/sec)
- Single point of failure

**✅ Better:** Geographic sharding (drivers:sf, drivers:nyc)

---

## Alternative Approaches

### Alternative 1: Elasticsearch

**Pros:**
- ✅ Rich queries (filter by rating + type + distance)
- ✅ Full-text search
- ✅ Built-in analytics

**Cons:**
- ❌ Slower writes (10K/sec/node vs Redis 100K)
- ❌ Higher latency (20-50ms vs <5ms)

**When:** Need complex filtering

---

### Alternative 2: DynamoDB

**Pros:**
- ✅ Serverless (no management)
- ✅ Auto-scales
- ✅ Durable

**Cons:**
- ❌ No native geo (manual geohash)
- ❌ Cost ($937/day for 750K writes/sec)
- ❌ Latency (10-30ms)

**When:** AWS ecosystem, willing to pay

---

### Alternative 3: QuadTree

**Pros:**
- ✅ Fast (O(log N))
- ✅ No external DB
- ✅ Fine-tuned

**Cons:**
- ❌ Complex (implement tree rebalancing)
- ❌ No persistence
- ❌ Difficult to shard

**When:** Embedded systems (not distributed)

---

## Monitoring

### Critical Metrics

| Metric | Target | Alert |
|--------|--------|-------|
| **Matching Latency P99** | < 100ms | > 200ms 🔴 |
| **Location Update Rate** | 750K/sec | < 500K 🟡 |
| **Redis Memory** | < 80% | > 90% 🔴 |
| **Kafka Lag** | < 1000 | > 10K 🔴 |
| **Match Success Rate** | > 95% | < 90% 🔴 |

### Dashboards

1. **Real-Time Ops:**
   - Active drivers by city (map)
   - Location updates/sec
   - Matching latency

2. **Geo-Index Health:**
   - Redis memory usage
   - GEOADD ops/sec
   - GEORADIUS latency

3. **Trip Funnel:**
   - Requests → Matches → Completions
   - Conversion rates

### Alerts

**🔴 Critical:**
- Matching service down
- Redis cluster down
- Kafka broker failure
- Match rate < 90%

**🟡 Warning:**
- ETA latency > 500ms
- Redis memory > 90%
- Kafka lag > 10K

---

## Trade-offs

### What We Gained ✅

| Benefit | Explanation |
|---------|-------------|
| **Low Latency** | <100ms matching via Redis |
| **High Throughput** | 750K writes/sec via Kafka |
| **Scalability** | Geographic sharding |
| **Availability** | 99.99% with replication |
| **Accuracy** | Geohash proximity search |

### What We Sacrificed ❌

| Trade-off | Impact |
|-----------|--------|
| **Eventual Consistency** | 200-500ms index lag |
| **Complexity** | Kafka + Redis + PostgreSQL |
| **Cost** | Redis RAM expensive |
| **Boundary Issues** | Geohash cell boundaries |
| **No History** | Current location only |

---

## Real-World Implementations

### Uber

- **Geo-Index:** H3 hexagonal grid (not Geohash)
- **Database:** RocksDB (not Redis)
- **Routing:** Custom "Otto" engine
- **Scale:** 5M drivers, 100M riders, 15M trips/day

**Why Custom?**
- Redis too expensive at 5M drivers
- H3: 15% better accuracy than Geohash
- RocksDB: 10× cheaper than Redis

### Lyft

- **Geo-Index:** S2 geometry (Google's spherical indexing)
- **Database:** DynamoDB + Redis hybrid
- **Routing:** Mapbox Directions API
- **Scale:** 2M drivers, 20M riders

### Grab (Southeast Asia)

- **Geo-Index:** Redis Geo (Geohash)
- **Database:** MySQL (sharded by city)
- **Routing:** OSRM (free)
- **Scale:** 9M drivers, 187M users, 8 countries

**Why Different?**
- Cost constraints (stuck with Geohash)
- Team expertise (MySQL over Cassandra)

---

## Advanced Topics

### Surge Pricing

**Algorithm:**
```
For each geohash cell:
  available_drivers = COUNT(available in cell)
  pending_requests = COUNT(unmatched riders)
  ratio = available_drivers / pending_requests
  
  if ratio < 0.5:  surge = 2.0×
  elif ratio < 1.0: surge = 1.5×
  else: surge = 1.0×
```

**Storage:**
```
Key: surge:{city}:{geohash_5}
Value: {"multiplier": 1.5, "updated_at": timestamp}
TTL: 300 seconds
```

### Driver Repositioning

**Problem:** Drivers cluster downtown, demand in suburbs

**Solutions:**
- Heat map showing high-demand areas
- Destination mode (heading home)
- Bonuses: "Drive to Airport, get $5 for next trip"

**ML Model:**
- Predict demand by geohash for next 30 minutes
- Input: historical trips, events, weather
- Output: Probability of ride in cell X at time T

### UberPool (Multi-Pickup)

**Challenge:** Match riders with overlapping routes

**Approach:**
1. Rider A requests: pickup A → dropoff A
2. Check compatible riders: pickup B near A's route
3. Calculate detour: original 10 min → 13 min with B
4. If detour < 30% → suggest pool

---

## Interview Discussion

### Q1: 10× Traffic Growth?

**Answer:**
- Redis: 20 shards → 200 shards
- Kafka: 100 partitions → 1000 partitions
- Regional datacenters: 20 → 50 cities
- Cost: $50K/month → $500K/month

### Q2: Redis Goes Down?

**Answer:**
- Single node: Sentinel promotes replica (<5s)
- Entire shard: Fallback to adjacent cities
- Complete failure: Rebuild from Kafka (10 minutes)
- Mitigation: Multi-AZ deployment

### Q3: Prevent GPS Spoofing?

**Answer:**
- Velocity check (impossible movements)
- Accelerometer cross-check
- Cell tower triangulation
- ML anomaly detection
- Response: warning → suspension → ban

### Q4: Cross-Border Trips?

**Answer:**
- Block by default (geofence)
- Special mode: drivers with permits only
- Higher fare (surcharge)
- Alternative: hand-off at border

### Q5: Optimize for EVs?

**Answer:**
- Track battery level
- Don't assign long trips to low battery
- Route to charging stations
- Incentives: "Charge during off-peak, earn bonus"

---

## Cost Analysis (AWS)

| Component | Specification | Monthly |
|-----------|---------------|---------|
| **Redis** | 20 × r5.large | $23K |
| **Kafka** | 50 × m5.2xlarge | $115K |
| **Compute** | Location + Matching | $400K |
| **PostgreSQL** | 10 × db.r5.4xlarge | $72K |
| **Load Balancers** | 20 × NLB | $3K |
| **Data Transfer** | 10 TB/month | $9K |
| **Total** | | **$622K/month** |

**Annual:** ~$7.5M/year

**Per Driver:** $7.50/year

**Optimized (Reserved + Spot):** ~$350K/month (~$4.2M/year)

---

## References

- [Geohash Deep Dive](../../02-components/2.5-algorithms/2.5.5-geohash-indexing.md)
- [Redis Geo Commands](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)
- [Kafka Streaming](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)
- [PostgreSQL Sharding](../../02-components/2.1-databases/2.1.4-database-scaling.md)

---

For complete details, see the **[Full Design Document](3.3.2-design-uber-ride-matching.md)**.

## Deep Dive: System Flow

### Complete Ride Request Flow

**1. Driver Goes Online:**
```
1. Driver opens app, location services enabled
2. App requests GPS coordinates (lat, lng)
3. POST /driver/online → Location Service
4. Service validates coordinates, publishes to Kafka
5. Indexer Worker consumes, calls GEOADD
6. Redis updates: drivers:sf adds driver:123
7. Driver State updated: status = "available"
```

**2. Rider Requests Ride:**
```
1. Rider enters pickup location (or uses current GPS)
2. POST /ride/request → Matching Service
3. Service calls GEORADIUS (find 20 nearest drivers)
4. Filters by status: MGET driver:*:status
5. Calculates ETA for each available driver
6. Ranks drivers by score
7. Returns top 5 drivers to rider app
```

**3. Driver Acceptance:**
```
1. Push notifications sent to top 5 drivers
2. First driver to accept gets the trip
3. Trip State: REQUESTED → MATCHED
4. Other drivers get "Trip no longer available"
5. Driver State updated: status = "on_trip"
6. Rider receives: "John is on the way (ETA 7 min)"
```

**4. Pickup Phase:**
```
1. Driver navigates to pickup using ETA Service
2. Location updates continue (every 4 seconds)
3. Rider sees real-time driver location on map
4. Driver arrives: Trip State → ARRIVED
5. Rider notified: "Your driver has arrived"
```

**5. In-Progress Phase:**
```
1. Driver starts trip: Trip State → IN_PROGRESS
2. Navigation to dropoff location
3. Location updates tracked
4. Rider sees estimated arrival time
5. Real-time updates if route changes
```

**6. Completion:**
```
1. Driver ends trip at dropoff
2. Trip State → COMPLETED
3. Fare calculation: distance × rate + time × rate + surge
4. Payment processed (credit card charged)
5. Driver State → "available"
6. Both parties rate each other
```

---

## Data Models in Detail

### Driver Data Model

**Location Index (Redis Geo):**
```
GEOADD drivers:sf -122.4194 37.7749 "driver:123"

Stored as:
  Key: drivers:sf
  Type: Sorted Set
  Member: "driver:123"
  Score: geohash integer (e.g., 4060140471971840)
```

**Driver State (Redis Hash):**
```
HSET driver:123:state
  status "available"
  vehicle_type "sedan"
  rating "4.8"
  total_trips "1523"
  acceptance_rate "95"
  current_trip_id ""
  battery_level "85"
  
EXPIRE driver:123:state 300
```

**Driver Profile (PostgreSQL):**
```sql
CREATE TABLE drivers (
    driver_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(255) UNIQUE,
    license_plate VARCHAR(20),
    vehicle_make VARCHAR(100),
    vehicle_model VARCHAR(100),
    vehicle_year INT,
    vehicle_color VARCHAR(50),
    joined_at TIMESTAMP DEFAULT NOW(),
    background_check_status VARCHAR(20),
    rating DECIMAL(3, 2) DEFAULT 5.00,
    total_trips INT DEFAULT 0,
    total_earnings DECIMAL(12, 2) DEFAULT 0
);
```

### Trip Data Model

**Active Trip (Redis):**
```
HSET trip:abc-123
  trip_id "abc-123"
  rider_id "rider:456"
  driver_id "driver:123"
  status "IN_PROGRESS"
  pickup_lat "37.7749"
  pickup_lng "-122.4194"
  dropoff_lat "37.7849"
  dropoff_lng "-122.4094"
  estimated_fare "15.50"
  started_at "2024-10-27T10:30:00Z"
  
EXPIRE trip:abc-123 86400  # 24 hours
```

**Historical Trip (PostgreSQL):**
```sql
CREATE TABLE trips (
    trip_id UUID PRIMARY KEY,
    rider_id UUID NOT NULL,
    driver_id UUID NOT NULL,
    status VARCHAR(20) NOT NULL,
    pickup_lat DECIMAL(10, 8),
    pickup_lng DECIMAL(11, 8),
    pickup_address TEXT,
    dropoff_lat DECIMAL(10, 8),
    dropoff_lng DECIMAL(11, 8),
    dropoff_address TEXT,
    distance_meters INT,
    duration_seconds INT,
    estimated_fare DECIMAL(10, 2),
    actual_fare DECIMAL(10, 2),
    surge_multiplier DECIMAL(3, 2) DEFAULT 1.00,
    payment_method VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    matched_at TIMESTAMP,
    picked_up_at TIMESTAMP,
    completed_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(255),
    rider_rating SMALLINT,
    driver_rating SMALLINT,
    tip_amount DECIMAL(10, 2)
);

CREATE INDEX idx_rider_trips ON trips(rider_id, created_at DESC);
CREATE INDEX idx_driver_trips ON trips(driver_id, created_at DESC);
CREATE INDEX idx_trip_status ON trips(status, created_at);
```

---

## Scaling Considerations

### Horizontal Scaling

**Redis Sharding:**
```
Current: 1M drivers globally
Shard by city: 100 cities × 10K drivers/city

Per-city Redis cluster:
  - Master: Handles writes (GEOADD)
  - 2 Replicas: Handle reads (GEORADIUS)
  - Memory per shard: 400 KB (10K drivers × 40 bytes)
  - Cost per shard: $20/month (r5.large)
  
Total cost: 100 cities × $20 = $2K/month (Redis only)
```

**Kafka Partitioning:**
```
Topic: location-updates
Partitions: 1000 (partitioned by city + driver_id)

Distribution:
  - SF: 50K drivers → 50 partitions
  - NYC: 80K drivers → 80 partitions
  - London: 30K drivers → 30 partitions
  
Consumers: 100 workers × 10 partitions each
Each worker: ~7.5K updates/sec
```

**Database Sharding (PostgreSQL):**
```
Shard by city:
  - trips_sf (San Francisco trips)
  - trips_nyc (New York trips)
  - trips_london (London trips)
  
Each shard: 100K trips/day
Storage: 100K × 1KB = 100 MB/day
Annual: 36 GB/year/city
```

### Vertical Scaling

**Redis Memory:**
```
Current: 1M drivers × 40 bytes = 40 MB
Future: 10M drivers × 40 bytes = 400 MB

Still fits in single r5.2xlarge (64 GB RAM)
Cost: $3.20/hour × 720 hours = $2,304/month
```

**Kafka Broker Sizing:**
```
Ingestion: 750K writes/sec × 200 bytes = 150 MB/sec
Retention: 7 days
Storage: 150 MB/sec × 86,400 sec/day × 7 days = 90 TB

Brokers: 50 × m5.2xlarge (8 vCPU, 32 GB)
Storage: 50 × 2 TB SSD = 100 TB
Cost: $3.20/hour × 50 × 720 = $115K/month
```

---

## Performance Optimization

### Query Optimization

**1. Geo-Query Optimization:**
```
Naive approach:
  GEORADIUS for each rider (50K queries/sec)
  Latency: 5-10ms per query
  Total: 250-500K ms of query time

Optimized:
  - Read replicas (3 replicas)
  - 50K queries distributed across 3 replicas
  - Each replica: 16.7K queries/sec (manageable)
  - Connection pooling (1000 connections per replica)
```

**2. ETA Caching Strategy:**
```
Cache key: eta:{start_geohash}:{end_geohash}:{time_bucket}

Example:
  start: 9q8yy (SF downtown)
  end: 9q8yz (SF north)
  time: 10:30 AM → bucket = 1030
  
  Key: eta:9q8yy:9q8yz:1030
  Value: {"distance": 2500, "time": 420, "cached_at": timestamp}
  TTL: 15 minutes
  
Hit rate: 60% (common routes cached)
Cache misses: 40% × 1M = 400K (query ETA service)
```

**3. Driver State Batching:**
```
Naive: 20 drivers × 20 searches/sec = 400 HGET queries/sec

Optimized: HMGET (batch query)
  HMGET driver:123:state driver:456:state ... (20 drivers)
  Single roundtrip: 1 query vs 20 queries
  Latency: 2ms vs 40ms (20× faster)
```

### Write Optimization

**1. Location Update Batching:**
```
Indexer worker batches updates:
  
  Single updates (slow):
    GEOADD drivers:sf -122.41 37.77 "driver:1"
    GEOADD drivers:sf -122.42 37.78 "driver:2"
    ... (100 updates)
    
  Batched (fast):
    GEOADD drivers:sf 
      -122.41 37.77 "driver:1"
      -122.42 37.78 "driver:2"
      ... (100 pairs)
    
  Improvement: 100 roundtrips → 1 roundtrip
  Latency: 100 × 2ms = 200ms → 2ms (100× faster)
```

**2. Kafka Producer Batching:**
```
Location Service batches messages:
  
  Linger: 10ms (wait up to 10ms before sending)
  Batch size: 100 messages
  
  Result:
    10 drivers update in 10ms window
    → 1 Kafka batch (instead of 10 separate)
    
  Throughput: 10× improvement
  Latency: +10ms acceptable (4-second update interval)
```

---

## Disaster Recovery

### Failure Scenarios

**1. Redis Shard Failure:**
```
Scenario: drivers:sf Redis master crashes

Detection:
  - Redis Sentinel monitors master (heartbeat)
  - Timeout after 5 seconds
  
Recovery:
  - Sentinel promotes replica to master
  - Clients redirect to new master
  - Total downtime: <5 seconds
  
Impact:
  - SF riders experience 5-second matching delay
  - Other cities unaffected (isolation)
```

**2. Kafka Broker Failure:**
```
Scenario: 1 of 50 brokers crashes

Detection:
  - ZooKeeper monitors broker liveness
  - Timeout after 10 seconds
  
Recovery:
  - Kafka rebalances partitions across remaining 49 brokers
  - Consumers reconnect to new partition leaders
  - Total disruption: 10-30 seconds
  
Impact:
  - Some location updates delayed by 30 seconds
  - No data loss (replication factor = 3)
```

**3. Complete Datacenter Failure:**
```
Scenario: SF datacenter goes offline

Detection:
  - Load balancer health checks fail
  - GeoDNS detects datacenter down
  
Recovery:
  - GeoDNS routes SF traffic to nearest datacenter (Oakland)
  - Oakland datacenter has replicated data
  - Cold index: Rebuild from Kafka replay
  
Impact:
  - 2-5 minutes downtime for SF region
  - Higher latency (cross-datacenter queries)
  
Mitigation:
  - Multi-region deployment
  - Cross-region Kafka replication
  - Async data sync between datacenters
```

---

## Security Considerations

### Authentication & Authorization

**1. Driver Authentication:**
```
JWT Token Structure:
{
  "driver_id": "driver:123",
  "session_id": "session-abc-456",
  "issued_at": 1698412800,
  "expires_at": 1698416400,  // 1 hour validity
  "scopes": ["update_location", "accept_trip", "complete_trip"]
}

Validation:
  - API Gateway validates JWT signature
  - Check expiration (expires_at > now())
  - Verify driver is active (not banned)
```

**2. Location Data Encryption:**
```
In-transit: TLS 1.3 (HTTPS)
At-rest: 
  - Redis: No native encryption (trust network security)
  - PostgreSQL: Transparent Data Encryption (TDE)
  - Kafka: Encryption at rest (AES-256)
```

**3. Rate Limiting:**
```
Per driver:
  - Location updates: 1 per 4 seconds (max 15/minute)
  - Exceed limit: 429 Too Many Requests
  
Per rider:
  - Search requests: 10 per minute
  - Trip requests: 5 per hour (prevent abuse)
```

### Privacy

**1. Location Data Retention:**
```
Real-time index (Redis): Current location only, TTL = 5 minutes
Trip history (PostgreSQL): 7 years (regulatory requirement)
Raw GPS logs (S3): 90 days, then deleted
```

**2. Anonymization:**
```
For analytics:
  - Hash driver_id: SHA256(driver_id + salt)
  - Round coordinates: lat=37.774922 → 37.77
  - Remove personally identifiable information (PII)
```

---

## Testing Strategy

### Load Testing

**Scenario: Rush Hour Traffic**
```
Simulate:
  - 750K location updates/sec (peak)
  - 50K ride requests/sec
  - Duration: 2 hours
  
Tools:
  - Locust (Python) for HTTP load
  - Kafka performance tool for producer load
  
Metrics:
  - Matching latency P99 < 100ms ✅
  - Redis memory < 80% ✅
  - Kafka lag < 1000 messages ✅
  - Error rate < 0.1% ✅
```

### Chaos Engineering

**Inject Failures:**
```
1. Kill random Redis node (test failover)
2. Network partition (split brain scenario)
3. Kafka broker crash (test rebalancing)
4. ETA service timeout (test fallback)
5. Database connection pool exhaustion
```

### Integration Testing

**End-to-End Flows:**
```
Test 1: Complete trip flow
  1. Driver goes online
  2. Rider requests ride
  3. Driver accepts
  4. Driver navigates to pickup
  5. Trip starts
  6. Driver completes trip
  7. Payment processed
  
  Assert: All state transitions correct, no data loss
```

---

## Migration Strategy

### From Monolith to Microservices

**Phase 1: Extract Location Service**
```
Before: Monolith handles everything
After: 
  - Monolith → Location Service (new)
  - Monolith → Kafka → Location Service
  - Gradual traffic migration (10% → 50% → 100%)
```

**Phase 2: Separate Matching**
```
Before: Monolith does matching
After:
  - Matching Service (new) with Redis Geo
  - Monolith forwards ride requests
  - A/B test: 50% monolith, 50% new service
```

**Phase 3: Extract Trip Management**
```
Before: Monolith handles trips
After:
  - Trip Service (new) with PostgreSQL
  - Event-driven: Matching → Kafka → Trip Service
  - Rollback plan: Feature flag to revert
```

### Data Migration

**Migrate from PostgreSQL to Redis Geo:**
```
1. Export driver locations from PostgreSQL
2. Bulk load into Redis:
   GEOADD drivers:sf $(cat locations.txt)
3. Run both systems in parallel (1 week)
4. Compare results: Redis vs PostgreSQL
5. Cut over: Point traffic to Redis
6. Monitor for anomalies
7. Rollback if errors > 0.1%
```

---

## Summary

**Uber/Lyft Ride Matching** requires careful balance of:

- **Performance:** <100ms matching via in-memory indexing
- **Scale:** 750K writes/sec via async processing
- **Accuracy:** Geohash proximity search
- **Cost:** $4-7M/year infrastructure
- **Complexity:** Multiple data stores, careful orchestration

**Key Technologies:**
- Redis Geo (geohash indexing)
- Kafka (location update buffer)
- PostgreSQL (ACID transactions)
- GraphHopper/OSRM (ETA calculation)

**Production Lessons from Uber:**
- Start with Geohash, migrate to H3 at scale
- Custom database (RocksDB) cheaper than Redis
- Multi-region essential for global latency
- Monitoring critical (match rate, latency, accuracy)

---

For **complete technical details**, see the [Full Design Document](3.3.2-design-uber-ride-matching.md).

For **visual architecture**, see [HLD Diagrams](hld-diagram.md) and [Sequence Diagrams](sequence-diagrams.md).

For **implementation code**, see [Pseudocode](pseudocode.md).

For **design rationale**, see [This Over That](this-over-that.md).
