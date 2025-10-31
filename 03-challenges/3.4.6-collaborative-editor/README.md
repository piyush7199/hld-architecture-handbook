# 3.4.6 Design a Collaborative Editor (Google Docs)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, conflict resolution flow, data synchronization
- **[Sequence Diagrams](./sequence-diagrams.md)** - Real-time collaboration flows, OT/CRDT processing, offline sync scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of OT vs CRDT, WebSockets, consistency models
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for OT, CRDT, conflict resolution

---

## Problem Statement

Design a highly scalable, real-time collaborative document editing platform like Google Docs that enables multiple users to edit the same document simultaneously. The system must handle concurrent edits with proper conflict resolution, provide sub-100ms latency for edit propagation, support offline editing with seamless sync, and maintain strong consistency across all clients while handling millions of concurrent users.

The platform must resolve conflicting edits deterministically, preserve document history for version control, and scale to support 100,000+ concurrent documents with 10+ simultaneous editors per document.

---

## Requirements and Scale Estimation

### Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **Real-Time Collaboration** | Multiple users can edit the same document simultaneously with instant updates | Must Have |
| **Conflict Resolution** | Edits from different users on the same section must merge correctly without corruption | Must Have |
| **Persistent Storage** | All edits and document history must be reliably saved | Must Have |
| **Version History** | Users can view and restore previous document versions | Must Have |
| **Offline Editing** | Support editing when disconnected, syncing changes upon reconnection | Must Have |
| **Cursor Tracking** | Display real-time cursor positions of all collaborators | Must Have |
| **Presence Awareness** | Show who is currently viewing/editing the document | Must Have |
| **Commenting & Suggestions** | Allow users to add comments and suggest edits | Should Have |
| **Rich Text Formatting** | Support bold, italic, lists, tables, images, etc. | Should Have |
| **Document Sharing** | Control permissions (view, comment, edit) | Must Have |

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Low Latency (Editing)** | < 100 ms propagation | Real-time collaboration requires instant feedback |
| **Strong Consistency** | Eventual strong consistency | All users must converge to the same document state |
| **High Write Throughput** | 1M+ events/sec peak | Handle hundreds of concurrent edits per document |
| **High Availability** | 99.99% uptime | Critical for business productivity |
| **Durability** | 99.9999% | Document data must never be lost |
| **Scalability** | 100M+ MAU, 100k concurrent docs | Support massive global user base |
| **Conflict-Free** | 100% deterministic resolution | No data corruption from concurrent edits |

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|------------|-------------|--------|
| **Total Users** | 100 Million MAU | - | 100M users |
| **Active Users** | 10% DAU | $100\text{M} \times 0.1$ | 10M daily active users |
| **Concurrent Editors** | 5% peak concurrency | $10\text{M} \times 0.05$ | 500k concurrent users |
| **Concurrent Documents** | 2-10 users per doc (avg 5) | $500\text{k} / 5$ | 100k active documents |
| **Edit Rate Per Doc** | 10 events/sec per document | - | 10 events/sec/doc |
| **Peak Edit Throughput** | 100k docs × 10 events/sec | $100\text{k} \times 10$ | **1 Million events/sec** |
| **Burst Load** | 5x normal (peak hours) | $1\text{M} \times 5$ | **5 Million events/sec** |
| **Average Document Size** | 50 KB per document | - | 50 KB |
| **Total Active Doc Storage** | 100k docs × 50 KB | $100\text{k} \times 50\text{KB}$ | 5 GB (hot data) |
| **Version History** | 100 versions/doc avg, 1 KB/edit | $10\text{B docs} \times 100 \times 1\text{KB}$ | **1 Petabyte** (cold storage) |
| **WebSocket Connections** | 500k concurrent editors | - | 500k persistent connections |
| **Network Bandwidth** | 1KB per edit × 1M events/sec | $1\text{KB} \times 1\text{M}$ | **1 GB/sec** total throughput |

**Key Insights:**
- Peak write load of 1-5M events/sec requires horizontally scalable architecture
- Strong consistency requirement rules out simple eventual consistency models
- 100ms latency constraint demands in-memory processing and efficient conflict resolution
- Version history storage grows to petabyte scale, requiring tiered storage strategy
- WebSocket connection state must be distributed across multiple servers

---

## High-Level Architecture

