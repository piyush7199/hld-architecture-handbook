# 3.2.3 Design a Distributed Web Crawler

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

Design a distributed web crawler system that can fetch and index billions of web pages efficiently while respecting
politeness policies, avoiding duplicates, and handling failures gracefully. The system must be scalable to process the
entire web (10+ billion pages) within a reasonable timeframe (3-4 months) while maintaining high throughput and fault
tolerance.

**Key Challenges:**

- **Scale:** Processing billions of unique URLs without overwhelming individual domains
- **Duplicate Detection:** Efficiently identifying already-crawled URLs with minimal memory
- **Politeness:** Respecting robots.txt and rate limiting per domain to avoid being blocked
- **Fault Tolerance:** Handling DNS failures, timeouts, malformed HTML, and network issues
- **Prioritization:** Crawling important pages first (PageRank-style importance)

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement                | Description                                            | Priority    |
|----------------------------|--------------------------------------------------------|-------------|
| **Fetch & Parse**          | Fetch HTML from URLs, extract text, discover new links | Must Have   |
| **Storage**                | Store raw HTML and parsed content in searchable format | Must Have   |
| **Politeness**             | Respect robots.txt, rate limit requests per domain     | Must Have   |
| **Duplicate Detection**    | Never process the same URL twice                       | Must Have   |
| **Prioritization**         | Crawl important/fresh pages first                      | Should Have |
| **Recrawl**                | Periodically recrawl pages to detect updates           | Should Have |
| **Content Type Filtering** | Only crawl HTML pages (skip PDFs, images, etc.)        | Should Have |

### Non-Functional Requirements (NFRs)

| Requirement             | Target                 | Rationale                             |
|-------------------------|------------------------|---------------------------------------|
| **High Throughput**     | 1,000+ URLs/sec        | Complete 10B pages in ~115 days       |
| **Fault Tolerance**     | 99.9% uptime           | Handle DNS errors, timeouts, bad HTML |
| **Scalability**         | Horizontal scaling     | Add more crawler workers as needed    |
| **Low Latency (Fetch)** | < 5 seconds per page   | Maximize throughput                   |
| **Storage Efficiency**  | < 100 TB for 10B pages | Cost-effective storage                |

---

### Scale Estimation

#### Scenario: General Web Crawler (Search Engine)

| Metric                | Assumption             | Calculation                                  | Result                    |
|-----------------------|------------------------|----------------------------------------------|---------------------------|
| **Total Pages**       | Entire web             | 10 billion unique pages                      | **10B pages**             |
| **Target Fetch Rate** | Efficient crawling     | 1,000 URLs/sec                               | **1,000 QPS**             |
| **Pages per Day**     | 1,000 × 86,400 sec/day | $1000 \times 86400$                          | **~86.4M pages/day**      |
| **Time to Crawl**     | 10B / 86.4M pages/day  | $\frac{10 \times 10^9}{86.4 \times 10^6}$    | **~115 days (4 months)**  |
| **Average Page Size** | HTML + text            | 10 KB/page                                   | **10 KB**                 |
| **Storage Required**  | 10B × 10 KB            | $10 \times 10^9 \times 10 \times 10^3$ bytes | **~100 TB storage**       |
| **Bandwidth**         | 1,000 QPS × 10 KB      | $1000 \times 10 \times 10^3$ bytes/sec       | **~10 MB/sec (~80 Mbps)** |
| **Bloom Filter Size** | 10B URLs, 1% FP rate   | [See 2.5.4 Bloom Filters]                    | **~12 GB RAM**            |

#### Back-of-the-Envelope Calculations

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

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

The system follows an **event-driven microservices architecture** with a central URL frontier (queue) and distributed
workers for fetching, parsing, and storing content.

