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

## 1. Problem Statement

Design a highly scalable, secure online code execution and judging platform like LeetCode or HackerRank that enables
millions of users to submit code solutions in multiple programming languages. The system must execute untrusted code
securely in isolated environments, run test cases, provide real-time feedback, and handle massive traffic spikes during
coding contests while preventing malicious code from compromising the infrastructure.

The platform must support 20+ programming languages, handle 10,000 submissions/sec during peak contests, provide
sub-second feedback for simple solutions, and maintain absolute security isolation between user code and the host
system.

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

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

### Non-Functional Requirements (NFRs)

| Requirement              | Target                           | Rationale                                |
|--------------------------|----------------------------------|------------------------------------------|
| **Security (Critical)**  | 100% isolation                   | Malicious code must never escape sandbox |
| **High Throughput**      | 10k submissions/sec peak         | Handle coding contest traffic spikes     |
| **Low Latency (Simple)** | < 2 seconds for "Hello World"    | Fast feedback improves UX                |
| **High Availability**    | 99.9% uptime                     | Contests must not fail                   |
| **Scalability**          | Auto-scale to 10x traffic        | Contests create unpredictable load       |
| **Fault Tolerance**      | Worker failure ≠ submission loss | Queue-based retry mechanism              |
| **Fairness**             | Equal resources per submission   | Prevent resource hogging                 |

### Scale Estimation

| Metric                    | Assumption               | Calculation                               | Result                        |
|---------------------------|--------------------------|-------------------------------------------|-------------------------------|
| **Total Users**           | 10 Million MAU           | -                                         | 10M users                     |
| **Daily Submissions**     | 1M submissions/day       | $10\text{M} \times 0.1$                   | 1M submissions/day            |
| **Peak Submissions**      | 10x normal (contests)    | $1\text{M} / 24 / 3600 \times 10$         | **10,000 submissions/sec**    |
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

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### Core Design Principles

1. **Security First**: Sandboxed execution (Docker containers with seccomp, AppArmor, network isolation)
2. **Asynchronous Processing**: Queue-based architecture to decouple submission from execution
3. **Horizontal Scalability**: Stateless workers that can scale from 10 to 1000+ servers
4. **Fail-Safe**: Worker crashes don't lose submissions (queue persistence)
5. **Fair Resource Allocation**: CPU and memory quotas enforced via cgroups

### System Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                    │
│  │   Web UI     │   │   Mobile     │   │   CLI Tool   │                    │
│  │  (Browser)   │   │     App      │   │  (VS Code)   │                    │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                    │
│         │                  │                   │                             │
│         └──────────────────┼───────────────────┘                             │
│                            │                                                 │
└────────────────────────────┼─────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       API GATEWAY LAYER                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    API Gateway (Load Balanced)                         │  │
│  │       Authentication, Rate Limiting, Request Validation                │  │
│  └───────────────────────┬───────────────────────────────────────────────┘  │
│                          │                                                   │
└──────────────────────────┼───────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SUBMISSION SERVICE LAYER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Submission Service                                 │   │
│  │          - Validate code (syntax, size limits)                        │   │
│  │          - Store submission metadata                                  │   │
│  │          - Publish to execution queue                                 │   │
│  └────────────────────────┬─────────────────────────────────────────────┘   │
│                           │                                                  │
└───────────────────────────┼──────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MESSAGE QUEUE LAYER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                   Submission Queue (Kafka/SQS)                         │  │
│  │           Topics: submissions.high_priority, submissions.normal        │  │
│  │              Partitioned by problem_id for load balancing              │  │
│  └─────────────────────────┬─────────────────────────────────────────────┘  │
│                            │                                                 │
└────────────────────────────┼─────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     EXECUTION SERVICE LAYER                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                   Judge Manager Service                                │  │
│  │       - Monitor queue depth                                            │  │
│  │       - Distribute jobs to available workers                           │  │
│  │       - Track worker health                                            │  │
│  │       - Trigger auto-scaling                                           │  │
│  └─────────────────────────┬─────────────────────────────────────────────┘  │
│                            │                                                 │
│         ┌──────────────────┼──────────────────┐                             │
│         ▼                  ▼                  ▼                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │  Worker 1   │    │  Worker 2   │    │  Worker N   │                     │
│  │             │    │             │    │             │                     │
│  │ Docker Host │    │ Docker Host │    │ Docker Host │                     │
│  │ (500 slots) │    │ (500 slots) │    │ (500 slots) │                     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                     │
│         │                  │                  │                             │
└─────────┼──────────────────┼──────────────────┼─────────────────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        WORKER EXECUTION LAYER                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                   Sandboxed Execution (Docker Container)                │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │  │  Security Layers:                                                 │  │ │
│  │  │  - Docker container (namespace isolation)                         │  │ │
│  │  │  - Seccomp profile (syscall filtering)                            │  │ │
│  │  │  - AppArmor (mandatory access control)                            │  │ │
│  │  │  - Cgroups (CPU, memory, I/O limits)                              │  │ │
│  │  │  - Network isolation (no internet access)                         │  │ │
│  │  │  - Read-only filesystem (except /tmp)                             │  │ │
│  │  └──────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                          │ │
│  │  Execution Steps:                                                        │ │
│  │  1. Compile code (if needed)                                             │ │
│  │  2. Run against test cases (stdin → stdout)                              │ │
│  │  3. Compare output with expected (diff)                                  │ │
│  │  4. Report verdict (AC, WA, TLE, MLE, RE, CE)                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PERSISTENCE LAYER                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Submission Database (PostgreSQL)                  │   │
│  │    Tables: submissions, verdicts, execution_logs, leaderboard        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                  Result Cache (Redis)                                 │   │
│  │       Key: submission:{id} → {status, verdict, runtime, memory}      │   │
│  │                       TTL: 1 hour                                     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                Test Case Storage (S3 / Object Store)                  │   │
│  │           Path: s3://testcases/{problem_id}/{test_id}.txt             │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

