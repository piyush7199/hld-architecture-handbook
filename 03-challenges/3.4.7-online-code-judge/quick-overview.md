# Online Code Judge - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A highly scalable, secure online code execution and judging platform like LeetCode or HackerRank that enables millions
of users to submit code solutions in multiple programming languages. The system executes untrusted code securely in
isolated environments, runs test cases, provides real-time feedback, and handles massive traffic spikes during coding
contests while preventing malicious code from compromising the infrastructure.

The platform supports 20+ programming languages, handles 10,000 submissions/sec during peak contests, provides
sub-second feedback for simple solutions, and maintains absolute security isolation between user code and the host
system.

**Key Characteristics:**

- **Secure execution** - 100% isolation (malicious code must never escape sandbox)
- **High throughput** - 10k submissions/sec peak
- **Low latency** - <2 seconds for "Hello World"
- **High availability** - 99.9% uptime (contests must not fail)
- **Scalability** - Auto-scale to 10× traffic
- **Fault tolerance** - Worker failure ≠ submission loss
- **Fairness** - Equal resources per submission

---

## Requirements & Scale

### Functional Requirements

| Requirement                   | Description                                                                                                  | Priority     |
|-------------------------------|--------------------------------------------------------------------------------------------------------------|--------------|
| **Code Submission**           | Accept code submissions in 20+ languages (Python, Java, C++, JavaScript, Go, Rust, etc.)                     | Must Have    |
| **Secure Execution**          | Execute untrusted code in isolated sandboxed environments                                                    | Must Have    |
| **Test Case Judging**         | Run code against hidden test cases and compare output                                                        | Must Have    |
| **Real-Time Feedback**        | Provide live status updates (Queued, Running, Accepted, Wrong Answer, TLE)                                   | Must Have    |
| **Resource Limits**           | Enforce time limits (1-10 seconds), memory limits (128-512 MB), output limits                                | Must Have    |
| **Multiple Verdicts**         | Support Accepted, Wrong Answer, Time Limit Exceeded, Memory Limit Exceeded, Runtime Error, Compilation Error | Must Have    |
| **Custom Test Cases**         | Allow users to test code with custom inputs before submission                                                | Should Have  |
| **Code Plagiarism Detection** | Detect copied solutions (optional, advanced feature)                                                         | Nice to Have |
| **Leaderboard**               | Real-time contest rankings with submission times                                                             | Should Have  |

### Non-Functional Requirements

| Requirement              | Target                           | Rationale                                |
|--------------------------|----------------------------------|------------------------------------------|
| **Security (Critical)**  | 100% isolation                   | Malicious code must never escape sandbox |
| **High Throughput**      | 10k submissions/sec peak         | Handle coding contest traffic spikes     |
| **Low Latency (Simple)** | < 2 seconds for "Hello World"    | Fast feedback improves UX                |
| **High Availability**    | 99.9% uptime                     | Contests must not fail                   |
| **Scalability**          | Auto-scale to 10× traffic        | Contests create unpredictable load       |
| **Fault Tolerance**      | Worker failure ≠ submission loss | Queue-based retry mechanism              |
| **Fairness**             | Equal resources per submission   | Prevent resource hogging                 |

### Scale Estimation

