# 3.4.5 Design a Stock Brokerage Platform (Zerodha/Groww)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, real-time data flow, WebSocket topology
- **[Sequence Diagrams](./sequence-diagrams.md)** - Order entry, quote subscription, search flows, ledger updates
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of WebSockets, Event Sourcing,
  protocol choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for quote handling and ledger

---

## Problem Statement

Design a highly scalable, low-latency stock brokerage platform like Zerodha or Groww that enables 10 million concurrent
users to view real-time stock quotes (<200ms latency), search and discover stocks instantly, place buy/sell orders with
sub-second execution, and manage portfolios with ACID-compliant ledger integrity.

The system must handle 100,000 market data updates/sec from exchanges and broadcast them to 200 million concurrent
WebSocket streams while maintaining financial data consistency and regulatory compliance.

**Core Challenges:**

- **Broadcast Fanout:** 100k updates/sec → 200M WebSocket streams (20 trillion messages/sec if naive)
- **Low Latency:** <200ms end-to-end (exchange → user's screen)
- **Strong Consistency:** ACID transactions for financial ledger (no double spending)
- **High Availability:** 99.99% uptime for real-time quote system

---

## Requirements and Scale Estimation

### Functional Requirements

| Requirement                 | Description                                                 |
|-----------------------------|-------------------------------------------------------------|
| **Low-Latency Order Entry** | Place Buy/Sell orders instantly (<1s)                       |
| **Real-Time Quotes**        | Display live stock prices (<200ms latency)                  |
| **Search & Discovery**      | Search any ticker with instant results (<50ms)              |
| **Account Management**      | Manage balances, holdings, risk limits with ACID guarantees |
| **Market Watch**            | Personalized list of watched stocks with real-time updates  |
| **Portfolio Analytics**     | Display P&L, holdings breakdown, performance charts         |

### Non-Functional Requirements

| Requirement              | Target               | Rationale                                  |
|--------------------------|----------------------|--------------------------------------------|
| **Low Latency (Quotes)** | < 200 ms (p99)       | Real-time market data critical for trading |
| **Low Latency (Orders)** | < 1 second           | Order execution speed impacts user trust   |
| **Strong Consistency**   | ACID guarantees      | Financial integrity non-negotiable         |
| **High Availability**    | 99.99% uptime        | Users need continuous market access        |
| **Scalability**          | 10M concurrent users | Support growing user base                  |

### Scale Estimation

| Metric                   | Calculation                                    | Result                                      |
|--------------------------|------------------------------------------------|---------------------------------------------|
| **Concurrent Users**     | 10 Million active users                        | -                                           |
| **Market Data QPS (In)** | 100,000 updates/sec from Exchange              | High volume streaming input                 |
| **Quote Pushes (Out)**   | $10 \text{M} \times 20 \text{ watched stocks}$ | $200 \text{M concurrent WebSocket streams}$ |
| **Order Entry QPS**      | 1,000 orders/sec (peak)                        | 1,000 QPS to Matching Engine                |
| **Search QPS**           | $50 \text{M searches} / (24 \times 3600)$      | $\sim 580$ searches per second              |
| **Ledger Transactions**  | 1,000 trades/sec + 5,000 balance checks/sec    | 6,000 DB transactions per second            |

**Key Insight:** Primary bottleneck is **broadcast fanout** (100k updates/sec → 200M streams).

---

## High-Level Architecture

The architecture splits into three independent flows:

1. **Real-Time Market Data Flow:** Exchange → Ingestion → Redis Cache → WebSocket Broadcast
2. **User Interaction Flow:** Search, Market Watch, Portfolio Display
3. **Transactional Flow:** Order Entry → Matching Engine → Ledger Update

### Core Components

| Component                     | Responsibility                            | Technology                      | Scalability              |
|-------------------------------|-------------------------------------------|---------------------------------|--------------------------|
| **Market Data Ingestor**      | Receive data from Exchange (FIX protocol) | Go, Java (FIX libraries)        | Horizontal               |
| **Quote Cache (Redis)**       | Store latest L1 quotes (LTP, Bid/Ask)     | Redis Cluster                   | Horizontal (sharding)    |
| **WebSocket Broadcast Layer** | Push live quotes to connected users       | Node.js, Go, Elixir             | Horizontal (partitioned) |
| **Kafka Message Bus**         | Distribute market data                    | Apache Kafka                    | Horizontal (partitions)  |
| **Order Entry Gateway**       | Validate and route orders                 | gRPC service (Go/Java)          | Horizontal               |
| **Search Index**              | Fast ticker search                        | Elasticsearch                   | Horizontal (sharding)    |
| **Account Ledger**            | Store balances, holdings (ACID)           | PostgreSQL (sharded by user_id) | Horizontal (sharding)    |

---

## Data Model

### Quote Data (Redis - In-Memory)

**Key-Value Schema:**

| Key                   | Value (JSON)                                                       | TTL    |
|-----------------------|--------------------------------------------------------------------|--------|
| `quote:{symbol}`      | `{"ltp": 150.25, "bid": 150.20, "ask": 150.30, "volume": 1M}`      | None   |
| `ohlc:{symbol}:{day}` | `{"open": 148.50, "high": 151.00, "low": 147.00, "close": 150.25}` | 7 days |

**Why Redis?**

- Sub-millisecond reads (<1ms) for 200M concurrent streams
- High write throughput (100k updates/sec)
- In-memory ensures consistent low latency

---

### User Account Ledger (PostgreSQL - ACID)

**Table: `accounts`**

| Field          | Type            | Notes                      |
|----------------|-----------------|----------------------------|
| `user_id`      | `BIGINT`        | Primary key, shard key     |
| `cash_balance` | `DECIMAL(15,2)` | Available cash             |
| `margin_used`  | `DECIMAL(15,2)` | Blocked for open positions |
| `total_value`  | `DECIMAL(15,2)` | Cash + holdings value      |

**Table: `holdings`**

| Field       | Type            | Notes                  |
|-------------|-----------------|------------------------|
| `user_id`   | `BIGINT`        | Foreign key, shard key |
| `symbol`    | `VARCHAR(20)`   | Stock ticker           |
| `quantity`  | `INT`           | Number of shares       |
| `avg_price` | `DECIMAL(10,2)` | Average purchase price |

**Table: `orders`**

| Field      | Type            | Notes                         |
|------------|-----------------|-------------------------------|
| `order_id` | `BIGINT`        | Primary key (Snowflake ID)    |
| `user_id`  | `BIGINT`        | Foreign key, shard key        |
| `symbol`   | `VARCHAR(20)`   | Stock ticker                  |
| `side`     | `ENUM`          | BUY, SELL                     |
| `quantity` | `INT`           | Order size                    |
| `price`    | `DECIMAL(10,2)` | Limit price (null for market) |
| `status`   | `ENUM`          | PENDING, FILLED, CANCELLED    |

**Why PostgreSQL?**

- Financial data requires strict ACID consistency
- Transactions ensure atomicity (order + balance update atomic)
- Sharding by `user_id` keeps user data co-located

---

### Search Index (Elasticsearch)

**Document Schema:**

```json
{
  "symbol": "AAPL",
  "name": "Apple Inc.",
  "exchange": "NASDAQ",
  "sector": "Technology",
  "market_cap": 2800000000000
}
```

**Why Elasticsearch?**

- Built for fast full-text and prefix search (<50ms)
- Handles 100k+ documents with sub-second queries
- PostgreSQL LIKE queries cannot match this performance

---

## Detailed Component Design

### Real-Time Market Data Flow

**Problem:** Broadcast 100,000 updates/sec from Exchange to 200 million WebSocket connections.

**Naive Approach (Does NOT Scale):**

- Each update triggers 2,000 user notifications (if 2,000 users watch each stock)
- **Total:** 100k × 2,000 = **200M messages/sec** (impossible)

**Scalable Solution: Multi-Level Pub/Sub Hierarchy**

**Architecture:**

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

**How It Works:**

**Step 1: Centralized Ingestion**

- Market Data Ingestor receives FIX protocol messages from Exchange
- Parses quotes and publishes to Kafka topic `market.quotes` (partitioned by symbol)
- Updates Redis quote cache asynchronously

**Step 2: Kafka Distribution**

- Kafka topic has 100 partitions (load balanced)
- WebSocket Server Fleet (1,000 nodes) consumes from Kafka

**Step 3: Selective Fanout**

- Each WebSocket server maintains in-memory subscription map:
  ```
  {
    "AAPL": [user123, user456, user789],  // 2,000 users on this node watching AAPL
    "GOOGL": [user234, user567]
  }
  ```
- When Kafka delivers `AAPL` update, server pushes ONLY to 2,000 subscribed users on that node
- **Fanout per node:** 100 updates/sec × 2,000 users = 200k messages/sec per node (manageable)

**Performance:**

- Ingestion latency: <10ms
- Kafka propagation: <20ms
- WebSocket push: <50ms
- **End-to-end: <100ms** (within 200ms SLA)

*See pseudocode.md::handle_market_data_update() for implementation.*

---

### Search and Discovery Flow

**User Experience:**

1. User types "AP" in search box
2. UI sends request to Search API
3. Returns top 10 matches in <50ms
4. UI displays: AAPL (Apple Inc.), APPL (AppLovin), etc.
5. For each result, UI fetches current price from Quote API (Redis)
6. User clicks AAPL → WebSocket subscribes to AAPL quote stream

**Technical Flow:**

```
User → Search API → Elasticsearch (prefix query: "AP*") → [AAPL, APPL, ...]
  ↓
Quote API → Redis (MGET quote:AAPL quote:APPL) → [{ltp: 150.25}, {ltp: 75.30}]
  ↓
UI (display search results with live prices)
```

**Optimization:**

- **Caching:** Cache popular queries in Redis (1-minute TTL)
- **Debouncing:** Wait 300ms after last keystroke before querying
- **Pagination:** Return top 10, fetch more on scroll

*See pseudocode.md::search_and_hydrate() for implementation.*

---

### Order Entry and Execution Flow

**Requirements:**

- Sub-second order placement
- ACID transaction (deduct balance + place order atomically)
- Route order to Exchange Matching Engine

**Technical Flow:**

**Step 1: Validation (100ms)**

```
User → Order Entry API (gRPC)
API validates:
  - Sufficient balance (fetch from PostgreSQL)
  - Risk limits (position size, margin)
  - Market hours (9:15 AM - 3:30 PM IST)
```

**Step 2: Atomic Transaction (200ms)**

```sql
BEGIN TRANSACTION;

SELECT cash_balance FROM accounts WHERE user_id = 12345 FOR UPDATE;

IF cash_balance < (order.quantity * order.price):
  ROLLBACK;
  RETURN "Insufficient balance";

UPDATE accounts SET margin_used = margin_used + (order.quantity * order.price) WHERE user_id = 12345;

INSERT INTO orders (user_id, symbol, side, quantity, price, status) VALUES (...);

COMMIT;
```

**Step 3: Route to Exchange (300ms)**

```
Order Entry API → Exchange Gateway (FIX protocol)
Exchange Matching Engine → Fill notification via FIX
Update order status: PENDING → FILLED
```

**Step 4: Ledger Update (Async)**

```
Kafka Event: ORDER_FILLED
Ledger Service consumes event
Update holdings table (add shares)
Release margin, deduct cash
```

**Total Latency:** 100ms + 200ms + 300ms = **600ms** (within 1s SLA)

*See pseudocode.md::place_order() for implementation.*

---

### Event Sourcing for Account Ledger

**Problem:** High contention on `cash_balance` field (thousands of concurrent orders).

**Traditional Approach (Locks, Slow):**

```sql
UPDATE accounts SET cash_balance = cash_balance - 1500 WHERE user_id = 12345;
-- Requires exclusive lock, serializes all updates
```

**Event Sourcing Approach (Lock-Free, Fast):**

**Concept:** Instead of updating balance directly, append immutable events to a log. Balance is derived by replaying
events.

**Event Log (Kafka Compacted Topic):**

```
Event 1: {user_id: 12345, type: DEPOSIT, amount: 10000, ts: 1609459200000}
Event 2: {user_id: 12345, type: ORDER_PLACED, amount: -1500, ts: 1609459201000}
Event 3: {user_id: 12345, type: ORDER_FILLED, amount: 0, ts: 1609459202000}
```

**Balance Calculation:**

```
Current balance = SUM(all DEPOSIT/WITHDRAWAL events) - SUM(margin_used from pending orders)
  = 10000 - 1500 = 8500
```

**Materialized View (PostgreSQL):**

- Background job replays events every 1 second
- Updates `accounts.cash_balance` (read-optimized snapshot)
- Queries read from snapshot (fast), writes append to event log (fast, no locks)

**Benefits:**

- ✅ No locks (Kafka appending is lock-free)
- ✅ Audit trail (complete history of all transactions)
- ✅ Replayable (reconstruct balance at any point in time)
- ✅ Scalable (Kafka handles 100k+ writes/sec)

**Trade-offs:**

- ❌ Eventual consistency (snapshot lags by ~1 second)
- ❌ Complexity (requires event replay logic)

*See pseudocode.md::append_ledger_event() for implementation.*

---

### FIX Protocol Integration (Exchange Connectivity)

**Problem:** Communicate with stock exchanges (NSE/BSE) for order routing using FIX protocol.

**FIX (Financial Information eXchange) Protocol:**

- Binary messaging protocol for financial transactions
- Session-based: Maintains persistent TCP connection
- Sequence numbers: Every message has unique `MsgSeqNum` for reliability
- Heartbeats: Keepalive messages every 30 seconds

**Key Messages:**

**Logon (Session Start):**

```
Broker → Exchange:
  35=A (Logon)
  49=BROKER_ID (SenderCompID)
  56=NSE (TargetCompID)
  34=1 (MsgSeqNum)
  108=30 (HeartBtInt: 30 seconds)
```

**Place Order:**

```
Broker → Exchange:
  35=D (NewOrderSingle)
  11=ORD_123456 (ClOrdID)
  55=RELIANCE (Symbol)
  54=1 (Side: Buy)
  38=100 (OrderQty)
  40=2 (OrdType: Limit)
  44=2450.00 (Price)
```

**Order Fill Notification:**

```
Exchange → Broker:
  35=8 (ExecutionReport)
  11=ORD_123456
  39=2 (OrdStatus: Filled)
  150=F (ExecType: Trade)
  31=2450.00 (LastPx)
  32=100 (LastQty)
```

**FIX Engine Architecture:**

```
┌────────────────────────────────────────┐
│  FIX Engine (Go/Java)                  │
│  ├── Session Manager                   │
│  ├── Message Parser (FIX 4.2/4.4)     │
│  ├── Sequence Number Manager           │
│  └── Heartbeat Monitor                 │
└────────────────────────────────────────┘
         ↕ TCP (port 5001)
┌────────────────────────────────────────┐
│  NSE/BSE Exchange Gateway              │
└────────────────────────────────────────┘
```

**Performance:**

- Latency: ~100ms (order placed to fill notification)
- Throughput: 1,000 orders/sec per FIX session
- Solution for higher throughput: Multiple parallel FIX sessions

*See pseudocode.md::send_fix_order() for implementation.*

---

### Risk Management System

**Problem:** Prevent users from taking excessive risk.

**1. Margin Trading Risk**

**Scenario:**

```
User account: $10,000 cash
User buys: 100 shares @ $200 = $20,000 total value
Margin used: $10,000 borrowed from broker
```

**Risk:** If stock drops to $130, equity = $13,000 - $10,000 = $3,000 (equity ratio = 23%, below 30% → MARGIN CALL)

**Solution: Margin Requirements**

```
Initial Margin: 50% (user must have 50% of position value)
Maintenance Margin: 30% (triggers margin call if breached)

Margin Call Process:
  1. System detects equity ratio < 30%
  2. Notify user: "Add funds or positions will be liquidated"
  3. Grace period: 1 hour
  4. Auto-liquidate if user doesn't add funds
```

**Real-Time Monitoring:**

```
Every 1 minute (during market hours):
  FOR each user with margin positions:
    position_value = SUM(holding.quantity × current_price)
    equity = position_value - margin_used
    equity_ratio = equity / position_value
    
    IF equity_ratio < 0.30:
      trigger_margin_call(user_id)
```

*See pseudocode.md::check_margin_requirements() for implementation.*

---

**2. Circuit Breakers (Market-Wide)**

**NSE/BSE Circuit Breaker Rules:**

| Level | Decline | Action                           |
|-------|---------|----------------------------------|
| 1     | 10%     | Trading halted for 45 minutes    |
| 2     | 15%     | Trading halted for 1 hour 45 min |
| 3     | 20%     | Trading halted for rest of day   |

**Implementation:**

```
every 1 second:
  current_index = get_nifty_level()
  previous_close = get_previous_close()
  decline_percent = (previous_close - current_index) / previous_close × 100
  
  IF decline_percent >= 20:
    halt_trading_for_day()
  ELSE IF decline_percent >= 15:
    halt_trading(duration=105 minutes)
  ELSE IF decline_percent >= 10:
    halt_trading(duration=45 minutes)
```

---

**3. Fraud Detection**

**Account Takeover Detection:**

```
Trigger: User logs in from new device + new IP + different country

Action:
  1. Block login
  2. Send email: "Suspicious login attempt from New York. Was this you?"
  3. Require additional verification (Email OTP, security question)
```

**Wash Trading Detection:**

```
Pattern: User buys and sells same stock >10 times in single day

Detection:
  Trigger: User completes >10 round-trips of same stock

Action:
  1. Warn user: "Wash trading is prohibited"
  2. Block further trades for 24 hours
  3. Report to exchange
```

*See pseudocode.md::detect_fraud() for implementation.*

---

### Performance Optimization Techniques

**1. WebSocket Message Compression**

**Problem:** Sending 200M messages/sec consumes massive bandwidth.

**Solution: Per-Message Deflate (RFC 7692)**

```
Uncompressed: 62 bytes
Compressed: 28 bytes
Compression ratio: 55% reduction
Bandwidth savings: 6.8 GB/sec saved
Cost savings: ~$50k/month
```

---

**2. Redis Pipeline (Batch Quote Updates)**

**Without Pipelining:**

```
FOR each quote in batch (100 quotes):
  redis.SET(f"quote:{symbol}", quote_data)  // 1ms per SET
Total time: 100ms
```

**With Pipelining:**

```
pipeline = redis.pipeline()
FOR each quote in batch (100 quotes):
  pipeline.SET(f"quote:{symbol}", quote_data)
pipeline.execute()  // Single network call
Total time: 5ms (20x faster)
```

---

**3. Kafka Producer Batching**

**Configuration:**

```
linger.ms = 10  // Wait 10ms to accumulate messages
batch.size = 100KB
compression.type = snappy

Result:
  Instead of 100,000 individual sends/sec
  Send ~1,000 batches/sec (100 messages per batch)
  Network calls: 1k/sec (100x reduction)
```

---

**4. CDN for Historical Charts**

**Problem:** Users request historical OHLC data (1-year chart = 365 data points).

**Solution: Cloudflare CDN**

```
Cache-Control: public, max-age=3600
  - 1 million users request same chart → Only 1 backend request/hour
  - 99.9999% cache hit rate
  
Savings:
  Backend load: 1M requests/hour → 1 request/hour
  Cost: $5k/month (CDN) vs $500k/month (origin servers)
```

---

### Security

**Multi-Factor Authentication:**

**Password + TOTP (Time-Based One-Time Password):**

```
Login Flow:
  1. User enters username + password
  2. Server validates credentials
  3. Server sends: "Enter 6-digit code from authenticator app"
  4. User opens Google Authenticator → sees: 847362
  5. User enters code
  6. Server validates TOTP (30-second time window)
  7. Issue JWT token
```

**TOTP Algorithm:**

```python
def generate_totp(secret, timestamp=None):
  if timestamp is None:
    timestamp = int(time.time())
  
  counter = timestamp // 30  // 30-second time step
  
  hmac_hash = hmac.new(secret.encode(), struct.pack(">Q", counter), hashlib.sha1).digest()
  
  offset = hmac_hash[-1] & 0x0F
  code = struct.unpack(">I", hmac_hash[offset:offset+4])[0]
  code = (code & 0x7FFFFFFF) % 1000000
  
  return f"{code:06d}"
```

---

### Advanced Order Types

**1. Stop-Loss Order**

```
User owns 100 shares RELIANCE @ $100
Current: $120 (profit)
Places: STOP-LOSS SELL 100 @ $110

Implementation:
  Redis: stop_orders:RELIANCE → [{user, qty, trigger_price}]
  Monitor quotes
  When LTP <= $110 → Convert to MARKET SELL
```

**2. Good-Till-Cancelled (GTC)**

```
BUY 100 RELIANCE @ $95 (current $100)
Validity: GTC

Implementation:
  PostgreSQL: status=ACTIVE
  Daily batch checks price
  Auto-cancel after 90 days
```

**3. Iceberg Order**

```
Total: 10,000, Display: 500
Benefits: Prevents market impact
Exchange sees only 500 at a time
```

**4. Bracket Order**

```
Entry: BUY 100 @ $100
Target: SELL 100 @ $110
Stop-Loss: SELL 100 @ $95
Implementation: OCO (One Cancels Other)
```

---

### Multi-Region Deployment

**Regions:**

1. Mumbai (Primary) - NSE/BSE exchanges
2. Singapore (Secondary) - Southeast Asia
3. London (Secondary) - Europe

**Data Residency:**

- SEBI: Indian data in India
- GDPR: EU data in EU
- Solution: Shard by region

**Mumbai Architecture:**

```
├── WebSocket Servers (500)
├── PostgreSQL (Indian users)
├── Redis Cache (all symbols)
├── FIX Engine (NSE/BSE)
└── Kafka Cluster
```

**Cross-Region:**

- Quote data: Mumbai → Singapore (100ms)
- User data: NO replication (compliance)

**Disaster Recovery:**

```
Scenario: Mumbai outage
1. DNS Failover (5 min)
2. Activate Standby FIX (10 min)
3. DB Restoration (30 min)

RTO: 1 hour
RPO: 15 minutes
```

---

### Audit Logging and Compliance

**Requirement:** Maintain immutable audit trail for 7 years (SEBI/SEC).

**What to Log:**

1. Order Events (placement, fills, cancellations)
2. Ledger Events (deposits, withdrawals, trades)
3. Market Snapshots (state at order time)
4. System Events (login, logout)
5. Admin Actions (freezes, margin calls)

**Storage: ClickHouse**

**Schema:**

```sql
CREATE TABLE audit_log (
  event_id UUID,
  user_id BIGINT,
  event_type ENUM('ORDER_PLACED', 'ORDER_FILLED', 'DEPOSIT'),
  timestamp TIMESTAMP,
  order_details JSON,
  market_snapshot JSON,
  ip_address VARCHAR(45)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (user_id, timestamp);
```

**Retention:**

- Hot: 6 months (SSD)
- Warm: 2 years (HDD)
- Cold: 7 years (S3 Glacier)

**Compliance Queries:**

**Reconstruct Trading History:**

```sql
SELECT * FROM audit_log
WHERE user_id = 12345
  AND timestamp BETWEEN '2023-10-01' AND '2023-10-31'
ORDER BY timestamp;
```

**Detect Wash Sales:**

```sql
WITH trades AS (
  SELECT user_id, symbol, side, timestamp
  FROM audit_log
  WHERE event_type = 'ORDER_FILLED'
)
SELECT t1.user_id, t1.symbol
FROM trades t1
JOIN trades t2 ON t1.user_id = t2.user_id
WHERE t1.side = 'BUY' AND t2.side = 'SELL'
  AND t2.timestamp BETWEEN t1.timestamp AND t1.timestamp + INTERVAL 30 DAY;
```

**GDPR (Right to be Forgotten):**

```
1. Replace user_id with SHA256 hash
2. Store encrypted mapping (separate DB)
3. Audit log retains records (immutable)
4. Delete mapping after statute expires
```

---

### Execution Algorithms

**1. VWAP (Volume Weighted Average Price)**

**Goal:** Match market's VWAP for large orders.

**Algorithm:**

```
Total: 10,000 shares
Window: 2 hours (9:15-11:15 AM)

Every 5 minutes:
  volume_pct = current_volume / expected_daily_volume
  target_qty = 10,000 × volume_pct
  place_order(target_qty - filled_qty)
```

**Example:**

```
9:20 AM:
  Market volume: 5% of daily
  Place: 5% × 10,000 = 500 shares
```

**Benefits:**

- Reduces market impact
- Matches market average
- Minimizes slippage

---

**2. TWAP (Time Weighted Average Price)**

**Goal:** Execute evenly over time.

**Algorithm:**

```
Total: 10,000 shares
Window: 2 hours = 120 minutes
Intervals: 24 (every 5 minutes)
Per interval: 10,000 / 24 = 417 shares

9:15 AM: 417 shares
9:20 AM: 417 shares
...
11:15 AM: remaining shares
```

**Advantages:**

- Simple, predictable
- Easy to implement

**Disadvantages:**

- Ignores volume patterns
- High impact in low-volume periods

---

### Portfolio Management

**Real-Time P&L Calculation:**

**Formula:**

```
Unrealized P&L (per stock):
  = (Current Price - Avg Buy Price) × Quantity

Total Portfolio P&L:
  = SUM(Unrealized P&L) + Realized P&L
```

**Implementation:**

```
function calculate_pnl(user_id):
  holdings = get_holdings(user_id)
  total_pnl = 0
  
  FOR each holding:
    current_price = redis.get(f"quote:{holding.symbol}").ltp
    pnl = (current_price - holding.avg_price) × holding.quantity
    total_pnl += pnl
  
  RETURN total_pnl + realized_pnl
```

**Update Frequency:**

- Every 1 second (market hours)
- Push to WebSocket if change > $10 or 0.5%

---

**Portfolio Breakdown:**

**Asset Allocation:**

```
RELIANCE: $25,000 (25%)
TCS: $20,000 (20%)
INFY: $18,000 (18%)
HDFC: $15,000 (15%)
ICICI: $12,000 (12%)
Cash: $10,000 (10%)

Total: $100,000
```

**Sector Allocation:**

```
Technology: 38% (TCS + INFY)
Finance: 27% (HDFC + ICICI)
Energy: 25% (RELIANCE)
Cash: 10%
```

**Performance:**

```
Overall Return: +12.5%
Best: TCS (+25%)
Worst: HDFC (-3%)
```

---

### Advanced Features

**1. Margin Financing**

**Concept:** Borrow from broker to buy stocks.

**Calculation:**

```
User cash: $10,000
Margin available: $10,000 (1:1 leverage)
Buying power: $20,000

User buys: 100 shares @ $200 = $20,000
Borrowed: $10,000
Interest: 12% annual = 1% monthly
```

**Risk Management:**

```
Initial Margin: 50%
Maintenance Margin: 30%

If equity < 30% → Margin Call
If user doesn't add funds → Auto-liquidate
```

---

**2. Dividend Tracking**

**Problem:** Credit dividends to user accounts automatically.

**Flow:**

```
1. Exchange announces dividend (RELIANCE: $10/share, ex-date: 2023-11-01)
2. Broker fetches announcement via API
3. On ex-date (2023-11-01 00:00):
   FOR each user holding RELIANCE:
     dividend = user.quantity × $10
     credit_account(user_id, dividend)
     send_notification(user_id, "Dividend credited: $" + dividend)
```

**Implementation:**

```sql
CREATE TABLE dividends (
  symbol VARCHAR(20),
  amount_per_share DECIMAL(10,2),
  ex_date DATE,
  pay_date DATE,
  status ENUM('ANNOUNCED', 'PROCESSED')
);

-- Daily job (runs at 00:00)
SELECT * FROM dividends WHERE ex_date = CURRENT_DATE AND status = 'ANNOUNCED';

FOR each dividend:
  users = SELECT user_id, quantity FROM holdings WHERE symbol = dividend.symbol;
  
  FOR each user:
    amount = user.quantity × dividend.amount_per_share;
    UPDATE accounts SET cash_balance = cash_balance + amount WHERE user_id = user.user_id;
    INSERT INTO audit_log (user_id, event_type, amount) VALUES (user.user_id, 'DIVIDEND', amount);
  
  UPDATE dividends SET status = 'PROCESSED' WHERE symbol = dividend.symbol;
```

---

**3. Tax Reporting (Form 1099/ITR)**

**Problem:** Generate annual tax forms for users.

**Required Data:**

- Total capital gains (short-term vs long-term)
- Dividend income
- Interest earned (margin financing)

**Calculation:**

```
Short-term capital gain: Holding period < 1 year
Long-term capital gain: Holding period >= 1 year

Example:
  Buy: 100 RELIANCE @ $100 on 2023-01-01
  Sell: 100 RELIANCE @ $150 on 2023-06-01 (6 months)
  Gain: ($150 - $100) × 100 = $5,000 (short-term)
  
  Tax rate: 15% short-term, 10% long-term (India)
  Tax owed: $5,000 × 0.15 = $750
```

**Report Generation:**

```
Annual job (runs Jan 1):
  FOR each user:
    trades = get_trades(user_id, year=2023)
    
    stcg = 0  // Short-term capital gains
    ltcg = 0  // Long-term capital gains
    
    FOR each trade:
      holding_period = trade.sell_date - trade.buy_date
      gain = (trade.sell_price - trade.buy_price) × trade.quantity
      
      IF holding_period < 365 days:
        stcg += gain
      ELSE:
        ltcg += gain
    
    dividend_income = get_dividends(user_id, year=2023)
    
    generate_pdf_report(user_id, stcg, ltcg, dividend_income)
    send_email(user_id, "Your 2023 tax report is ready")
```

---

**4. Corporate Actions (Stock Splits, Bonuses)**

**Stock Split Example:**

```
Event: RELIANCE announces 1:2 split (1 share → 2 shares)
Date: 2023-11-01

User owns: 100 RELIANCE @ avg $200
After split: 200 RELIANCE @ avg $100 (value unchanged)

Implementation:
  UPDATE holdings 
  SET quantity = quantity × 2, avg_price = avg_price / 2 
  WHERE symbol = 'RELIANCE';
```

**Bonus Issue Example:**

```
Event: TCS announces 1:1 bonus (1 free share for every 1 held)

User owns: 50 TCS @ avg $150
After bonus: 100 TCS @ avg $75 (value unchanged)

Implementation:
  UPDATE holdings 
  SET quantity = quantity × 2, avg_price = avg_price / 2 
  WHERE symbol = 'TCS';
```

---

### Performance Benchmarks

**Latency Measurements (Production):**

| Operation               | p50   | p99   | p99.9 | Target |
|-------------------------|-------|-------|-------|--------|
| **Quote Update (E2E)**  | 80ms  | 150ms | 250ms | <200ms |
| **Order Placement**     | 450ms | 800ms | 1.2s  | <1s    |
| **Search Query**        | 25ms  | 45ms  | 80ms  | <50ms  |
| **Portfolio Load**      | 120ms | 200ms | 350ms | <300ms |
| **WebSocket Reconnect** | 1.5s  | 3s    | 5s    | <5s    |

**Throughput Measurements:**

| Metric                         | Current | Peak  | Max Capacity |
|--------------------------------|---------|-------|--------------|
| **Concurrent WebSocket Conns** | 8M      | 10M   | 12M (scale)  |
| **Quote Updates/sec (In)**     | 80k     | 100k  | 150k         |
| **Order Entry/sec**            | 600     | 1,000 | 5,000        |
| **DB Transactions/sec**        | 4.5k    | 6k    | 10k          |
| **Search Queries/sec**         | 400     | 580   | 1,000        |

---

## WebSocket Broadcast Architecture

### WebSocket Server Topology

**Cluster Configuration:**

- **1,000 WebSocket servers** (horizontally scaled)
- Each server handles **10,000 concurrent connections**
- Total capacity: 10 million concurrent users

**Connection Routing:**

- **Load Balancer (Layer 7):** Routes initial handshake based on `user_id` hash
- **Sticky Sessions:** User always connects to same server

**Server Architecture (Per Node):**

```
Node.js (Single-Threaded Event Loop)
  ├── 10,000 WebSocket connections (in-memory)
  ├── Subscription Map: {"AAPL": [user1, user2, ...]}
  ├── Kafka Consumer
  └── Redis Client
```

**Memory Footprint:**

- 10,000 connections × 50 KB = 500 MB
- Subscription map: ~10 MB
- **Total:** ~600 MB per server

---

### Subscription Protocol

**Client → Server:**

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

**Server → Client:**

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

**Heartbeat:**

- Client sends PING every 30 seconds
- Server responds with PONG
- If no PING for 60 seconds → disconnect

*See pseudocode.md::handle_websocket_subscription() for implementation.*

---

### Failover and Redundancy

**Problem:** If WebSocket server crashes, 10,000 users lose connection.

**Solution: Graceful Failover**

1. **Health Checks:** Load balancer pings servers every 5 seconds
2. **Failure Detection:** Mark unhealthy if no response for 15 seconds
3. **Client Reconnect:** Clients detect disconnection, reconnect (exponential backoff)
4. **Resubscription:** Client sends `subscribe` with saved watchlist

**Stateless Design:**

- WebSocket servers store NO persistent state
- Clients maintain watchlist locally (localStorage)
- Reconnection rebuilds subscription map

**Downtime:** <5 seconds

---

## Bottlenecks and Future Scaling

### Quote Broadcast Fanout (Current: 200M streams)

**Problem:** Growing user base increases fanout exponentially.

**Mitigation 1: Coalescing**

- Batch updates every 100ms
- Reduces 100k updates/sec → 1k updates/sec (100x reduction)
- Trade-off: Slightly stale data (acceptable for retail)

**Mitigation 2: Differential Updates**

- Only push if price changed by >0.1%
- Reduces updates by 50%

**Mitigation 3: CDN-Based Pub/Sub**

- Use AWS IoT Core or Pusher (managed WebSocket service)
- Cost: ~$200k/month for 200M streams

---

### Ledger Database Contention (Current: 6k TPS)

**Problem:** HFT users place 100+ orders/sec, contending on same account row.

**Mitigation: Full Event Sourcing**

- Pure event sourcing (no snapshot)
- Use Kafka Streams to compute balance in real-time
- Eliminates PostgreSQL bottleneck

**Performance:**

- Kafka: 1M+ writes/sec (100x current)
- Kafka Streams: Sub-second balance computation

---

### Search Index Staleness

**Problem:** Newly listed stocks don't appear until next day.

**Mitigation: Real-Time Index Updates**

- Exchange publishes `NEW_LISTING` events to Kafka
- Elasticsearch consumer indexes in real-time
- Latency: <1 minute

---

## Common Anti-Patterns

### ❌ **Anti-Pattern 1: Polling for Quotes**

**Problem:** Client polls API every 1 second.

**Why It's Bad:**

- 10M users × 20 stocks × 1 poll/sec = **200M HTTP requests/sec**
- Higher latency (1-second interval)
- Wastes bandwidth

**Solution: ✅ WebSockets Push Model**

- Server pushes updates only when price changes
- Reduces requests from 200M/sec → ~100k/sec (2000x reduction)

---

### ❌ **Anti-Pattern 2: Synchronous Order Placement**

**Problem:** API waits for Exchange fill confirmation.

**Why It's Bad:**

- Exchange fill can take 5-10 seconds
- Blocks user's UI thread
- Ties up API resources

**Solution: ✅ Async Order Placement**

- API returns immediately with `order_id` and status `PENDING`
- WebSocket pushes status update when order fills

---

### ❌ **Anti-Pattern 3: Storing Quotes in PostgreSQL**

**Problem:** Writing 100k updates/sec to PostgreSQL.

**Why It's Bad:**

- PostgreSQL throughput: ~10k TPS (bottleneck)
- Disk I/O saturated

**Solution: ✅ Redis for Quotes**

- Redis throughput: 100k+ TPS (10x PostgreSQL)
- Sub-millisecond reads

---

## Alternative Approaches

### Server-Sent Events (SSE) Instead of WebSockets

**Pros:**

- ✅ Simpler (standard HTTP)
- ✅ Works through firewalls

**Cons:**

- ❌ One-way only (no client→server messages)
- ❌ Limited browser support

**When to Use:** Simpler use cases (price ticker only)

---

### Direct Exchange Integration (No Redis Cache)

**Pros:**

- ✅ Lower latency (one less hop)

**Cons:**

- ❌ Scales poorly (1000 duplicate feeds)
- ❌ No fallback if connection drops

**When to Use:** Ultra-low latency HFT (<1000 users)

---

### Cassandra for Ledger (NoSQL)

**Pros:**

- ✅ Higher write throughput (100k+ TPS)

**Cons:**

- ❌ **Eventual consistency unacceptable for finance**
- ❌ No ACID transactions

**When to Use:** Non-critical data only (logs, preferences). **Never for ledger.**

---

## Monitoring and Observability

### Key Metrics

| Metric                         | Target    | Alert Threshold | Purpose                        |
|--------------------------------|-----------|-----------------|--------------------------------|
| **Quote Latency (End-to-End)** | <200ms    | >500ms          | User experience                |
| **Order Placement Latency**    | <1 second | >3 seconds      | Trading execution speed        |
| **WebSocket Connection Count** | 10M       | >12M            | Capacity planning              |
| **Quote Update Rate**          | 100k/sec  | >150k/sec       | Detect unusual market activity |
| **Order Success Rate**         | >99%      | <95%            | Detect connectivity issues     |
| **Database Transaction Time**  | <200ms    | >1 second       | Ledger performance             |
| **Kafka Consumer Lag**         | <1 second | >10 seconds     | Pipeline health                |
| **Search Query Latency**       | <50ms     | >200ms          | Discovery experience           |

---

## Trade-offs Summary

| What We Gain                                | What We Sacrifice                              |
|---------------------------------------------|------------------------------------------------|
| ✅ **Low latency** (<200ms quotes)           | ❌ High infrastructure cost ($177k/month)       |
| ✅ **Strong consistency** (ACID ledger)      | ❌ Lower write throughput (6k TPS vs 100k)      |
| ✅ **Scalability** (10M users, 200M streams) | ❌ Complex architecture (Kafka, Event Sourcing) |
| ✅ **Real-time** (WebSockets push)           | ❌ Stateful servers (harder to scale)           |
| ✅ **Fast search** (Elasticsearch)           | ❌ Eventual consistency (search index lag)      |

---

## Real-World Examples

| Company                 | Use Case                       | Key Techniques                                            |
|-------------------------|--------------------------------|-----------------------------------------------------------|
| **Zerodha**             | Retail stock brokerage (India) | WebSockets for quotes, Redis cache, Event Sourcing        |
| **Robinhood**           | Commission-free trading (US)   | Real-time quotes via WebSockets, PostgreSQL ledger, Kafka |
| **ETRADE**              | Online brokerage (US)          | FIX protocol, multi-level pub/sub, ACID ledger            |
| **Interactive Brokers** | Professional trading           | Low-latency order routing, TWS API, multi-asset support   |
| **Groww**               | Mutual funds + stocks (India)  | Simplified UX, Redis caching, PostgreSQL transactions     |

---

## Cost Analysis

### Monthly Infrastructure Cost

| Component                | Configuration                   | Monthly Cost |
|--------------------------|---------------------------------|--------------|
| **WebSocket Servers**    | 1,000 instances (c5.xlarge)     | $72,000      |
| **Redis Quote Cache**    | 10-node cluster (r6g.2xlarge)   | $18,000      |
| **PostgreSQL Ledger**    | 5 shards × 3 nodes (r5.2xlarge) | $27,000      |
| **Kafka Cluster**        | 20 brokers (m5.2xlarge)         | $14,400      |
| **Elasticsearch Search** | 5-node cluster (r5.xlarge)      | $9,000       |
| **FIX Engine Servers**   | 10 instances (c5.2xlarge)       | $3,456       |
| **Order Entry API**      | 50 instances (c5.large)         | $3,600       |
| **Load Balancers**       | 5 ALBs                          | $1,500       |
| **Data Transfer**        | 100 TB/month                    | $9,000       |
| **ClickHouse (Audit)**   | 3-node cluster (i3.2xlarge)     | $7,200       |
| **Monitoring**           | 3 instances (m5.xlarge)         | $1,800       |
| **Backup Storage (S3)**  | 500 TB (7 years compliance)     | $11,500      |
| **Total**                | -                               | **$177,456** |

**Cost per User:** $0.018/month (10M users)

### Cost Optimization

1. **Reserved Instances (1-year):** Save $40k/month
2. **Spot Instances:** Save $30k/month
3. **Compression:** Save $5k/month
4. **Off-Peak Scaling:** Save $15k/month
5. **S3 Glacier (Old Logs):** Save $8k/month

**Total Savings:** $98k/month (55% reduction) → **$79k/month**

---

## References

### Technical Documentation

- **[FIX Protocol Specification](https://www.fixtrading.org/)** - Order routing protocol
- **[WebSockets RFC 6455](https://tools.ietf.org/html/rfc6455)** - WebSocket protocol
- **[Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)** - Martin Fowler
- **[Redis Pub/Sub](https://redis.io/topics/pubsub)** - Redis documentation

### Related Chapters

- **[3.4.1 Stock Exchange Matching Engine](../3.4.1-stock-exchange/)** - Order matching
- **[2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)** -
  WebSockets
- **[2.1.7 PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.7-postgresql-deep-dive.md)** - ACID transactions
- **[2.1.11 Redis Deep Dive](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)** - In-memory caching
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Event streaming
- **[2.1.13 Elasticsearch Deep Dive](../../02-components/2.1-databases/2.1.13-elasticsearch-deep-dive.md)** - Search

---

## Key Takeaways and Design Principles

### Critical Success Factors

**1. Broadcast Fanout is the Primary Bottleneck**

- Naive approach: 100k updates × 2,000 subscribers = 200M messages/sec (impossible)
- Solution: Multi-level pub/sub with selective fanout at edge (WebSocket servers)
- **Key insight:** Don't push all updates to all users. Push only what each user subscribed to.

**2. Financial Data Requires ACID**

- PostgreSQL for ledger, orders, holdings (strong consistency)
- Event Sourcing for audit trail (regulatory requirement)
- **Never compromise:** Use Cassandra/NoSQL for logs, NOT for money.

**3. Latency Optimization Techniques**

- In-memory cache (Redis) for hot data (quotes)
- Connection pooling and sticky sessions (WebSocket)
- Compression (WebSocket messages, Kafka batches)
- Pipelining (Redis, Kafka)
- CDN for static data (historical charts)

**4. Risk Management is Non-Negotiable**

- Real-time margin monitoring (every 1 minute)
- Circuit breakers (exchange-wide and per-stock)
- Position limits (max 25% in single stock)
- Fraud detection (wash trading, pump-and-dump)

**5. Compliance is Complex**

- Audit logs (7 years retention, immutable)
- Data residency (Indian users in India, EU users in EU)
- GDPR (pseudonymization for right to be forgotten)
- Tax reporting (capital gains, dividends)

---

### Architecture Decisions Summary

| Decision                  | Choice                     | Rationale                                                        | Trade-off                                     |
|---------------------------|----------------------------|------------------------------------------------------------------|-----------------------------------------------|
| **Market Data Transport** | WebSockets (Push)          | Lowest latency for real-time updates (2000x better than polling) | Stateful servers (harder to scale)            |
| **Quote Storage**         | Redis (In-Memory)          | Sub-ms reads for 200M concurrent streams                         | Expensive (10-node cluster = $18k/month)      |
| **Ledger Database**       | PostgreSQL (Sharded)       | ACID guarantees for financial integrity                          | Lower throughput (6k TPS vs NoSQL 100k TPS)   |
| **Event Bus**             | Kafka                      | High throughput (100k msg/sec), durable, replayable              | Operational complexity, eventual consistency  |
| **Search Engine**         | Elasticsearch              | Fast prefix search (<50ms), relevance ranking                    | Eventual consistency (lag behind real-time)   |
| **Order Protocol**        | FIX Protocol               | Industry standard, exchange compatibility                        | Complex (sequence numbers, heartbeats, gaps)  |
| **Event Sourcing**        | Hybrid (Events + Snapshot) | Audit trail + fast reads                                         | Complexity (event replay, materialized views) |
| **Authentication**        | Password + TOTP (2FA)      | Strong security, prevents account takeover                       | User friction (extra step at login)           |

---

### Scaling Strategy

**Current Capacity:**

- 10M concurrent users
- 100k quote updates/sec (in)
- 200M WebSocket streams (out)
- 1,000 orders/sec

**10x Growth (100M users):**

**1. WebSocket Servers**

```
Current: 1,000 servers
Future: 10,000 servers (10x)
Cost: $720k/month (up from $72k)

Alternative: Managed service (AWS IoT Core, Pusher)
  Cost: $2M/month (but zero operational burden)
```

**2. Redis Quote Cache**

```
Current: 10-node cluster
Future: 50-node cluster (5x, sharded by symbol hash)
Cost: $90k/month (up from $18k)
```

**3. PostgreSQL Ledger**

```
Current: 5 shards
Future: 50 shards (10x)
Sharding key: user_id mod 50
Cost: $270k/month (up from $27k)
```

**4. Kafka Cluster**

```
Current: 20 brokers, 100 partitions
Future: 100 brokers, 500 partitions
Cost: $72k/month (up from $14k)
```

**Total Infrastructure Cost (100M users):**

- Current (10M users): $177k/month
- Future (100M users): ~$1.5M/month (8.5x, not 10x due to economies of scale)
- **Cost per user:** Decreases from $0.018 to $0.015 (16% reduction)

---

### Technical Debt and Future Improvements

**1. Full Event Sourcing Migration**

- **Current:** Hybrid (events + materialized snapshot)
- **Future:** Pure event sourcing with Kafka Streams
- **Benefit:** Eliminates PostgreSQL write contention, scales to 1M+ TPS
- **Effort:** 6 months (high complexity, requires careful migration)

**2. Real-Time Search Index Updates**

- **Current:** Batch update (daily)
- **Future:** Real-time Kafka consumer → Elasticsearch
- **Benefit:** New stocks searchable within 1 minute
- **Effort:** 2 weeks

**3. Machine Learning for Fraud Detection**

- **Current:** Rule-based (wash trading, pump-and-dump)
- **Future:** ML model (anomaly detection, behavior patterns)
- **Benefit:** Detect sophisticated fraud (account takeover, insider trading)
- **Effort:** 3 months (data pipeline + model training)

**4. Multi-Asset Support (Options, Futures, Forex)**

- **Current:** Equities only
- **Future:** Multi-asset platform
- **Challenges:**
    - Options: Complex pricing (Black-Scholes), Greeks calculation
    - Futures: Mark-to-market (daily settlement)
    - Forex: 24/5 trading (requires global infrastructure)
- **Effort:** 12 months

**5. Algorithmic Trading API**

- **Current:** Manual trading via UI
- **Future:** REST/WebSocket API for algo trading
- **Features:**
    - Streaming market data (WebSocket)
    - Order placement API (REST/gRPC)
    - Backtesting environment (historical data)
    - Paper trading (sandbox)
- **Effort:** 6 months

---

### Lessons Learned (Post-Mortems)

**Incident 1: WebSocket Server Memory Leak (2023-08-15)**

**Problem:**

- WebSocket servers gradually consuming memory over 48 hours
- Eventually crashed when memory exceeded 16 GB
- 500k users disconnected simultaneously

**Root Cause:**

- Subscription map not cleaned up when users disconnected
- Memory leak: Dead connections accumulated

**Fix:**

```javascript
// Before (buggy)
ws.on('close', () => {
  console.log('Connection closed');
  // Bug: Subscription map not updated!
});

// After (fixed)
ws.on('close', () => {
  // Remove user from ALL subscription lists
  for (const symbol in subscriptionMap) {
    subscriptionMap[symbol] = subscriptionMap[symbol].filter(u => u !== userId);
    
    // Clean up empty symbol lists
    if (subscriptionMap[symbol].length === 0) {
      delete subscriptionMap[symbol];
    }
  }
});
```

**Prevention:**

- Added memory monitoring (alert if >12 GB)
- Implemented periodic cleanup job (every 5 minutes)
- Load testing with realistic connection churn

---

**Incident 2: PostgreSQL Lock Contention (2023-09-22)**

**Problem:**

- High-volume trading hour (9:15-9:30 AM)
- Order placement latency spiked to 5-10 seconds
- Database CPU at 100%

**Root Cause:**

- Many users placing orders on same popular stock (RELIANCE)
- `SELECT ... FOR UPDATE` locked entire account row
- Serialized all concurrent orders from same user

**Fix:**

- Migrated to Event Sourcing (Kafka)
- Eliminated locks entirely (append-only log)
- Latency dropped from 5s to <500ms

**Prevention:**

- Load testing with realistic order patterns (skewed toward popular stocks)
- Circuit breaker: Reject orders if DB latency >1s (graceful degradation)

---

**Incident 3: Kafka Consumer Lag During Market Crash (2023-10-19)**

**Problem:**

- Market crashed 15% (circuit breaker Level 2)
- Quote update rate spiked to 300k/sec (3x normal)
- Kafka consumer lag increased to 2 minutes
- Users saw stale quotes

**Root Cause:**

- WebSocket servers couldn't keep up with 3x message rate
- Single-threaded consumers (1 thread per server)

**Fix:**

- Increased consumer parallelism (5 threads per server)
- Added rate limiting (coalescing): Batch updates every 100ms
- Reduced update rate from 300k/sec → 30k/sec (10x reduction)

**Prevention:**

- Load testing with 5x normal peak rate
- Auto-scaling (add WebSocket servers when Kafka lag >10 seconds)

---

### Cost vs Performance Trade-offs

**Scenario 1: Startup (1M Users, Limited Budget)**

**Cost-Optimized Architecture:**

```
WebSocket Servers: 100 servers (c5.large) = $7,200/month
Redis: 1 cluster (r6g.xlarge) = $1,800/month
PostgreSQL: 1 shard (r5.xlarge) = $1,800/month
Kafka: 5 brokers (m5.large) = $1,800/month
Elasticsearch: 1 node (r5.large) = $900/month

Total: $13,500/month ($0.014/user)
```

**Trade-offs:**

- Higher latency (p99: 500ms vs 200ms)
- Lower availability (single shard = SPOF)
- Manual scaling (no auto-scaling)

**When to Use:** Pre-revenue startup, MVP phase

---

**Scenario 2: Mid-Stage (10M Users, Growth Focus)**

**Balanced Architecture (Current Design):**

```
Total: $177,456/month ($0.018/user)
```

**Optimizations:**

- Reserved instances (40% discount)
- Spot instances for non-critical workloads
- Off-peak scaling (reduce capacity 11 PM - 8 AM)

**Optimized Cost:** $79,000/month ($0.008/user)

**When to Use:** Post-PMF, revenue-generating, scaling fast

---

**Scenario 3: Enterprise (100M Users, Premium Performance)**

**Performance-Optimized Architecture:**

```
WebSocket Servers: 10,000 servers (c5.2xlarge) = $1.4M/month
Redis: 100-node cluster = $360k/month
PostgreSQL: 100 shards = $540k/month
Kafka: 200 brokers = $144k/month
Managed WebSocket (AWS IoT Core): $2M/month (alternative)

Total: $2.5M-4M/month ($0.025-0.040/user)
```

**Benefits:**

- Ultra-low latency (p99: <100ms)
- 99.99% availability (multi-region)
- Auto-scaling, managed services

**When to Use:** Mature company, premium product (like Interactive Brokers, ETRADE)

---

### Summary: What Makes This Design Special?

**1. Multi-Level Pub/Sub (Solves Broadcast Fanout)**

- Most elegant solution to the 200M stream problem
- Kafka as middle layer (1 → many)
- WebSocket servers as edge layer (many → millions)

**2. Hybrid Event Sourcing (Balances Audit + Performance)**

- Event log for compliance (immutable, replayable)
- Materialized view for fast reads
- Best of both worlds

**3. Risk Management Built-In (Not Afterthought)**

- Real-time margin monitoring
- Circuit breakers (market-wide + per-stock)
- Fraud detection (wash trading, pump-and-dump)

**4. FIX Protocol Integration (Industry Standard)**

- Direct exchange connectivity
- Sub-100ms order execution
- Production-grade (used by all major brokers)

**5. Regulatory Compliance First-Class (Not Bolted-On)**

- Audit logs (7 years, immutable, ClickHouse)
- Data residency (SEBI, GDPR)
- TOTP authentication (account security)

**6. Performance Optimization at Every Layer**

- WebSocket compression (55% bandwidth reduction)
- Redis pipelining (20x faster)
- Kafka batching (100x fewer network calls)
- CDN for static data (99.9999% cache hit)
