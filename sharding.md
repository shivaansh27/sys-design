# 🔪 Sharding & Database Scaling

> When a single database can no longer handle your data or traffic — you split it. Sharding is the art of partitioning data across multiple database nodes to achieve horizontal scalability.

---

## 📌 What is Sharding?

**Sharding** (also called **horizontal partitioning**) is the process of splitting a large database into smaller, faster, more manageable pieces called **shards**. Each shard is an independent database that holds a subset of the total data.

### Simple Mental Model

```
Without Sharding:
All 1 Billion users → Single DB → 💥 Slow queries, disk full, can't scale

With Sharding:
Users A–M → Shard 1 (DB Server 1)
Users N–Z → Shard 2 (DB Server 2)
Each shard handles 500M users → fast, scalable ✅
```

---

## 🆚 Vertical vs Horizontal Scaling

Before sharding, let's understand why we need it.

### Vertical Scaling (Scale Up)
Add more power to the **same machine** — more CPU, RAM, SSD.

```
Small DB Server → Medium DB Server → Large DB Server → 💀 Hardware Limit
(4 CPU, 16GB)     (16 CPU, 64GB)     (64 CPU, 256GB)     (can't go higher)
```

**Pros:** Simple, no application changes needed

**Cons:** Has a hard ceiling, expensive, still a single point of failure

---

### Horizontal Scaling (Scale Out) ⭐
Add **more machines**, distribute data across them.

```
1 DB Server → 2 DB Servers → 4 DB Servers → 8 DB Servers → ∞
```

**Pros:** Virtually unlimited scale, cost-effective (commodity hardware), fault tolerant

**Cons:** Complex to implement, requires sharding strategy

> **Sharding enables horizontal scaling for databases.**

---

## 🗂️ Sharding Strategies

### 1. Range-Based Sharding
Data is partitioned based on a **range of values** of the shard key.

```
Shard Key: user_id

Shard 1: user_id 1        – 10,000,000
Shard 2: user_id 10000001 – 20,000,000
Shard 3: user_id 20000001 – 30,000,000
```

**Pros:**
- Simple to implement and understand
- Range queries are efficient (all data in a range is on one shard)
- Easy to add new shards for new ranges

**Cons:**
- **Hotspot problem** — if most active users have low IDs, Shard 1 gets overwhelmed
- Uneven data distribution if data isn't uniformly distributed

**Best for:** Time-series data (logs, events), ordered datasets

---

### 2. Hash-Based Sharding ⭐ Most Common
Apply a **hash function** to the shard key, then use modulo to determine the shard.

```
shard_number = hash(user_id) % number_of_shards

user_id = 12345  → hash(12345) = 89234 → 89234 % 4 = 2 → Shard 2
user_id = 99999  → hash(99999) = 45231 → 45231 % 4 = 3 → Shard 3
```

**Pros:**
- Uniform data distribution (no hotspots)
- Simple to implement

**Cons:**
- **Resharding problem** — adding a new shard changes the modulo, requiring massive data migration
- Range queries become expensive (data is scattered across shards)

**Best for:** User data, social media posts, uniform key distribution

---

### 3. Consistent Hashing ⭐⭐ Best for Dynamic Systems
A smarter version of hash-based sharding that **minimizes data movement** when adding/removing shards.

#### How it works:
1. Imagine a **ring** (hash space from 0 to 2³²)
2. Place each shard at a position on the ring (via hash of shard name)
3. Place each data key at a position on the ring (via hash of key)
4. Each key is stored on the **first shard clockwise** from its position

```
         0
      ┌──────┐
  270 │      │ 90
      │  🔵  │        🔵 = Shard positions
  180 │      │         ●  = Data key positions
      └──────┘
        180

Key K1 at position 45  → goes to Shard at position 90  (next clockwise)
Key K2 at position 200 → goes to Shard at position 270 (next clockwise)
```

#### Adding a new shard:
Only keys between the new shard and its predecessor need to move — not all keys!

```
Without Consistent Hashing: Add 1 shard → move ~75% of all data
With Consistent Hashing:    Add 1 shard → move ~1/N of all data
```

**Pros:**
- Minimal data movement on shard addition/removal
- Naturally handles node failures
- Used by Redis Cluster, Cassandra, DynamoDB

**Cons:**
- More complex to implement
- Uneven distribution possible (solved with **virtual nodes**)

---

### 4. Directory-Based Sharding
A **lookup table** (shard map) stores which shard each record belongs to.

```
Shard Map:
user_id 1–1000    → Shard A
user_id 1001–5000 → Shard B
user_id 5001+     → Shard C (recently added)
```

**Pros:** Flexible — you can move individual records between shards

**Cons:** Lookup table is a single point of failure; adds latency

---

### 🔁 Strategy Comparison

| Strategy | Even Distribution | Range Queries | Resharding Cost | Complexity |
|---|---|---|---|---|
| Range-Based | ❌ (hotspots) | ✅ Efficient | Low | Low |
| Hash-Based | ✅ | ❌ Scattered | 🔴 Very High | Low |
| Consistent Hashing | ✅ | ❌ Scattered | ✅ Minimal | High |
| Directory-Based | ✅ | ✅ | Low | Medium |

