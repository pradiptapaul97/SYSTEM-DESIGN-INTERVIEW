# Server-Side Engineering & Large-Scale System Design: Interview Prep Guide

This guide compiles high-frequency backend engineering, scaling, caching, API design, security, reliability, observability, and classic system design interview solutions.

---

## 📌 Table of Contents
1. [Node.js Concurrency Basics](#1-nodejs-concurrency-basics)
2. [Concurrency & Race Conditions (Application Level)](#2-concurrency--race-conditions-application-level)
3. [Distributed Systems & Locking](#3-distributed-systems--locking)
4. [Job Processing & Queue Design](#4-job-processing--queue-design)
5. [High Traffic & Scalability](#5-high-traffic--scalability)
6. [Caching Strategies & Patterns](#6-caching-strategies--patterns)
7. [API Design & Protocols](#7-api-design--protocols)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Real-Time Systems](#9-real-time-systems)
10. [Microservices Architecture](#10-microservices-architecture)
11. [Reliability & Fault Tolerance](#11-reliability--fault-tolerance)
12. [Data Processing Pipelines](#12-data-processing-pipelines)
13. [Observability & Monitoring](#13-observability--monitoring)
14. [Multi-Tenant SaaS Systems](#14-multi-tenant-saas-systems)
15. [Classic System Design Solutions](#15-classic-system-design-solutions)

---

## 1. Node.js Concurrency Basics

### Q1: How does Node.js handle thousands of concurrent requests despite being single-threaded?
*   **The Core Concept:** Node.js achieves high concurrency by utilizing an **Event-Driven, Non-Blocking I/O Model** managed by the **Libuv** C++ library.
*   **Detailed Explanation:**
    1.  **Single Main Thread:** Node.js runs JavaScript on a single thread (the Event Loop), meaning only one JS instruction executes at a time.
    2.  **Delegation of I/O:** When an asynchronous I/O operation (e.g., network request, database query) is initiated, Node.js doesn't block the main thread waiting for the response. Instead, it delegates the operation to the OS Kernel (which uses native async interfaces like `epoll` on Linux, `kqueue` on macOS, or `IOCP` on Windows).
    3.  **Libuv Thread Pool:** For operations that cannot be handled asynchronously by the OS kernel (e.g., file system access, DNS queries, cryptographic functions), Libuv delegates them to its internal thread pool (default size: 4).
    4.  **Callback Queue:** Once the kernel or the thread pool completes the task, it pushes the associated callback into Node's execution queue. The Event Loop picks it up when the main thread becomes idle.
*   **Interview Tip:** Emphasize that Node.js is "single-threaded" only for JavaScript execution, but multi-threaded under the hood for I/O operations.

---

### Q2: What is the Event Loop, and how does it help concurrency?
*   **The Core Concept:** The Event Loop is an infinite loop managed by Libuv that continuously checks for, dispatches, and executes asynchronous callbacks in a strict order.
*   **Visualizing the Event Loop Phases:**
    ```mermaid
    graph TD
        A[Start] --> B["1. Timers (setTimeout, setInterval)"]
        B --> C["2. Pending Callbacks (Deferred System I/O)"]
        C --> D["3. Idle, Prepare (Internal Libuv)"]
        D --> E["4. Poll (Incoming Connections, Execute I/O Callbacks)"]
        E --> F["5. Check (setImmediate)"]
        F --> G["6. Close Callbacks (socket.on('close'))"]
        G --> H{Are there active handles/requests?}
        H -- Yes --> B
        H -- No --> I[Exit Program]
        
        style B fill:#f9f,stroke:#333,stroke-width:2px
        style E fill:#9cf,stroke:#333,stroke-width:2px
        style F fill:#f96,stroke:#333,stroke-width:2px
    ```
*   **Detailed Phase Breakdown:**
    1.  **Timers:** Executes callbacks scheduled by `setTimeout()` and `setInterval()`.
    2.  **Pending Callbacks:** Executes I/O callbacks deferred from the previous loop iteration (e.g., TCP errors).
    3.  **Idle, Prepare:** Used internally by Libuv for house-keeping.
    4.  **Poll:** Retrieves new I/O events. If the poll queue isn't empty, it executes callbacks synchronously until the queue is exhausted or system limits are hit.
    5.  **Check:** Executes callbacks registered via `setImmediate()`.
    6.  **Close Callbacks:** Executes callbacks for closed connections (e.g., `socket.on('close')`).
*   **Microtask Queues:** Between *every single phase* of the Event Loop, Node.js drains the Microtask Queues:
    *   `process.nextTick()` Queue (Executed first with highest priority).
    *   Promise `resolve/reject` Queue (Executed second).
*   **How it Helps Concurrency:** It ensures the CPU never sits idle waiting for I/O, maximizing hardware resource utilization.

---

### Q3: What operations can block the Event Loop?
*   **The Core Concept:** Any synchronous or CPU-intensive task running on the main JavaScript thread halts the Event Loop, preventing it from executing callbacks, routing requests, or accepting new connections.
*   **Common Blockers:**
    1.  **CPU-Intensive Math/Computation:** Sorting a massive array, complex cryptography (e.g., synchronous `bcrypt.hashSync()`), or rendering HTML templates synchronously.
    2.  **Synchronous I/O operations:** Using functions like `fs.readFileSync()`, `fs.writeFileSync()`, or synchronous shell commands (`child_process.execSync()`).
    3.  **Large-scale JSON operations:** Parsing or stringifying extremely large JSON payloads (e.g., a 100MB array of objects) synchronously blocks the event loop.
    4.  **Catastrophic Backtracking in Regex:** Poorly written regular expressions running against malicious inputs (ReDoS attacks).
    5.  **Infinite/Long Loops:** Any `while` or `for` loop that runs for millions of iterations without yielding.

---

### Q4: How would you identify Event Loop blocking in production?
*   **The Core Concept:** You must monitor the lag between when a callback is scheduled and when it actually runs.
*   **Production Identification Methods:**
    1.  **Event Loop Utilization (ELU):** Using Node's native API `process.eventLoopUtilization()`. It calculates the ratio of time the event loop spent executing JS vs. idling. High ELU (>80%) indicates blocking code or heavy CPU saturation.
    2.  **Monitor Event Loop Delay:** Use the native `perf_hooks` module to measure loop delay:
        ```javascript
        const { monitorEventLoopDelay } = require('perf_hooks');
        const h = monitorEventLoopDelay({ resolution: 10 });
        h.enable();
        ```
    3.  **Performance APM Tools:** Services like Datadog, New Relic, or Dynatrace monitor loop lag out of the box by measuring how long a standard `setTimeout` callback is delayed.
    4.  **The `blocked-at` Library:** An open-source utility that alerts you in development/staging and provides a stack trace showing *exactly* which line of code blocked the loop.

---

### Q5: When would you use Worker Threads?
*   **The Core Concept:** Use **Worker Threads** (`worker_threads` module) when you need to execute CPU-bound JavaScript code concurrently without blocking the main event loop.
*   **How it Works:** Workers run in their own thread space, with their own Event Loop, V8 instance, and memory. However, they can share memory with the parent thread using `SharedArrayBuffer` or pass messages via `MessagePort`.
*   **Ideal Use Cases:**
    *   In-memory data encryption/decryption.
    *   Image processing, video transcoding, or PDF generation on-the-fly.
    *   Running complex algorithms (e.g., machine learning inference or graph search).
*   **When *Not* to Use:** Do not use Worker Threads for I/O tasks. A standard async database query is already handled efficiently by Libuv/OS; adding a worker thread only adds scheduling overhead.

---

### Q6: When would you use Child Processes?
*   **The Core Concept:** Use the `child_process` module when you need to run external system commands, script files in other languages, or execute tasks that require complete process isolation.
*   **Comparison: Worker Threads vs. Child Processes:**
    *   **Worker Threads:** Run in the *same* OS process, share V8 instances, can share memory directly, and have low startup overhead.
    *   **Child Processes:** Run in *separate* OS processes with their own PID, have high memory overhead, and must communicate via IPC (Inter-Process Communication).
*   **Ideal Use Cases:**
    *   Executing system binaries (e.g., running `ffmpeg` to process video files).
    *   Executing scripts written in other languages (e.g., running a Python scraping script).
    *   Isolating tasks to prevent a crash in the child process from bringing down the main Node.js web server.

---

### Q7: How does Node.js handle concurrent I/O operations?
*   **The Core Concept:** Node.js splits I/O handling based on whether the underlying operating system provides a native asynchronous interface for that resource.
*   **Detailed Breakdown:**
    1.  **Network I/O (Asynchronous):** Operations like TCP sockets, UDP, HTTP/HTTPS are fully asynchronous. Libuv registers the socket descriptors with the OS event notification system (e.g., `epoll` on Linux). The OS notifies Node when the data arrives, using zero background threads.
    2.  **File System & DNS Lookup (Blocking under-the-hood):** Most OS kernels do not support asynchronous file operations. Therefore, Libuv relies on its **Thread Pool** to perform file I/O and DNS resolutions (`dns.lookup`). The calling JS thread receives a promise and yields, while a background thread in the pool performs the blocking OS call and notifies the Event Loop upon completion.
*   **Tuning Production Performance:** If your app is file-I/O heavy, you must increase the thread pool size via the environment variable `UV_THREADPOOL_SIZE` (default is 4, max is 1024) to prevent thread starvation.

---

## 2. Concurrency & Race Conditions (Application Level)

### Q1: Two users update the same resource simultaneously. How do you prevent race conditions?
*   **The Scenario:** User A and User B read a database row with `version = 1`. Both modify it and write it back. User B writes second, silently overwriting User A's changes (Lost Update anomaly).
*   **Solutions (Interview Checklist):**
    1.  **Optimistic Concurrency Control (OCC) / Versioning:**
        *   Add a `version` or `updated_at` column to the table.
        *   When updating, execute: `UPDATE users SET email = 'new@email.com', version = version + 1 WHERE id = 123 AND version = 1`.
        *   If the row count returned is `0`, it means another process updated it first. You roll back/retry.
    2.  **Pessimistic Locking:**
        *   Lock the row during read: `SELECT * FROM users WHERE id = 123 FOR UPDATE;`.
        *   This blocks other transactions from reading or updating the row until your transaction commits.
    3.  **Distributed Lock (Redis):**
        *   Acquire a temporary lock in Redis on the resource key (`lock:user:123`) before updating, and release it after writing.

---

### Q2: Multiple requests attempt to update the same wallet balance. What could go wrong?
*   **The Danger:** If your code fetches the current balance, adds a value in memory, and writes the new total back:
    1.  Request 1 reads balance: `$100`.
    2.  Request 2 reads balance: `$100`.
    3.  Request 1 adds `$50` -> writes `$150`.
    4.  Request 2 adds `$20` -> writes `$120` (Request 1's `$50` is lost!).
*   **How to Solve It:**
    1.  **Atomic Database Update (No Read-Modify-Write):** Use SQL arithmetic directly in the database:
        `UPDATE wallets SET balance = balance + 50 WHERE user_id = 456;`
        This delegates the serialization of the operations to the database's internal lock manager.
    2.  **Pessimistic Locking in DB Transaction:**
        ```sql
        START TRANSACTION;
        SELECT balance FROM wallets WHERE user_id = 456 FOR UPDATE;
        -- calculate new balance in Node.js --
        UPDATE wallets SET balance = 150 WHERE user_id = 456;
        COMMIT;
        ```

---

### Q3: How would you prevent duplicate order creation?
*   **The Scenario:** A user double-clicks the "Place Order" button, sending two identical HTTP requests milliseconds apart.
*   **Prevention Strategies:**
    1.  **Client-Side Throttling:** Disable the submit button immediately upon the first click and show a loading spinner.
    2.  **API Idempotency Key (Best Practice):**
        *   The frontend generates a unique UUID for the checkout session.
        *   The checkout API registers this key in Redis/DB using a unique constraint.
        *   If a second request arrives with the same key, it is blocked or served the cached response of the first transaction.
    3.  **Distributed Lock on User Cart:**
        *   Before starting the checkout transaction, acquire a Redis lock on `lock:checkout:user_id` with a 10-second TTL.
        *   If the second request fails to acquire the lock, return a `409 Conflict`.

---

### Q4: How would you prevent duplicate payment processing?
*   **The Scenario:** A network timeout occurs after a Node.js server sends a charge request to Stripe. Node doesn't know if the payment went through, and retrying might charge the customer twice.
*   **Design Solution:**
    1.  **Use Gateway Idempotency Keys:** Pass a unique `Idempotency-Key` (e.g., `idempotency_key_order_123`) to Stripe/Adyen. Payment processors guarantee they will only process the payment once per key.
    2.  **Database State Machine:**
        *   Store payment status as states: `CREATED` -> `PENDING` -> `SUCCESS` / `FAILED`.
        *   Before initiating payment, update database status to `PENDING` using optimistic locking to ensure no other worker can touch it.
    3.  **Strict Isolation Check:** If a retry request is received, query the database status. If it is already `PENDING` or `SUCCESS`, abort immediately and return the status.

---

## 3. Distributed Systems & Locking

### Q1: Your application runs on 10 Node.js servers behind a load balancer. How would you implement locking?
*   **The Problem:** Standard in-memory locks (like Node's `async-mutex`) only work within a single process. Since the 10 servers do not share memory, they cannot coordinate locks locally.
*   **The Solution:** Use an external, centralized system as a **Distributed Lock Manager (DLM)**.
*   **Candidate Systems:**
    *   **Redis:** Best for speed and temporary locks (high throughput, eventual safety).
    *   **PostgreSQL/MySQL:** Good for simple locks using raw transactional table rows or advisory locks (e.g., `pg_advisory_lock`), but has lower scale limits.
    *   **Consul / ZooKeeper:** Best for strict consistency and tolerance against network partitions.

---

### Q2: What is a distributed lock?
*   **Definition:** A distributed lock is a synchronization mechanism used to coordinate access to a shared resource (e.g., a database row, a file, a background job) across multiple independent servers or processes.
*   **Three Safety Properties of a Distributed Lock:**
    1.  **Mutual Exclusion:** Only one client can hold the lock at any given time.
    2.  **Deadlock Free:** The lock must eventually be released (even if the client that acquired it crashes or gets partitioned from the network—typically solved via a Time To Live/TTL).
    3.  **Fault Tolerance:** The lock storage system must not have a single point of failure (SPOF).

---

### Q3: How would you implement a distributed lock using Redis?
*   **Acquiring the Lock:** Use the atomic `SET` command with parameters:
    ```javascript
    const lockKey = 'lock:resource_name';
    const uniqueValue = uuidv4(); // Unique token to identify the lock owner
    const ttlMs = 30000; // 30 seconds expiration

    // SET key value NX (only if key doesn't exist) PX ttl (milliseconds)
    const acquired = await redis.set(lockKey, uniqueValue, 'NX', 'PX', ttlMs);
    ```
*   **Releasing the Lock:** You must *only* release the lock if you own it. This requires checking the value before deleting. To do this atomically, execute a **Lua Script**:
    ```lua
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    ```
    *   *Why Lua?* Because it executes atomically on the Redis thread, ensuring no context switches happen between checking the lock owner and deleting the lock.

---

### Q4: What is the Redlock algorithm?
*   **The Context:** Redlock is a distributed lock algorithm designed by the creator of Redis to achieve high availability and safety without relying on a single Redis master node.
*   **How it Works (with 5 independent Redis Masters):**
    1.  The client records the current timestamp.
    2.  The client attempts to acquire the lock in all 5 instances sequentially, using the same key name and a random unique value.
    3.  The client calculates the time taken to acquire the locks. If it successfully acquires the lock from a **majority** (at least 3 of 5) of the nodes, and the total elapsed time is *less* than the lock validity time, the lock is acquired.
    4.  The lock's actual lease time is the initial TTL minus the time spent acquiring it.
    5.  If acquisition fails (less than 3 nodes, or lease time is negative), the client immediately sends a delete Lua script to all 5 instances to clean up partial locks.

---

### Q5: What are the limitations of Redis-based locking?
*   **The Limitations:**
    1.  **Clock Drift Vulnerability:** Redlock relies on physical system clocks across Redis nodes. If one node's clock drifts forward rapidly, it might expire the lock prematurely, leading to a violation of mutual exclusion.
    2.  **Asynchronous Replication Loss:** In a standard master-replica Redis setup, writes are asynchronous. If a client acquires a lock on the Master, and the Master crashes before replicating the lock to the Replica, the Replica will be promoted to Master without the lock state. A second client can now acquire the same lock.
    3.  **Garbage Collection (GC) Pauses:** If a Node.js process experiences a long V8 garbage collection pause while holding the lock, the lock TTL might expire in Redis. Once GC finishes, the Node app continues processing under the false assumption that it still holds the lock, while another process has already acquired it.

---

## 4. Job Processing & Queue Design

### Q1: How would you design a background job processing system?
*   **The Architecture:**
    ```
    [ Node.js API Web Servers ] -- (Enqueue Jobs) --> [ Redis / BullMQ / RabbitMQ ]
                                                               |
                                                       (Poll / Push Jobs)
                                                               v
                                                    [ Worker Node Pool ]
                                                    (Scaled Horizontally)
    ```
*   **Core Architectural Components:**
    1.  **Job Queue Broker:** Redis (using `BullMQ` or `Bee-Queue` for Node.js) or dedicated brokers (RabbitMQ, AWS SQS).
    2.  **Worker Nodes:** Stateless Node.js processes running independently of the web API, dedicated to consuming and executing jobs.
    3.  **State Management Database:** Keeps track of job states (`WAITING`, `ACTIVE`, `COMPLETED`, `FAILED`).
    4.  **Dead Letter Queue (DLQ):** A secondary queue where poison-pill jobs (jobs that fail repeatedly) are stored for manual debugging.

---

### Q2: Multiple workers are consuming jobs from a queue. How do you ensure a job is processed only once?
*   **Mechanisms:**
    1.  **Visibility Timeout / Message Locking:** When Worker A pulls a job, the queue broker locks the job (making it invisible to other workers) for a predefined lease duration (e.g., 5 minutes).
        *   If Worker A finishes within 5 minutes, it deletes/acknowledges the job.
        *   If Worker A crashes, the lease expires, the job becomes visible again, and Worker B can pick it up.
    2.  **Atomic State Transitions:** In DB-backed queues, workers update the job record using a conditional query:
        `UPDATE jobs SET status = 'PROCESSING', worker_id = 'worker_99' WHERE id = 123 AND status = 'PENDING';`
        Only the worker that modifies 1 row is allowed to process the job.

---

### Q3: How would you handle job retries?
*   **The Best Practices:**
    1.  **Exponential Backoff:** If a job fails due to network issues, do not retry immediately. Double the delay time for each subsequent retry (e.g., retry 1 in 2s, retry 2 in 4s, retry 3 in 8s).
    2.  **Jitter:** Add random noise (e.g., +/- 500ms) to the backoff delay to prevent "thundering herd" bottlenecks on downstream dependencies.
    3.  **Max Retry Limit:** Limit retries (typically 3 to 5 times).
    4.  **Move to Dead Letter Queue (DLQ):** Once the max retry limit is exceeded, automatically route the job to a DLQ and alert the engineering team.

---

### Q4: How would you prevent duplicate execution of scheduled jobs?
*   **The Problem:** If you run multiple instances of a Node.js process using a library like `node-cron`, *every* instance will trigger the cron job simultaneously at the scheduled time, resulting in duplicate work.
*   **Solutions:**
    1.  **Orchestrator-Level Cron (Recommended):** Run the cron task via Kubernetes `CronJob` or AWS EventBridge. The orchestrator triggers a single, short-lived container to execute the job.
    2.  **Distributed Lock (Time-based):**
        *   When the cron triggers on all instances, each tries to acquire a Redis lock: `lock:cron:send_weekly_emails` with a TTL slightly shorter than the cron interval (e.g., 50 seconds for a 1-minute cron).
        *   The single instance that acquires the lock runs the job; the others exit immediately.

---

### Q5: How would you process 1 million image-processing requests concurrently?
*   **High-Scale Image Processing Pipeline:**
    1.  **Ingestion:** Client uploads image directly to AWS S3 using a **Presigned URL** (bypassing the Node.js API server to save network bandwidth and CPU).
    2.  **Queue Event:** S3 triggers an event that writes a message containing the image metadata to an **AWS SQS** or **Apache Kafka** queue.
    3.  **Autoscaling Workers:** Scale worker instances dynamically based on queue size (using tools like Kubernetes KEDA or AWS ECS Autoscaling).
    4.  **Worker Optimization:**
        *   Inside the worker, use the `sharp` library (written in native C++ for extreme performance) to resize and compress images.
        *   Offload processing to **Worker Threads** to prevent blocking the worker's own event loop, allowing it to continue fetching new messages from the queue.
    5.  **Output:** Save processed images back to a separate S3 bucket and serve them via a CDN (CloudFront).

---

## 5. High Traffic & Scalability

### Q1: Design an API Gateway for 500 microservices.
*   **The Architecture:**
    ```
    Clients (Web/Mobile) ---> [ DNS / Cloudflare WAF ]
                                     |
                                     v
                           [ API Gateway Cluster ] (Envoy / Kong / Nginx)
                                     |
                     +---------------+---------------+
                     |               |               |
                     v               v               v
               [ Auth Service ]  [ Service A ]  [ Service B ]
    ```
*   **Key Responsibilities:**
    1.  **Request Routing / Reverse Proxying:** Dynamic path-based routing (e.g., `/api/v1/orders` -> Orders Service) using service registry configurations.
    2.  **Authentication & Authorization:** Offloads JWT validation or OAuth token checks.
    3.  **Rate Limiting & Throttling:** Distributed rate limiting at the edge using Redis.
    4.  **SSL Termination:** Terminates HTTPS at the gateway, sending plain HTTP to internal services over a secure private network.
    5.  **BFF (Backend-For-Frontend) Optimization:** Aggregates responses from multiple services into a single payload to reduce client roundtrips.

---

### Q2: Design a backend that handles 10 million API requests per day.
*   **The Math:** 10,000,000 requests / 86,400 seconds ≈ **116 Requests Per Second (RPS)** average. Assuming a peak-to-average ratio of 5, peak traffic is **~580 RPS**.
*   **Design Solution:**
    *   *Single-Instance Capability:* A single Node.js instance on an AWS `t3.medium` can handle 1,000+ basic RPS.
    *   *High Availability Architecture:* Use **2 stateless application servers** behind a Load Balancer (AWS ALB) across 2 Availability Zones.
    *   *Caching:* Use Redis for hot path queries.
    *   *Database:* Primary SQL DB with **1 Read Replica** to offload read queries.

---

### Q3: How would you handle sudden traffic spikes (100x normal load)?
*   **The Blueprint:**
    1.  **Auto-scaling:** Configure horizontal pod autoscaling (HPA) in Kubernetes or AWS Auto Scaling groups based on CPU/Memory and request count metrics.
    2.  **Edge Caching:** Cache static and semi-static API responses at the CDN level (Cloudflare).
    3.  **Queue-Based Buffering (Backpressure):** Ingest write-heavy transactions into Kafka/SQS, decoupling the web API from processing databases.
    4.  **Rate Limiting & Load Shedding:** Reject low-priority requests with a `429 Too Many Requests` code.
    5.  **Connection Pooling:** Use PgBouncer or similar connection pools to prevent database connection limits from crashing the service.

---

### Q4: Design a system that can scale from 1 server to 100 servers without code changes.
*   **Requirements:**
    1.  **Shared Statelessness:** Move session states, file uploads, and local caches out of the application memory. Store sessions in **Redis**, file uploads in **S3**, and cache on a shared cache cluster.
    2.  **Externalized Configuration:** Load system environment variables and secrets dynamically (e.g., HashiCorp Vault or Kubernetes ConfigMaps) instead of hardcoding.
    3.  **Centralized Observability:** Ship logs to an external service (ELK/Loki) instead of writing to local files.
    4.  **Service Discovery:** Use DNS-based routing (Kubernetes Services) rather than hardcoded server IP addresses.

---

### Q5: How would you perform load shedding during overload?
*   **The Mechanism:** Load shedding drops low-priority requests to protect core system paths when resources (CPU, RAM, or DB connection pools) are saturated.
*   **Implementation Steps:**
    1.  **Identify Critical Paths:** Checkout, login, and payments are high priority. Search recommendations, analytics events, and profile picture updates are low priority.
    2.  **Monitor Queue Lag / System Metrics:** Track Event Loop lag or CPU utilization.
    3.  **Drop & Reject:** If lag is >100ms, API Gateway immediately rejects low-priority endpoints with HTTP `503 Service Unavailable`, avoiding executing database operations.

---

### Q6: Design a multi-region backend architecture.
*   **The Blueprint:**
    ```
    User (US) ---> [ GeoDNS Routing ] ---> [ US-Region ALB ] ---> [ US Node.js App ] ---> [ US Postgres ]
                                                                                             ^
                                                                                    (Active-Active Sync)
                                                                                             v
    User (EU) ---> [ GeoDNS Routing ] ---> [ EU-Region ALB ] ---> [ EU Node.js App ] ---> [ EU Postgres ]
    ```
*   **Key Decisions:**
    *   **Routing:** GeoDNS (Amazon Route 53) routes users to the geographically closest region.
    *   **Database Synchronization:**
        *   *Active-Passive:* Write to US (Master). Read from EU (Replica). EU writes undergo a cross-region network hop.
        *   *Active-Active:* Use multi-master engines (CockroachDB or AWS Aurora Global DB). Resolve write conflicts using "Last-Write-Wins" or UUID column keys.
    *   **Data Residency:** Comply with local laws (e.g., GDPR) by pinning tenant data to specific physical servers in the EU.

---

### Q7: How would you implement graceful degradation when dependencies fail?
*   **The Blueprint:**
    *   **Recommendation Service Fails:** Fall back to a static list of generic best-sellers instead of throwing an API error page.
    *   **Search Engine Fails:** Fall back to basic SQL pattern matches (`LIKE %query%`) rather than failing search requests.
    *   **User Avatar Storage Fails:** Return a default placeholder avatar.
*   **Implementation:** Wrap outbound dependency calls in **Circuit Breakers** configured with fallback functions.

---

### Q8: Design a system to serve 1 million concurrent WebSocket connections.
*   **Optimization Strategies:**
    1.  **Stateless WS Connection Managers:** Deploy a fleet of connection servers. Put an Nginx/ALB load balancer in front utilizing IP Hash.
    2.  **Kernel Tuning:** Increase the maximum file descriptors limit (`ulimit -n 1048576`) on Linux. Tune TCP buffers to minimize memory per connection (e.g., `sysctl -w net.ipv4.tcp_rmem="4096 4096 16384"`).
    3.  **Memory Optimization:** Use bare-bones WebSocket packages (like `ws` in Node.js) instead of heavy libraries (Socket.io) to keep memory footprint under 15KB per connection (allowing ~15GB RAM per 1 million connections).
    4.  **Broadcasting Layer:** Connect all WebSocket servers via a Redis Pub/Sub backplane or Kafka to coordinate cross-server message deliveries.

---

### Q9: How would you distribute traffic across multiple Node.js instances?
*   **Solutions:**
    1.  **PM2 Cluster Mode:** Natively forks the Node.js process to match CPU cores, sharing the incoming port.
    2.  **Nginx Load Balancer:** Configured as a reverse proxy, distributing requests to a pool of backend process ports using Round Robin or Least Connections algorithms.
    3.  **Container Routing (Kubernetes):** Let the Kubernetes Service load balancer distribute requests to Pod IPs.

---

### Q10: Design a zero-downtime deployment strategy.
*   **Three Classic Strategies:**
    1.  **Rolling Updates:** Gradually replaces old instances with new ones (e.g., replace 25% at a time). Requires backward-compatible API code.
    2.  **Blue-Green Deployment:** Spin up a complete duplicate environment (Green) running the new version. Perform tests, then switch the Load Balancer DNS target from old (Blue) to new (Green) instantly.
    3.  **Canary Releases:** Route 5% of user traffic to the new version. If error rates remain flat, slowly scale canary traffic to 100%.

---

## 6. Caching Strategies & Patterns

### Q1: Design a caching layer for an e-commerce platform.
*   **The Strategy:**
    *   **Static Pages / Images:** Cache at the CDN (Cloudflare).
    *   **Product Details / Catalog:** Cache in Redis using **Cache-Aside** pattern. Invalidate the cache when product updates occur via the admin panel.
    *   **Shopping Cart:** Store in Redis using **Write-Through** pattern. Session data is read and written frequently.
    *   **User Sessions:** In-memory Redis store with TTL matching the session expiry (e.g., 30 days).

---

### Q2: How would you prevent cache stampede?
*   **The Danger:** A popular cache key expires. 10,000 concurrent requests miss the cache simultaneously, and all query the database, overloading the DB.
*   **Solutions:**
    1.  **Mutex Locking (Single Flight / SET NX):** The first request that misses the cache acquires a Redis lock (`lock:product_123`) to query the database and update the cache. All other requests wait/retry, reading from the newly populated cache.
    2.  **Probabilistic Early Expiration (XFetch):** Background workers recalculate the cache item *before* it actually expires, using a probability algorithm based on remaining TTL and query frequency.

---

### Q3: How would you prevent cache penetration?
*   **The Danger:** A client requests non-existent keys (e.g., `id = -999` or randomly generated UUIDs) repeatedly. The requests miss the cache and hit the database every time.
*   **Solutions:**
    1.  **Cache Null Values:** If the DB returns "not found", cache a null or empty value with a short TTL (e.g., 5 minutes) to protect the DB.
    2.  **Bloom Filter:** Implement a Bloom Filter (a space-efficient probabilistic data structure) in Redis. Before querying the cache or database, check if the ID exists in the Bloom Filter. If not, reject the request immediately.

---

### Q4: How would you prevent cache avalanche?
*   **The Danger:** A database reboot or bulk cache warming schedules all cache keys to expire at the exact same time (e.g., midnight). The DB is suddenly crushed by requests.
*   **The Solution:** Add **Random Jitter** to the TTL of every cached item (e.g., standard TTL of 24 hours + a random variance of 0 to 60 minutes). This spreads out expiration times evenly.

---

### Q5: Design a distributed cache invalidation strategy.
*   **Strategies:**
    1.  **Active Invalidation via Pub/Sub:** When a node writes an update to the database, it publishes an invalidation event over Redis Pub/Sub. All other application nodes consume the event and purge their local memory caches.
    2.  **Change Data Capture (CDC):** Let a tool like Debezium monitor the database Transaction Log and write changes to Kafka. An invalidator consumer reads Kafka and invalidates cache entries.

---

### Q6: How would you cache user profiles updated frequently?
*   **The Strategy:** Use a **Write-Through** or **Write-Behind** cache pattern.
    *   *Write-Through:* Write update to cache and database in a single transaction block. Fast reads, but slow write latency.
    *   *Short TTL Cache-Aside:* Update DB directly and delete the cache key. Next read triggers a refresh. Keep the TTL short (e.g., 1 hour) to auto-expire any missed invalidations.

---

### Q7: Explain the Cache-Aside architecture.
*   **Read Path:**
    1. Application checks the cache for key $K$.
    2. If key exists (Cache Hit), return data.
    3. If key does not exist (Cache Miss), query Database.
    4. Write data to cache with a TTL, then return it.
*   **Write Path:**
    1. Update Database.
    2. Delete/invalidate key $K$ in Cache.

---

### Q8: How would you cache expensive database queries?
*   **The Solution:** Use **Query Results Caching** in Redis:
    1. Hash the SQL query string (e.g. `MD5("SELECT * FROM reports WHERE region = 'US' AND status = 'ACTIVE';")`) to create a unique cache key (`cache:query:hash`).
    2. Retrieve key from Redis. On miss, run query in DB, store JSON output in Redis with a reasonable TTL (e.g., 1 hour), and return.
    3. Pre-warm cache for critical slow queries via a background cron worker.

---

### Q9: Design a session cache system.
*   **The Solution:** Use Redis clusters:
    1. Store session data as a Redis Hash: `session:token_uuid` with fields `user_id`, `created_at`, `device_info`.
    2. Enforce a session expiration time by setting a TTL on the hash key (e.g., `EXPIRE session:token_uuid 2592000` for 30 days).
    3. On every active request, issue a sliding expiration update by calling `EXPIRE` again to extend the session lifespan.

---

### Q10: How would you handle stale cache data?
*   **Strategies:**
    1.  **Time-To-Live (TTL):** Hard time limits that guarantee data auto-expires eventually.
    2.  **Active Invalidation on Update:** Delete/invalidate cache key immediately during DB update transaction.
    3.  **Cache Busting (Versioning):** Prefix cache keys with a schema or database version: `v2:product_123`. Changing the version prefix instantly invalidates all old keys.

---

## 7. API Design & Protocols

### Q1: Design a public API platform used by third-party developers.
*   **Key Requirements:**
    1.  **Authentication:** Issue secure API Keys or client credentials via OAuth2.
    2.  **Rate Limiting:** Protect servers by enforcing tier-based rate limits (e.g., Developer: 60 req/min, Enterprise: 10,000 req/min).
    3.  **Versioning:** Enforce versioning in the URL path (`/v1/resources`).
    4.  **Developer Portal:** Provide clear OpenAPI (Swagger) documentation, sandboxes, and interactive SDKs.
    5.  **Analytics:** Capture usage metrics to monitor client activity and billing.

---

### Q2: Design API versioning for a large-scale application.
*   **Three Classic Approaches:**
    1.  **Path-Based (Recommended):** `GET /api/v1/users`. Highly cacheable, explicit, and easy to route at the API Gateway.
    2.  **Header-Based (Accept Header):** `Accept: application/vnd.company.v1+json`. Cleaner URLs, but hard to test in browsers and breaks standard CDN caching.
    3.  **Query Parameter:** `GET /api/users?version=1`.

---

### Q3: How would you handle backward compatibility?
*   **The Protocol:**
    1.  **Additions are Non-Breaking:** Adding optional query parameters or new fields to JSON responses does not break correctly written clients.
    2.  **Enforce Schema Constraints:** Use contract testing (e.g., Pact) to verify that API modifications do not break client expectations.
    3.  **Deprecation Policies:** If a breaking change is mandatory, mark the route with headers: `Sunset: Wed, 11 Nov 2026 00:00:00 GMT` and `Deprecation: true`, and support the deprecated route for a transition window (e.g., 6 months).

---

### Q4: Design an API request validation framework.
*   **The Blueprint:** Use schema validation middleware (e.g. **Zod**, **Joi**, or **AJV** JSON Schema validator) at the route layer.
    *   Requests are validated *before* hitting controllers, checking types, required fields, and string constraints (emails, phone numbers).
    *   Failures trigger an immediate `400 Bad Request` with structured error messages (e.g., field name and violation type), protecting business logic from dirty input.

---

### Q5: How would you design a bulk-processing API?
*   **The Pattern:** Make it **Asynchronous**.
    1.  Client sends `POST /api/v1/bulk-import` with an array of objects.
    2.  The server validates the payload envelope, generates a `job_id`, enqueues the job into a Message Queue (Kafka/RabbitMQ), and immediately returns `202 Accepted` with payload `{ "job_id": "xyz", "status": "queued" }`.
    3.  Background workers consume the queue and process jobs in batches.
    4.  The client polls `GET /api/v1/jobs/xyz` or registers a `webhook_url` to receive a callback on completion.

---

### Q6: Design a file-upload API supporting large files.
*   **The Scalable Design:**
    ```
    1. Client ---> [ App Server ] (Requests upload authorization)
    2. Client <--- [ App Server ] (Returns S3 Presigned Multipart URLs)
    3. Client ===> [ AWS S3 Storage ] (Uploads chunked parts directly to S3)
    4. Client ---> [ App Server ] (Reports upload completion)
    ```
*   *Why this pattern?* Bypasses the application servers for file transfer, saving network bandwidth and CPU overhead, and eliminating server storage allocation problems.

---

### Q7: Design an API throttling mechanism.
*   **The Blueprint:** Implement a **Token Bucket** or **Sliding Window Counter** rate limiter in Redis.
    *   Every incoming request invokes a Redis check using the client IP or User ID.
    *   Exceeded limits return `429 Too Many Requests` along with standard headers:
        *   `X-RateLimit-Limit`: Maximum requests allowed in the window.
        *   `X-RateLimit-Remaining`: Remaining request quota.
        *   `Retry-After`: Time in seconds to wait before trying again.

---

### Q8: How would you handle partial failures in APIs?
*   **The Solution:** Use HTTP **207 Multi-Status** responses.
    *   When processing bulk requests (e.g., updating 5 records), return status 207 with a payload detailing success or failure for each individual item:
        ```json
        {
          "results": [
            { "id": 1, "status": 200, "message": "Updated" },
            { "id": 2, "status": 400, "message": "Invalid email" }
          ]
        }
        ```

---

### Q9: Design an asynchronous API for long-running operations.
*   **The Blueprint:**
    1. Client posts task -> Server returns `202 Accepted` with a `Location: /api/v1/tasks/uuid` header.
    2. Client polls the URL. While processing, server returns `200 OK` with `{ "status": "PROCESSING", "progress": "50%" }`.
    3. When finished, the task endpoint returns `{ "status": "COMPLETED", "result_url": "/api/v1/reports/pdf-99" }`.

---

### Q10: How would you implement request tracing across services?
*   **The Solution:** Use **Distributed Tracing** context propagation.
    1. The API Gateway injects a unique correlation ID header (e.g. `X-Correlation-ID` or W3C standard `traceparent`) to every incoming request.
    2. Every downstream service intercepts this header and passes it to any outgoing HTTP/gRPC call, database query, or queue write.
    3. Application loggers output the trace ID in every log statement, enabling developers to query an entire request flow across dozens of microservices in a log aggregator (Elastic/Jaeger).

---

## 8. Authentication & Authorization

### Q1: Design a Single Sign-On (SSO) system.
*   **The Blueprint:** Use **OpenID Connect (OIDC)** and **OAuth 2.0**.
    1. User visits Application A (Service Provider / SP).
    2. Application A redirects the user to the Central Auth Server (Identity Provider / IdP).
    3. User logs in at IdP. IdP generates an Auth Code and redirects back to Application A with the code.
    4. Application A exchanges the Auth Code for an ID Token & Access Token via a secure backend-to-backend API call to the IdP.
    5. User visits Application B. Application B redirects to IdP. The IdP detects the active user session cookie and immediately redirects back to Application B with an Auth Code (no login form shown).

---

### Q2: Design a JWT-based authentication system.
*   **The Blueprint:** Stateless token authentication.
    *   **Structure:** Composed of three parts: Header, Payload (contains claims like `user_id`, `roles`, `exp`), and Signature.
    *   **Signing:** Use asymmetric algorithms (**RS256**). The Auth Server signs the JWT using its Private Key. Downstream microservices verify the signature using the Auth Server's public key (fetched from a JWKS endpoint), requiring no database lookups or inter-service API calls.

---

### Q3: Design a refresh-token mechanism.
*   **The Blueprint:**
    1.  **Access Token:** Short-lived (e.g., 15 minutes), stateless JWT containing user ID and permissions. Passed in the `Authorization: Bearer <token>` header.
    2.  **Refresh Token:** Long-lived (e.g., 30 days), stored in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie to prevent XSS attacks. Store the hash of this token in a database.
    3.  **Token Rotation:** Every time the client requests a new Access Token, the server issues a new Refresh Token and invalidates the old one (preventing replay attacks).

---

### Q4: How would you revoke JWT tokens?
*   **The Problem:** Since JWTs are stateless, they remain valid until their expiration date, even if a user resets their password or is blocked.
*   **Solutions:**
    1.  **Redis Blacklist (Token Revocation List):** Store revoked token IDs (jti) in Redis with a TTL matching the token's remaining lifespan. During auth check, verify the token is not blacklisted.
    2.  **User Version Count:** Add an integer `token_version` column to the `users` table. Embed this value in the JWT payload. When a password resets, increment `token_version`. During validation, verify the JWT version matches the DB version.

---

### Q5: Design Role-Based Access Control (RBAC).
*   **The Solution:** Assign users to Roles, and Roles to Permissions:
    *   **Database Schema:**
        *   `users (id, name)`
        *   `roles (id, name)` (e.g., 'Admin', 'Editor')
        *   `permissions (id, code)` (e.g., 'write_post', 'delete_user')
        *   `role_permissions (role_id, permission_id)`
        *   `user_roles (user_id, role_id)`
    *   During authentication, load permissions into user session/JWT, checked via route middleware.

---

### Q6: Design Attribute-Based Access Control (ABAC).
*   **The Solution:** Fine-grained authorization utilizing conditional policy evaluations:
    *   Permissions are evaluated dynamically on request.
    *   Example logic: `allowUpdate = user.id === resource.owner_id || (user.role === 'Editor' && resource.status === 'draft')`.

---

### Q7: Design secure API authentication between microservices.
*   **Solutions:**
    1.  **mTLS (Mutual TLS):** Forces both client and server microservices to present and validate each other's X.509 certificates before establishing a secure TCP connection (standard in Service Meshes like Istio).
    2.  **Client Credentials Grant (OAuth2):** Microservices fetch short-lived machine-to-machine JWT access tokens from a central authorization server (e.g., Auth0 or Okta) and pass them in HTTP headers.

---

### Q8: How would you protect APIs from replay attacks?
*   **Solutions:**
    1.  **Request Signature with Timestamp:** Client signs the request payload including a timestamp. The server rejects any request with a timestamp older than 5 minutes.
    2.  **Nonce Tracking:** Client sends a unique random number (Nonce) along with the request signature. The server caches this Nonce in Redis for 5 minutes. If a duplicate Nonce is detected within the window, the request is rejected as a replay attempt.

---

### Q9: Design an OTP verification system.
*   **The Blueprint:**
    1. Client requests OTP -> Server generates a random 6-digit number, hashes it, and saves it in Redis: `SET otp:user_id hashed_otp EX 300` (5-minute expiry).
    2. Dispatch OTP to the user via SMS (Twilio) or Email.
    3. Verification: Hash the user's input OTP and compare it against the value in Redis. Delete key on success. Track failed attempts in Redis; lock account for 24 hours if failures > 3.

---

### Q10: Design a password reset system.
*   **The Blueprint:**
    1. Client submits email. Server verifies email exists, generates a secure random token (UUIDv4), hashes it, and saves it in DB: `password_resets (email, token_hash, expires_at)`.
    2. Send a reset link containing the raw token to the user's email.
    3. On click, verify the token is not expired, let the user enter a new password, hash the password, update `users` table, and delete the database reset token. Increment `token_version` to invalidate all active JWT sessions.

---

## 9. Real-Time Systems

### Q1: Design a real-time notification system.
*   **The Blueprint:**
    ```
    API Clients ---> [ Event Ingestion Queue ] (SQS / Kafka)
                                |
                      (Event Router Worker)
                                |
                                v
                   [ Redis Pub/Sub Router ]
                                |
             +------------------+------------------+
             v                                     v
    [ WS Server A ] (active user connections)   [ WS Server B ]
  ```
*   **Mechanism:**
    1.  API writes a notification task to SQS.
    2.  The Event Router reads SQS and publishes the notification message to Redis Pub/Sub channel matching the recipient's user ID.
    3.  The WebSocket server instance hosting the recipient's active connection consumes the Redis Pub/Sub message and writes the payload directly to the open WebSocket descriptor.

---

### Q2: Design a real-time collaborative document editor.
*   **The Solutions:**
    1.  **Operational Transformation (OT):** Used by Google Docs. The server acts as a central sequencer. It resolves conflict events by rewriting character index operations (e.g., inserts/deletes) mathematically so all clients converge on the same document state.
    2.  **Conflict-Free Replicated Data Types (CRDTs):** Used by Figma. Merges operations natively without requiring a centralized sequencer, utilizing mathematically commutative data structures.

---

### Q3: Design a live sports score update system.
*   **The Blueprint:**
    *   Use **Server-Sent Events (SSE)** rather than WebSockets. Since data flows exclusively one-way (server to client), SSE has lower resource overhead and runs natively over standard HTTP.
    *   Score updates are published to a Redis Pub/Sub channel. The backend SSE controllers subscribe to Redis and flush events to active HTTP client connections.
    *   Cache the scores at the CDN edge with a 1-second TTL to offload traffic during peak periods.

---

### Q4: Design a stock price streaming service.
*   **The Blueprint:**
    *   **Data Ingestion:** gRPC stream from stock exchanges into a high-throughput **Apache Kafka** cluster. Partition the Kafka topic by ticker symbol.
    *   **Broadcast Engine:** WebSocket servers consume Kafka partitions and distribute the prices to clients using a pub/sub manager. Use a local memory ring buffer to batch and debounce price updates (e.g., send updates at most 10 times per second per user to prevent client browser rendering lag).

---

### Q5: Design a multiplayer game backend.
*   **The Blueprint:**
    *   **Networking:** Use **UDP** or **WebRTC data channels** instead of TCP/WebSockets to avoid latency overhead caused by head-of-line blocking.
    *   **Authoritative Server Model:** The server runs the game physics simulation loop (e.g., 30 ticks per second). Clients send raw inputs (e.g., "move left"), and the server broadcasts verified entity states.
    *   **Client Optimization:** Implement client-side prediction, entity interpolation, and lag compensation.

---

### Q6: Design a typing indicator feature for chat applications.
*   **The Blueprint:**
    1.  **Client Trigger:** When a user types in the input box, throttle events so they are sent via WebSocket at most once every 3 seconds:
        `ws.send({ type: "typing", room_id: 101, user_id: 99 })`.
    2.  **Routing Server:** The node server receives the event and broadcasts it over Redis Pub/Sub to all other servers hosting users in room 101.
    3.  **Client Render & Timeout:** The receiver's client displays the typing bubble and resets a local 5-second timer. If no new typing event arrives before the timer expires, the bubble is hidden.

---

### Q7: Design a presence system showing online/offline users.
*   **The Blueprint:**
    1.  **Heartbeat Loop:** Client sends a lightweight ping to the server via WebSocket or HTTP POST every 30 seconds.
    2.  **Redis State Storage:** The server updates a Redis key with a 40-second TTL:
        `SET active:user_99 1 EX 40`.
    3.  **Status Check:** To check if a user is online, check `EXISTS active:user_99`.
    4.  **Pub/Sub Notification:** On status transitions (online -> offline due to TTL expiration), trigger an event to notify friends. Use Redis keyspace notifications to detect expirations.

---

### Q8: Design a real-time dashboard for monitoring metrics.
*   **The Blueprint:**
    1.  **Collection:** Push-based metrics collector (Telegraf/StatsD) running on nodes.
    2.  **Storage:** Store time-series data in a database optimized for range aggregations (e.g., **Prometheus** or **InfluxDB**).
    3.  **Display:** Grafana querying the TSDB dynamically with auto-refresh dashboards.

---

### Q9: Design a pub/sub messaging system.
*   **The Choices:**
    *   *Transient (In-Memory):* Redis Pub/Sub. Messages are fire-and-forget; if a subscriber is offline, they lose the message.
    *   *Durable:* **Apache Kafka** or **NATS JetStream**. Messages are written to append-only disk logs, allowing consumers to replay messages from any offset.

---

### Q10: Design a live auction platform.
*   **The Blueprint:**
    *   **Bid Collision Resolution:** Run `WATCH` or a Lua script in Redis to ensure bids are incremented atomically.
    *   **Low Latency Broadcast:** Push new high-bid updates via WebSockets immediately to all auction viewers.

---

## 10. Microservices Architecture

### Q1: Design a microservices architecture for an e-commerce application.
*   **The Blueprint:**
    *   Split system into autonomous domains: User/Auth Service, Product Catalog Service, Inventory Service, Order Service, and Payment Service.
    *   Every service has its own dedicated database (Database-per-Service pattern) to prevent database lock couplings. Communication occurs via REST/gRPC (synchronous) or Kafka/RabbitMQ (asynchronous).

---

### Q2: How would you handle service discovery?
*   **The Solution:** Use **Service Discovery Registry** engines (like HashiCorp Consul or Kubernetes CoreDNS).
    *   When Service A wants to call Service B, it queries the registry using B's logical name (e.g. `http://payment-service/charge`).
    *   The registry resolves the name to a list of healthy, dynamic IP addresses of running Service B instances.

---

### Q3: gRPC vs. REST for Microservices
*   *gRPC:* Low latency, binary payload (Protobuf), multiplexing, strict schema, best for internal service-to-service communication.
*   *REST:* Wide compatibility, JSON payload, text-based, standard routing, best for edge public APIs.

---

### Q4: Design a service registry.
*   **The Design:** A highly available, strongly consistent key-value store (e.g. Raft-backed Consul or etcd).
    *   Active service nodes register their IP, port, and health check routes on startup.
    *   If a node fails its health check for 3 consecutive intervals, the registry deletes the node from its routing table.

---

### Q5: Design a centralized configuration management system.
*   **The Design:** Use a centralized configuration service (like Spring Cloud Config or Consul KV).
    *   Store configurations in a git repo. The Config Server pulls updates, encrypts secrets, and serves them to microservices on startup.
    *   For dynamic updates, the Config Server publishes a webhook refresh event to a message broker (e.g. Kafka), prompting microservices to reload properties in memory without rebooting.

---

### Q6: How would you implement distributed tracing?
*   **The Solution:** Install **OpenTelemetry** SDKs in each microservice.
    *   Inject trace headers at the edge.
    *   Ship trace span telemetry records (containing execution timings and trace/span IDs) asynchronously to a collector agent (e.g., Jaeger or Zipkin collector) to prevent latency overhead.

---

### Q7: Explain the Saga Pattern workflow.
*   **The Problem:** Microservices have separate databases. A multi-step transaction (e.g., Book Flight -> Book Hotel -> Charge Card) can fail at step 3, leaving database states inconsistent.
*   **The Solution:** A Saga is a sequence of local transactions. Each transaction updates database state and triggers the next step. If a step fails, the Saga runs **compensating transactions** in reverse order to undo the changes.
*   **Two Approaches:**
    1.  **Choreography:** Services listen to events and trigger their operations asynchronously without a central controller.
    2.  **Orchestrator (Recommended):** A central coordinator service tells each participant service which transaction to run. Easier to monitor and debug.

---

### Q8: Design an event-driven microservice architecture.
*   **The Design:**
    *   Microservices mutate local data, and emit events (e.g., `OrderCreated`) to a message log (**Apache Kafka**).
    *   Other microservices consume these events to execute downstream logic (e.g., Billing Service listens to `OrderCreated` and processes the credit card). Eventual consistency is maintained.

---

### Q9: How would you manage shared data across microservices?
*   **Solutions:**
    1.  **API Composition:** The caller queries Service A and Service B separately and merges the data in memory. (High latency overhead).
    2.  **Event-Driven Replication:** Service A publishes data change events. Service B subscribes and maintains a read-only replicated copy of the shared entity in its local database, optimized for local queries.

---

## 11. Reliability & Fault Tolerance

### Q1: Design a Circuit Breaker mechanism.
*   **The States:**
    *   **Closed:** Normal operation. Requests flow to the dependency. If failure rate exceeds a threshold (e.g., 50% errors over 10 seconds), trip the circuit to **Open**.
    *   **Open:** Fast-fail state. Requests fail immediately or return fallback data without hitting the downstream server. Start a timer.
    *   **Half-Open:** When the timer expires, allow a limited number of test requests through. If they succeed, close the circuit. If they fail, return to the Open state.
*   **Node.js Implementation:** Use the `opossum` library.

---

### Q2: How would you implement exponential backoff with jitter?
*   **The Formula:**
    $$T_{\text{wait}} = \min(T_{\text{max\_delay}}, 2^{\text{attempt}} \times T_{\text{base\_delay}} + \text{random\_jitter})$$
*   *Why Jitter?* If 1,000 requests fail, executing retries at identical intervals causes a "thundering herd" spike on downstream systems. Adding random variance (jitter) spreads out request rates evenly.

---

### Q3: Liveness vs. Readiness Probes
*   **Liveness Probe:** Checks if the container process has crashed or entered a deadlocked state. If it fails, the orchestrator (Kubernetes) kills and restarts the container.
*   **Readiness Probe:** Checks if the container is ready to accept incoming traffic (e.g. database connections initialized, caches warmed). If it fails, the load balancer stops routing traffic to this instance.

---

### Q4: How would you detect and recover from service crashes?
*   **The Solution:**
    *   **Process Manager (PM2 / systemd):** Automatically restarts the Node.js process if it exits due to an unhandled exception.
    *   **Container Orchestrator (Kubernetes ReplicaSets):** Restarts the container instance if it crashes, routing around it until it passes readiness probes.
    *   **Diagnostics:** Listen to `process.on('uncaughtException')` and `process.on('unhandledRejection')` to log errors before cleanly exiting the process (avoiding leaving sockets open).

---

### Q5: RPO vs. RTO Disaster Recovery
*   **RPO (Recovery Point Objective):** The maximum tolerable age of data that can be lost after a disaster (e.g., RPO = 1 hour means you must perform backups at least hourly).
*   **RTO (Recovery Time Objective):** The maximum tolerable duration of downtime before the system is restored online (e.g., RTO = 5 minutes means failovers must be automated).

---

## 12. Data Processing Pipelines

### Q1: Design a CSV import system for millions of records.
*   **The Architecture:**
    1.  **Stream Ingestion:** Do **not** read the entire file into memory (which crashes the server). Use a chunked stream processor (e.g., Node.js streams or `readline`).
    2.  **Batch Ingestion:** Group read rows into batches (e.g., 1,000 records) and write them to a database using bulk inserts.
    3.  **Worker Delegation:** Offload validation and formatting to a **worker thread pool** to prevent blocking the event loop.

---

### Q2: Design a Report Generation System.
*   **The Architecture:**
    1. Client submits request -> Web API pushes a task to **RabbitMQ** and returns job ID.
    2. A pool of worker instances consumes the queue. The worker queries a **read-replica database** to prevent resource contention on the primary transactional DB.
    3. The worker generates the report (PDF/CSV), uploads the artifact to **Amazon S3**, updates the job state in DB, and fires a push notification to the user.

---

### Q3: Design a Log Aggregation Service.
*   **The Architecture (ELK / PLG Stack):**
    *   *Collector:* A lightweight daemon (Fluentd/Promtail) running on application hosts reads log files and ships them.
    *   *Buffer:* **Apache Kafka** absorbs traffic spikes.
    *   *Storage & Index:* Elasticsearch or Grafana Loki.
    *   *Visualization:* Kibana or Grafana.

---

### Q4: Design an Analytics Event Collection Pipeline.
*   **The Architecture:**
    `Client Event ---> [ API Gateway Ingester ] ---> [ Kafka Buffer ] ---> [ ClickHouse Columnar DB ]`
    *   *Why ClickHouse?* Columnar databases are highly optimized for fast aggregation queries over billions of rows.

---

### Q5: Design a Webhook Processing Platform.
*   **The Architecture:**
    ```
    Provider HTTP POST ---> [ Ingestion API Gateway ] 
                                  |
                        (Validate Signature)
                                  |
                                  v
                          [ Message Queue ] (Kafka / SQS)
                                  |
                        (Consumer Processing)
                                  |
                                  v
                           [ Worker Nodes ] ---> (Downstream API execution)
    ```
*   **Requirements:**
    *   **Idempotency:** Track webhook IDs in a Redis cache to prevent duplicate processing.
    *   **Retries with Backoff:** If the downstream target returns a `5xx` error, retry up to 5 times using exponential backoff.

---

## 13. Observability & Monitoring

### Q1: How would you monitor Node.js performance in production?
*   **Key Metrics to Monitor:**
    1.  **Event Loop Lag:** Track loop delays. Lag > 50ms indicates blocking synchronous code.
    2.  **Event Loop Utilization (ELU):** Indicates how saturated the JS execution thread is.
    3.  **Garbage Collection Metrics:** Monitor frequency and duration of GC cycles (especially Full Mark-Sweep phases).
    4.  **Active Handles:** Number of open sockets, files, and server connections.

---

### Q2: How would you detect memory leaks in Node.js?
*   **Detection Strategy:**
    1.  **Heap Snapshots:** Take heap snapshots at regular intervals using the chrome-inspector flags or the `heapdump` module. Compare snapshots in Chrome DevTools to locate accumulating objects.
    2.  **Analyze Memory Usage Trends:** Track RSS (Resident Set Size) and heapUsed graphs in APMs. If memory usage climbs linearly and never drops after garbage collection, a leak is present.

---

## 14. Multi-Tenant SaaS Systems

### Q1: How would you isolate tenant data?
*   **Three Models:**
    1.  **Database-per-Tenant:** Separate physical database instances. High security and easy backups, but high cloud infrastructure cost.
    2.  **Schema-per-Tenant:** Single database with isolated internal schemas (e.g., Postgres namespaces). Good balance between isolation and cost.
    3.  **Shared-Database-Shared-Schema (Row-level Isolation):** Single database, single schema. Every table contains a `tenant_id` column. Queries must always filter: `WHERE tenant_id = :tenantId`. Implement via database row-level security (RLS) to prevent accidental data leaks.

---

### Q2: Design billing and usage tracking for tenants.
*   **The Architecture:**
    1. Every tenant action publishes a usage event (e.g., `api_calls: 1`) to Kafka.
    2. An event aggregator worker processes the Kafka stream, calculating hourly totals, and writes them to a time-series DB.
    3. A billing worker runs periodically, sums the usage, and pushes the billing data to Stripe Billing using webhooks.

---

## 15. Classic System Design Solutions

### Q1: Design a URL Shortener.
*   **System Components:**
    1.  **Hashing / ID Generation:** Use a distributed counter (e.g., ZooKeeper or Redis INCR) to get a unique auto-incrementing integer, and convert it to **Base62** string (characters `a-z`, `A-Z`, `0-9`). This generates short strings (e.g., ID 56.8 billion becomes `abc123`).
    2.  **Database Schema:** A simple relational table `urls (id, short_code, original_url, created_at)`.
    3.  **Caching:** Place a Redis cache in front of the database. The most active short links are cached, avoiding database queries entirely.
    4.  **Redirect Codes:** Use **HTTP 302 Found** (temporary redirect) if you want to track analytics metrics for every request. Use **HTTP 301 Moved Permanently** to offload redirect traffic to the client's browser cache.

---

### Q2: Design a WhatsApp-like messaging system.
*   **System Components:**
    ```
    Sender ---> [ WS Connection Manager ] ---> [ Chat Gateway ] ---> [ MQ / Kafka ]
                                                    |
                                          (Durable Inbox DB)
                                                    v
                                         [ Cassandra Message Store ]
                                                    |
                                          (Check Receiver Status)
                                                    v
    Receiver <--- [ WS Connection Manager ] <--- [ Pub/Sub ]
    ```
*   **Key Design Aspects:**
    *   **WebSocket Connections:** Maintain persistent connections across a fleet of stateless connection manager servers.
    *   **Message Delivery States:** Message states are stored as: `SENT` -> `DELIVERED` (notified by recipient check) -> `READ`.
    *   **Message Storage:** Cassandra or HBase partitioned by `conversation_id` for fast sequential reads.

---

### Q3: Design a LinkedIn feed generation service.
*   **The Blueprint:**
    *   **Feed Caching:** Redis sorted sets containing post IDs for each user, sorted by timestamp.
    *   **Fan-out-on-Write (Push):** When a user publishes a post, the system pushes the post ID to the feed caches of all their followers. Best for low-follow count users.
    *   **Fan-out-on-Read (Pull):** When a user views their feed, the system queries the posts of all their followed connections (especially high-follower counts/celebrities) and merges them on-the-fly.

---

### Q4: Design a YouTube video processing backend.
*   **The Pipeline:**
    1.  **Ingestion:** Client uploads video chunks directly to S3.
    2.  **Transcoding Queue:** S3 triggers an event that enqueues jobs into a message queue.
    3.  **Transcoder Workers:** Workers download the raw file, use FFmpeg to transcode it into multiple formats (MP4, WebM) and resolutions (1080p, 720p, 480p), and slice them into short ABR chunks.
    4.  **Storage & Delivery:** Save chunks back to S3 and distribute globally via CDN edge servers (CloudFront).

---

### Q5: Design a payment gateway aggregator.
*   **System Components:**
    *   **Adapter Pattern:** Standardized interface to interact with multiple payment gateways (Stripe, PayPal, Adyen).
    *   **Dynamic Routing:** Automatically routes transactions to the cheapest or highest-performing gateway based on real-time success rates.
    *   **Reconciliation Engine:** A daily cron job that aggregates transaction records from gateways and matches them against the internal database to find discrepancies.

---

### Q6: Design an Uber-like ride matching service.
*   **System Components:**
    1.  **Geospatial Indexing:** Divide the map into cells using **Uber H3** or **Google S2**.
    2.  **Location Tracking:** Drivers stream their GPS coordinates via WebSockets once every 4 seconds. The system updates their location index in Redis Geospatial sets (`GEOADD`).
    3.  **Matching Engine:** When a rider requests a pickup, query Redis to find the nearest online drivers (`GEORADIUS`). Send pickup requests sequentially using a matching algorithm.

---

### Q7: Design an Amazon order processing system.
*   **System Components:**
    1.  **Inventory Allocation:** Uses Redis Lua script decrements during checkout requests to reserve products immediately.
    2.  **Payment Gateway:** Call Stripe with an idempotency key.
    3.  **Transactional Database:** Relational DB (Postgres) storing orders, items, and payments. Enforce foreign keys.
    4.  **Fulfillment Worker Queue:** On payment success, publish an event to SQS. The warehouse fulfillment service picks it up to schedule packaging and delivery.

---

### Q8: Design a Food Delivery Backend (e.g., DoorDash).
*   **System Components:**
    1.  **Order Pipeline:** Customer places order -> Restaurant receives push event (via SSE or WebSocket) -> Restaurant accepts and starts preparing.
    2.  **Driver Matching:** Dispatch system queries Redis Geo for active drivers within a 3-mile radius of the restaurant. Match a driver using a routing engine.
    3.  **Real-Time Tracking:** Driver streams coordinates via WebSocket. The matching engine broadcasts coordinates to the customer's open socket.

---

### Q9: Design a Cloud File Storage Service (S3/Dropbox).
*   **System Components:**
    1.  **Metadata Database:** Relational database storing file names, file paths, versions, and permissions.
    2.  **Block Storage Engine:** Large files are sliced into smaller chunks (e.g., 4MB blocks), hashed (SHA-256), and stored in an object storage container.
    3.  **Deduplication:** Before saving a block, check if a block with the same hash already exists to save storage space.

---

### Q10: Design a distributed cron scheduler.
*   **System Components:**
    1.  **Central Scheduler Store:** DB storing job records, cron expressions, and execution states.
    2.  **Leader Election:** A cluster of scheduler managers elects a leader using Consul/Raft. Only the leader schedules and dispatches tasks.
    3.  **Execution Queue:** The leader writes executable jobs to a RabbitMQ/Kafka queue.
    4.  **Workers:** Stateless worker processes consume and execute the jobs, reporting execution results back to the database.
