# Web Crawler - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A distributed web crawler system that fetches and indexes billions of web pages efficiently while respecting politeness
policies, avoiding duplicates, and handling failures gracefully. The system processes the entire web (10+ billion pages)
within a reasonable timeframe (3-4 months) while maintaining high throughput and fault tolerance.

**Key Characteristics:**

- **Massive scale** - 10 billion unique pages, 1,000 URLs/second throughput
- **Duplicate detection** - Efficiently identifying already-crawled URLs with minimal memory (Bloom Filter + Redis)
- **Politeness** - Respecting robots.txt and rate limiting per domain to avoid being blocked
- **Fault tolerance** - Handling DNS failures, timeouts, malformed HTML, and network issues
- **Prioritization** - Crawling important pages first (PageRank-style importance)
- **Distributed architecture** - Event-driven microservices with central URL frontier

---

## Requirements & Scale

### Functional Requirements

| Requirement                | Description                                            | Priority    |
|----------------------------|--------------------------------------------------------|-------------|
| **Fetch & Parse**          | Fetch HTML from URLs, extract text, discover new links | Must Have   |
| **Storage**                | Store raw HTML and parsed content in searchable format | Must Have   |
| **Politeness**             | Respect robots.txt, rate limit requests per domain     | Must Have   |
| **Duplicate Detection**    | Never process the same URL twice                       | Must Have   |
| **Prioritization**         | Crawl important/fresh pages first                      | Should Have |
| **Recrawl**                | Periodically recrawl pages to detect updates           | Should Have |
| **Content Type Filtering** | Only crawl HTML pages (skip PDFs, images, etc.)        | Should Have |

### Scale Estimation

| Metric                | Assumption             | Calculation             | Result                |
|-----------------------|------------------------|-------------------------|-----------------------|
| **Total Pages**       | Entire web             | 10 billion unique pages | 10B pages             |
| **Target Fetch Rate** | Efficient crawling     | 1,000 URLs/sec          | 1,000 QPS             |
| **Pages per Day**     | 1,000 × 86,400 sec/day | 1000 × 86400            | ~86.4M pages/day      |
| **Time to Crawl**     | 10B / 86.4M pages/day  | 10B / 86.4M             | ~115 days (4 months)  |
| **Average Page Size** | HTML + text            | 10 KB/page              | 10 KB                 |
| **Storage Required**  | 10B × 10 KB            | 10B × 10 KB             | ~100 TB storage       |
| **Bandwidth**         | 1,000 QPS × 10 KB      | 1000 × 10 KB            | ~10 MB/sec (~80 Mbps) |
| **Bloom Filter Size** | 10B URLs, 1% FP rate   | 10B × 10 bits           | ~12 GB RAM            |

**Back-of-the-Envelope:**

```
Throughput:
- 1,000 URLs/sec = 86.4M URLs/day = 31.5B URLs/year
- To crawl 10B pages: ~115 days (assuming 100% uptime)

Storage:
- Raw HTML: 10B × 10 KB = 100 TB
- Parsed text (compressed): ~30 TB
- Metadata (URL, timestamp, links): ~500 GB
Total: ~130 TB

Bloom Filter:
- 10B URLs, 1% false positive rate
- Bits required: 10B × 10 bits = 100 Gb = 12.5 GB RAM
- With 3 hash functions: ~12 GB total

Worker Nodes:
- Each worker: ~100 URLs/sec (avg fetch + parse time = 10ms)
- Total workers needed: 1,000 QPS / 100 = 10 crawler workers
- With redundancy: 15-20 workers
```

---

## Key Components

| Component                 | Responsibility                          | Technology Options                            |
|---------------------------|-----------------------------------------|-----------------------------------------------|
| **URL Frontier**          | Priority queue of URLs to be crawled    | Kafka (topics/partitions)                     |
| **Duplicate Filter**      | Check if URL already crawled            | Bloom Filter (in-memory) + Redis (persistent) |
| **Politeness Controller** | Enforce robots.txt, rate limiting       | Redis (locks, token bucket)                   |
| **Fetcher Workers**       | Download HTML content from URLs         | Go, Python (async HTTP clients)               |
| **Parser Service**        | Extract text, links, metadata from HTML | BeautifulSoup, lxml, goquery                  |
| **Storage (Raw HTML)**    | Store original HTML for auditing        | S3, Azure Blob (object storage)               |
| **Storage (Searchable)**  | Store parsed content for search         | Elasticsearch (full-text search)              |
| **Headless Browser Pool** | Render JavaScript-heavy sites           | Puppeteer, Playwright (optional)              |

