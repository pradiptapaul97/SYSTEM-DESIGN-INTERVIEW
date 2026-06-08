# System Design Interview Guide: Request Flow & Infrastructure

This guide details how a request flows through modern, distributed system infrastructure. Structured in the exact order a request travels—from DNS lookup to downstream microservices—it highlights **why we use each component**, **how they solve problems**, **pros & cons**, and critical **💡 Interview Deep Dives** covering architectural trade-offs, algorithms, and real-world system design questions.

---

## 📌 The End-to-End Request Journey
When a user types a URL (e.g., `https://example.com/orders`) into their browser, the request journeys through the following infrastructure:

```
[ User Browser ]
       │
       ├─► [ 1. DNS Lookup ] ──(Translates Domain -> IP Address using Anycast/GeoDNS)
       │
       ▼
[ 2. Infrastructure Entry Load Balancer (L4/L7) ] ──(Entry point for Horizontal Scaling)
       │
       ▼
[ 3. API Gateway ] ──(Unified Entry: Performs Auth, Rate Limiting, Logging & Routes paths)
       │
       ├─► (If Path: /users) ──► [ 3.3 User-Service Load Balancer ] ──► [ User Service Instance 1/2 ]
       │
       └─► (If Path: /orders) ──► [ 3.3 Order-Service Load Balancer ] ─► [ Order Service Instance 1/2 ]
```

---

