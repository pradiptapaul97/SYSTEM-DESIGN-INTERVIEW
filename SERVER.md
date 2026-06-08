# Node.js Concurrency & Distributed Systems: Interview Prep Guide

This guide compiles high-frequency system design and coding interview questions focusing on **Node.js Concurrency**, **Race Conditions**, **Distributed Systems**, **Background Job Processing**, **API Design**, **Scaling**, and **Real-World Scenarios**. 

---

## 📌 Table of Contents
1. [Basic Node.js Concurrency](#1-basic-nodejs-concurrency)
2. [Race Conditions & Data Integrity](#2-race-conditions--data-integrity)
3. [Distributed Systems & Locking](#3-distributed-systems--locking)
4. [Job Processing & Queue Design](#4-job-processing--queue-design)
5. [API Design & Idempotency](#5-api-design--idempotency)
6. [Scaling High-Concurrency Services](#6-scaling-high-concurrency-services)
7. [Real-World System Design Scenarios](#7-real-world-system-design-scenarios)

---

## 1. Basic Node.js Concurrency

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
*   **Interview Tip:** Always explain the impact: a blocked Event Loop causes high latency, timeout errors, and dropped connections for *all* concurrent users, not just the one executing the heavy task.

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
        // Check h.mean, h.max, h.percentile(99) periodically
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

## 2. Race Conditions & Data Integrity

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

### Q2: Multiple users try to buy the last item in stock. How would you design the system?
*   **The Scenario:** A flash sale occurs where 5,000 requests try to purchase a product with only 1 item left in stock.
*   **High-Scale Architecture Solution:**
    ```
    Client Requests ---> [ Load Balancer ] ---> [ Node.js API Servers ]
                                                     |
                                            (Lua Script: Atomic Decrement)
                                                     v
                                              [ Redis Cache ] 
                                           (inventory:product_id)
                                                     |
                                           (Enqueue order if successful)
                                                     v
                                             [ Message Queue ]
                                                     |
                                            [ DB Worker Pool ]
                                                     |
                                                     v
                                           [ Relational DB (ACID) ]
    ```
*   **Implementation Steps:**
    1.  **Cache the Inventory in Redis:** Maintain inventory counts in Redis (`inventory:product_123 = 1`).
    2.  **Atomic Decr via Lua Scripting:** When a user checks out, run a Redis Lua script. Lua scripts run atomically in Redis:
        ```lua
        local stock = tonumber(redis.call('get', KEYS[1]))
        if stock and stock > 0 then
            redis.call('decr', KEYS[1])
            return 1 -- Success
        else
            return 0 -- Out of stock
        end
        ```
    3.  **Queue the Order:** If Redis returns `1`, immediately enqueue a job into a message queue (e.g., SQS/RabbitMQ) to finalize the transaction in the database asynchronously.
    4.  **Database Protection:** Ensure the database update has a safety constraint:
        `UPDATE products SET stock = stock - 1 WHERE id = 123 AND stock > 0;`

---

### Q3: Multiple requests attempt to update the same wallet balance. What could go wrong?
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

### Q4: How would you prevent duplicate order creation?
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

### Q5: How would you prevent duplicate payment processing?
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

## 5. API Design & Idempotency

### Q1: What is an idempotency key?
*   **Definition:** An idempotency key is a unique identifier (usually a UUID) sent by a client in an HTTP request header (e.g., `Idempotency-Key: f47ac10b-58cc-4372-a567-0e02b2c3d479`) that guarantees the server will execute the operation exactly once, regardless of how many times the request is resent.
*   **Primary Purpose:** Safely allows clients to retry failed network requests (timeouts, connection drops) without risk of performing duplicate transactions (like double-charging a card).

---

### Q2: How would you design an idempotent payment API?
*   **The Flow Diagram:**
    ```
    Client ---> [ API Gateway / Express App ] 
                      |
              (Check Cache for Key)
                      |
             +--------+--------+
             |                 |
         (Exists?)        (Not Exists)
             |                 |
       +-----+-----+           v
       |           |     Create DB record: [Key, Status=PENDING]
    [PENDING]  [COMPLETED]     |
       |           |     Call Stripe Payment Gateway
    Return     Return Cached   |
    409 error    Response      v
                         Update DB record: [Status=SUCCESS, ResponseData]
                               |
                               v
                         Return response to client
    ```
*   **Detailed Step-by-Step Implementation:**
    1.  **Check Key in Cache/Database:** Query `idempotency_keys` table.
    2.  **Handle Scenario A (Key Exists):**
        *   If status is `PENDING`: The transaction is currently being processed by another thread. Return `409 Conflict` (or standard waiting response).
        *   If status is `SUCCESS`: Return the saved response payload immediately without performing any business logic.
    3.  **Handle Scenario B (Key Does Not Exist):**
        *   Insert the key with status `PENDING` (must use unique DB index to prevent race conditions from concurrent duplicate requests).
        *   Execute the payment transaction.
        *   On success, update DB record status to `SUCCESS` and save the response data.
        *   Return response to the user.

---

### Q3: How would you prevent users from submitting the same request multiple times?
*   **Multi-layered Approach:**
    1.  **Frontend UX:** Disable the submit button immediately upon click. Show progress spinners.
    2.  **API Gateway Throttling:** Apply rate limiting (e.g., max 1 post request per user per 3 seconds on mutation routes).
    3.  **Backend Distributed Lock:** Acquire a Redis lock keyed by `lock:user_id:action` (e.g., `lock:123:create_post`) for a brief window (e.g., 2 seconds) upon request receipt. Drop subsequent requests if the lock cannot be acquired.
    4.  **Database Unique Constraints:** Use composite unique indexes (e.g., `user_id` + `target_id` on a "likes" table) to throw database-level errors on duplicates.

---

### Q4: How would you handle retry requests from clients?
*   **Server Design for Retries:**
    1.  **Differentiate Error Types:**
        *   *Transient Errors (429 Too Many Requests, 503 Service Unavailable, 504 Gateway Timeout):* Instruct the client that retrying is safe.
        *   *Non-Transient Errors (400 Bad Request, 403 Forbidden, 422 Unprocessable Entity):* Instruct the client *not* to retry, as the payload is invalid.
    2.  **Require Idempotency Keys:** For all write/mutation routes (`POST`, `PUT`, `PATCH`), enforce the use of idempotency keys.
    3.  **Expose Retry Headers:** Provide standard HTTP headers in response to rate limiting:
        *   `Retry-After: 30` (tells client to wait 30 seconds before retrying).

---

## 6. Scaling High-Concurrency Services

### Q1: How would you scale a Node.js application to handle 100,000 concurrent users?
*   **Scale Strategies (Edge-to-DB):**
    1.  **Horizontal Scale with Node.js Clusters:** Use PM2 or Docker/Kubernetes to run multiple instances of the Node.js process (typically 1 instance per CPU core) to utilize all host resources.
    2.  **Load Balancing:** Put a Layer 7 Load Balancer (Nginx, AWS ALB) in front of the server pool to distribute HTTP/WebSocket traffic.
    3.  **Stateless Architecture:** Ensure no session state is saved in memory on individual servers. Use a shared cache (Redis) for session management.
    4.  **CDN Offloading:** Cache static assets at the CDN edge (Cloudflare/CloudFront) so up to 80% of client requests never hit your Node.js servers.
    5.  **Database Optimization:** Use read replicas to offload query load, implement connection pooling (e.g., PgBouncer for Postgres) to prevent connection saturation, and index slow queries.
    6.  **Increase OS Limits:** Tune system configuration to allow massive socket connections (increase `nofile` limits in Linux limits.conf).

---

### Q2: What are the concurrency challenges in a microservices architecture?
*   **The Challenges:**
    1.  **Distributed Transactions (Lack of ACID):** A workflow spanning multiple services cannot rely on local database commits. (Solved by the **Saga Pattern**—a sequence of local transactions coordinated via events, with compensation steps for failures).
    2.  **Eventual Consistency Lag:** Reads might occur from a service before it has processed synchronization events from another service.
    3.  **Cascading Failures:** If Service A calls Service B, and Service B is slow, Service A's thread pool/connection pool will quickly saturate and crash. (Solved by implementing **Circuit Breakers** and timeouts).
    4.  **Data Replication Overhead:** Synchronizing shared entities (like user profiles) across independent service databases requires robust pub/sub event routing.

---

### Q3: How would you coordinate work across multiple Node.js instances?
*   **Coordination Tools & Strategies:**
    1.  **Message Brokers:** Use **Apache Kafka** or **RabbitMQ** to publish events and tasks. Workers subscribe to partitions/queues to divide and process work.
    2.  **Distributed Lock Managers:** Redis/Redlock to ensure only one instance executes a specific batch processing task.
    3.  **Shared Cache (Redis):** To share rate-limit counters, web session states, and active user tallies.
    4.  **Consul / ZooKeeper:** To handle leader election (electing one Node.js instance as the coordinator/master node that schedules work for other nodes).

---

### Q4: How would you design a notification service handling millions of concurrent requests?
*   **System Architecture:**
    1.  **Ingestion API (Fast & Stateless):** Node.js servers accept incoming notification payloads (user_id, message, channel), validate them, and write them directly to a high-throughput broker (**Apache Kafka**). The API responds with `202 Accepted` immediately.
    2.  **Message Partitioning:** Partition Kafka topics by `user_id` to ensure order delivery and prevent concurrent notification overlaps for the same user.
    3.  **Worker Groups:** Scaled consumer groups process events from Kafka.
    4.  **Distributed Rate Limiting:** Integrate a token-bucket rate limiter for outgoing SMS/Push/Email gateway APIs (e.g., Twilio, APNS) to prevent violating third-party provider limits.
    5.  **Failure Queues:** If a push gateway fails, route the message to a retry queue with exponential backoff.

---

## 7. Real-World System Design Scenarios

### Q1: Design a flash sale system where 1 million users try to buy 10,000 products.
*   **System Components:**
    1.  **Static Content Caching:** Cache the product description pages entirely at the CDN edge.
    2.  **Inventory Cache (Redis):** Load the 10,000 inventory stock counts into Redis.
    3.  **Checkout Gate (Redis Lua Decr):** When a user requests a purchase, execute an atomic Lua script:
        ```lua
        local stock = tonumber(redis.call('get', KEYS[1]))
        if stock and stock > 0 then
            redis.call('decr', KEYS[1])
            return 1
        end
        return 0
        ```
    4.  **Message Queue:** If Lua returns `1`, write an order-details message to a message queue (e.g., RabbitMQ).
    5.  **Asynchronous Order Processing Workers:** Consumers read from the queue and write orders to SQL DBs asynchronously.
    6.  **Write Sharding:** Shard the Order Database by `user_id` to handle concurrent write operations smoothly.

---

### Q2: Design a seat-booking system for movies or flights.
*   **System Components:**
    1.  **Pessimistic State Machine (Temporary Hold):**
        *   When a user clicks a seat, lock the seat in Redis with a TTL of 10 minutes: `lock:seat:flight_123:row_12:seat_A = user_999`.
        *   Database row status changes to `HELD` with an expiration timestamp.
    2.  **Preventing Double Bookings (Atomic Transition):**
        *   During payment completion, verify the lock in a database transaction:
            `UPDATE seats SET status = 'BOOKED', user_id = :userId WHERE id = :seatId AND status = 'HELD' AND expires_at > NOW();`
        *   If `rows_affected == 0`, rollback the payment charge and inform the user that the reservation expired.
    3.  **Real-Time Broadcast:** Use WebSockets to push seat status changes (`HELD`, `AVAILABLE`, `BOOKED`) instantly to all other clients viewing the seat map.

---

### Q3: Design a distributed rate limiter.
*   **System Components:**
    1.  **Algorithm Selection:** Sliding Window Counter (High accuracy, low memory overhead).
    2.  **Storage Engine:** Redis.
    3.  **Mechanism:**
        *   Keep two counters in Redis for each client IP: the current window counter and the previous window counter.
        *   Calculate the estimated request count mathematically based on elapsed time (see Cheat Sheet Section 11.1).
        *   Perform all checking and incrementing inside a Redis Lua script to prevent race conditions from concurrent requests.
    4.  **Local Memory Cache (Optimization):** To avoid hitting Redis for every single request, maintain a short-lived local cache (e.g., 500ms) of blocked IPs directly in the Node.js process memory.
    5.  **Fail-Open Strategy:** If Redis goes down, default to a local in-memory token bucket limiter or allow all requests (fail-open) to maintain core system availability.

---

### Q4: Design a real-time chat application supporting millions of users.
*   **System Components:**
    1.  **Connection Layer (WebSocket Servers):** A fleet of stateless Node.js servers handling persistent WebSocket connections (using the `ws` library or `Socket.io`).
    2.  **Session Routing & Sticky Connections:** Load Balancer distributes websocket handshakes using IP-Hash / Cookie-based sticky sessions.
    3.  **Pub/Sub Message Bus:** Run a Redis Pub/Sub cluster or Apache Kafka.
        *   Each WebSocket server subscribes to a channel matching its active users.
        *   When User A sends a message to User B, Node server A publishes the event `msg:user_B` to the Redis Pub/Sub cluster.
        *   Whichever WebSocket server hosts User B's active connection consumes the event and pushes the message down the socket.
    4.  **Message History Store:** Save chat history in a NoSQL database optimized for high-volume writes and range queries (e.g., Apache Cassandra or DynamoDB, partitioned by `conversation_id` and sorted by `timestamp`).

---

### Q5: Design a stock trading order processing service.
*   **System Components:**
    1.  **Order Ingestion (Kafka):** Ingest buy/sell orders. Partition the Kafka topic by **Ticker Symbol** (e.g., `AAPL`, `TSLA`). This guarantees that all orders for a specific stock are routed to the exact same partition in strict chronological order.
    2.  **In-Memory Matching Engine:**
        *   Write a single-threaded matching engine process per stock symbol. Because it runs on a single thread, it completely avoids locks and race conditions while matching orders.
        *   Maintain the Order Book (bid/ask queues) in-memory inside the matching process for microsecond-level speeds.
    3.  **LMAX Disruptor Pattern:** Use a ring-buffer queue to pass incoming messages to the matching engine thread at ultra-high throughput.
    4.  **Ledger Persistence:** Send matched trades to a database worker queue to save historical transactions into a relational SQL database for financial clearance.
    5.  **High Availability (Active-Passive):** Keep a hot-standby matching engine node replicating the same input Kafka partition. If the active matching engine crashes, the passive engine takes over instantly.
