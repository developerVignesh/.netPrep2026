# 🧩 Microservices – Interview Preparation Guide

> Each question: 📌 Definition | 🔍 Why It Exists | 🏭 Real-Time Use Case (full code) | ⚙️ Production Implementation

---

## Q152. What is microservices architecture?

### 📌 Definition
**Microservices architecture** decomposes an application into a suite of small, independently deployable services — each responsible for a specific bounded context (a well-defined business capability). Services communicate over the network (REST, gRPC, messaging), have their own dedicated data stores, and are independently deployable, scalable, and technology-heterogeneous. Contrast with a **monolith** where all business logic, data access, and infrastructure concerns are bundled into a single deployable unit.

### 🔍 Why It Exists / Problem It Solves

```
Trading Platform — Monolith vs Microservices:

MONOLITH (single deployable):
  [OrderService][MarketDataService][RiskEngine][SettlementService][ReportingService]
       └─── Single SQL Server
  Problems: deploy everything to fix one bug, scale entire app for one hot service,
            one failure can crash everything, tech stack locked, team coupling

MICROSERVICES:
  [Order Service]       → own PostgreSQL    → REST/gRPC → Gateway
  [Market Data Service] → own Redis/TimescaleDB      → WebSocket
  [Risk Engine]         → own in-memory store         → gRPC
  [Settlement Service]  → own SQL Server              → Kafka
  [Reporting Service]   → own data warehouse           → REST
  Benefits: independent deploy, independent scale, fault isolation, team autonomy
```

### 🏭 Real-Time Use Case — Trading Platform Microservices

```
Client (Browser / Mobile)
         │
         ▼
  [API Gateway — NGINX + Ocelot]          ← Single entry point
         │ Routes by path prefix
         ├─/api/orders   → [Order Service     :5001] → PostgreSQL
         ├─/api/prices   → [Market Data Svc   :5002] → Redis/WebSocket
         ├─/api/risk     → [Risk Engine        :5003] → gRPC
         └─/api/reports  → [Reporting Svc      :5004] → Data Warehouse
                 │
         [Message Bus — Azure Service Bus / Kafka]
                 │
         [Settlement Service :5005] → SQL Server
         [Notification Svc   :5006] → SendGrid/Firebase
```

```csharp
// Order Service — creates order, publishes domain event
public sealed class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly IMessageBus _bus;

    public async Task<OrderDto> PlaceOrderAsync(PlaceOrderRequest req, CancellationToken ct)
    {
        var order = new Order(Guid.NewGuid().ToString(), req.Symbol, req.Price, req.Quantity);
        await _repo.SaveAsync(order, ct);
        // Publish OrderPlaced event — other services react independently
        await _bus.PublishAsync(new OrderPlacedEvent(order.OrderId, order.Symbol, order.Price), ct);
        return new OrderDto(order.OrderId, order.Status);
    }
}
```

### ⚙️ Production-Grade Implementation

| Pattern | Purpose |
|---|---|
| **API Gateway** | Single entry, auth, rate limiting, routing |
| **Service discovery** | Kubernetes DNS, Consul — services find each other |
| **Sidecar proxy** | Istio/Envoy — mTLS, circuit breaking, observability |
| **SAGA** | Distributed transactions across services |
| **CQRS** | Separate read/write models per service |

---

## Q153. API Gateway pattern.

### 📌 Definition
An **API Gateway** is a reverse proxy that acts as the single entry point for all client requests. It aggregates cross-cutting concerns: authentication, SSL termination, rate limiting, request routing, load balancing, response transformation, correlation ID injection, and API versioning — so individual microservices don't need to implement them. Common implementations: Ocelot (.NET), NGINX, AWS API Gateway, Azure API Management, Kong.

### 🏭 Real-Time Use Case — Ocelot Configuration

