# 🗄️ EF Core – Interview Preparation Guide

> Each question: 📌 Definition | 🔍 Why It Exists | 🏭 Real-Time Use Case (full code) | ⚙️ Production Implementation

---

## Q102. What is DbContext and how does it work internally?

### 📌 Definition
`DbContext` is EF Core's central unit-of-work and identity-map. It maintains: (1) a **change tracker** that tracks entities retrieved from the database, (2) a **connection** to the underlying database via `DbConnection`, (3) **DbSet<T>** properties that serve as entry points for LINQ queries, and (4) `SaveChanges()` which converts tracked changes into SQL DML (INSERT/UPDATE/DELETE) and executes them in a transaction. Internally, each DbContext instance owns a scoped `ChangeTracker`, a `Database` facade (for raw SQL and transactions), and a `Model` (compiled schema definition). DbContext is **not thread-safe** — one instance per request.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Without DbContext / ORM — manual SQL for every operation:
using var conn = new SqlConnection(connStr);
await conn.OpenAsync();
var cmd = conn.CreateCommand();
cmd.CommandText = "SELECT OrderId, Symbol, Price FROM Orders WHERE OrderId = @id";
cmd.Parameters.AddWithValue("@id", orderId);
using var reader = await cmd.ExecuteReaderAsync();
Order? order = null;
if (await reader.ReadAsync())
    order = new Order { OrderId = reader.GetString(0), Symbol = reader.GetString(1), Price = reader.GetDecimal(2) };
// Repetitive, error-prone, no change tracking, no parameterization safety

// ✅ WITH DbContext:
var order = await _context.Orders.FindAsync(orderId); // SQL generated, parameterized, tracked
order!.Price = 24600m;           // Change tracked
await _context.SaveChangesAsync(); // UPDATE Orders SET Price=24600 WHERE OrderId=...
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Microsoft.EntityFrameworkCore;
using System.ComponentModel.DataAnnotations;

namespace TradingPlatform.Data
{
    // ── Entity ─────────────────────────────────────────────────────────────
    public sealed class Order
    {
        public string   OrderId    { get; set; } = "";
        public string   Symbol     { get; set; } = "";
        public decimal  Price      { get; set; }
        public int      Quantity   { get; set; }
        public string   Status     { get; set; } = "OPEN";
        public DateTime CreatedAt  { get; set; } = DateTime.UtcNow;
        public string   TraderId   { get; set; } = "";
        // Navigation
        public ICollection<Execution> Executions { get; set; } = new List<Execution>();
    }

    public sealed class Execution
    {
        public int     Id          { get; set; }
        public string  OrderId     { get; set; } = "";
        public decimal ExecPrice   { get; set; }
        public int     Quantity    { get; set; }
        public DateTime ExecutedAt { get; set; }
        public Order   Order      { get; set; } = null!; // Navigation
    }

    // ── DbContext ──────────────────────────────────────────────────────────
    public sealed class TradingDbContext : DbContext
    {
        public TradingDbContext(DbContextOptions<TradingDbContext> opts) : base(opts) { }

        public DbSet<Order>     Orders     => Set<Order>();
        public DbSet<Execution> Executions => Set<Execution>();

        protected override void OnModelCreating(ModelBuilder mb)
        {
            mb.Entity<Order>(e =>
            {
                e.HasKey(o => o.OrderId);
                e.Property(o => o.Price).HasPrecision(18, 4);
                e.Property(o => o.Status).HasMaxLength(20).HasDefaultValue("OPEN");
                e.HasIndex(o => o.Symbol);
                e.HasIndex(o => new { o.TraderId, o.CreatedAt });
                e.HasMany(o => o.Executions)
                 .WithOne(x => x.Order)
                 .HasForeignKey(x => x.OrderId)
                 .OnDelete(DeleteBehavior.Cascade);
            });

            mb.Entity<Execution>(e =>
            {
                e.HasKey(x => x.Id);
                e.Property(x => x.ExecPrice).HasPrecision(18, 4);
            });
        }
    }

    // ── Repository using DbContext ─────────────────────────────────────────
    public sealed class OrderRepository
    {
        private readonly TradingDbContext _ctx;
        public OrderRepository(TradingDbContext ctx) => _ctx = ctx;

        public async Task<Order?> FindAsync(string orderId, CancellationToken ct)
            => await _ctx.Orders
                         .Include(o => o.Executions)
                         .AsNoTracking()            // Read-only: no change tracking overhead
                         .FirstOrDefaultAsync(o => o.OrderId == orderId, ct);

        public async Task<string> CreateAsync(Order order, CancellationToken ct)
        {
            _ctx.Orders.Add(order);                 // Add to change tracker (EntityState.Added)
            await _ctx.SaveChangesAsync(ct);        // INSERT SQL executed
            return order.OrderId;
        }

