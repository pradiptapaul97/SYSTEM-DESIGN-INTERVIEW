# System Design Request Flow & Infrastructure Guide

This guide details how a request flows through modern, distributed system infrastructure. The sections are structured in the order a client request travels—starting from DNS lookup down to individual microservice instances.

---

## 📌 The End-to-End Request Journey
When a user types a URL (e.g., `https://example.com/orders`) into their browser, the request journeys through the following infrastructure:

```
[ User Browser ]
       │
       ├─► [ 1. DNS Lookup ] ──(Translates Domain -> IP Address)
       │
       ▼
[ 2. Infrastructure Entry Load Balancer ] ──(Handles Horizontal Scaling entry)
       │
       ▼
[ 3. API Gateway ] ──(Inspects request, Auth Check, Rate Limits, and Routes path)
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
2. [Scaling](#2-scaling)
   - [2.1 Why We Use Scaling & How It Solves the Problem](#21-why-we-use-scaling--how-it-solves-the-problem)
   - [2.2 Vertical Scaling (Scaling Up)](#22-vertical-scaling-scaling-up)
     - [2.2.1 Pros and Cons of Vertical Scaling](#221-pros-and-cons-of-vertical-scaling)
   - [2.3 Horizontal Scaling (Scaling Out) & Load Balancers](#23-horizontal-scaling-scaling-out--load-balancers)
     - [2.3.1 Why We Use a Load Balancer in Horizontal Scaling](#231-why-we-use-a-load-balancer-in-horizontal-scaling)
     - [2.3.2 Load Balancing Algorithms & Mechanisms (L4 vs. L7)](#232-load-balancing-algorithms--mechanisms-l4-vs-l7)
     - [2.3.3 Pros and Cons of Horizontal Scaling & Load Balancing](#233-pros-and-cons-of-horizontal-scaling--load-balancing)
3. [Microservices Architecture](#3-microservices-architecture)
   - [3.1 Why We Use Microservices & How It Solves the Problem](#31-why-we-use-microservices--how-it-solves-the-problem)
   - [3.2 The API Gateway (Unified Entry Point)](#32-the-api-gateway-unified-entry-point)
     - [3.2.1 Why We Use an API Gateway & How It Solves Routing Problems](#321-why-we-use-an-api-gateway--how-it-solves-routing-problems)
     - [3.2.2 Pros and Cons of an API Gateway](#322-pros-and-cons-of-an-api-gateway)
   - [3.3 Service-Specific Load Balancers (Request-Based Routing)](#33-service-specific-load-balancers-request-based-routing)
     - [3.3.1 Why We Use Service-Specific Load Balancers](#331-why-we-use-service-specific-load-balancers)
     - [3.3.2 How It Solves the Internal Service Routing Problem](#332-how-it-solves-the-internal-service-routing-problem)
     - [3.3.3 Pros and Cons of Service-Specific Load Balancers](#333-pros-and-cons-of-service-specific-load-balancers)
   - [3.4 The Downstream Microservices](#34-the-downstream-microservices)
     - [3.4.1 Why We Deploy Separate Services](#341-why-we-deploy-separate-services)
     - [3.4.2 Pros and Cons of Microservices Architecture](#342-pros-and-cons-of-microservices-architecture)

---

## 1. Domain Name System (DNS)

### 1.1 Why We Use DNS & How It Solves the Problem
*   **The Problem:** Computers identify and connect to each other over the network using numerical IP addresses (such as IPv4 `192.0.2.1` or IPv6 `2001:db8::1`). Humans, however, struggle to remember string sequences of numbers. Moreover, if a server's IP address changes due to migration or hardware upgrades, hardcoded client configurations will break immediately.
*   **How DNS Solves It:** DNS acts as the "phonebook of the Internet". It translates human-friendly domain names (e.g., `google.com`) into machine-readable IP addresses dynamically, allowing servers to change IP addresses without users ever noticing.

### 1.2 How It Works (DNS Lookup Workflow)
1.  **Resolver:** The client's browser queries the Local DNS Resolver (usually managed by the ISP).
2.  **Root Server:** If not cached, the Resolver queries the Root Nameserver (`.`) to find the Top-Level Domain (TLD) server.
3.  **TLD Server:** The TLD Server (e.g., `.com` or `.org`) directs the resolver to the Authoritative Nameserver.
4.  **Authoritative Nameserver:** Holds the definitive DNS records (A, AAAA, CNAME) and returns the actual server IP.
5.  **Caching:** The IP is sent back to the browser and cached locally for a duration defined by the Time To Live (TTL).

### 1.3 Pros and Cons of DNS
*   **Pros:**
    *   **User Friendliness:** Enables human-readable domains, boosting brand identity.
    *   **Abstraction Layer:** Decouples the human name from the actual server hardware/IP.
    *   **Global Load Distribution:** Advanced DNS (GeoDNS) routes requests to the geographically closest server, reducing initial connection latency.
*   **Cons:**
    *   **Latency in Updates:** When a DNS record changes, the update might take hours (or days) to propagate globally due to deep caching policies (TTL propagation lag).
    *   **Vulnerability to Attacks:** DNS is susceptible to hijacking, cache poisoning (spoofing), and large-scale DDoS attacks (e.g., Dyn DDoS attack) that can take down major portions of the web.
    *   **Single Point of Dependency:** If the authoritative DNS servers fail, your service becomes completely unreachable, regardless of backend health.

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

### 2.3 Horizontal Scaling (Scaling Out) & Load Balancers
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