```json
// ocelot.json — route configuration:
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders/{orderId}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "order-service", "Port": 5001 },
        { "Host": "order-service-2", "Port": 5001 }
      ],
      "UpstreamPathTemplate": "/orders/{orderId}",
      "UpstreamHttpMethod": [ "GET", "POST", "DELETE" ],
      "LoadBalancerOptions": { "Type": "RoundRobin" },
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": [ "orders:read" ]
      },
      "RateLimitOptions": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1s",
        "PeriodTimespan": 1,
        "Limit": 100
      }
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://api.trading.example.com"
  }
}
```

```csharp
// Program.cs — Ocelot gateway:
builder.Services.AddOcelot()
    .AddCacheManager(x => x.WithDictionaryHandle());
builder.Services.AddAuthentication("Bearer").AddJwtBearer("Bearer", opts => { /* JWT config */ });
app.UseOcelot().Wait();
```

---

## Q154. Service discovery.

### 📌 Definition
**Service discovery** allows microservices to find each other dynamically without hardcoding IP addresses. In **client-side discovery** (Consul + client library), each service registers on startup and queries Consul to resolve endpoints. In **server-side discovery** (Kubernetes), the platform's DNS and Service objects handle routing — no client library needed. Other tools: Eureka (Netflix), AWS Cloud Map, Azure Service Fabric.

```csharp
// Kubernetes (most common in production) — automatic discovery via DNS:
// Service: order-service.trading-namespace.svc.cluster.local:5001
// Pods register automatically via Kubernetes Service resource

// .NET client — resolve by Kubernetes DNS (no client-side registration needed):
builder.Services.AddHttpClient<OrderClient>(c =>
    c.BaseAddress = new Uri("http://order-service.trading-namespace:5001"));

// Consul (traditional service mesh alternative):
builder.Services.AddConsul(c => c.Address = new Uri("http://consul:8500"));
builder.Services.AddConsulRegistration(opts => {
    opts.ServiceName = "order-service";
    opts.Address = "localhost";
    opts.Port = 5001;
    opts.HealthCheckEndpoint = "/health/live";
    opts.HealthCheckInterval = "10s";
});
```

---

## Q155. Event-driven architecture – Kafka, Service Bus.

### 📌 Definition
**Event-driven architecture (EDA)** decouples microservices by having them communicate asynchronously through events published to a message broker (Kafka, Azure Service Bus, RabbitMQ). The publisher fires-and-forgets; subscribers react independently at their own pace. Enables: temporal decoupling (consumer can be down), scale (consumers can scale independently), audit log (events are append-only log), replay, and event sourcing. Kafka is a distributed, persistent, high-throughput log; Azure Service Bus provides guaranteed delivery with queues and topics.

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Confluent.Kafka;
using System.Text.Json;

namespace TradingPlatform.Messaging
{
    // ── Event contracts ────────────────────────────────────────────────────
    public record OrderPlacedEvent(string OrderId, string Symbol, decimal Price, int Quantity, DateTime OccurredAt);
    public record RiskApprovedEvent(string OrderId, decimal ApprovedValue, string RiskScore);

    // ── Kafka producer (Order Service) ────────────────────────────────────
    public sealed class KafkaOrderProducer : IAsyncDisposable
    {
        private readonly IProducer<string, string> _producer;
        private const string Topic = "order-events";

        public KafkaOrderProducer(string bootstrapServers)
        {
            var config = new ProducerConfig
            {
                BootstrapServers = bootstrapServers,
                Acks             = Acks.All,     // Wait for all replicas — no message loss
                EnableIdempotence = true,          // Exactly-once producer
                CompressionType  = CompressionType.Snappy
            };
            _producer = new ProducerBuilder<string, string>(config).Build();
        }

        public async Task PublishOrderPlacedAsync(OrderPlacedEvent ev)
        {
            var message = new Message<string, string>
            {
                Key   = ev.OrderId,                          // Key = OrderId → same partition = ordering
                Value = JsonSerializer.Serialize(ev),
                Headers = new Headers { { "eventType", System.Text.Encoding.UTF8.GetBytes("OrderPlaced") } }
            };
            var result = await _producer.ProduceAsync(Topic, message);
            Console.WriteLine($"[Kafka] Published {ev.OrderId} → Partition {result.Partition} Offset {result.Offset}");
        }

