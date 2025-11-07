# Web Crawler - Pseudocode Implementations

This document contains detailed algorithm implementations for the Web Crawler system. The main challenge document
references these functions.

---

## Table of Contents

1. [URL Frontier (Priority Queue)](#1-url-frontier-priority-queue)
2. [Bloom Filter (Duplicate Detection)](#2-bloom-filter-duplicate-detection)
3. [Politeness Controller](#3-politeness-controller)
4. [Fetcher Worker](#4-fetcher-worker)
5. [Parser Service](#5-parser-service)
6. [Token Bucket (Rate Limiting)](#6-token-bucket-rate-limiting)
7. [Distributed Lock (Redis)](#7-distributed-lock-redis)
8. [robots.txt Handler](#8-robotstxt-handler)
9. [URL Normalization](#9-url-normalization)
10. [Cache Operations](#10-cache-operations)

---

## 1. URL Frontier (Priority Queue)

### URLFrontier class

**Purpose:** Manages the priority queue of URLs to crawl using Kafka topics.

**Key Methods:**

- `add_url(url, priority)`: Add URL to appropriate topic
- `consume_batch(size)`: Get batch of URLs to process
- `get_stats()`: Get frontier statistics

**Algorithm:**

```
class URLFrontier:
  function __init__():
    this.kafka_producer = KafkaProducer()
    this.kafka_consumer = KafkaConsumer(
      topics=['frontier-high', 'frontier-medium', 'frontier-low'],
      group_id='crawler-workers'
    )
    this.topic_map = {
      'HIGH': 'frontier-high',
      'MEDIUM': 'frontier-medium',
      'LOW': 'frontier-low'
    }
  
  function add_url(url, priority='MEDIUM'):
    // Add URL to appropriate Kafka topic based on priority
    topic = this.topic_map[priority]
    partition = hash(extract_domain(url)) % num_partitions
    
    message = {
      'url': url,
      'timestamp': current_time(),
      'priority': priority,
      'retry_count': 0
    }
    
    try:
      this.kafka_producer.send(
        topic=topic,
        key=extract_domain(url),  // Partition by domain
        value=message,
        partition=partition
      )
      return True
    catch Exception as e:
      log_error(f"Failed to add URL: {e}")
      return False
  
  function consume_batch(batch_size=100):
    // Consume batch of URLs from Kafka
    urls = []
    messages = this.kafka_consumer.poll(timeout_ms=1000, max_records=batch_size)
    
    for topic_partition, records in messages:
      for record in records:
        urls.append(record.value)
    
    return urls
  
  function commit_offset():
    // Commit processed offsets to Kafka
    this.kafka_consumer.commit()
  
  function get_stats():
    // Get frontier statistics
    return {
      'high_priority_lag': get_consumer_lag('frontier-high'),
      'medium_priority_lag': get_consumer_lag('frontier-medium'),
      'low_priority_lag': get_consumer_lag('frontier-low'),
      'total_pending': sum_all_lags()
    }
```

**Time Complexity:**

- `add_url()`: O(1) - Kafka append is constant time
- `consume_batch()`: O(batch_size) - Poll returns batch of messages

**Example Usage:**

```
frontier = URLFrontier()

// Add seed URLs with high priority
frontier.add_url('https://wikipedia.org', priority='HIGH')
frontier.add_url('https://nytimes.com', priority='HIGH')

// Add discovered links with medium priority
frontier.add_url('https://example.com/page1', priority='MEDIUM')

// Consume batch for processing
urls = frontier.consume_batch(100)
for url in urls:
  process_url(url)

frontier.commit_offset()
```

---

## 2. Bloom Filter (Duplicate Detection)

### BloomFilter class

**Purpose:** Memory-efficient probabilistic data structure for duplicate URL detection.

**Parameters:**

- `capacity`: Number of expected URLs (10 billion)
- `false_positive_rate`: Acceptable FP rate (0.01 = 1%)

**Algorithm:**

```
class BloomFilter:
  function __init__(capacity=10_000_000_000, fp_rate=0.01):
    // Calculate optimal bit array size and number of hash functions
    // Formula: m = -n * ln(p) / (ln(2)^2)
    // where n = capacity, p = fp_rate, m = bit array size
    this.capacity = capacity
    this.fp_rate = fp_rate
    this.bit_array_size = calculate_bit_array_size(capacity, fp_rate)
    this.num_hash_functions = calculate_num_hashes(capacity, this.bit_array_size)
    this.bit_array = BitArray(this.bit_array_size)
  
  function calculate_bit_array_size(n, p):
    // m = -n * ln(p) / (ln(2)^2)
    return ceil(-n * log(p) / (log(2) ** 2))
  
  function calculate_num_hashes(n, m):
    // k = (m/n) * ln(2)
    return ceil((m / n) * log(2))
  
  function add(url):
    // Add URL to Bloom Filter
    for i in range(this.num_hash_functions):
      bit_position = this.hash_function(url, i) % this.bit_array_size
      this.bit_array[bit_position] = 1
  
  function contains(url):
    // Check if URL might be in the set
    for i in range(this.num_hash_functions):
      bit_position = this.hash_function(url, i) % this.bit_array_size
      if this.bit_array[bit_position] == 0:
        return False  // Definitely not in set
    return True  // Might be in set (or false positive)
  
  function hash_function(url, seed):
    // MurmurHash3 with different seeds
    return murmurhash3(url, seed=seed)
  
  function get_stats():
    // Calculate current false positive rate
    bits_set = count_set_bits(this.bit_array)
    current_fp_rate = (1 - exp(-this.num_hash_functions * bits_set / this.bit_array_size)) ** this.num_hash_functions
    
    return {
      'capacity': this.capacity,
      'bit_array_size': this.bit_array_size,
      'num_hash_functions': this.num_hash_functions,
      'bits_set': bits_set,
      'fill_ratio': bits_set / this.bit_array_size,
      'expected_fp_rate': this.fp_rate,
      'current_fp_rate': current_fp_rate
    }
```

**Time Complexity:**

- `add()`: O(k) where k = number of hash functions (typically 3-7)
- `contains()`: O(k)

**Space Complexity:** O(m) where m = bit array size

**Example Usage:**

```
bloom = BloomFilter(capacity=10_000_000_000, fp_rate=0.01)

// Add URLs
bloom.add('https://example.com/page1')
bloom.add('https://example.com/page2')

// Check for duplicates
if bloom.contains('https://example.com/page1'):
  // Might be duplicate (need to confirm with Redis)
  if redis.exists(hash('https://example.com/page1')):
    // Confirmed duplicate
    skip_url()

// Get statistics
stats = bloom.get_stats()
print(f"Fill ratio: {stats['fill_ratio']:.2%}")
print(f"Current FP rate: {stats['current_fp_rate']:.2%}")
```

---

## 3. Politeness Controller

### PolitenessController class

**Purpose:** Enforces robots.txt rules and rate limiting per domain.

**Algorithm:**

```
class PolitenessController:
  function __init__():
    this.redis = RedisClient()
    this.local_cache = LRUCache(capacity=1000)  // Cache 1000 domains
    this.token_buckets = {}  // Per-domain token buckets
  
  function can_fetch(url):
    // Check if URL can be fetched (politeness check)
    domain = extract_domain(url)
    
    // Step 1: Check robots.txt
    if not this.is_allowed_by_robots(url, domain):
      log(f"URL blocked by robots.txt: {url}")
      return False
    
    // Step 2: Check rate limit
    if not this.check_rate_limit(domain):
      log(f"Rate limit exceeded for domain: {domain}")
      return False
    
    // Step 3: Acquire distributed lock
    if not this.acquire_lock(domain):
      log(f"Lock held by another worker: {domain}")
      return False
    
    return True
  
  function is_allowed_by_robots(url, domain):
    // Check if URL is allowed by robots.txt
    
    // Try local cache first (99% hit rate)
    robots_txt = this.local_cache.get(f"robots:{domain}")
    
    if robots_txt is None:
      // Cache miss - check Redis
      robots_txt = this.redis.get(f"robots:{domain}")
      
      if robots_txt is None:
        // Redis miss - fetch from server
        robots_txt = this.fetch_robots_txt(domain)
        
        // Store in Redis (24h TTL)
        this.redis.setex(f"robots:{domain}", 86400, robots_txt)
      
      // Store in local cache (5 min TTL)
      this.local_cache.set(f"robots:{domain}", robots_txt, ttl=300)
    
    // Parse robots.txt and check if URL is allowed
    parser = RobotsTxtParser(robots_txt)
    return parser.can_fetch('MyBot', url)
  
  function fetch_robots_txt(domain):
    // Fetch robots.txt from server
    try:
      response = http_get(f"https://{domain}/robots.txt", timeout=5)
      if response.status_code == 200:
        return response.text
      else:
        return ""  // No robots.txt (allow all)
    catch Exception:
      return ""  // Error fetching (allow all)
  
  function check_rate_limit(domain):
    // Check rate limit using Token Bucket
    bucket = this.get_or_create_bucket(domain)
    return bucket.consume(tokens=1)
  
  function get_or_create_bucket(domain):
    // Get or create Token Bucket for domain
    if domain not in this.token_buckets:
      // Default: 1 req/sec, burst of 10
      this.token_buckets[domain] = TokenBucket(rate=1, capacity=10)
    return this.token_buckets[domain]
  
  function acquire_lock(domain, ttl=10):
    // Acquire distributed lock for domain
    worker_id = get_worker_id()
    lock_key = f"lock:{domain}"
    
    // SET with NX (only if not exists) and EX (expiration)
    acquired = this.redis.set(lock_key, worker_id, nx=True, ex=ttl)
    return acquired
  
  function release_lock(domain):
    // Release distributed lock
    lock_key = f"lock:{domain}"
    this.redis.delete(lock_key)
```

**Time Complexity:**

- `can_fetch()`: O(1) average (cache hit), O(network) worst case (fetch robots.txt)
- `acquire_lock()`: O(1) - Single Redis operation

**Example Usage:**

```
controller = PolitenessController()

url = 'https://example.com/page1'

if controller.can_fetch(url):
  // Fetch URL
  html = fetch_html(url)
  parse_html(html)
  
  // Release lock after fetching
  controller.release_lock('example.com')
else:
  // Cannot fetch - skip or retry later
  requeue_url(url)
```

---

## 4. Fetcher Worker

### CrawlerWorker class

**Purpose:** Asynchronous worker that fetches URLs using async I/O.

**Algorithm:**

```
class CrawlerWorker:
  function __init__():
    this.frontier = URLFrontier()
    this.politeness = PolitenessController()
    this.bloom_filter = BloomFilter()
    this.redis = RedisClient()
    this.http_client = AsyncHTTPClient()
    this.parser = ParserService()
  
  async function run():
    // Main worker loop
    while True:
      // Consume batch of URLs from Kafka
      urls = this.frontier.consume_batch(batch_size=100)
      
      // Process URLs concurrently
      tasks = [this.process_url(url) for url in urls]
      await asyncio.gather(*tasks)
      
      // Commit offset after processing
      this.frontier.commit_offset()
  
  async function process_url(url_data):
    // Process single URL
    url = url_data['url']
    
    try:
      // Step 1: Check duplicate
      if this.is_duplicate(url):
        log(f"Duplicate URL: {url}")
        return
      
      // Step 2: Check politeness
      if not this.politeness.can_fetch(url):
        log(f"Politeness check failed: {url}")
        this.requeue_with_delay(url)
        return
      
      // Step 3: Fetch HTML
      html = await this.fetch_with_retry(url)
      
      // Step 4: Parse HTML
      result = await this.parser.parse(url, html)
      
      // Step 5: Store results
      await this.store_results(url, html, result)
      
      // Step 6: Add to duplicate filter
      this.mark_as_crawled(url)
      
      // Step 7: Add discovered links to frontier
      for link in result['links']:
        this.frontier.add_url(link, priority='MEDIUM')
      
      // Step 8: Release lock
      this.politeness.release_lock(extract_domain(url))
      
    catch Exception as e:
      log_error(f"Error processing URL {url}: {e}")
      this.handle_error(url, e)
  
  function is_duplicate(url):
    // Check if URL already crawled (Bloom Filter + Redis)
    if not this.bloom_filter.contains(url):
      return False  // Definitely not crawled
    
    // Bloom Filter says "maybe" - confirm with Redis
    url_hash = hash(url)
    return this.redis.exists(f"crawled:{url_hash}")
  
  async function fetch_with_retry(url, max_retries=3):
    // Fetch HTML with exponential backoff retry
    for attempt in range(max_retries):
      try:
        response = await this.http_client.get(
          url,
          timeout=5,
          headers={'User-Agent': 'MyBot/1.0'}
        )
        
        if response.status == 200:
          return response.text
        elif response.status >= 500:
          // Server error - retry
          if attempt < max_retries - 1:
            await asyncio.sleep(2 ** attempt)  // 1s, 2s, 4s
            continue
          else:
            throw Exception(f"HTTP {response.status} after {max_retries} retries")
        else:
          // Client error (4xx) - don't retry
          throw Exception(f"HTTP {response.status}")
      
      catch TimeoutError:
        if attempt < max_retries - 1:
          await asyncio.sleep(2 ** attempt)
          continue
        else:
          throw Exception(f"Timeout after {max_retries} retries")
    
    throw Exception("Max retries exceeded")
  
  async function store_results(url, html, parsed_result):
    // Store raw HTML to S3 and parsed text to Elasticsearch
    
    // Compress HTML
    compressed_html = gzip.compress(html.encode('utf-8'))
    
    // Store to S3
    s3_key = f"{current_date()}/{hash(url)}.html.gz"
    await this.s3_client.put_object(bucket='crawler-data', key=s3_key, body=compressed_html)
    
    // Index to Elasticsearch
    doc = {
      'url': url,
      'title': parsed_result['title'],
      'content': parsed_result['text'],
      'crawl_timestamp': current_time(),
      'domain': extract_domain(url),
      'outgoing_links_count': len(parsed_result['links'])
    }
    await this.es_client.index(index='web_pages', document=doc)
  
  function mark_as_crawled(url):
    // Add URL to Bloom Filter and Redis
    this.bloom_filter.add(url)
    url_hash = hash(url)
    this.redis.set(f"crawled:{url_hash}", 1, ex=2592000)  // 30 days TTL
```

**Time Complexity:**

- `process_url()`: O(network + parsing) - Dominated by network I/O

**Example Usage:**

```
// Run multiple workers in parallel
async function main():
  workers = [CrawlerWorker() for _ in range(10)]
  tasks = [worker.run() for worker in workers]
  await asyncio.gather(*tasks)

asyncio.run(main())
```

---

## 5. Parser Service

### Parser class

**Purpose:** Extracts text, links, and metadata from HTML.

**Algorithm:**

```
class ParserService:
  function __init__():
    this.link_filter = LinkFilter()
  
  function parse(url, html):
    // Parse HTML and extract content
    
    try:
      soup = BeautifulSoup(html, 'lxml')
      
      return {
        'text': this.extract_text(soup),
        'links': this.extract_links(soup, url),
        'title': this.extract_title(soup),
        'metadata': this.extract_metadata(soup)
      }
    catch Exception as e:
      log_error(f"Parse error for {url}: {e}")
      return {'text': '', 'links': [], 'title': '', 'metadata': {}}
  
  function extract_text(soup):
    // Extract visible text from HTML
    
    // Remove script and style elements
    for script in soup(['script', 'style', 'noscript']):
      script.decompose()
    
    // Get text
    text = soup.get_text(separator=' ', strip=True)
    
    // Clean whitespace
    lines = [line.strip() for line in text.split('\n')]
    text = '\n'.join([line for line in lines if line])
    
    return text
  
  function extract_links(soup, base_url):
    // Extract and normalize all links
    links = []
    
    for anchor in soup.find_all('a', href=True):
      href = anchor['href']
      
      // Normalize URL (relative → absolute)
      absolute_url = urljoin(base_url, href)
      
      // Filter out non-HTTP(S) links
      if not absolute_url.startswith(('http://', 'https://')):
        continue
      
      // Filter out non-HTML links (PDFs, images, etc.)
      if not this.link_filter.is_html_link(absolute_url):
        continue
      
      // Normalize URL (remove fragment, lowercase domain)
      normalized_url = normalize_url(absolute_url)
      
      links.append(normalized_url)
    
    // Remove duplicates
    return list(set(links))
  
  function extract_title(soup):
    // Extract page title
    title_tag = soup.find('title')
    if title_tag:
      return title_tag.get_text().strip()
    
    // Fallback to h1
    h1_tag = soup.find('h1')
    if h1_tag:
      return h1_tag.get_text().strip()
    
    return ''
  
  function extract_metadata(soup):
    // Extract metadata (description, keywords, etc.)
    metadata = {}
    
    // Description
    meta_desc = soup.find('meta', attrs={'name': 'description'})
    if meta_desc and meta_desc.get('content'):
      metadata['description'] = meta_desc['content']
    
    // Keywords
    meta_keywords = soup.find('meta', attrs={'name': 'keywords'})
    if meta_keywords and meta_keywords.get('content'):
      metadata['keywords'] = meta_keywords['content']
    
    // Author
    meta_author = soup.find('meta', attrs={'name': 'author'})
    if meta_author and meta_author.get('content'):
      metadata['author'] = meta_author['content']
    
    // Language
    html_tag = soup.find('html')
    if html_tag and html_tag.get('lang'):
      metadata['language'] = html_tag['lang']
    
    return metadata

class LinkFilter:
  function is_html_link(url):
    // Check if link is likely an HTML page
    
    // Exclude common non-HTML extensions
    excluded_extensions = [
      '.pdf', '.jpg', '.jpeg', '.png', '.gif', '.mp4', '.mp3',
      '.zip', '.tar', '.gz', '.exe', '.dmg', '.doc', '.docx',
      '.xls', '.xlsx', '.ppt', '.pptx', '.csv', '.xml', '.json'
    ]
    
    url_lower = url.lower()
    for ext in excluded_extensions:
      if url_lower.endswith(ext):
        return False
    
    return True
```

**Time Complexity:**

- `parse()`: O(n) where n = HTML size
- `extract_links()`: O(k) where k = number of links

**Example Usage:**

```
parser = ParserService()

url = 'https://example.com/page1'
html = fetch_html(url)

result = parser.parse(url, html)

print(f"Title: {result['title']}")
print(f"Text length: {len(result['text'])} chars")
print(f"Links found: {len(result['links'])}")

// Add discovered links to frontier
for link in result['links']:
  frontier.add_url(link, priority='MEDIUM')
```

---

## 6. Token Bucket (Rate Limiting)

### TokenBucket class

**Purpose:** Implements token bucket algorithm for rate limiting.

**Algorithm:**

```
class TokenBucket:
  function __init__(rate=1, capacity=10):
    // rate: tokens per second
    // capacity: maximum burst size
    this.rate = rate
    this.capacity = capacity
    this.tokens = capacity
    this.last_refill = current_time()
  
  function consume(tokens=1):
    // Try to consume tokens
    this.refill()
    
    if this.tokens >= tokens:
      this.tokens -= tokens
      return True
    else:
      return False
  
  function refill():
    // Refill tokens based on elapsed time
    now = current_time()
    elapsed = now - this.last_refill
    
    // Calculate new tokens
    new_tokens = elapsed * this.rate
    this.tokens = min(this.capacity, this.tokens + new_tokens)
    this.last_refill = now
  
  function wait_time():
    // Calculate wait time until next token available
    if this.tokens >= 1:
      return 0
    
    tokens_needed = 1 - this.tokens
    return tokens_needed / this.rate
  
  function get_stats():
    // Get current state
    this.refill()
    return {
      'tokens_available': this.tokens,
      'capacity': this.capacity,
      'rate': this.rate,
      'fill_ratio': this.tokens / this.capacity
    }
```

**Time Complexity:** O(1) for all operations

**Example Usage:**

```
// Create bucket: 1 req/sec, burst of 10
bucket = TokenBucket(rate=1, capacity=10)

// Try to consume token
if bucket.consume(1):
  // Token available - proceed with request
  fetch_url()
else:
  // No tokens - wait
  wait_time = bucket.wait_time()
  sleep(wait_time)
  fetch_url()

// Burst example
for i in range(15):
  if bucket.consume(1):
    print(f"Request {i+1}: OK")
  else:
    print(f"Request {i+1}: Rate limited")
    sleep(1)  // Wait for refill

// Output:
// Request 1-10: OK (burst)
// Request 11-15: Rate limited (wait for refill)
```

---

## 7. Distributed Lock (Redis)

### DistributedLock class

**Purpose:** Implements distributed locking using Redis.

**Algorithm:**

```
class DistributedLock:
  function __init__(redis_client):
    this.redis = redis_client
    this.worker_id = get_worker_id()
  
  function acquire(resource_name, ttl=10):
    // Acquire lock for resource
    lock_key = f"lock:{resource_name}"
    
    // SET with NX (only if not exists) and EX (expiration)
    acquired = this.redis.set(
      lock_key,
      this.worker_id,
      nx=True,  // Only set if key doesn't exist
      ex=ttl    // Expire after ttl seconds
    )
    
    if acquired:
      log(f"Lock acquired: {resource_name} by {this.worker_id}")
      return True
    else:
      log(f"Lock held by another worker: {resource_name}")
      return False
  
  function release(resource_name):
    // Release lock (only if we own it)
    lock_key = f"lock:{resource_name}"
    
    // Lua script for atomic check-and-delete
    lua_script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
    """
    
    released = this.redis.eval(lua_script, 1, lock_key, this.worker_id)
    
    if released:
      log(f"Lock released: {resource_name} by {this.worker_id}")
      return True
    else:
      log(f"Lock not owned: {resource_name}")
      return False
  
  function with_lock(resource_name, ttl=10):
    // Context manager for automatic lock release
    acquired = this.acquire(resource_name, ttl)
    
    try:
      if acquired:
        yield True
      else:
        yield False
    finally:
      if acquired:
        this.release(resource_name)
```

**Time Complexity:** O(1) for all operations

**Example Usage:**

```
lock = DistributedLock(redis_client)

// Manual lock management
if lock.acquire('example.com', ttl=10):
  try:
    fetch_url('https://example.com/page1')
  finally:
    lock.release('example.com')

// Context manager (automatic release)
with lock.with_lock('example.com') as acquired:
  if acquired:
    fetch_url('https://example.com/page1')
  else:
    print("Could not acquire lock, skipping...")
```

---

## 8. robots.txt Handler

### RobotsTxtParser class

**Purpose:** Parses and enforces robots.txt rules.

**Algorithm:**

```
class RobotsTxtParser:
  function __init__(robots_txt_content):
    this.rules = this.parse_robots_txt(robots_txt_content)
    this.crawl_delay = this.extract_crawl_delay()
  
  function parse_robots_txt(content):
    // Parse robots.txt into rules
    rules = {
      'allow': [],
      'disallow': []
    }
    
    current_user_agent = None
    lines = content.split('\n')
    
    for line in lines:
      line = line.strip()
      
      // Skip comments and empty lines
      if line.startswith('#') or not line:
        continue
      
      // Parse directives
      if ':' in line:
        directive, value = line.split(':', 1)
        directive = directive.strip().lower()
        value = value.strip()
        
        if directive == 'user-agent':
          current_user_agent = value
        elif directive == 'disallow' and current_user_agent == '*':
          if value:
            rules['disallow'].append(value)
        elif directive == 'allow' and current_user_agent == '*':
          if value:
            rules['allow'].append(value)
        elif directive == 'crawl-delay' and current_user_agent == '*':
          this.crawl_delay = float(value)
    
    return rules
  
  function can_fetch(user_agent, url):
    // Check if URL is allowed for user agent
    path = extract_path(url)  // e.g., /admin/page.html
    
    // Check allow rules first (more specific)
    for allow_pattern in this.rules['allow']:
      if this.matches_pattern(path, allow_pattern):
        return True
    
    // Check disallow rules
    for disallow_pattern in this.rules['disallow']:
      if this.matches_pattern(path, disallow_pattern):
        return False
    
    // Default: allow
    return True
  
  function matches_pattern(path, pattern):
    // Check if path matches robots.txt pattern
    
    // Exact match
    if pattern == path:
      return True
    
    // Prefix match (e.g., /admin/ matches /admin/page.html)
    if pattern.endswith('/') and path.startswith(pattern):
      return True
    
    // Wildcard support (simple)
    if '*' in pattern:
      regex = pattern.replace('*', '.*')
      return re.match(regex, path) is not None
    
    return False
  
  function extract_crawl_delay():
    // Extract crawl-delay directive
    return this.rules.get('crawl_delay', 1)  // Default: 1 second
```

**Time Complexity:**

- `parse_robots_txt()`: O(n) where n = number of lines
- `can_fetch()`: O(k) where k = number of rules

**Example Usage:**

```
robots_txt = """
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/
Crawl-delay: 1
"""

parser = RobotsTxtParser(robots_txt)

// Check URLs
print(parser.can_fetch('MyBot', 'https://example.com/page1'))        // True
print(parser.can_fetch('MyBot', 'https://example.com/admin/'))       // False
print(parser.can_fetch('MyBot', 'https://example.com/public/data'))  // True

// Get crawl delay
print(f"Crawl delay: {parser.crawl_delay} seconds")
```

---

## 9. URL Normalization

### normalize_url() function

**Purpose:** Normalize URLs to canonical form for deduplication.

**Algorithm:**

```
function normalize_url(url):
  // Normalize URL to canonical form
  
  // Step 1: Parse URL
  parsed = urlparse(url)
  
  // Step 2: Lowercase scheme and domain
  scheme = parsed.scheme.lower()
  domain = parsed.netloc.lower()
  
  // Step 3: Remove default ports
  if ':80' in domain and scheme == 'http':
    domain = domain.replace(':80', '')
  if ':443' in domain and scheme == 'https':
    domain = domain.replace(':443', '')
  
  // Step 4: Remove fragment (#section)
  // Fragments are client-side only, don't affect server response
  path = parsed.path
  query = parsed.query
  
  // Step 5: Normalize path
  path = normalize_path(path)
  
  // Step 6: Sort query parameters (for consistency)
  if query:
    params = parse_qs(query)
    sorted_params = sorted(params.items())
    query = urlencode(sorted_params)
  
  // Step 7: Rebuild URL
  normalized = f"{scheme}://{domain}{path}"
  if query:
    normalized += f"?{query}"
  
  return normalized

function normalize_path(path):
  // Normalize path component
  
  // Remove duplicate slashes
  path = re.sub(r'/+', '/', path)
  
  // Add leading slash if missing
  if not path.startswith('/'):
    path = '/' + path
  
  // Remove trailing slash (except for root)
  if path != '/' and path.endswith('/'):
    path = path[:-1]
  
  // Resolve . and .. (relative path components)
  parts = path.split('/')
  stack = []
  
  for part in parts:
    if part == '..':
      if stack:
        stack.pop()
    elif part != '.' and part != '':
      stack.append(part)
  
  if not stack:
    return '/'
  
  return '/' + '/'.join(stack)
```

**Time Complexity:** O(n) where n = URL length

**Example Usage:**

```
// Example normalizations
normalize_url('HTTP://Example.Com/Page')
// → 'http://example.com/page'

normalize_url('https://example.com:443/path//page/')
// → 'https://example.com/path/page'

normalize_url('http://example.com:80/path#section')
// → 'http://example.com/path'

normalize_url('https://example.com/./a/../b/./c')
// → 'https://example.com/b/c'

normalize_url('http://example.com?z=3&a=1&b=2')
// → 'http://example.com?a=1&b=2&z=3'
```

---

## 10. Cache Operations

### LRUCache class

**Purpose:** Implements LRU (Least Recently Used) cache for local caching.

**Algorithm:**

```
class LRUCache:
  function __init__(capacity):
    this.capacity = capacity
    this.cache = {}  // key → (value, timestamp)
    this.access_order = []  // List of keys in access order
  
  function get(key):
    // Get value from cache
    if key in this.cache:
      // Move to end (most recently used)
      this.access_order.remove(key)
      this.access_order.append(key)
      
      return this.cache[key][0]  // Return value
    
    return None
  
  function set(key, value, ttl=None):
    // Set value in cache
    
    // If key exists, update and move to end
    if key in this.cache:
      this.access_order.remove(key)
    
    // If at capacity, evict least recently used
    elif len(this.cache) >= this.capacity:
      lru_key = this.access_order.pop(0)
      del this.cache[lru_key]
    
    // Add to cache
    timestamp = current_time()
    expiry = timestamp + ttl if ttl else None
    this.cache[key] = (value, expiry)
    this.access_order.append(key)
  
  function has(key):
    // Check if key exists and not expired
    if key not in this.cache:
      return False
    
    value, expiry = this.cache[key]
    
    // Check expiry
    if expiry and current_time() > expiry:
      // Expired - remove
      this.access_order.remove(key)
      del this.cache[key]
      return False
    
    return True
  
  function evict_expired():
    // Evict all expired entries
    now = current_time()
    expired_keys = []
    
    for key, (value, expiry) in this.cache.items():
      if expiry and now > expiry:
        expired_keys.append(key)
    
    for key in expired_keys:
      this.access_order.remove(key)
      del this.cache[key]
  
  function get_stats():
    // Get cache statistics
    return {
      'size': len(this.cache),
      'capacity': this.capacity,
      'fill_ratio': len(this.cache) / this.capacity,
      'oldest_key': this.access_order[0] if this.access_order else None,
      'newest_key': this.access_order[-1] if this.access_order else None
    }
```

**Time Complexity:**

- `get()`: O(n) due to list.remove() (could be O(1) with OrderedDict)
- `set()`: O(n) due to list.remove()

**Example Usage:**

```
// Create cache with capacity of 1000 entries
cache = LRUCache(capacity=1000)

// Store robots.txt in cache (5 min TTL)
cache.set('robots:example.com', robots_txt_content, ttl=300)

// Retrieve from cache
robots_txt = cache.get('robots:example.com')
if robots_txt:
  print("Cache hit!")
else:
  print("Cache miss - fetch from Redis")

// Check if exists
if cache.has('robots:example.com'):
  print("Robots.txt cached")

// Get stats
stats = cache.get_stats()
print(f"Cache fill ratio: {stats['fill_ratio']:.2%}")
```

---

## Anti-Pattern Examples

### ❌ Anti-Pattern 1: Not Checking Duplicates

**Bad:**

```
function crawl_url(url):
  // ❌ No duplicate check - wastes resources
  html = fetch_html(url)
  parse_html(html)
```

**Good:**

```
function crawl_url(url):
  // ✅ Check duplicates before fetching
  if bloom_filter.contains(url) and redis.exists(hash(url)):
    return  // Already crawled
  
  html = fetch_html(url)
  parse_html(html)
  
  // Mark as crawled
  bloom_filter.add(url)
  redis.set(hash(url), 1)
```

---

### ❌ Anti-Pattern 2: Ignoring robots.txt

**Bad:**

```
function fetch_url(url):
  // ❌ Ignores robots.txt - ethical violation
  return http_get(url)
```

**Good:**

```
function fetch_url(url):
  // ✅ Check robots.txt first
  domain = extract_domain(url)
  robots_txt = get_robots_txt(domain)
  
  if not robots_parser.can_fetch('MyBot', url):
    log(f"Blocked by robots.txt: {url}")
    return None
  
  return http_get(url)
```

---

### ❌ Anti-Pattern 3: Synchronous Fetching

**Bad:**

```
function process_urls(urls):
  // ❌ Synchronous - 1.6 URLs/sec (600ms per URL)
  for url in urls:
    html = fetch_html(url)  // Blocks for 500ms
    parse_html(html)         // Blocks for 100ms
```

**Good:**

```
async function process_urls(urls):
  // ✅ Asynchronous - 200 URLs/sec (100 concurrent)
  tasks = [fetch_and_parse_async(url) for url in urls]
  await asyncio.gather(*tasks)
```

