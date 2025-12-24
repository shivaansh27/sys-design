# 🗄️ Caching in System Design

> Caching is one of the most powerful techniques to improve system performance and reduce latency. Done right, it can reduce database load by **90%+** and cut response times from hundreds of milliseconds to single digits.

---

## 📌 What is Caching?

Caching is the process of **storing frequently accessed data in a fast-access layer** (like memory) so future requests for that data can be served faster — without hitting the original, slower data source (like a database or external API).

### Simple Mental Model

```
Without Cache:
User → App Server → Database (slow, ~100ms) → Response

With Cache:
User → App Server → Cache HIT (fast, ~1ms) → Response
                  → Cache MISS → Database → Store in Cache → Response
```

---

## 🧱 Where Can You Cache?

Caching exists at **multiple layers** of a system:

```
[ Browser Cache ]
       ↓
[ CDN Cache ]           ← static assets, images, HTML
       ↓
[ Load Balancer / Reverse Proxy Cache ]   ← Nginx, Varnish
       ↓
[ Application-Level Cache ]    ← in-memory, Redis, Memcached
       ↓
[ Database Query Cache ]       ← MySQL query cache, materialized views
       ↓
[ Database ]
```

Each layer serves a different purpose. A well-designed system uses **multiple caching layers** together.

---

## ⚙️ Caching Strategies

This is the most important section — interviewers love asking about this.

### 1. Cache-Aside (Lazy Loading) ⭐ Most Common

The application is responsible for loading data into the cache.

```
Read Flow:
1. App checks cache
2. Cache HIT → return data ✅
3. Cache MISS → fetch from DB → write to cache → return data

Write Flow:
1. App writes to DB
2. App invalidates (deletes) the cache entry
```

**Pros:**
- Only requested data gets cached (no wasted memory)
- Cache failures don't break the system (falls back to DB)

**Cons:**
- First request always slow (cache miss)
- Risk of stale data between DB write and cache invalidation

**Real-world use:** Twitter timelines, product pages on e-commerce

---

### 2. Write-Through

Data is written to cache AND database simultaneously on every write.

```
Write Flow:
1. App writes to Cache
2. Cache synchronously writes to DB
3. Both are always in sync ✅

Read Flow:
1. App checks cache → always fresh data
```

**Pros:**
- Cache is always consistent with DB
- No stale data problem

**Cons:**
- Write latency increases (two writes per operation)
- Cache fills with data that may never be read (write-heavy systems waste memory)

**Real-world use:** Banking systems, inventory management — anywhere consistency > speed

---

### 3. Write-Behind (Write-Back)

Data is written to cache first, and DB is updated **asynchronously** later.

```
Write Flow:
1. App writes to Cache ✅ (returns immediately)
2. Cache asynchronously flushes to DB (after delay or on eviction)
```

**Pros:**
- Very fast writes (no waiting for DB)
- Great for write-heavy workloads

**Cons:**
- Risk of data loss if cache crashes before flushing
- Complexity of async sync logic

**Real-world use:** Gaming leaderboards, analytics counters, shopping cart updates

---

### 4. Read-Through

Cache sits between app and DB. App always talks to cache only.

```
Read Flow:
1. App requests from Cache
2. Cache HIT → return ✅
3. Cache MISS → Cache fetches from DB, stores, returns to App
```

Difference from Cache-Aside: **the cache handles the DB fetch**, not the app.

**Pros:** Simpler app logic

**Cons:** First read is always slow; cache library must know DB schema

**Real-world use:** ORM-level caching (Hibernate, ActiveRecord)

---

### 🔁 Strategy Comparison Table

| Strategy | Read Performance | Write Performance | Consistency | Complexity |
|---|---|---|---|---|
| Cache-Aside | ⚡ Fast (after warmup) | Normal | Eventually consistent | Low |
| Write-Through | ⚡ Fast | 🐢 Slower | Strong | Medium |
| Write-Behind | ⚡ Fast | ⚡ Fast | Weak (risk of loss) | High |
| Read-Through | ⚡ Fast (after warmup) | Normal | Eventually consistent | Low |

---

## 🗑️ Cache Eviction Policies

When the cache is full, it needs to decide **which data to remove**. This is the eviction policy.

### LRU — Least Recently Used ⭐
Evicts the item that **hasn't been accessed for the longest time**.

```
Cache: [A, B, C, D]  ← D is oldest unused
New item E comes in → evict D → [E, A, B, C]
```
**Best for:** General-purpose caching, user sessions, feed data

---

### LFU — Least Frequently Used
Evicts the item with the **lowest access count**.

```
A accessed 10x, B accessed 2x, C accessed 1x
Cache full → evict C (least frequent)
```
**Best for:** Content recommendation, music/video streaming (popular content stays)

---

### FIFO — First In, First Out
Evicts the **oldest inserted item**, regardless of usage.

**Best for:** Simple queues, not great for caching (ignores access patterns)

---

### TTL — Time To Live
Each item has an **expiry time**. Expired items are evicted automatically.

```
cache.set("user:123", userData, TTL=300)  // expires in 5 minutes
```
**Best for:** Session tokens, OTP codes, rate limiting counters

