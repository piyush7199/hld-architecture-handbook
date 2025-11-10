# Stock Brokerage Platform - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A highly scalable, low-latency stock brokerage platform like Zerodha or Groww that enables 10 million concurrent users
to view real-time stock quotes with <200ms latency, search and discover stocks instantly, place buy/sell orders with
sub-second execution, manage portfolios with ACID-compliant ledger integrity, and access personalized market watch and
holdings.

The system must handle 100,000 market data updates/sec from exchanges and broadcast them to 200 million concurrent
WebSocket streams while maintaining financial data consistency.

**Key Characteristics:**

- **Real-time quotes** - <200ms latency for market data updates
- **Low-latency order entry** - <1 second order placement
- **Strong consistency** - ACID guarantees for financial ledger
- **High availability** - 99.99% uptime for quotes
- **High throughput** - 100k updates/sec from exchanges
- **Scalability** - 10M concurrent users, 200M WebSocket streams

---

## Requirements & Scale

### Functional Requirements

| Requirement                 | Description                                                 | Priority    |
|-----------------------------|-------------------------------------------------------------|-------------|
| **Low-Latency Order Entry** | Allow users to place Buy/Sell orders instantly (<1s)        | Must Have   |
| **Real-Time Quotes**        | Display live stock prices with <200ms latency               | Must Have   |
| **Search & Discovery**      | Search any ticker with instant results (<50ms)              | Must Have   |
| **Account Management**      | Manage balances, holdings, risk limits with ACID guarantees | Must Have   |
| **Market Watch**            | Personalized list of watched stocks with real-time updates  | Must Have   |
| **Order History**           | View complete order and trade history                       | Must Have   |
| **Portfolio Analytics**     | Display P&L, holdings breakdown, performance charts         | Should Have |
| **Alerts & Notifications**  | Price alerts, order fills, margin calls                     | Should Have |

### Non-Functional Requirements

| Requirement                     | Target               | Rationale                                  |
|---------------------------------|----------------------|--------------------------------------------|
| **Low Latency (Quotes)**        | < 200 ms (p99)       | Real-time market data critical for trading |
| **Low Latency (Orders)**        | < 1 second           | Order execution speed impacts user trust   |
| **Strong Consistency (Ledger)** | ACID guarantees      | Financial integrity non-negotiable         |
| **High Availability (Quotes)**  | 99.99% uptime        | Users need continuous market access        |
| **High Throughput (Data)**      | 100k updates/sec     | Peak market activity (9:15-9:30 AM)        |
| **Scalability**                 | 10M concurrent users | Support growing user base                  |

### Scale Estimation

| Metric                   | Assumption                                           | Calculation                                            | Result                                      |
|--------------------------|------------------------------------------------------|--------------------------------------------------------|---------------------------------------------|
| **Concurrent Users**     | 10 Million active users                              | -                                                      | -                                           |
| **Market Data QPS (In)** | 100,000 updates/sec from Exchange                    | -                                                      | High volume streaming input                 |
| **Quote Pushes (Out)**   | $10 \text{M users} \times 20 \text{ watched stocks}$ | -                                                      | $200 \text{M concurrent WebSocket streams}$ |
| **Order Entry QPS**      | 1,000 orders/sec (peak)                              | -                                                      | 1,000 QPS to Matching Engine                |
| **Search QPS**           | $10 \text{M users} \times 5 \text{ searches/day}$    | $\frac{50 \text{M}}{24 \text{h} \times 3600 \text{s}}$ | $\sim 580$ searches per second              |
| **Ledger Transactions**  | 1,000 trades/sec + 5,000 balance checks/sec          | -                                                      | 6,000 DB transactions per second            |

**Key Insight:** The primary bottleneck is **broadcast fanout** (100k updates/sec → 200M streams = 20 trillion
messages/sec if naive implementation).

---

## Key Components