**Submission Flow (Happy Path):**

1. User submits code via Web UI
2. API Gateway validates request (auth, rate limit)
3. Submission Service validates code (size, language)
4. Create submission record in PostgreSQL (status: "Queued")
5. Publish submission to Kafka queue
6. Judge Manager picks submission from queue
7. Assign to available Worker
8. Worker creates Docker container with security policies
9. Compile code (if compiled language)
10. Execute against test cases in container
11. Compare output with expected results
12. Update verdict in PostgreSQL and Redis
13. Client polls Redis for result (or WebSocket push)

**Why This Architecture?**

- **Kafka Queue**: Absorbs traffic spikes, guarantees at-least-once delivery, decouples submission from execution
- **Docker Containers**: Lightweight sandboxing with namespace isolation, fast startup (< 1 second)
- **Auto-Scaling Workers**: Elastic worker fleet scales based on queue depth (not CPU)
- **Redis Cache**: Fast result retrieval without hitting database
- **S3 for Test Cases**: Distributed storage with CDN-like caching on workers

---

## 4. Detailed Component Design

### 4.1 Security and Sandboxing (Critical)

The most critical challenge is executing untrusted code safely. A single security breach could compromise the entire
system.

**Threat Model:**

- **Malicious Code**: Code that tries to delete files, access network, fork bomb, infinite loops
- **Resource Abuse**: Code that consumes all CPU/memory to DOS other submissions
- **Data Theft**: Code that tries to read test cases or other users' code
- **Privilege Escalation**: Code that tries to escape container and access host

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

*See `pseudocode.md::execute_in_sandbox()` for detailed implementation.*

---

### 4.2 Submission Queue Architecture

**Why Queue?**

- Execution is slow (1-10 seconds per submission)
- Traffic is bursty (10x spike during contests)
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

*See `pseudocode.md::consume_submission()` for implementation.*

---

### 4.3 Judge Worker Architecture

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

*See `pseudocode.md::judge_submission()` for complete implementation.*

---

### 4.4 Real-Time Status Updates

**Challenge**: User wants to see live status (Queued → Running → Accepted)

**Solution 1: Polling (Simple)**

```
Client polls: GET /api/submissions/{id}/status every 1 second
Server returns: {status: "Running", progress: "Test case 5/10"}
```

**Pros:** Simple, works everywhere  
**Cons:** Wastes bandwidth, 1-second delay

**Solution 2: WebSocket (Preferred)**

```
Client: ws://server/submissions/{id}
Server pushes: {status: "Running", test_case: 5, total: 10}
```

