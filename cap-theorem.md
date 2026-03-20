# 📐 CAP Theorem

> CAP Theorem is the foundation of distributed systems design. It tells you exactly what trade-offs you're making every time you choose a database or architect a distributed system.

---

## 📌 What is CAP Theorem?

Proposed by Eric Brewer in 2000, the **CAP Theorem** states:

> *A distributed system can only guarantee **2 out of 3** properties simultaneously: **Consistency**, **Availability**, and **Partition Tolerance**.*

### The Three Properties

---

### C — Consistency
Every read receives the **most recent write** or an error

```
User A writes: price = $50
User B reads:  price = $50  ✅ (always sees latest value)

NOT consistent:
User A writes: price = $50
User B reads:  price = $30  ❌ (stale value from old node)
```

> Think of it as: **all nodes see the same data at the same time.**

---

### A — Availability
Every request receives a **response** (not an error) — but it might not be the most recent data.

```
Server is under stress / partially failed:
User sends request → System always responds ✅
(even if the response has slightly stale data)
```

> Think of it as: **the system is always up and always responds.**

---

### P — Partition Tolerance
The system **continues to operate** even when network communication between nodes is lost (a "partition").

```
Node 1 ←──── network cut ────→ Node 2

Partition Tolerant system: Both nodes keep working independently ✅
Not Partition Tolerant: System shuts down ❌
```

> Think of it as: **the system survives network failures between nodes.**

---

## ⚠️ The Catch — P is NOT Optional

In any real distributed system, **network partitions WILL happen**. Networks are unreliable. Cables get cut. Servers go down. Packets get lost.

This means **Partition Tolerance is mandatory** for any distributed system.

So the real choice is:

```
When a network partition occurs, do you choose:

C + P → Consistency + Partition Tolerance
         (sacrifice Availability — return error rather than stale data)

A + P → Availability + Partition Tolerance
         (sacrifice Consistency — return stale data rather than error)
```

> **CAP is really a choice between C and A when partitions happen.**

---

## 🗂️ The Three System Types

### CP Systems — Consistency + Partition Tolerance
When a partition occurs, the system **refuses to respond** rather than return stale data.

```
Network partition detected:
→ Node 2 is out of sync with Node 1
→ System returns ERROR to client ✅ (no stale data)
→ Waits until partition is resolved before serving data
```

**Real-world examples:**
- **Apache Zookeeper** — distributed coordination, config management
- **HBase** — Hadoop database
- **MongoDB** (with strong consistency settings)
- **Redis** (in single-node / sentinel mode)
- **Etcd** — Kubernetes config store

**Use when:** Data correctness is critical — banking, inventory, distributed locks

---

### AP Systems — Availability + Partition Tolerance
When a partition occurs, the system **keeps responding** but may return stale data.

```
Network partition detected:
→ Node 2 is out of sync with Node 1
→ System still serves requests from Node 2 ✅
→ Client may get slightly outdated data
→ Nodes sync up when partition heals (eventual consistency)
```

**Real-world examples:**
- **Amazon DynamoDB** — default mode
- **Apache Cassandra** — highly available, tunable consistency
- **CouchDB** — offline-first, sync later
- **DNS** — propagation delay means stale data temporarily

**Use when:** Availability is critical, stale data is acceptable — social media, shopping carts, DNS

---

### CA Systems — Consistency + Availability
No partition tolerance — only works on a **single node or perfectly reliable network**.

```
This is essentially a traditional single-node RDBMS:
→ No network partition possible (single machine)
→ Always consistent + always available ✅
→ But cannot scale horizontally ❌
```

**Real-world examples:**
- **PostgreSQL** (single node)
- **MySQL** (single node)
- **SQLite**

> **CA systems don't really exist in distributed systems** — as soon as you add a second node, you have a network that can partition.

---

## 🔁 CP vs AP — The Core Trade-off

```
                         Network Partition Occurs
                                    │
              ┌─────────────────────┴──────────────────────┐
              │                                            │
    ┌─────────▼──────────┐                      ┌──────────▼─────────┐
    │    CP System       │                      │    AP System       │
    │                    │                      │                    │
    │ Returns ERROR ❌   │                      │ Returns STALE DATA │
    │ until partition    │                      │ until partition    │
    │ is resolved        │                      │ is resolved        │
    │                    │                      │                    │
    │ ✅ Always correct  │                      │ ✅ Always responds │
    │ ❌ May be down     │                      │ ❌ May be outdated │
    └────────────────────┘                      └────────────────────┘
         Zookeeper, HBase                       Cassandra, DynamoDB
```

---

## 🌊 Eventual Consistency

AP systems use **eventual consistency** — nodes may be out of sync temporarily, but will **converge to the same state** once the partition heals.

```
Timeline:

T=0: User writes price = $50 to Node 1
T=1: Node 2 still has price = $30 (partition ongoing)
T=2: User reads from Node 2 → gets $30 😬 (stale)
T=3: Partition heals, Node 2 syncs → price = $50 ✅
T=4: All reads return $50 (eventually consistent)
```

### Consistency Models (weakest → strongest):