| Component                        | Responsibility                            | Technology Options                     | Scalability              |
|----------------------------------|-------------------------------------------|----------------------------------------|--------------------------|
| **Market Data Ingestor**         | Receive data from Exchange (FIX protocol) | Go, Java (FIX libraries)               | Horizontal               |
| **Quote Cache (Redis)**          | Store latest L1 quotes (LTP, Bid/Ask)     | Redis Cluster                          | Horizontal (sharding)    |
| **WebSocket Broadcast Layer**    | Push live quotes to connected users       | Node.js, Go, Elixir (Phoenix Channels) | Horizontal (partitioned) |
| **Kafka Message Bus**            | Distribute market data to WebSocket nodes | Apache Kafka                           | Horizontal (partitions)  |
| **Order Entry Gateway**          | Validate and route orders to Exchange     | gRPC service (Go/Java)                 | Horizontal               |
| **Search Index (Elasticsearch)** | Fast ticker search and discovery          | Elasticsearch                          | Horizontal (sharding)    |
| **Account Ledger (PostgreSQL)**  | Store balances, holdings, trades (ACID)   | PostgreSQL (sharded by user_id)        | Horizontal (sharding)    |
| **Event Store (Kafka)**          | Immutable log of all ledger events        | Kafka (compacted topics)               | Horizontal (partitions)  |

---

## Real-Time Market Data Flow

**Problem:** Broadcast 100,000 updates/sec from Exchange to 200 million WebSocket connections.

**Scalable Solution: Multi-Level Pub/Sub Hierarchy**

**Step 1: Centralized Ingestion**

- **Market Data Ingestor** receives FIX protocol messages from Exchange
- Parses quotes and publishes to Kafka topic `market.quotes` (partitioned by symbol)
- Updates Redis quote cache asynchronously

**Step 2: Kafka as Distribution Layer**

- Kafka topic has 100 partitions (load balanced)
- Each partition handles ~1,000 symbols
- **WebSocket Server Fleet** (1,000 nodes) consumes from Kafka

**Step 3: Selective Fanout at WebSocket Layer**

- Each WebSocket server maintains **in-memory subscription map:**
  ```
  {
    "AAPL": [user123, user456, user789],  // 2,000 users on this node watching AAPL
    "GOOGL": [user234, user567]
  }
  ```
- When Kafka delivers `AAPL` update, server pushes ONLY to 2,000 subscribed users on that node
- **Fanout per node:** 100 updates/sec × 2,000 users = 200k messages/sec per node (manageable)

**Total Architecture:**

```
Exchange (100k updates/sec)
  ↓
Market Data Ingestor (1 node)
  ↓
Kafka (100 partitions)
  ↓
WebSocket Servers (1,000 nodes, each handles 200k users)
  ↓
Users (10M total, 200M subscriptions)
```

**Performance:**

- Ingestion latency: <10ms
- Kafka propagation: <20ms
- WebSocket push: <50ms
- **End-to-end: <100ms** (within 200ms SLA)

---

## Order Entry and Execution Flow

**Requirements:**

- Sub-second order placement
- ACID transaction (deduct balance + place order atomically)
- Route order to Exchange Matching Engine

**Technical Flow:**

**Step 1: Validation (100ms)**

- User → Order Entry API (gRPC)
- API validates:
    - Sufficient balance (fetch from PostgreSQL)
    - Risk limits (position size, margin)
    - Market hours (9:15 AM - 3:30 PM IST)

**Step 2: Atomic Transaction (200ms)**

```sql
BEGIN TRANSACTION;

-- Lock user account (prevent concurrent order placement)
SELECT cash_balance FROM accounts WHERE user_id = 12345 FOR UPDATE;

-- Check sufficient balance
IF cash_balance < (order.quantity * order.price):
  ROLLBACK;
  RETURN "Insufficient balance";

-- Deduct balance (block for pending order)
UPDATE accounts SET margin_used = margin_used + (order.quantity * order.price) WHERE user_id = 12345;

-- Insert order
INSERT INTO orders (user_id, symbol, side, quantity, price, status) VALUES (...);

COMMIT;
```

**Step 3: Route to Exchange (300ms)**

- Order Entry API → Exchange Gateway (FIX protocol)
- Exchange Matching Engine processes order
- Fill notification via FIX
- Update order status: PENDING → FILLED

**Step 4: Ledger Update (Async)**

- Kafka Event: {"type": "ORDER_FILLED", "user_id": 12345, "symbol": "AAPL", "quantity": 10, "price": 150.25}
- Ledger Service consumes event
- Update holdings table (add 10 shares of AAPL)
- Release margin_used, deduct cash_balance