### System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         Seed URLs                                 │
│                    (Initial URLs to crawl)                        │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │  URL Frontier  │ ← Kafka Topic (Partitioned by Domain)
                    │   (Priority    │   - High priority: Seed URLs, important domains
                    │     Queue)     │   - Low priority: Discovered links
                    └────────┬───────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
                ▼            ▼            ▼
        ┌───────────┐  ┌───────────┐  ┌───────────┐
        │  Fetcher  │  │  Fetcher  │  │  Fetcher  │
        │  Worker 1 │  │  Worker 2 │  │  Worker N │
        └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
              │              │              │
              │  ┌───────────┴──────────────┘
              │  │
              ▼  ▼
        ┌──────────────────┐
        │ Duplicate Filter │ ← Bloom Filter (12 GB RAM)
        │ (Bloom Filter +  │   + Redis (Persistent confirmation)
        │  Redis Cache)    │
        └────────┬─────────┘
                 │
        ┌────────┴─────────────────┐
        │  Already crawled?        │
        │  Yes → Drop              │
        │  No  → Continue          │
        └────────┬─────────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │  Politeness      │ ← Redis (Distributed locks per domain)
        │  Controller      │   - robots.txt cache (24h TTL)
        │  (Rate Limiter)  │   - Rate limit per domain (1 req/sec default)
        └────────┬─────────┘
                 │
        ┌────────┴───────────┐
        │  Allowed?          │
        │  No  → Delay/Drop  │
        │  Yes → Fetch       │
        └────────┬───────────┘
                 │
                 ▼
        ┌─────────────────┐
        │  HTTP Fetcher   │ ← User-Agent, Timeout (5s), Retry (3x)
        │  (Download HTML)│
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────────┐
        │   Parser Service     │ ← Extract text, links, metadata
        │   (Extract Links &   │   - Beautif

ulSoup / Cheerio
        │    Content)          │   - NLP for text extraction
        └────────┬─────────────┘
                 │
        ┌────────┴─────────────────────┐
        │                              │
        ▼                              ▼
┌────────────────┐           ┌─────────────────┐
│  New URLs      │           │  Parsed Content │
│  → URL Frontier│           │  → Storage      │
└────────────────┘           └────────┬────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                        ▼                           ▼
                ┌───────────────┐         ┌──────────────────┐
                │  Raw HTML     │         │  Searchable Text │
                │  (S3 / Blob)  │         │  (Elasticsearch) │
                └───────────────┘         └──────────────────┘
```

### Key Components

| Component                 | Responsibility                   | Technology                    | Scaling Strategy           |
|---------------------------|----------------------------------|-------------------------------|----------------------------|
| **URL Frontier**          | Priority queue for URLs to crawl | Kafka (partitioned by domain) | Add more partitions        |
| **Duplicate Filter**      | Check if URL already crawled     | Bloom Filter + Redis          | Shard by URL hash          |
| **Fetcher Workers**       | Download HTML from URLs          | Python (asyncio) / Go         | Horizontal (10-20 workers) |
| **Politeness Controller** | Enforce robots.txt, rate limits  | Redis (distributed locks)     | One lock per domain        |
| **Parser Service**        | Extract text, links, metadata    | Python (BeautifulSoup) / Go   | Horizontal (10-20 workers) |
| **Storage (Raw HTML)**    | Store original HTML              | S3 / Azure Blob               | Object storage             |
| **Storage (Parsed Text)** | Searchable index                 | Elasticsearch                 | Horizontal sharding        |
| **Coordinator**           | Monitors health, rebalances work | Custom service + etcd         | Master-slave failover      |

---

## 4. Detailed Component Design

### 4.1 URL Frontier (Priority Queue)

**Purpose:** Maintain a queue of URLs to be crawled, prioritized by importance (e.g., PageRank score, freshness).

**Implementation:**

- **Technology:** Kafka with multiple topics/partitions
    - **High Priority Topic:** Seed URLs, important domains (news sites, Wikipedia)
    - **Medium Priority Topic:** Discovered links from high-priority pages
    - **Low Priority Topic:** Discovered links from low-priority pages
- **Partitioning Strategy:** Partition by domain hash to ensure all URLs from the same domain go to the same partition (
  for politeness enforcement)
- **Consumer Groups:** Multiple Fetcher workers consume from different partitions concurrently

**Why This Design?**

- Kafka provides **durable storage** (URLs aren't lost if workers crash)
- **Multiple consumers** can process different domains in parallel
- **Backpressure handling** (if workers are slow, Kafka buffers URLs)

*See [pseudocode.md::URLFrontier class](./pseudocode.md) for implementation*

---

### 4.2 Duplicate Filter (Bloom Filter + Redis)

**Purpose:** Quickly check if a URL has already been crawled to avoid redundant work.

**Two-Tier Approach:**

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

*See [pseudocode.md::BloomFilter class](./pseudocode.md) and [pseudocode.md::is_duplicate()](./pseudocode.md) for
implementation*

---

### 4.3 Politeness Controller

**Purpose:** Enforce politeness policies to avoid overloading domains and getting IP-banned.

**Policies:**

1. **robots.txt Compliance:**
    - Fetch robots.txt for each domain
    - Cache for 24 hours (TTL)
    - Respect `Disallow` directives, `Crawl-delay`

2. **Rate Limiting per Domain:**
    - Default: **1 request per second per domain**
    - Configurable per domain (e.g., large sites like Wikipedia: 10 req/sec)
    - Use **Token Bucket Algorithm** (see 2.5.1 Rate Limiting Algorithms)

3. **Distributed Locking:**
    - **Problem:** Multiple Fetcher workers might try to crawl the same domain simultaneously
    - **Solution:** Use Redis distributed lock (see 2.5.3 Distributed Locking) keyed by domain name
    - **Lock Acquisition:** Before fetching from `example.com`, acquire lock `lock:example.com`
    - **Lock Release:** After fetch completes (or times out), release lock
    - **TTL:** 10 seconds (to prevent deadlocks if worker crashes)

**Implementation:**

```
Domain: example.com
└─> Distributed Lock: lock:example.com (TTL: 10s)
    └─> Token Bucket: tokens:example.com (1 token/sec)
        └─> robots.txt Cache: robots:example.com (TTL: 24h)