---

## ⚔️ Redis vs Memcached

The two most popular caching systems — interviewers often ask you to compare them.

| Feature | Redis | Memcached |
|---|---|---|
| Data Structures | Strings, Lists, Sets, Hashes, Sorted Sets, Streams | Strings only |
| Persistence | ✅ Yes (RDB + AOF) | ❌ No |
| Replication | ✅ Yes (Master-Replica) | ❌ No |
| Pub/Sub | ✅ Yes | ❌ No |
| Clustering | ✅ Redis Cluster | ✅ Basic |
| Speed | Very fast | Slightly faster for simple ops |
| Use Case | Complex data, leaderboards, sessions, queues | Simple key-value, high-throughput caching |

### When to choose Redis?
- Need persistence (data survives restart)
- Complex data types (sorted sets for leaderboards, lists for queues)
- Need pub/sub messaging
- Session storage

### When to choose Memcached?
- Pure simple key-value caching
- Maximum throughput with minimal memory overhead
- Horizontal scaling is the top priority

> **In most modern systems, Redis wins.** It does everything Memcached does plus much more.

---

## 🔥 Cache Invalidation — The Hard Problem

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

Cache invalidation = deciding **when and how to remove stale data** from cache.

### Strategies:

**1. TTL-based Expiry**
Set a time limit. After X seconds, cache entry is automatically deleted.
```
Simple but can serve stale data until TTL expires.
```

**2. Event-based Invalidation**
When data changes in DB, explicitly delete the cache key.
```
user updates profile → app deletes cache["user:123"]
```

**3. Cache Versioning**
Instead of deleting, change the key on every update.
```
cache["user:123:v1"] → update → cache["user:123:v2"]
Old version naturally expires via TTL.
```

---

## 💥 Cache Problems to Know

### 1. Cache Stampede (Thundering Herd)
**Problem:** Cache entry expires. 10,000 requests hit DB simultaneously.

**Solution:**
- Use a **mutex/lock** — only one request fetches from DB, rest wait
- Use **probabilistic early expiration** — refresh cache slightly before TTL ends
- **Background refresh** — async worker refreshes cache before it expires

---

### 2. Cache Penetration
**Problem:** Requests for data that **doesn't exist** in DB or cache — every request hits the DB.

**Example:** Attacker queries `user_id = -1` repeatedly.

**Solution:**
- Cache **null results** with short TTL (`cache["user:-1"] = null, TTL=60s`)
- Use a **Bloom Filter** to check existence before querying DB

---

### 3. Cache Avalanche
**Problem:** **Many cache entries expire at the same time** → massive DB load spike.

**Solution:**
- Add **random jitter** to TTL values (`TTL = 300 + random(0, 60)`)
- Use **tiered caching** — L1 (local) + L2 (Redis) so L2 miss doesn't always hit DB

---

## 🏗️ Real-World Example: Caching a User Profile

```
Scenario: 1M users, profile viewed 100x/day each = 100M DB reads/day

Without cache: 100M queries/day → DB overloaded

With Cache-Aside + TTL:
- First request: DB fetch → store in Redis (TTL = 10 min)
- Next 99 requests: served from Redis in <1ms
- Cache hit rate: ~99% → DB receives ~1M queries/day (100x reduction)
```

**Cache Key Design:**
```
user:{user_id}:profile        → full profile object
user:{user_id}:followers      → follower count
feed:{user_id}:page:{page}    → paginated feed
```

---

## 📐 Architecture Diagram

```
                        ┌─────────────┐
                        │   Client    │
                        └──────┬──────┘
                               │
                        ┌──────▼──────┐
                        │  CDN Cache  │  ← static assets
                        └──────┬──────┘
                               │
                        ┌──────▼──────┐
                        │  App Server │
                        └──────┬──────┘
                     ┌─────────┴──────────┐
                     │                    │
              ┌──────▼──────┐    ┌────────▼────────┐
              │    Redis     │    │    Database     │
              │   (Cache)    │    │  (PostgreSQL /  │
              │              │    │    MongoDB)     │
              └─────────────┘    └────────────────┘
                  ~1ms latency        ~100ms latency
```

---

## ✅ Quick Revision — Interview Cheatsheet

| Question | Answer |
|---|---|
| Most common strategy? | Cache-Aside |
| Fastest writes? | Write-Behind |
| Strongest consistency? | Write-Through |
| Redis vs Memcached? | Redis (more features), Memcached (simpler/faster) |
| Cache full, what happens? | Eviction (LRU, LFU, TTL) |
| Data doesn't exist in DB? | Cache Penetration → use Bloom Filter |
| Many keys expire together? | Cache Avalanche → add TTL jitter |
| Key expires, 10k requests rush? | Cache Stampede → use mutex or early refresh |

---

## 🔗 Related Topics
- [Load Balancing](./load-balancing.md)
- [Sharding & Database Scaling](./sharding.md)
- [CAP Theorem](./cap-theorem.md)
- [URL Shortener Case Study](../case-studies/url-shortener.md)

---

*Part of [System Design Notes](../README.md) — by [Your Name]*
