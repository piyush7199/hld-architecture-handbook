# Uber/Lyft Ride Matching System - Sequence Diagrams

## Table of Contents

1. [Driver Location Update Flow](#1-driver-location-update-flow)
2. [Rider Search and Matching Flow](#2-rider-search-and-matching-flow)
3. [Trip Acceptance Flow (First Driver Wins)](#3-trip-acceptance-flow-first-driver-wins)
4. [Trip Lifecycle - Complete Flow](#4-trip-lifecycle---complete-flow)
5. [ETA Calculation Flow](#5-eta-calculation-flow)
6. [Surge Pricing Activation Flow](#6-surge-pricing-activation-flow)
7. [Driver Heartbeat and Timeout Flow](#7-driver-heartbeat-and-timeout-flow)
8. [Geohash Boundary Query Flow](#8-geohash-boundary-query-flow)
9. [Redis Cluster Failover Flow](#9-redis-cluster-failover-flow)
10. [Kafka Consumer Lag Recovery Flow](#10-kafka-consumer-lag-recovery-flow)
11. [Cross-City Trip Handoff Flow](#11-cross-city-trip-handoff-flow)
12. [Payment Processing with Retry Flow](#12-payment-processing-with-retry-flow)

---

## 1. Driver Location Update Flow

**Flow:**

Shows the asynchronous pipeline for processing a driver's GPS update, from mobile app to Redis geo-index.

**Steps:**
1. **Driver App** (0ms): GPS sensor detects new coordinates (lat, lng)
2. **HTTPS POST** (10ms): Send location to API Gateway with auth token
3. **API Gateway** (5ms): Validate JWT, rate limit check, route by city
4. **Location Service** (10ms): Validate coordinates (valid lat/lng range), enrich with timestamp
5. **Kafka Publish** (5ms): Publish to "location-updates" topic, return 200 OK immediately
6. **Driver receives ACK** (30ms total): Driver app doesn't wait for DB write
7. **Kafka Replication** (async, 50-100ms): Replicate to 3 brokers for durability
8. **Indexer Worker** (100-500ms): Consumer pulls message from Kafka partition
9. **Redis GEOADD** (10ms): Update driver position in geo-index
10. **Status Update** (5ms): Update driver status in separate Redis store

**Performance:**
- **Driver response time:** 30ms (fire-and-forget)
- **End-to-end latency:** 200-500ms (update visible to riders)
- **Throughput:** 750K updates/sec buffered by Kafka

**Benefits:**
- Non-blocking for driver (instant ACK)
- Kafka absorbs traffic spikes
- Decoupled pipeline (failure isolation)

**Trade-offs:**
- Eventual consistency (slight delay)
- Kafka operational complexity

```mermaid
sequenceDiagram
    participant Driver as Driver App
    participant Gateway as API Gateway
    participant LocationSvc as Location Service
    participant Kafka as Kafka Cluster
    participant Indexer as Indexer Worker
    participant RedisGeo as Redis Geo Index
    participant RedisState as Redis State Store
    
    Note over Driver: GPS update every 4 seconds
    Driver->>Gateway: POST /v1/location<br/>{lat: 37.7749, lng: -122.4194, driver_id: 123}
    Note right of Driver: Include JWT token
    
    Gateway->>Gateway: Validate JWT<br/>Rate limit: 1 req/4s<br/>Route by city: SF
    Gateway->>LocationSvc: Forward request
    
    LocationSvc->>LocationSvc: Validate coordinates<br/>-90 ≤ lat ≤ 90<br/>-180 ≤ lng ≤ 180
    LocationSvc->>LocationSvc: Enrich data<br/>timestamp, city_code
    
    LocationSvc->>Kafka: Publish to topic: location-updates<br/>Partition by driver_id
    Note right of Kafka: Fire-and-forget<br/>Async write
    
    LocationSvc-->>Driver: 200 OK (30ms total)
    Note left of Driver: Driver doesn't wait<br/>for DB write
    
    Kafka->>Kafka: Replicate to 3 brokers<br/>(50-100ms async)
    
    Indexer->>Kafka: Pull batch (100 messages)
    Note right of Indexer: Consumer group<br/>100 workers
    
    Kafka-->>Indexer: Batch of 100 updates
    
    loop For each update
        Indexer->>RedisGeo: GEOADD drivers:sf<br/>-122.4194 37.7749 "driver:123"
        RedisGeo-->>Indexer: OK
        
        Indexer->>RedisState: SET driver:123:status<br/>{status: "available", updated_at: now}
        RedisState-->>Indexer: OK
    end
    
    Note over RedisGeo,RedisState: Location now visible to riders<br/>Total latency: 200-500ms
```

---

## 2. Rider Search and Matching Flow

**Flow:**

Shows the complete flow of a rider requesting a ride, from opening the app to receiving top 5 driver options.

**Steps:**
1. **Rider opens app** (0ms): App loads current GPS location
2. **Search request** (10ms): POST /v1/rides/search with pickup location
3. **Matching Service** (0ms): Receives request, starts matching algorithm
4. **GEORADIUS query** (10ms): Find 20 nearest drivers within 5km radius
5. **Filter availability** (5ms): Query driver status, remove busy/offline (15 available)
6. **Calculate ETAs** (50ms): For each driver, call ETA service (parallel batch request)
7. **Rank drivers** (5ms): Score = 0.7×distance + 0.2×ETA + 0.1×rating
8. **Select top 5** (1ms): Return best 5 drivers to rider
9. **Display on map** (100ms total): Rider sees 5 drivers with ETAs

**Performance:**
- **Total latency:** 80-100ms (p99)
- **Breakdown:** Redis (10ms) + Filter (5ms) + ETA (50ms) + Ranking (5ms)

**Benefits:**
- Fast matching prevents rider abandonment
- Parallel ETA calculation reduces latency
- Multiple drivers increase acceptance rate

**Trade-offs:**
- ETA service is bottleneck (50ms)
- 15 ETA calculations per search (expensive)

```mermaid
sequenceDiagram
    participant Rider as Rider App
    participant Gateway as API Gateway
    participant MatchSvc as Matching Service
    participant RedisGeo as Redis Geo Index
    participant DriverState as Driver State Store
    participant ETASvc as ETA Service
    participant RankEngine as Ranking Engine
    
    Rider->>Gateway: POST /v1/rides/search<br/>{pickup: {lat, lng}, vehicle_type: "UberX"}
    Note right of Rider: Rider location:<br/>37.7749, -122.4194
    
    Gateway->>MatchSvc: Forward search request
    
    MatchSvc->>RedisGeo: GEORADIUS drivers:sf<br/>-122.4194 37.7749 5km<br/>WITHDIST COUNT 20
    Note right of RedisGeo: O(N log N)<br/>N = drivers in area
    
    RedisGeo-->>MatchSvc: 20 drivers with distances<br/>[{driver:123, dist:0.5km}, ...]
    
    MatchSvc->>DriverState: MGET driver:123:status<br/>driver:456:status<br/>... (20 keys)
    Note right of DriverState: Batch read<br/>5ms total
    
    DriverState-->>MatchSvc: 15 available, 5 busy
    Note left of MatchSvc: Filter out busy drivers
    
    MatchSvc->>ETASvc: Batch ETA request<br/>15 driver-rider pairs
    Note right of ETASvc: Parallel calculation<br/>50ms total
    
    ETASvc-->>MatchSvc: 15 ETAs<br/>[{driver:123, eta:3.2min}, ...]
    
    MatchSvc->>RankEngine: Rank drivers<br/>score = 0.7×dist + 0.2×eta + 0.1×rating
    
    RankEngine-->>MatchSvc: Sorted list (best to worst)
    
    MatchSvc->>MatchSvc: Select top 5 drivers
    
    MatchSvc-->>Rider: 200 OK<br/>5 drivers with ETAs<br/>(100ms total)
    
    Note over Rider: Display 5 drivers on map<br/>User can request ride
```

---

## 3. Trip Acceptance Flow (First Driver Wins)

**Flow:**

Shows the race condition when 5 drivers are notified simultaneously, and only the first to accept gets the trip.

**Steps:**
1. **Rider confirms ride** (0ms): Rider taps "Request UberX"
2. **Create pending trip** (20ms): Insert trip record in PostgreSQL with status=PENDING
3. **Broadcast to 5 drivers** (50ms): Send push notifications via FCM/APNs
4. **Drivers receive notification** (100-500ms): Variable latency due to mobile network
5. **Driver 1 accepts first** (200ms): Taps "Accept" button
6. **Atomic CAS operation** (10ms): UPDATE trips SET status=ACCEPTED, driver_id=123 WHERE trip_id=456 AND status=PENDING
7. **CAS succeeds for Driver 1** (success): Trip assigned to Driver 1
8. **Drivers 2-5 accept** (220-300ms): Slightly slower
9. **CAS fails for Drivers 2-5** (fail): status already ACCEPTED, no update
10. **Notify Driver 1** (success): "Trip assigned! Navigate to pickup."
11. **Notify Drivers 2-5** (fail): "Trip already taken"

**Performance:**
- **First driver response:** 200ms
- **Total acceptance window:** 30 seconds (timeout)

**Benefits:**
- Atomic operation prevents double-assignment
- Fair: Fastest driver wins
- High acceptance rate (5 drivers = 80% chance)

**Trade-offs:**
- 4 wasted notifications per trip
- Drivers frustrated by missed opportunities

```mermaid
sequenceDiagram
    participant Rider as Rider App
    participant MatchSvc as Matching Service
    participant TripDB as PostgreSQL
    participant NotifySvc as Notification Service
    participant Driver1 as Driver 1 (Fastest)
    participant Driver2 as Driver 2
    participant Driver3 as Driver 3
    
    Rider->>MatchSvc: POST /v1/rides/request<br/>{pickup, destination, vehicle_type}
    
    MatchSvc->>TripDB: INSERT INTO trips<br/>(trip_id, rider_id, status=PENDING)
    Note right of TripDB: ACID transaction<br/>20ms
    
    TripDB-->>MatchSvc: trip_id: 789
    
    MatchSvc->>NotifySvc: Broadcast to 5 drivers<br/>[driver:123, driver:456, ...]
    
    par Parallel notifications
        NotifySvc->>Driver1: Push notification<br/>"New ride request!"
        NotifySvc->>Driver2: Push notification
        NotifySvc->>Driver3: Push notification
    end
    
    Note over Driver1,Driver3: Network latency varies<br/>100-500ms
    
    Driver1->>MatchSvc: POST /v1/trips/789/accept<br/>(200ms - fastest)
    
    MatchSvc->>TripDB: UPDATE trips<br/>SET status=ACCEPTED, driver_id=123<br/>WHERE trip_id=789 AND status=PENDING
    Note right of TripDB: Atomic CAS<br/>Only one UPDATE succeeds
    
    TripDB-->>MatchSvc: 1 row updated (SUCCESS)
    
    MatchSvc-->>Driver1: 200 OK<br/>"Trip assigned! Navigate to pickup."
    
    Driver2->>MatchSvc: POST /v1/trips/789/accept<br/>(220ms - too late)
    
    MatchSvc->>TripDB: UPDATE trips<br/>SET status=ACCEPTED, driver_id=456<br/>WHERE trip_id=789 AND status=PENDING
    
    TripDB-->>MatchSvc: 0 rows updated (FAIL)
    Note left of TripDB: status already ACCEPTED
    
    MatchSvc-->>Driver2: 409 Conflict<br/>"Trip already taken"
    
    Driver3->>MatchSvc: POST /v1/trips/789/accept<br/>(300ms - too late)
    
    MatchSvc->>TripDB: Same CAS attempt
    TripDB-->>MatchSvc: 0 rows updated (FAIL)
    
    MatchSvc-->>Driver3: 409 Conflict<br/>"Trip already taken"
    
    Note over Driver1: Winner! Navigate to pickup
    Note over Driver2,Driver3: Lost race, continue waiting
```

---

## 4. Trip Lifecycle - Complete Flow

**Flow:**

Shows the complete end-to-end trip lifecycle from request to payment completion.

**Steps:**
1. **REQUESTED** (0s): Rider submits ride request
2. **SEARCHING** (0-2s): Matching service finds drivers
3. **PENDING** (2-30s): Drivers notified, waiting for acceptance
4. **ACCEPTED** (30s): Driver 1 accepts, navigates to pickup
5. **ARRIVED** (5-10min): Driver reaches pickup location
6. **ACTIVE** (10-30min): Rider in vehicle, trip in progress
7. **COMPLETED** (30min): Driver ends trip at destination
8. **PAID** (30.5min): Payment processed successfully

**Performance:**
- **Average trip duration:** 15 minutes
- **Payment processing:** 10-30 seconds
- **Total lifecycle:** 20-40 minutes

**Benefits:**
- Clear state transitions
- Easy to debug and monitor
- Idempotent state machine

**Trade-offs:**
- Many edge cases (no-show, cancellation, payment failure)
- Complex error handling

```mermaid
sequenceDiagram
    participant Rider as Rider App
    participant Driver as Driver App
    participant MatchSvc as Matching Service
    participant TripDB as PostgreSQL
    participant PaymentSvc as Payment Service
    
    Note over Rider,PaymentSvc: Trip Lifecycle: 20-40 minutes total
    
    Rider->>MatchSvc: Request ride
    MatchSvc->>TripDB: INSERT trip (status=REQUESTED)
    TripDB-->>Rider: Trip created
    
    Note over MatchSvc: State: REQUESTED → SEARCHING
    MatchSvc->>MatchSvc: Find nearby drivers
    MatchSvc->>TripDB: UPDATE status=PENDING
    
    MatchSvc->>Driver: Push notification (5 drivers)
    Note over Driver: Driver decides: Accept or Reject
    
    Driver->>MatchSvc: Accept trip
    MatchSvc->>TripDB: UPDATE status=ACCEPTED<br/>WHERE status=PENDING (CAS)
    TripDB-->>Driver: Trip assigned
    
    Note over Driver: State: ACCEPTED<br/>Driver navigates to pickup<br/>5-10 minutes
    
    Driver->>Driver: GPS detects: Arrived at pickup
    Driver->>MatchSvc: POST /trips/789/arrived
    MatchSvc->>TripDB: UPDATE status=ARRIVED
    TripDB-->>Rider: Notification: "Driver arrived"
    
    Note over Rider: Rider enters vehicle
    Rider->>MatchSvc: Confirm pickup
    MatchSvc->>TripDB: UPDATE status=ACTIVE
    
    Note over Driver,Rider: State: ACTIVE<br/>Trip in progress<br/>GPS tracking every 2s<br/>10-30 minutes
    
    Driver->>Driver: GPS detects: Arrived at destination
    Driver->>MatchSvc: POST /trips/789/complete
    MatchSvc->>TripDB: UPDATE status=COMPLETED
    
    Note over PaymentSvc: State: COMPLETED → PAID<br/>Process payment
    
    MatchSvc->>PaymentSvc: Charge rider $15.50
    PaymentSvc->>PaymentSvc: Stripe API call
    PaymentSvc-->>MatchSvc: Payment successful
    
    MatchSvc->>TripDB: UPDATE status=PAID
    TripDB-->>Rider: Receipt: $15.50
    TripDB-->>Driver: Earnings: $12.50
    
    Note over Rider,Driver: Trip complete!<br/>Both can rate each other
```

---

## 5. ETA Calculation Flow

**Flow:**

Shows how ETA is calculated using road network graphs and real-time traffic data.

**Steps:**
1. **Matching Service** (0ms): Needs ETA for 15 driver-rider pairs
2. **Batch request** (5ms): Send all 15 pairs to ETA Service
3. **Cache check** (5ms): Query Redis cache for recent ETAs (8 hits, 7 misses)
4. **Return cached** (10ms): Return 8 cached ETAs immediately
5. **Calculate new** (50ms): For 7 misses, query OSRM routing engine
6. **OSRM pathfinding** (30ms): A* algorithm finds shortest path on road graph
7. **Traffic adjustment** (10ms): Adjust edge weights based on current traffic speeds
8. **Total time calculation** (5ms): Sum edge weights, convert to minutes
9. **Cache result** (5ms): Store in Redis with TTL=5 minutes
10. **Return all 15 ETAs** (60ms): Merge cached + calculated ETAs

**Performance:**
- **Cache hit rate:** 60% (8 out of 15)
- **Cached ETA latency:** 10ms
- **Calculated ETA latency:** 50ms
- **Average latency:** (8×10 + 7×50) / 15 = 28ms

**Benefits:**
- Cache reduces load on OSRM
- Batch requests improve efficiency
- Traffic-aware ETAs are accurate

**Trade-offs:**
- OSRM is single point of failure
- Cold start (new region) is slow
- Traffic data costs money

```mermaid
sequenceDiagram
    participant MatchSvc as Matching Service
    participant ETASvc as ETA Service
    participant Cache as Redis Cache
    participant OSRM as OSRM Routing Engine
    participant TrafficSvc as Traffic Service
    participant GraphDB as Road Network Graph
    
    MatchSvc->>ETASvc: Batch ETA request<br/>15 driver-rider pairs
    Note right of MatchSvc: [(driver:123, rider:456),<br/>(driver:789, rider:456), ...]
    
    ETASvc->>Cache: MGET eta:{hash1} eta:{hash2} ...<br/>(15 cache keys)
    Note right of Cache: Key = hash(driver_lat,<br/>driver_lng, rider_lat, rider_lng)
    
    Cache-->>ETASvc: 8 cache hits, 7 misses
    
    Note over ETASvc: Cache hits: Return immediately<br/>Cache misses: Calculate
    
    loop For each of 7 misses
        ETASvc->>OSRM: GET /route/v1/driving<br/>coordinates: driver to rider
        
        OSRM->>GraphDB: Query road network<br/>A* pathfinding
        Note right of GraphDB: 100M+ road nodes<br/>Pre-processed OSM
        
        GraphDB-->>OSRM: Shortest path<br/>[edge1, edge2, ...]
        
        OSRM->>TrafficSvc: Get current speeds<br/>for edges
        
        TrafficSvc-->>OSRM: Traffic speeds<br/>Highway: 45 mph<br/>City: 15 mph
        
        OSRM->>OSRM: Calculate total time<br/>Sum: distance / speed
        Note left of OSRM: Edge1: 2 mi / 45 mph = 2.7 min<br/>Edge2: 1 mi / 15 mph = 4 min<br/>Total: 6.7 min
        
        OSRM-->>ETASvc: ETA: 6.7 minutes
        
        ETASvc->>Cache: SET eta:{hash}<br/>Value: 6.7<br/>TTL: 300s (5 min)
    end
    
    ETASvc->>ETASvc: Merge results<br/>8 cached + 7 calculated
    
    ETASvc-->>MatchSvc: All 15 ETAs<br/>(60ms total)
    
    Note over MatchSvc: Proceed to ranking
```

---

## 6. Surge Pricing Activation Flow

**Flow:**

Shows how surge pricing is dynamically calculated based on supply/demand ratio in each geohash cell.

**Steps:**
1. **Timer triggers** (every 30 seconds): Surge calculator runs
2. **Count available drivers** (10ms): Query Redis for drivers in geohash "9q8yy"
3. **Count pending requests** (10ms): Query PostgreSQL for pending trips in same cell
4. **Calculate ratio** (1ms): demand / supply = 50 requests / 20 drivers = 2.5
5. **Apply multiplier** (1ms): Ratio 2.5 → 2.0× surge (high surge bracket)
6. **Update cache** (5ms): SET surge:9q8yy = 2.0 with TTL=60s
7. **Notify riders** (100ms): WebSocket broadcast to all riders in cell
8. **Display surge** (0ms): Rider app shows "Prices are 2× higher due to increased demand"

**Performance:**
- **Calculation frequency:** Every 30 seconds per geohash
- **Total calculation time:** 30ms per cell
- **Cells monitored:** ~10,000 cells globally

**Benefits:**
- Balances supply and demand
- Incentivizes drivers to go to high-demand areas
- Revenue optimization

**Trade-offs:**
- Rider frustration with high prices
- PR risk ("price gouging")
- Drivers may game system

```mermaid
sequenceDiagram
    participant Timer as Cron Timer
    participant SurgeSvc as Surge Pricing Service
    participant RedisGeo as Redis Geo Index
    participant TripDB as PostgreSQL
    participant RedisCache as Redis Cache
    participant NotifySvc as Notification Service
    participant RiderApp as Rider Apps (WebSocket)
    
    Note over Timer: Every 30 seconds
    Timer->>SurgeSvc: Trigger surge calculation<br/>for all geohash cells
    
    loop For each geohash cell
        SurgeSvc->>RedisGeo: Count available drivers<br/>ZCOUNT drivers:sf:9q8yy
        RedisGeo-->>SurgeSvc: 20 available drivers
        
        SurgeSvc->>TripDB: SELECT COUNT(*)<br/>FROM trips<br/>WHERE geohash='9q8yy'<br/>AND status='PENDING'
        TripDB-->>SurgeSvc: 50 pending requests
        
        SurgeSvc->>SurgeSvc: Calculate ratio<br/>demand / supply = 50 / 20 = 2.5
        
        SurgeSvc->>SurgeSvc: Apply multiplier logic
        Note right of SurgeSvc: Ratio 2.5 falls in<br/>2.0-5.0 bracket<br/>Surge = 2.0×
        
        SurgeSvc->>RedisCache: SET surge:9q8yy<br/>Value: 2.0<br/>TTL: 60s
        RedisCache-->>SurgeSvc: OK
        
        SurgeSvc->>NotifySvc: Broadcast surge update<br/>cell: 9q8yy, multiplier: 2.0
        
        NotifySvc->>RiderApp: WebSocket push<br/>"Prices are 2× higher due to demand"
        Note right of RiderApp: Only riders in<br/>geohash 9q8yy receive
        
        RiderApp->>RiderApp: Update UI<br/>Show surge indicator
    end
    
    Note over Timer,RiderApp: Surge pricing active!<br/>Base fare: $10 → Surge fare: $20
```

---

## 7. Driver Heartbeat and Timeout Flow

**Flow:**

Shows how the system detects crashed or disconnected drivers using heartbeat timeouts.

**Steps:**
1. **Driver online** (0s): Driver logs in, status=available
2. **Heartbeat sent** (every 4s): Driver sends location + heartbeat
3. **Update TTL** (4s): Redis key driver:123:status has TTL=60s, refresh on each heartbeat
4. **Network issue** (12s): Driver loses connection, no heartbeat for 60 seconds
5. **TTL expires** (72s): Redis automatically deletes driver:123:status key
6. **Cleanup job** (75s): Background job detects expired driver
7. **Remove from geo-index** (80s): ZREM drivers:sf driver:123
8. **Driver reappears** (100s): Network restored, driver sends heartbeat
9. **Re-add to index** (105s): GEOADD drivers:sf, driver visible again

**Performance:**
- **Heartbeat interval:** 4 seconds
- **Timeout threshold:** 60 seconds (15 missed heartbeats)
- **Cleanup delay:** 5-10 seconds after expiry

**Benefits:**
- Auto-cleanup of crashed drivers
- No manual intervention needed
- Riders don't see offline drivers

**Trade-offs:**
- 60-second delay before removal
- False positives during brief network issues
- Driver must re-authenticate after timeout

```mermaid
sequenceDiagram
    participant Driver as Driver App
    participant Gateway as API Gateway
    participant LocationSvc as Location Service
    participant RedisState as Redis State Store
    participant RedisGeo as Redis Geo Index
    participant CleanupJob as Cleanup Background Job
    
    Note over Driver: Driver logs in
    Driver->>Gateway: POST /v1/drivers/online
    Gateway->>RedisState: SET driver:123:status<br/>{status: "available"}<br/>TTL: 60s
    RedisState-->>Driver: 200 OK
    
    loop Every 4 seconds (normal operation)
        Driver->>LocationSvc: POST /v1/location<br/>{lat, lng, heartbeat}
        LocationSvc->>RedisState: SET driver:123:status<br/>EXPIRE 60<br/>(refresh TTL)
        Note right of RedisState: TTL reset to 60s<br/>Driver still alive
    end
    
    Note over Driver: Network issue!<br/>Driver disconnected
    
    Note over Driver,RedisState: 60 seconds pass<br/>No heartbeat received
    
    Note over RedisState: TTL expires<br/>Key auto-deleted
    
    CleanupJob->>RedisState: SCAN for expired drivers<br/>(every 10 seconds)
    RedisState-->>CleanupJob: driver:123 missing
    
    CleanupJob->>RedisGeo: ZREM drivers:sf driver:123
    Note right of RedisGeo: Driver removed from geo-index<br/>No longer visible to riders
    
    RedisGeo-->>CleanupJob: 1 driver removed
    
    Note over Driver: Network restored!<br/>Driver reconnects
    
    Driver->>LocationSvc: POST /v1/location<br/>{lat, lng, heartbeat}
    LocationSvc->>RedisState: SET driver:123:status<br/>TTL: 60s
    LocationSvc->>RedisGeo: GEOADD drivers:sf<br/>-122.4194 37.7749 driver:123
    
    Note over RedisGeo: Driver visible again!<br/>Back in geo-index
    
    LocationSvc-->>Driver: 200 OK<br/>Re-added to index
```

---

## 8. Geohash Boundary Query Flow

**Flow:**

Shows how to handle the edge case where a rider is near a geohash cell boundary, requiring queries to multiple adjacent cells.

**Steps:**
1. **Rider location** (0ms): Rider at lat=37.7749, lng=-122.4194 (cell "9q8yy9")
2. **Detect boundary** (5ms): Calculate distance to cell edges (150m to north edge)
3. **Query primary cell** (10ms): GEORADIUS in cell "9q8yy9" → 8 drivers found
4. **Insufficient results** (check): Need 20 drivers, only have 8
5. **Query adjacent cells** (30ms): Query 8 neighboring cells in parallel
6. **Merge results** (10ms): Combine all drivers from 9 cells → 45 drivers total
7. **Calculate exact distance** (20ms): Haversine formula for each driver
8. **Sort by distance** (5ms): Sort all 45 drivers
9. **Return top 20** (1ms): Select 20 nearest drivers

**Performance:**
- **Single cell query:** 10ms (fast)
- **Multi-cell query:** 80ms (slower but necessary)
- **Frequency:** 15% of queries are near boundaries

**Benefits:**
- No missed drivers near boundaries
- Complete coverage of search radius
- Accurate results

**Trade-offs:**
- 8× more Redis queries (expensive)
- Higher latency for boundary queries
- Increased Redis load

```mermaid
sequenceDiagram
    participant Rider as Rider App
    participant MatchSvc as Matching Service
    participant GeoCalc as Geohash Calculator
    participant RedisGeo as Redis Geo Index
    participant Merger as Result Merger
    
    Rider->>MatchSvc: Search for drivers<br/>lat: 37.7749, lng: -122.4194
    
    MatchSvc->>GeoCalc: Calculate geohash<br/>precision: 6 chars
    GeoCalc-->>MatchSvc: Primary cell: "9q8yy9"
    
    MatchSvc->>GeoCalc: Check distance to cell edges
    GeoCalc-->>MatchSvc: 150m to north edge<br/>400m to east edge<br/>(Close to boundary!)
    
    Note over MatchSvc: Distance to edge < 1km<br/>Query adjacent cells
    
    MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yy9<br/>5km radius
    RedisGeo-->>MatchSvc: 8 drivers
    
    par Query 8 adjacent cells in parallel
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yy8
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yyb
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yyc
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yy3
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yy6
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yy4
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yy5
        MatchSvc->>RedisGeo: GEORADIUS drivers:sf:9q8yyd
    end
    
    Note over RedisGeo: Return partial results<br/>from each cell
    
    RedisGeo-->>Merger: Cell 1: 8 drivers<br/>Cell 2: 5 drivers<br/>...<br/>Cell 9: 3 drivers
    
    Merger->>Merger: Merge all results<br/>Remove duplicates<br/>Total: 45 unique drivers
    
    Merger->>Merger: Calculate exact distance<br/>Haversine formula<br/>for all 45 drivers
    
    Merger->>Merger: Sort by distance<br/>ascending order
    
    Merger->>Merger: Select top 20 drivers
    
    Merger-->>MatchSvc: 20 nearest drivers
    
    MatchSvc-->>Rider: Display 20 drivers on map<br/>(80ms total)
    
    Note over Rider: Complete coverage!<br/>No drivers missed
```

---

## 9. Redis Cluster Failover Flow

**Flow:**

Shows how the system handles Redis master node failure with automatic failover to replica.

**Steps:**
1. **Normal operation** (0s): Master node serving 250K writes/sec
2. **Master crashes** (10s): Master node becomes unresponsive
3. **Sentinel detects** (13s): Sentinel pings master, no response (3s timeout)
4. **Quorum vote** (15s): 3 Sentinels vote to promote replica
5. **Promote replica** (18s): Replica promoted to master
6. **Clients reconnect** (20s): Indexer workers detect master change, reconnect
7. **Resume writes** (25s): Writes resume to new master
8. **Data loss** (check): 0-100ms of writes lost (async replication lag)

**Performance:**
- **Detection time:** 3 seconds (Sentinel ping timeout)
- **Failover time:** 5 seconds (vote + promote)
- **Total downtime:** 8 seconds
- **Data loss:** 0-100ms of writes (acceptable)

**Benefits:**
- Automatic failover (no manual intervention)
- High availability (99.99%)
- Fast recovery

**Trade-offs:**
- 8-second write downtime
- Potential data loss (async replication)
- Requires 3+ Sentinel nodes

```mermaid
sequenceDiagram
    participant Indexer as Indexer Worker
    participant Master as Redis Master
    participant Replica as Redis Replica
    participant Sentinel1 as Sentinel 1
    participant Sentinel2 as Sentinel 2
    participant Sentinel3 as Sentinel 3
    
    Note over Master: Normal operation<br/>250K writes/sec
    
    Indexer->>Master: GEOADD drivers:sf<br/>driver:123
    Master-->>Indexer: OK
    
    Master->>Replica: Async replication<br/>(10-100ms lag)
    
    Note over Master: Master crashes!<br/>(Hardware failure)
    
    Indexer->>Master: GEOADD drivers:sf<br/>driver:456
    Note right of Master: No response<br/>(timeout after 1s)
    
    par Sentinels detect failure
        Sentinel1->>Master: PING
        Sentinel2->>Master: PING
        Sentinel3->>Master: PING
    end
    
    Note over Master: No PONG<br/>(3 second timeout)
    
    Sentinel1-->>Sentinel2: Master down?
    Sentinel2-->>Sentinel1: Confirmed
    Sentinel1-->>Sentinel3: Master down?
    Sentinel3-->>Sentinel1: Confirmed
    
    Note over Sentinel1,Sentinel3: Quorum reached (3/3)<br/>Initiate failover
    
    Sentinel1->>Replica: SLAVEOF NO ONE<br/>(promote to master)
    
    Replica->>Replica: Stop replication<br/>Become master
    
    Replica-->>Sentinel1: Promotion complete
    
    Sentinel1->>Indexer: Broadcast master change<br/>New master: redis-replica-1:6379
    
    Indexer->>Indexer: Update connection pool<br/>Reconnect to new master
    
    Indexer->>Replica: GEOADD drivers:sf<br/>driver:456 (retry)
    Replica-->>Indexer: OK
    
    Note over Replica: New master operational!<br/>Writes resumed<br/>Total downtime: 8 seconds
    
    Note over Master: Old master offline<br/>Ops team investigates
```

---

## 10. Kafka Consumer Lag Recovery Flow

**Flow:**

Shows how to recover when Kafka consumer lag exceeds 10,000 messages due to indexer worker crash or slowdown.

**Steps:**
1. **Normal operation** (0s): 100 workers consuming at 750K msgs/sec, lag=0
2. **Worker crashes** (60s): 10 workers crash, now only 90 workers
3. **Lag increases** (120s): Production rate > consumption rate, lag grows to 10,000
4. **Alert triggered** (125s): PagerDuty alert "Kafka consumer lag > 10K"
5. **Scale up workers** (180s): Deploy 20 new workers (total: 110)
6. **Catch up** (300s): Consumption rate > production rate, lag decreases
7. **Lag cleared** (600s): Lag back to 0, system stable

**Performance:**
- **Detection time:** 5 seconds (Prometheus alert)
- **Scale-up time:** 60 seconds (Kubernetes pod creation)
- **Recovery time:** 5 minutes (clear 10K backlog)

**Benefits:**
- Automatic detection via monitoring
- Scalable recovery (add more workers)
- No data loss (Kafka retains messages)

**Trade-offs:**
- Temporary delay in location updates (stale data)
- Riders may see outdated driver positions
- Cost of spinning up extra workers

```mermaid
sequenceDiagram
    participant Producer as Location Service (Producer)
    participant Kafka as Kafka Cluster
    participant Worker1 as Indexer Workers (100 pods)
    participant Prometheus as Prometheus Monitoring
    participant K8s as Kubernetes
    participant Worker2 as New Workers (10 pods)
    
    Note over Producer,Worker1: Normal operation<br/>Production: 750K msgs/sec<br/>Consumption: 750K msgs/sec<br/>Lag: 0
    
    loop Steady state (60 seconds)
        Producer->>Kafka: Publish 750K msgs/sec
        Kafka->>Worker1: Consume 750K msgs/sec<br/>100 workers × 7.5K/sec each
        Worker1->>Worker1: Process and index
    end
    
    Note over Worker1: 10 workers crash!<br/>(Pod failure)
    
    Worker1->>Worker1: 90 workers remaining
    
    Note over Kafka: Production > Consumption<br/>750K > 675K<br/>Lag increasing!
    
    loop Lag accumulation (60 seconds)
        Producer->>Kafka: 750K msgs/sec
        Kafka->>Worker1: 675K msgs/sec<br/>90 workers × 7.5K/sec
        Kafka->>Kafka: Lag += 75K msgs/sec
    end
    
    Kafka->>Prometheus: Export metrics<br/>consumer_lag: 10,000
    
    Prometheus->>Prometheus: Alert rule triggered<br/>consumer_lag > 10,000
    
    Prometheus->>K8s: Alert: Scale up indexer workers
    Note right of K8s: PagerDuty notification<br/>On-call engineer notified
    
    K8s->>Worker2: Deploy 20 new pods<br/>(60 seconds to ready)
    
    Worker2-->>K8s: Pods healthy
    
    Note over Worker1,Worker2: 110 workers total<br/>Consumption: 825K msgs/sec
    
    loop Catch-up phase (5 minutes)
        Producer->>Kafka: 750K msgs/sec
        Kafka->>Worker1: 675K msgs/sec (90 workers)
        Kafka->>Worker2: 150K msgs/sec (20 workers)
        Note right of Kafka: Consumption > Production<br/>825K > 750K<br/>Lag decreasing!
        Kafka->>Kafka: Lag -= 75K msgs/sec
    end
    
    Note over Kafka: Lag cleared!<br/>Back to 0
    
    K8s->>Worker2: Scale down to 100 workers<br/>(cost optimization)
    
    Note over Producer,Worker1: System stable again
```

---

## 11. Cross-City Trip Handoff Flow

**Flow:**

Shows the rare but complex scenario where a trip starts in one city and ends in another (e.g., San Francisco to San Jose).

**Steps:**
1. **Trip starts** (0s): Rider in SF requests ride to San Jose (50 km away)
2. **Initial shard** (0s): Trip managed by SF shard (drivers:sf)
3. **Trip active** (10min): Driver driving south on Highway 101
4. **Boundary detection** (25min): GPS detects crossing city boundary (lat=37.3382)
5. **Trigger handoff** (25min): Trip Service initiates cross-shard handoff
6. **Update shards** (25.1min): Remove from SF shard, add to SJ shard
7. **Switch tracking** (25.1min): San Jose datacenter now tracks trip
8. **Trip completes** (40min): Arrival in San Jose, managed by SJ shard

**Performance:**
- **Handoff latency:** 100ms (seamless)
- **Frequency:** <1% of trips cross city boundaries

**Benefits:**
- Seamless experience for rider
- Correct regional attribution
- Local datacenter handles completion

**Trade-offs:**
- Complex coordination between shards
- Potential race conditions during handoff
- Edge case testing required

```mermaid
sequenceDiagram
    participant Rider as Rider App
    participant TripSvcSF as Trip Service (SF)
    participant RedisSF as Redis SF (drivers:sf)
    participant TripSvcSJ as Trip Service (SJ)
    participant RedisSJ as Redis SJ (drivers:sj)
    participant Driver as Driver App
    
    Note over Rider,Driver: Trip starts in San Francisco
    
    Rider->>TripSvcSF: Request ride<br/>pickup: SF, destination: San Jose
    TripSvcSF->>RedisSF: Find nearby drivers<br/>GEORADIUS drivers:sf
    RedisSF-->>TripSvcSF: 20 drivers
    TripSvcSF-->>Driver: Assign trip:789
    
    Note over Driver: Trip active<br/>Driving south on Highway 101
    
    loop Every 2 seconds (location tracking)
        Driver->>TripSvcSF: GPS update<br/>lat: 37.7749 → 37.5000 → 37.3382
        TripSvcSF->>RedisSF: Update driver location
    end
    
    Note over TripSvcSF: GPS: lat=37.3382<br/>Crossed SF-SJ boundary!
    
    TripSvcSF->>TripSvcSF: Detect boundary crossing<br/>lat < 37.3500 → San Jose region
    
    TripSvcSF->>TripSvcSJ: Handoff trip:789<br/>{trip_id, rider, driver, destination}
    
    TripSvcSJ->>TripSvcSJ: Accept handoff<br/>Create trip record in SJ shard
    
    TripSvcSJ-->>TripSvcSF: Handoff confirmed
    
    TripSvcSF->>RedisSF: ZREM drivers:sf driver:123<br/>(remove from SF shard)
    
    TripSvcSJ->>RedisSJ: GEOADD drivers:sj driver:123<br/>(add to SJ shard)
    
    Note over Driver: Driver now tracked by<br/>San Jose datacenter
    
    Driver->>TripSvcSJ: GPS updates now route to SJ
    TripSvcSJ->>RedisSJ: Update driver location
    
    Note over Driver: Arrive at destination<br/>San Jose
    
    Driver->>TripSvcSJ: Complete trip
    TripSvcSJ-->>Rider: Trip complete<br/>Receipt: $45.50
    
    Note over RedisSJ: Trip attributed to SJ region<br/>for analytics
```

---

## 12. Payment Processing with Retry Flow

**Flow:**

Shows the payment processing flow with automatic retry logic for transient failures.

**Steps:**
1. **Trip completes** (0s): Driver ends trip, total fare: $15.50
2. **Payment request** (1s): Trip Service calls Payment Service
3. **Stripe API call** (2s): Payment Service charges card via Stripe
4. **Payment fails** (3s): Network timeout or card declined
5. **Retry attempt 1** (13s): Exponential backoff, retry after 10s
6. **Payment fails** (14s): Stripe still unavailable
7. **Retry attempt 2** (44s): Retry after 30s (exponential backoff)
8. **Payment succeeds** (45s): Stripe API returns success
9. **Update trip status** (46s): Trip status=PAID
10. **Notify rider** (47s): Receipt sent via email/push

**Performance:**
- **Success rate:** 99.5% (first attempt: 95%, retry: 4.5%)
- **Max retries:** 3 attempts
- **Total timeout:** 90 seconds

**Benefits:**
- Automatic retry handles transient failures
- Exponential backoff prevents overwhelming Stripe
- High success rate

**Trade-offs:**
- Delayed payment confirmation (up to 90s)
- Rider may wait for receipt
- Failed payments require manual follow-up

```mermaid
sequenceDiagram
    participant Driver as Driver App
    participant TripSvc as Trip Service
    participant PaymentSvc as Payment Service
    participant Stripe as Stripe API
    participant TripDB as PostgreSQL
    participant Rider as Rider App
    
    Driver->>TripSvc: POST /trips/789/complete<br/>Total fare: $15.50
    
    TripSvc->>TripDB: UPDATE trips<br/>SET status=COMPLETED
    
    TripSvc->>PaymentSvc: POST /payments/charge<br/>{rider_id, amount: $15.50, trip_id: 789}
    
    Note over PaymentSvc: Attempt 1
    
    PaymentSvc->>Stripe: POST /v1/charges<br/>{amount: 1550, currency: USD}
    
    Note over Stripe: Network timeout!<br/>(Transient failure)
    
    Stripe-->>PaymentSvc: 408 Request Timeout
    
    PaymentSvc->>PaymentSvc: Retry logic triggered<br/>Exponential backoff: 10s
    
    Note over PaymentSvc: Wait 10 seconds
    
    Note over PaymentSvc: Attempt 2
    
    PaymentSvc->>Stripe: POST /v1/charges<br/>(retry)
    
    Note over Stripe: Stripe still down<br/>(Infrastructure issue)
    
    Stripe-->>PaymentSvc: 503 Service Unavailable
    
    PaymentSvc->>PaymentSvc: Retry logic triggered<br/>Exponential backoff: 30s
    
    Note over PaymentSvc: Wait 30 seconds
    
    Note over PaymentSvc: Attempt 3
    
    PaymentSvc->>Stripe: POST /v1/charges<br/>(final retry)
    
    Note over Stripe: Stripe recovered!<br/>Service healthy
    
    Stripe-->>PaymentSvc: 200 OK<br/>{charge_id: ch_123, status: succeeded}
    
    PaymentSvc-->>TripSvc: Payment successful<br/>charge_id: ch_123
    
    TripSvc->>TripDB: UPDATE trips<br/>SET status=PAID, charge_id=ch_123
    
    TripSvc->>Rider: Push notification<br/>"Payment confirmed: $15.50"
    
    TripSvc->>Rider: Email receipt
    
    TripSvc->>Driver: Push notification<br/>"Earnings: $12.50"
    
    Note over Rider,Driver: Trip complete!<br/>Total latency: 46 seconds
```