```

**Optimization: Local Cache**

- Each Fetcher worker maintains an in-memory cache of:
    - Recently accessed robots.txt files (LRU cache, 1000 domains)
    - Rate limit status for recently crawled domains
- **Benefit:** Reduces Redis lookups by 80-90% for repeated domain access

*See [pseudocode.md::PolitenessController class](./pseudocode.md) for implementation*

---

### 4.4 Fetcher Workers

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

*See [pseudocode.md::Fetcher class](./pseudocode.md) for implementation*

---

### 4.5 Parser Service

**Purpose:** Extract useful information from HTML: text content, links, metadata.

**Extraction Tasks:**

1. **Links:** Extract all `<a href="...">` tags, normalize URLs (relative → absolute)
2. **Text Content:** Extract visible text (strip HTML tags, JavaScript, CSS)
3. **Metadata:** Title, description, keywords, author, publish date
4. **Outgoing Links:** Filter out non-HTML links (PDFs, images, videos)

**Technology:**

- **Python:** BeautifulSoup, lxml (for HTML parsing)
- **Go:** goquery (jQuery-like HTML parser)
- **URL Normalization:** [urlparse](https://docs.python.org/3/library/urllib.parse.html) to convert relative URLs to
  absolute

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

*See [pseudocode.md::Parser class](./pseudocode.md) for implementation*

---

### 4.6 Storage

**Two Types of Storage:**

**1. Raw HTML Storage (S3 / Azure Blob)**

- **Purpose:** Store original HTML for future re-parsing or auditing
- **Format:** Gzip-compressed HTML files
- **Partitioning:** By crawl date (e.g., `s3://bucket/2025-01-15/page123.html.gz`)
- **Cost:** ~100 TB × $0.023/GB/month = **$2,300/month**

**2. Searchable Text Storage (Elasticsearch)**

- **Purpose:** Enable fast full-text search over crawled content
- **Schema:**
  ```
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

## 5. Data Flow

### Complete Crawl Flow

```
1. Seed URL added to URL Frontier (Kafka)
   ↓
2. Fetcher Worker consumes URL from Kafka
   ↓
3. Check Duplicate Filter (Bloom Filter + Redis)
   - If already crawled → Drop
   - If not crawled → Continue
   ↓
4. Check Politeness Controller
   - Fetch robots.txt (if not cached)
   - Check if URL allowed
   - Check rate limit (Token Bucket)
   - Acquire distributed lock for domain
   ↓
5. Fetch HTML (HTTP GET request)
   - Timeout: 5s
   - Retry: 3x with exponential backoff
   ↓
6. Parse HTML (Parser Service)
   - Extract text content
   - Extract outgoing links
   - Extract metadata
   ↓
7. Store Results:
   - Raw HTML → S3
   - Parsed content → Elasticsearch
   - New URLs → URL Frontier (Kafka)
   ↓
8. Mark URL as crawled:
   - Add to Bloom Filter
   - Add to Redis Set
   ↓