| Model | Description | Example |
|---|---|---|
| **Eventual Consistency** | Will converge, no time guarantee | DynamoDB default, DNS |
| **Monotonic Read** | You never read older data than you've already seen | User session reads |
| **Read Your Writes** | You always see your own latest write | Profile updates |
| **Causal Consistency** | Causally related operations seen in order | Comments on a post |
| **Strong Consistency** | Every read sees the latest write | Banking transactions |

---

## 🏦 Real-World Examples by Domain

### Banking / Payments → CP
```
User transfers $1000:
→ Consistency required — can't show $1000 in both accounts
→ System returns error during partition (better than wrong balance)
→ Tools: PostgreSQL, Zookeeper, traditional RDBMS
```

### Social Media Feed → AP
```
User posts a photo:
→ Availability required — post must succeed instantly
→ Some followers may see it 2 seconds later (eventual consistency)
→ Stale feed is acceptable — nobody loses money
→ Tools: Cassandra, DynamoDB
```

### Shopping Cart → AP
```
User adds item to cart:
→ Availability required — cart must always work
→ Amazon famously chose AP for carts
→ Two browser tabs may briefly show different cart state
→ Conflict resolved on checkout
→ Tools: DynamoDB, CouchDB
```

### Inventory Management → CP
```
Last item in stock:
→ Consistency required — can't oversell
→ System must lock and confirm before confirming order
→ Better to show "out of stock" than sell the same item twice
→ Tools: PostgreSQL with transactions, Redis with Lua scripts
```

### Ride Sharing (Driver Location) → AP
```
Driver location updates every second:
→ Availability required — must always show nearby drivers
→ 1-2 second stale location is fine
→ Tools: Cassandra, DynamoDB
```

---

## 🔧 PACELC — The Extended Model

CAP only describes behavior during partitions. **PACELC** extends it to normal operations too:

> *If there is a **P**artition, choose between **A**vailability and **C**onsistency. **E**lse (during normal operation), choose between **L**atency and **C**onsistency.*

```
PACELC:
  P → A or C  (same as CAP)
  E → L or C  (new addition)

DynamoDB:    PA / EL  → Available during partition, low latency normally
Zookeeper:   PC / EC  → Consistent during partition, consistent normally
Cassandra:   PA / EL  → Tunable — defaults to AP/low-latency
```

> PACELC is more realistic — in normal operation, the latency vs consistency trade-off matters just as much.

---

## 🎛️ Tunable Consistency — The Real World

Modern databases like **Cassandra** and **DynamoDB** let you **tune** consistency per query:

### Cassandra Consistency Levels:
```
Write:
  ONE    → write to 1 replica (fast, low consistency)
  QUORUM → write to majority of replicas (balanced)
  ALL    → write to all replicas (slow, strong consistency)

Read:
  ONE    → read from 1 replica (fast, may be stale)
  QUORUM → read from majority (balanced)
  ALL    → read from all (slow, always latest)

The trick:
  Write QUORUM + Read QUORUM = Strong Consistency
  (majority overlap guarantees at least one node has latest write)
```

This means **CAP is not always binary** — you can tune where on the spectrum you sit.

---

## 📐 Architecture Diagram

```
         ┌─────────────────────────────────────┐
         │           CAP Triangle              │
         │                                     │
         │           Consistency               │
         │               /\                    │
         │              /  \                   │
         │             / CP \                  │
         │            /      \                 │
         │           /────────\                │
         │          /    CA    \               │
         │         /  (single   \              │
         │        /    node)     \             │
         │       /────────────────\            │
         │  Availability    Partition          │
         │      AP         Tolerance           │
         │                                     │
         │  ● CP: Zookeeper, HBase, MongoDB    │
         │  ● AP: Cassandra, DynamoDB, CouchDB │
         │  ● CA: PostgreSQL (single node)     │
         └─────────────────────────────────────┘
```

---

## ✅ Quick Revision — Interview Cheatsheet

| Question | Answer |
|---|---|
| What is CAP Theorem? | A distributed system can guarantee only 2 of: Consistency, Availability, Partition Tolerance |
| Is P optional? | No — network partitions always happen in real systems |
| Real choice in CAP? | Between C and A when a partition occurs |
| CP system behavior? | Returns error during partition — no stale data |
| AP system behavior? | Returns stale data during partition — always responds |
| What is eventual consistency? | AP systems converge to correct state after partition heals |
| CP examples? | Zookeeper, HBase, MongoDB (strong mode) |
| AP examples? | Cassandra, DynamoDB, CouchDB, DNS |
| CA examples? | Single-node PostgreSQL, MySQL (not truly distributed) |
| What is PACELC? | Extension of CAP — also considers Latency vs Consistency in normal operation |
| Banking uses? | CP — consistency over availability |
| Social media uses? | AP — availability over consistency |

---

## 🔗 Related Topics
- [Caching](./caching.md)
- [Load Balancing](./load-balancing.md)
- [Sharding & Database Scaling](./sharding.md)
- [URL Shortener Case Study](../case-studies/url-shortener.md)

---

*Part of [System Design Notes](../README.md) — by [Your Name]*
