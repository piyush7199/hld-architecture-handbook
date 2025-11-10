# Collaborative Editor - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A highly scalable, real-time collaborative document editing platform like Google Docs that enables multiple users to
edit the same document simultaneously. The system handles concurrent edits with proper conflict resolution, provides
sub-100ms latency for edit propagation, supports offline editing with seamless sync, and maintains strong consistency
across all clients while handling millions of concurrent users.

The platform resolves conflicting edits deterministically, preserves document history for version control, and scales to
support 100,000+ concurrent documents with 10+ simultaneous editors per document.

**Key Characteristics:**

- **Real-time collaboration** - Multiple users editing simultaneously with instant updates
- **Conflict resolution** - OT (Operational Transformation) or CRDT algorithms
- **Low latency** - <100ms propagation for edits
- **Offline editing** - Support editing when disconnected, syncing changes upon reconnection
- **Strong consistency** - All users converge to the same document state
- **High throughput** - 1M+ events/sec peak
- **Version history** - Complete edit history preserved

---

## Requirements & Scale

### Functional Requirements

| Requirement                  | Description                                                                            | Priority    |
|------------------------------|----------------------------------------------------------------------------------------|-------------|
| **Real-Time Collaboration**  | Multiple users can edit the same document simultaneously with instant updates          | Must Have   |
| **Conflict Resolution**      | Edits from different users on the same section must merge correctly without corruption | Must Have   |
| **Persistent Storage**       | All edits and document history must be reliably saved                                  | Must Have   |
| **Version History**          | Users can view and restore previous document versions                                  | Must Have   |
| **Offline Editing**          | Support editing when disconnected, syncing changes upon reconnection                   | Must Have   |
| **Cursor Tracking**          | Display real-time cursor positions of all collaborators                                | Must Have   |
| **Presence Awareness**       | Show who is currently viewing/editing the document                                     | Must Have   |
| **Commenting & Suggestions** | Allow users to add comments and suggest edits                                          | Should Have |
| **Rich Text Formatting**     | Support bold, italic, lists, tables, images, etc.                                      | Should Have |
| **Document Sharing**         | Control permissions (view, comment, edit)                                              | Must Have   |

### Non-Functional Requirements

| Requirement               | Target                          | Rationale                                          |
|---------------------------|---------------------------------|----------------------------------------------------|
| **Low Latency (Editing)** | < 100 ms propagation            | Real-time collaboration requires instant feedback  |
| **Strong Consistency**    | Eventual strong consistency     | All users must converge to the same document state |
| **High Write Throughput** | 1M+ events/sec peak             | Handle hundreds of concurrent edits per document   |
| **High Availability**     | 99.99% uptime                   | Critical for business productivity                 |
| **Durability**            | 99.9999%                        | Document data must never be lost                   |
| **Scalability**           | 100M+ MAU, 100k concurrent docs | Support massive global user base                   |
| **Conflict-Free**         | 100% deterministic resolution   | No data corruption from concurrent edits           |

### Scale Estimation

| Metric                       | Assumption                      | Calculation                                    | Result                        |
|------------------------------|---------------------------------|------------------------------------------------|-------------------------------|
| **Total Users**              | 100 Million MAU                 | -                                              | 100M users                    |
| **Active Users**             | 10% DAU                         | $100\text{M} \times 0.1$                       | 10M daily active users        |
| **Concurrent Editors**       | 5% peak concurrency             | $10\text{M} \times 0.05$                       | 500k concurrent users         |
| **Concurrent Documents**     | 2-10 users per doc (avg 5)      | $500\text{k} / 5$                              | 100k active documents         |
| **Edit Rate Per Doc**        | 10 events/sec per document      | -                                              | 10 events/sec/doc             |
| **Peak Edit Throughput**     | 100k docs × 10 events/sec       | $100\text{k} \times 10$                        | **1 Million events/sec**      |
| **Burst Load**               | 5x normal (peak hours)          | $1\text{M} \times 5$                           | **5 Million events/sec**      |
| **Average Document Size**    | 50 KB per document              | -                                              | 50 KB                         |
| **Total Active Doc Storage** | 100k docs × 50 KB               | $100\text{k} \times 50\text{KB}$               | 5 GB (hot data)               |
| **Version History**          | 100 versions/doc avg, 1 KB/edit | $10\text{B docs} \times 100 \times 1\text{KB}$ | **1 Petabyte** (cold storage) |
| **WebSocket Connections**    | 500k concurrent editors         | -                                              | 500k persistent connections   |
| **Network Bandwidth**        | 1KB per edit × 1M events/sec    | $1\text{KB} \times 1\text{M}$                  | **1 GB/sec** total throughput |

