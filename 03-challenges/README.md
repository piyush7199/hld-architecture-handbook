# 03-challenges: High-Level System Design Challenges

Welcome to the **Design Challenges** section of the HLD Architecture Handbook. This directory contains 28 real-world system design challenges, categorized by complexity and domain.

Each challenge is structured as a **complete 6-file suite** providing comprehensive coverage from high-level architecture to low-level pseudocode algorithms.

---

## 📁 6-File Challenge Structure

Each challenge directory contains:

```
03-challenges/3.x.y-problem-name/
├── README.md            # Complete comprehensive guide with all content & trade-offs
├── quick-overview.md    # Revision cheat sheet with key metrics & architectural takeaways
├── hld-diagram.md       # High-level architecture & block diagrams (10-15 Mermaid diagrams)
├── sequence-diagrams.md # Detailed interaction flows & failure recovery sequences
├── this-over-that.md    # In-depth architectural trade-offs ("Why Option A over B")
└── pseudocode.md        # Production algorithms with time & space complexity analysis
```

---

## 🗺️ Complete Challenges Roadmap (28 Problems)

### 🟢 Easy Challenges (Core Scale, Caching, & ID Generation)

| ID | Challenge | Domain | Core Architectural Trade-offs |
| :--- | :--- | :--- | :--- |
| **3.1.1** | **[Design a URL Shortener](./3.1.1-url-shortener/)** | Web Scale | Base62 Encoding vs. MD5 Hashing; 301 Permanent vs. 302 Temporary Redirects; Sharding by Alias vs. User ID. |
| **3.1.2** | **[Design a Distributed Cache](./3.1.2-distributed-cache/)** | Caching | Consistent Hashing with Virtual Nodes vs. Fixed Slots; LRU vs. LFU Eviction; Cache Stampede XFetch Algorithm. |
| **3.1.3** | **[Design a Distributed ID Generator](./3.1.3-distributed-id-generator/)** | Core Infra | Snowflake 64-bit ID Layout vs. UUIDv4 vs. DB Auto-Increment; Clock drift handling via etcd leases. |

---

### 🟡 Medium Challenges (Asynchrony, Feeds, Microservices, & Geo-Spatial)

| ID | Challenge | Domain | Core Architectural Trade-offs |
| :--- | :--- | :--- | :--- |
| **3.2.1** | **[Design a Twitter/X Timeline](./3.2.1-twitter-timeline/)** | Social Feed | Fanout-on-Write for normal users vs. Fanout-on-Read for celebrity users; Redis timeline caching hierarchy. |
| **3.2.2** | **[Design a Notification Service](./3.2.2-notification-service/)** | Messaging | Multi-channel provider fallback; WebSockets vs. Push Notifications; DLQ retry exponential backoff. |
| **3.2.3** | **[Design a Distributed Web Crawler](./3.2.3-web-crawler/)** | Web Scale | URL Frontier host-based politeness delay queues; Bloom Filters vs. Cuckoo Filters for URL deduplication. |
| **3.2.4** | **[Design a Global Rate Limiter](./3.2.4-global-rate-limiter/)** | Security | Local L1 Memory Cache + Remote Redis L2 Lua script; Token Bucket vs. Sliding Window Counter under bursty load. |

---

### 🔴 Hard & Enterprise Challenges (Consistency, Low Latency, Consensus, AI & Real-Time)

