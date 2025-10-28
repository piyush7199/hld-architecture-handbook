# 💡 HLD Architecture Handbook: The Intuitive System Designer

## Project Goal

This repository, the **HLD Architecture Handbook**, is designed to be a comprehensive, self-paced learning guide for
mastering **High-Level Design (HLD)** and large-scale system architecture. We focus on providing
**intuitive definitions** and **in-depth explanations** of core concepts, followed by structured design challenges. The
ultimate goal is to help you understand **the 'Why'** behind every architectural choice—the trade-offs, constraints, and
future-proofing considerations necessary for building systems at scale.

**Audience:** Engineers with basic programming knowledge looking to transition from small-scale development to designing
highly scalable, reliable, and performant distributed systems.

## 📁 Repository Structure

The content is organized into three progressive categories:

| Folder                                                                 | Category Name         | Focus                                                                                                                                       |
|------------------------------------------------------------------------|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| [01-principles](./01-principles)                                       | Core Principles       | Core theoretical concepts: Scale, Availability, CAP Theorem, and foundational architecture styles.                                          |
| [02-components](./02-components)                                       | Components Deep Dive  | In-depth analysis of specialized databases, caching, sharding, messaging, and concurrency control.                                          |
| [03-challenges](./03-challenges)                                       | Design Challenges     | Real-world design problems (e.g., URL Shortener, Twitter, E-commerce Flash Sale) applying the concepts learned in the first two categories. |
| README.md                                                              | (This File)           | The main project index and roadmap.                                                                                                         |
| [system-design-reference.md](./system-design-reference.md)             | Quick Reference Guide | Latency numbers, comparison tables, formulas, and decision matrices.                                                                        |
| [resources-and-further-reading.md](./resources-and-further-reading.md) | Learning Resources    | Books, papers, courses, blogs, and tools for continued learning.                                                                            |

### 📊 New: Comprehensive Design Challenges Structure

Each completed design challenge now includes **6 comprehensive files** for complete understanding:

```
03-challenges/
└── 3.x.y-problem-name/
    ├── 3.x.y-design-problem-name.md    # FULL comprehensive guide (main file)
    ├── README.md                        # Similar to main file with quick navigation
    ├── hld-diagram.md                   # System architecture diagrams (10-15 Mermaid diagrams)
    ├── sequence-diagrams.md             # Detailed interaction flows (10-15 Mermaid diagrams)
    ├── this-over-that.md                # In-depth design decisions & trade-offs analysis
    └── pseudocode.md                    # Detailed algorithm implementations (10-20 functions)
```

**Benefits:**

- 📊 **Visual Learning:** 20-30 interactive Mermaid diagrams per challenge for system architecture and sequence flows
- 📁 **Better Organization:** Separate theory, visuals, design decisions, and implementations
- 🔗 **Easy Navigation:** README links directly to all supplementary files
- 🎨 **Maintainable:** Text-based diagrams and pseudocode that are version-controlled
- 🚀 **GitHub Native:** Renders beautifully in GitHub without external tools
- 🧠 **Deep Understanding:** Detailed "This Over That" analysis explains WHY each architectural choice was made
- 💻 **Implementation Ready:** Comprehensive pseudocode with time complexity analysis

## 🗺️ Learning Roadmap: Core Concepts

We will cover the following topics in sequence before moving to the Design Challenges.

### Category 1: Core Principles (Folder: [01-principles](./01-principles))