**Key Insights:**

- Peak write load of 1-5M events/sec requires horizontally scalable architecture
- Strong consistency requirement rules out simple eventual consistency models
- 100ms latency constraint demands in-memory processing and efficient conflict resolution
- Version history storage grows to petabyte scale, requiring tiered storage strategy
- WebSocket connection state must be distributed across multiple servers

---

## Key Components

| Component             | Responsibility                                   | Technology          | Scalability                                 |
|-----------------------|--------------------------------------------------|---------------------|---------------------------------------------|
| **WebSocket Servers** | Maintain persistent connections, broadcast edits | Node.js, Go, Elixir | Horizontal (10 nodes, 50k connections each) |
| **OT/CRDT Engine**    | Transform operations, resolve conflicts          | Custom algorithm    | Horizontal (sharded by doc_id)              |
| **Event Store**       | Store all edits as immutable events              | PostgreSQL, Kafka   | Horizontal (partitioned)                    |
| **Snapshot Store**    | Periodic document snapshots for fast loading     | Redis, PostgreSQL   | Horizontal (sharded)                        |
| **Presence Service**  | Track active users, cursors, selections          | Redis               | Horizontal (sharded)                        |
| **Document Service**  | Load documents, apply operations                 | Go, Java            | Horizontal (stateless)                      |
| **Search Index**      | Full-text search for documents                   | Elasticsearch       | Horizontal (sharded)                        |

---

## Conflict Resolution: OT vs CRDT

**Problem:** Two users edit the same document region simultaneously. How to merge without corruption?

**Solution 1: Operational Transformation (OT)**

OT transforms operations against each other to maintain consistency.

**How OT Works:**

1. Server maintains canonical operation sequence
2. Each operation has a context (state version it was created against)
3. When operation arrives late, it's transformed against all intervening operations
4. Transformation rules ensure all clients converge to the same state

**Example:**

```
Initial: "Hello" (state version 0)

User A's operation: insert("X", pos=5) at version 0
Server applies → "HelloX" (version 1)

User B's operation: insert("Y", pos=5) at version 0
Server transforms against User A's operation:
  - User A inserted at pos 5, so shift User B's position: pos 5 → pos 6
Transformed operation: insert("Y", pos=6) at version 1
Server applies → "HelloXY" (version 2)

Final state (all clients): "HelloXY"
```

**OT Trade-offs:**

- ✅ **Mature:** Used by Google Docs, proven at scale
- ✅ **Efficient:** Minimal metadata overhead (just position/length)
- ✅ **Low Bandwidth:** Small operation payloads
- ❌ **Complex:** Requires carefully designed transformation functions, prone to edge case bugs
- ❌ **Server-Centric:** Central server must process all transformations (single point of bottleneck)

**Solution 2: Conflict-Free Replicated Data Types (CRDT)**

CRDT uses special data structures where any two concurrent modifications can be merged automatically without
transformation.

**How CRDT Works:**

1. Each character has a globally unique identifier (e.g., `{siteId: "user_a", clock: 42}`)
2. Characters are stored in a tree or list structure ordered by IDs
3. Deletions mark characters as tombstones (not removed)
4. Merging two states is a simple set union operation

**Example:**

```
Initial: "Hello" → [H(1), e(2), l(3), l(4), o(5)]

User A inserts "X" after "o":
  - Generates ID between o(5) and end: X(5.5)
  - State: [H(1), e(2), l(3), l(4), o(5), X(5.5)]

User B inserts "Y" after "o":
  - Generates ID between o(5) and end: Y(5.7)
  - State: [H(1), e(2), l(3), l(4), o(5), Y(5.7)]

Merge both states (sort by ID):
  - [H(1), e(2), l(3), l(4), o(5), X(5.5), Y(5.7)]
  - Result: "HelloXY"

Both clients converge to: "HelloXY"
```

**CRDT Trade-offs:**

- ✅ **Simple:** No complex transformation logic, merging is commutative
- ✅ **Decentralized:** Clients can merge states peer-to-peer (multi-region support)
- ✅ **Partition Tolerant:** Works offline, merges on reconnect
- ❌ **Memory Overhead:** Each character needs unique ID (3-10× storage vs plain text)
- ❌ **Bandwidth:** Larger payloads due to metadata
- ❌ **Tombstone Accumulation:** Deleted characters remain as tombstones (requires periodic compaction)

