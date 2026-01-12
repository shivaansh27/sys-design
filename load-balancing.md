# ⚖️ Load Balancing in System Design

> Load balancing is the backbone of any scalable system. It ensures no single server becomes a bottleneck — distributing traffic intelligently across multiple servers to maximize availability and throughput.

---

## 📌 What is Load Balancing?

A **Load Balancer (LB)** is a server (or service) that sits between clients and backend servers, distributing incoming requests across multiple servers based on a defined algorithm.

### Without Load Balancer
```
1000 users → Single Server → 💥 Overloaded / Single Point of Failure
```

### With Load Balancer
```
1000 users → Load Balancer → Server 1 (333 users)
                           → Server 2 (333 users)
                           → Server 3 (334 users)
```

---

## 🧱 Where Does a Load Balancer Sit?

Load balancers can exist at **multiple layers** in a system:

```
[ Users / Clients ]
        ↓
[ DNS Load Balancing ]         ← distributes by geography / IP
        ↓
[ Global Load Balancer ]       ← routes to nearest data center
        ↓
[ Layer 7 Load Balancer ]      ← HTTP-aware (Nginx, AWS ALB)
        ↓
[ Layer 4 Load Balancer ]      ← TCP/UDP level (AWS NLB)
        ↓
[ App Servers ]
        ↓
[ Internal Load Balancer ]     ← between app servers and DBs
        ↓
[ Databases / Microservices ]
```

---

## 🔀 Load Balancing Algorithms

This is what interviewers focus on most — knowing **when to use which algorithm**.

---

### 1. Round Robin ⭐ Most Common
Requests are distributed **sequentially** across servers in a cycle.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  ← cycles back
```

**Pros:** Simple, equal distribution, no overhead

**Cons:** Ignores server load — a slow Server A gets same traffic as a fast Server C

**Best for:** Servers with identical hardware and stateless requests

---

### 2. Weighted Round Robin
Same as Round Robin but **servers get traffic proportional to their weight**.

```
Server A (weight 3) → gets 3 requests
Server B (weight 1) → gets 1 request
Server C (weight 2) → gets 2 requests
```

**Best for:** Mixed hardware — a beefy server handles more traffic than a smaller one

---

### 3. Least Connections ⭐
Routes new request to the server with the **fewest active connections**.

```
Server A: 100 active connections
Server B: 45 active connections  ← next request goes here
Server C: 80 active connections
```

**Pros:** Adapts to actual server load in real time

**Cons:** Slightly more overhead (LB must track connection counts)

**Best for:** Long-lived connections (WebSockets, file uploads, video streaming)

---

### 4. Weighted Least Connections
Combines **Least Connections** with server weights. Considers both current load and server capacity.

```
Score = Active Connections / Weight
Lowest score gets the next request
```

**Best for:** Heterogeneous servers with variable request durations

---

### 5. IP Hash (Sticky Sessions)
Routes requests based on a **hash of the client's IP address** — same client always hits the same server.

```
hash(client_ip) % num_servers = server_index

Client 192.168.1.1 → always → Server B
Client 10.0.0.5    → always → Server A
```

**Pros:** Useful when server-side session state is needed

**Cons:** Uneven distribution if many users share an IP (corporate NAT); poor for scaling

**Best for:** Stateful applications, shopping carts, gaming sessions

---

### 6. Random
Pick a server **at random** for each request.

**Pros:** Zero overhead, surprisingly effective at scale (law of large numbers)

**Cons:** No guarantee of even distribution for small traffic volumes

---

### 7. Resource-Based (Adaptive)
LB queries each server for its **current resource usage** (CPU, memory) and routes to the least loaded.

**Pros:** Most intelligent distribution

**Cons:** High overhead — requires health/metrics endpoint on every server

**Best for:** CPU-intensive workloads where request cost varies a lot

---

### 🔁 Algorithm Comparison Table

| Algorithm | Tracks State | Handles Unequal Servers | Best For |
|---|---|---|---|
| Round Robin | ❌ | ❌ | Identical servers, stateless |
| Weighted Round Robin | ❌ | ✅ | Mixed hardware |
| Least Connections | ✅ | ❌ | Long-lived connections |
| Weighted Least Conn | ✅ | ✅ | Mixed hardware + variable load |
| IP Hash | ✅ | ❌ | Stateful sessions |
| Random | ❌ | ❌ | High-scale stateless |
| Resource-Based | ✅ | ✅ | CPU-intensive tasks |

---

## 🌐 L4 vs L7 Load Balancing

One of the most common interview questions — what's the difference?

### Layer 4 — Transport Layer (TCP/UDP)
Operates at the **network level**. Routes based on IP address and port only. Does NOT inspect packet content.

```
Client → LB sees: Source IP, Dest IP, Port
LB routes based on: IP + port only
Cannot see: HTTP headers, cookies, URLs
```

**Pros:** Extremely fast, low latency, simple

**Cons:** Cannot make intelligent routing decisions based on content

**Examples:** AWS Network Load Balancer (NLB), HAProxy (TCP mode)

**Use when:** Ultra-low latency needed, non-HTTP traffic (gaming, VoIP, database connections)

---

### Layer 7 — Application Layer (HTTP/HTTPS)
Operates at the **HTTP level**. Can inspect headers, URLs, cookies, and request body.

```
Client → LB sees: Full HTTP request
LB can route based on:
  - URL path  (/api → API servers, /images → media servers)
  - Headers   (Content-Type, Authorization)
  - Cookies   (session affinity)
  - HTTP method (GET vs POST)