| Topic ID | Concept                                                                                   |
|----------|-------------------------------------------------------------------------------------------|
| 1.1.1    | [CAP Theorem](01-principles/1.1.1-cap-theorem.md)                                         |
| 1.1.2    | [Latency, Throughput, and Scaling](01-principles/1.1.2-latency-throughput-scale.md)       |
| 1.1.3    | [Availability and Reliability](01-principles/1.1.3-availability-reliability.md)           |
| 1.1.4    | [Data Consistency Models](01-principles/1.1.4-data-consistency-models.md)                 |
| 1.1.5    | [Back-of-the-Envelope Calculations](01-principles/1.1.5-back-of-envelope-calculations.md) |
| 1.1.6    | [Failure Modes and Fault Tolerance](01-principles/1.1.6-failure-modes-fault-tolerance.md) |
| 1.2.1    | [System Architecture Styles](01-principles/1.2.1-system-architecture-styles.md)           |
| 1.2.2    | [Networking Components](01-principles/1.2.2-networking-components.md)                     |
| 1.2.3    | [API Gateway and Service Mesh](01-principles/1.2.3-api-gateway-servicemesh.md)            |
| 1.2.4    | [Domain-Driven Design (DDD) Basics](01-principles/1.2.4-domain-driven-design.md)          |

## Category 2: Components Deep Dive (Folder: [02-components](./02-components))

> **📁 Organized into 6 logical categories for easier navigation:**
> - 🌐 **Communication** (Protocols, APIs, Real-time)
> - 🗄️ **Databases** (15 database deep dives!)
> - ⚡ **Caching** (Redis, Memcached, Consistent Hashing)
> - 📨 **Messaging & Streaming** (Kafka, Spark, Flink, Message Queues)
> - 🔒 **Security & Observability** (Auth, Monitoring, Tracing)
> - 🧮 **Algorithms** (Rate Limiting, Consensus, Locking, Bloom Filters)

### 2.0 Communication (Folder: [2.0-communication](./02-components/2.0-communication))

| Topic ID | Concept                                                                                                               | Focus                                                                                                     |
|----------|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 2.0.1    | [Foundational Communication Protocols](02-components/2.0-communication/2.0.1-foundational-communication-protocols.md) | TCP vs. UDP, HTTP/S, WebSockets, WebRTC, DASH.                                                            |
| 2.0.2    | [API Communication Styles](02-components/2.0-communication/2.0.2-api-communication-styles.md)                         | REST, gRPC, SOAP, GraphQL (Pros, Cons, and Use Cases).                                                    |
| 2.0.3    | [Real-Time Communication](02-components/2.0-communication/2.0.3-real-time-communication.md)                           | Comparison of techniques for maintaining persistent or near-persistent connections for real-time updates. |

### 2.1 Databases (Folder: [2.1-databases](./02-components/2.1-databases)) — 15 Deep Dives

#### Core Database Concepts

| Topic ID | Concept                                                                                                 | Focus                                                                                 |
|----------|---------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| 2.1.1    | [RDBMS Deep Dive: SQL & ACID](02-components/2.1-databases/2.1.1-rdbms-deep-dive.md)                     | Transactions, Isolation Levels, ACID vs. BASE.                                        |
| 2.1.2    | [NoSQL Deep Dive: The BASE Principle](02-components/2.1-databases/2.1.2-no-sql-deep-dive.md)            | Document Stores, Key-Value Stores, Column-Family.                                     |
| 2.1.3    | [Specialized Databases](02-components/2.1-databases/2.1.3-specialized-databases.md)                     | Time-Series, Graph, Geospatial DBs (e.g., Redis Streams, Neo4j).                      |
| 2.1.4    | [Database Scaling](02-components/2.1-databases/2.1.4-database-scaling.md)                               | Replication (Master-Slave), Federation, Sharding Strategies.                          |
| 2.1.5    | [Indexing and Query Optimization](02-components/2.1-databases/2.1.5-indexing-and-query-optimization.md) | B-Trees, LSM-Trees, Denormalization Trade-offs.                                       |
| 2.1.6    | [Data Modeling for Scale (CQRS)](02-components/2.1-databases/2.1.6-data-modeling-for-scale.md)          | Denormalization, Data Decomposition, Command-Query Responsibility Segregation (CQRS). |

#### SQL Databases