9. Release distributed lock for domain
```

---

## 6. Bottlenecks and Optimizations

### 6.1 Bottleneck: Bloom Filter False Positives

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

---

### 6.2 Bottleneck: Politeness Controller Latency

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

---

### 6.3 Bottleneck: JavaScript-Heavy Sites

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

---

### 6.4 Bottleneck: URL Frontier Congestion

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

## 7. Common Anti-Patterns

### Anti-Pattern 1: Ignoring robots.txt

**Problem:**

```
// ❌ Bad: Ignoring robots.txt
function fetch_url(url):
  html = http_get(url)  // Fetch without checking robots.txt
  return html
```

**Why It's Bad:**

- **Ethical:** Violates Robots Exclusion Protocol (RFC 9309)
- **Practical:** Sites will detect and ban your crawler's IP address
- **Legal:** May violate terms of service, leading to lawsuits

**✅ Good:**

```
// ✅ Good: Check robots.txt before fetching
function fetch_url(url):
  domain = extract_domain(url)
  robots_txt = fetch_robots_txt(domain)  // Cache for 24h
  
  if not is_allowed(robots_txt, url):
    return None  // Drop URL
  
  html = http_get(url)
  return html
```

*See [pseudocode.md::check_robots_txt()](./pseudocode.md) for implementation*

---

### Anti-Pattern 2: No Rate Limiting (Overloading Domains)

**Problem:**

```
// ❌ Bad: Fetching 1000 URLs from the same domain in 1 second
for url in urls_from_example_com:
  fetch_url(url)  // No delay between requests
```

**Why It's Bad:**

- Overloads the target server (DDoS-like behavior)
- Gets your crawler banned/blacklisted
- Violates `Crawl-delay` directive in robots.txt

**✅ Good:**

```
// ✅ Good: Rate limit per domain (1 req/sec default)
domain_last_request_time = {}

function fetch_url(url):
  domain = extract_domain(url)
  last_request = domain_last_request_time.get(domain, 0)
  
  time_since_last = current_time() - last_request
  if time_since_last < 1.0:  // 1 second delay
    sleep(1.0 - time_since_last)
  
  html = http_get(url)
  domain_last_request_time[domain] = current_time()
  return html
```

*See [pseudocode.md::rate_limit_check()](./pseudocode.md) for implementation*

---

### Anti-Pattern 3: Single-Threaded Crawler (Too Slow)

**Problem:**

```
// ❌ Bad: Single-threaded, sequential crawling
for url in url_frontier:
  html = fetch_url(url)  // Blocks for 500ms
  parse_html(html)       // Blocks for 100ms
  // Total: 600ms per URL = ~1.6 URLs/sec
```

**Why It's Bad:**

- To crawl 10B URLs at 1.6 URLs/sec: **197 years**
- Wastes CPU time (90% spent waiting for I/O)

**✅ Good:**

```
// ✅ Good: Asynchronous, multi-threaded crawling
async function crawl_worker():
  while True:
    url = await url_frontier.pop()
    html = await fetch_url_async(url)  // Non-blocking
    await parse_html_async(html)
    // Each worker handles 100-200 concurrent requests
    // 10 workers × 200 URLs/sec = 2,000 URLs/sec
```

*See [pseudocode.md::CrawlerWorker class](./pseudocode.md) for implementation*

---

### Anti-Pattern 4: No Duplicate Detection (Crawling Same URLs Repeatedly)

**Problem:**

```
// ❌ Bad: No duplicate checking
function process_url(url):
  fetch_and_parse(url)  // Fetches even if already crawled
```

**Why It's Bad:**

- Wastes bandwidth fetching the same page multiple times
- Wastes storage storing duplicate content
- Reduces throughput (time spent on duplicates instead of new pages)

**✅ Good:**

```
// ✅ Good: Check Bloom Filter before fetching
function process_url(url):
  if bloom_filter.contains(url):
    if redis.exists(url):  // Confirm with Redis
      return  // Already crawled, drop
  
  fetch_and_parse(url)
  bloom_filter.add(url)
  redis.add(url)