**Total Latency:** 100ms (validation) + 200ms (DB transaction) + 300ms (exchange) = **600ms** (within 1s SLA)

---

## Event Sourcing for Account Ledger

**Problem:** High contention on `cash_balance` field (thousands of concurrent orders updating same field).

**Event Sourcing Approach (Lock-Free, Fast):**

**Concept:** Instead of updating balance directly, append immutable events to a log. Balance is derived by replaying
events.

**Event Log (Kafka Compacted Topic):**

```
Event 1: {"user_id": 12345, "type": "DEPOSIT", "amount": 10000, "ts": 1609459200000}
Event 2: {"user_id": 12345, "type": "ORDER_PLACED", "amount": -1500, "ts": 1609459201000}
Event 3: {"user_id": 12345, "type": "ORDER_FILLED", "amount": 0, "ts": 1609459202000}
Event 4: {"user_id": 12345, "type": "HOLDING_ADDED", "symbol": "AAPL", "quantity": 10, "ts": 1609459203000}
```

**Balance Calculation:**

- Replay all events for user_id = 12345
- Balance = 10000 - 1500 = 8500 (after ORDER_PLACED)
- Holdings = [{"symbol": "AAPL", "quantity": 10}]

**Benefits:**

- **Lock-free writes:** Append-only log (no row-level locks)
- **Audit trail:** Complete history of all transactions
- **Time travel:** Query balance at any point in time
- **Scalability:** Kafka handles 1M+ writes/sec (vs PostgreSQL 6k TPS)

**Materialized View (PostgreSQL):**

- Periodically snapshot balance from event log (every 1 minute)
- Fast reads: `SELECT cash_balance FROM accounts WHERE user_id = 12345` (O(1))
- Trade-off: 1-minute staleness (acceptable for retail trading)

---

## WebSocket Broadcast Architecture

**Challenge:** Maintain 200 million concurrent WebSocket connections with <200ms latency.

### WebSocket Server Topology

**Cluster Configuration:**

- **1,000 WebSocket servers** (horizontally scaled)
- Each server handles **10,000 concurrent connections**
- Total capacity: 10 million concurrent users

**Connection Routing:**

- **Load Balancer (Layer 7):** Routes initial WebSocket handshake based on `user_id` hash
- **Sticky Sessions:** Ensures user always connects to same server (maintains subscription state)

**Server Architecture (Per Node):**

```
Node.js (Single-Threaded Event Loop)
  ├── 10,000 WebSocket connections (in-memory)
  ├── Subscription Map: {"AAPL": [user1, user2, ...], "GOOGL": [...]}
  ├── Kafka Consumer (subscribes to market.quotes)
  └── Redis Client (fetch initial quote snapshot)
```

**Memory Footprint:**

- 10,000 connections × 50 KB/connection = 500 MB per server
- Subscription map: ~10 MB (assuming 20 watched stocks/user)
- **Total:** ~600 MB per server (fits in 8 GB RAM)

### Subscription Protocol

**Client → Server Messages:**

```json
{
  "action": "subscribe",
  "symbols": [
    "AAPL",
    "GOOGL",
    "TSLA"
  ]
}
```

**Server → Client Messages:**

```json
{
  "symbol": "AAPL",
  "ltp": 150.25,
  "change":
  +
  1.5,
  "timestamp": 1609459200000
}
```

**Heartbeat (Keepalive):**

- Client sends PING every 30 seconds
- Server responds with PONG
- If no PING for 60 seconds → disconnect (free resources)

### Failover and Redundancy

**Problem:** If a WebSocket server crashes, 10,000 users lose connection.

**Solution: Graceful Failover**

1. **Health Checks:** Load balancer pings servers every 5 seconds
2. **Failure Detection:** If server doesn't respond for 15 seconds → mark unhealthy
3. **Client Reconnect:** Clients detect disconnection, reconnect to new server (exponential backoff)
4. **Resubscription:** Client sends `subscribe` message with saved watchlist

**Stateless Design:**

- WebSocket servers store NO persistent state (subscription map is in-memory only)
- Clients maintain watchlist locally (localStorage)
- Reconnection rebuilds subscription map from scratch