The system follows a **layered architecture** with WebSocket-based communication, centralized operational transformation (OT) processing, and event sourcing for persistence.

### Core Design Principles

1. **Operational Transformation (OT) or CRDT**: Core algorithm for conflict-free collaborative editing
2. **Event Sourcing**: Store all edits as immutable events for version history and replay
3. **WebSocket-Based Real-Time Communication**: Bi-directional persistent connections for instant updates
4. **Sharding by Document ID**: Partition documents across servers for horizontal scalability
5. **CQRS Pattern**: Separate read and write paths for optimized performance

### System Architecture

The architecture consists of four main layers:

**1. Client Layer**
- Web browsers and mobile apps maintain WebSocket connections
- Local document model for optimistic UI updates
- IndexedDB for offline storage

**2. Connection Layer**
- Load balancer with sticky sessions (route all collaborators to same server)
- WebSocket server cluster (50k connections per server)
- Presence tracking and cursor broadcasting

**3. Processing Layer**
- Kafka event stream (partitioned by document ID for ordering)
- OT worker cluster (sharded by document ID)
- Transformation engine for conflict resolution

**4. Persistence Layer**
- PostgreSQL for event log (append-only, write-optimized)
- Redis for document snapshots (CQRS read model)
- S3 for version history (cold storage)
- PostgreSQL for metadata (users, permissions, comments)

### Data Flow

**Write Path (New Edit):**
1. User types character → Client generates operation: `{type: "insert", pos: 42, char: "a"}`
2. Client sends operation via WebSocket
3. WebSocket Server publishes to Kafka (partitioned by doc_id)
4. OT Worker consumes event, transforms against concurrent operations
5. OT Worker appends transformed operation to PostgreSQL event log
6. OT Worker broadcasts to all connected clients via WebSocket
7. Clients apply operation to local document model

**Read Path (Open Document):**
1. Client requests document via REST API
2. Check Redis cache for latest snapshot
3. If hit → Return snapshot (< 10ms)
4. If miss → Reconstruct from event log (replay since last snapshot)
5. Client subscribes to WebSocket for real-time updates

**Why This Architecture?**
- **Kafka**: Strict ordering per partition ensures deterministic OT processing
- **Sharding**: All edits for a document go to same OT worker (no distributed consensus)
- **Event Sourcing**: Complete history for version control and audit
- **CQRS**: Fast reads without replaying entire history
- **WebSockets**: Only protocol with true bi-directional, low-latency support

---

## Detailed Component Design

### Conflict Resolution: OT vs CRDT

The core challenge is resolving conflicting edits when two users modify the same document simultaneously.

**Problem Example:**
```
Initial: "Hello"
User A: insert("X", pos=5) → "HelloX"
User B: insert("Y", pos=5) → "HelloY"
Without resolution: Ambiguous final state!
```

**Solution 1: Operational Transformation (OT)**

OT transforms operations against each other to maintain consistency.

**How OT Works:**
- Server maintains canonical operation sequence
- Each operation has context (state version)
- Late operations are transformed against intervening operations
- Transformation rules ensure convergence

**Example:**
```
Initial: "Hello" (version 0)
User A: insert("X", pos=5) at v0 → Server applies → "HelloX" (v1)
User B: insert("Y", pos=5) at v0 → Server transforms: pos 5 → 6
Transformed: insert("Y", pos=6) at v1 → "HelloXY" (v2)
```

**OT Trade-offs:**
- ✅ Mature (Google Docs uses this)
- ✅ Efficient (minimal metadata)
- ✅ Low bandwidth
- ❌ Complex (hard to implement correctly)
- ❌ Server-centric (bottleneck risk)

*See `pseudocode.md::transform()` for detailed implementation.*

**Solution 2: CRDT (Conflict-Free Replicated Data Types)**

CRDT uses special data structures where concurrent modifications merge automatically.

**How CRDT Works:**
- Each character has globally unique ID (`{siteId, clock}`)
- Characters stored in ordered structure
- Deletions use tombstones
- Merging is set union (commutative)

**Example:**
```
Initial: "Hello" → [H(1), e(2), l(3), l(4), o(5)]
User A: insert "X" → X(5.5)
User B: insert "Y" → Y(5.7)
Merge (sort by ID): [H(1), e(2), l(3), l(4), o(5), X(5.5), Y(5.7)]
Result: "HelloXY"
```