```

**Pros:** Smart, content-aware routing; SSL termination; A/B testing; canary deployments

**Cons:** Higher latency than L4, more CPU overhead

**Examples:** AWS Application Load Balancer (ALB), Nginx, Traefik

**Use when:** HTTP/HTTPS traffic, microservices routing, API gateways

---

### L4 vs L7 — Quick Comparison

| Feature | L4 | L7 |
|---|---|---|
| Inspects content | ❌ | ✅ |
| Speed | ⚡ Faster | Slightly slower |
| SSL termination | ❌ | ✅ |
| URL-based routing | ❌ | ✅ |
| Cookie-based session | ❌ | ✅ |
| Best for | Raw throughput | Smart HTTP routing |

---

## 🏥 Health Checks

Load balancers continuously monitor backend servers and **stop sending traffic to unhealthy servers**.

### Types of Health Checks:

**1. Active Health Checks**
LB periodically sends a request (ping / HTTP GET) to each server.
```
Every 10 seconds:
LB → GET /health → Server A → 200 OK ✅ (healthy)
LB → GET /health → Server B → timeout ❌ (remove from pool)
```

**2. Passive Health Checks**
LB monitors actual traffic. If a server returns errors (5xx) consistently, it's marked unhealthy.

---

## 🔒 SSL Termination

HTTPS traffic is encrypted. Decrypting it on every app server is expensive.

**Solution:** Let the Load Balancer handle SSL decryption once, then forward plain HTTP to backend servers.

```
Client ---(HTTPS/encrypted)--→ Load Balancer ---(HTTP/plain)--→ App Servers
```

**Benefits:**
- App servers don't need SSL certificates
- Reduces CPU load on app servers
- Centralized certificate management

---

## 💥 Load Balancer as Single Point of Failure

Wait — if the LB goes down, everything goes down?

### Solution: Redundant Load Balancers

```
                    ┌─────────────────────┐
                    │   Virtual IP (VIP)  │  ← clients connect here
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
    ┌─────────▼──────────┐          ┌───────────▼────────┐
    │  Primary LB        │          │  Secondary LB       │
    │  (Active)          │◄────────►│  (Passive/Standby)  │
    └─────────┬──────────┘  heartbeat└──────────┬─────────┘
              │                                 │
    ┌─────────▼─────────────────────────────────▼─────────┐
    │              Backend Servers                         │
    └──────────────────────────────────────────────────────┘
```

If Primary LB fails → Secondary takes over the Virtual IP automatically (**failover**).

This is called **Active-Passive** setup. You can also have **Active-Active** where both LBs handle traffic simultaneously.

---

## 🏗️ Real-World Example: Load Balancing at Twitter Scale

```
Scenario: Twitter gets ~6,000 tweets/second at peak

Architecture:
1. DNS LB        → routes users to nearest data center (US / EU / Asia)
2. L7 LB (Nginx) → routes /api/tweet → Tweet Service
                 → routes /api/timeline → Timeline Service
                 → routes /media → Media Service
3. L4 LB         → distributes DB connections across read replicas
4. Health checks  → auto-removes servers during deployments

Result: Zero downtime deployments, automatic failover, 
        traffic spikes handled gracefully
```

---

## 📐 Architecture Diagram

```
                        ┌─────────────┐
                        │   Clients   │
                        └──────┬──────┘
                               │ HTTPS
                    ┌──────────▼──────────┐
                    │   Load Balancer     │
                    │  (SSL Termination)  │
                    │  Health Checks      │
                    └──────────┬──────────┘
                               │ HTTP
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐ ┌───────▼──────┐ ┌──────▼───────┐
    │   App Server 1  │ │ App Server 2 │ │ App Server 3 │
    └─────────┬───────┘ └──────┬───────┘ └──────┬───────┘
              └────────────────┼─────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │      Database       │
                    └─────────────────────┘
```

---

## ✅ Quick Revision — Interview Cheatsheet

| Question | Answer |
|---|---|
| Default/simplest algorithm? | Round Robin |
| Best for long-lived connections? | Least Connections |
| Same user, same server? | IP Hash (Sticky Sessions) |
| L4 vs L7? | L4 = fast/IP only, L7 = smart/HTTP-aware |
| LB as SPOF? | Active-Passive redundancy with Virtual IP |
| SSL handled where? | At the LB (SSL Termination) |
| How does LB know server is down? | Health Checks (active + passive) |
| Best LB for microservices routing? | L7 (Nginx / AWS ALB) |

---

## 🔗 Related Topics
- [Caching](./caching.md)
- [Sharding & Database Scaling](./sharding.md)
- [CAP Theorem](./cap-theorem.md)
- [URL Shortener Case Study](../case-studies/url-shortener.md)

---

*Part of [System Design Notes](../README.md) — by [Your Name]*
