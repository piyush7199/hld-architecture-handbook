# 3.4.6 Design a Collaborative Editor (Google Docs)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, conflict resolution flow, data
  synchronization
- **[Sequence Diagrams](./sequence-diagrams.md)** - Real-time collaboration flows, OT/CRDT processing, offline sync
  scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of OT vs CRDT, WebSockets,
  consistency models
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for OT, CRDT, conflict
  resolution

---

## 1. Problem Statement

Design a highly scalable, real-time collaborative document editing platform like Google Docs that enables multiple users
to edit the same document simultaneously. The system must handle concurrent edits with proper conflict resolution,
provide sub-100ms latency for edit propagation, support offline editing with seamless sync, and maintain strong
consistency across all clients while handling millions of concurrent users.

The platform must resolve conflicting edits deterministically, preserve document history for version control, and scale
to support 100,000+ concurrent documents with 10+ simultaneous editors per document.

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

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

### Non-Functional Requirements (NFRs)

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

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### Core Design Principles

1. **Operational Transformation (OT) or CRDT**: Core algorithm for conflict-free collaborative editing
2. **Event Sourcing**: Store all edits as immutable events for version history and replay
3. **WebSocket-Based Real-Time Communication**: Bi-directional persistent connections for instant updates
4. **Sharding by Document ID**: Partition documents across servers for horizontal scalability
5. **CQRS Pattern**: Separate read and write paths for optimized performance

### System Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                    │
│  │   Browser    │   │   Browser    │   │  Mobile App  │                    │
│  │   Client 1   │   │   Client 2   │   │   Client 3   │                    │
│  │  (WebSocket) │   │  (WebSocket) │   │  (WebSocket) │                    │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                    │
│         │                  │                   │                             │
│         └──────────────────┼───────────────────┘                             │
│                            │                                                 │
└────────────────────────────┼─────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CONNECTION LAYER                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Load Balancer (Layer 7)                             │  │
│  │              (WebSocket-aware, sticky sessions)                        │  │
│  └───────────────────────┬───────────────────────────────────────────────┘  │
│                          │                                                   │
│         ┌────────────────┼────────────────┐                                 │
│         ▼                ▼                ▼                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                         │
│  │ WebSocket   │  │ WebSocket   │  │ WebSocket   │                         │
│  │  Server 1   │  │  Server 2   │  │  Server N   │                         │
│  │             │  │             │  │             │                         │
│  │ (Stateful)  │  │ (Stateful)  │  │ (Stateful)  │                         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                         │
│         │                │                │                                 │
└─────────┼────────────────┼────────────────┼─────────────────────────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     PROCESSING LAYER                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                   Kafka Event Stream                                   │  │
│  │              (Partitioned by Document ID)                              │  │
│  │  Topic: document-edits                                                 │  │
│  └─────────────────────────┬─────────────────────────────────────────────┘  │
│                            │                                                 │
│                            ▼                                                 │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │          Operational Transformation (OT) Service Cluster               │  │
│  │                  (Sharded by Document ID)                              │  │
│  │                                                                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │  │
│  │  │ OT Worker 1  │  │ OT Worker 2  │  │ OT Worker N  │                │  │
│  │  │ (Docs 1-10k) │  │(Docs 10k-20k)│  │(Docs 90k-100k)│               │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │  │
│  └─────────┼──────────────────┼──────────────────┼───────────────────────┘  │
│            │                  │                  │                           │
└────────────┼──────────────────┼──────────────────┼───────────────────────────┘
             │                  │                  │
             └──────────────────┼──────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PERSISTENCE LAYER                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Event Store (Event Sourcing)                      │   │
│  │                   PostgreSQL (Write-Optimized)                        │   │
│  │          Table: document_events (append-only log)                     │   │
│  │   Columns: event_id, doc_id, user_id, operation, position, content   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                Document Snapshot Store (CQRS Read Model)              │   │
│  │                      Redis (In-Memory Cache)                          │   │
│  │         Key: doc:{doc_id} → Full document state (50KB avg)           │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Version History (Cold Storage)                     │   │
│  │                         S3 / Object Store                             │   │
│  │           Path: s3://docs/{doc_id}/versions/{version_id}              │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Metadata Database                                │   │
│  │                      PostgreSQL (ACID)                                │   │
│  │       Tables: documents, users, permissions, comments                 │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

**Write Path (New Edit):**

1. User types character → Client generates operation: `{type: "insert", pos: 42, char: "a"}`
2. Client sends operation via WebSocket → WebSocket Server
3. WebSocket Server publishes to Kafka → Topic: `document-edits`, partition by `doc_id`
4. OT Worker (assigned to that `doc_id`) consumes event from Kafka
5. OT Worker transforms operation against concurrent operations (see `pseudocode.md::transform()`)
6. OT Worker appends transformed operation to PostgreSQL event log
7. OT Worker broadcasts transformed operation to all connected WebSocket servers
8. WebSocket Servers push operation to all clients editing that document
9. Clients apply operation to local document model

**Read Path (Open Document):**