| Topic ID | Concept                                                                        | Focus                                                                                            |
|----------|--------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| 2.1.7    | [PostgreSQL Deep Dive](02-components/2.1-databases/2.1.7-postgresql-deep-dive.md) | MVCC, JSONB, Full-Text Search, PostGIS, Advanced Indexing (GIN, BRIN), Replication, Extensions.  |
| 2.1.8    | [MySQL Deep Dive](02-components/2.1-databases/2.1.8-mysql-deep-dive.md)       | InnoDB Storage Engine, MVCC, Replication (Async, Semi-Sync, Group), Indexing (B+Tree), ProxySQL. |

#### NoSQL Databases

| Topic ID | Concept                                                                         | Focus                                                                                              |
|----------|---------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 2.1.9    | [Cassandra Deep Dive](02-components/2.1-databases/2.1.9-cassandra-deep-dive.md) | Masterless Architecture, Wide-Column Store, Tunable Consistency, Write Path, Compaction, Multi-DC. |
| 2.1.10   | [MongoDB Deep Dive](02-components/2.1-databases/2.1.10-mongodb-deep-dive.md)    | Document Model (BSON), Embedded vs. Referenced, Aggregation Framework, Sharding, Change Streams.   |
| 2.1.11   | [Redis Deep Dive](02-components/2.1-databases/2.1.11-redis-deep-dive.md)        | In-Memory Data Structures (Strings, Lists, Sets, Sorted Sets), Persistence (RDB/AOF), Cluster.     |
| 2.1.12   | [DynamoDB Deep Dive](02-components/2.1-databases/2.1.12-dynamodb-deep-dive.md)  | Serverless NoSQL, Partition/Sort Keys, GSI/LSI, On-Demand vs. Provisioned, Global Tables, Streams. |

#### Specialized Databases

| Topic ID | Concept                                                                                          | Focus                                                                                          |
|----------|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 2.1.13   | [Elasticsearch Deep Dive](02-components/2.1-databases/2.1.13-elasticsearch-deep-dive.md)         | Inverted Indexes, Full-Text Search, Aggregations, Integration with RDBMS (CDC), Sharding, ILM. |
| 2.1.14   | [Neo4j Deep Dive (Graph Databases)](02-components/2.1-databases/2.1.14-neo4j-deep-dive.md)       | Property Graph Model, Cypher Query Language, Index-Free Adjacency, Graph Algorithms.           |
| 2.1.15   | [ClickHouse Deep Dive (Columnar)](02-components/2.1-databases/2.1.15-clickhouse-deep-dive.md)    | Columnar Storage, MergeTree Engine, Vectorized Query Execution, OLAP Workloads.                |

### 2.2 Caching (Folder: [2.2-caching](./02-components/2.2-caching))

| Topic ID | Concept                                                                       | Focus                                                                      |
|----------|-------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| 2.2.1    | [Caching Deep Dive](02-components/2.2-caching/2.2.1-caching-deep-dive.md)     | Cache-Aside, Write-Through, CDN vs. App-Level Cache.                       |
| 2.2.2    | [Consistent Hashing](02-components/2.2-caching/2.2.2-consistent-hashing.md)   | Algorithm mechanics, Ring implementation, how it minimizes data movement.  |
| 2.2.3    | [Memcached Deep Dive](02-components/2.2-caching/2.2.3-memcached-deep-dive.md) | In-Memory Key-Value Cache, Slab Allocation, LRU Eviction, Multi-Threading. |

### 2.3 Messaging & Streaming (Folder: [2.3-messaging-streaming](./02-components/2.3-messaging-streaming))

