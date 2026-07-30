# 02-components: System Components Deep Dive

This directory contains comprehensive deep dive documents covering all major system components used in distributed systems design.

## 📁 Organization

Components are organized into **10 logical categories** for easier navigation:

### 🌐 [2.0-communication](./2.0-communication/) — Communication Protocols & APIs (7 files)
- **[2.0.1 Foundational Communication Protocols](./2.0-communication/2.0.1-foundational-communication-protocols.md)** — TCP/UDP, HTTP/S, WebSockets, WebRTC
- **[2.0.2 API Communication Styles](./2.0-communication/2.0.2-api-communication-styles.md)** — REST, gRPC, GraphQL, SOAP
- **[2.0.3 Real-Time Communication](./2.0-communication/2.0.3-real-time-communication.md)** — Long Polling, SSE, WebSockets
- **[2.0.4 Load Balancers Deep Dive](./2.0-communication/2.0.4-load-balancers-deep-dive.md)** — Layer 4/7, algorithms, health checks, SSL termination
- **[2.0.5 API Gateway Deep Dive](./2.0-communication/2.0.5-api-gateway-deep-dive.md)** — Request routing, authentication, rate limiting, BFF pattern
- **[2.0.6 Service Mesh Deep Dive](./2.0-communication/2.0.6-service-mesh-deep-dive.md)** — Sidecar pattern, mTLS, circuit breakers, traffic management
- **[2.0.7 DNS Deep Dive](./2.0-communication/2.0.7-dns-deep-dive.md)** — Resolution, record types, caching, geographic routing

### 🗄️ [2.1-databases](./2.1-databases/) — Database Systems (20 files)
- **[2.1.1 RDBMS Deep Dive](./2.1-databases/2.1.1-rdbms-deep-dive.md)** — SQL, ACID, Transactions
- **[2.1.2 NoSQL Deep Dive](./2.1-databases/2.1.2-no-sql-deep-dive.md)** — BASE, Document/Key-Value/Column stores
- **[2.1.3 Specialized Databases](./2.1-databases/2.1.3-specialized-databases.md)** — Time-series, Graph, Geospatial
- **[2.1.4 Database Scaling](./2.1-databases/2.1.4-database-scaling.md)** — Replication, Sharding, Federation
- **[2.1.5 Indexing & Query Optimization](./2.1-databases/2.1.5-indexing-and-query-optimization.md)** — B-Trees, LSM-Trees
- **[2.1.6 Data Modeling for Scale](./2.1-databases/2.1.6-data-modeling-for-scale.md)** — CQRS, Denormalization
- **[2.1.7 PostgreSQL Deep Dive](./2.1-databases/2.1.7-postgresql-deep-dive.md)** — MVCC, JSONB, PostGIS, Extensions
- **[2.1.8 MySQL Deep Dive](./2.1-databases/2.1.8-mysql-deep-dive.md)** — InnoDB, Replication, ProxySQL
- **[2.1.9 Cassandra Deep Dive](./2.1-databases/2.1.9-cassandra-deep-dive.md)** — Wide-column, Tunable consistency
- **[2.1.10 MongoDB Deep Dive](./2.1-databases/2.1.10-mongodb-deep-dive.md)** — Document model, Sharding
- **[2.1.11 Redis Deep Dive](./2.1-databases/2.1.11-redis-deep-dive.md)** — In-memory, Data structures
- **[2.1.12 DynamoDB Deep Dive](./2.1-databases/2.1.12-dynamodb-deep-dive.md)** — Serverless, GSI/LSI
- **[2.1.13 Elasticsearch Deep Dive](./2.1-databases/2.1.13-elasticsearch-deep-dive.md)** — Full-text search, Aggregations
- **[2.1.14 Neo4j Deep Dive](./2.1-databases/2.1.14-neo4j-deep-dive.md)** — Graph database, Cypher
- **[2.1.15 ClickHouse Deep Dive](./2.1-databases/2.1.15-clickhouse-deep-dive.md)** — Columnar, OLAP
- **[2.1.16 Object Storage Deep Dive](./2.1-databases/2.1.16-object-storage-deep-dive.md)** — S3, GCS, Azure Blob, multipart uploads
- **[2.1.17 Time Series Databases Deep Dive](./2.1-databases/2.1.17-time-series-databases-deep-dive.md)** — InfluxDB, TimescaleDB, Prometheus
- **[2.1.18 Vector Databases Deep Dive](./2.1-databases/2.1.18-vector-databases-deep-dive.md)** — Pinecone, Weaviate, Milvus, FAISS
- **[2.1.19 Distributed SQL Databases Deep Dive](./2.1-databases/2.1.19-distributed-sql-databases-deep-dive.md)** — CockroachDB, TiDB, Spanner
- **[2.1.20 CQRS Deep Dive](./2.1-databases/2.1.20-cqrs-deep-dive.md)** — Command-Query Responsibility Segregation

### ⚡ [2.2-caching](./2.2-caching/) — Caching Systems (4 files)
- **[2.2.1 Caching Deep Dive](./2.2-caching/2.2.1-caching-deep-dive.md)** — Cache-aside, Write-through, CDN
- **[2.2.2 Consistent Hashing](./2.2-caching/2.2.2-consistent-hashing.md)** — Ring algorithm, Virtual nodes
- **[2.2.3 Memcached Deep Dive](./2.2-caching/2.2.3-memcached-deep-dive.md)** — Slab allocation, LRU eviction
- **[2.2.4 CDN Deep Dive](./2.2-caching/2.2.4-cdn-deep-dive.md)** — Edge caching, cache invalidation, global distribution

