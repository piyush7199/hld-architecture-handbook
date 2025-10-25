# 💡 HLD Architecture Handbook: The Intuitive System Designer

## Project Goal

This repository, the **HLD Architecture Handbook**, is designed to be a comprehensive, self-paced learning guide for
mastering **High-Level Design (HLD)** and large-scale system architecture. We focus on providing **intuitive definitions
** and
**in-depth explanations** of core concepts, followed by structured design challenges. The ultimate goal is to help you
understand **the 'Why'** behind every architectural choice—the trade-offs, constraints, and future-proofing
considerations
necessary for building systems at scale.

**Audience:** Engineers with basic programming knowledge looking to transition from small-scale development to designing
highly scalable, reliable, and performant distributed systems.

## 📁 Repository Structure

The content is organized into three progressive categories:

| Folder                           | Category Name        | Focus                                                                                                                                       |
|----------------------------------|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| [01-principles](./01-principles) | Core Principles      | Core theoretical concepts: Scale, Availability, CAP Theorem, and foundational architecture styles.                                          |
| [02-components](./02-components) | Components Deep Dive | In-depth analysis of specialized databases, caching, sharding, messaging, and concurrency control.                                          |
| [03-challenges](./03-challenges) | Design Challenges    | Real-world design problems (e.g., URL Shortener, Twitter, E-commerce Flash Sale) applying the concepts learned in the first two categories. |
| README.md                                                            | (This File)                   | The main project index and roadmap.                                                 |
| [system-design-reference.md](./system-design-reference.md)           | Quick Reference Guide         | Latency numbers, comparison tables, formulas, and decision matrices.                |
| [resources-and-further-reading.md](./resources-and-further-reading.md) | Learning Resources            | Books, papers, courses, blogs, and tools for continued learning.                    |

## 🗺️ Learning Roadmap: Core Concepts

We will cover the following topics in sequence before moving to the Design Challenges.

### Category 1: Core Principles (Folder: [01-principles](./01-principles))

| Topic ID | Concept                                                                                              |
|----------|------------------------------------------------------------------------------------------------------|
| 1.1.1    | [CAP Theorem](01-principles/1.1.1-cap-theorem.md)                                                    |
| 1.1.2    | [Latency, Throughput, and Scaling](01-principles/1.1.2-latency-throughput-scale.md)                  |
| 1.1.3    | [Availability and Reliability](01-principles/1.1.3-availability-reliability.md)                      |
| 1.1.4    | [Data Consistency Models](01-principles/1.1.4-data-consistency-models.md)                            |
| 1.1.5    | [Back-of-the-Envelope Calculations](01-principles/1.1.5-back-of-envelope-calculations.md)            |
| 1.1.6    | [Failure Modes and Fault Tolerance](01-principles/1.1.6-failure-modes-fault-tolerance.md)            |
| 1.2.1    | [System Architecture Styles](01-principles/1.2.1-system-architecture-styles.md)                      |
| 1.2.2    | [Networking Components](01-principles/1.2.2-networking-components.md)                                |
| 1.2.3    | [API Gateway and Service Mesh](01-principles/1.2.3-api-gateway-servicemesh.md)                       |
| 1.2.4    | [Domain-Driven Design (DDD) Basics](01-principles/1.2.4-domain-driven-design.md)                     |

## Category 2: Components Deep Dive (Folder: [02-components](./02-components))