| Topic ID | Concept                                                                                                                           | Focus                                                                                             |
|----------|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| 2.3.1    | [Asynchronous Communication](02-components/2.3-messaging-streaming/2.3.1-asynchronous-communication.md)                           | Queues vs. Streams, Pub/Sub Models, Backpressure.                                                 |
| 2.3.2    | [Kafka Deep Dive](02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)                                                 | Broker, Producer, Consumer Group, Partitions, Offset Management, Log Compaction.                  |
| 2.3.3    | [Advanced Message Queues (RabbitMQ, SQS, SNS)](02-components/2.3-messaging-streaming/2.3.3-advanced-message-queues.md)            | Comparison of broker-based vs. managed queues, Dead-Letter Queues (DLQs).                         |
| 2.3.4    | [Distributed Transactions & Idempotency](02-components/2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md) | Two-Phase Commit (2PC), Sagas, ensuring atomic operations.                                        |
| 2.3.5    | [Batch vs Stream Processing](02-components/2.3-messaging-streaming/2.3.5-batch-vs-stream-processing.md)                           | Detailed look at the Lambda and Kappa Architectures, latency vs. completeness trade-offs.         |
| 2.3.6    | [Push vs Pull Data Flow](02-components/2.3-messaging-streaming/2.3.6-push-vs-pull-data-flow.md)                                   | Architectural choices in messaging systems (e.g., Kafka (Pull) vs. RabbitMQ (Push)).              |
| 2.3.7    | [Apache Spark Deep Dive](02-components/2.3-messaging-streaming/2.3.7-apache-spark-deep-dive.md)                                   | Unified Analytics Engine, RDD/DataFrame API, In-Memory Computing, MLlib, Batch & Stream.          |
| 2.3.8    | [Apache Flink Deep Dive](02-components/2.3-messaging-streaming/2.3.8-apache-flink-deep-dive.md)                                   | True Stream Processing, Event-by-Event, Stateful Operators, Exactly-Once, CEP, Ultra-Low Latency. |

### 2.4 Security & Observability (Folder: [2.4-security-observability](./02-components/2.4-security-observability))

| Topic ID | Concept                                                                                          | Focus                                                                         |
|----------|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| 2.4.1    | [Security Fundamentals](02-components/2.4-security-observability/2.4.1-security-fundamentals.md) | Authn/Authz (JWT), TLS/Encryption, Cross-Site Scripting (XSS) & CSRF.         |
| 2.4.2    | [Observability](02-components/2.4-security-observability/2.4.2-observability.md)                 | Logging, Metrics (Prometheus), Distributed Tracing (Jaeger/Zipkin), Alerting. |

### 2.5 Distributed Algorithms (Folder: [2.5-algorithms](./02-components/2.5-algorithms))

| Topic ID | Concept                                                                                    | Focus                                                                              |
|----------|--------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| 2.5.1    | [Rate Limiting Algorithms](02-components/2.5-algorithms/2.5.1-rate-limiting-algorithms.md) | Token Bucket, Leaky Bucket, Fixed Window counter mechanisms.                       |
| 2.5.2    | [Consensus Algorithms](02-components/2.5-algorithms/2.5.2-consensus-algorithms.md)         | Paxos / Raft, Distributed Locks (ZooKeeper/etcd), solving the concurrency problem. |
| 2.5.3    | [Distributed Locking](02-components/2.5-algorithms/2.5.3-distributed-locking.md)           | $\text{Redis}$ locks, $\text{TTL}$, Fencing Tokens, ensuring mutual exclusion.     |
| 2.5.4    | [Bloom Filters](02-components/2.5-algorithms/2.5.4-bloom-filters.md)                       | Intuition, Hash Functions, False Positives, use cases (e.g., CDN cache lookups).   |

## 🗺️ Design Challenges Roadmap (Category 3)

### Easy Challenges (Focus: Core Components, Caching, Databases)

These problems require solid application of scaling fundamentals, hashing, and database choices.

> 📊 **Each challenge includes comprehensive visual diagrams (Mermaid) for system architecture and sequence flows!**