| Metric                    | Assumption               | Calculation                               | Result                        |
|---------------------------|--------------------------|-------------------------------------------|-------------------------------|
| **Total Users**           | 10 Million MAU           | -                                         | 10M users                     |
| **Daily Submissions**     | 1M submissions/day       | $10\text{M} \times 0.1$                   | 1M submissions/day            |
| **Peak Submissions**      | 10× normal (contests)    | $1\text{M} / 24 / 3600 \times 10$         | **10,000 submissions/sec**    |
| **Avg Execution Time**    | 5 seconds per submission | -                                         | 5 seconds                     |
| **Concurrent Executions** | Peak QPS × Avg Time      | $10\text{k} \times 5\text{s}$             | **50,000 concurrent jobs**    |
| **Worker Count**          | 500 jobs per server      | $50\text{k} / 500$                        | **100 worker servers** (peak) |
| **Container Overhead**    | 512 MB per container     | $500 \times 512\text{MB}$                 | 256 GB RAM per server         |
| **Storage (Code)**        | 10 KB per submission     | $1\text{M} \times 10\text{KB} \times 365$ | 3.65 TB/year (code storage)   |
| **Storage (Test Cases)**  | 1 MB per problem avg     | $10\text{k problems} \times 1\text{MB}$   | 10 GB (test case data)        |
| **Network Bandwidth**     | 1 KB/sec per execution   | $50\text{k} \times 1\text{KB}$            | 50 MB/sec (logs, results)     |

**Key Insights:**

- Peak load of 10k submissions/sec requires 50k concurrent execution environments
- Auto-scaling must spin up 100+ servers in minutes during contests
- Security is paramount: one escaped container = entire infrastructure compromised
- Execution time variance (0.1s to 10s) makes capacity planning difficult
- Queue-based architecture mandatory to absorb traffic spikes

---

## Key Components

| Component               | Responsibility                                  | Technology         | Scalability               |
|-------------------------|-------------------------------------------------|--------------------|---------------------------|
| **Submission Service**  | Validate code, store metadata, publish to queue | Go, Java           | Horizontal (stateless)    |
| **Submission Queue**    | Buffer submissions, ensure durability           | Apache Kafka       | Horizontal (partitions)   |
| **Judge Workers**       | Execute code in sandboxed containers            | Docker, Kubernetes | Horizontal (100+ workers) |
| **Test Case Store**     | Store test cases for problems                   | S3 + CloudFront    | Distributed               |
| **Result Store**        | Store submission results and verdicts           | PostgreSQL         | Horizontal (sharded)      |
| **Status Service**      | Provide real-time status updates                | WebSocket, Redis   | Horizontal (stateless)    |
| **Leaderboard Service** | Calculate and update contest rankings           | Redis Sorted Sets  | Horizontal (sharded)      |

---

## Security and Sandboxing (Critical)

The most critical challenge is executing untrusted code safely. A single security breach could compromise the entire
system.

**Threat Model:**

- **Malicious Code:** Code that tries to delete files, access network, fork bomb, infinite loops
- **Resource Abuse:** Code that consumes all CPU/memory to DOS other submissions
- **Data Theft:** Code that tries to read test cases or other users' code
- **Privilege Escalation:** Code that tries to escape container and access host

**Security Layers (Defense in Depth):**

**Layer 1: Docker Container Isolation**

- Each submission runs in a separate Docker container
- Containers use Linux namespaces (PID, network, mount, UTS, IPC)
- Cannot see or affect other containers or host processes

**Layer 2: Seccomp (Secure Computing Mode)**

- Whitelist of allowed system calls (read, write, exit, brk, mmap, etc.)
- Block dangerous syscalls (execve, fork, clone, socket, bind, connect)
- Example: Block network syscalls to prevent external communication

**Layer 3: AppArmor (Mandatory Access Control)**

- Define what files/directories container can access
- Example: Allow read /usr (for libraries), allow read/write /tmp, deny everything else

**Layer 4: Cgroups (Resource Limits)**

```
CPU: 1 core max
Memory: 512 MB max
Disk I/O: 10 MB/sec max
PIDs: 100 processes max (prevent fork bomb)
```

**Layer 5: Network Isolation**

- Containers run on isolated network (no internet access)
- Cannot communicate with other containers
- Cannot make outbound requests

**Layer 6: Read-Only Filesystem**

- Root filesystem mounted read-only
- Only /tmp writable (for temporary files)
- Prevents code from modifying system files

**Example Docker Run Command:**

