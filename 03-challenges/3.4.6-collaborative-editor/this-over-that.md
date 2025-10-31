# Collaborative Editor - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions for the Collaborative Editor system, explaining what we chose, why we chose it, what alternatives we considered, and when to reconsider.

---

## Table of Contents

1. [Operational Transformation (OT) vs CRDT](#1-operational-transformation-ot-vs-crdt)
2. [WebSocket vs Server-Sent Events (SSE) vs Long Polling](#2-websocket-vs-server-sent-events-sse-vs-long-polling)
3. [Event Sourcing vs State-Based Persistence](#3-event-sourcing-vs-state-based-persistence)
4. [PostgreSQL vs MongoDB for Event Log](#4-postgresql-vs-mongodb-for-event-log)
5. [Redis vs Memcached for Caching](#5-redis-vs-memcached-for-caching)
6. [Kafka vs RabbitMQ vs AWS SQS](#6-kafka-vs-rabbitmq-vs-aws-sqs)
7. [Synchronous vs Asynchronous Replication](#7-synchronous-vs-asynchronous-replication)
8. [Client-Side Optimistic Updates vs Server Confirmation](#8-client-side-optimistic-updates-vs-server-confirmation)
9. [Sticky Sessions vs Stateless WebSocket Servers](#9-sticky-sessions-vs-stateless-websocket-servers)
10. [CQRS (Snapshots) vs Pure Event Sourcing](#10-cqrs-snapshots-vs-pure-event-sourcing)

---

## 1. Operational Transformation (OT) vs CRDT

### The Problem

When two users edit the same document simultaneously, their operations may conflict. We need an algorithm that resolves conflicts automatically while preserving both users' intentions and ensuring all clients converge to the same final state.

**Example Conflict:**
```
Initial: "Hello"
User A: insert("X", pos=5) → "HelloX"
User B: insert("Y", pos=5) → "HelloY"
Final: "HelloXY" or "HelloYX"? (must be deterministic)
```

### Options Considered

| Feature | Operational Transformation (OT) | CRDT (Conflict-Free Replicated Data Types) |
|---------|--------------------------------|-------------------------------------------|
| **Algorithm** | Transform operations against each other based on context | Data structures with built-in merge semantics |
| **Metadata Overhead** | Minimal (just position/length) | High (3-10x, unique ID per character) |
| **Bandwidth** | Low (~100 bytes per operation) | High (300-1000 bytes per operation) |
| **Complexity** | Very complex (transformation functions hard to implement correctly) | Moderate (merge is simple set union) |
| **Latency** | 20-50ms (transformation processing) | 10-30ms (no transformation, direct merge) |
| **Offline Support** | Poor (requires central server) | Excellent (peer-to-peer merge) |
| **Multi-Region** | Difficult (single primary region) | Easy (multi-master writes) |
| **Convergence Guarantee** | Guaranteed (if transformation functions correct) | Guaranteed (mathematical proof) |
| **Maturity** | Mature (Google Docs uses Jupiter OT, 15+ years) | Emerging (Yjs, Automerge, 5 years) |
| **Tombstone Problem** | N/A | Yes (deleted characters accumulate, need GC) |

### Decision Made

**Hybrid Approach: OT for Real-Time Collaboration, CRDT for Offline Sync**

| Use Case | Algorithm | Reason |
|----------|-----------|--------|
| **Real-time editing (online users)** | OT | Lower bandwidth, more efficient for 95% use case |
| **Offline editing (mobile, intermittent)** | CRDT | Partition tolerant, automatic merge on reconnect |
| **Multi-region writes** | CRDT | No central authority, works across WAN |

### Rationale

1. **OT for Real-Time (Primary Use Case):**
   - **95% of users are online**: Most collaborative editing happens with stable internet connections
   - **Bandwidth Efficiency**: OT operations are 3-10x smaller than CRDT (critical for mobile users)
   - **Proven at Scale**: Google Docs processes billions of OT operations daily
   - **Lower Latency**: OT transformation (20-50ms) faster than CRDT merge + reconciliation (50-100ms) for online case
   - **Example**: Two users typing simultaneously → OT resolves in < 50ms with minimal bandwidth

2. **CRDT for Offline (Edge Case):**
   - **Mobile Users**: Frequently disconnect (subways, elevators, poor signal)
   - **Partition Tolerance**: CRDT works without server connection (local merge)
   - **Graceful Degradation**: Even if server fails, clients can sync peer-to-peer
   - **Example**: User edits on airplane → lands → CRDT merge seamlessly syncs changes

3. **Why Not Pure OT:**
   - **Offline Support**: OT requires central server for transformation (doesn't work offline)
   - **Multi-Region**: OT needs single primary region (WAN latency hurts global users)
   - **Single Point of Failure**: If OT server down, no conflict resolution

4. **Why Not Pure CRDT:**
   - **Bandwidth**: 3-10x larger payloads (each character needs unique ID)
   - **Memory**: 2-5x higher memory usage (metadata stored per character)
   - **Tombstone Accumulation**: Deleted characters remain as tombstones (need periodic GC)
   - **Example**: 10 KB document with 1000 edits → 50 KB CRDT state vs 15 KB OT state

### Implementation Details

**OT Implementation (Jupiter Algorithm):**
```
function transform(op1, op2):
    if op1.type == "insert" and op2.type == "insert":
        if op1.position < op2.position:
            return op2  // No change needed
        else if op1.position > op2.position:
            op2.position += op1.length  // Shift position
            return op2
        else:  // Same position
            if op1.user_id < op2.user_id:  // Tie-breaker
                op2.position += op1.length
            return op2
```

*See `pseudocode.md::transform()` for complete implementation.*

**CRDT Implementation (RGA - Replicated Growable Array):**
```
Character: {
    id: {siteId: "user_a", logicalClock: 42},
    content: "X",
    deleted: false
}

Merge: Sort all characters by (siteId, logicalClock), filter deleted=false
```

*See `pseudocode.md::CRDTDocument` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Low bandwidth for real-time (OT) | ❌ Complexity of implementing OT correctly |
| ✅ Offline support (CRDT) | ❌ Higher storage for CRDT metadata |
| ✅ Best of both worlds | ❌ Two algorithms to maintain |

### When to Reconsider

**Switch to Pure CRDT if:**
- Majority of users work offline (e.g., mobile-first app with poor connectivity)
- Multi-region writes are common (e.g., global company with distributed teams)
- Bandwidth is not a constraint (e.g., desktop-only app with broadband)
- CRDT libraries mature significantly (e.g., Yjs becomes industry standard)

**Switch to Pure OT if:**
- 99%+ users are online (e.g., enterprise intranet with stable networks)
- Bandwidth is critical (e.g., emerging markets with expensive data plans)
- Single-region deployment (e.g., all users in one country)

---

## 2. WebSocket vs Server-Sent Events (SSE) vs Long Polling

### The Problem

Real-time collaboration requires instant propagation of edits to all collaborators. We need a communication protocol that supports:
- **Bi-directional communication** (client → server and server → client)
- **Low latency** (< 100ms propagation)
- **Persistent connections** (avoid connection overhead per message)
- **Scalability** (support 500k concurrent connections)

### Options Considered

| Feature | WebSocket | Server-Sent Events (SSE) | Long Polling |
|---------|-----------|-------------------------|--------------|
| **Bi-Directional** | ✅ Yes (full duplex) | ❌ No (server → client only) | ✅ Yes (via separate request) |
| **Latency** | < 10ms (no overhead) | < 20ms (HTTP overhead) | 50-200ms (request/response cycle) |
| **Connection Overhead** | Low (persistent) | Low (persistent) | High (new connection per message) |
| **Browser Support** | ✅ All modern browsers | ✅ All modern browsers | ✅ All browsers (including legacy) |
| **Firewall/Proxy** | ⚠️ May be blocked | ✅ Works everywhere (HTTP) | ✅ Works everywhere (HTTP) |
| **Message Ordering** | ✅ Guaranteed (TCP) | ✅ Guaranteed (HTTP) | ⚠️ Not guaranteed (parallel requests) |
| **Scalability** | High (stateful but efficient) | High (stateful) | Low (connection churn) |
| **Compression** | ✅ Built-in (deflate) | ✅ HTTP compression | ✅ HTTP compression |
| **Implementation Complexity** | Moderate | Low | Low |

### Decision Made

**WebSocket for Real-Time Collaboration**

### Rationale

1. **Bi-Directional Communication:**
   - **Requirement**: Client sends edits to server AND server pushes edits to client
   - **WebSocket**: Native support (single connection)
   - **SSE**: Server → client only, need separate HTTP POST for client → server (two connections)
   - **Long Polling**: Requires two connections (polling + POST requests)
   - **Winner**: WebSocket (single connection, simpler)

2. **Latency:**
   - **WebSocket**: < 10ms per message (no HTTP overhead)
   - **SSE**: < 20ms (HTTP headers per message)
   - **Long Polling**: 50-200ms (request/response cycle, TCP handshake)
   - **Winner**: WebSocket (critical for < 100ms edit propagation requirement)

3. **Connection Efficiency:**
   - **WebSocket**: 1 persistent connection per client (500k connections)
   - **SSE**: 1 persistent connection + periodic POST requests (750k connections equivalent)
   - **Long Polling**: 2-5 connections per client (connection pool) = 1-2.5M connections!
   - **Winner**: WebSocket (scales to 500k connections per server)

4. **Message Ordering:**
   - **WebSocket**: TCP guarantees in-order delivery
   - **SSE**: HTTP/2 multiplexing with ordering
   - **Long Polling**: Parallel requests may arrive out-of-order (need sequence numbers)
   - **Winner**: WebSocket (critical for OT transformation correctness)

5. **Why Not SSE:**
   - No bi-directional support (need two connections)
   - HTTP overhead (headers per message)
   - HTTP/1.1 has 6-connection limit per domain (workaround: multiple domains)
   - Use case: Better for one-way notifications (stock tickers, news feeds)

6. **Why Not Long Polling:**
   - High latency (50-200ms unacceptable for real-time editing)
   - Connection churn (new TCP handshake per message)
   - Wastes bandwidth (HTTP headers per request)
   - Use case: Legacy browser support (IE 9, ancient mobile browsers)

### Implementation Details

**WebSocket Server (Node.js/Go):**
```
ws_server = WebSocketServer(port=8080)

connection_pool = {}  // doc_id → [conn1, conn2, ...]

on_connect(client, doc_id):
    connection_pool[doc_id].add(client)
    send_presence_update(doc_id, client.user_id, "join")

on_message(client, message):
    operation = parse(message)
    publish_to_kafka(operation)

on_kafka_message(operation):
    doc_id = operation.doc_id
    for client in connection_pool[doc_id]:
        send(client, operation)
```

*See `pseudocode.md::WebSocketServer` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Low latency (< 10ms) | ❌ Stateful servers (harder to scale horizontally) |
| ✅ Bi-directional (single connection) | ❌ May be blocked by corporate firewalls (fallback needed) |
| ✅ Efficient (minimal overhead) | ❌ Requires load balancer support (sticky sessions) |

### When to Reconsider

**Use SSE if:**
- Communication is primarily server → client (e.g., live notifications, dashboards)
- Corporate firewalls block WebSockets (HTTP-only environments)
- Simpler implementation preferred (SSE is HTTP, easier to debug)

**Use Long Polling if:**
- Legacy browser support required (IE 8, old Android)
- WebSocket proxies unavailable (very restrictive networks)
- Low concurrency (< 1000 users, connection churn acceptable)

**Use HTTP/2 Server Push if:**
- Modern infrastructure (HTTP/2 everywhere)
- CDN integration needed (CloudFlare, Fastly support HTTP/2 push)

---

## 3. Event Sourcing vs State-Based Persistence

### The Problem

How do we store document data? We need:
- **Complete version history** (audit trail, version control)
- **Fast document load** (< 500ms)
- **Efficient writes** (1M operations/sec)
- **Reproducibility** (reconstruct document at any point in time)

### Options Considered

| Feature | Event Sourcing | State-Based (Traditional) |
|---------|----------------|--------------------------|
| **Storage Model** | Store all operations (immutable log) | Store current state (overwrite) |
| **Version History** | ✅ Complete history (every edit preserved) | ⚠️ Snapshots only (periodic backups) |
| **Write Performance** | ✅ Excellent (append-only, no updates) | ⚠️ Good (update queries) |
| **Read Performance** | ❌ Slow (replay events) | ✅ Fast (single query) |
| **Storage Cost** | High (every event stored) | Low (only current state) |
| **Reproducibility** | ✅ Perfect (replay to any point) | ❌ Limited (snapshots only) |
| **Audit Trail** | ✅ Complete (compliance) | ⚠️ Partial (need audit log table) |
| **Debugging** | ✅ Excellent (replay events to reproduce bugs) | ❌ Difficult (state lost) |
| **Complexity** | High (need snapshot strategy) | Low (standard CRUD) |

### Decision Made

**Event Sourcing with CQRS (Command Query Responsibility Segregation)**

- **Write Model**: Event Sourcing (store all operations in append-only log)
- **Read Model**: CQRS Snapshots (periodic snapshots for fast reads)

### Rationale

1. **Complete Version History (Must-Have):**
   - **Requirement**: Users can view and restore any previous version
   - **Event Sourcing**: Every operation stored, perfect history
   - **State-Based**: Only current state + periodic backups (lose intermediate versions)
   - **Winner**: Event Sourcing (compliance and version control requirement)

2. **Write Performance:**
   - **Event Sourcing**: Append-only (no UPDATE queries, no locks)
   - **State-Based**: UPDATE queries (need row locks, index updates)
   - **Benchmark**: Event sourcing 10x faster writes (100k inserts/sec vs 10k updates/sec)
   - **Winner**: Event Sourcing (1M operations/sec requirement)

3. **Read Performance (Problem with Pure Event Sourcing):**
   - **Pure Event Sourcing**: Load document = replay ALL events (millions of events = minutes!)
   - **State-Based**: Single query (< 10ms)
   - **Solution**: CQRS Snapshots (snapshot every 100 events, replay < 100 events = 50ms)
   - **Winner**: Event Sourcing + CQRS (fast reads AND complete history)

4. **Audit Trail (Compliance):**
   - **Event Sourcing**: Built-in (every operation has user_id, timestamp)
   - **State-Based**: Need separate audit log table (duplicate writes)
   - **Winner**: Event Sourcing (GDPR, SOC 2 compliance)

5. **Reproducibility (Debugging):**
   - **Event Sourcing**: Replay events to reproduce bug (exact state at any time)
   - **State-Based**: State lost, hard to debug production issues
   - **Winner**: Event Sourcing (critical for production debugging)

6. **Why Not Pure Event Sourcing:**
   - Slow reads (replay millions of events)
   - Example: Document with 1M edits = 30 seconds to load!
   - Solution: CQRS (snapshots solve this)

7. **Why Not State-Based:**
   - No version history (can't restore old versions)
   - No audit trail (compliance failure)
   - Slower writes (UPDATE queries vs INSERT)

### Implementation Details

**Event Log Schema (PostgreSQL):**
```sql
CREATE TABLE document_events (
    event_id BIGSERIAL PRIMARY KEY,
    doc_id UUID NOT NULL,
    user_id UUID NOT NULL,
    operation_type VARCHAR(20),  -- 'insert', 'delete', 'format'
    position INT,
    content TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_doc_created (doc_id, created_at)
) PARTITION BY HASH (doc_id);
```

**Snapshot Schema (CQRS Read Model):**
```sql
CREATE TABLE document_snapshots (
    doc_id UUID PRIMARY KEY,
    snapshot_version BIGINT NOT NULL,  -- Last event_id included
    content TEXT NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Document Load Algorithm:**
```
function load_document(doc_id):
    snapshot = redis.get(f"doc:{doc_id}:snapshot")
    if snapshot:
        return snapshot  // Cache hit (< 10ms)
    
    snapshot = db.query("SELECT * FROM document_snapshots WHERE doc_id = ?", doc_id)
    events = db.query("SELECT * FROM document_events WHERE doc_id = ? AND event_id > ?", 
                      doc_id, snapshot.version)
    
    document = snapshot.content
    for event in events:
        document = apply(document, event)  // Replay events
    
    redis.set(f"doc:{doc_id}:snapshot", document, ttl=3600)
    return document
```

*See `pseudocode.md::EventStore` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Complete version history | ❌ Storage cost (1 PB for 10B docs with 100 versions each) |
| ✅ Fast writes (append-only) | ❌ Snapshot management complexity |
| ✅ Perfect reproducibility | ❌ Read latency on cache miss (50-100ms to replay) |
| ✅ Built-in audit trail | ❌ Operational complexity (snapshot cleanup, archiving) |

### When to Reconsider

**Use State-Based if:**
- Version history not required (e.g., temporary notes, chat messages)
- Storage cost is critical (e.g., 1B+ documents, limited budget)
- Read performance > write performance (e.g., 1000:1 read:write ratio)
- Compliance not required (e.g., internal tools, prototypes)

**Use Hybrid (Recent Events Only) if:**
- Version history needed for last 30 days only (e.g., collaboration history, not full audit)
- Storage budget limited (purge old events, keep snapshots)
- Example: Notion keeps last 30 days of events, older versions in snapshots only

---

## 4. PostgreSQL vs MongoDB for Event Log

### The Problem

We need a database to store the event log (billions of operations). Requirements:
- **High write throughput** (1M operations/sec)
- **Append-only workload** (no updates)
- **Strong consistency** (ACID transactions)
- **Querying capabilities** (fetch events by doc_id, timestamp)
- **Scalability** (billions of events, petabyte scale)

### Options Considered

| Feature | PostgreSQL | MongoDB |
|---------|-----------|----------|
| **Data Model** | Relational (SQL) | Document (NoSQL) |
| **Write Throughput** | 100k inserts/sec (single node) | 200k inserts/sec (single node) |
| **Consistency** | ✅ ACID (strong consistency) | ⚠️ Eventual consistency (default) |
| **Transactions** | ✅ Full ACID | ⚠️ Limited (document-level) |
| **Querying** | ✅ SQL (complex queries, joins) | ✅ Flexible (but no joins) |
| **Indexing** | ✅ B-tree, GiST, GIN | ✅ B-tree, compound indexes |
| **Partitioning** | ✅ Native (hash, range, list) | ✅ Sharding |
| **Replication** | ✅ Streaming replication | ✅ Replica sets |
| **Horizontal Scaling** | ⚠️ Difficult (Citus extension) | ✅ Native sharding |
| **Operational Maturity** | ✅ 30+ years, battle-tested | ✅ 15 years, mature |
| **Tooling** | ✅ Excellent (pgAdmin, pgBouncer) | ✅ Good (Compass, Ops Manager) |

### Decision Made

**PostgreSQL with Partitioning**

### Rationale

1. **Strong Consistency (Critical Requirement):**
   - **PostgreSQL**: ACID transactions (all-or-nothing writes)
   - **MongoDB**: Eventual consistency by default (stale reads possible)
   - **Example**: User's operation must be immediately visible on reload (not eventually)
   - **Winner**: PostgreSQL (strong consistency guarantee)

2. **Append-Only Workload:**
   - **PostgreSQL**: Optimized for INSERT (no UPDATE/DELETE)
   - **MongoDB**: Also optimized for inserts
   - **Benchmark**: Both perform well (100k-200k inserts/sec)
   - **Winner**: Tie (both excellent for append-only)

3. **Querying Capabilities:**
   - **PostgreSQL**: Complex SQL queries (joins, aggregations, window functions)
   - **MongoDB**: Flexible queries but no joins (need aggregation pipeline)
   - **Example**: "Fetch all operations by user_id across multiple docs" (join required)
   - **Winner**: PostgreSQL (more powerful query engine)

4. **Partitioning (Scalability):**
   - **PostgreSQL**: Native partitioning by hash(doc_id) (10+ years proven)
   - **MongoDB**: Native sharding (excellent horizontal scaling)
   - **Winner**: Tie (both scale to petabytes)

5. **Operational Maturity:**
   - **PostgreSQL**: 30+ years, massive ecosystem, extensive tooling
   - **MongoDB**: 15 years, mature but fewer DBAs with expertise
   - **Winner**: PostgreSQL (easier to hire talent, more resources)

6. **Why Not MongoDB:**
   - Eventual consistency (unacceptable for collaborative editing)
   - Example: User submits operation, immediately reloads page, doesn't see their edit (bad UX)
   - Workaround: Read-your-writes consistency (complex configuration)

7. **Why PostgreSQL:**
   - Strong consistency (ACID)
   - Complex queries (version history queries, analytics)
   - Mature replication (streaming replication for HA)

### Implementation Details

**PostgreSQL Partitioning:**
```sql
CREATE TABLE document_events (
    event_id BIGSERIAL,
    doc_id UUID NOT NULL,
    user_id UUID NOT NULL,
    operation_type VARCHAR(20),
    position INT,
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
) PARTITION BY HASH (doc_id);

-- Create 10 partitions
CREATE TABLE document_events_0 PARTITION OF document_events FOR VALUES WITH (MODULUS 10, REMAINDER 0);
CREATE TABLE document_events_1 PARTITION OF document_events FOR VALUES WITH (MODULUS 10, REMAINDER 1);
...
```

**Write Performance Optimization:**
```
-- Batch inserts (100 events per transaction)
BEGIN;
INSERT INTO document_events VALUES (...), (...), ...;  // 100 rows
COMMIT;

-- Async replication (don't block writes waiting for replica ACK)
ALTER SYSTEM SET synchronous_commit = off;
```

*See `pseudocode.md::EventStore` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Strong consistency (ACID) | ❌ Horizontal scaling harder (vs MongoDB sharding) |
| ✅ Complex queries (SQL) | ❌ Schema rigidity (vs MongoDB flexibility) |
| ✅ Mature tooling | ❌ Slightly lower write throughput (100k vs 200k/sec single node) |

### When to Reconsider

**Use MongoDB if:**
- Eventual consistency acceptable (e.g., analytics, logs)
- Horizontal scaling critical (petabyte+ scale, many shards)
- Schema flexibility needed (e.g., varying operation types, dynamic fields)
- Team has MongoDB expertise (hiring/training)

**Use Cassandra if:**
- Write throughput > 1M/sec (Cassandra excels at writes)
- Multi-region writes (eventual consistency acceptable)
- Simple queries only (Cassandra's query language limited)

**Use DynamoDB if:**
- Serverless architecture (no database management)
- Unpredictable load (auto-scaling important)
- AWS ecosystem (already using AWS)

---

## 5. Redis vs Memcached for Caching

### The Problem

We need a cache for document snapshots (50 KB average, 100k hot documents = 5 GB cache). Requirements:
- **Low latency** (< 5ms read)
- **Data structures** (support complex types like sorted sets for presence)
- **Persistence** (survive restarts)
- **Pub/Sub** (cross-server broadcast)
- **TTL** (automatic expiry)

### Options Considered

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data Structures** | ✅ Strings, lists, sets, sorted sets, hashes | ❌ Strings only |
| **Persistence** | ✅ RDB snapshots + AOF log | ❌ No persistence (in-memory only) |
| **Replication** | ✅ Master-slave replication | ⚠️ No built-in replication |
| **Pub/Sub** | ✅ Native pub/sub | ❌ No pub/sub |
| **Transactions** | ✅ MULTI/EXEC | ❌ No transactions |
| **Lua Scripting** | ✅ Atomic Lua scripts | ❌ No scripting |
| **TTL** | ✅ Per-key TTL | ✅ Per-key TTL |
| **Performance** | < 1ms (in-memory) | < 1ms (in-memory) |
| **Memory Efficiency** | Good | ✅ Excellent (10-20% less memory) |
| **Multithreading** | ❌ Single-threaded | ✅ Multi-threaded |
| **Cluster Mode** | ✅ Redis Cluster | ⚠️ Requires client-side sharding |

### Decision Made

**Redis**

### Rationale

1. **Data Structures (Critical):**
   - **Redis**: Supports sets for presence tracking (`SADD doc:abc:presence user_id`)
   - **Memcached**: Strings only (need serialization/deserialization for sets)
   - **Winner**: Redis (native data structures simplify code)

2. **Pub/Sub (Required):**
   - **Redis**: Native pub/sub for cross-server broadcast
   - **Memcached**: No pub/sub (need external message broker)
   - **Example**: OT worker broadcasts operation to all WebSocket servers via Redis pub/sub
   - **Winner**: Redis (critical for WebSocket broadcast)

3. **Persistence (Important):**
   - **Redis**: RDB snapshots + AOF log (survive restarts)
   - **Memcached**: No persistence (lose all data on restart)
   - **Winner**: Redis (avoid cold cache on restart)

4. **Transactions (Useful):**
   - **Redis**: MULTI/EXEC for atomic operations
   - **Memcached**: No transactions (race conditions possible)
   - **Example**: Atomically increment rate limit counter and check threshold
   - **Winner**: Redis (safer concurrent access)

5. **Why Not Memcached:**
   - No data structures (presence tracking complex)
   - No pub/sub (need external broker like RabbitMQ)
   - No persistence (cold cache on restart)
   - Use case: Simple key-value cache (e.g., HTTP response caching)

6. **Why Redis:**
   - Rich data structures (sets, sorted sets, hashes)
   - Pub/sub (cross-server communication)
   - Persistence (warm cache on restart)
   - Lua scripting (complex atomic operations)

### Implementation Details

**Presence Tracking (Redis Sets):**
```
SADD doc:abc123:presence alice
EXPIRE doc:abc123:presence 10

SMEMBERS doc:abc123:presence  // Returns: ["alice", "bob"]
```

**Pub/Sub for Broadcast:**
```
# OT Worker publishes operation
PUBLISH doc:abc123:operations '{"type": "insert", "pos": 42, "char": "X"}'

# WebSocket Servers subscribe
SUBSCRIBE doc:abc123:operations

# Receive operation, broadcast to all clients
```

**Rate Limiting (Atomic Lua Script):**
```lua
-- rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = redis.call("INCR", key)
if current == 1 then
    redis.call("EXPIRE", key, 1)
end
if current > limit then
    return 0  -- Rate limit exceeded
else
    return 1  -- Allow
end
```

*See `pseudocode.md::RedisCache` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Rich data structures | ❌ 10-20% higher memory usage (vs Memcached) |
| ✅ Pub/Sub (cross-server communication) | ❌ Single-threaded (vs Memcached multi-threaded) |
| ✅ Persistence (warm cache) | ❌ Slightly more complex operations (vs Memcached simplicity) |

### When to Reconsider

**Use Memcached if:**
- Simple key-value cache only (no complex data structures)
- Memory efficiency critical (limited RAM budget)
- Multi-threaded performance needed (high CPU core count)
- Use case: HTTP response caching, session storage

**Use Hazelcast/Ignite if:**
- Need distributed cache with compute capabilities
- ACID transactions across cache (stronger consistency)
- SQL queries on cached data

---

## 6. Kafka vs RabbitMQ vs AWS SQS

### The Problem

We need a message queue for:
- **Ordering guarantee** (all OT workers see operations in same order)
- **High throughput** (1M messages/sec)
- **Durability** (survive broker failures)
- **Partitioning** (shard by doc_id)
- **Replaying** (reprocess messages on failure)

### Options Considered

| Feature | Kafka | RabbitMQ | AWS SQS |
|---------|-------|----------|---------|
| **Throughput** | ✅ Very high (1M+ msg/sec) | Good (100k msg/sec) | Good (100k msg/sec) |
| **Ordering** | ✅ Per-partition guarantee | ⚠️ Per-queue (no partitions) | ❌ Best-effort only |
| **Durability** | ✅ Replicated log | ✅ Persistent queues | ✅ Managed (11 nines) |
| **Replay** | ✅ Consumers can rewind | ❌ Message deleted after ACK | ❌ Message deleted after processing |
| **Partitioning** | ✅ Native partitions | ❌ Manual (multiple queues) | ❌ No partitions |
| **Message Retention** | ✅ Configurable (days/weeks) | ⚠️ Until consumed | ⚠️ 14 days max |
| **Latency** | < 10ms | < 5ms | 10-50ms (network) |
| **Operational Complexity** | High (ZooKeeper, tuning) | Moderate | ✅ Fully managed (serverless) |
| **Cost** | Moderate (self-hosted) | Moderate (self-hosted) | Low (pay-per-request) |

### Decision Made

**Kafka**

### Rationale

1. **Ordering Guarantee (Critical for OT):**
   - **Kafka**: Guaranteed order within partition (partition by doc_id)
   - **RabbitMQ**: Order per queue (need many queues for partitioning)
   - **SQS**: Best-effort ordering (FIFO queues have 300 msg/sec limit)
   - **Example**: User A's insert and User B's insert on same doc must be processed in same order by all OT workers
   - **Winner**: Kafka (partition by doc_id ensures deterministic OT)

2. **High Throughput:**
   - **Kafka**: 1M+ messages/sec (proven at LinkedIn, Uber)
   - **RabbitMQ**: 100k messages/sec (good but not sufficient)
   - **SQS**: 100k messages/sec standard, 300/sec FIFO
   - **Winner**: Kafka (handles 1M operations/sec requirement)

3. **Replay Capability:**
   - **Kafka**: Consumers store offset, can rewind and replay
   - **RabbitMQ**: Messages deleted after ACK (can't replay)
   - **SQS**: Messages deleted after processing (can't replay)
   - **Example**: OT worker crashes, need to reprocess last 100 messages (idempotent)
   - **Winner**: Kafka (critical for failure recovery)

4. **Message Retention:**
   - **Kafka**: Configurable retention (7 days, 30 days, unlimited)
   - **RabbitMQ**: Until consumed (can't keep history)
   - **SQS**: Max 14 days (limited)
   - **Winner**: Kafka (acts as event log, not just queue)

5. **Why Not RabbitMQ:**
   - Lower throughput (100k vs 1M messages/sec)
   - No replay (can't recover from failures by reprocessing)
   - Manual partitioning (need many queues)
   - Use case: Task queues, job processing (not event streaming)

6. **Why Not SQS:**
   - No ordering guarantee (unacceptable for OT)
   - FIFO queues limited to 300 msg/sec (too slow)
   - Network latency (10-50ms to AWS)
   - Use case: Serverless architectures, bursty workloads

### Implementation Details

**Kafka Topic Configuration:**
```
Topic: document-edits
Partitions: 10 (hash(doc_id) % 10)
Replication Factor: 3 (survive 2 broker failures)
Retention: 7 days (replay recent events)

Producer Config:
  acks: all (wait for all replicas to ACK)
  max.in.flight.requests.per.connection: 1 (preserve order)
  compression: lz4 (reduce bandwidth)

Consumer Config:
  group.id: ot-workers
  enable.auto.commit: false (manual offset commit)
  max.poll.records: 100 (batch processing)
```

**Producing Operations:**
```
producer.send(
    topic="document-edits",
    key=doc_id,  // Partition by doc_id (same doc → same partition)
    value=operation
)
```

**Consuming Operations:**
```
consumer.subscribe(["document-edits"])
while True:
    messages = consumer.poll(timeout=1000)
    for message in messages:
        operation = parse(message.value)
        process_operation(operation)
        consumer.commit()  // Commit offset (checkpoint)
```

*See `pseudocode.md::KafkaProducer` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Ordering guarantee (per partition) | ❌ Operational complexity (ZooKeeper, tuning) |
| ✅ High throughput (1M msg/sec) | ❌ Higher latency (10-20ms vs RabbitMQ's 5ms) |
| ✅ Replay capability | ❌ Storage cost (retain messages for days) |

### When to Reconsider

**Use RabbitMQ if:**
- Throughput < 100k messages/sec (RabbitMQ sufficient)
- Ordering not critical (e.g., independent tasks)
- Replay not needed (e.g., transient notifications)
- Simpler operations (easier to manage than Kafka)

**Use AWS SQS if:**
- Serverless architecture (no infrastructure management)
- Bursty workloads (auto-scaling important)
- Ordering not required (standard SQS, not FIFO)
- Cost-sensitive (pay-per-request cheaper for low volume)

**Use Redis Streams if:**
- Lower latency critical (< 5ms)
- Moderate throughput (100k-500k msg/sec)
- Already using Redis (reduce infrastructure)

---

## 7. Synchronous vs Asynchronous Replication

### The Problem

PostgreSQL event log must be replicated to secondary regions for disaster recovery. Should we use synchronous or asynchronous replication?

**Synchronous Replication:**
- Primary waits for replica ACK before committing transaction
- Guarantees: Zero data loss (RPO = 0)
- Trade-off: Higher latency (wait for network roundtrip)

**Asynchronous Replication:**
- Primary commits immediately, replica catches up eventually
- Guarantees: Low latency
- Trade-off: Potential data loss (RPO = 1-2 seconds)

### Options Considered

| Feature | Synchronous | Asynchronous |
|---------|-------------|--------------|
| **Write Latency** | High (50-200ms + network RTT) | ✅ Low (10-20ms, no waiting) |
| **Data Loss (RPO)** | ✅ Zero (replica has all data) | ⚠️ 1-2 seconds (replication lag) |
| **Availability** | ⚠️ Lower (if replica down, writes block) | ✅ Higher (replica failure doesn't affect writes) |
| **Consistency** | ✅ Strong (replica always up-to-date) | ⚠️ Eventual (1-2 sec lag) |
| **Throughput** | Lower (wait for replica) | ✅ Higher (no blocking) |

### Decision Made

**Asynchronous Replication (with Kafka buffering for durability)**

### Rationale

1. **Write Latency (Critical for User Experience):**
   - **Synchronous**: 50-200ms (wait for replica ACK across WAN)
   - **Asynchronous**: 10-20ms (primary only, replica catches up)
   - **Requirement**: < 100ms edit propagation (synchronous replication makes this impossible)
   - **Winner**: Asynchronous (meets latency requirement)

2. **Data Loss Acceptable (With Mitigation):**
   - **Risk**: Primary crashes, replica is 1-2 seconds behind (lose operations)
   - **Mitigation**: Kafka acts as durable buffer (replication factor 3, operations not lost)
   - **Scenario**: Primary down → Promote replica → Replay missing operations from Kafka
   - **Actual RPO**: < 1 minute (time to replay Kafka backlog)
   - **Winner**: Asynchronous (with Kafka, data loss minimized)

3. **Availability:**
   - **Synchronous**: If replica down, writes block (unavailable)
   - **Asynchronous**: Replica down doesn't affect primary (writes continue)
   - **Winner**: Asynchronous (higher availability)

4. **Why Not Synchronous:**
   - Latency too high (50-200ms unacceptable for real-time editing)
   - Example: User types, waits 200ms to see character (feels broken)
   - Availability risk (replica failure blocks writes)

5. **Why Asynchronous:**
   - Low latency (< 20ms write latency)
   - Kafka provides durability (operations buffered)
   - High availability (replica failure doesn't affect writes)

### Implementation Details

**PostgreSQL Replication Configuration:**
```
# Primary
wal_level = replica
max_wal_senders = 10
synchronous_commit = off  # Asynchronous replication

# Replica
hot_standby = on  # Allow read queries on replica
```

**Disaster Recovery Procedure:**
```
1. Primary crashes
2. Promote replica to primary (10-30 seconds)
3. Replay missing operations from Kafka:
   - Last PostgreSQL commit: event_id 1000
   - Kafka has events 1001-1050
   - Replay events 1001-1050 to new primary
4. Resume normal operations
```

*See `pseudocode.md::ReplicationManager` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Low write latency (< 20ms) | ⚠️ Potential data loss (1-2 sec replication lag) |
| ✅ High availability (replica failure OK) | ⚠️ Eventual consistency (replica slightly stale) |
| ✅ High throughput (no blocking) | ⚠️ Complexity (replay from Kafka on failover) |

### When to Reconsider

**Use Synchronous if:**
- Zero data loss required (financial transactions, medical records)
- Latency not critical (batch processing, analytics)
- Single-region deployment (low network RTT, synchronous feasible)

**Use Quorum-Based (1 of 2 replicas) if:**
- Balance between latency and durability
- PostgreSQL 12+ (synchronous_standby_names = 'ANY 1 (replica1, replica2)')
- Lower latency than full synchronous (only wait for fastest replica)

---

## 8. Client-Side Optimistic Updates vs Server Confirmation

### The Problem

When a user types a character, should we:
- **Optimistic**: Display immediately, send to server asynchronously
- **Pessimistic**: Wait for server confirmation before displaying

### Options Considered

| Feature | Optimistic Updates | Server Confirmation (Pessimistic) |
|---------|-------------------|----------------------------------|
| **Perceived Latency** | ✅ < 5ms (instant) | ❌ 50-100ms (network + processing) |
| **User Experience** | ✅ Feels instant, responsive | ❌ Feels laggy, unresponsive |
| **Correctness** | ⚠️ May need rollback if server rejects | ✅ Always correct |
| **Complexity** | High (reconciliation logic) | Low (simple request/response) |
| **Offline Support** | ✅ Works offline (buffer operations) | ❌ Doesn't work offline |

### Decision Made

**Optimistic Updates with Rollback**

### Rationale

1. **User Experience (Critical):**
   - **Optimistic**: User types, sees character immediately (< 5ms)
   - **Pessimistic**: User types, waits 50-100ms to see character (feels broken)
   - **Winner**: Optimistic (Google Docs uses this, proven UX)

2. **Network Latency is Real:**
   - Even 50ms feels laggy for typing (human perception threshold)
   - Mobile networks: 100-300ms latency (pessimistic unusable)
   - Winner**: Optimistic (only way to meet < 5ms perceived latency)

3. **Rollback Complexity Acceptable:**
   - **Scenario**: Server rejects operation (permission denied, rate limit)
   - **Solution**: Revert optimistic update, show error message
   - **Frequency**: < 0.1% (rare)
   - **Winner**: Optimistic (complexity worth the UX benefit)

4. **Why Not Pessimistic:**
   - Feels laggy (50-100ms delay per keystroke)
   - Example: "Hello" takes 500ms to type (5 characters × 100ms)
   - Use case: Low-latency not critical (form submissions, comments)

5. **Why Optimistic:**
   - Instant feedback (< 5ms)
   - Works offline (buffer operations, sync later)
   - Industry standard (Google Docs, Notion, Figma all use optimistic)

### Implementation Details

**Optimistic Update Logic:**
```
on_user_types(character, position):
    // 1. Apply locally immediately (optimistic)
    local_document.insert(position, character)
    render(local_document)  // User sees character (~5ms)
    
    // 2. Send to server asynchronously
    operation = {type: "insert", pos: position, char: character}
    send_to_server(operation)
    pending_operations.add(operation)
    
    // 3. Wait for server ACK
    on_server_ack(operation):
        pending_operations.remove(operation)
    
    // 4. Rollback if server rejects
    on_server_reject(operation, reason):
        local_document.delete(position)  // Revert
        render(local_document)
        show_error("Operation rejected: " + reason)
```

*See `pseudocode.md::ClientDocument::apply_optimistic()` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Instant feedback (< 5ms) | ❌ Rollback complexity (rare but must handle) |
| ✅ Offline support | ❌ Temporary inconsistency (before server confirms) |
| ✅ Better UX (feels responsive) | ❌ User confusion if rejected (rare) |

### When to Reconsider

**Use Pessimistic (Server Confirmation) if:**
- Latency not critical (e.g., form submissions, comments)
- Operations expensive/irreversible (e.g., payments, deletes)
- Offline not needed (always-online environment)

**Use Hybrid (Optimistic with Validation) if:**
- Client validates locally first (e.g., permission check cached)
- Rollback only for unexpected failures (e.g., network errors)

---

## 9. Sticky Sessions vs Stateless WebSocket Servers

### The Problem

WebSocket servers maintain connection state (which clients are connected). Should we use:
- **Sticky Sessions**: Route all collaborators on same document to same server
- **Stateless**: Distribute connections evenly, use Redis Pub/Sub for broadcast

### Options Considered

| Feature | Sticky Sessions | Stateless (with Redis Pub/Sub) |
|---------|-----------------|-------------------------------|
| **Broadcast Efficiency** | ✅ In-memory (0 hops) | ⚠️ Redis hop (10ms) |
| **Load Balancing** | ⚠️ Uneven (hot documents) | ✅ Perfectly even |
| **Failure Recovery** | ⚠️ All collaborators disconnect | ✅ Only affected clients disconnect |
| **Scalability** | ✅ Horizontal (add servers) | ✅ Horizontal (add servers) |
| **Complexity** | Low (simple routing) | Moderate (Redis pub/sub) |
| **Latency** | ✅ < 10ms (in-memory) | ⚠️ 10-20ms (Redis roundtrip) |

### Decision Made

**Sticky Sessions (with Redis Pub/Sub for cross-server broadcast)**

### Rationale

1. **Broadcast Efficiency (Most Common Case):**
   - **Sticky**: All collaborators on same server → in-memory broadcast (0 hops)
   - **Stateless**: Every broadcast goes through Redis → Redis roundtrip (10ms)
   - **Benchmark**: 95% of broadcasts in-memory (sticky), 5% cross-server (Redis)
   - **Winner**: Sticky (lower latency for most broadcasts)

2. **Latency:**
   - **Sticky**: < 10ms (in-memory map lookup + WebSocket send)
   - **Stateless**: 10-20ms (Redis publish + subscribe + WebSocket send)
   - **Winner**: Sticky (meets < 100ms propagation requirement easily)

3. **Load Balancing (Potential Issue with Sticky):**
   - **Problem**: Hot document (100+ collaborators) overloads one server
   - **Solution**: Read replicas (primary server handles writes, replicas handle broadcasts)
   - **Winner**: Sticky (with mitigation for hot documents)

4. **Why Not Stateless:**
   - Every broadcast goes through Redis (10-20ms overhead)
   - Example: 1000 operations/sec × 10ms = 10 seconds of latency overhead!
   - Use case: Stateless microservices (not WebSocket servers with high broadcast frequency)

5. **Why Sticky:**
   - In-memory broadcasts (faster)
   - Simpler presence tracking (all collaborators in same server's memory)
   - Lower Redis load (only cross-server broadcasts)

### Implementation Details

**Load Balancer Configuration (HAProxy):**
```
backend websocket_servers
    balance source  # Sticky by client IP + doc_id hash
    hash-type consistent  # Consistent hashing
    server ws1 10.0.0.1:8080 check
    server ws2 10.0.0.2:8080 check
    server ws3 10.0.0.3:8080 check
```

**WebSocket Server (with Redis Pub/Sub for cross-server):**
```
connection_pool = {}  // doc_id → [conn1, conn2, ...]

on_operation(operation):
    doc_id = operation.doc_id
    
    # Broadcast to local connections (in-memory)
    for conn in connection_pool[doc_id]:
        conn.send(operation)
    
    # Publish to Redis (for cross-server broadcast)
    redis.publish(f"doc:{doc_id}:ops", operation)

on_redis_message(doc_id, operation):
    # Broadcast to local connections (from another server)
    for conn in connection_pool[doc_id]:
        conn.send(operation)
```

*See `pseudocode.md::WebSocketServer` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Fast in-memory broadcasts (< 10ms) | ⚠️ Uneven load (hot documents) |
| ✅ Simple presence tracking | ⚠️ Server failure affects all collaborators on that doc |
| ✅ Lower Redis load | ⚠️ Sticky session routing complexity |

### When to Reconsider

**Use Stateless if:**
- Load balancing critical (unpredictable hot documents)
- Server failures common (unreliable infrastructure)
- Broadcast frequency low (e.g., < 10 operations/sec per document)

**Use Hybrid (Sticky + Stateless Fallback) if:**
- Hot documents common (use stateless for docs with 50+ collaborators)
- Best of both worlds (sticky for most docs, stateless for hot docs)

---

## 10. CQRS (Snapshots) vs Pure Event Sourcing

### The Problem

Pure event sourcing stores all events, but loading a document requires replaying all events (slow for documents with millions of edits). Should we:
- **Pure Event Sourcing**: Replay all events on every load
- **CQRS (Snapshots)**: Periodically save snapshots, replay only events since snapshot

### Options Considered

| Feature | Pure Event Sourcing | CQRS (Snapshots) |
|---------|-------------------|------------------|
| **Document Load Latency** | ❌ High (replay millions of events = minutes) | ✅ Low (load snapshot + replay < 100 events = 50ms) |
| **Storage Cost** | Lower (only events) | Higher (events + snapshots) |
| **Complexity** | Low (simple replay) | Moderate (snapshot management) |
| **Consistency** | ✅ Always correct (replay is source of truth) | ⚠️ Snapshot might be stale (need validation) |
| **Version Control** | ✅ Perfect (any point in time) | ✅ Perfect (snapshots + events) |

### Decision Made

**CQRS (Command Query Responsibility Segregation) with Periodic Snapshots**

- **Write Model**: Event Sourcing (append-only event log)
- **Read Model**: Snapshots (saved every 100 operations)

### Rationale

1. **Document Load Latency (Critical):**
   - **Pure Event Sourcing**: Document with 1M events = replay 1M operations = minutes!
   - **CQRS**: Load snapshot at event 999,900 + replay 100 events = 50-100ms
   - **Winner**: CQRS (meets < 500ms document load requirement)

2. **Storage Cost (Acceptable):**
   - **Events**: 1 PB (10B documents × 100 versions × 1 KB)
   - **Snapshots**: 500 GB (10B documents × 50 KB, snapshot every 100 ops)
   - **Total**: 1.0005 PB (0.05% overhead)
   - **Winner**: CQRS (storage cost negligible)

3. **Complexity (Manageable):**
   - **Snapshot Generation**: Every 100 operations, save snapshot (async, non-blocking)
   - **Snapshot GC**: Delete old snapshots (keep last 10 per document)
   - **Winner**: CQRS (complexity worth the performance gain)

4. **Why Not Pure Event Sourcing:**
   - Unacceptable load latency (minutes for large documents)
   - Example: Document with 1M edits (popular blog post, years of history)
   - Use case: Small documents (< 1000 events), prototypes

5. **Why CQRS:**
   - Fast document load (< 100ms even for 1M events)
   - Minimal storage overhead (0.05%)
   - Industry standard (Google Docs, Notion, Figma all use snapshots)

### Implementation Details

**Snapshot Generation Logic:**
```
function on_operation(operation):
    apply_operation(operation)
    persist_to_event_log(operation)
    
    event_count_since_snapshot += 1
    
    if event_count_since_snapshot >= 100:
        snapshot = generate_snapshot()
        save_snapshot(snapshot)  // Redis + PostgreSQL
        event_count_since_snapshot = 0
```

**Document Load with Snapshot:**
```
function load_document(doc_id):
    # 1. Try Redis cache (hot)
    snapshot = redis.get(f"doc:{doc_id}:snapshot")
    if snapshot:
        return snapshot  // < 10ms
    
    # 2. Load latest snapshot from PostgreSQL (warm)
    snapshot = db.query("SELECT * FROM document_snapshots WHERE doc_id = ? ORDER BY version DESC LIMIT 1", doc_id)
    
    # 3. Fetch events since snapshot
    events = db.query("SELECT * FROM document_events WHERE doc_id = ? AND event_id > ?", 
                      doc_id, snapshot.version)
    
    # 4. Replay events (typically < 100)
    document = snapshot.content
    for event in events:
        document = apply(document, event)
    
    # 5. Cache result
    redis.set(f"doc:{doc_id}:snapshot", document, ttl=3600)
    
    return document
```

*See `pseudocode.md::SnapshotManager` for complete implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Fast document load (< 100ms) | ❌ Storage cost (0.05% overhead for snapshots) |
| ✅ Scalability (millions of events OK) | ❌ Snapshot management complexity (generation, GC) |
| ✅ Cache-friendly (snapshots cached) | ❌ Snapshot staleness (need to replay events since snapshot) |

### When to Reconsider

**Use Pure Event Sourcing if:**
- Documents small (< 1000 events, load latency < 100ms without snapshots)
- Storage cost critical (every byte counts)
- Simplicity preferred (no snapshot management)

**Use Hybrid (Snapshots for Large Docs Only) if:**
- Most documents small (< 1000 events)
- Only large documents need snapshots (e.g., > 10k events)
- Reduces snapshot storage cost