**Pros:** Real-time, efficient  
**Cons:** Stateful connections, harder to scale

**Implementation:**

- Worker publishes status updates to Redis Pub/Sub: `channel:submission:{id}`
- WebSocket server subscribes to channel
- Pushes updates to client

*See `sequence-diagrams.md` for detailed flow.*

---

### 4.5 Language Support

Supporting 20+ languages requires:

1. **Docker Images**: Pre-built images with compilers/interpreters
2. **Compilation Commands**: Language-specific compile steps
3. **Execution Commands**: How to run compiled/interpreted code
4. **Resource Profiles**: Different limits per language (Java needs more memory)

**Language Configuration:**

```python
LANGUAGES = {
    "python3": {
        "docker_image": "judge-runtime:python3.9",
        "compile_cmd": None,  # Interpreted
        "run_cmd": "python3 solution.py",
        "memory_limit": "256MB",
        "time_multiplier": 1.0
    },
    "java": {
        "docker_image": "judge-runtime:java11",
        "compile_cmd": "javac Solution.java",
        "run_cmd": "java Solution",
        "memory_limit": "512MB",  # JVM overhead
        "time_multiplier": 1.5  # Slower than C++
    },
    "cpp": {
        "docker_image": "judge-runtime:gcc10",
        "compile_cmd": "g++ -std=c++17 -O2 solution.cpp -o solution",
        "run_cmd": "./solution",
        "memory_limit": "256MB",
        "time_multiplier": 1.0  # Baseline
    }
}
```

**Docker Image Management:**

- Multi-stage Dockerfile includes all runtimes
- Base image: 5 GB (all compilers)
- Cached on all workers (no pull on every execution)
- Updated weekly with security patches

---

## 5. Scalability and Performance Optimizations

### 5.1 Auto-Scaling Strategy

**Challenge**: Traffic varies 10x (100 submissions/sec → 1000+ during contests)

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

**Implementation (Kubernetes HPA):**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: judge-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
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

**Pre-Warming Strategy:**

- Schedule contests in advance
- Pre-scale workers 30 minutes before contest start
- Gradually scale down after contest (30-minute cooldown)

---

### 5.2 Test Case Distribution

**Challenge**: 10 GB of test cases, workers need fast access

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

**Optimization: Lazy Loading**

- Don't pre-cache all test cases
- Fetch on-demand (first submission for a problem)
- Popular problems naturally cached

---

### 5.3 Database Optimization

**Write Pattern: High Volume**

- 10k submissions/sec = 10k INSERTs/sec
- Solution: Batch inserts (100 submissions per txn)

**Read Pattern: Recent Submissions**

- Users check last 10 submissions frequently
- Solution: Index on (user_id, created_at DESC)

**Schema Optimization:**

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

**Partitioning Strategy:**

- Partition by month (12 partitions per year)
- Old partitions archived to S3 (retain 1 year online)

---

### 5.4 Network Optimization

**Problem: Docker Pulls Slow**

- Pulling 5 GB Docker image per worker takes 10 minutes
- Slows down auto-scaling

**Solution: Pre-Baked AMIs**

- Create EC2 AMI with Docker images pre-loaded
- New workers launch from AMI (< 1 minute startup)
- Update AMI weekly with latest images

**Problem: Test Case Download**

- 1 MB test case × 50k concurrent jobs = 50 GB/sec bandwidth

**Solution: Local Caching (see 5.2)**

---

## 6. Fault Tolerance and Error Handling

### 6.1 Worker Failures

**Scenario: Worker crashes mid-execution**

**Detection:**

- Worker heartbeat every 10 seconds
- If no heartbeat for 30 seconds → Mark worker dead

**Recovery:**

- Kafka redelivers unacknowledged messages to other workers
- Submission re-executed (idempotent)
- User sees no interruption (submission stays "Running")

**Graceful Shutdown:**

- Worker receives SIGTERM (Kubernetes pod termination)
- Finish current executions (max 30 seconds)
- Stop consuming new messages
- Exit cleanly

---

### 6.2 Container Escape Prevention

**Scenario: Malicious code tries to escape container**

**Detection:**

- AppArmor logs blocked access attempts
- Seccomp logs blocked syscalls
- Anomaly detection (unusual syscall patterns)

