# System Design Reference Guide

This reference guide contains quick-lookup tables, numbers, and decision matrices to support your system design process.

---

## 📊 Latency Numbers Every Programmer Should Know

Based on Peter Norvig's famous list, updated for modern hardware:

| Operation                              | Latency           | Notes                       |
|----------------------------------------|-------------------|-----------------------------|
| L1 cache reference                     | 0.5 ns            |                             |
| Branch mispredict                      | 5 ns              |                             |
| L2 cache reference                     | 7 ns              | 14× L1 cache                |
| Mutex lock/unlock                      | 100 ns            |                             |
| Main memory reference                  | 100 ns            | 20× L2 cache, 200× L1 cache |
| Compress 1KB with Snappy               | 10 μs (10,000 ns) |                             |
| Send 2KB over 1 Gbps network           | 20 μs             |                             |
| Read 1 MB sequentially from memory     | 250 μs            |                             |
| Round trip within same datacenter      | 500 μs (0.5 ms)   |                             |
| Disk seek (HDD)                        | 10 ms             | 20× datacenter RTT          |
| Read 1 MB sequentially from SSD        | 1 ms              | 4× memory                   |
| Read 1 MB sequentially from disk (HDD) | 30 ms             | 30× SSD, 120× memory        |
| Send packet CA → Netherlands → CA      | 150 ms            | Half the globe              |

### Key Takeaways

- **Memory is ~100,000× faster than disk**
- **SSD is ~30× faster than HDD**
- **Network within datacenter is ~300× faster than cross-continent**
- **Sequential access is ~10× faster than random access**

---

## 💾 Powers of Two / Storage Units

| Power | Exact Value      | Approximate    | Bytes       | Common Name |
|-------|------------------|----------------|-------------|-------------|
| 10    | 1,024            | ~1 thousand    | 1,024 B     | 1 KB        |
| 16    | 65,536           | ~65 thousand   | 64 KB       | -           |
| 20    | 1,048,576        | ~1 million     | 1,048,576 B | 1 MB        |
| 30    | 1,073,741,824    | ~1 billion     | 1 GB        | 1 GB        |
| 32    | 4,294,967,296    | ~4 billion     | 4 GB        | -           |
| 40    | ~1.1 trillion    | ~1 trillion    | 1 TB        | 1 TB        |
| 50    | ~1.1 quadrillion | ~1 quadrillion | 1 PB        | 1 PB        |

### Quick Conversions

```
1 KB = 1,024 bytes ≈ 1,000 bytes
1 MB = 1,024 KB ≈ 1,000 KB ≈ 1 million bytes
1 GB = 1,024 MB ≈ 1 billion bytes
1 TB = 1,024 GB ≈ 1 trillion bytes
1 PB = 1,024 TB ≈ 1,000 TB
```

---

## 🔢 Common Data Sizes

| Data Type             | Typical Size  | Notes                    |
|-----------------------|---------------|--------------------------|
| Character (ASCII)     | 1 byte        |                          |
| Character (Unicode)   | 2-4 bytes     | UTF-8 variable length    |
| Integer (32-bit)      | 4 bytes       |                          |
| Long (64-bit)         | 8 bytes       |                          |
| UUID/GUID             | 16 bytes      | 128-bit                  |
| Timestamp (Unix)      | 8 bytes       | Milliseconds since epoch |
| IPv4 Address          | 4 bytes       |                          |
| IPv6 Address          | 16 bytes      |                          |
| MD5 Hash              | 16 bytes      | 128-bit                  |
| SHA-256 Hash          | 32 bytes      | 256-bit                  |
| Tweet (text only)     | ~280 bytes    | 280 chars + metadata     |
| Small JSON object     | ~1 KB         | User profile             |
| Email (text)          | 5-10 KB       | Without attachments      |
| Web page (HTML)       | 50-100 KB     | Average                  |
| High-res image (JPEG) | 200 KB - 2 MB | Compressed               |
| Music (MP3, 3 min)    | ~3 MB         | 128 kbps                 |
| Video (1080p, 1 min)  | ~50-100 MB    | H.264 codec              |
| Video (4K, 1 min)     | ~400 MB       | H.265 codec              |

---

