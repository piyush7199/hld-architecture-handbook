# 3.2.3 Design a Distributed Web Crawler

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow, and scaling strategy
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows for crawling, politeness checks, duplicate detection, and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices and trade-offs
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for all core functions

---

## Problem Statement

Design a distributed web crawler system capable of fetching and indexing **10 billion web pages** within 4 months while respecting politeness policies, efficiently detecting duplicates, and handling various failures (DNS errors, timeouts, malformed HTML). The system must achieve **1,000 URLs/second** throughput, use **< 100 TB storage**, and never overwhelm any single domain.

---

## 1. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement                 | Description                                                                  | Priority     |
|-----------------------------|------------------------------------------------------------------------------|--------------|
| **Fetch & Parse**           | Fetch HTML from URLs, extract text, discover new links                       | Must Have    |
| **Storage**                 | Store raw HTML (S3) and parsed content (Elasticsearch) in searchable format  | Must Have    |
| **Politeness**              | Respect robots.txt, rate limit requests per domain (1 req/sec default)       | Must Have    |
| **Duplicate Detection**     | Never process the same URL twice                                             | Must Have    |
| **Prioritization**          | Crawl important/fresh pages first (PageRank-style)                           | Should Have  |
| **Recrawl**                 | Periodically recrawl pages to detect updates                                 | Should Have  |
| **Content Type Filtering**  | Only crawl HTML pages (skip PDFs, images, videos)                            | Should Have  |

### Non-Functional Requirements (NFRs)

| Requirement             | Target              | Rationale                                  |
|-------------------------|---------------------|--------------------------------------------|
| **High Throughput**     | 1,000+ URLs/sec     | Complete 10B pages in ~115 days            |
| **Fault Tolerance**     | 99.9% uptime        | Handle DNS errors, timeouts, bad HTML      |
| **Scalability**         | Horizontal scaling  | Add more crawler workers as needed         |
| **Low Latency (Fetch)** | < 5 seconds per page| Maximize throughput                        |
| **Storage Efficiency**  | < 100 TB total      | Cost-effective storage                     |

### Scale Estimation

| Metric                  | Assumption                                   | Calculation                                  | Result                                   |
|-------------------------|----------------------------------------------|----------------------------------------------|------------------------------------------|
| **Total Pages**         | Entire web                                   | 10 billion unique pages                      | **10B pages**                            |
| **Target Fetch Rate**   | Efficient crawling                           | 1,000 URLs/sec                               | **1,000 QPS**                            |
| **Pages per Day**       | 1,000 × 86,400 sec/day                       | $1000 \times 86400$                          | **~86.4M pages/day**                     |
| **Time to Crawl**       | 10B / 86.4M pages/day                        | $\frac{10 \times 10^9}{86.4 \times 10^6}$   | **~115 days (4 months)**                 |
| **Average Page Size**   | HTML + text                                  | 10 KB/page                                   | **10 KB**                                |
| **Storage Required**    | 10B × 10 KB                                  | $10 \times 10^9 \times 10 \times 10^3$ bytes | **~100 TB storage**                      |
| **Bandwidth**           | 1,000 QPS × 10 KB                            | $1000 \times 10 \times 10^3$ bytes/sec       | **~10 MB/sec (~80 Mbps)**                |
| **Bloom Filter Size**   | 10B URLs, 1% false positive rate             | 10B × 10 bits                                | **~12 GB RAM**                           |

---

## 2. High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

### Key Components

| Component               | Responsibility                       | Technology Options                  | Scalability              |
|-------------------------|--------------------------------------|-------------------------------------|--------------------------|
| **Seed URLs**           | Initial URLs to start crawling       | Configuration file, database        | -                        |
| **URL Frontier**        | Priority queue of URLs to crawl      | Kafka (partitioned by domain)       | Horizontal (partitioned) |
| **Duplicate Filter**    | Fast lookup: already crawled?        | Bloom Filter (RAM) + Redis (disk)   | Vertical (memory)        |
| **Politeness Controller**| Rate limit per domain                | Redis (distributed locks)           | Horizontal               |
| **Fetcher Workers**     | HTTP GET, download HTML              | Python (asyncio), Go (goroutines)   | Horizontal               |
| **Parser Workers**      | Extract text, links, metadata        | BeautifulSoup, goquery              | Horizontal               |
| **Content Store**       | Store raw HTML                       | S3, Google Cloud Storage            | Horizontal (object store)|
| **Search Index**        | Store parsed text (searchable)       | Elasticsearch, Solr                 | Horizontal (sharding)    |

---

## 3. Detailed Component Design

### 3.1 URL Frontier (Kafka Priority Queue)

The URL Frontier is a massive, distributed queue containing URLs waiting to be fetched. It must support:
- **Prioritization**: Important URLs (seed URLs, high PageRank) should be crawled first
- **Partitioning by Domain**: URLs from the same domain should be routed to the same partition for politeness
- **Fault Tolerance**: Persist URLs to disk to survive crashes

#### Kafka Topic Structure

```
Topic: url-frontier
Partitions: 64 (or more)
Replication Factor: 3
Message: { url, priority, domain, discovered_at }
```

**Partitioning Strategy:** Hash(domain) % num_partitions

**Why?** URLs from the same domain end up in the same partition, allowing a single worker to enforce politeness per domain.

*See `pseudocode.md::URLFrontier.enqueue()` and `pseudocode.md::URLFrontier.dequeue()` for implementation*

---

### 3.2 Duplicate Detection (Two-Tier System)

**Problem:** Checking if a URL has been crawled requires fast lookup across 10 billion URLs. Storing all URLs in Redis requires:

```
10B URLs × 100 bytes/URL = 1 TB RAM
Cost: AWS ElastiCache r5.24xlarge (~$6,000/month)
```

**Solution:** Two-tier duplicate detection using Bloom Filter + Redis

#### Tier 1: Bloom Filter (In-Memory, Fast Check)

- **Purpose**: Fast preliminary check (<1μs lookup)
- **Size**: 12 GB RAM for 10B URLs with 1% false positive rate
- **How it works**: Hash URL with 3 hash functions, check if all bits are set

