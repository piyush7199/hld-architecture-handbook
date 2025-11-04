# Yelp/Google Maps - Pseudocode Implementations

This document contains detailed algorithm implementations for the Yelp/Google Maps geo-spatial search system. The main
challenge document references these functions.

---

## Table of Contents

1. [Geohash Operations](#1-geohash-operations)
2. [POI Indexing and Storage](#2-poi-indexing-and-storage)
3. [Proximity Search](#3-proximity-search)
4. [Filtering and Ranking](#4-filtering-and-ranking)
5. [Cross-Boundary Query Handling](#5-cross-boundary-query-handling)
6. [Review Management](#6-review-management)
7. [Caching Operations](#7-caching-operations)
8. [POI Updates and Synchronization](#8-poi-updates-and-synchronization)
9. [Geographic Sharding](#9-geographic-sharding)
10. [Search Result Aggregation](#10-search-result-aggregation)

---

## 1. Geohash Operations

### encode_geohash()

**Purpose:** Convert latitude/longitude coordinates to a geohash string for spatial indexing.

**Parameters:**

- latitude (float): Latitude coordinate (-90 to 90)
- longitude (float): Longitude coordinate (-180 to 180)
- precision (int): Number of characters in geohash (1-12, typically 7 for POI-level)

**Returns:**

- string: Geohash encoded string (e.g., "9q8yy9m")

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
      if longitude >= mid:
        bits = bits | (1 << (4 - (bit_index % 5)))
        lng_min = mid
      else:
        lng_max = mid
    else:  // Odd bit: latitude
      mid = (lat_min + lat_max) / 2
      if latitude >= mid:
        bits = bits | (1 << (4 - (bit_index % 5)))
        lat_min = mid
      else:
        lat_max = mid
    
    bit_index = bit_index + 1
    
    if bit_index % 5 == 0:
      geohash = geohash + base32[bits]
      bits = 0
  
  return geohash
```

**Time Complexity:** O(precision) = O(1) for fixed precision (7 chars)

**Space Complexity:** O(1)

**Example Usage:**

```
geohash = encode_geohash(37.7749, -122.4194, 7)
// Returns: "9q8yy9m"
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
    
    for i in range(4, -1, -1):  // Process 5 bits (MSB to LSB)
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
lat, lng, lat_err, lng_err = decode_geohash("9q8yy9m")
// Returns: (37.7749, -122.4194, ±0.0001, ±0.0001)
```

---

### get_adjacent_geohashes()

**Purpose:** Find all 8 adjacent geohash cells for boundary queries (N, NE, E, SE, S, SW, W, NW).

**Parameters:**

- geohash (string): Target geohash cell

**Returns:**

- list: 8 adjacent geohash strings

**Algorithm:**

```
function get_adjacent_geohashes(geohash):
  base32 = "0123456789bcdefghjkmnpqrstuvwxyz"
  
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
    base32 = "0123456789bcdefghjkmnpqrstuvwxyz"
    length = length(geohash)
    last_char = geohash[length - 1]
    
    is_even = (length % 2 == 0)
    border = borders[direction]
    neighbor_chars = neighbors[direction]
    
    if last_char in border[is_even ? 'even' : 'odd']:
      // Recursively get neighbor of parent cell
      parent = geohash[0:length-1]
      parent_neighbor = get_neighbor(parent, direction)
      return parent_neighbor + base32[0]  // Append base char
    else:
      // Find neighbor in same cell
      index = base32.indexOf(last_char)
      neighbor_index = neighbor_chars.indexOf(last_char)
      return geohash[0:length-1] + base32[neighbor_index]
  
  // Get all 8 neighbors
  north = get_neighbor(geohash, 'top')
  south = get_neighbor(geohash, 'bottom')
  east = get_neighbor(geohash, 'right')
  west = get_neighbor(geohash, 'left')
  
  northeast = get_neighbor(east, 'top')
  northwest = get_neighbor(west, 'top')
  southeast = get_neighbor(east, 'bottom')
  southwest = get_neighbor(west, 'bottom')
  
  return [north, northeast, east, southeast, south, southwest, west, northwest]
```

**Time Complexity:** O(length(geohash)) = O(1) for fixed precision

**Example Usage:**

```
neighbors = get_adjacent_geohashes("9q8yy9m")
// Returns: ["9q8yy9t", "9q8yy9q", "9q8yy9k", "9q8yy9h", "9q8yy9j", "9q8yy9v", "9q8yy9s", "9q8yy9u"]
```

---

### get_geohash_precision()

**Purpose:** Determine optimal Geohash precision based on search radius.

**Parameters:**

- radius_km (float): Search radius in kilometers

**Returns:**

- int: Optimal Geohash precision (number of characters)

**Algorithm:**

```
function get_geohash_precision(radius_km):
  // Geohash precision table (approximate cell sizes)
  // Precision 1: ~5000km × 5000km
  // Precision 2: ~1250km × 625km
  // Precision 3: ~156km × 156km
  // Precision 4: ~39km × 19km
  // Precision 5: ~4.9km × 4.9km
  // Precision 6: ~1.2km × 0.6km
  // Precision 7: ~153m × 153m
  // Precision 8: ~38m × 19m
  // Precision 9: ~4.8m × 4.8m
  
  if radius_km < 0.5:
    return 8  // ~38m × 19m
  else if radius_km < 1:
    return 7  // ~153m × 153m
  else if radius_km < 5:
    return 6  // ~1.2km × 0.6km
  else if radius_km < 20:
    return 5  // ~4.9km × 4.9km
  else if radius_km < 100:
    return 4  // ~39km × 19km
  else:
    return 3  // ~156km × 156km
```

**Time Complexity:** O(1)

**Example Usage:**

```
precision = get_geohash_precision(5.0)  // 5km radius
// Returns: 6
```

---

## 2. POI Indexing and Storage

### index_poi()

**Purpose:** Index a new POI in Elasticsearch with geohash encoding.

**Parameters:**

- poi_id (string): Unique POI identifier
- name (string): POI name
- latitude (float): Latitude coordinate
- longitude (float): Longitude coordinate
- category (list): List of categories (e.g., ["restaurant", "italian"])
- rating (float): Average rating (0.0 to 5.0)
- review_count (int): Number of reviews
- address (object): Address object with street, city, state, zip
- hours (object): Hours of operation
- attributes (object): Custom attributes (wheelchair_accessible, parking, etc.)

**Returns:**

- boolean: true if indexing successful, false otherwise

**Algorithm:**

```
function index_poi(poi_id, name, latitude, longitude, category, rating, review_count, address, hours, attributes):
  try:
    // Calculate geohash
    geohash = encode_geohash(latitude, longitude, precision=7)
    
    // Create Elasticsearch document
    document = {
      "poi_id": poi_id,
      "name": name,
      "location": {
        "lat": latitude,
        "lon": longitude
      },
      "geohash": geohash,
      "category": category,
      "rating": rating,
      "review_count": review_count,
      "address": address,
      "hours": hours,
      "attributes": attributes,
      "created_at": current_timestamp(),
      "updated_at": current_timestamp()
    }
    
    // Index in Elasticsearch
    elasticsearch.index(
      index="pois",
      id=poi_id,
      document=document
    )
    
    // Also store in PostgreSQL for metadata
    postgresql.insert("pois", {
      "poi_id": poi_id,
      "name": name,
      "latitude": latitude,
      "longitude": longitude,
      "geohash": geohash,
      "address": address,
      "phone": attributes.phone,
      "website": attributes.website,
      "created_at": current_timestamp(),
      "updated_at": current_timestamp()
    })
    
    return true
  catch error:
    log_error("Failed to index POI", poi_id, error)
    return false
```

**Time Complexity:** O(1) for geohash encoding + O(log N) for Elasticsearch indexing

**Example Usage:**

```
success = index_poi(
  poi_id="poi_123",
  name="Golden Gate Pizza",
  latitude=37.7749,
  longitude=-122.4194,
  category=["restaurant", "italian", "pizza"],
  rating=4.5,
  review_count=1250,
  address={...},
  hours={...},
  attributes={...}
)
```

---

### get_poi_details()

**Purpose:** Retrieve detailed POI information from PostgreSQL.

**Parameters:**

- poi_id (string): POI identifier

**Returns:**

- object: POI details object or null if not found

**Algorithm:**

```
function get_poi_details(poi_id):
  try:
    // Query PostgreSQL for POI metadata
    poi = postgresql.query(
      "SELECT * FROM pois WHERE poi_id = $1",
      [poi_id]
    )
    
    if poi is null:
      return null
    
    // Query categories
    categories = postgresql.query(
      "SELECT category FROM poi_categories WHERE poi_id = $1",
      [poi_id]
    )
    
    // Query hours
    hours = postgresql.query(
      "SELECT day_of_week, open_time, close_time FROM poi_hours WHERE poi_id = $1 ORDER BY day_of_week",
      [poi_id]
    )
    
    // Combine results
    poi_details = {
      "poi_id": poi.poi_id,
      "name": poi.name,
      "latitude": poi.latitude,
      "longitude": poi.longitude,
      "address": poi.address,
      "phone": poi.phone,
      "website": poi.website,
      "categories": categories.map(c => c.category),
      "hours": format_hours(hours),
      "created_at": poi.created_at,
      "updated_at": poi.updated_at
    }
    
    return poi_details
  catch error:
    log_error("Failed to get POI details", poi_id, error)
    return null
```

**Time Complexity:** O(1) for indexed queries

**Example Usage:**

```
poi = get_poi_details("poi_123")
```

---

## 3. Proximity Search

### search_pois()

**Purpose:** Search for POIs within a specified radius with optional filters.

**Parameters:**

- latitude (float): Search center latitude
- longitude (float): Search center longitude
- radius_km (float): Search radius in kilometers
- category (string, optional): Filter by category (e.g., "restaurant")
- min_rating (float, optional): Minimum rating filter (e.g., 4.0)
- price_range (int, optional): Price range filter (1-4)
- open_now (boolean, optional): Filter for POIs open now
- limit (int): Maximum number of results (default: 20)

**Returns:**

- list: List of POI objects sorted by distance and rating

**Algorithm:**

```
function search_pois(latitude, longitude, radius_km, category, min_rating, price_range, open_now, limit):
  try:
    // Calculate geohash for search center
    precision = get_geohash_precision(radius_km)
    center_geohash = encode_geohash(latitude, longitude, precision)
    
    // Get adjacent cells for boundary accuracy
    adjacent_cells = get_adjacent_geohashes(center_geohash)
    all_cells = [center_geohash] + adjacent_cells
    
    // Build Elasticsearch query
    query = {
      "bool": {
        "must": [
          {
            "terms": {"geohash": all_cells}  // Query center + 8 neighbors
          },
          {
            "geo_distance": {
              "distance": radius_km + "km",
              "location": {
                "lat": latitude,
                "lon": longitude
              }
            }
          }
        ]
      }
    }
    
    // Add optional filters
    if category is not null:
      query.bool.must.append({"term": {"category": category}})
    
    if min_rating is not null:
      query.bool.must.append({"range": {"rating": {"gte": min_rating}}})
    
    if price_range is not null:
      query.bool.must.append({"term": {"price_range": price_range}})
    
    if open_now is true:
      current_time = get_current_time()
      day_of_week = get_day_of_week()
      query.bool.must.append({
        "bool": {
          "should": [
            {"range": {
              "hours." + day_of_week + ".open": {"lte": current_time}},
            {"range": {
              "hours." + day_of_week + ".close": {"gte": current_time}}
          ]
        }
      })
    
    // Sort by distance, then rating
    sort = [
      {
        "_geo_distance": {
          "location": {"lat": latitude, "lon": longitude},
          "order": "asc",
          "unit": "km"
        }
      },
      {"rating": {"order": "desc"}}
    ]
    
    // Execute Elasticsearch query
    results = elasticsearch.search(
      index="pois",
      query=query,
      sort=sort,
      size=limit
    )
    
    // Format results
    pois = []
    for hit in results.hits:
      poi = {
        "poi_id": hit._source.poi_id,
        "name": hit._source.name,
        "location": hit._source.location,
        "category": hit._source.category,
        "rating": hit._source.rating,
        "review_count": hit._source.review_count,
        "distance_km": hit.sort[0]  // Distance from search center
      }
      pois.append(poi)
    
    return pois
  catch error:
    log_error("Failed to search POIs", error)
    return []
```

**Time Complexity:** O(log N) for Elasticsearch query, where N is number of POIs

**Example Usage:**

```
results = search_pois(
  latitude=37.7749,
  longitude=-122.4194,
  radius_km=5.0,
  category="restaurant",
  min_rating=4.0,
  limit=20
)
```

---

## 4. Filtering and Ranking

### rank_pois()

**Purpose:** Rank POIs by multi-factor scoring (distance, rating, popularity).

**Parameters:**

- pois (list): List of POI objects with distance, rating, review_count
- weights (object, optional): Custom weights for ranking factors

**Returns:**

- list: Ranked list of POIs sorted by score

**Algorithm:**

```
function rank_pois(pois, weights):
  // Default weights
  if weights is null:
    weights = {
      "distance": 0.4,
      "rating": 0.4,
      "popularity": 0.2
    }
  
  max_reviews = 0
  max_distance = 0
  
  // Find max values for normalization
  for poi in pois:
    if poi.review_count > max_reviews:
      max_reviews = poi.review_count
    if poi.distance_km > max_distance:
      max_distance = poi.distance_km
  
  // Calculate scores
  ranked_pois = []
  for poi in pois:
    // Distance score (inverse: closer = higher score)
    distance_score = 1.0 / (1.0 + poi.distance_km)
    
    // Rating score (normalized to 0-1)
    rating_score = poi.rating / 5.0
    
    // Popularity score (logarithmic scale)
    popularity_score = log(poi.review_count + 1) / log(max_reviews + 1)
    
    // Combined score
    total_score = (
      weights.distance * distance_score +
      weights.rating * rating_score +
      weights.popularity * popularity_score
    )
    
    ranked_pois.append({
      "poi": poi,
      "score": total_score,
      "distance_score": distance_score,
      "rating_score": rating_score,
      "popularity_score": popularity_score
    })
  
  // Sort by score (descending)
  ranked_pois.sort(key=lambda x: x.score, reverse=true)
  
  return ranked_pois
```

**Time Complexity:** O(N log N) where N is number of POIs

**Example Usage:**

```
ranked = rank_pois(pois, weights={"distance": 0.5, "rating": 0.3, "popularity": 0.2})
```

---

## 5. Cross-Boundary Query Handling

### query_with_boundary_check()

**Purpose:** Query POIs with automatic adjacent cell handling for boundary accuracy.

**Parameters:**

- latitude (float): Search center latitude
- longitude (float): Search center longitude
- radius_km (float): Search radius
- filters (object): Additional filters (category, rating, etc.)

**Returns:**

- list: POIs within radius (no missed POIs at boundaries)

**Algorithm:**

```
function query_with_boundary_check(latitude, longitude, radius_km, filters):
  // Calculate center geohash
  precision = get_geohash_precision(radius_km)
  center_geohash = encode_geohash(latitude, longitude, precision)
  
  // Get 8 adjacent cells
  adjacent_cells = get_adjacent_geohashes(center_geohash)
  all_cells = [center_geohash] + adjacent_cells
  
  // Query Elasticsearch with all cells
  query = {
    "bool": {
      "must": [
        {"terms": {"geohash": all_cells}},
        {
          "geo_distance": {
            "distance": radius_km + "km",
            "location": {"lat": latitude, "lon": longitude}
          }
        }
      ]
    }
  }
  
  // Apply additional filters
  if filters.category is not null:
    query.bool.must.append({"term": {"category": filters.category}})
  
  if filters.min_rating is not null:
    query.bool.must.append({"range": {"rating": {"gte": filters.min_rating}}})
  
  // Execute query
  results = elasticsearch.search(index="pois", query=query)
  
  // Filter by exact distance (remove false positives from adjacent cells)
  filtered_results = []
  for hit in results.hits:
    poi_lat = hit._source.location.lat
    poi_lng = hit._source.location.lon
    distance = calculate_distance(latitude, longitude, poi_lat, poi_lng)
    
    if distance <= radius_km:
      filtered_results.append({
        "poi": hit._source,
        "distance": distance
      })
  
  return filtered_results
```

**Time Complexity:** O(log N + M) where N is POIs in cells, M is results to filter

**Example Usage:**

```
results = query_with_boundary_check(
  37.7749, -122.4194, 5.0,
  filters={"category": "restaurant", "min_rating": 4.0}
)
```

---

## 6. Review Management

### add_review()

**Purpose:** Add a new review for a POI and update aggregates.

**Parameters:**

- poi_id (string): POI identifier
- user_id (string): User identifier
- rating (int): Rating (1-5)
- text (string): Review text
- photos (list, optional): List of photo URLs

**Returns:**

- string: Review ID if successful, null otherwise

**Algorithm:**

```
function add_review(poi_id, user_id, rating, text, photos):
  try:
    review_id = generate_id("rev_")
    
    // Create review document
    review = {
      "review_id": review_id,
      "poi_id": poi_id,
      "user_id": user_id,
      "rating": rating,
      "text": text,
      "photos": photos or [],
      "created_at": current_timestamp(),
      "updated_at": current_timestamp()
    }
    
    // Store in MongoDB
    mongodb.insert("reviews", review)
    
    // Update aggregates in PostgreSQL (synchronous)
    update_poi_aggregates(poi_id)
    
    // Publish event to Kafka for Elasticsearch update (async)
    kafka.publish("poi-updates", {
      "poi_id": poi_id,
      "action": "review_added",
      "review_id": review_id
    })
    
    return review_id
  catch error:
    log_error("Failed to add review", error)
    return null
```

**Time Complexity:** O(1) for MongoDB insert + O(log N) for aggregate update

**Example Usage:**

```
review_id = add_review(
  poi_id="poi_123",
  user_id="user_456",
  rating=5,
  text="Great pizza!",
  photos=["photo1.jpg"]
)
```

---

### update_poi_aggregates()

**Purpose:** Update POI rating and review count aggregates in PostgreSQL.

**Parameters:**

- poi_id (string): POI identifier

**Returns:**

- boolean: true if successful, false otherwise

**Algorithm:**

```
function update_poi_aggregates(poi_id):
  try:
    // Calculate aggregates from MongoDB
    pipeline = [
      {"$match": {"poi_id": poi_id}},
      {
        "$group": {
          "_id": "$poi_id",
          "avg_rating": {"$avg": "$rating"},
          "review_count": {"$sum": 1}
        }
      }
    ]
    
    aggregates = mongodb.aggregate("reviews", pipeline)
    
    if aggregates is empty:
      return false
    
    // Update PostgreSQL
    postgresql.update(
      "pois",
      {"poi_id": poi_id},
      {
        "avg_rating": aggregates[0].avg_rating,
        "review_count": aggregates[0].review_count,
        "updated_at": current_timestamp()
      }
    )
    
    return true
  catch error:
    log_error("Failed to update aggregates", poi_id, error)
    return false
```

**Time Complexity:** O(M) where M is number of reviews for POI

**Example Usage:**

```
update_poi_aggregates("poi_123")
```

---

### get_reviews()

**Purpose:** Retrieve reviews for a POI, sorted by most recent.

**Parameters:**

- poi_id (string): POI identifier
- limit (int): Maximum number of reviews (default: 20)
- offset (int): Pagination offset (default: 0)

**Returns:**

- list: List of review objects

**Algorithm:**

```
function get_reviews(poi_id, limit, offset):
  try:
    // Query MongoDB
    reviews = mongodb.find(
      "reviews",
      {"poi_id": poi_id},
      sort={"created_at": -1},
      limit=limit,
      skip=offset
    )
    
    return reviews
  catch error:
    log_error("Failed to get reviews", poi_id, error)
    return []
```

**Time Complexity:** O(log N) for indexed query

**Example Usage:**

```
reviews = get_reviews("poi_123", limit=20, offset=0)
```

---

## 7. Caching Operations

### cache_search_results()

**Purpose:** Cache search results in Redis for popular queries.

**Parameters:**

- cache_key (string): Cache key (e.g., "search:9q8:restaurant:4.0:20")
- results (list): Search results to cache
- ttl_seconds (int): Time-to-live in seconds (default: 300)

**Returns:**

- boolean: true if cached successfully

**Algorithm:**

```
function cache_search_results(cache_key, results, ttl_seconds):
  try:
    redis.setex(
      key=cache_key,
      value=json_encode(results),
      seconds=ttl_seconds
    )
    return true
  catch error:
    log_error("Failed to cache results", cache_key, error)
    return false
```

**Time Complexity:** O(1)

**Example Usage:**

```
cache_search_results("search:9q8:restaurant:4.0:20", results, ttl_seconds=300)
```

---

### get_cached_results()

**Purpose:** Retrieve cached search results from Redis.

**Parameters:**

- cache_key (string): Cache key

**Returns:**

- list: Cached results or null if not found

**Algorithm:**

```
function get_cached_results(cache_key):
  try:
    cached = redis.get(cache_key)
    if cached is null:
      return null
    
    return json_decode(cached)
  catch error:
    log_error("Failed to get cached results", cache_key, error)
    return null
```

**Time Complexity:** O(1)

**Example Usage:**

```
results = get_cached_results("search:9q8:restaurant:4.0:20")
```

---

### invalidate_cache()

**Purpose:** Invalidate cache entries for a POI or region.

**Parameters:**

- pattern (string): Cache key pattern (e.g., "search:9q8:*" or "poi:poi_123")

**Returns:**

- int: Number of keys invalidated

**Algorithm:**

```
function invalidate_cache(pattern):
  try:
    keys = redis.keys(pattern)
    if keys is empty:
      return 0
    
    deleted = redis.delete(keys)
    return deleted
  catch error:
    log_error("Failed to invalidate cache", pattern, error)
    return 0
```

**Time Complexity:** O(N) where N is number of matching keys

**Example Usage:**

```
invalidated = invalidate_cache("search:9q8:*")  // Invalidate all San Francisco searches
```

---

## 8. POI Updates and Synchronization

### update_poi()

**Purpose:** Update POI information and sync to Elasticsearch asynchronously.

**Parameters:**

- poi_id (string): POI identifier
- updates (object): Fields to update (name, address, hours, etc.)

**Returns:**

- boolean: true if update initiated, false otherwise

**Algorithm:**

```
function update_poi(poi_id, updates):
  try:
    // Update PostgreSQL (synchronous)
    postgresql.update(
      "pois",
      {"poi_id": poi_id},
      {
        ...updates,
        "updated_at": current_timestamp()
      }
    )
    
    // Publish event to Kafka for Elasticsearch update (async)
    kafka.publish("poi-updates", {
      "poi_id": poi_id,
      "action": "update",
      "updates": updates
    })
    
    // Invalidate cache
    invalidate_cache("poi:" + poi_id)
    invalidate_cache("search:*")  // Invalidate all search caches (conservative)
    
    return true
  catch error:
    log_error("Failed to update POI", poi_id, error)
    return false
```

**Time Complexity:** O(1) for PostgreSQL update + O(1) for Kafka publish

**Example Usage:**

```
update_poi("poi_123", {
  "name": "Golden Gate Pizza & Pasta",
  "hours": {...}
})
```

---

### sync_poi_to_elasticsearch()

**Purpose:** Sync POI data from PostgreSQL to Elasticsearch (Kafka consumer).

**Parameters:**

- poi_id (string): POI identifier

**Returns:**

- boolean: true if sync successful

**Algorithm:**

```
function sync_poi_to_elasticsearch(poi_id):
  try:
    // Fetch POI data from PostgreSQL
    poi = get_poi_details(poi_id)
    if poi is null:
      return false
    
    // Fetch aggregates
    aggregates = postgresql.query(
      "SELECT avg_rating, review_count FROM pois WHERE poi_id = $1",
      [poi_id]
    )
    
    // Recalculate geohash if location changed
    geohash = encode_geohash(poi.latitude, poi.longitude, precision=7)
    
    // Update Elasticsearch document
    document = {
      "poi_id": poi.poi_id,
      "name": poi.name,
      "location": {
        "lat": poi.latitude,
        "lon": poi.longitude
      },
      "geohash": geohash,
      "category": poi.categories,
      "rating": aggregates.avg_rating,
      "review_count": aggregates.review_count,
      "address": poi.address,
      "hours": poi.hours,
      "updated_at": current_timestamp()
    }
    
    elasticsearch.index(
      index="pois",
      id=poi_id,
      document=document
    )
    
    return true
  catch error:
    log_error("Failed to sync POI to Elasticsearch", poi_id, error)
    return false
```

**Time Complexity:** O(1) for PostgreSQL query + O(log N) for Elasticsearch index

**Example Usage:**

```
// Called by Kafka consumer
sync_poi_to_elasticsearch("poi_123")
```

---

## 9. Geographic Sharding

### get_shard_for_geohash()

**Purpose:** Determine which shard contains a given Geohash prefix.

**Parameters:**

- geohash (string): Geohash string
- prefix_length (int): Number of prefix characters to use (default: 3)

**Returns:**

- int: Shard number (0 to num_shards-1)

**Algorithm:**

```
function get_shard_for_geohash(geohash, prefix_length):
  // Extract prefix
  prefix = geohash[0:prefix_length]
  
  // Hash prefix to shard number
  shard_num = hash(prefix) % num_shards
  
  return shard_num
```

**Time Complexity:** O(1)

**Example Usage:**

```
shard = get_shard_for_geohash("9q8yy9m", prefix_length=3)
// Returns: shard number for "9q8" prefix
```

---

### get_shards_for_query()

**Purpose:** Determine which shards to query for a geographic search.

**Parameters:**

- latitude (float): Search center latitude
- longitude (float): Search center longitude
- radius_km (float): Search radius

**Returns:**

- list: List of shard numbers to query

**Algorithm:**

```
function get_shards_for_query(latitude, longitude, radius_km):
  // Calculate geohash precision
  precision = get_geohash_precision(radius_km)
  
  // Get center geohash
  center_geohash = encode_geohash(latitude, longitude, precision)
  
  // Get adjacent cells (for boundary accuracy)
  adjacent_cells = get_adjacent_geohashes(center_geohash)
  all_cells = [center_geohash] + adjacent_cells
  
  // Get unique shards
  shards = set()
  for cell in all_cells:
    shard = get_shard_for_geohash(cell, prefix_length=3)
    shards.add(shard)
  
  return list(shards)
```

**Time Complexity:** O(9) = O(1) for fixed number of cells

**Example Usage:**

```
shards = get_shards_for_query(37.7749, -122.4194, 5.0)
// Returns: [1, 2] (shards containing center + adjacent cells)
```

---

## 10. Search Result Aggregation

### search_with_cache()

**Purpose:** Search POIs with Redis cache lookup and caching.

**Parameters:**

- latitude (float): Search center latitude
- longitude (float): Search center longitude
- radius_km (float): Search radius
- filters (object): Search filters
- limit (int): Maximum results

**Returns:**

- list: Search results

**Algorithm:**

```
function search_with_cache(latitude, longitude, radius_km, filters, limit):
  // Generate cache key
  cache_key = generate_cache_key(latitude, longitude, radius_km, filters, limit)
  
  // Check cache
  cached_results = get_cached_results(cache_key)
  if cached_results is not null:
    return cached_results
  
  // Cache miss: query Elasticsearch
  results = search_pois(latitude, longitude, radius_km, filters, limit)
  
  // Rank results
  ranked_results = rank_pois(results)
  
  // Cache results (TTL: 5 minutes for popular queries, 1 minute for rare)
  ttl = 300 if is_popular_query(cache_key) else 60
  cache_search_results(cache_key, ranked_results, ttl)
  
  return ranked_results
```

**Time Complexity:** O(1) for cache hit, O(log N) for cache miss

**Example Usage:**

```
results = search_with_cache(
  37.7749, -122.4194, 5.0,
  filters={"category": "restaurant", "min_rating": 4.0},
  limit=20
)
```

---

### calculate_distance()

**Purpose:** Calculate distance between two geographic points using Haversine formula.

**Parameters:**

- lat1 (float): First point latitude
- lon1 (float): First point longitude
- lat2 (float): Second point latitude
- lon2 (float): Second point longitude

**Returns:**

- float: Distance in kilometers

**Algorithm:**

```
function calculate_distance(lat1, lon1, lat2, lon2):
  // Earth radius in kilometers
  R = 6371.0
  
  // Convert to radians
  lat1_rad = lat1 * PI / 180.0
  lat2_rad = lat2 * PI / 180.0
  delta_lat = (lat2 - lat1) * PI / 180.0
  delta_lon = (lon2 - lon1) * PI / 180.0
  
  // Haversine formula
  a = sin(delta_lat / 2) * sin(delta_lat / 2) +
      cos(lat1_rad) * cos(lat2_rad) *
      sin(delta_lon / 2) * sin(delta_lon / 2)
  
  c = 2 * atan2(sqrt(a), sqrt(1 - a))
  
  distance = R * c
  
  return distance
```

**Time Complexity:** O(1)

**Example Usage:**

```
distance = calculate_distance(37.7749, -122.4194, 37.7849, -122.4094)
// Returns: ~1.4 km
```

---

## Summary

This pseudocode document provides implementations for all major algorithms in the Yelp/Google Maps geo-spatial search
system:

1. **Geohash Operations**: Encoding, decoding, adjacent cell calculation, precision selection
2. **POI Indexing**: Indexing new POIs in Elasticsearch and PostgreSQL
3. **Proximity Search**: Querying POIs within radius with filters
4. **Ranking**: Multi-factor scoring (distance, rating, popularity)
5. **Boundary Handling**: Querying adjacent cells for accuracy
6. **Review Management**: Adding reviews, updating aggregates
7. **Caching**: Redis caching for popular queries
8. **Synchronization**: Async updates from PostgreSQL to Elasticsearch
9. **Sharding**: Geographic sharding by Geohash prefix
10. **Search Aggregation**: Cached search with ranking

All functions are designed for sub-100ms search latency at scale (50M POIs, 11K QPS).