        public async Task UpdateStatusAsync(string orderId, string status, CancellationToken ct)
        {
            var order = await _ctx.Orders.FindAsync(new object[] { orderId }, ct);
            if (order is null) throw new KeyNotFoundException(orderId);
            order.Status = status;                  // Entity state → Modified
            await _ctx.SaveChangesAsync(ct);        // UPDATE SQL executed (only Status column)
        }

        public async Task<List<Order>> GetOpenOrdersBySymbolAsync(string symbol, CancellationToken ct)
            => await _ctx.Orders
                         .Where(o => o.Symbol == symbol && o.Status == "OPEN")
                         .OrderByDescending(o => o.CreatedAt)
                         .AsNoTracking()
                         .ToListAsync(ct);
    }
}
```

### ⚙️ Production-Grade Implementation

| Practice | Reason |
|---|---|
| `AsNoTracking()` for reads | 30-40% faster — no change tracker overhead |
| `SaveChangesAsync` (async) | Non-blocking DB I/O |
| One DbContext per request | Not thread-safe — `AddDbContext` is Scoped by default |
| `DbContextPool` | Reduce DbContext construction cost (`AddDbContextPool`) |
| `HasPrecision(18, 4)` on decimal | Avoid 28-digit `decimal` mapped to SQL `decimal(18,2)` precision loss |

---

## Q103. DbContext lifecycle and scoping.

### 📌 Definition
`DbContext` must be **scoped** — one instance per logical unit of work. In ASP.NET Core: one DbContext per HTTP request (registered via `AddDbContext` → Scoped lifetime). The request starts a scope; DbContext is created; all operations within the request use the same instance (shared change tracker = identity map); at request end, scope is disposed → DbContext is disposed → connection returned to pool. For background services (Singleton): never inject DbContext directly — create a scope per background operation using `IServiceScopeFactory`.

```csharp
// Registration (correct):
builder.Services.AddDbContext<TradingDbContext>(opts =>
    opts.UseSqlServer(connStr)
        .EnableDetailedErrors(builder.Environment.IsDevelopment())
        .EnableSensitiveDataLogging(builder.Environment.IsDevelopment()), // log parameter values
    ServiceLifetime.Scoped); // Default

// DbContext pooling (high-throughput APIs):
builder.Services.AddDbContextPool<TradingDbContext>(opts =>
    opts.UseSqlServer(connStr), poolSize: 128); // Reuse context instances, don't construct/dispose
// ⚠️ With pooling: DbContext.ChangeTracker is reset between uses, but OnModelCreating not re-run

// Background service — correct scoping:
public sealed class SettlementBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _factory;
    public SettlementBackgroundService(IServiceScopeFactory f) => _factory = f;
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = _factory.CreateScope();
            var ctx = scope.ServiceProvider.GetRequiredService<TradingDbContext>();
            await RunSettlementAsync(ctx, ct); // Fresh DbContext per settlement run
            await Task.Delay(TimeSpan.FromMinutes(30), ct);
        }
    }
    private Task RunSettlementAsync(TradingDbContext ctx, CancellationToken ct) => Task.CompletedTask;
}
```

---

## Q104. Migrations – code-first approach.

### 📌 Definition
**EF Core Migrations** are the code-first mechanism for evolving the database schema in sync with your C# model changes. Each migration is a C# file with `Up()` and `Down()` methods that describe schema changes (add table, add column, add index) in a provider-agnostic API. The migration history is tracked in `__EFMigrationsHistory` table. Migrations generate SQL at `dotnet ef migrations script` or apply at runtime via `dbContext.Database.MigrateAsync()`.

```csharp
// Create migration after model change:
// dotnet ef migrations add AddOrderIndexes
// dotnet ef migrations script --idempotent
// dotnet ef database update

// Migration file (auto-generated, review before applying):
public partial class AddOrderIndexes : Migration
{
    protected override void Up(MigrationBuilder mb)
    {
        mb.CreateIndex("IX_Orders_Symbol_CreatedAt", "Orders", new[] { "Symbol", "CreatedAt" },
            descending: new[] { false, true });
        mb.AddColumn<string>("Priority", "Orders", maxLength: 10, defaultValue: "NORMAL");
    }
    protected override void Down(MigrationBuilder mb)
    {
        mb.DropIndex("IX_Orders_Symbol_CreatedAt", "Orders");
        mb.DropColumn("Priority", "Orders");
    }
}

// Apply migrations at startup (dev/staging — not recommended for prod):
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<TradingDbContext>();
    await db.Database.MigrateAsync(); // Applies pending migrations
}
// Production: use CI/CD pipeline with "dotnet ef migrations script" + reviewed DBA apply
```

---

## Q105. Change tracking – how it works.

### 📌 Definition
EF Core's **ChangeTracker** monitors entity state for all non-AsNoTracking entities. When `SaveChanges()` is called, it generates SQL only for changed entities. Entity states: `Detached` (not tracked), `Unchanged` (tracked, no changes), `Modified` (tracked, at least one property changed), `Added` (new entity to INSERT), `Deleted` (marked for DELETE). The tracker uses **snapshot change detection** (compares current values vs stored snapshot) or **notification-based** (entities implementing `INotifyPropertyChanged`).

```csharp
// Understanding state transitions:
var order = await ctx.Orders.FindAsync("ORD-001"); // State: Unchanged
ctx.Entry(order!).State; // EntityState.Unchanged

