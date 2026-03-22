
---

## Q2. Explain events vs delegates. How does the publisher-subscriber model work in C#?

### 📌 Definition
An **event** is a language-level encapsulation wrapper built on top of a multicast delegate that enforces the publish-subscribe contract. While a delegate is a type-safe method reference, an event restricts external access so that only the **declaring class (publisher) can invoke or reassign it**, while external classes (subscribers) can only **add (`+=`) or remove (`-=`) handlers**. The compiler converts an `event` declaration into a private backing delegate field plus generated thread-safe `add`/`remove` accessor methods — analogous to how a `property` wraps a backing field with `get`/`set`. This ensures strict publish-subscribe discipline: publishers fire events, subscribers react to them — neither side can violate the other's boundary.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ PROBLEM: raw delegate as public field — zero protection
public class Button
{
    public Action? Clicked; // Anyone can: button.Clicked = null; (wipes ALL subscribers!)
                             //             button.Clicked?.Invoke(); (external invocation!)
}
```

```csharp
// ✅ SOLUTION: event keyword enforces access control
public class Button
{
    public event Action? Clicked;            // 'event' generates private backing field
    protected void OnClicked() => Clicked?.Invoke(); // Only Button can invoke

    // Compiler-generated equivalent:
    // private Action? _clicked;
    // add    { lock(this) { _clicked += value; } }
    // remove { lock(this) { _clicked -= value; } }
}
// External code: b.Clicked = null;       → COMPILE ERROR
// External code: b.Clicked?.Invoke();    → COMPILE ERROR
// External code: b.Clicked += handler;   → OK
```

**Built-in event patterns:**
```csharp
// Pattern 1: EventHandler — no custom data needed
public event EventHandler? ServerStarted;
// Raise: ServerStarted?.Invoke(this, EventArgs.Empty);

// Pattern 2: EventHandler<TEventArgs> — typed event data
public sealed class OrderEventArgs : EventArgs
{
    public string  OrderId { get; init; } = "";
    public decimal Amount  { get; init; }
}
public event EventHandler<OrderEventArgs>? OrderPlaced;
// Raise: OrderPlaced?.Invoke(this, new OrderEventArgs { OrderId="ORD1", Amount=5000 });

// Pattern 3: Custom delegate type (use when EventHandler<T> doesn't fit)
public delegate void PriceChangedHandler(string symbol, decimal oldPrice, decimal newPrice);
public event PriceChangedHandler? PriceChanged;
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Stock Price Alert System**
```csharp
using System;

namespace StockPlatform.Events
{
    public sealed class PriceTickEventArgs : EventArgs
    {
        public string  Symbol   { get; }
        public decimal OldPrice { get; }
        public decimal NewPrice { get; }
        public decimal Change   => NewPrice - OldPrice;
        public double  ChangePct => (double)((NewPrice - OldPrice) / OldPrice * 100m);

        public PriceTickEventArgs(string symbol, decimal old, decimal @new)
            => (Symbol, OldPrice, NewPrice) = (symbol, old, @new);
    }

    // ── PUBLISHER ───────────────────────────────────────────────────────────
    public sealed class MarketFeed
    {
        public event EventHandler<PriceTickEventArgs>? OnPriceTick;
        private decimal _lastPrice;
        public string Symbol { get; }

        public MarketFeed(string symbol, decimal initial) => (Symbol, _lastPrice) = (symbol, initial);

        public void PublishTick(decimal newPrice)
        {
            var args = new PriceTickEventArgs(Symbol, _lastPrice, newPrice);
            _lastPrice = newPrice;
            Console.WriteLine($"\n[FEED] {Symbol}: {args.OldPrice:C} → {args.NewPrice:C} ({args.ChangePct:+0.00;-0.00}%)");
            OnPriceTick?.Invoke(this, args); // Thread-safe snapshot invocation
        }
    }

    // ── SUBSCRIBER 1: Alert Engine ─────────────────────────────────────────
    public sealed class AlertEngine : IDisposable
    {
        private readonly MarketFeed _feed;
        private readonly double     _threshold;

        public AlertEngine(MarketFeed feed, double thresholdPct)
        {
            _feed      = feed;
            _threshold = thresholdPct;
            _feed.OnPriceTick += OnTick; // Subscribe
        }

        private void OnTick(object? sender, PriceTickEventArgs e)
        {
            if (Math.Abs(e.ChangePct) >= _threshold)
                Console.WriteLine($"  [ALERT] ⚠️  {(e.Change > 0 ? "SURGE" : "DROP")} on {e.Symbol}: {e.ChangePct:+0.0;-0.0}%");
        }

        public void Dispose() => _feed.OnPriceTick -= OnTick; // Critical: prevent memory leak
    }

    // ── SUBSCRIBER 2: Dashboard ────────────────────────────────────────────
    public sealed class TradingDashboard
    {
        public TradingDashboard(MarketFeed feed) => feed.OnPriceTick += (_, e) =>
            Console.WriteLine($"  [UI] {e.Symbol} → {e.NewPrice:C} [{(e.Change >= 0 ? "GREEN" : "RED")}]");
    }

    public static class Program
    {
        public static void Main()
        {
            var feed    = new MarketFeed("NIFTY50", 24_500m);
            using var alert = new AlertEngine(feed, alertThresholdPct: 1.0);
            var dash    = new TradingDashboard(feed);

            feed.PublishTick(24_520m); // <1% — no alert, dashboard updates
            feed.PublishTick(24_800m); // >1% — alert + dashboard
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Concern | Detail |
|---|---|
| **Memory** | Subscriber held in publisher's invocation list → GC cannot collect subscriber until `-=` called. Always unsubscribe in `IDisposable.Dispose()` |
| **Thread safety** | Compiler-generated `add`/`remove` use `lock(this)` — safe for concurrent subscribe/unsubscribe |
| **Design** | Events are `immutable` from the subscriber's perspective; the backing delegate field is `mutable` only inside the publisher |
| **Custom accessors** | Override `add`/`remove` for `WeakReference<T>` subscriber lists (prevents leak) or lock-free `Interlocked.CompareExchange` patterns |

---

## Q3. What is the CLR? Explain JIT compilation lifecycle.

### 📌 Definition
The **Common Language Runtime (CLR)** is the managed execution engine of .NET — a virtual machine that sits between compiled .NET code and the host OS. It provides automatic memory management (Garbage Collector), JIT compilation, type safety enforcement, exception handling, thread management, security, and cross-language interoperability. When you compile C# source, the Roslyn compiler does NOT emit native machine code — it emits **Intermediate Language (IL)** (also called CIL/MSIL), a CPU-agnostic bytecode that any CLR implementation can understand. The CLR's JIT compiler (**RyuJIT** in .NET 5+) translates IL into native CPU instructions on first method invocation, enabling write-once, run-anywhere semantics with runtime-adaptive optimization.

### 🔍 Why It Exists / Problem It Solves

```
Without CLR (C/C++ model):
  source.cpp → compiler → machine.exe   (tied to one CPU arch, no GC, no type safety)

With CLR (.NET model):
  source.cs → Roslyn → assembly.dll (IL) → CLR/JIT → optimized native code
                                                 ↑ GC, exceptions, interop, security
```

**JIT Lifecycle — step by step:**
```csharp
// Step 1: C# source → IL (platform-neutral bytecode)
public static int Add(int a, int b) => a + b;
// IL: ldarg.0 / ldarg.1 / add / ret

// Step 2: CLR loads the assembly, reads type & method metadata
// Step 3: First call to Add() — JIT stub intercepts
//   RyuJIT runs optimization passes:
//     - Method inlining (tiny methods merged into caller, zero call overhead)
//     - Loop unrolling and vectorization (SIMD/AVX2 for span/array ops)
//     - Dead code elimination, constant folding
//     - Register allocation (prefer CPU registers over stack)
//   Result: native x64 code stored in Code Heap (non-GC memory)
// Step 4: Method table updated to point directly to native code
// Step 5: All subsequent calls execute native code directly — JIT cost paid once
```

**Compilation tiers in .NET 6+ (Tiered Compilation):**
```csharp
// Tier 0: Quick JIT — fast compilation, minimal optimization, call counters injected
// Tier 1: After ~30 hot calls → RyuJIT re-compiles with full optimization (PGO)
// This gives: fast startup (Tier 0) + peak throughput (Tier 1)

// AOT option — for cold-start sensitive workloads (Lambda, CLIs, IoT):
// <PublishAot>true</PublishAot>  → no JIT at runtime, fully native binary
// <PublishReadyToRun>true</PublishReadyToRun> → partial AOT, faster startup

// Check current JIT mode:
Console.WriteLine(System.Runtime.CompilerServices.RuntimeFeature.IsDynamicCodeSupported
    ? "JIT" : "AOT");
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Measuring JIT warm-up cost in a trading microservice**

```csharp
using System;
using System.Diagnostics;
using System.Runtime;
using System.Runtime.CompilerServices;

namespace CLRDemo
{
    public static class JitDemo
    {
        [MethodImpl(MethodImplOptions.NoInlining)] // Prevent JIT from inlining so we can measure
        static decimal ComputeVWAP(decimal[] prices)
        {
            decimal sum = 0;
            foreach (var p in prices) sum += p;
            return sum / prices.Length;
        }

        public static void Main()
        {
            Console.WriteLine($"CLR   : {System.Runtime.InteropServices.RuntimeEnvironment.GetRuntimeDirectory()}");
            Console.WriteLine($"GC    : {(GCSettings.IsServerGC ? "Server" : "Workstation")} | " +
                              $"64-bit: {Environment.Is64BitProcess}");

            var prices = new decimal[10_000];
            for (int i = 0; i < prices.Length; i++) prices[i] = 24_000m + i;

            // Cold call — JIT must compile ComputeVWAP first
            var sw = Stopwatch.StartNew();
            decimal vwap = ComputeVWAP(prices);
            sw.Stop();
            Console.WriteLine($"\nCold  (JIT + exec): {sw.Elapsed.TotalMicroseconds:F1} µs | VWAP={vwap:C}");

            // Warm call — native code in Code Heap
            sw.Restart(); vwap = ComputeVWAP(prices); sw.Stop();
            Console.WriteLine($"Warm  (native):     {sw.Elapsed.TotalMicroseconds:F1} µs | VWAP={vwap:C}");

            // Tiered JIT kicks in after ~30 calls with full optimization
            for (int i = 0; i < 50; i++) ComputeVWAP(prices);
            sw.Restart(); vwap = ComputeVWAP(prices); sw.Stop();
            Console.WriteLine($"Tier-1 JIT:         {sw.Elapsed.TotalMicroseconds:F1} µs | VWAP={vwap:C}");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Memory Region | Contents | GC Managed |
|---|---|---|
| Managed Heap | Objects, arrays, strings | ✅ Yes |
| Code Heap | JIT-compiled native methods | ❌ No (permanent) |
| Loader Heap | Type metadata, method tables, vtables | ❌ No |
| Stack | Stack frames, value locals | ❌ No |
| LOH (>85 KB) | Large arrays/strings | Gen2 GC only |

```xml
<!-- .csproj — production startup tuning -->
<PublishReadyToRun>true</PublishReadyToRun>   <!-- Pre-JIT hot paths at publish -->
<TieredPGO>true</TieredPGO>                  <!-- Profile-guided optimization -->
```

---

## Q4. Difference between Task, Thread, and ThreadPool. When to use each?

### 📌 Definition
- **`Thread`** — A raw OS-level execution unit with its own ~1MB stack. You have full lifecycle control (priority, name, foreground/background) but creation is expensive (OS kernel call). Best for long-running, dedicated blocking work where you need OS-level thread semantics.
- **`ThreadPool`** — A CLR-managed pool of pre-created threads shared across the process. Work items are queued and picked up by idle threads — eliminates thread creation overhead. Limited control: no priority, no blocking waits, anonymous. Best for short fire-and-forget CPU work.
- **`Task`** (TPL — Task Parallel Library) — The modern highest-level abstraction. A Task represents **a unit of work**, not a thread. It runs on the ThreadPool (or a custom scheduler), supports typed return values (`Task<T>`), built-in cancellation (`CancellationToken`), exception aggregation, and composes naturally with `async/await`. Should be the default choice for all asynchronous and parallel work.

### 🔍 Why It Exists / Problem It Solves

```
.NET 1.0  →  Thread         — manual, expensive, verbose, no return value
.NET 2.0  →  ThreadPool     — cheaper, pooled, but still no result/cancellation
.NET 4.0  →  Task / TPL    — composable, cancellable, awaitable, result-bearing
.NET 5+   →  ValueTask      — zero-allocation async for synchronous-fast hot paths
```

```csharp
// Thread — for long-running dedicated work
var listenerThread = new Thread(() =>
{
    Thread.CurrentThread.Name     = "MarketDataListener";
    Thread.CurrentThread.Priority = ThreadPriority.AboveNormal;
    while (!_stopping) ProcessNextPacket(); // blocks — dedicated thread OK
}) { IsBackground = true };
listenerThread.Start();

// ThreadPool — fire-and-forget short CPU work
ThreadPool.QueueUserWorkItem(_ => ComputeRiskSnapshot());
// ⚠️ No return value, no exception propagation, no await

// Task — everything modern
Task<decimal> priceTask = Task.Run(() => CalculateVWAP(prices));  // CPU-bound
decimal vwap = await priceTask;                                   // await result

// I/O-bound — no thread consumed during wait
decimal price = await _redis.GetAsync("NIFTY50:price");

