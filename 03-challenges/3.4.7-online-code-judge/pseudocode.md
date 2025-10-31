# Online Code Judge - Pseudocode Implementations

This document contains detailed algorithm implementations for the Online Code Judge system. The main challenge document
references these functions.

---

## Table of Contents

1. [Submission Handling](#1-submission-handling)
2. [Queue Management](#2-queue-management)
3. [Docker Container Execution](#3-docker-container-execution)
4. [Test Case Execution](#4-test-case-execution)
5. [Verdict Determination](#5-verdict-determination)
6. [Security and Isolation](#6-security-and-isolation)
7. [Auto-Scaling Logic](#7-auto-scaling-logic)
8. [Caching Strategy](#8-caching-strategy)
9. [Real-Time Notifications](#9-real-time-notifications)
10. [Leaderboard and Contest](#10-leaderboard-and-contest)

---

## 1. Submission Handling

### submitCode()

**Purpose:** Accept code submission from user and publish to queue

**Parameters:**

- user_id (string): ID of the user submitting
- problem_id (string): ID of the problem being solved
- code (string): Source code submitted
- language (string): Programming language (python3, java, cpp)

**Returns:**

- submission_id (string): Unique ID for tracking submission

**Algorithm:**

```
function submitCode(user_id, problem_id, code, language):
  // 1. Validate input
  if not isAuthenticated(user_id):
    throw AuthenticationError("User not authenticated")
  
  if len(code) > 64 * 1024:  // 64 KB limit
    throw ValidationError("Code too large")
  
  if language not in SUPPORTED_LANGUAGES:
    throw ValidationError("Unsupported language")
  
  // 2. Rate limit check
  submission_count = redis.get(f"rate_limit:{user_id}:hour")
  if submission_count >= 100:
    throw RateLimitError("Too many submissions")
  
  redis.incr(f"rate_limit:{user_id}:hour")
  redis.expire(f"rate_limit:{user_id}:hour", 3600)
  
  // 3. Generate unique submission ID
  submission_id = generateSnowflakeID()
  
  // 4. Create database record
  submission = {
    id: submission_id,
    user_id: user_id,
    problem_id: problem_id,
    code: code,
    language: language,
    status: "Queued",
    verdict: null,
    runtime: null,
    memory: null,
    timestamp: now()
  }
  
  postgres.execute(
    "INSERT INTO submissions (id, user_id, problem_id, code, language, status, timestamp) VALUES ($1, $2, $3, $4, $5, $6, $7)",
    [submission_id, user_id, problem_id, code, language, "Queued", now()]
  )
  
  // 5. Detect if contest submission
  active_contests = redis.smembers("active_contests")
  is_contest = any(problem_id in contest.problems for contest in active_contests)
  
  // 6. Publish to appropriate queue
  topic = "submissions.high_priority" if is_contest else "submissions.normal"
  partition = hash(problem_id) % NUM_PARTITIONS
  
  kafka.produce(
    topic: topic,
    partition: partition,
    key: submission_id,
    value: json.stringify(submission)
  )
  
  // 7. Cache in Redis
  redis.setex(f"submission:{submission_id}", 3600, json.stringify(submission))
  
  return submission_id
```

**Time Complexity:** O(1)

**Example Usage:**

```
submission_id = submitCode(
  user_id="user_123",
  problem_id="two_sum",
  code="def solution(nums, target): ...",
  language="python3"
)
```

---

### getSubmissionStatus()

**Purpose:** Retrieve current status of a submission

**Parameters:**

- submission_id (string): ID of the submission

**Returns:**

- submission (object): Submission object with status, verdict, runtime, memory

**Algorithm:**

```
function getSubmissionStatus(submission_id):
  // 1. Check Redis cache (80% hit rate, 1ms)
  cached = redis.get(f"submission:{submission_id}")
  if cached:
    return json.parse(cached)
  
  // 2. Query PostgreSQL (15% hit rate, 10ms)
  result = postgres.query(
    "SELECT * FROM submissions WHERE id = $1",
    [submission_id]
  )
  
  if result.rows.length > 0:
    submission = result.rows[0]
    
    // Cache in Redis
    redis.setex(f"submission:{submission_id}", 3600, json.stringify(submission))
    
    return submission
  
  // 3. Query DynamoDB archive (5% hit rate, 20ms - old submissions)
  archived = dynamodb.get(
    TableName: "submissions-archive",
    Key: {submission_id: submission_id}
  )
  
  if archived.Item:
    submission = archived.Item
    
    // Cache in Redis
    redis.setex(f"submission:{submission_id}", 3600, json.stringify(submission))
    
    return submission
  
  throw NotFoundError("Submission not found")
```

**Time Complexity:** O(1) average

**Example Usage:**

```
submission = getSubmissionStatus("sub_123")
print(submission.verdict)  // "Accepted"
```

---

## 2. Queue Management

### consumeSubmissions()

**Purpose:** Worker consumes submissions from Kafka queue

**Parameters:**

- worker_id (string): ID of the worker

**Returns:**

- None (runs indefinitely)

**Algorithm:**

```
function consumeSubmissions(worker_id):
  // 1. Subscribe to Kafka topics (high priority first)
  consumer = kafka.createConsumer(
    groupId: "judge-workers",
    clientId: worker_id
  )
  
  consumer.subscribe([
    "submissions.high_priority",
    "submissions.normal"
  ])
  
  // 2. Configure consumer to prioritize high priority
  consumer.setPriority("submissions.high_priority", 1)
  consumer.setPriority("submissions.normal", 2)
  
  // 3. Process messages
  while true:
    try:
      // Poll for messages (prioritizes high_priority)
      message = consumer.poll(timeout=1000)
      
      if message:
        submission = json.parse(message.value)
        
        // Log start
        log(f"Worker {worker_id} processing submission {submission.id}")
        
        // Execute submission
        verdict = executeSubmission(submission)
        
        // Update database
        updateSubmissionVerdict(submission.id, verdict)
        
        // Commit offset (acknowledge)
        consumer.commitOffset(message)
        
        log(f"Worker {worker_id} completed submission {submission.id}: {verdict.status}")
      
      else:
        // No messages, idle
        sleep(100)
    
    catch error:
      // Log error but continue processing
      logError(f"Worker {worker_id} error: {error}")
      
      // Don't commit offset (message will be redelivered)
      continue
```

**Time Complexity:** O(∞) runs indefinitely

**Example Usage:**

```
// Start worker
consumeSubmissions("worker-001")
```

---

### calculateQueueDepth()

**Purpose:** Calculate total queue depth for auto-scaling

**Parameters:**

- topic (string): Kafka topic name

**Returns:**

- depth (integer): Total number of unconsumed messages

**Algorithm:**

```
function calculateQueueDepth(topic):
  // 1. Get all partitions for topic
  partitions = kafka.getPartitions(topic)
  
  total_depth = 0
  
  // 2. For each partition, calculate lag
  for partition in partitions:
    // Get latest offset (producer side)
    latest_offset = kafka.getLatestOffset(topic, partition)
    
    // Get consumer group offset (consumer side)
    consumer_offset = kafka.getConsumerOffset(
      groupId: "judge-workers",
      topic: topic,
      partition: partition
    )
    
    // Lag = latest - consumed
    lag = latest_offset - consumer_offset
    
    total_depth += lag
  
  return total_depth
```

**Time Complexity:** O(P) where P is number of partitions

**Example Usage:**

```
depth = calculateQueueDepth("submissions.high_priority")
print(depth)  // 5000 messages waiting
```

---

## 3. Docker Container Execution

### createSecureContainer()

**Purpose:** Create Docker container with security policies

**Parameters:**

- image (string): Docker image name (e.g., "judge-runtime:python3.9")
- code (string): User code to execute
- memory_limit (integer): Memory limit in MB (default 512)
- cpu_limit (float): CPU limit in cores (default 1.0)

**Returns:**

- container_id (string): ID of created container

**Algorithm:**

```
function createSecureContainer(image, code, memory_limit=512, cpu_limit=1.0):
  // 1. Generate unique container name
  container_name = f"judge-{generateUUID()}"
  
  // 2. Prepare code file
  code_path = f"/tmp/{container_name}/solution"
  writeFile(code_path, code)
  
  // 3. Load security profiles
  seccomp_profile = "/etc/seccomp/judge-strict.json"
  apparmor_profile = "judge-worker"
  
  // 4. Create container with security options
  container_id = docker.run(
    image: image,
    name: container_name,
    
    // Security options
    network: "none",  // No internet access
    memory: f"{memory_limit}m",
    cpus: cpu_limit,
    pids_limit: 100,  // Max 100 processes
    read_only: true,  // Read-only root filesystem
    tmpfs: {"/tmp": "size=100m,noexec,nosuid,nodev"},
    
    // Security profiles
    security_opt: [
      f"seccomp={seccomp_profile}",
      f"apparmor={apparmor_profile}",
      "no-new-privileges"
    ],
    
    // Drop all capabilities
    cap_drop: "ALL",
    
    // Run as non-root user
    user: "1000:1000",
    
    // Detached mode
    detach: true
  )
  
  // 5. Copy code to container
  docker.cp(code_path, f"{container_id}:/tmp/solution")
  
  // 6. Set up monitoring
  startResourceMonitor(container_id, memory_limit)
  
  return container_id
```

**Time Complexity:** O(1) - 800ms average

**Example Usage:**

```
container_id = createSecureContainer(
  image="judge-runtime:python3.9",
  code="def solution(x): return x + 1",
  memory_limit=512,
  cpu_limit=1.0
)
```

---

### executeInContainer()

**Purpose:** Execute code inside container with timeout

**Parameters:**

- container_id (string): Container ID
- command (string): Command to execute
- input_data (string): stdin input
- time_limit (integer): Time limit in seconds

**Returns:**

- result (object): {stdout, stderr, exit_code, runtime, memory}

**Algorithm:**

```
function executeInContainer(container_id, command, input_data, time_limit):
  // 1. Start execution timer
  start_time = now()
  
  // 2. Prepare input
  input_path = f"/tmp/judge-input-{generateUUID()}"
  writeFile(input_path, input_data)
  
  // 3. Execute with timeout
  try:
    result = docker.exec(
      container: container_id,
      command: command,
      stdin: input_path,
      timeout: time_limit,
      detach: false
    )
    
    // 4. Calculate runtime
    runtime = (now() - start_time) * 1000  // milliseconds
    
    // 5. Get resource usage
    stats = docker.stats(container_id, stream=false)
    memory_used = stats.memory_usage / (1024 * 1024)  // MB
    
    // 6. Capture output
    stdout = result.stdout
    stderr = result.stderr
    exit_code = result.exit_code
    
    return {
      stdout: stdout,
      stderr: stderr,
      exit_code: exit_code,
      runtime: runtime,
      memory: memory_used,
      timed_out: false
    }
  
  catch TimeoutError:
    // Execution exceeded time limit
    docker.kill(container_id)
    
    return {
      stdout: "",
      stderr: "Time Limit Exceeded",
      exit_code: 124,  // Timeout exit code
      runtime: time_limit * 1000,
      memory: 0,
      timed_out: true
    }
  
  finally:
    // Cleanup input file
    deleteFile(input_path)
```

**Time Complexity:** O(T) where T is execution time

**Example Usage:**

```
result = executeInContainer(
  container_id="abc123",
  command="python3 /tmp/solution.py",
  input_data="1 2 3\n",
  time_limit=1
)
```

---

### cleanupContainer()

**Purpose:** Remove container and clean up resources

**Parameters:**

- container_id (string): Container ID to remove

**Returns:**

- None

**Algorithm:**

```
function cleanupContainer(container_id):
  try:
    // 1. Stop container (if still running)
    docker.stop(container_id, timeout=5)
    
    // 2. Remove container
    docker.rm(container_id, force=true)
    
    // 3. Clean up temporary files
    temp_dir = f"/tmp/judge-{container_id}"
    if directoryExists(temp_dir):
      deleteDirectory(temp_dir)
    
    log(f"Cleaned up container {container_id}")
  
  catch error:
    logError(f"Failed to cleanup container {container_id}: {error}")
    
    // Force remove even if error
    try:
      docker.rm(container_id, force=true, volumes=true)
    catch:
      pass
```

**Time Complexity:** O(1)

**Example Usage:**

```
cleanupContainer("abc123")
```

---

## 4. Test Case Execution

### loadTestCases()

**Purpose:** Load test cases from three-tier cache (local → CloudFront → S3)

**Parameters:**

- problem_id (string): ID of the problem

**Returns:**

- test_cases (array): List of test case objects

**Algorithm:**

```
function loadTestCases(problem_id):
  // 1. Check local cache (L1 - 95% hit rate, 5ms)
  local_cache_path = f"/var/cache/testcases/{problem_id}"
  
  if directoryExists(local_cache_path):
    test_cases = loadFromDisk(local_cache_path)
    log(f"Test cases loaded from local cache: {problem_id}")
    return test_cases
  
  // 2. Fetch from CloudFront (L2 - 99% hit rate, 50ms)
  cdn_url = f"https://cdn.judge.com/testcases/{problem_id}.tar.gz"
  
  try:
    response = http.get(cdn_url, timeout=1)
    
    if response.status == 200:
      // Extract archive to local cache
      extractTarGz(response.body, local_cache_path)
      
      test_cases = loadFromDisk(local_cache_path)
      log(f"Test cases loaded from CloudFront: {problem_id}")
      return test_cases
  
  catch error:
    log(f"CloudFront miss: {problem_id}")
  
  // 3. Fetch from S3 (L3 - 100% hit rate, 200ms)
  s3_key = f"testcases/{problem_id}.tar.gz"
  s3_response = s3.getObject(
    Bucket: "judge-testcases",
    Key: s3_key
  )
  
  // Extract archive to local cache
  extractTarGz(s3_response.Body, local_cache_path)
  
  test_cases = loadFromDisk(local_cache_path)
  log(f"Test cases loaded from S3: {problem_id}")
  return test_cases
```

**Time Complexity:** O(1) average

**Example Usage:**

```
test_cases = loadTestCases("two_sum")
// Returns: [{input: "...", output: "..."}, ...]
```

---

### runTestCases()

**Purpose:** Execute code against all test cases

**Parameters:**

- container_id (string): Container ID
- language (string): Programming language
- test_cases (array): List of test case objects
- time_limit (integer): Time limit per test case in seconds
- memory_limit (integer): Memory limit in MB

**Returns:**

- verdict (object): {status, passed, failed, runtime, memory, first_failure}

**Algorithm:**

```
function runTestCases(container_id, language, test_cases, time_limit, memory_limit):
  // 1. Initialize result
  verdict = {
    status: "Accepted",
    passed: 0,
    failed: 0,
    runtime: 0,  // max runtime across all tests
    memory: 0,   // max memory across all tests
    first_failure: null
  }
  
  // 2. Get execution command for language
  command = getLanguageCommand(language)
  
  // 3. Execute each test case
  for i, test_case in enumerate(test_cases):
    // Prepare input
    input_data = test_case.input
    expected_output = test_case.output
    
    // Execute
    result = executeInContainer(
      container_id: container_id,
      command: command,
      input_data: input_data,
      time_limit: time_limit
    )
    
    // Update max runtime and memory
    verdict.runtime = max(verdict.runtime, result.runtime)
    verdict.memory = max(verdict.memory, result.memory)
    
    // Check for errors
    if result.timed_out:
      verdict.status = "Time Limit Exceeded"
      verdict.failed = len(test_cases) - i
      verdict.first_failure = i
      break
    
    if result.memory > memory_limit:
      verdict.status = "Memory Limit Exceeded"
      verdict.failed = len(test_cases) - i
      verdict.first_failure = i
      break
    
    if result.exit_code != 0:
      verdict.status = "Runtime Error"
      verdict.failed = len(test_cases) - i
      verdict.first_failure = i
      break
    
    // Compare output
    actual_output = result.stdout.strip()
    expected_output_clean = expected_output.strip()
    
    if actual_output == expected_output_clean:
      verdict.passed += 1
    else:
      verdict.status = "Wrong Answer"
      verdict.failed = len(test_cases) - i
      verdict.first_failure = i
      break
  
  return verdict
```

**Time Complexity:** O(N × T) where N is number of test cases, T is execution time per test

**Example Usage:**

```
verdict = runTestCases(
  container_id="abc123",
  language="python3",
  test_cases=[{input: "1 2", output: "3"}, {input: "4 5", output: "9"}],
  time_limit=1,
  memory_limit=512
)
```

---

## 5. Verdict Determination

### executeSubmission()

**Purpose:** Main entry point to execute a submission

**Parameters:**

- submission (object): Submission object from queue

**Returns:**

- verdict (object): Final verdict with status, runtime, memory

**Algorithm:**

```
function executeSubmission(submission):
  container_id = null
  
  try:
    // 1. Load test cases
    test_cases = loadTestCases(submission.problem_id)
    
    // 2. Get problem constraints
    problem = getProblemMetadata(submission.problem_id)
    time_limit = problem.time_limit  // seconds
    memory_limit = problem.memory_limit  // MB
    
    // 3. Create secure container
    image = getLanguageImage(submission.language)
    container_id = createSecureContainer(
      image: image,
      code: submission.code,
      memory_limit: memory_limit,
      cpu_limit: 1.0
    )
    
    // 4. Compile if needed (Java, C++, etc.)
    if requiresCompilation(submission.language):
      compilation_result = compileCode(
        container_id: container_id,
        language: submission.language,
        time_limit: 10  // 10 seconds compile time
      )
      
      if not compilation_result.success:
        return {
          status: "Compilation Error",
          passed: 0,
          failed: len(test_cases),
          runtime: 0,
          memory: 0,
          error_message: compilation_result.error
        }
    
    // 5. Run test cases
    verdict = runTestCases(
      container_id: container_id,
      language: submission.language,
      test_cases: test_cases,
      time_limit: time_limit,
      memory_limit: memory_limit
    )
    
    return verdict
  
  catch error:
    logError(f"Execution error for submission {submission.id}: {error}")
    
    return {
      status: "System Error",
      passed: 0,
      failed: len(test_cases),
      runtime: 0,
      memory: 0,
      error_message: str(error)
    }
  
  finally:
    // Always cleanup container
    if container_id:
      cleanupContainer(container_id)
```

**Time Complexity:** O(N × T) where N is number of test cases, T is execution time

**Example Usage:**

```
submission = {
  id: "sub_123",
  user_id: "user_456",
  problem_id: "two_sum",
  code: "def solution(nums, target): ...",
  language: "python3"
}

verdict = executeSubmission(submission)
```

---

### updateSubmissionVerdict()

**Purpose:** Update submission verdict in database and cache

**Parameters:**

- submission_id (string): Submission ID
- verdict (object): Verdict object with status, runtime, memory

**Returns:**

- None

**Algorithm:**

```
function updateSubmissionVerdict(submission_id, verdict):
  // 1. Update PostgreSQL
  postgres.execute(
    "UPDATE submissions SET status = $1, verdict = $2, runtime = $3, memory = $4 WHERE id = $5",
    ["Completed", verdict.status, verdict.runtime, verdict.memory, submission_id]
  )
  
  // 2. Update Redis cache
  submission = getSubmissionStatus(submission_id)
  submission.status = "Completed"
  submission.verdict = verdict.status
  submission.runtime = verdict.runtime
  submission.memory = verdict.memory
  
  redis.setex(
    f"submission:{submission_id}",
    3600,
    json.stringify(submission)
  )
  
  // 3. Publish real-time update
  redis.publish(
    f"submission:{submission_id}",
    json.stringify({
      status: "Completed",
      verdict: verdict.status,
      runtime: verdict.runtime,
      memory: verdict.memory
    })
  )
  
  // 4. Update contest leaderboard if needed
  if isContestSubmission(submission_id):
    updateLeaderboard(submission)
```

**Time Complexity:** O(1)

**Example Usage:**

```
verdict = {status: "Accepted", runtime: 234, memory: 45, passed: 10, failed: 0}
updateSubmissionVerdict("sub_123", verdict)
```

---

## 6. Security and Isolation

### enforceSecurityPolicy()

**Purpose:** Validate security policies are properly applied

**Parameters:**

- container_id (string): Container ID

**Returns:**

- is_secure (boolean): True if all security checks pass

**Algorithm:**

```
function enforceSecurityPolicy(container_id):
  // 1. Verify seccomp profile loaded
  seccomp_status = docker.inspect(container_id).SecurityOpt
  if "seccomp" not in seccomp_status:
    logError(f"Seccomp not enabled for container {container_id}")
    return false
  
  // 2. Verify AppArmor profile loaded
  if "apparmor" not in seccomp_status:
    logError(f"AppArmor not enabled for container {container_id}")
    return false
  
  // 3. Verify network isolation
  network_mode = docker.inspect(container_id).HostConfig.NetworkMode
  if network_mode != "none":
    logError(f"Network not isolated for container {container_id}")
    return false
  
  // 4. Verify cgroups limits
  memory_limit = docker.inspect(container_id).HostConfig.Memory
  if memory_limit == 0:
    logError(f"Memory limit not set for container {container_id}")
    return false
  
  cpu_limit = docker.inspect(container_id).HostConfig.CpuQuota
  if cpu_limit == -1:
    logError(f"CPU limit not set for container {container_id}")
    return false
  
  // 5. Verify read-only filesystem
  read_only = docker.inspect(container_id).HostConfig.ReadonlyRootfs
  if not read_only:
    logError(f"Filesystem not read-only for container {container_id}")
    return false
  
  // All checks passed
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
is_secure = enforceSecurityPolicy("abc123")
if not is_secure:
  cleanupContainer("abc123")
  raise SecurityViolationError("Container failed security checks")
```

---

### detectSecurityViolation()

**Purpose:** Detect if code attempted security violation

**Parameters:**

- stderr (string): Standard error output from execution
- exit_code (integer): Process exit code

**Returns:**

- violation (object): {detected, type, description}

**Algorithm:**

```
function detectSecurityViolation(stderr, exit_code):
  violation = {
    detected: false,
    type: null,
    description: null
  }
  
  // 1. Check for permission denied errors
  if "Permission denied" in stderr or "EPERM" in stderr:
    violation.detected = true
    violation.type = "PERMISSION_DENIED"
    violation.description = "Attempted to access restricted resource"
    return violation
  
  // 2. Check for network access attempts
  if "Network unreachable" in stderr or "socket" in stderr.lower():
    violation.detected = true
    violation.type = "NETWORK_ACCESS"
    violation.description = "Attempted to make network connection"
    return violation
  
  // 3. Check for seccomp violations (syscall blocked)
  if exit_code == 159:  // SIGSYS - bad system call
    violation.detected = true
    violation.type = "SYSCALL_BLOCKED"
    violation.description = "Attempted to make prohibited system call"
    return violation
  
  // 4. Check for file access violations
  if "Read-only file system" in stderr:
    violation.detected = true
    violation.type = "FILE_WRITE"
    violation.description = "Attempted to write to read-only filesystem"
    return violation
  
  // 5. Check for fork bomb attempts
  if "Resource temporarily unavailable" in stderr and "fork" in stderr.lower():
    violation.detected = true
    violation.type = "FORK_BOMB"
    violation.description = "Attempted to create too many processes"
    return violation
  
  return violation
```

**Time Complexity:** O(N) where N is length of stderr

**Example Usage:**

```
violation = detectSecurityViolation(
  stderr="socket: Permission denied",
  exit_code=1
)

if violation.detected:
  banUser(user_id, violation.description)
```

---

## 7. Auto-Scaling Logic

### calculateDesiredWorkers()

**Purpose:** Calculate desired number of workers based on queue depth

**Parameters:**

- queue_depth (integer): Current queue depth
- target_per_worker (integer): Target messages per worker (default 1000)

**Returns:**

- desired_workers (integer): Desired number of worker pods

**Algorithm:**

```
function calculateDesiredWorkers(queue_depth, target_per_worker=1000):
  // 1. Calculate desired workers
  desired = ceil(queue_depth / target_per_worker)
  
  // 2. Apply min/max constraints
  MIN_WORKERS = 10
  MAX_WORKERS = 500
  
  desired = max(MIN_WORKERS, desired)
  desired = min(MAX_WORKERS, desired)
  
  // 3. Log calculation
  log(f"Queue depth: {queue_depth}, Target: {target_per_worker}/worker, Desired: {desired} workers")
  
  return desired
```

**Time Complexity:** O(1)

**Example Usage:**

```
queue_depth = 50000
desired_workers = calculateDesiredWorkers(queue_depth)
// Returns: 50 (50000 / 1000 = 50)
```

---

### scaleWorkerFleet()

**Purpose:** Scale worker fleet to desired size

**Parameters:**

- current_workers (integer): Current number of workers
- desired_workers (integer): Desired number of workers

**Returns:**

- None

**Algorithm:**

```
function scaleWorkerFleet(current_workers, desired_workers):
  if desired_workers == current_workers:
    return  // No change needed
  
  if desired_workers > current_workers:
    // Scale up
    scale_up_count = desired_workers - current_workers
    
    log(f"Scaling up: adding {scale_up_count} workers")
    
    // Update Kubernetes deployment
    kubernetes.scale(
      deployment: "judge-worker",
      replicas: desired_workers
    )
  
  elif desired_workers < current_workers:
    // Scale down (with stabilization window)
    scale_down_count = current_workers - desired_workers
    
    // Only scale down by 10% max per iteration
    max_scale_down = max(1, current_workers * 0.1)
    actual_scale_down = min(scale_down_count, max_scale_down)
    
    new_count = current_workers - actual_scale_down
    
    log(f"Scaling down: removing {actual_scale_down} workers")
    
    // Update Kubernetes deployment
    kubernetes.scale(
      deployment: "judge-worker",
      replicas: new_count
    )
```

**Time Complexity:** O(1)

**Example Usage:**

```
scaleWorkerFleet(current_workers=10, desired_workers=50)
// Scales up to 50 workers
```

---

### autoScaleLoop()

**Purpose:** Continuously monitor queue and auto-scale

**Parameters:**

- None

**Returns:**

- None (runs indefinitely)

**Algorithm:**

```
function autoScaleLoop():
  while true:
    try:
      // 1. Calculate queue depth
      high_priority_depth = calculateQueueDepth("submissions.high_priority")
      normal_depth = calculateQueueDepth("submissions.normal")
      
      // 2. Calculate desired workers
      // High priority: 500 messages/worker (faster response)
      // Normal: 1000 messages/worker (more efficient)
      desired_high = calculateDesiredWorkers(high_priority_depth, 500)
      desired_normal = calculateDesiredWorkers(normal_depth, 1000)
      
      total_desired = desired_high + desired_normal
      
      // 3. Get current worker count
      current_workers = kubernetes.getReplicaCount("judge-worker")
      
      // 4. Scale if needed
      if total_desired != current_workers:
        scaleWorkerFleet(current_workers, total_desired)
      
      // 5. Sleep before next check
      sleep(15)  // Check every 15 seconds
    
    catch error:
      logError(f"Auto-scale error: {error}")
      sleep(60)  // Wait longer on error
```

**Time Complexity:** O(∞) runs indefinitely

**Example Usage:**

```
// Start auto-scaling loop
autoScaleLoop()
```

---

## 8. Caching Strategy

### cacheTestCases()

**Purpose:** Cache test cases in local storage

**Parameters:**

- problem_id (string): Problem ID
- test_cases_archive (bytes): Compressed archive of test cases

**Returns:**

- cache_path (string): Path to cached test cases

**Algorithm:**

```
function cacheTestCases(problem_id, test_cases_archive):
  // 1. Create cache directory
  cache_path = f"/var/cache/testcases/{problem_id}"
  
  if not directoryExists(cache_path):
    createDirectory(cache_path)
  
  // 2. Extract archive
  extractTarGz(test_cases_archive, cache_path)
  
  // 3. Verify extraction
  if not directoryExists(cache_path):
    throw CacheError("Failed to extract test cases")
  
  // 4. Set permissions (read-only)
  setPermissions(cache_path, "0444")
  
  log(f"Cached test cases for problem {problem_id} at {cache_path}")
  
  return cache_path
```

**Time Complexity:** O(N) where N is archive size

**Example Usage:**

```
archive_bytes = downloadFromS3("testcases/two_sum.tar.gz")
cache_path = cacheTestCases("two_sum", archive_bytes)
```

---

### invalidateCache()

**Purpose:** Invalidate local cache when problem updated

**Parameters:**

- problem_id (string): Problem ID to invalidate

**Returns:**

- None

**Algorithm:**

```
function invalidateCache(problem_id):
  // 1. Delete local cache
  cache_path = f"/var/cache/testcases/{problem_id}"
  
  if directoryExists(cache_path):
    deleteDirectory(cache_path)
    log(f"Invalidated cache for problem {problem_id}")
  
  // 2. Notify CloudFront to invalidate
  cloudfront.createInvalidation(
    DistributionId: "DISTRIBUTION_ID",
    Paths: [f"/testcases/{problem_id}.tar.gz"]
  )
```

**Time Complexity:** O(1)

**Example Usage:**

```
// When problem updated
invalidateCache("two_sum")
```

---

### subscribeToCacheInvalidation()

**Purpose:** Subscribe to Redis Pub/Sub for cache invalidation events

**Parameters:**

- None

**Returns:**

- None (runs indefinitely)

**Algorithm:**

```
function subscribeToCacheInvalidation():
  // 1. Create Redis subscriber
  subscriber = redis.createSubscriber()
  
  // 2. Subscribe to problem update channel
  subscriber.subscribe("problem:updated")
  
  // 3. Handle messages
  subscriber.on("message", (channel, message) => {
    try:
      event = json.parse(message)
      problem_id = event.problem_id
      
      log(f"Received cache invalidation for problem {problem_id}")
      
      // Invalidate local cache
      invalidateCache(problem_id)
    
    catch error:
      logError(f"Cache invalidation error: {error}")
  })
  
  log("Subscribed to cache invalidation events")
```

**Time Complexity:** O(∞) runs indefinitely

**Example Usage:**

```
// Start cache invalidation listener
subscribeToCacheInvalidation()
```

---

## 9. Real-Time Notifications

### publishSubmissionUpdate()

**Purpose:** Publish submission status update to Redis Pub/Sub

**Parameters:**

- submission_id (string): Submission ID
- update (object): Update object with status, test progress, etc.

**Returns:**

- None

**Algorithm:**

```
function publishSubmissionUpdate(submission_id, update):
  // 1. Add timestamp
  update.timestamp = now()
  
  // 2. Publish to Redis Pub/Sub
  channel = f"submission:{submission_id}"
  message = json.stringify(update)
  
  redis.publish(channel, message)
  
  log(f"Published update for submission {submission_id}: {update.status}")
```

**Time Complexity:** O(1)

**Example Usage:**

```
publishSubmissionUpdate("sub_123", {
  status: "Running",
  test: 5,
  total: 10
})
```

---

### handleWebSocketConnection()

**Purpose:** Handle WebSocket connection for real-time updates

**Parameters:**

- socket (WebSocket): WebSocket connection
- submission_id (string): Submission ID to subscribe to

**Returns:**

- None

**Algorithm:**

```
function handleWebSocketConnection(socket, submission_id):
  // 1. Create Redis subscriber
  subscriber = redis.createSubscriber()
  
  // 2. Subscribe to submission channel
  channel = f"submission:{submission_id}"
  subscriber.subscribe(channel)
  
  // 3. Forward messages to WebSocket
  subscriber.on("message", (channel, message) => {
    try:
      // Send to client
      socket.send(message)
      
      // Parse to check if complete
      update = json.parse(message)
      if update.status == "Completed":
        // Close connection after final update
        socket.close()
        subscriber.unsubscribe(channel)
    
    catch error:
      logError(f"WebSocket send error: {error}")
  })
  
  // 4. Handle WebSocket close
  socket.on("close", () => {
    subscriber.unsubscribe(channel)
    log(f"WebSocket closed for submission {submission_id}")
  })
  
  // 5. Send initial status
  submission = getSubmissionStatus(submission_id)
  socket.send(json.stringify({
    status: submission.status,
    verdict: submission.verdict,
    runtime: submission.runtime,
    memory: submission.memory
  }))
```

**Time Complexity:** O(∞) runs until socket closes

**Example Usage:**

```
// On WebSocket connection
wss.on("connection", (socket, request) => {
  submission_id = parseURL(request.url).submission_id
  handleWebSocketConnection(socket, submission_id)
})
```

---

## 10. Leaderboard and Contest

### updateLeaderboard()

**Purpose:** Update contest leaderboard after submission

**Parameters:**

- submission (object): Submission object with verdict

**Returns:**

- None

**Algorithm:**

```
function updateLeaderboard(submission):
  // 1. Get contest info
  contest_id = getActiveContest(submission.problem_id)
  if not contest_id:
    return  // Not a contest submission
  
  // 2. Check if accepted
  if submission.verdict != "Accepted":
    return  // Only accepted submissions count
  
  // 3. Get user's current score
  user_id = submission.user_id
  score_key = f"contest:{contest_id}:user:{user_id}"
  
  current_score = redis.hgetall(score_key)
  
  // 4. Check if already solved this problem
  problem_solved_key = f"solved_problems:{submission.problem_id}"
  if problem_solved_key in current_score:
    return  // Already solved, don't count again
  
  // 5. Calculate penalty (time from contest start to submission)
  contest_start = redis.get(f"contest:{contest_id}:start_time")
  penalty = (submission.timestamp - contest_start) / 60  // minutes
  
  // 6. Update user score
  redis.hincrby(score_key, "problems_solved", 1)
  redis.hincrbyfloat(score_key, "total_penalty", penalty)
  redis.hset(score_key, problem_solved_key, "1")
  
  // 7. Update sorted leaderboard
  leaderboard_key = f"contest:{contest_id}:leaderboard"
  
  // Score formula: (problems_solved * 1,000,000) - total_penalty
  // This ensures problems_solved is primary sort, penalty is tiebreaker
  problems_solved = int(redis.hget(score_key, "problems_solved"))
  total_penalty = float(redis.hget(score_key, "total_penalty"))
  
  score = (problems_solved * 1000000) - total_penalty
  
  redis.zadd(leaderboard_key, {user_id: score})
  
  // 8. Get user's rank
  rank = redis.zrevrank(leaderboard_key, user_id) + 1
  
  log(f"Updated leaderboard for user {user_id}: rank {rank}, {problems_solved} problems, {total_penalty} penalty")
```

**Time Complexity:** O(log N) where N is number of participants

**Example Usage:**

```
submission = {
  id: "sub_123",
  user_id: "user_456",
  problem_id: "two_sum",
  verdict: "Accepted",
  timestamp: 1699999999
}

updateLeaderboard(submission)
```

---

### getLeaderboard()

**Purpose:** Retrieve current leaderboard rankings

**Parameters:**

- contest_id (string): Contest ID
- start (integer): Start rank (0-indexed)
- end (integer): End rank (0-indexed)

**Returns:**

- leaderboard (array): List of {rank, user_id, problems_solved, penalty}

**Algorithm:**

```
function getLeaderboard(contest_id, start=0, end=99):
  // 1. Get top users from sorted set
  leaderboard_key = f"contest:{contest_id}:leaderboard"
  
  top_users = redis.zrevrange(
    key: leaderboard_key,
    start: start,
    end: end,
    withscores: true
  )
  
  // 2. Build leaderboard
  leaderboard = []
  
  for i, (user_id, score) in enumerate(top_users):
    // Decode score
    problems_solved = floor(score / 1000000)
    total_penalty = (problems_solved * 1000000) - score
    
    // Get user details
    score_key = f"contest:{contest_id}:user:{user_id}"
    user_score = redis.hgetall(score_key)
    
    leaderboard.append({
      rank: start + i + 1,
      user_id: user_id,
      problems_solved: problems_solved,
      penalty: total_penalty,
      last_submission: user_score.get("last_submission_time")
    })
  
  return leaderboard
```

**Time Complexity:** O(log N + K) where N is total users, K is result size

**Example Usage:**

```
leaderboard = getLeaderboard("contest_123", start=0, end=99)
// Returns top 100 users
```