## 🌐 Bandwidth & Throughput

| Connection Type      | Bandwidth          | Notes                 |
|----------------------|--------------------|-----------------------|
| Dial-up              | 56 Kbps            | Legacy                |
| DSL                  | 1-100 Mbps         |                       |
| Cable                | 10-500 Mbps        |                       |
| Fiber (Residential)  | 100 Mbps - 10 Gbps |                       |
| 4G LTE               | 5-50 Mbps          | Mobile                |
| 5G                   | 50 Mbps - 10 Gbps  | Mobile, varies widely |
| Ethernet (Fast)      | 100 Mbps           |                       |
| Ethernet (Gigabit)   | 1 Gbps             | Standard for servers  |
| 10 Gigabit Ethernet  | 10 Gbps            | Data center           |
| 100 Gigabit Ethernet | 100 Gbps           | Backbone              |

### Quick Bandwidth Calculations

```
1 Gbps = 125 MB/s (Megabytes per second)
1 Mbps = 125 KB/s
To convert: Divide bits by 8 to get bytes
```

---

## 🗄️ Database Comparison Matrix

| Feature               | RDBMS (SQL)              | Document Store     | Wide-Column           | Key-Value             | Time-Series            | Graph                   |
|-----------------------|--------------------------|--------------------|-----------------------|-----------------------|------------------------|-------------------------|
| **Data Model**        | Tables, Rows             | Documents (JSON)   | Column families       | Key-value pairs       | Time-stamped events    | Nodes, Edges            |
| **Schema**            | Rigid                    | Flexible           | Flexible              | None                  | Semi-structured        | Flexible                |
| **Transactions**      | ACID (strong)            | Varies             | Eventually consistent | Eventually consistent | Varies                 | ACID (some)             |
| **Scalability**       | Vertical (hard to scale) | Horizontal         | Horizontal            | Horizontal            | Horizontal             | Varies                  |
| **Query Complexity**  | Complex (JOINs)          | Medium (no JOINs)  | Simple (key-based)    | Simple (key-based)    | Time-range queries     | Graph traversals        |
| **Use Cases**         | Financial, User data     | CMS, Catalogs      | Analytics, IoT        | Session, Cache        | Metrics, Logs          | Social, Recommendations |
| **Examples**          | PostgreSQL, MySQL        | MongoDB, Couchbase | Cassandra, HBase      | Redis, DynamoDB       | InfluxDB, TimescaleDB  | Neo4j, Neptune          |
| **Read Performance**  | Good (with indexes)      | Very Good          | Excellent             | Excellent             | Excellent (time-range) | Good (relationships)    |
| **Write Performance** | Good                     | Very Good          | Excellent             | Excellent             | Excellent              | Good                    |
| **Consistency**       | Strong                   | Tunable            | Eventual              | Eventual              | Varies                 | Strong (some)           |

### When to Use Each Database Type

**RDBMS (PostgreSQL, MySQL):**

- ✅ Structured data with relationships
- ✅ ACID transactions critical
- ✅ Complex queries with JOINs
- ❌ Massive scale (>TB data)
- ❌ Frequent schema changes
- **Examples:** Banking, ERP, User accounts

**Document Store (MongoDB, Couchbase):**

- ✅ Semi-structured data
- ✅ Rapid development, flexible schema
- ✅ Read-heavy workloads
- ❌ Complex transactions across documents
- ❌ Heavy relational queries
- **Examples:** CMS, Product catalogs, User profiles

**Wide-Column (Cassandra, HBase):**

- ✅ Massive write throughput
- ✅ Time-series data
- ✅ Eventual consistency acceptable
- ❌ Complex queries
- ❌ Strong consistency required
- **Examples:** IoT sensors, Activity logs, Analytics events

**Key-Value (Redis, DynamoDB):**

- ✅ Simple get/set operations
- ✅ Caching, session storage
- ✅ Ultra-low latency required
- ❌ Complex queries
- ❌ Large data per key
- **Examples:** Session store, Rate limiting, Leaderboards

**Time-Series (InfluxDB, TimescaleDB):**

- ✅ Timestamped data
- ✅ High write volume
- ✅ Time-range queries
- ❌ Random updates
- ❌ Complex relationships
- **Examples:** Metrics, Monitoring, Stock prices