**Response:**

- Kill container immediately
- Mark submission as "Security Violation"
- Ban user (if repeated attempts)
- Alert security team

**Testing:**

- Regular penetration testing with "escape attempts"
- Bug bounty program for security researchers

---

### 6.3 Queue Overflow

**Scenario: Queue depth exceeds capacity (1M messages)**

**Prevention:**

- Max queue size: 10M messages
- If full, reject new submissions with 503 Service Unavailable
- Show user: "System overloaded, try again in 5 minutes"

**Mitigation:**

- Priority queue for contest submissions (always accepted)
- Normal submissions throttled first

---

## 7. Security Deep Dive

### 7.1 Attack Vectors and Mitigations

| Attack Vector       | Example                 | Mitigation                        |
|---------------------|-------------------------|-----------------------------------|
| **Fork Bomb**       | `while True: os.fork()` | Cgroups pid limit (100 processes) |
| **Infinite Loop**   | `while True: pass`      | Timeout (kill after time limit)   |
| **Memory Bomb**     | `x = [0] * 10**10`      | Cgroups memory limit (512 MB)     |
| **Disk Fill**       | Write 100 GB to /tmp    | Tmpfs size limit (100 MB)         |
| **Network Attack**  | `socket.connect()`      | Network isolation (no internet)   |
| **File Access**     | `open('/etc/passwd')`   | AppArmor deny (read-only fs)      |
| **Syscall Exploit** | `execve('/bin/sh')`     | Seccomp whitelist (block execve)  |

