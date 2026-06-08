# Server-Side Engineering & Large-Scale System Design: Interview Prep Guide

This guide compiles high-frequency backend engineering, scaling, caching, API design, security, reliability, observability, and classic system design interview solutions.

---

## 📌 Table of Contents
1. [Node.js Concurrency Basics](#1-nodejs-concurrency-basics)
2. [Race Conditions & Data Integrity](#2-race-conditions--data-integrity)
3. [Distributed Systems & Locking](#3-distributed-systems--locking)
4. [Job Processing & Queue Design](#4-job-processing--queue-design)
5. [API Design & Idempotency](#5-api-design--idempotency)
6. [Scaling Concurrency](#6-scaling-concurrency)
7. [High Traffic & Scalability](#7-high-traffic--scalability)
8. [Caching Strategies & Patterns](#8-caching-strategies--patterns)
9. [API Design (Advanced)](#9-api-design-advanced)
10. [Authentication & Authorization](#10-authentication--authorization)
11. [Real-Time Systems](#11-real-time-systems)
12. [Microservices Architecture](#12-microservices-architecture)
13. [Reliability & Fault Tolerance](#13-reliability--fault-tolerance)
14. [Data Processing Pipelines](#14-data-processing-pipelines)
15. [Observability & Monitoring](#15-observability--monitoring)
16. [Multi-Tenant SaaS Systems](#16-multi-tenant-saas-systems)
17. [Common Senior-Level System Design Scenarios](#17-common-senior-level-system-design-scenarios)

---

## 1. Node.js Concurrency Basics

### Q1: How does Node.js handle thousands of concurrent requests despite being single-threaded?
*   **The Core Concept:** Node.js runs JavaScript on a single main thread (the Event Loop), but delegates all I/O operations asynchronously to the OS Kernel (via native interfaces like `epoll` or `kqueue`) or to the internal **Libuv thread pool** (for blocking operations like file systems or DNS queries). The main thread is freed immediately to handle the next request, and executes the callbacks once the I/O operations complete.

---

### Q2: What is the Event Loop, and how does it help concurrency?
*   **The Core Concept:** The Event Loop is an infinite coordination loop managed by Libuv that monitors and executes asynchronous callbacks in a strict order:
    1.  **Timers:** Executes callbacks from `setTimeout` and `setInterval`.
    2.  **Pending Callbacks:** Executes deferred I/O callbacks (e.g., TCP socket errors).
    3.  **Poll Phase:** Retrieves new I/O events and executes callbacks.
    4.  **Check Phase:** Executes callbacks registered with `setImmediate`.
    5.  **Close Callbacks:** Executes close events (e.g., `socket.on('close')`).
*   **Microtask Execution:** Between *every single operation* in these phases, Node.js completely drains `process.nextTick` callbacks and Promise microtasks.

---

### Q3: What operations can block the Event Loop?
*   **The Blockers:** Any CPU-bound synchronous code blocks the single main thread:
    *   Large array sorting, filtering, or heavy computations.
    *   Synchronous I/O operations (e.g., `fs.readFileSync`).
    *   Parsing/Stringifying huge JSON payloads synchronously.
    *   Catastrophic backtracking in regular expressions (ReDoS).
    *   Synchronous cryptographic hashes (e.g., synchronous `bcrypt.hashSync`).

---

### Q4: How would you identify Event Loop blocking in production?
*   **Methods:**
    1.  **Event Loop Utilization (ELU):** Track the ratio of active JS execution time vs. idle time using `process.eventLoopUtilization()`.
    2.  **Loop Delay Monitoring:** Use the native `perf_hooks` module with `monitorEventLoopDelay` to track execution delay percentiles.
    3.  **APMs:** Integrate APM tools (e.g., Datadog, New Relic) that measure loop delay using background timers.
    4.  **`blocked-at` Library:** In staging, use the `blocked-at` npm module to output stack traces of code blocking the loop.

---

### Q5: When would you use Worker Threads?
*   **The Choice:** Use **Worker Threads** (`worker_threads` module) for CPU-bound tasks inside a single process (e.g., cryptography, video transcoding, image compression). They allow executing JavaScript concurrently in separate thread spaces, sharing memory using `SharedArrayBuffer` for fast communication without the overhead of OS process creation.

---

### Q6: When would you use Child Processes?
*   **The Choice:** Use **Child Processes** (`child_process` module) when executing external CLI executables (e.g., running `ffmpeg` binary directly or a Python script), or when tasks require strict process and memory sandboxing to prevent a crash in one task from bringing down the main Node.js web server.

---

### Q7: How does Node.js handle concurrent I/O operations?
*   **Mechanism:** Network I/O (sockets, HTTP requests) is delegated to OS non-blocking system calls, running with zero helper threads. File system, DNS resolution, and crypto operations are blocking by default at the OS level, so Libuv routes them to a worker thread pool (configurable size via `UV_THREADPOOL_SIZE`, default is 4).

---

## 2. Race Conditions & Data Integrity

### Q1: Two users update the same resource simultaneously. How do you prevent race conditions?
*   **Solutions:**
    1.  **Optimistic Concurrency Control (OCC):** Add a `version` column. Update only if the version matches the read state: `WHERE id = :id AND version = :version`.
    2.  **Pessimistic Locking:** Lock rows on read using `SELECT ... FOR UPDATE` to block other transactions.
    3.  **Distributed Lock:** Acquire a Redis key lock on the resource ID before processing the request.

---

### Q2: Multiple users try to buy the last item in stock. How would you design the system?
*   **Design Solution:**
    1.  **Redis Cache Check:** Maintain stock count in Redis.
    2.  **Atomic Decrement:** Perform stock check and decrement inside an atomic Redis **Lua script** (`DECRBY`).
    3.  **Asynchronous Order Processing:** If the decrement is successful, publish a booking job to a message queue (SQS) for the database workers to insert the order record. Enforce database-level check constraints: `ALTER TABLE inventory ADD CONSTRAINT check_stock CHECK (stock >= 0)`.

---

### Q3: Multiple requests attempt to update the same wallet balance. What could go wrong?
*   **The Danger:** A **Lost Update** occurs. Request A and Request B read the balance as $100 simultaneously. Request A adds $20 and writes $120. Request B adds $50 and writes $150, silently overwriting Request A's $20 deposit.
*   **Solution:** Perform atomic database updates: `UPDATE wallets SET balance = balance + 50 WHERE user_id = :userId` or use pessimistic row-level locking (`FOR UPDATE`).

---

### Q4: How would you prevent duplicate order creation?
*   **Solutions:**
    1.  **Idempotency Keys:** Require a unique client-generated UUID on checkout requests. Enforce uniqueness in Redis or a DB unique constraint.
    2.  **Pessimistic Order Lock:** Acquire a Redis lock keyed by `lock:order:user_id:cart_id` to block concurrent checkout clicks.

---

### Q5: How would you prevent duplicate payment processing?
*   **Solutions:**
    1.  **Payment Gateway Idempotency:** Forward the client's idempotency key to Stripe/Adyen, which guarantees single processing.
    2.  **State Machine Checks:** Track states (`CREATED` -> `PENDING` -> `CHARGED`). Before calling the gateway, transition status to `PENDING` in the database. Lock the transaction row so no other worker can process it.

---

## 3. Distributed Systems & Locking

### Q1: Your application runs on 10 Node.js servers behind a load balancer. How would you implement locking?
*   **The Solution:** Standard in-memory locks are useless across multiple instances. You must use a centralized **Distributed Lock Manager (DLM)**. Implement the locking layer using a fast shared database like **Redis** (for high performance) or **ZooKeeper/Consul** (for strict consistency).

---

### Q2: What is a distributed lock?
*   **Definition:** A distributed lock is a mechanism that coordinates mutual exclusion to a shared resource across multiple independent processes/servers that do not share memory. It must guarantee **Mutual Exclusion** (only one holds the lock), **Deadlock-safety** (lock auto-releases via TTL if holder crashes), and **Fault Tolerance** (no single point of failure).

---

### Q3: How would you implement a distributed lock using Redis?
*   **Acquire Lock:** `SET lock:resource_name uuid NX PX 30000` (Atomic Set-If-Not-Exists with a 30-second TTL).
*   **Release Lock:** Execute a **Lua Script** to check the owner before deleting, preventing a slow worker from deleting a lock owned by a newer request:
    ```lua
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    ```

---

### Q4: What is the Redlock algorithm?
*   **The Concept:** Redlock is a distributed lock algorithm for multi-node Redis clusters. To acquire a lock, a client attempts to write a lock key in $N$ independent Redis master nodes (usually 5) with a short timeout. The lock is successfully acquired if the client writes to a **majority** (at least 3 of 5) of the nodes within a time less than the lock's validity lease.

---

### Q5: What are the limitations of Redis-based locking?
*   **The Limitations:**
    1.  **Clock Drift:** Redlock relies on physical system clocks for key expiry; system clock synchronization errors can violate locks.
    2.  **Asynchronous Replication Loss:** In replica setups, master node failures before replication can lead to duplicate locks.
    3.  **GC Pauses:** Process pauses (V8 garbage collection) can last longer than the lock TTL, causing workers to edit resources without holding a valid lock.

---

## 4. Job Processing & Queue Design

### Q1: How would you design a background job processing system?
*   **The Design:** Use a producer-consumer model:
    *   **Queue Broker:** Redis (with BullMQ) or RabbitMQ/AWS SQS.
    *   **Producer:** Express API servers write job metadata (payload, task type) to the queue.
    *   **Consumers:** Stateless worker processes poll the queue, process tasks, update states in database, and publish results to DLQs on failure.

---

### Q2: Multiple workers are consuming jobs from a queue. How do you ensure a job is processed only once?
*   **The Mechanism:** Use **Visibility Timeouts** (or Message Locks). When a worker pulls a job, the queue locks the job for a lease period (e.g., 5 minutes). If the worker succeeds, it deletes the job. If it crashes, the lock expires, the job reappears in the queue, and another worker can pull it.

---

### Q3: How would you handle job retries?
*   **Best Practices:**
    1.  Configure **exponential backoff** with random **jitter** to prevent thundering herds on downstream services.
    2.  Set a maximum retry cap (e.g., 3-5 times).
    3.  If all retries fail, route the message to a **Dead Letter Queue (DLQ)** for engineering analysis and alerting.

---

### Q4: How would you prevent duplicate execution of scheduled jobs?
*   **Solutions:**
    1.  **Orchestrator-Level Cron:** Run cron schedulers via Kubernetes `CronJob` to trigger a single executor container.
    2.  **Redis Distributed Lock:** Schedulers running on multiple servers attempt to acquire a lock (`lock:cron:send_newsletter`) with a TTL slightly shorter than the cron interval (e.g., 50s for a 1m cron). The node that gets the lock runs the job.

---

### Q5: How would you process 1 million image-processing requests concurrently?
*   **The Architecture:**
    1.  **S3 Upload:** Client uploads images directly to S3 via pre-signed URLs to save server bandwidth.
    2.  **Event Trigger:** S3 emits an upload event that places a message in AWS SQS.
    3.  **Autoscaling Workers:** Run worker nodes autoscaled by queue depth (using KEDA in Kubernetes).
    4.  **Process Optimization:** Inside workers, run the C++ `sharp` library in Worker Threads to process images, write outputs back to S3, and deliver via CloudFront CDN.

---

## 5. API Design & Idempotency

### Q1: What is an idempotency key?
*   **Definition:** An idempotency key is a client-provided UUID in request headers (e.g., `Idempotency-Key`) that allows the server to uniquely identify a request and guarantee that executing a mutating operation (like a checkout or balance transfer) multiple times yields the exact same outcome as the first invocation.

---

### Q2: How would you design an idempotent payment API?
*   **Implementation Steps:**
    1.  **Read Check:** Check if `idempotency_keys` table/Redis contains the key.
    2.  **In-Progress:** If key exists with status `PENDING`, return a `409 Conflict`.
    3.  **Completed:** If key exists with status `SUCCESS`, return the cached response payload immediately.
    4.  **New Request:** If not exists, insert key with status `PENDING` (using a database unique constraint), execute the payment transaction, update key status to `SUCCESS` along with the response payload, and return the response.

---

### Q3: How would you prevent users from submitting the same request multiple times?
*   **Solutions:**
    1.  **Frontend:** Disable the submit button and show progress spinners immediately on click.
    2.  **API Gateway:** Apply a rate limiter (e.g., max 1 POST to `/checkout` per user per 3 seconds).
    3.  **Backend Lock:** Acquire a Redis distributed lock keyed by `lock:action:user_id` upon request arrival, and release it after processing.

---

### Q4: How would you handle retry requests from clients?
*   **The Solution:** Require an idempotency key for all mutation routes. For transient failures (503 Service Unavailable, 429 Too Many Requests), return headers instructing the client to retry using exponential backoff with jitter. For non-transient failures (400 Bad Request, 401 Unauthorized), instruct clients not to retry.

---

## 6. Scaling Concurrency

### Q1: How would you scale a Node.js application to handle 100,000 concurrent users?
*   **Strategies:**
    1.  **Horizontal Scale:** Spin up stateless Node.js containers across multiple zones, routed via Layer 7 Load Balancers (ALB).
    2.  **Statelessness:** Externalize sessions to Redis clusters.
    3.  **Node Clustering:** Run PM2 cluster mode to utilize all CPU cores of hosting instances.
    4.  **Static Offloading:** Cache static assets and public APIs at the CDN edge (Cloudflare).
    5.  **Database Scale:** Implement pgBouncer connection pools, scale DB reads via Read Replicas, and add a Redis cache in front of DB queries.

---

### Q2: What are the concurrency challenges in a microservices architecture?
*   **The Challenges:**
    1.  **Data Consistency:** Enforcing transactions across databases (resolved by the **Saga Pattern** or Eventual Consistency).
    2.  **Distributed Race Conditions:** Multiple microservices attempting to update a shared entity concurrently (resolved by central distributed locks).
    3.  **Cascading Failures:** Service bottlenecks causing resource exhaustion in caller services (resolved by using **Circuit Breakers** and timeouts).

---

### Q3: How would you coordinate work across multiple Node.js instances?
*   **Solutions:**
    1.  **Message Brokers:** Publish/Subscribe to shared Kafka topics or RabbitMQ queues.
    2.  **Central Cache (Redis):** Share state counters, session tokens, and rate limiting buckets.
    3.  **Leader Election:** Use Consul or etcd consensus protocols to elect a single worker coordinator.

---

### Q4: How would you design a notification service handling millions of concurrent requests?
*   **The Design:**
    1.  **Ingest Gate:** Ingest request payloads rapidly into **Apache Kafka** partitioned by `user_id` to guarantee order. Return `202 Accepted` immediately to callers.
    2.  **Worker Fleet:** Run consumer groups that read partitions, format payloads, and distribute to third-party SMS/Push APIs.
    3.  **Throttling:** Use Redis token buckets to rate-limit outbound requests to third-party providers.

---

## 7. High Traffic & Scalability

### Q1: Design an API Gateway for 500 microservices.
*   **The Design:** Deploy an Envoy or Kong proxy cluster:
    *   **Reverse Proxying:** Route paths (`/orders/**` -> Orders service) utilizing a dynamic service registry.
    *   **Cross-Cutting concerns:** Terminate SSL, validate JWT headers, perform edge rate-limiting via Redis, and aggregate split service responses (BFF pattern).

---

### Q2: Design a backend that handles 10 million API requests per day.
*   **The Design:** 10M requests per day is **~116 RPS average** and **~580 RPS peak**.
    *   **Infrastructure:** Deploy 2 load-balanced Node.js instances across 2 availability zones.
    *   **Database:** A single primary PostgreSQL database with PgBouncer connection pooling and 1 Read Replica.
    *   **Caching:** Redis in front of database for hot catalog reads.

---

### Q3: How would you handle sudden traffic spikes (100x normal load)?
*   **Solutions:**
    1.  **Horizontal Pod Autoscaling (HPA):** Auto-scale containers based on CPU and request volumes.
    2.  **Edge Caching:** Aggressively cache GET endpoints at the CDN.
    3.  **Backpressure Queuing:** Route write operations to SQS/Kafka, smoothing out database write spikes.
    4.  **Load Shedding:** Immediately drop low-priority endpoints (e.g., recommendation feeds) with a HTTP `503` status.

---

### Q4: Design a system that can scale from 1 server to 100 servers without code changes.
*   **Requirements:**
    1.  **Zero Local State:** No local files, memory sessions, or in-memory caches. Use S3, Redis sessions, and Redis caching.
    2.  **Dynamic Discovery:** Register containers to a virtual network routed via DNS (Kubernetes cluster Services).
    3.  **External Configs:** Inject DB strings, keys, and feature flags via Env variables/vault systems.

---

### Q5: How would you perform load shedding during overload?
*   **Implementation:** Monitor Event Loop lag or CPU utilization in Node.js. If loop delay exceeds 100ms:
    *   Let the API Gateway filter and drop incoming requests to non-critical routes (e.g., `/search-recommendations`, `/analytics`) returning `503 Service Unavailable`, shielding checkout and payment services from crash failures.

---

### Q6: Design a multi-region backend architecture.
*   **The Blueprint:** Route users to the nearest active region via AWS Route 53 GeoDNS. Run application instances locally in each region. Connect databases globally using cockRoachDB (multi-master) or Aurora Global Database (1 writer, multiple global read-replicas). Sync static assets via cross-region S3 replication.

---

### Q7: How would you implement graceful degradation when dependencies fail?
*   **Solutions:** Wrap all outbound dependency calls (e.g., email service, recommendation service) in a **Circuit Breaker**. If the recommendation service fails, fallback to returning a hardcoded best-seller list or a generic profile configuration instead of failing the request.

---

### Q8: Design a system to serve 1 million concurrent WebSocket connections.
*   **Tuning Blueprint:**
    1.  **OS limits:** Set Linux file descriptor limits `ulimit -n 1048576`.
    2.  **TCP Buffers:** Tune memory buffers to 4KB per socket connection.
    3.  **Server Stack:** Deploy lightweight `ws` package in Node.js cluster modes behind Nginx reverse proxies utilizing round-robin load distribution.
    4.  **Cluster Sync:** Coordinate socket message routing across instances using Redis Pub/Sub channels.

---

### Q9: How would you distribute traffic across multiple Node.js instances?
*   **Solutions:** Use **PM2 Cluster Mode** locally to fork the application to match virtual CPU cores, sharing the network port. Use **Nginx** or **Application Load Balancers (ALBs)** to distribute HTTP requests round-robin across separate hosting server instances.

---

### Q10: Design a zero-downtime deployment strategy.
*   **Strategies:** Use **Blue-Green Deployments** (spin up a duplicate system, verify, and redirect the load balancer DNS) or **Rolling Updates** (Kubernetes deployment replacing pod instances one-by-one, validating readiness probes before termination of old instances).

---

## 8. Caching Strategies & Patterns

### Q1: Design a caching layer for an e-commerce platform.
*   **The Strategy:** Cache static assets at the CDN edge. Cache product listings and pricing catalog in a Redis cluster using a **Cache-Aside** pattern. Cache shopping carts and user sessions in Redis using a **Write-Through** pattern with strict expirations.

---

### Q2: How would you prevent cache stampede?
*   **Solutions:**
    1.  **Mutex Lock:** When a cache miss occurs, require the worker node to acquire a Redis distributed lock before querying the database and updating the cache.
    2.  **Probabilistic Early Expiration (XFetch):** Periodically re-calculate cache items in background workers before key expiration.

---

### Q3: How would you prevent cache penetration?
*   **Solutions:**
    1.  **Cache Nulls:** Cache empty search outcomes as nulls/empty objects with a short TTL (e.g., 3 minutes).
    2.  **Bloom Filter:** Deploy a Bloom filter in front of the cache to quickly identify and reject queries for non-existent IDs.

---

### Q4: How would you prevent cache avalanche?
*   **Solution:** Prevent mass key expiration at the same time by adding a **Random Jitter** (e.g., standard TTL of 2 hours plus or minus 5-15 minutes of random variance) to the expiration time of every cache key.

---

### Q5: Design a distributed cache invalidation strategy.
*   **Strategies:** Implement **Event-Driven Invalidation** (publish database update events over Redis Pub/Sub so all nodes invalidate their local memory caches) or **CDC (Change Data Capture)** monitoring DB logs via Debezium to invalidate cache elements.

---

### Q6: How would you cache user profiles updated frequently?
*   **Strategy:** Use a **Cache-Aside** model with write-invalidation. When a profile is updated, modify the database, then delete the cached profile key. The next profile request will miss the cache, fetch the updated database row, and update the cache.

---

### Q7: Design a cache-aside architecture.
*   **Read Workflow:** App checks Cache. If Hit, return data. If Miss, query Database, write output to Cache with a TTL, and return.
*   **Write Workflow:** App updates Database, then deletes the corresponding Cache key.

---

### Q8: How would you cache expensive database queries?
*   **Solution:** Calculate the MD5 hash of the SQL query string. Use `cache:query:md5_hash` as the Redis key. Store the query output payload as JSON with a strict TTL (e.g. 30 minutes). Run background warmers for critical slow dashboard queries.

---

### Q9: Design a session cache system.
*   **Solution:** Store session data as a Redis Hash (`session:token`). Set a TTL of 30 days (`EXPIRE`). On every subsequent client API request, call `EXPIRE` again (sliding expiration) to extend active user session durations.

---

### Q10: How would you handle stale cache data?
*   **Solutions:** Enforce short Time-To-Lives (TTLs) on transient records, use database trigger alerts to delete cache nodes, or append version prefixes (`v2:product_123`) to cache keys to invalidate old entries instantly.

---

## 9. API Design (Advanced)

### Q1: Design a public API platform used by third-party developers.
*   **Requirements:** Secure access via API Keys/OAuth2, apply rate limiting (tiered quotas), provide standard OpenAPI documentation, validate JSON envelopes, and provide sandbox endpoints for developer testing.

---

### Q2: Design API versioning for a large-scale application.
*   **Strategies:** Use **Path-Based Versioning** (`GET /api/v1/users`) because it is explicit, easy to route at the gateway, and works seamlessly with CDN caches. Avoid Accept headers (`Accept: application/vnd.company.v1+json`) as they complicate caching.

---

### Q3: How would you handle backward compatibility?
*   **Solutions:** Only make non-breaking additions (new properties or fields). Enforce backward compatibility via **Contract Testing** (Pact). If breaking changes are required, deprecate endpoints using headers (`Sunset`, `Deprecation`) and run parallel versions for a transition period.

---

### Q4: Design an API request validation framework.
*   **The Blueprint:** Use schema validation library middleware (e.g., **Zod** or **Joi**) on Express route handlers. Validate request parameters, queries, and bodies before executing controller business logic, returning `400 Bad Request` on mismatch.

---

### Q5: How would you design a bulk-processing API?
*   **The Design:** Bulk operations must be **Asynchronous**. Accept requests, return `202 Accepted` immediately with a `job_id`, write the items to Kafka, process them in batches in background workers, and expose `GET /jobs/:id` for polling progress.

---

### Q6: Design a file-upload API supporting large files.
*   **The Design:** Bypasses backend API servers. The client requests upload authorization -> Server returns an S3 **Multipart Presigned URL** -> Client uploads chunks directly to S3.

---

### Q7: Design an API throttling mechanism.
*   **The Design:** Implement a **Sliding Window Counter** inside Redis. Every incoming API request executes a Redis check on the client IP or token bucket. If limits are hit, return a `429 Too Many Requests` code with a `Retry-After` header.

---

### Q8: How would you handle partial failures in APIs?
*   **Solution:** Return HTTP **207 Multi-Status** with a response body detailing the status code and result of each item in the batch request, avoiding failing the entire batch due to a single bad record.

---

### Q9: Design an asynchronous API for long-running operations.
*   **The Blueprint:** Client triggers task -> Server returns `202 Accepted` with a `Location: /api/v1/tasks/:id` header. Client polls the task status. When completed, the task endpoint returns a `303 See Other` redirecting to the result resource.

---

### Q10: How would you implement request tracing across services?
*   **The Solution:** Generate a unique `Correlation-ID` header at the API Gateway. Propagate this header to all downstream microservices, database queries, and message queue payloads, outputting the trace ID in every log statement.

---

## 10. Authentication & Authorization

### Q1: Design a Single Sign-On (SSO) system.
*   **Mechanism:** Use **OIDC (OpenID Connect)**. When a user logs into Application A (Service Provider), it redirects them to the central Identity Provider (IdP). The IdP authenticates the user, sets a global cookie, and redirects them back with an Auth Code. App A exchanges the code for tokens. When the user visits App B, the IdP detects the global cookie and redirects back with an Auth Code without prompting for login.

---

### Q2: Design a JWT-based authentication system.
*   **The Design:** The Auth Server issues stateless **JWTs** containing user ID, scopes, and expiration, signed using an asymmetric private key (**RS256**). Resource microservices fetch the public key (via JWKS) and verify the signature locally without querying the Auth Server.

---

### Q3: Design a refresh-token mechanism.
*   **The Blueprint:** Access Token is short-lived (15 minutes), passed in Auth header. Refresh Token is long-lived (30 days), stored in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie. When the access token expires, the client sends the cookie to get a new access token, using refresh token rotation (issuing a new refresh token and deleting the old one) to prevent reuse.

---

### Q4: How would you revoke JWT tokens?
*   **Solutions:**
    1.  **Blacklist:** Store revoked token IDs (jti) in Redis with a TTL matching the token's remaining lifetime.
    2.  **User token version:** Store `token_version` in the database user record and JWT payload; increment the version upon user password resets or logout.

---

### Q5: Design Role-Based Access Control (RBAC).
*   **The Design:** Define roles (e.g., admin, editor) and map permissions (e.g., `write:post`, `read:users`) to roles. Assign roles to users. In the Node.js router, intercept routes with middleware checking if the user's role contains the required permission.

---

### Q6: Design Attribute-Based Access Control (ABAC).
*   **The Design:** A fine-grained authorization system that evaluates access rules based on attributes: Subject (user department, age), Resource (file classification, owner), and Environment (current time, client IP). Implement using policy engines (e.g., Open Policy Agent).

---

### Q7: Design secure API authentication between microservices.
*   **Solutions:** Implement **mTLS** (mutual TLS) using a Service Mesh (Istio) to encrypt and authenticate transport channels using X.509 certificates, or use machine-to-machine OAuth2 JWTs signed by a central authorization server.

---

### Q8: How would you protect APIs from replay attacks?
*   **Solutions:**
    1.  **Timestamp validation:** Require signatures to include a timestamp; reject requests older than 5 minutes.
    2.  **Nonces:** Require a unique transaction number (Nonce) on requests. Store it in Redis with a 5-minute expiry, and reject any duplicate nonces.

---

### Q9: Design an OTP verification system.
*   **The Blueprint:** Client requests OTP -> Server generates a random 6-digit code, hashes it, and saves it in Redis: `otp:userId = hashed_otp` with a 3-minute TTL. Send the code via Twilio SMS. During verify, compare the hashed input against Redis, deleting the key on success. Limit attempts to 3.

---

### Q10: Design a password reset system.
*   **The Blueprint:** User submits email. Server generates a random reset token, hashes it, writes it to a table with a 15-minute expiration, and sends the link to the user. User resets password; server updates the hash, increments `token_version` to log out active sessions, and deletes the reset token.

---

## 11. Real-Time Systems

### Q1: Design a real-time notification system.
*   **The Design:** Web API accepts notifications -> writes to Kafka -> Notification Worker processes and routes messages to a **Redis Pub/Sub** channel matching the recipient's user ID -> The WebSocket server hosting that user receives the message and sends it down the socket.

---

### Q2: Design a real-time collaborative document editor.
*   **The Solutions:** Use **Operational Transformation (OT)** (centralized conflict resolution resequencing document index inserts/deletes, suitable for Google Docs style) or **CRDTs (Conflict-Free Replicated Data Types)** (state-merging, serverless/decentralized compatible, suitable for Figma).

---

### Q3: Design a live sports score update system.
*   **The Blueprint:** Use **Server-Sent Events (SSE)** instead of WebSockets. SSE operates over HTTP, natively reconnects, and has lower memory footprints for one-way updates. Broadcast score updates via Redis Pub/Sub, and cache outputs at CDN edges with a 1-second TTL.

---

### Q4: Design a stock price streaming service.
*   **The Blueprint:** Ingest prices via gRPC into Kafka (partitioned by ticker symbol). WebSocket servers consume Kafka, aggregate updates in a memory ring buffer (e.g., batching updates to 10 ticks/second to prevent browser UI lag), and push to clients.

---

### Q5: Design a multiplayer game backend.
*   **The Blueprint:** Use **UDP** or **WebRTC data channels** to prevent TCP head-of-line blocking. The game server runs a loop (e.g. 30 ticks/sec), verifies inputs, executes physics, and broadcasts client state snapshots. Clients use prediction and interpolation.

---

### Q6: Design a typing indicator feature for chat applications.
*   **The Blueprint:** When a user types, client throttles WS events to 1 every 3 seconds. The server receives and broadcasts `typing` events to the room via Redis Pub/Sub. Receiving clients show the indicator and hide it after a 5-second inactivity timeout.

---

### Q7: Design a presence system showing online/offline users.
*   **The Blueprint:** Clients send a heartbeat to the server via WS or HTTP every 30 seconds. The server sets a Redis key `active:user_id` with a 40-second TTL. If the key expires, a Redis keyspace event triggers, notifying the friend circle via pub/sub.

---

### Q8: Design a real-time dashboard for monitoring metrics.
*   **The Blueprint:** Run collection daemons (StatsD/Telegraf) on servers. Ship metrics to InfluxDB/Prometheus (time-series databases). Render real-time graphs using Grafana dashboards with custom query intervals.

---

### Q9: Design a pub/sub messaging system.
*   **The Concept:** Use **Redis Pub/Sub** for transient, low-latency, fire-and-forget message routing. Use **Apache Kafka** or **NATS JetStream** for durable, append-only logs that allow consumers to replay historical streams.

---

### Q10: Design a live auction platform.
*   **The Blueprint:** Use WebSockets for real-time bid updates. Manage the current highest bid atomically in Redis using a Lua script. Write successful bids to a message queue for processing, and use event sourcing for auditing bids.

---

## 12. Microservices Architecture

### Q1: Design a microservices architecture for an e-commerce application.
*   **The Design:** Split the system into stateless domains: Auth, Catalog, Cart, Orders, and Payments. Enforce database-per-service. Use an API Gateway at the edge, and coordinate transactions using the **Saga Pattern** over a message broker (Kafka).

---

### Q2: How would you handle service discovery?
*   **The Solution:** Use Kubernetes DNS routing or Consul. When Service A makes a request to `http://catalog-service/products`, the service discovery layer routes the request to a healthy instance IP registered in the registry.

---

### Q3: Design inter-service communication.
*   **Solutions:** Use **Synchronous gRPC** for low-latency request-response workflows (e.g., verifying stock during checkout). Use **Asynchronous Message Queues** (Kafka/RabbitMQ) for decoupled workflows (e.g., shipping order confirmation after payment).

---

### Q4: REST vs gRPC for microservices?
*   **REST:** HTTP/1.1, text-based JSON, human-readable, wide compatibility. Best for public edge APIs.
*   **gRPC:** HTTP/2, Protobuf binary, multiplexed, code generation, 5-10x faster. Best for internal service-to-service communication.

---

### Q5: Design a service registry.
*   **The Design:** A highly available KV store (etcd/Consul) running the Raft consensus algorithm. Active nodes register their IPs and send heartbeats every 10 seconds. The registry removes nodes that fail health checks.

---

### Q6: Design a centralized configuration management system.
*   **The Design:** Store configuration files in a secure Git repository. Deploy a Config Server (Consul KV) that fetches files, decrypts secrets, and feeds them to microservices on startup, updating values dynamically via Webhooks.

---

### Q7: How would you implement distributed tracing?
*   **The Solution:** Use **OpenTelemetry** middleware in all microservices. When a request hits the gateway, inject a W3C trace ID. Propagate this ID through all network calls and ship execution trace spans asynchronously to a Jaeger collector.

---

### Q8: Design a Saga pattern workflow.
*   **The Design:** Choose **Orchestrator-based Saga**. An Order Orchestrator manages transaction steps. It calls Inventory (reserve stock), then Payment (charge card). If Payment fails, the Orchestrator executes compensation events (release stock in Inventory).

---

### Q9: Design an event-driven microservice architecture.
*   **The Design:** Microservices write all state mutations as events (e.g., `OrderPaid`) to **Apache Kafka**. Downstream services (e.g., Shipping Service) consume these event streams and update their local read-only database copies.

---

### Q10: How would you manage shared data across microservices?
*   **Solutions:** Implement **Event-Driven Replication** (Service A publishes change events, and Service B subscribes to build a local read-only table of the data) or use **API Composition** (aggregate data on-the-fly at the Gateway layer).

---

## 13. Reliability & Fault Tolerance

### Q1: Design a circuit breaker mechanism.
*   **The Design:** Wrap outbound network calls in a state machine:
    *   **Closed:** Normal path. If failure rate > 50%, transition to **Open**.
    *   **Open:** Fast-fail path. Block requests, immediately returning fallback/cached data. Start a timer.
    *   **Half-Open:** When the timer expires, allow a few requests. If they succeed, return to **Closed**; if they fail, return to **Open**.

---

### Q2: Design a retry strategy for failed requests.
*   **Strategy:** Configure clients to retry only transient HTTP failures (503, 429). Do not retry bad requests (400) or authorization issues (401). Limit retries to 3 attempts, and enforce idempotency on target API endpoints.

---

### Q3: How would you implement exponential backoff?
*   **The Solution:** Calculate the wait time mathematically: `delay = min(max_delay, base_delay * 2^attempt)`. Add random **jitter** (noise) to the delay to prevent all retrying clients from hitting the server at the exact same millisecond.

---

### Q4: Design a failover strategy for critical services.
*   **Strategy:** Deploy **Active-Active** setups across multiple availability zones using ALB. If Zone A goes down, the load balancer routes traffic to Zone B. For databases, deploy active-passive configurations with automated master failover.

---

### Q5: Design a health-check system.
*   **The Design:** Deploy two endpoints:
    *   `/liveness`: Returns HTTP 200 if the process is running.
    *   `/readiness`: Returns HTTP 200 if the app is connected to the DB, cache is warmed, and it is ready to receive traffic.

---

### Q6: How would you detect and recover from service crashes?
*   **The Solution:** Run processes using **PM2** or **systemd** locally to auto-restart crashed applications. In Kubernetes, ReplicaSets monitor pods and automatically replace crashed instances.

---

### Q7: Design a disaster recovery architecture.
*   **The Blueprint:** Run multi-region backups. Replicate database transactions asynchronously to a backup region (RPO < 1 hour). Set up automated infrastructure scripts (Terraform) to spin up the backup region (RTO < 15 minutes) if the main region fails.

---

### Q8: Design a high-availability backend.
*   **The Design:** Eradicate Single Points of Failure. Run stateless server instances across 3 Availability Zones behind a load balancer. Deploy database clusters with 1 master and 2 read-replicas, utilizing automatic failover orchestration.

---

### Q9: How would you handle dependency failures?
*   **Solutions:** Use Circuit Breakers to fail fast, set strict request timeout thresholds (e.g., 2 seconds) to avoid blocking threads, and provide fallback cache/mock payloads to maintain core app functionality.

---

### Q10: Design a self-healing system.
*   **The Design:** Deploy orchestrators (Kubernetes) that monitor container health probes. If a service becomes unhealthy (fails readiness/liveness checks), the orchestrator automatically kills the container, spins up a replacement, and alerts the engineering team.

---

## 14. Data Processing Pipelines

### Q1: Design a CSV import system for millions of records.
*   **The Design:** Stream the file using Node.js Streams and parse chunks with `csv-parser`. Delegate parsing and validations to **Worker Threads**. Group records into batches of 1,000 and write them to the database using bulk insert commands.

---

### Q2: Design a report generation system.
*   **The Design:** Client requests report -> API returns `job_id` and enqueues task to RabbitMQ. Workers read RabbitMQ, query a **read-replica database** (preventing primary DB lockups), format the PDF/CSV, save it to S3, and trigger a push notification.

---

### Q3: Design a log aggregation service.
*   **The Design:** Run Fluent Bit collectors on nodes to tail log files. Ship logs to **Apache Kafka** to absorb spikes. Kafka consumers parse logs and write them to **Elasticsearch**, visualized in Kibana (ELK stack).

---

### Q4: Design an analytics event collection pipeline.
*   **The Design:** Client hits a lightweight ingestion API. The API validates the envelope and writes the raw event directly to **Apache Kafka**. A stream parser consumes Kafka and inserts records in batches into a columnar database (**ClickHouse**).

---

### Q5: Design a batch-processing system.
*   **The Design:** Ingest large data dumps into **HDFS/S3**. Run **Apache Spark** batch jobs scheduled by Apache Airflow. Spark partitions the data, runs parallel transformations across worker nodes, and saves the output to a data warehouse.

---

### Q6: Design a stream-processing architecture.
*   **The Design:** Ingest real-time data into **Apache Kafka**. Connect **Apache Flink** or Spark Streaming to consume Kafka partitions. Perform sliding window aggregations, join streams, and output results to a Redis cache for dashboard displays.

---

### Q7: Design a search indexing pipeline.
*   **The Design:** Deploy **Debezium** to read the database transaction logs (Change Data Capture / CDC). Debezium publishes modifications to Kafka. An indexing worker consumes Kafka and writes updates to **Elasticsearch** in real-time.

---

### Q8: Design a recommendation data processing system.
*   **The Design:** Stream user actions (clicks, buys) to Kafka. Process the stream via Spark to generate feature matrices. Run an offline ML model daily to calculate recommendations, and save outcomes to Redis keyed by `recommendations:user_id`.

---

### Q9: Design a webhook processing platform.
*   **The Design:** Verify the incoming request signature at the gateway. Write the payload to **Amazon SQS**. Worker instances consume SQS, call target endpoints, and handle failures using exponential backoff retries.

---

### Q10: Design a data synchronization service.
*   **The Design:** Use CDC (Change Data Capture) to detect modifications. Publish updates to a message broker with a sequence number. Consuming nodes write changes to target stores, resolving conflicts using vector clocks or last-write-wins (LWW) rules.

---

## 15. Observability & Monitoring

### Q1: Design centralized logging for microservices.
*   **The Blueprint:** Standardize all microservice logs into structured JSON format containing trace IDs and levels. Run log-shipper daemons (Filebeat) on nodes to collect and forward logs to a centralized Elasticsearch cluster.

---

### Q2: Design a metrics collection platform.
*   **The Blueprint:** Deploy Prometheus. Configure Prometheus to scrape (pull) metrics from application `/metrics` endpoints (exposed via Prometheus client libraries) every 15 seconds, storing them in a time-series database.

---

### Q3: Design distributed tracing across services.
*   **The Blueprint:** Instrument services with **OpenTelemetry**. Inject a `traceparent` header at the gateway. Propagate this header through all downstream network calls, and export trace span metrics to a Zipkin or Jaeger collector.

---

### Q4: How would you monitor Node.js performance in production?
*   **Key Metrics:** Track loop delays (Event Loop Lag), Event Loop Utilization (ELU), memory usage (heapUsed, RSS), active handles, and garbage collection duration using the native `perf_hooks` and client libraries.

---

### Q5: Design an alerting system.
*   **The Design:** Deploy Prometheus Alertmanager. Define alerting rules (e.g. CPU > 85% for 5 minutes, API error rate > 2%). Configure Alertmanager to deduplicate, group, and route alerts to PagerDuty or Slack based on severity.

---

### Q6: How would you detect memory leaks in Node.js applications?
*   **Strategy:** Monitor RSS and heapUsed memory trends in APMs. If memory grows continuously and never declines after GC, generate heap snapshots using the V8 inspector and compare them to isolate leaking objects.

---

### Q7: Design a request correlation system.
*   **The Design:** Inject a unique `X-Correlation-ID` header at the API Gateway. Pass this ID through all microservice calls, thread contexts (using `AsyncLocalStorage` in Node.js), and log statements to trace individual client request paths.

---

### Q8: Design an application monitoring dashboard.
*   **The Design:** Deploy Grafana. Build dashboards that display RED metrics (Rate, Errors, Duration) for APIs, and USE metrics (Utilization, Saturation, Errors) for system resources (CPU, Memory).

---

### Q9: Design an SLA monitoring system.
*   **The Design:** Run synthetic monitoring probers from multiple global regions to ping core API endpoints every minute. Calculate availability (uptime percentage) and latency percentiles (p95, p99) against target SLAs.

---

### Q10: Design an audit logging system.
*   **The Design:** Append-only ledger design. Write security-sensitive actions (e.g. user privilege changes) to an immutable table. Sign log payloads cryptographically to make audit logs tamper-evident.

---

## 16. Multi-Tenant SaaS Systems

### Q1: Design a SaaS application serving thousands of companies.
*   **The Blueprint:** Set up a tenant routing gateway that extracts the tenant identifier from the request host (e.g., `tenantA.saas.com`) or headers, resolves the tenant context, and applies database isolation policies dynamically.

---

### Q2: How would you isolate tenant data?
*   **Models:**
    1.  **Database-per-tenant:** Maximum isolation, most expensive.
    2.  **Schema-per-tenant:** Isolated schemas in a shared database.
    3.  **Shared-Database-Shared-Schema:** Row-level isolation via a `tenant_id` column, enforced using Postgres Row-Level Security (RLS) policies.

---

### Q3: Design tenant-specific configuration management.
*   **The Design:** Store tenant configurations in a database table. Cache configs in Redis keyed by `tenant:config:tenant_id` with a long TTL, invalidating the cache key when modifications are made in the admin portal.

---

### Q4: Design tenant-level rate limiting.
*   **The Design:** Implement a Redis sliding window counter. Key the counters by tenant ID (`rate:tenant_id`). Apply different rate thresholds based on the tenant's billing tier.

---

### Q5: Design a tenant-aware caching strategy.
*   **The Design:** Namespace all Redis cache keys using the tenant ID as a prefix: `tenant_id:product:product_id`. This prevents cross-tenant data leaks and simplifies cache purging when a tenant updates their catalog.

---

### Q6: Design billing and usage tracking for tenants.
*   **The Design:** Log usage events (e.g., number of active users, database size) to a message queue. Run workers to aggregate usage totals daily, and sync billing states using the Stripe Billing API.

---

### Q7: Design tenant-specific feature flags.
*   **The Design:** Maintain a feature flag table mapped to tenant IDs. Cache flags in memory, evaluating rules (e.g., "Enable beta feature only for premium tier tenants") dynamically during request routing.

---

### Q8: Design multi-tenant authentication.
*   **The Design:** Support federated identity portals. Allow tenants to configure their own identity providers (OIDC/SAML). Partition user pools so that usernames are scoped and verified strictly within their tenant workspace.

---

### Q9: Design tenant-level monitoring.
*   **The Design:** Inject `tenant_id` as a tag in application logs and Prometheus metrics. Build Grafana dashboards filtered by tenant ID to monitor usage, error rates, and latency for specific high-volume enterprise clients.

---

### Q10: Design tenant data migration strategies.
*   **Strategy:** Maintain database migration scripts in version control. Run migration pipelines sequentially or in batches across tenant schemas, utilizing backward-compatible column schemas to prevent downtime.

---

## 17. Common Senior-Level System Design Scenarios

### Q1: Design a URL shortener.
*   **System Components:**
    1.  **Distributed Counter:** Use a Redis cluster (`INCR`) to generate unique auto-incrementing integer IDs.
    2.  **Base62 Hashing:** Convert the integer ID to a Base62 string (e.g. ID `10,000,000` -> `6LAg`).
    3.  **Database:** Store mappings in a relational table `urls (id, short_code, original_url, created_at)` with a unique index on `short_code`.
    4.  **Redirection:** Place a Redis cache in front of database lookups. Use HTTP `302 Found` to redirect clients and track analytics.

---

### Q2: Design a WhatsApp-like messaging system.
*   **System Components:**
    *   **Connections:** Persistent WebSockets across stateless connection gateway servers.
    *   **Data Broker:** Kafka ingests messages and routes them to user inbox partitions.
    *   **Storage:** NoSQL DB (**Cassandra**) partitioned by `conversation_id` and sorted by `timestamp` for fast sequential reads.
    *   **States:** Messages transition: `SENT` -> `DELIVERED` (notified via recipient socket check) -> `READ`.

---

### Q3: Design a LinkedIn feed generation service.
*   **The Design:**
    *   **Feed Cache:** Redis sorted sets containing post IDs for each user, sorted by timestamp.
    *   **Push Model (Fan-out-on-write):** When a standard user posts, write the post ID directly to the feed caches of all their followers.
    *   **Pull Model (Fan-out-on-read):** When a user views their feed, fetch their push feed cache, query the posts of followed high-profile users (celebrities/influencers), and merge them on-the-fly.

---

### Q4: Design a YouTube video processing backend.
*   **The Pipeline:**
    1.  **Upload:** Client uploads video directly to S3 via presigned multipart links.
    2.  **Queue:** S3 triggers an event that enqueues jobs in a message broker.
    3.  **Transcode:** Workers download files, run FFmpeg to transcode them into multiple formats and resolutions (1080p, 720p, etc.), slice them into short chunks (ABR), and save them to S3.
    4.  **Distribution:** Deliver chunks globally via CloudFront CDN.

---

### Q5: Design a payment gateway aggregator.
*   **System Components:**
    *   **Adapter Interface:** Unified class design to interact with Stripe, PayPal, and Adyen.
    *   **Routing Engine:** Dynamically routes transactions based on gateway fee structures and real-time success rates.
    *   **Reconciliation Cron:** A daily background cron job that fetches gateway reports, matches them against database transactions, and alerts on discrepancies.

---

### Q6: Design an Uber-like ride matching service.
*   **System Components:**
    1.  **Geospatial Index:** Divide the map into cells using **Uber H3** or Google S2 grids.
    2.  **Driver Location Index:** Drivers stream GPS coordinates via WS every 4 seconds. The system updates their H3 cell index in Redis Geospatial sets (`GEOADD`).
    3.  **Matching Engine:** Request pickup -> query Redis for drivers within a 3-mile radius (`GEORADIUS`) -> send request sequentially to the closest driver.

---

### Q7: Design an Amazon order processing system.
*   **System Components:**
    1.  **Inventory Check:** Use Redis Lua script decrements during checkout requests to reserve products atomically.
    2.  **Payment Gateway:** Charge card with an idempotency key.
    3.  **Orders Database:** Relational DB (Postgres) using Read Committed isolation to store transaction records.
    4.  **Fulfillment:** Publish event to SQS queue; warehouse fulfillment service picks it up to schedule packaging and delivery.

---

### Q8: Design a Food Delivery Backend (e.g., DoorDash).
*   **System Components:**
    1.  **Order Pipeline:** Customer places order -> Restaurant receives push event (SSE/WebSockets) -> Restaurant accepts and prepares.
    2.  **Driver Dispatch:** System queries Redis Geospatial for drivers near the restaurant, and routes matching offers via a matching engine.
    3.  **Real-Time Tracking:** Drivers stream GPS coordinate locations via WebSockets. The system pushes updates to the customer's open socket.

---

### Q9: Design a Cloud File Storage Service (S3/Dropbox).
*   **System Components:**
    1.  **Chunking:** Slices large files into 4MB blocks.
    2.  **Deduplication:** Hash blocks using SHA-256. Before saving a block, check if the hash exists in the metadata DB to prevent duplicate storage.
    3.  **Metadata DB:** Relational DB storing file schemas, path references, block hashes, versions, and permissions.
    4.  **Block Store:** Amazon S3.

---

### Q10: Design a distributed cron scheduler.
*   **System Components:**
    1.  **State DB:** relational database storing job schedules, execution stats, and states.
    2.  **Leader Election:** Raft consensus (Consul) elects a leader manager. Only the leader schedules and dispatches tasks.
    3.  **Execution Queue:** Schedulers write executable jobs to RabbitMQ.
    4.  **Workers:** Stateless worker processes consume and execute the jobs, reporting execution results back to the database.

---

### Q11: Design a flash sale system where 1 million users try to buy 10,000 products.
*   **System Components:**
    1.  **Edge Cache (CDN):** Cache product description pages at CDN POPs to shield servers from traffic.
    2.  **Inventory Reservation (Redis):** Cache product stock in Redis. When checkout is clicked, run a Lua script to check and decrement stock atomically. Reject further requests immediately if stock is 0.
    3.  **Async Order Processing:** If Redis decr succeeds, enqueue the transaction to Kafka. Workers consume Kafka and write orders to PostgreSQL shards asynchronously.

---

### Q12: Design a seat-booking system for movies or flights.
*   **System Components:**
    1.  **Temporary Hold (Redis):** When a user clicks a seat, lock the seat in Redis with a 10-minute TTL: `lock:seat:flight_id:seat_no = user_id`. Row status changes to `HELD`.
    2.  **Atomic Checkout (DB):** During payment completion, update seat status inside an ACID transaction:
        `UPDATE seats SET status = 'BOOKED', user_id = :userId WHERE id = :seatId AND status = 'HELD' AND expires_at > NOW();`
        Rollback payment if 0 rows affected.
    3.  **Real-time status updates:** Broadcast state changes (`HELD`, `AVAILABLE`, `BOOKED`) via WebSockets to all active viewers.

---

### Q13: Design a distributed rate limiter.
*   **System Components:**
    1.  **Storage Engine:** Redis.
    2.  **Limiting Algorithm:** Sliding Window Counter using Redis Sorted Sets (ZADD).
    3.  **Flow:** Hash client IP/Token to create key. Run check and increment inside a Redis Lua script to maintain atomicity and eliminate race conditions.
    4.  **Local Memory Cache (Optimization):** To reduce Redis connection latency, maintain a local cache (500ms TTL) of blocked client IPs in backend server memory.

---

### Q14: Design a stock trading order processing service.
*   **System Components:**
    1.  **Ingestion:** Ingest buy/sell orders into **Apache Kafka** partitioned by **Ticker Symbol**. This guarantees that all orders for a specific stock are routed to the exact same partition in strict chronological order.
    2.  **Matching Engine:** Write a single-threaded matching process per stock symbol. Because it is single-threaded, it completely avoids database lock contention and race conditions while matching orders in microsecond-level speeds.
    3.  **Clearance:** Send matched trades to a database worker queue to save historical transactions to a relational SQL database.