```
function isDuplicateBloom(url):
  hash1 = murmur3(url) % BLOOM_SIZE
  hash2 = fnv1a(url) % BLOOM_SIZE
  hash3 = xxhash(url) % BLOOM_SIZE
  
  if bloom_filter[hash1] AND bloom_filter[hash2] AND bloom_filter[hash3]:
    return PROBABLY_SEEN  // 1% false positive
  else:
    return DEFINITELY_NEW
```

#### Tier 2: Redis Set (Persistent, Confirmation)

- **Purpose**: Confirm if URL is truly a duplicate (when Bloom Filter says "maybe")
- **Size**: Store only URL hash (8 bytes) instead of full URL (100 bytes)
- **Storage**: 10B URLs × 8 bytes = 80 GB disk (Redis AOF persistence)

**Workflow:**

```
1. Check Bloom Filter
   → If DEFINITELY_NEW: Add to Bloom + Redis, crawl URL
   
2. If PROBABLY_SEEN (1% of URLs):
   → Check Redis for confirmation
   → If in Redis: Skip (true duplicate)
   → If NOT in Redis: False positive, add to Redis, crawl URL
```

**Memory Savings:**
- Pure Redis: 1 TB RAM
- Bloom + Redis: 12 GB RAM + 80 GB disk (99% reduction!)

*See `pseudocode.md::DuplicateFilter.check()` for detailed implementation*

---

### 3.3 Politeness Controller (Distributed Rate Limiting)

**Politeness Rules:**
1. **Respect robots.txt**: Never crawl disallowed paths
2. **Rate Limiting**: Maximum 1 request per second per domain (default)
3. **Crawl-delay**: Honor the `Crawl-delay` directive in robots.txt
4. **User-Agent**: Identify crawler with proper User-Agent string

#### Implementation with Redis Distributed Locks

**Problem:** Multiple crawler workers might fetch from the same domain simultaneously, violating politeness.

**Solution:** Distributed lock per domain using Redis.

```
Politeness Algorithm:

function canFetchFromDomain(domain):
  // 1. Check robots.txt (cached in Redis with 24h TTL)
  robots = getRobotsTxt(domain)  // Redis cache
  if robots.disallows(url):
    return false
  
  // 2. Acquire distributed lock for domain
  lock_key = "crawl:lock:" + domain
  acquired = redis.SET(lock_key, "1", NX=true, EX=1)  // 1-second lock
  
  if acquired:
    return true  // This worker can crawl
  else:
    return false  // Another worker is crawling, wait
```

**Redis Commands:**

```redis
# Try to acquire lock (atomic operation)
SET crawl:lock:example.com 1 NX EX 1

# If successful (returns OK), worker can crawl
# If failed (returns nil), another worker holds the lock
```

#### robots.txt Caching

```
Key: robots:example.com
Value: { allowed_paths: ["/"], disallowed_paths: ["/admin"], crawl_delay: 1 }
TTL: 24 hours
```

*See `pseudocode.md::PolitenessController.canFetch()` for implementation*

---

### 3.4 Fetcher Workers (Asynchronous HTTP Clients)

**Goal:** Fetch HTML from URLs with high concurrency and handle various failure modes.

#### Asynchronous I/O (100-200× Faster)

**Synchronous Fetching (Slow):**
```
for url in urls:
  response = http.get(url)  // Blocks 100ms+ per request
  process(response)

Throughput: 1 worker = ~10 URLs/sec (sequential)
```

**Asynchronous Fetching (Fast):**
```
async function fetchWorker():
  tasks = []
  for url in batch:
    tasks.append(http.get_async(url))  // Non-blocking
  
  responses = await asyncio.gather(tasks)  // Parallel fetch
  for response in responses:
    process(response)

Throughput: 1 worker = ~100-200 URLs/sec (parallel)
```

#### Failure Handling

| Failure Type             | Handling Strategy                                     |
|--------------------------|-------------------------------------------------------|
| **DNS Error**            | Skip URL, log error, don't retry                      |
| **Connection Timeout**   | Retry up to 3 times with exponential backoff          |
| **HTTP 4xx (Client Error)** | Don't retry (bad URL), mark as failed             |
| **HTTP 5xx (Server Error)** | Retry 3 times, then move to Dead Letter Queue     |
| **Malformed HTML**       | Log warning, attempt best-effort parsing              |
| **Redirect Loop**        | Follow max 5 redirects, then give up                  |

*See `pseudocode.md::FetcherWorker.fetch()` for implementation with retry logic*

---

### 3.5 Parser Workers (Extract Text and Links)

**Responsibilities:**
1. Extract clean text (remove scripts, styles, ads)
2. Discover new links (absolute URLs)
3. Extract metadata (title, description, keywords)
4. Normalize URLs (remove fragments, lowercase domain)

#### HTML Parsing Algorithm

```
function parseHTML(html, base_url):
  soup = BeautifulSoup(html)
  
  // 1. Extract text
  text = soup.get_text()
  text = clean_text(text)  // Remove extra whitespace, ads
  
  // 2. Extract links
  links = []
  for anchor in soup.find_all('a'):
    href = anchor.get('href')
    if href:
      absolute_url = urljoin(base_url, href)
      normalized_url = normalize(absolute_url)
      links.append(normalized_url)
  
  // 3. Extract metadata
  title = soup.find('title').text
  description = soup.find('meta', attrs={'name': 'description'})
  
  return { text, links, title, description }
```

#### URL Normalization

**Goal:** Treat these URLs as identical:
```
http://example.com/page
http://example.com/page?utm_source=email
http://example.com/page#section1
HTTP://EXAMPLE.COM/page
```

