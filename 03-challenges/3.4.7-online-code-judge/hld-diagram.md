# Online Code Judge - High-Level Design Diagrams

This document contains high-level design diagrams for the Online Code Judge system, including system architecture,
execution pipeline, sandboxing architecture, and scaling strategies.

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Submission Flow Pipeline](#2-submission-flow-pipeline)
3. [Docker Sandboxing Architecture](#3-docker-sandboxing-architecture)
4. [Security Layers (Defense in Depth)](#4-security-layers-defense-in-depth)
5. [Queue-Based Asynchronous Architecture](#5-queue-based-asynchronous-architecture)
6. [Judge Worker Execution Pipeline](#6-judge-worker-execution-pipeline)
7. [Auto-Scaling Strategy](#7-auto-scaling-strategy)
8. [Test Case Distribution](#8-test-case-distribution)
9. [Real-Time Status Updates](#9-real-time-status-updates)
10. [Language Runtime Architecture](#10-language-runtime-architecture)
11. [Failure Recovery Flow](#11-failure-recovery-flow)
12. [Multi-Region Deployment](#12-multi-region-deployment)

---

## 1. System Architecture Overview

**Flow Explanation:**

This diagram shows the complete end-to-end architecture of the online code judge system, from client submission to
result delivery.

**Layers:**

1. **Client Layer**: Web UI, mobile apps, CLI tools for code submission
2. **API Gateway**: Authentication, rate limiting, request validation
3. **Submission Service**: Code validation, metadata storage, queue publishing
4. **Message Queue**: Kafka topics for buffering submissions (decouples submission from execution)
5. **Execution Layer**: Judge Manager and Worker fleet for code execution
6. **Sandboxed Execution**: Docker containers with security layers (seccomp, AppArmor, cgroups)
7. **Persistence**: PostgreSQL (metadata), Redis (cache), S3 (test cases)

**Key Design Decisions:**

- Queue-based for absorbing traffic spikes (10x during contests)
- Docker containers for lightweight sandboxing (< 1 second startup)
- Auto-scaling workers based on queue depth (not CPU)
- Multi-layer security (defense in depth)

**Performance:**

- Submission to queue: < 100ms
- Queue to execution: < 2 seconds (queue wait)
- Execution: 1-10 seconds (depends on code complexity)
- Result delivery: < 200ms (Redis cache)

```mermaid
graph TB
    subgraph ClientLayer["Client Layer"]
        WebUI["Web UI<br/>Browser"]
        MobileApp["Mobile App<br/>iOS/Android"]
        CLI["CLI Tool<br/>VS Code Extension"]
    end

    subgraph APILayer["API Gateway Layer"]
        LB["Load Balancer<br/>ALB/Nginx"]
        APIGateway["API Gateway<br/>Authentication<br/>Rate Limiting<br/>Validation"]
    end

    subgraph SubmissionLayer["Submission Service Layer"]
        SubmissionSvc["Submission Service<br/>- Validate code<br/>- Store metadata<br/>- Publish to queue"]
        ValidationSvc["Validation Service<br/>- Code size check<br/>- Language support<br/>- Syntax validation"]
    end

    subgraph QueueLayer["Message Queue Layer"]
        KafkaCluster["Kafka Cluster<br/>Topics:<br/>- submissions.high_priority<br/>- submissions.normal<br/>Partitioned by problem_id"]
    end

    subgraph ExecutionLayer["Execution Service Layer"]
        JudgeManager["Judge Manager<br/>- Monitor queue depth<br/>- Distribute jobs<br/>- Track worker health<br/>- Trigger auto-scaling"]
        WorkerPool["Worker Pool<br/>100+ servers<br/>500 containers each"]
    end

    subgraph SandboxLayer["Sandboxed Execution"]
        Docker["Docker Container<br/>Security Layers:<br/>- Namespace isolation<br/>- Seccomp profile<br/>- AppArmor policy<br/>- Cgroups limits<br/>- Network isolation"]
    end

    subgraph PersistenceLayer["Persistence Layer"]
        PostgreSQL["PostgreSQL<br/>submissions<br/>verdicts<br/>leaderboard"]
        Redis["Redis Cache<br/>submission status<br/>results<br/>user sessions"]
        S3["S3 Object Store<br/>test cases<br/>code archive"]
    end

    WebUI --> LB
    MobileApp --> LB
    CLI --> LB
    LB --> APIGateway
    APIGateway --> SubmissionSvc
    SubmissionSvc --> ValidationSvc
    ValidationSvc --> KafkaCluster
    KafkaCluster --> JudgeManager
    JudgeManager --> WorkerPool
    WorkerPool --> Docker
    Docker --> PostgreSQL
    Docker --> Redis
    Docker --> S3
    SubmissionSvc --> PostgreSQL
    JudgeManager --> Redis
    style Docker fill: #FF6B6B
    style KafkaCluster fill: #FFD700
    style WorkerPool fill: #90EE90
```

---

## 2. Submission Flow Pipeline

**Flow Explanation:**

This diagram illustrates the complete submission flow from user clicking "Submit" to receiving verdict.

**Steps:**

1. **User Submits**: Code submitted via Web UI (POST /api/submit)
2. **API Gateway**: Validates authentication (JWT), rate limit check (100 submissions/hour)
3. **Submission Service**: Validates code size (< 64 KB), language support, stores in PostgreSQL
4. **Queue Publish**: Publishes to Kafka topic (high_priority or normal)
5. **Queue Wait**: Submission waits in queue (average 2 seconds during normal load)
6. **Worker Pickup**: Judge Manager assigns to available worker
7. **Container Creation**: Worker creates Docker container with security policies (800ms)
8. **Compilation**: Compile code if needed (Java/C++: 500ms, Python: skip)
9. **Execution**: Run against test cases (2-5 seconds)
10. **Verdict**: Compare output, determine AC/WA/TLE/MLE/RE/CE
11. **Result Write**: Update PostgreSQL and Redis (50ms)
12. **Client Notification**: WebSocket push or polling returns result

**Total Latency:**

- Best case (Hello World): 2 seconds
- Average case: 6 seconds
- Worst case (complex code): 15 seconds

```mermaid
graph LR
    subgraph ClientSide["Client Side"]
        User["User<br/>Clicks Submit"]
        UI["Web UI<br/>Code Editor"]
    end

    subgraph Validation["Validation Pipeline"]
        Auth["Authentication<br/>JWT Validation"]
        RateLimit["Rate Limit<br/>100 submissions/hour"]
        CodeValidation["Code Validation<br/>Size: < 64 KB<br/>Language: Whitelist<br/>Syntax: Basic check"]
    end

    subgraph Storage["Metadata Storage"]
        CreateRecord["Create Submission<br/>status: Queued<br/>PostgreSQL INSERT"]
    end

    subgraph Queue["Queue Layer"]
        QueuePublish["Publish to Kafka<br/>topic: submissions.normal<br/>partition: hash problem_id"]
        QueueWait["Queue Wait<br/>avg: 2 seconds<br/>peak: 10 seconds"]
    end

    subgraph Execution["Execution Pipeline"]
        ManagerAssign["Judge Manager<br/>Assign to Worker"]
        WorkerPickup["Worker Pickup<br/>Kafka Consumer"]
        ContainerCreate["Create Container<br/>800ms startup"]
        Compile["Compile Code<br/>if needed"]
        Execute["Execute Tests<br/>2-5 seconds"]
        Judge["Compare Output<br/>Determine Verdict"]
    end

    subgraph Results["Result Delivery"]
        UpdateDB["Update DB<br/>verdict, runtime<br/>memory usage"]
        UpdateCache["Update Redis<br/>submission:id"]
        Notify["Notify Client<br/>WebSocket or Polling"]
    end

    User --> UI
    UI --> Auth
    Auth --> RateLimit
    RateLimit --> CodeValidation
    CodeValidation --> CreateRecord
    CreateRecord --> QueuePublish
    QueuePublish --> QueueWait
    QueueWait --> ManagerAssign
    ManagerAssign --> WorkerPickup
    WorkerPickup --> ContainerCreate
    ContainerCreate --> Compile
    Compile --> Execute
    Execute --> Judge
    Judge --> UpdateDB
    UpdateDB --> UpdateCache
    UpdateCache --> Notify
    Notify --> UI
    style ContainerCreate fill: #FFD700
    style Execute fill: #FF6B6B
    style Judge fill: #90EE90
```

---

## 3. Docker Sandboxing Architecture

**Flow Explanation:**

This diagram shows the Docker container architecture with all security layers for safe code execution.

**Container Configuration:**

- **Base Image**: Alpine Linux (minimal, 5 MB)
- **Runtime Layers**: Python 3.9, Java 11, GCC 10, Node.js 16
- **Security Layers**: Seccomp, AppArmor, cgroups, network isolation
- **Filesystem**: Read-only root, writable /tmp (100 MB limit)
- **Resources**: 1 CPU, 512 MB RAM, 100 processes max

**Security Guarantees:**

- Cannot access host filesystem (except /tmp)
- Cannot access network (no internet)
- Cannot fork bomb (pid limit)
- Cannot memory bomb (memory limit)
- Cannot escape container (seccomp blocks dangerous syscalls)

**Performance:**

- Container creation: 800ms (image cached)
- Container destruction: 100ms
- Total overhead: 900ms per submission

```mermaid
graph TB
    subgraph HostOS["Host Operating System - Ubuntu 22.04"]
        Docker["Docker Engine<br/>Version: 20.10+"]
        Kernel["Linux Kernel<br/>cgroups, namespaces, seccomp"]
    end

    subgraph Container["Docker Container - Isolated Sandbox"]
        BaseImage["Base Image<br/>Alpine Linux 3.16<br/>Size: 5 MB"]

        subgraph Runtimes["Language Runtimes"]
            Python["Python 3.9<br/>interpreter"]
            Java["Java 11 JDK<br/>compiler + JVM"]
            GCC["GCC 10<br/>C/C++ compiler"]
            Node["Node.js 16<br/>JavaScript runtime"]
        end

        subgraph SecurityLayers["Security Layers"]
            Seccomp["Seccomp Profile<br/>Whitelist syscalls:<br/>read, write, exit, brk<br/>Block: execve, fork, socket"]
            AppArmor["AppArmor Policy<br/>Allow: /usr read<br/>Allow: /tmp rw<br/>Deny: everything else"]
            Cgroups["Cgroups Limits<br/>CPU: 1 core<br/>Memory: 512 MB<br/>PIDs: 100<br/>Disk I/O: 10 MB/s"]
            Network["Network Isolation<br/>Mode: none<br/>No internet access"]
        end

        subgraph Filesystem["Filesystem"]
            RootFS["Root FS<br/>Read-only<br/>Cannot modify system"]
            TmpFS["Tmpfs /tmp<br/>Read-write<br/>100 MB limit<br/>Cleared after execution"]
        end
    end

    subgraph UserCode["User Submitted Code"]
        SourceCode["solution.py<br/>User's code"]
        Input["input.txt<br/>Test case input"]
        Output["output.txt<br/>Execution output"]
    end

    Docker --> Container
    Kernel --> Docker
    Container --> BaseImage
    BaseImage --> Runtimes
    Container --> SecurityLayers
    Container --> Filesystem
    UserCode --> TmpFS
    Python --> SourceCode
    SourceCode --> Output
    style Container fill: #FFD700
    style SecurityLayers fill: #FF6B6B
    style UserCode fill: #90EE90
```

---

## 4. Security Layers (Defense in Depth)

**Flow Explanation:**

This diagram illustrates the multiple layers of security protecting the host system from malicious code.

**Layer 1: Docker Namespace Isolation**

- Separate PID namespace (cannot see host processes)
- Separate network namespace (no host network access)
- Separate mount namespace (isolated filesystem)

**Layer 2: Seccomp (System Call Filtering)**

- Whitelist only safe syscalls (read, write, brk, mmap, exit)
- Block dangerous syscalls (execve, fork, clone, socket, connect, bind)
- Prevents privilege escalation attacks

**Layer 3: AppArmor (Mandatory Access Control)**

- Define accessible paths (/usr for libraries, /tmp for temporary files)
- Block access to sensitive files (/etc/passwd, /proc, /sys)
- Prevents filesystem-based attacks

**Layer 4: Cgroups (Resource Limits)**

- CPU limit: 1 core (prevent CPU hogging)
- Memory limit: 512 MB (prevent memory bomb)
- PIDs limit: 100 (prevent fork bomb)
- Disk I/O: 10 MB/sec (prevent disk thrashing)

**Layer 5: Network Isolation**

- Container on isolated network (--network none)
- Cannot access internet or other containers
- Prevents DDoS attacks and data exfiltration

**Layer 6: Read-Only Filesystem**

- Root filesystem mounted read-only
- Only /tmp writable (with size limit)
- Prevents code from modifying system files

**Attack Mitigation:**

- Fork bomb: Blocked by PID limit (100)
- Memory bomb: Blocked by memory limit (512 MB)
- Infinite loop: Blocked by timeout (5-10 seconds)
- Network attack: Blocked by network isolation
- File access: Blocked by AppArmor + read-only filesystem
- Syscall exploit: Blocked by seccomp whitelist

```mermaid
graph TB
    subgraph MaliciousCode["Malicious Code Attempts"]
        ForkBomb["Fork Bomb<br/>while True: os.fork"]
        MemBomb["Memory Bomb<br/>x = [0] * 10**10"]
        NetAttack["Network Attack<br/>socket.connect external"]
        FileAccess["File Access<br/>open /etc/passwd"]
        InfiniteLoop["Infinite Loop<br/>while True: pass"]
    end

    subgraph Layer1["Layer 1: Docker Namespaces"]
        PIDNamespace["PID Namespace<br/>Isolated process tree"]
        NetNamespace["Network Namespace<br/>No host network"]
        MountNamespace["Mount Namespace<br/>Isolated filesystem"]
    end

    subgraph Layer2["Layer 2: Seccomp"]
        SyscallFilter["Syscall Whitelist<br/>Allow: read, write, exit<br/>Block: execve, fork, socket"]
    end

    subgraph Layer3["Layer 3: AppArmor"]
        PathControl["Path-Based Access Control<br/>Allow: /usr read, /tmp rw<br/>Deny: /etc, /proc, /sys"]
    end

    subgraph Layer4["Layer 4: Cgroups"]
        CPULimit["CPU Limit<br/>1 core max"]
        MemLimit["Memory Limit<br/>512 MB max"]
        PIDLimit["PID Limit<br/>100 processes max"]
        IOLimit["Disk I/O Limit<br/>10 MB/sec max"]
    end

    subgraph Layer5["Layer 5: Network Isolation"]
        NoInternet["Network Mode: none<br/>No internet access<br/>No container communication"]
    end

    subgraph Layer6["Layer 6: Read-Only FS"]
        ReadOnlyRoot["Root FS Read-Only<br/>Cannot modify system files"]
        WritableTmp["Tmpfs /tmp<br/>100 MB limit<br/>Cleared after execution"]
    end

    subgraph Result["Security Outcome"]
        Blocked["All Attacks Blocked<br/>Code executes safely<br/>Host system protected"]
    end

    ForkBomb --> PIDLimit
    MemBomb --> MemLimit
    NetAttack --> NoInternet
    FileAccess --> PathControl
    InfiniteLoop --> CPULimit
    PIDLimit --> Blocked
    MemLimit --> Blocked
    NoInternet --> Blocked
    PathControl --> Blocked
    CPULimit --> Blocked
    style MaliciousCode fill: #FF6B6B
    style Layer4 fill: #FFD700
    style Blocked fill: #90EE90
```

---

## 5. Queue-Based Asynchronous Architecture

**Flow Explanation:**

This diagram shows why queue-based architecture is essential for handling traffic spikes and ensuring no submission is
lost.

**Why Queue?**

- Execution is slow (1-10 seconds per submission)
- Traffic is bursty (10x spike during contests)
- Workers can fail (need retry mechanism)
- Need fair scheduling (FIFO or priority)

**Kafka Configuration:**

- **Topics**: high_priority (contests), normal (practice)
- **Partitions**: 10 partitions per topic (load balancing)
- **Replication Factor**: 3 (survive 2 broker failures)
- **Retention**: 7 days (debug recent submissions)

**Consumer Groups:**

- Workers join "judge-workers" consumer group
- Kafka automatically distributes partitions across workers
- If worker dies, Kafka rebalances partitions to remaining workers

**Retry Mechanism:**

- Worker crashes mid-execution → Message not acknowledged
- Kafka redelivers message to another worker
- Idempotent execution (check if submission already judged)

**Throughput:**

- Normal load: 100 submissions/sec → 100 messages/sec
- Peak load: 10,000 submissions/sec → 10,000 messages/sec
- Queue absorbs spike, workers process at their capacity

```mermaid
graph TB
    subgraph Submission["Submission Sources"]
        Contest["Contest Submissions<br/>10k/sec peak<br/>High priority"]
        Practice["Practice Submissions<br/>100/sec normal<br/>Normal priority"]
    end

    subgraph Queue["Kafka Queue Architecture"]
        HighPriority["Topic: submissions.high_priority<br/>Partitions: 10<br/>Replication: 3x<br/>Retention: 7 days"]
        Normal["Topic: submissions.normal<br/>Partitions: 10<br/>Replication: 3x<br/>Retention: 7 days"]
    end

    subgraph ConsumerGroup["Consumer Group: judge-workers"]
        Worker1["Worker 1<br/>Consuming:<br/>Partition 0, 1"]
        Worker2["Worker 2<br/>Consuming:<br/>Partition 2, 3"]
        Worker3["Worker 3<br/>Consuming:<br/>Partition 4, 5"]
        WorkerN["Worker N<br/>Consuming:<br/>Partition 8, 9"]
    end

    subgraph Execution["Execution"]
        Processing["Processing<br/>1-10 seconds per submission"]
    end

    subgraph Benefits["Benefits"]
        Decoupling["Decoupling<br/>Submission service doesn't wait<br/>for execution"]
        BufferSpikes["Buffer Spikes<br/>Queue absorbs 10x traffic<br/>Workers process at capacity"]
        Persistence["Persistence<br/>Submissions not lost<br/>if workers crash"]
        FairScheduling["Fair Scheduling<br/>FIFO within priority<br/>High priority processed first"]
    end

    Contest --> HighPriority
    Practice --> Normal
    HighPriority --> Worker1
    HighPriority --> Worker2
    Normal --> Worker3
    Normal --> WorkerN
    Worker1 --> Processing
    Worker2 --> Processing
    Worker3 --> Processing
    WorkerN --> Processing
    Processing --> Decoupling
    Processing --> BufferSpikes
    Processing --> Persistence
    Processing --> FairScheduling
    style HighPriority fill: #FF6B6B
    style Queue fill: #FFD700
    style Benefits fill: #90EE90
```

---

## 6. Judge Worker Execution Pipeline

**Flow Explanation:**

This diagram shows the detailed execution pipeline within a judge worker from picking up a submission to reporting
verdict.

**Steps:**

1. **Consume Message**: Worker pulls submission from Kafka (100ms)
2. **Create Container**: Docker create with security policies (800ms)
3. **Copy Code**: Write user code to /tmp/solution.py (10ms)
4. **Copy Test Cases**: Fetch from cache or S3 (50ms cached, 200ms S3)
5. **Compile** (if needed): javac Solution.java (500ms for Java, skip for Python)
6. **Execute Test Cases** (loop):
    - Read input: cat test_input.txt (1ms)
    - Run code: timeout 5s python3 solution.py < input (2-5s)
    - Capture output: stdout > output.txt (1ms)
    - Compare: diff output.txt expected.txt (5ms)
    - Check time: Must complete within limit
    - Check memory: Must not exceed limit
7. **Determine Verdict**:
    - All passed → AC (Accepted)
    - Any wrong → WA (Wrong Answer)
    - Timeout → TLE (Time Limit Exceeded)
    - Segfault → RE (Runtime Error)
    - OOM → MLE (Memory Limit Exceeded)
    - Compilation failed → CE (Compilation Error)
8. **Cleanup**: Remove container (100ms)
9. **Report Result**: Update PostgreSQL + Redis (50ms)
10. **Acknowledge**: Kafka commit offset (10ms)

**Total Time:**

- Setup: 1 second (container + test cases)
- Execution: 2-5 seconds (depends on code)
- Cleanup: 0.2 seconds
- **Total: 3-6 seconds**

```mermaid
graph TB
    subgraph KafkaQueue["Kafka Queue"]
        Message["Submission Message<br/>sub_id: 123<br/>problem: two_sum<br/>language: python3<br/>code: base64_encoded"]
    end

    subgraph WorkerSetup["Worker Setup Phase - 1 second"]
        Consume["Consume Message<br/>Kafka poll<br/>100ms"]
        CreateContainer["Create Docker Container<br/>Security policies applied<br/>800ms"]
        CopyCode["Write Code to /tmp<br/>echo code > solution.py<br/>10ms"]
        FetchTests["Fetch Test Cases<br/>Cache hit: 50ms<br/>S3 miss: 200ms"]
    end

    subgraph Compilation["Compilation Phase - 0-500ms"]
        CheckLang["Check Language"]
        CompileJava["Compile Java<br/>javac Solution.java<br/>500ms"]
        CompileCpp["Compile C++<br/>g++ solution.cpp<br/>400ms"]
        SkipPython["Skip Python<br/>Interpreted<br/>0ms"]
    end

    subgraph Execution["Execution Phase - 2-5 seconds"]
        TestLoop["For each test case:<br/>10 test cases"]
        ReadInput["Read input.txt<br/>1ms"]
        RunCode["Run code<br/>timeout 5s python3 solution.py<br/>stdin from input.txt<br/>2-5 seconds"]
        CaptureOutput["Capture stdout<br/>output.txt<br/>1ms"]
        Compare["Compare output<br/>diff output.txt expected.txt<br/>5ms"]
        CheckLimits["Check Limits<br/>Time: < 5s<br/>Memory: < 512MB"]
    end

    subgraph Verdict["Verdict Determination"]
        AllPassed["All Passed<br/>AC - Accepted"]
        WrongOutput["Wrong Output<br/>WA - Wrong Answer"]
        Timeout["Timeout<br/>TLE - Time Limit Exceeded"]
        MemoryExceed["Memory Exceeded<br/>MLE - Memory Limit Exceeded"]
        RuntimeErr["Segfault/Exception<br/>RE - Runtime Error"]
        CompileErr["Compilation Failed<br/>CE - Compilation Error"]
    end

    subgraph Cleanup["Cleanup Phase - 200ms"]
        DestroyContainer["Destroy Container<br/>docker rm<br/>100ms"]
        UpdateDB["Update Database<br/>PostgreSQL + Redis<br/>50ms"]
        CommitOffset["Commit Kafka Offset<br/>Acknowledge message<br/>10ms"]
    end

    Message --> Consume
    Consume --> CreateContainer
    CreateContainer --> CopyCode
    CopyCode --> FetchTests
    FetchTests --> CheckLang
    CheckLang --> CompileJava
    CheckLang --> CompileCpp
    CheckLang --> SkipPython
    CompileJava --> TestLoop
    CompileCpp --> TestLoop
    SkipPython --> TestLoop
    TestLoop --> ReadInput
    ReadInput --> RunCode
    RunCode --> CaptureOutput
    CaptureOutput --> Compare
    Compare --> CheckLimits
    CheckLimits --> TestLoop
    TestLoop --> AllPassed
    TestLoop --> WrongOutput
    TestLoop --> Timeout
    TestLoop --> MemoryExceed
    TestLoop --> RuntimeErr
    CheckLang --> CompileErr
    AllPassed --> DestroyContainer
    WrongOutput --> DestroyContainer
    Timeout --> DestroyContainer
    MemoryExceed --> DestroyContainer
    RuntimeErr --> DestroyContainer
    CompileErr --> DestroyContainer
    DestroyContainer --> UpdateDB
    UpdateDB --> CommitOffset
    style Execution fill: #FFD700
    style Verdict fill: #90EE90
    style Cleanup fill: #87CEEB
```

---

## 7. Auto-Scaling Strategy

**Flow Explanation:**

This diagram shows how the system auto-scales workers based on queue depth (not CPU).

**Why Queue Depth (Not CPU)?**

- Workers may be idle but queue backing up (slow executions)
- CPU low but queue depth high → Need more workers
- Queue depth directly indicates backlog

**Scaling Policy:**

```
Target: < 1000 messages per worker
Scale Up: Queue depth > 1000 × worker_count
Scale Down: Queue depth < 500 × worker_count
Cooldown: 5 minutes (avoid thrashing)
```

**Example:**

- 10 workers, queue depth 15,000 → 15,000 / 10 = 1,500/worker → Scale up to 15 workers
- 20 workers, queue depth 8,000 → 8,000 / 20 = 400/worker → Scale down to 16 workers

**Response Time:**

- Kubernetes HPA detects high queue: 30 seconds
- EC2 instance launch: 90 seconds
- Container pull + startup: 60 seconds
- **Total: 3 minutes** to add new workers

**Pre-Warming for Contests:**

- Contests scheduled in advance (known start time)
- Pre-scale workers 30 minutes before contest
- Gradually scale down 30 minutes after (avoid sudden drop)

```mermaid
graph TB
    subgraph Monitoring["Queue Monitoring"]
        Prometheus["Prometheus<br/>Scrapes Kafka metrics<br/>Every 15 seconds"]
        QueueDepth["Queue Depth Metric<br/>kafka_consumer_lag<br/>Current: 15,000 messages"]
        WorkerCount["Worker Count<br/>Current: 10 workers"]
        Ratio["Messages per Worker<br/>15,000 / 10 = 1,500"]
    end

    subgraph Decision["Scaling Decision"]
        CheckThreshold["Check Threshold<br/>Target: < 1,000/worker<br/>Current: 1,500/worker<br/>Status: EXCEED"]
        ScaleUp["Decision: Scale Up<br/>Add 5 workers<br/>New total: 15"]
    end

    subgraph Execution["Auto-Scaling Execution"]
        K8sHPA["Kubernetes HPA<br/>Horizontal Pod Autoscaler<br/>Triggers scale event"]
        LaunchPods["Launch New Pods<br/>5 new worker pods<br/>EC2 instances"]
        PodStartup["Pod Startup<br/>Pull Docker image<br/>Join consumer group"]
        Rebalance["Kafka Rebalance<br/>Redistribute partitions<br/>across 15 workers"]
    end

    subgraph Result["Scaling Result"]
        NewRatio["New Ratio<br/>15,000 / 15 = 1,000/worker<br/>✅ Within target"]
        ReducedWait["Reduced Wait Time<br/>Queue wait: 10s → 5s<br/>Better user experience"]
    end

    subgraph PreWarming["Pre-Warming Strategy"]
        ScheduleDetect["Contest Scheduled<br/>Start time: 14:00<br/>Current: 13:30"]
        PreScale["Pre-Scale Workers<br/>30 min before<br/>10 → 50 workers"]
        ContestStart["Contest Starts<br/>Handle 10k/sec spike<br/>No delay"]
        GradualDown["Gradual Scale Down<br/>30 min after<br/>50 → 10 workers"]
    end

    Prometheus --> QueueDepth
    QueueDepth --> WorkerCount
    WorkerCount --> Ratio
    Ratio --> CheckThreshold
    CheckThreshold --> ScaleUp
    ScaleUp --> K8sHPA
    K8sHPA --> LaunchPods
    LaunchPods --> PodStartup
    PodStartup --> Rebalance
    Rebalance --> NewRatio
    NewRatio --> ReducedWait
    ScheduleDetect --> PreScale
    PreScale --> ContestStart
    ContestStart --> GradualDown
    style CheckThreshold fill: #FF6B6B
    style ScaleUp fill: #FFD700
    style NewRatio fill: #90EE90
```

---

## 8. Test Case Distribution

**Flow Explanation:**

This diagram shows the three-tier test case distribution strategy for fast access.

**Tier 1: Worker Local Cache (Hottest)**

- Each worker caches test cases on local SSD
- Cache size: 50 GB (covers 5,000 problems × 10 MB avg)
- Access time: < 10ms (local disk read)
- Hit rate: 95% (hot problems cached)

**Tier 2: S3 with CloudFront (Warm)**

- All test cases stored in S3 (source of truth)
- CloudFront CDN caches at edge locations
- Access time: 50-100ms (CloudFront edge)
- Hit rate: 99% (CloudFront cache)

**Tier 3: S3 Origin (Cold)**

- Original S3 bucket (us-east-1)
- Access time: 200-500ms (S3 direct)
- Hit rate: 1% (CloudFront miss)

**Cache Invalidation:**

- Problem updated → Publish event to Redis Pub/Sub: `channel:problem:{id}:updated`
- All workers subscribed to channel
- On event: Delete local cached test cases for that problem
- Next execution fetches fresh from S3

**Storage:**

- Total test cases: 10,000 problems
- Average size: 1 MB per problem
- Total: 10 GB in S3
- S3 cost: $0.23/month (dirt cheap)

```mermaid
graph TB
    subgraph Request["Test Case Request"]
        Worker["Judge Worker<br/>Needs test cases for<br/>problem: two_sum"]
    end

    subgraph Tier1["Tier 1: Worker Local Cache"]
        LocalCache["Local SSD Cache<br/>50 GB capacity<br/>5,000 problems cached<br/>Access: < 10ms"]
        CacheHit["Cache Hit?<br/>95% hit rate"]
    end

    subgraph Tier2["Tier 2: CloudFront CDN"]
        CloudFront["CloudFront Edge<br/>Global CDN<br/>Caches at nearest location<br/>Access: 50-100ms"]
        EdgeHit["Edge Cache Hit?<br/>99% hit rate"]
    end

    subgraph Tier3["Tier 3: S3 Origin"]
        S3["S3 Bucket<br/>us-east-1<br/>10 GB total<br/>Access: 200-500ms"]
    end

    subgraph Invalidation["Cache Invalidation"]
        ProblemUpdate["Problem Updated<br/>Test cases changed"]
        RedisPubSub["Redis Pub/Sub<br/>Publish event:<br/>problem:two_sum:updated"]
        WorkerSub["All Workers<br/>Subscribed to channel<br/>Receive event"]
        DeleteCache["Delete Local Cache<br/>Remove cached test cases<br/>for two_sum"]
    end

    subgraph Performance["Performance Metrics"]
        AvgLatency["Average Latency<br/>95% × 10ms + 4% × 75ms + 1% × 350ms<br/>= 13ms average"]
        Throughput["Throughput<br/>50k concurrent executions<br/>13ms avg → negligible overhead"]
    end

    Worker --> CacheHit
    CacheHit -->|Yes 95%| LocalCache
    CacheHit -->|No 5%| EdgeHit
    EdgeHit -->|Yes 4%| CloudFront
    EdgeHit -->|No 1%| S3
    LocalCache --> Worker
    CloudFront --> LocalCache
    S3 --> CloudFront
    ProblemUpdate --> RedisPubSub
    RedisPubSub --> WorkerSub
    WorkerSub --> DeleteCache
    LocalCache --> AvgLatency
    CloudFront --> AvgLatency
    S3 --> AvgLatency
    AvgLatency --> Throughput
    style LocalCache fill: #90EE90
    style CloudFront fill: #FFD700
    style S3 fill: #87CEEB
```

---

## 9. Real-Time Status Updates

**Flow Explanation:**

This diagram shows how clients receive real-time status updates using WebSocket and Redis Pub/Sub.

**Flow:**

1. **User Submits**: Code submitted, submission ID returned (sub_123)
2. **Client Subscribes**: WebSocket connection to `ws://server/submissions/sub_123`
3. **Worker Updates**: Worker publishes status updates to Redis Pub/Sub channel
4. **WebSocket Server**: Subscribes to Redis channel, receives updates
5. **Push to Client**: WebSocket server pushes updates to client in real-time

**Status Updates:**

- Queued: Submission accepted, waiting in queue
- Running: Worker picked up, executing test case 5/10
- Completed: All test cases done, verdict determined
- Accepted/WA/TLE/MLE/RE/CE: Final verdict

**Alternative: Polling (Fallback)**

- Client polls `GET /api/submissions/sub_123/status` every 1 second
- Server returns cached status from Redis
- Works when WebSocket not supported (old browsers, firewalls)

**Performance:**

- WebSocket: Real-time (< 50ms update delay)
- Polling: 1-second delay (acceptable fallback)
- Redis Pub/Sub latency: < 10ms

```mermaid
graph TB
    subgraph Client["Client Side"]
        User["User Submits Code<br/>POST /api/submit"]
        SubmitResponse["Response:<br/>submission_id: sub_123<br/>status: Queued"]
        WebSocketConnect["WebSocket Connect<br/>ws://server/submissions/sub_123"]
        DisplayStatus["Display Status<br/>Queued → Running → Accepted"]
    end

    subgraph WebSocketServer["WebSocket Server"]
        WSConnection["WebSocket Connection<br/>Client connected<br/>Tracking: sub_123"]
        RedisSubscribe["Subscribe to Redis<br/>channel: submission:sub_123"]
        ReceiveUpdate["Receive Update from Redis<br/>{status: Running, test: 5/10}"]
        PushToClient["Push to Client<br/>WebSocket send"]
    end

    subgraph Worker["Judge Worker"]
        Executing["Executing Code<br/>Test case 5/10"]
        PublishUpdate["Publish Status<br/>Redis Pub/Sub<br/>channel: submission:sub_123<br/>message: {status: Running, test: 5}"]
    end

    subgraph RedisPubSub["Redis Pub/Sub"]
        Channel["Channel: submission:sub_123<br/>Subscribers: 1 WebSocket server"]
    end

    subgraph Polling["Polling Alternative Fallback"]
        PollRequest["GET /api/submissions/sub_123/status<br/>Every 1 second"]
        RedisCache["Redis Cache<br/>GET submission:sub_123<br/>Return: {status: Running}"]
        PollResponse["Response:<br/>{status: Running, test: 5/10}"]
    end

    User --> SubmitResponse
    SubmitResponse --> WebSocketConnect
    WebSocketConnect --> WSConnection
    WSConnection --> RedisSubscribe
    Executing --> PublishUpdate
    PublishUpdate --> Channel
    Channel --> ReceiveUpdate
    ReceiveUpdate --> PushToClient
    PushToClient --> DisplayStatus
    PollRequest --> RedisCache
    RedisCache --> PollResponse
    PollResponse --> DisplayStatus
    style WebSocketConnect fill: #90EE90
    style RedisPubSub fill: #FFD700
    style Polling fill: #87CEEB
```

---

## 10. Language Runtime Architecture

**Flow Explanation:**

This diagram shows how multiple language runtimes are organized in Docker images.

**Multi-Stage Docker Image:**

- **Base Layer**: Alpine Linux (5 MB)
- **System Layer**: gcc, make, binutils (50 MB)
- **Language Layers**:
    - Python 3.9 (150 MB)
    - Java 11 JDK (300 MB)
    - GCC 10 C/C++ (200 MB)
    - Node.js 16 (100 MB)
    - Go 1.19 (500 MB)
    - Rust 1.65 (1 GB)
- **Total**: 2.3 GB per image

**Language Selection:**

- Worker checks submission language: `language: python3`
- Runs appropriate command: `python3 solution.py`
- No image pulling needed (all languages pre-baked)

**Image Management:**

- All workers pre-pull image (cached locally)
- No network overhead on execution
- Updated weekly with security patches
- Versioned images: `judge-runtime:v2.3`

**Compilation Commands:**

```
Python: No compilation (interpreted)
Java: javac Solution.java
C++: g++ -std=c++17 -O2 solution.cpp -o solution
JavaScript: node solution.js (no compilation)
Go: go build solution.go
Rust: rustc solution.rs
```

```mermaid
graph TB
    subgraph DockerImage["Docker Image: judge-runtime:v2.3"]
        BaseLayer["Base Layer<br/>Alpine Linux 3.16<br/>5 MB"]

        subgraph SystemLayer["System Layer"]
            BuildTools["Build Tools<br/>gcc, make, binutils<br/>50 MB"]
        end

        subgraph LanguageRuntimes["Language Runtime Layers"]
            Python["Python 3.9<br/>Interpreter<br/>150 MB"]
            Java["Java 11 JDK<br/>Compiler + JVM<br/>300 MB"]
            Cpp["GCC 10<br/>C/C++ Compiler<br/>200 MB"]
            Node["Node.js 16<br/>JavaScript Runtime<br/>100 MB"]
            Go["Go 1.19<br/>Compiler<br/>500 MB"]
            Rust["Rust 1.65<br/>Compiler<br/>1 GB"]
        end

        subgraph TotalSize["Total Image Size"]
            Size["2.3 GB<br/>Pre-cached on all workers<br/>No pull overhead"]
        end
    end

    subgraph Execution["Execution Commands by Language"]
        PythonCmd["Python:<br/>python3 solution.py<br/>No compilation"]
        JavaCmd["Java:<br/>javac Solution.java<br/>java Solution"]
        CppCmd["C++:<br/>g++ -std=c++17 -O2 solution.cpp<br/>./solution"]
        NodeCmd["JavaScript:<br/>node solution.js<br/>No compilation"]
        GoCmd["Go:<br/>go build solution.go<br/>./solution"]
        RustCmd["Rust:<br/>rustc solution.rs<br/>./solution"]
    end

    subgraph Management["Image Management"]
        PreCache["Pre-Cached on Workers<br/>All workers have image<br/>No network overhead"]
        WeeklyUpdate["Weekly Updates<br/>Security patches<br/>Version: v2.3 → v2.4"]
        Versioning["Versioned Tags<br/>Rollback if issues<br/>Blue-green deployment"]
    end

    BaseLayer --> BuildTools
    BuildTools --> Python
    BuildTools --> Java
    BuildTools --> Cpp
    BuildTools --> Node
    BuildTools --> Go
    BuildTools --> Rust
    Python --> PythonCmd
    Java --> JavaCmd
    Cpp --> CppCmd
    Node --> NodeCmd
    Go --> GoCmd
    Rust --> RustCmd
    Size --> PreCache
    PreCache --> WeeklyUpdate
    WeeklyUpdate --> Versioning
    style LanguageRuntimes fill: #90EE90
    style Execution fill: #FFD700
    style Management fill: #87CEEB
```

---

## 11. Failure Recovery Flow

**Flow Explanation:**

This diagram shows how the system handles worker failures without losing submissions.

**Failure Scenario: Worker Crashes Mid-Execution**

**Detection:**

- Worker sends heartbeat every 10 seconds to Judge Manager
- No heartbeat for 30 seconds → Worker marked dead

**Recovery:**

- Worker was processing submission sub_123
- Message not acknowledged to Kafka
- Kafka redelivers message to another worker after 60 seconds
- New worker picks up, executes, reports result

**Client Experience:**

- Submission status: Running
- Worker crash: Status stays Running (client doesn't know)
- New worker picks up: Status continues Running
- Execution complete: Status changes to Accepted
- **Total delay: 60-90 seconds** (acceptable for retry)

**Idempotency:**

- New worker checks: Has sub_123 been judged already?
- If yes: Skip execution, return cached result
- If no: Execute normally
- Prevents duplicate execution

**Graceful Shutdown:**

- Worker receives SIGTERM (Kubernetes pod termination)
- Finish current executions (max 30 seconds)
- Stop consuming new messages from Kafka
- Acknowledge all completed messages
- Exit cleanly (no lost submissions)

```mermaid
graph TB
    subgraph NormalFlow["Normal Flow"]
        Submit["Submission sub_123<br/>User submits code"]
        Queue["Kafka Queue<br/>Message: sub_123"]
        Worker1["Worker 1<br/>Consuming message<br/>Executing code"]
        Heartbeat["Heartbeat<br/>Every 10 seconds<br/>Status: Alive"]
    end

    subgraph Failure["Failure Scenario"]
        Crash["Worker 1 Crashes<br/>Process killed<br/>Heartbeat stops"]
        Detect["Judge Manager<br/>No heartbeat for 30s<br/>Mark worker dead"]
        Unack["Message Unacknowledged<br/>Kafka detects no commit<br/>Message still in queue"]
    end

    subgraph Recovery["Recovery Flow"]
        Rebalance["Kafka Rebalance<br/>Redistribute partitions<br/>to remaining workers"]
        Redeliver["Redeliver Message<br/>sub_123 back in queue<br/>after 60 seconds"]
        Worker2["Worker 2<br/>Picks up message<br/>Check if already judged"]
        IdempotencyCheck["Idempotency Check<br/>Query: Has sub_123 been judged?"]
        Execute["Not Judged<br/>Execute normally<br/>Report result"]
    end

    subgraph ClientExperience["Client Experience"]
        ClientView["Client sees:<br/>Status: Running<br/>No interruption visible<br/>Just longer wait time"]
        Result["Result Delivered<br/>Status: Accepted<br/>Total delay: +60-90s"]
    end

    subgraph GracefulShutdown["Graceful Shutdown Alternative"]
        SIGTERM["Receive SIGTERM<br/>Kubernetes pod termination"]
        FinishCurrent["Finish Current<br/>Complete executions<br/>Max 30 seconds"]
        StopConsuming["Stop Consuming<br/>No new messages<br/>from Kafka"]
        AckComplete["Acknowledge Completed<br/>Commit offsets<br/>to Kafka"]
        CleanExit["Clean Exit<br/>No lost submissions<br/>✅ All messages processed"]
    end

    Submit --> Queue
    Queue --> Worker1
    Worker1 --> Heartbeat
    Worker1 --> Crash
    Crash --> Detect
    Detect --> Unack
    Unack --> Rebalance
    Rebalance --> Redeliver
    Redeliver --> Worker2
    Worker2 --> IdempotencyCheck
    IdempotencyCheck --> Execute
    Execute --> ClientView
    ClientView --> Result
    Worker1 --> SIGTERM
    SIGTERM --> FinishCurrent
    FinishCurrent --> StopConsuming
    StopConsuming --> AckComplete
    AckComplete --> CleanExit
    style Crash fill: #FF6B6B
    style Recovery fill: #FFD700
    style CleanExit fill: #90EE90
```

---

## 12. Multi-Region Deployment

**Flow Explanation:**

This diagram shows multi-region deployment for global latency reduction and disaster recovery.

**Architecture:**

- **Primary Region (US-East)**: Handles all submissions and execution
- **Secondary Regions (EU-West, Asia-Pacific)**: Read replicas for leaderboard, problem data

**Submission Flow (Global):**

1. User in Asia submits code
2. Request routed to nearest API Gateway (Asia-Pacific)
3. Submission forwarded to US-East (primary region) for execution
4. Result replicated back to Asia-Pacific (1-2 second lag)
5. User queries result from local region (low latency)

**Why Single Primary for Execution?**

- Contest fairness requires consistent execution environment
- Multi-region execution has timing differences (different hardware, network)
- Leaderboard must be globally consistent (single source of truth)

**Disaster Recovery:**

- Primary failure: Promote EU-West to primary
- RTO: 10 minutes (manual promotion)
- RPO: 1 minute (queue messages buffered in Kafka)

**Benefits:**

- Low latency API (< 100ms globally)
- High availability (survive region failure)
- Contest fairness (consistent execution)

**Trade-offs:**

- Higher latency for Asia users (submission → US → result: +200ms)
- Single region execution (no geographic distribution of load)

```mermaid
graph TB
    subgraph Users["Global Users"]
        USUser["US User"]
        EUUser["EU User"]
        AsiaUser["Asia User"]
    end

    subgraph APIGateways["Regional API Gateways"]
        USAPI["US-East API<br/>Latency: Local"]
        EUAPI["EU-West API<br/>Latency: Local"]
        AsiaAPI["Asia-Pacific API<br/>Latency: Local"]
    end

    subgraph PrimaryRegion["Primary Region US-East"]
        USSubmission["Submission Service"]
        USQueue["Kafka Queue"]
        USWorkers["Judge Workers<br/>100+ servers<br/>Handles ALL execution"]
        USDB["PostgreSQL Primary<br/>Submissions<br/>Verdicts"]
    end

    subgraph SecondaryEU["Secondary Region EU-West"]
        EURead["Read Replica<br/>Leaderboard<br/>Problem data<br/>Lag: 1-2 seconds"]
    end

    subgraph SecondaryAsia["Secondary Region Asia-Pacific"]
        AsiaRead["Read Replica<br/>Leaderboard<br/>Problem data<br/>Lag: 1-2 seconds"]
    end

    subgraph Replication["Cross-Region Replication"]
        PGRepl["PostgreSQL Replication<br/>Async streaming<br/>1-2 second lag"]
    end

    subgraph DisasterRecovery["Disaster Recovery"]
        Failover["Primary Failure<br/>Promote EU-West<br/>RTO: 10 min<br/>RPO: 1 min"]
    end

    USUser --> USAPI
    EUUser --> EUAPI
    AsiaUser --> AsiaAPI
    USAPI --> USSubmission
    EUAPI --> USSubmission
    AsiaAPI --> USSubmission
    USSubmission --> USQueue
    USQueue --> USWorkers
    USWorkers --> USDB
    USDB --> PGRepl
    PGRepl --> EURead
    PGRepl --> AsiaRead
    EUUser --> EURead
    AsiaUser --> AsiaRead
    USDB -.->|If fails| Failover
    Failover -.->|Promote| EURead
    style PrimaryRegion fill: #FFD700
    style USWorkers fill: #FF6B6B
    style Failover fill: #90EE90
```