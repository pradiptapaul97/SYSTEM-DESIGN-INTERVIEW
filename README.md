# System Design Interview Guide

Welcome to the System Design Interview Guide. This document provides a structured, high-hierarchy overview of fundamental system design concepts, detailing **why we use them**, **how they solve problems**, and their **pros & cons**.

---

## 📌 Table of Contents
1. [Scaling](#1-scaling)
   - [1.1 Why We Use Scaling & How It Solves the Problem](#11-why-we-use-scaling--how-it-solves-the-problem)
   - [1.2 Vertical Scaling (Scaling Up)](#12-vertical-scaling-scaling-up)
   - [1.3 Horizontal Scaling (Scaling Out)](#13-horizontal-scaling-scaling-out)
   - [1.4 Comparison: Vertical vs. Horizontal Scaling](#14-comparison-vertical-vs-horizontal-scaling)
2. [Microservices Architecture](#2-microservices-architecture)
   - [2.1 Why We Use Microservices & How It Solves the Problem](#21-why-we-use-microservices--how-it-solves-the-problem)
   - [2.2 Core Characteristics](#22-core-characteristics)
   - [2.3 Pros and Cons](#23-pros-and-cons)
3. [API Gateway](#3-api-gateway)
   - [3.1 Why We Use an API Gateway & How It Solves the Problem](#31-why-we-use-an-api-gateway--how-it-solves-the-problem)
   - [3.2 Key Responsibilities & Mechanism](#32-key-responsibilities--mechanism)
   - [3.3 Pros and Cons](#33-pros-and-cons)
   - [3.4 Common Gateway Implementations](#34-common-gateway-implementations)
4. [Load Balancer](#4-load-balancer)
   - [4.1 Why We Use a Load Balancer & How It Solves the Problem](#41-why-we-use-a-load-balancer--how-it-solves-the-problem)
   - [4.2 Distribution Algorithms & Mechanism](#42-distribution-algorithms--mechanism)
   - [4.3 Layer 4 (L4) vs. Layer 7 (L7) Load Balancing](#43-layer-4-l4-vs-layer-7-l7-load-balancing)
   - [4.4 Pros and Cons](#44-pros-and-cons)

---

## 1. Scaling

### 1.1 Why We Use Scaling & How It Solves the Problem
*   **The Problem:** As applications grow in popularity, they experience a surge in user requests, concurrent sessions, and data volume. A single default server has finite resources (CPU, memory, disk I/O, network bandwidth). When traffic exceeds these resource limits, the system suffers from high latency, timeouts, or complete crashes.
*   **How Scaling Solves It:** Scaling increases the capacity of the infrastructure to handle the workload. It does this either by enhancing the performance of a single machine (Vertical) or by adding more machines to share the workload (Horizontal).

---

### 1.2 Vertical Scaling (Scaling Up)
Vertical scaling involves adding more power (faster CPU, more RAM, larger SSDs) to an existing single server or database node.

*   **Pros:**
    *   **Simplicity:** No architectural changes or code modification are required. Application logic remains identical.
    *   **Data Consistency:** Relational database management systems (RDBMS) run easily on a single node without dealing with complex distributed transactions.
    *   **Zero Network Latency:** All processing happens within a single machine, avoiding network communication overhead between instances.
*   **Cons:**
    *   **Hard Limits:** There is a physical upper bound to how much CPU and memory a single server can support.
    *   **Single Point of Failure (SPOF):** If the single server crashes or experiences hardware failure, the entire system goes offline.
    *   **Exponential Cost:** Upgrading to high-end hardware becomes exponentially expensive compared to standard commodity servers.
    *   **Downtime during Upgrades:** Upgrading hardware often requires taking the server offline temporarily.

---

### 1.3 Horizontal Scaling (Scaling Out)
Horizontal scaling involves adding more machines to the resource pool and distributing the incoming load across them.

*   **Pros:**
    *   **Infinite Scalability:** Theoretically, you can keep adding standard commodity servers indefinitely as demand grows.
    *   **High Availability & Fault Tolerance:** If one node fails, other healthy nodes in the cluster continue to serve traffic, eliminating a Single Point of Failure (SPOF).
    *   **Cost Effectiveness:** Standard commodity servers are cheap and can be dynamically provisioned/de-provisioned (elasticity).
    *   **Zero-Downtime Upgrades:** Software updates can be rolled out progressively (rolling updates) server-by-server.
*   **Cons:**
    *   **Architectural Complexity:** Requires routing, session management, and load-balancing infrastructure.
    *   **Data Consistency Challenges (CAP Theorem):** Keeping data synchronized across multiple database instances in real-time is highly complex.
    *   **Network Overhead:** Microservices or database replicas must communicate over the network, introducing network latency.

---

### 1.4 Comparison: Vertical vs. Horizontal Scaling

| Feature | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
| :--- | :--- | :--- |
| **Approach** | Upgrade existing machine resources | Add more machines to the system |
| **Max Capacity** | Hard limit (Hardware capacity ceiling) | Virtually limitless |
| **Redundancy** | None (Single Point of Failure) | Built-in redundancy and high availability |
| **Complexity** | Extremely low | High (Requires load balancers, clustering) |
| **Cost Curve** | Exponential (Premium hardware) | Linear (Commodity hardware) |
| **Data Consistency** | Easy to maintain | Complex (Requires replication, consensus) |

---

## 2. Microservices Architecture

### 2.1 Why We Use Microservices & How It Solves the Problem
*   **The Problem:** In a classic **Monolithic Architecture**, the entire codebase is bundled and deployed as a single unit. As the organization grows, developers step on each other's toes with merge conflicts. A single bug in one feature (e.g., a memory leak in the reporting module) can crash the entire application. Moreover, scaling the app requires scaling the entire monolith, which is highly inefficient.
*   **How Microservices Solve It:** Microservices break the large monolith down into a collection of small, autonomous, and loosely coupled services. Each service is dedicated to a single business capability (e.g., Auth, Product Catalog, Orders) and can be developed, deployed, and scaled independently by separate teams.

```
       ┌────────────────────────┐
       │      Client App        │
       └───────────┬────────────┘
                   │ HTTP/gRPC
                   ▼
       ┌────────────────────────┐
       │      API Gateway       │
       └─────┬─────┬──────┬─────┘
             │     │      │
     ┌───────┘     │      └───────┐
     ▼             ▼              ▼
┌──────────┐ ┌──────────┐   ┌──────────┐
│  Auth    │ │ Product  │   │  Order   │
│ Service  │ │ Service  │   │ Service  │
└──────────┘ └──────────┘   └──────────┘
```

---

### 2.2 Core Characteristics
*   **Single Responsibility:** Each service does one thing well (e.g., manages user profiles).
*   **Decentralized Data:** Each microservice owns its private database. Sharing database access directly is forbidden; communication occurs strictly via APIs (REST, gRPC, or Message Brokers).
*   **Polyglot Freedom:** Teams can choose different tech stacks (e.g., Python for AI/ML services, Go/Rust for high-performance APIs, Node.js for rapid CRUD APIs).
*   **Independent Deployability:** A change in the Billing service can be shipped to production immediately without redeploying the catalog or auth services.

---

### 2.3 Pros and Cons

*   **Pros:**
    *   **Fault Isolation:** A crash in the recommendation service does not disrupt the checkout or billing service.
    *   **Team Autonomy & Velocity:** Teams own their services end-to-end, reducing cross-team coordination bottlenecks and deployment dependencies.
    *   **Granular Scaling:** You can scale only the resource-heavy services (e.g., scaling up order processing during a sale) instead of scaling the entire monolith.
    *   **Ease of Tech Upgrades:** It is easier to rewrite a small service with modern technologies than refactoring a massive monolith.
*   **Cons:**
    *   **Operational overhead:** Requires sophisticated CI/CD pipelines, container orchestration (like Kubernetes), centralized logging (ELK stack), and distributed tracing (Jaeger/Zipkin).
    *   **Distributed Systems Complexity:** Debugging issues across service boundaries is difficult. You have to handle network latencies, retries, and cascading failures.
    *   **Data Consistency:** Maintaining transaction integrity across multiple databases requires complex patterns (e.g., the **Saga Pattern** or Event Sourcing) instead of simple database transactions.

---

## 3. API Gateway

### 3.1 Why We Use an API Gateway & How It Solves the Problem
*   **The Problem:** In a microservices environment, a single client request might require data from multiple backend services. If clients connect directly to each service:
    1.  Clients must manage multiple complex endpoints.
    2.  Each microservice must implement its own security, authentication, and rate-limiting rules.
    3.  Internal IP addresses and services are exposed to the public internet, creating a large attack surface.
*   **How API Gateway Solves It:** The API Gateway acts as a single, centralized entry point (reverse proxy) positioned between the client and the internal microservices. It intercepts all incoming requests, handles security, routes requests to their appropriate destinations, and consolidates backend responses.

---

### 3.2 Key Responsibilities & Mechanism
1.  **Request Routing:** Maps incoming public URLs to internal microservice endpoints (e.g., `/api/v1/users` -> `http://user-service:8080/`).
2.  **Centralized Authentication/Authorization:** Validates JWTs, OAuth tokens, or API keys at the edge, blocking unauthorized requests before they consume backend compute resources.
3.  **Rate Limiting & Throttling:** Restricts the number of requests a client can make in a given timeframe, protecting downstream services from abuse or DDoS attacks.
4.  **API Composition & Aggregation:** Combines results from multiple microservices and sends a single payload back to the client, reducing round-trip network times.
5.  **SSL Termination:** Decrypts SSL/TLS certificates at the gateway layer, sparing backend services the CPU overhead of encryption/decryption.

---

### 3.3 Pros and Cons

*   **Pros:**
    *   **Simplifies Client Integration:** Clients only need to communicate with one domain and IP, hiding internal refactoring.
    *   **Centralized Policies:** Security, CORS rules, logging, metrics, and rate limiting are implemented in one place rather than replicated across all microservices.
    *   **Reduced Overhead:** Offloads resource-heavy tasks like SSL decryption and token validation from microservices.
*   **Cons:**
    *   **Single Point of Failure (SPOF):** If the gateway goes down, the entire system is inaccessible. (Requires high-availability deployments behind a load balancer).
    *   **Performance Bottleneck:** Every request passes through the gateway. If not optimized, it can introduce additional latency.
    *   **Development Bottleneck:** If multiple teams must update the gateway configuration for every API change, it can slow down development cycles.

---

### 3.4 Common Gateway Implementations
*   **Kong:** Highly extensible and performant open-source gateway built on Nginx.
*   **AWS API Gateway:** Fully managed service that scales automatically and integrates with serverless architectures.
*   **KrakenD:** Ultra-high performance open-source API gateway written in Go.
*   **Spring Cloud Gateway:** API gateway built on Spring WebFlux, ideal for Java-based ecosystems.

---

## 4. Load Balancer

### 4.1 Why We Use a Load Balancer & How It Solves the Problem
*   **The Problem:** When running a horizontally scaled system, you have multiple servers running the exact same application. If clients access servers directly, some servers might become heavily overloaded while others sit idle. Additionally, if a server crashes, traffic will still be sent to it, leading to failed requests for users.
*   **How Load Balancer Solves It:** A Load Balancer (LB) acts as a traffic dispatcher. It sits in front of the server pool, accepting all incoming traffic and distributing it evenly (or based on capacity) across healthy backend servers. It performs regular **health checks** to detect failed instances and dynamically routes traffic away from them.

```
                   Incoming Traffic
                          │
                          ▼
                ┌──────────────────┐
                │  Load Balancer   │
                └─┬──────┬──────┬──┘
                  │      │      │
            ┌─────┘      │      └─────┐
            ▼            ▼            ▼
       ┌─────────┐  ┌─────────┐  ┌─────────┐
       │ Server1 │  │ Server2 │  │ Server3 │
       └─────────┘  └─────────┘  └─────────┘
```

---

### 4.2 Distribution Algorithms & Mechanism
*   **Round Robin:** Routes requests sequentially to each server in the pool. (e.g., Server 1 -> Server 2 -> Server 3 -> Server 1). Best when all backend servers have equal hardware capacity.
*   **Weighted Round Robin:** Assigns weights to servers based on their processing power. Servers with higher weights receive a larger proportion of traffic.
*   **Least Connections:** Directs incoming traffic to the server with the fewest active connections. Ideal for long-lived connections (like WebSockets or database queries).
*   **IP Hash:** Hashes the client's IP address to map it to a specific server. This guarantees that a client consistently connects to the same backend server (useful for session persistence/stickiness).

---

### 4.3 Layer 4 (L4) vs. Layer 7 (L7) Load Balancing

*   **Layer 4 (L4) Load Balancing:**
    *   Operates at the **Transport Layer** (TCP/UDP).
    *   Routes traffic based on network-level details (IP address and port number).
    *   Does not decrypt or inspect the contents of the payload.
    *   *Best for:* High-throughput, low-latency, simple TCP routing.
*   **Layer 7 (L7) Load Balancing:**
    *   Operates at the **Application Layer** (HTTP/HTTPS/gRPC).
    *   Inspects application-level data (HTTP headers, cookies, query parameters, URL path).
    *   *Best for:* Advanced routing (e.g., routing `/auth` to auth servers and `/images` to image servers), cookie-based sticky sessions, and SSL termination.

---

### 4.4 Pros and Cons

*   **Pros:**
    *   **High Availability:** Instantly routes traffic away from failing servers to healthy ones.
    *   **Resource Optimization:** Prevents single-server bottlenecks by spreading the compute load evenly.
    *   **Seamless Maintenance:** Allows administrators to take servers offline for maintenance or updates without disrupting the user experience.
    *   **Security Isolation:** Acts as a barrier between the public internet and private application servers.
*   **Cons:**
    *   **Configuration Complexity:** Requires setting up health checks, sticky sessions, SSL certificates, and connection timeouts.
    *   **Infrastructure Cost:** Dedicated software/hardware load balancers (e.g., AWS ALB, F5) add to infrastructure overhead.
    *   **Potential SPOF:** If the load balancer fails, no traffic can reach any server. (Mitigated by running active-passive or active-active load balancer clusters).