```

*See [pseudocode.md::is_duplicate()](./pseudocode.md) for implementation*

---

## 8. Alternative Approaches

### Approach A: Centralized Queue (Single Server)

**Architecture:**

- Single server maintains the URL frontier (PostgreSQL or MySQL)
- Fetcher workers poll the central database for URLs

**Pros:**

- Simple to implement
- Easy to prioritize URLs (SQL `ORDER BY priority DESC`)

**Cons:**

- **Single point of failure** (if queue server crashes, entire crawler stops)
- **Scalability bottleneck** (database can't handle 1,000 QPS of inserts/deletes)
- **Contention** (all workers competing for locks on the same table)

**When to Use:**

- Small-scale crawlers (< 1M pages)
- Proof-of-concept prototypes

---

### Approach B: Distributed Queue (Kafka) — **Our Choice**

**Architecture:**

- Kafka maintains the URL frontier (distributed, fault-tolerant)
- Multiple partitions allow parallel consumption

**Pros:**

- **Fault tolerant** (Kafka replicates data across brokers)
- **Scalable** (add more partitions to increase throughput)
- **Backpressure handling** (if workers slow down, Kafka buffers URLs)

**Cons:**

- **Operational complexity** (Kafka cluster requires maintenance)
- **Prioritization is harder** (need multiple topics for different priorities)

**When to Use:**

- **Large-scale crawlers** (10B+ pages) ← **Our use case**
- High-throughput systems (1,000+ QPS)

---

### Approach C: Peer-to-Peer Crawler (BitTorrent-style)

**Architecture:**

- No central queue
- Workers communicate peer-to-peer to share URLs

**Pros:**

- **No single point of failure**
- **Self-organizing** (workers discover each other automatically)

**Cons:**

- **Very complex** (distributed consensus, URL deduplication across peers)
- **Hard to prioritize** (no global view of which URLs are important)
- **Debugging nightmare** (no central logs)

**When to Use:**

- Experimental research projects
- Decentralized web crawling (e.g., IPFS indexing)

---

## 9. Monitoring and Observability

### Key Metrics to Monitor

| Metric                          | Target     | Alert Threshold | Why Important           |
|---------------------------------|------------|-----------------|-------------------------|
| **URLs Crawled/sec**            | 1,000 QPS  | < 500 QPS       | Throughput indicator    |
| **URL Frontier Size**           | < 10M URLs | > 50M URLs      | Backpressure indicator  |
| **Duplicate Rate**              | < 5%       | > 20%           | Bloom Filter health     |
| **Fetch Success Rate**          | > 95%      | < 80%           | Network/DNS health      |
| **Average Fetch Time**          | < 500ms    | > 2s            | Network latency         |
| **Parser Success Rate**         | > 99%      | < 90%           | HTML quality            |
| **Storage Write Lag**           | < 1s       | > 10s           | Storage bottleneck      |
| **Worker CPU Usage**            | 50-70%     | > 90%           | Over/under-provisioning |
| **Redis Hit Rate (robots.txt)** | > 90%      | < 70%           | Cache effectiveness     |

### Dashboards

**1. Throughput Dashboard:**

- URLs crawled per second (line chart)
- URLs parsed per second
- URLs stored per second
- Breakdown by priority level (high/medium/low)

**2. Health Dashboard:**

- Fetcher worker status (up/down)
- Parser worker status
- Kafka lag per partition
- Redis memory usage

**3. Error Dashboard:**

- HTTP error rate (4xx, 5xx)
- Timeout rate
- DNS error rate
- Duplicate detection false positive rate

### Alerting

```
Alert: Throughput Drop
Condition: URLs_crawled_per_sec < 500 for 5 minutes
Action: Scale up Fetcher workers, check network
Severity: High

Alert: URL Frontier Overflow
Condition: URL_frontier_size > 50M
Action: Reduce link extraction rate, increase workers
Severity: Medium