---

## Architecture Flow

### Complete Crawl Flow

1. **Seed URL added to URL Frontier (Kafka)**
2. **Fetcher Worker consumes URL from Kafka**
3. **Check Duplicate Filter (Bloom Filter + Redis)**
    - If already crawled → Drop
    - If not crawled → Continue
4. **Check Politeness Controller**
    - Fetch robots.txt (if not cached)
    - Check if URL allowed
    - Check rate limit (Token Bucket)
    - Acquire distributed lock for domain
5. **Fetch HTML (HTTP GET request)**
    - Timeout: 5s
    - Retry: 3x with exponential backoff
6. **Parse HTML (Parser Service)**
    - Extract text content
    - Extract outgoing links
    - Extract metadata
7. **Store Results:**
    - Raw HTML → S3
    - Parsed content → Elasticsearch
    - New URLs → URL Frontier (Kafka)
8. **Mark URL as crawled:**
    - Add to Bloom Filter
    - Add to Redis Set
9. **Release distributed lock for domain**

---

## URL Frontier (Priority Queue)

**Purpose:** Maintain a queue of URLs to be crawled, prioritized by importance (e.g., PageRank score, freshness).

**Implementation:**

- **Technology:** Kafka with multiple topics/partitions
    - **High Priority Topic:** Seed URLs, important domains (news sites, Wikipedia)
    - **Medium Priority Topic:** Discovered links from high-priority pages
    - **Low Priority Topic:** Discovered links from low-priority pages
- **Partitioning Strategy:** Partition by domain hash to ensure all URLs from the same domain go to the same partition (
  for politeness enforcement)
- **Consumer Groups:** Multiple Fetcher workers consume from different partitions concurrently

**Why Kafka?**