1. Client requests document via REST API
2. API Gateway checks Redis cache for latest snapshot
3. If cache hit → Return document state (< 10ms)
4. If cache miss → Reconstruct from event log (replay last 100 events from snapshot) → Cache result
5. Client receives document state and subscribes to WebSocket for real-time updates

**Why This Architecture?**

- **Kafka as Event Log**: Provides strict ordering guarantee per partition (by doc_id), ensuring all OT workers see
  edits in the same order, critical for deterministic conflict resolution.
- **Sharding by Document ID**: All edits for a document go to the same OT worker, avoiding distributed consensus
  overhead.
- **Event Sourcing**: Complete history for version control, audit, and document reconstruction.
- **CQRS (Redis Snapshots)**: Fast reads without replaying entire history; snapshots updated incrementally.
- **WebSockets**: Only protocol that supports true bi-directional, low-latency, persistent connections.

---

## 4. Detailed Component Design

### 4.1 Conflict Resolution Engine: OT vs CRDT

The hardest part of collaborative editing is resolving conflicting edits when two users modify the same document region
simultaneously.

**Problem Example:**

- Initial document: `"Hello"`
- User A inserts `"X"` at position 5: `"HelloX"` (operation: `{type: "insert", pos: 5, char: "X"}`)
- User B (simultaneously) inserts `"Y"` at position 5: `"HelloY"` (operation: `{type: "insert", pos: 5, char: "Y"}`)
- Without conflict resolution, final state is ambiguous: `"HelloXY"` or `"HelloYX"`?

**Solution 1: Operational Transformation (OT)**

OT transforms operations against each other to maintain consistency. When User B's operation arrives at the server, it's
transformed against User A's already-applied operation.

*See `pseudocode.md::transform()` for detailed OT implementation.*

**How OT Works:**

1. Server maintains canonical operation sequence
2. Each operation has a context (state version it was created against)
3. When operation arrives late, it's transformed against all intervening operations
4. Transformation rules ensure all clients converge to the same state

**Example Transformation:**

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

- ✅ **Mature**: Used by Google Docs, proven at scale
- ✅ **Efficient**: Minimal metadata overhead (just position/length)
- ✅ **Low Bandwidth**: Small operation payloads
- ❌ **Complex**: Requires carefully designed transformation functions, prone to edge case bugs
- ❌ **Server-Centric**: Central server must process all transformations (single point of bottleneck)

**Solution 2: Conflict-Free Replicated Data Types (CRDT)**

CRDT uses special data structures where any two concurrent modifications can be merged automatically without
transformation.

*See `pseudocode.md::CRDTDocument` for detailed CRDT implementation.*

**How CRDT Works:**

1. Each character has a globally unique identifier (e.g., `{siteId: "user_a", clock: 42}`)
2. Characters are stored in a tree or list structure ordered by IDs
3. Deletions mark characters as tombstones (not removed)
4. Merging two states is a simple set union operation

**Example CRDT Insertion:**

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

- ✅ **Simple**: No complex transformation logic, merging is commutative
- ✅ **Decentralized**: Clients can merge states peer-to-peer (multi-region support)
- ✅ **Partition Tolerant**: Works offline, merges on reconnect
- ❌ **Memory Overhead**: Each character needs unique ID (3-10x storage vs plain text)
- ❌ **Bandwidth**: Larger payloads due to metadata
- ❌ **Tombstone Accumulation**: Deleted characters remain as tombstones (requires periodic compaction)

**Our Decision: Hybrid OT + CRDT**

For Google Docs-scale system, we use **OT for real-time collaboration** (low latency, efficient) and **CRDT for offline
sync and multi-region** (partition tolerance).

| Scenario                             | Algorithm | Reason                                |
|--------------------------------------|-----------|---------------------------------------|
| **Real-time editing (online users)** | OT        | Lower latency, less bandwidth, mature |
| **Offline editing**                  | CRDT      | Partition tolerant, automatic merge   |
| **Multi-region conflict**            | CRDT      | No central authority needed           |
| **Mobile sync (intermittent)**       | CRDT      | Resilient to network failures         |

*See `this-over-that.md` for detailed analysis of OT vs CRDT trade-offs.*

---

### 4.2 WebSocket Connection Layer

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

*See `pseudocode.md::WebSocketServer` for implementation details.*

---

### 4.3 Event Sourcing and Persistence

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

- **Complete History**: Every edit is preserved for version control
- **Audit Trail**: Track who changed what and when (compliance requirement)
- **Reproducibility**: Reconstruct document at any point in time
- **Debugging**: Replay events to diagnose issues

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

- **Hot (Redis)**: Last 1000 events per document (< 1 hour old) → 5 GB
- **Warm (PostgreSQL)**: Last 30 days of events → 500 GB
- **Cold (S3)**: Full history → 1 PB

*See `pseudocode.md::EventStore` for detailed implementation.*

---

### 4.4 Document State Representation

Each document is represented as a sequence of operations applied to an initial empty state:

```
Document State = apply(apply(apply(EMPTY, op1), op2), op3)
```