Alert: Bloom Filter False Positive Rate High
Condition: duplicate_rate > 20%
Action: Rebuild Bloom Filter, increase size
Severity: Low
```

---

## 10. Trade-offs Summary

### What We Gained

✅ **High Throughput:** 1,000+ URLs/sec (can crawl 10B pages in 4 months)  
✅ **Fault Tolerance:** Kafka + Redis replication ensures no data loss  
✅ **Politeness:** Respects robots.txt, rate limits per domain  
✅ **Cost-Effective:** Bloom Filter reduces memory cost by 99%  
✅ **Scalability:** Horizontal scaling of Fetcher and Parser workers

### What We Sacrificed

❌ **Complexity:** Distributed system with Kafka, Redis, Elasticsearch  
❌ **Bloom Filter False Positives:** ~1% of URLs may be crawled twice  
❌ **Eventual Consistency:** URL deduplication is not 100% real-time  
❌ **Operational Overhead:** Requires monitoring, maintenance of multiple services  
❌ **JavaScript-Heavy Sites:** Slow crawling (requires headless browsers)

---

## 11. Real-World Implementations

### Google Crawler (Googlebot)

**Architecture:**

- **URL Frontier:** Custom distributed queue (likely Bigtable-based)
- **Duplicate Detection:** Bloom Filters + custom deduplication
- **Fetcher:** Thousands of servers globally (distributed by region)
- **Politeness:** Respects robots.txt, adaptive rate limiting
- **JavaScript:** Renders JavaScript sites (uses headless Chrome)

**Key Innovations:**

- **PageRank Prioritization:** Crawls important pages first
- **Incremental Crawling:** Recrawls frequently-updated pages more often
- **Mobile-First Indexing:** Crawls mobile versions of sites preferentially

---

### Common Crawl (Open-Source Web Crawler)

**Architecture:**

- **URL Frontier:** AWS SQS (managed queue service)
- **Duplicate Detection:** Bloom Filters
- **Fetcher:** EC2 spot instances (cost-optimized)
- **Storage:** S3 (WARC format for archival)

**Scale:**

- Crawls **3+ billion pages per month**
- Stores **250+ TB per month** in S3
- Open dataset available for research

**Website:** [commoncrawl.org](https://commoncrawl.org/)

---

### Scrapy (Python Web Crawling Framework)

**Architecture:**

- **URL Frontier:** In-memory priority queue (not distributed)
- **Duplicate Detection:** In-memory set (not scalable to billions)
- **Fetcher:** Asynchronous (Twisted framework)
- **Politeness:** Built-in rate limiting, robots.txt support

**Use Case:**

- Small to medium-scale crawling (< 1M pages)
- Single-machine deployment
- Easy to extend with custom middleware

**Website:** [scrapy.org](https://scrapy.org/)

---

## 12. References

### Related System Design Components

- **[2.5.1 Rate Limiting Algorithms](../../02-components/2.5.1-rate-limiting-algorithms.md)** - Token Bucket, Leaky
  Bucket
- **[2.5.3 Distributed Locking](../../02-components/2.5.3-distributed-locking.md)** - Redis locks, etcd
- **[2.5.4 Bloom Filters](../../02-components/2.5.4-bloom-filters.md)** - Memory-efficient duplicate detection
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)** - Message queue for URL frontier
- **[2.2.1 Caching Deep Dive](../../02-components/2.2.1-caching-deep-dive.md)** - Redis caching strategies
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - Sharding, replication strategies

### Related Design Challenges

- **[3.1.2 Distributed Cache](../3.1.2-distributed-cache/)** - Redis caching strategies for duplicate detection
- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/)** - Kafka message queue patterns
- **[3.2.2 Notification Service](../3.2.2-notification-service/)** - Distributed worker patterns

### External Resources

- **Google's Original Paper:
  ** [The Anatomy of a Large-Scale Hypertextual Web Search Engine](http://infolab.stanford.edu/~backrub/google.html) -
  Google's original crawler design
- **Mercator Paper:
  ** [Mercator: A Scalable, Extensible Web Crawler](https://www.cis.upenn.edu/~sudipto/mypapers/mercator.pdf) - Research
  paper on web crawler design
- **RFC 9309:** [Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309.html) - robots.txt specification
- **Common Crawl:** [Common Crawl Architecture](https://commoncrawl.org/the-data/get-started/) - Open-source crawler
  dataset
- **Kafka Documentation:** [Apache Kafka](https://kafka.apache.org/documentation/) - Distributed message queue
- **Elasticsearch Documentation:
  ** [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html) - Full-text search
  engine

### Tools and Libraries

- **Scrapy** - Python web crawling framework
- **Puppeteer** - Headless Chrome for JavaScript rendering
- **BeautifulSoup** - HTML parsing library (Python)
- **Redis** - In-memory cache and distributed locks

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Chapters on distributed systems, message queues, and
  storage
- *Web Scraping with Python* by Ryan Mitchell - Practical web crawling techniques
- *Introduction to Information Retrieval* by Christopher Manning - Search engine architecture