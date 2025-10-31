# 3.4.7 Design an Online Code Editor / Judge (LeetCode / HackerRank)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, execution pipeline, sandboxing architecture
- **[Sequence Diagrams](./sequence-diagrams.md)** - Submission flow, execution process, real-time status updates
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of Docker vs VM, queue selection,
  security
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for judging, sandboxing,
  resource management

---

## Problem Statement

Design a highly scalable, secure online code execution and judging platform like LeetCode or HackerRank that enables
millions of users to submit code solutions in multiple programming languages. The system must execute untrusted code
securely in isolated environments, run test cases, provide real-time feedback, and handle massive traffic spikes during
coding contests while preventing malicious code from compromising the infrastructure.

---

## Requirements and Scale Estimation

### Functional Requirements

| Requirement            | Description                                                                              | Priority    |
|------------------------|------------------------------------------------------------------------------------------|-------------|
| **Code Submission**    | Accept code submissions in 20+ languages (Python, Java, C++, JavaScript, Go, Rust, etc.) | Must Have   |
| **Secure Execution**   | Execute untrusted code in isolated sandboxed environments                                | Must Have   |
| **Test Case Judging**  | Run code against hidden test cases and compare output                                    | Must Have   |
| **Real-Time Feedback** | Provide live status updates (Queued, Running, Accepted, Wrong Answer, TLE)               | Must Have   |
| **Resource Limits**    | Enforce time limits (1-10 seconds), memory limits (128-512 MB), output limits            | Must Have   |
| **Multiple Verdicts**  | Support AC, WA, TLE, MLE, RE, CE                                                         | Must Have   |
| **Custom Test Cases**  | Allow users to test code with custom inputs before submission                            | Should Have |

### Non-Functional Requirements

| Requirement              | Target                           | Rationale                                |
|--------------------------|----------------------------------|------------------------------------------|
| **Security (Critical)**  | 100% isolation                   | Malicious code must never escape sandbox |
| **High Throughput**      | 10k submissions/sec peak         | Handle coding contest traffic spikes     |
| **Low Latency (Simple)** | < 2 seconds for "Hello World"    | Fast feedback improves UX                |
| **High Availability**    | 99.9% uptime                     | Contests must not fail                   |
| **Scalability**          | Auto-scale to 10x traffic        | Contests create unpredictable load       |
| **Fault Tolerance**      | Worker failure ≠ submission loss | Queue-based retry mechanism              |

### Scale Estimation

| Metric                    | Assumption               | Calculation                               | Result                        |
|---------------------------|--------------------------|-------------------------------------------|-------------------------------|
| **Total Users**           | 10 Million MAU           | -                                         | 10M users                     |
| **Peak Submissions**      | 10x normal (contests)    | $1\text{M} / 24 / 3600 \times 10$         | **10,000 submissions/sec**    |
| **Avg Execution Time**    | 5 seconds per submission | -                                         | 5 seconds                     |
| **Concurrent Executions** | Peak QPS × Avg Time      | $10\text{k} \times 5\text{s}$             | **50,000 concurrent jobs**    |
| **Worker Count**          | 500 jobs per server      | $50\text{k} / 500$                        | **100 worker servers** (peak) |
| **Container Overhead**    | 512 MB per container     | $500 \times 512\text{MB}$                 | 256 GB RAM per server         |
| **Storage (Code)**        | 10 KB per submission     | $1\text{M} \times 10\text{KB} \times 365$ | 3.65 TB/year                  |
| **Storage (Test Cases)**  | 1 MB per problem avg     | $10\text{k problems} \times 1\text{MB}$   | 10 GB                         |

**Key Insights:**

- Peak load of 10k submissions/sec requires 50k concurrent execution environments
- Auto-scaling must spin up 100+ servers in minutes during contests
- Security is paramount: one escaped container = entire infrastructure compromised
- Execution time variance (0.1s to 10s) makes capacity planning difficult
- Queue-based architecture mandatory to absorb traffic spikes

---

## High-Level Architecture

The system follows a **queue-based asynchronous architecture** with Docker-based sandboxed execution.

### Core Design Principles

1. **Security First**: Sandboxed execution (Docker containers with seccomp, AppArmor, network isolation)
2. **Asynchronous Processing**: Queue-based architecture to decouple submission from execution
3. **Horizontal Scalability**: Stateless workers that can scale from 10 to 1000+ servers
4. **Fail-Safe**: Worker crashes don't lose submissions (queue persistence)
5. **Fair Resource Allocation**: CPU and memory quotas enforced via cgroups

