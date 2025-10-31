# Online Code Judge - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made when designing an Online Code Judge
system like LeetCode or HackerRank, including isolation strategies, queue choices, scaling approaches, and security
mechanisms.

---

## Table of Contents

1. [Docker Containers vs Virtual Machines vs Bare Metal](#1-docker-containers-vs-virtual-machines-vs-bare-metal)
2. [Kafka vs SQS vs RabbitMQ for Submission Queue](#2-kafka-vs-sqs-vs-rabbitmq-for-submission-queue)
3. [Synchronous vs Asynchronous Judging](#3-synchronous-vs-asynchronous-judging)
4. [Multi-Language Support: Multi-Image vs Single Image](#4-multi-language-support-multi-image-vs-single-image)
5. [Test Case Storage: S3 vs Database vs In-Memory](#5-test-case-storage-s3-vs-database-vs-in-memory)
6. [Auto-Scaling Metric: Queue Depth vs CPU Utilization](#6-auto-scaling-metric-queue-depth-vs-cpu-utilization)
7. [Security: Seccomp + AppArmor vs gVisor vs Firecracker](#7-security-seccomp--apparmor-vs-gvisor-vs-firecracker)
8. [Real-Time Updates: WebSockets vs Server-Sent Events vs Polling](#8-real-time-updates-websockets-vs-server-sent-events-vs-polling)
9. [Verdict Storage: PostgreSQL vs DynamoDB vs Redis](#9-verdict-storage-postgresql-vs-dynamodb-vs-redis)
10. [Contest vs Practice: Separate vs Shared Infrastructure](#10-contest-vs-practice-separate-vs-shared-infrastructure)

---

## 1. Docker Containers vs Virtual Machines vs Bare Metal

### The Problem

Submitted code is untrusted and potentially malicious. We need absolute security isolation to prevent:

- Accessing host filesystem or other containers
- Launching network attacks
- Consuming excessive resources (CPU, memory, disk)
- Interfering with other submissions

However, we also need:

- Fast startup time (< 1 second)
- High throughput (1000s of executions/second)
- Cost efficiency

### Options Considered

| Approach                   | Security                         | Startup Time | Resource Overhead  | Cost     | Complexity  |
|----------------------------|----------------------------------|--------------|--------------------|----------|-------------|
| **Bare Metal**             | ❌ Terrible (no isolation)        | ✅ 0ms        | ✅ Zero             | ✅ Lowest | ✅ Simple    |
| **Docker Containers**      | ⚠️ Good (namespace isolation)    | ✅ 800ms      | ✅ Low (100 MB)     | ✅ Low    | ✅ Moderate  |
| **Virtual Machines (KVM)** | ✅ Excellent (hardware isolation) | ❌ 5-10s      | ❌ High (1 GB+)     | ❌ High   | ⚠️ Moderate |
| **AWS Firecracker**        | ✅ Excellent (microVM)            | ✅ 125ms      | ✅ Low (5 MB)       | ✅ Low    | ❌ Complex   |
| **gVisor**                 | ✅ Excellent (user-space kernel)  | ⚠️ 2s        | ⚠️ Medium (200 MB) | ✅ Low    | ❌ Complex   |

### Decision Made

**Docker Containers with Seccomp + AppArmor + Cgroups**

### Rationale

1. **Fast Startup (800ms)**: Much faster than VMs, acceptable for judging
2. **Good Security**: Namespace isolation + mandatory security layers (seccomp, AppArmor, cgroups) provide strong
   protection
3. **Low Overhead**: 100 MB per container vs 1 GB+ for VMs
4. **Cost Efficient**: Run 50+ containers per host vs 10 VMs
5. **Mature Ecosystem**: Docker tooling, orchestration (Kubernetes), monitoring well-established

**Security Layers Applied:**

```
Docker Container
├── Namespace Isolation (PID, network, mount, UTS, IPC)
├── Seccomp Profile (whitelist syscalls)
├── AppArmor Policy (file access restrictions)
├── Cgroups (CPU, memory, PIDs limits)
├── Read-Only Filesystem (only /tmp writable)
└── Network Isolation (--network none)
```

### Implementation Details

**Docker Run Command:**

```
docker run --rm \
  --network none \
  --memory 512m \
  --cpus 1.0 \
  --pids-limit 100 \
  --read-only \
  --tmpfs /tmp:size=100m,noexec \
  --security-opt seccomp=strict.json \
  --security-opt apparmor=judge \
  --security-opt no-new-privileges \
  --cap-drop ALL \
  judge-runtime:python3.9
```

**Seccomp Profile (strict.json):**

```
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {"names": ["read", "write", "open", "close", "stat", "fstat", "mmap", "brk", "exit_group"], "action": "SCMP_ACT_ALLOW"}
  ]
}
```

### Trade-offs Accepted

| What We Gain                           | What We Sacrifice                                        |
|----------------------------------------|----------------------------------------------------------|
| ✅ Fast startup (800ms)                 | ❌ Slightly weaker isolation than VMs (kernel shared)     |
| ✅ High density (50 containers/host)    | ❌ Potential kernel vulnerabilities affect all containers |
| ✅ Low cost ($0.10 per 1000 executions) | ❌ Complex security configuration required                |
| ✅ Mature ecosystem                     | ❌ Must maintain custom seccomp/AppArmor profiles         |

### When to Reconsider

**Switch to Firecracker if:**

- Security incidents occur (container escapes)
- Need hardware-level isolation
- Budget allows (AWS Lambda pricing)
- Willing to manage custom infrastructure

**Switch to VMs if:**

- Extremely sensitive workloads (financial transactions)
- Compliance requirements demand hardware isolation
- Startup time < 10s acceptable

### Real-World Examples

- **LeetCode**: Uses Docker containers with custom seccomp profiles
- **HackerRank**: Uses Docker containers with gVisor for added security
- **AWS Lambda**: Uses Firecracker microVMs (hardware isolation)
- **Google Cloud Run**: Uses gVisor (user-space kernel)

---

## 2. Kafka vs SQS vs RabbitMQ for Submission Queue

### The Problem

Judging is a slow, CPU-bound operation (avg 5 seconds). We need a queue to:

- Buffer traffic spikes (10k submissions/sec during contests)
- Decouple submission acceptance from execution
- Guarantee at-least-once delivery (no lost submissions)
- Support priority queuing (contest vs practice)

### Options Considered

| Feature                    | Kafka                       | AWS SQS                 | RabbitMQ                    |
|----------------------------|-----------------------------|-------------------------|-----------------------------|
| **Throughput**             | ✅ 1M+ msg/sec               | ⚠️ 3k msg/sec per queue | ⚠️ 50k msg/sec              |
| **Latency**                | ✅ < 10ms                    | ❌ 100ms+ (polling)      | ✅ < 10ms (push)             |
| **Ordering**               | ✅ Per-partition             | ❌ Best-effort           | ⚠️ Per-queue only           |
| **Durability**             | ✅ Persistent (disk)         | ✅ Persistent            | ✅ Persistent (optional)     |
| **Scalability**            | ✅ Horizontal (partitions)   | ✅ Fully managed         | ⚠️ Vertical (single broker) |
| **Priority Queue**         | ⚠️ Multiple topics          | ✅ Native support        | ✅ Native support            |
| **Cost**                   | ⚠️ $0.10/GB                 | ✅ $0.40/million         | ⚠️ Self-hosted              |
| **Operational Complexity** | ❌ High (ZooKeeper, brokers) | ✅ Zero (managed)        | ⚠️ Moderate                 |

### Decision Made

**Kafka for High Throughput + SQS for Simplicity (Hybrid)**

**Primary Queue: Kafka**

- Used for high-throughput scenarios (contests, peak traffic)
- Provides ordering guarantees (partition by problem_id)
- Supports multi-consumer pattern (multiple worker groups)

**Secondary Queue: SQS**

- Used for low-traffic scenarios (practice submissions)
- Fully managed, zero operational overhead
- Priority queue support (FIFO queues with message groups)

### Rationale

1. **Throughput**: Kafka handles 1M+ msg/sec (10x more than SQS)
2. **Ordering**: Partition by problem_id ensures test case data locality
3. **Scalability**: Horizontal scaling via partitions (add workers seamlessly)
4. **Replay Capability**: Can replay submissions for debugging (persistent log)
5. **Cost**: $0.10/GB vs $0.40/million messages (cheaper at scale)

**Why NOT SQS Alone:**

- SQS has 3k msg/sec per queue limit (need 4 queues for 10k QPS)
- No ordering guarantees (FIFO queues limited to 300 msg/sec)
- Higher latency (100ms+ polling vs 10ms Kafka)
- No partition-based locality (can't route same problem_id to same worker)

**Why NOT RabbitMQ:**

- Vertical scaling only (single broker bottleneck)
- Less mature operational tooling than Kafka
- No native partitioning (must manually shard)

### Implementation Details

**Kafka Topic Structure:**

```
submissions.high_priority (10 partitions)
├── partition 0: problem_id % 10 == 0
├── partition 1: problem_id % 10 == 1
├── ...
└── partition 9: problem_id % 10 == 9

submissions.normal (50 partitions)
├── partition 0-49: hash(problem_id) % 50
```

**Producer Code:**

```
// Publish submission
kafka.produce({
  topic: is_contest ? 'submissions.high_priority' : 'submissions.normal',
  partition: hash(problem_id) % num_partitions,
  key: submission_id,
  value: {
    submission_id,
    user_id,
    problem_id,
    code,
    language,
    timestamp
  }
});
```

**Consumer Code:**

```
// Worker consumes from assigned partitions
kafka.subscribe(['submissions.high_priority', 'submissions.normal']);

kafka.on('message', async (message) => {
  const submission = JSON.parse(message.value);
  
  // Process submission
  const verdict = await executeSubmission(submission);
  
  // Commit offset only after success
  kafka.commitOffset(message);
});
```

### Trade-offs Accepted

| What We Gain                                 | What We Sacrifice                                  |
|----------------------------------------------|----------------------------------------------------|
| ✅ 1M+ msg/sec throughput                     | ❌ High operational complexity (ZooKeeper, brokers) |
| ✅ Ordering guarantees (partition-level)      | ❌ Must manage partitions (add/remove complexity)   |
| ✅ Replay capability (debugging)              | ❌ Storage cost (retain 7 days = 100 GB/day)        |
| ✅ Data locality (same problem → same worker) | ❌ Rebalancing delay (30s when workers added)       |

### When to Reconsider

**Switch to SQS if:**

- Traffic < 10k QPS (SQS simpler for low traffic)
- Ordering not critical
- Want zero operational overhead
- Team lacks Kafka expertise

**Switch to RabbitMQ if:**

- Need complex routing (topic exchanges, routing keys)
- Already using RabbitMQ in infrastructure
- Traffic < 50k QPS

### Real-World Examples

- **LeetCode**: Uses Kafka for submissions (high throughput)
- **HackerRank**: Uses RabbitMQ with priority queues
- **Codeforces**: Uses custom queue (Redis-based)
- **TopCoder**: Uses SQS for simplicity

---

## 3. Synchronous vs Asynchronous Judging

### The Problem

Judging takes 5 seconds on average. Should the API Gateway wait for the result (synchronous) or return immediately and
notify later (asynchronous)?

**Constraints:**

- User expects fast feedback (< 1 second ideal)
- Execution takes 5 seconds (too slow to wait)
- High traffic during contests (10k QPS)
- Some users want instant results (blocking)

### Options Considered

| Approach         | User Experience    | Throughput                   | API Gateway Load         | Complexity  |
|------------------|--------------------|------------------------------|--------------------------|-------------|
| **Synchronous**  | ✅ Immediate result | ❌ Low (1 request = 1 worker) | ❌ High (threads blocked) | ✅ Simple    |
| **Asynchronous** | ⚠️ Delayed result  | ✅ High (decoupled)           | ✅ Low (return instantly) | ⚠️ Moderate |
| **Hybrid**       | ✅ Best of both     | ✅ High                       | ✅ Low                    | ❌ Complex   |

### Decision Made

**Asynchronous Judging with Optional WebSocket Real-Time Updates**

### Rationale

1. **High Throughput**: API Gateway not blocked, can handle 100k submissions/sec
2. **Better Resource Utilization**: Workers process at their own pace (5s/submission)
3. **Resilience**: Queue buffers traffic spikes (submissions don't fail if workers busy)
4. **Fairness**: All submissions queued in order (FIFO or priority)
5. **Scalability**: Workers auto-scale independently of API Gateway

**Why NOT Synchronous:**

- API Gateway threads blocked for 5 seconds (10k threads needed for 10k QPS)
- No buffering (if all workers busy, requests fail immediately)
- Poor user experience (5 second wait, browser timeout at 30s)
- Expensive (need massive API Gateway fleet)

### Implementation Details

**Submission Flow:**

1. **User submits** → API Gateway (50ms)
2. **Gateway publishes to Kafka** → Returns submission_id (100ms total)
3. **User polls or subscribes** → GET /api/submissions/{id} or WebSocket
4. **Worker processes** → Updates PostgreSQL + Redis (5 seconds)
5. **User receives verdict** → Polling returns result or WebSocket push

**API Endpoints:**

```
POST /api/submit
→ Returns: {submission_id, status: "Queued"}

GET /api/submissions/{id}
→ Returns: {submission_id, status: "Running", test: 5/10}

WebSocket /ws/submissions/{id}
→ Pushes: {status: "Completed", verdict: "Accepted"}
```

**Client Code (Polling):**

```
async function submitCode(code) {
  // Submit
  const {submission_id} = await fetch('/api/submit', {
    method: 'POST',
    body: JSON.stringify({code, language: 'python3'})
  }).then(r => r.json());
  
  // Poll every 1 second
  while (true) {
    const {status, verdict} = await fetch(`/api/submissions/${submission_id}`)
      .then(r => r.json());
    
    if (status === 'Completed') {
      console.log('Verdict:', verdict);
      break;
    }
    
    await sleep(1000);
  }
}
```

**Client Code (WebSocket):**

```
async function submitCode(code) {
  // Submit
  const {submission_id} = await fetch('/api/submit', {
    method: 'POST',
    body: JSON.stringify({code, language: 'python3'})
  }).then(r => r.json());
  
  // Connect WebSocket
  const ws = new WebSocket(`ws://judge.com/submissions/${submission_id}`);
  
  ws.onmessage = (event) => {
    const {status, verdict} = JSON.parse(event.data);
    console.log('Status:', status, 'Verdict:', verdict);
    
    if (status === 'Completed') {
      ws.close();
    }
  };
}
```

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice                         |
|--------------------------------------|-------------------------------------------|
| ✅ High throughput (100k QPS)         | ❌ Delayed result (5 seconds vs immediate) |
| ✅ Low API Gateway load               | ❌ Client must poll or maintain WebSocket  |
| ✅ Resilience (queue buffers spikes)  | ❌ More complex client code                |
| ✅ Auto-scaling workers independently | ❌ Must maintain WebSocket infrastructure  |

### When to Reconsider

**Switch to Synchronous if:**

- Execution time < 500ms (fast enough to wait)
- Low traffic (< 100 QPS)
- Simple client requirements (no polling)
- Use case demands immediate response

**Hybrid Approach if:**

- Want both immediate feedback AND high throughput
- Long polling with timeout (wait 5s, then timeout and poll)
- Server-Sent Events (SSE) for one-way updates

### Real-World Examples

- **LeetCode**: Asynchronous with WebSocket updates
- **HackerRank**: Asynchronous with polling (GET /api/submissions/{id})
- **Codeforces**: Asynchronous with real-time scoreboard
- **TopCoder**: Synchronous (old system, low traffic)

---

## 4. Multi-Language Support: Multi-Image vs Single Image

### The Problem

Support 20+ languages (Python, Java, C++, JavaScript, Go, Rust, etc.). Each language needs:

- Compiler/interpreter (gcc, javac, python3, node, rustc)
- Standard libraries
- Different versions (Python 3.8, 3.9, 3.10)

**Challenges:**

- Image size (each language adds 100-500 MB)
- Build time (multi-stage builds complex)
- Deployment (update one language → redeploy all workers)
- Resource usage (unused languages waste memory)

### Options Considered

| Approach                       | Image Size | Build Time | Deployment            | Flexibility | Maintenance |
|--------------------------------|------------|------------|-----------------------|-------------|-------------|
| **Single Monolithic Image**    | ❌ 5 GB+    | ❌ 30 min   | ❌ Slow (redeploy all) | ❌ Low       | ❌ Complex   |
| **Multi-Image (per language)** | ✅ 500 MB   | ✅ 5 min    | ✅ Fast (redeploy one) | ✅ High      | ✅ Simple    |
| **Dynamic Image Layers**       | ⚠️ 2 GB    | ⚠️ 15 min  | ✅ Fast                | ✅ High      | ❌ Complex   |

### Decision Made

**Multi-Image Strategy with Dynamic Layer Caching**

**Approach:**

- **Base Image**: judge-runtime-base (Ubuntu 22.04, common tools) - 200 MB
- **Language Images**: judge-runtime:python3.9, judge-runtime:java11, etc. - 500 MB each
- **Worker Pods**: Specify which languages to support (e.g., worker-python, worker-cpp)

### Rationale

1. **Small Image Size**: 500 MB per language vs 5 GB monolithic
2. **Fast Builds**: Update Python → rebuild only Python image (5 min vs 30 min)
3. **Fast Deployment**: Deploy python workers → only Python pods restarted
4. **Resource Efficient**: Workers only load needed languages (no wasted memory)
5. **Version Isolation**: Python 3.8 and 3.9 in separate images (no conflicts)

**Why NOT Monolithic Image:**

- 5 GB+ image size (slow to pull, costly storage)
- 30 minute build time (slow CI/CD)
- Update Java → all workers restarted (including Python, C++)
- All workers load all languages (wasted RAM)

### Implementation Details

**Dockerfile Structure:**

```
# Base image (common tools)
FROM ubuntu:22.04 AS base
RUN apt-get update && apt-get install -y \
    ca-certificates curl wget git \
    && rm -rf /var/lib/apt/lists/*

# Python image
FROM base AS python
RUN apt-get update && apt-get install -y python3.9 python3-pip \
    && rm -rf /var/lib/apt/lists/*
COPY seccomp-python.json /etc/seccomp/python.json

# Java image
FROM base AS java
RUN apt-get update && apt-get install -y openjdk-11-jdk \
    && rm -rf /var/lib/apt/lists/*
COPY seccomp-java.json /etc/seccomp/java.json

# C++ image
FROM base AS cpp
RUN apt-get update && apt-get install -y g++ \
    && rm -rf /var/lib/apt/lists/*
COPY seccomp-cpp.json /etc/seccomp/cpp.json
```

**Worker Deployment:**

```
# Python workers
kubectl apply -f worker-python.yaml
→ Uses image: judge-runtime:python3.9

# Java workers
kubectl apply -f worker-java.yaml
→ Uses image: judge-runtime:java11

# Multi-language workers (if needed)
kubectl apply -f worker-all.yaml
→ Uses images: [python3.9, java11, cpp17]
```

**Dynamic Layer Caching:**

```
# Build pipeline (CI/CD)
1. Build base image → Cache layer
2. Build python3.9 → Use cached base
3. Build java11 → Use cached base
4. Build cpp17 → Use cached base

# Update Python only
1. Change Dockerfile.python
2. Build python3.9 → Use cached base (fast!)
3. Deploy worker-python pods only
```

### Trade-offs Accepted

| What We Gain                               | What We Sacrifice                           |
|--------------------------------------------|---------------------------------------------|
| ✅ Small images (500 MB vs 5 GB)            | ❌ Must maintain multiple Dockerfiles        |
| ✅ Fast builds (5 min vs 30 min)            | ❌ More complex deployment (multiple images) |
| ✅ Fast deployment (redeploy one language)  | ❌ Workers must specify supported languages  |
| ✅ Resource efficient (no unused languages) | ❌ Must manage image registry (20+ images)   |

### When to Reconsider

**Switch to Monolithic if:**

- Support < 5 languages (small image size acceptable)
- All submissions need all languages (no waste)
- Simple deployment preferred
- Team small (can't maintain multiple images)

**Switch to Dynamic Layers if:**

- Need extreme customization (user-installed libraries)
- Serverless environment (AWS Lambda, Cloud Run)
- Image size not critical (fast network)

### Real-World Examples

- **LeetCode**: Multi-image (separate Python, Java, C++ images)
- **HackerRank**: Dynamic layers (base + language layers pulled on demand)
- **Codeforces**: Monolithic image (all languages, custom environment)
- **TopCoder**: Multi-image with on-demand loading

---

## 5. Test Case Storage: S3 vs Database vs In-Memory

### The Problem

Test cases for 10,000 problems:

- Avg size: 1 MB per problem (input + expected output)
- Total size: 10 GB
- Access pattern: Read-heavy (1M reads/sec), write-rare (update monthly)
- Performance: Must load test cases in < 100ms

### Options Considered

| Approach               | Read Latency     | Throughput    | Cost        | Durability | Complexity  |
|------------------------|------------------|---------------|-------------|------------|-------------|
| **S3 + CloudFront**    | ✅ 50ms (CDN hit) | ✅ 1M req/sec  | ✅ $0.02/GB  | ✅ 11-nines | ✅ Simple    |
| **PostgreSQL BYTEA**   | ⚠️ 100ms         | ❌ 10k req/sec | ⚠️ $0.10/GB | ✅ High     | ⚠️ Moderate |
| **Redis In-Memory**    | ✅ 1ms            | ✅ 1M req/sec  | ❌ $0.50/GB  | ❌ Volatile | ⚠️ Moderate |
| **Local Worker Cache** | ✅ 5ms (SSD)      | ✅ Unlimited   | ✅ Free      | ❌ Volatile | ✅ Simple    |

### Decision Made

**Three-Tier Caching: Local SSD → CloudFront → S3**

**Architecture:**

```
Worker Local Cache (L1)
├── Hit Rate: 95%
├── Latency: 5ms
└── Size: 1 GB SSD per worker

CloudFront (L2)
├── Hit Rate: 99% (of misses)
├── Latency: 50ms
└── Size: Unlimited (edge caches)

S3 Origin (L3)
├── Hit Rate: 100%
├── Latency: 200ms
└── Size: Unlimited
```

### Rationale

1. **Low Latency**: 95% hits from local cache (5ms) vs 50ms CloudFront vs 200ms S3
2. **High Throughput**: Local cache unlimited, no network bottleneck
3. **Cost Efficient**: S3 storage $0.02/GB, CloudFront $0.085/GB transfer
4. **Durability**: S3 provides 11-nines durability (no data loss)
5. **Scalability**: Add more workers → cache scales horizontally

**Why NOT PostgreSQL:**

- Slow for large BLOBs (100ms vs 5ms local cache)
- Low throughput (10k req/sec vs 1M req/sec local)
- Expensive (provisioned IOPS for high throughput)
- Database not designed for static file storage

**Why NOT Redis Alone:**

- Expensive ($0.50/GB vs $0.02/GB S3)
- Volatile (data lost on restart, need persistence)
- Must warm cache on startup (cold start problem)
- Not suitable for 10 GB dataset (Redis best < 1 GB)

### Implementation Details

**Worker Cache Logic:**

```
function getTestCases(problem_id):
  // L1: Check local cache
  if file_exists('/var/cache/testcases/' + problem_id):
    return read_from_disk(problem_id)  // 5ms
  
  // L2: Fetch from CloudFront
  url = 'https://cdn.judge.com/testcases/' + problem_id + '.tar.gz'
  response = http_get(url, timeout=1s)
  
  if response.status == 200:
    // Extract and cache locally
    extract_tar_gz(response.body, '/var/cache/testcases/' + problem_id)
    return read_from_disk(problem_id)  // 50ms total
  
  // L3: Fetch from S3 (CloudFront miss)
  s3_response = s3.get_object(Bucket='judge-testcases', Key=problem_id + '.tar.gz')
  extract_tar_gz(s3_response.body, '/var/cache/testcases/' + problem_id)
  return read_from_disk(problem_id)  // 200ms total
```

**Cache Invalidation:**

```
// When problem updated (rare)
redis.publish('problem:updated', problem_id)

// All workers subscribe
redis.subscribe('problem:updated', (problem_id) => {
  delete_cache('/var/cache/testcases/' + problem_id)
})
```

**S3 Structure:**

```
s3://judge-testcases/
├── problem_1.tar.gz (1 MB)
├── problem_2.tar.gz (1 MB)
├── problem_3.tar.gz (1 MB)
├── ...
└── problem_10000.tar.gz (1 MB)

Each tar.gz contains:
├── input_1.txt
├── output_1.txt
├── input_2.txt
├── output_2.txt
├── ...
└── metadata.json
```

### Trade-offs Accepted

| What We Gain                                  | What We Sacrifice                                      |
|-----------------------------------------------|--------------------------------------------------------|
| ✅ 5ms latency (95% cache hit)                 | ❌ Cache warm-up on worker start (1-2 min)              |
| ✅ High throughput (unlimited local)           | ❌ Stale cache possible (rare, handled by invalidation) |
| ✅ Low cost ($0.02/GB S3 + $0.085/GB transfer) | ❌ Must manage cache invalidation (Redis Pub/Sub)       |
| ✅ Durability (S3 11-nines)                    | ❌ Local cache lost on worker restart (acceptable)      |

### When to Reconsider

**Switch to PostgreSQL if:**

- Small dataset (< 1 GB, fits in memory)
- Need ACID transactions (update test cases atomically)
- Already using PostgreSQL (no new infrastructure)

**Switch to Redis if:**

- Dataset < 1 GB (affordable in memory)
- Need sub-millisecond latency (1ms vs 5ms)
- Can afford cost ($0.50/GB vs $0.02/GB)

**Switch to Database (PostgreSQL BYTEA) if:**

- Need rich querying (SELECT * WHERE problem_difficulty = 'hard')
- Test cases < 100 KB each (not large BLOBs)
- Low traffic (< 10k req/sec)

### Real-World Examples

- **LeetCode**: S3 + CloudFront + worker local cache
- **HackerRank**: S3 + Redis cache (no local cache)
- **Codeforces**: Database (PostgreSQL) + application-level cache
- **TopCoder**: NFS shared filesystem (old approach)

---

## 6. Auto-Scaling Metric: Queue Depth vs CPU Utilization

### The Problem

Workers must auto-scale to handle traffic spikes (10k QPS during contests). Which metric should trigger scaling?

**Constraints:**

- Judging is CPU-bound (95% CPU during execution)
- Queue depth indicates backlog (how many submissions waiting)
- Want fast scale-up (< 2 minutes) and slow scale-down (avoid flapping)

### Options Considered

| Metric                           | Responsiveness         | Predictiveness           | Accuracy | Complexity  |
|----------------------------------|------------------------|--------------------------|----------|-------------|
| **CPU Utilization**              | ⚠️ Slow (lags reality) | ❌ Poor (reactive)        | ⚠️ OK    | ✅ Simple    |
| **Queue Depth**                  | ✅ Fast (proactive)     | ✅ Excellent (predictive) | ✅ High   | ⚠️ Moderate |
| **Queue Time**                   | ✅ Fast                 | ✅ Good                   | ✅ High   | ⚠️ Moderate |
| **Custom Metric (Queue/Worker)** | ✅ Fast                 | ✅ Excellent              | ✅ High   | ❌ Complex   |

### Decision Made

**Queue Depth (Messages Per Worker) as Primary Metric**

**Formula:**

```
desired_workers = queue_depth / target_messages_per_worker
where target_messages_per_worker = 1000
```

### Rationale

1. **Proactive Scaling**: Queue depth indicates future load (predictive vs reactive)
2. **Fast Response**: Detects traffic spike immediately (CPU lags by 30-60s)
3. **Accurate**: Directly measures work to be done (queue depth) vs proxy (CPU)
4. **Simple**: Single metric (vs complex multi-metric rules)

**Why NOT CPU Utilization:**

- **Lagging Indicator**: CPU high AFTER workers already overwhelmed
- **Slow Response**: Takes 30-60s to detect high CPU → scale-up too late
- **Inaccurate During Spike**: CPU 95% could mean 10 workers or 100 workers (doesn't indicate backlog)
- **Scale-Down Problem**: CPU drops immediately when work done, but queue still has backlog

**Example Scenario:**

```
T0: Normal load
- Queue: 8,000 messages
- Workers: 10
- CPU: 80% (healthy)

T1: Contest starts (10k QPS spike)
- Queue: 8,000 → 50,000 (42k added in 30s)
- Workers: 10
- CPU: 80% → 95% (workers now overloaded)

CPU-Based Scaling:
- Detects high CPU at T1 + 30s
- Starts scale-up at T1 + 60s
- Workers ready at T1 + 150s (90s launch time)
- Queue grew to 100,000 during wait
- Result: Long delays (10+ min to process backlog)

Queue-Based Scaling:
- Detects queue spike at T1 + 15s (Prometheus scrape)
- Starts scale-up at T1 + 15s
- Workers ready at T1 + 105s
- Queue stabilizes at 50,000
- Result: Manageable delays (5 min to process backlog)
```

### Implementation Details

**Prometheus Metric:**

```
kafka_consumer_lag{topic="submissions", partition="*"}
→ Total messages in queue not yet processed
```

**Kubernetes HPA:**

```
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
        selector:
          matchLabels:
            topic: submissions
      target:
        type: AverageValue
        averageValue: "1000"  # 1000 messages per worker
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Double workers if needed
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
      - type: Percent
        value: 10  # Reduce by 10% max
        periodSeconds: 60
```

**Scale-Up Logic:**

```
# Every 15 seconds
current_queue_depth = kafka.get_consumer_lag('submissions')
current_workers = kubernetes.get_replica_count('judge-worker')

messages_per_worker = current_queue_depth / current_workers

if messages_per_worker > 1000:
  desired_workers = current_queue_depth / 1000
  kubernetes.scale('judge-worker', desired_workers)
```

### Trade-offs Accepted

| What We Gain                               | What We Sacrifice                                |
|--------------------------------------------|--------------------------------------------------|
| ✅ Fast scale-up (detect spike in 15s)      | ❌ Must maintain queue depth metrics              |
| ✅ Predictive (queue indicates future load) | ❌ More complex than CPU metric                   |
| ✅ Accurate (measures actual work)          | ❌ Requires external metrics (Prometheus)         |
| ✅ Prevents queue growth                    | ❌ Potential over-scaling if queue not processing |

### When to Reconsider

**Switch to CPU if:**

- Simple requirements (no complex auto-scaling)
- Low traffic (queue rarely grows)
- No access to queue metrics (Kafka lag not available)

**Switch to Queue Time if:**

- SLA based on wait time (e.g., "95th percentile < 5s")
- Want to optimize user experience directly

**Hybrid Approach if:**

- Use queue depth for scale-up (proactive)
- Use CPU for scale-down (reactive, safe)

### Real-World Examples

- **LeetCode**: Queue depth (Kafka consumer lag)
- **HackerRank**: Queue depth + queue time (hybrid)
- **Codeforces**: CPU utilization (simple, low traffic)
- **AWS Lambda**: Concurrent executions (similar to queue depth)

---

## 7. Security: Seccomp + AppArmor vs gVisor vs Firecracker

### The Problem

Submitted code is untrusted and potentially malicious. We need multiple layers of security to prevent:

- Container escape (access host kernel)
- Network attacks (DDoS, port scanning)
- Resource exhaustion (fork bombs, OOM)
- File access (read /etc/passwd, write to /tmp)

### Options Considered

| Approach                        | Security Level | Performance Overhead | Complexity  | Cost      |
|---------------------------------|----------------|----------------------|-------------|-----------|
| **Docker + Seccomp + AppArmor** | ⚠️ Good        | ✅ Low (5%)           | ⚠️ Moderate | ✅ Low     |
| **gVisor (Runsc)**              | ✅ Excellent    | ⚠️ Medium (30%)      | ❌ High      | ✅ Low     |
| **Firecracker (MicroVM)**       | ✅ Excellent    | ✅ Low (10%)          | ❌ Very High | ⚠️ Medium |
| **Kata Containers**             | ✅ Excellent    | ⚠️ Medium (20%)      | ❌ Very High | ⚠️ Medium |

### Decision Made

**Docker + Seccomp + AppArmor + Cgroups (Multi-Layer Defense)**

**Security Layers:**

1. **Namespace Isolation**: PID, network, mount, UTS, IPC, user namespaces
2. **Seccomp**: Whitelist allowed syscalls (read, write, open, close, stat, mmap)
3. **AppArmor**: Deny access to sensitive files (/etc, /proc, /sys)
4. **Cgroups**: Limit CPU, memory, PIDs, disk I/O
5. **Capabilities**: Drop all capabilities (CAP_NET_RAW, CAP_SYS_ADMIN, etc.)
6. **Read-Only Filesystem**: Only /tmp writable
7. **Network Isolation**: --network none (no internet access)

### Rationale

1. **Good Security**: Multiple layers provide defense-in-depth
2. **Low Overhead**: 5% performance impact vs 30% for gVisor
3. **Proven Technology**: Used by Docker, LXC, systemd for years
4. **Cost Efficient**: No additional infrastructure (vs Firecracker)
5. **Mature Tooling**: Easy to debug, monitor, audit

**Why NOT gVisor:**

- 30% performance overhead (guest kernel in userspace)
- Complex deployment (custom runtime)
- Slower syscalls (every syscall intercepted)
- Harder to debug (opaque runtime)

**Why NOT Firecracker:**

- Very complex (requires custom orchestration)
- Higher cost (microVMs need more RAM)
- Overkill for most use cases
- Best for AWS Lambda (managed environment)

### Implementation Details

**Seccomp Profile:**

```
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    {
      "names": [
        "read", "write", "open", "close", "stat", "fstat", "lstat",
        "poll", "lseek", "mmap", "mprotect", "munmap", "brk", "rt_sigaction",
        "rt_sigprocmask", "rt_sigreturn", "ioctl", "pread64", "pwrite64",
        "readv", "writev", "access", "pipe", "select", "sched_yield",
        "mremap", "msync", "mincore", "madvise", "dup", "dup2", "pause",
        "nanosleep", "getitimer", "alarm", "setitimer", "getpid", "sendfile",
        "socket", "connect", "accept", "sendto", "recvfrom", "sendmsg",
        "recvmsg", "shutdown", "bind", "listen", "getsockname", "getpeername",
        "socketpair", "setsockopt", "getsockopt", "clone", "fork", "vfork",
        "execve", "exit", "wait4", "kill", "uname", "fcntl", "flock",
        "fsync", "fdatasync", "truncate", "ftruncate", "getdents", "getcwd",
        "chdir", "fchdir", "rename", "mkdir", "rmdir", "creat", "link",
        "unlink", "symlink", "readlink", "chmod", "fchmod", "chown", "fchown",
        "lchown", "umask", "gettimeofday", "getrlimit", "getrusage", "sysinfo",
        "times", "ptrace", "getuid", "syslog", "getgid", "setuid", "setgid",
        "geteuid", "getegid", "setpgid", "getppid", "getpgrp", "setsid"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

**AppArmor Profile:**

```
#include <tunables/global>

profile judge-worker flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # Allow read access to /tmp
  /tmp/** rw,
  
  # Deny access to sensitive directories
  deny /etc/** wl,
  deny /proc/** wl,
  deny /sys/** wl,
  deny /boot/** rwklx,
  deny /dev/** wl,
  deny /usr/bin/** wl,
  
  # Allow execution of interpreter
  /usr/bin/python3.9 rix,
  /usr/bin/java rix,
  /usr/bin/gcc rix,
  
  # Deny network access
  deny network,
}
```

**Docker Run Command:**

```
docker run \
  --rm \
  --network none \
  --memory 512m \
  --cpus 1.0 \
  --pids-limit 100 \
  --read-only \
  --tmpfs /tmp:size=100m,noexec,nosuid,nodev \
  --security-opt seccomp=/etc/seccomp/judge.json \
  --security-opt apparmor=judge-worker \
  --security-opt no-new-privileges \
  --cap-drop ALL \
  --user 1000:1000 \
  judge-runtime:python3.9
```

### Trade-offs Accepted

| What We Gain                          | What We Sacrifice                                   |
|---------------------------------------|-----------------------------------------------------|
| ✅ Good security (multi-layer defense) | ❌ Not hardware-level isolation (shared kernel)      |
| ✅ Low overhead (5%)                   | ❌ Complex configuration (seccomp/AppArmor profiles) |
| ✅ Mature technology                   | ❌ Potential kernel vulnerabilities (CVEs)           |
| ✅ Easy to debug                       | ❌ Requires regular profile updates                  |

### When to Reconsider

**Switch to gVisor if:**

- Security incidents occur (container escapes)
- Need user-space kernel (isolation from host kernel)
- Can tolerate 30% performance overhead
- Have expert team to manage gVisor

**Switch to Firecracker if:**

- Running on AWS (native Firecracker support)
- Need hardware-level isolation (regulatory requirements)
- Can build custom orchestration
- Budget allows (higher RAM cost)

### Real-World Examples

- **LeetCode**: Docker + Seccomp + AppArmor
- **HackerRank**: Docker + gVisor (extra security)
- **AWS Lambda**: Firecracker (microVMs)
- **Google Cloud Run**: gVisor

---

## 8. Real-Time Updates: WebSockets vs Server-Sent Events vs Polling

### The Problem

Users want real-time feedback on submission status (Queued → Running → Completed). Execution takes 5 seconds on average.

**Requirements:**

- Low latency updates (< 100ms from verdict to client)
- Support 100k concurrent connections
- Efficient resource usage (minimize server load)

### Options Considered

| Approach                  | Latency     | Server Load                  | Browser Support | Complexity  | Firewall Friendly |
|---------------------------|-------------|------------------------------|-----------------|-------------|-------------------|
| **Polling (GET /status)** | ⚠️ 1s (avg) | ❌ High (constant requests)   | ✅ 100%          | ✅ Simple    | ✅ Yes             |
| **Long Polling**          | ✅ 100ms     | ⚠️ Medium (held connections) | ✅ 100%          | ⚠️ Moderate | ✅ Yes             |
| **WebSockets**            | ✅ < 100ms   | ✅ Low (single connection)    | ✅ 99%           | ⚠️ Moderate | ⚠️ Sometimes      |
| **Server-Sent Events**    | ✅ < 100ms   | ✅ Low (single connection)    | ✅ 99%           | ✅ Simple    | ✅ Yes             |

### Decision Made

**WebSockets for Real-Time + Polling as Fallback**

**Architecture:**

- **Primary**: WebSocket connection to ws://server/submissions/{id}
- **Fallback**: Polling GET /api/submissions/{id} (if WebSocket fails)
- **Notification**: Redis Pub/Sub pushes updates to WebSocket servers

### Rationale

1. **Low Latency**: < 100ms from verdict to client (vs 1s polling)
2. **Low Server Load**: Single connection vs polling every 1s
3. **Bi-Directional**: Can send control commands (cancel submission)
4. **Real-Time Progress**: Can push test progress (Test 5/10)
5. **Efficient**: 100k connections = 100k sockets (manageable)

**Why NOT Polling Alone:**

- High server load (100k clients × 1 req/sec = 100k QPS)
- High latency (avg 500ms delay)
- Wastes bandwidth (most polls return "no change")
- Poor user experience (visible delay)

**Why NOT Server-Sent Events:**

- One-directional only (can't cancel submission)
- Browser limit (6 connections per domain)
- Less mature tooling (vs WebSockets)

### Implementation Details

**WebSocket Server:**

```
const WebSocket = require('ws');
const Redis = require('redis');

const wss = new WebSocket.Server({ port: 8080 });
const redis = Redis.createClient();

wss.on('connection', (ws, req) => {
  const submission_id = req.url.split('/')[2];
  
  // Subscribe to Redis Pub/Sub
  redis.subscribe(`submission:${submission_id}`);
  
  redis.on('message', (channel, message) => {
    // Push update to client
    ws.send(message);
  });
  
  ws.on('close', () => {
    redis.unsubscribe(`submission:${submission_id}`);
  });
});
```

**Worker (Publisher):**

```
// After each update
redis.publish(`submission:${submission_id}`, JSON.stringify({
  status: 'Running',
  test: 5,
  total: 10
}));

// Final verdict
redis.publish(`submission:${submission_id}`, JSON.stringify({
  status: 'Completed',
  verdict: 'Accepted',
  runtime: 234,
  memory: 45
}));
```

**Client (with fallback):**

```
async function subscribeToSubmission(submission_id) {
  try {
    // Try WebSocket
    const ws = new WebSocket(`ws://server/submissions/${submission_id}`);
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      updateUI(data);
      
      if (data.status === 'Completed') {
        ws.close();
      }
    };
    
    ws.onerror = () => {
      // Fallback to polling
      pollSubmission(submission_id);
    };
  } catch (e) {
    // Fallback to polling
    pollSubmission(submission_id);
  }
}

async function pollSubmission(submission_id) {
  while (true) {
    const response = await fetch(`/api/submissions/${submission_id}`);
    const data = await response.json();
    updateUI(data);
    
    if (data.status === 'Completed') {
      break;
    }
    
    await sleep(1000);
  }
}
```

### Trade-offs Accepted

| What We Gain                                  | What We Sacrifice                                    |
|-----------------------------------------------|------------------------------------------------------|
| ✅ Real-time updates (< 100ms)                 | ❌ WebSocket infrastructure (servers, load balancers) |
| ✅ Low server load (1 connection vs 100 polls) | ❌ Firewall issues (some networks block WebSockets)   |
| ✅ Bi-directional (can cancel)                 | ❌ Connection management (heartbeat, reconnect)       |
| ✅ Better UX (instant feedback)                | ❌ More complex client code (WebSocket + fallback)    |

### When to Reconsider

**Switch to Polling if:**

- Low traffic (< 1k concurrent connections)
- Simple requirements (no real-time needed)
- Want maximum compatibility (firewall concerns)

**Switch to Server-Sent Events if:**

- One-directional only (no cancel needed)
- Want simpler protocol (HTTP-based)
- Better browser support (EventSource API)

**Use Long Polling if:**

- Need real-time but WebSocket blocked
- Server can handle held connections
- Want HTTP compatibility

### Real-World Examples

- **LeetCode**: WebSockets with polling fallback
- **HackerRank**: Server-Sent Events (SSE)
- **Codeforces**: Polling (simple, low traffic)
- **TopCoder**: WebSockets (real-time contests)

---

## 9. Verdict Storage: PostgreSQL vs DynamoDB vs Redis

### The Problem

Store submission results:

- Submission metadata (user_id, problem_id, timestamp, language)
- Verdict (Accepted, Wrong Answer, TLE, MLE, RE)
- Runtime and memory usage
- Code (for review)

**Access Patterns:**

- Write: 10k QPS (during contests)
- Read: 50k QPS (leaderboard, user history, problem stats)
- Query: Filter by user, problem, verdict, time range
- Retention: Store forever (historical data)

### Options Considered

| Feature                    | PostgreSQL                     | DynamoDB                         | Redis                             |
|----------------------------|--------------------------------|----------------------------------|-----------------------------------|
| **Write Throughput**       | ⚠️ 10k QPS (with optimization) | ✅ 100k+ WCU                      | ✅ 100k+ QPS                       |
| **Read Throughput**        | ⚠️ 50k QPS (with replicas)     | ✅ 100k+ RCU                      | ✅ 1M+ QPS                         |
| **Query Flexibility**      | ✅ SQL (JOIN, GROUP BY, WHERE)  | ❌ Limited (partition + sort key) | ❌ None (key-value)                |
| **Cost**                   | ⚠️ $0.10/GB + compute          | ⚠️ $0.25/GB + throughput         | ❌ $0.50/GB (in-memory)            |
| **Durability**             | ✅ ACID, replication            | ✅ 11-nines                       | ❌ Volatile (persistence optional) |
| **Operational Complexity** | ⚠️ Moderate (backups, scaling) | ✅ Low (fully managed)            | ⚠️ Moderate (clustering)          |

### Decision Made

**Hybrid: PostgreSQL (Primary) + Redis (Cache) + DynamoDB (Archive)**

**Architecture:**

```
Write Path:
User submits → Worker → PostgreSQL (hot data, 7 days)
              → Redis (cache, 1 hour TTL)
              → DynamoDB (archive, forever)

Read Path:
1. Check Redis (1ms, 80% hit rate)
2. Check PostgreSQL (10ms, 15% hit rate)
3. Check DynamoDB (20ms, 5% hit rate - old submissions)
```

### Rationale

1. **PostgreSQL (Primary)**: Rich querying (leaderboard, user stats, problem difficulty)
2. **Redis (Cache)**: Fast reads (1ms vs 10ms PostgreSQL)
3. **DynamoDB (Archive)**: Infinite retention, low cost ($0.25/GB vs $0.10/GB PostgreSQL)

**Why NOT PostgreSQL Alone:**

- Expensive for infinite retention (TBs of data)
- Read replicas needed for 50k QPS (costly)
- Cache required anyway (reduce load)

**Why NOT DynamoDB Alone:**

- Limited querying (can't do complex JOINs)
- Expensive for frequent queries ($0.25 per 1M reads)
- No aggregations (COUNT, AVG, SUM)

**Why NOT Redis Alone:**

- Volatile (data lost on restart)
- Expensive for large dataset ($0.50/GB vs $0.10/GB PostgreSQL)
- Not suitable for permanent storage

### Implementation Details

**Write Path:**

```
async function storeVerdict(submission) {
  // 1. Write to PostgreSQL (hot data, 7 days retention)
  await postgres.query(`
    INSERT INTO submissions (id, user_id, problem_id, verdict, runtime, memory, code, timestamp)
    VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
  `, [submission.id, submission.user_id, submission.problem_id, submission.verdict, submission.runtime, submission.memory, submission.code]);
  
  // 2. Write to Redis (cache, 1 hour TTL)
  await redis.setex(`submission:${submission.id}`, 3600, JSON.stringify(submission));
  
  // 3. Async write to DynamoDB (archive, forever retention)
  await sqs.send({
    topic: 'archive-submissions',
    message: submission
  });
}
```

**Read Path:**

```
async function getSubmission(submission_id) {
  // 1. Check Redis (80% hit rate, 1ms)
  const cached = await redis.get(`submission:${submission_id}`);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // 2. Check PostgreSQL (15% hit rate, 10ms)
  const result = await postgres.query(`
    SELECT * FROM submissions WHERE id = $1
  `, [submission_id]);
  
  if (result.rows.length > 0) {
    // Cache in Redis
    await redis.setex(`submission:${submission_id}`, 3600, JSON.stringify(result.rows[0]));
    return result.rows[0];
  }
  
  // 3. Check DynamoDB (5% hit rate, 20ms - old submissions)
  const archived = await dynamodb.get({
    TableName: 'submissions-archive',
    Key: { submission_id }
  });
  
  if (archived.Item) {
    // Cache in Redis
    await redis.setex(`submission:${submission_id}`, 3600, JSON.stringify(archived.Item));
    return archived.Item;
  }
  
  throw new Error('Submission not found');
}
```

**Archival Process (Cron):**

```
// Run daily at 2 AM
async function archiveOldSubmissions() {
  // Move submissions older than 7 days to DynamoDB
  const old_submissions = await postgres.query(`
    SELECT * FROM submissions
    WHERE timestamp < NOW() - INTERVAL '7 days'
  `);
  
  for (const submission of old_submissions.rows) {
    await dynamodb.put({
      TableName: 'submissions-archive',
      Item: submission
    });
  }
  
  // Delete from PostgreSQL
  await postgres.query(`
    DELETE FROM submissions
    WHERE timestamp < NOW() - INTERVAL '7 days'
  `);
}
```

### Trade-offs Accepted

| What We Gain                           | What We Sacrifice                    |
|----------------------------------------|--------------------------------------|
| ✅ Fast reads (1ms Redis, 80% hit)      | ❌ Complex architecture (3 databases) |
| ✅ Rich querying (PostgreSQL SQL)       | ❌ Eventual consistency (Redis cache) |
| ✅ Infinite retention (DynamoDB)        | ❌ Must manage archival process       |
| ✅ Cost efficient (hot/cold separation) | ❌ Higher operational complexity      |

### When to Reconsider

**Switch to PostgreSQL Only if:**

- Low traffic (< 10k QPS)
- Small dataset (< 1 TB)
- Budget allows (read replicas for scaling)
- Simple architecture preferred

**Switch to DynamoDB Only if:**

- Simple key-value lookups only
- No complex queries needed
- Willing to pay for throughput
- Want zero operational overhead

**Switch to Redis Only if:**

- Temporary data (submissions can be lost)
- Dataset < 100 GB (affordable in memory)
- No complex queries needed

### Real-World Examples

- **LeetCode**: PostgreSQL + Redis + S3 (code storage)
- **HackerRank**: DynamoDB + ElastiCache (Redis)
- **Codeforces**: PostgreSQL only (lower traffic)
- **TopCoder**: MySQL + Memcached

---

## 10. Contest vs Practice: Separate vs Shared Infrastructure

### The Problem

Contests have different requirements than practice:

- **Contest**: High priority, time-sensitive, SLA < 5s, traffic spike (10k QPS)
- **Practice**: Low priority, best-effort, SLA < 10s, steady traffic (1k QPS)

Should we use separate infrastructure or share resources?

### Options Considered

| Approach                        | Cost                 | Isolation  | Complexity  | Fairness                   | Resource Utilization |
|---------------------------------|----------------------|------------|-------------|----------------------------|----------------------|
| **Shared Infrastructure**       | ✅ Low ($1k/month)    | ❌ None     | ✅ Simple    | ❌ Practice impacts contest | ✅ High (90%)         |
| **Separate Infrastructure**     | ❌ High ($5k/month)   | ✅ Complete | ❌ Complex   | ✅ Perfect                  | ❌ Low (50%)          |
| **Shared with Priority Queues** | ✅ Medium ($2k/month) | ⚠️ Good    | ⚠️ Moderate | ✅ Good                     | ✅ High (85%)         |

### Decision Made

**Shared Infrastructure with Priority Queues and Resource Limits**

**Architecture:**

```
Kafka Topics:
├── submissions.high_priority (contest)
│   ├── Dedicated workers: 50 pods
│   ├── SLA: < 5 seconds
│   └── Priority: 1
└── submissions.normal (practice)
    ├── Shared workers: 10-100 pods (auto-scale)
    ├── SLA: < 10 seconds
    └── Priority: 2

Worker Configuration:
- Always consume high_priority first
- Process normal only if high_priority empty
- Auto-scale normal workers (don't interfere with contest)
```

### Rationale

1. **Cost Efficient**: Shared infrastructure ($2k vs $5k)
2. **Good Isolation**: Priority queue ensures contests not impacted
3. **High Utilization**: Normal workers scale down during contests, up during off-peak
4. **Simple**: Single worker deployment, different topics
5. **Fair**: Contest users get fast service, practice users tolerate slight delay

**Why NOT Separate Infrastructure:**

- Double cost ($5k vs $2k)
- Low utilization (contest infrastructure idle 90% of time)
- Complex deployment (two separate clusters)
- Wasted resources (can't share spare capacity)

**Why NOT Fully Shared:**

- Contest impacted by practice traffic (SLA violations)
- Unfair (practice users starve contest users)
- No QoS (quality of service) guarantees

### Implementation Details

**Worker Consumer:**

```
// Consumer always checks high priority first
const consumer = kafka.consumer({ groupId: 'judge-workers' });

await consumer.subscribe({
  topics: ['submissions.high_priority', 'submissions.normal'],
  fromBeginning: false
});

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const submission = JSON.parse(message.value);
    
    // Log priority
    console.log(`Processing ${topic} submission: ${submission.id}`);
    
    // Process
    await executeSubmission(submission);
    
    // Commit offset
    await consumer.commitOffsets([{
      topic,
      partition,
      offset: (parseInt(message.offset) + 1).toString()
    }]);
  }
});
```

**Auto-Scaling Configuration:**

```
# Contest workers (fixed size during contest)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: judge-worker-contest
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: judge-worker
  minReplicas: 50
  maxReplicas: 500
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
        selector:
          matchLabels:
            topic: submissions.high_priority
      target:
        type: AverageValue
        averageValue: "500"  # 500 messages per worker (lower than normal)

# Normal workers (scale dynamically)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: judge-worker-normal
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: judge-worker
  minReplicas: 10
  maxReplicas: 100
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
        selector:
          matchLabels:
            topic: submissions.normal
      target:
        type: AverageValue
        averageValue: "1000"  # 1000 messages per worker (higher than contest)
```

**Contest Detection:**

```
// Submission Service detects active contest
async function submitCode(submission) {
  const active_contests = await redis.smembers('active_contests');
  
  const is_contest = active_contests.some(contest => 
    contest.problems.includes(submission.problem_id)
  );
  
  const topic = is_contest ? 'submissions.high_priority' : 'submissions.normal';
  
  await kafka.produce({
    topic,
    partition: hash(submission.problem_id) % num_partitions,
    key: submission.id,
    value: JSON.stringify(submission)
  });
}
```

### Trade-offs Accepted

| What We Gain                             | What We Sacrifice                                    |
|------------------------------------------|------------------------------------------------------|
| ✅ Cost efficient (shared infrastructure) | ❌ Potential resource contention (if misconfigured)   |
| ✅ High utilization (85% vs 50%)          | ❌ Must monitor queue depth carefully                 |
| ✅ Simple deployment (single cluster)     | ❌ Practice SLA degraded during contests (acceptable) |
| ✅ Fair (contest priority)                | ❌ More complex worker logic (priority handling)      |

### When to Reconsider

**Switch to Separate Infrastructure if:**

- Contests extremely critical (financial impact)
- SLA violations unacceptable
- Budget allows (double cost)
- Compliance requires strict isolation

**Switch to Fully Shared if:**

- Low contest traffic (< 1k QPS)
- No strict SLA requirements
- Want simplest architecture
- Users tolerate delays

### Real-World Examples

- **LeetCode**: Shared with priority queues (similar to our design)
- **HackerRank**: Separate infrastructure (contests critical)
- **Codeforces**: Fully shared (simple, lower scale)
- **TopCoder**: Separate infrastructure (high-stakes contests)

---

**Summary Table: All Design Decisions**

| Decision           | Chosen Approach                  | Key Rationale                                   | When to Reconsider                           |
|--------------------|----------------------------------|-------------------------------------------------|----------------------------------------------|
| **Isolation**      | Docker + Seccomp/AppArmor        | Fast startup, good security, low cost           | Switch to Firecracker for hardware isolation |
| **Queue**          | Kafka (primary) + SQS (fallback) | High throughput (1M msg/sec), ordering          | Switch to SQS for simplicity at low scale    |
| **Judging**        | Asynchronous + WebSocket updates | High throughput, low latency feedback           | Switch to synchronous if < 500ms execution   |
| **Multi-Language** | Multi-image per language         | Small images, fast builds, flexibility          | Switch to monolithic if < 5 languages        |
| **Test Storage**   | S3 + CloudFront + local cache    | Low latency (5ms), low cost ($0.02/GB)          | Switch to PostgreSQL if < 1 GB dataset       |
| **Auto-Scaling**   | Queue depth (messages/worker)    | Proactive, predictive, accurate                 | Switch to CPU if simple requirements         |
| **Security**       | Seccomp + AppArmor + Cgroups     | Good security, low overhead (5%)                | Switch to gVisor for user-space kernel       |
| **Real-Time**      | WebSockets + polling fallback    | Low latency (< 100ms), bi-directional           | Switch to polling if < 1k connections        |
| **Storage**        | PostgreSQL + Redis + DynamoDB    | Rich querying + fast reads + infinite retention | Switch to PostgreSQL only if < 1 TB          |
| **Contest**        | Shared with priority queues      | Cost efficient, good isolation, fair            | Switch to separate if SLA critical           |