**Operation Types:**

| Type       | Fields                | Example                                          |
|------------|-----------------------|--------------------------------------------------|
| **INSERT** | `position`, `content` | `{type: "insert", pos: 5, content: "hello"}`     |
| **DELETE** | `position`, `length`  | `{type: "delete", pos: 10, length: 5}`           |
| **FORMAT** | `range`, `attribute`  | `{type: "format", range: [0, 10], attr: "bold"}` |
| **CURSOR** | `user_id`, `position` | `{type: "cursor", user: "alice", pos: 42}`       |

**Client-Side Document Model:**

Clients maintain an in-memory representation optimized for fast local edits:

```
class Document:
    content: Array<Character>
    operations: Queue<Operation>  // Local pending operations
    version: int                  // Last acknowledged server version
    cursors: Map<UserId, Position>
```

When user types:

1. Apply operation to local `content` immediately (optimistic UI)
2. Send operation to server with current `version`
3. On acknowledgment, increment `version` and remove from pending queue
4. On conflict, receive transformed operation from server and reconcile

*See `pseudocode.md::ClientDocument` for full implementation.*

---

### 4.5 Offline Editing and Synchronization

**Challenge**: User edits document offline, how to merge changes when reconnected?

**Solution: CRDT-Based Merge**

1. **Offline Editing**:
    - Client maintains local CRDT state
    - All operations generate unique IDs (site ID + logical clock)
    - Operations saved in IndexedDB (browser local storage)

2. **Reconnection**:
    - Client sends last known server version and all local CRDT operations
    - Server merges client CRDT state with server state
    - Server sends back missing operations
    - Client applies and reconciles

**Conflict Example:**

```
Server state:  "Hello World"
Client A (offline) edits: "Hello Beautiful World"
Client B (online) edits: "Hello Wonderful World"

When A reconnects:
  - Server has: "Hello Wonderful World"
  - Client A has: insert("Beautiful ", pos=6)
  
CRDT merge:
  - Both operations have unique IDs
  - Sort by ID: "Beautiful"(6, site_A, clock_1) vs "Wonderful"(6, site_B, clock_2)
  - Assuming site_A < site_B lexicographically
  - Final: "Hello Beautiful Wonderful World"
```

**Why This Works:**

- CRDT guarantees convergence (all clients eventually see same state)
- No data loss (both edits preserved)
- Deterministic (same merge result regardless of order)

*See `pseudocode.md::mergeOfflineEdits()` for implementation.*

---

### 4.6 Presence and Cursor Tracking

**Requirements:**

- Show real-time cursor positions of all collaborators
- Display active users (online presence)
- Highlight selected text ranges

**Implementation:**

**Cursor Position Broadcasts:**

- Client sends cursor updates every 200ms (throttled)
- Server broadcasts to all collaborators via WebSocket
- No persistence (ephemeral data)

**Presence Tracking:**

```
Redis Key: doc:{doc_id}:presence
Value: Set{user1, user2, user3}
Expiry: 10 seconds (heartbeat required)
```

- Client sends heartbeat every 5 seconds
- Server refreshes Redis expiry
- Presence query: `SMEMBERS doc:{doc_id}:presence`

**Optimization:**

- Cursor positions stored in WebSocket server memory (not Redis)
- Only broadcast to same-document collaborators (no cross-doc leak)
- Batch multiple cursor updates (max 10/sec per user)

*See `pseudocode.md::PresenceManager` for implementation.*

---

### 4.7 Rich Text Formatting

**Challenge**: How to represent complex formatting (bold, italic, colors, links) in a conflict-resistant way?

**Solution: Attributed String Model**

Each character has associated attributes:

```
Character: {
    char: 'H',
    attributes: {
        bold: true,
        italic: false,
        color: '#FF0000',
        link: 'https://example.com'
    },
    id: {siteId: 'user_a', clock: 42}  // CRDT ID
}
```

**Formatting Operations:**

```
{
    type: 'format',
    range: [10, 20],  // Apply to characters 10-20
    attribute: 'bold',
    value: true
}
```

**Conflict Resolution:**