**Our Decision: Hybrid OT + CRDT**

| Scenario                             | Algorithm | Reason                                |
|--------------------------------------|-----------|---------------------------------------|
| **Real-time editing (online users)** | OT        | Lower latency, less bandwidth, mature |
| **Offline editing**                  | CRDT      | Partition tolerant, automatic merge   |
| **Multi-region conflict**            | CRDT      | No central authority needed           |
| **Mobile sync (intermittent)**       | CRDT      | Resilient to network failures         |

---

## Event Sourcing and Persistence

**Event Store Schema (PostgreSQL):**

```sql
CREATE TABLE document_events (
    event_id BIGSERIAL PRIMARY KEY,
    doc_id UUID NOT NULL,
    user_id UUID NOT NULL,
    operation_type VARCHAR(20) NOT NULL, -- 'insert', 'delete', 'format'
    position INT NOT NULL,
    content TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_doc_created (doc_id, created_at)
);
```

**Why Event Sourcing?**

- **Complete History:** Every edit is preserved for version control
- **Audit Trail:** Track who changed what and when (compliance requirement)
- **Reproducibility:** Reconstruct document at any point in time
- **Debugging:** Replay events to diagnose issues

**Snapshot Strategy (CQRS):**

To avoid replaying millions of events, we periodically save snapshots:

```sql
CREATE TABLE document_snapshots (
    doc_id UUID PRIMARY KEY,
    snapshot_version BIGINT NOT NULL, -- Last event_id included
    content TEXT NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Snapshot Policy:**

- Save snapshot every 100 operations
- Store in Redis (hot) and PostgreSQL (warm)
- Document load: Apply events since last snapshot (typically < 100 events)

**Storage Tiering:**

- **Hot (Redis):** Last 1000 events per document (< 1 hour old) → 5 GB
- **Warm (PostgreSQL):** Last 30 days of events → 500 GB
- **Cold (S3):** Full history → 1 PB

---

## WebSocket Connection Layer

**Requirements:**

- 500k concurrent WebSocket connections
- < 100ms message propagation
- Automatic reconnection handling
- Efficient broadcast to all document collaborators

**Architecture:**

```
Load Balancer (HAProxy/Nginx)
  ↓ (Sticky sessions by doc_id hash)
┌─────────────────────────────────────┐
│  WebSocket Server Cluster (10 nodes)│
│  Each server: 50k connections       │
│                                     │
│  Connection Pool:                   │
│  - doc_id → [conn1, conn2, ...]     │
│  - user_id → conn                   │
│  - Presence tracking                │
└─────────────────────────────────────┘
  ↓