order.Status = "FILLED"; // State auto-transitions to: Modified
ctx.Entry(order).State; // EntityState.Modified

var newOrder = new Order { OrderId = "ORD-NEW", Symbol = "NIFTY50", Price = 24500m };
ctx.Orders.Add(newOrder); // State: Added
ctx.Entry(newOrder).State; // EntityState.Added

ctx.Orders.Remove(order); // State: Deleted
await ctx.SaveChangesAsync(); // Generates: UPDATE + INSERT + DELETE

// AsNoTracking — explicitly avoid tracking (read-only scenarios):
var orders = await ctx.Orders.AsNoTracking().ToListAsync();
// Faster: no snapshot allocation, no ChangeTracker bookkeeping

// Detached updates (e.g., received from API, update without fetching first):
var updated = new Order { OrderId = "ORD-001", Status = "CANCELLED" };
ctx.Orders.Attach(updated);       // State: Unchanged (attached but not fetched)
ctx.Entry(updated).Property(o => o.Status).IsModified = true; // State: Modified for Status only
await ctx.SaveChangesAsync();     // UPDATE Orders SET Status='CANCELLED' WHERE OrderId='ORD-001'
// Only STATUS column updated — no N+1 fetch-then-update round trip
```

---

## Q106. EF Core global query filters.

### 📌 Definition
**Global query filters** automatically append a WHERE clause to every LINQ query for a given entity type — without requiring callers to add the filter manually. Common use: **soft deletes** (filter out `IsDeleted = true` records), **multi-tenancy** (filter by `TenantId`), **audit visibility** (filter `IsArchived`). Filters apply to both direct queries and related entity navigation loads (includes). Callers can opt out with `.IgnoreQueryFilters()`.

```csharp
// Entity with soft delete + multi-tenant:
public sealed class Order
{
    public string  OrderId  { get; set; } = "";
    public bool    IsDeleted { get; set; } = false;
    public string  TenantId  { get; set; } = "";
    // ... other properties
}

// DbContext with global filters:
public sealed class TradingDbContext : DbContext
{
    private readonly ICurrentUserService _user;
    public TradingDbContext(DbContextOptions<TradingDbContext> opts, ICurrentUserService user) : base(opts)
        => _user = user;

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // Filter 1: Multi-tenancy — always scope to current tenant
        mb.Entity<Order>().HasQueryFilter(o => o.TenantId == _user.TenantId);

        // Filter 2: Soft delete — never return deleted entities
        mb.Entity<Order>().HasQueryFilter(o => !o.IsDeleted && o.TenantId == _user.TenantId);
    }

    public DbSet<Order> Orders => Set<Order>();
}

// Usage — filter is automatically applied:
var orders = await ctx.Orders.ToListAsync();
// SQL: SELECT * FROM Orders WHERE IsDeleted=0 AND TenantId='TENANT-1'

// Bypass global filter (admin/audit queries):
var allOrders = await ctx.Orders.IgnoreQueryFilters().ToListAsync();
// SQL: SELECT * FROM Orders (no filter)

// Soft delete — mark deleted instead of physical DELETE:
public async Task SoftDeleteAsync(string orderId, CancellationToken ct)
{
    await ctx.Orders.Where(o => o.OrderId == orderId)
                    .ExecuteUpdateAsync(s => s.SetProperty(o => o.IsDeleted, true), ct);
}
```

---

## Q107. Lazy vs Eager vs Explicit loading.

### 📌 Definition
Three strategies for loading related entities (navigation properties):
- **Eager loading** (`Include()`): loads related data in a single SQL query (JOIN or subquery). Use when you always need the related data.
- **Lazy loading** (proxy or `ILazyLoader`): loads related data on first access via generated proxy classes — each navigation access fires a new SQL query. Easy but N+1 problem risk.
- **Explicit loading** (`context.Entry(entity).Collection(...).LoadAsync()`): loads related data on demand, explicitly in code. Good for conditional loading (only load executions if order has a specific status).

```csharp
// EAGER loading — one query, explicit JOIN:
var order = await ctx.Orders
    .Include(o => o.Executions)          // JOIN Executions
    .ThenInclude(x => x.Counterparty)   // JOIN Counterparties (nested)
    .Where(o => o.OrderId == id)
    .FirstOrDefaultAsync();