**CRDT Trade-offs:**
- ✅ Simple (no transformation logic)
- ✅ Decentralized (peer-to-peer)
- ✅ Partition tolerant (works offline)
- ❌ Memory overhead (3-10x storage)
- ❌ Higher bandwidth (metadata)
- ❌ Tombstone accumulation

**Our Decision: Hybrid OT + CRDT**

| Scenario | Algorithm | Reason |
|----------|-----------|--------|
| **Real-time editing** | OT | Lower latency, efficient |
| **Offline editing** | CRDT | Partition tolerant |
| **Multi-region conflict** | CRDT | No central authority |
| **Mobile sync** | CRDT | Network resilient |

*See `this-over-that.md` for detailed OT vs CRDT analysis.*

---

### WebSocket Connection Layer

**Requirements:**
- 500k concurrent connections
- < 100ms propagation
- Automatic reconnection
- Efficient broadcast

**Architecture:**
- Load balancer with sticky sessions (by doc_id hash)
- WebSocket server cluster (10 nodes, 50k connections each)
- Connection pool: `doc_id → [conn1, conn2, ...]`
- Redis Pub/Sub for cross-server broadcast

**Reconnection Strategy:**
- Client sends last known event ID on reconnect
- Server replays missed operations
- Exponential backoff for retries

*See `pseudocode.md::WebSocketServer` for implementation.*

---

### Event Sourcing and Persistence

**Event Store Schema:**
```sql
CREATE TABLE document_events (
    event_id BIGSERIAL PRIMARY KEY,
    doc_id UUID NOT NULL,
    user_id UUID NOT NULL,
    operation_type VARCHAR(20), -- 'insert', 'delete', 'format'
    position INT NOT NULL,
    content TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_doc_created (doc_id, created_at)
);
```

**Why Event Sourcing?**
- Complete history for version control
- Audit trail (compliance)
- Reproducibility (reconstruct any state)
- Debugging (replay events)

**Snapshot Strategy (CQRS):**
- Save snapshot every 100 operations
- Store in Redis (hot) and PostgreSQL (warm)
- Document load: Apply events since last snapshot

**Storage Tiering:**
- Hot (Redis): Last 1000 events (< 1 hour) → 5 GB
- Warm (PostgreSQL): Last 30 days → 500 GB
- Cold (S3): Full history → 1 PB

*See `pseudocode.md::EventStore` for implementation.*

---

### Document State Representation

**Operation Types:**

| Type | Fields | Example |
|------|--------|---------|
| **INSERT** | position, content | `{type: "insert", pos: 5, content: "hello"}` |
| **DELETE** | position, length | `{type: "delete", pos: 10, length: 5}` |
| **FORMAT** | range, attribute | `{type: "format", range: [0, 10], attr: "bold"}` |
| **CURSOR** | user_id, position | `{type: "cursor", user: "alice", pos: 42}` |

**Client-Side Document Model:**
```
class Document:
    content: Array<Character>
    operations: Queue<Operation>  // Pending ops
    version: int                  // Last server version
    cursors: Map<UserId, Position>
```

**Optimistic UI:**
- Apply operation locally immediately
- Send to server with current version
- On ACK, increment version
- On conflict, reconcile with server's transformation

*See `pseudocode.md::ClientDocument` for implementation.*

---

### Offline Editing and Synchronization

**Challenge:** User edits offline, how to merge on reconnect?

**Solution: CRDT-Based Merge**

1. **Offline Editing:**
   - Client maintains local CRDT state
   - Operations generate unique IDs
   - Saved in IndexedDB

2. **Reconnection:**
   - Client sends last server version + local operations
   - Server merges CRDT states
   - Server returns missing operations
   - Client reconciles

**Conflict Example:**
```
Server: "Hello World"
Client A (offline): "Hello Beautiful World"
Client B (online): "Hello Wonderful World"

A reconnects:
  CRDT merge: "Hello Beautiful Wonderful World"
  (Both edits preserved, deterministic ordering)
```

**Why This Works:**
- CRDT guarantees convergence
- No data loss
- Deterministic result

*See `pseudocode.md::mergeOfflineEdits()` for implementation.*

---

### Presence and Cursor Tracking

**Cursor Broadcasts:**
- Client sends cursor position every 200ms (throttled)
- Server broadcasts to collaborators
- No persistence (ephemeral)