| ID | Challenge | Domain | Core Architectural Trade-offs |
| :--- | :--- | :--- | :--- |
| **3.3.1** | **[Design a Live Chat System](./3.3.1-live-chat-system/)** | Real-Time | WebSockets vs. Long Polling; Cassandra Wide-Column (`chat_id, message_id DESC`) vs. RDBMS; Monotonic sequence ordering. |
| **3.3.2** | **[Design Uber/Lyft Ride Matching](./3.3.2-uber-ride-matching/)** | Real-Time Geo | S2 Geometry / Geohash spatial index partitioning vs. R-Tree; Driver location stream processing via Kafka & Redis Geo. |
| **3.3.3** | **[Design an E-Commerce Flash Sale](./3.3.3-flash-sale/)** | E-Commerce | Atomic Redis DECR with Inventory Sub-Counter Sharding (10 sub-buckets); Saga pattern orchestration vs. 2PC. |
| **3.3.4** | **[Design a Distributed Database](./3.3.4-distributed-database/)** | Storage Engine | Raft Consensus leader election; Multi-Version Concurrency Control (MVCC) snapshot isolation; Range vs. Hash Sharding. |
| **3.4.1** | **[Design a Stock Exchange Matching Engine](./3.4.1-stock-exchange/)** | Low Latency | LMAX Disruptor RingBuffer lock-free processing (<100µs p99); Red-Black tree price-time priority; DPDK Kernel Bypass. |
| **3.4.2** | **[Design a Global News Feed](./3.4.2-news-feed/)** | Big Data | Real-Time Feature Store + LSH (Locality Sensitive Hashing) deduplication; Kappa stream vs. Lambda batch architecture. |
| **3.4.3** | **[Design a Distributed Monitoring System](./3.4.3-monitoring-system/)** | Observability | Delta-of-Delta encoding (Gorilla compression) for time-series; M3DB TSDB high-cardinality label indexing. |
| **3.4.4** | **[Design a Recommendation System](./3.4.4-recommendation-system/)** | ML & AI | Candidate Generation (FAISS Approximate Nearest Neighbors) + Candidate Scoring (Deep & Cross Network) 2-stage pipeline. |
| **3.4.5** | **[Design a Stock Brokerage Platform](./3.4.5-stock-brokerage/)** | Fintech | FIX Protocol gateway integration; Event Sourcing for audit ledgers; Redis quotes push engine. |
| **3.4.6** | **[Design a Collaborative Editor](./3.4.6-collaborative-editor/)** | Real-Time | Conflict-free Replicated Data Types (CRDT) vs. Operational Transformation (OT) for decentralized vs. centralized sync. |
| **3.4.7** | **[Design an Online Code Editor / Judge](./3.4.7-online-code-judge/)** | Compute | Linux Cgroups v2 & Docker Seccomp container isolation; CPU/RAM/pids limit enforcement; Priority execution queues. |
| **3.4.8** | **[Design a Video Streaming System](./3.4.8-video-streaming-system/)** | Media | Adaptive Bitrate Streaming (DASH / HLS); CDN edge caching hierarchy; Parallel video chunk encoding pipeline. |
| **3.5.1** | **[Design a Payment Gateway](./3.5.1-payment-gateway/)** | Fintech | Two-Phase Commit vs. Saga orchestrator; Idempotency key state machine (`PENDING` $\rightarrow$ `COMPLETED`); HMAC webhook signatures. |
| **3.5.2** | **[Design Ad Click Aggregator](./3.5.2-ad-click-aggregator/)** | AdTech | Sliding window aggregation in Apache Flink; Exactly-once processing via two-phase commit sinks; Bot click filtering. |
| **3.5.3** | **[Design YouTube Top K Algorithm](./3.5.3-youtube-top-k/)** | Algorithms | Heavy-Hitters Count-Min Sketch + Min-Heap algorithm vs. Redis Sorted Sets with time-decay functions ($S = V / (T+2)^G$). |
| **3.5.4** | **[Design Instagram/Pinterest Feed](./3.5.4-instagram-pinterest-feed/)** | Social Media | Media transcoding & CDN push; Graph-based recommendation merge for heterogeneous media types. |
| **3.5.5** | **[Design Live Commenting](./3.5.5-live-commenting/)** | Streaming | Mass fanout WebSockets with adaptive rate throttling (sampling comments under 100K/sec burst). |
| **3.5.6** | **[Design Yelp / Google Maps](./3.5.6-yelp-google-maps/)** | Geo-Spatial | Hierarchical Geohash spatial indexing vs. H3 Hexagonal; 8-neighbor cell querying for boundary resolution. |
| **3.5.7** | **[Design Authenticator App](./3.5.7-authenticator-app/)** | Security | Time-based One-Time Password (TOTP, RFC 6238) algorithm; HMAC-SHA1 counter calculation; Clock skew tolerance. |
| **3.5.8** | **[Design Single Sign-On (SSO) System](./3.5.8-single-sign-on-sso/)** | Identity | OAuth 2.0 Authorization Code Grant with PKCE vs. SAML 2.0; OIDC JWT token validation; Silent refresh rotation. |
| **3.6.1** | **[Design Enterprise LLM & RAG Copilot](./3.6.1-llm-rag-copilot-system/)** | GenAI | vLLM Continuous Batching; PagedAttention KV Cache block allocation; Hybrid HNSW+BM25 Search; RBAC ACL Vector Filtering. |