Redis Pub/Sub (for cross-server broadcast)
```

**Why Sticky Sessions?**

- All collaborators on a document connect to the same WebSocket server
- Enables efficient in-memory broadcast (no network hop for most messages)
- Simplifies presence tracking (cursors, active users)

**Reconnection Strategy:**

- Client sends last known event ID on reconnect
- Server sends missed operations since that ID (from Redis cache or Kafka log)
- Exponential backoff for reconnection attempts

---

## Bottlenecks & Solutions

### Bottleneck 1: OT Processing Latency

**Problem:** OT transformation adds 20-50ms latency per operation.

**Solution:**

- **Optimistic UI Updates:** Apply operation locally immediately, send to server asynchronously
- **Batch Operations:** Client buffers operations for 50ms, sends batch as single message
- **Parallel Processing:** Process multiple documents in parallel (sharding)

**Performance:**

- Before: 100-200ms (blocking OT)
- After: <10ms (optimistic UI) + 20ms (server reconciliation)

### Bottleneck 2: Hot Documents (100+ Editors)

**Problem:** Document with 100+ simultaneous editors creates bottleneck.

**Solution:**

- **Read Replicas for Hot Documents:** Primary shard handles writes (OT processing), read replicas handle broadcasts
- **Coalescing:** Batch updates every 100ms instead of pushing every single edit
- **Differential Updates:** Only push edits if content changed by >1% (filter noise)

**Performance:**

- Before: 1000 operations/sec per document (bottleneck)
- After: 10,000 operations/sec per document (with coalescing)

### Bottleneck 3: Version History Storage (1 PB)

**Problem:** Storing full history for all documents grows to petabyte scale.

**Solution:**

- **Tiered Storage:** Hot (Redis), Warm (PostgreSQL), Cold (S3)
- **Aggressive Compaction:** Remove tombstones older than 30 days
- **Selective Retention:** Only keep full history for important documents, others keep last 100 versions

**Cost:**

- Before: 1 PB × $0.02/GB = $20M/month
- After: 100 TB hot + 400 TB warm + 500 TB cold = $2M/month (10× reduction)

### Bottleneck 4: WebSocket Server Memory

**Problem:** 50k connections × 50 KB/connection = 2.5 GB per server.

**Solution:**

- **Connection Pooling:** Reuse connections for multiple documents
- **Presence Data Compression:** Use efficient encoding (protobuf)
- **Memory Limits:** Evict inactive connections after 5 minutes

---

## Common Anti-Patterns to Avoid

### 1. No Operation Ordering

❌ **Bad:** Processing operations in arrival order without considering logical causality

**Why It's Bad:**

- User A: insert("X", pos=5) at version 1
- User B: insert("Y", pos=6) at version 1
- Server receives B before A (network delay)
- Applies B first → Wrong position, document corrupted

✅ **Good:** Use vector clocks or logical timestamps to order operations by causality, not arrival time. Kafka
partitioning by doc_id guarantees order within a document.

### 2. Storing Full Document on Every Edit

❌ **Bad:** Saving entire document state on every operation

**Why It's Bad:**

- Event 1: Store full doc (50 KB)
- Event 2: Store full doc (50 KB)
- ...
- Event 1000: Store full doc (50 KB)
- Total: 50 MB for 1000 edits!

✅ **Good:** Use event sourcing (store only operations) + periodic snapshots. Reconstructing document: load latest
snapshot + apply operations since snapshot.

### 3. Synchronous OT Processing

❌ **Bad:** Blocking user's edit until OT transformation completes

**Why It's Bad:**

- User types → Send to server → Wait for OT → Wait for DB write → Receive ACK → Show character
- Total latency: 100-200ms (feels laggy)

✅ **Good:** Optimistic UI updates: Apply operation locally immediately, send to server asynchronously. Reconcile if
server returns different transformation. Latency: < 10ms (feels instant).

### 4. Global Lock on Document

❌ **Bad:** Using a distributed lock for every edit serializes all operations

**Why It's Bad:**

- 10 users editing simultaneously
- Each operation acquires lock → processes → releases lock
- Throughput: 1 operation at a time = 10 ops/sec (not 100+ ops/sec)

✅ **Good:** OT/CRDT algorithms allow concurrent operations without locks. Shard by document ID, single-threaded
processing per document (no lock needed). Throughput: 100+ ops/sec per document.

### 5. Keeping Deleted Content Forever

❌ **Bad:** CRDT tombstones accumulate indefinitely

**Why It's Bad:**

- Document after 1M edits (500k deletions):
- Visible content: 10 KB
- Tombstones: 500 KB
- Memory usage 50× larger than actual content!

✅ **Good:** Periodic tombstone garbage collection: Remove tombstones older than 30 days (after all clients have synced).
Use snapshot compaction to clean history.

---

## Monitoring & Observability

### Key Metrics

**System Metrics:**

- **Edit propagation latency:** <100ms p99 (target)
- **OT processing latency:** <50ms p99 (target)
- **WebSocket connections:** 500k concurrent (target)
- **Event throughput:** 1M events/sec (target)
- **Snapshot generation time:** <1 second (target)
- **Cache hit rate:** >80% (target)

**Business Metrics:**

- **Active documents:** 100k concurrent (target)
- **Concurrent editors:** 500k (target)
- **Edit success rate:** >99.9% (target)
- **Offline sync success rate:** >99% (target)

### Dashboards

1. **Real-Time Operations Dashboard**
    - Edit propagation latency (p50, p95, p99)
    - OT processing latency
    - WebSocket connection count (by server)
    - Event throughput (events/sec)

2. **Document Health Dashboard**
    - Active documents (by shard)
    - Hot documents (100+ editors)
    - Document load time (p50, p95, p99)
    - Snapshot generation rate

3. **System Health Dashboard**
    - Redis memory usage (by shard)
    - PostgreSQL connection pool (by shard)
    - Kafka consumer lag (by partition)
    - WebSocket server health (CPU, memory)

### Alerts

**Critical Alerts (PagerDuty, 24/7 on-call):**

- Edit propagation latency >500ms for 5 minutes
- OT processing latency >200ms for 5 minutes
- WebSocket server down (50k users disconnected)
- Event store write failures

**Warning Alerts (Slack, email):**

- Kafka consumer lag >10K messages
- Redis memory usage >90%
- Snapshot generation time >5 seconds
- Document load time >1 second

---

## Trade-offs Summary

| What We Gain                                                | What We Sacrifice                                                       |
|-------------------------------------------------------------|-------------------------------------------------------------------------|
| ✅ **Real-time collaboration** (< 100ms)                     | ❌ **Complexity** (OT/CRDT algorithms are hard to implement correctly)   |
| ✅ **Strong consistency** (all users converge to same state) | ❌ **Latency overhead** (OT transformation adds 20-50ms)                 |
| ✅ **Offline editing** (CRDT-based merge)                    | ❌ **Storage overhead** (CRDT metadata is 3-10× larger)                  |
| ✅ **Complete version history** (event sourcing)             | ❌ **Storage cost** (petabyte-scale for full history)                    |
| ✅ **Scalability** (sharding by document)                    | ❌ **Hot document problem** (single doc with 100+ editors is bottleneck) |
| ✅ **Low latency** (WebSocket + in-memory)                   | ❌ **Infrastructure cost** (stateful WebSocket servers, Redis cache)     |
| ✅ **Fault tolerance** (Kafka + PostgreSQL replication)      | ❌ **Operational complexity** (many moving parts, difficult to debug)    |

---

## Real-World Examples

### Google Docs (OT-Based)

- **Scale:** 2 billion documents, 1 billion MAU, millions of concurrent editors
- **Architecture:** Custom OT algorithm (Jupiter), centralized OT server, WebSocket for real-time communication, event
  sourcing for version history
- **Lessons:** OT requires extensive testing (edge cases are subtle), snapshots critical for performance, offline
  support added later (CRDT-like merge)

### Microsoft Office 365 (Hybrid OT/CRDT)

- **Scale:** 345 million paid seats, 50+ petabytes of document storage
- **Architecture:** OT for real-time collaboration (Word, Excel), CRDT for OneNote (offline-first design), Azure SignalR
  for WebSocket management
- **Lessons:** Different tools need different algorithms, multi-region writes require CRDT or complex conflict
  resolution, caching snapshots in CDN reduces latency by 80%

### Notion (CRDT-Based)

- **Scale:** 30 million users, < 100ms sync latency
- **Architecture:** Pure CRDT (no OT), blocks as fundamental unit (not characters), PostgreSQL for persistence,
  WebSocket for real-time sync
- **Lessons:** Block-based CRDT reduces metadata overhead, CRDT works well for structured documents, tombstone GC
  required every 24 hours

### Figma (CRDT + Operational Transform)

- **Scale:** 4 million users, 50k concurrent editing sessions
- **Architecture:** Hybrid approach (CRDT for offline, OT for real-time), custom multiplayer server (C++ for
  performance), WebAssembly client
- **Lessons:** Design tools need fast rendering (60 FPS) while syncing, conflict resolution for visual elements differs
  from text, presence (cursors, selections) is 80% of multiplayer traffic

---

## Key Takeaways

1. **OT vs CRDT:** OT for real-time (low latency), CRDT for offline (partition tolerance)
2. **Event sourcing** enables complete version history and audit trail
3. **Snapshots** avoid replaying millions of events (load latest snapshot + apply recent events)
4. **Sticky sessions** enable efficient in-memory broadcast (all collaborators on same server)
5. **Optimistic UI** provides instant feedback (<10ms) while server reconciles
6. **Sharding by document ID** enables horizontal scalability (no distributed transactions)
7. **Tiered storage** reduces costs (hot Redis, warm PostgreSQL, cold S3)
8. **Tombstone GC** prevents memory explosion in CRDT (remove tombstones after 30 days)
9. **Operation batching** reduces network overhead (50ms buffer → 20 msg/sec vs 1000 msg/sec)
10. **Coalescing** handles hot documents (batch updates every 100ms)

---

## Recommended Stack

- **WebSocket Servers:** Node.js/Go/Elixir (10 nodes, 50k connections each)
- **OT/CRDT Engine:** Custom algorithm (sharded by doc_id)
- **Event Store:** PostgreSQL (partitioned) + Kafka (event log)
- **Snapshot Store:** Redis (hot) + PostgreSQL (warm)
- **Presence Service:** Redis (sharded by doc_id)
- **Search:** Elasticsearch (full-text search)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, conflict resolution, data synchronization)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (real-time collaboration, OT/CRDT
  processing, offline sync)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (OT vs CRDT, WebSockets, consistency models)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (OT, CRDT, conflict resolution)

