# Stock Exchange Matching Engine - Sequence Diagrams

## Table of Contents

1. [Limit Order Submission (Happy Path)](#1-limit-order-submission-happy-path)
2. [Market Order Execution (Immediate Match)](#2-market-order-execution-immediate-match)
3. [Partial Fill Scenario](#3-partial-fill-scenario)
4. [Order Cancellation](#4-order-cancellation)
5. [Flash Crash Circuit Breaker](#5-flash-crash-circuit-breaker)
6. [Matching Engine Crash Recovery](#6-matching-engine-crash-recovery)
7. [Multi-Level Matching (Price Walking)](#7-multi-level-matching-price-walking)
8. [IOC Order (Immediate-or-Cancel)](#8-ioc-order-immediate-or-cancel)
9. [Market Data Fanout](#9-market-data-fanout)
10. [Audit Log Async Write](#10-audit-log-async-write)
11. [Hot Standby Failover](#11-hot-standby-failover)
12. [DPDK Packet Processing](#12-dpdk-packet-processing)

---

## 1. Limit Order Submission (Happy Path)

**Flow:**

Shows the complete flow of a limit order submission from client to order book, with all latency breakdowns.

**Steps:**

1. **Client Request** (0μs): Trader submits limit order (BUY 100 shares @ $100.50) via FIX protocol
2. **Gateway Receive** (5μs): DPDK kernel bypass receives packet, validates JWT token (cached in memory)
3. **Order Validation** (10μs): Check order size limits, price collar, account balance
4. **Ring Buffer Enqueue** (15μs): Lock-free enqueue to shared memory queue (atomic CAS operation)
5. **Matching Engine Dequeue** (20μs): Single-threaded engine polls ring buffer, dequeues order
6. **Order Book Insert** (50μs): Red-Black Tree insert at price $100.50, append to linked list (no match, passive order)
7. **Audit Log Write** (55μs): Async write to ring buffer (non-blocking, background thread handles disk flush)
8. **ACK to Client** (60μs): Order accepted, assigned order_id, sent back via TCP
9. **Market Data Publish** (80μs): Top-of-book update published to Kafka (if best bid/ask changed)
10. **Ledger Update** (1ms): Async Kafka consumer updates PostgreSQL ledger (eventual consistency)

**Performance:**

- **Client response:** 60μs (user sees order accepted)
- **Order book updated:** 50μs
- **Fully durable:** 1-10ms (audit log flushed to disk)

**Trade-offs:**

- ✅ **Fast acknowledgment:** Client doesn't wait for disk I/O
- ⚠️ **Eventual durability:** Last 1-10ms at risk if crash (mitigated by battery-backed cache)

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant RingBuffer
    participant MatchingEngine
    participant OrderBook
    participant AuditLog
    participant MarketData
    participant Ledger

    Client->>Gateway: Limit Order BUY 100 at 100.50<br/>FIX Protocol TCP<br/>0 microseconds
    Gateway->>Gateway: DPDK Receive Packet<br/>Validate JWT Cached<br/>5 microseconds
    Gateway->>Gateway: Validate Order<br/>Size Price Account<br/>10 microseconds
    Gateway->>RingBuffer: Enqueue Order<br/>Lock-Free CAS<br/>15 microseconds
    
    RingBuffer->>MatchingEngine: Dequeue Order<br/>Poll Mode<br/>20 microseconds
    
    MatchingEngine->>OrderBook: Check Best Ask 100.52<br/>No Match Passive Order<br/>30 microseconds
    MatchingEngine->>OrderBook: Insert at Price 100.50<br/>Red-Black Tree Log N<br/>Append to Linked List<br/>50 microseconds
    
    MatchingEngine->>AuditLog: Async Write Log Entry<br/>Non-Blocking<br/>55 microseconds
    
    MatchingEngine->>Gateway: Order Accepted ACK<br/>Order ID 12345<br/>60 microseconds
    Gateway->>Client: Confirmation<br/>60 microseconds Total
    
    MatchingEngine->>MarketData: Publish Top-of-Book Update<br/>Kafka<br/>80 microseconds
    
    MarketData->>Ledger: Async Update Balance<br/>PostgreSQL<br/>1 millisecond
    
    Note over AuditLog: Background Thread<br/>Batch Flush to NVMe<br/>1 to 10 milliseconds
    
    Note over Client,Ledger: End-to-End 60 microseconds Client ACK<br/>Durability 1 to 10 milliseconds Audit Log
```

---

## 2. Market Order Execution (Immediate Match)

**Flow:**

Shows a market order that executes immediately against the best available price in the order book.

**Steps:**

1. **Client Request** (0μs): Trader submits market order (SELL 100 shares at market price)
2. **Gateway Processing** (10μs): Validates order, enqueues to ring buffer
3. **Matching Engine Dequeue** (15μs): Polls ring buffer, receives market order
4. **Best Bid Lookup** (20μs): O(1) lookup (cached at tree root) - Best bid = $100.50 with 100 shares
5. **Execute Match** (30μs): Remove best bid order from book, generate execution event
6. **Update Order Book** (35μs): Remove filled order from tree and hash map
7. **Audit Log Write** (40μs): Log both the market order and execution event (async)
8. **Execution Report** (50μs): Send execution report to both buyer and seller
9. **Market Data Publish** (60μs): Broadcast trade execution (price $100.50, volume 100) + new best bid
10. **Ledger Update** (1ms): Credit seller account, debit buyer account (async, ACID in PostgreSQL)

**Performance:**

- **Execution latency:** 30μs (from dequeue to match)
- **Client notification:** 50μs
- **Market data broadcast:** 60μs

**Key Insight:**

Market orders execute at best available price, not a fixed price. In this case, seller gets $100.50 (best bid).

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant MatchingEngine
    participant OrderBook
    participant AuditLog
    participant MarketData
    participant Ledger

    Client->>Gateway: Market Order SELL 100<br/>Execute at Best Price<br/>0 microseconds
    Gateway->>MatchingEngine: Enqueue to Ring Buffer<br/>10 microseconds
    
    MatchingEngine->>OrderBook: Lookup Best Bid O 1<br/>Best Bid 100.50 with 100 shares<br/>20 microseconds
    
    Note over MatchingEngine,OrderBook: Price-Time Priority<br/>Execute against highest bid
    
    MatchingEngine->>OrderBook: Execute Match<br/>Buyer Order ID 5678<br/>Seller Order ID 12345<br/>Price 100.50 Qty 100<br/>30 microseconds
    
    MatchingEngine->>OrderBook: Remove Filled Order<br/>Update Tree and HashMap<br/>35 microseconds
    
    MatchingEngine->>AuditLog: Log Market Order<br/>Log Execution Event<br/>Async 40 microseconds
    
    MatchingEngine->>Gateway: Execution Report Buyer<br/>Bought 100 at 100.50<br/>45 microseconds
    MatchingEngine->>Gateway: Execution Report Seller<br/>Sold 100 at 100.50<br/>50 microseconds
    
    Gateway->>Client: Execution Confirmation<br/>50 microseconds Total
    
    MatchingEngine->>MarketData: Publish Trade<br/>AAPL 100.50 Volume 100<br/>New Best Bid 100.49<br/>60 microseconds
    
    MarketData->>Ledger: Credit Seller 10050 dollars<br/>Debit Buyer 10050 dollars<br/>ACID Transaction<br/>1 millisecond
    
    Note over Ledger: PostgreSQL ensures<br/>no double-spending<br/>atomic balance update
```

---

## 3. Partial Fill Scenario

**Flow:**

Shows a large order that partially matches against available liquidity, with the remainder staying in the order book.

**Steps:**

1. **Large Order** (0μs): Client submits SELL 500 shares at $100.48 (aggressive limit order)
2. **First Match** (20μs): Best bid is $100.50 with 100 shares - MATCH (buyer's order fully filled, removed)
3. **Second Match** (30μs): Next best bid is $100.49 with 200 shares - MATCH (buyer's order fully filled, removed)
4. **Third Match** (40μs): Next best bid is $100.49 with 150 shares - PARTIAL MATCH (buyer has 150, seller needs 200)
5. **Remaining** (45μs): Seller order still has 200 shares unfilled (500 - 100 - 200 = 200)
6. **Insert Remainder** (55μs): Insert remaining 200 shares at price $100.48 into ask side of order book
7. **Status Update** (60μs): Order status = PARTIAL_FILL (filled_quantity = 300, remaining = 200)
8. **Execution Reports** (70μs): Send 3 execution reports (one for each match) to all parties
9. **Market Data** (80μs): Publish 3 trade executions + updated order book depth
10. **Client Notification** (90μs): Send partial fill notification to seller

**Performance:**

- **Multi-level matching:** 40μs for 3 matches (13μs per match)
- **Tree updates:** 3 deletions + 1 insertion = 4 × O(log N)

**Key Insight:**

Large orders "walk the book" - executing at multiple price levels until fully filled or liquidity exhausted.

```mermaid
sequenceDiagram
    participant Client
    participant MatchingEngine
    participant OrderBook
    participant AuditLog
    participant MarketData

    Client->>MatchingEngine: Limit Order SELL 500 at 100.48<br/>Aggressive crosses spread<br/>0 microseconds
    
    Note over MatchingEngine,OrderBook: Order Book State<br/>Bid 100.50 100 shares<br/>Bid 100.49 200 shares<br/>Bid 100.49 150 shares
    
    MatchingEngine->>OrderBook: Match 1 Best Bid 100.50<br/>Execute 100 shares<br/>Remaining 400<br/>20 microseconds
    
    OrderBook->>OrderBook: Remove Filled Order<br/>Bid 100.50 deleted<br/>25 microseconds
    
    MatchingEngine->>OrderBook: Match 2 Next Bid 100.49<br/>Execute 200 shares<br/>Remaining 200<br/>30 microseconds
    
    OrderBook->>OrderBook: Remove Filled Order<br/>Bid 100.49 deleted<br/>35 microseconds
    
    MatchingEngine->>OrderBook: Match 3 Next Bid 100.49<br/>Partial Execute 150 shares<br/>Buyer wants 150<br/>Seller needs 200<br/>Remaining 200<br/>40 microseconds
    
    OrderBook->>OrderBook: Update Partial Order<br/>Buyer fully filled removed<br/>45 microseconds
    
    MatchingEngine->>OrderBook: Insert Remainder<br/>Ask Side 100.48 with 200 shares<br/>Passive Order<br/>55 microseconds
    
    MatchingEngine->>AuditLog: Log 3 Executions<br/>Log Partial Fill<br/>60 microseconds
    
    MatchingEngine->>Client: Partial Fill Report<br/>Filled 300 of 500<br/>Status PARTIAL<br/>90 microseconds
    
    MatchingEngine->>MarketData: Publish 3 Trades<br/>100 at 100.50<br/>200 at 100.49<br/>100 at 100.49<br/>80 microseconds
    
    Note over Client,MarketData: Seller still has 200 shares<br/>in book at 100.48<br/>waiting for match
```

---

## 4. Order Cancellation

**Flow:**

Shows a client cancelling an existing order in the order book.

**Steps:**

1. **Cancel Request** (0μs): Client sends cancel request for order_id 12345
2. **Gateway Validation** (5μs): Verify client owns order_id 12345 (lookup in hash map)
3. **Matching Engine Dequeue** (10μs): Receives cancel request from ring buffer
4. **Hash Map Lookup** (15μs): O(1) lookup in orders hash map - finds order_id 12345 at price $100.50
5. **Remove from Order Book** (25μs): Remove from linked list (O(1)) + update tree if price level now empty (O(log N))
6. **Update Status** (30μs): Set order status = CANCELLED
7. **Audit Log** (35μs): Log cancellation event (async)
8. **ACK to Client** (40μs): Cancellation confirmed
9. **Market Data** (50μs): Publish top-of-book update if best bid/ask changed
10. **Ledger Update** (1ms): Release reserved balance (async)

**Performance:**

- **Cancellation latency:** 40μs (hash map lookup + tree/list removal)
- **No matching required:** Faster than execution

**Edge Case:**

If order already partially filled, only cancel the remaining quantity. Return partial fill status.

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant MatchingEngine
    participant OrderBook
    participant AuditLog
    participant MarketData

    Client->>Gateway: Cancel Order<br/>Order ID 12345<br/>0 microseconds
    
    Gateway->>Gateway: Validate Ownership<br/>Client ID matches Order<br/>5 microseconds
    
    Gateway->>MatchingEngine: Cancel Request<br/>via Ring Buffer<br/>10 microseconds
    
    MatchingEngine->>OrderBook: Hash Map Lookup O 1<br/>Find Order ID 12345<br/>Price 100.50 Qty 100<br/>15 microseconds
    
    OrderBook->>OrderBook: Remove from Linked List<br/>O 1 Operation<br/>20 microseconds
    
    OrderBook->>OrderBook: Check Price Level Empty<br/>If empty update Red-Black Tree<br/>O Log N<br/>25 microseconds
    
    MatchingEngine->>OrderBook: Update Status<br/>CANCELLED<br/>30 microseconds
    
    MatchingEngine->>AuditLog: Log Cancellation<br/>Async Write<br/>35 microseconds
    
    MatchingEngine->>Gateway: Cancellation ACK<br/>Order ID 12345 Cancelled<br/>40 microseconds
    
    Gateway->>Client: Confirmation<br/>40 microseconds Total
    
    MatchingEngine->>MarketData: Publish if Best Bid Ask Changed<br/>50 microseconds
    
    Note over OrderBook: Order removed<br/>no longer in book<br/>balance released
```

---

## 5. Flash Crash Circuit Breaker

**Flow:**

Shows the circuit breaker system detecting abnormal price movement and halting trading to prevent a flash crash.

**Steps:**

1. **Normal Trading** (0s): AAPL trading at $100.50, VWAP (volume-weighted average price) = $100.45
2. **Price Drop** (60s): Series of sell orders execute, price drops to $95.00 (5.5% drop)
3. **Monitor Detection** (120s): Monitor thread (runs every 1 second) detects price drop >5% in 2 minutes
4. **Level 1 Warning** (121s): Circuit breaker Level 1 triggered - slow down order acceptance (rate limiting)
5. **Continued Drop** (180s): Price continues to drop to $88.00 (12.5% drop from VWAP)
6. **Level 2 Halt** (181s): Circuit breaker Level 2 triggered - HALT trading
7. **Reject Orders** (182s): All new orders rejected with "Trading Halted" message
8. **Cancel Aggressive** (183s): Cancel all market orders and aggressive limit orders
9. **Preserve Passive** (184s): Keep passive limit orders in book
10. **Notify Participants** (185s): Broadcast halt message to all connected clients via Kafka + WebSocket
11. **Operator Review** (5 minutes): Manual review of market conditions, check for erroneous trades
12. **Resume Trading** (10 minutes): Operator manually resumes trading, orders accepted again

**Performance:**

- **Detection latency:** 1 second (monitor thread polling interval)
- **Halt execution:** <1 second (stop accepting orders)

**Key Insight:**

Circuit breakers prevent cascading liquidations and give market participants time to assess abnormal conditions.

```mermaid
sequenceDiagram
    participant Client
    participant MatchingEngine
    participant Monitor
    participant CircuitBreaker
    participant Operator
    participant MarketData

    Note over MatchingEngine: Normal Trading<br/>AAPL 100.50<br/>VWAP 100.45<br/>0 seconds
    
    Client->>MatchingEngine: Series of Sell Orders<br/>Price drops to 95.00<br/>60 seconds
    
    Monitor->>Monitor: Check Price Movement<br/>Every 1 second<br/>Delta 5.5 percent drop<br/>120 seconds
    
    Monitor->>CircuitBreaker: Trigger Level 1 Warning<br/>5 to 10 percent drop<br/>121 seconds
    
    CircuitBreaker->>MatchingEngine: Enable Rate Limiting<br/>Slow Down Orders<br/>Continue Trading<br/>121 seconds
    
    CircuitBreaker->>MarketData: Broadcast Warning<br/>Level 1 Circuit Breaker<br/>122 seconds
    
    MarketData->>Client: Warning Notification<br/>Abnormal Market Conditions<br/>123 seconds
    
    Client->>MatchingEngine: More Sell Orders<br/>Price drops to 88.00<br/>180 seconds
    
    Monitor->>Monitor: Check Price Movement<br/>Delta 12.5 percent drop<br/>180 seconds
    
    Monitor->>CircuitBreaker: Trigger Level 2 Halt<br/>greater than 10 percent drop<br/>181 seconds
    
    CircuitBreaker->>MatchingEngine: HALT Trading<br/>Reject All New Orders<br/>181 seconds
    
    MatchingEngine->>MatchingEngine: Cancel Market Orders<br/>Cancel Aggressive Limits<br/>Preserve Passive Orders<br/>183 seconds
    
    CircuitBreaker->>MarketData: Broadcast Halt<br/>Trading Halted<br/>185 seconds
    
    MarketData->>Client: Halt Notification<br/>All Orders Rejected<br/>185 seconds
    
    Client->>MatchingEngine: New Order Attempt<br/>190 seconds
    MatchingEngine->>Client: Reject Trading Halted<br/>190 seconds
    
    Operator->>CircuitBreaker: Manual Review<br/>Check Erroneous Trades<br/>5 minutes
    
    Operator->>CircuitBreaker: Resume Trading Command<br/>10 minutes
    
    CircuitBreaker->>MatchingEngine: Resume Normal Operation<br/>10 minutes
    
    MatchingEngine->>MarketData: Broadcast Resume<br/>10 minutes
    
    MarketData->>Client: Trading Resumed<br/>10 minutes
    
    Note over Client,MarketData: Total Halt Duration<br/>10 minutes<br/>market stabilized
```

---

## 6. Matching Engine Crash Recovery

**Flow:**

Shows the crash recovery process with hot standby failover and audit log replay.

**Steps:**

1. **Normal Operation** (0s): Primary matching engine processing orders, standby engine receiving audit log replication
2. **Primary Crash** (60s): Primary matching engine crashes (hardware failure, software bug, etc.)
3. **Heartbeat Timeout** (61s): Monitoring system detects missing heartbeat (1-second timeout)
4. **Failover Decision** (62s): Auto-failover system decides to promote standby to primary
5. **Promote Standby** (63s): Standby engine promoted to primary, starts accepting orders
6. **Audit Log Gap** (64s): Check for any missing log entries (last 10ms of primary's buffer)
7. **Replay Gap** (65s): Replay missing log entries from shared storage (if any)
8. **Resume Trading** (66s): New primary fully operational, accepts orders
9. **Client Reconnect** (67s): Clients automatically reconnect to new primary (failover transparent)
10. **Total Downtime** (6 seconds): Clients experience 6 seconds of rejected orders

**Cold Start Alternative:**

If no hot standby, read audit log from disk, replay all events:
- Read time: 90ms (272 MB at 3 GB/sec)
- Replay time: 1 second (1M orders at 1μs each)
- Total: ~1 second downtime

**Performance:**

- **Hot standby failover:** 5-10 seconds
- **Cold start recovery:** 1-60 seconds
- **Data loss:** 0 orders (with battery-backed cache)

```mermaid
sequenceDiagram
    participant Client
    participant Primary
    participant Standby
    participant AuditLog
    participant Monitor
    participant Operator

    Note over Primary,Standby: Normal Operation<br/>Primary processes orders<br/>Standby replicates<br/>0 seconds
    
    Primary->>AuditLog: Write Orders<br/>Write Trades<br/>Continuous
    AuditLog->>Standby: Async Replication<br/>under 10 milliseconds lag<br/>Continuous
    Standby->>Standby: Maintain Shadow Order Book<br/>Replay Events<br/>Continuous
    
    Primary->>Monitor: Heartbeat<br/>Every 1 second<br/>0 to 60 seconds
    
    Note over Primary: CRASH<br/>Hardware Failure<br/>60 seconds
    
    Monitor->>Monitor: Heartbeat Timeout<br/>No response for 1 second<br/>61 seconds
    
    Monitor->>Operator: Alert Primary Down<br/>Auto-Failover Initiated<br/>61 seconds
    
    Monitor->>Standby: Promote to Primary<br/>62 seconds
    
    Standby->>AuditLog: Check for Missing Entries<br/>Last 10 milliseconds gap<br/>63 seconds
    
    AuditLog->>Standby: Return Missing Entries<br/>Replay from Shared Storage<br/>64 seconds
    
    Standby->>Standby: Replay Missing Events<br/>Rebuild Order Book State<br/>65 seconds
    
    Standby->>Monitor: Ready to Accept Orders<br/>New Primary<br/>66 seconds
    
    Monitor->>Client: Failover Complete<br/>Reconnect to New Primary<br/>67 seconds
    
    Client->>Standby: Submit Order<br/>68 seconds
    Standby->>Client: Order Accepted<br/>68 seconds
    
    Note over Client,Standby: Total Downtime 6 seconds<br/>Hot Standby Failover<br/>Zero Data Loss
    
    Note over Primary: Cold Start Alternative<br/>Read Audit Log 90 milliseconds<br/>Replay 1 second<br/>Total 1 to 60 seconds
```

---

## 7. Multi-Level Matching (Price Walking)

**Flow:**

Shows a large market order that "walks the book" - executing at multiple price levels as it exhausts available liquidity.

**Steps:**

1. **Large Market Order** (0μs): Client submits SELL 1000 shares at market price
2. **First Price Level** (10μs): Best bid $100.50 with 100 shares - MATCH all 100
3. **Second Price Level** (20μs): Next bid $100.49 with 300 shares - MATCH all 300
4. **Third Price Level** (30μs): Next bid $100.48 with 200 shares - MATCH all 200
5. **Fourth Price Level** (40μs): Next bid $100.47 with 400 shares - MATCH all 400
6. **Fully Filled** (45μs): Order fully executed (100 + 300 + 200 + 400 = 1000 shares)
7. **Execution Reports** (50μs): Generate 4 execution reports (one per price level)
8. **VWAP Calculation** (55μs): Calculate volume-weighted average price = $100.478
9. **Market Data** (60μs): Publish 4 trade executions + updated order book depth
10. **Ledger Update** (1ms): Credit seller based on VWAP ($100,478 total)

**Performance:**

- **Multi-level matching:** 45μs for 4 matches (11μs per match)
- **Tree traversal:** 4 × O(log N) lookups

**Key Insight:**

Market orders provide immediate execution but sacrifice price control - seller gets average of available bids.

```mermaid
sequenceDiagram
    participant Client
    participant MatchingEngine
    participant OrderBook
    participant MarketData

    Note over OrderBook: Initial Order Book<br/>Bid 100.50 100 shares<br/>Bid 100.49 300 shares<br/>Bid 100.48 200 shares<br/>Bid 100.47 400 shares<br/>Total Liquidity 1000 shares
    
    Client->>MatchingEngine: Market Order SELL 1000<br/>Execute at Best Prices<br/>0 microseconds
    
    MatchingEngine->>OrderBook: Match Level 1<br/>Price 100.50 Qty 100<br/>Remaining 900<br/>10 microseconds
    
    OrderBook->>OrderBook: Remove Price Level<br/>100.50 fully consumed<br/>15 microseconds
    
    MatchingEngine->>OrderBook: Match Level 2<br/>Price 100.49 Qty 300<br/>Remaining 600<br/>20 microseconds
    
    OrderBook->>OrderBook: Remove Price Level<br/>100.49 fully consumed<br/>25 microseconds
    
    MatchingEngine->>OrderBook: Match Level 3<br/>Price 100.48 Qty 200<br/>Remaining 400<br/>30 microseconds
    
    OrderBook->>OrderBook: Remove Price Level<br/>100.48 fully consumed<br/>35 microseconds
    
    MatchingEngine->>OrderBook: Match Level 4<br/>Price 100.47 Qty 400<br/>Remaining 0<br/>40 microseconds
    
    OrderBook->>OrderBook: Remove Price Level<br/>100.47 fully consumed<br/>45 microseconds
    
    MatchingEngine->>MatchingEngine: Calculate VWAP<br/>100 times 100.50 plus 300 times 100.49<br/>plus 200 times 100.48 plus 400 times 100.47<br/>divided by 1000 equals 100.478<br/>55 microseconds
    
    MatchingEngine->>Client: Execution Report<br/>Filled 1000 at Avg 100.478<br/>4 Executions<br/>60 microseconds
    
    MatchingEngine->>MarketData: Publish 4 Trades<br/>100 at 100.50<br/>300 at 100.49<br/>200 at 100.48<br/>400 at 100.47<br/>60 microseconds
    
    Note over Client,MarketData: Price Impact<br/>Seller got 100.478 avg<br/>vs 100.50 best bid<br/>0.022 dollar per share slippage
```

---

## 8. IOC Order (Immediate-or-Cancel)

**Flow:**

Shows an IOC (Immediate-or-Cancel) order that executes available quantity and cancels the rest.

**Steps:**

1. **IOC Order** (0μs): Client submits SELL 500 shares at $100.48 IOC (must execute immediately or cancel)
2. **Check Liquidity** (10μs): Matching engine scans bid side for available liquidity at $100.48 or better
3. **Available Liquidity** (15μs): Best bid $100.50 with 100 shares, next bid $100.49 with 200 shares (total 300 shares)
4. **Partial Execution** (25μs): Execute 300 shares (100 @ $100.50, 200 @ $100.49)
5. **Cancel Remainder** (30μs): Cancel remaining 200 shares (cannot be filled immediately)
6. **Execution + Cancel Report** (40μs): Send report: "Filled 300 of 500, Cancelled 200"
7. **No Passive Order** (45μs): IOC does not insert remainder into order book (key difference from limit order)
8. **Market Data** (50μs): Publish 2 trade executions (no order book update, nothing added)

**Performance:**

- **IOC latency:** 40μs (same as normal execution + cancel)
- **No order book pollution:** Prevents passive orders sitting in book

**Use Case:**

HFT algorithms use IOC to test liquidity without leaving footprint in order book.

**Key Difference:**

- **Limit Order:** Unfilled portion stays in book (passive)
- **IOC:** Unfilled portion cancelled immediately (aggressive)

```mermaid
sequenceDiagram
    participant Client
    participant MatchingEngine
    participant OrderBook
    participant MarketData

    Note over OrderBook: Initial Order Book<br/>Bid 100.50 100 shares<br/>Bid 100.49 200 shares<br/>Total Available 300 shares
    
    Client->>MatchingEngine: IOC Order SELL 500 at 100.48<br/>Immediate-or-Cancel<br/>0 microseconds
    
    MatchingEngine->>OrderBook: Scan Available Liquidity<br/>at 100.48 or better<br/>10 microseconds
    
    OrderBook->>MatchingEngine: Return Available<br/>300 shares total<br/>100 at 100.50<br/>200 at 100.49<br/>15 microseconds
    
    MatchingEngine->>OrderBook: Execute Match 1<br/>100 shares at 100.50<br/>20 microseconds
    
    MatchingEngine->>OrderBook: Execute Match 2<br/>200 shares at 100.49<br/>25 microseconds
    
    MatchingEngine->>MatchingEngine: Check Remaining<br/>500 minus 300 equals 200 shares left<br/>Cannot fill immediately<br/>30 microseconds
    
    MatchingEngine->>MatchingEngine: Cancel Remainder<br/>200 shares cancelled<br/>IOC does NOT insert to book<br/>30 microseconds
    
    MatchingEngine->>Client: Execution Report<br/>Filled 300 of 500<br/>Cancelled 200<br/>Status PARTIAL IOC<br/>40 microseconds
    
    MatchingEngine->>MarketData: Publish 2 Trades<br/>100 at 100.50<br/>200 at 100.49<br/>No Order Book Update<br/>50 microseconds
    
    Note over Client,MarketData: Key Difference<br/>No passive order left in book<br/>Clean execution only
```

---

## 9. Market Data Fanout

**Flow:**

Shows how a single trade execution is fanned out to millions of subscribers via hierarchical broadcast.

**Steps:**

1. **Trade Execution** (0μs): Matching engine executes trade (AAPL: 100 shares @ $100.50)
2. **Publish to Kafka** (100μs): Matching engine publishes execution event to Kafka topic (100 partitions)
3. **Kafka Persistence** (200μs): Kafka persists to disk (3 replicas for durability)
4. **Market Data Services** (300μs): 100 Market Data Services consume Kafka (each handles 1 partition)
5. **Filter Subscriptions** (400μs): Each service filters by client subscriptions (who subscribes to AAPL?)
6. **Conflation** (500μs): Conflate multiple updates within 10ms window (reduce message count 10×)
7. **Protocol Buffers** (600μs): Encode message as Protocol Buffers (16 bytes vs 48 bytes JSON)
8. **WebSocket Broadcast** (700μs): Send to 100 WebSocket servers (each handles 10K clients)
9. **Client Receive** (1ms): 1M clients receive trade execution update
10. **HFT Direct Feed** (10μs): HFT clients co-located in datacenter receive direct feed (bypass Kafka)

**Performance:**

- **Standard clients:** 1ms latency (Kafka + WebSocket)
- **HFT clients:** 10μs latency (direct feed, co-location)

**Scalability:**

- 100K updates/sec × 1M clients = 100B messages/sec (hierarchical fanout required)
- Conflation reduces to 10K updates/sec × 1M clients = 10B messages/sec (manageable)

```mermaid
sequenceDiagram
    participant MatchingEngine
    participant Kafka
    participant MDS1 as Market Data Service 1
    participant MDS2 as Market Data Service N
    participant WS1 as WebSocket Server 1
    participant WS2 as WebSocket Server N
    participant Client1 as Standard Client
    participant Client2 as HFT Client

    MatchingEngine->>Kafka: Publish Trade<br/>AAPL 100 at 100.50<br/>Partition by Symbol<br/>100 microseconds
    
    Kafka->>Kafka: Persist to Disk<br/>3 Replicas<br/>200 microseconds
    
    Kafka->>MDS1: Consume Partition 1<br/>Filter AAPL Subscribers<br/>300 microseconds
    Kafka->>MDS2: Consume Partition N<br/>Filter Other Symbols<br/>300 microseconds
    
    MDS1->>MDS1: Conflate Updates<br/>Last Price in 10 milliseconds Window<br/>10x Reduction<br/>400 microseconds
    
    MDS1->>MDS1: Encode Protocol Buffers<br/>16 bytes vs 48 bytes JSON<br/>3x Smaller<br/>500 microseconds
    
    MDS1->>WS1: Send to WebSocket Server<br/>10K Clients<br/>600 microseconds
    MDS1->>WS2: Send to WebSocket Server<br/>10K Clients<br/>600 microseconds
    
    WS1->>Client1: WebSocket Push<br/>Trade Update<br/>1 millisecond Total Latency
    WS2->>Client1: WebSocket Push<br/>1 millisecond
    
    Note over Kafka,Client1: Standard Path<br/>1 millisecond Latency<br/>For Retail Traders
    
    MatchingEngine->>Client2: Direct Feed<br/>Co-Located<br/>Bypass Kafka<br/>10 microseconds Total Latency
    
    Note over MatchingEngine,Client2: HFT Fast Path<br/>10 microseconds Latency<br/>Same Datacenter
    
    Note over MatchingEngine,Client2: Scalability<br/>100K updates per sec<br/>1M clients<br/>Hierarchical Fanout<br/>Conflation 10x Reduction
```

---

## 10. Audit Log Async Write

**Flow:**

Shows the async audit log write strategy that avoids blocking the matching engine.

**Steps:**

1. **Order Processed** (0μs): Matching engine processes order, generates log entry
2. **Copy to Ring Buffer** (5μs): Copy log entry to ring buffer (shared memory, lock-free)
3. **Return to Matching** (10μs): Matching engine immediately returns to processing next order (non-blocking)
4. **Background Thread** (Async): Separate thread consumes ring buffer
5. **Batch Accumulation** (1ms): Accumulate 1000 log entries in batch buffer (1000 × 272 bytes = 272 KB)
6. **Disk Write** (2ms): Single write syscall for entire batch (amortizes fsync overhead)
7. **fsync** (3ms): Force flush to NVMe SSD (durability guaranteed)
8. **Battery-Backed Cache** (On power loss): Capacitor backup ensures writes persisted even on power failure

**Performance:**

- **Matching engine:** 5μs overhead (just copy to ring buffer)
- **Background thread:** 272 MB/sec throughput (1M entries/sec × 272 bytes)
- **Durability:** 1-10ms lag (acceptable for financial systems)

**Trade-offs:**

- ✅ **Non-blocking writes:** Matching engine never waits for disk I/O
- ⚠️ **Crash window:** Last 1-10ms at risk (mitigated by battery backup)

```mermaid
sequenceDiagram
    participant MatchingEngine
    participant RingBuffer
    participant BackgroundThread
    participant BatchBuffer
    participant NVMeSSD

    MatchingEngine->>MatchingEngine: Process Order<br/>Generate Log Entry<br/>272 bytes<br/>0 microseconds
    
    MatchingEngine->>RingBuffer: Copy to Ring Buffer<br/>Lock-Free Enqueue<br/>Shared Memory<br/>5 microseconds
    
    Note over MatchingEngine: Non-Blocking Return<br/>Continue Processing<br/>Next Order<br/>10 microseconds
    
    par Background Thread Async
        loop Every 1 millisecond
            BackgroundThread->>RingBuffer: Consume Entries<br/>Dequeue Available
            
            RingBuffer->>BackgroundThread: Return Entry 1
            RingBuffer->>BackgroundThread: Return Entry 2
            RingBuffer->>BackgroundThread: Return Entry N
            
            BackgroundThread->>BatchBuffer: Accumulate 1000 Entries<br/>272 KB Total<br/>1 millisecond
            
            BackgroundThread->>NVMeSSD: Write Batch<br/>Single Syscall<br/>272 KB<br/>2 milliseconds
            
            NVMeSSD->>NVMeSSD: fsync Flush<br/>Force to Disk<br/>Durability Guaranteed<br/>3 milliseconds
        end
    end
    
    Note over BatchBuffer,NVMeSSD: Batch Write Benefits<br/>1000 entries per syscall<br/>1000x fewer system calls<br/>Amortize fsync overhead
    
    Note over NVMeSSD: Battery-Backed Cache<br/>Capacitor Backup<br/>Persists on Power Loss<br/>No Data Loss
    
    Note over MatchingEngine,NVMeSSD: Crash Window 1 to 10 milliseconds<br/>Mitigated by Battery Backup<br/>Trade-off Acceptable
```

---

## 11. Hot Standby Failover

**Flow:**

Shows the hot standby failover process for high availability with minimal downtime.

**Steps:**

1. **Primary Active** (0s): Primary matching engine processing orders, writing audit log
2. **Standby Replication** (Continuous): Standby engine receives audit log replication (async, <10ms lag)
3. **Shadow Order Book** (Continuous): Standby maintains shadow order book by replaying audit log events
4. **Heartbeat Monitoring** (Every 1s): Monitoring system checks primary heartbeat
5. **Primary Failure** (60s): Primary crashes (hardware failure, software bug, power loss)
6. **Heartbeat Miss** (61s): Monitor detects missing heartbeat (1-second timeout)
7. **Failover Trigger** (62s): Monitor triggers auto-failover to standby
8. **Check Lag** (63s): Standby checks audit log for missing entries (last 10ms)
9. **Replay Gap** (64s): Replay any missing entries from shared storage
10. **Promote Standby** (65s): Standby promoted to primary, starts accepting orders
11. **Client Reconnect** (66s): Clients reconnect to new primary (TCP failover, transparent to application)
12. **Resume Trading** (66s): Trading resumes on new primary

**Performance:**

- **Failover time:** 5-10 seconds (hot standby promotion)
- **Data loss:** 0 orders (audit log replicated)
- **Downtime:** 6 seconds (clients experience rejected orders during failover)

**Alternative: Cold Start**

If no hot standby available:
- Read audit log: 90ms
- Replay: 1 second
- Total: 1-60 seconds downtime (acceptable for non-critical markets)

```mermaid
sequenceDiagram
    participant Primary
    participant AuditLog
    participant Standby
    participant Monitor
    participant Client

    Note over Primary,Standby: Normal Operation<br/>Primary Active<br/>Standby Replicating<br/>0 seconds
    
    loop Continuous Replication
        Primary->>AuditLog: Write Orders Trades<br/>Continuous
        AuditLog->>Standby: Async Replication<br/>under 10 milliseconds Lag
        Standby->>Standby: Replay Events<br/>Shadow Order Book<br/>Continuous
    end
    
    loop Heartbeat Monitoring
        Primary->>Monitor: Heartbeat Alive<br/>Every 1 Second<br/>0 to 60 seconds
    end
    
    Note over Primary: PRIMARY CRASH<br/>Hardware Failure<br/>60 seconds
    
    Monitor->>Monitor: Heartbeat Timeout<br/>No Response 1 Second<br/>61 seconds
    
    Monitor->>Monitor: Trigger Failover<br/>Auto Promote Standby<br/>62 seconds
    
    Standby->>AuditLog: Check for Missing Entries<br/>Last 10 milliseconds Gap<br/>63 seconds
    
    AuditLog->>Standby: Return Gap Entries<br/>If Any<br/>63 seconds
    
    Standby->>Standby: Replay Missing Entries<br/>Ensure Consistency<br/>64 seconds
    
    Monitor->>Standby: Promote to Primary<br/>Start Accepting Orders<br/>65 seconds
    
    Standby->>Monitor: Ready ACK<br/>New Primary Online<br/>65 seconds
    
    Monitor->>Client: Failover Complete<br/>Reconnect to New Primary<br/>66 seconds
    
    Client->>Standby: Submit Order<br/>TCP Reconnect<br/>66 seconds
    
    Standby->>Client: Order Accepted<br/>Trading Resumed<br/>66 seconds
    
    Note over Primary,Client: Total Downtime 6 Seconds<br/>Zero Data Loss<br/>Hot Standby Failover
```

---

## 12. DPDK Packet Processing

**Flow:**

Shows the DPDK (Data Plane Development Kit) kernel bypass networking for ultra-low latency packet processing.

**Steps:**

1. **Packet Arrival** (0ns): Ethernet packet arrives at NIC (100 Gbps Mellanox ConnectX-6)
2. **DMA Transfer** (500ns): NIC writes packet directly to application memory via DMA (Direct Memory Access)
3. **Poll Mode Driver** (1000ns): Application thread polls NIC descriptor ring (100% CPU utilization, no interrupts)
4. **Zero-Copy Read** (1500ns): Application reads packet data directly from NIC memory (no kernel copy)
5. **FIX Parsing** (3000ns): Parse FIX protocol message (order fields)
6. **Order Validation** (5000ns): Validate order (JWT auth cached, no DB lookup)
7. **Ring Buffer Enqueue** (5500ns): Enqueue to matching engine ring buffer
8. **Total Gateway Latency** (5500ns = 5.5μs): From packet arrival to ring buffer

**Traditional Stack Comparison:**

1. **Packet Arrival** (0μs): Packet arrives at NIC
2. **Hardware Interrupt** (1μs): NIC generates interrupt, CPU context switches
3. **Kernel Processing** (10μs): Linux network stack (TCP/IP, checksums, routing)
4. **Copy to User Buffer** (15μs): Copy from kernel buffer to application buffer
5. **Application Processing** (20μs): Parse FIX protocol
6. **Total Latency** (20μs): 4× slower than DPDK!

**DPDK Benefits:**

- **10× latency reduction:** 20μs → 2-5μs
- **No interrupts:** Polling eliminates context switch overhead (1000-10000 cycles)
- **Zero-copy:** Direct memory access, no kernel buffer copies

**Trade-offs:**

- ❌ **CPU overhead:** 100% utilization for polling (dedicated core required)
- ❌ **Complexity:** Custom userspace TCP/IP stack, NIC-specific drivers
- ❌ **Portability:** Tied to specific NICs (Intel X710, Mellanox ConnectX)

```mermaid
sequenceDiagram
    participant NIC as NIC Mellanox ConnectX-6
    participant DMA as DMA Engine
    participant AppMem as Application Memory
    participant PollThread as Poll Mode Thread
    participant Gateway as Gateway Application
    participant RingBuffer

    Note over NIC: Packet Arrives<br/>100 Gbps Link<br/>0 nanoseconds
    
    NIC->>DMA: Trigger DMA Transfer<br/>Direct Memory Access<br/>100 nanoseconds
    
    DMA->>AppMem: Write Packet<br/>Zero-Copy to App Memory<br/>Huge Pages 2 MB<br/>500 nanoseconds
    
    loop Poll Mode 100 percent CPU
        PollThread->>NIC: Poll Descriptor Ring<br/>Check New Packets<br/>No Interrupts<br/>1000 nanoseconds
    end
    
    PollThread->>AppMem: Read Packet Data<br/>Zero-Copy Direct Access<br/>No Kernel Copy<br/>1500 nanoseconds
    
    PollThread->>Gateway: FIX Protocol Parsing<br/>Extract Order Fields<br/>3000 nanoseconds
    
    Gateway->>Gateway: Validate Order<br/>JWT Cached in Memory<br/>Check Limits<br/>5000 nanoseconds
    
    Gateway->>RingBuffer: Enqueue to Matching Engine<br/>Lock-Free<br/>5500 nanoseconds
    
    Note over NIC,RingBuffer: Total Latency 5.5 microseconds<br/>DPDK Kernel Bypass<br/>Zero-Copy No Interrupts
    
    Note over NIC,RingBuffer: Traditional Stack Comparison<br/>Hardware Interrupt 1 microsecond<br/>Kernel Processing 10 microseconds<br/>Copy to User 15 microseconds<br/>Total 20 microseconds<br/>4x Slower
```