// LAZY loading — install: Microsoft.EntityFrameworkCore.Proxies
// builder.Services.AddDbContext<Ctx>(o => o.UseSqlServer(conn).UseLazyLoadingProxies());
// Navigation properties must be virtual:
public virtual ICollection<Execution> Executions { get; set; } = new List<Execution>(); // virtual!
// When accessed, EF proxies intercept and fire SQL:
var order = await ctx.Orders.FindAsync(id);     // SELECT * FROM Orders WHERE...
var execs = order!.Executions;                   // SELECT * FROM Executions WHERE OrderId=...  ← new query!
// ⚠️ N+1 risk: if you load 100 orders, accessing .Executions fires 100 additional queries!

// EXPLICIT loading — manual, conditional:
var order = await ctx.Orders.FindAsync(id); // Load order only
if (order!.Status == "PARTIALLY_FILLED")
{
    await ctx.Entry(order).Collection(o => o.Executions).LoadAsync(); // Load executions only when needed
}
```

### ⚙️ Production-Grade Implementation

| Strategy | SQL queries | Use when |
|---|---|---|
| Eager | 1 (JOIN) | Related data always needed |
| Lazy | 1 + N (per access) | Optional nav, read-heavy ORMs (avoid in APIs) |
| Explicit | 2+ (manual) | Conditional loading, partial loads |

**Rule:** Default to eager loading. Disable lazy loading in APIs (too easy to trigger N+1). Use explicit loading for optional related data.

---

## Q108. Raw SQL queries in EF Core.

### 📌 Definition
EF Core provides `FromSqlRaw`, `FromSqlInterpolated`, and `ExecuteSqlRaw`/`ExecuteSqlInterpolated` for executing raw SQL while still benefiting from object mapping, parameterization safety, and (for FromSql) LINQ composability. Use raw SQL for queries that cannot be expressed efficiently in LINQ (complex CTEs, window functions, stored procedures, full-text search).

```csharp
// ✅ Parameterized raw SQL queries — safe from SQL injection:
var symbol = "NIFTY50";
var minPrice = 24000m;

// FromSqlInterpolated: C# string interpolation → SQL parameters (safe)
var orders = await ctx.Orders
    .FromSqlInterpolated($"SELECT * FROM Orders WHERE Symbol = {symbol} AND Price >= {minPrice}")
    .AsNoTracking()
    .OrderByDescending(o => o.CreatedAt)
    .ToListAsync(); // LINQ composable on top of raw SQL

// ❌ FromSqlRaw with string concat — SQL injection risk!
// var bad = ctx.Orders.FromSqlRaw($"SELECT * FROM Orders WHERE Symbol = '{symbol}'"); // NEVER!

// FromSqlRaw with SqlParameter — safe:
var orders2 = await ctx.Orders
    .FromSqlRaw("SELECT * FROM Orders WHERE Symbol = {0}", symbol)
    .ToListAsync();

// ExecuteSql — for DML (UPDATE, DELETE, stored procs):
int affected = await ctx.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Orders SET Status = 'EXPIRED' WHERE CreatedAt < {DateTime.UtcNow.AddDays(-1)} AND Status = 'OPEN'");
Console.WriteLine($"{affected} orders expired");

// ExecuteUpdate / ExecuteDelete (EF Core 7+ — no change tracking overhead):
await ctx.Orders
    .Where(o => o.Status == "OPEN" && o.CreatedAt < DateTime.UtcNow.AddDays(-1))
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, "EXPIRED")); // Direct SQL UPDATE

await ctx.Orders
    .Where(o => o.IsDeleted && o.CreatedAt < DateTime.UtcNow.AddYears(-7))
    .ExecuteDeleteAsync(); // Direct SQL DELETE — no entity loading!

// Stored procedure:
var result = await ctx.Orders
    .FromSqlRaw("EXEC sp_GetOrdersByDesk @Desk", new SqlParameter("@Desk", "EQUITY"))
    .AsNoTracking()
    .ToListAsync();