**Presence Tracking:**
```
Redis: doc:{doc_id}:presence → Set{user1, user2}
Expiry: 10 seconds (heartbeat required)
```

- Heartbeat every 5 seconds
- Query: `SMEMBERS doc:{doc_id}:presence`

**Optimization:**
- Cursors in WebSocket server memory (not Redis)
- Only broadcast to same-document collaborators
- Batch cursor updates (max 10/sec per user)

*See `pseudocode.md::PresenceManager` for implementation.*

---

### Rich Text Formatting

**Attributed String Model:**
```
Character: {
    char: 'H',
    attributes: {bold: true, color: '#FF0000'},
    id: {siteId: 'user_a', clock: 42}
}
```

**Formatting Operations:**
```
{type: 'format', range: [10, 20], attribute: 'bold', value: true}
```

**Conflict Resolution:**
- Formatting operations are commutative
- Last-write-wins with Lamport timestamps

**Optimization: Run-Length Encoding**
```
[
    {content: "Hello", attrs: {bold: true}, start: 0},
    {content: " World", attrs: {}, start: 5}
]
```
Reduces memory by 10-50x.

---

## Scalability and Performance Optimizations

### Sharding Strategy

**By Document ID:**
```
doc_id → hash(doc_id) % 10 → Shard ID
```

**Shard Contains:**
- OT Worker
- Kafka Partition
- PostgreSQL Partition
- Redis Cache Partition

**Benefits:**
- Linear scalability
- No distributed transactions
- Load balancing

**Hot Documents:**
- Use read replicas for broadcast
- Primary handles writes, replicas handle WebSocket distribution

---

### Caching Strategy

**Multi-Layer Cache:**

1. **Client-Side:** Full document in memory, IndexedDB for offline
2. **CDN/Edge:** Cached snapshots for public docs (60s TTL)
3. **Redis (Server):** Snapshots, recent operations, presence (1 hour TTL)

**Cache Invalidation:**
- Write-through for snapshots
- Lazy expiry for operations
- Heartbeat expiry for presence

---

### Network Optimization

**Compression:**
- Gzip for large ops (> 1 KB)
- Binary protocol for small ops
- 60-80% bandwidth reduction

**Operation Batching:**
- Buffer operations for 50ms
- Send as single message
- Reduces overhead 50x (1000 msg/sec → 20 msg/sec)

**Delta Sync:**
- Send only changed regions
- Client tracks dirty ranges
- 98% payload reduction (50 KB → 1 KB)

---

### Database Optimization

**PostgreSQL Tuning:**
```sql
-- Partitioning by doc_id
CREATE TABLE document_events (...) PARTITION BY HASH (doc_id);

-- Indexes
CREATE INDEX idx_doc_recent ON document_events(doc_id, created_at DESC);
```

**Write Optimization:**
- Batch inserts (100 events per txn)
- Append-only table
- Async replication

**Read Optimization:**
- Read replicas for history queries
- Connection pooling (PgBouncer)
- Prepared statements

---

### Handling Burst Traffic

**Auto-Scaling:**
- Kubernetes HPA for WebSocket servers
- Scale based on connections/CPU
- Add pods in < 30 seconds

**Rate Limiting:**
- Per-user: 100 ops/sec
- Per-document: 1000 ops/sec
- Graceful degradation (queue excess ops)

**Circuit Breaker:**
- If latency > 500ms, switch to eventual consistency
- Queue operations, process when load decreases

---

## Fault Tolerance and Disaster Recovery

### Handling Server Failures

**WebSocket Server Crash:**
- Client detects disconnection (heartbeat timeout)
- Reconnects to different server
- Replays missed operations
- User experience: 1-2 second freeze

**OT Worker Crash:**
- Kafka consumer rebalances
- Another worker takes over
- Reprocesses from checkpoint (idempotent)

**Database Failure:**
- PostgreSQL replication (primary + 2 replicas)
- Auto failover (< 30 seconds)
- Writes buffered in Kafka during failover

---

### Data Durability Guarantees

**Event Log Durability:**
- Kafka replication factor: 3
- PostgreSQL synchronous replication
- S3 versioning: 11 nines durability

**Preventing Data Loss:**
- Client ACK before marking sent
- Kafka at-least-once delivery
- Idempotent operations

