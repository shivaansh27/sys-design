# 🔗 System Design: URL Shortener (TinyURL / Bitly)

> A URL shortener is one of the most popular system design interview questions — deceptively simple on the surface, but packed with interesting design decisions around hashing, scalability, caching, and database design.

---

## 📌 Problem Statement

Design a URL shortening service like TinyURL or Bitly that:
- Takes a long URL as input
- Returns a short URL (e.g., `short.ly/abc123`)
- Redirects users to the original URL when the short URL is visited

---

## 🧠 Step 1 — Requirements Gathering

Always clarify requirements before designing. This shows structured thinking.

### Functional Requirements
- ✅ Given a long URL, generate a unique short URL
- ✅ Redirect short URL to original long URL
- ✅ Short URLs should be as short as possible (ideally 6–8 characters)
- ✅ (Optional) Custom aliases — user picks their own short code
- ✅ (Optional) URL expiration — short URL expires after X days

### Non-Functional Requirements
- ⚡ Low latency redirects — redirection must be < 10ms
- 📈 High availability — system must be up 99.99% of the time
- 🔒 Short URLs must be unpredictable (not sequential) — security
- 📦 Scalable to billions of URLs

### Out of Scope (for this design)
- User authentication / accounts
- Analytics dashboard
- Spam detection

---

## 📊 Step 2 — Capacity Estimation

Always estimate scale before designing. This drives your architecture decisions.

### Assumptions
```
Write (shorten URL): 100 million new URLs per day
Read (redirect):     10 billion redirects per day  (100:1 read/write ratio)
URL retention:       5 years
Short code length:   6 characters
```

### Traffic Estimates
```
Writes:
  100M URLs/day ÷ 86,400 sec = ~1,160 writes/sec

Reads:
  10B redirects/day ÷ 86,400 sec = ~115,740 reads/sec
  Peak (3x):                      = ~350,000 reads/sec
```

### Storage Estimates
```
Per URL record:
  long_url:   2,048 bytes (max URL length)
  short_code: 6 bytes
  created_at: 8 bytes
  expires_at: 8 bytes
  Total:      ~2,100 bytes ≈ 2 KB per record

5 years storage:
  100M URLs/day × 365 × 5 = 182.5 Billion URLs
  182.5B × 2KB = ~365 TB total storage
```

### Cache Estimates
```
80% of reads come from 20% of URLs (Pareto principle)
Cache the hot 20%:
  100M URLs × 20% = 20M URLs
  20M × 2KB = 40 GB of cache  ← fits in Redis easily
```

---

## 🏗️ Step 3 — High-Level Design

```
Client
  │
  ├── POST /shorten  ──→ [ Write Service ] ──→ [ Database ]
  │                              │
  │                              └──→ [ Cache (Redis) ]
  │
  └── GET /{code}   ──→ [ Read Service ]  ──→ [ Cache HIT  ] ──→ Redirect 301
                                          └──→ [ Cache MISS ] ──→ [ Database ] ──→ Redirect
```

### Core Components
1. **Write Service** — generates short codes, stores URL mappings
2. **Read Service** — looks up short code, returns redirect
3. **Database** — persistent storage of URL mappings
4. **Cache (Redis)** — stores hot URLs for fast redirects
5. **Load Balancer** — distributes traffic across service instances

---

## 🔑 Step 4 — The Core Problem: Short Code Generation

This is the heart of the system. How do you generate a **unique, short, unpredictable** code?

### Option 1: MD5 / SHA256 Hash + Truncate

```
long_url = "https://www.example.com/very/long/path?query=param"
hash     = MD5(long_url) = "a9993e364706816aba3e25717850c26c"
short    = first 6 chars = "a9993e"  → short.ly/a9993e
```

**Pros:** Deterministic — same URL always gets same short code

**Cons:**
- Hash collisions — two different URLs could produce same 6 chars
- Need collision detection + retry logic
- Predictable if attacker knows the algorithm

---

### Option 2: Base62 Encoding ⭐ Most Common

Use a **unique ID generator** (like auto-increment or Snowflake ID), then encode it in Base62.

```
Base62 charset = [0-9, a-z, A-Z] = 62 characters

ID = 1,000,000  →  Base62 encode  →  "4c92"
ID = 9,999,999  →  Base62 encode  →  "FXsk"

6 characters of Base62 = 62⁶ = 56.8 Billion unique codes ✅
```

**Why Base62?**
- No special characters (URL-safe)
- Case-sensitive (more combinations in fewer chars)
- 6 chars = 56B codes → enough for years