```

---

## Q109. Transactions in EF Core.

### 📌 Definition
**Transactions** in EF Core ensure multiple database operations either all succeed (commit) or all fail (rollback) atomically. EF Core wraps each `SaveChanges()` call in an implicit transaction by default. For multi-step workflows requiring multiple `SaveChanges()` calls in a single atomic unit, use `context.Database.BeginTransactionAsync()`. EF Core also supports coordinating transactions across multiple DbContext instances and participating in ambient `System.Transactions.TransactionScope`.

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace TradingPlatform.Transactions
{
    public sealed class TradeSettlementService
    {
        private readonly TradingDbContext _ctx;
        public TradeSettlementService(TradingDbContext ctx) => _ctx = ctx;

        // Atomic settlement: debit one account, credit another — must be all-or-nothing
        public async Task SettleTradeAsync(string orderId, decimal settlementPrice, CancellationToken ct)
        {
            await using IDbContextTransaction tx = await _ctx.Database.BeginTransactionAsync(
                System.Data.IsolationLevel.ReadCommitted, ct);
            try
            {
                // Step 1: Mark order as settled
                var order = await _ctx.Orders.FindAsync(new[] { (object)orderId }, ct)
                    ?? throw new KeyNotFoundException(orderId);
                order.Status = "SETTLED";

                // Step 2: Create execution record
                _ctx.Executions.Add(new Execution
                {
                    OrderId     = orderId,
                    ExecPrice   = settlementPrice,
                    Quantity    = order.Quantity,
                    ExecutedAt  = DateTime.UtcNow
                });

                // Step 3: Update ledger (from a different DbSet, same transaction)
                await _ctx.Database.ExecuteSqlInterpolatedAsync(
                    $"UPDATE Ledger SET Balance = Balance - {settlementPrice * order.Quantity} WHERE AccountId = {order.TraderId}", ct);

                await _ctx.SaveChangesAsync(ct); // Both order + execution in one commit
                await tx.CommitAsync(ct);        // Commits ALL three steps

                Console.WriteLine($"Trade {orderId} settled @ {settlementPrice:C}");
            }
            catch (Exception ex)
            {
                await tx.RollbackAsync(ct);      // Rolls back ALL three steps
                Console.WriteLine($"Settlement failed for {orderId}: {ex.Message}");
                throw;
            }
        }

        // SavepointAsync — partial rollback within a transaction:
        public async Task BulkSettleWithSavepointAsync(IList<string> orderIds, CancellationToken ct)
        {
            await using var tx = await _ctx.Database.BeginTransactionAsync(ct);
            foreach (var orderId in orderIds)
            {
                await tx.CreateSavepointAsync($"before_{orderId}", ct);
                try
                {
                    var order = await _ctx.Orders.FindAsync(new[] { (object)orderId }, ct);
                    order!.Status = "SETTLED";
                    await _ctx.SaveChangesAsync(ct);
                }
                catch
                {
                    await tx.RollbackToSavepointAsync($"before_{orderId}", ct); // Skip failed, continue
                }
            }
            await tx.CommitAsync(ct);
        }
    }

    // Supporting types referenced above:
    public record Execution { public int Id; public string OrderId = ""; public decimal ExecPrice; public int Quantity; public DateTime ExecutedAt; public Order Order = null!; }
    public sealed class Order { public string OrderId = ""; public string Symbol = ""; public decimal Price; public int Quantity; public string Status = ""; public string TraderId = ""; public ICollection<Execution> Executions = new List<Execution>(); }
    public sealed class TradingDbContext : DbContext {
        public TradingDbContext(DbContextOptions<TradingDbContext> o) : base(o) {}
        public DbSet<Order> Orders => Set<Order>();
        public DbSet<Execution> Executions => Set<Execution>();
    }
}
```

### ⚙️ Production-Grade Implementation

| Isolation Level | Use case | Risk |
|---|---|---|
| `ReadUncommitted` | Dirty reads OK (dashboard stats) | Reads uncommitted data |
| `ReadCommitted` (default SQL Server) | Standard OLTP | Non-repeatable reads possible |
| `RepeatableRead` | Prevent reads from changing mid-transaction | Higher contention |
| `Serializable` | Maximum isolation | Highest contention, slowest |
| `Snapshot` (SQL Server) | Read-consistent without blocking writers | Tempdb overhead |

---

## Q110-Q151. Remaining EF Core Questions.

### Q110. N+1 problem and how to solve it

```csharp
// ❌ N+1: load orders, then access navigation per order — N separate SELECT queries
var orders = await ctx.Orders.ToListAsync();
foreach (var o in orders)
    Console.WriteLine(o.Executions.Count); // N lazy-load queries!

// ✅ Fix 1: Include (eager loading)
var orders2 = await ctx.Orders.Include(o => o.Executions).ToListAsync(); // 1 query

// ✅ Fix 2: Projection — only load what you need
var summary = await ctx.Orders
    .Select(o => new { o.OrderId, ExecCount = o.Executions.Count() })
    .ToListAsync(); // 1 SQL with subquery COUNT
```

### Q111. EF Core projections and DTOs

```csharp
// Project to DTO — avoids loading full entity and related data:
var dtos = await ctx.Orders
    .Where(o => o.Status == "FILLED")
    .Select(o => new OrderSummaryDto(
        o.OrderId, o.Symbol, o.Price,
        o.Executions.Sum(x => x.ExecPrice) / o.Executions.Count(),  // AVG exec price
        o.Executions.Count()))
    .AsNoTracking()
    .ToListAsync();
// SQL: single query with computed columns — no Order entity, no Execution entities in RAM
public record OrderSummaryDto(string Id, string Symbol, decimal Price, double AvgExecPrice, int ExecCount);
```

### Q112. Value objects and owned entities

