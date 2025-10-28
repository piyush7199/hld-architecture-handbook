# Uber/Lyft Ride Matching System - Pseudocode Implementations

This document contains detailed algorithm implementations for the Uber/Lyft ride matching system. The main challenge document references these functions.

---

## Table of Contents

1. [Geohash Operations](#1-geohash-operations)
2. [Driver Location Management](#2-driver-location-management)
3. [Proximity Search and Matching](#3-proximity-search-and-matching)
4. [ETA Calculation](#4-eta-calculation)
5. [Trip State Management](#5-trip-state-management)
6. [Surge Pricing](#6-surge-pricing)
7. [Geographic Sharding](#7-geographic-sharding)
8. [Boundary Handling](#8-boundary-handling)
9. [Driver State Management](#9-driver-state-management)
10. [Payment Processing](#10-payment-processing)

---

## 1. Geohash Operations

### encode_geohash()

**Purpose:** Convert latitude/longitude coordinates to a geohash string for spatial indexing.

**Parameters:**
- latitude (float): Latitude coordinate (-90 to 90)
- longitude (float): Longitude coordinate (-180 to 180)
- precision (int): Number of characters in geohash (1-12)

**Returns:**
- string: Geohash encoded string (e.g., "9q8yy9")

**Algorithm:**
```
function encode_geohash(latitude, longitude, precision):
  // Base32 alphabet for geohash encoding
  base32 = "0123456789bcdefghjkmnpqrstuvwxyz"
  
  // Initialize bounds
  lat_min = -90.0
  lat_max = 90.0
  lng_min = -180.0
  lng_max = 180.0
  
  geohash = ""
  bits = 0
  bit_index = 0
  
  while length(geohash) < precision:
    if bit_index % 2 == 0:  // Even bit: longitude
      mid = (lng_min + lng_max) / 2
      if longitude > mid:
        bits = bits | (1 << (4 - (bit_index % 5)))
        lng_min = mid
      else:
        lng_max = mid
    else:  // Odd bit: latitude
      mid = (lat_min + lat_max) / 2
      if latitude > mid:
        bits = bits | (1 << (4 - (bit_index % 5)))
        lat_min = mid
      else:
        lat_max = mid
    
    bit_index++
    
    if bit_index % 5 == 0:
      geohash = geohash + base32[bits]
      bits = 0
  
  return geohash
```

**Time Complexity:** O(precision) = O(1) for fixed precision

**Space Complexity:** O(1)

**Example Usage:**
```
geohash = encode_geohash(37.7749, -122.4194, 6)
// Returns: "9q8yy9"
```

---

### decode_geohash()

**Purpose:** Convert geohash string back to latitude/longitude coordinates (center of cell).

**Parameters:**
- geohash (string): Geohash encoded string

**Returns:**
- tuple: (latitude, longitude, lat_error, lng_error)

**Algorithm:**
```
function decode_geohash(geohash):
  base32 = "0123456789bcdefghjkmnpqrstuvwxyz"
  
  lat_min = -90.0
  lat_max = 90.0
  lng_min = -180.0
  lng_max = 180.0
  
  is_longitude = true
  
  for char in geohash:
    index = base32.indexOf(char)
    
    for i in range(4, -1, -1):  // Process 5 bits
      bit = (index >> i) & 1
      
      if is_longitude:
        mid = (lng_min + lng_max) / 2
        if bit == 1:
          lng_min = mid
        else:
          lng_max = mid
      else:
        mid = (lat_min + lat_max) / 2
        if bit == 1:
          lat_min = mid
        else:
          lat_max = mid
      
      is_longitude = not is_longitude
  
  latitude = (lat_min + lat_max) / 2
  longitude = (lng_min + lng_max) / 2
  lat_error = (lat_max - lat_min) / 2
  lng_error = (lng_max - lng_min) / 2
  
  return (latitude, longitude, lat_error, lng_error)
```

**Time Complexity:** O(length(geohash)) = O(1) for fixed precision

**Example Usage:**
```
lat, lng, lat_err, lng_err = decode_geohash("9q8yy9")
// Returns: (37.7749, -122.4194, ±0.0001, ±0.0001)
```

---

### get_adjacent_geohashes()

**Purpose:** Find all 8 adjacent geohash cells for boundary queries.

**Parameters:**
- geohash (string): Target geohash cell

**Returns:**
- list: 8 adjacent geohash strings (N, NE, E, SE, S, SW, W, NW)

**Algorithm:**
```
function get_adjacent_geohashes(geohash):
  // Neighbor lookup tables for geohash
  neighbors = {
    'right':  {
      'even': "bc01fg45238967deuvhjyznpkmstqrwx",
      'odd':  "p0r21436x8zb9dcf5h7kjnmqesgutwvy"
    },
    'left': {
      'even': "238967debc01fg45kmstqrwxuvhjyznp",
      'odd':  "14365h7k9dcfesgujnmqp0r2twvyx8zb"
    },
    'top': {
      'even': "p0r21436x8zb9dcf5h7kjnmqesgutwvy",
      'odd':  "bc01fg45238967deuvhjyznpkmstqrwx"
    },
    'bottom': {
      'even': "14365h7k9dcfesgujnmqp0r2twvyx8zb",
      'odd':  "238967debc01fg45kmstqrwxuvhjyznp"
    }
  }
  
  borders = {
    'right':  {
      'even': "bcfguvyz",
      'odd':  "prxz"
    },
    'left': {
      'even': "0145hjnp",
      'odd':  "028b"
    },
    'top': {
      'even': "prxz",
      'odd':  "bcfguvyz"
    },
    'bottom': {
      'even': "028b",
      'odd':  "0145hjnp"
    }
  }
  
  function get_neighbor(geohash, direction):
    if length(geohash) == 0:
      return ""
    
    last_char = geohash[-1]
    parent = geohash[:-1]
    type = 'even' if length(geohash) % 2 == 0 else 'odd'
    
    if last_char in borders[direction][type] and length(parent) > 0:
      parent = get_neighbor(parent, direction)
    
    base = neighbors[direction][type]
    return parent + base[neighbors['right']['even'].indexOf(last_char)]
  
  // Calculate 8 neighbors
  north = get_neighbor(geohash, 'top')
  south = get_neighbor(geohash, 'bottom')
  east = get_neighbor(geohash, 'right')
  west = get_neighbor(geohash, 'left')
  
  northeast = get_neighbor(north, 'right')
  northwest = get_neighbor(north, 'left')
  southeast = get_neighbor(south, 'right')
  southwest = get_neighbor(south, 'left')
  
  return [north, northeast, east, southeast, south, southwest, west, northwest]
```

**Time Complexity:** O(precision × 8) = O(1)

**Example Usage:**
```
adjacent = get_adjacent_geohashes("9q8yy9")
// Returns: ["9q8yyc", "9q8yyd", "9q8yyb", "9q8yy8", "9q8yy3", "9q8yy6", "9q8yy4", "9q8yy5"]
```

---

## 2. Driver Location Management

### update_driver_location()

**Purpose:** Update driver's GPS location in Redis geo-index and status store.

**Parameters:**
- driver_id (string): Unique driver identifier
- latitude (float): New latitude coordinate
- longitude (float): New longitude coordinate
- status (string): Driver status (available, busy, offline)

**Returns:**
- bool: Success/failure

**Algorithm:**
```
function update_driver_location(driver_id, latitude, longitude, status):
  // Validate coordinates
  if latitude < -90 or latitude > 90:
    throw InvalidCoordinateError("Latitude must be between -90 and 90")
  
  if longitude < -180 or longitude > 180:
    throw InvalidCoordinateError("Longitude must be between -180 and 180")
  
  // Determine city/region shard
  city_code = map_coordinates_to_city(latitude, longitude)
  redis_geo_key = f"drivers:{city_code}"
  redis_state_key = f"driver:{driver_id}:status"
  
  // Update geo-index (Redis GEOADD)
  try:
    redis.geoadd(redis_geo_key, longitude, latitude, driver_id)
    
    // Update driver status with TTL (auto-expire after 60 seconds)
    driver_state = {
      "status": status,
      "latitude": latitude,
      "longitude": longitude,
      "updated_at": current_timestamp(),
      "city_code": city_code
    }
    
    redis.setex(redis_state_key, 60, json_encode(driver_state))
    
    return true
  
  catch RedisError as error:
    log_error(f"Failed to update driver location: {error}")
    return false
```

**Time Complexity:** O(log N) where N = number of drivers in geohash

**Example Usage:**
```
success = update_driver_location("driver:123", 37.7749, -122.4194, "available")
```

---

### batch_update_driver_locations()

**Purpose:** Batch update multiple driver locations for efficiency (used by Kafka consumer).

**Parameters:**
- updates (list): List of (driver_id, lat, lng, status) tuples

**Returns:**
- int: Number of successful updates

**Algorithm:**
```
function batch_update_driver_locations(updates):
  // Group updates by city for efficient batching
  updates_by_city = {}
  
  for (driver_id, lat, lng, status) in updates:
    city_code = map_coordinates_to_city(lat, lng)
    
    if city_code not in updates_by_city:
      updates_by_city[city_code] = []
    
    updates_by_city[city_code].append((driver_id, lat, lng, status))
  
  successful_updates = 0
  
  // Process each city's updates in a Redis pipeline
  for city_code, city_updates in updates_by_city.items():
    redis_geo_key = f"drivers:{city_code}"
    
    pipeline = redis.pipeline()
    
    for (driver_id, lat, lng, status) in city_updates:
      // Add to geo-index
      pipeline.geoadd(redis_geo_key, lng, lat, driver_id)
      
      // Update status
      redis_state_key = f"driver:{driver_id}:status"
      driver_state = {
        "status": status,
        "latitude": lat,
        "longitude": lng,
        "updated_at": current_timestamp(),
        "city_code": city_code
      }
      pipeline.setex(redis_state_key, 60, json_encode(driver_state))
    
    // Execute pipeline
    try:
      results = pipeline.execute()
      successful_updates += length(city_updates)
    catch RedisError as error:
      log_error(f"Batch update failed for {city_code}: {error}")
  
  return successful_updates
```

**Time Complexity:** O(M log N) where M = batch size, N = drivers per city

**Example Usage:**
```
updates = [
  ("driver:123", 37.7749, -122.4194, "available"),
  ("driver:456", 37.7750, -122.4195, "busy"),
  ("driver:789", 37.7751, -122.4196, "available")
]
count = batch_update_driver_locations(updates)
// Returns: 3 (all successful)
```

---

## 3. Proximity Search and Matching

### find_nearest_drivers()

**Purpose:** Find N nearest available drivers within a radius of the rider's location.

**Parameters:**
- rider_lat (float): Rider's latitude
- rider_lng (float): Rider's longitude
- radius_km (float): Search radius in kilometers (default: 5km)
- count (int): Number of drivers to return (default: 20)
- vehicle_type (string): Filter by vehicle type (e.g., "UberX")

**Returns:**
- list: List of driver objects with distance, sorted by distance

**Algorithm:**
```
function find_nearest_drivers(rider_lat, rider_lng, radius_km, count, vehicle_type):
  // Determine city shard
  city_code = map_coordinates_to_city(rider_lat, rider_lng)
  redis_geo_key = f"drivers:{city_code}"
  
  // Query Redis geo-index
  drivers = redis.georadius(
    redis_geo_key,
    rider_lng,
    rider_lat,
    radius_km,
    unit="km",
    withdist=true,
    count=count * 2,  // Get 2× count to account for filtering
    sort="ASC"  // Nearest first
  )
  
  // Filter by availability and vehicle type
  available_drivers = []
  
  for (driver_id, distance_km) in drivers:
    // Get driver status
    redis_state_key = f"driver:{driver_id}:status"
    driver_state_json = redis.get(redis_state_key)
    
    if driver_state_json == null:
      continue  // Driver offline (TTL expired)
    
    driver_state = json_decode(driver_state_json)
    
    // Filter by status and vehicle type
    if driver_state["status"] == "available" and driver_state["vehicle_type"] == vehicle_type:
      available_drivers.append({
        "driver_id": driver_id,
        "distance_km": distance_km,
        "latitude": driver_state["latitude"],
        "longitude": driver_state["longitude"],
        "rating": driver_state["rating"],
        "vehicle_type": driver_state["vehicle_type"]
      })
    
    if length(available_drivers) >= count:
      break
  
  // If insufficient drivers, expand search radius
  if length(available_drivers) < count and radius_km < 50:
    expanded_drivers = find_nearest_drivers(
      rider_lat,
      rider_lng,
      radius_km * 2,  // Double radius
      count,
      vehicle_type
    )
    return expanded_drivers
  
  return available_drivers[:count]
```

**Time Complexity:** O(M log N) where M = count, N = drivers in radius

**Example Usage:**
```
drivers = find_nearest_drivers(37.7749, -122.4194, 5.0, 20, "UberX")
// Returns: [
//   {driver_id: "driver:123", distance_km: 0.5, rating: 4.8, ...},
//   {driver_id: "driver:456", distance_km: 1.2, rating: 4.9, ...},
//   ...
// ]
```

---

### match_rider_with_driver()

**Purpose:** Complete matching algorithm that finds drivers, calculates ETAs, ranks, and notifies.

**Parameters:**
- rider_id (string): Rider identifier
- pickup_lat (float): Pickup location latitude
- pickup_lng (float): Pickup location longitude
- vehicle_type (string): Requested vehicle type

**Returns:**
- string: trip_id if successful, null if no drivers available

**Algorithm:**
```
function match_rider_with_driver(rider_id, pickup_lat, pickup_lng, vehicle_type):
  // Step 1: Find nearby drivers
  nearby_drivers = find_nearest_drivers(
    pickup_lat,
    pickup_lng,
    radius_km=5.0,
    count=20,
    vehicle_type=vehicle_type
  )
  
  if length(nearby_drivers) == 0:
    log_info(f"No drivers available for rider {rider_id}")
    return null
  
  // Step 2: Calculate ETAs for all drivers (parallel batch request)
  eta_requests = []
  for driver in nearby_drivers:
    eta_requests.append({
      "driver_lat": driver["latitude"],
      "driver_lng": driver["longitude"],
      "rider_lat": pickup_lat,
      "rider_lng": pickup_lng
    })
  
  etas = calculate_etas_batch(eta_requests)
  
  // Step 3: Rank drivers by composite score
  ranked_drivers = []
  for i in range(length(nearby_drivers)):
    driver = nearby_drivers[i]
    eta = etas[i]
    
    // Weighted scoring: 70% distance + 20% ETA + 10% rating
    score = (0.7 * driver["distance_km"]) + (0.2 * eta["minutes"]) + (0.1 * (5.0 - driver["rating"]))
    
    ranked_drivers.append({
      "driver": driver,
      "eta": eta,
      "score": score
    })
  
  // Sort by score (lower is better)
  ranked_drivers.sort(key=lambda x: x["score"])
  
  // Step 4: Select top 5 drivers
  top_drivers = ranked_drivers[:5]
  
  // Step 5: Create pending trip
  trip_id = generate_uuid()
  trip = {
    "trip_id": trip_id,
    "rider_id": rider_id,
    "status": "PENDING",
    "pickup": {"lat": pickup_lat, "lng": pickup_lng},
    "vehicle_type": vehicle_type,
    "created_at": current_timestamp()
  }
  
  db.insert("trips", trip)
  
  // Step 6: Broadcast notifications to top 5 drivers
  for driver_data in top_drivers:
    send_push_notification(
      driver_data["driver"]["driver_id"],
      {
        "trip_id": trip_id,
        "rider_id": rider_id,
        "pickup_location": {"lat": pickup_lat, "lng": pickup_lng},
        "eta_minutes": driver_data["eta"]["minutes"],
        "distance_km": driver_data["driver"]["distance_km"]
      }
    )
  
  return trip_id
```

**Time Complexity:** O(N log N + M) where N = nearby drivers, M = ETA calculations

**Example Usage:**
```
trip_id = match_rider_with_driver("rider:789", 37.7749, -122.4194, "UberX")
// Returns: "trip:123e4567-e89b-12d3-a456-426614174000"
```

---

## 4. ETA Calculation

### calculate_eta()

**Purpose:** Calculate estimated time of arrival from driver to rider using OSRM routing engine.

**Parameters:**
- driver_lat (float): Driver's current latitude
- driver_lng (float): Driver's current longitude
- rider_lat (float): Rider's pickup latitude
- rider_lng (float): Rider's pickup longitude

**Returns:**
- dict: {minutes: float, distance_km: float, route: list}

**Algorithm:**
```
function calculate_eta(driver_lat, driver_lng, rider_lat, rider_lng):
  // Generate cache key
  cache_key = f"eta:{hash(driver_lat, driver_lng, rider_lat, rider_lng)}"
  
  // Check cache (TTL: 5 minutes)
  cached_eta = redis.get(cache_key)
  if cached_eta != null:
    return json_decode(cached_eta)
  
  // Call OSRM routing engine
  osrm_url = f"http://osrm-cluster/route/v1/driving/{driver_lng},{driver_lat};{rider_lng},{rider_lat}"
  
  params = {
    "overview": "false",  // Don't need full geometry
    "steps": "false",      // Don't need turn-by-turn
    "annotations": "false"
  }
  
  try:
    response = http_get(osrm_url, params, timeout=50ms)
    
    if response.status != 200:
      throw OSRMError("OSRM request failed")
    
    route = response.json["routes"][0]
    distance_meters = route["distance"]
    duration_seconds = route["duration"]
    
    // Adjust for real-time traffic (query Traffic Service)
    traffic_multiplier = get_traffic_multiplier(driver_lat, driver_lng, rider_lat, rider_lng)
    adjusted_duration_seconds = duration_seconds * traffic_multiplier
    
    result = {
      "minutes": adjusted_duration_seconds / 60.0,
      "distance_km": distance_meters / 1000.0,
      "route_found": true
    }
    
    // Cache result
    redis.setex(cache_key, 300, json_encode(result))  // TTL: 5 minutes
    
    return result
  
  catch OSRMError, HTTPError as error:
    log_error(f"ETA calculation failed: {error}")
    
    // Fallback: straight-line distance estimation
    distance_km = haversine_distance(driver_lat, driver_lng, rider_lat, rider_lng)
    estimated_minutes = distance_km / 0.5  // Assume 30 km/h average speed
    
    return {
      "minutes": estimated_minutes,
      "distance_km": distance_km,
      "route_found": false
    }
```

**Time Complexity:** O(1) cached, O(E log V) for OSRM A* pathfinding

**Example Usage:**
```
eta = calculate_eta(37.7749, -122.4194, 37.7750, -122.4000)
// Returns: {minutes: 5.2, distance_km: 2.3, route_found: true}
```

---

### calculate_etas_batch()

**Purpose:** Calculate ETAs for multiple driver-rider pairs in parallel (batch optimization).

**Parameters:**
- eta_requests (list): List of {driver_lat, driver_lng, rider_lat, rider_lng} dicts

**Returns:**
- list: List of ETA results

**Algorithm:**
```
function calculate_etas_batch(eta_requests):
  etas = []
  cache_keys = []
  cache_hits = []
  cache_misses = []
  
  // Step 1: Check cache for all requests
  for request in eta_requests:
    cache_key = f"eta:{hash(request)}"
    cache_keys.append(cache_key)
  
  cached_values = redis.mget(cache_keys)
  
  for i in range(length(eta_requests)):
    if cached_values[i] != null:
      cache_hits.append((i, json_decode(cached_values[i])))
    else:
      cache_misses.append((i, eta_requests[i]))
  
  // Step 2: Calculate ETAs for cache misses (parallel HTTP requests)
  if length(cache_misses) > 0:
    osrm_requests = []
    
    for (index, request) in cache_misses:
      url = f"http://osrm-cluster/route/v1/driving/{request['driver_lng']},{request['driver_lat']};{request['rider_lng']},{request['rider_lat']}"
      osrm_requests.append((index, url))
    
    // Execute parallel HTTP requests (async/await)
    osrm_responses = parallel_http_get(osrm_requests, timeout=50ms)
    
    for (index, response) in osrm_responses:
      if response.status == 200:
        route = response.json["routes"][0]
        eta = {
          "minutes": route["duration"] / 60.0,
          "distance_km": route["distance"] / 1000.0,
          "route_found": true
        }
        
        // Cache result
        cache_key = cache_keys[index]
        redis.setex(cache_key, 300, json_encode(eta))
        
        cache_misses[index] = (index, eta)
  
  // Step 3: Merge cache hits and calculated ETAs
  results = [null] * length(eta_requests)
  
  for (index, eta) in cache_hits:
    results[index] = eta
  
  for (index, eta) in cache_misses:
    results[index] = eta
  
  return results
```

**Time Complexity:** O(M) where M = number of requests (parallelized)

**Example Usage:**
```
requests = [
  {driver_lat: 37.7749, driver_lng: -122.4194, rider_lat: 37.7750, rider_lng: -122.4000},
  {driver_lat: 37.7751, driver_lng: -122.4195, rider_lat: 37.7750, rider_lng: -122.4000}
]
etas = calculate_etas_batch(requests)
// Returns: [{minutes: 5.2, ...}, {minutes: 4.8, ...}]
```

---

## 5. Trip State Management

### accept_trip()

**Purpose:** Handle driver acceptance of trip using atomic compare-and-set (CAS) operation.

**Parameters:**
- trip_id (string): Trip identifier
- driver_id (string): Driver attempting to accept

**Returns:**
- bool: True if acceptance succeeded (first driver), False if trip already taken

**Algorithm:**
```
function accept_trip(trip_id, driver_id):
  // Atomic CAS operation: only one driver can win
  query = "UPDATE trips SET status = 'ACCEPTED', driver_id = ?, accepted_at = ? WHERE trip_id = ? AND status = 'PENDING'"
  
  params = [driver_id, current_timestamp(), trip_id]
  
  result = db.execute(query, params)
  
  if result.rows_affected == 1:
    // Success: This driver won the race
    trip = db.query("SELECT * FROM trips WHERE trip_id = ?", [trip_id])
    
    // Notify rider
    send_push_notification(trip["rider_id"], {
      "type": "driver_accepted",
      "driver_id": driver_id,
      "message": "Driver is on the way!",
      "eta_minutes": calculate_eta(
        trip["driver_lat"],
        trip["driver_lng"],
        trip["pickup_lat"],
        trip["pickup_lng"]
      )["minutes"]
    })
    
    // Update driver status to busy
    update_driver_status(driver_id, "busy")
    
    return true
  
  else:
    // Failure: Another driver already accepted
    return false
```

**Time Complexity:** O(1) for atomic UPDATE

**Example Usage:**
```
success = accept_trip("trip:123", "driver:456")
// Returns: true (first driver) or false (too late)
```

---

### complete_trip()

**Purpose:** Mark trip as completed and process payment.

**Parameters:**
- trip_id (string): Trip identifier
- driver_id (string): Driver completing trip
- end_lat (float): Drop-off latitude
- end_lng (float): Drop-off longitude

**Returns:**
- dict: {success: bool, fare: float, payment_status: string}

**Algorithm:**
```
function complete_trip(trip_id, driver_id, end_lat, end_lng):
  // Step 1: Validate trip state
  trip = db.query("SELECT * FROM trips WHERE trip_id = ? AND driver_id = ? AND status = 'ACTIVE'", [trip_id, driver_id])
  
  if trip == null:
    throw InvalidTripStateError("Trip not found or not active")
  
  // Step 2: Calculate fare
  distance_km = haversine_distance(
    trip["pickup_lat"],
    trip["pickup_lng"],
    end_lat,
    end_lng
  )
  
  duration_minutes = (current_timestamp() - trip["started_at"]) / 60
  
  // Get surge multiplier for pickup location
  pickup_geohash = encode_geohash(trip["pickup_lat"], trip["pickup_lng"], 6)
  surge_multiplier = get_surge_multiplier(pickup_geohash)
  
  // Fare formula: base + (distance × rate) + (time × rate)
  base_fare = 2.50
  distance_rate = 1.50  // $ per km
  time_rate = 0.30      // $ per minute
  
  fare = (base_fare + (distance_km * distance_rate) + (duration_minutes * time_rate)) * surge_multiplier
  fare = round(fare, 2)
  
  // Step 3: Update trip status
  db.execute(
    "UPDATE trips SET status = 'COMPLETED', end_lat = ?, end_lng = ?, distance_km = ?, duration_minutes = ?, fare = ?, completed_at = ? WHERE trip_id = ?",
    [end_lat, end_lng, distance_km, duration_minutes, fare, current_timestamp(), trip_id]
  )
  
  // Step 4: Process payment (synchronous)
  payment_result = charge_rider(trip["rider_id"], fare, trip_id)
  
  if payment_result["success"]:
    db.execute("UPDATE trips SET status = 'PAID', charge_id = ? WHERE trip_id = ?", [payment_result["charge_id"], trip_id])
    
    // Update driver status back to available
    update_driver_status(driver_id, "available")
    
    // Send receipts
    send_receipt(trip["rider_id"], trip_id, fare)
    send_earnings_notification(driver_id, fare * 0.80)  // Driver gets 80%
    
    return {
      "success": true,
      "fare": fare,
      "payment_status": "paid"
    }
  
  else:
    db.execute("UPDATE trips SET status = 'PAYMENT_FAILED', payment_error = ? WHERE trip_id = ?", [payment_result["error"], trip_id])
    
    return {
      "success": false,
      "fare": fare,
      "payment_status": "failed",
      "error": payment_result["error"]
    }
```

**Time Complexity:** O(1) database operations + O(1) payment API call

**Example Usage:**
```
result = complete_trip("trip:123", "driver:456", 37.7850, -122.4100)
// Returns: {success: true, fare: 15.50, payment_status: "paid"}
```

---

## 6. Surge Pricing

### calculate_surge_multiplier()

**Purpose:** Calculate surge pricing multiplier based on supply/demand ratio in a geohash cell.

**Parameters:**
- geohash (string): Geohash cell identifier
- city_code (string): City identifier

**Returns:**
- float: Surge multiplier (1.0× to 3.0×)

**Algorithm:**
```
function calculate_surge_multiplier(geohash, city_code):
  // Step 1: Count available drivers in cell
  redis_geo_key = f"drivers:{city_code}"
  
  // Get center coordinates of geohash
  lat, lng, _, _ = decode_geohash(geohash)
  
  // Query drivers in cell (use geohash bounding box)
  cell_size_km = 1.2  // 6-character geohash ~1.2km
  drivers = redis.georadius(redis_geo_key, lng, lat, cell_size_km, unit="km")
  
  // Filter by availability
  available_count = 0
  for driver_id in drivers:
    driver_state = redis.get(f"driver:{driver_id}:status")
    if driver_state != null and json_decode(driver_state)["status"] == "available":
      available_count++
  
  // Step 2: Count pending ride requests in cell
  pending_count = db.query(
    "SELECT COUNT(*) FROM trips WHERE geohash = ? AND status = 'PENDING' AND created_at > (NOW() - INTERVAL 5 MINUTE)",
    [geohash]
  )["count"]
  
  // Step 3: Calculate demand/supply ratio
  if available_count == 0:
    ratio = float('inf')  // No drivers available
  else:
    ratio = float(pending_count) / float(available_count)
  
  // Step 4: Apply surge brackets
  if ratio < 0.5:
    multiplier = 1.0   // No surge
  else if ratio < 1.0:
    multiplier = 1.2   // Low surge
  else if ratio < 2.0:
    multiplier = 1.5   // Medium surge
  else if ratio < 5.0:
    multiplier = 2.0   // High surge
  else:
    multiplier = 3.0   // Maximum surge (cap)
  
  // Step 5: Cache result (TTL: 60 seconds)
  redis.setex(f"surge:{geohash}", 60, multiplier)
  
  return multiplier
```

**Time Complexity:** O(D + P) where D = drivers in cell, P = pending trips

**Example Usage:**
```
multiplier = calculate_surge_multiplier("9q8yy9", "sf")
// Returns: 2.0 (high surge)
```

---

### refresh_surge_pricing()

**Purpose:** Background job that recalculates surge pricing for all active geohash cells every 30 seconds.

**Parameters:**
- city_code (string): City to refresh

**Returns:**
- int: Number of cells updated

**Algorithm:**
```
function refresh_surge_pricing(city_code):
  // Get all active geohash cells (cells with recent activity)
  active_geohashes = db.query(
    "SELECT DISTINCT geohash FROM trips WHERE city_code = ? AND created_at > (NOW() - INTERVAL 10 MINUTE)",
    [city_code]
  )
  
  updated_count = 0
  
  for geohash_row in active_geohashes:
    geohash = geohash_row["geohash"]
    
    try:
      multiplier = calculate_surge_multiplier(geohash, city_code)
      
      // If surge changed significantly, notify riders in area
      previous_multiplier = redis.get(f"surge:{geohash}:previous") or 1.0
      
      if abs(multiplier - previous_multiplier) >= 0.5:
        notify_riders_in_geohash(geohash, multiplier)
        redis.set(f"surge:{geohash}:previous", multiplier)
      
      updated_count++
    
    catch Exception as error:
      log_error(f"Failed to calculate surge for {geohash}: {error}")
  
  log_info(f"Refreshed surge pricing for {updated_count} cells in {city_code}")
  
  return updated_count
```

**Time Complexity:** O(C × (D + P)) where C = number of cells

**Example Usage:**
```
count = refresh_surge_pricing("sf")
// Returns: 47 (47 cells updated)
```

---

## 7. Geographic Sharding

### map_coordinates_to_city()

**Purpose:** Map latitude/longitude coordinates to a city code for geographic sharding.

**Parameters:**
- latitude (float): Latitude coordinate
- longitude (float): Longitude coordinate

**Returns:**
- string: City code (e.g., "sf", "nyc", "lon")

**Algorithm:**
```
function map_coordinates_to_city(latitude, longitude):
  // City bounding boxes (simplified)
  city_boundaries = {
    "sf": {  // San Francisco Bay Area
      "lat_min": 37.2, "lat_max": 38.0,
      "lng_min": -122.7, "lng_max": -122.0
    },
    "nyc": {  // New York City
      "lat_min": 40.5, "lat_max": 41.0,
      "lng_min": -74.3, "lng_max": -73.7
    },
    "lon": {  // London
      "lat_min": 51.3, "lat_max": 51.7,
      "lng_min": -0.5, "lng_max": 0.3
    },
    "tok": {  // Tokyo
      "lat_min": 35.5, "lat_max": 35.9,
      "lng_min": 139.5, "lng_max": 140.0
    }
  }
  
  // Check each city boundary
  for city_code, bounds in city_boundaries.items():
    if (latitude >= bounds["lat_min"] and latitude <= bounds["lat_max"] and
        longitude >= bounds["lng_min"] and longitude <= bounds["lng_max"]):
      return city_code
  
  // Default: map to nearest city using haversine distance
  city_centers = {
    "sf": (37.7749, -122.4194),
    "nyc": (40.7128, -74.0060),
    "lon": (51.5074, -0.1278),
    "tok": (35.6762, 139.6503)
  }
  
  nearest_city = null
  min_distance = float('inf')
  
  for city_code, (city_lat, city_lng) in city_centers.items():
    distance = haversine_distance(latitude, longitude, city_lat, city_lng)
    if distance < min_distance:
      min_distance = distance
      nearest_city = city_code
  
  return nearest_city
```

**Time Complexity:** O(C) where C = number of cities (constant)

**Example Usage:**
```
city = map_coordinates_to_city(37.7749, -122.4194)
// Returns: "sf"
```

---

## 8. Boundary Handling

### is_near_city_boundary()

**Purpose:** Detect if a location is near a city/geohash boundary (requires multi-shard query).

**Parameters:**
- latitude (float): Location latitude
- longitude (float): Location longitude
- threshold_km (float): Distance threshold (default: 10km)

**Returns:**
- bool: True if near boundary

**Algorithm:**
```
function is_near_city_boundary(latitude, longitude, threshold_km):
  city_code = map_coordinates_to_city(latitude, longitude)
  
  // Get city boundaries
  city_boundaries = get_city_boundaries()
  bounds = city_boundaries[city_code]
  
  // Calculate distances to each edge
  // Convert lat/lng degrees to km (approximate)
  lat_km_per_degree = 111.0
  lng_km_per_degree = 111.0 * cos(radians(latitude))
  
  distance_to_north = (bounds["lat_max"] - latitude) * lat_km_per_degree
  distance_to_south = (latitude - bounds["lat_min"]) * lat_km_per_degree
  distance_to_east = (bounds["lng_max"] - longitude) * lng_km_per_degree
  distance_to_west = (longitude - bounds["lng_min"]) * lng_km_per_degree
  
  min_distance = min(distance_to_north, distance_to_south, distance_to_east, distance_to_west)
  
  return min_distance < threshold_km
```

**Time Complexity:** O(1)

**Example Usage:**
```
near_boundary = is_near_city_boundary(37.7749, -122.4194, 10.0)
// Returns: false (in center of SF)
```

---

### query_cross_boundary()

**Purpose:** Query multiple city shards when near boundary to avoid missing drivers.

**Parameters:**
- rider_lat (float): Rider latitude
- rider_lng (float): Rider longitude
- radius_km (float): Search radius
- count (int): Number of drivers to return

**Returns:**
- list: Combined list of drivers from multiple shards

**Algorithm:**
```
function query_cross_boundary(rider_lat, rider_lng, radius_km, count):
  primary_city = map_coordinates_to_city(rider_lat, rider_lng)
  
  // Check if near boundary
  if not is_near_city_boundary(rider_lat, rider_lng, radius_km):
    // Not near boundary, single shard query is sufficient
    return find_nearest_drivers(rider_lat, rider_lng, radius_km, count, primary_city)
  
  // Near boundary: determine adjacent cities
  adjacent_cities = get_adjacent_cities(primary_city)
  all_cities = [primary_city] + adjacent_cities
  
  // Query all relevant shards in parallel
  all_drivers = []
  
  parallel_results = parallel_map(all_cities, lambda city_code:
    query_drivers_in_shard(rider_lat, rider_lng, radius_km, count * 2, city_code)
  )
  
  for city_drivers in parallel_results:
    all_drivers.extend(city_drivers)
  
  // Remove duplicates (driver may be indexed in multiple shards)
  unique_drivers = {}
  for driver in all_drivers:
    driver_id = driver["driver_id"]
    if driver_id not in unique_drivers:
      unique_drivers[driver_id] = driver
    else:
      // Keep driver with shorter distance
      if driver["distance_km"] < unique_drivers[driver_id]["distance_km"]:
        unique_drivers[driver_id] = driver
  
  // Sort by distance and return top N
  sorted_drivers = sorted(unique_drivers.values(), key=lambda d: d["distance_km"])
  
  return sorted_drivers[:count]
```

**Time Complexity:** O(S × M log N) where S = shards queried, M = count, N = drivers per shard

**Example Usage:**
```
drivers = query_cross_boundary(37.2500, -122.0000, 5.0, 20)
// Queries both SF and San Jose shards if near boundary
```

---

## 9. Driver State Management

### update_driver_status()

**Purpose:** Update driver's availability status (available, busy, offline, on_break).

**Parameters:**
- driver_id (string): Driver identifier
- new_status (string): New status value

**Returns:**
- bool: Success/failure

**Algorithm:**
```
function update_driver_status(driver_id, new_status):
  valid_statuses = ["available", "busy", "offline", "on_break"]
  
  if new_status not in valid_statuses:
    throw InvalidStatusError(f"Invalid status: {new_status}")
  
  redis_state_key = f"driver:{driver_id}:status"
  
  // Get current state
  current_state_json = redis.get(redis_state_key)
  
  if current_state_json == null:
    // Driver not found (TTL expired)
    if new_status == "offline":
      return true  // Already offline
    else:
      throw DriverNotFoundError(f"Driver {driver_id} not found")
  
  current_state = json_decode(current_state_json)
  current_state["status"] = new_status
  current_state["updated_at"] = current_timestamp()
  
  // Update Redis with TTL
  if new_status == "offline":
    // Offline: delete from Redis (no TTL)
    redis.delete(redis_state_key)
    
    // Remove from geo-index
    city_code = current_state["city_code"]
    redis.zrem(f"drivers:{city_code}", driver_id)
  
  else:
    // Online states: refresh TTL
    redis.setex(redis_state_key, 60, json_encode(current_state))
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**
```
success = update_driver_status("driver:123", "busy")
```

---

## 10. Payment Processing

### charge_rider()

**Purpose:** Charge rider's credit card via Stripe API with retry logic.

**Parameters:**
- rider_id (string): Rider identifier
- amount (float): Amount to charge in dollars
- trip_id (string): Trip identifier for idempotency

**Returns:**
- dict: {success: bool, charge_id: string, error: string}

**Algorithm:**
```
function charge_rider(rider_id, amount, trip_id):
  // Get rider's payment method
  payment_method = db.query("SELECT stripe_customer_id, stripe_payment_method_id FROM riders WHERE rider_id = ?", [rider_id])
  
  if payment_method == null:
    return {"success": false, "error": "No payment method found"}
  
  // Stripe API call with retry logic
  max_retries = 3
  retry_count = 0
  backoff_seconds = 1
  
  while retry_count < max_retries:
    try:
      // Call Stripe API (idempotent with trip_id as idempotency key)
      charge = stripe.create_charge({
        "amount": int(amount * 100),  // Cents
        "currency": "usd",
        "customer": payment_method["stripe_customer_id"],
        "payment_method": payment_method["stripe_payment_method_id"],
        "description": f"Uber ride {trip_id}",
        "idempotency_key": trip_id  // Prevents duplicate charges
      })
      
      if charge["status"] == "succeeded":
        return {
          "success": true,
          "charge_id": charge["id"],
          "error": null
        }
      
      else:
        return {
          "success": false,
          "charge_id": null,
          "error": f"Charge failed: {charge['failure_message']}"
        }
    
    catch StripeNetworkError as error:
      // Transient network error: retry with exponential backoff
      retry_count++
      if retry_count < max_retries:
        log_warning(f"Stripe network error, retry {retry_count}/{max_retries}: {error}")
        sleep(backoff_seconds)
        backoff_seconds *= 2  // Exponential backoff
      else:
        return {
          "success": false,
          "charge_id": null,
          "error": f"Payment failed after {max_retries} retries: {error}"
        }
    
    catch StripeCardError as error:
      // Card declined: don't retry
      return {
        "success": false,
        "charge_id": null,
        "error": f"Card declined: {error.message}"
      }
```

**Time Complexity:** O(R) where R = retry attempts (max 3)

**Example Usage:**
```
result = charge_rider("rider:789", 15.50, "trip:123")
// Returns: {success: true, charge_id: "ch_1A2B3C4D", error: null}
```

---

## Utility Functions

### haversine_distance()

**Purpose:** Calculate great-circle distance between two lat/lng points.

**Parameters:**
- lat1, lng1 (float): First point coordinates
- lat2, lng2 (float): Second point coordinates

**Returns:**
- float: Distance in kilometers

**Algorithm:**
```
function haversine_distance(lat1, lng1, lat2, lng2):
  R = 6371.0  // Earth radius in kilometers
  
  lat1_rad = radians(lat1)
  lat2_rad = radians(lat2)
  delta_lat = radians(lat2 - lat1)
  delta_lng = radians(lng2 - lng1)
  
  a = sin(delta_lat / 2)^2 + cos(lat1_rad) * cos(lat2_rad) * sin(delta_lng / 2)^2
  c = 2 * atan2(sqrt(a), sqrt(1 - a))
  
  distance_km = R * c
  
  return distance_km
```

**Time Complexity:** O(1)

**Example Usage:**
```
distance = haversine_distance(37.7749, -122.4194, 37.7750, -122.4000)
// Returns: 1.5 (km)
```