**Backup Strategy:**
- Hourly snapshots to S3
- Daily event log backups
- 30-day retention

---

### Disaster Recovery

**Multi-Region:**
- Primary region handles writes
- Secondary region as read replica (1-2 sec lag)
- Manual/auto failover on disaster

**RTO: 5 minutes**  
**RPO: 1 minute**

---

## Security and Privacy

### Authentication and Authorization

**Authentication:**
- JWT tokens for WebSocket
- OAuth 2.0 for login
- Token refresh every hour

**Authorization:**
```sql
CREATE TABLE document_permissions (
    doc_id UUID,
    user_id UUID,
    role VARCHAR(20), -- 'owner', 'editor', 'commenter', 'viewer'
    PRIMARY KEY (doc_id, user_id)
);
```

**Permission Checks:**
- On connect: Verify access
- On operation: Verify editor role
- On share: Only owner can change permissions

---

### Data Encryption

**At Rest:**
- PostgreSQL TDE
- S3 SSE-S3
- Redis encrypted persistence

**In Transit:**
- TLS 1.3 for WebSocket
- TLS for database connections

---

### Rate Limiting

**Per-User:**
- 100 ops/sec
- 10 doc opens/min
- 50 comments/hour

**Per-Document:**
- 1000 ops/sec total
- 50 simultaneous editors

**Abuse Detection:**
- Flag excessive edit rate
- Flag unusual patterns
- Quarantine suspected malicious docs

---

## Monitoring and Observability

### Key Metrics

| Metric | Target | Alert |
|--------|--------|-------|
| **Edit Latency** | < 100ms (p99) | > 200ms |
| **WebSocket Connections** | 500k | > 600k |
| **OT Processing** | < 50ms (p99) | > 100ms |
| **Throughput** | 1M events/sec | < 500k |
| **Kafka Lag** | < 1 second | > 5 seconds |
| **DB Query Latency** | < 10ms (p99) | > 50ms |
| **Cache Hit Rate** | > 95% | < 90% |
| **Document Load** | < 500ms (p99) | > 1 second |
| **Conflict Rate** | < 1% | > 5% |
| **Reconnection Rate** | < 5/min | > 50/min |

---

### Distributed Tracing

**Trace Example:**
```
Total: 85ms
  Client → WebSocket: 10ms
  WebSocket → Kafka: 5ms
  Kafka → OT: 15ms
  OT Processing: 30ms
  PostgreSQL Write: 10ms
  Broadcast: 15ms
```

---

### Logging

**Structured Logs (JSON):**
```json
{
  "timestamp": "2025-10-31T12:34:56Z",
  "level": "INFO",
  "service": "ot-worker",
  "doc_id": "abc123",
  "operation": "insert",
  "latency_ms": 45,
  "trace_id": "xyz789"
}
```

**Log Aggregation:**
- ELK Stack / DataDog
- Query by doc_id, user_id, trace_id
- 30-day hot, 1-year cold

---

### Alerting

**Critical (Page):**
- Edit latency > 500ms for 5 min
- OT worker down > 1 min
- DB primary down
- Kafka lag > 10 sec

**Warning (Slack):**
- Cache hit < 90%
- Conflict rate > 5%
- Reconnection spike

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: No Operation Ordering

**Problem:** Processing by arrival order ignores causality, corrupts document.

**Solution:** ✅ Use vector clocks or Kafka partitioning for causal ordering.

---

### ❌ Anti-Pattern 2: Storing Full Document on Every Edit

**Problem:** Wastes storage (50 MB for 1000 edits of 50 KB doc).

**Solution:** ✅ Event sourcing (store operations) + periodic snapshots.

---

### ❌ Anti-Pattern 3: Synchronous OT Processing

**Problem:** Blocking on OT increases latency (100-200ms feels laggy).

**Solution:** ✅ Optimistic UI updates (< 10ms perceived latency).

---

### ❌ Anti-Pattern 4: Global Lock on Document

**Problem:** Serializes all operations, kills throughput (1 op at a time).

**Solution:** ✅ OT/CRDT allow concurrent ops without locks.

---

### ❌ Anti-Pattern 5: Keeping Deleted Content Forever

**Problem:** CRDT tombstones grow indefinitely (500 KB tombstones for 10 KB doc).

**Solution:** ✅ Periodic tombstone GC (remove tombstones > 30 days old).