**Graph (Neo4j, Neptune):**

- ✅ Highly connected data
- ✅ Relationship traversals
- ✅ Recommendation engines
- ❌ Simple key-value lookups
- ❌ Massive scale (>TB)
- **Examples:** Social networks, Fraud detection, Knowledge graphs

---

## 🔄 Caching Strategies Comparison

| Strategy                       | Write Path                   | Read Path                                       | Consistency           | Complexity | Use Case                               |
|--------------------------------|------------------------------|-------------------------------------------------|-----------------------|------------|----------------------------------------|
| **Cache-Aside (Lazy Loading)** | App → DB only                | Check cache → if miss, read DB → populate cache | Eventually consistent | Low        | Read-heavy, stale data OK              |
| **Write-Through**              | App → Cache → DB (sync)      | Check cache (always hit)                        | Strong                | Medium     | Small datasets, consistency critical   |
| **Write-Back (Write-Behind)**  | App → Cache (DB async)       | Check cache (always hit)                        | Eventually consistent | High       | High write volume, durability eventual |
| **Refresh-Ahead**              | Background job updates cache | Check cache (always hit)                        | Predictably stale     | Medium     | Predictable access patterns            |

### Cache Eviction Policies

| Policy                          | Strategy                        | Best For                  |
|---------------------------------|---------------------------------|---------------------------|
| **LRU (Least Recently Used)**   | Evict least recently accessed   | General purpose, balanced |
| **LFU (Least Frequently Used)** | Evict least frequently accessed | Hot data identification   |
| **FIFO (First-In-First-Out)**   | Evict oldest entry              | Simple, predictable       |
| **TTL (Time-To-Live)**          | Evict after expiration time     | Time-sensitive data       |
| **Random**                      | Evict random entry              | Simple, low overhead      |

---

## 🔀 Load Balancing Algorithms

| Algorithm                      | How It Works                                    | Pros                         | Cons                | Use Case                   |
|--------------------------------|-------------------------------------------------|------------------------------|---------------------|----------------------------|
| **Round Robin**                | Cycles through servers sequentially             | Simple, fair                 | Ignores server load | Homogeneous servers        |
| **Weighted Round Robin**       | Proportional to server capacity                 | Handles different capacities | Static weights      | Known capacity differences |
| **Least Connections**          | Routes to server with fewest active connections | Balances load dynamically    | Higher overhead     | Long-lived connections     |
| **Weighted Least Connections** | Combines capacity and connections               | Best load balancing          | Complex             | Variable server capacity   |
| **IP Hash**                    | Hash client IP to server                        | Session affinity             | Uneven distribution | Stateful sessions          |
| **Least Response Time**        | Routes to fastest server                        | Optimal performance          | Requires monitoring | Performance-sensitive      |

---

## 📡 API Communication Styles

| Style                        | Protocol        | Data Format       | Use Case                    | Pros                   | Cons                      |
|------------------------------|-----------------|-------------------|-----------------------------|------------------------|---------------------------|
| **REST**                     | HTTP/HTTPS      | JSON, XML         | Public APIs, CRUD           | Universal, simple      | Over/under-fetching       |
| **gRPC**                     | HTTP/2          | Protobuf (binary) | Microservices               | Fast, efficient        | Harder to debug           |
| **GraphQL**                  | HTTP            | JSON              | Mobile, complex clients     | Flexible queries       | Complex server-side       |
| **WebSocket**                | WebSocket       | Any               | Real-time, bidirectional    | Low latency, push      | Stateful, scaling complex |
| **SSE (Server-Sent Events)** | HTTP            | Text              | Real-time, server-to-client | Simple, auto-reconnect | One-way only              |
| **SOAP**                     | HTTP, SMTP, TCP | XML               | Enterprise, legacy          | Strict contracts       | Verbose, heavy            |

---

## 🎯 Consistency Models