### 7.2 Seccomp Profile Example

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "read",
        "write",
        "open",
        "close",
        "stat",
        "fstat",
        "lstat",
        "poll",
        "lseek",
        "mmap",
        "mprotect",
        "munmap",
        "brk",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "ioctl",
        "pread64",
        "pwrite64",
        "readv",
        "writev",
        "access",
        "pipe",
        "select",
        "sched_yield",
        "mremap",
        "msync",
        "mincore",
        "madvise",
        "shmget",
        "shmat",
        "shmctl",
        "dup",
        "dup2",
        "pause",
        "nanosleep",
        "getitimer",
        "alarm",
        "setitimer",
        "getpid",
        "sendfile",
        "socket",
        "connect",
        "accept",
        "sendto",
        "recvfrom",
        "sendmsg",
        "recvmsg",
        "shutdown",
        "bind",
        "listen",
        "getsockname",
        "getpeername",
        "socketpair",
        "setsockopt",
        "getsockopt",
        "clone",
        "fork",
        "vfork",
        "execve",
        "exit",
        "wait4",
        "kill",
        "uname",
        "semget",
        "semop",
        "semctl",
        "shmdt",
        "msgget",
        "msgsnd",
        "msgrcv",
        "msgctl",
        "fcntl",
        "flock",
        "fsync",
        "fdatasync",
        "truncate",
        "ftruncate",
        "getdents",
        "getcwd",
        "chdir",
        "fchdir",
        "rename",
        "mkdir",
        "rmdir",
        "creat",
        "link",
        "unlink",
        "symlink",
        "readlink",
        "chmod",
        "fchmod",
        "chown",
        "fchown",
        "lchown",
        "umask",
        "gettimeofday",
        "getrlimit",
        "getrusage",
        "sysinfo",
        "times",
        "ptrace",
        "getuid",
        "syslog",
        "getgid",
        "setuid",
        "setgid",
        "geteuid",
        "getegid",
        "setpgid",
        "getppid",
        "getpgrp",
        "setsid",
        "setreuid",
        "setregid",
        "getgroups",
        "setgroups",
        "setresuid",
        "getresuid",
        "setresgid",
        "getresgid",
        "getpgid",
        "setfsuid",
        "setfsgid",
        "getsid",
        "capget",
        "capset",
        "rt_sigpending",
        "rt_sigtimedwait",
        "rt_sigqueueinfo",
        "rt_sigsuspend",
        "sigaltstack",
        "utime",
        "mknod",
        "uselib",
        "personality",
        "ustat",
        "statfs",
        "fstatfs",
        "sysfs",
        "getpriority",
        "setpriority",
        "sched_setparam",
        "sched_getparam",
        "sched_setscheduler",
        "sched_getscheduler",
        "sched_get_priority_max",
        "sched_get_priority_min",
        "sched_rr_get_interval",
        "mlock",
        "munlock",
        "mlockall",
        "munlockall",
        "vhangup",
        "modify_ldt",
        "pivot_root",
        "_sysctl",
        "prctl",
        "arch_prctl",
        "adjtimex",
        "setrlimit",
        "chroot",
        "sync",
        "acct",
        "settimeofday",
        "mount",
        "umount2",
        "swapon",
        "swapoff",
        "reboot",
        "sethostname",
        "setdomainname",
        "iopl",
        "ioperm",
        "create_module",
        "init_module",
        "delete_module",
        "get_kernel_syms",
        "query_module",
        "quotactl",
        "nfsservctl",
        "getpmsg",
        "putpmsg",
        "afs_syscall",
        "tuxcall",
        "security",
        "gettid",
        "readahead",
        "setxattr",
        "lsetxattr",
        "fsetxattr",
        "getxattr",
        "lgetxattr",
        "fgetxattr",
        "listxattr",
        "llistxattr",
        "flistxattr",
        "removexattr",
        "lremovexattr",
        "fremovexattr",
        "tkill",
        "time",
        "futex",
        "sched_setaffinity",
        "sched_getaffinity",
        "set_thread_area",
        "io_setup",
        "io_destroy",
        "io_getevents",
        "io_submit",
        "io_cancel",
        "get_thread_area",
        "lookup_dcookie",
        "epoll_create",
        "epoll_ctl_old",
        "epoll_wait_old",
        "remap_file_pages",
        "getdents64",
        "set_tid_address",
        "restart_syscall",
        "semtimedop",
        "fadvise64",
        "timer_create",
        "timer_settime",
        "timer_gettime",
        "timer_getoverrun",
        "timer_delete",
        "clock_settime",
        "clock_gettime",
        "clock_getres",
        "clock_nanosleep",
        "exit_group",
        "epoll_wait",
        "epoll_ctl",
        "tgkill",
        "utimes",
        "vserver",
        "mbind",
        "set_mempolicy",
        "get_mempolicy",
        "mq_open",
        "mq_unlink",
        "mq_timedsend",
        "mq_timedreceive",
        "mq_notify",
        "mq_getsetattr",
        "kexec_load",
        "waitid",
        "add_key",
        "request_key",
        "keyctl",
        "ioprio_set",
        "ioprio_get",
        "inotify_init",
        "inotify_add_watch",
        "inotify_rm_watch",
        "migrate_pages",
        "openat",
        "mkdirat",
        "mknodat",
        "fchownat",
        "futimesat",
        "newfstatat",
        "unlinkat",
        "renameat",
        "linkat",
        "symlinkat",
        "readlinkat",
        "fchmodat",
        "faccessat",
        "pselect6",
        "ppoll",
        "unshare",
        "set_robust_list",
        "get_robust_list",
        "splice",
        "tee",
        "sync_file_range",
        "vmsplice",
        "move_pages",
        "utimensat",
        "epoll_pwait",
        "signalfd",
        "timerfd_create",
        "eventfd",
        "fallocate",
        "timerfd_settime",
        "timerfd_gettime",
        "accept4",
        "signalfd4",
        "eventfd2",
        "epoll_create1",
        "dup3",
        "pipe2",
        "inotify_init1",
        "preadv",
        "pwritev",
        "rt_tgsigqueueinfo",
        "perf_event_open",
        "recvmmsg",
        "fanotify_init",
        "fanotify_mark",
        "prlimit64",
        "name_to_handle_at",
        "open_by_handle_at",
        "clock_adjtime",
        "syncfs",
        "sendmmsg",
        "setns",
        "getcpu",
        "process_vm_readv",
        "process_vm_writev",
        "kcmp",
        "finit_module",
        "sched_setattr",
        "sched_getattr",
        "renameat2",
        "seccomp",
        "getrandom",
        "memfd_create",
        "kexec_file_load",
        "bpf",
        "execveat",
        "userfaultfd",
        "membarrier",
        "mlock2",
        "copy_file_range",
        "preadv2",
        "pwritev2"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

This profile allows only safe syscalls and blocks dangerous ones like `execve`, `socket`, `fork`.

