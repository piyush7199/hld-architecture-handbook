# 02-components: System Components Deep Dive

This directory contains comprehensive deep dive documents covering all major system components used in distributed systems design.

## 📁 Organization

Components are organized into **6 logical categories** for easier navigation:

### 🌐 [2.0-communication](./2.0-communication/) — Communication Protocols & APIs (3 files)

Foundational communication concepts, API design styles, and real-time communication patterns.

- **[2.0.1 Foundational Communication Protocols](./2.0-communication/2.0.1-foundational-communication-protocols.md)** — TCP/UDP, HTTP/S, WebSockets, WebRTC
- **[2.0.2 API Communication Styles](./2.0-communication/2.0.2-api-communication-styles.md)** — REST, gRPC, GraphQL, SOAP
- **[2.0.3 Real-Time Communication](./2.0-communication/2.0.3-real-time-communication.md)** — Long Polling, SSE, WebSockets

### 🗄️ [2.1-databases](./2.1-databases/) — Database Systems (15 files)

Comprehensive coverage of SQL, NoSQL, and specialized databases with practical design challenges.

#### Core Concepts (6 files)
- **[2.1.1 RDBMS Deep Dive](./2.1-databases/2.1.1-rdbms-deep-dive.md)** — SQL, ACID, Transactions
- **[2.1.2 NoSQL Deep Dive](./2.1-databases/2.1.2-no-sql-deep-dive.md)** — BASE, Document/Key-Value/Column stores
- **[2.1.3 Specialized Databases](./2.1-databases/2.1.3-specialized-databases.md)** — Time-series, Graph, Geospatial
- **[2.1.4 Database Scaling](./2.1-databases/2.1.4-database-scaling.md)** — Replication, Sharding, Federation
- **[2.1.5 Indexing & Query Optimization](./2.1-databases/2.1.5-indexing-and-query-optimization.md)** — B-Trees, LSM-Trees
- **[2.1.6 Data Modeling for Scale](./2.1-databases/2.1.6-data-modeling-for-scale.md)** — CQRS, Denormalization

#### SQL Databases (2 files)
- **[2.1.7 PostgreSQL Deep Dive](./2.1-databases/2.1.7-postgresql-deep-dive.md)** — MVCC, JSONB, PostGIS, Extensions
- **[2.1.8 MySQL Deep Dive](./2.1-databases/2.1.8-mysql-deep-dive.md)** — InnoDB, Replication, ProxySQL

#### NoSQL Databases (4 files)
- **[2.1.9 Cassandra Deep Dive](./2.1-databases/2.1.9-cassandra-deep-dive.md)** — Wide-column, Tunable consistency
- **[2.1.10 MongoDB Deep Dive](./2.1-databases/2.1.10-mongodb-deep-dive.md)** — Document model, Sharding
- **[2.1.11 Redis Deep Dive](./2.1-databases/2.1.11-redis-deep-dive.md)** — In-memory, Data structures
- **[2.1.12 DynamoDB Deep Dive](./2.1-databases/2.1.12-dynamodb-deep-dive.md)** — Serverless, GSI/LSI

#### Specialized Databases (3 files)
- **[2.1.13 Elasticsearch Deep Dive](./2.1-databases/2.1.13-elasticsearch-deep-dive.md)** — Full-text search, Aggregations
- **[2.1.14 Neo4j Deep Dive](./2.1-databases/2.1.14-neo4j-deep-dive.md)** — Graph database, Cypher
- **[2.1.15 ClickHouse Deep Dive](./2.1-databases/2.1.15-clickhouse-deep-dive.md)** — Columnar, OLAP

### ⚡ [2.2-caching](./2.2-caching/) — Caching Systems (3 files)

Caching strategies, distributed caching, and consistency.

- **[2.2.1 Caching Deep Dive](./2.2-caching/2.2.1-caching-deep-dive.md)** — Cache-aside, Write-through, CDN
- **[2.2.2 Consistent Hashing](./2.2-caching/2.2.2-consistent-hashing.md)** — Ring algorithm, Virtual nodes
- **[2.2.3 Memcached Deep Dive](./2.2-caching/2.2.3-memcached-deep-dive.md)** — Slab allocation, LRU eviction