| Problem ID | System Name                                                                                                    | Key Concepts Applied                                                                                                 |
|------------|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| 3.1.1      | **[Design a URL Shortener](03-challenges/3.1.1-url-shortener/)** ($\text{TinyURL}$)                            | Hashing, Base62 Encoding, Read-Heavy Scaling, Sharding Key, $\text{Cache}$ $\text{Aside}$, Multi-region deployment   |
| 3.1.2      | **[Design a Distributed Cache](03-challenges/3.1.2-distributed-cache/)** ($\text{Redis}$ / $\text{Memcached}$) | Consistent Hashing, Eviction Policies ($\text{LRU}$), Replication, Failover, $\text{TTL}$, Cache Stampede Prevention |
| 3.1.3      | **[Design a Distributed ID Generator](03-challenges/3.1.3-distributed-id-generator/)** ($\text{Snowflake}$)    | 64-bit ID Structure, Worker ID Assignment, Clock Drift Handling, Sequence Management, $\text{etcd}$ Coordination     |

**📊 Each challenge folder contains 6 comprehensive files:**

- **[3.x.y-design-problem-name.md]** - Complete comprehensive guide with all content (main file)
- **[README.md]** - Quick navigation with links to all supplementary files
- **[hld-diagram.md]** - 10-15 system architecture diagrams with detailed flow explanations
- **[sequence-diagrams.md]** - 10-15 interaction flows with step-by-step explanations
- **[this-over-that.md]** - In-depth analysis of 5-10 major design decisions and trade-offs
- **[pseudocode.md]** - 10-20 detailed algorithm implementations with complexity analysis

### Medium Challenges (Focus: Asynchrony, Feeds, Microservices, Geo-Spatial)

These problems involve decoupling services, handling fan-out, and managing complex data models.

| Problem ID | System Name                                                                    | Key Concepts Applied                                                                                                                                                                                                               |
|------------|--------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.2.1      | [**Design a Twitter/X Timeline**](03-challenges/3.2.1-twitter-timeline/)       | $\text{Fanout}$ $\text{on}$ $\text{Write}$ vs. $\text{Fanout}$ $\text{on}$ $\text{Read}$, $\text{Caching}$ $\text{Hierarchy}$, $\text{Queuing}$ ($\text{Kafka}$).                                                                  |
| 3.2.2      | [**Design a Notification Service**](03-challenges/3.2.2-notification-service/) | $\text{Multi}$-$\text{Channel}$ ($\text{Email}$/$\text{SMS}$/$\text{Push}$/$\text{Web}$), $\text{WebSockets}$, $\text{Kafka}$ $\text{Streams}$, $\text{Circuit}$ $\text{Breakers}$, $\text{Rate}$ $\text{Limiting}$, $\text{DLQ}$. |
| 3.2.3      | [**Design a Distributed Web Crawler**](03-challenges/3.2.3-web-crawler/)       | $\text{URL}$ $\text{Frontier}$, $\text{Bloom}$ $\text{Filter}$, $\text{Duplicate}$ $\text{Detection}$, $\text{Politeness}$, $\text{Rate}$ $\text{Limiting}$, $\text{robots}$.$\text{txt}$.                                         |
| 3.2.4      | [**Design a Global Rate Limiter**](03-challenges/3.2.4-global-rate-limiter/)   | $\text{Token}$ $\text{Bucket}$, $\text{Sliding}$ $\text{Window}$, $\text{Atomic}$ $\text{INCR}$, $\text{Circuit}$ $\text{Breakers}$, $\text{Fail}$-$\text{Open}$, $\text{Hot}$ $\text{Key}$ $\text{Mitigation}$.                   |

### Hard Challenges (Focus: Consistency, Low-Latency, Transactions, Consensus, Real-Time Geo)

These problems require advanced pattern usage, strong consistency guarantees, and managing complex real-time state.

