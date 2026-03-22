# 🏗️ High-Level Design (HLD) – Interview Preparation Guide

> Each question: 📌 Definition | 🔍 Why It Exists | 🏭 Real-Time Use Case (full design) | ⚙️ Production Implementation

---

## Q177. How to design a trading platform?

### 📌 Definition
A trading platform is a high-availability, low-latency, high-throughput system for submitting and executing financial orders. Core components: Order Entry (ingest), Risk Engine (pre-trade validation), Order Management System (OMS), Matching Engine (price-time priority), Market Data Service (price dissemination), Settlement Engine (post-trade), and Reporting. Designed for 99.99% uptime, sub-millisecond order processing, and regulatory compliance.

### 🏭 Architecture Diagram

```
                    ┌─────────────────────────────────────────┐
                    │           API Gateway (Ocelot/NGINX)     │
                    │   Auth | Rate Limit | Routing | SSL      │
                    └──────────────────┬──────────────────────┘
                                       │
          ┌────────────────────────────┼──────────────────────────┐
          │                            │                           │
          ▼                            ▼                           ▼
  [Order Service]              [Market Data Svc]          [Reporting Svc]
  PostgreSQL | Redis           TimescaleDB | Redis         Data Warehouse
          │                            │                           │
          ▼                            │                           │
  [Risk Engine] ←──────────────────────┘                          │
  In-memory rules                                                  │
          │                                                        │
          ▼                                                        │
  [Matching Engine]                                                │
  In-memory order book                                             │
          │                                                        │
          └──── Kafka (trade events) ────────► [Settlement Svc]   │
                                               [Notification Svc] │
                                               [Audit Svc] ───────┘
```

### 🔍 Key Design Decisions

| Component | Design choice | Why |
|---|---|---|
| **Order Entry** | REST + gRPC | REST for web clients, gRPC for internal services |
| **Risk Engine** | In-process, in-memory | Sub-ms latency — no network hop |
| **Matching Engine** | Single-threaded, in-memory priority queue | Avoid concurrency overhead, deterministic ordering |
| **Market Data** | WebSocket + Redis pub/sub | Push-based, low latency |
| **Events** | Kafka | High throughput, replay, exactly-once |
| **Storage** | PostgreSQL (OLTP), TimescaleDB (time-series), Redis (cache) | Right tool for each access pattern |
| **HA** | Active-passive Matching Engine + Paxos/Raft leader election | Only ONE engine processes at a time — eliminates dual execution |

---

## Q178. How to design a URL shortener?

### 📌 Definition
URL shortener converts long URLs to compact short codes and redirects. Requirements: generate unique 6-8 char codes, redirect with 301/302, handle 100M+ URLs, 100K+ redirects/sec read, 1K write/sec, analytics (click counts, geography).

### 🏭 Design

```
Write flow:
POST /shorten  → [API Server] → generate base62 code → [DB Write] + [Redis Cache]
              → return short URL: https://tr.de/aBcD3f

Read flow:
GET /aBcD3f   → [API Server] → Redis cache lookup
                 └── hit:  301 Redirect to original URL     (< 1ms)
                 └── miss: DB lookup → cache → 301 Redirect (5-10ms)

Code generation strategies:
  1. Auto-increment ID → base62 encode (1234567 → "5RVM") — guaranteed unique, predictable
  2. UUID first 8 chars → collision probability low, check DB
  3. Hash (MD5/SHA256) of long URL → first 8 chars of base62 — same URL = same code

DB schema:
  ShortCodes: (code PK, original_url, created_at, expires_at, owner_id, click_count)
  Index on code (PK), index on original_url (unique — same URL reuse)

Scale:
  Read: Redis cluster in front — 99%+ cache hit rate for popular URLs
  Write: partitioned DB by code hash, pre-generated code pool (worker fills code pool)
  Analytics: Kafka click events → Spark/Flink → analytics DB (ClickHouse)
```

---

## Q179. How to design a distributed cache?

### 📌 Definition
A distributed cache is a key-value store accessed by multiple application instances over the network. Provides: low-latency reads (< 1ms), horizontal scaling, TTL-based expiry, eviction policies (LRU, LFU), optional persistence, clustering with consistent hashing for shard distribution.

### 🏭 Distributed Cache Design