**Normalization Steps:**
1. Lowercase scheme and domain
2. Remove URL fragments (#section)
3. Remove tracking parameters (utm_*, fbclid)
4. Sort query parameters alphabetically
5. Remove default ports (80 for HTTP, 443 for HTTPS)
6. Remove trailing slash (if not homepage)

*See `pseudocode.md::ParserWorker.parse()` and `pseudocode.md::normalize_url()` for implementation*

---

## 4. Crawling Workflow (End-to-End)

> 📊 **See visual sequence diagram:** [Complete Crawl Flow](./sequence-diagrams.md#complete-crawl-flow-happy-path)

### Happy Path

```
1. Seed URL → URL Frontier (Kafka)

2. Fetcher Worker:
   - Dequeue URL from Kafka
   - Check Duplicate Filter (Bloom + Redis)
     → If duplicate: Drop, ack message
     → If new: Continue
   
3. Politeness Controller:
   - Check robots.txt (Redis cache)
   - Acquire distributed lock for domain (Redis)
     → If locked: Requeue URL with delay
     → If acquired: Continue
   
4. Fetch HTML:
   - HTTP GET with timeout (5 seconds)
   - Handle redirects (max 5)
     → If success: Continue
     → If error: Retry with exponential backoff (3 attempts)
   
5. Store Raw HTML:
   - Upload to S3: s3://crawler-data/{domain}/{url_hash}.html
   
6. Parser Worker:
   - Extract text, links, metadata
   - Store parsed text in Elasticsearch
   - Normalize discovered links
   - Enqueue new links to Kafka (URL Frontier)
   
7. Mark URL as Crawled:
   - Add URL hash to Redis
   - Update Bloom Filter
   - Log crawl status (success, timestamp, size)
```

### Failure Scenarios

| Scenario                 | Handling                                             |
|--------------------------|------------------------------------------------------|
| **DNS Failure**          | Log error, mark URL as failed, don't retry           |
| **Connection Timeout**   | Retry 3 times with 2s, 4s, 8s backoff               |
| **HTTP 503 (Overload)**  | Respect Retry-After header, increase crawl delay     |
| **robots.txt Disallows** | Drop URL, don't crawl                                |
| **Redirect Loop**        | Follow max 5 redirects, then fail                    |
| **Malformed HTML**       | Best-effort parse, log warning                       |
| **S3 Upload Failure**    | Retry 3 times, then move to Dead Letter Queue        |

*See [Sequence Diagrams](./sequence-diagrams.md) for detailed failure flow diagrams*

---

## 5. Bottlenecks and Future Scaling

### Bottleneck 1: Bloom Filter False Positives Accumulate

**Problem:** Bloom Filter false positive rate increases over time as more URLs are added.

```
Initial: 10M URLs → 1% false positive
After 1B URLs → 5% false positive (wastes 5% of Redis checks)
After 10B URLs → 10-15% false positive
```

**Solutions:**

1. **Periodic Rebuild**: Every month, rebuild Bloom Filter from Redis
2. **Sharded Bloom Filters**: Split by domain TLD (.com, .org, etc.)
3. **Counting Bloom Filter**: Use 4-bit counters to support removal

*See `pseudocode.md::BloomFilter.rebuild()` for implementation*

---

### Bottleneck 2: JavaScript-Heavy Sites

**Problem:** Many modern sites render content via JavaScript. Raw HTML has no content.

```
Example: Twitter, Facebook, Gmail
HTML: <div id="root"></div><script src="app.js"></script>
Content: Empty!
```

**Solutions:**

| Approach                | Pros                                  | Cons                                 | Use Case                    |
|-------------------------|---------------------------------------|--------------------------------------|-----------------------------|
| **Headless Browser**    | ✅ Renders JS, accurate content        | ❌ 10-20× slower, memory-intensive   | High-value sites            |
| **Pre-rendering Service**| ✅ Offload rendering to service        | ❌ Cost, complexity                   | Medium-value sites          |
| **API Access**          | ✅ Fast, structured data               | ❌ Requires API key, limited sites   | Sites with public APIs      |
| **Skip JS Sites**       | ✅ Simple, fast                        | ❌ Miss 30-40% of modern web         | Basic crawlers              |

**Recommended Hybrid Approach:**
- Detect JS-heavy sites (check for `<noscript>` tag, empty body)
- Route to dedicated headless browser pool (Puppeteer, Playwright)
- Limit headless crawling to 10% of total throughput

*See `pseudocode.md::JavaScriptRenderer.render()` for implementation*

---

### Bottleneck 3: URL Frontier Overflow

**Problem:** If discovery rate exceeds crawl rate, URL Frontier grows unbounded.

```
Discovery Rate: 1,000 URLs/sec
Crawl Rate: 800 URLs/sec
Queue Growth: +200 URLs/sec → 17M URLs/day
```

**Solutions:**

1. **Priority Topics**: Separate Kafka topics for high/medium/low priority
2. **Backpressure**: Pause low-priority crawling when queue exceeds threshold
3. **Aggressive Filtering**: Skip certain URL patterns (pagination, filters, calendars)
4. **Crawl Budget**: Limit URLs per domain (e.g., max 10,000 pages per site)

*See `pseudocode.md::URLFrontier.applyBackpressure()` for implementation*

---

### Bottleneck 4: Politeness Latency

**Problem:** Acquiring distributed lock for every URL adds 5-10ms overhead.

```
Latency: 1,000 URLs/sec × 5ms = 5 seconds of lock overhead per second!
```

**Solutions:**

1. **Local Caching**: Cache robots.txt and rate limit status locally (1-minute TTL)
2. **Batch Locking**: Acquire lock for batch of URLs from same domain
3. **Probabilistic Politeness**: Only enforce strict politeness for 90% of requests

*See `pseudocode.md::PolitenessController.localCache()` for implementation*

---

## 6. Common Anti-Patterns

> 📚 **Note:** For detailed pseudocode implementations of anti-patterns and their solutions, see **[3.2.3-design-web-crawler.md](./3.2.3-design-web-crawler.md#6-common-anti-patterns)** and **[pseudocode.md](./pseudocode.md)**.

### Anti-Pattern 1: Not Normalizing URLs

**Problem:**

❌ Treating these as different URLs:
```
http://example.com/page
http://example.com/page?utm_source=email
HTTP://EXAMPLE.COM/page
```

Result: Waste resources crawling duplicates, exhaust storage, violate politeness.

**Solution:**

✅ Normalize URLs before duplicate checking:
```
function normalize(url):
  parsed = urlparse(url)
  scheme = parsed.scheme.lower()  // http
  domain = parsed.netloc.lower()  // example.com
  path = parsed.path.rstrip('/')  // /page
  query = remove_tracking_params(parsed.query)  // Remove utm_*
  return scheme + "://" + domain + path + query

normalized = normalize("HTTP://EXAMPLE.COM/page?utm_source=email")
// Result: "http://example.com/page"
```

*See `pseudocode.md::normalize_url()` for full implementation*

---

### Anti-Pattern 2: No Rate Limiting (Spider Trap)

**Problem:**

❌ Crawler discovers calendar site with infinite URLs:
```
example.com/calendar/2024/01/01
example.com/calendar/2024/01/02
...
example.com/calendar/9999/12/31  // 3.6 million pages!
```

Result: Waste months crawling one site, never finish.

**Solution:**

✅ Enforce crawl budget per domain:
```
MAX_PAGES_PER_DOMAIN = 10,000

function shouldCrawlFromDomain(domain):
  crawled_count = redis.GET("crawl:count:" + domain)
  if crawled_count >= MAX_PAGES_PER_DOMAIN:
    return false  // Skip, already crawled enough
  else:
    redis.INCR("crawl:count:" + domain)
    return true
```

**Additional Protections:**
- Detect URL patterns (calendar, pagination, filters)
- Limit crawl depth per site (max 5 levels deep)
- Skip URLs with too many query parameters (> 5)

*See `pseudocode.md::CrawlBudgetController.check()` for implementation*

---

### Anti-Pattern 3: Ignoring robots.txt

**Problem:**

❌ Crawler ignores robots.txt and crawls admin pages, APIs, private content.

```
# robots.txt
User-agent: *
Disallow: /admin/
Disallow: /api/
Crawl-delay: 2
```

Result: Site blocks crawler IP, legal issues, wasted bandwidth on uncrawlable content.

**Solution:**

✅ Always fetch and respect robots.txt before crawling:
```
function canCrawl(url):
  domain = extract_domain(url)
  
  // 1. Fetch robots.txt (cache for 24h)
  robots = getRobotsTxt(domain)
  
  // 2. Check if URL is allowed
  if robots.is_disallowed(url):
    return false
  
  // 3. Respect crawl-delay
  delay = robots.get_crawl_delay()
  sleep(delay)
  
  return true
```

**robots.txt Parsing:**
```
# Example robots.txt
User-agent: *
Disallow: /admin/
Disallow: /private/
Crawl-delay: 1

# Parsed structure
{
  "user_agent": "*",
  "disallowed_paths": ["/admin/", "/private/"],
  "crawl_delay": 1
}
```

*See `pseudocode.md::RobotsTxtParser.parse()` and `pseudocode.md::PolitenessController.respectRobots()` for implementation*

---

### Anti-Pattern 4: No Timeout on HTTP Requests

**Problem:**

❌ Some sites respond very slowly or never respond. Without timeout, worker hangs forever.

```
worker.fetch("http://slow-site.com/page")
// Waiting... waiting... waiting... (infinite)
// Worker is stuck, can't process other URLs
```

Result: Workers get stuck, throughput drops to zero.

**Solution:**

✅ Always set timeout on HTTP requests (5 seconds recommended):
```
function fetchWithTimeout(url):
  try:
    response = http.get(url, timeout=5)  // 5-second timeout
    return response
  except TimeoutException:
    log("Timeout fetching " + url)
    return null
  except ConnectionError:
    log("Connection error fetching " + url)
    return null
```

**Recommended Timeouts:**
- DNS resolution: 2 seconds
- Connection: 5 seconds
- Total request: 10 seconds (including redirects)

*See `pseudocode.md::FetcherWorker.fetchWithTimeout()` for implementation*

---

### Anti-Pattern 5: Storing Full URLs in Bloom Filter

**Problem:**

❌ Bloom Filter requires fixed-size input. Storing variable-length URLs is inefficient.

```
bloom.add("http://example.com/page")  // 26 bytes
bloom.add("http://example.com/very/long/path/to/page?query=value")  // 60 bytes
```

Result: Bloom Filter implementation becomes complex, inconsistent hashing.

**Solution:**

✅ Hash URLs to fixed-size before adding to Bloom Filter:
```
function addToBloomFilter(url):
  url_hash = sha256(url)  // Fixed 32-byte hash
  bloom.add(url_hash)
```

**Why Hash First:**
- Consistent size (32 bytes)
- Faster hashing (SHA256 is optimized)
- Better distribution across Bloom Filter bits

*See `pseudocode.md::DuplicateFilter.hashURL()` for implementation*

---

### Anti-Pattern 6: Single-Threaded Crawler

**Problem:**

❌ Sequential crawling is extremely slow:
```
for url in urls:
  html = fetch(url)  // 100ms network latency
  parse(html)        // 10ms CPU time

Throughput: 1000ms / 110ms = ~9 URLs/sec
```

Result: Would take 35 years to crawl 10 billion pages!

**Solution:**

✅ Use asynchronous I/O and worker pools:
```
async function crawlerWorker():
  while true:
    batch = frontier.dequeue(batch_size=100)
    
    // Fetch all URLs in parallel (non-blocking)
    responses = await asyncio.gather([
      http.get_async(url) for url in batch
    ])
    
    // Process responses
    for response in responses:
      parse_and_store(response)

# Run multiple workers
workers = [crawlerWorker() for i in range(10)]
await asyncio.gather(workers)
```

**Performance Comparison:**
- Sequential: ~10 URLs/sec
- Async (single worker): ~100-200 URLs/sec
- Async (10 workers): ~1,000-2,000 URLs/sec

*See `pseudocode.md::AsyncFetcherWorker.run()` for implementation*

---

## 7. Alternative Approaches (Not Chosen)

### Approach A: Centralized Queue (Redis) Instead of Kafka

**Architecture:**
- Use Redis LIST as URL frontier
- LPUSH to add URLs, RPOP to fetch URLs

**Pros:**
- ✅ Simpler setup than Kafka
- ✅ Lower latency (~1ms vs ~10ms)
- ✅ Built-in priority queues (sorted sets)

**Cons:**
- ❌ Lower throughput (~20k msg/sec vs Kafka's 1M msg/sec)
- ❌ No message replay (if worker crashes, URL is lost)
- ❌ Single point of failure (unless Redis Cluster)
- ❌ Memory-only (limited by RAM)

**Why Not Chosen:**
- Kafka's durability guarantees are critical for long-running crawls
- Need to scale to millions of URLs per second during bursts
- Kafka partitioning aligns with domain-based politeness

**When to Reconsider:**
- Small-scale crawling (< 10M pages)
- Lower throughput requirements (< 100 URLs/sec)
- Need very low latency (real-time crawling)

---

### Approach B: In-Memory Duplicate Detection (Hash Set)

**Architecture:**
- Store all crawled URLs in a giant hash set in memory
- Check membership: O(1) average time

**Pros:**
- ✅ Zero false positives (unlike Bloom Filter)
- ✅ Simpler implementation
- ✅ Faster lookups (~100ns vs ~1μs)

**Cons:**
- ❌ **Memory requirement: 1 TB RAM for 10B URLs**
- ❌ Very expensive (AWS r5.24xlarge: $6,000/month)
- ❌ Single point of failure (all data in RAM)
- ❌ No persistence (crash loses all data)

**Why Not Chosen:**
- Cost: 100× more expensive than Bloom Filter + Redis
- 1% false positives are acceptable (minor wasted crawls)
- Bloom Filter + Redis provides 99.9% accuracy at 1% of the cost

**When to Reconsider:**
- Small-scale crawling (< 100M URLs → ~10 GB RAM)
- Zero tolerance for duplicate crawls
- Budget allows for large memory instances

---

### Approach C: Synchronous Crawling (No Kafka)

**Architecture:**
- Direct flow: Fetch URL → Parse → Store → Repeat
- No message queue in between

**Pros:**
- ✅ Simpler architecture (fewer components)
- ✅ Lower operational complexity
- ✅ Immediate feedback (no queue lag)

**Cons:**
- ❌ **Cannot handle bursts** (sudden discovery of 1M URLs)
- ❌ Tight coupling (fetcher waits for parser, parser waits for storage)
- ❌ No failure isolation (one component failure stops everything)
- ❌ Harder to scale (can't independently scale fetchers vs parsers)

**Why Not Chosen:**
- Web crawling has highly variable load (bursts of link discovery)
- Need to decouple components for independent scaling
- Kafka provides backpressure and fault tolerance

**When to Reconsider:**
- Very small scale (< 1M pages)
- Simple, single-machine crawler
- Low availability requirements

---

## 8. Deep Dive: Handling Edge Cases

### Edge Case 1: Infinite Scroll / Dynamic Content Loading

**Problem:** Modern sites load content via JavaScript infinite scroll (Twitter, Facebook, Instagram).

**Detection:**
```javascript
// Page has empty body but loads content via JS
<body>
  <div id="root"></div>
  <script src="app.js"></script>
</body>
```

**Solution:**

**Option 1: Headless Browser Rendering**
```
1. Detect JS-heavy site (empty body + <script> tags)
2. Route to headless browser pool (Puppeteer/Playwright)
3. Render page, scroll to trigger lazy loading
4. Wait for network idle
5. Extract fully-rendered HTML
```

**Option 2: Intercept API Calls**
```
1. Use browser DevTools to find API endpoints
2. Directly call APIs to fetch data
3. Much faster than rendering (10× speedup)
```

**Trade-off:** Headless rendering is 10-20× slower. Only use for high-value sites.

*See `pseudocode.md::JavaScriptRenderer.render()` for implementation*

---

### Edge Case 2: Soft 404s (Page Exists but Empty)

**Problem:** Some sites return HTTP 200 for non-existent pages with "Page Not Found" content.

```
GET /nonexistent-page HTTP/1.1
HTTP/1.1 200 OK
<html><body>Sorry, page not found</body></html>
```

**Detection:**
```
function isSoft404(html, status_code):
  if status_code != 200:
    return false
  
  // Check for common "not found" phrases
  not_found_phrases = ["page not found", "404", "does not exist"]
  html_lower = html.lower()
  for phrase in not_found_phrases:
    if phrase in html_lower:
      return true
  
  // Check if page is mostly empty
  text = extract_text(html)
  if len(text) < 100:  // Less than 100 characters
    return true
  
  return false
```

**Solution:** Mark as 404, don't index, don't follow links.

*See `pseudocode.md::detectSoft404()` for implementation*

---

### Edge Case 3: CAPTCHA / Bot Detection

**Problem:** Sites use CAPTCHA or bot detection to block crawlers.

**Signs of Bot Detection:**
- Redirected to /captcha or /verify
- HTTP 403 Forbidden
- JavaScript challenge page (Cloudflare, Akamai)

**Solutions:**

| Strategy                | Pros                                | Cons                             | Use Case                     |
|-------------------------|-------------------------------------|----------------------------------|------------------------------|
| **Rotate User-Agents**  | ✅ Simple                            | ❌ Easily detected                | Basic sites                  |
| **Rotate IPs**          | ✅ Effective                         | ❌ Expensive (proxy cost)         | Medium protection sites      |
| **CAPTCHA Solving Service** | ✅ Automated                     | ❌ $1-3 per 1,000 CAPTCHAs        | High-value sites             |
| **Respect Blocks**      | ✅ Ethical, legal                    | ❌ Can't crawl some sites         | **Recommended default**      |

**Recommended Approach:**
- Always identify as a crawler (proper User-Agent)
- Respect robots.txt
- If blocked, don't circumvent
- For high-value sites, use official APIs instead

---

### Edge Case 4: URL Shorteners (bit.ly, tinyurl)

**Problem:** Crawling a shortened URL leads to the real URL. Need to avoid crawling the same destination twice.

```
Example:
bit.ly/abc123 → redirects to → example.com/article
tinyurl.com/xyz456 → redirects to → example.com/article

Should not crawl example.com/article twice!
```

**Solution:** Normalize to final destination URL before duplicate checking.

```
function normalizeURL(url):
  // Follow redirects to get final URL
  response = http.get(url, allow_redirects=true)
  final_url = response.url
  
  // Normalize final URL
  return normalize(final_url)
```

**Trade-off:** Adds latency (HTTP request for every URL shortener). Only apply to known shortener domains.

*See `pseudocode.md::URLNormalizer.resolveShorteners()` for implementation*

---

### Edge Case 5: Multilingual Content

**Problem:** Same page available in multiple languages.

```
example.com/article?lang=en
example.com/article?lang=es
example.com/article?lang=fr
```

**Solution 1: Treat as Different Pages** (Recommended for general crawlers)
- Index all language versions
- Users can find content in their language

**Solution 2: Canonical URL Detection**
- Check for `<link rel="canonical">` tag
- Only crawl canonical version

```html
<link rel="canonical" href="https://example.com/article?lang=en" />
```

*See `pseudocode.md::extractCanonical()` for implementation*

---

## 9. Monitoring and Observability

### Key Metrics to Track

| Metric                        | Type      | Threshold               | Alert Action                                  |
|-------------------------------|-----------|-------------------------|-----------------------------------------------|
| **Crawl Throughput**          | Gauge     | > 800 URLs/sec          | If < 500, check worker health                 |
| **URL Frontier Size**         | Gauge     | < 10M URLs              | If > 50M, apply backpressure                  |
| **Duplicate Check Latency**   | Histogram | < 5ms (p99)             | If > 20ms, check Redis/Bloom Filter           |
| **Fetch Success Rate**        | Gauge     | > 95%                   | If < 90%, investigate DNS/timeouts            |
| **Bloom Filter False Positive Rate** | Gauge | < 5%              | If > 10%, rebuild Bloom Filter                |
| **Redis Memory Usage**        | Gauge     | < 80% capacity          | If > 90%, scale Redis or shard                |
| **Kafka Consumer Lag**        | Gauge     | < 1,000 messages        | If > 10k, scale workers                       |
| **Storage Growth Rate**       | Gauge     | ~100 GB/day             | Monitor cost, archive old data                |

### Distributed Tracing Example

**Successful Crawl:**
```
Request: Crawl example.com/page
  ├─ URL Frontier Dequeue [5ms]
  ├─ Duplicate Check (Bloom Filter) [1ms]
  ├─ Duplicate Check (Redis) [skipped - Bloom Filter said new]
  ├─ Politeness Check [3ms]
  │   ├─ robots.txt Cache Lookup [1ms]
  │   └─ Acquire Lock [2ms]
  ├─ HTTP Fetch [87ms]
  ├─ Store HTML (S3) [45ms]
  ├─ Parse HTML [12ms]
  ├─ Store Text (Elasticsearch) [23ms]
  ├─ Enqueue 15 New URLs [8ms]
  └─ Total: 184ms ✅
```

**Failed Crawl (Timeout):**
```
Request: Crawl slow-site.com/page
  ├─ URL Frontier Dequeue [5ms]
  ├─ Duplicate Check [2ms]
  ├─ Politeness Check [3ms]
  ├─ HTTP Fetch [5000ms] ⚠️ TIMEOUT
  │   ├─ Retry 1 [5000ms] ⚠️ TIMEOUT
  │   ├─ Retry 2 [5000ms] ⚠️ TIMEOUT
  │   └─ Retry 3 [5000ms] ⚠️ TIMEOUT
  ├─ Move to Dead Letter Queue [10ms]
  └─ Total: 20,020ms ❌ FAILED
```

---

## 10. Interview Discussion Points

### Question 1: How would you detect and handle duplicate content (not just URLs)?

**Problem:** Different URLs pointing to identical content.

```
example.com/article?id=123
example.com/article?id=123&source=email
example.com/article?id=123&utm_campaign=summer
```

**Answer:**

**Solution: Content Fingerprinting with SimHash**

```
1. Extract main text content from HTML
2. Compute SimHash (64-bit fingerprint)
3. Store fingerprint in Redis
4. Compare new fingerprints with existing ones
5. If Hamming distance < 3: Consider duplicate

function detectDuplicateContent(html):
  text = extract_text(html)
  fingerprint = simhash(text)
  
  // Check Redis for similar fingerprints
  similar = redis.ZSCAN("fingerprints", fingerprint, radius=3)
  
  if similar:
    return DUPLICATE_CONTENT
  else:
    redis.ZADD("fingerprints", fingerprint)
    return UNIQUE_CONTENT
```

**Trade-off:** Adds 5-10ms per page. Only use for high-quality crawling.

*See `pseudocode.md::SimHashDuplicateDetector.check()` for implementation*

---

### Question 2: How do you prioritize which URLs to crawl first?

**Answer:**

**Multi-Factor Priority Score:**

```
priority_score = (
  pagerank_score × 0.4 +
  freshness_score × 0.3 +
  domain_authority × 0.2 +
  backlink_count × 0.1
)
```

**Factors:**

| Factor                | Weight | Source                          | Example                              |
|-----------------------|--------|---------------------------------|--------------------------------------|
| **PageRank Score**    | 40%    | Pre-computed (seed pages)       | google.com = 10, blog.example.com = 2 |
| **Freshness Score**   | 30%    | Last crawl timestamp            | Never crawled = 10, 1 year old = 5   |
| **Domain Authority**  | 20%    | Historical quality              | .edu/.gov = 10, unknown = 5          |
| **Backlink Count**    | 10%    | How many sites link to this     | 1,000 backlinks = 10, 0 = 1          |

**Implementation:**
```
function calculatePriority(url, metadata):
  pagerank = get_pagerank(url)  // From seed data
  age_days = (now() - metadata.last_crawled) / 86400
  freshness = min(10, age_days / 365 * 10)
  domain_authority = get_domain_authority(extract_domain(url))
  backlinks = count_backlinks(url)
  
  priority = (
    pagerank * 0.4 +
    freshness * 0.3 +
    domain_authority * 0.2 +
    min(10, backlinks / 100) * 0.1
  )
  
  return priority
```

**Kafka Topic Strategy:**
- High priority (score > 8): `url-frontier-high`
- Medium priority (score 4-8): `url-frontier-medium`
- Low priority (score < 4): `url-frontier-low`

Workers consume from high → medium → low in that order.

*See `pseudocode.md::PriorityCalculator.calculate()` for implementation*

---

### Question 3: How do you handle 100× scale (1 trillion pages)?

**Answer:**

**Current Scale (10B pages):**
- Bloom Filter: 12 GB RAM
- Redis: 80 GB disk
- Storage: 100 TB
- Workers: 10-20 crawlers

**100× Scale (1T pages):**

**Challenge 1: Duplicate Detection**
- Bloom Filter: 1.2 TB RAM (too large for single machine)
- **Solution:** Shard Bloom Filter by URL hash
  ```
  shard_id = hash(url) % 100
  bloom_filters = [BloomFilter(12GB) for _ in range(100)]
  bloom_filters[shard_id].check(url)
  ```

**Challenge 2: Storage**
- Storage: 10 PB (very expensive on S3)
- **Solution:**
  - Compress HTML (gzip: 5× reduction → 2 PB)
  - Archive cold data to Glacier (10× cheaper)
  - Only store full HTML for important pages
  - For others, store only extracted text

**Challenge 3: URL Frontier**
- Kafka topic size: 1 trillion URLs × 200 bytes = 200 TB
- **Solution:**
  - Use Kafka log compaction (remove old entries)
  - Implement crawl budget per domain (max 10k pages)
  - Aggressive filtering (skip pagination, calendars)

**Challenge 4: Crawl Time**
- Current: 115 days for 10B pages at 1,000 QPS
- 1T pages: 31 years! (unacceptable)
- **Solution:** Scale to 10,000 QPS
  - 100-200 crawler workers
  - Multi-region deployment (reduce latency)
  - 1T pages in ~3 years (acceptable for complete web crawl)

**Cost Estimate:**
- Storage (10 PB): $200k/month (S3 + Glacier)
- Compute (200 workers): $50k/month
- Network (10 GB/sec): $100k/month
- **Total: ~$350k/month**

---

### Question 4: How do you handle geo-distributed crawling?

**Answer:**

**Problem:** Crawling sites from distant regions is slow (high latency).

**Example:**
- US crawler → Asia site: 200ms RTT
- Asia crawler → Asia site: 20ms RTT (10× faster)

**Solution: Multi-Region Deployment**

```
┌─────────────────────────────────────────────────────┐
│              Global URL Frontier (Kafka)            │
│         (Partitioned by geographic region)          │
└───────┬──────────────────┬──────────────────────┬───┘
        │                  │                      │
        ▼                  ▼                      ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  US Region    │  │  EU Region    │  │ APAC Region   │
│  - Workers    │  │  - Workers    │  │  - Workers    │
│  - Bloom      │  │  - Bloom      │  │  - Bloom      │
│  - Redis      │  │  - Redis      │  │  - Redis      │
└───────────────┘  └───────────────┘  └───────────────┘
```

**Partitioning Strategy:**
```
function assignRegion(url):
  domain = extract_domain(url)
  tld = domain.split('.')[-1]
  
  region_map = {
    'us', 'com', 'org': 'US',
    'uk', 'eu', 'de', 'fr': 'EU',
    'cn', 'jp', 'in', 'kr': 'APAC'
  }
  
  return region_map.get(tld, 'US')  // Default to US
```

**Benefits:**
- 10× lower latency (20ms vs 200ms)
- Respect regional regulations (GDPR)
- Fault tolerance (region failure doesn't stop crawling)

**Trade-off:** Higher infrastructure cost, more complex coordination.

---

### Question 5: How do you ensure crawler is polite and ethical?

**Answer:**

**Ethical Crawling Principles:**

1. **Always Respect robots.txt**
   ```
   - Fetch robots.txt before crawling
   - Cache for 24 hours
   - Honor all directives (Disallow, Crawl-delay, Allow)
   ```

2. **Proper User-Agent Identification**
   ```
   User-Agent: MyCrawler/1.0 (+https://example.com/crawler-info)
   ```
   - Include crawler name, version, contact URL
   - Never spoof as a browser

3. **Rate Limiting Per Domain**
   ```
   - Default: 1 request per second per domain
   - Honor Crawl-delay directive (even if higher than 1s)
   - Reduce rate if receiving 503 errors
   ```

4. **Respect HTTP Status Codes**
   ```
   - 429 (Too Many Requests): Back off exponentially
   - 503 (Service Unavailable): Respect Retry-After header
   - 410 (Gone): Don't retry, mark as permanently deleted
   ```

5. **Provide Contact Information**
   ```
   - Host crawler info page: https://example.com/crawler-info
   - Include email for webmasters to request removal
   - Respond within 24 hours to removal requests
   ```

6. **Avoid Peak Hours** (Optional)
   ```
   - Crawl during off-peak hours (midnight-6am site's timezone)
   - Detect timezone from domain TLD
   ```

7. **Implement Opt-Out Mechanism**
   ```
   - Honor <meta name="robots" content="noindex">
   - Provide web form for domain exclusion
   - Maintain blacklist of domains that opted out
   ```

**Real-World Example: Googlebot**
```
User-Agent: Googlebot/2.1 (+http://www.google.com/bot.html)

- Respects robots.txt
- Default rate: 1 req/sec (can be adjusted in Search Console)
- Provides detailed documentation
- Fast response to webmaster complaints
```

---

## 11. Comparison with Real-World Systems

| Feature                  | Our Design                   | Googlebot           | Common Crawl      | Scrapy Framework   |
|--------------------------|------------------------------|---------------------|-------------------|--------------------|
| **Scale**                | 10B pages                    | Hundreds of billions| 3B pages/month    | < 1M pages typical|
| **Throughput**           | 1,000 URLs/sec               | Unknown (very high) | ~1,000 URLs/sec   | 10-100 URLs/sec    |
| **Duplicate Detection**  | Bloom Filter + Redis         | Proprietary         | URL fingerprinting| URL set            |
| **Politeness**           | Redis locks, robots.txt      | Advanced ML-based   | robots.txt        | AutoThrottle       |
| **Storage**              | S3 + Elasticsearch           | Proprietary         | S3 (open dataset) | Files/database     |
| **Priority**             | PageRank-based               | ML ranking model    | Breadth-first     | Depth-first        |
| **JavaScript Rendering** | Headless browser (selective) | Full rendering      | No rendering      | Splash (optional)  |
| **Cost**                 | ~$19k/month                  | Unknown (massive)   | Free (AWS Open Data)| Minimal          |

**Key Takeaways:**
- **Googlebot**: Most sophisticated, uses ML for everything, massive scale
- **Common Crawl**: Open dataset, similar architecture to ours, great for research
- **Scrapy**: Simple framework for small-scale crawling, no distributed features
- **Our Design**: Production-ready for medium-large scale (10B pages), cost-effective

---

## 12. Cost Analysis (AWS Example)

**Assumptions:**
- 10 billion pages
- 1,000 URLs/sec throughput
- 115 days crawl duration
- 100 TB storage

| Component                        | Specification                     | Monthly Cost        |
|----------------------------------|-----------------------------------|---------------------|
| **Crawler Workers (EC2)**        | 20× c5.2xlarge (8 vCPU, 16 GB)   | $2,720              |
| **Kafka Cluster**                | 3× kafka.m5.xlarge                | $1,092              |
| **Redis (Duplicate Filter)**     | cache.r5.2xlarge (52 GB RAM)      | $547                |
| **Elasticsearch**                | 5× i3.xlarge (4 vCPU, 30 GB RAM) | $3,990              |
| **S3 Storage (HTML)**            | 100 TB storage + requests         | $2,300              |
| **Data Transfer**                | 10 MB/sec egress                  | $7,884              |
| **CloudWatch Monitoring**        | Logs + Metrics + Dashboards       | $150                |
| **VPC, NAT Gateway**             | Multi-AZ setup                    | $90                 |
| **Total**                        |                                   | **~$18,773/month**  |

**Cost Optimization:**

| Optimization                  | Savings      | Notes                                    |
|-------------------------------|--------------|------------------------------------------|
| **Reserved Instances (1-year)**| -40% EC2     | Save $1,088/month on EC2                 |
| **S3 Glacier for Old Data**   | -70% storage | Archive after 30 days → Save $1,610/month|
| **Spot Instances for Workers**| -70% EC2     | Use Spot for 50% of workers → Save $950/month|
| **Compress HTML (gzip)**      | -70% storage | 5× compression → Save $1,400/month       |
| **Use S3 Intelligent Tiering**| -20% storage | Auto-archive → Save $460/month           |

**Optimized Cost: ~$13,265/month** (~30% savings)

**Cost per Page:**
- Full crawl: $18,773 × 4 months = $75,092
- Cost per page: $75,092 / 10B = **$0.0000075 per page** (~$7.50 per million pages)

---

## 13. Trade-offs Summary

| Decision                      | Choice                                | Alternative               | Why Chosen                                   | Trade-off                              |
|-------------------------------|---------------------------------------|---------------------------|----------------------------------------------|----------------------------------------|
| **Duplicate Detection**       | Bloom Filter + Redis                  | Pure Redis hash set       | 99% memory savings (12 GB vs 1 TB RAM)       | 1% false positive rate                 |
| **URL Frontier**              | Kafka (partitioned by domain)         | Redis LIST                | Fault tolerance, replay, higher throughput   | Higher latency (~10ms vs ~1ms)         |
| **Politeness**                | Distributed locks (Redis)             | Local state per worker    | Consistent enforcement across workers        | 5ms lock overhead per URL              |
| **Storage**                   | S3 (HTML) + Elasticsearch (text)      | Single database           | Cost-effective, searchable, scalable         | Eventual consistency                   |
| **Crawl Strategy**            | Asynchronous I/O                      | Synchronous (sequential)  | 100× faster (100 URLs/sec vs 10 URLs/sec)    | More complex code                      |
| **JavaScript Rendering**      | Selective (headless browser)          | Always render             | 10× faster (skip JS for static pages)        | Miss some dynamic content              |
| **Priority**                  | PageRank-based scoring                | Breadth-first (FIFO)      | Crawl important pages first                  | Complexity, pre-computation needed     |

---

## Summary

A distributed web crawler requires careful balance between throughput, politeness, and cost:

**Key Design Choices:**

1. ✅ **Two-Tier Duplicate Detection** (Bloom Filter + Redis) for 99% memory savings
2. ✅ **Kafka-Based URL Frontier** for fault tolerance and partitioning by domain
3. ✅ **Distributed Politeness** with Redis locks to respect robots.txt and rate limits
4. ✅ **Asynchronous I/O** for 100× throughput improvement
5. ✅ **Tiered Storage** (S3 for HTML, Elasticsearch for searchable text)
6. ✅ **PageRank-Based Prioritization** to crawl important pages first

**Performance Characteristics:**

- **Throughput:** 1,000 URLs/sec (86.4M pages/day)
- **Crawl Time:** 115 days for 10 billion pages
- **Storage:** 100 TB (10 KB per page average)
- **Memory:** 12 GB for Bloom Filter, 80 GB Redis for confirmation
- **Workers:** 10-20 asynchronous crawler nodes

**Critical Components:**

- **URL Frontier (Kafka):** Must handle bursts, partitioned by domain
- **Duplicate Filter:** 99% accuracy, 1% false positives acceptable
- **Politeness Controller:** Essential to avoid being blocked
- **Failure Handling:** Retry logic, timeouts, error classification

**Scalability Path:**

1. **0-10M pages:** Single-machine crawler, SQLite for URLs
2. **10M-1B pages:** Add Kafka, Redis, multiple workers
3. **1B-10B pages:** Shard Bloom Filter, scale Kafka/Redis
4. **10B-1T pages:** Multi-region deployment, sharded Bloom Filters (100× shards)

**Common Pitfalls to Avoid:**

1. ❌ Not normalizing URLs (waste resources on duplicates)
2. ❌ No rate limiting (spider traps, infinite calendars)
3. ❌ Ignoring robots.txt (get blocked, legal issues)
4. ❌ No timeout on HTTP requests (workers hang forever)
5. ❌ Storing full URLs in Bloom Filter (inefficient)
6. ❌ Single-threaded crawling (would take 35 years!)

**Recommended Stack:**

- **URL Frontier:** Apache Kafka (fault-tolerant, partitioned)
- **Duplicate Detection:** Bloom Filter (RAM) + Redis (disk confirmation)
- **Politeness:** Redis (distributed locks, robots.txt caching)
- **Fetcher:** Python (asyncio/aiohttp) or Go (goroutines)
- **Parser:** BeautifulSoup (Python) or goquery (Go)
- **Storage:** AWS S3 (raw HTML) + Elasticsearch (searchable text)
- **Monitoring:** Prometheus + Grafana + Distributed Tracing (Jaeger)

**Cost Efficiency:**

- Optimize with Spot Instances (save 70% on EC2)
- Use S3 Glacier for old data (save 70% on storage)
- Compress HTML with gzip (5× reduction)
- **Optimized cost:** ~$13k/month for 10B pages (~$7.50 per million pages)

This design provides a **production-ready, scalable blueprint** for building a web crawler that can index billions of web pages while being polite, cost-effective, and fault-tolerant! 🚀

---

For complete details, algorithms, and trade-off analysis, see the **[Full Design Document](./3.2.3-design-web-crawler.md)**.