- Formatting operations are commutative (order doesn't matter)
- Last-write-wins for conflicting formats on same character
- Uses Lamport timestamps or vector clocks for ordering

**Optimization: Run-Length Encoding**

Instead of storing attributes per character, store runs:

```
Text: "Hello World" with "Hello" bold

Storage:
[
    {content: "Hello", attrs: {bold: true}, start_id: 0},
    {content: " World", attrs: {}, start_id: 5}
]
```

Reduces memory by 10-50x for typical documents.

---

## 5. Scalability and Performance Optimizations

### 5.1 Sharding Strategy

**By Document ID:**

```
Document ID → Hash → Shard ID
doc_id: "abc123" → hash("abc123") % 10 = 3 → Shard 3
```

**Shard Contains:**

- OT Worker (processes all operations for documents in this shard)
- Kafka Partition (stores event log for this shard)
- PostgreSQL Partition (stores events for this shard)
- Redis Cache Partition (stores snapshots for this shard)

**Benefits:**

- Linear scalability (add more shards to handle more documents)
- No distributed transactions (all operations for a document go to same shard)
- Load balancing (hash function distributes evenly)

**Handling Hot Documents:**

- Document with 100+ simultaneous editors (rare, but possible)
- Solution: **Read Replicas for Hot Documents**
    - Primary shard handles writes (OT processing)
    - Read replicas handle broadcasts (WebSocket distribution)
    - Kafka replicates to multiple consumers

---

### 5.2 Caching Strategy

**Multi-Layer Cache:**

1. **Client-Side Cache (Browser):**
    - Full document in memory
    - Recent operations in IndexedDB
    - Reduces server load for repeated opens

2. **CDN/Edge Cache (CloudFlare):**
    - Cached snapshots for public/read-only documents
    - TTL: 60 seconds
    - Reduces origin server load by 80%

3. **Redis Cache (Server-Side):**
    - Document snapshots: `doc:{doc_id}:snapshot`
    - Recent operations: `doc:{doc_id}:operations` (last 1000)
    - Presence data: `doc:{doc_id}:presence`
    - TTL: 1 hour (hot documents), 10 minutes (warm documents)

**Cache Invalidation:**

- Write-through for snapshots (update cache on every snapshot save)
- Lazy invalidation for operations (TTL expiry)
- Presence uses heartbeat expiry (self-cleaning)

---

### 5.3 Network Optimization

**Message Compression:**

- Gzip compression for large operations (> 1 KB)
- Custom binary protocol for small operations (< 1 KB)
- Reduces bandwidth by 60-80%

**Operation Batching:**

- Client buffers operations for 50ms
- Sends batch as single WebSocket message
- Reduces per-message overhead (from 1000 msg/sec to 20 msg/sec)

**Delta Sync:**

- Instead of sending full document, send only changed regions
- Client tracks "dirty ranges" (modified sections)
- Reduces sync payload from 50 KB to < 1 KB (98% reduction)

---

### 5.4 Database Optimization

**PostgreSQL Tuning:**

```sql
-- Partitioning by doc_id for fast queries
CREATE TABLE document_events (
    ...
) PARTITION BY HASH (doc_id);

-- Indexes for common queries
CREATE INDEX idx_doc_recent ON document_events(doc_id, created_at DESC);
CREATE INDEX idx_version_lookup ON document_events(doc_id, event_id);
```

**Write Optimization:**

- Batch insert events (100 events per transaction)
- Append-only table (no updates, only inserts)
- Asynchronous replication (eventual consistency for replicas)

**Read Optimization:**

- Read replicas for version history queries (offload from primary)
- Connection pooling (PgBouncer): 10k clients → 100 DB connections
- Prepared statements (reduce parse overhead)

---

### 5.5 Handling Burst Traffic

**Problem**: Sudden spike in traffic (e.g., popular document shared on social media)

**Solutions:**

1. **Auto-Scaling WebSocket Servers:**
    - Kubernetes HPA (Horizontal Pod Autoscaler)
    - Scale based on CPU/memory/connection count
    - Add new pods in < 30 seconds

2. **Kafka Partition Expansion:**
    - Increase partitions for hot topics
    - Rebalance consumers automatically

3. **Rate Limiting:**
    - Per-user: 100 operations/sec (prevents abuse)
    - Per-document: 1000 operations/sec (prevents single doc overload)
    - Graceful degradation: Queue excess operations, process asynchronously

4. **Circuit Breaker:**
    - If OT service latency > 500ms, switch to eventual consistency mode
    - Queue operations, process when load decreases
    - Prevents cascading failures

---

## 6. Fault Tolerance and Disaster Recovery

### 6.1 Handling Server Failures

**WebSocket Server Crash:**

- Client detects disconnection (heartbeat timeout)
- Client reconnects to different server (load balancer routes)
- Client sends last known version → Server replays missed operations
- User experience: Brief freeze (1-2 seconds), then resumes

**OT Worker Crash:**

- Kafka consumer group rebalances
- Another worker takes over the partition
- Operations queued in Kafka are not lost
- Reprocesses last checkpoint (idempotent operations)

**Database Failure:**

- PostgreSQL replication (primary + 2 replicas)
- Automatic failover (< 30 seconds)
- Write operations queued during failover (buffered in Kafka)

---

### 6.2 Data Durability Guarantees

**Event Log Durability:**

- Kafka replication factor: 3 (data survives 2 node failures)
- PostgreSQL replication: Synchronous (primary + 1 replica)
- S3 versioning: 11 nines durability

**Preventing Data Loss:**

- Client acknowledgment: Only mark operation as "sent" after server ACK
- At-least-once delivery: Kafka guarantees no message loss
- Idempotent operations: Duplicate operations don't corrupt state

**Backup Strategy:**

- Hourly snapshots to S3 (full document state)
- Daily backups of event log (PostgreSQL dump)
- Retention: 30 days (compliance requirement)

---

### 6.3 Disaster Recovery

**Multi-Region Deployment:**

```
Primary Region (US-East):  
  - Handles all writes
  - Active OT processing

Secondary Region (EU-West):
  - Read replicas (eventual consistency, 1-2 sec lag)
  - Disaster recovery standby

Failover Process:
  1. Detect primary failure (health check timeout)
  2. Promote secondary to primary (manual or automated)
  3. Update DNS (clients reconnect to new region)
  4. Resume operations (some in-flight operations may be lost)
```

**RTO (Recovery Time Objective): 5 minutes**  
**RPO (Recovery Point Objective): 1 minute** (max data loss)

---

## 7. Security and Privacy

### 7.1 Authentication and Authorization

**Authentication:**

- JWT tokens for WebSocket connections
- OAuth 2.0 for user login (Google/Microsoft SSO)
- Token refresh every 1 hour

**Authorization (Document Permissions):**

```sql
CREATE TABLE document_permissions (
    doc_id UUID,
    user_id UUID,
    role VARCHAR(20), -- 'owner', 'editor', 'commenter', 'viewer'
    granted_at TIMESTAMP,
    PRIMARY KEY (doc_id, user_id)
);
```

**Permission Checks:**

- On WebSocket connect: Verify user has access to document
- On operation: Verify user has 'editor' role
- On comment: Verify user has 'commenter' or 'editor' role
- On share: Only 'owner' can change permissions

---

### 7.2 Data Encryption

**Encryption at Rest:**

- PostgreSQL: Transparent Data Encryption (TDE)
- S3: Server-side encryption (SSE-S3)
- Redis: Encrypted persistence (AOF/RDB files)

**Encryption in Transit:**

- TLS 1.3 for all WebSocket connections
- TLS for all database connections
- End-to-end encryption for sensitive documents (optional feature)

---

### 7.3 Rate Limiting and Abuse Prevention

**Per-User Rate Limits:**

- 100 operations/sec (prevents spam/bots)
- 10 document opens/min (prevents scraping)
- 50 comments/hour (prevents abuse)

**Per-Document Rate Limits:**

- 1000 operations/sec total (prevents single doc overload)
- 50 simultaneous editors (prevents performance degradation)

**Abuse Detection:**

- Flag documents with excessive edit rate (potential data exfiltration)
- Flag users with unusual patterns (bot behavior)
- Quarantine suspected malicious documents

---

## 8. Monitoring and Observability

### 8.1 Key Metrics

| Metric                          | Target             | Alert Threshold      |
|---------------------------------|--------------------|----------------------|
| **Edit Propagation Latency**    | < 100ms (p99)      | > 200ms              |
| **WebSocket Connection Count**  | 500k concurrent    | > 600k (scale up)    |
| **OT Processing Latency**       | < 50ms (p99)       | > 100ms              |
| **Event Log Write Throughput**  | 1M events/sec      | < 500k (investigate) |
| **Kafka Consumer Lag**          | < 1 second         | > 5 seconds          |
| **PostgreSQL Query Latency**    | < 10ms (p99)       | > 50ms               |
| **Cache Hit Rate (Redis)**      | > 95%              | < 90%                |
| **Document Load Time**          | < 500ms (p99)      | > 1 second           |
| **Operation Conflict Rate**     | < 1%               | > 5%                 |
| **WebSocket Reconnection Rate** | < 5/min per server | > 50/min             |

---

### 8.2 Distributed Tracing

**Trace Propagation:**

- Each operation has unique trace ID
- Trace follows operation: Client → WebSocket → Kafka → OT → PostgreSQL → Broadcast
- Visualize full request flow with Jaeger/Zipkin

**Trace Span Breakdown:**

```
Total Latency: 85ms
  - Client → WebSocket: 10ms (network)
  - WebSocket → Kafka: 5ms (publish)
  - Kafka → OT Worker: 15ms (consumer lag)
  - OT Processing: 30ms (transformation)
  - PostgreSQL Write: 10ms (database)
  - Broadcast to Clients: 15ms (WebSocket push)
```

---

### 8.3 Logging Strategy

**Structured Logs (JSON):**

```json
{
  "timestamp": "2025-10-31T12:34:56Z",
  "level": "INFO",
  "service": "ot-worker",
  "doc_id": "abc123",
  "user_id": "user_456",
  "operation": "insert",
  "latency_ms": 45,
  "conflict": false,
  "trace_id": "xyz789"
}
```

**Log Aggregation:**

- Centralized logging (ELK Stack / DataDog)
- Query logs by doc_id, user_id, trace_id
- Retention: 30 days (hot), 1 year (cold/S3)

---

### 8.4 Alerting Rules

**Critical Alerts (Page On-Call):**

- Edit propagation latency > 500ms for 5 minutes (user-facing impact)
- OT worker down for > 1 minute (data loss risk)
- PostgreSQL primary down (write failure)
- Kafka consumer lag > 10 seconds (processing backlog)

**Warning Alerts (Slack Notification):**

- Cache hit rate < 90% (performance degradation)
- Operation conflict rate > 5% (potential OT bug)
- WebSocket reconnection spike (network issue)

---

## 9. Common Anti-Patterns

### ❌ **Anti-Pattern 1: No Operation Ordering**

**Problem:**
Processing operations in arrival order without considering logical causality leads to incorrect document state.

**Example:**

```
User A: insert("X", pos=5) at version 1
User B: insert("Y", pos=6) at version 1
Server receives B before A (network delay)
Applies B first → Wrong position, document corrupted
```

**Solution:** ✅
Use vector clocks or logical timestamps to order operations by causality, not arrival time. Kafka partitioning by doc_id
guarantees order within a document.

---

### ❌ **Anti-Pattern 2: Storing Full Document on Every Edit**

**Problem:**
Saving entire document state on every operation wastes storage and makes version history queries slow.

**Example:**

```
Event 1: Store full doc (50 KB)
Event 2: Store full doc (50 KB)
...
Event 1000: Store full doc (50 KB)
Total: 50 MB for 1000 edits!
```

**Solution:** ✅
Use event sourcing (store only operations) + periodic snapshots. Reconstructing document: load latest snapshot + apply
operations since snapshot.

---

### ❌ **Anti-Pattern 3: Synchronous OT Processing**

**Problem:**
Blocking user's edit until OT transformation completes increases perceived latency.

**Example:**

```
User types → Send to server → Wait for OT → Wait for DB write → Receive ACK → Show character
Total latency: 100-200ms (feels laggy)
```

**Solution:** ✅
Optimistic UI updates: Apply operation locally immediately, send to server asynchronously. Reconcile if server returns
different transformation. Latency: < 10ms (feels instant).

---

### ❌ **Anti-Pattern 4: Global Lock on Document**

**Problem:**
Using a distributed lock for every edit serializes all operations, killing throughput.

**Example:**

```
10 users editing simultaneously
Each operation acquires lock → processes → releases lock
Throughput: 1 operation at a time = 10 ops/sec (not 100+ ops/sec)
```

**Solution:** ✅
OT/CRDT algorithms allow concurrent operations without locks. Shard by document ID, single-threaded processing per
document (no lock needed). Throughput: 100+ ops/sec per document.

---

### ❌ **Anti-Pattern 5: Keeping Deleted Content Forever**

**Problem:**
CRDT tombstones accumulate indefinitely, growing document size even after deletions.

**Example:**

```
Document after 1M edits (500k deletions):
- Visible content: 10 KB
- Tombstones: 500 KB
- Memory usage 50x larger than actual content!
```

**Solution:** ✅
Periodic tombstone garbage collection: Remove tombstones older than 30 days (after all clients have synced). Use
snapshot compaction to clean history.

---

### ❌ **Anti-Pattern 6: Broadcasting All Operations to All Users**

**Problem:**
Sending every cursor movement and formatting change to all users wastes bandwidth.

**Example:**

```
100 users in a document, each moves cursor 10 times/sec
Total broadcast: 100 × 100 × 10 = 100k messages/sec
Bandwidth per user: 1000 msgs/sec (overwhelming!)
```

**Solution:** ✅

- Throttle cursor updates (max 10/sec per user)
- Only broadcast content operations (insert/delete) in real-time
- Batch formatting operations (apply every 500ms)
- Reduces bandwidth by 95%

---

### ❌ **Anti-Pattern 7: No Conflict Metrics**

**Problem:**
Not tracking OT transformation conflicts makes it impossible to detect algorithm bugs.

**Example:**

```
OT bug causes rare conflict resolution error
Users occasionally see corrupted documents
No metrics → No alerts → Bug undetected for months
```

**Solution:** ✅
Track conflict rate metric: `(operations_transformed / total_operations) × 100%`. Expected: < 1%. If > 5%, investigate
OT algorithm or network issues.

---

## 10. Alternative Approaches and Trade-offs

### Alternative 1: Pure CRDT (No OT)

**Approach:**
Use CRDT for all operations (real-time and offline), eliminate OT entirely.

**Pros:**

- ✅ Simpler codebase (no complex OT transformation logic)
- ✅ Better partition tolerance (works offline, multi-region)
- ✅ Decentralized (no central OT server bottleneck)

**Cons:**

- ❌ 3-10x higher bandwidth (CRDT metadata)
- ❌ 2-5x higher memory usage (character IDs)
- ❌ Tombstone accumulation (requires periodic GC)

**When to Use:**

- Applications with frequent offline usage (mobile-first apps)
- Peer-to-peer collaboration (no central server)
- Multi-region writes with high WAN latency

**Why We Didn't Choose It:**
Google Docs-scale requires minimal latency and bandwidth for real-time collaboration. OT is more efficient for the 95%
online use case.

---

### Alternative 2: Lock-Based Editing

**Approach:**
Only one user can edit a section at a time (acquire lock on paragraph/line).

**Pros:**

- ✅ No conflict resolution needed (only one writer)
- ✅ Simple implementation (no OT/CRDT)
- ✅ Easy to reason about (serializable)

**Cons:**

- ❌ Poor user experience (blocked by other users)
- ❌ Lock contention (hot sections become bottleneck)
- ❌ Deadlock risk (user acquires lock and disconnects)

**When to Use:**

- Financial documents (strict consistency, low concurrency)
- Legal contracts (only one editor at a time acceptable)
- Simple note-taking apps (1-2 simultaneous editors)

**Why We Didn't Choose It:**
Google Docs requires seamless multi-user collaboration. Lock-based editing feels restrictive and frustrates users.

---

### Alternative 3: Last-Write-Wins (LWW)

**Approach:**
Each operation has timestamp, conflicts resolved by keeping latest timestamp.

**Pros:**

- ✅ Extremely simple (no transformation or merge logic)
- ✅ Low latency (no complex processing)
- ✅ Scales easily (stateless)

**Cons:**

- ❌ Data loss (overwrites concurrent edits)
- ❌ Poor user experience (edits disappear mysteriously)
- ❌ Requires synchronized clocks (difficult in distributed systems)

**When to Use:**

- Key-value stores (DynamoDB, Cassandra)
- User preferences (settings, themes)
- Non-critical data (last update is acceptable)

**Why We Didn't Choose It:**
Unacceptable for collaborative documents. Users expect every keystroke to be preserved, not overwritten.

---

### Alternative 4: Diff-Based Synchronization

**Approach:**
Periodically compute diff (like Git) between client and server versions, merge diffs.

**Pros:**

- ✅ Mature algorithms (Git merge, diff3)
- ✅ Works well for code files
- ✅ Human review of conflicts (merge conflicts)

**Cons:**

- ❌ Not real-time (periodic sync, not instant)
- ❌ Merge conflicts require manual resolution (poor UX)
- ❌ Latency (computing diffs is slow for large documents)

**When to Use:**

- Version control systems (Git, SVN)
- Asynchronous collaboration (wikis, code reviews)
- Document comparison tools

**Why We Didn't Choose It:**
Google Docs requires real-time collaboration with automatic conflict resolution. Manual merge conflicts are
unacceptable.

---

## 11. Real-World Examples and Case Studies

### Google Docs (OT-Based)

**Architecture:**

- Custom OT algorithm developed over 10+ years
- Centralized OT server (Jupiter algorithm)
- WebSocket for real-time communication
- Event sourcing for version history

**Scale:**

- 2 billion documents
- 1 billion MAU
- Millions of concurrent editors

**Lessons:**

- OT requires extensive testing (edge cases are subtle)
- Snapshots critical for performance (can't replay millions of events)
- Offline support added later (CRDT-like merge for offline edits)

---

### Microsoft Office 365 (Hybrid OT/CRDT)

**Architecture:**

- OT for real-time collaboration (Word, Excel)
- CRDT for OneNote (offline-first design)
- Azure SignalR for WebSocket management
- Blob storage for version history

**Scale:**

- 345 million paid seats
- 50+ petabytes of document storage

**Lessons:**

- Different tools need different algorithms (Word = OT, OneNote = CRDT)
- Multi-region writes require CRDT or complex conflict resolution
- Caching snapshots in CDN reduces latency by 80%

---

### Notion (CRDT-Based)

**Architecture:**

- Pure CRDT (no OT)
- Blocks as fundamental unit (not characters)
- PostgreSQL for persistence
- WebSocket for real-time sync

**Scale:**

- 30 million users
- < 100ms sync latency

**Lessons:**

- Block-based CRDT reduces metadata overhead (vs character-based)
- CRDT works well for structured documents (not free-form text)
- Tombstone GC required every 24 hours (otherwise memory grows)

---

### Figma (CRDT + Operational Transform)

**Architecture:**

- Hybrid approach (CRDT for offline, OT for real-time)
- Custom multiplayer server (C++ for performance)
- WebAssembly client (fast rendering)

**Scale:**

- 4 million users
- 50k concurrent editing sessions

**Lessons:**

- Design tools need fast rendering (60 FPS) while syncing
- Conflict resolution for visual elements (shapes, layers) differs from text
- Presence (cursors, selections) is 80% of multiplayer traffic

---

## 12. Future Enhancements

### 13.1 End-to-End Encryption

**Challenge:**
OT server must see plaintext to transform operations.

**Solution:**

- Client-side encryption with shared key (for private docs)
- Server operates on encrypted CRDT (position-based, doesn't need content)
- Trade-off: Can't do server-side search or indexing

---

### 13.2 AI-Powered Collaboration

**Features:**

- Smart suggestions (grammar, tone, brevity)
- Auto-complete based on context
- Summarization of long documents

**Implementation:**

- Run ML models on document snapshots (not real-time)
- Suggestions appear as ghost text (user can accept/reject)
- Privacy: Only process if user opts in

---

### 13.3 Voice/Video Integration

**Use Case:**
Collaborative editing during video call.

**Architecture:**

- WebRTC for peer-to-peer video
- Document edits still go through central server
- Cursor tracking synced with voice (highlight speaker's cursor)

---

### 13.4 Mobile Optimization

**Challenges:**

- Limited bandwidth (cellular networks)
- Intermittent connectivity (subway, airplane mode)
- Battery constraints

**Solutions:**

- Aggressive operation batching (send every 5 seconds, not 50ms)
- Delta sync (only send changed regions)
- CRDT-based offline editing (seamless reconnect)

---

## 13. Interview Discussion Points

### For System Design Interviews

**Key Topics to Discuss:**

1. **Conflict Resolution (Core Focus):**
    - Explain OT vs CRDT trade-offs
    - Walk through transformation example
    - Discuss convergence guarantees

2. **Scalability:**
    - Sharding by document ID
    - Handling hot documents (100+ simultaneous editors)
    - Multi-region deployment challenges

3. **Latency Optimizations:**
    - WebSocket vs long polling vs SSE
    - Optimistic UI updates
    - Message batching and compression

4. **Consistency Guarantees:**
    - Event sourcing for strong consistency
    - Snapshot strategy (CQRS)
    - Handling network partitions

5. **Failure Scenarios:**
    - WebSocket disconnection (reconnect strategy)
    - OT worker crash (Kafka rebalancing)
    - Database failover (PostgreSQL replication)

**Follow-Up Questions to Expect:**

- "How do you handle a user editing while offline for hours?"
    - Answer: CRDT-based merge, store operations locally, replay on reconnect

- "What if two users delete and insert at the same position?"
    - Answer: OT transformation rules (insert after delete), deterministic ordering by user ID

- "How do you prevent one user from corrupting the document (malicious edits)?"
    - Answer: Server-side validation, rate limiting, permission checks, version rollback

- "Can you scale to 1000 simultaneous editors on one document?"
    - Answer: Challenges with broadcast fanout (1000² messages), use hierarchical broadcast, read replicas, or impose
      soft limit (50-100 editors)

---

## 14. Trade-offs Summary

| What We Gain                                                | What We Sacrifice                                                       |
|-------------------------------------------------------------|-------------------------------------------------------------------------|
| ✅ **Real-time collaboration** (< 100ms)                     | ❌ **Complexity** (OT/CRDT algorithms are hard to implement correctly)   |
| ✅ **Strong consistency** (all users converge to same state) | ❌ **Latency overhead** (OT transformation adds 20-50ms)                 |
| ✅ **Offline editing** (CRDT-based merge)                    | ❌ **Storage overhead** (CRDT metadata is 3-10x larger)                  |
| ✅ **Complete version history** (event sourcing)             | ❌ **Storage cost** (petabyte-scale for full history)                    |
| ✅ **Scalability** (sharding by document)                    | ❌ **Hot document problem** (single doc with 100+ editors is bottleneck) |
| ✅ **Low latency** (WebSocket + in-memory)                   | ❌ **Infrastructure cost** (stateful WebSocket servers, Redis cache)     |
| ✅ **Fault tolerance** (Kafka + PostgreSQL replication)      | ❌ **Operational complexity** (many moving parts, difficult to debug)    |

---

## 15. References

### Related System Design Components

- **[2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)** -
  WebSocket architecture and alternatives
- **[2.1.7 PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.7-postgresql-deep-dive.md)** - Event sourcing and
  database design
- **[2.1.11 Redis Deep Dive](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)** - Caching snapshots and
  presence tracking
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Event streaming
  and ordering guarantees
- *
  *[2.3.4 Distributed Transactions and Idempotency](../../02-components/2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md)
  ** - Ensuring exactly-once processing
- **[1.1.4 Data Consistency Models](../../01-principles/1.1.4-data-consistency-models.md)** - Strong vs eventual
  consistency
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - Sharding strategies

### Related Design Challenges

- **[3.3.1 Live Chat System](../3.3.1-live-chat-system/)** - WebSocket connection management patterns
- **[3.4.1 Stock Exchange](../3.4.1-stock-exchange/)** - Low-latency event processing
- **[3.4.5 Stock Brokerage](../3.4.5-stock-brokerage/)** - Real-time broadcast patterns

### External Resources

- **Operational Transformation (OT):**
    - [Google Wave OT Paper](https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html) -
      Original OT algorithm
    - [Jupiter Collaboration System](https://dl.acm.org/doi/10.1145/215585.215706) - Early collaborative editing system

- **CRDT (Conflict-Free Replicated Data Types):**
    - [CRDT Primer](https://crdt.tech/) - Comprehensive CRDT guide
    - [Yjs CRDT Framework](https://github.com/yjs/yjs) - JavaScript CRDT library
    - [Automerge CRDT Library](https://github.com/automerge/automerge) - CRDT implementation

- **Real-World Implementations:**
    - [Google Docs Engineering](https://drive.googleblog.com/) - Google Docs architecture
    - [Figma's Multiplayer Technology](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/) - Real-time
      collaboration
    - [Notion's Data Model](https://www.notion.so/blog/data-model-behind-notion) - Notion architecture

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Event sourcing, CRDTs, consistency models
- *Building Microservices* by Sam Newman - CQRS patterns, event-driven architecture