### 📨 [2.3-messaging-streaming](./2.3-messaging-streaming/) — Messaging & Stream Processing (9 files)
- **[2.3.1 Asynchronous Communication](./2.3-messaging-streaming/2.3.1-asynchronous-communication.md)** — Queues vs Streams
- **[2.3.2 Kafka Deep Dive](./2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** — Distributed streaming
- **[2.3.3 Advanced Message Queues](./2.3-messaging-streaming/2.3.3-advanced-message-queues.md)** — RabbitMQ, SQS, SNS
- **[2.3.4 Distributed Transactions & Idempotency](./2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md)** — 2PC, Sagas
- **[2.3.5 Batch vs Stream Processing](./2.3-messaging-streaming/2.3.5-batch-vs-stream-processing.md)** — Lambda/Kappa
- **[2.3.6 Push vs Pull Data Flow](./2.3-messaging-streaming/2.3.6-push-vs-pull-data-flow.md)** — Design patterns
- **[2.3.7 Apache Spark Deep Dive](./2.3-messaging-streaming/2.3.7-apache-spark-deep-dive.md)** — Unified analytics
- **[2.3.8 Apache Flink Deep Dive](./2.3-messaging-streaming/2.3.8-apache-flink-deep-dive.md)** — True streaming, CEP
- **[2.3.9 Event Sourcing Deep Dive](./2.3-messaging-streaming/2.3.9-event-sourcing-deep-dive.md)** — Immutable event logs, state reconstruction

### 🔒 [2.4-security-observability](./2.4-security-observability/) — Security & Monitoring (6 files)
- **[2.4.1 Security Fundamentals](./2.4-security-observability/2.4.1-security-fundamentals.md)** — Auth, TLS, XSS/CSRF
- **[2.4.2 Observability](./2.4-security-observability/2.4.2-observability.md)** — Logging, Metrics, Tracing
- **[2.4.3 Prometheus & Grafana Deep Dive](./2.4-security-observability/2.4.3-prometheus-grafana-deep-dive.md)** — Time-series, PromQL
- **[2.4.4 OAuth 2.0 & JWT Deep Dive](./2.4-security-observability/2.4.4-oauth-jwt-deep-dive.md)** — OAuth 2.0 flows, OIDC
- **[2.4.5 ELK Stack & Logging Deep Dive](./2.4-security-observability/2.4.5-elk-stack-logging-deep-dive.md)** — Logstash, Kibana, Beats
- **[2.4.6 Distributed Tracing Deep Dive](./2.4-security-observability/2.4.6-distributed-tracing-deep-dive.md)** — Jaeger, Zipkin, OpenTelemetry

### 🧮 [2.5-algorithms](./2.5-algorithms/) — Distributed Algorithms (4 files)
- **[2.5.1 Rate Limiting Algorithms](./2.5-algorithms/2.5.1-rate-limiting-algorithms.md)** — Token bucket, Leaky bucket
- **[2.5.2 Consensus Algorithms](./2.5-algorithms/2.5.2-consensus-algorithms.md)** — Paxos, Raft
- **[2.5.3 Distributed Locking](./2.5-algorithms/2.5.3-distributed-locking.md)** — Redis locks, Fencing tokens
- **[2.5.4 Bloom Filters](./2.5-algorithms/2.5.4-bloom-filters.md)** — Probabilistic data structure

### 🏗️ [2.6-infrastructure](./2.6-infrastructure/) — Infrastructure & Orchestration (3 files)
- **[2.6.1 Kubernetes and Docker Deep Dive](./2.6-infrastructure/2.6.1-kubernetes-docker-deep-dive.md)** — Container orchestration
- **[2.6.2 Configuration Management Deep Dive](./2.6-infrastructure/2.6.2-configuration-management-deep-dive.md)** — etcd, Consul, Vault
- **[2.6.3 Infrastructure as Code Deep Dive](./2.6-infrastructure/2.6.3-infrastructure-as-code-deep-dive.md)** — Terraform, CloudFormation

### 🤖 [2.7-ai-ml-systems](./2.7-ai-ml-systems/) — AI & Generative ML Systems (2 files)
- **[2.7.1 Generative AI & LLM Inference RAG Architecture](./2.7-ai-ml-systems/2.7.1-llm-inference-rag-architecture.md)** — LLM serving, PagedAttention, KV Cache, RAG hybrid search, streaming tokens
- **[2.7.2 Machine Learning Feature Store](./2.7-ai-ml-systems/2.7.2-machine-learning-feature-store.md)** — Point-in-time joins, online/offline feature sync, Feast/Tecton

### 🛡️ [2.8-resilience-engineering](./2.8-resilience-engineering/) — Resilience & Reliability Engineering (1 file)
- **[2.8.1 Chaos Engineering & Resilience](./2.8-resilience-engineering/2.8.1-chaos-engineering-resilience.md)** — Circuit breakers, bulkhead, load shedding, cascading failure prevention, RPO/RTO

### 📐 [2.9-api-schema-design](./2.9-api-schema-design/) — API & Schema Evolution (1 file)
- **[2.9.1 API Design & Schema Evolution](./2.9-api-schema-design/2.9.1-api-design-schema-evolution.md)** — Backward/forward compatibility, Protobuf/Avro schema registries, versioning strategies