```
Cache architecture:
  - Consistent hashing ring: keys distributed across N nodes — adding/removing node rebalances minimum keys
  - Replication factor: 2-3 replicas per shard (replica set) — fault tolerance
  - Client library: hash(key) % N shards → route to correct node

Redis cluster mode:
  16384 hash slots divided among N masters
  Each master has replica(s)
  If master fails → replica promoted automatically

Eviction policies (when cache is full):
  allkeys-lru:    Evict least-recently-used key across all keys (trading prices: ✅)
  volatile-lru:   LRU only from keys with TTL set
  allkeys-random: Random eviction
  noeviction:     Reject writes (for session store: ✅ fail explicitly)

Key design for trading:
  price:{symbol}            → MARKET_PRICE (TTL: 1s, updated by feed)
  order:{orderId}           → ORDER_JSON   (TTL: 24h)
  session:{sessionId}       → SESSION_DATA (TTL: 30min, sliding)
  ratelimit:{userId}:{min}  → COUNT        (TTL: 60s)

Cache-aside pattern (most common):
  1. Check cache → hit: return
  2. Miss: read from DB
  3. Write to cache with TTL
  4. Return
```

---

## Q180. How to design a notification system?

### 📌 Definition
Notification system sends real-time messages across channels (push, email, SMS, in-app). Requirements: 10M+ notifications/day, multi-channel, template-based, user preference management, delivery tracking, retry on failure.

```
Architecture:
  [Triggering services (Order, Risk, Settlement)]
         │
         ▼ Kafka topic: notification-requests
  [Notification Service — Fan-out]
         │ reads preferences from UserSettings DB
         ├── Push queue  → [Push Worker]  → FCM/APNs
         ├── Email queue → [Email Worker] → SendGrid/SES
         ├── SMS queue   → [SMS Worker]   → Twilio
         └── InApp queue → [InApp Worker] → SignalR hub → WebSocket → Browser

Reliability:
  - At-least-once delivery via Kafka (consumer commits after send success)
  - Idempotency: DeliveryKey = {orderId}:{channel}:{eventType} — db unique key prevents duplicates
  - Retry: exponential backoff for transient failures (3rd party API down)
  - Dead-letter: after 5 retries → DLQ → operations alert

Template engine: Scriban/Handlebars
User preferences: opt-out per channel per notification type (stored in Redis for fast lookup)
```

---

## Q181. How to design a rate limiter?

### 📌 Definition
Rate limiter restricts API call rate per user/IP to prevent abuse. Algorithms: Fixed Window, Sliding Window, Token Bucket, Leaky Bucket. Must work across multiple server instances (distributed state in Redis).

```
Token Bucket in Redis (Lua script — atomic):
  local tokens = redis.call('GET', KEYS[1])
  if not tokens then
    redis.call('SET', KEYS[1], ARGV[1])     -- bucket_size - 1
    redis.call('EXPIRE', KEYS[1], ARGV[2])   -- window_seconds
    return 1                                  -- allowed
  end
  if tonumber(tokens) > 0 then
    redis.call('DECR', KEYS[1])
    return 1  -- allowed
  end
  return 0  -- rejected

ASP.NET Core built-in: see Q67
Response headers:
  X-RateLimit-Limit:     100
  X-RateLimit-Remaining: 34
  X-RateLimit-Reset:     1709000060  (Unix timestamp when window resets)
  Retry-After:           14          (seconds, on 429 response)
```

---

## Q182-Q216. Remaining HLD Questions.

### Q182. CAP Theorem
```
Consistency + Availability + Partition tolerance — pick 2:
CP: strong consistency, handles partitions, sacrifices availability (ZooKeeper, HBase, MongoDB w/majority writes)
AP: always available, handles partitions, eventually consistent (Cassandra, DynamoDB, CouchDB)
CA: consistent + available — impossible under network partition (theoretical — no real distributed system)

Trading platform:
  Order book (matching engine): CP — incorrect orders catastrophic, brief outage acceptable
  Market data (prices): AP — stale tick for 100ms acceptable, must never go down
  User sessions: AP — eventual consistency acceptable
```

### Q183. Consistent hashing
```
Problem: naive (key % N) fails on node add/remove — massive rehashing
Solution: hash ring (0 to 2^32) — nodes at positions; key maps to clockwise-nearest node
  Adding node: only rehash keys between new node and its predecessor (~k/N keys)
  Virtual nodes (vnodes): each physical node placed N times on ring — even distribution
Used by: Redis Cluster, Cassandra, Memcached, DynamoDB
```