| Topic ID | Concept                                                                                                   | Focus                                                                                                     |
|----------|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 2.0.1    | [Foundational Communication Protocols](02-components/2.0.1-foundational-communication-protocols.md)       | TCP vs. UDP, HTTP/S, WebSockets, WebRTC, DASH.                                                            |
| 2.0.2    | [API Communication Styles](02-components/2.0.2-api-communication-styles.md)                               | REST, gRPC, SOAP, GraphQL (Pros, Cons, and Use Cases).                                                    |
| 2.0.3    | [Real-Time Communication](02-components/2.0.3-real-time-communication.md)                                 | Comparison of techniques for maintaining persistent or near-persistent connections for real-time updates. |
| 2.1.1    | [RDBMS Deep Dive: SQL & ACID](02-components/2.1.1-rdbms-deep-dive.md)                                     | Transactions, Isolation Levels, ACID vs. BASE.                                                            |
| 2.1.2    | [NoSQL Deep Dive: The BASE Principle](02-components/2.1.2-no-sql-deep-dive.md)                            | Document Stores, Key-Value Stores, Column-Family.                                                         |
| 2.1.3    | [Specialized Databases](02-components/2.1.3-specialized-databases.md)                                     | Time-Series, Graph, Geospatial DBs (e.g., Redis Streams, Neo4j).                                          |
| 2.1.4    | [Database Scaling](02-components/2.1.4-database-scaling.md)                                               | Replication (Master-Slave), Federation, Sharding Strategies.                                              |
| 2.1.5    | [Indexing and Query Optimization](02-components/2.1.5-indexing-and-query-optimization.md)                 | B-Trees, LSM-Trees, Denormalization Trade-offs.                                                           |
| 2.1.6    | [Data Modeling for Scale (CQRS)](02-components/2.1.6-data-modeling-for-scale.md)                          | Denormalization, Data Decomposition, Command-Query Responsibility Segregation (CQRS).                     |
| 2.2.1    | [Caching Deep Dive](02-components/2.2.1-caching-deep-dive.md)                                             | Cache-Aside, Write-Through, CDN vs. App-Level Cache.                                                      |
| 2.2.2    | [Consistent Hashing](02-components/2.2.2-consistent-hashing.md)                                           | Algorithm mechanics, Ring implementation, how it minimizes data movement.                                 |
| 2.3.1    | [Asynchronous Communication](02-components/2.3.1-asynchronous-communication.md)                           | Queues vs. Streams, Pub/Sub Models, Backpressure.                                                         |
| 2.3.2    | [Kafka Deep Dive](02-components/2.3.2-kafka-deep-dive.md)                                                 | Broker, Producer, Consumer Group, Partitions, Offset Management, Log Compaction.                          |
| 2.3.3    | [Advanced Message Queues (RabbitMQ, SQS, SNS)](02-components/2.3.3-advanced-message-queues.md)            | Comparison of broker-based vs. managed queues, Dead-Letter Queues (DLQs).                                 |
| 2.3.4    | [Distributed Transactions & Idempotency](02-components/2.3.4-distributed-transactions-and-idempotency.md) | Two-Phase Commit (2PC), Sagas, ensuring atomic operations.                                                |
| 2.3.5    | [Batch vs Stream Processing](02-components/2.3.5-batch-vs-stream-processing.md)                           | Detailed look at the Lambda and Kappa Architectures, latency vs. completeness trade-offs.                 |
| 2.3.6    | [Push vs Pull Data Flow](02-components/2.3.6-push-vs-pull-data-flow.md)                                   | Architectural choices in messaging systems (e.g., Kafka (Pull) vs. RabbitMQ (Push)).                      |
| 2.4.1    | [Security Fundamentals](02-components/2.4.1-security-fundamentals.md)                                     | Authn/Authz (JWT), TLS/Encryption, Cross-Site Scripting (XSS) & CSRF.                                     |
| 2.4.2    | [Observability](02-components/2.4.2-observability.md)                                                     | Logging, Metrics (Prometheus), Distributed Tracing (Jaeger/Zipkin), Alerting.                             |
| 2.5.1    | [Rate Limiting Algorithms](02-components/2.5.1-rate-limiting-algorithms.md)                               | Token Bucket, Leaky Bucket, Fixed Window counter mechanisms.                                              |
| 2.5.2    | [Consensus Algorithms](02-components/2.5.2-consensus-algorithms.md)                                       | Paxos / Raft, Distributed Locks (ZooKeeper/etcd), solving the concurrency problem.                        |
| 2.5.3    | [Distributed Locking](02-components/2.5.3-distributed-locking.md)                                         | $\text{Redis}$ locks, $\text{TTL}$, Fencing Tokens, ensuring mutual exclusion.                            |
| 2.5.4    | [Bloom Filters](02-components/2.5.4-bloom-filters.md)                                                     | Intuition, Hash Functions, False Positives, use cases (e.g., CDN cache lookups).                          |

## 🗺️ Design Challenges Roadmap (Category 3)

### Easy Challenges (Focus: Core Components, Caching, Databases)

These problems require solid application of scaling fundamentals, hashing, and database choices.

| Problem ID | System Name                                                                                  | Key Concepts Applied                                                                                                              |
|------------|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| 3.1.1      | [**Design a URL Shortener** ($\text{TinyURL}$)](03-challenges/3.1.1-design-url-shortener.md) | Hashing, Base62 Encoding, Read-Heavy Scaling, Sharding Key, $\text{Cache}$ $\text{Aside}$.                                        |
| 3.1.2      | [**Design a Distributed Cache** ($\text{Redis}$/$\text{Memcached}$)](03-challenges/3.1.2-design-distributed-cache.md) | Consistent Hashing, Eviction Policies ($\text{LRU}$), Replication, Failover, $\text{TTL}$, Cache Stampede Prevention. |
| 3.1.3      | **Design a Distributed ID Generator**                                                        | SnowFlake Algorithm, $\text{Timestamp}$ $\text{vs.}$ $\text{Sequence}$, $\text{High}$ $\text{Availability}$.                      |

### Medium Challenges (Focus: Asynchrony, Feeds, Microservices, Geo-Spatial)

These problems involve decoupling services, handling fan-out, and managing complex data models.