### Architecture Layers

**1. Client Layer**

- Web UI, mobile apps, CLI tools
- Submit code, view results, real-time status

**2. API Gateway Layer**

- Load balancing, authentication, rate limiting
- Request validation (code size, language support)

**3. Submission Service Layer**

- Validate code syntax, store metadata
- Publish to execution queue (Kafka/SQS)

**4. Message Queue Layer**

- Kafka topics: high_priority, normal
- Partitioned by problem_id for load balancing
- Persistent storage (survive worker crashes)

**5. Execution Service Layer**

- Judge Manager: Monitor queue, distribute jobs
- Judge Workers: 100+ servers, 500 containers each
- Auto-scaling based on queue depth

**6. Worker Execution Layer**

- Docker containers with security layers
- Seccomp, AppArmor, cgroups, network isolation
- Compile, execute, compare output

**7. Persistence Layer**

- PostgreSQL: Submission metadata, verdicts
- Redis: Result cache, real-time status
- S3: Test cases, code storage

### Data Flow

**Submission Flow:**

1. User submits code via Web UI
2. API Gateway validates (auth, rate limit, code size)
3. Submission Service stores metadata, publishes to Kafka
4. Judge Manager picks from queue, assigns to worker
5. Worker creates Docker container with security policies
6. Compile code (if compiled language)
7. Execute against test cases
8. Compare output with expected
9. Update verdict in PostgreSQL and Redis
10. Client polls or receives WebSocket update

**Why This Architecture?**

- **Kafka Queue**: Absorbs spikes, guarantees delivery, decouples submission from execution
- **Docker Containers**: Lightweight sandboxing, fast startup (< 1s)
- **Auto-Scaling**: Elastic worker fleet scales on queue depth
- **Redis Cache**: Fast result retrieval
- **S3 for Test Cases**: Distributed storage with local caching

---

## Detailed Component Design

### Security and Sandboxing (Critical)

The most critical challenge is executing untrusted code safely.

**Threat Model:**

- Malicious code: Delete files, access network, fork bomb, infinite loops
- Resource abuse: Consume all CPU/memory to DOS others
- Data theft: Read test cases or other users' code
- Privilege escalation: Escape container, access host

**Security Layers (Defense in Depth):**

**Layer 1: Docker Container Isolation**

- Linux namespaces (PID, network, mount, UTS, IPC)
- Cannot see or affect other containers

**Layer 2: Seccomp (Secure Computing Mode)**

- Whitelist allowed syscalls (read, write, exit, brk, mmap)
- Block dangerous syscalls (execve, fork, clone, socket, bind, connect)

**Layer 3: AppArmor (Mandatory Access Control)**

- Define accessible files/directories
- Allow read /usr (libraries), allow read/write /tmp, deny rest

**Layer 4: Cgroups (Resource Limits)**

- CPU: 1 core max
- Memory: 512 MB max
- Disk I/O: 10 MB/sec max
- PIDs: 100 processes max (prevent fork bomb)

**Layer 5: Network Isolation**

- Containers on isolated network (no internet)
- Cannot communicate with other containers

**Layer 6: Read-Only Filesystem**

- Root filesystem read-only
- Only /tmp writable (temporary files)

**Example Docker Run:**

```bash
docker run --rm \
  --network none \
  --memory 512m \
  --cpus 1.0 \
  --pids-limit 100 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --security-opt seccomp=strict_profile.json \
  --security-opt apparmor=judge-profile \
  judge-runtime:python3.9 \
  /bin/timeout 5s python3 solution.py
```

*See `pseudocode.md::execute_in_sandbox()` for implementation.*

---

### Submission Queue Architecture

**Why Queue?**

- Execution slow (1-10 seconds)
- Traffic bursty (10x spike during contests)
- Workers can fail (need retry)
- Fair scheduling (FIFO or priority)

**Queue Implementation: Kafka**

**Topic Structure:**

- `submissions.high_priority` (contest submissions)
- `submissions.normal` (practice submissions)

**Partitioning:**

- Partition by `problem_id` for load balancing
- Same problem's submissions distributed evenly
- Workers cache test cases per partition

**Message Format:**