### Q184. Database sharding
```
Horizontal partitioning: split rows across multiple DB servers (shards)
  Hash sharding:   hash(OrderId) % N → shard 0..N-1  (even distribution, no hot spots)
  Range sharding:  CreatedAt: Jan→Mar on shard-1, Apr→Jun on shard-2 (easy range queries, hot spots possible)
  Directory sharding: lookup table maps key → shard (flexible, but lookup overhead)

Challenges:
  Cross-shard transactions: avoid — design data model so transactions stay within one shard
  Cross-shard queries: scatter-gather (parallel queries + merge results)
  Rebalancing: consistent hashing minimizes data movement on shard add
```

### Q185. Read replicas
```
Primary (master): all writes
Read replicas: copies of primary (synchronous or async replication) — serve reads
  Replication lag: async replica may be up to N ms behind — acceptable for non-critical reads
  Trading use: order book writes to primary; reporting, analytics read from replica (with NOLOCK)
  EF Core:
    optionsBuilder.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
    // or: ctx.Database.SetConnectionString(replicaConnStr); for read operations
```

### Q186. Message queue vs event streaming
```
Message Queue (RabbitMQ, Azure Service Bus):
  - Point-to-point or pub/sub
  - Message deleted after consumption
  - Good for: task queues, background jobs
  - Delivery: at-least-once, exactly-once (with sessions)

Event Streaming (Kafka):
  - Append-only log, messages retained (configurable days/size)
  - Consumer groups — replay from any offset
  - Good for: event sourcing, audit log, real-time analytics
  - Delivery: at-least-once consumer, idempotent producer for exactly-once

Trading platform:
  Order placement → Kafka (replay, multiple consumers: risk, settlement, audit)
  Email notifications → Azure Service Bus queue (simple, guaranteed delivery)
```

### Q187. CDN
```
Content Delivery Network: geographically distributed edge servers that cache static content
  Static assets (JS, CSS, images): cache at CDN edge — no origin hit for most requests
  Dynamic API: CDN with cache-busting (query string version, ETag validation)
  Purge: invalidate edge cache on new deployment
  
.NET:
  Response.Headers["Cache-Control"] = "public, max-age=86400";  // 1 day CDN cache
  Response.Headers["ETag"] = fileVersion; // Conditional revalidation
```

### Q188. CQRS + Event Sourcing
```
CQRS: separate command model (write) and query model (read)
  Write: PlaceOrderCommand → OrderPlaced event stored in EventStore
  Read:  OrderSummaryQuery → denormalized read model (rebuilt from events via projection)

Event Sourcing:
  Current state = fold(initial, events)
  Event log: [OrderPlaced, MarginReserved, SubmittedToExchange, PartialFill, PartialFill, Filled]
  Benefits: complete audit, time travel, replay projections
  Challenges: event schema evolution, snapshots for performance

Snapshot: every 1000 events, save current state snapshot
  Load snapshot + events since snapshot (not entire history)
```

### Q189. Data warehouse design (reporting)
```
Star schema for trading analytics:
  Fact_Trades:    TradeId, DateKey, SymbolKey, TraderKey, Price, Quantity, TradeValue, Commission
  Dim_Date:       DateKey, Date, Year, Quarter, Month, DayOfWeek, IsMarketDay
  Dim_Symbol:     SymbolKey, Symbol, Exchange, Sector, ISIN
  Dim_Trader:     TraderKey, TraderId, Name, Desk, Region

ETL pipeline:
  [Kafka trade events] → [Spark Streaming] → [Delta Lake (raw)] → [Spark batch] → [Star schema]
  Or: [PostgreSQL OLTP] → [Debezium CDC] → [Kafka] → [Flink/Spark] → [ClickHouse (OLAP)]
```

### Q190. Load balancing algorithms
```
Round Robin:     requests distributed sequentially (same weight, same capacity)
Least Connections: route to server with fewest active connections (variable request cost)
IP Hash:         same client IP always routes to same server (session affinity)
Weighted:        servers with more capacity get proportionally more requests
Health-aware:    only route to healthy instances (k8s Service: remove unhealthy pod from endpoints)

.NET / Ocelot: QoSOptions lb policy per route
Kubernetes: kube-proxy (round robin) or Envoy/Istio (advanced)
```

