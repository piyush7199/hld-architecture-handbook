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
| [system-design-reference.md](./system-design-reference.md)             | Quick Reference Guide | Latency numbers, comparison tables, formulas, and decision matrices.                                                                        |
| [resources-and-further-reading.md](./resources-and-further-reading.md) | Learning Resources    | Books, papers, courses, blogs, and tools for continued learning.                                                                            |
| README.md                                                              | (This File)           | The main project index and roadmap.                                                                                                         |

### 📊 New: Comprehensive Design Challenges Structure

Each completed design challenge now includes **6 comprehensive files** for complete understanding:

```
03-challenges/
└── 3.x.y-problem-name/
    ├── README.md                        # FULL comprehensive guide (primary document, replaces main design file)
    ├── quick-overview.md                # Quick revision guide with core concepts, architecture flows, key takeaways
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
- 📖 **Quick Revision:** quick-overview.md provides concise summaries for fast review

## 🗺️ Learning Roadmap: Core Concepts

### Category 1: Core Principles (Folder: [01-principles](./01-principles))

| Topic ID | Concept |
|---|---|
| 1.1.1 | [CAP Theorem](01-principles/1.1.1-cap-theorem.md) |
| 1.1.2 | [Latency, Throughput, and Scaling](01-principles/1.1.2-latency-throughput-scale.md) |
| 1.1.3 | [Availability and Reliability](01-principles/1.1.3-availability-reliability.md) |
| 1.1.4 | [Data Consistency Models](01-principles/1.1.4-data-consistency-models.md) |
| 1.1.5 | [Back-of-the-Envelope Calculations](01-principles/1.1.5-back-of-envelope-calculations.md) |
| 1.1.6 | [Failure Modes and Fault Tolerance](01-principles/1.1.6-failure-modes-fault-tolerance.md) |
| 1.1.7 | [Idempotency](01-principles/1.1.7-idempotency.md) |
| 1.1.8 | [Data Partitioning and Sharding](01-principles/1.1.8-data-partitioning-sharding.md) |
| 1.1.9 | [Replication Strategies](01-principles/1.1.9-replication-strategies.md) |
| 1.1.10 | [Message Delivery Guarantees](01-principles/1.1.10-message-delivery-guarantees.md) |
| 1.1.11 | [PACELC Theorem & Consensus Mechanics](01-principles/1.1.11-pacelc-theorem-consensus.md) |
| 1.2.1 | [System Architecture Styles](01-principles/1.2.1-system-architecture-styles.md) |
| 1.2.2 | [Networking Components](01-principles/1.2.2-networking-components.md) |
| 1.2.3 | [API Gateway and Service Mesh](01-principles/1.2.3-api-gateway-servicemesh.md) |
| 1.2.4 | [Domain-Driven Design (DDD) Basics](01-principles/1.2.4-domain-driven-design.md) |
| 1.2.5 | [Service Discovery](01-principles/1.2.5-service-discovery.md) |
| 1.2.6 | [Multi-Region Active-Active Systems](01-principles/1.2.6-multi-region-active-active.md) |
| 1.2.7 | [The Master HLD System Design Interview Framework](01-principles/1.2.7-hld-interview-playbook.md) |

## Category 2: Components Deep Dive (Folder: [02-components](./02-components))

> **📁 Organized into 10 logical categories for easier navigation:**
> - 🌐 **Communication** (Protocols, APIs, Real-time, Load Balancers, API Gateway, Service Mesh, DNS)
> - 🗄️ **Databases** (20 database deep dives including Object Storage, Time Series, Vector DBs, Distributed SQL & CQRS!)
> - ⚡ **Caching** (Redis, Memcached, Consistent Hashing, CDN)
> - 📨 **Messaging & Streaming** (Kafka, Spark, Flink, Message Queues, Event Sourcing)
> - 🔒 **Security & Observability** (Auth, OAuth/JWT, Monitoring, Prometheus/Grafana, Logging, ELK Stack, Distributed Tracing)
> - 🧮 **Algorithms** (Rate Limiting, Consensus, Locking, Bloom Filters)
> - 🏗️ **Infrastructure** (Kubernetes, Docker, Configuration Management, Infrastructure as Code)
> - 🤖 **AI & Generative ML Systems** (LLM Serving, PagedAttention, RAG Architecture, Feature Stores)
> - 🛡️ **Resilience Engineering** (Chaos Engineering, Circuit Breakers, Bulkhead, Cascading Failures)
> - 📐 **API & Schema Evolution** (Protobuf/Avro Schema Registries, Backward/Forward Compatibility)

### 2.0 Communication (Folder: [2.0-communication](./02-components/2.0-communication))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.0.1 | [Foundational Communication Protocols](02-components/2.0-communication/2.0.1-foundational-communication-protocols.md) | TCP vs. UDP, HTTP/S, WebSockets, WebRTC, DASH. |
| 2.0.2 | [API Communication Styles](02-components/2.0-communication/2.0.2-api-communication-styles.md) | REST, gRPC, SOAP, GraphQL (Pros, Cons, and Use Cases). |
| 2.0.3 | [Real-Time Communication](02-components/2.0-communication/2.0.3-real-time-communication.md) | Comparison of techniques for maintaining persistent connections. |
| 2.0.4 | [Load Balancers Deep Dive](02-components/2.0-communication/2.0.4-load-balancers-deep-dive.md) | Layer 4 vs Layer 7, algorithms, health checks, SSL termination. |
| 2.0.5 | [API Gateway Deep Dive](02-components/2.0-communication/2.0.5-api-gateway-deep-dive.md) | Request routing, authentication, rate limiting, BFF pattern. |
| 2.0.6 | [Service Mesh Deep Dive](02-components/2.0-communication/2.0.6-service-mesh-deep-dive.md) | Sidecar pattern, mTLS, retries, circuit breakers, distributed tracing. |
| 2.0.7 | [DNS Deep Dive](02-components/2.0-communication/2.0.7-dns-deep-dive.md) | DNS resolution, record types, caching, load balancing, geographic routing. |

### 2.1 Databases (Folder: [2.1-databases](./02-components/2.1-databases)) — 20 Deep Dives

#### Core Database Concepts

| Topic ID | Concept | Focus |
|---|---|---|
| 2.1.1 | [RDBMS Deep Dive: SQL & ACID](02-components/2.1-databases/2.1.1-rdbms-deep-dive.md) | Transactions, Isolation Levels, ACID vs. BASE. |
| 2.1.2 | [NoSQL Deep Dive: The BASE Principle](02-components/2.1-databases/2.1.2-no-sql-deep-dive.md) | Document Stores, Key-Value Stores, Column-Family. |
| 2.1.3 | [Specialized Databases](02-components/2.1-databases/2.1.3-specialized-databases.md) | Time-Series, Graph, Geospatial DBs. |
| 2.1.4 | [Database Scaling](02-components/2.1-databases/2.1.4-database-scaling.md) | Replication, Federation, Sharding Strategies. |
| 2.1.5 | [Indexing and Query Optimization](02-components/2.1-databases/2.1.5-indexing-and-query-optimization.md) | B-Trees, LSM-Trees, Denormalization Trade-offs. |
| 2.1.6 | [Data Modeling for Scale (CQRS)](02-components/2.1-databases/2.1.6-data-modeling-for-scale.md) | Data Decomposition, CQRS patterns. |

#### SQL Databases

| Topic ID | Concept | Focus |
|---|---|---|
| 2.1.7 | [PostgreSQL Deep Dive](02-components/2.1-databases/2.1.7-postgresql-deep-dive.md) | MVCC, JSONB, Full-Text Search, PostGIS, Extensions. |
| 2.1.8 | [MySQL Deep Dive](02-components/2.1-databases/2.1.8-mysql-deep-dive.md) | InnoDB Engine, MVCC, Replication, ProxySQL. |

#### NoSQL Databases

| Topic ID | Concept | Focus |
|---|---|---|
| 2.1.9 | [Cassandra Deep Dive](02-components/2.1-databases/2.1.9-cassandra-deep-dive.md) | Masterless Architecture, Wide-Column Store, Tunable Consistency. |
| 2.1.10 | [MongoDB Deep Dive](02-components/2.1-databases/2.1.10-mongodb-deep-dive.md) | Document Model, Aggregation Framework, Sharding. |
| 2.1.11 | [Redis Deep Dive](02-components/2.1-databases/2.1.11-redis-deep-dive.md) | In-Memory Data Structures, Persistence, Cluster. |
| 2.1.12 | [DynamoDB Deep Dive](02-components/2.1-databases/2.1.12-dynamodb-deep-dive.md) | Serverless NoSQL, Partition/Sort Keys, Global Tables. |

#### Specialized Databases

| Topic ID | Concept | Focus |
|---|---|---|
| 2.1.13 | [Elasticsearch Deep Dive](02-components/2.1-databases/2.1.13-elasticsearch-deep-dive.md) | Inverted Indexes, Full-Text Search, Aggregations. |
| 2.1.14 | [Neo4j Deep Dive (Graph)](02-components/2.1-databases/2.1.14-neo4j-deep-dive.md) | Property Graph Model, Cypher Query Language. |
| 2.1.15 | [ClickHouse Deep Dive (Columnar)](02-components/2.1-databases/2.1.15-clickhouse-deep-dive.md) | Columnar Storage, MergeTree Engine, Vectorized Execution. |
| 2.1.16 | [Object Storage Deep Dive](02-components/2.1-databases/2.1.16-object-storage-deep-dive.md) | S3, GCS, Azure Blob, multipart uploads, lifecycle policies. |
| 2.1.17 | [Time Series Databases Deep Dive](02-components/2.1-databases/2.1.17-time-series-databases-deep-dive.md) | InfluxDB, TimescaleDB, Prometheus, downsampling. |
| 2.1.18 | [Vector Databases Deep Dive](02-components/2.1-databases/2.1.18-vector-databases-deep-dive.md) | Pinecone, Weaviate, Milvus, FAISS, semantic search. |
| 2.1.19 | [Distributed SQL Databases Deep Dive](02-components/2.1-databases/2.1.19-distributed-sql-databases-deep-dive.md) | CockroachDB, TiDB, Google Spanner, Raft consensus. |
| 2.1.20 | [CQRS Deep Dive](02-components/2.1-databases/2.1.20-cqrs-deep-dive.md) | Command-Query Responsibility Segregation, read/write models. |

### 2.2 Caching (Folder: [2.2-caching](./02-components/2.2-caching))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.2.1 | [Caching Deep Dive](02-components/2.2-caching/2.2.1-caching-deep-dive.md) | Cache-Aside, Write-Through, CDN vs. App Cache. |
| 2.2.2 | [Consistent Hashing](02-components/2.2-caching/2.2.2-consistent-hashing.md) | Algorithm mechanics, Ring implementation. |
| 2.2.3 | [Memcached Deep Dive](02-components/2.2-caching/2.2.3-memcached-deep-dive.md) | Slab Allocation, LRU Eviction, Multi-Threading. |
| 2.2.4 | [CDN Deep Dive](02-components/2.2-caching/2.2.4-cdn-deep-dive.md) | Content Delivery Networks, edge caching, invalidation. |

### 2.3 Messaging & Streaming (Folder: [2.3-messaging-streaming](./02-components/2.3-messaging-streaming))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.3.1 | [Asynchronous Communication](02-components/2.3-messaging-streaming/2.3.1-asynchronous-communication.md) | Queues vs. Streams, Pub/Sub Models. |
| 2.3.2 | [Kafka Deep Dive](02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md) | Broker, Producer, Consumer Group, Log Compaction. |
| 2.3.3 | [Advanced Message Queues](02-components/2.3-messaging-streaming/2.3.3-advanced-message-queues.md) | RabbitMQ, SQS, SNS, Dead-Letter Queues. |
| 2.3.4 | [Distributed Transactions & Idempotency](02-components/2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md) | Two-Phase Commit (2PC), Sagas. |
| 2.3.5 | [Batch vs Stream Processing](02-components/2.3-messaging-streaming/2.3.5-batch-vs-stream-processing.md) | Lambda & Kappa Architectures. |
| 2.3.6 | [Push vs Pull Data Flow](02-components/2.3-messaging-streaming/2.3.6-push-vs-pull-data-flow.md) | Kafka (Pull) vs. RabbitMQ (Push). |
| 2.3.7 | [Apache Spark Deep Dive](02-components/2.3-messaging-streaming/2.3.7-apache-spark-deep-dive.md) | Unified Analytics Engine, RDD/DataFrame. |
| 2.3.8 | [Apache Flink Deep Dive](02-components/2.3-messaging-streaming/2.3.8-apache-flink-deep-dive.md) | True Stream Processing, Stateful Operators. |
| 2.3.9 | [Event Sourcing Deep Dive](02-components/2.3-messaging-streaming/2.3.9-event-sourcing-deep-dive.md) | Immutable event logs, state reconstruction. |

### 2.4 Security & Observability (Folder: [2.4-security-observability](./02-components/2.4-security-observability))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.4.1 | [Security Fundamentals](02-components/2.4-security-observability/2.4.1-security-fundamentals.md) | Authn/Authz, TLS, XSS/CSRF. |
| 2.4.2 | [Observability](02-components/2.4-security-observability/2.4.2-observability.md) | Logging, Metrics, Tracing. |
| 2.4.3 | [Prometheus & Grafana Deep Dive](02-components/2.4-security-observability/2.4.3-prometheus-grafana-deep-dive.md) | Time-series metrics, PromQL, alerting. |
| 2.4.4 | [OAuth 2.0 & JWT Deep Dive](02-components/2.4-security-observability/2.4.4-oauth-jwt-deep-dive.md) | OAuth 2.0 flows, JWT structure, OIDC. |
| 2.4.5 | [ELK Stack & Logging Deep Dive](02-components/2.4-security-observability/2.4.5-elk-stack-logging-deep-dive.md) | Elasticsearch, Logstash, Kibana, Beats. |
| 2.4.6 | [Distributed Tracing Deep Dive](02-components/2.4-security-observability/2.4.6-distributed-tracing-deep-dive.md) | Jaeger, Zipkin, OpenTelemetry. |

### 2.5 Distributed Algorithms (Folder: [2.5-algorithms](./02-components/2.5-algorithms))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.5.1 | [Rate Limiting Algorithms](02-components/2.5-algorithms/2.5.1-rate-limiting-algorithms.md) | Token Bucket, Leaky Bucket mechanisms. |
| 2.5.2 | [Consensus Algorithms](02-components/2.5-algorithms/2.5.2-consensus-algorithms.md) | Paxos / Raft, Distributed Locks. |
| 2.5.3 | [Distributed Locking](02-components/2.5-algorithms/2.5.3-distributed-locking.md) | Redis locks, TTL, Fencing Tokens. |
| 2.5.4 | [Bloom Filters](02-components/2.5-algorithms/2.5.4-bloom-filters.md) | Hash Functions, False Positives. |

### 2.6 Infrastructure (Folder: [2.6-infrastructure](./02-components/2.6-infrastructure))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.6.1 | [Kubernetes and Docker Deep Dive](02-components/2.6-infrastructure/2.6.1-kubernetes-docker-deep-dive.md) | Container orchestration, pods, deployments. |
| 2.6.2 | [Configuration Management Deep Dive](02-components/2.6-infrastructure/2.6.2-configuration-management-deep-dive.md) | etcd, Consul, Vault, secrets management. |
| 2.6.3 | [Infrastructure as Code Deep Dive](02-components/2.6-infrastructure/2.6.3-infrastructure-as-code-deep-dive.md) | Terraform, CloudFormation, Pulumi. |

### 2.7 AI & Generative ML Systems (Folder: [2.7-ai-ml-systems](./02-components/2.7-ai-ml-systems))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.7.1 | [Generative AI & LLM Inference RAG Architecture](02-components/2.7-ai-ml-systems/2.7.1-llm-inference-rag-architecture.md) | LLM serving, PagedAttention, KV Cache, RAG hybrid search, streaming tokens. |
| 2.7.2 | [Machine Learning Feature Store](02-components/2.7-ai-ml-systems/2.7.2-machine-learning-feature-store.md) | Point-in-time joins, online/offline feature sync, Feast/Tecton. |

### 2.8 Resilience Engineering (Folder: [2.8-resilience-engineering](./02-components/2.8-resilience-engineering))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.8.1 | [Chaos Engineering & Resilience](02-components/2.8-resilience-engineering/2.8.1-chaos-engineering-resilience.md) | Circuit breakers, bulkhead, load shedding, cascading failure prevention, RPO/RTO. |

### 2.9 API & Schema Evolution (Folder: [2.9-api-schema-design](./02-components/2.9-api-schema-design))

| Topic ID | Concept | Focus |
|---|---|---|
| 2.9.1 | [API Design & Schema Evolution](02-components/2.9-api-schema-design/2.9.1-api-design-schema-evolution.md) | Backward/forward compatibility, Protobuf/Avro schema registries, versioning strategies. |

## 🗺️ Design Challenges Roadmap (Category 3)

**📊 Each challenge folder contains 6 comprehensive files:**

- **[README.md]** - Complete comprehensive guide with all content
- **[quick-overview.md]** - Quick revision guide with core concepts and key takeaways
- **[hld-diagram.md]** - System architecture diagrams
- **[sequence-diagrams.md]** - Detailed interaction flows
- **[this-over-that.md]** - Design decisions & trade-offs
- **[pseudocode.md]** - Detailed algorithm implementations

### Easy Challenges (Focus: Core Components, Caching, Databases)

| Problem ID | System Name | Key Concepts Applied |
|---|---|---|
| 3.1.1 | **[Design a URL Shortener](03-challenges/3.1.1-url-shortener/)** (TinyURL) | Hashing, Base62 Encoding, Read-Heavy Scaling, Sharding Key, Cache Aside |
| 3.1.2 | **[Design a Distributed Cache](03-challenges/3.1.2-distributed-cache/)** (Redis / Memcached) | Consistent Hashing, Eviction Policies (LRU), Replication, Failover, TTL |
| 3.1.3 | **[Design a Distributed ID Generator](03-challenges/3.1.3-distributed-id-generator/)** (Snowflake) | 64-bit ID Structure, Worker ID Assignment, Clock Drift, etcd |

### Medium Challenges (Focus: Asynchrony, Feeds, Microservices, Geo-Spatial)

| Problem ID | System Name | Key Concepts Applied |
|---|---|---|
| 3.2.1 | [**Design a Twitter/X Timeline**](03-challenges/3.2.1-twitter-timeline/) | Fanout on Write vs. Fanout on Read, Caching Hierarchy, Kafka |
| 3.2.2 | [**Design a Notification Service**](03-challenges/3.2.2-notification-service/) | Multi-Channel, WebSockets, Kafka Streams, Circuit Breakers, Rate Limiting |
| 3.2.3 | [**Design a Distributed Web Crawler**](03-challenges/3.2.3-web-crawler/) | URL Frontier, Bloom Filter, Duplicate Detection, Politeness, Rate Limiting |
| 3.2.4 | [**Design a Global Rate Limiter**](03-challenges/3.2.4-global-rate-limiter/) | Token Bucket, Sliding Window, Atomic INCR, Circuit Breakers |

### Hard & Advanced Enterprise Challenges

| Problem ID | System Name | Key Concepts Applied |
|---|---|---|
| 3.3.1 | [**Design a Live Chat System**](03-challenges/3.3.1-live-chat-system/) (WhatsApp / Slack) | WebSockets, Kafka Ordering, Presence Service, Sequence IDs |
| 3.3.2 | [**Design Uber/Lyft Ride Matching**](03-challenges/3.3.2-uber-ride-matching/) | Redis Geo, Geohash Indexing, Kafka Buffer, Geographic Sharding |
| 3.3.3 | [**Design an E-commerce Flash Sale**](03-challenges/3.3.3-flash-sale/) | Redis Atomic DECR, Saga Pattern, Load Shedding, Idempotency Keys |
| 3.3.4 | [**Design a Distributed Database**](03-challenges/3.3.4-distributed-database/) | Raft Consensus, 2PC, Range Sharding, LSM Tree, MVCC |
| 3.4.1 | [**Design a Stock Exchange Matching Engine**](03-challenges/3.4.1-stock-exchange/) | LMAX Disruptor, DPDK Kernel Bypass, Red-Black Tree, WAL |
| 3.4.2 | [**Design a Global News Feed**](03-challenges/3.4.2-news-feed/) (Google News) | NLP Pipelines, LSH Deduplication, Elasticsearch, Kappa Architecture |
| 3.4.3 | [**Design a Distributed Monitoring System**](03-challenges/3.4.3-monitoring-system/) (Prometheus) | M3DB TSDB, Delta-of-Delta Encoding, Rollup Aggregations |
| 3.4.4 | [**Design a Recommendation System**](03-challenges/3.4.4-recommendation-system/) (Netflix) | Lambda Architecture, ALS Collaborative Filtering, FAISS ANN |
| 3.4.5 | [**Design a Stock Brokerage Platform**](03-challenges/3.4.5-stock-brokerage/) (Zerodha) | FIX Protocol, WebSockets Push, Event Sourcing, Redis Quotes |
| 3.4.6 | [**Design a Collaborative Editor**](03-challenges/3.4.6-collaborative-editor/) (Google Docs) | OT/CRDT, WebSockets, Event Sourcing, CQRS |
| 3.4.7 | [**Design an Online Code Editor / Judge**](03-challenges/3.4.7-online-code-judge/) | Execution Isolation (Sandboxing), Queue Priority, Throttling |
| 3.4.8 | [**Design a Video Streaming System**](03-challenges/3.4.8-video-streaming-system/) (YouTube) | CDN Hierarchy, DASH/HLS, Encoding Pipelines, DRM |
| 3.5.1 | [**Design a Payment Gateway**](03-challenges/3.5.1-payment-gateway) (Stripe) | Idempotency, PCI Compliance, Tokenization, Fraud Detection |
| 3.5.2 | [**Design Ad Click Aggregator**](03-challenges/3.5.2-ad-click-aggregator/) (Google Ads) | Kappa Architecture, Low-Latency Counters, Reconciliation |
| 3.5.3 | [**Design YouTube Top K**](03-challenges/3.5.3-youtube-top-k/) (Trending Algorithm) | Redis Sorted Sets, Decay Functions, Ranking Pipeline |
| 3.5.4 | [**Design Instagram/Pinterest Feed**](03-challenges/3.5.4-instagram-pinterest-feed/) | Media Pipeline, Fanout on Write vs. Recommendation Merge |
| 3.5.5 | [**Design Live Commenting**](03-challenges/3.5.5-live-commenting/) (Twitch) | Massive Fanout, WebSockets, Adaptive Throttling |
| 3.5.6 | [**Design Yelp/Google Maps**](03-challenges/3.5.6-yelp-google-maps/) | Geospatial Search, Geohash Partitioning, Hierarchical Sharding |
| 3.5.7 | [**Design Authenticator App**](03-challenges/3.5.7-authenticator-app/) | TOTP Algorithm, Offline Operation, Device HSM, Push Auth |
| 3.5.8 | [**Design Single Sign-On (SSO) System**](03-challenges/3.5.8-single-sign-on-sso/) | OAuth 2.0 / OIDC, SAML 2.0, JWT Tokens, Token Rotation |
| 3.6.1 | [**Design Enterprise LLM & RAG Copilot**](03-challenges/3.6.1-llm-rag-copilot-system/) (Enterprise ChatGPT) | vLLM Serving, PagedAttention, Hybrid HNSW+BM25 Search, RBAC ACL Vector Filtering, Token Streaming |

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
├── README.md                        # Main comprehensive guide (primary document, replaces old main design file)
├── quick-overview.md                # Quick revision guide with core concepts, architecture flows, key takeaways
├── hld-diagram.md                   # 10-15 architecture diagrams (Mermaid)
├── sequence-diagrams.md             # 10-15 sequence diagrams (Mermaid)
├── this-over-that.md                # In-depth design decision analysis
└── pseudocode.md                    # Algorithm implementations
```

**⚠️ IMPORTANT:** The main design file (`3.x.y-design-problem-name.md`) should NOT exist in the final structure. Its
content should be moved to `README.md`, and a `quick-overview.md` file should be created for quick revision purposes.

#### Main File Template (README.md):

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

- **STANDARDIZED FORMAT**: All README files MUST follow this exact structure:
    1. Title
    2. "Note on Implementation Details" block (referencing pseudocode.md)
    3. "📊 Visual Diagrams & Resources" section (with links to all supplementary files)
    4. Section numbering starts at "## 1. Problem Statement"
    5. Continue with "## 2. Requirements...", "## 3. High-Level Architecture", etc.
- **README.md**: NO programming language code, NO detailed pseudocode (describe in words, reference pseudocode.md)
- **quick-overview.md**: Concise revision guide (300-600 lines) with core concepts, architecture flows, key design
  decisions, bottlenecks, anti-patterns, trade-offs, real-world examples, and key takeaways
- All diagrams MUST have flow explanations (steps, benefits, trade-offs)
- this-over-that.md: 5-10 major decisions with detailed analysis
- pseudocode.md: 10-20 functions with complexity analysis
- See `03-challenges/3.1.1-url-shortener/` as reference implementation