```json
{
  "submission_id": "sub_123456",
  "user_id": "user_789",
  "problem_id": "two_sum",
  "language": "python3",
  "code": "base64_encoded_code",
  "priority": "high",
  "timestamp": "2025-10-31T12:00:00Z"
}
```

**Consumer Groups:**

- Workers join "judge-workers" group
- Kafka auto load balances
- If worker dies, Kafka rebalances

**Retry Mechanism:**

- Execution fails → Message not acknowledged
- Kafka redelivers to another worker
- Idempotent execution (check if already judged)

---

### Judge Worker Architecture

Each worker:

1. Consumes submissions from Kafka
2. Creates Docker container
3. Executes code with resource limits
4. Compares output with test cases
5. Reports verdict

**Worker Capacity:**

- 500 concurrent containers per worker
- Container overhead: 512 MB RAM + 0.1 CPU
- Total: 256 GB RAM, 50 CPUs per server

**Execution Pipeline:**

**Step 1: Compilation (if needed)**

```
Java: javac Solution.java
C++: g++ -std=c++17 -O2 solution.cpp -o solution
Python: No compilation (interpreted)
```

**Step 2: Execution Against Test Cases**

```
For each test case:
  1. Read input: test_cases/{test_id}.in
  2. Run: cat input.txt | docker run ... solution
  3. Capture stdout: output.txt
  4. Compare: diff output.txt expected.txt
  5. Check time: Must complete within limit
  6. Check memory: Must not exceed limit
```

**Step 3: Verdict Determination**

```
All passed → Accepted (AC)
Wrong output → Wrong Answer (WA)
Timeout → Time Limit Exceeded (TLE)
Segfault → Runtime Error (RE)
OOM → Memory Limit Exceeded (MLE)
Compilation failed → Compilation Error (CE)
```

**Optimization: Test Case Caching**

- Test cases cached on worker's local disk
- No S3 fetch on every execution
- Cache invalidated on problem update

---

### Real-Time Status Updates

**Challenge:** User wants live status (Queued → Running → Accepted)

**Solution 1: Polling**

```
Client polls: GET /api/submissions/{id}/status every 1 second
Server returns: {status: "Running", progress: "Test 5/10"}
```

Pros: Simple, works everywhere  
Cons: Wastes bandwidth, 1-second delay

**Solution 2: WebSocket (Preferred)**

```
Client: ws://server/submissions/{id}
Server pushes: {status: "Running", test_case: 5, total: 10}
```

Pros: Real-time, efficient  
Cons: Stateful, harder to scale

**Implementation:**

- Worker publishes to Redis Pub/Sub: `channel:submission:{id}`
- WebSocket server subscribes to channel
- Pushes updates to client

---

### Language Support

Supporting 20+ languages requires:

1. Docker images with compilers/interpreters
2. Compilation commands (language-specific)
3. Execution commands (run compiled/interpreted code)
4. Resource profiles (Java needs more memory)

**Language Configuration:**

```python
LANGUAGES = {
    "python3": {
        "docker_image": "judge-runtime:python3.9",
        "compile_cmd": None,
        "run_cmd": "python3 solution.py",
        "memory_limit": "256MB",
        "time_multiplier": 1.0
    },
    "java": {
        "docker_image": "judge-runtime:java11",
        "compile_cmd": "javac Solution.java",
        "run_cmd": "java Solution",
        "memory_limit": "512MB",
        "time_multiplier": 1.5
    },
    "cpp": {
        "docker_image": "judge-runtime:gcc10",
        "compile_cmd": "g++ -std=c++17 -O2 solution.cpp -o solution",
        "run_cmd": "./solution",
        "memory_limit": "256MB",
        "time_multiplier": 1.0
    }
}
```

---

## Scalability and Performance Optimizations

### Auto-Scaling Strategy

**Challenge:** Traffic varies 10x (100 → 1000+ submissions/sec during contests)

**Solution: Queue-Driven Auto-Scaling**

Traditional CPU-based scaling doesn't work (workers idle waiting for slow executions).

**Queue depth-based scaling:**

```
Metric: Queue depth (messages waiting)
Target: < 1000 messages per worker
Scaling policy:
  - Depth > 1000/worker → Add workers
  - Depth < 500/worker → Remove workers
Response time: < 2 minutes
```

**Kubernetes HPA:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: judge-worker-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: judge-worker
  minReplicas: 10
  maxReplicas: 500
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumer_lag
        target:
          type: AverageValue
          averageValue: "1000"
