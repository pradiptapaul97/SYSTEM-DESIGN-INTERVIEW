# System Design Interview Cheat Sheet

This guide is designed for quick review right before a system design interview. For each core architectural component, it outlines the **Interview Description**, **What Problem It Solves**, and the **Pros & Cons (Trade-offs)**.

---

## 📌 Table of Contents
1. [Server](#1-server)
2. [Domain Name System (DNS)](#2-domain-name-system-dns)
3. [Vertical Scaling (Scaling Up)](#3-vertical-scaling-scaling-up)
4. [Horizontal Scaling (Scaling Out)](#4-horizontal-scaling-scaling-out)
5. [Load Balancer](#5-load-balancer)
6. [Microservices Architecture](#6-microservices-architecture)
7. [API Gateway & Request Routing](#7-api-gateway--request-routing)
8. [Service-Specific Load Balancers](#8-service-specific-load-balancers)
9. [Background Service / Worker](#9-background-service--worker)
10. [Message Queue (Asynchronous Communication)](#10-message-queue-asynchronous-communication)
11. [Rate Limiting](#11-rate-limiting)
   - [11.1 Rate Limiting Algorithms](#111-rate-limiting-algorithms)
   - [11.2 Algorithms Comparison & Best Use Cases](#112-algorithms-comparison--best-use-cases)
12. [Polling (Short vs. Long Polling)](#12-polling-short-vs-long-polling)
13. [Pub/Sub vs. Message Queue (SQS)](#13-pubsub-vs-message-queue-sqs)
14. [Fan-Out Architecture](#14-fan-out-architecture)
15. [Scale-Out Database (Read Replicas)](#15-scale-out-database-read-replicas)
16. [Redis (In-Memory Caching)](#16-redis-in-memory-caching)
17. [Content Delivery Network (CDN)](#17-content-delivery-network-cdn)
18. [Event Sourcing](#18-event-sourcing)
19. [CQRS (Command Query Responsibility Segregation)](#19-cqrs-command-query-responsibility-segregation)

---

## 1. Server

*   **Interview Description:** A physical or virtual machine (VM) hosting your runtime application environment (e.g., Express/Node.js), listening to network ports, executing business logic, and querying storage layers.
*   **What Problem It Solves:** Provides a centralized compute container to process user requests, maintain state connections, and coordinate access to internal resources.
*   **Pros:**
    *   **Control:** Absolute flexibility over runtime software, frameworks, and deployment configurations.
    *   **Security:** Enables secure backend-to-backend communication shielded from direct client exposure.
*   **Cons:**
    *   **Finite Limits:** CPU, RAM, storage, and open port sockets are physically constrained on a single machine.
    *   **SPOF:** Represent a Single Point of Failure if running as a single instance.

---

## 2. Domain Name System (DNS)

*   **Interview Description:** A globally distributed database that acts as the "phonebook of the Internet", translating human-friendly domain names (e.g., `google.com`) into numerical IP addresses.
*   **What Problem It Solves:** Decouples client requests from hardcoded IP addresses, allowing backend servers to change locations/IPs seamlessly without breaking client connections.
*   **Pros:**
    *   **User-Friendly:** Users navigate via names, not complex numeric IPs.
    *   **Geographic Routing:** Can route users to the geographically closest server (GeoDNS), reducing latency.
*   **Cons:**
    *   **Cache Propagation Lag:** Changes to records can take hours to update globally due to clients caching old IPs until the Time To Live (TTL) expires.
    *   **SPOF Dependency:** If your authoritative DNS hosting goes down, your entire application is unreachable.

---

## 3. Vertical Scaling (Scaling Up)

*   **Interview Description:** Upgrading the hardware specifications (CPU, RAM, storage size) of an existing single server or database node to accommodate larger workloads.
*   **What Problem It Solves:** Resolves resource exhaustion (out of memory, 100% CPU usage) on a single server without modifying application code or database architecture.
*   **Pros:**
    *   **Simplicity:** No architectural changes required; application code remains exactly the same.
    *   **Strict Consistency:** Simplifies database design since all operations happen on one host (no distributed transaction locking needed).
*   **Cons:**
    *   **Downtime:** Hardware upgrades typically require taking the physical server offline temporarily (downtime).
    *   **Hardware Ceiling:** Motherboards have absolute capacity limits; you cannot scale vertically indefinitely.
    *   **Cost Curve:** Upgrading to premium, high-end enterprise servers is exponentially more expensive than purchasing standard commodity servers.

---

## 4. Horizontal Scaling (Scaling Out)

*   **Interview Description:** Adding more standard commodity servers (nodes) to your resource pool and distributing incoming client load across them.
*   **What Problem It Solves:** Bypasses the hardware limits of a single host, allowing systems to scale infinitely and achieve high availability.
*   **Pros:**
    *   **High Availability & Redundancy:** If one node crashes, other nodes absorb the traffic (no SPOF).
    *   **Linear Cost Scaling:** Uses cost-effective standard servers; you only pay for what you currently use.
    *   **Zero-Downtime Deployments:** Allows progressive rolling deployments node-by-node.
*   **Cons:**
    *   **Stateless Requirement:** Application servers must be stateless. Session data cannot be stored in-memory and must be moved to shared caches (like Redis).
    *   **High Complexity:** Requires load balancers, container orchestration, and centralized logging.

---

## 5. Load Balancer

*   **Interview Description:** A dedicated software (e.g., Nginx, HAProxy) or hardware proxy that sits in front of a server pool, distributing incoming network (Layer 4) or application (Layer 7) traffic.
*   **What Problem It Solves:** Prevents server overload, performs SSL termination to save backend CPU cycles, and stops routing traffic to crashed nodes using health check checks.
*   **Pros:**
    *   **Enables Horizontal Scale:** Acts as a single entry point for clients while managing multiple backend nodes.
    *   **Robust Health Checking:** Automatically routes around dead server instances.
*   **Cons:**
    *   **Extra Network Hop:** Adds a layer of routing latency between the client and the application.
    *   **Potential SPOF:** If the load balancer crashes, the entire system falls offline (requires Active-Passive failover configurations using VRRP).

---

## 6. Microservices Architecture

*   **Interview Description:** An architectural style that structures an application as a collection of small, autonomous, loosely coupled services (e.g., Auth, Product, Order) that communicate over lightweight protocols (HTTP/gRPC).
*   **What Problem It Solves:** Prevents monolithic development bottlenecks where large codebases result in deployment queues, team merge conflicts, and cascading code crashes.
*   **Pros:**
    *   **Fault Isolation:** A bug crashing the recommendation service does not impact checkout or payments.
    *   **Team Velocity:** Teams independently develop, deploy, and scale their services.
    *   **Polyglot Stack:** Allows matching the best language/database to each specific task.
*   **Cons:**
    *   **Operational Overhead:** Demands complex orchestration (Kubernetes), distributed logging, and tracing.
    *   **Distributed Transactions:** Enforcing ACID consistency across databases is difficult (requires eventual consistency patterns like Saga).

---

## 7. API Gateway & Request Routing

*   **Interview Description:** A centralized front-end entry point that intercepts all client requests, handles security, routes requests to specific microservice endpoints, and aggregates downstream responses.
*   **What Problem It Solves:** Shields clients from the complexity of multiple internal endpoints, provides a single interface for security (authentication/authorization), and reduces client network roundtrips.
*   **Pros:**
    *   **Unified Client Entry:** Clients only connect to one domain, permitting internal microservice refactoring.
    *   **Centralized Cross-Cutting Policies:** Implements Auth, SSL termination, CORS, and Rate Limiting in one gate layer.
*   **Cons:**
    *   **CPU Bottleneck (Node.js Gateway):** Gateways are I/O heavy; if they execute CPU-heavy tasks (like cryptography or token decoding), it blocks the single-threaded event loop.
    *   **Development Bottleneck:** Gateway routing rules must be maintained as new services scale up.

---

## 8. Service-Specific Load Balancers

*   **Interview Description:** Internal load balancers positioned in front of each individual microservice group to balance traffic among scaled instances of that specific service.
*   **What Problem It Solves:** Manages internal routing to dynamic, autoscaling containers/instances of internal services (e.g., routing orders API traffic to one of 10 Order workers).
*   **Pros:**
    *   **Dynamic Scaling Support:** Seamlessly integrates with Service Registries (like Consul) to route traffic around spinning/dying instances.
    *   **Internal Redundancy:** Provides service-to-service load balancing and isolated crash tolerance.
*   **Cons:**
    *   **Latency Accumulation:** Adding internal load balancers or sidecar proxies (service meshes) adds milliseconds to request times.
    *   **Tracing Complexity:** Makes debugging requests across multiple internal network hops harder.

---

## 9. Background Service / Worker

*   **Interview Description:** Decoupled processes running independently of the main web server, dedicated to executing long-running, CPU-intensive, or non-interactive tasks.
*   **What Problem It Solves:** Prevents blocking the web server (especially Node.js's single-threaded event loop) during heavy compute tasks (e.g., video transcoding, PDF generation), maintaining fast API response times.
*   **Pros:**
    *   **Event Loop Responsiveness:** Isolates CPU-heavy computation from the live client request path.
    *   **Fail-safety:** Background tasks can fail and retry independently without interrupting the user interface.
*   **Cons:**
    *   **Delayed Feedback:** The client cannot get an immediate response; requires polling, long polling, or WebSockets to display task status.
    *   **Infrastructure Overhead:** Requires task queues and workers to monitor job states.

---

## 10. Message Queue (Asynchronous Communication)

*   **Interview Description:** A temporary data buffer (e.g., SQS, RabbitMQ, BullMQ) that stores asynchronous tasks (messages) until background workers consume and process them.
*   **What Problem It Solves:** Decouples message production from processing, smoothing out sudden traffic spikes (acting as a backpressure shock absorber).
*   **Pros:**
    *   **Traffic Smoothing:** Absorbs massive spikes (e.g., flash sales) by queuing tasks safely rather than crashing workers.
    *   **Durability:** Messages are retained safely even if worker services go offline.
*   **Cons:**
    *   **Eventual Consistency:** Operations do not complete instantly; system updates are delayed.
    *   **Complex Debugging:** Tracing distributed transactional failures across queues is difficult.

---

## 11. Rate Limiting

*   **Interview Description:** A policy layer that throttles incoming user requests when they exceed a defined threshold (e.g., maximum 100 requests/minute per IP) using dedicated throttling algorithms.
*   **What Problem It Solves:** Protects server resources from denial-of-service (DDoS) attacks, brute-forcing, scraping, API abuse, and resource starvation caused by "noisy neighbors."
*   **Pros:**
    *   **API Availability:** Guarantees service availability for legitimate users.
    *   **Cost Control:** Prevents runaway cloud hosting costs by blocking abusive traffic at the edge.
*   **Cons:**
    *   **Shared State Overhead:** Scaling rate limiters requires a centralized, fast database (like Redis) to store request counters, introducing network latency.
    *   **False Positives:** Legitimate heavy users may get blocked during periods of rapid action.

### 11.1 Rate Limiting Algorithms

#### 1. Token Bucket
*   **Mechanism:** A bucket holds a maximum capacity $C$ of tokens. Tokens are refilled at a constant rate $r$ tokens/second. Each request consumes 1 token. If the bucket has tokens, the request is allowed. If the bucket is empty, the request is dropped/throttled.
*   **Best for:** Handling bursts of traffic (e.g., allows a burst of $C$ requests instantly, then throttles to the steady refill rate $r$).
*   **Pros:** Highly memory efficient (only stores the integer token count and the last refill timestamp); allows temporary bursts of traffic naturally.
*   **Cons:** Fine-tuning parameters ($C$ and $r$) can be challenging.

#### 2. Leaky Bucket
*   **Mechanism:** Requests enter a queue (the bucket) of capacity $C$ and leak (are processed) at a constant, fixed rate $r$ (First-In, First-Out). If the queue is full, incoming requests overflow and are discarded.
*   **Best for:** Smoothing out traffic rates (e.g., database writes or API proxying where downstream systems require a steady, uniform flow of requests).
*   **Pros:** Prevents traffic bursts entirely, ensuring a highly predictable and consistent outflow rate.
*   **Cons:** Can introduce latency for requests if the queue is large; drops requests during sudden bursts even if system capacity is idle.

#### 3. Fixed Window Counter
*   **Mechanism:** Divides time into fixed windows (e.g., 1-minute blocks). Each window has a counter. Every request in that window increments the counter. If counter > limit, requests are blocked. The counter resets when a new window starts.
*   **Best for:** Basic rate limiting where simple implementation is preferred.
*   **Pros:** Easy to implement and highly memory efficient (only stores one integer counter per user per window).
*   **Cons:** **Traffic Burst at Window Boundaries:** A user can make all their allowed requests at the end of Window A and all allowed requests at the start of Window B, doubling their allowed rate in a short time frame.

#### 4. Sliding Window Log
*   **Mechanism:** Keeps a log of timestamps for every request made by a user (typically stored in a sorted set in Redis). When a request arrives, remove all logs older than the current window (e.g., current time - 1 minute). The number of remaining logs is the request count. If count < limit, add the new timestamp to log and allow request.
*   **Best for:** High-accuracy rate limiting where window boundary spikes are completely unacceptable.
*   **Pros:** Extremely accurate; completely eliminates window boundary spikes.
*   **Cons:** High memory footprint since it stores a timestamp for *every single request* in the window, making it expensive for high-volume endpoints.

#### 5. Sliding Window Counter
*   **Mechanism:** A hybrid of Fixed Window and Sliding Log. It stores request counts for the current window and the previous window. When a request arrives, it computes:
    $$\text{Estimated Count} = \text{Current Count} + \left(\text{Previous Count} \times \frac{\text{Time remaining in current window}}{\text{Window Size}}\right)$$
    If this estimated count is less than the limit, allow the request and increment current window counter.
*   **Best for:** High-scale rate limiting requiring high accuracy without the memory footprint of Sliding Window Log.
*   **Pros:** Extremely memory efficient (only stores two counters per user) and prevents window boundary spikes with high accuracy (~99%).
*   **Cons:** The count is an estimate, assuming requests in the previous window were distributed uniformly (minor approximation).

### 11.2 Algorithms Comparison & Best Use Cases

| Algorithm | Memory Footprint | Accuracy | Traffic Shaping | Best Use Case (Recommended) |
| :--- | :--- | :--- | :--- | :--- |
| **Token Bucket** | Very Low (1 counter + 1 timestamp) | Medium-High | Burst-friendly | **AWS/SaaS APIs:** Standard choice for user-facing APIs where clients expect elastic burst capabilities for sudden spikes. |
| **Leaky Bucket** | Medium (Queue of size $C$) | High (Strict flow rate) | Steady-outflow | **DB Operations & API Proxies:** Protecting downstream databases from write spikes by queuing and releasing requests at a steady rate. |
| **Fixed Window** | Extremely Low (1 counter per window) | Low | Burst-vulnerable | **Low-Scale Internal Tools:** Basic usage limits (e.g., "max 100/day") where boundary spike limits are not a critical concern. |
| **Sliding Window Log** | High (Stores every timestamp in Redis Set) | Extremely High | Precise limit | **High Security Endpoints:** Auth, login attempts, and credit card validation interfaces where any burst bypass could be exploited. |
| **Sliding Window Counter** | Low (2 counters per window) | High (~99% accurate) | Precise limit | **High-Scale Public APIs (e.g., Stripe):** Perfect for high-traffic environments where accuracy is needed without memory bloat in Redis. |

---

## 12. Polling (Short vs. Long Polling)

*   **Interview Description:** The mechanism where a client or worker repeatedly queries a server/queue to check for updates.
    *   **Short Polling:** Consumer asks for updates; server returns instantly (empty or with data).
    *   **Long Polling:** Consumer asks for updates; if empty, the server holds the connection open (e.g., 20s) until data arrives.
*   **What Problem It Solves:** Allows clients/workers to pull data from servers in environments where real-time push protocols (like WebSockets) are not feasible.
*   **Pros & Cons Comparison:**
    *   **Short Polling Pros:** Simple to implement; client immediately knows status.
    *   **Short Polling Cons:** Causes extreme server/network churn, wastes CPU cycles, and inflates cloud costs due to empty query loops.
    *   **Long Polling Pros:** Saves up to 95% in API request costs on empty queues and reduces consumer busy-waiting.
    *   **Long Polling Cons:** Keeps persistent HTTP connections open, occupying socket connections on the server.

---

## 13. Pub/Sub vs. Message Queue (SQS)

*   **Interview Description:**
    *   **Message Queue (Point-to-Point):** 1:1 delivery. Messages are consumed and processed by *exactly one* worker instance. Once processed, it is deleted.
    *   **Pub/Sub (Publish/Subscribe):** 1:N delivery. Messages are published to a topic and broadcasted (copied) to *all active subscribers* simultaneously.
*   **What Problem It Solves:**
    *   **Message Queue:** Distributes and balances work tasks safely among a pool of competing workers.
    *   **Pub/Sub:** Decouples event-driven architectures, allowing multiple microservices to react to the same event independently.
*   **Pros & Cons Comparison:**
    *   **Queue Pros:** Native message locking and visibility management prevent duplicate worker tasks.
    *   **Queue Cons:** Limited to one destination; cannot easily broadcast to multiple services.
    *   **Pub/Sub Pros:** Easy to scale; you can add new downstream subscribers without touching the publisher's code.
    *   **Pub/Sub Cons:** Ephemeral delivery (subscribers lose offline events unless combined with persistent storage).

---

## 14. Fan-Out Architecture

*   **Interview Description:** An architecture pattern combining Pub/Sub (SNS) and Message Queues (SQS) where a single published event is duplicated and routed into multiple separate SQS queues (one queue per service type).
*   **What Problem It Solves:** Solves the limitations of both systems, allowing you to broadcast an event to multiple service types while retaining queue durability, locking, and independent retry policies for each worker pool.
*   **Pros:**
    *   **Failure Isolation:** If the Email Service queue crashes or backs up, the Shipping Queue continues processing orders normally.
    *   **Worker Autonomy:** Each service team manages their own queue parameters, worker numbers, and dead-letter queues.
*   **Cons:**
    *   **Complexity:** Higher management complexity of managing SNS topics, queue policies, and individual worker pools.
    *   **Cloud Costs:** Multiple duplicate messages traversing SNS and SQS add up to higher resource costs.

---

## 15. Scale-Out Database (Read Replicas)

*   **Interview Description:** Database replication where write operations are executed on a primary database instance (Primary), and data is replicated asynchronously to multiple read-only instances (Read Replicas).
*   **What Problem It Solves:** Alleviates database I/O bottlenecks in read-heavy applications by distributing query loads across multiple nodes.
*   **Pros:**
    *   **High Read Throughput:** Distributes read traffic across multiple replica instances.
    *   **Redundancy:** If the primary database crashes, a replica can be promoted to primary, minimizing downtime.
*   **Cons:**
    *   **Eventual Consistency:** Replicas lag behind the primary node. Users might perform a write, refresh, and query stale data (replication lag).
    *   **Write Bottlenecks:** Write capacity is still constrained by the single Primary node. (Requires sharding to scale writes).

---

## 16. Redis (In-Memory Caching)

*   **Interview Description:** An open-source, in-memory, key-value data structure store used as a database, cache, message broker, and rate-limiting counter.
*   **What Problem It Solves:** Reduces database read load and eliminates expensive computation cycles by returning pre-calculated data in sub-milliseconds.
*   **Pros:**
    *   **Ultra-Low Latency:** Operations occur entirely in RAM, offering speeds of < 1 millisecond.
    *   **Rich Structures:** Natively supports strings, lists, hashes, sorted sets, and pub/sub.
*   **Cons:**
    *   **Cache Invalidation:** Complex cache eviction strategies (TTL, LRU) must be coded to prevent users from seeing stale data.
    *   **RAM Cost:** Volatile memory (RAM) is significantly more expensive than persistent disk storage.
    *   **Data Loss Risk:** Since data is in-memory, a power loss or host crash can lose data (requires AOF/RDB persistence configurations).

---

## 17. Content Delivery Network (CDN)

*   **Interview Description:** A geographically distributed network of proxy servers that cache static resources (images, HTML, CSS, JavaScript, videos) close to the user's location.
*   **What Problem It Solves:** Eliminates geographic network latency caused by distance and offloads bandwidth/compute requests from origin web servers.
*   **Pros:**
    *   **Faster Loading Time:** Delivers static content from the nearest Edge location (POP), reducing latency.
    *   **Bandwidth Cost Reduction:** Offloads up to 90% of static asset traffic from your application servers.
    *   **DDoS Protection:** Absorbs high-volume static surges, serving as an internet firewall shield.
*   **Cons:**
    *   **Cache Invalidation Complexity:** Forcing CDNs to purge outdated cached files (e.g., CSS updates) requires purging calls or query-string hashing.
    *   **Debugging Difficulties:** Troubleshooting CDN configuration edge-cases or routing failures can be highly complex.

---

## 18. Event Sourcing

*   **Interview Description:** A design pattern where state changes are stored as a chronological sequence of immutable events in an append-only log (the Event Store), rather than just updating the current state in a traditional database. The current state is reconstructed dynamically by replaying all historical events from the beginning.
*   **What Problem It Solves:**
    *   **Loss of State History:** Traditional update-in-place databases delete historical history. Event Sourcing preserves the complete record of *why* and *how* a state changed over time.
    *   **Auditability & Compliance:** Serves as a 100% accurate, tamper-proof audit trail out of the box (crucial for banking, checkout systems, and logistics).
    *   **Time-Travel Queries:** Allows developers to inspect system state at any exact second in the past (e.g., "What was the account balance last Friday at 2:00 PM?").
    *   **Write Performance & Lock Contention:** Eliminates expensive read-modify-write locks on rows. Since writes are append-only inserts, lock contention is minimized, boosting write performance.
*   **Pros:**
    *   **Perfect Audit Log:** By definition, every action is logged, ensuring zero gaps in system events.
    *   **Reproducible Bugs:** Developers can copy production event logs to staging and replay them step-by-step to isolate and fix race conditions.
    *   **CQRS Synergy:** Perfectly aligns with CQRS (Command Query Responsibility Segregation). The write-model appends events, while a read-processor consumes events to update highly optimized search indices or query views.
*   **Cons:**
    *   **Replay Latency (High Scale):** Replaying millions of events to reconstruct an entity's current state introduces significant latency. (Mitigated by **Snapshots**—saving the state periodically, e.g., every 100 events, so you only replay from the last snapshot).
    *   **Schema Evolution (Versioning):** As requirements change, event payload formats change. Since the event store is immutable, old events cannot be rewritten; you must write **Upcasters** (mappers) to translate old event schemas to new ones during replay.
    *   **Steep Learning Curve:** Increases overall system architecture complexity, forcing teams to handle eventual consistency, CQRS, and out-of-order event issues.

---

## 19. CQRS (Command Query Responsibility Segregation)

*   **Interview Description:** An architectural pattern that segregates operations that mutate data (**Commands**) from operations that read data (**Queries**), using separate code paths, data models, and databases for each.
*   **Why It Came & What Problem It Solves:**
    *   **Read/Write Asymmetry:** Web apps are highly read-heavy (e.g., 99% reads, 1% writes). Combining them in a single database schema forces developers to optimize for both, which is a losing battle (writes require indexing removal/normalization; reads require heavy indexing/denormalization).
    *   **Conflicting Shared Models:** A single domain model handles both strict business validation (writes) and UI reporting displays (reads). This leads to complex database joins, slow UI query response times, and bloated database entity classes.
    *   **Locking & Resource Contention:** Transactional write updates (OLTP) compete for locks with long-running, CPU-heavy read reports (OLAP), causing system performance degradation.
*   **How It Works (The Mechanism):**
    *   **Command Side (Write Path):** Accepts mutation requests, validates business invariants, and writes to a write-optimized database (or Event Store).
    *   **Query Side (Read Path):** Queries a separate, highly denormalized read database (e.g., Elasticsearch for text search, Redis for cache, or a flat relational table) tailored directly to specific UI views.
    *   **Synchronization Bridge:** The write model publishes events (via a message broker like SQS/SNS or Kafka) when state changes. A background handler consumes these events and updates the read-only database asynchronously.
*   **Pros:**
    *   **Independent Scaling:** You can scale and optimize reads and writes separately (e.g., running 50 cheap read-replicas while keeping write masters small).
    *   **Highly Optimized Queries:** Read queries perform simple `SELECT` statements from flat tables, avoiding performance-destroying SQL `JOIN` statements.
    *   **Clean Domain Separation:** Decouples validation code from reporting presentation details, resulting in a cleaner codebase.
*   **Cons:**
    *   **Eventual Consistency:** Synchronization takes time. A user may click "save" but not see the update immediately if the read-model projection lags by a few milliseconds.
    *   **Architectural Bloat:** Managing dual databases, code models, message brokers, and projection engines increases maintenance overhead.
    *   **Data Drift Risks:** If an event is lost or processed out of order, the read-model will drift from the source-of-truth write database, requiring periodic reconciliation scripts.