// Fan-out: 3 parallel I/O calls, await all
var results = await Task.WhenAll(
    FetchPriceAsync("NIFTY50"),
    FetchPriceAsync("BANKNIFTY"),
    FetchPriceAsync("SENSEX")
);
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Order processing service with mixed I/O and CPU workloads**

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TradingPlatform.Concurrency
{
    public sealed class OrderProcessingService : IDisposable
    {
        private readonly CancellationTokenSource _cts = new();
        private Thread? _ingestionThread;

        // ── Dedicated Thread: long-running order listener ──────────────────
        public void StartIngestion()
        {
            _ingestionThread = new Thread(IngestOrders)
            {
                Name         = "OrderIngestion",
                IsBackground = true,          // Dies when app exits
                Priority     = ThreadPriority.AboveNormal
            };
            _ingestionThread.Start();
        }

        private void IngestOrders()
        {
            Console.WriteLine($"[Ingestion] Started. OS Thread: {Thread.CurrentThread.ManagedThreadId}");
            while (!_cts.IsCancellationRequested)
            {
                Thread.Sleep(100); // Simulate blocking queue receive
                string orderId = $"ORD-{DateTime.UtcNow.Ticks % 10000:D4}";
                // Offload per-order work to Task — don't block the ingestion thread
                _ = ProcessOrderAsync(orderId, _cts.Token);
            }
        }

        // ── Task: async I/O + CPU pipeline per order ───────────────────────
        private async Task ProcessOrderAsync(string orderId, CancellationToken ct)
        {
            try
            {
                // I/O-bound fan-out — no thread blocked during network wait
                (decimal spot, decimal futures) = await FetchMarketDataAsync(ct);

                // CPU-bound computation — offloaded to ThreadPool thread
                decimal premium = await Task.Run(
                    () => CalculateFuturesPremium(spot, futures), ct);

                Console.WriteLine($"[{orderId}] Spot={spot:C} Futures={futures:C} Premium={premium:C} " +
                                  $"| Thread={Thread.CurrentThread.ManagedThreadId}");
            }
            catch (OperationCanceledException) { /* graceful shutdown */ }
            catch (Exception ex) { Console.WriteLine($"[{orderId}] Error: {ex.Message}"); }
        }

        // Simulates Redis/DB I/O (0 threads blocked during await)
        private static async Task<(decimal spot, decimal futures)> FetchMarketDataAsync(CancellationToken ct)
        {
            await Task.WhenAll(Task.Delay(30, ct), Task.Delay(30, ct));
            return (24_500m, 24_620m);
        }

        // CPU-bound — runs on ThreadPool, not inline
        private static decimal CalculateFuturesPremium(decimal spot, decimal futures)
            => futures - spot; // Simplified; real: cost-of-carry model

        public void Dispose() => _cts.Cancel();
    }

    public static class Program
    {
        public static async Task Main()
        {
            // Tune ThreadPool min threads to avoid starvation under burst
            ThreadPool.SetMinThreads(
                workerThreads: Environment.ProcessorCount * 4,
                completionPortThreads: Environment.ProcessorCount * 2);

            using var svc = new OrderProcessingService();
            svc.StartIngestion();
            await Task.Delay(500);
            Console.WriteLine("Shutting down...");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| | Thread | ThreadPool | Task |
|---|---|---|---|
| **Stack memory** | ~1MB per thread | ~1MB per pooled thread | Borrows ThreadPool thread stack |
| **Creation cost** | High (OS syscall, ~µs) | Low (reuse existing) | Very low (heap allocation) |
| **Return value** | No | No | `Task<T>` — typed, awaitable |
| **Cancellation** | Manual volatile flag | Manual volatile flag | `CancellationToken` built-in |
| **Exceptions** | Unhandled = crash | Silently swallowed | Captured, re-thrown on `await` |
| **Best for** | Long-running blocking | Short fire-and-forget | Everything async/parallel |

---

## Q5. How does async/await work internally? What is a state machine?

### 📌 Definition
`async`/`await` is a compiler transformation feature that lets you write **asynchronous, non-blocking code** in a sequential, readable style without manually managing callbacks or continuations. When you mark a method `async`, the C# compiler rewrites it into a **state machine** — a struct that implements `IAsyncStateMachine`. This struct captures all local variables and parameters as fields, and encodes the method's execution as a series of states separated by each `await` point. When an `await` encounters an incomplete `Task`, the state machine suspends (returns control to the caller), and the remainder of the method is registered as a **continuation** that fires when the awaited task completes — all without blocking any thread.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ BEFORE async/await: callback hell (Node.js style)
_db.BeginFetch("order123", result =>
{
    if (result.Error != null) { HandleError(result.Error); return; }
    _cache.BeginStore(result.Data, stored =>
    {
        if (stored.Error != null) { HandleError(stored.Error); return; }
        _log.BeginWrite("Saved", written => { /* nested 3 levels deep */ });
    });
});

// ✅ WITH async/await: sequential-looking, compiler handles callbacks
async Task ProcessOrderAsync(string id)
{
    var order  = await _db.FetchAsync(id);      // suspend here, free the thread
    await _cache.StoreAsync(order);              // suspend here
    await _log.WriteAsync("Saved");              // suspend here
    // reads top-to-bottom — no nesting, full exception handling, cancellation support
}
```

**What the compiler generates for an async method:**
```csharp
// Your code:
async Task<decimal> FetchPriceAsync(string symbol)
{
    await Task.Delay(50);               // await point 1 — State 0
    decimal raw = await _api.GetAsync(symbol);  // await point 2 — State 1
    return raw * 1.005m;                // final state — State -1
}

// Compiler generates (simplified) — a struct state machine:
private struct FetchPriceAsyncStateMachine : IAsyncStateMachine
{
    public int _state;               // -1 = done, 0 = before delay, 1 = before api call
    public AsyncTaskMethodBuilder<decimal> _builder; // wraps the Task<decimal>
    public string symbol;            // captured parameter
    private decimal raw;             // captured local variable
    private TaskAwaiter _awaiter0;   // awaiter for Task.Delay
    private TaskAwaiter<decimal> _awaiter1; // awaiter for _api.GetAsync

    public void MoveNext()  // called by the runtime when each awaited Task completes
    {
        switch (_state)
        {
            case 0:
                var delayTask = Task.Delay(50);
                if (!delayTask.IsCompleted) {
                    _state = 1;
                    _awaiter0 = delayTask.GetAwaiter();
                    _awaiter0.OnCompleted(MoveNext); // register continuation
                    return;  // RETURN to caller — thread is FREE
                }
                goto case 1;

            case 1:
                var apiTask = _api.GetAsync(symbol);
                if (!apiTask.IsCompleted) {
                    _state = 2;
                    _awaiter1 = apiTask.GetAwaiter();
                    _awaiter1.OnCompleted(MoveNext); // register continuation
                    return;  // thread is FREE again
                }
                raw = _awaiter1.GetResult();
                _builder.SetResult(raw * 1.005m); // complete the returned Task<decimal>
                break;
        }
    }
}
// The state machine is a STRUCT → allocated on the STACK initially
// BUT if it suspends (goes async), it gets BOXED to the heap as IAsyncStateMachine
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Order validation pipeline with multiple async I/O steps**

```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

namespace TradingPlatform.Async
{
    public record Order(string Id, string Symbol, decimal Quantity, decimal Price);
    public record ValidationResult(bool IsValid, string Reason);

    public sealed class OrderValidationService
    {
        private readonly HttpClient _httpClient;
        private readonly SemaphoreSlim _rateLimiter = new(10); // max 10 concurrent validations

        public OrderValidationService(HttpClient httpClient) => _httpClient = httpClient;

        // Orchestrates 3 async validation steps sequentially
        public async Task<ValidationResult> ValidateAsync(Order order, CancellationToken ct = default)
        {
            // Step 1: Rate-limit concurrency (await releases thread if at capacity)
            await _rateLimiter.WaitAsync(ct);
            try
            {
                // Step 2: Check circuit breaker / risk limits (I/O to Redis)
                bool withinRiskLimit = await CheckRiskLimitAsync(order, ct);
                if (!withinRiskLimit)
                    return new ValidationResult(false, "Risk limit exceeded");

                // Step 3: Validate market hours and symbol (I/O to market data service)
                bool marketOpen = await CheckMarketStatusAsync(order.Symbol, ct);
                if (!marketOpen)
                    return new ValidationResult(false, "Market closed for symbol");

                // Step 4: Fraud screening (I/O to fraud service)
                bool passesFraud = await ScreenForFraudAsync(order, ct);
                if (!passesFraud)
                    return new ValidationResult(false, "Flagged by fraud screening");

                return new ValidationResult(true, "All checks passed");
            }
            finally
            {
                _rateLimiter.Release(); // Always release the semaphore slot
            }
        }

        private async Task<bool> CheckRiskLimitAsync(Order order, CancellationToken ct)
        {
            await Task.Delay(10, ct); // Simulate Redis GET
            return order.Quantity * order.Price <= 1_000_000m;
        }

        private async Task<bool> CheckMarketStatusAsync(string symbol, CancellationToken ct)
        {
            await Task.Delay(15, ct); // Simulate market data API call
            var now = DateTime.UtcNow.AddHours(5.5); // IST offset
            return now.Hour >= 9 && now.Hour < 16;   // NSE market hours
        }

        private async Task<bool> ScreenForFraudAsync(Order order, CancellationToken ct)
        {
            await Task.Delay(20, ct); // Simulate fraud service HTTP call
            return !order.Id.StartsWith("BLACKLIST"); // Simplified fraud check
        }
    }

    public static class Program
    {
        public static async Task Main()
        {
            var svc = new OrderValidationService(new HttpClient());

            // Validate 5 orders concurrently — no threads blocked during I/O waits
            var orders = new[]
            {
                new Order("ORD-001", "NIFTY50",   10m, 24_500m),
                new Order("ORD-002", "BANKNIFTY", 50m, 52_100m),
                new Order("ORD-003", "RELIANCE", 100m, 2_800m),
            };

            using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

            var results = await Task.WhenAll(
                Array.ConvertAll(orders, o => svc.ValidateAsync(o, cts.Token)));

            for (int i = 0; i < orders.Length; i++)
                Console.WriteLine($"[{orders[i].Id}] Valid={results[i].IsValid} | {results[i].Reason}");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Aspect | Detail |
|---|---|
| **State machine memory** | Initially stack-allocated struct; boxed to heap (Gen0) when first suspension occurs |
| **Thread usage** | 0 threads consumed during I/O wait — continuation resumes on ThreadPool thread |
| **SynchronizationContext** | In ASP.NET Core there is none — continuations run on any ThreadPool thread. In UI apps (WPF/WinForms) continuations marshal back to UI thread unless `ConfigureAwait(false)` is used |
| **ValueTask** | Use `ValueTask<T>` instead of `Task<T>` in hot paths where completion is often synchronous — avoids heap allocation of Task object |
| **Async void** | Never use `async void` except for event handlers — exceptions are unobserved and crash the process |

---

## Q6. What is SynchronizationContext and why is it important?

### 📌 Definition
`SynchronizationContext` is an abstract base class in .NET that represents the **execution environment's threading model and controls how continuations (the code after `await`) are dispatched**. It allows the `async`/`await` machinery to post continuation callbacks back to the appropriate thread — for example, back to the **UI thread** in WPF/WinForms, or to an **ASP.NET request context** in ASP.NET (classic). When you `await` a Task, the runtime captures the current `SynchronizationContext` (via `SynchronizationContext.Current`) and, after the awaited operation completes, schedules the continuation via that context's `Post` method. In ASP.NET Core, no custom `SynchronizationContext` is installed — continuations run on any `ThreadPool` thread — making `ConfigureAwait(false)` essentially a no-op (though still a best practice for library code).

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Problem in WPF — accessing UI element from non-UI thread
async void LoadData_Click(object sender, RoutedEventArgs e)
{
    string data = await _api.FetchAsync(); // resumes on ThreadPool thread
    MyLabel.Content = data;               // ← CRASH: InvalidOperationException
                                          //   UI controls can only be accessed on UI thread
}

// ✅ With SynchronizationContext captured before await:
// WPF installs DispatcherSynchronizationContext on the UI thread
// After await, continuation is Post()ed back to the UI thread automatically:
async void LoadData_Click(object sender, RoutedEventArgs e) // runs on UI thread
{
    // SyncContext captured: DispatcherSynchronizationContext
    string data = await _api.FetchAsync(); // suspends; SyncContext remembered
    // After fetch completes: continuation posted via Dispatcher.BeginInvoke()
    MyLabel.Content = data; // ✅ Safe — back on UI thread
}
```

**SynchronizationContext types:**
```csharp
// 1. null (ASP.NET Core / Console) — no special context, continues on ThreadPool
// 2. DispatcherSynchronizationContext (WPF) — posts to Dispatcher (UI thread)
// 3. WindowsFormsSynchronizationContext (WinForms) — posts to Control.BeginInvoke
// 4. AspNetSynchronizationContext (ASP.NET Classic) — ensures one thread at a time
//    per request (emulates single-threaded request model despite async)

// ConfigureAwait(false) — tells await: don't capture SynchronizationContext
// Continuation runs on whatever ThreadPool thread completes the task
// Use in library code to avoid deadlocks and unnecessary marshaling:

public async Task<decimal> FetchPriceAsync(string symbol)
{
    var response = await _httpClient.GetAsync($"/price/{symbol}")
                                    .ConfigureAwait(false); // don't capture context
    var body = await response.Content.ReadAsStringAsync()
                                     .ConfigureAwait(false);
    return decimal.Parse(body);
}
```

**The classic deadlock problem (ASP.NET Classic / WPF):**
```csharp
// ❌ DEADLOCK pattern — blocks a context-bound thread while async code waits for it
decimal GetPrice(string symbol)
{
    // .Result blocks current thread (UI thread / ASP.NET request thread)
    return FetchPriceAsync(symbol).Result;  // <-- blocks UI/request thread
    // FetchPriceAsync's continuation needs to run on the SAME thread
    // But that thread is BLOCKED by .Result → DEADLOCK
}

// ✅ Fix: always await, never .Result / .Wait() on context-bound threads
async Task<decimal> GetPriceAsync(string symbol)
    => await FetchPriceAsync(symbol); // no deadlock
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: WPF trading dashboard that safely updates UI from async data fetches**

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;

namespace TradingDashboard.WPF
{
    public partial class MainWindow : Window
    {
        private readonly MarketDataService _market = new();
        private Label _priceLabel = null!;

        // Constructor runs on UI thread — SyncContext = DispatcherSynchronizationContext
        public MainWindow()
        {
            InitializeComponent();
            _priceLabel = new Label { Content = "Loading..." };
            Content = _priceLabel;

            // Fire-and-forget task — continuation will come back to UI thread
            _ = RefreshPricesAsync();
        }

        private async Task RefreshPricesAsync()
        {
            while (true)
            {
                try
                {
                    // Await releases UI thread during fetch
                    // SynchronizationContext is captured BEFORE await
                    decimal price = await _market.FetchPriceAsync("NIFTY50");

                    // After await: DispatcherSynchronizationContext.Post() brings us
                    // back to the UI thread — safe to update UI controls
                    _priceLabel.Content = $"NIFTY50: {price:C}";
                }
                catch (Exception ex)
                {
                    _priceLabel.Content = $"Error: {ex.Message}";
                }

                await Task.Delay(1000); // Wait 1 second — UI thread free during delay
            }
        }
    }

    // ── Library class — use ConfigureAwait(false) to avoid context capture ──
    public sealed class MarketDataService
    {
        private readonly System.Net.Http.HttpClient _http = new();

        // Library method: ConfigureAwait(false) throughout
        // - Avoids unnecessary marshaling back to caller's context
        // - Prevents deadlock if caller uses .Result incorrectly
        // - Slight performance improvement (no SyncContext.Post overhead)
        public async Task<decimal> FetchPriceAsync(string symbol)
        {
            Console.WriteLine($"  [Fetch] Thread: {Thread.CurrentThread.ManagedThreadId} (ThreadPool)");
            await Task.Delay(50).ConfigureAwait(false); // simulate I/O; no context capture
            return 24_500m + (decimal)(new Random().NextDouble() * 100);
        }
    }

    // ── ASP.NET Core controller (no SyncContext — ConfigureAwait is a no-op) ─
    // public class PriceController : ControllerBase
    // {
    //     [HttpGet] public async Task<decimal> Get()
    //         => await _market.FetchPriceAsync("NIFTY50"); // fine without ConfigureAwait(false)
    // }
}
```

### ⚙️ Production-Grade Implementation

| Context Type | `SynchronizationContext.Current` | Continuation runs on |
|---|---|---|
| ASP.NET Core | `null` | Any ThreadPool thread |
| WPF | `DispatcherSynchronizationContext` | UI (Dispatcher) thread |
| WinForms | `WindowsFormsSynchronizationContext` | UI thread via `Control.Invoke` |
| ASP.NET Classic | `AspNetSynchronizationContext` | Request thread (one at a time) |
| Console / Worker | `null` | Any ThreadPool thread |

**Rules:**
- Library code: always `ConfigureAwait(false)` — avoid context capture, prevent deadlocks
- Application code (UI handlers, controllers): can omit `ConfigureAwait(false)` — context marshaling is desired
- Never `.Result` / `.Wait()` in synchronous code calling async methods without `ConfigureAwait(false)` in the async chain

---

## Q7. Explain garbage collection generations (Gen0, Gen1, Gen2).

### 📌 Definition
The .NET Garbage Collector (GC) uses a **generational collection model** based on the empirical observation that most objects die young (the "generational hypothesis"). Memory on the managed heap is divided into three generations:
- **Gen0** — newest, smallest (~256 KB by default). All new small objects are allocated here. Collected most frequently (potentially hundreds of times per second). Very fast — typically sub-millisecond.
- **Gen1** — a short-lived buffer between Gen0 and Gen2. Objects that survive one Gen0 collection are promoted here. Acts as a filter to catch medium-lived objects. Collected less frequently than Gen0.
- **Gen2** — oldest, largest. Contains long-lived objects (static state, caches, app-lifetime services). Collected rarely. A Gen2 collection triggers a **Full GC** — suspends all threads (Stop-The-World), scans the entire heap. Can take tens of milliseconds — avoid promoting unnecessary objects here.
- **Large Object Heap (LOH)** — separate region for objects ≥ 85,000 bytes. Always treated as Gen2. Not compacted by default (since moving large objects is expensive), which causes fragmentation.

### 🔍 Why It Exists / Problem It Solves

```
Problem: How do you collect memory efficiently without scanning the entire heap every time?

Solution: Generational GC — only scan young regions (Gen0) most of the time.
          Old regions (Gen2) only scanned when Gen0 + Gen1 are full.

Generational heap layout:
┌─────────────────────────────────────────────────────────────┐
│  Gen0  │    Gen1    │              Gen2              │  LOH  │
│ ~256KB │  ~2-4 MB   │           many MB              │ large │
│ newest │ medium-age │ long-lived (singletons, caches) │ ≥85KB │
└─────────────────────────────────────────────────────────────┘
         ↑            ↑                                ↑
    frequent      occasional                         rare
    (ms apart)    (seconds)                     (seconds-minutes)
```

**GC modes:**
```csharp
// Workstation GC (default for client apps):
// - Single thread, lower latency, smaller footprint
// - Gen0/1 collected on allocating thread

// Server GC (default for ASP.NET Core, microservices):
// - One GC thread per CPU core, parallel collection
// - Larger heap segments for throughput
// Confirm mode:
Console.WriteLine(System.Runtime.GCSettings.IsServerGC); // true in containers/server

// GC.Collect() — force collection (almost never use in production!)
GC.Collect(0); // collect Gen0 only
GC.Collect(2); // full collection — avoid, causes long STW pause

// Check object generation (debugging only):
var obj = new object();
Console.WriteLine(GC.GetGeneration(obj)); // 0 — just allocated
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Object allocation patterns in a high-throughput order processing service**

```csharp
using System;
using System.Runtime;
using System.Collections.Generic;

namespace TradingPlatform.Memory
{
    // ── GOOD: short-lived, struct-based — stays in Gen0, collected fast ─────
    public readonly record struct MarketTick(string Symbol, decimal Price, DateTime At);

    // ── BAD: long-lived cache holding many objects → forced into Gen2 ────────
    public sealed class NaiveOrderCache
    {
        // ❌ This grows indefinitely → regularly promoted to Gen2
        private readonly List<object> _allOrders = new();

        public void Add(string orderId) => _allOrders.Add(orderId);
        // In production this is a memory leak — Gen2 fills up, Full GC fires
    }

    // ── GOOD: bounded cache, controls Gen2 pressure ───────────────────────
    public sealed class BoundedOrderCache
    {
        private readonly int _maxSize;
        private readonly Queue<string> _recentOrders = new();

        public BoundedOrderCache(int maxSize) => _maxSize = maxSize;

        public void Add(string orderId)
        {
            if (_recentOrders.Count >= _maxSize)
                _recentOrders.Dequeue(); // Remove oldest — allow GC to reclaim
            _recentOrders.Enqueue(orderId);
        }
    }

    // ── Memory diagnostics for production monitoring ───────────────────────
    public static class GCDiagnostics
    {
        public static void PrintGCStats()
        {
            Console.WriteLine($"Gen0 collections : {GC.CollectionCount(0)}");
            Console.WriteLine($"Gen1 collections : {GC.CollectionCount(1)}");
            Console.WriteLine($"Gen2 collections : {GC.CollectionCount(2)}");
            Console.WriteLine($"Total GC memory  : {GC.GetTotalMemory(forceFullCollection: false) / 1024:N0} KB");

            var memInfo = GC.GetGCMemoryInfo();
            Console.WriteLine($"Heap size        : {memInfo.HeapSizeBytes / 1024 / 1024:N0} MB");
            Console.WriteLine($"Fragmented bytes : {memInfo.FragmentedBytes / 1024:N0} KB");
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var cache = new BoundedOrderCache(100);

            // Simulate 10,000 short-lived ticks + cache writes
            for (int i = 0; i < 10_000; i++)
            {
                var tick = new MarketTick("NIFTY50", 24_500m + i, DateTime.UtcNow);
                // tick is a struct — stack allocated, no heap pressure
                cache.Add($"ORD-{i:D6}");
            }

            GCDiagnostics.PrintGCStats();
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Performance Pattern | Reason |
|---|---|
| Use `struct` for small, short-lived data (ticks, events) | Stack-allocated or Gen0 — no GC pressure |
| Use `record struct` for immutable value messages | Same benefits as struct + value equality |
| Use `ArrayPool<T>.Shared` for temporary byte arrays | Avoids LOH allocation for buffers ≥ 85KB |
| Avoid finalizers on hot-path objects | Finalizable objects promoted to Gen1 before collection |
| Use `Span<T>` for slicing arrays | Stack-based, zero allocation |
| Monitor `GC.CollectionCount(2)` in metrics | Rising Gen2 rate = memory pressure alarm |

---

## Q8. What causes memory leaks in .NET despite GC?

### 📌 Definition
A **memory leak in .NET** occurs when **managed objects remain reachable** (have at least one live reference chain to a GC root) even though they are logically no longer needed. The GC can only collect unreachable objects — if your code keeps a reference alive through a forgotten event subscription, a static collection, a long-lived cache, or a captured closure, the GC sees the object as "in use" and never reclaims it. Over time this causes the managed heap (especially Gen2) to grow unboundedly, eventually causing `OutOfMemoryException` or heavy Full GC pauses under load.

### 🔍 Why It Exists / Problem It Solves

**Root Cause 1: Forgotten event subscriptions (most common)**
```csharp
// ❌ Publisher lives forever; subscriber never unsubscribed
//    → subscriber is reachable via publisher's invocation list indefinitely
public class OrderDashboard
{
    private readonly MarketFeed _feed;
    public OrderDashboard(MarketFeed feed)
    {
        _feed = feed;
        _feed.OnPriceTick += UpdateDisplay;  // Leak: never unsubscribed
    }

    private void UpdateDisplay(decimal price) { /* ... */ }
    // OrderDashboard cannot be GC'd as long as MarketFeed is alive
}

// ✅ Fix: implement IDisposable and unsubscribe
public class OrderDashboard : IDisposable
{
    private readonly MarketFeed _feed;
    public OrderDashboard(MarketFeed feed)
    {
        _feed = feed;
        _feed.OnPriceTick += UpdateDisplay;
    }
    private void UpdateDisplay(decimal price) { /* ... */ }
    public void Dispose() => _feed.OnPriceTick -= UpdateDisplay; // Breaks the reference
}
```

**Root Cause 2: Static collections growing unboundedly**
```csharp
// ❌ Static dictionary lives for app lifetime — grows forever
public static class OrderCache
{
    private static readonly Dictionary<string, Order> _cache = new();
    public static void Add(Order o) => _cache[o.Id] = o; // Never cleaned up
}

// ✅ Fix: use bounded cache with eviction
private static readonly MemoryCache _cache = new(new MemoryCacheOptions
    { SizeLimit = 1000 });
```

**Root Cause 3: Captured closures holding large objects**
```csharp
// ❌ Lambda captures 'largeReport' (10MB object) — keeps it alive as long as 
//    the Action is alive
byte[] largeReport = File.ReadAllBytes("report.pdf"); // 10MB
Action sendReport = () => SendEmail(largeReport);     // largeReport captured in closure
_pendingActions.Add(sendReport);                       // added to long-lived list
// largeReport cannot be GC'd until sendReport is removed from _pendingActions

// ✅ Fix: capture only the minimum needed, or copy what's required
string reportPath = "report.pdf";
Action sendReport = () => SendEmail(File.ReadAllBytes(reportPath)); // reads on demand
```

**Root Cause 4: Finalizable objects in Gen1 limbo**
```csharp
// ❌ Object with finalizer gets promoted to Gen1 on first GC
//    (placed on finalizer queue, can't be collected in Gen0 pass)
//    After finalizer runs it goes back to GC — two GC cycles to collect!
public class ResourceHolder
{
    ~ResourceHolder() { /* Finalizer runs on GC thread — slow, unpredictable */ }
}

// ✅ Fix: IDisposable + GC.SuppressFinalize — explicit release, no finalizer queue
public class ResourceHolder : IDisposable
{
    private bool _disposed;
    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        // release resources
        GC.SuppressFinalize(this); // Remove from finalizer queue
    }
    ~ResourceHolder() { Dispose(); } // Safety net only
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Detecting and fixing a leak in a trading notification service**

```csharp
using System;
using System.Collections.Generic;
using Microsoft.Extensions.Caching.Memory;

namespace TradingPlatform.MemoryLeaks
{
    // Simulates long-lived publisher (app-lifetime singleton)
    public sealed class PriceAggregator
    {
        public event Action<string, decimal>? OnUpdate;
        public void Publish(string sym, decimal price) => OnUpdate?.Invoke(sym, price);
    }

    // ❌ LEAKY: subscribes but never unsubscribes
    public sealed class LeakySubscriber
    {
        public LeakySubscriber(PriceAggregator agg)
        {
            agg.OnUpdate += (sym, price) =>
                Console.WriteLine($"[LEAK] {sym}={price:C}");
            // Lambda is captured in agg.OnUpdate indefinitely
        }
    }

    // ✅ FIXED: proper IDisposable with unsubscription
    public sealed class SafeSubscriber : IDisposable
    {
        private readonly PriceAggregator _agg;
        private readonly Action<string, decimal> _handler;

        public SafeSubscriber(PriceAggregator agg)
        {
            _agg     = agg;
            _handler = OnUpdate;
            _agg.OnUpdate += _handler;
        }

        private void OnUpdate(string sym, decimal price)
            => Console.WriteLine($"[SAFE] {sym}={price:C}");

        public void Dispose()
        {
            _agg.OnUpdate -= _handler; // Breaks GC root → subscriber becomes reclaimable
            Console.WriteLine("[SAFE] Unsubscribed. Ready for GC.");
        }
    }

    // ✅ WeakReference pattern: subscriber can be GC'd even if publisher alive
    public sealed class WeakSubscriberRegistry
    {
        private readonly List<WeakReference<Action<string, decimal>>> _subs = new();

        public void Register(Action<string, decimal> handler)
            => _subs.Add(new WeakReference<Action<string, decimal>>(handler));

        public void Broadcast(string sym, decimal price)
        {
            _subs.RemoveAll(wr => !wr.TryGetTarget(out _)); // Prune dead refs
            foreach (var wr in _subs)
                if (wr.TryGetTarget(out var handler)) handler(sym, price);
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var agg = new PriceAggregator();

            // ScOPE the subscriber — Dispose() called at end of using block
            using (var safe = new SafeSubscriber(agg))
            {
                agg.Publish("NIFTY50",   24_500m);
                agg.Publish("BANKNIFTY", 52_100m);
            } // Dispose() called here — unsubscribed

            agg.Publish("SENSEX", 72_000m); // No more subscribers — no output
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Leak Source | Detection | Fix |
|---|---|---|
| Event subscriptions | dotMemory / PerfView — check invocation list of events | `IDisposable` + `-=` on Dispose |
| Static collections | Monitor heap Gen2 size over time | Bound size, use `MemoryCache` with expiry |
| Large closures | Heap snapshot diff — see lambda objects retaining large graphs | Capture minimum; null captured refs after use |
| Finalizer queue backup | `GC.GetTotalMemory()` rising; WinDbg `!finalizequeue` | `IDisposable` + `GC.SuppressFinalize` |
| Long-lived Tasks | Async method never completes, state machine stays on heap | Use `CancellationToken` with timeout |
| String interning | `string.Intern()` values never GC'd | Avoid `string.Intern` on dynamic data |

---

---

## Q9. IDisposable pattern – why and how? Implement it.

### 📌 Definition
`IDisposable` is a standard .NET interface with a single `Dispose()` method that provides a **deterministic, developer-controlled mechanism to release unmanaged resources** (file handles, database connections, network sockets, COM objects, native memory) immediately when they are no longer needed — without waiting for the Garbage Collector. The GC only manages memory; it has no knowledge of OS handles or native resources. `IDisposable` bridges this gap by giving callers a reliable `using` block / `Dispose()` call to trigger cleanup at a predictable point. The full pattern also includes a protected `Dispose(bool disposing)` virtual method and a finalizer safety net for cases where `Dispose()` is not called.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Without IDisposable — resource leak
public class DatabaseSession
{
    private SqlConnection _conn = new SqlConnection(connStr);
    // _conn.Open() called; no Close() unless caller manually does it
    // GC might eventually finalize SqlConnection — but timing is unpredictable
    // Under load: SQLExceptions from connection pool exhaustion
}

// ✅ With IDisposable — using block guarantees cleanup
using (var session = new DatabaseSession(connStr))
{
    session.ExecuteQuery("SELECT 1");
} // Dispose() called here — connection returned to pool immediately, even if exception thrown
```

**Full standard IDisposable implementation pattern:**
```csharp
// Full pattern: managed + unmanaged resources + finalizer safety net
public class ResourceManager : IDisposable
{
    private bool _disposed = false;

    // Managed resources (implement IDisposable themselves)
    private readonly SqlConnection  _connection;
    private readonly FileStream     _logFile;

    // Unmanaged resource (raw OS handle, native pointer)
    private readonly IntPtr _nativeHandle;

    public ResourceManager(string connStr, string logPath)
    {
        _connection   = new SqlConnection(connStr);
        _logFile      = File.OpenWrite(logPath);
        _nativeHandle = NativeLib.Open(); // hypothetical P/Invoke
    }

    // Public Dispose — called by consumer (using block)
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this); // Tell GC: no need to finalize, we cleaned up already
    }

    // Protected virtual: allows derived classes to extend cleanup
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return; // Idempotent — safe to call multiple times

        if (disposing)
        {
            // Dispose managed resources (only when called by Dispose(), not finalizer)
            _connection?.Dispose();
            _logFile?.Dispose();
        }

        // Always release unmanaged resources (even if called from finalizer)
        if (_nativeHandle != IntPtr.Zero)
            NativeLib.Close(_nativeHandle);

        _disposed = true;
    }

    // Finalizer — safety net if consumer forgets to call Dispose()
    // Runs on GC Finalizer thread — cannot touch managed objects (may already be collected)
    ~ResourceManager() => Dispose(disposing: false);

    // Guard all public methods against use-after-dispose
    private void ThrowIfDisposed()
    {
        if (_disposed) throw new ObjectDisposedException(nameof(ResourceManager));
    }
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Trade execution database session with pooled connections**

```csharp
using System;
using System.Data;
using Microsoft.Data.SqlClient;

namespace TradingPlatform.Data
{
    public sealed class TradeSession : IDisposable
    {
        private readonly SqlConnection  _connection;
        private SqlTransaction?         _transaction;
        private bool                    _disposed;
        private readonly string         _sessionId;

        public TradeSession(string connectionString)
        {
            _sessionId  = Guid.NewGuid().ToString("N")[..8];
            _connection = new SqlConnection(connectionString);
            _connection.Open();
            Console.WriteLine($"[{_sessionId}] Session opened. Pool size: {GetPoolCount()}");
        }

        public void BeginTransaction()
        {
            ThrowIfDisposed();
            _transaction = _connection.BeginTransaction(IsolationLevel.ReadCommitted);
            Console.WriteLine($"[{_sessionId}] Transaction started.");
        }

        public void ExecuteTrade(string orderId, decimal price)
        {
            ThrowIfDisposed();
            using var cmd = new SqlCommand(
                "INSERT INTO Trades (OrderId, Price, ExecutedAt) VALUES (@id, @price, @at)",
                _connection, _transaction);
            cmd.Parameters.AddWithValue("@id",    orderId);
            cmd.Parameters.AddWithValue("@price", price);
            cmd.Parameters.AddWithValue("@at",    DateTime.UtcNow);
            cmd.ExecuteNonQuery();
            Console.WriteLine($"[{_sessionId}] Trade {orderId} recorded.");
        }

        public void Commit()
        {
            ThrowIfDisposed();
            _transaction?.Commit();
            Console.WriteLine($"[{_sessionId}] Transaction committed.");
        }

        public void Rollback()
        {
            _transaction?.Rollback(); // Safe to call without ThrowIfDisposed (cleanup path)
            Console.WriteLine($"[{_sessionId}] Transaction rolled back.");
        }

        public void Dispose()
        {
            if (_disposed) return;
            _disposed = true;
            _transaction?.Dispose();
            _connection.Dispose(); // Returns connection to pool — not destroyed
            Console.WriteLine($"[{_sessionId}] Session disposed. Connection returned to pool.");
            GC.SuppressFinalize(this);
        }

        private void ThrowIfDisposed()
        {
            if (_disposed) throw new ObjectDisposedException(nameof(TradeSession));
        }

        private static int GetPoolCount() => 0; // production: SqlConnection pool telemetry
    }

    public static class Program
    {
        private const string ConnStr = "Server=.;Database=Trading;Integrated Security=true";

        public static void Main()
        {
            // Pattern 1: using declaration (C# 8+) — Dispose() at end of enclosing scope
            using var session = new TradeSession(ConnStr);
            session.BeginTransaction();
            try
            {
                session.ExecuteTrade("ORD-001", 24_500m);
                session.ExecuteTrade("ORD-002", 24_510m);
                session.Commit();
            }
            catch
            {
                session.Rollback();
                throw;
            }
            // Dispose() called here at end of Main — connection returned to pool
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Pattern | Reason |
|---|---|
| `sealed class` + `Dispose()` | Simple pattern for most cases — no inheritance = no `virtual Dispose(bool)` needed |
| `GC.SuppressFinalize(this)` | Removes from finalizer queue — avoids extra GC cycle and unnecessary promotion to Gen1 |
| Idempotent (double-dispose safe) | `if (_disposed) return` — calling Dispose() twice is always safe |
| Guard with `ObjectDisposedException` | Throw from all public methods if `_disposed == true` |
| `using` / `await using` | Always wrap IDisposable consumption in `using` — compiler generates try/finally |

---

## Q10. Finalizer vs Dispose – execution flow.

### 📌 Definition
- **`Dispose()`** — An explicit, **deterministic** developer-triggered cleanup method from the `IDisposable` interface. Called synchronously at a known point (end of `using` block or manual `.Dispose()`). Runs on the calling thread. Allows releasing both managed and unmanaged resources.
- **Finalizer (`~ClassName()`)** — An **implicit, non-deterministic** GC-triggered cleanup method. The GC calls finalizers on objects it finds during collection. Runs on the dedicated background **Finalizer Thread** (one per process), not the calling thread. Cannot touch managed objects safely (they may already be finalized). Timing is unpredictable — could be seconds or never if the process shuts down. Should only be a safety net.

### 🔍 Why It Exists / Problem It Solves

**Execution flow comparison:**
```
IDisposable.Dispose() path:
  Developer calls Dispose() / using block ends
       → Dispose(true) called on calling thread
       → Managed + unmanaged resources released
       → GC.SuppressFinalize(this) called
       → Object added to Gen0 (no finalizer queue) → collected normally
  
Finalizer path (developer forgot to call Dispose()):
  Object allocated → survives Gen0 GC
       → Object found alive with finalizer → MOVED TO FINALIZER QUEUE (Gen1)
       → Cannot be collected yet — must wait for finalizer to run
  Finalizer thread runs ~Cleanup()
       → Dispose(false) called — only release unmanaged resources
       → Object removed from finalizer queue
  Next GC cycle
       → Object finally collected (2 GC cycles total)
  Result: SLOWER collection, Gen1/Gen2 promotion, higher GC pressure
```

```csharp
// The two-phase dispose pattern explained:
public class HybridResource : IDisposable
{
    private IntPtr   _unmanagedHandle;         // OS resource
    private FileStream? _logStream;            // Managed resource
    private bool     _disposed;

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // ✅ Safe to touch managed objects — called from Dispose() on user thread
            _logStream?.Dispose();
            _logStream = null;
        }

        // ✅ Always release unmanaged — called from both paths
        if (_unmanagedHandle != IntPtr.Zero)
        {
            NativeLib.Free(_unmanagedHandle);
            _unmanagedHandle = IntPtr.Zero;
        }

        _disposed = true;
    }

    public    void Dispose()    { Dispose(true);  GC.SuppressFinalize(this); }
    protected ~HybridResource() { Dispose(false); } // disposing=false: don't touch managed
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Native memory buffer for zero-copy market data parsing**

```csharp
using System;
using System.Runtime.InteropServices;

namespace TradingPlatform.NativeBuffers
{
    // Wraps a native (unmanaged) memory buffer allocated via Marshal.AllocHGlobal
    // Must implement full finalizer + Dispose pattern — unmanaged memory not GC-tracked
    public sealed unsafe class NativeMarketDataBuffer : IDisposable
    {
        private IntPtr _buffer;
        private readonly int _size;
        private bool _disposed;

        public NativeMarketDataBuffer(int sizeBytes)
        {
            _size   = sizeBytes;
            _buffer = Marshal.AllocHGlobal(sizeBytes); // Allocate native heap memory
            Console.WriteLine($"[Buffer] Allocated {sizeBytes:N0} bytes at 0x{_buffer:X}");
        }

        // Write a price tick into the native buffer at given offset
        public void WriteTick(int offset, decimal price)
        {
            if (_disposed) throw new ObjectDisposedException(nameof(NativeMarketDataBuffer));
            if (offset + sizeof(decimal) > _size)
                throw new ArgumentOutOfRangeException(nameof(offset));

            *(decimal*)(_buffer + offset) = price; // Direct native memory write
        }

        public decimal ReadTick(int offset)
        {
            if (_disposed) throw new ObjectDisposedException(nameof(NativeMarketDataBuffer));
            return *(decimal*)(_buffer + offset);
        }

        // Dispose: deterministic, developer-triggered
        public void Dispose()
        {
            if (_disposed) return;
            _disposed = true;

            if (_buffer != IntPtr.Zero)
            {
                Marshal.FreeHGlobal(_buffer); // Return to native heap
                Console.WriteLine($"[Buffer] Freed native memory at 0x{_buffer:X} [Dispose()]");
                _buffer = IntPtr.Zero;
            }

            GC.SuppressFinalize(this); // Don't need finalizer now
        }

        // Finalizer: safety net if Dispose() never called
        // Runs on Finalizer thread — only touch unmanaged resources (_buffer is IntPtr, safe)
        ~NativeMarketDataBuffer()
        {
            if (_buffer != IntPtr.Zero)
            {
                Marshal.FreeHGlobal(_buffer);
                Console.WriteLine($"[Buffer] ⚠️  Native memory freed by FINALIZER — Dispose() was NOT called!");
                _buffer = IntPtr.Zero;
            }
        }
    }

    public static class Program
    {
        public static void Main()
        {
            // ✅ Correct: using block — Dispose() called deterministically
            using (var buf = new NativeMarketDataBuffer(4096))
            {
                buf.WriteTick(0,  24_500m);
                buf.WriteTick(16, 52_100m);
                Console.WriteLine($"Read: {buf.ReadTick(0):C}, {buf.ReadTick(16):C}");
            } // Dispose() fires here

            // ❌ Incorrect: GC.Collect forces finalizer to run (demo only — never do this)
            var leaked = new NativeMarketDataBuffer(1024);
            leaked = null!;
            GC.Collect();
            GC.WaitForPendingFinalizers(); // Output: "freed by FINALIZER" warning
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Dimension | `Dispose()` | Finalizer |
|---|---|---|
| **Triggered by** | Developer (`using` / explicit call) | GC (non-deterministic) |
| **Thread** | Calling thread | Dedicated Finalizer thread |
| **Timing** | Immediate, predictable | Unknown — could be never |
| **Access managed objects** | ✅ Yes | ❌ No — may be collected |
| **GC impact** | Neutral | Promotes object to Gen1 (2-cycle collection) |
| **Use case** | Primary cleanup path | Safety net only |

---

## Q11. Value types vs reference types – memory layout.

### 📌 Definition
- **Value types** (`struct`, `enum`, primitive types like `int`, `decimal`, `bool`, `DateTime`) store their **data directly where they are declared** — on the stack (for local variables) or inline within the containing object on the heap (for struct fields of a class). They are copied on assignment. Default equality is field-by-field value comparison. Value types cannot be `null` unless wrapped in `Nullable<T>` (`int?`).
- **Reference types** (`class`, `interface`, `delegate`, `record class`, `string`, arrays) store an **object header + fields on the managed heap**, and variables hold only a **pointer (object reference)** on the stack. Assignment copies the reference, not the object — both variables point to the same heap object. Default equality compares references (memory addresses), not contents.

### 🔍 Why It Exists / Problem It Solves

**Memory layout visualization:**
```
Stack                       Managed Heap
───────                     ─────────────────────────────────────
int x = 5;    → [5]
int y = x;    → [5]         (independent copy)

Order o1 = new Order();     → [ref ─────────────► [ObjHeader][orderId][price][...]]
Order o2 = o1;              → [ref ─────────────► SAME object ↑]
// mutation via o2 is visible through o1

struct Point { int X, Y; }
Point p1 = new(3, 4);  → [3][4]    (both fields inline on stack)
Point p2 = p1;         → [3][4]    (full copy — independent)
p2.X = 99;             // p1.X is still 3
```

```csharp
// Value type in a class — stored INLINE on the heap object
public class TradeRecord
{
    public int    Quantity;  // stored inline in TradeRecord heap object (no extra allocation)
    public decimal Price;    // stored inline
    public DateTime ExecutedAt; // stored inline
}

// Reference type in a class — extra heap allocation
public class TradeRecord2
{
    public string? Symbol;    // 8-byte reference on heap + separate string object on heap
    public Order?  Details;   // 8-byte reference + separate Order object on heap
}
```

**Choosing between class and struct:**
```csharp
// Use struct when:
// ✅ Small (guideline: ≤ 16 bytes; must verify with benchmarks)
// ✅ Logically represents a single value (a coordinate, a price tick)
// ✅ Short-lived — avoids heap allocation, no GC pressure
// ✅ Immutable (prevents mutable struct bugs)
public readonly record struct PriceTick(string Symbol, decimal Price, DateTime At);

// Use class when:
// ✅ Large object (struct > 16 bytes causes expensive copy overhead)
// ✅ Has identity — two instances with same data are different objects
// ✅ Needs inheritance / polymorphism
// ✅ Long-lived, shared, mutable
public class OrderBook { /* ... */ }
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Market tick processing — struct vs class performance comparison**

```csharp
using System;
using System.Diagnostics;

namespace TradingPlatform.Types
{
    // ── STRUCT: zero allocation, stack/inline, copy semantics ─────────────
    public readonly record struct TickStruct(
        string Symbol,
        decimal Bid,
        decimal Ask,
        DateTime Timestamp
    )
    {
        public decimal Spread => Ask - Bid;
    }

    // ── CLASS: heap allocated, reference semantics ─────────────────────────
    public sealed class TickClass
    {
        public string   Symbol    { get; init; } = "";
        public decimal  Bid       { get; init; }
        public decimal  Ask       { get; init; }
        public DateTime Timestamp { get; init; }
        public decimal  Spread    => Ask - Bid;
    }

    public static class ValueVsReferenceDemo
    {
        private const int Iterations = 1_000_000;

        public static void Main()
        {
            // ── Struct benchmark ────────────────────────────────────────────
            var sw = Stopwatch.StartNew();
            decimal structSum = 0;
            for (int i = 0; i < Iterations; i++)
            {
                var tick = new TickStruct("NIFTY50", 24_498m, 24_502m, DateTime.UtcNow);
                structSum += tick.Spread; // Stack allocation — no GC pressure
            }
            sw.Stop();
            Console.WriteLine($"Struct: {sw.ElapsedMilliseconds} ms | Sum={structSum:N0}");

            // ── Class benchmark ─────────────────────────────────────────────
            sw.Restart();
            decimal classSum = 0;
            for (int i = 0; i < Iterations; i++)
            {
                var tick = new TickClass   // Heap allocation every iteration
                {
                    Symbol = "NIFTY50", Bid = 24_498m, Ask = 24_502m, Timestamp = DateTime.UtcNow
                };
                classSum += tick.Spread;
            }
            sw.Stop();
            Console.WriteLine($"Class:  {sw.ElapsedMilliseconds} ms | Sum={classSum:N0}");
            // Result: Struct typically 3-5x faster — no heap allocation = no GC overhead

            // ── Copy semantics demo ─────────────────────────────────────────
            var s1 = new TickStruct("NIFTY50", 24_498m, 24_502m, DateTime.UtcNow);
            var s2 = s1;                   // Full copy of all fields
            // s2 is independent — cannot modify (readonly record struct)
            Console.WriteLine($"s1==s2: {s1 == s2}"); // True: value equality

            var c1 = new TickClass { Symbol = "NIFTY50", Bid = 24_498m, Ask = 24_502m };
            var c2 = c1;                   // Copy reference only — same heap object
            Console.WriteLine($"c1==c2: {c1 == c2}"); // True: same reference
            // (with record class: value equality; with plain class: reference equality)
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Dimension | `struct` (Value Type) | `class` (Reference Type) |
|---|---|---|
| **Storage** | Stack (locals) or inline in containing object | Managed heap |
| **Assignment** | Full copy of all fields | Copy of 8-byte reference |
| **GC pressure** | None (stack-allocated locals) | One allocation = one GC-tracked object |
| **Null** | Not nullable (use `Nullable<T>`) | Nullable by default |
| **Default equality** | Field-by-field (record struct) / bitwise | Reference equality (unless overridden) |
| **Best for** | Small, short-lived, immutable values | Large, long-lived, identity-based objects |

---

## Q12. Stack vs Heap – deep dive with examples.

### 📌 Definition
- **Stack** — A contiguous, LIFO (Last-In-First-Out) memory region managed automatically by the CPU. Each thread has its own stack (~1MB default on Windows). Stack frames are pushed on method entry and popped on return — O(1) and effectively free. Stores: stack frame metadata, local value-type variables, method parameters, and object references (not the objects themselves). Cannot be fragmented; allocation/deallocation is a single pointer increment/decrement.
- **Heap** — A large, dynamically allocated memory region managed by the CLR Garbage Collector. Shared across threads. Used for all `new` allocations of reference types. The managed heap is divided into generations (Gen0, Gen1, Gen2) and the Large Object Heap (LOH). Heap allocation involves finding space, zeroing memory, writing object header, and updating pointers — significantly more expensive than stack allocation.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ── STACK allocation: automatic, fast, deterministic ──────────────────────
void ProcessTick()
{
    int count       = 0;               // Stack: 4 bytes (local variable)
    decimal price   = 24_500m;         // Stack: 16 bytes (local decimal)
    DateTime when   = DateTime.UtcNow; // Stack: 8 bytes (DateTime is a struct)
    //
    // At method return: entire stack frame popped in O(1)
    // No GC involvement whatsoever
}

// ── HEAP allocation: managed, slower, GC-tracked ──────────────────────────
void CreateOrder()
{
    var order = new Order("ORD-001", 24_500m); // Heap: 1. find space in Gen0
                                                //       2. zero memory
                                                //       3. write sync block + type handle
                                                //       4. execute constructor
                                                //       5. stack: 8-byte reference to heap obj
    // On method return: 'order' reference popped from stack
    //                   Order object stays on heap until GC collects it
}
```

**Object layout on the heap:**
```
Managed Heap — Order object:
┌──────────────────────────────────────────────────┐
│  Sync Block Index (8 bytes)  ← used by Monitor.Enter() / lock()
│  Type Handle / Method Table pointer (8 bytes) ← vtable, type info, GC roots
│  ── Instance fields ──────────────────────────── │
│  string OrderId (8-byte reference)               │
│  decimal Price  (16 bytes inline)                │
│  DateTime At    (8 bytes inline)                 │
└──────────────────────────────────────────────────┘
Total: ~48 bytes + size of referenced objects (OrderId string on heap separately)
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Span<T> and stackalloc for zero-heap tick parsing**

```csharp
using System;
using System.Text;

namespace TradingPlatform.StackVsHeap
{
    public static class TickParser
    {
        // ✅ stackalloc: allocate on STACK, not heap — ideal for small, temporary buffers
        // Span<T>: stack-safe slice over any contiguous memory (stack, heap, or native)
        public static (string symbol, decimal price) ParseTickFast(ReadOnlySpan<char> raw)
        {
            // raw format: "NIFTY50:24500.50"
            int sep = raw.IndexOf(':');

            // Stack-allocated char array — zero heap allocation for symbol extraction
            Span<char> symbolBuf = stackalloc char[sep];
            raw[..sep].CopyTo(symbolBuf);
            string symbol = new string(symbolBuf); // One string allocation — unavoidable

            decimal price = decimal.Parse(raw[(sep + 1)..]);
            return (symbol, price);
        }

        // ❌ Naive version: multiple heap allocations
        public static (string symbol, decimal price) ParseTickSlow(string raw)
        {
            var parts  = raw.Split(':');           // new string[] — heap
            string sym = parts[0];                 // string reference
            decimal px = decimal.Parse(parts[1]);  // string Parse — heap
            return (sym, px);
        }

        public static void Main()
        {
            string tick = "NIFTY50:24500.50";

            var (sym1, px1) = ParseTickFast(tick.AsSpan()); // AsSpan: zero alloc
            Console.WriteLine($"Fast parse: {sym1} @ {px1:C}");

            var (sym2, px2) = ParseTickSlow(tick);
            Console.WriteLine($"Slow parse: {sym2} @ {px2:C}");

            // stackalloc demo for byte buffer processing
            Span<byte> buffer = stackalloc byte[64]; // 64 bytes on THIS thread's stack
            Encoding.UTF8.TryGetBytes("NIFTY50:24500", buffer, out int written);
            Console.WriteLine($"Stack-encoded {written} bytes, no heap allocation");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Dimension | Stack | Managed Heap |
|---|---|---|
| **Size** | ~1MB per thread (configurable) | GBs (limited by system RAM) |
| **Allocation cost** | O(1) — pointer decrement | Object header + zeroing + GC bookkeeping |
| **Deallocation** | O(1) — stack frame pop on method return | GC collection (Gen0 fast, Gen2 slow) |
| **Thread isolation** | Each thread has its own stack | Shared across all threads |
| **Fragmentation** | Never | Yes — LOH especially vulnerable |
| **Max object size** | ~1MB (stack overflow risk) | GB range |

**Zero-allocation patterns:**
- `Span<T>` / `Memory<T>` — slice without allocation
- `stackalloc` — stack-allocate small temporary buffers
- `ArrayPool<T>.Shared.Rent()` — pool and reuse large arrays
- `record struct` — value types that don't allocate
- `ValueTask<T>` — async without Task allocation for sync-fast paths

---

## Q13. Boxing & Unboxing – performance impact.

### 📌 Definition
**Boxing** is the implicit runtime operation of wrapping a value type (struct, int, decimal, etc.) inside a reference-type object on the managed heap, so it can be treated as `object` or an interface reference. The CLR allocates a new heap object, copies the value type's bytes into it, and returns a reference. **Unboxing** is the reverse: extracting the value type bytes back out of the heap object with an explicit cast. Both operations involve heap allocation, memory copy, and GC tracking overhead — in high-frequency hot paths (order matching, price calculation loops), boxing is a significant performance penalty.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Boxing occurs when value type is assigned to object / interface reference
int price = 24500;
object boxed = price;         // BOX: heap allocates new object, copies int bytes
int unboxed = (int)boxed;     // UNBOX: type-checks, copies bytes back to stack

// Visualizing the boxing heap allocation:
// Stack: [price=24500] [boxed = 0x7F3A2C10 ──────────────────────►]
//                                                                    Heap:
//                                                              [SyncBlock][TypeHandle]
//                                                              [24500 bytes copied here]

// Common hidden boxing traps:
string msg = string.Format("Price: {0}", price);   // ❌ {0} parameter is object → boxes int
Console.WriteLine(price);                           // ❌ Console.WriteLine(object) → boxes
ArrayList list = new(); list.Add(price);            // ❌ ArrayList is pre-generics → boxes

// ✅ Fixes — generics prevent boxing (T is known at compile time → no object cast)
string msg2    = $"Price: {price}";                 // ✅ string interpolation avoids box
List<int> list2 = new(); list2.Add(price);          // ✅ List<T> — no boxing
```

**Interface boxing trap (hidden!):**
```csharp
public interface IPriceable { decimal GetPrice(); }
public struct Order : IPriceable
{
    public decimal Price { get; init; }
    public decimal GetPrice() => Price;
}

// ❌ Calling through interface BOXES the struct!
IPriceable p = new Order { Price = 24500m }; // Boxing occurs here
decimal price = p.GetPrice();                // Works, but heap allocation happened

// ✅ Use generics with constraints to avoid boxing:
static decimal GetPrice<T>(T item) where T : IPriceable
    => item.GetPrice(); // T is known as Order — no boxing, direct call
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Order book operations — ArrayList (boxing) vs List<T> (no boxing)**

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Diagnostics;

namespace TradingPlatform.Boxing
{
    public readonly record struct OrderEntry(string Symbol, decimal Quantity, decimal Price);

    public static class BoxingDemo
    {
        private const int N = 500_000;

        public static void Main()
        {
            // ❌ BOXING PATH: ArrayList stores object → every struct is boxed
            var sw = Stopwatch.StartNew();
            var legacyBook = new ArrayList();
            for (int i = 0; i < N; i++)
                legacyBook.Add(new OrderEntry("NIFTY50", i, 24_000m + i)); // Boxing here!

            decimal legacySum = 0;
            foreach (object item in legacyBook)
                legacySum += ((OrderEntry)item).Price; // Unboxing here!
            sw.Stop();
            Console.WriteLine($"ArrayList (boxing):    {sw.ElapsedMilliseconds} ms | Sum={legacySum:N0}");

            // ✅ NO BOXING PATH: List<T> preserves value type layout inline
            sw.Restart();
            var modernBook = new List<OrderEntry>();
            for (int i = 0; i < N; i++)
                modernBook.Add(new OrderEntry("NIFTY50", i, 24_000m + i)); // No boxing

            decimal modernSum = 0;
            foreach (var entry in modernBook)
                modernSum += entry.Price; // Direct struct access — no cast
            sw.Stop();
            Console.WriteLine($"List<T>  (no boxing):  {sw.ElapsedMilliseconds} ms | Sum={modernSum:N0}");

            // Formatted string boxing comparison
            int quantity = 100;
            sw.Restart();
            for (int i = 0; i < N; i++)
                _ = string.Format("Qty: {0}", quantity); // Boxes int N times
            Console.WriteLine($"string.Format (boxing): {sw.ElapsedMilliseconds} ms");

            sw.Restart();
            for (int i = 0; i < N; i++)
                _ = $"Qty: {quantity}"; // Interpolated string — no boxing via ISpanFormattable
            Console.WriteLine($"Interpolation (no box): {sw.ElapsedMilliseconds} ms");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Scenario | Boxing? | Fix |
|---|---|---|
| `ArrayList.Add(struct)` | ✅ Yes | Use `List<T>` |
| `Dictionary<object, T>` | ✅ Yes | Use `Dictionary<TKey, T>` |
| `string.Format("{0}", int)` | ✅ Yes | Use `$"{int}"` or `ToString()` |
| `Console.WriteLine(int)` | ✅ Yes | `Console.WriteLine(int.ToString())` |
| Struct implementing interface (non-generic) | ✅ Yes | Generic constraints `where T : IFoo` |
| `Nullable<T>.GetValueOrDefault()` | ✅ Yes (implicit) | Direct comparison against `null` |

---

---

## Q14. What are records? How are they different from classes?

### 📌 Definition
A **record** (introduced in C# 9) is a class or struct declaration that the compiler enriches with **value-based equality, non-destructive mutation (`with` expressions), positional construction, and a human-readable `ToString()`** — all auto-generated. Unlike classes where `==` compares references and `Equals()` compares references by default, records generate `Equals()` and `==` that compare all declared properties by value. Records come in two flavors: **`record class`** (heap-allocated reference type, default when you write `record`) and **`record struct`** (stack-compatible value type, C# 10+). They are specifically designed for **immutable data-transfer objects, domain events, DTOs, and value objects** — scenarios where data identity is defined by content, not memory address.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ BEFORE records — verbose boilerplate for a simple immutable data object:
public sealed class OrderLegacy : IEquatable<OrderLegacy>
{
    public string  OrderId  { get; }
    public string  Symbol   { get; }
    public decimal Price    { get; }

    public OrderLegacy(string id, string sym, decimal price)
        => (OrderId, Symbol, Price) = (id, sym, price);

    public bool Equals(OrderLegacy? other)
        => other is not null && OrderId == other.OrderId
                             && Symbol  == other.Symbol
                             && Price   == other.Price;
    public override bool Equals(object? obj) => Equals(obj as OrderLegacy);
    public override int  GetHashCode() => HashCode.Combine(OrderId, Symbol, Price);
    public static bool operator ==(OrderLegacy? l, OrderLegacy? r) => l?.Equals(r) ?? r is null;
    public static bool operator !=(OrderLegacy? l, OrderLegacy? r) => !(l == r);
    public override string ToString() => $"Order {{ Id={OrderId}, Symbol={Symbol}, Price={Price} }}";
    // + copy constructor for 'with' semantics... another 10 lines
}

// ✅ WITH record — all of the above in ONE line:
public sealed record Order(string OrderId, string Symbol, decimal Price);
// Compiler generates: constructor, init-only properties, Equals, ==, GetHashCode, ToString, with-expression support
```

**Record vs Class comparison:**
```csharp
public record Order(string OrderId, string Symbol, decimal Price);

var o1 = new Order("ORD-1", "NIFTY50", 24500m);
var o2 = new Order("ORD-1", "NIFTY50", 24500m);
var o3 = o1;

Console.WriteLine(o1 == o2); // true  — value equality (same content)
Console.WriteLine(o1 == o3); // true  — same reference, same values

// Non-destructive mutation with 'with' expression
var o4 = o1 with { Price = 24600m }; // New record; o1 unchanged
Console.WriteLine(o1.Price);  // 24500 — original untouched
Console.WriteLine(o4.Price);  // 24600 — new record

// Deconstruction (positional records only)
var (id, sym, price) = o1;
Console.WriteLine($"{id}: {sym} @ {price:C}");

// Record class vs record struct
record struct Point(int X, int Y);    // value type, stack-allocated, mutable by default
record class  Event(string Name);     // reference type, heap-allocated (default 'record')
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Domain events in an event-sourced order management system**

```csharp
using System;
using System.Collections.Generic;

namespace TradingPlatform.Domain.Events
{
    // ── BASE: abstract record for all domain events ───────────────────────
    public abstract record DomainEvent(
        Guid      EventId    = default,
        DateTime  OccurredAt = default
    )
    {
        // Default values assigned via property initializer
        public Guid     EventId    { get; init; } = EventId    == default ? Guid.NewGuid()   : EventId;
        public DateTime OccurredAt { get; init; } = OccurredAt == default ? DateTime.UtcNow : OccurredAt;
    }

    // ── SPECIFIC EVENTS — inherit from DomainEvent ─────────────────────────
    public sealed record OrderPlaced(
        string  OrderId,
        string  Symbol,
        decimal Quantity,
        decimal Price,
        string  TraderId
    ) : DomainEvent;

    public sealed record OrderExecuted(
        string  OrderId,
        decimal ExecutedPrice,
        decimal SlippageBps  // slippage in basis points
    ) : DomainEvent;

    public sealed record OrderCancelled(
        string  OrderId,
        string  Reason
    ) : DomainEvent;

    // ── EVENT STORE — demonstrates value equality and pattern matching ─────
    public sealed class EventStore
    {
        private readonly List<DomainEvent> _events = new();

        public void Append(DomainEvent ev)
        {
            _events.Add(ev);
            Console.WriteLine($"[EventStore] Stored: {ev}"); // ToString() auto-generated
        }

        // Pattern matching on record types
        public void Replay()
        {
            Console.WriteLine("\n[EventStore] Replaying events:");
            foreach (var ev in _events)
            {
                switch (ev)
                {
                    case OrderPlaced p:
                        Console.WriteLine($"  PLACED: {p.Symbol} x{p.Quantity} @ {p.Price:C} by {p.TraderId}");
                        break;
                    case OrderExecuted e:
                        Console.WriteLine($"  EXEC:   {e.OrderId} @ {e.ExecutedPrice:C} | Slippage={e.SlippageBps}bps");
                        break;
                    case OrderCancelled c:
                        Console.WriteLine($"  CANCEL: {c.OrderId} — {c.Reason}");
                        break;
                }
            }
        }

        // Value equality enables deduplication
        public bool IsDuplicate(DomainEvent ev)
            => _events.Contains(ev); // Uses record value equality
    }

    // ── ORDER AGGREGATE ───────────────────────────────────────────────────
    public sealed class OrderAggregate
    {
        private readonly EventStore _store = new();

        public void Place(string orderId, string symbol, decimal qty, decimal price, string trader)
        {
            var ev = new OrderPlaced(orderId, symbol, qty, price, trader);
            if (!_store.IsDuplicate(ev)) _store.Append(ev);
        }

        public void Execute(string orderId, decimal execPrice)
        {
            decimal slippage = Math.Round((execPrice - 24500m) / 24500m * 10000m, 1);
            _store.Append(new OrderExecuted(orderId, execPrice, slippage));
        }

        public void Cancel(string orderId, string reason)
            => _store.Append(new OrderCancelled(orderId, reason));

        public void Replay() => _store.Replay();
    }

    public static class Program
    {
        public static void Main()
        {
            var aggregate = new OrderAggregate();
            aggregate.Place("ORD-001", "NIFTY50", 10, 24_500m, "TRADER-42");
            aggregate.Execute("ORD-001", 24_512m);
            aggregate.Place("ORD-002", "BANKNIFTY", 5, 52_100m, "TRADER-42");
            aggregate.Cancel("ORD-002", "Risk limit breach");

            // 'with' expression: create modified event for correction flows
            var original   = new OrderPlaced("ORD-001", "NIFTY50", 10, 24_500m, "TRADER-42");
            var corrected  = original with { Price = 24_510m };
            Console.WriteLine($"\nOriginal:  {original.Price:C}");
            Console.WriteLine($"Corrected: {corrected.Price:C}");
            Console.WriteLine($"Equal?     {original == corrected}"); // false — different price

            aggregate.Replay();
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Feature | `record class` | `record struct` | `class` |
|---|---|---|---|
| **Equality** | Value-based | Value-based | Reference-based |
| **Allocation** | Heap | Stack / inline | Heap |
| **Mutability** | Immutable (init-only) by default | Mutable by default | Configurable |
| **`with` expression** | ✅ | ✅ | ❌ |
| **Deconstruct** | ✅ positional | ✅ positional | Manual |
| **Best for** | DTOs, domain events, responses | Small immutable values | Entities, services |

---

## Q15. Immutable objects – why important?

### 📌 Definition
An **immutable object** is one whose state cannot be changed after construction. All fields are set in the constructor (or via `init`-only properties) and no public or protected mutation paths exist. Immutability is a design choice that makes objects **inherently thread-safe** (no lock needed to read them since they never change), **easier to reason about** (no defensive copying needed), **safe to share across contexts** (functions, threads, async continuations), and well-suited for caching and value semantics. In .NET, immutability is achieved via `readonly` fields, `init`-only properties, `record` types, `ImmutableArray<T>`, and `IReadOnlyList<T>`.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ MUTABLE object — thread-unsafe, unpredictable sharing
public class Order
{
    public string Symbol { get; set; } = "";
    public decimal Price  { get; set; }
}
var order = new Order { Symbol = "NIFTY50", Price = 24500m };
// Any thread can modify order.Price at any time — racing mutations, bugs

// ✅ IMMUTABLE — once created, cannot change; safe to share across threads
public sealed record Order(string Symbol, decimal Price);
var order = new Order("NIFTY50", 24500m);
// Reads from any thread are always consistent — no locks needed

// Immutability patterns in .NET:
// 1. readonly fields
public sealed class Config
{
    private readonly string _connStr;         // Assignable only in constructor
    public Config(string connStr) => _connStr = connStr;
}

// 2. init-only properties (C# 9+)
public sealed class TradeConfig
{
    public string Exchange { get; init; } = "NSE";  // Set only during object initialization
    public int    MaxLots  { get; init; } = 100;
}
var cfg = new TradeConfig { Exchange = "BSE", MaxLots = 50 };
// cfg.Exchange = "NSE"; ← COMPILE ERROR after construction

// 3. ImmutableArray<T> (System.Collections.Immutable)
using System.Collections.Immutable;
ImmutableArray<string> symbols = ImmutableArray.Create("NIFTY50", "BANKNIFTY", "SENSEX");
// symbols.Add(...)  → returns a NEW ImmutableArray, original unchanged

// 4. string — .NET's most famous immutable reference type
string s = "NIFTY";
string t = s.ToUpperInvariant(); // Returns new string; s unchanged
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Immutable order book snapshot shared safely across threads**

```csharp
using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using System.Threading;
using System.Threading.Tasks;

namespace TradingPlatform.Immutability
{
    // ── Immutable price level — safe to share across threads ──────────────
    public sealed record PriceLevel(decimal Price, decimal TotalQuantity, int OrderCount)
    {
        public PriceLevel AddOrder(decimal qty) =>
            this with { TotalQuantity = TotalQuantity + qty, OrderCount = OrderCount + 1 };
    }

    // ── Immutable order book snapshot ─────────────────────────────────────
    public sealed class OrderBookSnapshot
    {
        public string Symbol { get; }
        public ImmutableSortedDictionary<decimal, PriceLevel> Bids { get; }
        public ImmutableSortedDictionary<decimal, PriceLevel> Asks { get; }
        public DateTime CapturedAt { get; }

        public OrderBookSnapshot(
            string symbol,
            ImmutableSortedDictionary<decimal, PriceLevel> bids,
            ImmutableSortedDictionary<decimal, PriceLevel> asks)
        {
            Symbol     = symbol;
            Bids       = bids;
            Asks       = asks;
            CapturedAt = DateTime.UtcNow;
        }

        public decimal BestBid => Bids.IsEmpty ? 0m : Bids.Keys[^1]; // highest bid
        public decimal BestAsk => Asks.IsEmpty ? 0m : Asks.Keys[0];  // lowest ask
        public decimal Spread  => BestAsk - BestBid;
    }

    // ── Mutable order book that publishes immutable snapshots ─────────────
    public sealed class LiveOrderBook
    {
        private readonly object _lock = new();
        private readonly string _symbol;
        private readonly SortedDictionary<decimal, PriceLevel> _bids = new(Comparer<decimal>.Create((a, b) => b.CompareTo(a)));
        private readonly SortedDictionary<decimal, PriceLevel> _asks = new();

        public LiveOrderBook(string symbol) => _symbol = symbol;

        public void AddBid(decimal price, decimal qty)
        {
            lock (_lock)
            {
                _bids[price] = _bids.TryGetValue(price, out var existing)
                    ? existing.AddOrder(qty)
                    : new PriceLevel(price, qty, 1);
            }
        }

        public void AddAsk(decimal price, decimal qty)
        {
            lock (_lock)
            {
                _asks[price] = _asks.TryGetValue(price, out var existing)
                    ? existing.AddOrder(qty)
                    : new PriceLevel(price, qty, 1);
            }
        }

        // Returns an IMMUTABLE snapshot — callers can read it from any thread safely
        public OrderBookSnapshot GetSnapshot()
        {
            lock (_lock)
            {
                return new OrderBookSnapshot(
                    _symbol,
                    _bids.ToImmutableSortedDictionary(Comparer<decimal>.Create((a, b) => b.CompareTo(a))),
                    _asks.ToImmutableSortedDictionary()
                );
            }
        }
    }

    public static class Program
    {
        public static async Task Main()
        {
            var book = new LiveOrderBook("NIFTY50");
            book.AddBid(24_498m, 10);
            book.AddBid(24_499m, 5);
            book.AddAsk(24_501m, 8);
            book.AddAsk(24_502m, 12);

            // Snapshot is immutable — share across 5 concurrent readers safely
            var snapshot = book.GetSnapshot();
            Console.WriteLine($"Best Bid: {snapshot.BestBid:C} | Best Ask: {snapshot.BestAsk:C} | Spread: {snapshot.Spread:C}");

            await Task.WhenAll(Enumerable.Range(0, 5).Select(i => Task.Run(() =>
            {
                // Multiple threads read the same immutable snapshot — no locks needed
                Console.WriteLine($"  [Thread {i}] Spread: {snapshot.Spread:C}");
            })));
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Pattern | Mechanism | Thread-safe reads? |
|---|---|---|
| `readonly` field | Constructor-only assignment | ✅ Yes |
| `init` property | Object initializer-only | ✅ Yes |
| `record` / `record struct` | Compiler-generated immutability | ✅ Yes |
| `ImmutableArray<T>` / `ImmutableDictionary<K,V>` | Returns new instance on modification | ✅ Yes |
| `string` | All operations return new strings | ✅ Yes |

**Design rule:** Prefer immutable objects for messages, events, snapshots, configurations, and DTOs. Reserve mutability for aggregates/entities where state change is the intent, and protect mutable state with locks or `Interlocked` operations.

---

## Q16. Span\<T\> and Memory\<T\> – when and why?

### 📌 Definition
`Span<T>` is a **stack-only, ref struct** that represents a contiguous slice of memory — regardless of whether that memory lives on the stack (`stackalloc`), the managed heap (arrays, strings), or the native heap (unmanaged memory) — **without copying it**. It enables zero-allocation, high-performance operations on contiguous data. Because it is a `ref struct`, it cannot be boxed, stored on the heap, or used in async methods. `Memory<T>` is the **heap-safe counterpart** — it wraps the same concept as a struct (not `ref struct`), letting you store slices in fields, lists, or async state machines. Together they replace `byte[]` copies and `string.Substring()` allocations in high-throughput parsing and serialization code.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Traditional string/array slicing — allocates new objects per slice
string raw = "NIFTY50:24500.50:BUY";
string[] parts = raw.Split(':');   // Allocates: new string[], 3 new strings
string symbol = parts[0];          // new string on heap
decimal price  = decimal.Parse(parts[1]); // another string on heap

// ✅ Span<char>: zero-allocation slicing over the SAME string memory
ReadOnlySpan<char> span = raw.AsSpan(); // Zero allocation — just a pointer + length
int p1 = span.IndexOf(':');
ReadOnlySpan<char> symbolSpan = span[..p1];                    // No new string
int p2 = span.IndexOf(':', p1 + 1);
ReadOnlySpan<char> priceSpan  = span[(p1+1)..p2];             // No new string
decimal price2 = decimal.Parse(priceSpan);                     // Parses directly from span

// stackalloc with Span<T>: temporary buffer on stack — zero heap allocation
Span<byte> buffer   = stackalloc byte[256];
int written = Encoding.UTF8.GetBytes(raw.AsSpan(), buffer);
// buffer is stack memory — automatically freed on method return, no GC
```

**Span<T> vs Memory<T>:**
```csharp
// Span<T>: stack-only — cannot escape to heap
// ✅ Can use in synchronous methods
// ❌ Cannot store in fields, lists, or async methods
ref struct SpanUser
{
    void Parse(Span<byte> data) { /* use Span here */ }
    // Cannot: private Span<byte> _stored; ← compile error (ref struct field in regular struct)
}

// Memory<T>: heap-safe — can store anywhere
// ✅ Can store in fields, pass to async methods
// ✅ Converts to Span<T> on demand via .Span property
public sealed class AsyncParser
{
    private readonly Memory<byte> _buffer; // ✅ OK in a class field

    public AsyncParser(byte[] data) => _buffer = data.AsMemory();

    public async Task ParseAsync(CancellationToken ct)
    {
        await Task.Yield(); // suspension point — Span<T> not allowed across await
        Span<byte> span = _buffer.Span; // Get Span for synchronous processing
        ProcessSlice(span[..100]);
    }

    private void ProcessSlice(Span<byte> slice) { /* fast synchronous processing */ }
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Zero-copy FIX protocol message parser for order book updates**

```csharp
using System;
using System.Text;
using System.Threading.Tasks;

namespace TradingPlatform.Network
{
    // FIX protocol message example: "8=FIX.4.4|35=D|49=BROKER|56=EXCHANGE|11=ORD001|55=NIFTY50|44=24500.50|38=10"
    // | = field delimiter (SOH in real FIX, using | for readability)

    public readonly ref struct FixMessage
    {
        private readonly ReadOnlySpan<char> _raw;

        public FixMessage(ReadOnlySpan<char> raw) => _raw = raw;

        // Extract field value by tag number — zero allocation, zero copy
        public ReadOnlySpan<char> GetField(int tag)
        {
            ReadOnlySpan<char> search = _raw;
            Span<char> tagPrefix = stackalloc char[16];
            int len = tag.TryFormat(tagPrefix, out int written) ? written : 0;
            tagPrefix[len++] = '=';
            ReadOnlySpan<char> prefix = tagPrefix[..len];

            while (!search.IsEmpty)
            {
                int pipe = search.IndexOf('|');
                ReadOnlySpan<char> field = pipe < 0 ? search : search[..pipe];

                if (field.StartsWith(prefix))
                    return field[len..]; // Return slice of original memory — no copy

                search = pipe < 0 ? ReadOnlySpan<char>.Empty : search[(pipe + 1)..];
            }
            return ReadOnlySpan<char>.Empty;
        }
    }

    public static class FixParser
    {
        public static void ParseAndProcess(ReadOnlySpan<char> rawMessage)
        {
            var msg = new FixMessage(rawMessage);

            // All field extractions: ZERO allocations — slices over original memory
            ReadOnlySpan<char> orderId = msg.GetField(11);  // tag 11 = ClOrdID
            ReadOnlySpan<char> symbol  = msg.GetField(55);  // tag 55 = Symbol
            ReadOnlySpan<char> price   = msg.GetField(44);  // tag 44 = Price
            ReadOnlySpan<char> qty     = msg.GetField(38);  // tag 38 = OrderQty

            decimal parsedPrice = decimal.Parse(price);
            int     parsedQty   = int.Parse(qty);

            Console.WriteLine($"Order: {orderId.ToString()} | {symbol.ToString()} x{parsedQty} @ {parsedPrice:C}");
        }

        public static void Main()
        {
            // Simulated incoming FIX message as stack-allocated buffer
            string raw = "8=FIX.4.4|35=D|49=BROKER|56=EXCHANGE|11=ORD001|55=NIFTY50|44=24500.50|38=10";

            // Process as Span — no heap allocation for parsing
            ParseAndProcess(raw.AsSpan());

            // Memory<T> for async pipeline
            byte[] buffer = Encoding.UTF8.GetBytes(raw);
            Memory<byte> mem = buffer.AsMemory();

            // Slice: first 20 bytes header — no copy
            Memory<byte> header = mem[..20];
            Memory<byte> body   = mem[20..];
            Console.WriteLine($"Header: {header.Length} bytes | Body: {body.Length} bytes");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Type | Stack-only | Async-safe | Store in field | Over heap/stack/native |
|---|---|---|---|---|
| `Span<T>` | ✅ (`ref struct`) | ❌ | ❌ | ✅ |
| `ReadOnlySpan<T>` | ✅ | ❌ | ❌ | ✅ |
| `Memory<T>` | ❌ | ✅ | ✅ | Heap only |
| `ReadOnlyMemory<T>` | ❌ | ✅ | ✅ | Heap only |

**Performance rules:**
- Use `Span<T>` in hot synchronous parsing loops — eliminates all substring/Split allocations
- Use `Memory<T>` when you need to pass slices to `async` methods or store in class fields
- Use `stackalloc` + `Span<T>` for small temporary buffers (< 1KB safe guideline)
- Use `ArrayPool<T>.Shared.Rent()` + `Memory<T>` for larger buffers that must survive on heap

---

## Q17. Reflection – how it works internally?

### 📌 Definition
**Reflection** is the runtime capability to **inspect, query, and dynamically invoke type metadata** (types, methods, properties, fields, attributes, constructors) that the CLR bakes into every assembly as metadata tables. When you compile C# code, Roslyn writes not just IL bytecode but also rich metadata — a formal description of every type, member, parameter, and custom attribute. The CLR's `System.Reflection` API provides access to these metadata tables at runtime through `Type`, `MethodInfo`, `PropertyInfo`, `FieldInfo`, and `ConstructorInfo` objects. Because reflection bypasses compile-time type checking and involves metadata table lookups and dynamic dispatch, it is significantly slower than direct code — reserve it for framework/tooling code, not hot business logic paths.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Without reflection: adding a new type requires changing framework code
// (tight coupling, closed to extension)
void Serialize(object obj)
{
    if (obj is Order o)         { SerializeOrder(o); }
    else if (obj is Trade t)    { SerializeTrade(t); }
    // ... must update this every time a new type is added
}

// With reflection: framework discovers and writes all properties dynamically
void Serialize(object obj)
{
    Type type = obj.GetType(); // Lookup type metadata
    foreach (PropertyInfo prop in type.GetProperties(BindingFlags.Public | BindingFlags.Instance))
    {
        object? value = prop.GetValue(obj); // Dynamic property read
        Console.WriteLine($"{prop.Name}: {value}");
    }
    // Works for ANY type without code changes — open for extension
}
```

**Internal mechanics:**
```csharp
// What the CLR does internally:
// 1. Assembly metadata table (Module.ResolveMethod, TypeDef/TypeRef tables)
// 2. Type.GetType() → looks up type by name in loaded assemblies
// 3. MethodInfo.Invoke() → constructs argument array, calls MethodBase.Invoke via P/Invoke to CLR
//    Cost: argument boxing (value types), security demand, IL stub generation, virtual dispatch
// 4. Expression trees or compiled delegates: compile once, fast repeated calls

// ❌ SLOW: reflection invoke in a loop (100x slower than direct call)
var method = typeof(Calculator).GetMethod("Add")!;
for (int i = 0; i < 1_000_000; i++)
    method.Invoke(null, new object[] { 1, 2 }); // Boxing + virtual dispatch every iteration

// ✅ FAST: compile reflection to a delegate once, call repeatedly
Func<int, int, int> addDelegate = method.CreateDelegate<Func<int, int, int>>();
for (int i = 0; i < 1_000_000; i++)
    addDelegate(1, 2); // Direct call — JIT-optimized, no boxing
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Configuring objects from appsettings without hardcoding property names (mini-mapper)**

```csharp
using System;
using System.Collections.Generic;
using System.Reflection;

namespace TradingPlatform.Reflection
{
    // Target POCO — populated from dictionary without hardcoding property names
    public sealed class TradingConfig
    {
        public string  Exchange       { get; set; } = "NSE";
        public int     MaxOrdersPerSec { get; set; } = 1000;
        public decimal MaxOrderValue  { get; set; } = 500_000m;
        public bool    PaperTrade     { get; set; } = false;
    }

    // Mark properties that can be overridden from environment
    [AttributeUsage(AttributeTargets.Property)]
    public sealed class ConfigurableAttribute : Attribute
    {
        public string Key { get; }
        public ConfigurableAttribute(string key) => Key = key;
    }

    // ── Reflection-based property mapper (runs once at startup) ───────────
    public static class PropertyMapper
    {
        // Cache compiled setters — pay reflection cost once, call fast thereafter
        private static readonly Dictionary<string, Action<object, string>> _setters = new();

        public static T MapFromDictionary<T>(Dictionary<string, string> source) where T : new()
        {
            var instance = new T();
            Type type    = typeof(T);

            foreach (PropertyInfo prop in type.GetProperties(BindingFlags.Public | BindingFlags.Instance))
            {
                if (!prop.CanWrite) continue;

                string key = prop.Name;
                if (!source.TryGetValue(key, out string? rawValue)) continue;

                // Convert string value to property type
                object? converted = Convert.ChangeType(rawValue, prop.PropertyType);
                prop.SetValue(instance, converted); // Reflection set

                Console.WriteLine($"  [Mapper] Set {type.Name}.{prop.Name} = {converted}");
            }
            return instance;
        }

        // Compile to Action<object, value> for repeated hot-path calls
        public static Action<object, string>? GetFastSetter(PropertyInfo prop)
        {
            string cacheKey = $"{prop.DeclaringType!.FullName}.{prop.Name}";
            if (_setters.TryGetValue(cacheKey, out var cached)) return cached;

            var setter = prop.GetSetMethod();
            if (setter is null) return null;

            // Emit a compiled lambda — fast as direct call
            Action<object, string> compiled = (obj, val) =>
                setter.Invoke(obj, new object[] { Convert.ChangeType(val, prop.PropertyType)! });

            _setters[cacheKey] = compiled;
            return compiled;
        }
    }

    public static class Program
    {
        public static void Main()
        {
            // Simulate config dictionary (e.g., from appsettings.json or env vars)
            var configSource = new Dictionary<string, string>
            {
                ["Exchange"]         = "BSE",
                ["MaxOrdersPerSec"]  = "500",
                ["MaxOrderValue"]    = "250000",
                ["PaperTrade"]       = "True"
            };

            Console.WriteLine("Mapping config:");
            var config = PropertyMapper.MapFromDictionary<TradingConfig>(configSource);

            Console.WriteLine($"\nResult: Exchange={config.Exchange}, MaxOrders={config.MaxOrdersPerSec}, " +
                              $"MaxValue={config.MaxOrderValue:C}, Paper={config.PaperTrade}");

            // Type inspection
            Type t = typeof(TradingConfig);
            Console.WriteLine($"\nType: {t.FullName} | Assembly: {t.Assembly.GetName().Name}");
            Console.WriteLine("Properties:");
            foreach (var prop in t.GetProperties())
                Console.WriteLine($"  {prop.PropertyType.Name} {prop.Name} [CanWrite={prop.CanWrite}]");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Operation | Cost | Mitigation |
|---|---|---|
| `Type.GetType()` | Medium — assembly scan | Cache `Type` reference in static field |
| `PropertyInfo.GetValue()` | High — boxing, virtual dispatch | Compile to `Func<T,TResult>` delegate |
| `MethodInfo.Invoke()` | High | `method.CreateDelegate<TDelegate>()` |
| `Activator.CreateInstance()` | High | Cache compiled `new()` factory |
| `Assembly.GetTypes()` | Very high — scan all types | Run once at startup, cache results |

**When to use reflection:** Dependency injection containers (scan assemblies for types), ORMs (map columns to properties), JSON serializers, validation frameworks, plugin systems. Never in tight loops or hot business paths.

---

## Q18. Expression trees – real-world use case.

### 📌 Definition
An **expression tree** (`System.Linq.Expressions.Expression<TDelegate>`) is a data structure that represents code **as a tree of objects** (nodes) that can be inspected, traversed, modified, and compiled at runtime — rather than executing it directly. When you assign a lambda to `Expression<Func<T, bool>>` instead of `Func<T, bool>`, the compiler emits code that builds the expression tree object graph rather than compiling the lambda to IL. This allows frameworks (Entity Framework, LINQ-to-SQL, AutoMapper) to **translate the lambda into another language** (SQL in EF's case) by walking the expression tree nodes and generating the equivalent query string.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Func<T,bool>: compiled IL — can only be CALLED, not inspected
Func<Order, bool> funcFilter = o => o.Price > 24000m && o.Symbol == "NIFTY50";
// What's inside? Unknown — just a delegate pointer to native code. Cannot inspect conditions.

// Expression<Func<T,bool>>: expression TREE — can be inspected and translated
Expression<Func<Order, bool>> exprFilter = o => o.Price > 24000m && o.Symbol == "NIFTY50";
// exprFilter.Body = BinaryExpression(AndAlso,
//   BinaryExpression(GreaterThan, MemberAccess(o, Price), Constant(24000)),
//   BinaryExpression(Equal,       MemberAccess(o, Symbol),  Constant("NIFTY50")))
// EF Core / LINQ provider reads this tree and generates:
// WHERE [Price] > 24000 AND [Symbol] = 'NIFTY50'  ← executes on SQL Server, not in .NET!

// You can compile an expression tree to a delegate when you want to execute it in-process:
Func<Order, bool> compiled = exprFilter.Compile();
bool result = compiled(new Order("ORD1", "NIFTY50", 24500m)); // true
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Dynamic query builder for the order search API**

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;

namespace TradingPlatform.DynamicQuery
{
    public sealed record Order(string OrderId, string Symbol, decimal Price, int Quantity, string Status);

    // ── Dynamic filter builder using Expression Trees ─────────────────────
    public static class OrderQueryBuilder
    {
        // Builds a combined predicate expression from a search DTO
        public static Expression<Func<Order, bool>> BuildFilter(OrderSearchRequest req)
        {
            var param = Expression.Parameter(typeof(Order), "o");

            // Start with a "true" expression (always matches)
            Expression body = Expression.Constant(true);

            if (!string.IsNullOrEmpty(req.Symbol))
            {
                // o.Symbol == req.Symbol
                var symbolProp = Expression.Property(param, nameof(Order.Symbol));
                var symbolVal  = Expression.Constant(req.Symbol);
                body = Expression.AndAlso(body, Expression.Equal(symbolProp, symbolVal));
            }

            if (req.MinPrice.HasValue)
            {
                // o.Price >= req.MinPrice
                var priceProp = Expression.Property(param, nameof(Order.Price));
                var minPriceVal = Expression.Constant(req.MinPrice.Value);
                body = Expression.AndAlso(body, Expression.GreaterThanOrEqual(priceProp, minPriceVal));
            }

            if (!string.IsNullOrEmpty(req.Status))
            {
                // o.Status == req.Status
                var statusProp = Expression.Property(param, nameof(Order.Status));
                var statusVal  = Expression.Constant(req.Status);
                body = Expression.AndAlso(body, Expression.Equal(statusProp, statusVal));
            }

            return Expression.Lambda<Func<Order, bool>>(body, param);
        }

        // In EF Core: query.Where(BuildFilter(req)) → translated to SQL WHERE clause
        // In-memory for demo: compile the expression and apply to list
        public static List<Order> Search(IEnumerable<Order> orders, OrderSearchRequest req)
        {
            var filter    = BuildFilter(req);
            var compiled  = filter.Compile(); // Compile once, reuse

            // Print the generated expression tree (for debugging/logging)
            Console.WriteLine($"[QueryBuilder] Expression: {filter}");

            return orders.Where(compiled).ToList();
        }
    }

    public sealed class OrderSearchRequest
    {
        public string? Symbol   { get; init; }
        public decimal? MinPrice { get; init; }
        public string?  Status  { get; init; }
    }

    public static class Program
    {
        public static void Main()
        {
            var orders = new List<Order>
            {
                new("ORD-001", "NIFTY50",   24500m, 10, "FILLED"),
                new("ORD-002", "BANKNIFTY", 52100m, 5,  "OPEN"),
                new("ORD-003", "NIFTY50",   24300m, 20, "OPEN"),
                new("ORD-004", "SENSEX",    72000m, 1,  "FILLED"),
                new("ORD-005", "NIFTY50",   24800m, 8,  "FILLED"),
            };

            // Dynamic filter: NIFTY50 orders, price >= 24400, status = FILLED
            var req = new OrderSearchRequest
            {
                Symbol   = "NIFTY50",
                MinPrice = 24_400m,
                Status   = "FILLED"
            };

            var results = OrderQueryBuilder.Search(orders, req);
            Console.WriteLine($"\nFound {results.Count} matching orders:");
            foreach (var o in results)
                Console.WriteLine($"  {o.OrderId}: {o.Symbol} @ {o.Price:C} [{o.Status}]");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Use Case | How Expression Trees Help |
|---|---|
| **EF Core / LINQ-to-SQL** | Lambda → SQL WHERE clause translation |
| **AutoMapper** | Property mapping compiled to fast delegates |
| **Specification pattern** | Composable filter predicates, deferred to DB |
| **Validation frameworks** | Inspect property access to generate error messages |
| **Dynamic serializers** | Walk object graph without reflection overhead |

**Performance:** Compile expression trees to delegates **once** (at startup or on first use) and cache the compiled `Func<T>`. Recompiling is expensive; caching in a `ConcurrentDictionary<CacheKey, Delegate>` makes repeated executions as fast as direct method calls.

---

---

## Q19. LINQ – deferred vs immediate execution.

### 📌 Definition
**LINQ (Language-Integrated Query)** queries operate in two execution modes. **Deferred (lazy) execution** means the query definition is stored as an `IEnumerable<T>` or `IQueryable<T>` expression that is **not evaluated until you iterate over it** (e.g., `foreach`, `.ToList()`, `.Count()`). Every time you enumerate the result, the query re-executes against the current state of the data source. **Immediate execution** means the query result is materialized into memory right now — operators like `.ToList()`, `.ToArray()`, `.ToDictionary()`, `.Count()`, `.First()`, `.Sum()` force immediate evaluation. Understanding this distinction is critical for performance: unintended multiple enumeration causes repeated DB hits or CPU work, while holding onto deferred queries too long can read stale data.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Hidden multiple enumeration — query runs TWICE
IEnumerable<Order> expensiveOrders = orders.Where(o => o.Price > 24000m);
Console.WriteLine(expensiveOrders.Count()); // Execution 1: iterates all orders
foreach (var o in expensiveOrders) { /* ... */ } // Execution 2: iterates all orders again
// For in-memory: wasted CPU. For IQueryable (EF): 2 database round trips!

// ✅ Materialize once, use many times
List<Order> expensiveList = orders.Where(o => o.Price > 24000m).ToList(); // ONE execution
Console.WriteLine(expensiveList.Count);      // O(1) — already in memory
foreach (var o in expensiveList) { /* ... */ } // Still in memory
```

**Deferred operators (build the query, don't run yet):**
```csharp
// These return IEnumerable<T> or IQueryable<T> — deferred:
.Where(), .Select(), .SelectMany(), .OrderBy(), .GroupBy(),
.Skip(), .Take(), .Distinct(), .Zip(), .Join(), .Union()
```

**Immediate operators (trigger execution):**
```csharp
// These force evaluation and return a concrete result:
.ToList(), .ToArray(), .ToDictionary(), .ToHashSet()   // materialize collections
.Count(), .LongCount()                                  // aggregate
.First(), .FirstOrDefault(), .Single(), .Last()         // element fetchers
.Sum(), .Average(), .Min(), .Max()                      // aggregates
.Any(), .All(), .Contains()                             // short-circuit booleans
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Trade reporting service — deferred pipeline vs eager materialization**

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Diagnostics;

namespace TradingPlatform.Linq
{
    public record Trade(string Id, string Symbol, decimal Price, int Quantity, DateTime ExecutedAt, string Desk);

    public static class TradeReportingService
    {
        public static void Main()
        {
            var trades = GenerateTrades(100_000);

            // ── Deferred pipeline: chained lazy operators ──────────────────
            // None of these execute yet — just builds an expression chain
            var pipeline = trades
                .Where(t  => t.Symbol == "NIFTY50")         // deferred filter
                .Where(t  => t.Price  > 24_000m)             // deferred filter
                .Select(t => new { t.Symbol, t.Price, t.Quantity, Value = t.Price * t.Quantity }); // project

            // Pipeline executes ONCE here when we call ToList()
            var sw = Stopwatch.StartNew();
            var results = pipeline.ToList(); // Materializes — single pass through data
            sw.Stop();
            Console.WriteLine($"Deferred->ToList: {results.Count} records in {sw.ElapsedMilliseconds}ms");

            // ── Multiple deferred enumeration problem ──────────────────────
            IEnumerable<Trade> deferredQuery = trades.Where(t => t.Symbol == "BANKNIFTY");

            sw.Restart();
            int count  = deferredQuery.Count();             // Execution 1 — full scan
            sw.Stop(); Console.WriteLine($"Count     (1st enum): {count} — {sw.ElapsedMilliseconds}ms");

            sw.Restart();
            decimal total = deferredQuery.Sum(t => t.Price); // Execution 2 — full scan again!
            sw.Stop(); Console.WriteLine($"Sum (2nd enum, BAD): {total:N0} — {sw.ElapsedMilliseconds}ms");

            // ✅ Fix: materialize once
            sw.Restart();
            var bankNifty = trades.Where(t => t.Symbol == "BANKNIFTY").ToList();
            int cnt2  = bankNifty.Count;
            decimal ttl2 = bankNifty.Sum(t => t.Price);
            sw.Stop(); Console.WriteLine($"Materialized (GOOD): count={cnt2} sum={ttl2:N0} — {sw.ElapsedMilliseconds}ms");

            // ── Grouping and aggregation (immediate) ───────────────────────
            var summary = trades
                .GroupBy(t => t.Symbol)
                .Select(g => new
                {
                    Symbol     = g.Key,
                    TotalValue = g.Sum(t => t.Price * t.Quantity),
                    AvgPrice   = g.Average(t => (double)t.Price),
                    Count      = g.Count()
                })
                .OrderByDescending(s => s.TotalValue)
                .ToList(); // Immediate — grouped and sorted

            Console.WriteLine("\nTrade Summary by Symbol:");
            foreach (var s in summary.Take(3))
                Console.WriteLine($"  {s.Symbol,-12} | Count={s.Count,5} | Avg={s.AvgPrice:N0} | Total={s.TotalValue:N0}");
        }

        private static List<Trade> GenerateTrades(int n)
        {
            var rng     = new Random(42);
            var symbols = new[] { "NIFTY50", "BANKNIFTY", "SENSEX", "RELIANCE", "TCS" };
            var desks   = new[] { "EQUITY", "DERIVATIVES", "ALGO" };
            return Enumerable.Range(1, n).Select(i => new Trade(
                $"TRD-{i:D6}",
                symbols[rng.Next(symbols.Length)],
                24_000m + rng.Next(2000),
                rng.Next(1, 50),
                DateTime.UtcNow.AddMinutes(-rng.Next(480)),
                desks[rng.Next(desks.Length)]
            )).ToList();
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Rule | Reason |
|---|---|
| Call `.ToList()` / `.ToArray()` before multiple iterations | Prevents repeated DB round trips or CPU work |
| Never expose `IQueryable<T>` across service boundaries | Allows callers to add unintended DB filters |
| Use `IEnumerable<T>` for deferred in-memory streaming | Avoids materializing entire dataset into RAM |
| Avoid `.Count()` on `IQueryable` if data already materialized | Use `.Count` on `List<T>` — O(1) |

---

## Q20. IEnumerable vs IQueryable.

### 📌 Definition
- **`IEnumerable<T>`** — defined in `System.Collections.Generic`. Represents a **pull-based in-memory sequence**. LINQ extension methods on `IEnumerable<T>` (from `System.Linq.Enumerable`) operate on the data **in the .NET process** using compiled delegates. Filtering, ordering, grouping — all executed by the CLR against data already pulled into memory.
- **`IQueryable<T>`** — defined in `System.Linq`. Extends `IEnumerable<T>` but adds an `Expression` tree and a `Provider` (`IQueryProvider`). LINQ extension methods on `IQueryable<T>` (from `System.Linq.Queryable`) capture the lambda as an expression tree **instead of compiling it**, and the provider (e.g., EF Core's LINQ-to-SQL engine) translates the expression tree into a target query language (SQL, OData, Cosmos query) that executes **remotely on the data source**. Only the matching records are transferred over the network.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ IEnumerable filter runs IN .NET PROCESS — fetches ALL records first
IEnumerable<Order> allOrders = _dbContext.Orders.AsEnumerable(); // pulls ALL rows into RAM
var expensive = allOrders.Where(o => o.Price > 24000m); // filters in C# against all rows
// Problem: if there are 10 million orders → 10M rows transferred, massive memory usage

// ✅ IQueryable filter runs ON THE DATABASE — transfers only matching records
IQueryable<Order> query = _dbContext.Orders;              // deferred query
var expensive2 = query.Where(o => o.Price > 24000m);      // adds to expression tree
// EF Core generates: SELECT * FROM Orders WHERE Price > 24000
// Only matching rows transferred over the wire
var results = expensive2.ToList(); // Executes SQL — minimal network transfer
```

**Mixing accidentally — common bug:**
```csharp
IQueryable<Order> query = _dbContext.Orders.Where(o => o.Symbol == "NIFTY50");

// ❌ Calling AsEnumerable() too early — pulls NIFTY50 rows into memory, THEN filters in .NET
var wrong = query.AsEnumerable()
                 .Where(o => o.Price > 24000m)  // This runs in C#, not SQL
                 .OrderBy(o => o.Price);

// ✅ Keep IQueryable through all filters — single SQL statement
var correct = query
    .Where(o => o.Price > 24000m)  // Added to SQL WHERE
    .OrderBy(o => o.Price)          // Added to SQL ORDER BY
    .ToList();                       // Single DB round trip
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Order search API with server-side filtering and paging**

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;

namespace TradingPlatform.DataLayer
{
    public record Order(int Id, string Symbol, decimal Price, int Quantity, string Status, DateTime CreatedAt);

    public class OrderSearchParams
    {
        public string? Symbol   { get; init; }
        public decimal? MinPrice { get; init; }
        public string?  Status  { get; init; }
        public string   SortBy  { get; init; } = "Price";
        public int      Page    { get; init; } = 1;
        public int      PageSize { get; init; } = 20;
    }

    // Simulates EF DbSet<Order> (IQueryable)
    public static class OrderRepository
    {
        private static readonly IQueryable<Order> _fakeDb =
            Enumerable.Range(1, 10_000).Select(i => new Order(
                i, i % 3 == 0 ? "NIFTY50" : i % 3 == 1 ? "BANKNIFTY" : "SENSEX",
                24_000m + (i % 1000), i % 50 + 1,
                i % 4 == 0 ? "CANCELLED" : i % 2 == 0 ? "FILLED" : "OPEN",
                DateTime.UtcNow.AddMinutes(-i)
            )).AsQueryable(); // Important: AsQueryable() enables IQueryable semantics

        // All filters applied in the QUERY (translated to SQL in real EF scenario)
        public static (List<Order> Items, int TotalCount) Search(OrderSearchParams p)
        {
            IQueryable<Order> query = _fakeDb; // Start with full IQueryable

            // Conditionally add WHERE clauses — each appended to the expression tree
            if (!string.IsNullOrEmpty(p.Symbol))
                query = query.Where(o => o.Symbol == p.Symbol);

            if (p.MinPrice.HasValue)
                query = query.Where(o => o.Price >= p.MinPrice.Value);

            if (!string.IsNullOrEmpty(p.Status))
                query = query.Where(o => o.Status == p.Status);

            // Count BEFORE paging — single SQL COUNT(*) query
            int totalCount = query.Count(); // IQueryable executes COUNT SQL

            // Apply sort
            query = p.SortBy switch
            {
                "Price"  => query.OrderBy(o => o.Price),
                "Symbol" => query.OrderBy(o => o.Symbol),
                _        => query.OrderByDescending(o => o.CreatedAt)
            };

            // Apply paging — SQL OFFSET / FETCH NEXT
            var items = query
                .Skip((p.Page - 1) * p.PageSize)
                .Take(p.PageSize)
                .ToList(); // ← Single SQL execution with all clauses combined

            return (items, totalCount);
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var (items, total) = OrderRepository.Search(new OrderSearchParams
            {
                Symbol   = "NIFTY50",
                MinPrice = 24_500m,
                Status   = "FILLED",
                SortBy   = "Price",
                Page     = 1,
                PageSize = 5
            });

            Console.WriteLine($"Total matching: {total} | Showing {items.Count}");
            foreach (var o in items)
                Console.WriteLine($"  [{o.Id}] {o.Symbol} x{o.Quantity} @ {o.Price:C} [{o.Status}]");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Dimension | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| **Execution location** | .NET process (in memory) | Remote data source (DB, OData) |
| **Lambda handling** | Compiled to IL delegate | Captured as expression tree, translated |
| **Best for** | In-memory filtering, post-load transforms | DB queries, minimize network transfer |
| **`AsEnumerable()`** | Forces in-memory execution from this point | Crosses boundary — rest runs in .NET |
| **`AsQueryable()`** | Wraps `IEnumerable<T>` as queryable (limited) | Native for EF, LINQ providers |

**Production rule:** Keep `IQueryable<T>` composition as long as possible. Cross to `IEnumerable<T>` (via `.ToList()` or `.AsEnumerable()`) only when you need CLR-specific operations that cannot be translated to SQL (custom C# methods, complex CLR logic).

---

## Q21. Extension methods – how compiler treats them.

### 📌 Definition
An **extension method** is a static method defined in a static class with `this` as the first parameter type, which the compiler allows to be invoked **as if it were an instance method of that type** — without modifying the original class. The compiler performs a straightforward syntactic transformation at compile time: `myObject.ExtensionMethod(arg)` is rewritten to `ExtensionClass.ExtensionMethod(myObject, arg)`. No IL magic — it is purely a compile-time syntax convenience. The method has no access to private members of the extended type (it can only use the type's public API). Extension methods enable adding behavior to types you cannot modify (sealed classes, third-party types, interfaces).

### 🔍 Why It Exists / Problem It Solves

```csharp
// Without extension methods: verbose static call syntax
string price = OrderFormatter.FormatPrice(order.Price, "₹");
IEnumerable<Order> filtered = OrderExtensions.FilterBySymbol(orders, "NIFTY50");

// ✅ With extension methods: fluent, readable, discoverable via IntelliSense
string price    = order.Price.FormatPrice("₹");
var filtered    = orders.FilterBySymbol("NIFTY50");
// output.ToList().Count() — the entire LINQ API is extension methods on IEnumerable<T>!

// Definition — how to write one:
public static class OrderExtensions
{
    // 'this Order order' — 'order' is the object the method is called on
    public static string ToDisplay(this Order order)
        => $"{order.Symbol} x{order.Quantity} @ {order.Price:C} [{order.Status}]";

    // Extension on interface — applies to ALL implementations
    public static bool IsHighValue(this IEnumerable<Order> orders, decimal threshold)
        => orders.Any(o => o.Price * o.Quantity > threshold);

    // Extension on generic type with constraint
    public static T Clamp<T>(this T value, T min, T max) where T : IComparable<T>
        => value.CompareTo(min) < 0 ? min : value.CompareTo(max) > 0 ? max : value;
}

// Compiler transformation (IL level):
// orders.FilterBySymbol("NIFTY50")
// ↓ compiles to ↓
// OrderExtensions.FilterBySymbol(orders, "NIFTY50")
// Completely equivalent — extension methods are syntactic sugar, zero runtime overhead
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Fluent builder API for order queries without modifying the Order class**

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace TradingPlatform.Extensions
{
    public record Order(string Id, string Symbol, decimal Price, int Quantity, string Status, string Desk);

    // ── Extension methods on IQueryable<Order> — fluent, composable ───────
    public static class OrderQueryExtensions
    {
        public static IQueryable<Order> ForSymbol(this IQueryable<Order> query, string symbol)
            => string.IsNullOrEmpty(symbol) ? query : query.Where(o => o.Symbol == symbol);

        public static IQueryable<Order> WithMinValue(this IQueryable<Order> query, decimal minValue)
            => query.Where(o => o.Price * o.Quantity >= minValue);

        public static IQueryable<Order> InStatus(this IQueryable<Order> query, params string[] statuses)
            => statuses.Length == 0 ? query : query.Where(o => statuses.Contains(o.Status));

        public static IQueryable<Order> FromDesk(this IQueryable<Order> query, string desk)
            => string.IsNullOrEmpty(desk) ? query : query.Where(o => o.Desk == desk);

        public static IQueryable<Order> PagedBy(this IQueryable<Order> query, int page, int size)
            => query.Skip((page - 1) * size).Take(size);
    }

    // ── Extension methods on Order — enrich the type without modifying it ─
    public static class OrderExtensions
    {
        public static decimal TradeValue(this Order o)     => o.Price * o.Quantity;
        public static bool    IsHighValue(this Order o)    => o.TradeValue() > 500_000m;
        public static string  ToSummary(this Order o)      =>
            $"[{o.Id}] {o.Symbol,-12} x{o.Quantity,4} @ {o.Price,10:C} | {o.Status,-10} | Value={o.TradeValue():N0}";

        // Extension on IEnumerable<Order> for batch metrics
        public static decimal TotalValue(this IEnumerable<Order> orders)
            => orders.Sum(o => o.TradeValue());

        public static Dictionary<string, int> CountBySymbol(this IEnumerable<Order> orders)
            => orders.GroupBy(o => o.Symbol).ToDictionary(g => g.Key, g => g.Count());
    }

    public static class Program
    {
        public static void Main()
        {
            var orders = new List<Order>
            {
                new("O1", "NIFTY50",   24500m, 20, "FILLED",    "EQUITY"),
                new("O2", "BANKNIFTY", 52100m, 5,  "OPEN",      "DERIVATIVES"),
                new("O3", "NIFTY50",   24300m, 50, "FILLED",    "ALGO"),
                new("O4", "SENSEX",    72000m, 1,  "CANCELLED", "EQUITY"),
                new("O5", "NIFTY50",   24800m, 10, "FILLED",    "EQUITY"),
            }.AsQueryable();

            // Fluent query using extension methods — reads like domain language
            var results = orders
                .ForSymbol("NIFTY50")
                .InStatus("FILLED")
                .WithMinValue(200_000m)
                .PagedBy(page: 1, size: 10)
                .ToList();

            Console.WriteLine("Matching orders:");
            foreach (var o in results)
                Console.WriteLine($"  {o.ToSummary()}");  // Instance extension

            Console.WriteLine($"\nTotal value: {results.TotalValue():N0}");     // Batch extension
            Console.WriteLine($"By symbol:   {string.Join(", ", results.CountBySymbol().Select(kv => $"{kv.Key}={kv.Value}"))}");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Rule | Reason |
|---|---|
| **Put in same namespace as extended type** (or a sub-namespace) | Discovered by IntelliSense without extra `using` |
| **Don't overuse** | Misuse leads to extension method "pollution" — IntelliSense cluttered |
| **Can't access private members** | Extension methods are static helpers, not true instance methods |
| **Null checking** | Extension methods ARE called on null instances — add null guard if needed: `if (this Order o == null) throw new ArgumentNullException(nameof(o))` |
| **LINQ is entirely extension methods** | `Where`, `Select`, `GroupBy` etc. are all extensions on `IEnumerable<T>` / `IQueryable<T>` |

---

## Q22. Dependency Injection in .NET – how it works internally.

### 📌 Definition
**Dependency Injection (DI)** is a design pattern where an object's dependencies (services it needs) are provided to it by an external container rather than created by the object itself. In .NET, the built-in DI container (`Microsoft.Extensions.DependencyInjection`) manages a **service registry** (`IServiceCollection`) mapping service types to implementation types, and a **service provider** (`IServiceProvider`) that resolves instances on demand. Internally, the container builds a compiled expression tree (or factory delegate) for each registered service at startup during a container "build" step. At resolution time, the compiled factory is invoked — creating, wiring, and lifetime-managing objects. This inverts control: classes declare what they need via constructor parameters, and the container provides the correct implementations.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ WITHOUT DI — tight coupling, untestable, hard to swap
public class OrderService
{
    private readonly SqlOrderRepository _repo;    // hard-coded concrete type
    private readonly EmailNotifier      _notifier; // hard-coded concrete type

    public OrderService()
    {
        _repo     = new SqlOrderRepository("Server=prod-db;..."); // coupled to prod DB
        _notifier = new EmailNotifier("smtp.prod.com");            // coupled to prod SMTP
    }
    // Can't unit test without hitting prod DB and SMTP!
    // Swapping SqlOrderRepository for a Redis-backed one requires changing this class.
}

// ✅ WITH DI — loosely coupled, testable, swappable
public class OrderService
{
    private readonly IOrderRepository _repo;    // depends on abstraction (interface)
    private readonly INotifier        _notifier;

    // Dependencies INJECTED by the container — OrderService doesn't create them
    public OrderService(IOrderRepository repo, INotifier notifier)
        => (_repo, _notifier) = (repo, notifier);
}

// Unit test: inject mock implementations
var mockRepo     = new Mock<IOrderRepository>();
var mockNotifier = new Mock<INotifier>();
var svc = new OrderService(mockRepo.Object, mockNotifier.Object); // ✅ testable
```

**How .NET DI container works internally:**
```csharp
// 1. Registration (startup)
var services = new ServiceCollection();
services.AddSingleton<IMarketDataService, RedisMarketDataService>(); // one instance forever
services.AddScoped<IOrderRepository, SqlOrderRepository>();           // one per request
services.AddTransient<IRiskEngine, DefaultRiskEngine>();              // new instance each time

// 2. Build — container compiles resolution factories (done once)
IServiceProvider provider = services.BuildServiceProvider();
// Internally: for each registered service, build a compiled Func<IServiceProvider, object>
// that constructs the type and injects its own dependencies recursively

// 3. Resolution — factory invoked, lifetime managed
// For Scoped: IServiceScope created per HTTP request in ASP.NET Core
using var scope = provider.CreateScope();
IOrderRepository repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
// Internally: calls the compiled factory, checks lifetime cache (scoped dict), returns instance
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Wiring an order processing service using .NET DI with all three lifetimes**

```csharp
using System;
using Microsoft.Extensions.DependencyInjection;

namespace TradingPlatform.DI
{
    // ── Abstractions (interfaces) ──────────────────────────────────────────
    public interface IMarketDataService { decimal GetPrice(string symbol); }
    public interface IOrderRepository   { void Save(string orderId); }
    public interface INotificationService { void Notify(string msg); }

    // ── Implementations ────────────────────────────────────────────────────
    public class RedisMarketDataService : IMarketDataService
    {
        private readonly string _connectionId = Guid.NewGuid().ToString("N")[..6];
        public decimal GetPrice(string symbol)
        {
            Console.WriteLine($"  [Redis:{_connectionId}] Fetching {symbol}");
            return symbol == "NIFTY50" ? 24_500m : 52_100m;
        }
    }

    public class SqlOrderRepository : IOrderRepository
    {
        private readonly string _id = Guid.NewGuid().ToString("N")[..6];
        public void Save(string orderId)
            => Console.WriteLine($"  [SQL:{_id}] Saving {orderId}");
    }

    public class EmailNotificationService : INotificationService
    {
        public void Notify(string msg)
            => Console.WriteLine($"  [EMAIL] {msg}");
    }

    // ── Consumer service — receives all deps via constructor ───────────────
    public class OrderProcessingService
    {
        private readonly IMarketDataService  _market;
        private readonly IOrderRepository    _repo;
        private readonly INotificationService _notify;

        public OrderProcessingService(
            IMarketDataService market,
            IOrderRepository repo,
            INotificationService notify)
        {
            _market = market;
            _repo   = repo;
            _notify = notify;
        }

        public void Process(string orderId, string symbol)
        {
            decimal price = _market.GetPrice(symbol);
            _repo.Save(orderId);
            _notify.Notify($"Order {orderId} executed @ {price:C}");
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var services = new ServiceCollection();

            // Singleton: one shared instance for the entire app lifetime
            services.AddSingleton<IMarketDataService, RedisMarketDataService>();

            // Scoped: one instance per scope (one per HTTP request in web apps)
            services.AddScoped<IOrderRepository, SqlOrderRepository>();

            // Transient: new instance every time it is requested
            services.AddTransient<INotificationService, EmailNotificationService>();

            // OrderProcessingService itself — transient, gets injected deps automatically
            services.AddTransient<OrderProcessingService>();

            IServiceProvider provider = services.BuildServiceProvider();

            Console.WriteLine("=== Scope 1 (Request 1) ===");
            using (var scope1 = provider.CreateScope())
            {
                var svc1 = scope1.ServiceProvider.GetRequiredService<OrderProcessingService>();
                var svc2 = scope1.ServiceProvider.GetRequiredService<OrderProcessingService>();
                svc1.Process("ORD-001", "NIFTY50");
                svc2.Process("ORD-002", "NIFTY50"); // Same SqlOrderRepository instance (scoped)
            }

            Console.WriteLine("\n=== Scope 2 (Request 2) ===");
            using (var scope2 = provider.CreateScope())
            {
                var svc3 = scope2.ServiceProvider.GetRequiredService<OrderProcessingService>();
                svc3.Process("ORD-003", "BANKNIFTY"); // NEW SqlOrderRepository for this scope
                // Same RedisMarketDataService (singleton)
            }
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Lifetime | Instances per | Use for |
|---|---|---|
| `Singleton` | Application | Shared caches, config, connection factories |
| `Scoped` | HTTP request / unit of work | DbContext, per-request state |
| `Transient` | Each resolution | Stateless services, lightweight helpers |

**Captive dependency trap — common bug:**
> A `Singleton` injecting a `Scoped` dependency captures the first-created scoped instance and reuses it forever — effectively making it a singleton. DI container validation (`ValidateScopes = true`) catches this at startup.

```csharp
// Enable validation in production builds:
services.BuildServiceProvider(new ServiceProviderOptions
{
    ValidateScopes   = true, // Catch captive dependency bugs
    ValidateOnBuild  = true  // Validate all registrations at build time
});
```

---

## Q23. Singleton vs Scoped vs Transient lifetimes.

### 📌 Definition
These are the three **service lifetime options** in .NET DI that control how long an instance lives and how many instances of a service exist:
- **Singleton** — One instance created the first time it is requested, then that same instance is reused for every subsequent request for the **entire application lifetime**. Allocated once, lives until the process exits.
- **Scoped** — One instance per **DI scope**. In ASP.NET Core, a new scope is created automatically per HTTP request — so one instance per request, shared within that request, discarded at request end. Scopes can be created manually for background workers.
- **Transient** — A **new instance** is created every time the service is requested from the container. No sharing, no reuse. Cheapest to reason about but most allocations.

### 🔍 Why It Exists / Problem It Solves

```csharp
// SINGLETON use cases:
// ✅ Expensive-to-create shared state that is THREAD SAFE
// ✅ Connection factories (HttpClientFactory, Kafka producer)
// ✅ In-memory caches (IMemoryCache)
// ✅ Configuration objects (IOptions<T>)
services.AddSingleton<IMemoryCache, MemoryCache>();
services.AddSingleton<IConfiguration>(configuration);

// ❌ WRONG: Singleton holding a non-thread-safe mutable dictionary
services.AddSingleton<OrderTracker>(); // If OrderTracker has Dictionary<string,Order> ← race condition

// SCOPED use cases:
// ✅ DbContext (EF Core) — one context per request, change tracking per request
// ✅ Unit-of-work pattern
// ✅ Per-request caching (avoid repeated DB calls in same request)
services.AddDbContext<TradingDbContext>(opts => opts.UseSqlServer(connStr));
// One EF DbContext per HTTP request — tracks changes for that request, disposed at end

// TRANSIENT use cases:
// ✅ Lightweight stateless services (validators, calculators)
// ✅ Services that are NOT thread-safe and MUST NOT be shared
// ✅ Services that need a fresh state on every operation
services.AddTransient<IOrderValidator, OrderValidator>();
services.AddTransient<IRiskCalculator, VaRCalculator>();
```

**Captive dependency detection:**
```csharp
// ❌ CAPTIVE DEPENDENCY: Singleton captures a Scoped → bug
public class OrderCache // Singleton
{
    private readonly TradingDbContext _db; // Scoped!
    // _db is the FIRST-REQUEST DbContext, reused forever → stale data, multi-request bugs
    public OrderCache(TradingDbContext db) => _db = db;
}
// Fix: inject IServiceScopeFactory and create scopes on demand
public class OrderCache // Singleton
{
    private readonly IServiceScopeFactory _scopeFactory;
    public OrderCache(IServiceScopeFactory factory) => _scopeFactory = factory;

    public async Task<Order> GetOrder(string id)
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<TradingDbContext>();
        return await db.Orders.FindAsync(id); // Fresh DbContext per operation
    }
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Request ID tracking per HTTP request using scoped service**

```csharp
using System;
using Microsoft.Extensions.DependencyInjection;

namespace TradingPlatform.Lifetimes
{
    // ── Singleton: shared connection factory ───────────────────────────────
    public class KafkaProducerFactory
    {
        private readonly string _id = Guid.NewGuid().ToString("N")[..6];
        public string GetProducerId() => _id;
        public void Publish(string topic, string msg)
            => Console.WriteLine($"  [Kafka:{_id}] → {topic}: {msg}");
    }

    // ── Scoped: per-request context with unique ID ─────────────────────────
    public class RequestContext
    {
        public string RequestId { get; } = Guid.NewGuid().ToString("N")[..8];
        public DateTime StartedAt { get; } = DateTime.UtcNow;
    }

    // ── Transient: fresh stateless risk calculator each time ───────────────
    public class RiskCalculator
    {
        private readonly string _instanceId = Guid.NewGuid().ToString("N")[..4];
        public decimal ComputeVaR(decimal position) {
            Console.WriteLine($"  [Risk:{_instanceId}] Computing VaR for {position:C}");
            return position * 0.02m;
        }
    }

    // ── Consumer: receives all three lifetime types ────────────────────────
    public class OrderHandler
    {
        private readonly KafkaProducerFactory _kafka;    // singleton
        private readonly RequestContext       _ctx;       // scoped
        private readonly RiskCalculator       _risk;      // transient

        public OrderHandler(KafkaProducerFactory kafka, RequestContext ctx, RiskCalculator risk)
            => (_kafka, _ctx, _risk) = (kafka, ctx, risk);

        public void Handle(string orderId, decimal price)
        {
            decimal vaR = _risk.ComputeVaR(price);
            _kafka.Publish("orders", $"[{_ctx.RequestId}] {orderId} VaR={vaR:C}");
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var services = new ServiceCollection();
            services.AddSingleton<KafkaProducerFactory>();
            services.AddScoped<RequestContext>();
            services.AddTransient<RiskCalculator>();
            services.AddTransient<OrderHandler>();
            var provider = services.BuildServiceProvider(validateScopes: true);

            // Simulate 2 HTTP requests (2 scopes)
            for (int req = 1; req <= 2; req++)
            {
                Console.WriteLine($"\n=== Request {req} ===");
                using var scope = provider.CreateScope();
                var h1 = scope.ServiceProvider.GetRequiredService<OrderHandler>();
                var h2 = scope.ServiceProvider.GetRequiredService<OrderHandler>();
                h1.Handle($"ORD-{req:D3}-A", 24_500m);
                h2.Handle($"ORD-{req:D3}-B", 52_100m);
                // KafkaProducerFactory: same ID both requests (singleton)
                // RequestContext: same ID within request, different between requests (scoped)
                // RiskCalculator: different instance IDs every call (transient)
            }
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Lifetime | Shared by | Thread Safety Required | GC Generation |
|---|---|---|---|
| Singleton | All requests, all threads | ✅ Must be thread-safe | Gen2 (app lifetime) |
| Scoped | One request / scope | ❌ Single thread per request | Gen1 → collected at scope end |
| Transient | Nobody — always new | ❌ Fresh instance = no sharing | Gen0 → fast collection |

---

---

## Q24. What is middleware pipeline in .NET?

### 📌 Definition
The **middleware pipeline** in .NET is a chain of **request/response processing components** that are executed in sequence for every HTTP request. Each middleware component in the chain receives the `HttpContext`, can execute logic **before** passing the request to the next component (pre-processing), can call the next middleware, and can execute logic **after** it returns (post-processing). The pipeline is built in `Program.cs` using `app.Use()`, `app.Run()`, and `app.UseXxx()` extension methods. Middleware is arranged like nested function calls — the first registered middleware is the outermost function; the last (terminal) middleware is the innermost. This creates a powerful, composable pipeline for cross-cutting concerns: authentication, logging, exception handling, CORS, routing, response caching.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Without middleware: each controller handles all cross-cutting concerns manually
public class OrderController : ControllerBase
{
    public IActionResult PlaceOrder()
    {
        if (!Request.Headers.ContainsKey("Authorization")) return Unauthorized(); // auth
        _logger.LogInformation("Request received");             // logging
        try { /* business logic */ }
        catch (Exception ex) { return StatusCode(500); }        // error handling
        // Every controller repeats this — massive duplication
    }
}

// ✅ With middleware: cross-cutting concerns handled once, applied to all endpoints
app.UseExceptionHandling(); // wraps everything — catches all unhandled exceptions
app.UseAuthentication();    // validates JWT token before reaching any controller
app.UseAuthorization();     // checks permissions
app.UseRequestLogging();    // logs every request/response
app.MapControllers();       // routing to controllers — terminal middleware
```

**Pipeline execution flow (Russian dolls / onion model):**
```
Request →  [ExceptionMiddleware] →  [AuthMiddleware] →  [LoggingMiddleware] →  [Controller]
         ↓                       ↓                    ↓                     ↓
Response ←  [ExceptionMiddleware] ←  [AuthMiddleware] ←  [LoggingMiddleware] ←  [Controller]
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Production-grade middleware chain for a trading API**

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

namespace TradingAPI.Middleware
{
    // ── Custom middleware 1: Request timing and correlation ID ────────────
    public class RequestTimingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<RequestTimingMiddleware> _logger;

        public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
            => (_next, _logger) = (next, logger);

        public async Task InvokeAsync(HttpContext context)
        {
            // Assign correlation ID for distributed tracing
            string correlationId = context.Request.Headers["X-Correlation-Id"]
                                           .ToString()
                                           .DefaultIfEmpty(Guid.NewGuid().ToString("N")[..8]);
            context.Items["CorrelationId"] = correlationId;
            context.Response.Headers["X-Correlation-Id"] = correlationId;

            var sw = Stopwatch.StartNew();

            _logger.LogInformation("[{CorrelationId}] {Method} {Path} started",
                correlationId, context.Request.Method, context.Request.Path);

            await _next(context); // ← Call next middleware in pipeline

            sw.Stop();
            _logger.LogInformation("[{CorrelationId}] {Method} {Path} completed {StatusCode} in {ElapsedMs}ms",
                correlationId, context.Request.Method, context.Request.Path,
                context.Response.StatusCode, sw.ElapsedMilliseconds);
        }
    }

    // ── Custom middleware 2: Rate limiting per client IP ───────────────────
    public class RateLimitingMiddleware
    {
        private readonly RequestDelegate _next;
        private static readonly System.Collections.Concurrent.ConcurrentDictionary<string, (int count, DateTime window)>
            _counters = new();
        private const int RequestsPerMinute = 60;

        public RateLimitingMiddleware(RequestDelegate next) => _next = next;

        public async Task InvokeAsync(HttpContext context)
        {
            string ip = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
            var now   = DateTime.UtcNow;

            var entry = _counters.AddOrUpdate(ip,
                _ => (1, now),
                (_, old) => old.window.AddMinutes(1) < now
                    ? (1, now)                         // Window expired — reset
                    : (old.count + 1, old.window));    // Within window — increment

            if (entry.count > RequestsPerMinute)
            {
                context.Response.StatusCode = 429; // Too Many Requests
                context.Response.Headers["Retry-After"] = "60";
                await context.Response.WriteAsync("Rate limit exceeded. Try again in 60 seconds.");
                return; // Short-circuit — don't call next
            }

            await _next(context);
        }
    }

    // ── Register middleware in Program.cs ──────────────────────────────────
    // Order matters: exception handler outermost, routing innermost
    public static class MiddlewareExtensions
    {
        public static WebApplication UseApiMiddleware(this WebApplication app)
        {
            app.UseMiddleware<RequestTimingMiddleware>(); // Outermost — times everything
            app.UseMiddleware<RateLimitingMiddleware>(); // Second — blocks before auth work
            app.UseAuthentication();                     // Third — validate JWT
            app.UseAuthorization();                      // Fourth — check permissions
            // app.MapControllers();                     // Terminal — reached only if all pass
            return app;
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Ordering rule | Why |
|---|---|
| `UseExceptionHandler` first | Catches exceptions from ALL subsequent middleware |
| `UseHsts`/`UseHttpsRedirection` early | HTTPS enforcement before any processing |
| `UseAuthentication` before `UseAuthorization` | Must know WHO before checking WHAT they can do |
| `UseRouting` before `UseAuthorization` | Authorization needs route data to apply policies |
| Terminal middleware (`app.Run`) last | No `_next` — swallows the request |

---

## Q25. What is Kestrel server?

### 📌 Definition
**Kestrel** is the default, cross-platform, asynchronous HTTP server built into ASP.NET Core — written entirely in C# using `System.IO.Pipelines` for zero-copy I/O. Unlike legacy IIS-hosted ASP.NET which required Windows and the IIS process model, Kestrel runs in-process with the .NET application on Linux, macOS, and Windows. Kestrel handles raw TCP connections, HTTP/1.1, HTTP/2, HTTP/3 (QUIC), WebSockets, and gRPC. It uses the CLR's asynchronous I/O completion ports (IOCP on Windows, epoll on Linux) under the hood — meaning one thread can serve thousands of concurrent connections without blocking. In production, Kestrel typically sits behind a reverse proxy (NGINX, IIS, Azure Application Gateway) that handles SSL termination, load balancing, and static file serving.

### 🔍 Why It Exists / Problem It Solves

```
Legacy ASP.NET:        ASP.NET Core with Kestrel:
  IIS (Windows-only)    Any OS (Linux containers, macOS dev, Windows prod)
  Process model:         In-process: faster, lower overhead
  Synchronous I/O        async I/O via System.IO.Pipelines (zero-copy)
  ~1K req/s typical      ~1M req/s benchmark (TechEmpower plaintext)
```

**System.IO.Pipelines — the engine under Kestrel:**
```csharp
// Traditional Read path (old):
byte[] buffer = new byte[4096];                    // Heap allocation
int read = await stream.ReadAsync(buffer, 0, 4096); // Copy bytes into buffer
ProcessBytes(buffer, 0, read);                     // Process from buffer copy

// Kestrel's Pipe path (zero-copy):
var pipe = new Pipe();
PipeReader reader = pipe.Reader;
// Kestrel fills pipe directly from OS I/O buffer — no intermediate copy
ReadResult result = await reader.ReadAsync();
ReadOnlySequence<byte> data = result.Buffer; // Slice of OS-provided memory
ProcessRequestHeaders(data);                // Parse directly from OS memory
reader.AdvanceTo(data.Start, data.End);    // Tell pipe how much we consumed
```

**Kestrel configuration in production:**
```csharp
// Program.cs — production Kestrel tuning
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxConcurrentConnections         = 100;
    options.Limits.MaxConcurrentUpgradedConnections = 100;
    options.Limits.MaxRequestBodySize               = 10 * 1024 * 1024; // 10MB
    options.Limits.KeepAliveTimeout                 = TimeSpan.FromMinutes(2);
    options.Limits.RequestHeadersTimeout            = TimeSpan.FromSeconds(30);

    // HTTP/2 for gRPC
    options.ListenLocalhost(5001, o => o.Protocols = HttpProtocols.Http2);

    // HTTP/1.1 + HTTP/2 for REST API
    options.ListenLocalhost(5000, o => {
        o.Protocols = HttpProtocols.Http1AndHttp2;
        o.UseHttps();
    });
});
```

### 🏭 Real-Time Use Case

**Production topology for a trading API:**
```
  Internet
      │
  [NGINX] (reverse proxy: SSL termination, static files, DDOS protection)
      │ HTTP/2 to Kestrel
  [Kestrel] (ASP.NET Core) ─── [In-process] ─── [Trading API app]
      │
  [Redis / PostgreSQL]
```
- NGINX handles SSL, rate limiting at network level, serves static files
- Kestrel handles all dynamic API requests in-process with the .NET app — eliminates IIS process switching overhead
- Result: ~50% lower latency vs classic IIS-hosted ASP.NET for the same hardware

### ⚙️ Production-Grade Implementation

| Feature | Kestrel | IIS (in-process) | IIS (out-of-process) |
|---|---|---|---|
| **OS** | Cross-platform | Windows only | Windows only |
| **Protocol** | HTTP/1.1, HTTP/2, HTTP/3, WS, gRPC | HTTP/1.1, HTTP/2 | HTTP/1.1 |
| **Performance** | Highest | High | Lower (process switch overhead) |
| **Deployment** | Docker, Linux, Windows | Windows IIS | Windows IIS |
| **SSL** | Supports (or delegate to proxy) | IIS handles | IIS handles |

---

## Q26. How logging works in .NET?

### 📌 Definition
.NET's logging system (`Microsoft.Extensions.Logging`) is built around three core abstractions: **`ILogger<T>`** (the injection point for typed logging), **`ILoggerFactory`** (creates `ILogger` instances and dispatches to providers), and **`ILoggerProvider`** (a sink adapter — console, file, Seq, Application Insights, OpenTelemetry). When you call `_logger.LogInformation(...)`, the system checks if the message's log level passes the configured minimum threshold, then formats the message and dispatches it to all registered providers concurrently. Structured logging (message templates with named parameters like `{OrderId}`) preserves the parameters as machine-readable key-value pairs alongside the rendered string — essential for querying logs in Kibana, Azure Monitor, or Seq.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Plain Console.WriteLine — not structured, not filterable, not production-ready
Console.WriteLine("Order 12345 executed at price 24500");
// In Kibana: can't query "where OrderId=12345" — it's embedded in freetext

// ✅ Structured logging — parameters are first-class key-value pairs
_logger.LogInformation("Order {OrderId} executed at price {Price:C}", "12345", 24500m);
// JSON output: { "OrderId": "12345", "Price": 24500, "Message": "Order 12345 executed at price ₹24,500.00" }
// In Kibana: WHERE OrderId = "12345" — machine-queryable!
```

**Log levels (lowest to highest, in order):**
```csharp
_logger.LogTrace("Very verbose — method entry/exit tracking");
_logger.LogDebug("Diagnostic info — values, conditions during development");
_logger.LogInformation("Normal operations — request handled, order processed");
_logger.LogWarning("Unexpected but recoverable — retry triggered, slow query");
_logger.LogError(exception, "Error handled — order failed, DB unreachable");
_logger.LogCritical("Fatal — system cannot continue, needs immediate attention");
// Production: typically filter to Information and above (Debug/Trace too noisy)
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Structured logging in an order processing service**

```csharp
using System;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.DependencyInjection;

namespace TradingPlatform.Logging
{
    public record Order(string Id, string Symbol, decimal Price, int Qty);

    public sealed class OrderProcessingService
    {
        private readonly ILogger<OrderProcessingService> _logger;

        public OrderProcessingService(ILogger<OrderProcessingService> logger)
            => _logger = logger;

        public void Process(Order order)
        {
            // Use scoped logging context — all logs within scope share these properties
            using var scope = _logger.BeginScope(new
            {
                CorrelationId = Guid.NewGuid().ToString("N")[..8],
                OrderId       = order.Id,
                Symbol        = order.Symbol
            });

            _logger.LogInformation("Processing order {OrderId} for {Symbol} x{Qty} @ {Price:C}",
                order.Id, order.Symbol, order.Qty, order.Price);

            try
            {
                decimal value = order.Price * order.Qty;

                if (value > 500_000m)
                    _logger.LogWarning("High-value order {OrderId}: {Value:C} exceeds threshold",
                        order.Id, value);

                // Simulate processing
                ValidateAndExecute(order);

                _logger.LogInformation("Order {OrderId} executed successfully. TradeValue={TradeValue:C}",
                    order.Id, value);
            }
            catch (Exception ex)
            {
                // Always include exception object as first param — providers capture stack trace
                _logger.LogError(ex, "Order {OrderId} failed during processing",  order.Id);
                throw;
            }
        }

        private void ValidateAndExecute(Order order)
        {
            if (order.Qty <= 0)
                throw new InvalidOperationException($"Invalid quantity: {order.Qty}");
            // Simulate execution...
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var services = new ServiceCollection();
            services.AddLogging(cfg =>
            {
                cfg.AddConsole();          // Console provider
                cfg.SetMinimumLevel(LogLevel.Debug);
                // Production: cfg.AddApplicationInsights(...)
                //             cfg.AddOpenTelemetry(...)
            });
            services.AddTransient<OrderProcessingService>();

            var provider = services.BuildServiceProvider();
            var svc = provider.GetRequiredService<OrderProcessingService>();

            svc.Process(new Order("ORD-001", "NIFTY50", 24_500m, 10));
            svc.Process(new Order("ORD-002", "BANKNIFTY", 52_100m, 15)); // High value warning
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Provider | Use case |
|---|---|
| Console | Development, containers (stdout) |
| Application Insights | Azure hosted apps — traces + metrics + querying |
| OpenTelemetry | Cloud-agnostic — Jaeger, Zipkin, OTLP |
| Serilog / NLog | File sinks, enrichers, rolling log files |
| Seq | Local structured log server for development |

**Performance:** Use `LoggerMessage.Define<T1,T2>()` compiled delegates in hot paths to avoid allocations from message template parsing on every call.

---

## Q27. Configuration system in .NET Core.

### 📌 Definition
The .NET configuration system (`Microsoft.Extensions.Configuration`) is a **layered, provider-based key-value store** that aggregates configuration from multiple sources in a defined priority order. Sources are stacked — later sources override earlier ones for the same key. Built-in providers include JSON files (`appsettings.json`, `appsettings.{Environment}.json`), environment variables, command-line arguments, Azure Key Vault, AWS Parameter Store, and in-memory collections. Configuration is accessed as a flat key-value map with colon-separated hierarchy (`"Database:ConnectionString"`). Strongly-typed configuration is achieved via the Options pattern (`IOptions<T>`) which binds a configuration section to a POCO class and injects it via DI.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Default priority order (each level overrides the previous):
// 1. appsettings.json             ← base/default values
// 2. appsettings.{Environment}.json ← environment-specific overrides
// 3. User Secrets (Development)   ← local developer secrets, not in source control
// 4. Environment Variables        ← deployment-time overrides (Docker, K8s)
// 5. Command-line arguments       ← highest priority, runtime overrides

// This means: a setting in appsettings.json for DB connection string
// can be overridden by an environment variable in Kubernetes — no code change needed
```

**Reading configuration — three ways:**
```csharp
// Way 1: Direct IConfiguration access (strings only, no type safety)
string connStr = configuration["Database:ConnectionString"]!;
int timeout    = int.Parse(configuration["Database:TimeoutSeconds"]!);

// Way 2: GetSection + Bind (strongly typed, but manual)
var dbConfig = new DatabaseConfig();
configuration.GetSection("Database").Bind(dbConfig);

// Way 3: Options pattern (best — DI-integrated, validated, reloadable)
services.Configure<DatabaseConfig>(configuration.GetSection("Database"));
// Inject: IOptions<DatabaseConfig> opts → opts.Value.ConnectionString
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Multi-environment trading platform configuration**

```json
// appsettings.json (base — shared defaults)
{
  "TradingEngine": {
    "Exchange": "NSE",
    "MaxOrdersPerSecond": 1000,
    "RiskLimits": {
      "MaxPositionValue": 500000,
      "DailyLossLimit": 100000
    }
  },
  "ConnectionStrings": {
    "OrderDb": "Server=dev-db;Database=Orders;Integrated Security=true"
  }
}
```

```json
// appsettings.Production.json (overrides for prod)
{
  "TradingEngine": {
    "MaxOrdersPerSecond": 5000
  }
}
```

```csharp
using System;
using System.ComponentModel.DataAnnotations;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

namespace TradingPlatform.Config
{
    // ── Strongly-typed config POCO ────────────────────────────────────────
    public sealed class TradingEngineConfig
    {
        [Required]
        public string Exchange { get; set; } = "";
        [Range(1, 100_000)]
        public int MaxOrdersPerSecond { get; set; } = 1000;
        public RiskLimitsConfig RiskLimits { get; set; } = new();
    }

    public sealed class RiskLimitsConfig
    {
        public decimal MaxPositionValue { get; set; } = 500_000m;
        public decimal DailyLossLimit   { get; set; } = 100_000m;
    }

    // ── Consumer service using IOptions<T> ────────────────────────────────
    public sealed class TradingEngineService
    {
        private readonly TradingEngineConfig _config;

        // IOptions<T>: singleton, value fixed at startup
        // IOptionsSnapshot<T>: scoped, reloads per request if config changes
        // IOptionsMonitor<T>: singleton, notifies on config file change (hot reload)
        public TradingEngineService(IOptions<TradingEngineConfig> opts)
            => _config = opts.Value;

        public void PrintConfig()
        {
            Console.WriteLine($"Exchange       : {_config.Exchange}");
            Console.WriteLine($"MaxOrders/sec  : {_config.MaxOrdersPerSecond}");
            Console.WriteLine($"Max Position   : {_config.RiskLimits.MaxPositionValue:C}");
            Console.WriteLine($"Daily Loss Limit: {_config.RiskLimits.DailyLossLimit:C}");
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var config = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: false)
                .AddJsonFile($"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Development"}.json", optional: true)
                .AddEnvironmentVariables()  // ENV VAR: TradingEngine__Exchange=BSE overrides
                .AddCommandLine(args: Array.Empty<string>())
                .Build();

            var services = new ServiceCollection();
            services.AddOptions<TradingEngineConfig>()
                    .Bind(config.GetSection("TradingEngine"))
                    .ValidateDataAnnotations()  // Validates [Required], [Range] at startup
                    .ValidateOnStart();          // Fails fast if config invalid

            services.AddSingleton<TradingEngineService>();
            var provider = services.BuildServiceProvider();
            provider.GetRequiredService<TradingEngineService>().PrintConfig();
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Pattern | When to use |
|---|---|
| `IOptions<T>` | Values fixed at app start, singleton consumers |
| `IOptionsSnapshot<T>` | Values that can change, scoped consumers (per-request) |
| `IOptionsMonitor<T>` | Hot-reload: re-read from file/env on change without restart |
| `ValidateDataAnnotations()` | Fail fast on invalid config — catch errors at startup, not runtime |
| Azure Key Vault provider | Secrets (connection strings, API keys) — never in source control |

---

## Q28. Options pattern – why needed?

### 📌 Definition
The **Options pattern** is the .NET mechanism to inject strongly-typed configuration objects into services via the DI container, rather than injecting raw `IConfiguration` (which is string-only and not type-safe). It uses the `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>` wrappers that the DI system populates from the bound configuration section. The pattern enforces **separation of concerns**: services declare the settings they need as a POCO class, not the entire configuration graph. Each class only knows about its own settings. This also enables built-in validation, named options (multiple instances of the same options type with different values), and change notification.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ WITHOUT Options pattern — injecting IConfiguration directly
public class EmailService
{
    private readonly string _smtpHost;
    private readonly int    _smtpPort;

    public EmailService(IConfiguration config)
    {
        // Magic strings → typo-prone, not refactorable, no IntelliSense
        _smtpHost = config["Email:SmtpHost"]!;
        _smtpPort = int.Parse(config["Email:SmtpPort"]!);
        // No validation: app starts even if port is null/invalid
    }
}

// ✅ WITH Options pattern — type-safe, validated, IntelliSense-friendly
public class SmtpOptions
{
    public string Host { get; set; } = "";
    [Range(1, 65535)]
    public int Port { get; set; } = 587;
    public bool UseSsl { get; set; } = true;
}

public class EmailService
{
    private readonly SmtpOptions _opts;
    public EmailService(IOptions<SmtpOptions> options) => _opts = options.Value;
    // _opts.Host — IntelliSense, type-safe, validated before app starts
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Named options for multiple exchange connections**

```csharp
using System;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

namespace TradingPlatform.Options
{
    public sealed class ExchangeConnectionOptions
    {
        public string Name       { get; set; } = "";
        public string ApiEndpoint { get; set; } = "";
        public int    MaxRetries  { get; set; } = 3;
        public int    TimeoutMs   { get; set; } = 5000;
    }

    // Consumer uses IOptionsMonitor for named options
    public sealed class ExchangeConnector
    {
        private readonly IOptionsMonitor<ExchangeConnectionOptions> _monitor;

        public ExchangeConnector(IOptionsMonitor<ExchangeConnectionOptions> monitor)
            => _monitor = monitor;

        public void Connect(string exchangeName)
        {
            // Named options: get config for a specific exchange by name
            var opts = _monitor.Get(exchangeName);
            Console.WriteLine($"[{exchangeName}] Connecting to {opts.ApiEndpoint} " +
                              $"| Timeout={opts.TimeoutMs}ms | MaxRetries={opts.MaxRetries}");
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var services = new ServiceCollection();

            // Named options: two exchange configurations
            services.Configure<ExchangeConnectionOptions>("NSE", opts => {
                opts.Name        = "NSE";
                opts.ApiEndpoint = "wss://api.nse.co.in/live";
                opts.TimeoutMs   = 3000;
                opts.MaxRetries  = 5;
            });
            services.Configure<ExchangeConnectionOptions>("BSE", opts => {
                opts.Name        = "BSE";
                opts.ApiEndpoint = "wss://api.bseindia.com/live";
                opts.TimeoutMs   = 5000;
                opts.MaxRetries  = 3;
            });

            services.AddTransient<ExchangeConnector>();
            var provider = services.BuildServiceProvider();

            var connector = provider.GetRequiredService<ExchangeConnector>();
            connector.Connect("NSE");
            connector.Connect("BSE");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Interface | Lifetime | Reload on change | Use for |
|---|---|---|---|
| `IOptions<T>` | Singleton | ❌ No | Immutable startup config |
| `IOptionsSnapshot<T>` | Scoped | ✅ Per scope | Per-request fresh config |
| `IOptionsMonitor<T>` | Singleton | ✅ Yes (event) | Hot-reload, named options |

---

## Q29. What is CancellationToken?

### 📌 Definition
`CancellationToken` is a lightweight, thread-safe struct that signals **cooperative cancellation** between a token source (producer of the cancel signal) and observing code (consumer that checks the token and exits gracefully). The `CancellationTokenSource` creates and controls the token — calling `.Cancel()` sets the token's `IsCancellationRequested` to `true` and fires registered callbacks. Observing code (async methods, loops, I/O awaits) checks `.IsCancellationRequested` or calls `.ThrowIfCancellationRequested()`. When passed to I/O APIs (`HttpClient.GetAsync(ct)`, `SqlCommand.ExecuteAsync(ct)`, `Task.Delay(ms, ct)`), the cancellation abortsthe underlying I/O and throws `OperationCanceledException`. This model ensures clean shutdown — no abrupt thread abort, no resource leaks.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ WITHOUT CancellationToken — no way to stop in-progress work
public async Task<Order> FetchOrderAsync(string id)
{
    await Task.Delay(30_000); // 30-second operation — caller has no way to abort
    return await _db.FindAsync(id);
    // If user navigates away / request times out → this runs to completion anyway
    // Wastes resources on work that's no longer needed
}

// ✅ WITH CancellationToken — graceful cooperative cancellation
public async Task<Order> FetchOrderAsync(string id, CancellationToken ct)
{
    await Task.Delay(30_000, ct);     // Throws OperationCanceledException if cancelled
    ct.ThrowIfCancellationRequested(); // Manual check in CPU-bound sections
    return await _db.FindAsync(id, ct); // Propagates to DB driver
}

// Creating and triggering cancellation:
using var cts = new CancellationTokenSource(timeout: TimeSpan.FromSeconds(5)); // auto-cancel after 5s
try
{
    var order = await FetchOrderAsync("ORD-001", cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Request timed out or was cancelled");
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Order execution with timeout, user cancellation, and linked tokens**

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TradingPlatform.Cancellation
{
    public sealed class OrderExecutionService
    {
        // Simulates multi-step order execution with cancellation support at each step
        public async Task<string> ExecuteOrderAsync(
            string orderId,
            CancellationToken callerToken)  // Passed from HTTP request context
        {
            // Create a timeout token: auto-cancel if execution exceeds 10 seconds
            using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

            // Link: cancel if EITHER caller cancels OR timeout fires
            using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
                callerToken, timeoutCts.Token);

            CancellationToken ct = linkedCts.Token;

            try
            {
                Console.WriteLine($"[{orderId}] Step 1: Validating with risk engine...");
                await Task.Delay(2000, ct); // Simulate risk check I/O (2s)

                ct.ThrowIfCancellationRequested(); // Manual check after CPU work

                Console.WriteLine($"[{orderId}] Step 2: Reserving margin...");
                await Task.Delay(1000, ct); // Simulate margin reservation

                Console.WriteLine($"[{orderId}] Step 3: Submitting to exchange...");
                await Task.Delay(1500, ct); // Simulate exchange submission

                Console.WriteLine($"[{orderId}] ✅ Execution complete.");
                return $"Executed: {orderId}";
            }
            catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
            {
                Console.WriteLine($"[{orderId}] ⏱️  Execution TIMED OUT after 10s");
                throw new TimeoutException($"Order {orderId} execution timed out");
            }
            catch (OperationCanceledException) when (callerToken.IsCancellationRequested)
            {
                Console.WriteLine($"[{orderId}] 🚫 Execution CANCELLED by caller (client disconnected)");
                throw;
            }
        }
    }

    public static class Program
    {
        public static async Task Main()
        {
            var svc = new OrderExecutionService();

            // Test 1: Normal execution
            using var normalCts = new CancellationTokenSource();
            await svc.ExecuteOrderAsync("ORD-001", normalCts.Token);

            // Test 2: Caller cancels (simulates HTTP client disconnect)
            using var cancelCts = new CancellationTokenSource();
            var task = svc.ExecuteOrderAsync("ORD-002", cancelCts.Token);
            await Task.Delay(1500); // Let it start...
            cancelCts.Cancel();    // Simulate HTTP client disconnect
            try { await task; }
            catch (OperationCanceledException) { }
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Pattern | Purpose |
|---|---|
| `CancellationTokenSource(TimeSpan)` | Auto-timeout without external timer management |
| `CreateLinkedTokenSource(ct1, ct2)` | Combine caller CT + deadline CT — cancel on either |
| `ct.Register(() => cleanup())` | Run cleanup when cancellation fires (without throwing) |
| ASP.NET Core: `HttpContext.RequestAborted` | Pre-wired CT that fires when HTTP client disconnects |
| Pass `ct` to ALL async I/O calls | Ensures cancellation propagates through entire call chain |

---

## Q30. Parallel programming – PLINQ vs Task.

### 📌 Definition
- **PLINQ (Parallel LINQ)** — An extension of LINQ that automatically parallelizes query operations by partitioning the data source, running operators concurrently on multiple threads, and merging results. Activated by calling `.AsParallel()` on any `IEnumerable<T>`. Best suited for **data-parallel, stateless, CPU-bound computations** on in-memory collections — where each element is processed independently. Result ordering is not guaranteed by default but can be preserved with `.AsOrdered()` (at a performance cost).
- **`Task` / TPL (Task Parallel Library)** — A task-based model for expressing structured concurrency. `Parallel.ForEach`, `Parallel.Invoke`, and `Task.WhenAll` give precise control over work partitioning, cancellation, and exception aggregation. Better than PLINQ when: work items have different sizes (PLINQ can't load-balance well), you need cancellation, you need to monitor progress, or the work is async (PLINQ does not support `async` lambdas).

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Sequential: processes 1M records one at a time — underutilizes multi-core CPU
List<decimal> vaRValues = prices.Select(ComputeVaR).ToList();

// ✅ PLINQ: partitions data across CPU cores automatically
List<decimal> vaRValues = prices
    .AsParallel()                           // Activate PLINQ
    .WithDegreeOfParallelism(4)             // Limit to 4 threads
    .WithCancellation(cancellationToken)    // Stop all if token fires
    .Select(ComputeVaR)                     // Each partition runs concurrently
    .ToList();                              // Collect results (merged)

// ✅ Parallel.ForEach: explicit loop parallelism with options
var results = new System.Collections.Concurrent.ConcurrentBag<decimal>();
Parallel.ForEach(prices,
    new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount },
    price => results.Add(ComputeVaR(price)));

// ✅ Task.WhenAll: async fan-out (PLINQ doesn't support async)
var tasks = symbols.Select(sym => FetchPriceAsync(sym));
decimal[] fetched = await Task.WhenAll(tasks); // All concurrent, all async
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Portfolio VaR calculation across 10,000 positions — CPU-bound parallelism**

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Diagnostics;

namespace TradingPlatform.Parallel
{
    public record Position(string Symbol, decimal Quantity, decimal CurrentPrice);

    public static class PortfolioRiskEngine
    {
        // CPU-bound Monte Carlo VaR per position (no I/O)
        private static decimal ComputePositionVaR(Position pos)
        {
            double sum = 0;
            var rng = new Random(pos.Symbol.GetHashCode());
            for (int i = 0; i < 10_000; i++) // Monte Carlo iterations
                sum += rng.NextDouble() * (double)(pos.Quantity * pos.CurrentPrice);
            return (decimal)(sum / 10_000) * 0.02m; // 95th percentile approximation
        }

        public static void Main()
        {
            var positions = Enumerable.Range(1, 10_000).Select(i =>
                new Position($"SYM-{i:D5}", quantity: i % 100 + 1, currentPrice: 100m + (i % 500))).ToList();

            // Sequential baseline
            var sw = Stopwatch.StartNew();
            var seqResults = positions.Select(ComputePositionVaR).ToList();
            sw.Stop(); Console.WriteLine($"Sequential: {sw.ElapsedMilliseconds}ms");

            // PLINQ parallel — 4x speedup on 4-core machine
            sw.Restart();
            var plinqResults = positions
                .AsParallel()
                .WithDegreeOfParallelism(Environment.ProcessorCount)
                .Select(ComputePositionVaR)
                .ToList();
            sw.Stop(); Console.WriteLine($"PLINQ:      {sw.ElapsedMilliseconds}ms");

            // Parallel.ForEach — similar speedup, more control
            var parallelResults = new decimal[positions.Count];
            sw.Restart();
            System.Threading.Tasks.Parallel.For(0, positions.Count,
                new System.Threading.Tasks.ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount },
                i => parallelResults[i] = ComputePositionVaR(positions[i]));
            sw.Stop(); Console.WriteLine($"Parallel.For: {sw.ElapsedMilliseconds}ms");

            decimal totalSeq = seqResults.Sum();
            decimal totalPar = plinqResults.Sum();
            Console.WriteLine($"Portfolio VaR (Sequential): {totalSeq:C}");
            Console.WriteLine($"Portfolio VaR (PLINQ):      {totalPar:C}");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Dimension | PLINQ | Task / Parallel.ForEach | Task.WhenAll |
|---|---|---|---|
| **Best for** | In-memory CPU-bound data | Structured parallel loops | Async I/O fan-out |
| **Async lambda** | ❌ Not supported | ❌ No (use Task array) | ✅ Native |
| **Cancellation** | `WithCancellation(ct)` | `ParallelOptions.CancellationToken` | `Task.WhenAll` + CT in each Task |
| **Exception** | `AggregateException` | `AggregateException` | `AggregateException` (await unwraps first) |
| **Ordering** | Not guaranteed (use `.AsOrdered()`) | Not guaranteed | Ordered by index |

**Performance rules:**
- PLINQ overhead dominates for < 1000 simple elements — benchmark before parallelizing
- Call `WithDegreeOfParallelism(Environment.ProcessorCount)` — prevent ThreadPool starvation
- Never use PLINQ with async lambdas — use `Task.WhenAll` instead
- For I/O bound work: always `Task.WhenAll` — no thread per operation

---

---

## Q31. Thread safety – locks vs concurrent collections.

### 📌 Definition
**Thread safety** means a data structure or code path produces correct results when accessed concurrently from multiple threads — no race conditions, no data corruption, no missed updates. .NET provides two primary mechanisms: **explicit locks** (`lock`, `Monitor`, `Mutex`, `SemaphoreSlim`, `ReaderWriterLockSlim`) that serialize access to shared state, and **concurrent collections** (`ConcurrentDictionary<K,V>`, `ConcurrentQueue<T>`, `ConcurrentBag<T>`, `ConcurrentStack<T>`, `BlockingCollection<T>`) that use fine-grained internal locking, `Interlocked` operations, and compare-and-swap (CAS) hardware instructions to achieve lock-free or low-contention thread safety without requiring external synchronization by the caller.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Non-thread-safe shared state — race condition
private Dictionary<string, decimal> _prices = new();

// Thread A: _prices["NIFTY50"] = 24_500m;
// Thread B: _prices["NIFTY50"] = 24_510m;
// Possible: both threads resize the internal array simultaneously → corruption, infinite loop

// ✅ Option 1: lock — simple, serializes all access
private readonly Dictionary<string, decimal> _prices = new();
private readonly object _lock = new();

public void UpdatePrice(string symbol, decimal price)
{
    lock (_lock) { _prices[symbol] = price; } // Only one thread at a time
}

public decimal GetPrice(string symbol)
{
    lock (_lock) { return _prices.TryGetValue(symbol, out var p) ? p : 0m; }
}

// ✅ Option 2: ConcurrentDictionary — no manual locking needed
private readonly ConcurrentDictionary<string, decimal> _prices = new();
public void UpdatePrice(string s, decimal v) => _prices[s] = v;       // Thread-safe
public decimal GetPrice(string s) => _prices.GetValueOrDefault(s, 0m); // Thread-safe

// ✅ Option 3: ReaderWriterLockSlim — multiple concurrent readers, exclusive writers
// Best when reads >> writes (e.g., price reads by 100 threads, writes by 1 feed)
private readonly ReaderWriterLockSlim _rwLock = new();
private readonly Dictionary<string, decimal> _prices2 = new();

public decimal GetPrice2(string symbol)
{
    _rwLock.EnterReadLock();    // Multiple threads can read simultaneously
    try { return _prices2.TryGetValue(symbol, out var p) ? p : 0m; }
    finally { _rwLock.ExitReadLock(); }
}
public void UpdatePrice2(string symbol, decimal price)
{
    _rwLock.EnterWriteLock();   // Exclusive — all readers blocked
    try { _prices2[symbol] = price; }
    finally { _rwLock.ExitWriteLock(); }
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Thread-safe order book with concurrent readers and writers**

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace TradingPlatform.ThreadSafety
{
    public sealed class ThreadSafeOrderBook
    {
        // ConcurrentDictionary: fine-grained locking per bucket — reads are lock-free
        private readonly ConcurrentDictionary<string, decimal> _bestBid = new();
        private readonly ConcurrentDictionary<string, decimal> _bestAsk = new();

        // ConcurrentQueue: multiple producers, multiple consumers — lock-free FIFO
        private readonly ConcurrentQueue<(string Symbol, decimal Price, string Side)> _tradeLog = new();

        // Atomic counter — Interlocked: no lock, hardware CAS instruction
        private long _totalTrades = 0;

        public void UpdateBid(string symbol, decimal price)
        {
            // AddOrUpdate is atomic for the KEY operation — safe under concurrency
            _bestBid.AddOrUpdate(symbol, price, (_, _) => price);
        }

        public void UpdateAsk(string symbol, decimal price)
            => _bestAsk.AddOrUpdate(symbol, price, (_, _) => price);

        public (decimal bid, decimal ask, decimal spread)? GetQuote(string symbol)
        {
            if (_bestBid.TryGetValue(symbol, out decimal bid) &&
                _bestAsk.TryGetValue(symbol, out decimal ask))
                return (bid, ask, ask - bid);
            return null;
        }

        public void RecordTrade(string symbol, decimal price, string side)
        {
            _tradeLog.Enqueue((symbol, price, side)); // Lock-free enqueue
            Interlocked.Increment(ref _totalTrades);   // Atomic increment — no lock
        }

        public long TotalTrades => Interlocked.Read(ref _totalTrades); // Atomic read

        public IList<string> GetRecentTrades(int count)
        {
            // ConcurrentQueue is safe to enumerate (snapshot semantics)
            return _tradeLog.TakeLast(count)
                            .Select(t => $"{t.Symbol} {t.Side} @ {t.Price:C}")
                            .ToList();
        }
    }

    public static class Program
    {
        public static async Task Main()
        {
            var book = new ThreadSafeOrderBook();

            // 4 concurrent feed updaters
            var feedTask = Task.WhenAll(Enumerable.Range(0, 4).Select(i => Task.Run(() =>
            {
                var rng = new Random(i);
                for (int j = 0; j < 1000; j++)
                {
                    book.UpdateBid("NIFTY50", 24_490m + rng.Next(20));
                    book.UpdateAsk("NIFTY50", 24_510m + rng.Next(20));
                }
            })));

            // 10 concurrent readers
            var readTask = Task.WhenAll(Enumerable.Range(0, 10).Select(_ => Task.Run(() =>
            {
                for (int j = 0; j < 500; j++)
                    book.GetQuote("NIFTY50");
            })));

            await Task.WhenAll(feedTask, readTask);

            book.RecordTrade("NIFTY50", 24_501m, "BUY");
            book.RecordTrade("NIFTY50", 24_499m, "SELL");

            var quote = book.GetQuote("NIFTY50");
            Console.WriteLine($"Final: Bid={quote?.bid:C} Ask={quote?.ask:C} Spread={quote?.spread:C}");
            Console.WriteLine($"Total trades: {book.TotalTrades}");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Mechanism | Use when | Performance |
|---|---|---|
| `lock` / `Monitor` | Simple, infrequent critical sections | Moderate |
| `ReaderWriterLockSlim` | Many readers, few writers | Good (readers concurrent) |
| `ConcurrentDictionary<K,V>` | Frequent concurrent reads + writes | Excellent (fine-grained) |
| `ConcurrentQueue<T>` | Multi-producer/consumer FIFO | Excellent (lock-free) |
| `Interlocked` | Single counter/flag operations | Best (hardware atomics) |
| `SemaphoreSlim` | Async-compatible limiting concurrency | Good |

---

## Q32. volatile vs lock.

### 📌 Definition
- **`volatile`** — A keyword applied to a field that tells the JIT and CPU: do not cache this field's value in a register or reorder reads/writes around it. Every read must fetch from main memory; every write must flush to main memory immediately. Prevents compiler and CPU reordering optimizations for that specific field. Provides the weakest thread-safety guarantee — visibility only, not atomicity. Only safe for single `bool`, `int`, reference reads/writes where atomicity is inherent (on 32-bit: `int`/`bool`/`float`; on 64-bit: `long`/`double` too).
- **`lock`** — A mutual exclusion mechanism that ensures only one thread executes the protected code block at a time. Internally uses `Monitor.Enter`/`Monitor.Exit` which establish full memory barriers (prevent all reordering across the lock boundary) AND provide mutual exclusion. Suitable for any compound operation that must be atomic as a whole.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Without volatile: JIT may cache _running in a register
private bool _running = true;

// Thread A:
while (_running) DoWork(); // JIT may read _running ONCE, hoist it as constant → infinite loop

// Thread B:
_running = false; // Thread A may never see this if JIT cached the true value

// ✅ volatile prevents caching:
private volatile bool _running = true;
while (_running) DoWork(); // Always reads from memory — Thread B's write is visible

// What volatile DOESN'T provide: atomicity for compound operations
private volatile int _count = 0;
_count++; // ❌ NOT atomic! Read-Modify-Write is three instructions — race condition possible

// ✅ For atomic increment: use Interlocked
Interlocked.Increment(ref _count); // Single atomic hardware instruction

// ✅ For any compound operation: use lock
lock (_lockObj) { _count++; if (_count > 100) _count = 0; } // Atomic as a unit
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Market data feed processor with graceful shutdown flag**

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TradingPlatform.Volatile
{
    public sealed class MarketFeedProcessor
    {
        // volatile bool: thread A writes, thread B reads — no compound ops needed → volatile OK
        private volatile bool _isRunning = false;

        // For counts: Interlocked (no lock overhead, hardware atomic)
        private long _processedCount = 0;
        private long _errorCount     = 0;

        // For publishing current price (reference type write is atomic on 64-bit)
        // volatile ensures all threads see latest reference immediately
        private volatile decimal _lastPrice = 0m;

        // For a compound check-and-update: lock is needed
        private readonly object _stateLock = new();
        private string _currentState       = "IDLE";

        public void Start()
        {
            lock (_stateLock) // Compound: check + assign must be atomic
            {
                if (_currentState != "IDLE")
                    throw new InvalidOperationException($"Cannot start from state: {_currentState}");
                _currentState = "RUNNING";
            }
            _isRunning = true;
            Task.Run(ProcessLoop);
        }

        public void Stop()
        {
            _isRunning = false; // Simple bool write — volatile ensures visibility
            lock (_stateLock) { _currentState = "STOPPED"; }
        }

        private void ProcessLoop()
        {
            var rng = new Random();
            while (_isRunning) // volatile read — always gets latest value
            {
                try
                {
                    _lastPrice = 24_500m + (decimal)rng.NextDouble() * 100; // atomic ref write
                    Interlocked.Increment(ref _processedCount);
                    Thread.Sleep(10);
                }
                catch
                {
                    Interlocked.Increment(ref _errorCount);
                }
            }
        }

        public decimal CurrentPrice  => _lastPrice;                      // volatile read
        public long ProcessedCount   => Interlocked.Read(ref _processedCount);
        public long ErrorCount       => Interlocked.Read(ref _errorCount);
    }

    public static class Program
    {
        public static async Task Main()
        {
            var processor = new MarketFeedProcessor();
            processor.Start();
            await Task.Delay(200);
            Console.WriteLine($"Price: {processor.CurrentPrice:C} | Processed: {processor.ProcessedCount}");
            processor.Stop();
            await Task.Delay(50);
            Console.WriteLine($"Final count: {processor.ProcessedCount} | Errors: {processor.ErrorCount}");
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| | `volatile` | `Interlocked` | `lock` |
|---|---|---|---|
| **Guarantees** | Visibility only | Atomic single op | Visibility + Mutual exclusion |
| **Performance** | Fastest | Very fast (hardware) | Slowest (OS involvement if contended) |
| **Use for** | Single bool/int flag | Counters, CAS operations | Any compound critical section |

---

## Q33. What is async void problem?

### 📌 Definition
`async void` is a method that has the `async` keyword but returns `void` instead of `Task` or `ValueTask`. While syntactically valid (and required for legacy event handlers like `Button.Click_Handler`), it has a critical flaw: **unhandled exceptions thrown inside an `async void` method are not captured in any `Task` — they are rethrown on the current `SynchronizationContext` and typically crash the process**. Unlike `async Task`, you cannot `await` an `async void` method (so callers cannot know when it completes), cannot `try/catch` exceptions from it at the call site, and cannot compose it with other Tasks.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ async void — exception kills the process
async void ProcessOrderAsync()
{
    await Task.Delay(100);
    throw new Exception("Order failed!"); // Uncaught → process crash
}

// Caller:
ProcessOrderAsync();           // Fire-and-forget — no await, no try/catch possible
// try { ProcessOrderAsync(); } catch { } // ← Does NOT catch the async exception!
// Exception fires after 100ms on thread pool → AppDomain.UnhandledException

// ✅ async Task — exception captured in Task, re-thrown on await
async Task ProcessOrderAsync()
{
    await Task.Delay(100);
    throw new Exception("Order failed!");
}

// Caller can handle:
try { await ProcessOrderAsync(); }
catch (Exception ex) { Console.WriteLine($"Caught: {ex.Message}"); } // ✅ Works!

// ✅ Only valid use of async void: event handlers (language requirement)
private async void Button_Click(object sender, EventArgs e)
{
    try { await LoadDataAsync(); }   // ← Always wrap in try/catch inside async void
    catch (Exception ex) { ShowError(ex); } // Handle exceptions INSIDE the method
}
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Background order processor — wrong vs right approach**

```csharp
using System;
using System.Threading.Tasks;

namespace TradingPlatform.AsyncVoid
{
    public sealed class OrderProcessor
    {
        // ❌ DANGEROUS: async void fire-and-forget
        public async void ProcessOrderBad(string orderId)
        {
            await Task.Delay(50);
            if (orderId.StartsWith("BAD"))
                throw new InvalidOperationException("Bad order!"); // → process crash!
            Console.WriteLine($"[BAD] Processed: {orderId}");
        }

        // ✅ GOOD: async Task — exception is captured, caller can handle
        public async Task ProcessOrderGood(string orderId)
        {
            await Task.Delay(50);
            if (orderId.StartsWith("BAD"))
                throw new InvalidOperationException("Bad order!"); // → captured in Task
            Console.WriteLine($"[GOOD] Processed: {orderId}");
        }

        // ✅ GOOD: fire-and-forget with explicit error handling
        public void TriggerOrderProcessing(string orderId)
        {
            // Discard with _ but ensure exceptions are observed
            _ = ProcessWithErrorHandling(orderId);
        }

        private async Task ProcessWithErrorHandling(string orderId)
        {
            try { await ProcessOrderGood(orderId); }
            catch (Exception ex)
            {
                // Log error — don't let it propagate to void context
                Console.WriteLine($"[ERROR] {orderId}: {ex.Message}");
            }
        }
    }

    public static class Program
    {
        public static async Task Main()
        {
            var processor = new OrderProcessor();

            // ✅ Task-based: catch works
            try { await processor.ProcessOrderGood("BAD-001"); }
            catch (InvalidOperationException ex) { Console.WriteLine($"Caught: {ex.Message}"); }

            // ✅ Fire-and-forget with error handling
            processor.TriggerOrderProcessing("BAD-002");
            await Task.Delay(200); // Wait for background task
        }
    }
}
```

### ⚙️ Production-Grade Implementation | Rule | Reason |
|---|---|
| Never use `async void` except for event handlers | Exceptions crash the process |
| Inside event handlers: always wrap in `try/catch` | Handle exceptions within the `async void` method |
| Fire-and-forget: use `_ = RunWithHandling(...)` pattern | Exceptions observed and logged, not silently swallowed |
| Set `TaskScheduler.UnobservedTaskException` handler | Catch any missed Task exceptions in legacy code |

---

## Q34. Custom attributes – how to create and use.

### 📌 Definition
**Custom attributes** are classes that inherit from `System.Attribute` and decorate code elements (classes, methods, properties, parameters, assemblies) with arbitrary metadata. The metadata is stored in the assembly's metadata tables at compile time and retrieved via reflection at runtime. Attributes are a primary extensibility mechanism in .NET frameworks — ASP.NET Core uses `[HttpGet]`, `[Authorize]`, `[Route]` attributes; EF Core uses `[Key]`, `[Column]`, `[Required]`; data validation uses `[Range]`, `[MaxLength]`. Creating a custom attribute means defining a class with `[AttributeUsage(...)]` to specify where it can be applied, then reading it with `Type.GetCustomAttribute<T>()` or `MemberInfo.GetCustomAttributes()`.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Without attributes: convention-based code inspection is hard and fragile
// With attributes: self-describing metadata baked into the type/member itself

// Define a custom attribute:
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
public sealed class AuditLogAttribute : Attribute
{
    public string Action      { get; }
    public bool   LogResponse { get; init; } = false;

    public AuditLogAttribute(string action) => Action = action;
}

// Apply the attribute to a method:
public class OrderController
{
    [AuditLog("PlaceOrder", LogResponse = true)]
    public void PlaceOrder(string orderId) { /* ... */ }

    [AuditLog("CancelOrder")]
    public void CancelOrder(string orderId) { /* ... */ }
}

// Read the attribute at runtime via reflection (or interceptors):
var method = typeof(OrderController).GetMethod("PlaceOrder")!;
var attr   = method.GetCustomAttribute<AuditLogAttribute>();
if (attr != null)
    Console.WriteLine($"Action: {attr.Action}, LogResponse: {attr.LogResponse}");
```

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Attribute-driven validation in an order submission pipeline**

```csharp
using System;
using System.Reflection;
using System.ComponentModel.DataAnnotations;
using System.Collections.Generic;

namespace TradingPlatform.Attributes
{
    // ── Custom attribute 1: marks a property as a trading symbol ──────────
    [AttributeUsage(AttributeTargets.Property)]
    public sealed class TradingSymbolAttribute : Attribute
    {
        public string Exchange { get; }
        public TradingSymbolAttribute(string exchange) => Exchange = exchange;
    }

    // ── Custom attribute 2: marks a method as requiring audit logging ──────
    [AttributeUsage(AttributeTargets.Method, Inherited = true)]
    public sealed class AuditTrailAttribute : Attribute
    {
        public string Category { get; }
        public bool   CaptureArgs { get; init; }
        public AuditTrailAttribute(string category) => Category = category;
    }

    // ── Custom attribute 3: marks a method's max allowed execution time ───
    [AttributeUsage(AttributeTargets.Method)]
    public sealed class MaxExecutionTimeAttribute : Attribute
    {
        public int Milliseconds { get; }
        public MaxExecutionTimeAttribute(int ms) => Milliseconds = ms;
    }

    // ── Order class using attributes ───────────────────────────────────────
    public sealed class OrderRequest
    {
        [TradingSymbol("NSE")]          // Custom validation attribute
        [Required, MaxLength(20)]
        public string Symbol { get; set; } = "";

        [Range(1, 10_000)]
        public decimal Quantity { get; set; }

        [Range(0.01, 1_000_000)]
        public decimal Price { get; set; }
    }

    // ── Attribute-driven validator and auditor ─────────────────────────────
    public static class AttributeProcessor
    {
        // Reads TradingSymbolAttribute and validates format
        public static List<string> ValidateOrder(OrderRequest order)
        {
            var errors = new List<string>();
            var props = typeof(OrderRequest).GetProperties();

            foreach (var prop in props)
            {
                object? value = prop.GetValue(order);

                // Check TradingSymbolAttribute
                var symbolAttr = prop.GetCustomAttribute<TradingSymbolAttribute>();
                if (symbolAttr != null && value is string sym)
                {
                    if (string.IsNullOrEmpty(sym))
                        errors.Add($"{prop.Name} is required for {symbolAttr.Exchange} exchange");
                    if (sym.Length > 20)
                        errors.Add($"{prop.Name} too long for {symbolAttr.Exchange}");
                    Console.WriteLine($"  [{prop.Name}] Trading symbol on {symbolAttr.Exchange}: '{sym}'");
                }

                // Check standard DataAnnotations
                var context = new ValidationContext(order) { MemberName = prop.Name };
                var results = new List<ValidationResult>();
                Validator.TryValidateProperty(value, context, results);
                errors.AddRange(results.Select(r => r.ErrorMessage ?? ""));
            }
            return errors;
        }

        // Checks AuditTrailAttribute on a method and logs accordingly
        public static void InvokeWithAudit(object target, string methodName, object[] args)
        {
            var method = target.GetType().GetMethod(methodName);
            if (method == null) throw new MissingMethodException(methodName);

            var audit = method.GetCustomAttribute<AuditTrailAttribute>();
            var maxTime = method.GetCustomAttribute<MaxExecutionTimeAttribute>();

            if (audit != null)
                Console.WriteLine($"[AUDIT] {audit.Category}: {methodName}" +
                    (audit.CaptureArgs ? $" args=[{string.Join(", ", args)}]" : ""));

            var sw = System.Diagnostics.Stopwatch.StartNew();
            method.Invoke(target, args);
            sw.Stop();

            if (maxTime != null && sw.ElapsedMilliseconds > maxTime.Milliseconds)
                Console.WriteLine($"[PERF] ⚠️ {methodName} took {sw.ElapsedMilliseconds}ms > limit {maxTime.Milliseconds}ms");
        }
    }

    public class OrderService
    {
        [AuditTrail("OrderManagement", CaptureArgs = true)]
        [MaxExecutionTime(100)]
        public void PlaceOrder(string orderId, decimal price)
        {
            Console.WriteLine($"  Placing order {orderId} @ {price:C}");
            System.Threading.Thread.Sleep(50); // Simulate work
        }
    }

    public static class Program
    {
        public static void Main()
        {
            // Validate order using attributes
            var order = new OrderRequest { Symbol = "NIFTY50", Quantity = 10, Price = 24_500m };
            Console.WriteLine("Validating order:");
            var errors = AttributeProcessor.ValidateOrder(order);
            Console.WriteLine(errors.Count == 0 ? "  ✅ Valid" : $"  ❌ Errors: {string.Join("; ", errors)}");

            // Invoke with audit trail
            var svc = new OrderService();
            AttributeProcessor.InvokeWithAudit(svc, "PlaceOrder", new object[] { "ORD-001", 24_500m });
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| `AttributeTargets` value | What it decorates |
|---|---|
| `Class` | Type declaration |
| `Method` | Method definition |
| `Property` | Property declaration |
| `Parameter` | Method parameter |
| `Assembly` | Entire assembly |
| `All` | Any code element |

**Performance:** `GetCustomAttribute<T>()` uses reflection — cache results in a static `ConcurrentDictionary<MemberInfo, T?>` at startup to avoid per-call reflection overhead in production.

---

## Q35. What is IL code?

### 📌 Definition
**IL (Intermediate Language)**, also called **CIL (Common Intermediate Language)** or **MSIL (Microsoft Intermediate Language)**, is the CPU-agnostic bytecode that the C# compiler (Roslyn) emits into assembly `.dll`/`.exe` files. Rather than targeting a specific CPU architecture, IL is a stack-based instruction set that the CLR's JIT compiler translates to native code at runtime. This indirection enables: cross-platform portability (same IL runs on x64, ARM64, WASM), runtime code verification (CLR checks IL for type safety before running), cross-language interoperability (C#, F#, VB.NET all compile to IL and can interop), and runtime metaprogramming (IL can be emitted dynamically via `System.Reflection.Emit`).

### 🔍 Why It Exists / Problem It Solves

```csharp
// C# source — what you write:
public static int Add(int a, int b) => a + b;

// IL output (view with: ildasm.exe or dotnet-ildasm tool):
// .method public hidebysig static int32 Add(int32 a, int32 b) cil managed
// {
//   .maxstack 2
//   IL_0000: ldarg.0       // push arg 'a' onto evaluation stack
//   IL_0001: ldarg.1       // push arg 'b' onto evaluation stack
//   IL_0002: add           // pop both, add them, push result
//   IL_0003: ret           // return top of evaluation stack
// }

// Inspecting IL at runtime:
using System.Reflection;
MethodBody body = typeof(Calculator).GetMethod("Add")!.GetMethodBody()!;
byte[] ilBytes = body.GetILAsByteArray()!;
Console.WriteLine($"IL bytes: {BitConverter.ToString(ilBytes)}"); // 03 04 58 2A

// Dynamic IL emission via Reflection.Emit (used by ORMs, proxy generators):
using System.Reflection.Emit;
var method = new DynamicMethod("Add", typeof(int), new[] { typeof(int), typeof(int) });
var il     = method.GetILGenerator();
il.Emit(OpCodes.Ldarg_0);
il.Emit(OpCodes.Ldarg_1);
il.Emit(OpCodes.Add);
il.Emit(OpCodes.Ret);
var add = (Func<int,int,int>)method.CreateDelegate(typeof(Func<int,int,int>));
Console.WriteLine(add(3, 4)); // 7
```

### ⚙️ Production-Grade Implementation
IL is important for: diagnosing JIT inlining decisions (PerfView, dotnet-trace), understanding why code is slow (can't inline, too many locals), implementing AOP via PostSharp/Fody (IL weaving), building source generators that avoid runtime reflection, and understanding how generics work (each `List<int>` and `List<string>` share one IL definition but JIT produces separate native code for value types).

---

## Q36. What is strong naming?

### 📌 Definition
A **strong-named assembly** is a .NET assembly signed with a cryptographic public/private key pair, producing a 4-part identity: name + version + culture + public key token. Strong naming ensures: (1) **Uniqueness** — no two publishers can produce an assembly with the same strong name, (2) **Integrity** — tampering with the assembly bytes after signing is detectable, (3) **Side-by-side versioning** — the GAC (Global Assembly Cache) distinguishes assemblies by full strong name, allowing two versions to coexist. In modern .NET (Core/5+), the GAC and strong naming are less critical (no shared GAC) but still used for NuGet package identity, assembly binding redirects, and security-sensitive scenarios.

```csharp
// Create key pair:
// sn.exe -k TradingSDK.snk

// Sign in .csproj:
// <AssemblyOriginatorKeyFile>TradingSDK.snk</AssemblyOriginatorKeyFile>
// <SignAssembly>true</SignAssembly>

// Result: assembly identity
// TradingSDK, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089
// PublicKeyToken = 8-byte hash of public key — unique, unforgeable without private key
```

---

## Q37. Assembly loading and binding.

### 📌 Definition
**Assembly loading** is the CLR process of locating, loading into memory, and validating a .NET assembly. **Binding** is the resolution step — given an assembly reference (name, version, culture, public key token), find the matching physical file on disk. In .NET Core/5+, binding is controlled by `AssemblyLoadContext` — an isolation boundary that allows multiple versions of the same assembly to coexist in the same process (plugin systems). The default context probes: application base directory, NuGet dependencies in `deps.json`, and runtime store.

```csharp
// AssemblyLoadContext for plugin isolation:
public sealed class PluginLoadContext : AssemblyLoadContext
{
    private readonly AssemblyDependencyResolver _resolver;

    public PluginLoadContext(string pluginPath) : base(isCollectible: true) // Can unload
        => _resolver = new AssemblyDependencyResolver(pluginPath);

    protected override Assembly? Load(AssemblyName name)
    {
        string? path = _resolver.ResolveAssemblyToPath(name);
        return path != null ? LoadFromAssemblyPath(path) : null; // Load isolated
    }
}

// Usage — load and unload plugins without restarting the app:
var ctx  = new PluginLoadContext("C:/plugins/risk-engine.dll");
var asm  = ctx.LoadFromAssemblyPath("C:/plugins/risk-engine.dll");
var type = asm.GetType("RiskPlugin.Engine")!;
// ... use plugin ...
ctx.Unload(); // GC the plugin assembly (isCollectible: true)
```

---

## Q38. What is AppDomain (legacy vs .NET Core)?

### 📌 Definition
An **AppDomain** (Application Domain) was a .NET Framework mechanism for isolating code within a single OS process — each AppDomain had its own type system, security policy, and could be independently loaded and unloaded. Used for plugin systems, sandboxing, and multi-tenant hosting in ASP.NET Classic. In **.NET Core / .NET 5+, AppDomain is not supported as a real isolation boundary** — there is only one AppDomain per process. The APIs still exist but `AppDomain.CreateDomain()` throws `PlatformNotSupportedException`. The replacement for plugin isolation is `AssemblyLoadContext` (Q37). For process isolation (security sandboxing), .NET Core uses OS-level process isolation.

```csharp
// .NET Framework (legacy):
AppDomain sandbox = AppDomain.CreateDomain("PluginSandbox");
// Load plugin into sandbox, run with restricted permissions, unload sandbox when done
AppDomain.Unload(sandbox);

// .NET Core / .NET 5+ — only one AppDomain:
Console.WriteLine(AppDomain.CurrentDomain.FriendlyName); // Still works (always "default")
// AppDomain.CreateDomain("...") → throws PlatformNotSupportedException

// Modern replacement: AssemblyLoadContext for type isolation (Q37)
// Or: out-of-process with gRPC/named pipes for true sandboxing
```
---

## Q39. Serialization – JSON vs binary.

### 📌 Definition
**Serialization** is the process of converting an in-memory object graph into a storable/transmittable format. **JSON serialization** produces human-readable, language-agnostic text (universal REST API format); **binary serialization** (MessagePack, Protobuf, CBOR) produces compact, fast-to-parse binary representations. JSON is self-describing (field names in every payload) but verbose; binary is type-described by a schema (Protobuf `.proto` files) but 2-10x smaller and faster to parse. Choose JSON for APIs where readability and interoperability matter; binary for internal microservice communication, storage, and high-throughput scenarios.

```csharp
// JSON (System.Text.Json):
string json = JsonSerializer.Serialize(order);
// {"OrderId":"ORD-001","Symbol":"NIFTY50","Price":24500.5,"Quantity":10}
Order restored = JsonSerializer.Deserialize<Order>(json)!;

// MessagePack (binary — requires MessagePack NuGet):
byte[] bytes = MessagePackSerializer.Serialize(order);    // ~30 bytes vs ~80 bytes JSON
Order back  = MessagePackSerializer.Deserialize<Order>(bytes);
// 3-5x smaller, 2-4x faster to serialize/deserialize

// Protobuf (Google protocol buffers — requires Grpc.Tools or protobuf-net):
// Define in .proto file, generate C# classes, serialize:
byte[] proto = order.ToByteArray(); // proto-net or Google.Protobuf
// Used in gRPC — strict schema, backward-compatible versioning
```

---

## Q40. System.Text.Json vs Newtonsoft.

### 📌 Definition
- **`System.Text.Json`** — Microsoft's built-in, high-performance, standards-compliant JSON serializer (included in .NET 3.0+). Uses `Span<T>` and `Utf8JsonWriter` for zero-copy, allocation-friendly serialization. Strict by default (no reference loops, no comment support, case-sensitive). Designed for throughput and correctness. Source generators (`.NET 6+`) can eliminate runtime reflection entirely.
- **`Newtonsoft.Json` (`Json.NET`)** — Third-party library by James Newton-King, the de-facto .NET JSON standard before `System.Text.Json`. Feature-rich: reference loop handling, JSON comments, `dynamic`, custom converters, flexible polymorphism (`$type` property), LINQ-to-JSON (`JObject`). Slower than `System.Text.Json` (reflection-based, more allocations) but more forgiving and flexible.

```csharp
// System.Text.Json (built-in, fast, strict):
var opts = new JsonSerializerOptions
{
    PropertyNameCaseInsensitive = true,
    PropertyNamingPolicy        = JsonNamingPolicy.CamelCase,
    WriteIndented               = true
};
string json    = JsonSerializer.Serialize(order, opts);
Order? restored = JsonSerializer.Deserialize<Order>(json, opts);

// Source-generated (zero reflection, fastest possible):
// [JsonSerializable(typeof(Order))]
// public partial class OrderJsonContext : JsonSerializerContext {}
// var json = JsonSerializer.Serialize(order, OrderJsonContext.Default.Order);

// Newtonsoft (flexible, third-party):
string json2    = Newtonsoft.Json.JsonConvert.SerializeObject(order, Formatting.Indented);
Order? restored2 = Newtonsoft.Json.JsonConvert.DeserializeObject<Order>(json2);

// When to use Newtonsoft:
// - Migrating legacy projects with existing Newtonsoft converters
// - Complex polymorphism with $type discriminators
// - Need LINQ-to-JSON (JObject/JArray) for dynamic JSON manipulation
// - Reference loop handling (circular object graphs)
```
---

## Q41. Performance tuning techniques.

### 📌 Definition
Performance tuning in .NET involves systematically identifying bottlenecks (CPU, memory, I/O, lock contention) using profiling tools (dotnet-trace, PerfView, BenchmarkDotNet, Visual Studio Profiler, dotnet-counters) and then applying targeted optimization techniques. Key categories: **allocation reduction** (fewer GC pauses), **CPU optimization** (vectorization, inlining, branch reduction), **I/O optimization** (async, pooling, buffering), and **concurrency tuning** (reducing lock contention, optimal ThreadPool sizing).

```csharp
// 1. Allocation reduction: use ArrayPool<T>
byte[] buf = ArrayPool<byte>.Shared.Rent(4096); // Reuse; no Gen0 allocation
try { /* use buf */ }
finally { ArrayPool<byte>.Shared.Return(buf); }

// 2. Struct for hot path data (no heap allocation)
public readonly record struct OrderId(Guid Value) { ... }

// 3. Span<T> to avoid string/array allocations in parsing hot paths
ReadOnlySpan<char> slice = rawLine.AsSpan(10, 20); // No new string

// 4. Cache compiled delegates instead of repeated Reflection
private static readonly Func<Order, decimal> GetPrice =
    o => o.Price; // Compile once, call N times without boxing

// 5. Use StringComparison.Ordinal for string comparisons in hot paths
// (avoids culture-aware comparison overhead)
if (symbol.Equals("NIFTY50", StringComparison.Ordinal)) { ... }

// 6. Pre-size collections:
var orders = new List<Order>(capacity: 10_000); // Avoid repeated internal array resize

// 7. ValueTask<T> in hot async paths (avoids Task heap allocation when sync-fast)
public ValueTask<decimal> GetCachedPriceAsync(string sym)
{
    if (_cache.TryGetValue(sym, out decimal p)) return ValueTask.FromResult(p); // No heap
    return new ValueTask<decimal>(FetchFromDbAsync(sym)); // Heap only if cache miss
}
```

---

## Q42. Benchmarking in .NET.

### 📌 Definition
`BenchmarkDotNet` is the standard .NET micro-benchmarking library. It handles all the complexities of accurate benchmarking: JIT warm-up, multiple iterations, statistical analysis, GC interference elimination, and OS process priority management. Running code in a `Stopwatch` loop is unreliable for micro-benchmarks (JIT inlining, OS scheduling noise, GC interference). BenchmarkDotNet handles all of this and produces statistically-validated Mean, StdDev, Median, and allocation reports.

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]     // Reports allocations per run
[RankColumn]          // Adds rank by performance
[SimpleJob]
public class SerializationBench
{
    private readonly Order _order = new("ORD-001", "NIFTY50", 24500m, 10, DateTime.UtcNow);

    [Benchmark(Baseline = true)]
    public string NewtonsoftSerialize() =>
        Newtonsoft.Json.JsonConvert.SerializeObject(_order);

    [Benchmark]
    public string SystemTextJsonSerialize() =>
        System.Text.Json.JsonSerializer.Serialize(_order);
}

// Run: BenchmarkRunner.Run<SerializationBench>();
// Output:
// | Method                  | Mean     | Gen0   | Allocated |
// | NewtonsoftSerialize     | 1.2 µs   | 0.0300 | 400 B     |
// | SystemTextJsonSerialize | 0.45 µs  | 0.0100 | 128 B     |
```

---

## Q43. What is yield return?

### 📌 Definition
`yield return` is a C# keyword pair that transforms a method into an **iterator** — a method that returns an `IEnumerable<T>` or `IEnumerator<T>` but generates values **lazily, on demand**, one at a time, without building the entire sequence in memory. When the compiler sees `yield return`, it transforms the method into a state machine (similar to async/await) that remembers its position between `MoveNext()` calls. Each call to `MoveNext()` executes until the next `yield return`, produces one value, and suspends. `yield break` terminates the sequence early. Enables infinite sequences, lazy pipelines, and memory-efficient data streaming.

```csharp
// ❌ Eager: builds entire list in memory before returning
IEnumerable<decimal> GetPricesEager(int count)
{
    var prices = new List<decimal>(count);
    for (int i = 0; i < count; i++)
        prices.Add(24_000m + i * 0.5m);
    return prices; // All N prices in memory at once
}

// ✅ Lazy: produces one price at a time — consumer controls how many it needs
IEnumerable<decimal> GetPricesLazy(int count)
{
    for (int i = 0; i < count; i++)
        yield return 24_000m + i * 0.5m; // State machine saves position between calls
}

// Infinite sequence (impossible without yield):
IEnumerable<decimal> InfiniteTickFeed()
{
    var rng = new Random();
    while (true)
        yield return 24_000m + (decimal)(rng.NextDouble() * 1000);
}
// var first10 = InfiniteTickFeed().Take(10).ToList(); // Only generates 10 values
```

---

## Q44. Covariance and contravariance.

### 📌 Definition
**Covariance** (`out T`) on generic interfaces/delegates means you can use a `IEnumerable<Derived>` where `IEnumerable<Base>` is expected — assigning a more-specific type to a more-general reference. Safe only for read-only ("producing") scenarios where T is always an output. **Contravariance** (`in T`) means you can use an `Action<Base>` where `Action<Derived>` is expected — assigning a less-specific type to a more-specific reference. Safe only for write-only ("consuming") scenarios where T is always an input. These enable polymorphic assignment rules for generic types that C#'s type system cannot infer on its own.

```csharp
// Covariance (IEnumerable<out T>) — T is only output (produced):
IEnumerable<string> strings = new List<string> { "NIFTY50", "BANKNIFTY" };
IEnumerable<object> objects = strings; // ✅ Covariant: string IS-A object, safe to read

// Contravariance (Action<in T>) — T is only input (consumed):
Action<object> printAny = obj => Console.WriteLine(obj);
Action<string> printStr = printAny; // ✅ Contravariant: Action<object> can handle strings

// Custom covariant interface:
public interface IOrderProducer<out T> { T Next(); }  // out = covariant
public interface IOrderConsumer<in T> { void Accept(T item); } // in = contravariant
```

---

## Q45. What is pattern matching?

### 📌 Definition
**Pattern matching** in C# is a set of language features for testing values against shapes/types/values and extracting sub-components — in a concise, declarative style. Patterns include: type patterns (`is Order o`), constant patterns (`is 42`), relational patterns (`is > 100`), logical patterns (`is > 0 and < 100`), property patterns (`is { Status: "FILLED" }`), positional patterns (for records/tuples), list patterns (`is [_, _, ..]`), and discard (`_`). Available in `if`, `switch` expressions, `switch` statements, and `is` operator.

```csharp
// Type pattern + property pattern:
void Process(object ev)
{
    switch (ev)
    {
        case OrderPlaced   { Symbol: "NIFTY50" } p: Console.WriteLine($"NIFTY order: {p.Price}"); break;
        case OrderExecuted e when e.Price > 25000: Console.WriteLine($"High price exec: {e.Price}"); break;
        case OrderCancelled { Reason: var reason }: Console.WriteLine($"Cancelled: {reason}"); break;
        case null: throw new ArgumentNullException();
        default:   Console.WriteLine($"Unknown event: {ev.GetType().Name}"); break;
    }
}

// Switch expression (C# 8+):
string ClassifyOrder(Order o) => o switch
{
    { Price: > 50_000 }               => "Premium",
    { Quantity: > 100, Price: > 10000 } => "Bulk High-Value",
    { Status: "CANCELLED" }           => "Cancelled",
    _                                 => "Standard"
};

// Relational + logical patterns (C# 9+):
string ClassifyRisk(decimal exposure) => exposure switch
{
    < 0               => "Invalid",
    >= 0 and < 100_000 => "Low",
    >= 100_000 and < 500_000 => "Medium",
    >= 500_000         => "High"
};
```

---

## Q46. Exception handling best practices.

### 📌 Definition
Effective exception handling in .NET means: catching exceptions at the right level of abstraction (not too specific, not swallowing all), preserving the call stack (`throw` not `throw ex`), using custom exception types for domain errors, logging with full context before handling, and avoiding using exceptions for normal control flow. Exception handling adds overhead only when an exception is actually thrown — the `try` block itself has near-zero cost.

```csharp
// ❌ catch (Exception) and ignore — silent failure
try { PlaceOrder(); }
catch (Exception) { } // Swallows everything — bugs hidden

// ❌ throw ex — destroys original stack trace
catch (Exception ex) { throw ex; } // Stack trace points to this line, not origin

// ✅ Best practices:
try
{
    await PlaceOrderAsync(order, ct);
}
catch (OrderValidationException ex) // Specific domain exception — expected, handle gracefully
{
    _logger.LogWarning(ex, "Validation failed for order {OrderId}", order.Id);
    return BadRequest(ex.Message);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    _logger.LogInformation("Order {OrderId} cancelled by client", order.Id);
    return StatusCode(499); // Client closed request
}
catch (Exception ex) // Unexpected — log full context and rethrow
{
    _logger.LogError(ex, "Unexpected error placing order {OrderId}", order.Id);
    throw; // Preserves original stack trace
}
// Custom exception:
public sealed class OrderValidationException : Exception
{
    public string OrderId { get; }
    public OrderValidationException(string orderId, string message) : base(message)
        => OrderId = orderId;
}
```

---

## Q47. What is ConfigureAwait(false)?

### 📌 Definition
`ConfigureAwait(false)` is an instruction to the `await` machinery to **not capture the current `SynchronizationContext`** when scheduling the continuation after the awaited Task completes. Without it, the continuation is posted back to the captured context (UI thread in WPF, request thread in ASP.NET Classic). With `ConfigureAwait(false)`, the continuation runs on whatever `ThreadPool` thread happens to be available — avoiding marshaling overhead and preventing deadlocks. In ASP.NET Core, `SynchronizationContext.Current` is always `null`, so `ConfigureAwait(false)` is a no-op for the deadlock concern — but it is still recommended in library code for portability.

```csharp
// Library class — always ConfigureAwait(false):
public async Task<decimal> FetchPriceAsync(string symbol)
{
    var resp = await _http.GetAsync($"/price/{symbol}").ConfigureAwait(false);
    var body = await resp.Content.ReadAsStringAsync().ConfigureAwait(false);
    return decimal.Parse(body); // Runs on ThreadPool — not UI thread
}

// The deadlock scenario (ASP.NET Classic / WPF — NOT ASP.NET Core):
// public decimal GetPrice(string symbol)
// {
//     return FetchPriceAsync(symbol).Result;  // Blocks UI/request thread
//     // Without ConfigureAwait(false): continuation waits for UI thread → DEADLOCK
//     // With ConfigureAwait(false): continues on ThreadPool → no deadlock
// }
```

---

## Q48. How does HttpClient work internally?

### 📌 Definition
`HttpClient` in .NET is a high-level HTTP client that wraps a **pipeline of `HttpMessageHandler` objects**. The outermost handler is `HttpClient` itself; the innermost is `SocketsHttpHandler` (on .NET 5+), which manages a pool of TCP connections per host. When you call `GetAsync()`, the request travels through the handler pipeline (logging, auth, retry handlers) and eventually to `SocketsHttpHandler` which: reuses an existing pooled connection or creates a new one, writes the HTTP request, awaits the response asynchronously (no thread blocked during network I/O), and returns the response. Connection pooling avoids the TCP + TLS handshake cost on every request.

```csharp
// ❌ Common mistake: new HttpClient per request — exhausts sockets
for (int i = 0; i < 1000; i++)
{
    using var client = new HttpClient(); // 1000 socket connections, TIME_WAIT exhaustion
    var price = await client.GetStringAsync("https://api.nse.co.in/price/NIFTY50");
}

// ✅ Singleton HttpClient or IHttpClientFactory:
// Option 1: static singleton (simple)
private static readonly HttpClient _client = new();
var price = await _client.GetStringAsync("...");

// Option 2: IHttpClientFactory (preferred — manages lifetime, allows named/typed clients)
// Registration:
services.AddHttpClient<MarketDataClient>(c =>
{
    c.BaseAddress = new Uri("https://api.nse.co.in");
    c.Timeout     = TimeSpan.FromSeconds(5);
});
// Internally: factory creates SocketsHttpHandler instances with managed pool lifetime
// Rotates handlers every 2 minutes to pick up DNS changes

// Handler pipeline customization:
services.AddHttpClient<MarketDataClient>()
    .AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3, _ => TimeSpan.FromMilliseconds(200)))
    .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(5));
```

---

## Q49. What is connection pooling?

### 📌 Definition
**Connection pooling** is the practice of maintaining a cache of pre-established connections (TCP+auth) to a database or external service, reusing them across requests rather than creating and destroying connections per request. Creating a DB connection involves TCP handshake, TLS handshake, authentication, and session setup — hundreds of milliseconds. Connection pooling amortizes this cost: connections are borrowed from the pool, used, and returned. ADO.NET's `SqlConnection` implements pooling transparently — calling `.Open()` borrows from the physical pool; `.Close()` / `.Dispose()` returns it (not physically closed).

```csharp
// ADO.NET connection pooling (transparent, configured via connection string):
// "Server=...;Database=Orders;Pooling=true;Min Pool Size=5;Max Pool Size=100;Connect Timeout=30"

// ✅ Always dispose — returns connection to pool, not physically closed
public async Task<Order> GetOrderAsync(string id)
{
    await using var conn = new SqlConnection(_connStr); // Borrow from pool
    await conn.OpenAsync();
    // ... execute query ...
    return order;
} // Dispose here: connection returned to pool for reuse

// Pool exhaustion: if all 100 pool connections are in use and a new Open() is called,
// the thread waits (Connect Timeout seconds) then throws SqlException
// Monitor: SqlConnection.ConnectionString + performance counters

// EF Core: DbContext internally manages connection lifetime
// DbContext pooling (AddDbContextPool): reuse DbContext instances across requests
// Reduces DbContext constructor + connection open overhead for high-throughput APIs
services.AddDbContextPool<TradingDbContext>(opts =>
    opts.UseSqlServer(connStr), poolSize: 128);
```

---

## Q50. What is connection pooling? (HttpClient) / Write a custom middleware from scratch.

### 📌 Definition — Custom Middleware
A custom ASP.NET Core middleware is a class with a constructor taking `RequestDelegate next` and an `InvokeAsync(HttpContext context)` method. The `RequestDelegate` represents the next middleware in the pipeline. The middleware can read/modify the request, call `await next(context)` to pass to next, and read/modify the response after `next` returns.

### 🏭 Real-Time Use Case — Full Implementation

**Scenario: Full production-grade API key authentication + request logging middleware**

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

namespace TradingAPI.Middleware
{
    // ── Middleware 1: API Key Authentication ───────────────────────────────
    public sealed class ApiKeyAuthMiddleware
    {
        private const  string    ApiKeyHeader   = "X-API-Key";
        private static readonly  HashSet<string> _validKeys = new() { "PROD-KEY-001", "PROD-KEY-002" };
        private readonly RequestDelegate _next;

        public ApiKeyAuthMiddleware(RequestDelegate next) => _next = next;

        public async Task InvokeAsync(HttpContext ctx)
        {
            // Public paths bypass API key check
            if (ctx.Request.Path.StartsWithSegments("/health"))
            {
                await _next(ctx); return;
            }

            if (!ctx.Request.Headers.TryGetValue(ApiKeyHeader, out var key) || !_validKeys.Contains(key.ToString()))
            {
                ctx.Response.StatusCode = 401;
                ctx.Response.ContentType = "application/json";
                await ctx.Response.WriteAsync("{\"error\": \"Invalid or missing API key\"}");
                return; // Short-circuit — don't call next
            }

            await _next(ctx); // Valid key — proceed to next middleware
        }
    }

    // ── Middleware 2: Request/Response logging ─────────────────────────────
    public sealed class RequestResponseLoggingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<RequestResponseLoggingMiddleware> _logger;

        public RequestResponseLoggingMiddleware(RequestDelegate next, ILogger<RequestResponseLoggingMiddleware> logger)
            => (_next, _logger) = (next, logger);

        public async Task InvokeAsync(HttpContext ctx)
        {
            var correlationId = Guid.NewGuid().ToString("N")[..8];
            ctx.Items["CorrelationId"] = correlationId;

            _logger.LogInformation(
                "[{CorrelationId}] → {Method} {Path}{QueryString}",
                correlationId, ctx.Request.Method, ctx.Request.Path, ctx.Request.QueryString);

            var sw = Stopwatch.StartNew();
            await _next(ctx); // Execute rest of pipeline
            sw.Stop();

            _logger.LogInformation(
                "[{CorrelationId}] ← {StatusCode} in {ElapsedMs}ms",
                correlationId, ctx.Response.StatusCode, sw.ElapsedMilliseconds);

            if (sw.ElapsedMilliseconds > 500)
                _logger.LogWarning("[{CorrelationId}] Slow request! {ElapsedMs}ms", correlationId, sw.ElapsedMilliseconds);
        }
    }

    // ── Extension methods for clean registration ───────────────────────────
    public static class MiddlewareExtensions
    {
        public static IApplicationBuilder UseApiKeyAuth(this IApplicationBuilder app)
            => app.UseMiddleware<ApiKeyAuthMiddleware>();

        public static IApplicationBuilder UseRequestResponseLogging(this IApplicationBuilder app)
            => app.UseMiddleware<RequestResponseLoggingMiddleware>();
    }

    // Program.cs usage:
    // app.UseRequestResponseLogging(); // Outermost — logs ALL requests including failures
    // app.UseApiKeyAuth();              // Rejects unauthorized before routing
    // app.UseRouting();
    // app.MapControllers();
}
```

### ⚙️ Production-Grade Implementation

| Design choice | Benefit |
|---|---|
| Constructor: `RequestDelegate next` | Pipeline linkage — call `next(ctx)` to continue |
| `return` without calling `next` | Short-circuit — response sent, no further middleware runs |
| Read/write `ctx.Items[]` | Per-request data bag shared across middleware |
| Extension method on `IApplicationBuilder` | Clean `app.UseXxx()` registration in Program.cs |
| Inject services via `InvokeAsync` params | Unlike constructor, InvokeAsync params are resolved per-request (allows Scoped services) |

---

---

## Q51. Write a custom middleware from scratch. (Production Grade — Extended)

*Note: Core middleware pattern covered in Q50. This question extends to a full production-grade middleware stack with request body reading, response buffering, and global exception handling.*

### 📌 Definition
A complete production middleware stack for a trading API typically includes:
1. **Global exception handler** — catch all unhandled exceptions, return consistent problem details
2. **Request body logging** — log request JSON for audit (stream must be made seekable first)
3. **Correlation tracking** — assign/propagate unique request IDs for distributed tracing
4. **Response compression** — transparent gzip/brotli for large JSON responses

### 🏭 Full Implementation

```csharp
using System;
using System.IO;
using System.Text;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;

namespace TradingAPI.Middleware
{
    // ── Global Exception Handler (must be first — outermost wrapper) ───────
    public sealed class GlobalExceptionMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<GlobalExceptionMiddleware> _logger;

        public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
            => (_next, _logger) = (next, logger);

        public async Task InvokeAsync(HttpContext ctx)
        {
            try
            {
                await _next(ctx);
            }
            catch (Exception ex)
            {
                string correlationId = ctx.Items["CorrelationId"]?.ToString() ?? "N/A";
                _logger.LogError(ex, "[{CorrelationId}] Unhandled exception occurred", correlationId);
                await WriteErrorResponse(ctx, ex);
            }
        }

        private static async Task WriteErrorResponse(HttpContext ctx, Exception ex)
        {
            if (ctx.Response.HasStarted) return; // Can't modify response once started

            ctx.Response.Clear();
            ctx.Response.ContentType = "application/problem+json";
            ctx.Response.StatusCode = ex switch
            {
                ArgumentException     => 400,
                UnauthorizedAccessException => 401,
                InvalidOperationException   => 422,
                TimeoutException      => 504,
                _                    => 500
            };

            // RFC 7807 Problem Details format
            var problem = new ProblemDetails
            {
                Status   = ctx.Response.StatusCode,
                Title    = ex.GetType().Name,
                Detail   = ex.Message,
                Instance = ctx.Request.Path,
                Extensions = { ["correlationId"] = ctx.Items["CorrelationId"] }
            };

            await ctx.Response.WriteAsJsonAsync(problem);
        }
    }

    // ── Request Body Logging (for audit trail) ─────────────────────────────
    public sealed class RequestBodyLoggingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<RequestBodyLoggingMiddleware> _logger;

        public RequestBodyLoggingMiddleware(RequestDelegate next, ILogger<RequestBodyLoggingMiddleware> logger)
            => (_next, _logger) = (next, logger);

        public async Task InvokeAsync(HttpContext ctx)
        {
            // Only log request bodies for mutating HTTP methods
            if (ctx.Request.ContentLength > 0 &&
                (ctx.Request.Method == "POST" || ctx.Request.Method == "PUT"))
            {
                // EnableBuffering allows multiple reads of the request body
                ctx.Request.EnableBuffering();

                using var reader = new StreamReader(ctx.Request.Body,
                    encoding: Encoding.UTF8,
                    detectEncodingFromByteOrderMarks: false,
                    leaveOpen: true); // Don't close the stream!

                string body = await reader.ReadToEndAsync();
                ctx.Request.Body.Position = 0; // Rewind for downstream middleware

                string correlationId = ctx.Items["CorrelationId"]?.ToString() ?? "";
                _logger.LogInformation("[{CorrelationId}] Request body: {Body}", correlationId,
                    body.Length > 1000 ? body[..1000] + "...[truncated]" : body);
            }

            await _next(ctx);
        }
    }

    // ── Extension methods for Program.cs ───────────────────────────────────
    public static class TradingApiMiddlewareExtensions
    {
        public static IApplicationBuilder UseTradingApiMiddleware(this WebApplication app)
        {
            app.UseMiddleware<GlobalExceptionMiddleware>();      // 1. Exception handler — outermost
            app.UseMiddleware<RequestResponseLoggingMiddleware>(); // 2. Timing + correlation ID
            app.UseMiddleware<ApiKeyAuthMiddleware>();           // 3. Authentication
            app.UseMiddleware<RequestBodyLoggingMiddleware>();   // 4. Body audit logging
            app.UseResponseCompression();                       // 5. Compress responses
            return app;
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Middleware | Must be first? | Why |
|---|---|---|
| `GlobalExceptionMiddleware` | ✅ Yes | Must wrap all others to catch their exceptions |
| `RequestResponseLoggingMiddleware` | Near first | Needs to time the full pipeline |
| `ApiKeyAuthMiddleware` | Before routing | Reject unauthorized early |
| `UseRouting` | Before `UseAuthorization` | `UseAuthorization` needs route data |
| `UseOutputCaching` / `UseResponseCaching` | Late | Cache final responses only |

---