| Model                                    | Guarantee                           | Read Latency | Use Case               | Example            |
|------------------------------------------|-------------------------------------|--------------|------------------------|--------------------|
| **Strong Consistency (Linearizability)** | Read always returns latest write    | Higher       | Financial transactions | Banking, Inventory |
| **Sequential Consistency**               | All processes see ops in same order | Higher       | Coordination           | Distributed locks  |
| **Causal Consistency**                   | Related ops seen in order           | Medium       | Social feeds           | Facebook timeline  |
| **Eventual Consistency**                 | Eventually all replicas converge    | Lower        | High availability      | DNS, Cassandra     |
| **Read-Your-Writes**                     | User always sees own updates        | Medium       | User profiles          | Profile updates    |
| **Monotonic Reads**                      | Never see older data after newer    | Medium       | Session data           | Shopping carts     |

---

## ⚡ Rate Limiting Algorithms

| Algorithm                  | How It Works                       | Allows Bursts? | Memory Usage | Best For          |
|----------------------------|------------------------------------|----------------|--------------|-------------------|
| **Fixed Window Counter**   | Reset counter each time window     | No             | Low          | Simple limits     |
| **Sliding Window Log**     | Track timestamp of each request    | Yes            | High         | Precise limits    |
| **Sliding Window Counter** | Weighted current + previous window | Partially      | Low          | Balanced accuracy |
| **Token Bucket**           | Tokens added at fixed rate         | Yes            | Low          | Bursty traffic    |
| **Leaky Bucket**           | Process requests at fixed rate     | No             | Medium       | Smooth traffic    |

---

## 🏗️ System Architecture Patterns

| Pattern                 | Description                        | Pros                    | Cons                             | Use Case                      |
|-------------------------|------------------------------------|-------------------------|----------------------------------|-------------------------------|
| **Monolithic**          | Single codebase, shared DB         | Simple deployment       | Hard to scale                    | Small apps, startups          |
| **Microservices**       | Independent services, separate DBs | Scalable, flexible      | Complex operations               | Large systems, multiple teams |
| **Serverless (FaaS)**   | Event-driven functions             | Auto-scale, pay-per-use | Cold starts, vendor lock-in      | Event processing, APIs        |
| **Event-Driven**        | Async communication via events     | Decoupled, resilient    | Eventually consistent            | E-commerce, IoT               |
| **CQRS**                | Separate read/write models         | Optimized for each      | Complexity, eventual consistency | High read/write ratio         |
| **Lambda Architecture** | Batch + Stream layers              | Comprehensive           | Duplicate logic                  | Analytics                     |
| **Kappa Architecture**  | Stream-only processing             | Simplified              | Requires powerful streaming      | Real-time analytics           |

---

## 🔒 Security Patterns

| Concern            | Pattern            | Implementation           | Use Case                  |
|--------------------|--------------------|--------------------------|---------------------------|
| **Authentication** | JWT                | Stateless tokens         | APIs, microservices       |
|                    | Session Cookies    | Server-side state        | Traditional web apps      |
|                    | OAuth 2.0          | Delegated auth           | Third-party login         |
| **Authorization**  | RBAC               | Role-based permissions   | Enterprise apps           |
|                    | ABAC               | Attribute-based rules    | Complex policies          |
| **Encryption**     | TLS/SSL            | In-transit encryption    | All network communication |
|                    | At-Rest Encryption | Database/disk encryption | Stored sensitive data     |
| **API Security**   | API Keys           | Simple identification    | Public APIs               |
|                    | Rate Limiting      | Throttle requests        | Prevent abuse             |

---

## 🧮 Quick Calculation Formulas

### QPS (Queries Per Second)

```
Average QPS = Total requests per day / 86,400
Peak QPS ≈ 2× to 3× Average QPS
```

### Storage

```
Total Storage = Daily operations × Data size per operation × Retention days × Replication factor
```

### Bandwidth

```
Ingress Bandwidth = Write QPS × Request size
Egress Bandwidth = Read QPS × Response size
```

### Cache Size (80/20 Rule)

```
Cache Size ≈ Total data × 0.20
```

### Number of Servers (Horizontal Scaling)

```
Servers needed = Peak QPS / (QPS per server × Target CPU utilization)
Target CPU utilization ≈ 0.7 (70%)
```

### Database Connections

```
Connection pool size = (Core count × 2) + Effective spindle count
For cloud databases: 100-500 connections typical
```

---

## 📏 Service Level Objectives (SLO)