```csharp
// Owned entity — value object stored in same table (no separate FK table):
public record Money(decimal Amount, string Currency);
public sealed class Order
{
    public string OrderId { get; set; } = "";
    public Money  TradeValue { get; set; } = new(0, "INR");
}
// DbContext:
mb.Entity<Order>().OwnsOne(o => o.TradeValue, m => {
    m.Property(x => x.Amount).HasPrecision(18, 4).HasColumnName("TradeValue_Amount");
    m.Property(x => x.Currency).HasMaxLength(3).HasColumnName("TradeValue_Currency");
});
// Stored inline in Orders table: TradeValue_Amount, TradeValue_Currency columns
```

### Q113. EF Core interceptors

```csharp
// Intercept DB commands for logging, auditing, retry:
public sealed class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData data, InterceptionResult<int> result, CancellationToken ct)
    {
        var ctx = data.Context!;
        foreach (var entry in ctx.ChangeTracker.Entries().Where(e => e.State == EntityState.Modified))
            Console.WriteLine($"Modifying: {entry.Entity.GetType().Name} [{entry.State}]");
        return base.SavingChangesAsync(data, result, ct);
    }
}
// Register: builder.Services.AddDbContext<TradingDbContext>(o => o.UseSqlServer(conn).AddInterceptors(new AuditInterceptor()));
```

### Q114. Concurrency tokens (optimistic concurrency)

```csharp
public sealed class Order
{
    public string  OrderId  { get; set; } = "";
    [Timestamp]
    public byte[]  RowVersion { get; set; } = null!; // SQL Server rowversion — auto updated
}
// Or: mb.Entity<Order>().Property(o => o.RowVersion).IsRowVersion()
// EF Core adds: WHERE OrderId='ORD-001' AND RowVersion=<original version>
// If another user updated between read and save → DbUpdateConcurrencyException
try { await ctx.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException ex)
{
    // Resolve: let user decide, auto-retry, prefer client/server values
}
```

### Q115. EF Core compiled queries

```csharp
// Compiled queries — parse/translate LINQ once, re-execute fast:
private static readonly Func<TradingDbContext, string, Task<Order?>> FindOrderQuery
    = EF.CompileAsyncQuery((TradingDbContext ctx, string id) =>
        ctx.Orders.AsNoTracking().FirstOrDefault(o => o.OrderId == id));

// Usage — skips LINQ translation on every call:
var order = await FindOrderQuery(_ctx, "ORD-001");
// 5-30% faster in hot paths (no expression tree recompilation)
```

### Q116. Bulk insert / ExecuteUpdate / ExecuteDelete

```csharp
// EF Core 7+: server-side bulk operations — no entity tracking:
// Bulk update:
await ctx.Orders
    .Where(o => o.CreatedAt < DateTime.UtcNow.AddDays(-30) && o.Status == "OPEN")
    .ExecuteUpdateAsync(s => s
        .SetProperty(o => o.Status, "EXPIRED")
        .SetProperty(o => o.UpdatedAt, DateTime.UtcNow));

// Bulk delete:
await ctx.Orders
    .Where(o => o.IsDeleted && o.CreatedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync();

// Bulk insert — no built-in EF bulk insert; use EFCore.BulkExtensions NuGet:
// await ctx.BulkInsertAsync(orders);                       // Single round-trip INSERT
// await ctx.BulkInsertOrUpdateAsync(orders);               // UPSERT (MERGE statement)
```

### Q117. JSON columns (EF Core 7+)

```csharp
public sealed class Order
{
    public string OrderId { get; set; } = "";
    public OrderMetadata Metadata { get; set; } = new(); // Stored as JSON column
}
public class OrderMetadata { public string Source { get; set; } = ""; public List<string> Tags { get; set; } = new(); }
// Configuration:
mb.Entity<Order>().OwnsOne(o => o.Metadata, m => m.ToJson()); // Stored as JSON column
// Query into JSON:
var tagged = await ctx.Orders.Where(o => o.Metadata.Source == "ALGO").ToListAsync();
```

### Q118-Q120. EF Core performance patterns

```csharp
// Q118. Split queries (avoid cartesian explosion on multi-includes):
ctx.Orders
    .Include(o => o.Executions)
    .Include(o => o.Amendments)
    .AsSplitQuery()  // 3 separate queries instead of 1 large JOIN (fewer rows returned)
    .ToList();

// Q119. Query tags (trace SQL in logs):
ctx.Orders.TagWith("GetOpenOrders").Where(o => o.Status == "OPEN").ToList();
// SQL: -- GetOpenOrders\nSELECT ...

// Q120. No-parameterized literals (for IN query performance):
var symbols = new[] { "NIFTY50", "BANKNIFTY" };
ctx.Orders.Where(o => symbols.Contains(o.Symbol)).ToList();
// SQL: WHERE Symbol IN ('NIFTY50', 'BANKNIFTY')
```