| Problem ID | System Name                      | Key Concepts Applied                                                                                                                                              |
|------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.2.1      | Design a Twitter/X Timeline      | $\text{Fanout}$ $\text{on}$ $\text{Write}$ vs. $\text{Fanout}$ $\text{on}$ $\text{Read}$, $\text{Caching}$ $\text{Hierarchy}$, $\text{Queuing}$ ($\text{Kafka}$). |
| 3.2.2      | Design a Notification Service    | $\text{Pub}$/$\text{Sub}$ ($\text{SNS}$), $\text{Real}$-$\text{Time}$ ($\text{WebSockets}$), $\text{Batching}$ $\text{Notifications}$, $\text{DLQ}$.              |
| 3.2.3      | Design a Distributed Web Crawler | $\text{URL}$ $\text{Frontier}$, $\text{Bloom}$ $\text{Filter}$, $\text{Rate}$ $\text{Limiting}$, $\text{Queueing}$.                                               |
| 3.2.4      | Design a Global Rate Limiter     | $\text{Distributed}$ $\text{Counters}$ ($\text{Redis}$ $\text{Cluster}$), $\text{Leaky}$ $\text{Bucket}$ $\text{Algorithm}$, $\text{API}$ $\text{Gateway}$.       |

### Hard Challenges (Focus: Consistency, Transactions, Consensus, Real-Time Geo)

These problems require advanced pattern usage, strong consistency guarantees, and managing complex real-time state.

| Problem ID | System Name                                                        | Key Concepts Applied                                                                                                                                                                         |
|------------|--------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.3.1      | **Design a Live Chat System ($\text{WhatsApp}$ / $\text{Slack}$)** | $\text{WebSockets}$, $\text{Message}$ $\text{Broker}$ ($\text{Kafka}$), $\text{Presence}$ $\text{Service}$, $\text{Multi}$-$\text{Region}$ $\text{Consistency}$.                             |
| 3.3.2      | **Design Uber/Lyft Ride Matching**                                 | $\text{Geospatial}$ $\text{Indexing}$ ($\text{H3}$/$\text{Geohash}$), $\text{Real}$-$\text{Time}$ $\text{Updates}$ ($\text{WebSockets}$), $\text{Load}$ $\text{Balancing}$ $\text{Drivers}$. |
| 3.3.3      | **Design an E-commerce Flash Sale**                                | $\text{Distributed}$ $\text{Locking}$, $\text{Soft}$ $\text{Inventory}$ $\text{Reservation}$, $\text{Idempotency}$, $\text{Payment}$ $\text{Sagas}$.                                         |
| 3.3.4      | **Design a Distributed Database**                                  | $\text{Raft}$/$\text{Paxos}$ $\text{Consensus}$, $\text{Sharding}$, $\text{Replication}$ $\text{Topology}$, $\text{Fault}$ $\text{Tolerance}$.                                               |

---

---

## 📚 Additional Resources

- **[System Design Reference Guide](./system-design-reference.md):** Quick-lookup tables for latency numbers, database comparisons, caching strategies, and more.
- **[Resources and Further Reading](./resources-and-further-reading.md):** Curated books, papers, courses, blogs, and tools to deepen your knowledge.

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

Use this structure for all design problem files (placed in `03-challenges/`). This format mimics the standard system
design interview process, with a strong emphasis on architectural justification.

```
# [ID] Design a [System Name] (e.g., Design a Twitter Timeline)

## 1. Requirements and Scale

### Functional Requirements (What the system MUST do)
* [e.g., Users must be able to post messages (tweets).]

### Non-Functional Requirements (Constraints/Performance)
* **Scale:** [e.g., 500 Million Daily Active Users (DAU)]
* **QPS:** [e.g., Read QPS: 100k, Write QPS: 5k]
* **Availability:** [e.g., High availability is critical (99.99%)]

## 2. Capacity Estimation and Data Model
[Provide basic calculations for Storage and Bandwidth. Detail the initial database schemas.]

## 3. High-Level Architecture
[A diagram or description of the main components: CDN, LB, API Gateway, Services, Databases.]

## 4. Deep Dive: Architectural Choices and Trade-offs
[This section is critical. For every major component (DB, Caching, Messaging), you MUST explain the choice.]

### Example: Database Choice (SQL vs. NoSQL)
| Choice | Rationale (Why this over the alternative?) | Trade-off / Future Scalability Issue |
| :--- | :--- | :--- |
| **PostgreSQL (SQL)** | Chosen for its ACID properties, which are critical for financial transactions and order integrity. | Vertical scaling limits; future horizontal scaling will require complex application-level sharding. |
| **Cassandra (NoSQL)** | Chosen for extreme read/write performance and horizontal scaling for the social feed data. | **Eventual Consistency** means a user might briefly miss a new post. We accept this trade-off for speed. |

## 5. Failure Handling and Future Scaling
[Discuss how the chosen architecture scales further, what happens if a service fails (fault tolerance), and the key bottlenecks of the final design.]
```