---

## 8. Monitoring and Observability

### 8.1 Key Metrics

| Metric                       | Target                  | Alert Threshold            |
|------------------------------|-------------------------|----------------------------|
| **Queue Depth**              | < 1000/worker           | > 10,000 total             |
| **Execution Latency**        | < 5s (p99)              | > 10s                      |
| **Worker Availability**      | > 90%                   | < 80%                      |
| **Verdict Distribution**     | 50% AC, 30% WA, 10% TLE | > 50% errors (investigate) |
| **Container Creation Time**  | < 1s                    | > 3s                       |
| **Test Case Cache Hit Rate** | > 95%                   | < 90%                      |
| **Database Write Latency**   | < 50ms (p99)            | > 200ms                    |
| **Security Violations**      | 0 per day               | > 10/day                   |

### 8.2 Distributed Tracing

**Trace Example:**

```
Total: 6.2s
  - API Gateway: 10ms (auth, validation)
  - Submission Service: 50ms (DB write, queue publish)
  - Queue Wait: 2s (queue depth backlog)
  - Worker Pickup: 100ms (Kafka consumer lag)
  - Container Creation: 800ms (Docker startup)
  - Compilation: 500ms (javac)
  - Execution: 2.5s (test cases)
  - Result Write: 50ms (DB + Redis)
  - Client Notification: 200ms (WebSocket push)
```

### 8.3 Logging

**Log Aggregation:**

- All workers send logs to centralized ELK stack
- Searchable by submission_id, user_id, problem_id
- Retention: 30 days (compliance)

**Sample Log Entry:**

```json
{
  "timestamp": "2025-10-31T12:00:00Z",
  "level": "INFO",
  "service": "judge-worker-5",
  "submission_id": "sub_123456",
  "user_id": "user_789",
  "problem_id": "two_sum",
  "language": "python3",
  "event": "execution_complete",
  "verdict": "Accepted",
  "runtime_ms": 234,
  "memory_kb": 15360,
  "test_cases_passed": 10,
  "test_cases_total": 10
}
```

---

## 9. Common Anti-Patterns

### ❌ **Anti-Pattern 1: Synchronous Execution**

**Problem:**
Blocking API request until execution completes (5-10 seconds) creates terrible UX and wastes connections.

**Example:**

```
POST /api/submit
  → Execute code (wait 5s)
  → Return result
Client waits 5 seconds (connection blocked)
```

**Solution:** ✅
Asynchronous processing with queue. Return submission ID immediately, client polls/subscribes for status.

---

### ❌ **Anti-Pattern 2: No Resource Limits**

**Problem:**
Malicious code consumes all server resources, affecting other submissions.

**Example:**

```python
# Memory bomb
x = [0] * 10**10  # Allocates 80 GB RAM, kills server
```

**Solution:** ✅
Enforce strict resource limits via cgroups (CPU, memory, disk, PIDs).

---

### ❌ **Anti-Pattern 3: Shared Execution Environment**

**Problem:**
Running multiple submissions in same container allows data leakage between users.

**Example:**

```
User A writes file: /tmp/secret.txt
User B reads file: /tmp/secret.txt (sees User A's data!)
```

**Solution:** ✅
Each submission gets fresh, isolated container. No state sharing between executions.

---

### ❌ **Anti-Pattern 4: Trusting User Input**

**Problem:**
Not validating code size or language allows abuse.

**Example:**

```
User submits 100 MB code file (crashes parser)
User submits code in unsupported language (execution error)
```

**Solution:** ✅
Validate code size (max 64 KB), language (whitelist), and syntax before queuing.

---

### ❌ **Anti-Pattern 5: No Timeout Enforcement**

**Problem:**
Infinite loops hang workers indefinitely.

**Example:**

```python
while True:
    pass  # Runs forever, worker stuck
```

**Solution:** ✅
Use `timeout` command or process monitoring. Kill after time limit (e.g., 5 seconds).

---

### ❌ **Anti-Pattern 6: Storing Code in Memory**

**Problem:**
Keeping all submissions in memory causes OOM.

**Example:**

```
submissions = {}  # In-memory dict
submissions[id] = code  # 10K submissions × 10KB = 100 MB
```

**Solution:** ✅
Store code in database, cache only active submissions in Redis (LRU eviction).