### Q121-Q135. Configuration, Fluent API patterns

```csharp
// Q121. Fluent API configurations (organized in IEntityTypeConfiguration<T>):
public sealed class OrderEntityConfig : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("Orders", schema: "trading");
        b.HasKey(o => o.OrderId);
        b.Property(o => o.OrderId).HasMaxLength(20).IsRequired();
        b.Property(o => o.Symbol).HasMaxLength(20).IsRequired();
        b.Property(o => o.Price).HasPrecision(18, 4);
        b.HasIndex(o => new { o.Symbol, o.Status }).HasFilter("[Status] = 'OPEN'"); // Filtered index
    }
}
// Apply in OnModelCreating: mb.ApplyConfigurationsFromAssembly(typeof(TradingDbContext).Assembly);

// Q122. Table-per-hierarchy (TPH) — inheritance in one table:
public abstract class BaseEvent { public int Id { get; set; } public DateTime OccurredAt { get; set; } }
public sealed class OrderPlacedEvent : BaseEvent { public string OrderId { get; set; } = ""; }
public sealed class OrderCancelledEvent : BaseEvent { public string Reason { get; set; } = ""; }
// EF stores all in Events table with Discriminator column; default: class name

// Q123. Connection resiliency:
opts.UseSqlServer(conn, sqlOpts => sqlOpts
    .EnableRetryOnFailure(maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(10),
        errorNumbersToAdd: null)); // Auto-retry on transient SQL errors

// Q124. DbContext factory for Blazor / non-request scopes:
builder.Services.AddDbContextFactory<TradingDbContext>(opts => opts.UseSqlServer(conn));
// Usage: var ctx = await _factory.CreateDbContextAsync(); using (ctx) { ... }

// Q125. EF Core with repositories pattern:
public interface IOrderRepository { Task<Order?> FindAsync(string id, CT ct); Task CreateAsync(Order o, CT ct); }
public sealed class EfOrderRepository(TradingDbContext ctx) : IOrderRepository { /* ... */ }
// Register: services.AddScoped<IOrderRepository, EfOrderRepository>();

// Q126. Keyless entities (view-backed / raw SQL results):
[Keyless]
public class OrderSummaryView { public string Symbol { get; set; } = ""; public decimal TotalValue { get; set; } }
// mb.Entity<OrderSummaryView>().ToView("vw_OrderSummary"); // Maps to DB view
// ctx.Set<OrderSummaryView>().ToListAsync() → SELECT * FROM vw_OrderSummary

// Q127. Seeding data:
mb.Entity<Order>().HasData(new Order { OrderId = "SEED-001", Symbol = "NIFTY50", Status = "CLOSED", Price = 24000m, Quantity = 1, TraderId = "SYSTEM", CreatedAt = new DateTime(2024, 1, 1) });

// Q128. Database functions:
[DbFunction("dbo", "fn_GetOrderValue")]
public static decimal GetOrderValue(string orderId) => throw new NotImplementedException();
// ctx.Orders.Select(o => GetOrderValue(o.OrderId)).ToList() → SQL: SELECT dbo.fn_GetOrderValue(OrderId)

// Q129. Spatial / geography types (NetTopologySuite):
// Not typical for trading — skip

// Q130. EF Core diagnostics:
opts.LogTo(Console.WriteLine, LogLevel.Information)
    .EnableDetailedErrors()
    .EnableSensitiveDataLogging();

// Q131. Unit of work pattern:
public sealed class UnitOfWork(TradingDbContext ctx) : IDisposable
{
    public IOrderRepository Orders { get; } = new EfOrderRepository(ctx);
    public Task<int> SaveAsync(CT ct) => ctx.SaveChangesAsync(ct);
    public void Dispose() => ctx.Dispose();
}

// Q132. Pagination with EF Core:
var page = await ctx.Orders
    .Where(o => o.Symbol == symbol)
    .OrderByDescending(o => o.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .CountAsync(o => o.Symbol == symbol, ct); // For total count (separate query)

// Q133. Optimizing with AsNoTrackingWithIdentityResolution:
// AsNoTracking: no tracking — may duplicate navigations
// AsNoTrackingWithIdentityResolution: no tracking but deduplicates shared entities
var orders = await ctx.Orders.Include(o => o.Executions).AsNoTrackingWithIdentityResolution().ToListAsync();

// Q134. EF Core with CQRS:
// Command side: use DbContext directly (or via Repository)
// Query side: raw SQL / Dapper for performance — bypass EF tracking entirely
// public Task<List<T>> QueryAsync<T>(string sql, object param) — Dapper

// Q135. Table splitting (two entities to one table):
// Share a table between a "thin" Order and "extended" OrderDetails entity
mb.Entity<OrderDetails>().ToTable("Orders");
mb.Entity<Order>().HasOne(o => o.Details).WithOne().HasForeignKey<OrderDetails>(d => d.OrderId);
```