**Downtime:** <5 seconds (reconnect + resubscribe)

---

## Bottlenecks & Solutions

### Bottleneck 1: Quote Broadcast Fanout (200M streams)

**Problem:** Growing user base increases fanout exponentially.

**Mitigation 1: Coalescing**

- Instead of pushing every single update, batch updates every 100ms
- Reduces 100k updates/sec → 1k updates/sec (100× reduction)
- Trade-off: Slightly stale data (acceptable for retail traders, not HFT)

**Mitigation 2: Differential Updates**

- Only push quote if price changed by >0.1% (filter noise)
- Reduces updates by 50% (many quotes don't change significantly)

**Mitigation 3: CDN-Based Pub/Sub**

- Use AWS IoT Core or Pusher (managed WebSocket service)
- Offload fanout to CDN infrastructure
- Cost: ~$0.001 per million messages = $200k/month for 200M streams

### Bottleneck 2: Ledger Database Contention (6k TPS)

**Problem:** High-frequency trading users place 100+ orders/sec, contending on same account row.

**Mitigation: Full Event Sourcing Migration**

- Current: Hybrid (events + materialized view)
- Future: Pure event sourcing (no snapshot, derive balance on-demand)
- Use Kafka Streams to compute balance in real-time from event log
- Eliminates PostgreSQL bottleneck entirely

**Performance:**

- Kafka: 1M+ writes/sec (100× current throughput)
- Kafka Streams: Sub-second balance computation

**Trade-off:**

- More complex (event replay logic)
- Higher operational cost (Kafka cluster)

### Bottleneck 3: Search Index Staleness

**Problem:** Newly listed stocks don't appear in search until next day.

**Solution: Real-Time Index Updates**

- Publish new listings to Kafka topic `stock.listings`
- Elasticsearch consumer updates index in real-time (<1 second lag)
- Trade-off: Slightly higher write load on Elasticsearch

---

## Common Anti-Patterns to Avoid

### 1. Polling for Quotes (Instead of WebSockets)

❌ **Bad:** Client polls API every 1 second for quote updates

**Why It's Bad:**

- 10M users × 20 stocks × 1 poll/sec = **200M HTTP requests/sec** (crushes API)
- Higher latency (1-second polling interval)
- Wastes bandwidth (fetches unchanged quotes)

✅ **Good:** Use WebSockets Push Model

- Server pushes updates only when price changes
- Reduces requests from 200M/sec → ~100k/sec (2000× reduction)
- Lower latency (<200ms)

### 2. Synchronous Order Placement (Blocking API)

❌ **Bad:** Order Entry API waits for Exchange fill confirmation before returning to user

**Why It's Bad:**

- Exchange fill can take 5-10 seconds (market order during volatility)
- Blocks user's UI thread (bad UX)
- Ties up API server resources (cannot handle other requests)

✅ **Good:** Async Order Placement with Status Updates

- API returns immediately with `order_id` and status `PENDING`
- User can continue using app
- WebSocket pushes status update when order fills: `{"order_id": 123, "status": "FILLED"}`

### 3. Storing Quotes in PostgreSQL (Slow Writes)

❌ **Bad:** Writing 100k quote updates/sec to PostgreSQL `quotes` table

**Why It's Bad:**

- PostgreSQL write throughput: ~10k TPS (bottleneck)
- Disk I/O becomes saturated
- Slows down critical ACID transactions (orders, ledger)

✅ **Good:** Use Redis for Quotes (In-Memory)

- Redis write throughput: 100k+ TPS (10× PostgreSQL)
- Sub-millisecond reads
- Reserve PostgreSQL for ACID-critical data only (ledger, orders)

---

## Monitoring & Observability

### Key Metrics

**System Metrics:**

- **WebSocket connections:** 10M concurrent (target)
- **Quote broadcast latency:** <200ms p99 (target)
- **Order placement latency:** <1 second p99 (target)
- **Kafka consumer lag:** <1000 messages (target)
- **Redis memory usage:** <80% (target)
- **PostgreSQL connection pool:** <80% utilization (target)

**Business Metrics:**

- **Orders placed:** 1,000 orders/sec (peak)
- **Trades executed:** 1,000 trades/sec (peak)
- **Active users:** 10M concurrent
- **Market data updates:** 100k updates/sec

### Dashboards

1. **Real-Time Operations Dashboard**
    - WebSocket connections (by server)
    - Quote broadcast latency (p50, p95, p99)
    - Order placement latency (p50, p95, p99)
    - Kafka consumer lag (by partition)

2. **Financial Dashboard**
    - Orders placed (by status: PENDING, FILLED, CANCELLED)
    - Trades executed (by symbol, by user)
    - Account balances (total, average)
    - Holdings distribution

3. **System Health Dashboard**
    - Redis memory usage (by shard)
    - PostgreSQL connection pool (by shard)
    - Elasticsearch query latency
    - WebSocket server health (CPU, memory)

### Alerts

**Critical Alerts (PagerDuty, 24/7 on-call):**

- WebSocket server down (10k users disconnected)
- Quote broadcast latency >500ms for 5 minutes
- Order placement latency >2 seconds for 5 minutes
- PostgreSQL connection pool exhausted

**Warning Alerts (Slack, email):**

- Kafka consumer lag >10K messages
- Redis memory usage >90%
- Elasticsearch query latency >100ms
- WebSocket server CPU >80%

---

## Trade-offs Summary

| What We Gain                                | What We Sacrifice                                                   |
|---------------------------------------------|---------------------------------------------------------------------|
| ✅ **Low latency** (<200ms quotes)           | ❌ High infrastructure cost (1000 WebSocket servers, Redis cluster)  |
| ✅ **Strong consistency** (ACID ledger)      | ❌ Lower write throughput (6k TPS vs NoSQL 100k TPS)                 |
| ✅ **Scalability** (10M users, 200M streams) | ❌ Complex architecture (Kafka, Event Sourcing, multi-level pub/sub) |
| ✅ **Real-time** (WebSockets push)           | ❌ Stateful servers (harder to scale, failover complexity)           |
| ✅ **Fast search** (Elasticsearch)           | ❌ Eventual consistency (search index lags behind by seconds)        |

---

## Real-World Examples

| Company                 | Use Case                       | Key Techniques                                                      |
|-------------------------|--------------------------------|---------------------------------------------------------------------|
| **Zerodha**             | Retail stock brokerage (India) | WebSockets for quotes, Redis cache, Event Sourcing for ledger       |
| **Robinhood**           | Commission-free trading (US)   | Real-time quotes via WebSockets, PostgreSQL ledger, Kafka event bus |
| **ETRADE**              | Online brokerage (US)          | FIX protocol integration, multi-level pub/sub, ACID ledger          |
| **Interactive Brokers** | Professional trading platform  | Low-latency order routing, TWS API, multi-asset support             |
| **Groww**               | Mutual funds + stocks (India)  | Simplified UX, Redis for caching, PostgreSQL for transactions       |

---

## Key Takeaways

1. **Multi-level pub/sub** reduces fanout from 20 trillion to 200k messages/sec per node
2. **WebSocket broadcast** enables real-time quotes with <200ms latency
3. **Event sourcing** eliminates database contention for high-frequency trading
4. **Kafka distribution** decouples market data ingestion from WebSocket broadcast
5. **Redis caching** provides sub-millisecond quote reads
6. **PostgreSQL ACID** ensures financial integrity for ledger
7. **Elasticsearch** enables instant stock search (<50ms)
8. **Sticky sessions** maintain WebSocket connection state
9. **Async order placement** improves UX (non-blocking)
10. **Graceful failover** minimizes downtime (<5 seconds)

---

## Recommended Stack

- **Market Data:** FIX protocol, Kafka (100 partitions)
- **Quote Cache:** Redis Cluster (sharded by symbol)
- **WebSocket Broadcast:** Node.js/Go (1,000 servers, 10k connections each)
- **Order Entry:** gRPC (Go/Java), PostgreSQL (ACID transactions)
- **Search:** Elasticsearch (sharded by symbol prefix)
- **Event Store:** Kafka (compacted topics for event sourcing)
- **Ledger:** PostgreSQL (sharded by user_id) + Kafka (event log)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, WebSocket topology, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (order entry, quote subscription,
  search)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (quote handling, ledger updates)