---

## 🔑 Choosing a Shard Key

The shard key is the **most critical decision** in sharding. A bad shard key causes hotspots and uneven distribution.

### Good Shard Key Properties:
- **High cardinality** — many distinct values (user_id: good, country: bad)
- **Uniform distribution** — data spreads evenly across shards
- **Frequently used in queries** — queries should target one shard, not all

### Examples:

| Use Case | Good Shard Key | Bad Shard Key |
|---|---|---|
| Social media | user_id | country (too few values) |
| E-commerce | order_id | product_category |
| Messaging | conversation_id | message_type |
| Logging | log_id (hash) | log_level (only 5 values) |

---

## 💥 Sharding Problems to Know

### 1. Hotspot / Celebrity Problem
**Problem:** A famous user (Elon Musk, Taylor Swift) generates 1000x more traffic than average users. Their shard gets overloaded.

**Solutions:**
- Add a **random suffix** to hot keys: `user:123:shard_0`, `user:123:shard_1` ... spread reads
- **Cache** hot data aggressively (Redis)
- **Dedicated shard** for celebrity accounts

---

### 2. Cross-Shard Queries (Fan-out)
**Problem:** A query needs data from multiple shards — e.g., "find all users in India"

```
SELECT * FROM users WHERE country = 'India'
→ Must query ALL shards and merge results 😬
```

**Solutions:**
- Design queries to include the shard key: `WHERE user_id = X AND country = 'India'`
- Maintain a **secondary index service** for cross-shard lookups
- Use a **denormalized read model** (duplicate data for fast reads)

---

### 3. Resharding
**Problem:** You outgrow your current shard count and need to add more shards.

With hash-based: `hash(key) % 4` → `hash(key) % 5` — almost all data moves!

**Solutions:**
- Use **Consistent Hashing** from the start
- Plan shard count ahead — start with more shards than needed (e.g., 1024 logical shards on 4 servers — later add servers without changing shard logic)

---

### 4. Cross-Shard Transactions
**Problem:** A transaction that spans multiple shards — e.g., transfer money from User A (Shard 1) to User B (Shard 2).

**Solutions:**
- **Two-Phase Commit (2PC)** — distributed transaction protocol (slow, complex)
- **Saga Pattern** — break transaction into steps, compensate on failure
- **Design to avoid** cross-shard transactions wherever possible

---

## 🆚 Sharding vs Replication

These are often confused — they solve different problems.

| | Sharding | Replication |
|---|---|---|
| **What it does** | Splits data across nodes | Copies data across nodes |
| **Solves** | Storage & write scalability | Read scalability & availability |
| **Each node has** | Different data | Same data |
| **Use for** | Too much data for one DB | Too many reads / need failover |

> **In production, you use BOTH** — shard your data AND replicate each shard.

```
Shard 1: Primary + 2 Replicas
Shard 2: Primary + 2 Replicas
Shard 3: Primary + 2 Replicas
```

---

## 🏗️ Real-World Example: Instagram's Sharding

```
Problem (2012): 1B+ photos, growing 5M photos/day — single DB can't handle it

Solution: Shard by user_id using consistent hashing
- 512 logical shards mapped to physical PostgreSQL servers
- Each photo gets ID = (shard_id | timestamp | sequence)
- Shard key embedded in every ID → no lookup table needed

Result:
- Queries always go to exactly ONE shard
- Adding servers = moving only a fraction of shards
- Scales to billions of photos with sub-millisecond lookups
```

---

## 📐 Architecture Diagram

```
                    ┌─────────────────┐
                    │   App Server    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Shard Router   │  ← knows which shard has what
                    │ (Consistent     │
                    │  Hashing Ring)  │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
┌────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
│    Shard 1      │ │    Shard 2      │ │    Shard 3      │
│  user_id: 1–33% │ │ user_id: 34–66% │ │ user_id: 67–99% │
│                 │ │                 │ │                 │
│ Primary DB      │ │ Primary DB      │ │ Primary DB      │
│ Replica DB      │ │ Replica DB      │ │ Replica DB      │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## ✅ Quick Revision — Interview Cheatsheet

| Question | Answer |
|---|---|
| What is sharding? | Splitting data across multiple DB nodes horizontally |
| Most common strategy? | Hash-based sharding |
| Best for dynamic scaling? | Consistent Hashing |
| What is a hotspot? | One shard overloaded due to skewed access patterns |
| Sharding vs Replication? | Sharding splits data, Replication copies data |
| Cross-shard queries problem? | Fan-out — must query all shards and merge |
| How to pick a shard key? | High cardinality, uniform distribution, used in queries |
| Adding shards in hash-based? | Causes massive data migration — use consistent hashing instead |
| Used by? | Cassandra, DynamoDB, Redis Cluster, MongoDB, Instagram |

---

## 🔗 Related Topics
- [Caching](./caching.md)
- [Load Balancing](./load-balancing.md)
- [CAP Theorem](./cap-theorem.md)
- [URL Shortener Case Study](../case-studies/url-shortener.md)

---

*Part of [System Design Notes](../README.md) — by [Shivansh Sharma]*