---

### ❌ Anti-Pattern 6: Broadcasting All Operations to All Users

**Problem:** Every cursor movement broadcasted (100k msgs/sec per user).

**Solution:** ✅ Throttle cursor updates (10/sec), batch formatting ops.

---

### ❌ Anti-Pattern 7: No Conflict Metrics

**Problem:** OT bugs undetected, users see corrupted docs.

**Solution:** ✅ Track conflict rate metric, alert if > 5%.

---

## Alternative Approaches

### Alternative 1: Pure CRDT (No OT)

**Pros:**
- Simpler (no transformation logic)
- Decentralized (peer-to-peer)
- Partition tolerant

**Cons:**
- 3-10x bandwidth overhead
- 2-5x memory overhead
- Tombstone accumulation

**When to Use:** Offline-first apps, peer-to-peer, multi-region writes.

**Why Not Chosen:** OT is more efficient for real-time online collaboration (95% use case).

---

### Alternative 2: Lock-Based Editing

**Pros:**
- No conflict resolution
- Simple implementation
- Serializable

**Cons:**
- Poor UX (blocked by others)
- Lock contention
- Deadlock risk

**When to Use:** Financial docs, legal contracts (strict consistency, low concurrency).

**Why Not Chosen:** Restrictive for multi-user collaboration.

---

### Alternative 3: Last-Write-Wins (LWW)

**Pros:**
- Extremely simple
- Low latency
- Scales easily

**Cons:**
- Data loss (overwrites)
- Poor UX
- Clock sync required

**When to Use:** Key-value stores, user preferences (non-critical data).

**Why Not Chosen:** Unacceptable for documents (users expect every keystroke preserved).

---

### Alternative 4: Diff-Based Synchronization

**Pros:**
- Mature (Git merge)
- Works for code files
- Human review of conflicts

**Cons:**
- Not real-time
- Manual conflict resolution
- Slow for large docs

**When to Use:** Version control, wikis, code reviews (asynchronous collaboration).

**Why Not Chosen:** Requires real-time automatic conflict resolution.

---

## Real-World Examples

### Google Docs (OT-Based)

**Architecture:**
- Custom OT algorithm (10+ years development)
- Jupiter algorithm
- WebSocket communication
- Event sourcing

**Scale:**
- 2 billion documents
- 1 billion MAU

**Lessons:**
- OT requires extensive testing
- Snapshots critical for performance
- Offline support added later (CRDT-like merge)

---

### Microsoft Office 365 (Hybrid)

**Architecture:**
- OT for real-time (Word, Excel)
- CRDT for OneNote (offline-first)
- Azure SignalR for WebSocket
- Blob storage for history

**Scale:**
- 345 million paid seats
- 50+ PB storage

**Lessons:**
- Different tools need different algorithms
- Multi-region writes need CRDT
- CDN caching reduces latency 80%

---

### Notion (CRDT-Based)

**Architecture:**
- Pure CRDT
- Block-based (not character-based)
- PostgreSQL persistence
- WebSocket sync

**Scale:**
- 30 million users
- < 100ms sync latency

**Lessons:**
- Block-based CRDT reduces overhead
- Works well for structured docs
- Tombstone GC every 24 hours

---

### Figma (CRDT + OT)

**Architecture:**
- Hybrid (CRDT offline, OT real-time)
- Custom C++ multiplayer server
- WebAssembly client

**Scale:**
- 4 million users
- 50k concurrent sessions

**Lessons:**
- Design tools need 60 FPS rendering
- Visual conflict resolution differs from text
- Presence is 80% of multiplayer traffic

---

## Cost Analysis

### Monthly Infrastructure Costs (100k Concurrent Documents)

| Component | Specification | Cost |
|-----------|---------------|------|
| **WebSocket Servers** | 20 × c5.2xlarge | $6,800 |
| **OT Workers** | 10 × c5.4xlarge | $6,800 |
| **Kafka Cluster** | 10 × m5.xlarge | $3,400 |
| **PostgreSQL** | db.r5.4xlarge + 2 replicas | $8,500 |
| **Redis Cluster** | 10 × cache.r5.xlarge | $3,400 |
| **S3 Storage** | 1 PB (history) | $23,000 |
| **Data Transfer** | 1 TB/day | $2,700 |
| **Load Balancers** | ALB + NLB | $500 |
| **Monitoring** | DataDog/Grafana | $2,000 |
| **Total** | | **~$57,000/month** |