```

**Pre-Warming:**

- Pre-scale 30 minutes before contests
- Gradually scale down after (30-min cooldown)

---

### Test Case Distribution

**Challenge:** 10 GB test cases, workers need fast access

**Solution: Distributed Caching**

**Tier 1: Worker Local Cache**

- Each worker caches on local SSD
- Cache size: 50 GB (covers 5000 problems)
- Hit rate: 95% (hot problems cached)

**Tier 2: S3 with CloudFront**

- All test cases in S3
- CloudFront CDN distributes globally
- Miss latency: 50-100ms (first run)

**Cache Invalidation:**

- Problem updated → Redis Pub/Sub event
- All workers invalidate cached test cases
- Next execution fetches from S3

---

### Database Optimization

**Write Pattern: High Volume**

- 10k submissions/sec = 10k INSERTs/sec
- Solution: Batch inserts (100 per txn)

**Read Pattern: Recent Submissions**

- Users check last 10 submissions
- Solution: Index on (user_id, created_at DESC)

**Schema:**

```sql
CREATE TABLE submissions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    problem_id VARCHAR(100) NOT NULL,
    language VARCHAR(20) NOT NULL,
    code TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'Queued',
    verdict VARCHAR(20),
    runtime_ms INT,
    memory_kb INT,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_recent (user_id, created_at DESC),
    INDEX idx_problem (problem_id, created_at DESC)
) PARTITION BY RANGE (created_at);
```

**Partitioning:**

- Partition by month (12 partitions/year)
- Old partitions archived to S3 (retain 1 year)

---

## Fault Tolerance and Error Handling

### Worker Failures

**Scenario:** Worker crashes mid-execution

**Detection:**

- Heartbeat every 10 seconds
- No heartbeat for 30 seconds → Mark dead

**Recovery:**

- Kafka redelivers unacknowledged messages
- Submission re-executed (idempotent)
- User sees no interruption

**Graceful Shutdown:**

- Worker receives SIGTERM
- Finish current executions (max 30s)
- Stop consuming new messages
- Exit cleanly

---

### Container Escape Prevention

**Scenario:** Malicious code tries to escape

**Detection:**

- AppArmor logs blocked access
- Seccomp logs blocked syscalls
- Anomaly detection (unusual patterns)

**Response:**

- Kill container immediately
- Mark as "Security Violation"
- Ban user (if repeated)
- Alert security team

---

### Queue Overflow

**Scenario:** Queue depth exceeds capacity

**Prevention:**

- Max queue size: 10M messages
- If full, reject with 503
- Show: "System overloaded, try in 5 min"

**Mitigation:**

- Priority queue for contests (always accepted)
- Normal submissions throttled first

---

## Security Deep Dive

### Attack Vectors and Mitigations

| Attack Vector       | Example                 | Mitigation                    |
|---------------------|-------------------------|-------------------------------|
| **Fork Bomb**       | `while True: os.fork()` | Cgroups pid limit (100)       |
| **Infinite Loop**   | `while True: pass`      | Timeout (kill after limit)    |
| **Memory Bomb**     | `x = [0] * 10**10`      | Cgroups memory limit (512 MB) |
| **Disk Fill**       | Write 100 GB to /tmp    | Tmpfs size limit (100 MB)     |
| **Network Attack**  | `socket.connect()`      | Network isolation             |
| **File Access**     | `open('/etc/passwd')`   | AppArmor deny (read-only fs)  |
| **Syscall Exploit** | `execve('/bin/sh')`     | Seccomp whitelist             |

### Seccomp Profile

Allows only safe syscalls (read, write, brk, mmap, etc.) and blocks dangerous ones (execve, socket, fork).

---

## Monitoring and Observability

### Key Metrics

| Metric                       | Target         | Alert          |
|------------------------------|----------------|----------------|
| **Queue Depth**              | < 1000/worker  | > 10,000 total |
| **Execution Latency**        | < 5s (p99)     | > 10s          |
| **Worker Availability**      | > 90%          | < 80%          |
| **Verdict Distribution**     | 50% AC, 30% WA | > 50% errors   |
| **Container Creation Time**  | < 1s           | > 3s           |
| **Test Case Cache Hit Rate** | > 95%          | < 90%          |
| **DB Write Latency**         | < 50ms (p99)   | > 200ms        |
| **Security Violations**      | 0/day          | > 10/day       |

### Distributed Tracing

**Trace Example:**

```
Total: 6.2s
  - API Gateway: 10ms
  - Submission Service: 50ms
  - Queue Wait: 2s
  - Worker Pickup: 100ms
  - Container Creation: 800ms
  - Compilation: 500ms
  - Execution: 2.5s
  - Result Write: 50ms
  - Client Notification: 200ms