### Q136-Q151. Additional EF Core Topics (Concise)

```csharp
// Q136. Second-level cache: use EFCoreSecondLevelCacheInterceptor NuGet
// Q137. Shadow properties (not in entity class, tracked by EF):
mb.Entity<Order>().Property<DateTime>("LastModifiedAt");
ctx.Entry(order).Property<DateTime>("LastModifiedAt").CurrentValue = DateTime.UtcNow;

// Q138. Temporal tables (SQL Server, EF Core 6+):
mb.Entity<Order>().ToTable(t => t.IsTemporal());
// Auto-tracks history — query historical state:
ctx.Orders.TemporalAsOf(new DateTime(2024, 1, 1)).Where(o => o.OrderId == id).FirstOrDefault();

// Q139. LINQ query translation to SQL:
var query  = ctx.Orders.Where(o => o.Price > 24000);
var sql    = query.ToQueryString(); // View generated SQL without executing
Console.WriteLine(sql);

// Q140. DbContext.ChangeTracker.DetectChanges():
// Called automatically before SaveChanges — checks all tracked entities for modifications
// Disable for performance: ctx.ChangeTracker.AutoDetectChangesEnabled = false;
// Then manually: ctx.ChangeTracker.DetectChanges() before SaveChanges

// Q141. Repository + Specification pattern:
public interface ISpecification<T> { Expression<Func<T, bool>> Criteria { get; } }
public sealed class OpenOrdersSpec : ISpecification<Order>
{
    public Expression<Func<Order, bool>> Criteria => o => o.Status == "OPEN";
}

// Q142. EF Core Unit Test with InMemory:
var opts = new DbContextOptionsBuilder<TradingDbContext>().UseInMemoryDatabase("test").Options;
using var ctx = new TradingDbContext(opts);
ctx.Orders.Add(new Order { OrderId = "T1", Symbol = "NIFTY50", Status = "OPEN" });
ctx.SaveChanges();
// ⚠️ InMemory doesn't enforce FK constraints or column types — use SQLite for closer SQL testing

// Q143. SQLite for EF Core testing:
var opts2 = new DbContextOptionsBuilder<TradingDbContext>()
    .UseSqlite("Data Source=:memory:").Options;
using var ctx2 = new TradingDbContext(opts2);
await ctx2.Database.EnsureCreatedAsync(); // Create schema

// Q144. async LINQ operators:
await ctx.Orders.ForEachAsync(o => Console.WriteLine(o.OrderId)); // ← anti-pattern
// Prefer: var list = await ctx.Orders.ToListAsync(); foreach(var o in list) ...

// Q145. ExecuteSqlInterpolated vs FromSqlInterpolated:
// FromSqlInterpolated → maps to entities (SELECT)
// ExecuteSqlInterpolated → DML (INSERT/UPDATE/DELETE) → returns rows affected

// Q146. DbContextFactory vs AddDbContext:
// AddDbContext → Scoped (per request) — standard for web APIs
// AddDbContextFactory → Create contexts on demand — Blazor, background services

// Q147. Bulk copy with SqlBulkCopy (for mass inserts):
using var bulkCopy = new SqlBulkCopy(conn) { DestinationTableName = "Orders" };
bulkCopy.ColumnMappings.Add("OrderId", "OrderId"); // map property to column
await bulkCopy.WriteToServerAsync(dataTable); // 100k rows in seconds

// Q148. EF Core migrations in production best practices:
// 1. Never auto-migrate in production startup
// 2. Generate migration script: dotnet ef migrations script --idempotent
// 3. Review with DBA
// 4. Apply in maintenance window
// 5. Use GitHub Actions/Azure DevOps pipeline for automated deploy

// Q149. Dapper alongside EF Core (hybrid):
using Dapper;
var conn = ctx.Database.GetDbConnection();
var report = await conn.QueryAsync<ReportRow>(
    "SELECT Symbol, SUM(Price*Qty) AS Total FROM Orders GROUP BY Symbol WITH (NOLOCK)");
// Dapper for read-heavy reporting; EF for writes/transactions

// Q150. EF Core relationships — full matrix:
// One-to-One:    HasOne().WithOne().HasForeignKey<T>(...)
// One-to-Many:   HasOne().WithMany().HasForeignKey(...)
// Many-to-Many:  HasMany().WithMany() - EF 5+ auto-creates join table
// Self-referential: HasMany(o => o.Children).WithOne(o => o.Parent).HasForeignKey(o => o.ParentId)

// Q151. EF Core provider-specific features:
// SQL Server: .UseHierarchyId(), .IsTemporal(), row-level security via filter
// PostgreSQL (Npgsql): .UseNetTopologySuite(), JSONB column type
// SQLite: testing only — no decimal precision, limited date functions
// Cosmos DB: .UseCosmos() — NoSQL document model with EF surface
```

---