**Cost Per User:** $0.57/month (500k active users)

**Optimization:**
- Spot instances for OT workers (save 70%)
- S3 compression (save 60%)
- Reserved instances (save 40%)
- **Optimized: ~$25,000/month ($0.25/user)**

---

## Future Enhancements

### End-to-End Encryption

**Challenge:** OT server needs plaintext to transform.

**Solution:**
- Client-side encryption with shared key
- Server operates on encrypted CRDT
- Trade-off: No server-side search

---

### AI-Powered Collaboration

**Features:**
- Smart suggestions (grammar, tone)
- Auto-complete
- Summarization

**Implementation:**
- ML models on snapshots (not real-time)
- Ghost text suggestions
- Opt-in for privacy

---

### Voice/Video Integration

**Use Case:** Collaborative editing during video call.

**Architecture:**
- WebRTC for peer-to-peer video
- Document edits via central server
- Cursor synced with voice

---

### Mobile Optimization

**Challenges:**
- Limited bandwidth
- Intermittent connectivity
- Battery constraints

**Solutions:**
- Aggressive batching (5 sec intervals)
- Delta sync (only changed regions)
- CRDT offline editing

---

## Interview Discussion Points

### Key Topics

1. **Conflict Resolution:**
   - OT vs CRDT trade-offs
   - Transformation examples
   - Convergence guarantees

2. **Scalability:**
   - Sharding by document ID
   - Hot documents (100+ editors)
   - Multi-region challenges

3. **Latency Optimizations:**
   - WebSocket vs alternatives
   - Optimistic UI
   - Message batching

4. **Consistency:**
   - Event sourcing
   - Snapshot strategy (CQRS)
   - Network partitions

5. **Failure Scenarios:**
   - Disconnection handling
   - Worker crash (Kafka rebalancing)
   - Database failover

### Follow-Up Questions

**Q: How do you handle offline editing for hours?**  
A: CRDT-based merge, local storage, replay on reconnect.

**Q: What if two users delete and insert at same position?**  
A: OT transformation rules, deterministic ordering by user ID.

**Q: How to prevent malicious edits?**  
A: Server validation, rate limiting, permissions, version rollback.

**Q: Can you scale to 1000 simultaneous editors on one doc?**  
A: Challenges with broadcast fanout (1000² messages). Use hierarchical broadcast, read replicas, or impose soft limit (50-100).

---

## Trade-offs Summary

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Real-time collaboration (< 100ms) | ❌ Complexity (OT/CRDT hard to implement) |
| ✅ Strong consistency (convergence) | ❌ Latency overhead (20-50ms for OT) |
| ✅ Offline editing (CRDT merge) | ❌ Storage overhead (3-10x for CRDT) |
| ✅ Complete version history | ❌ Storage cost (petabyte scale) |
| ✅ Scalability (sharding) | ❌ Hot document problem (100+ editors) |
| ✅ Low latency (WebSocket + in-memory) | ❌ Infrastructure cost (stateful servers) |
| ✅ Fault tolerance (replication) | ❌ Operational complexity (many components) |

---

## References

### Related Chapters

- **[2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)** - WebSocket architecture
- **[2.1.7 PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.7-postgresql-deep-dive.md)** - Event sourcing
- **[2.1.11 Redis Deep Dive](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)** - Caching strategy
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Event streaming
- **[2.3.4 Distributed Transactions](../../02-components/2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md)** - Idempotency
- **[3.3.1 Live Chat System](../3.3.1-live-chat-system/3.3.1-design-live-chat-system.md)** - WebSocket patterns
- **[1.1.4 Data Consistency Models](../../01-principles/1.1.4-data-consistency-models.md)** - Consistency models

### External Resources

**Operational Transformation:**
- [Google Wave OT Paper](https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html)
- [Jupiter Collaboration System](https://dl.acm.org/doi/10.1145/215585.215706)

**CRDT:**
- [CRDT Primer](https://crdt.tech/)
- [Yjs CRDT Framework](https://github.com/yjs/yjs)
- [Automerge Library](https://github.com/automerge/automerge)

**Real-World:**
- [Figma's Multiplayer](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)
- [Notion's Data Model](https://www.notion.so/blog/data-model-behind-notion)