| Metric            | Definition                      | Typical Target | Formula                 |
|-------------------|---------------------------------|----------------|-------------------------|
| **Availability**  | % of time system is operational | 99.9% - 99.99% | Uptime / Total time     |
| **Latency (P50)** | Median response time            | < 100ms        | 50th percentile         |
| **Latency (P95)** | 95% of requests faster than     | < 300ms        | 95th percentile         |
| **Latency (P99)** | 99% of requests faster than     | < 1s           | 99th percentile         |
| **Error Rate**    | % of failed requests            | < 0.1%         | Failed / Total requests |
| **Throughput**    | Requests per second             | Varies         | Total requests / time   |

### Availability Table (The Nines)

| Availability         | Downtime/Year | Downtime/Month | Downtime/Week | Use Case          |
|----------------------|---------------|----------------|---------------|-------------------|
| 90% (one nine)       | 36.5 days     | 3 days         | 16.8 hours    | Unacceptable      |
| 99% (two nines)      | 3.65 days     | 7.2 hours      | 1.68 hours    | Internal tools    |
| 99.9% (three nines)  | 8.76 hours    | 43.8 minutes   | 10.1 minutes  | Standard SLA      |
| 99.99% (four nines)  | 52.6 minutes  | 4.38 minutes   | 1.01 minutes  | High availability |
| 99.999% (five nines) | 5.26 minutes  | 26.3 seconds   | 6.05 seconds  | Mission-critical  |

---

## 🛠️ Technology Stack Examples

### Typical Web Application Stack

**Frontend:**

- React / Vue / Angular
- CDN (CloudFlare, CloudFront)

**API Layer:**

- Node.js / Go / Java Spring Boot
- API Gateway (Kong, AWS API Gateway)
- Load Balancer (NGINX, HAProxy)

**Business Logic:**

- Microservices or Monolith
- Docker containers
- Kubernetes orchestration

**Data Layer:**

- Primary DB: PostgreSQL (RDBMS) or MongoDB (Document)
- Cache: Redis or Memcached
- Search: Elasticsearch
- Message Queue: Kafka or RabbitMQ

**Storage:**

- Object Storage: S3 or GCS
- CDN for static assets

**Observability:**

- Logs: ELK Stack (Elasticsearch, Logstash, Kibana)
- Metrics: Prometheus + Grafana
- Tracing: Jaeger or Zipkin

---

## 🎓 System Design Interview Checklist

### 1. Requirements (5-10 min)

- [ ] Clarify functional requirements
- [ ] Clarify non-functional requirements (scale, latency, consistency)
- [ ] Define scope (what's in, what's out)

### 2. Estimation (5 min)

- [ ] Calculate QPS (read/write)
- [ ] Estimate storage needs
- [ ] Estimate bandwidth needs
- [ ] Identify bottlenecks

### 3. High-Level Design (10 min)

- [ ] Draw major components
- [ ] Define APIs
- [ ] Define data model/schema
- [ ] Choose database type

### 4. Deep Dive (15-20 min)

- [ ] Scale each component
- [ ] Address bottlenecks
- [ ] Discuss trade-offs
- [ ] Add caching strategy
- [ ] Add monitoring/alerting

### 5. Wrap Up (5 min)

- [ ] Identify potential failures
- [ ] Discuss future scalability
- [ ] Address interviewer questions

---

## 💡 Common Pitfalls to Avoid

| Pitfall                       | Why It's Wrong                     | Better Approach                      |
|-------------------------------|------------------------------------|--------------------------------------|
| **Single database**           | SPOF, doesn't scale                | Replication, sharding                |
| **No caching**                | Expensive DB queries               | Multi-layer caching                  |
| **Synchronous everything**    | Tight coupling, cascading failures | Async messaging, queues              |
| **Ignoring security**         | Data breaches, attacks             | HTTPS, authentication, rate limiting |
| **No monitoring**             | Can't detect/debug issues          | Logging, metrics, tracing            |
| **Premature optimization**    | Wasted time, complexity            | Start simple, measure, optimize      |
| **Ignoring data consistency** | Bugs, data corruption              | Choose consistency model carefully   |

---

This reference guide should serve as a quick lookup during system design discussions, interviews, and real-world
architecture decisions.