| Problem ID | System Name                                                                                                 | Key Concepts Applied                                                                                                                                                                         |
|------------|-------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.3.1      | [**Design a Live Chat System**](03-challenges/3.3.1-live-chat-system/) ($\text{WhatsApp}$ / $\text{Slack}$) | $\text{WebSockets}$, $\text{Kafka}$ $\text{Ordering}$, $\text{Presence}$ $\text{Service}$, $\text{Sequence}$ $\text{IDs}$, $\text{Read}$ $\text{Receipts}$, $\text{Group}$ $\text{Chat}$.    |
| 3.3.2      | **Design Uber/Lyft Ride Matching**                                                                          | $\text{Geospatial}$ $\text{Indexing}$ ($\text{H3}$/$\text{Geohash}$), $\text{Real}$-$\text{Time}$ $\text{Updates}$ ($\text{WebSockets}$), $\text{Load}$ $\text{Balancing}$ $\text{Drivers}$. |
| 3.3.3      | **Design an E-commerce Flash Sale**                                                                         | $\text{Distributed}$ $\text{Locking}$, $\text{Soft}$ $\text{Inventory}$ $\text{Reservation}$, $\text{Idempotency}$, $\text{Payment}$ $\text{Sagas}$.                                         |
| 3.3.4      | **Design a Distributed Database**                                                                           | $\text{Raft}$/$\text{Paxos}$ $\text{Consensus}$, $\text{Sharding}$, $\text{Replication}$ $\text{Topology}$, $\text{Fault}$ $\text{Tolerance}$.                                               |
| 4.1.1      | **Design a Stock Exchange Matching Engine**                                                                 |                                                                                                                                                                                              |
| 4.1.2      | **Design a Global News Feed (Google News)**                                                                 |                                                                                                                                                                                              |
| 4.1.3      | **Design a Distributed Monitoring System**                                                                  |                                                                                                                                                                                              |
| 4.1.4      | **Design a Recommendation System**                                                                          |                                                                                                                                                                                              |
| 4.1.5      | **Design a Stock Brokerage Platform**                                                                       |                                                                                                                                                                                              |

---

## 📚 Additional Resources

- **[System Design Reference Guide](./system-design-reference.md):** Quick-lookup tables for latency numbers, database
  comparisons, caching strategies, and more.
- **[Resources and Further Reading](./resources-and-further-reading.md):** Curated books, papers, courses, blogs, and
  tools to deepen your knowledge.

---

## 🎉 Contributions

We highly encourage community contributions to expand this resource! Before submitting a Pull Request, please read and
follow these guidelines:

### General Guidelines

1. **Clarity and Depth:** Content must maintain the project's goal: providing **intuitive**, easy-to-understand
   definitions while retaining technical **depth**.
2. **Naming Convention:** All new topic files must be placed in the correct category folder (e.g., 01-principles/,
   02-components/) and follow the format: `[ID]-[short-name].md` (e.g., `1.2.1-architecture-styles.md`).

### Template for Adding a New Concept Topic (Category 1 or 2)

Use this structure for any new concept file. The file should provide a clear progression from basic intuition to
technical details.

```
# [ID] Topic Title: Subtitle/Focus

## Intuitive Explanation
[Start with a simple, high-level analogy or definition that a beginner can grasp.]

## In-Depth Analysis
[Dive into the technical specifics, internal workings, and algorithms.]

### Key Concepts / Tradeoffs
* **Concept 1:** ...
* **Tradeoff:** [Discuss the pros/cons of a choice, e.g., speed vs. consistency.]

## 💡 Real-World Use Cases
* [List 2-3 specific examples of companies or scenarios where this concept is applied.]

---

## ✏️ Design Challenge
[Create a concise, open-ended question that forces the reader to apply the concepts from the file.]

```

### Template for Adding a New Design Problem (Category 3)

When adding a new design challenge to `03-challenges/`, create a folder `3.x.y-problem-name/` with **6 required files**:

#### File Structure:

```
03-challenges/3.x.y-problem-name/
├── 3.x.y-design-problem-name.md    # Main comprehensive guide
├── README.md                        # Quick overview with navigation links
├── hld-diagram.md                   # 10-15 architecture diagrams (Mermaid)
├── sequence-diagrams.md             # 10-15 sequence diagrams (Mermaid)
├── this-over-that.md                # In-depth design decision analysis
└── pseudocode.md                    # Algorithm implementations
```