        public async ValueTask DisposeAsync() => _producer.Dispose();
    }

    // ── Kafka consumer (Risk Engine Service) as BackgroundService ─────────
    public sealed class KafkaRiskConsumer : BackgroundService
    {
        private readonly ILogger<KafkaRiskConsumer> _logger;
        private readonly string _bootstrapServers;

        public KafkaRiskConsumer(ILogger<KafkaRiskConsumer> logger, IConfiguration config)
        {
            _logger           = logger;
            _bootstrapServers = config["Kafka:BootstrapServers"]!;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            var config = new ConsumerConfig
            {
                BootstrapServers       = _bootstrapServers,
                GroupId                = "risk-engine-consumer-group",
                AutoOffsetReset        = AutoOffsetReset.Earliest,
                EnableAutoCommit       = false,  // Manual commit — ensure processing before commit
                MaxPollIntervalMs      = 300_000
            };

            using var consumer = new ConsumerBuilder<string, string>(config).Build();
            consumer.Subscribe("order-events");

            while (!stoppingToken.IsCancellationRequested)
            {
                try
                {
                    var consumeResult = consumer.Consume(stoppingToken);
                    var ev = JsonSerializer.Deserialize<OrderPlacedEvent>(consumeResult.Message.Value)!;

                    _logger.LogInformation("[Risk] Processing {OrderId} from partition {P} offset {O}",
                        ev.OrderId, consumeResult.Partition.Value, consumeResult.Offset.Value);

                    await ProcessRiskCheckAsync(ev, stoppingToken);
                    consumer.Commit(consumeResult); // Commit AFTER successful processing
                }
                catch (OperationCanceledException) { break; }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Risk processing failed");
                    // Dead-letter or retry logic here
                }
            }
        }