### 📨 [2.3-messaging-streaming](./2.3-messaging-streaming/) — Messaging & Stream Processing (8 files)

Message queues, event streaming, and big data processing frameworks.

- **[2.3.1 Asynchronous Communication](./2.3-messaging-streaming/2.3.1-asynchronous-communication.md)** — Queues vs Streams
- **[2.3.2 Kafka Deep Dive](./2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** — Distributed streaming
- **[2.3.3 Advanced Message Queues](./2.3-messaging-streaming/2.3.3-advanced-message-queues.md)** — RabbitMQ, SQS, SNS
- **[2.3.4 Distributed Transactions & Idempotency](./2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md)** — 2PC, Sagas
- **[2.3.5 Batch vs Stream Processing](./2.3-messaging-streaming/2.3.5-batch-vs-stream-processing.md)** — Lambda/Kappa
- **[2.3.6 Push vs Pull Data Flow](./2.3-messaging-streaming/2.3.6-push-vs-pull-data-flow.md)** — Design patterns
- **[2.3.7 Apache Spark Deep Dive](./2.3-messaging-streaming/2.3.7-apache-spark-deep-dive.md)** — Unified analytics engine
- **[2.3.8 Apache Flink Deep Dive](./2.3-messaging-streaming/2.3.8-apache-flink-deep-dive.md)** — True streaming, CEP

### 🔒 [2.4-security-observability](./2.4-security-observability/) — Security & Monitoring (2 files)

Security fundamentals and system observability.

- **[2.4.1 Security Fundamentals](./2.4-security-observability/2.4.1-security-fundamentals.md)** — Auth, TLS, XSS/CSRF
- **[2.4.2 Observability](./2.4-security-observability/2.4.2-observability.md)** — Logging, Metrics, Tracing

### 🧮 [2.5-algorithms](./2.5-algorithms/) — Distributed Algorithms (4 files)

Core algorithms for distributed systems.

- **[2.5.1 Rate Limiting Algorithms](./2.5-algorithms/2.5.1-rate-limiting-algorithms.md)** — Token bucket, Leaky bucket
- **[2.5.2 Consensus Algorithms](./2.5-algorithms/2.5.2-consensus-algorithms.md)** — Paxos, Raft
- **[2.5.3 Distributed Locking](./2.5-algorithms/2.5.3-distributed-locking.md)** — Redis locks, Fencing tokens
- **[2.5.4 Bloom Filters](./2.5-algorithms/2.5.4-bloom-filters.md)** — Probabilistic data structure

---

## 🎯 How to Use This Directory

### For Learning:
1. Start with **communication** basics (2.0.x)
2. Learn **database fundamentals** (2.1.1-2.1.6)
3. Dive into **specific databases** based on use case (2.1.7-2.1.15)
4. Understand **caching** patterns (2.2.x)
5. Master **asynchronous communication** (2.3.x)
6. Study **security & observability** (2.4.x)
7. Learn **distributed algorithms** (2.5.x)

### For Interview Prep:
- **Must-read:** 2.1.1 (RDBMS), 2.1.2 (NoSQL), 2.2.1 (Caching), 2.3.2 (Kafka), 2.5.1 (Rate Limiting)
- **Database comparisons:** Read "When to Use" sections in each DB deep dive
- **Design challenges:** Every deep dive has a practical design challenge at the end

### For System Design:
- Use comparison tables to choose the right technology
- Study real-world examples from companies (Netflix, Uber, Twitter)
- Understand trade-offs (what you gain vs. sacrifice)
- Review anti-patterns (❌ bad vs ✅ good approaches)

---

## ✏️ Design Challenges

**Every component deep dive includes a practical design challenge!**

Examples:
- **PostgreSQL:** Multi-feature e-commerce platform (search + geo + consistency)
- **Cassandra:** IoT time-series platform (100K writes/sec)
- **Redis:** Gaming leaderboard (10M players)
- **MongoDB:** Social media profile system (celebrity problem)
- **Elasticsearch:** Multi-tenant SaaS search
- **DynamoDB:** Serverless shopping cart (Black Friday traffic)
- **Spark:** Real-time fraud detection with ML
- **Flink:** Real-time surge pricing (<100ms latency)

---

## 🔗 Related Sections

- **[01-principles](../01-principles/)** — Core system design principles (CAP, consistency models, etc.)
- **[03-challenges](../03-challenges/)** — Full system design challenges with diagrams and pseudocode

---

## 📖 Document Format

Each component deep dive follows a consistent structure:

1. **Intuitive Explanation** — High-level overview with analogies
2. **In-Depth Analysis** — Technical details, architecture, algorithms
3. **When to Use** — ✅ Use cases and ❌ Anti-use cases
4. **Real-World Examples** — How companies use this technology
5. **Comparisons** — vs. alternative technologies (with tables)
6. **Common Anti-Patterns** — ❌ Bad practices and ✅ Good solutions
7. **Trade-offs Summary** — What you gain vs. sacrifice
8. **References** — Official docs and related chapters
9. **✏️ Design Challenge** — Practical scenario with detailed solution

---

## 🚀 Quick Navigation

### I want to learn about...

**Databases:**
- SQL → [PostgreSQL (2.1.7)](./2.1-databases/2.1.7-postgresql-deep-dive.md) or [MySQL (2.1.8)](./2.1-databases/2.1.8-mysql-deep-dive.md)
- Document store → [MongoDB (2.1.10)](./2.1-databases/2.1.10-mongodb-deep-dive.md)
- Wide-column → [Cassandra (2.1.9)](./2.1-databases/2.1.9-cassandra-deep-dive.md)
- Key-value → [Redis (2.1.11)](./2.1-databases/2.1.11-redis-deep-dive.md) or [DynamoDB (2.1.12)](./2.1-databases/2.1.12-dynamodb-deep-dive.md)
- Search → [Elasticsearch (2.1.13)](./2.1-databases/2.1.13-elasticsearch-deep-dive.md)
- Graph → [Neo4j (2.1.14)](./2.1-databases/2.1.14-neo4j-deep-dive.md)
- Analytics → [ClickHouse (2.1.15)](./2.1-databases/2.1.15-clickhouse-deep-dive.md)

**Caching:**
- Concepts → [Caching Deep Dive (2.2.1)](./2.2-caching/2.2.1-caching-deep-dive.md)
- Distributed caching → [Consistent Hashing (2.2.2)](./2.2-caching/2.2.2-consistent-hashing.md)
- Implementation → [Memcached (2.2.3)](./2.2-caching/2.2.3-memcached-deep-dive.md) or [Redis (2.1.11)](./2.1-databases/2.1.11-redis-deep-dive.md)

**Messaging:**
- Concepts → [Async Communication (2.3.1)](./2.3-messaging-streaming/2.3.1-asynchronous-communication.md)
- Event streaming → [Kafka (2.3.2)](./2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)
- Message queues → [RabbitMQ/SQS/SNS (2.3.3)](./2.3-messaging-streaming/2.3.3-advanced-message-queues.md)

**Stream Processing:**
- Batch + Stream → [Spark (2.3.7)](./2.3-messaging-streaming/2.3.7-apache-spark-deep-dive.md)
- Real-time streaming → [Flink (2.3.8)](./2.3-messaging-streaming/2.3.8-apache-flink-deep-dive.md)

**Algorithms:**
- Rate limiting → [Rate Limiting Algorithms (2.5.1)](./2.5-algorithms/2.5.1-rate-limiting-algorithms.md)
- Consensus → [Paxos/Raft (2.5.2)](./2.5-algorithms/2.5.2-consensus-algorithms.md)
- Distributed locks → [Distributed Locking (2.5.3)](./2.5-algorithms/2.5.3-distributed-locking.md)
