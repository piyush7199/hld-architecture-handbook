# URL Shortener - Pseudocode Implementations

This document contains detailed algorithm implementations for the URL Shortener system. The main challenge document
references these functions.

---

## Table of Contents

1. [Base62 Encoding/Decoding](#base62-encodingdecoding)
2. [URL Shortening Service](#url-shortening-service)
3. [Redirection Service](#redirection-service)
4. [Analytics Tracking](#analytics-tracking)
5. [Cache Stampede Prevention](#cache-stampede-prevention)
6. [Rate Limiting](#rate-limiting)
7. [URL Validation](#url-validation)

---

## Base62 Encoding/Decoding

### base62_encode()

Converts a numeric ID to a Base62-encoded string.

**Parameters:**

- `num` (integer): The numeric ID to encode

**Returns:**

- `string`: Base62-encoded representation

**Algorithm:**

```
Base62Encoder:
  ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
  BASE = 62
  
  function encode(num):
    if num == 0:
      return ALPHABET[0]
    
    result = empty_array
    while num > 0:
      result.append(ALPHABET[num % BASE])
      num = num / BASE  // integer division
    
    return reverse(result)
```

**Example:**

```
Input: 123456789
Output: "8M0kX"
```

### base62_decode()

Converts a Base62-encoded string back to numeric ID.

**Parameters:**

- `encoded` (string): The Base62 string to decode

**Returns:**

- `integer`: Original numeric ID

**Algorithm:**

```
Base62Encoder:
  function decode(encoded):
    num = 0
    for each char in encoded:
      num = num * BASE + index_of(char in ALPHABET)
    return num
```

**Example:**

```
Input: "8M0kX"
Output: 123456789
```

---

## URL Shortening Service

### shorten_url()

Main function for creating a short URL from a long URL.

**Parameters:**

- `request`: Object containing `long_url` and optionally `custom_alias`, `expires_at`

**Returns:**

- Object with `short_url` and `long_url`, or error

**Algorithm:**

```
function shorten_url(request):
  long_url = request.long_url
  
  // Handle custom alias
  if request.custom_alias:
    short_alias = request.custom_alias
    
    // Check availability in cache
    if cache.exists("url:" + short_alias):
      return error("Alias already taken")
    
    // Attempt atomic insert in database
    try:
      db.insert(short_alias, long_url, created_at, expires_at)
    catch UniqueConstraintViolation:
      return error("Alias already taken")
  
  else:
    // Generate alias from sequential ID
    unique_id = id_generator.get_next_id()
    short_alias = base62_encode(unique_id)
    
    // Store in database
    db.insert(short_alias, long_url, created_at, expires_at)
  
  // Cache the mapping (Cache-Aside write)
  cache.set("url:" + short_alias, long_url, TTL=24_hours)
  
  // Return short URL
  return {
    short_url: "https://short.ly/" + short_alias,
    long_url: long_url
  }
```

---

## Redirection Service

### handle_redirect()

Handles incoming requests to short URLs and redirects to long URLs.

**Parameters:**

- `short_alias` (string): The short URL alias

**Returns:**

- HTTP redirect response (302) or error (404/410)

**Algorithm:**

```
function handle_redirect(short_alias):
  // 1. Check cache first (Redis)
  long_url = cache.get("url:" + short_alias)
  
  if long_url is null:
    // Cache miss - query database
    result = db.query("SELECT long_url, expires_at, status FROM urls WHERE short_alias = ?", short_alias)
    
    if result is empty:
      return error_404("URL not found")
    
    // Check if expired
    if result.expires_at < current_time():
      return error_404("URL expired")
    
    // Check status
    if result.status != "ACTIVE":
      return error_404("URL not active")
    
    long_url = result.long_url
    
    // Store in cache (Cache-Aside pattern)
    cache.set("url:" + short_alias, long_url, TTL=24_hours)
  
  // 2. Track analytics asynchronously (non-blocking)
  async_track_click(short_alias, user_agent, ip, referer)
  
  // 3. Redirect to long URL (HTTP 302)
  return redirect_302(long_url)
```

---

## Analytics Tracking

### async_track_click()

Asynchronously tracks click events for analytics.

**Parameters:**

- `short_alias` (string): The short URL that was clicked
- `user_agent` (string): Browser/client user agent
- `ip` (string): Client IP address
- `referer` (string): Referring URL

**Returns:**

- None (fire-and-forget)

**Algorithm:**

```
function async_track_click(short_alias, user_agent, ip, referer):
  // Publish to message queue for async processing
  event = {
    short_alias: short_alias,
    timestamp: current_timestamp(),
    user_agent: user_agent,
    ip: ip,
    referer: referer
  }
  
  // Push to Kafka or Redis Stream
  message_queue.publish("click_events", event)
  
  // Increment real-time counter
  cache.increment("clicks:" + short_alias)
```

---

## Cache Stampede Prevention

### get_url_with_lock()

Fetches URL with distributed locking to prevent cache stampede.

**Parameters:**

- `short_alias` (string): The short URL alias

**Returns:**

- `long_url` (string): The long URL, or null if not found

**Algorithm:**

```
function get_url_with_lock(short_alias):
  url = cache.get(short_alias)
  
  if url is null:
    // Acquire distributed lock
    lock_key = "lock:url:" + short_alias
    acquire_lock(lock_key, timeout=10_seconds):
      // Double-check pattern
      url = cache.get(short_alias)
      if url is null:
        url = db.query(short_alias)
        cache.set(short_alias, url, TTL=3600)
    release_lock(lock_key)
  
  return url
```

### get_url_with_probabilistic_refresh()

Alternative: Probabilistic early expiration to prevent stampede.

**Parameters:**

- `short_alias` (string): The short URL alias

**Returns:**

- `long_url` (string): The long URL

**Algorithm:**

```
function get_url_with_early_refresh(short_alias):
  url, ttl_remaining = cache.get_with_ttl(short_alias)
  
  if url is null:
    url = fetch_and_cache(short_alias)
  else if ttl_remaining < 300:  // Less than 5 minutes left
    // Probabilistically refresh (10% chance)
    if random() < 0.1:
      // Async refresh in background
      async_execute(refresh_cache(short_alias))
  
  return url
```

---

## Rate Limiting

### check_rate_limit()

Implements rate limiting using token bucket algorithm.

**Parameters:**

- `user_id` (integer or null): Authenticated user ID
- `client_ip` (string): Client IP address

**Returns:**

- `boolean`: True if within rate limit, False if exceeded

**Algorithm:**

```
function check_rate_limit(user_id, client_ip):
  // Rate limit by IP (anonymous users)
  ip_count = cache.increment("rate_limit:ip:" + client_ip)
  cache.expire("rate_limit:ip:" + client_ip, 3600)  // 1 hour
  
  if ip_count > 10:  // 10 URLs per hour per IP
    return false
  
  // Additional per-user limit (authenticated users)
  if user_id:
    user_count = cache.increment("rate_limit:user:" + user_id)
    cache.expire("rate_limit:user:" + user_id, 3600)
    
    if user_count > 100:  // 100 per hour for authenticated users
      return false
  
  return true
```

---

## URL Validation

### validate_url()

Validates and sanitizes input URLs to prevent security issues.

**Parameters:**

- `url` (string): The URL to validate

**Returns:**

- `boolean`: True if valid, False otherwise

**Algorithm:**

```
ALLOWED_SCHEMES = ["http", "https"]
BLOCKED_DOMAINS = ["localhost", "127.0.0.1", "0.0.0.0"]

function validate_url(url):
  // Check URL format
  try:
    parsed = parse_url(url)
  catch:
    return false
  
  // Validate scheme (XSS protection)
  if parsed.scheme not in ALLOWED_SCHEMES:
    return false
  
  // Block internal/localhost URLs (SSRF protection)
  hostname = parsed.hostname
  if hostname in BLOCKED_DOMAINS or hostname.starts_with("192.168."):
    return false
  
  // Check URL length
  if length(url) > 2048:
    return false
  
  // Optional: Check if URL is reachable
  try:
    response = http_head(url, timeout=5_seconds)
    return response.status_code < 400
  catch:
    return false
  
  return true
```

### shorten_with_validation()

URL shortening with validation.

**Parameters:**

- `long_url` (string): URL to shorten
- `user_id` (integer or null): User ID
- `client_ip` (string): Client IP

**Returns:**

- `short_alias` (string) or error

**Algorithm:**

```
function shorten_with_validation(long_url, user_id, client_ip):
  // 1. Check rate limit
  if not check_rate_limit(user_id, client_ip):
    return error_429("Rate limit exceeded")
  
  // 2. Validate URL
  if not validate_url(long_url):
    return error_400("Invalid URL")
  
  // 3. Generate short alias
  unique_id = id_generator.get_next_id()
  short_alias = base62_encode(unique_id)
  
  // 4. Store in database
  db.insert(short_alias, long_url)
  
  return short_alias
```

---

## Helper Functions

### cleanup_expired_links()

Background job to clean up expired URLs.

**Parameters:**

- None

**Returns:**

- None

**Algorithm:**

```
function cleanup_expired_links():
  expired_links = db.query(
    "SELECT short_alias FROM urls WHERE expires_at < NOW() AND status = 'ACTIVE'"
  )
  
  for each alias in expired_links:
    db.update("UPDATE urls SET status = 'EXPIRED' WHERE short_alias = ?", alias)
    cache.delete(alias)  // Remove from cache
  
  log_info("Cleaned up " + count(expired_links) + " expired links")
```

### graceful_cache_fallback()

Handles cache failures gracefully without affecting redirects.

**Parameters:**

- `short_alias` (string): The short URL alias

**Returns:**

- `long_url` (string): The long URL, or null if not found

**Algorithm:**

```
function redirect_with_fallback(short_alias):
  try:
    url = cache.get("url:" + short_alias)
    if url:
      return redirect_302(url)
  catch CacheError as e:
    log_error("Cache error: " + e)
    // Fall through to database
  
  // Cache miss or Redis down - query database
  url = db.query(short_alias)
  
  if url is null:
    return error_404("URL not found")
  
  // Try to cache, but don't fail if Redis is down
  try:
    cache.set("url:" + short_alias, url, TTL=24_hours)
  catch CacheError:
    // Continue without caching (degraded mode)
    pass
  
  return redirect_302(url)
```

---

## Anti-Pattern Examples (What NOT to do)

### ❌ create_custom_alias_bad()

**Problem:** Race condition - doesn't use atomic database constraints.

```
function create_custom_alias_bad(alias, long_url):
  // Check if exists
  if cache.exists("url:" + alias):
    return "Alias taken"
  
  // RACE CONDITION! Another request could insert between check and write
  db.insert(alias, long_url)
  cache.set("url:" + alias, long_url)
```

### ✅ create_custom_alias_good()

**Solution:** Use database atomic constraints.

```
function create_custom_alias_good(alias, long_url):
  try:
    // Let database enforce uniqueness atomically
    db.execute("INSERT INTO urls (short_alias, long_url) VALUES (?, ?)", alias, long_url)
    cache.set("url:" + alias, long_url, TTL=24_hours)
    return "Success"
  catch UniqueConstraintViolation:
    return "Alias already taken"
```

### ❌ redirect_with_blocking_analytics_bad()

**Problem:** Analytics blocking redirect response.

```
function redirect_bad(short_alias):
  long_url = get_from_cache_or_db(short_alias)
  
  // BLOCKING call! User waits 50ms+ extra
  analytics_service.track_click(short_alias)
  
  return redirect_302(long_url)
```

### ✅ redirect_with_async_analytics_good()

**Solution:** Fire-and-forget async tracking.

```
function redirect_good(short_alias):
  long_url = get_from_cache_or_db(short_alias)
  
  // Non-blocking - fire and forget
  async_execute(track_click_async(short_alias))
  
  return redirect_302(long_url)  // Immediate redirect
```

---

## Performance Optimization Examples

### batch_url_creation()

Create multiple short URLs efficiently.

```
function batch_create_urls(long_urls_array):
  // Get batch of IDs at once
  id_batch = id_generator.get_batch(count=length(long_urls_array))
  
  // Encode all at once
  short_aliases = empty_array
  for i from 0 to length(long_urls_array):
    short_aliases[i] = base62_encode(id_batch[i])
  
  // Batch insert to database
  db.batch_insert(short_aliases, long_urls_array)
  
  // Batch cache write
  cache.batch_set(short_aliases, long_urls_array, TTL=24_hours)
  
  return short_aliases
```

### connection_pooling()

Efficient cache connection management.

```
// ❌ Bad: New connection per request
function get_user_bad(user_id):
  connection = create_connection('localhost', 6379)
  user = connection.get("user:" + user_id)
  connection.close()
  return user

// ✅ Good: Connection pool (reuse connections)
pool = create_connection_pool(
  host='localhost',
  port=6379,
  max_connections=50,
  socket_timeout=1_second
)
connection = get_connection_from_pool(pool)

function get_user_good(user_id):
  user = connection.get("user:" + user_id)
  return user
```