---

## 10. Alternative Approaches

### Alternative 1: VM-Based Isolation (AWS Lambda)

**Approach:**
Each submission runs in separate VM (AWS Lambda, Google Cloud Functions).

**Pros:**

- ✅ Stronger isolation than Docker
- ✅ Fully managed (no infrastructure)
- ✅ Auto-scaling built-in

**Cons:**

- ❌ Cold start latency (3-5 seconds)
- ❌ Higher cost ($0.20 per million invocations)
- ❌ 15-minute execution limit (too short for complex problems)

**When to Use:**

- Small-scale platform (< 1000 submissions/day)
- Budget for serverless costs
- Willing to trade latency for simplicity

**Why We Didn't Choose It:**
Cold start latency unacceptable for competitive programming (users expect < 2s feedback). Cost prohibitive at 10k
submissions/sec scale.

---

### Alternative 2: Shared Workers (No Isolation)

**Approach:**
Run all code on shared worker machines without containers.

**Pros:**

- ✅ Lowest latency (no container overhead)
- ✅ Simple implementation
- ✅ Lowest cost

**Cons:**

- ❌ Zero security (malicious code can compromise system)
- ❌ No resource isolation (one submission can DOS others)
- ❌ Data leakage risk

**When to Use:**

- Trusted users only (e.g., internal company coding challenges)
- Security not a concern
- Small scale (< 100 submissions/day)

**Why We Didn't Choose It:**
Completely unacceptable for public platform. Security breach would destroy reputation.

---

### Alternative 3: Browser-Based Execution (WebAssembly)

**Approach:**
Compile code to WebAssembly, run in user's browser.

**Pros:**

- ✅ Zero server cost (client-side execution)
- ✅ Instant feedback (no network latency)
- ✅ Perfect isolation (browser sandbox)

**Cons:**

- ❌ Limited language support (only WASM-compatible)
- ❌ Client can cheat (modify browser code)
- ❌ Performance varies by client device

**When to Use:**

- Educational platforms (security less critical)
- Simple languages (Python, JavaScript)
- Cost-sensitive (avoid server costs)

**Why We Didn't Choose It:**
Cannot trust client for competitive programming (cheating too easy). Need server-side verification.

---

## 11. Real-World Examples

### LeetCode (200M Users)

**Architecture:**

- Submission queue: RabbitMQ (100k messages/sec)
- Execution: Docker containers (3000+ workers)
- Languages: 20+ (Python, Java, C++, JavaScript, Go, Rust, etc.)
- Security: Docker + seccomp + cgroups
- Scaling: Auto-scaling based on queue depth

**Lessons:**

- Queue-based architecture critical for handling spikes
- Pre-warming workers before contests reduces latency
- Test case caching improves performance 10x

---

### HackerRank (50M Users)

**Architecture:**

- Submission queue: AWS SQS (managed queue)
- Execution: Custom sandbox (ptrace-based isolation)
- Languages: 40+ (most comprehensive)
- Security: Custom seccomp profiles per language
- Scaling: Kubernetes auto-scaling

**Lessons:**

- Custom sandbox allows finer control than Docker
- Per-language security profiles reduce false positives
- Distributed test case storage (S3 + CloudFront) essential

---

### Codeforces (3M Users)

**Architecture:**

- Submission queue: Custom in-memory queue (Redis-based)
- Execution: Polygon system (custom judging engine)
- Languages: 10+ (focus on competitive programming)
- Security: Linux namespaces + resource limits
- Scaling: Fixed worker pool (contest pre-scheduled)

**Lessons:**

- Simple queue sufficient for predictable load (contests scheduled)
- Custom judging engine optimized for competitive programming
- Fixed worker pool viable when traffic predictable

---

## 12. Future Enhancements

### 13.1 AI-Powered Code Review

**Feature:**

- Analyze code for common mistakes
- Suggest optimizations (time/space complexity)
- Detect code smells

**Implementation:**

- Run static analysis after submission
- Use ML model trained on millions of solutions
- Show hints without revealing solution

---

### 13.2 Live Collaborative Debugging

**Feature:**

- Multiple users debug code together in real-time
- Share execution state (variables, stack trace)
- Like Google Docs for debugging

**Implementation:**