```

### Logging

**Log Aggregation:**

- All workers → ELK stack
- Searchable by submission_id, user_id, problem_id
- Retention: 30 days

**Sample Log:**

```json
{
  "timestamp": "2025-10-31T12:00:00Z",
  "level": "INFO",
  "service": "judge-worker-5",
  "submission_id": "sub_123456",
  "verdict": "Accepted",
  "runtime_ms": 234,
  "memory_kb": 15360,
  "test_cases_passed": 10
}
```

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Synchronous Execution

**Problem:** Blocking API until execution completes (5-10s) creates bad UX.

**Solution:** ✅ Asynchronous processing with queue. Return ID immediately, client polls.

---

### ❌ Anti-Pattern 2: No Resource Limits

**Problem:** Malicious code consumes all resources.

**Solution:** ✅ Strict cgroups limits (CPU, memory, disk, PIDs).

---

### ❌ Anti-Pattern 3: Shared Execution Environment

**Problem:** Running multiple submissions in same container allows data leakage.

**Solution:** ✅ Each submission gets fresh, isolated container.

---

### ❌ Anti-Pattern 4: Trusting User Input

**Problem:** Not validating code size or language allows abuse.

**Solution:** ✅ Validate code size (max 64 KB), language (whitelist), syntax before queuing.

---

### ❌ Anti-Pattern 5: No Timeout Enforcement

**Problem:** Infinite loops hang workers.

**Solution:** ✅ Use timeout command or process monitoring. Kill after time limit.

---

### ❌ Anti-Pattern 6: Storing Code in Memory

**Problem:** Keeping all submissions in memory causes OOM.

**Solution:** ✅ Store in database, cache only active in Redis (LRU eviction).

---

## Alternative Approaches

### Alternative 1: VM-Based Isolation (AWS Lambda)

**Approach:** Each submission runs in separate VM.

**Pros:**

- Stronger isolation than Docker
- Fully managed (no infrastructure)
- Auto-scaling built-in

**Cons:**

- Cold start latency (3-5s)
- Higher cost ($0.20 per million)
- 15-minute execution limit

**When to Use:** Small-scale platform (< 1000 submissions/day)

**Why Not Chosen:** Cold start latency unacceptable for competitive programming. Cost prohibitive at scale.

---

### Alternative 2: Shared Workers (No Isolation)

**Approach:** Run all code on shared workers without containers.

**Pros:**

- Lowest latency (no overhead)
- Simple implementation
- Lowest cost

**Cons:**

- Zero security (malicious code can compromise system)
- No resource isolation
- Data leakage risk

**When to Use:** Trusted users only (internal company challenges)

**Why Not Chosen:** Completely unacceptable for public platform. Security breach would destroy reputation.

---

### Alternative 3: Browser-Based Execution (WebAssembly)

**Approach:** Compile to WebAssembly, run in browser.

**Pros:**

- Zero server cost (client-side)
- Instant feedback (no network)
- Perfect isolation (browser sandbox)

**Cons:**

- Limited language support
- Client can cheat (modify browser)
- Performance varies by device

**When to Use:** Educational platforms (security less critical)

**Why Not Chosen:** Cannot trust client for competitive programming. Need server-side verification.

---

## Real-World Examples

### LeetCode (200M Users)

**Architecture:**

- Queue: RabbitMQ (100k msg/sec)
- Execution: Docker (3000+ workers)
- Languages: 20+
- Security: Docker + seccomp + cgroups
- Scaling: Auto-scaling on queue depth

**Lessons:**

- Queue-based critical for spikes
- Pre-warming before contests reduces latency
- Test case caching improves performance 10x

---

### HackerRank (50M Users)

**Architecture:**

- Queue: AWS SQS (managed)
- Execution: Custom sandbox (ptrace-based)
- Languages: 40+ (most comprehensive)
- Security: Custom seccomp per language
- Scaling: Kubernetes auto-scaling

**Lessons:**

- Custom sandbox allows finer control
- Per-language security profiles reduce false positives
- Distributed test case storage (S3 + CloudFront) essential

---

### Codeforces (3M Users)

**Architecture:**

- Queue: Custom in-memory (Redis-based)
- Execution: Polygon system (custom engine)
- Languages: 10+ (competitive focus)
- Security: Linux namespaces + resource limits
- Scaling: Fixed worker pool (pre-scheduled contests)

**Lessons:**

- Simple queue sufficient for predictable load
- Custom judging engine optimized for competitive programming
- Fixed worker pool viable when traffic predictable

---

## Cost Analysis

### Infrastructure Costs (10k Peak Submissions/Sec)

| Component                       | Cost                |
|---------------------------------|---------------------|
| **API Gateway**                 | $600/month          |
| **Submission Service**          | $1,000/month        |
| **Kafka Cluster**               | $1,700/month        |
| **Judge Workers (100 servers)** | $180,000/month      |
| **PostgreSQL**                  | $3,000/month        |
| **Redis Cluster**               | $1,000/month        |
| **S3 Storage**                  | $30/month           |
| **Data Transfer**               | $900/month          |
| **Load Balancers**              | $200/month          |
| **Monitoring**                  | $500/month          |
| **Total**                       | **~$189,000/month** |

**Cost Per Submission:** $0.019 (at 10M/month)

**Optimization:**

- Spot instances (save 70%): $54,000/month
- Auto-scale down (save 50%): $27,000/month
- Reserved instances (save 40%): $3,400/month
- **Optimized: ~$85,000/month ($0.009/submission)**

---

## Future Enhancements

### AI-Powered Code Review

- Analyze code for mistakes
- Suggest optimizations
- Detect code smells

### Live Collaborative Debugging

- Multiple users debug together
- Share execution state
- Like Google Docs for debugging

### GPU-Accelerated Judging

- Support ML/AI problems
- PyTorch, TensorFlow problem sets
- GPU-enabled worker pool

---

## Interview Discussion Points

### Key Topics

1. **Security (Primary):**
    - Why Docker over VMs?
    - Defense in depth
    - Attack vectors and mitigations

2. **Scalability:**
    - Queue-driven auto-scaling
    - Why queue depth (not CPU)?
    - Handling 10x traffic spikes

3. **Asynchronous Architecture:**
    - Why queue-based?
    - Trade-offs vs synchronous
    - Retry mechanisms

4. **Real-Time Feedback:**
    - Polling vs WebSocket
    - Scaling WebSocket connections

5. **Resource Management:**
    - Prevent fork bomb
    - Memory limit enforcement
    - Timeout handling

### Follow-Up Questions

**Q: How prevent container escape?**  
A: Defense in depth (seccomp, AppArmor, read-only fs, network isolation)

**Q: What if submission takes 10 minutes?**  
A: Timeout enforcement (5-10s limit), kill process, return TLE

**Q: How handle 100k submissions/sec?**  
A: Scale to 1000+ workers, queue absorbs spike, pre-warm for contests

**Q: Can users see test cases?**  
A: No, test cases not exposed to container, results only shown after judging

---

## Trade-offs Summary

| What We Gain                                   | What We Sacrifice                                  |
|------------------------------------------------|----------------------------------------------------|
| ✅ Security (Docker isolation)                  | ❌ Latency overhead (container startup 0.5-1s)      |
| ✅ Scalability (queue-based, horizontal)        | ❌ Complexity (many moving parts)                   |
| ✅ Fault tolerance (queue persistence, retries) | ❌ Cost (100+ servers for peak)                     |
| ✅ Language support (20+ languages)             | ❌ Docker image size (5 GB with all runtimes)       |
| ✅ Real-time feedback (WebSocket)               | ❌ Stateful WebSocket servers                       |
| ✅ Fair resource allocation (cgroups)           | ❌ Reduced performance (limits prevent optimal CPU) |

---

## References

### Related Chapters

- **[2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)** -
  WebSocket for real-time status
- **[2.3.1 Asynchronous Communication](../../02-components/2.3-messaging-streaming/2.3.1-asynchronous-communication.md)
  ** - Queue-based architecture
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Submission queue
- **[2.4.1 Security Fundamentals](../../02-components/2.4-security-observability/2.4.1-security-fundamentals.md)** -
  Security principles
- **[3.4.1 Stock Exchange](../3.4.1-stock-exchange/3.4.1-design-stock-exchange.md)** - Low-latency patterns
- **[3.2.3 Web Crawler](../3.2.3-web-crawler/3.2.3-design-web-crawler.md)** - Distributed workers