        private Task ProcessRiskCheckAsync(OrderPlacedEvent ev, CancellationToken ct)
        {
            // Risk validation logic
            Console.WriteLine($"[Risk] Approved {ev.OrderId}: {ev.Symbol} x{ev.Quantity} @ {ev.Price:C}");
            return Task.CompletedTask;
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Aspect | Kafka | Azure Service Bus |
|---|---|---|
| **Throughput** | Millions/sec per partition | 10K–1M msg/sec |
| **Retention** | Configurable (days/TB) | TTL-based (max 14 days) |
| **Consumers** | Consumer groups (parallel) | Competing consumers |
| **Ordering** | Per partition (key-based) | Per session |
| **Replay** | ✅ Seek to any offset | ❌ No replay after consume |
| **Best for** | High-volume event streaming | Enterprise messaging, reliability |

---

## Q156. SAGA pattern – distributed transactions.

### 📌 Definition
**SAGA** is a pattern for managing distributed transactions across microservices without a 2-phase commit. A SAGA is a sequence of local transactions; each step publishes an event triggering the next. On failure, **compensating transactions** are executed in reverse to undo completed steps. Two styles: **Choreography** (services react to events — decentralized, no coordinator) and **Orchestration** (central saga orchestrator sends commands to services and tracks state).

### 🏭 Real-Time Use Case

```csharp
// ORCHESTRATED SAGA — Order Placement:
// 1. PlaceOrder → 2. ReserveMargin → 3. SubmitToExchange → 4. ConfirmExecution
// On failure at step 3: execute compensating transactions 2→C (ReleaseMargin), 1→C (CancelOrder)

public sealed class OrderPlacementSaga
{
    private readonly IOrderService   _orders;
    private readonly IRiskService    _risk;
    private readonly IExchangeClient _exchange;

    public async Task ExecuteAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        string? orderId  = null;
        bool    marginReserved = false;

        try
        {
            // Step 1: Create order
            orderId = await _orders.CreatePendingAsync(cmd, ct);

            // Step 2: Reserve margin (compensating: ReleaseMargin)
            await _risk.ReserveMarginAsync(orderId, cmd.Price * cmd.Quantity, ct);
            marginReserved = true;

            // Step 3: Submit to exchange (compensating: CancelWithExchange)
            string exchangeRef = await _exchange.SubmitAsync(orderId, cmd, ct);

            // Step 4: Confirm
            await _orders.ConfirmAsync(orderId, exchangeRef, ct);
            Console.WriteLine($"[SAGA] Order {orderId} placed successfully");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[SAGA] Failed at step, running compensations: {ex.Message}");
            // Run compensating transactions in reverse:
            if (marginReserved && orderId != null)
                await _risk.ReleaseMarginAsync(orderId!, ct).ConfigureAwait(false);
            if (orderId != null)
                await _orders.CancelAsync(orderId!, "SAGA_ROLLBACK", ct).ConfigureAwait(false);
            throw;
        }
    }
}
// MassTransit / NServiceBus / Duende Workflow provide SAGA state machine frameworks for .NET
```

---

## Q157. Circuit Breaker pattern.

### 📌 Definition
**Circuit Breaker** prevents cascading failures: if downstream service calls fail beyond a threshold, the circuit "trips" to **Open** state and fails fast (returns immediately without calling the service) for a cooldown period. After cooldown, transitions to **Half-Open** (allows test request); if successful → **Closed** (normal); if still failing → **Open** again.

```csharp
// Polly with Resilience Pipeline (Polly v8 / .NET 8):
builder.Services.AddHttpClient<RiskClient>()
    .AddResilienceHandler("risk-pipeline", pipeline =>
    {
        // Retry: 3 times with exponential backoff
        pipeline.AddRetry(new HttpRetryStrategyOptions {
            MaxRetryAttempts = 3,
            BackoffType      = DelayBackoffType.Exponential,
            Delay            = TimeSpan.FromMilliseconds(200)
        });
        // Circuit breaker: open after 5 failures in 30s, half-open after 30s
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions {
            FailureRatio         = 0.5,               // 50% failure rate
            SamplingDuration     = TimeSpan.FromSeconds(30),
            MinimumThroughput    = 5,
            BreakDuration        = TimeSpan.FromSeconds(30),
            OnOpened = static args => {
                Console.WriteLine($"[CB] Circuit OPENED — {args.Outcome.Exception?.Message}");
                return ValueTask.CompletedTask;
            }
        });
        // Timeout per request
        pipeline.AddTimeout(TimeSpan.FromSeconds(5));
    });
```

---

## Q158. Health checks in microservices (Kubernetes).

```csharp
// Three probe types in Kubernetes:
// Liveness: is the container alive? (restart if fails)
// Readiness: ready for traffic? (remove from load balancer if fails)
// Startup: for slow-starting apps (delays liveness/readiness checks)

// k8s deployment spec:
// livenessProbe:  httpGet: { path: /health/live,  port: 8080 } initialDelaySeconds: 15 periodSeconds: 10
// readinessProbe: httpGet: { path: /health/ready, port: 8080 } initialDelaySeconds: 5  periodSeconds: 5
// startupProbe:   httpGet: { path: /health/live,  port: 8080 } failureThreshold: 30 periodSeconds: 10

// Implementation: see Q66 (health checks in ASP.NET Core)
```

---

## Q159-Q176. Remaining Microservices Questions.

```csharp
// Q159. Sidecar pattern (Istio/Envoy):
// Each pod gets a sidecar proxy container handling: mTLS, load balancing, retries, circuit breaking
// App code focuses on business logic — infrastructure concerns in sidecar
// app container ←→ sidecar proxy ←→ service mesh

// Q160. CQRS in microservices:
// Command side:  [Order Service] → Write to PostgreSQL
// Query side:    [Order Query Service] → Reads from read replica / Redis cache / Elasticsearch
// Separate scaling: query side can scale to 100 replicas; command side stays at 5

// Q161. Event sourcing:
// Store state as sequence of events, not current state snapshot
// EventStore: append-only log of OrderPlaced, OrderFilled, OrderCancelled events
// Rebuild current state by replaying events (fold)
// Benefits: full audit log, time travel (state at any point), event replay

// Q162. Outbox pattern (guaranteed event delivery):
// Problem: save to DB AND publish event — one can fail
// Solution: write event to OutboxMessages table IN SAME TRANSACTION as business data
// Background worker polls OutboxMessages and publishes to Kafka
// Idempotent consumer on Kafka side

// Q163. API composition pattern:
// Frontend calls single aggregate API → API Gateway composes calls to 3 microservices → returns combined DTO
// Avoid chatty frontend → multiple API calls

// Q164. BFF (Backend for Frontend):
// Build separate API layer per client type (mobile, web, third-party)
// Mobile BFF: compressed, battery-optimized payloads
// Web BFF: full-featured, session-based
// Third-party BFF: versioned, rate-limited, API-key secured

// Q165. Service mesh with Istio:
// Traffic control: canary deployments, A/B testing, fault injection (testing resilience)
// Observability: distributed tracing (Jaeger), metrics (Prometheus), logs (Kiali)
// Security: mTLS between all services, RBAC policies

// Q166. Distributed tracing with OpenTelemetry:
// Every request gets a trace ID propagated via HTTP headers (traceparent)
// Each service adds spans to the trace — Jaeger/Zipkin/Azure Monitor visualizes the full trace
// builder.Services.AddOpenTelemetry().WithTracing(t => t.AddAspNetCoreInstrumentation().AddOtlpExporter())

// Q167. Rate limiting per service:
// Token bucket per API key in Redis (Lua script — atomic)
// Shared across all instances without race conditions
// Kubernetes: HPA (Horizontal Pod Autoscaler) based on CPU/memory or custom metrics

// Q168. Service-to-service authentication:
// mTLS: mutual TLS certificates for both client and server identity — zero-trust
// JWT with client credentials flow: service obtains JWT from identity provider (Keycloak/AAD)
// API Key: simple but less secure — no rotation, no expiry by default

// Q169. Distributed cache with Redis:
builder.Services.AddStackExchangeRedisCache(opts => opts.Configuration = redisConn);
// IDistributedCache.GetAsync/SetAsync — shared cache across all service instances
// Redis pub/sub for real-time price broadcasting to all connected clients

// Q170. Dead letter queues (DLQ):
// Messages that fail after N retries → moved to DLQ
// Operations team monitors DLQ — investigate, fix, replay
// Azure Service Bus: DeadLetterQueue auto-created per queue/subscription
// Kafka: manual DLQ topic + consumer that reads failed messages

// Q171. Idempotency in microservices:
// Consumer processes event multiple times (at-least-once delivery) → must be idempotent
// Track processed EventIds in DB: INSERT INTO ProcessedEvents (EventId) — UNIQUE constraint
// If constraint violation: already processed → skip (idempotent)

// Q172. Canary deployments:
// Deploy new version to 5% of pods
// Monitor error rate and latency
// Gradually increase to 100% or rollback
// Kubernetes: Argo Rollouts or Istio TrafficSplit

// Q173. Chaos engineering:
// Randomly kill pods (chaos monkey), inject latency, drop packets
// Test resilience: does circuit breaker trip? Does retry recover? Does health check fail fast?
// Tools: Chaos Monkey, Gremlin, Azure Chaos Studio

// Q174. Service decomposition strategies:
// Decompose by domain (DDD bounded contexts): Order, Payment, User, Inventory
// Decompose by team: Conway's Law — org structure mirrors system architecture
// Decompose by volatility: frequently-changing services separate from stable ones

// Q175. Transactions vs eventual consistency:
// Within one service: ACID transactions (SQL + EF Core)
// Across services: eventual consistency (SAGA + compensating transactions + idempotency)
// Accept: brief window where data appears inconsistent across services
// Design for it: "Order accepted" (not "Order confirmed") — final state via events

// Q176. Microservices anti-patterns to avoid:
// ❌ Death-star: every service calls every other service
// ❌ Shared DB: two services sharing same schema — hidden coupling
// ❌ Chatty communication: 20 service calls per request — latency compounds
// ❌ Too fine-grained: 200 services for a 20-developer team — accidental complexity
// ❌ Distributed monolith: microservice deployment but monolith coupling — worst of both worlds
```

---