```bash
docker run --rm \
  --network none \
  --memory 512m \
  --cpus 1.0 \
  --pids-limit 100 \
  --ulimit nofile=1024:1024 \
  --ulimit nproc=100:100 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --security-opt seccomp=strict_profile.json \
  --security-opt apparmor=judge-profile \
  judge-runtime:python3.9 \
  /bin/timeout 5s python3 solution.py
```

---

## Submission Queue Architecture

**Why Queue?**

- Execution is slow (1-10 seconds per submission)
- Traffic is bursty (10× spike during contests)
- Workers can fail (need retry mechanism)
- Need fair scheduling (FIFO or priority)

**Queue Implementation: Kafka**

**Topic Structure:**

```
submissions.high_priority (contest submissions)
submissions.normal (practice submissions)
```

**Partitioning Strategy:**

- Partition by `problem_id` for load balancing
- Ensures same problem's submissions distributed evenly
- Allows workers to cache test cases per partition

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

- Workers join consumer group "judge-workers"
- Kafka automatically load balances across workers
- If worker dies, Kafka rebalances partitions

**Retry Mechanism:**

- If execution fails (worker crash), message not acknowledged
- Kafka redelivers to another worker
- Idempotent execution (check if submission already judged)

---

## Judge Worker Architecture

Each worker is a stateless server that:

1. Consumes submissions from Kafka
2. Creates Docker container
3. Executes code with resource limits
4. Compares output with test cases
5. Reports verdict

**Worker Capacity:**

- Each worker can handle 500 concurrent containers
- Container overhead: 512 MB RAM + 0.1 CPU
- Total: 256 GB RAM, 50 CPUs per worker server

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
  1. Read input from file: test_cases/{test_id}.in
  2. Run code with input: cat input.txt | docker run ... solution
  3. Capture stdout: output.txt
  4. Compare with expected: diff output.txt expected.txt
  5. Check time: Must complete within time limit
  6. Check memory: Must not exceed memory limit
```

**Step 3: Verdict Determination**

```
All test cases passed → Accepted (AC)
Any wrong output → Wrong Answer (WA)
Timeout → Time Limit Exceeded (TLE)
Segfault/Exception → Runtime Error (RE)
OOM → Memory Limit Exceeded (MLE)
Compilation failed → Compilation Error (CE)
```

**Optimization: Test Case Caching**

- Test cases cached on worker's local disk
- No S3 fetch on every execution
- Cache invalidated on problem update

---

## Real-Time Status Updates

**Challenge:** User wants to see live status (Queued → Running → Accepted)

**Solution 1: Polling (Simple)**

```
Client polls: GET /api/submissions/{id}/status every 1 second
Server returns: {status: "Running", progress: "Test case 5/10"}
```

**Solution 2: WebSocket Push (Better)**

```
Client connects: WebSocket to /ws/submissions/{id}
Server pushes: {"status": "Running", "test_case": 5, "total": 10}
Latency: <100ms (vs 1 second polling)
```

**Implementation:**

- Submission Service publishes status updates to Redis Pub/Sub
- WebSocket servers subscribe to Redis channel
- Push updates to connected clients in real-time

---

## Bottlenecks & Solutions

### Bottleneck 1: Auto-Scaling During Contests

**Challenge:** Traffic varies 10× (100 submissions/sec → 1000+ during contests)

**Solution: Queue-Driven Auto-Scaling**

**Traditional CPU-based scaling doesn't work:**

- Workers may be idle (waiting for slow executions)
- CPU low, but queue backing up

**Queue depth-based scaling:**

```
Metric: Queue depth (messages waiting)
Target: < 1000 messages per worker
Scaling policy:
  - Queue depth > 1000/worker → Add workers
  - Queue depth < 500/worker → Remove workers