```python
# Base62 Encoding Logic
CHARSET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

def encode(num):
    result = []
    while num > 0:
        result.append(CHARSET[num % 62])
        num //= 62
    return ''.join(reversed(result)).zfill(6)

encode(1000000)  # → "4c92"
```

**Pros:** No collisions, fast, URL-safe, easy to implement

**Cons:** Requires a unique ID source — need a distributed ID generator at scale

---

### Option 3: Pre-generate Short Codes (KGS — Key Generation Service)

A dedicated **Key Generation Service** pre-generates millions of random codes and stores them in a database. Services request a code when needed.

```
KGS pre-generates: [abc123, xy9z8k, p2q4r6, ...]
                        ↓
Write Service requests a key → KGS returns "abc123" (marks it used)
Write Service stores: abc123 → long_url
```

**Pros:**
- No collision risk
- Very fast (no computation at write time)
- Works well across distributed write services

**Cons:**
- KGS is a single point of failure (needs replication)
- Need to handle KGS crashes (pre-assigned but unused keys)

---

### 🔁 Comparison

| Method | Collision Risk | Speed | Complexity | Best For |
|---|---|---|---|---|
| MD5 + Truncate | ✅ Possible | Fast | Low | Small scale |
| Base62 Encoding | ❌ None | Fast | Medium | Most systems |
| KGS | ❌ None | Fastest | High | Large scale |

> **Recommended:** Base62 encoding with a Snowflake ID generator for most interview answers.

---

## 🗄️ Step 5 — Database Design

### Which Database?

This is a key decision — SQL or NoSQL?

```
Our access pattern:
  Write: INSERT one row (short_code, long_url, metadata)
  Read:  SELECT long_url WHERE short_code = 'abc123'

No complex joins needed.
No transactions spanning multiple records.
Need to scale to 365 TB over 5 years.
Need 350K reads/sec at peak.
```

**→ NoSQL (Cassandra or DynamoDB) wins here.**

| Factor | SQL (PostgreSQL) | NoSQL (Cassandra/DynamoDB) |
|---|---|---|
| Scale | Hard to shard | Built for horizontal scale |
| Read speed | Good with indexes | Excellent (key-value lookup) |
| Flexibility | Schema rigid | Schema flexible |
| Transactions | ✅ Strong | ❌ Eventual |
| Best for | Complex queries | Simple key-value at scale |

### Schema Design

```
Table: url_mappings

short_code  │ long_url                          │ created_at  │ expires_at  │ user_id
────────────┼───────────────────────────────────┼─────────────┼─────────────┼────────
abc123      │ https://example.com/very/long/url  │ 2026-01-01  │ 2027-01-01  │ u_891
xy9z8k      │ https://google.com/search?q=hello  │ 2026-01-02  │ NULL        │ NULL
p2q4r6      │ https://github.com/user/repo       │ 2026-01-03  │ 2026-06-01  │ u_442

Primary Key: short_code  (partition key → direct lookup, O(1))
```

---

## ⚡ Step 6 — Read Path (Redirect Flow)

This is the most critical path — must be < 10ms.

```
1. User visits: short.ly/abc123

2. Read Service checks Redis Cache:
   cache.get("abc123") → HIT → return long_url → HTTP 301 Redirect ✅ (~1ms)
                       → MISS → go to step 3

3. Read Service queries Database:
   SELECT long_url FROM url_mappings WHERE short_code = 'abc123'
   → found → store in Redis (TTL = 24hrs) → HTTP 301 Redirect (~10ms)
   → not found → return HTTP 404

HTTP 301 vs 302:
  301 Permanent Redirect → browser caches it, future requests skip our server
                         → reduces load ✅ but we lose analytics ❌
  302 Temporary Redirect → browser always hits our server
                         → enables analytics ✅ but higher load ❌
```

---

## ✍️ Step 7 — Write Path (Shorten Flow)

```
1. User sends: POST /shorten { "url": "https://example.com/long" }

2. Validate URL:
   - Is it a valid URL format?
   - Is it reachable? (optional)
   - Is it blacklisted? (spam check)

3. Check if URL already shortened:
   SELECT short_code FROM url_mappings WHERE long_url = 'https://...'
   → exists → return existing short code (deduplication)
   → not exists → go to step 4

4. Generate short code:
   id = SnowflakeIDGenerator.next()
   short_code = base62_encode(id)

5. Store in Database:
   INSERT INTO url_mappings (short_code, long_url, created_at, expires_at)

6. Return: { "short_url": "short.ly/abc123" }
```