- WebSocket-based collaborative editor
- Shared execution environment
- Cursor tracking + presence

---

### 13.3 GPU-Accelerated Judging

**Feature:**

- Support ML/AI problems requiring GPU
- PyTorch, TensorFlow problem sets

**Implementation:**

- GPU-enabled worker pool (g4 instances)
- CUDA containers
- Higher resource limits (16 GB GPU RAM)

---

## 13. Interview Discussion Points

### Key Topics to Discuss

1. **Security (Primary Focus):**
    - Why Docker over VMs?
    - Defense in depth (seccomp, AppArmor, cgroups)
    - Attack vectors and mitigations

2. **Scalability:**
    - Queue-driven auto-scaling
    - Why queue depth (not CPU) for scaling trigger?
    - Handling 10x traffic spikes

3. **Asynchronous Architecture:**
    - Why queue-based?
    - Trade-offs vs synchronous execution
    - Retry mechanisms

4. **Real-Time Feedback:**
    - Polling vs WebSocket
    - How to scale WebSocket connections?

5. **Resource Management:**
    - How to prevent fork bomb?
    - Memory limits enforcement
    - Timeout handling

**Follow-Up Questions to Expect:**

- "How do you prevent container escape?"
    - Answer: Defense in depth (seccomp, AppArmor, read-only fs, network isolation)

- "What if a submission takes 10 minutes?"
    - Answer: Timeout enforcement (5-10s limit), kill process, return TLE verdict

- "How to handle 100k submissions/sec?"
    - Answer: Scale workers to 1000+ servers, queue absorbs spike, pre-warm for contests

- "Can users see test cases?"
    - Answer: No, test cases not exposed to container, results only shown after judging

---

## 14. Trade-offs Summary

| What We Gain                                       | What We Sacrifice                                            |
|----------------------------------------------------|--------------------------------------------------------------|
| ✅ **Security** (Docker isolation)                  | ❌ **Latency overhead** (container startup 0.5-1s)            |
| ✅ **Scalability** (queue-based, horizontal)        | ❌ **Complexity** (many moving parts)                         |
| ✅ **Fault tolerance** (queue persistence, retries) | ❌ **Cost** (100+ servers for peak)                           |
| ✅ **Language support** (20+ languages)             | ❌ **Docker image size** (5 GB with all runtimes)             |
| ✅ **Real-time feedback** (WebSocket)               | ❌ **Stateful WebSocket servers** (harder to scale)           |
| ✅ **Fair resource allocation** (cgroups)           | ❌ **Reduced performance** (limits prevent optimal CPU usage) |

---

## 15. References

### Related System Design Components

- **[2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)** -
  WebSocket for real-time status
- **[2.3.1 Asynchronous Communication](../../02-components/2.3-messaging-streaming/2.3.1-asynchronous-communication.md)
  ** - Queue-based architecture
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Submission queue
  implementation
- **[1.1.2 Latency, Throughput, Scale](../../01-principles/1.1.2-latency-throughput-scale.md)** - Scaling strategies
- **[2.4.1 Security Fundamentals](../../02-components/2.4-security-observability/2.4.1-security-fundamentals.md)** -
  Security principles, sandboxing
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - Database sharding strategies
- **[2.2.1 Caching Deep Dive](../../02-components/2.2.1-caching-deep-dive.md)** - Caching patterns

### Related Design Challenges

- **[3.4.1 Stock Exchange](../3.4.1-stock-exchange/)** - Low-latency processing patterns
- **[3.2.3 Web Crawler](../3.2.3-web-crawler/)** - Distributed worker architecture
- **[3.4.6 Collaborative Editor](../3.4.6-collaborative-editor/)** - Real-time processing patterns

### External Resources

- **Docker Documentation:** [Docker Security](https://docs.docker.com/engine/security/) - Container isolation
- **Kubernetes:** [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) - CPU and memory limits
- **LeetCode Engineering:** [System Design Blog](https://leetcode.com/discuss/) - Code execution architecture
- **HackerRank Engineering:** [Scaling Code Execution](https://www.hackerrank.com/engineering/) - Worker fleet
  management

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Queue-based architectures, distributed systems
- *The Linux Programming Interface* by Michael Kerrisk - Process isolation, cgroups, namespaces