- Provides **durable storage** (URLs aren't lost if workers crash)
- **Multiple consumers** can process different domains in parallel
- **Backpressure handling** (if workers are slow, Kafka buffers URLs)

---

## Duplicate Detection: Bloom Filter + Redis

**Purpose:** Quickly check if a URL has already been crawled to avoid redundant work.

### Two-Tier Approach

**Tier 1: Bloom Filter (Fast, In-Memory)**

- **Size:** 12 GB RAM for 10B URLs with 1% false positive rate
- **Hash Functions:** 3 independent hash functions (MurmurHash3)
- **False Positive Rate:** ~1% (means 1% of URLs might be incorrectly flagged as "already crawled")

**Tier 2: Redis (Persistent Confirmation)**

- **Purpose:** Confirm Bloom Filter positives (eliminate false positives)
- **Data Structure:** Redis Set with URL hashes (SHA-256)
- **Size:** ~500 GB for 10B URL hashes
- **Optimization:** Only check Redis if Bloom Filter returns "maybe crawled"

**Flow:**

1. URL arrives → Check Bloom Filter (O(1))
2. Bloom Filter says "definitely not crawled" → **Crawl it**
3. Bloom Filter says "maybe crawled" → Check Redis (O(1))
4. Redis confirms "not crawled" → **Crawl it** (false positive)
5. Redis confirms "already crawled" → **Drop it**

**Why Bloom Filter + Redis?**

- Bloom Filter **reduces Redis lookups by 99%** (only 1% false positives need Redis check)
- Redis provides **100% accuracy** for confirmation
- Cost-effective: **12 GB RAM (Bloom) + 500 GB Redis** vs **10B keys in Redis = 5 TB RAM**

---

## Politeness Controller

**Purpose:** Enforce politeness policies to avoid overloading domains and getting IP-banned.

### Policies

1. **robots.txt Compliance:**
    - Fetch robots.txt for each domain
    - Cache for 24 hours (TTL)
    - Respect `Disallow` directives, `Crawl-delay`

2. **Rate Limiting per Domain:**
    - Default: **1 request per second per domain**
    - Configurable per domain (e.g., large sites like Wikipedia: 10 req/sec)
    - Use **Token Bucket Algorithm**

3. **Distributed Locking:**
    - **Problem:** Multiple Fetcher workers might try to crawl the same domain simultaneously
    - **Solution:** Use Redis distributed lock keyed by domain name
    - **Lock Acquisition:** Before fetching from `example.com`, acquire lock `lock:example.com`
    - **Lock Release:** After fetch completes (or times out), release lock
    - **TTL:** 10 seconds (to prevent deadlocks if worker crashes)

**Optimization: Local Cache**

- Each Fetcher worker maintains an in-memory cache of:
    - Recently accessed robots.txt files (LRU cache, 1000 domains)
    - Rate limit status for recently crawled domains
- **Benefit:** Reduces Redis lookups by 80-90% for repeated domain access

---

## Fetcher Workers

**Purpose:** Download HTML content from URLs using HTTP requests.

**Design:**

- **Asynchronous I/O:** Use async HTTP clients (Python `aiohttp`, Go `http.Client` with goroutines)
- **Concurrency:** Each worker handles 100-200 concurrent requests
- **Timeout:** 5 seconds per request (to avoid slow servers blocking workers)
- **Retry Logic:** Retry up to 3 times with exponential backoff (1s, 2s, 4s)
- **User-Agent:** Identify as a bot (e.g., `MyBot/1.0 (+https://mybot.com)`)

**Error Handling:**

- **DNS Errors:** Log and mark URL as failed (don't retry)
- **Timeouts:** Retry up to 3 times, then mark as failed
- **HTTP 4xx/5xx:** Log error, don't retry for 4xx (client errors), retry for 5xx (server errors)
- **Malformed HTML:** Log warning, attempt to parse anyway (use lenient parser)

**Throughput Calculation:**

- Each worker: 100 concurrent requests
- Average fetch time: 500ms (including network + DNS + server response)
- Throughput per worker: 100 / 0.5s = **200 URLs/sec**
- Total workers needed for 1,000 QPS: 1,000 / 200 = **5 workers** (use 10 for redundancy)

---

## Parser Service

**Purpose:** Extract useful information from HTML: text content, links, metadata.

**Extraction Tasks:**

1. **Links:** Extract all `<a href="...">` tags, normalize URLs (relative → absolute)
2. **Text Content:** Extract visible text (strip HTML tags, JavaScript, CSS)
3. **Metadata:** Title, description, keywords, author, publish date
4. **Outgoing Links:** Filter out non-HTML links (PDFs, images, videos)

**Technology:**

- **Python:** BeautifulSoup, lxml (for HTML parsing)
- **Go:** goquery (jQuery-like HTML parser)
- **URL Normalization:** urlparse to convert relative URLs to absolute

**Challenges:**

- **JavaScript-heavy Sites:** Modern SPAs (React, Angular) require JavaScript execution
    - **Solution:** Use headless browser (Puppeteer, Playwright) for JavaScript-heavy sites
    - **Trade-off:** Much slower (10-20 seconds per page) → Use sparingly (only for important sites)
- **Malformed HTML:** Use lenient parsers (BeautifulSoup with `lxml` parser)
- **Language Detection:** Use NLP libraries (langdetect) to identify language

**Output:**

- **New URLs** → Push back to URL Frontier (Kafka)
- **Parsed Content** → Push to Storage (Elasticsearch)
- **Raw HTML** → Push to Object Storage (S3)

---

## Storage

### Two Types of Storage

**1. Raw HTML Storage (S3 / Azure Blob)**

- **Purpose:** Store original HTML for future re-parsing or auditing
- **Format:** Gzip-compressed HTML files
- **Partitioning:** By crawl date (e.g., `s3://bucket/2025-01-15/page123.html.gz`)
- **Cost:** ~100 TB × $0.023/GB/month = **$2,300/month**

**2. Searchable Text Storage (Elasticsearch)**

- **Purpose:** Enable fast full-text search over crawled content
- **Schema:**
  ```json
  {
    "url": "https://example.com/page1",
    "title": "Example Page",
    "content": "This is the page content...",
    "crawl_timestamp": "2025-01-15T10:30:00Z",
    "domain": "example.com",
    "outgoing_links": ["https://example.com/page2", ...],
    "language": "en"
  }
  ```
- **Indexing:** Full-text index on `title` and `content` fields
- **Sharding:** Shard by domain hash (distribute load across Elasticsearch nodes)
- **Cost:** ~30 TB (compressed text) × $0.10/GB/month = **$3,000/month**

---

## Bottlenecks & Solutions

### Bottleneck 1: Bloom Filter False Positives

**Problem:** As the Bloom Filter fills up (after billions of URLs), the false positive rate increases from 1% to 5-10%,
leading to wasted Redis lookups.

**Solutions:**

1. **Periodic Rebuild:**
    - Every 30 days, rebuild Bloom Filter from Redis (takes ~2 hours for 10B URLs)
    - Use two Bloom Filters: one active, one being rebuilt (swap atomically)

2. **Sharding:**
    - Shard Bloom Filter by URL prefix (e.g., 10 shards: a-z, 0-9)
    - Each shard: 1B URLs → smaller, more accurate filters

3. **Hybrid Approach:**
    - Use Bloom Filter for **first-pass filtering** (99% rejection rate)
    - Use Redis for **second-pass confirmation** (1% lookups)
    - Use **persistent database** (Cassandra) for **long-term storage** (archival, analytics)

### Bottleneck 2: Politeness Controller Latency

**Problem:** Acquiring a distributed lock for every single URL fetch adds 5-10ms latency per request.

**Solutions:**

1. **Local Domain Cache:**
    - Each Fetcher worker maintains an in-memory cache of:
        - robots.txt files (LRU cache, 1000 domains, 5-minute TTL)
        - Rate limit status (last request time per domain)
    - **Benefit:** Reduces Redis lookups by 80-90%

2. **Batch Lock Acquisition:**
    - Instead of locking per URL, lock per domain and fetch multiple URLs from the same domain in sequence
    - Example: Acquire `lock:example.com`, fetch 10 URLs from `example.com`, release lock
    - **Trade-off:** Reduces concurrency (only one worker per domain at a time)

3. **Domain-Specific Workers:**
    - Assign specific domains to specific workers (sticky assignment)
    - Worker "owns" the domain, no locking needed
    - **Trade-off:** Load imbalance if some domains have more URLs than others

### Bottleneck 3: JavaScript-Heavy Sites

**Problem:** Modern SPAs (Single Page Applications) require JavaScript execution to render content, which is 10-20x
slower than static HTML parsing.

**Solutions:**

1. **Headless Browser Pool:**
    - Run a small pool of headless browsers (Puppeteer, Playwright) for JavaScript rendering
    - Use only for important/high-priority sites (e.g., news sites, social media)
    - **Cost:** 100 headless browser instances × $0.10/hour = $10/hour

2. **Pre-Rendering Service:**
    - Use a third-party service (Prerender.io, Rendertron) to pre-render JavaScript sites
    - Cache rendered HTML for 24 hours
    - **Cost:** ~$0.001 per page × 10M JS-heavy pages/day = $10,000/day

3. **API Crawling:**
    - For sites with public APIs (Twitter, Reddit), use the API instead of scraping HTML
    - **Benefit:** Faster, more reliable, structured data
    - **Trade-off:** Requires API keys, rate limits

### Bottleneck 4: URL Frontier Congestion

**Problem:** If the URL Frontier (Kafka) gets too large (billions of URLs), consumers can't keep up, and the system
grinds to a halt.

**Solutions:**

1. **Priority-Based Partitioning:**
    - Create separate topics for different priority levels:
        - `frontier-high`: Seed URLs, important domains (Wikipedia, news sites)
        - `frontier-medium`: Discovered links from high-priority pages
        - `frontier-low`: Discovered links from low-priority pages
    - Allocate more workers to high-priority topic

2. **Batch Processing:**
    - Instead of processing URLs one-by-one, fetch them in batches (e.g., 100 URLs at a time)
    - **Benefit:** Reduces Kafka overhead (fewer commits, fewer network round-trips)

3. **Backpressure:**
    - If URL Frontier size > 10M URLs, **slow down** link extraction (don't add every discovered link)
    - Prioritize important links (high PageRank, fresh content)

---

## Common Anti-Patterns to Avoid

### 1. Ignoring robots.txt

❌ **Bad:** Fetching URLs without checking robots.txt

- Violates Robots Exclusion Protocol (RFC 9309)
- Sites will detect and ban your crawler's IP address
- May violate terms of service, leading to lawsuits

✅ **Good:** Check robots.txt before fetching

- Fetch robots.txt for each domain
- Cache for 24 hours
- Respect `Disallow` directives and `Crawl-delay`

### 2. No Rate Limiting (Overloading Domains)

❌ **Bad:** Fetching 1000 URLs from the same domain in 1 second

- Overloads the target server (DDoS-like behavior)
- Gets your crawler banned/blacklisted
- Violates `Crawl-delay` directive in robots.txt

✅ **Good:** Rate limit per domain (1 req/sec default)

- Use Token Bucket algorithm
- Acquire distributed lock per domain
- Respect domain-specific rate limits

### 3. Single-Threaded Crawler (Too Slow)

❌ **Bad:** Single-threaded, sequential crawling

- To crawl 10B URLs at 1.6 URLs/sec: **197 years**
- Wastes CPU time (90% spent waiting for I/O)

✅ **Good:** Asynchronous, multi-threaded crawling

- Each worker handles 100-200 concurrent requests
- 10 workers × 200 URLs/sec = 2,000 URLs/sec

### 4. No Duplicate Detection (Crawling Same URLs Repeatedly)

❌ **Bad:** No duplicate checking

- Wastes bandwidth fetching the same page multiple times
- Wastes storage storing duplicate content
- Reduces throughput (time spent on duplicates instead of new pages)

✅ **Good:** Check Bloom Filter before fetching

- Bloom Filter for fast check (O(1))
- Redis for confirmation (eliminate false positives)
- Mark URLs as crawled after successful fetch

---

## Monitoring & Observability

### Key Metrics

| Metric                               | Target         | Alert Threshold | Why Important          |
|--------------------------------------|----------------|-----------------|------------------------|
| **Crawl Rate**                       | 1,000 URLs/sec | < 500 URLs/sec  | Throughput target      |
| **Duplicate Rate**                   | < 5%           | > 20%           | Efficiency indicator   |
| **Fetch Success Rate**               | > 95%          | < 90%           | System health          |
| **Average Fetch Latency**            | < 500ms        | > 2 seconds     | Performance            |
| **Bloom Filter False Positive Rate** | < 2%           | > 5%            | Memory efficiency      |
| **URL Frontier Size**                | < 10M URLs     | > 50M URLs      | Backpressure indicator |
| **Storage Growth Rate**              | Baseline ±20%  | > 2x baseline   | Storage planning       |
| **Error Rate**                       | < 5%           | > 10%           | System health          |

---

## Trade-offs Summary

| Decision                 | Choice                          | Alternative      | Why Chosen                        | Trade-off              |
|--------------------------|---------------------------------|------------------|-----------------------------------|------------------------|
| **URL Frontier**         | Kafka                           | PostgreSQL Queue | Durable, scalable, handles bursts | Increased complexity   |
| **Duplicate Detection**  | Bloom Filter + Redis            | Redis-only       | Memory efficient (12GB vs 5TB)    | 1% false positive rate |
| **Politeness**           | Distributed Lock + Token Bucket | Simple delay     | Prevents race conditions          | Adds 5-10ms latency    |
| **Storage (Raw)**        | S3/Object Storage               | Database         | Cost-effective, scalable          | Slower retrieval       |
| **Storage (Searchable)** | Elasticsearch                   | PostgreSQL       | Full-text search optimized        | Higher cost            |
| **JavaScript Sites**     | Headless Browser Pool           | Skip JS sites    | Complete content extraction       | 10-20x slower          |
| **Architecture**         | Event-driven (Kafka)            | Synchronous      | Handles scale, fault tolerance    | Eventual consistency   |

---

## Key Takeaways

1. **Kafka for URL Frontier** enables durable, scalable queue management
2. **Bloom Filter + Redis** provides memory-efficient duplicate detection (12GB vs 5TB)
3. **Politeness Controller** prevents IP bans via robots.txt and rate limiting
4. **Distributed locking** ensures only one worker per domain at a time
5. **Asynchronous I/O** enables high concurrency (100-200 requests per worker)
6. **Priority-based partitioning** ensures important pages are crawled first
7. **Two-tier storage** (S3 for raw, Elasticsearch for search) optimizes cost and performance
8. **Headless browser pool** handles JavaScript-heavy sites (sparingly)
9. **Fault tolerance** via retries, timeouts, and error handling
10. **Monitoring** tracks crawl rate, duplicates, errors, and storage growth

---

## Recommended Stack

- **URL Frontier:** Apache Kafka (durable, scalable queue)
- **Duplicate Detection:** Bloom Filter (in-memory) + Redis (persistent)
- **Politeness:** Redis (distributed locks, token bucket rate limiting)
- **Fetcher Workers:** Go or Python (async HTTP clients)
- **Parser:** BeautifulSoup (Python) or goquery (Go)
- **Storage (Raw):** S3 or Azure Blob (object storage)
- **Storage (Searchable):** Elasticsearch (full-text search)
- **Headless Browser:** Puppeteer or Playwright (for JavaScript sites)
- **Monitoring:** Prometheus + Grafana (metrics, dashboards)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (crawling, politeness checks,
  duplicate detection)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (Bloom Filter, politeness controller, fetcher)