Response time: < 2 minutes (EC2 instance startup)
```

**Pre-Warming Strategy:**

- Schedule contests in advance
- Pre-scale workers 30 minutes before contest start
- Gradually scale down after contest (30-minute cooldown)

### Bottleneck 2: Test Case Distribution

**Challenge:** 10 GB of test cases, workers need fast access

**Solution: Distributed Caching**

**Tier 1: Worker Local Cache**

- Each worker caches test cases on local SSD
- Cache size: 50 GB per worker (covers 5000 problems)
- Cache hit rate: 95% (hot problems cached)

**Tier 2: S3 with CloudFront**

- All test cases stored in S3
- CloudFront CDN distributes globally
- Cache miss latency: 50-100ms (acceptable for first run)

**Cache Invalidation:**

- Problem updated → Publish event to Redis Pub/Sub
- All workers invalidate cached test cases
- Next execution fetches from S3

### Bottleneck 3: Database Write Overload

**Challenge:** 10k submissions/sec = 10k INSERTs/sec

**Solution:**

- **Batch inserts:** 100 submissions per transaction
- **Partitioning:** Partition by month (12 partitions per year)
- **Read replicas:** Offload read queries (leaderboard, history)

**Performance:**

- Before: 10k INSERTs/sec (bottleneck)
- After: 100 INSERTs/sec (batched) + read replicas

---

## Common Anti-Patterns to Avoid

### 1. Synchronous Execution

❌ **Bad:** Blocking API request until execution completes (5-10 seconds)

**Why It's Bad:**

- Terrible UX (connection blocked for 5 seconds)
- Wastes connections (cannot handle other requests)
- Cannot scale (one request = one worker)

✅ **Good:** Asynchronous processing with queue. Return submission ID immediately, client polls/subscribes for status.

### 2. No Resource Limits

❌ **Bad:** Malicious code consumes all server resources

**Why It's Bad:**

```python
# Memory bomb
x = [0] * 10**10  # Allocates 80 GB RAM, kills server
```

✅ **Good:** Enforce strict resource limits via cgroups (CPU, memory, disk, PIDs).

### 3. Shared Execution Environment

❌ **Bad:** Running multiple submissions in same container

**Why It's Bad:**

```
User A writes file: /tmp/secret.txt
User B reads file: /tmp/secret.txt (sees User A's data!)
```

✅ **Good:** Each submission gets fresh, isolated container. No state sharing between executions.

### 4. Trusting User Input

❌ **Bad:** Not validating code size or language

**Why It's Bad:**

```
User submits 100 MB code file (crashes parser)
User submits code in unsupported language (execution error)
```

✅ **Good:** Validate code size (max 64 KB), language (whitelist), and syntax before queuing.

### 5. No Timeout Enforcement

❌ **Bad:** Infinite loops hang workers indefinitely

**Why It's Bad:**

```python
while True:
    pass  # Runs forever, worker stuck
```

✅ **Good:** Use `timeout` command or process monitoring. Kill after time limit (e.g., 5 seconds).

---

## Monitoring & Observability

### Key Metrics

**System Metrics:**

- **Submission throughput:** 10k submissions/sec (peak target)
- **Execution latency:** <2 seconds p99 (simple solutions)
- **Queue depth:** <1000 messages per worker (target)
- **Worker utilization:** 70-80% (target)
- **Container startup time:** <1 second (target)
- **Test case cache hit rate:** >95% (target)

**Business Metrics:**

- **Submissions processed:** 1M submissions/day
- **Success rate:** >99% (submissions judged successfully)
- **Average execution time:** 5 seconds
- **Contest participation:** 100k users per contest

### Dashboards

1. **Real-Time Operations Dashboard**
    - Queue depth (by priority)
    - Worker count (active, idle)
    - Execution latency (p50, p95, p99)
    - Container startup time

2. **Submission Dashboard**
    - Submissions by status (Queued, Running, Accepted, etc.)
    - Submissions by language
    - Submissions by problem
    - Verdict distribution

3. **System Health Dashboard**
    - Worker health (CPU, memory, disk)
    - Kafka consumer lag
    - Database connection pool
    - Test case cache hit rate

### Alerts

**Critical Alerts (PagerDuty, 24/7 on-call):**

- Queue depth >10K messages (scale up workers)
- Worker crash rate >5% (investigate)
- Execution latency >10 seconds for 5 minutes
- Security breach detected (container escape)

**Warning Alerts (Slack, email):**

- Queue depth >5K messages
- Test case cache hit rate <90%
- Worker CPU >90%
- Database connection pool >80%

---

## Trade-offs Summary

| What We Gain                                       | What We Sacrifice                                            |
|----------------------------------------------------|--------------------------------------------------------------|
| ✅ **Security** (Docker isolation)                  | ❌ **Latency overhead** (container startup 0.5-1s)            |
| ✅ **Scalability** (queue-based, horizontal)        | ❌ **Complexity** (many moving parts)                         |
| ✅ **Fault tolerance** (queue persistence, retries) | ❌ **Cost** (100+ servers for peak)                           |
| ✅ **Language support** (20+ languages)             | ❌ **Docker image size** (5 GB with all runtimes)             |
| ✅ **Real-time feedback** (WebSocket)               | ❌ **Stateful WebSocket servers** (harder to scale)           |
| ✅ **Fair resource allocation** (cgroups)           | ❌ **Reduced performance** (limits prevent optimal CPU usage) |

---

## Real-World Examples

### LeetCode (200M Users)

- **Architecture:** Submission queue: RabbitMQ (100k messages/sec), Execution: Docker containers (3000+ workers),
  Languages: 20+, Security: Docker + seccomp + cgroups, Scaling: Auto-scaling based on queue depth
- **Lessons:** Queue-based architecture critical for handling spikes, pre-warming workers before contests reduces
  latency, test case caching improves performance 10×

### HackerRank (50M Users)

- **Architecture:** Submission queue: AWS SQS (managed queue), Execution: Custom sandbox (ptrace-based isolation),
  Languages: 40+, Security: Custom seccomp profiles per language, Scaling: Kubernetes auto-scaling
- **Lessons:** Custom sandbox allows finer control than Docker, per-language security profiles reduce false positives,
  distributed test case storage (S3 + CloudFront) essential

### Codeforces (3M Users)

- **Architecture:** Submission queue: Custom in-memory queue (Redis-based), Execution: Polygon system (custom judging
  engine), Languages: 10+, Security: Linux namespaces + resource limits, Scaling: Fixed worker pool (contest
  pre-scheduled)
- **Lessons:** Simple queue sufficient for predictable load, custom judging engine optimized for competitive
  programming, fixed worker pool viable when traffic predictable

---

## Key Takeaways

1. **Security layers** (Docker + seccomp + AppArmor + cgroups) prevent container escape
2. **Queue-based architecture** decouples submission from execution (handles bursts)
3. **Auto-scaling** based on queue depth (not CPU) handles traffic spikes
4. **Test case caching** reduces latency by 10× (local SSD vs S3)
5. **WebSocket push** provides real-time status (<100ms vs 1s polling)
6. **Resource limits** (cgroups) prevent resource abuse (CPU, memory, PIDs)
7. **Idempotent execution** prevents duplicate judging (check if already processed)
8. **Batch inserts** reduce database load (100 submissions per transaction)
9. **Pre-warming workers** before contests reduces latency (30-minute advance)
10. **Container isolation** ensures no data leakage between users

---

## Recommended Stack

- **Submission Service:** Go/Java (gRPC), PostgreSQL (metadata)
- **Submission Queue:** Apache Kafka (partitioned by problem_id)
- **Judge Workers:** Docker containers, Kubernetes (auto-scaling)
- **Test Case Store:** S3 + CloudFront CDN (distributed caching)
- **Result Store:** PostgreSQL (sharded by user_id)
- **Status Service:** WebSocket (Node.js/Go), Redis Pub/Sub
- **Leaderboard:** Redis Sorted Sets (real-time rankings)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (submission, execution, status
  updates)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (sandbox execution, verdict determination)