## 📌 Table of Contents
1. [Domain Name System (DNS)](#1-domain-name-system-dns)
   - [1.1 Why We Use DNS & How It Solves the Problem](#11-why-we-use-dns--how-it-solves-the-problem)
   - [1.2 How It Works (DNS Lookup Workflow)](#12-how-it-works-dns-lookup-workflow)
   - [1.3 Pros and Cons of DNS](#13-pros-and-cons-of-dns)
   - [1.4 💡 Interview Deep Dive: DNS Design Questions](#14--interview-deep-dive-dns-design-questions)
2. [Scaling](#2-scaling)
   - [2.1 Why We Use Scaling & How It Solves the Problem](#21-why-we-use-scaling--how-it-solves-the-problem)
   - [2.2 Vertical Scaling (Scaling Up)](#22-vertical-scaling-scaling-up)
     - [2.2.1 Pros and Cons of Vertical Scaling](#221-pros-and-cons-of-vertical-scaling)
   - [2.3 Horizontal Scaling (Scaling Out) & Entry Load Balancers](#23-horizontal-scaling-scaling-out--entry-load-balancers)
     - [2.3.1 Why We Use a Load Balancer in Horizontal Scaling](#231-why-we-use-a-load-balancer-in-horizontal-scaling)
     - [2.3.2 Load Balancing Algorithms & Mechanisms (L4 vs. L7)](#232-load-balancing-algorithms--mechanisms-l4-vs-l7)
     - [2.3.3 Pros and Cons of Horizontal Scaling & Load Balancing](#233-pros-and-cons-of-horizontal-scaling--load-balancing)
     - [2.3.4 💡 Interview Deep Dive: Consistent Hashing & High Availability](#234--interview-deep-dive-consistent-hashing--high-availability)
3. [Microservices Architecture](#3-microservices-architecture)
   - [3.1 Why We Use Microservices & How It Solves the Problem](#31-why-we-use-microservices--how-it-solves-the-problem)
   - [3.2 The API Gateway (Unified Entry Point)](#32-the-api-gateway-unified-entry-point)
     - [3.2.1 Why We Use an API Gateway & How It Solves Routing Problems](#321-why-we-use-an-api-gateway--how-it-solves-routing-problems)
     - [3.2.2 Pros and Cons of an API Gateway](#322-pros-and-cons-of-an-api-gateway)
     - [3.2.3 💡 Interview Deep Dive: Rate Limiting & Gateway Failover](#323--interview-deep-dive-rate-limiting--gateway-failover)
   - [3.3 Service-Specific Load Balancers (Request-Based Routing)](#33-service-specific-load-balancers-request-based-routing)
     - [3.3.1 Why We Use Service-Specific Load Balancers](#331-why-we-use-service-specific-load-balancers)
     - [3.3.2 How It Solves the Internal Service Routing Problem](#332-how-it-solves-the-internal-service-routing-problem)
     - [3.3.3 Pros and Cons of Service-Specific Load Balancers](#333-pros-and-cons-of-service-specific-load-balancers)
     - [3.3.4 💡 Interview Deep Dive: Client-Side vs. Server-Side Load Balancing](#334--interview-deep-dive-client-side-vs-server-side-load-balancing)
   - [3.4 The Downstream Microservices](#34-the-downstream-microservices)
     - [3.4.1 Why We Deploy Separate Services](#341-why-we-deploy-separate-services)
     - [3.4.2 Pros and Cons of Microservices Architecture](#342-pros-and-cons-of-microservices-architecture)
     - [3.4.3 💡 Interview Deep Dive: Distributed Transactions & Join Patterns](#343--interview-deep-dive-distributed-transactions--join-patterns)
4. [Batch Processing](#4-batch-processing)
   - [4.1 Why We Use Batch Processing & How It Solves the Problem](#41-why-we-use-batch-processing--how-it-solves-the-problem)
   - [4.2 Core Architecture & Workflow](#42-core-architecture--workflow)
   - [4.3 Pros and Cons of Batch Processing](#43-pros-and-cons-of-batch-processing)
   - [4.4 💡 Interview Deep Dive: Batch vs. Stream Processing (Lambda vs. Kappa)](#44--interview-deep-dive-batch-vs-stream-processing-lambda-vs-kappa)

---

## 1. Domain Name System (DNS)

### 1.1 Why We Use DNS & How It Solves the Problem
*   **The Problem:** Computers identify and connect to each other over the network using numerical IP addresses (IPv4 or IPv6). Humans cannot easily memorize these number sequences. Furthermore, if a server's IP address changes due to migrations, hardware failure, or scaling, hardcoded client configurations will break immediately.
*   **How DNS Solves It:** DNS acts as the "phonebook of the Internet". It translates human-readable domain names (e.g., `google.com`) into machine-readable IP addresses dynamically, separating user navigation from physical infrastructure configurations.

### 1.2 How It Works (DNS Lookup Workflow)
1.  **Resolver:** The client's browser queries the Local DNS Resolver (usually managed by the ISP).
2.  **Root Server:** If the IP is not cached, the Resolver queries the Root Nameserver (`.`) to locate the Top-Level Domain (TLD) server.
3.  **TLD Server:** The TLD Server (e.g., `.com` or `.org`) directs the resolver to the Authoritative Nameserver.
4.  **Authoritative Nameserver:** Holds the definitive DNS records (A, AAAA, CNAME) and returns the actual server IP.
5.  **Caching:** The IP is returned to the client and cached locally for a duration defined by the Time To Live (TTL).

### 1.3 Pros and Cons of DNS
*   **Pros:**
    *   **User Friendliness:** Enables human-readable domains, boosting brand accessibility.
    *   **Abstraction Layer:** Decouples domain names from host IPs, facilitating seamless server migrations.
    *   **Global Traffic Management (GeoDNS):** Resolves requests to the geographically closest data center IP, reducing latency and scaling traffic globally.
*   **Cons:**
    *   **Cache Propagation Delay:** Changes to DNS records can take hours (or days) to distribute globally because clients cache old IPs until their TTL expires.
    *   **Security Vulnerabilities:** DNS queries are traditionally unencrypted, making them susceptible to **DNS hijacking**, **cache poisoning (spoofing)**, and large-scale DDoS attacks.
    *   **SPOF Dependency:** If your authoritative DNS provider goes offline (e.g., Dyn DNS attack), your domain becomes completely unreachable.

### 1.4 💡 Interview Deep Dive: DNS Design Questions
> [!NOTE]
> **Q: How do massive websites implement high availability at the DNS level?**
> **A:** They use **Anycast Routing**. In Anycast, multiple physical servers in different locations share the exact same IP address. Routers use BGP (Border Gateway Protocol) to send the user's traffic to the topologically closest physical server. If one Anycast location fails, the network path dynamically recalculates to route traffic to the next closest location.

> [!TIP]
> **Q: What is the risk of setting a very low TTL (e.g., 0 seconds)?**
> **A:** While a low TTL allows near-instant failover and IP changes, it forces clients to request DNS lookups on every connection. This introduces significant latency to the request flow and places a heavy compute and cost load on your authoritative nameservers.

---

## 2. Scaling

### 2.1 Why We Use Scaling & How It Solves the Problem
*   **The Problem:** An application experiencing growth faces high concurrent traffic. A default single-server configuration contains physical limitations (CPU, RAM, storage, network interface cards). When demand overflows these limits, servers experience thread starvation, memory swapping, connection drops, and eventual outages.
*   **How Scaling Solves It:** Scaling increases system resources to accommodate growth. It does this either by enhancing the capacity of the current machine (Vertical) or by adding more machines to split the load (Horizontal).

---

### 2.2 Vertical Scaling (Scaling Up)
Vertical scaling is the process of upgrading a single physical server by adding more resources (e.g., adding 64GB RAM, upgrading to a 32-core CPU).

#### 2.2.1 Pros and Cons of Vertical Scaling
*   **Pros:**
    *   **Zero Code Modifications:** Easy to execute; does not require complex software refactoring or session distribution.
    *   **Strict Consistency:** Simplifies database transactions because all transactions run on a single node without distributed consensus overhead.
    *   **Ultra-Low Latency:** Inter-process communication happens inside the CPU/RAM bus, avoiding network transit overhead.
*   **Cons:**
    *   **Hard Ceiling:** Limited by hardware technology limits. You cannot scale beyond the maximum configuration of a single motherboard.
    *   **Hardware SPOF:** If the motherboard, RAM, or power supply unit fails, the entire application crashes.
    *   **Exponential Pricing:** Cost curves rise exponentially; high-end enterprise servers are far more expensive than several standard commodity servers combined.

---

### 2.3 Horizontal Scaling (Scaling Out) & Entry Load Balancers
Horizontal scaling is the process of adding more independent servers (nodes) to your resource pool to share the overall computational load.

#### 2.3.1 Why We Use a Load Balancer in Horizontal Scaling
*   **The Problem:** Simply turning on 10 new servers does not solve the scaling issue unless incoming client traffic can be split among them. If clients send requests randomly or to a single IP address, servers will experience unbalanced distribution—leaving some servers overloaded while others sit idle.
*   **How the Load Balancer Solves It:** The Load Balancer (LB) acts as a single endpoint for clients. It intercepts incoming traffic and distributes it systematically across the backend pool using health checks to bypass failed instances.

#### 2.3.2 Load Balancing Algorithms & Mechanisms (L4 vs. L7)
*   **Routing Algorithms:**
    *   *Round Robin:* Sequentially distributes requests. Best for identical hardware servers.
    *   *Least Connections:* Sends traffic to servers with the lowest active concurrent connections. Perfect for varying query complexities.
    *   *IP Hash:* Uses client IP hash to map requests to the same backend server (supports session stickiness).
*   **Layer 4 (L4) vs. Layer 7 (L7) Balancing:**
    *   *L4 (Transport Layer):* Routes traffic fast without opening the packet payload (uses TCP/UDP ports/IPs). Extremely fast.
    *   *L7 (Application Layer):* Inspects HTTP headers, cookies, and URI paths (e.g., routing `/auth` to one server and `/images` to another). Offers highly flexible routing.

#### 2.3.3 Pros and Cons of Horizontal Scaling & Load Balancing
*   **Pros:**
    *   **Theoretical Infinite Scale:** You can endlessly add standard nodes as user count rises.
    *   **High Availability:** Instantly shifts traffic away from crashed instances.
    *   **Zero-Downtime Deployments:** Allows rolling updates where servers are upgraded one by one without stopping the service.
*   **Cons:**
    *   **Complex Management:** Requires monitoring, log aggregation, and orchestration (e.g., Kubernetes).
    *   **Session Management:** Sessions must be stateless or stored in shared caches (like Redis) since requests from the same user might hit different servers.
    *   **Network Overhead:** Adds network hops and latency as data traverses from Load Balancer -> Server -> Database.

#### 2.3.4 💡 Interview Deep Dive: Consistent Hashing & High Availability
> [!IMPORTANT]
> **Q: What is Consistent Hashing and why is it preferred over simple hashing (`hash(key) % N`) when scaling server rings?**
> **A:** In standard modular hashing, if the number of servers ($N$) changes (e.g., a node fails or scales up), almost all keys are mapped to different servers. This instantly invalidates caches, triggering a database-destroying cache stampede. 
>
> **Consistent Hashing** maps both the keys and the servers to a circular ring (hash ring). A key is assigned to the next closest server in a clockwise direction. When a server is added or removed, **only a fraction of the keys ($K/N$) need to be remapped**, preserving cache hit rates and system performance.
>
> ```
>               Server A (Hash 100)
>              /                  \
>             /                    \
>   Key 2 (Hash 350)              Key 1 (Hash 120) -> Mapped to Server A
>           \                        /
>            \                      /
>             Server B (Hash 300) ───
> ```

> [!WARNING]
> **Q: How do you prevent the entry-level Load Balancer from becoming a Single Point of Failure (SPOF)?**
> **A:** Set up an **Active-Passive pair** with a shared Virtual IP (VIP). The active load balancer broadcasts its heartbeat to the passive balancer using a protocol like VRRP (Virtual Router Redundancy Protocol). If the active node stops broadcasting heartbeats, the passive node instantly assumes the Virtual IP and continues processing traffic without downtime.

---

## 3. Microservices Architecture

### 3.1 Why We Use Microservices & How It Solves the Problem
*   **The Problem:** In a massive **Monolithic Architecture**, all business modules (authentication, catalog, shipping, notifications) are packaged into a single codebase.
    *   A bug in the notification module can leak memory and crash the whole application.
    *   Scaling requires scaling the entire monolith, duplicating unused resources.
    *   Large engineering teams face bottlenecked code deployment queues and continuous merge conflicts.
*   **How Microservices Solve It:** The system divides the application into a network of small, specialized, and loosely coupled services. Each service owns its database and communicates via lightweight APIs (HTTP, gRPC, or events).

---

### 3.2 The API Gateway (Unified Entry Point)

#### 3.2.1 Why We Use an API Gateway & How It Solves Routing Problems
*   **The Problem:** With a microservice architecture, clients (e.g., mobile apps) would have to call dozens of individual downstream endpoints. This forces the client to handle authentication repeatedly, exposes internal microservice IPs to the public internet, and creates high latency due to multiple round-trips.
*   **How API Gateway Solves It:** It serves as a unified proxy front-end. The gateway intercepts all requests, handles cross-cutting concerns (authentication, SSL decryption, rate limiting), and maps public paths to internal services (e.g., `/orders` routes to the internal Orders service).

#### 3.2.2 Pros and Cons of an API Gateway
*   **Pros:**
    *   **Security Perimeter:** Shields internal microservice networks from direct exposure.
    *   **Client Abstraction:** Clients connect to one clean endpoint, decoupling clients from internal refactoring.
    *   **Efficiency:** Centralizes authentication, rate-limiting, and CORS configuration in a single gateway layer.
*   **Cons:**
    *   **SPOF Risk:** If the API Gateway fails, the entire application becomes inaccessible.
    *   **Increased Latency:** Adds an extra hop to every external request.
    *   **Configuration Bottlenecks:** Gateway routing configuration files must be constantly maintained as new services are introduced.

#### 3.2.3 💡 Interview Deep Dive: Rate Limiting & Gateway Failover
> [!IMPORTANT]
> **Q: What rate-limiting algorithms are implemented in API Gateways?**
> 1.  **Token Bucket:** A bucket holds a maximum number of tokens. Tokens are added at a constant rate. Every request consumes a token. If the bucket is empty, requests are dropped. *Allows for traffic bursts.*
> 2.  **Leaky Bucket:** Requests enter a queue (bucket) and flow out at a constant, fixed rate. If the queue overflows, requests are dropped. *Smooths out traffic spikes.*
> 3.  **Sliding Window Counter:** Tracks the count of requests made in the current window and calculates the fraction of requests from the previous window. *Highly accurate; avoids boundary limit spikes.*

---

### 3.3 Service-Specific Load Balancers (Request-Based Routing)

#### 3.3.1 Why We Use Service-Specific Load Balancers
*   **The Problem:** Once the API Gateway determines that a request for `/orders` must go to the `Order Service`, it cannot simply route it to a hardcoded IP. In a horizontal environment, the `Order Service` itself consists of 5 or 10 separate running container instances that are constantly scaling up, scaling down, or restarting.
*   **How It Solves It:** Service-specific load balancers sit between the API Gateway and the downstream service instances. They coordinate with a **Service Registry** (like Consul or Kubernetes CoreDNS) to automatically distribute the gateway's request among the currently active, healthy instances of that specific microservice.

```
                  [ API Gateway ]
                         │
        ┌────────────────┴────────────────┐
        ▼ (Rout: /users)                  ▼ (Route: /orders)
  [ User LB ]                       [ Order LB ]
    │      │                          │      │
    ▼      ▼                          ▼      ▼
[User-1] [User-2]                 [Order-1] [Order-2]
```

#### 3.3.2 How It Solves the Internal Service Routing Problem
1.  **Service Registration:** When a new `Order Service` instance spins up, it registers its IP with the Service Registry.
2.  **Health Verification:** The registry checks its health.
3.  **Load Balancing:** The service-specific load balancer queries the registry and routes the API Gateway's request to a healthy instance.

#### 3.3.3 Pros and Cons of Service-Specific Load Balancers
*   **Pros:**
    *   **Dynamic Discovery:** Seamlessly handles internal dynamic scaling without manual routing configuration.
    *   **Targeted Failover:** If an individual service instance crashes, the service's internal LB isolates the failure immediately.
*   **Cons:**
    *   **Latency Accumulation:** Adding internal load balancers or sidecar proxies (service meshes) adds milliseconds to request times.
    *   **Debugging Difficulty:** Finding where a request failed in a multi-hop internal LB structure requires complex distributed tracing.

#### 3.3.4 💡 Interview Deep Dive: Client-Side vs. Server-Side Load Balancing
> [!IMPORTANT]
> **Q: What is the difference between Server-Side and Client-Side Load Balancing in microservices?**
>
> | Feature | Server-Side Load Balancing | Client-Side Load Balancing |
> | :--- | :--- | :--- |
> | **Mechanism** | Client hits a hardware/software LB (e.g., AWS Nginx), which proxies request. | Client queries Service Registry, caches IPs, and runs LB algorithm locally. |
> | **Network Hops** | Introduces an extra hop (Client -> LB -> Instance). | Direct call (Client -> Instance). No extra network hop. |
> | **Points of Failure**| Extra hardware component that can crash (SPOF). | Failures are localized to individual client instances. |
> | **Tech Stack** | Agnostic; handled entirely in the network infrastructure. | Requires specific library integration on client-side (e.g., Netflix Ribbon). |

---

### 3.4 The Downstream Microservices

#### 3.4.1 Why We Deploy Separate Services
*   **The Design:** Microservices represent the final leaf nodes of the request journey. Each service executes its business logic independently, utilizing its isolated database schema (Database-per-Service pattern).

#### 3.4.2 Pros and Cons of Microservices Architecture
*   **Pros:**
    *   **Fault Isolation:** Memory leaks or exceptions in one service do not impact others.
    *   **Independent Deployments:** Updates to the checkout engine can be deployed multiple times a day without touching the billing or catalog engines.
    *   **Polyglot Storage:** Allows using MongoDB for catalog search, PostgreSQL for billing, and Redis for user session storage.
*   **Cons:**
    *   **Distributed Data Transactions:** Joining data across services requires implementing event-driven eventual consistency patterns (e.g., Saga Pattern), which are hard to audit.
    *   **High Complexity & Monitoring Costs:** Demands sophisticated infrastructure tooling (Kubernetes, Jaeger, Prometheus) and dedicated DevOps engineering teams.

#### 3.4.3 💡 Interview Deep Dive: Distributed Transactions & Join Patterns
> [!CAUTION]
> **Q: How do you maintain transaction consistency across multiple microservices without using performance-degrading database locks (like Two-Phase Commit / 2PC)?**
> **A:** You implement the **Saga Pattern**. A Saga is a sequence of local transactions. Each local transaction updates the database inside a single service and publishes an event. Subsequent services listen to the event and execute their local transaction. 
> 
> If a transaction in the chain fails (e.g., payment fails after shipping is reserved), the Saga orchestrator/coordinator triggers **compensating transactions** in reverse order to undo the changes (e.g., release reserved shipping).

> [!TIP]
> **Q: How do you join data from two different databases owned by different microservices?**
> *   **Option 1: API Composition (Gateway Join):** The API Gateway fetches data from Service A and Service B separately, joining them in memory. *Pros: Simple. Cons: High latency, memory overhead.*
> *   **Option 2: CQRS (Command Query Responsibility Segregation):** Create a dedicated read-only database service that subscribes to events from both Service A and Service B. When data changes, events update a pre-joined, read-optimized materialized view. *Pros: Fast queries. Cons: Eventual consistency delay.*

---

## 4. Batch Processing

### 4.1 Why We Use Batch Processing & How It Solves the Problem
*   **The Problem:** Certain critical system tasks involve processing massive volumes of data (e.g., millions of records, daily logs, bank reconciliations, monthly billing invoices). Running these complex, long-running queries directly on active transactional databases (OLTP) during business hours degrades user performance by locking database tables and consuming vital CPU/RAM resources.
*   **How Batch Processing Solves It:** It processes data in bulk offline during off-peak windows (non-interactive). It separates computation from the live transaction system using distributed data processing frameworks, ensuring that intensive calculations do not affect the real-time client request path.

### 4.2 Core Architecture & Workflow
Batch processing architectures typically employ a three-tier model:
1.  **Extract-Transform-Load (ETL) pipeline:** Data is extracted from transactional databases or log stores, transformed (cleaned, aggregated, structured), and loaded into a target system (Data Warehouse or Data Lake).
2.  **Distributed File System (HDFS / S3):** Stores massive unstructured datasets across clusters.
3.  **Compute Engine (e.g., Apache Spark, MapReduce):** Distributes the work by splitting data into chunks, processing them in parallel on different worker nodes, and aggregating the final result.

```
┌─────────────────┐       ┌──────────────────────┐       ┌─────────────────┐
│ OLTP Database   │ ────► │ Batch Compute Engine │ ────► │ Data Warehouse  │
│ (Live App Data) │       │ (Apache Spark/ETL)   │       │ (OLAP/Analytics)│
└─────────────────┘       └──────────────────────┘       └─────────────────┘
```

### 4.3 Pros and Cons of Batch Processing
*   **Pros:**
    *   **Resource & Cost Efficiency:** Job execution can be scheduled during off-peak hours (e.g., 2 AM) when system load and cloud compute costs are minimal.
    *   **High Throughput:** Bulk operations reduce database transaction overhead (like connection creation and indexing updates) compared to processing records one by one.
    *   **Fault Tolerance (Checkpointing):** If a batch process fails halfway through, the framework can resume from the last saved state or checkpoint, avoiding full execution restarts.
*   **Cons:**
    *   **Data Latency (Not Real-time):** Data is only as fresh as the last completed batch run (e.g., 24-hour latency for daily batches).
    *   **Debugging Difficulty:** Tracing errors in massive datasets requires sophisticated logging, as a single bad input row can cause a multi-hour job to fail or produce corrupt aggregate outputs.
    *   **Operational Complexity:** Requires robust orchestration tools (like Apache Airflow or Kubernetes CronJobs) to manage complex dependency graphs between jobs.

### 4.4 💡 Interview Deep Dive: Batch vs. Stream Processing (Lambda vs. Kappa)
> [!IMPORTANT]
> **Q: What is the difference between Batch and Stream Processing?**
>
> | Feature | Batch Processing | Stream Processing |
> | :--- | :--- | :--- |
> | **Data Input** | Bounded (finite datasets processed in blocks). | Unbounded (infinite data stream processed continuously). |
> | **Latency** | High (minutes to hours/days). | Low (milliseconds to seconds). |
> | **Throughput** | High (optimized for processing total volume). | Low to Medium (optimized for low-latency delivery). |
> | **Typical Tech Stack** | Hadoop, Apache Spark, Airflow, Spring Batch. | Apache Kafka, Apache Flink, Spark Streaming. |

> [!CAUTION]
> **Q: How do you design a system that requires both real-time analytical accuracy and historical completeness? (Compare Lambda vs. Kappa Architectures)**
>
> *   **Lambda Architecture:** Employs two parallel pathways:
>     1.  **Speed Layer:** A stream processing engine (e.g., Flink) that processes real-time data with low latency but potentially lower accuracy (e.g., using approximate algorithms).
>     2.  **Batch Layer:** A batch processing engine (e.g., Spark) that periodically computes highly accurate, authoritative historical views.
>     *   *Serving Layer:* Merges queries from both layers at run-time.
>     *   *Interview Critique:* Lambda is notoriously difficult to maintain because developers must write and debug the same business logic twice (once in the speed layer, once in the batch layer).
>
> *   **Kappa Architecture:** Simplifies the pipeline by removing the batch layer entirely. Everything is treated as an unbounded stream.
>     *   *How it works:* A stream processing engine (e.g., Flink) handles both real-time data and historical reprocessing (by rewinding the Kafka offset to the beginning of the topic log and replaying the stream).
>     *   *Interview Critique:* Highly preferred in modern architectures because it maintains a single codebase for logic.