#### Main File Template (3.x.y-design-problem-name.md):

**REQUIRED STRUCTURE** (must follow this exact order):

```
# [ID] Design a [System Name]

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
[Clear problem description]

---

## 2. Requirements and Scale Estimation
### Functional Requirements
* [What the system MUST do]

### Non-Functional Requirements
* **Scale:** [e.g., 500M DAU]
* **QPS:** [Read: 100k, Write: 5k]
* **Latency:** [e.g., <100ms]

### Capacity Estimation
[Back-of-envelope calculations for storage, bandwidth, QPS]

## 3. High-Level Architecture
[ASCII diagram with main components]

## 4. Data Model
[Database schemas - use ```sql for SQL only]

## 5. Component Design
[Detailed component descriptions]

## 6. Why This Over That?
[Inline explanations for major choices: DB, cache, sync/async]
* **Why PostgreSQL over MongoDB?** [Rationale with bullets]
* **Why Kafka over RabbitMQ?** [Rationale with bullets]

## 7. Bottlenecks and Scaling
[Identify bottlenecks and future scaling strategies]

## 8. Common Anti-Patterns
❌ **Anti-Pattern:** [Bad approach]
✅ **Best Practice:** [Good approach]

## 9. Alternative Approaches
[Discuss 2-3 alternative architectures not chosen]

## 10. Monitoring and Observability
[Key metrics, alerts, dashboards]

## 11. Trade-offs Summary
[Final comparison table of all major decisions]

## 12. Real-World Examples
[How Twitter, Uber, etc. solve this problem]
```

#### this-over-that.md Template:

```
# Design Decisions: [System Name]

## Decision 1: [e.g., Fanout Strategy]
### The Problem
[What are we trying to solve?]

### Options Considered
| Option | Pros | Cons | Performance | Cost |
|--------|------|------|-------------|------|
| Option A | ... | ... | ... | ... |
| Option B | ... | ... | ... | ... |

### Decision Made
[What we chose and why - 3-5 bullets]

### Rationale
1. [Detailed point 1]
2. [Detailed point 2]

### Trade-offs Accepted
[What we're sacrificing]

### When to Reconsider
[Conditions that would change this decision]

[Repeat for 5-10 major decisions]

## Summary Comparison
[Final table comparing all decisions]
```

#### pseudocode.md Template:

```
# Pseudocode Implementations: [System Name]

## Table of Contents
- [Section 1: Feature Name](#section-1)
- [Section 2: Feature Name](#section-2)

## Section 1: Feature Name

### function_name()
**Purpose:** One-line description

**Parameters:**
- param1: type - description
- param2: type - description

**Returns:** return_type - description

**Algorithm:**
\`\`\`
function function_name(param1, param2):
  // Detailed implementation
  return result
\`\`\`

**Time Complexity:** O(n)
**Space Complexity:** O(1)

**Example Usage:**
\`\`\`
result = function_name(arg1, arg2)
\`\`\`

[Include 10-20 functions organized by feature]
```

**Key Requirements:**

- **STANDARDIZED FORMAT**: All main challenge files MUST follow this exact structure:
    1. Title
    2. "Note on Implementation Details" block (referencing pseudocode.md)
    3. "📊 Visual Diagrams & Resources" section (with links to all 6 files)
    4. Section numbering starts at "## 1. Problem Statement"
    5. Continue with "## 2. Requirements...", "## 3. High-Level Architecture", etc.
- Main file: NO programming language code, NO detailed pseudocode (describe in words, reference pseudocode.md)
- All diagrams MUST have flow explanations (steps, benefits, trade-offs)
- this-over-that.md: 5-10 major decisions with detailed analysis
- pseudocode.md: 10-20 functions with complexity analysis
- See `03-challenges/3.2.2-notification-service/` as reference implementation