---

## 📈 Step 8 — Scaling the System

### Scaling Reads (350K reads/sec)
```
Problem: Database can't handle 350K reads/sec

Solutions:
1. Redis Cache (40GB) → handles 80% of reads (cache hit rate ~80%)
   → DB only sees 70K reads/sec

2. Read Replicas → scale DB reads horizontally
   Primary DB (writes only) + 3-5 Read Replicas

3. CDN → cache redirects at edge for globally popular URLs
   → user in Mumbai hits Mumbai CDN, not your US servers
```

### Scaling Writes (1,160 writes/sec)
```
Problem: Single DB can't handle writes at massive scale

Solutions:
1. Shard DB by short_code (hash-based sharding)
   short_code hash % 8 → 8 DB shards, each handles ~145 writes/sec

2. Async writes → write to cache immediately, DB asynchronously
   (risk: data loss on crash — only for non-critical data)
```

### Full Scaled Architecture
```
                         ┌─────────────┐
                         │   Clients   │
                         └──────┬──────┘
                                │
                    ┌───────────▼────────────┐
                    │     Load Balancer      │
                    │  (Nginx / AWS ALB)     │
                    └───────────┬────────────┘
                                │
              ┌─────────────────┴──────────────────┐
              │                                    │
   ┌──────────▼──────────┐            ┌────────────▼──────────┐
   │    Write Service    │            │     Read Service       │
   │  (shorten URL)      │            │  (redirect URL)        │
   │  3 instances        │            │  10 instances          │
   └──────────┬──────────┘            └────────────┬──────────┘
              │                                    │
              │                         ┌──────────▼──────────┐
              │                         │   Redis Cache       │
              │                         │   (40GB, ~80% hit)  │
              │                         └──────────┬──────────┘
              │                                    │ MISS
              └──────────────┬─────────────────────┘
                             │
              ┌──────────────▼──────────────────┐
              │         Database Layer           │
              │                                 │
              │  ┌──────────┐  ┌──────────┐     │
              │  │ Shard 1  │  │ Shard 2  │ ... │
              │  │ Primary  │  │ Primary  │     │
              │  │ Replica  │  │ Replica  │     │
              │  └──────────┘  └──────────┘     │
              └─────────────────────────────────┘
```

---

## 🔒 Step 9 — Additional Considerations

### URL Expiration
```
- Store expires_at timestamp in DB
- Background cleanup job runs every hour:
  DELETE FROM url_mappings WHERE expires_at < NOW()
- Also invalidate Redis cache on expiry
```

### Custom Aliases
```
User wants: short.ly/my-brand
- Check if alias already taken
- Reserve it in DB
- Skip code generation step
- Risk: custom aliases reduce available namespace
```

### Rate Limiting
```
Prevent abuse — limit each IP/user to:
  - 100 shortens per hour (free tier)
  - 10,000 shortens per hour (paid tier)
Implementation: Redis counter with TTL
  INCR rate:ip:192.168.1.1
  EXPIRE rate:ip:192.168.1.1 3600
```

### Security
```
- Blacklist malicious/spam URLs
- Don't make short codes sequential (reveals volume)
- Use HTTPS only
- Validate URL format before storing
```

---

## ✅ Quick Revision — Interview Cheatsheet

| Question | Answer |
|---|---|
| Core data model? | short_code → long_url (key-value) |
| Best encoding? | Base62 (62⁶ = 56B codes from 6 chars) |
| SQL or NoSQL? | NoSQL (Cassandra/DynamoDB) — simple KV at scale |
| How to handle reads at scale? | Redis cache (80% hit rate) + Read Replicas |
| How to scale writes? | Shard DB by short_code hash |
| 301 vs 302 redirect? | 301 = browser caches (less load), 302 = always hits server (analytics) |
| Avoid hash collisions? | Use auto-increment ID + Base62, not MD5 truncation |
| URL expiry? | expires_at field + background cleanup job |
| Rate limiting? | Redis INCR counter per IP with TTL |
| How many codes in 6 Base62 chars? | 62⁶ = ~56.8 Billion |

---

## 🔗 Related Topics
- [Caching](../fundamentals/caching.md) — Redis for hot URL redirects
- [Load Balancing](../fundamentals/load-balancing.md) — distributing read/write traffic
- [Sharding](../fundamentals/sharding.md) — scaling the URL database
- [CAP Theorem](../fundamentals/cap-theorem.md) — AP system, eventual consistency acceptable

---

*Part of [System Design Notes](../README.md) — by [Your Name]*