### Q191. Distributed locking
```csharp
// Redis SETNX (SET if Not eXists) — distributed lock:
// Lock: SET lock:symbol:NIFTY50 <uuid> NX EX 10 (10 second TTL auto-unlock)
// Unlock: Lua script — only delete if value matches (avoid releasing another's lock)

// .NET: StackExchange.Redis or Medallion.ShovelIt:
var lockToken = Guid.NewGuid().ToString();
bool acquired = await _redis.StringSetAsync($"lock:{key}", lockToken, TimeSpan.FromSeconds(10), When.NotExists);
if (acquired)
{
    try { await DoWorkAsync(); }
    finally
    {
        var script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        await _redis.ScriptEvaluateAsync(script, new RedisKey[] { $"lock:{key}" }, new RedisValue[] { lockToken });
    }
}
```

### Q192-Q216. Further HLD Topics (Concise)

```
Q192. Designing a search engine: inverted index, tokenization, ranking (TF-IDF, BM25), Elasticsearch
Q193. Designing real-time analytics: Apache Flink / Spark Streaming, sliding window aggregations
Q194. Database connection pooling: PgBouncer (PostgreSQL), max_pool_size, pool mode (transaction vs session)
Q195. Data replication strategies: synchronous (CP, durability), async (AP, performance), semi-sync
Q196. Service degradation / graceful degradation: circuit breaker returns stale data vs error; fallback to cache
Q197. Designing a file storage system: S3-style, chunked upload (multipart), metadata in DB, content in blob
Q198. Designing a chat system: WebSocket connections, message fanout (Redis pub/sub or Kafka), message history (Cassandra)
Q199. Authorization service (OPA/Casbin): centralized policy evaluation, RBAC/ABAC, policy-as-code
Q200. Leader election: ZooKeeper, etcd, or Kubernetes leader election API — only one active instance
Q201. Two-phase commit (2PC): coordinator + participants, prepare phase + commit phase — blocking on coordinator failure
Q202. Paxos / Raft consensus: distributed agreement on log entries — Raft preferred (simpler), used by etcd/TiKV
Q203. Back-pressure: producer slows down when consumer queue full — Channel.BoundedCapacity, Kafka consumer lag monitoring
Q204. Time-series database: purpose-built for append-only time-ordered data; TimescaleDB, InfluxDB, Prometheus
Q205. WebSocket vs SSE vs Long Polling:
       WebSocket: bidirectional, full-duplex — trading dashboards, order status
       SSE: server-push, unidirectional, auto-reconnect — market price feeds
       Long Polling: fallback, higher latency, works everywhere
Q206. Zero-downtime deployment: rolling update, blue-green, canary; readiness probe prevents traffic to unready pods
Q207. Feature flags: LaunchDarkly / custom Redis flags — enable features per user/segment without deploy
Q208. Database migrations in production: backward-compatible migrations, expand-then-contract pattern
Q209. Disaster recovery: RTO (recovery time objective), RPO (recovery point objective), active-passive vs active-active
Q210. Multi-region architecture: data sovereignty, latency-based routing, conflict resolution for writes
Q211. Data privacy / GDPR: PII encryption, right-to-erasure (logical delete + anonymize), data lineage tracking
Q212. Audit logging: append-only audit log table, structured log entries, hash chaining for tamper detection
Q213. Cost optimization: right-sizing instances, spot instances for batch, reserved for baseline, auto-scaling
Q214. SLA / SLO / SLI:
       SLI = Service Level Indicator (latency p99, error rate, availability)
       SLO = Service Level Objective (p99 < 200ms, error rate < 0.1%, uptime > 99.9%)
       SLA = Service Level Agreement (contractual SLO with penalties)
       Error budget = 100% - SLO (remaining downtime/errors available before SLA breach)
Q215. Observability (3 pillars):
       Metrics (Prometheus/Grafana): quantitative (req/s, latency p99, error rate, GC pauses)
       Logs (Serilog/Elasticsearch): event records with context (correlationId, orderId)
       Traces (OpenTelemetry/Jaeger): request flow across services, latency per span
Q216. Database index design:
       Covering index: include all columns in SELECT → no table lookup (index-only scan)
       Partial/filtered index: WHERE Status='OPEN' → smaller index, faster query
       Composite index: column order matters — high-cardinality filter first
       Avoid: index on low-cardinality columns (boolean), over-indexing (slows writes)
```

---
