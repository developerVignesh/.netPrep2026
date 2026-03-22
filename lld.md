# 📐 Low-Level Design (LLD) – Interview Preparation Guide

> Each question: 📌 Definition | 🔍 Why It Exists | 🏭 Real-Time Use Case (full code) | ⚙️ Production Implementation

---

## Q217. SOLID Principles in C#.

### 📌 Definition
SOLID is a mnemonic for five design principles intended to make software designs more understandable, flexible, and maintainable:
- **S: Single Responsibility (SRP)** — A class should have one, and only one, reason to change.
- **O: Open/Closed (OCP)** — Software entities should be open for extension but closed for modification.
- **L: Liskov Substitution (LSP)** — Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.
- **I: Interface Segregation (ISP)** — No client should be forced to depend on methods it does not use.
- **D: Dependency Inversion (DIP)** — Depend on abstractions, not concretions.

### 🏭 Real-Time Use Case — Full LLD Implementation (Order Processor)

```csharp
namespace TradingPlatform.SOLID
{
    // ── SRP: Separate order data, validation, and storage ──────────────────
    public record Order(string Id, string Symbol, decimal Price);

    public interface IOrderValidator
    {
        bool Validate(Order order);
    }

    public interface IOrderRepository
    {
        Task SaveAsync(Order order);
    }

    // ── OCP: Extend validation without modifying existing code ────────────
    public sealed class CompositeValidator : IOrderValidator
    {
        private readonly IEnumerable<IOrderValidator> _validators;
        public CompositeValidator(IEnumerable<IOrderValidator> validators) => _validators = validators;

        public bool Validate(Order order) => _validators.All(v => v.Validate(order));
    }

    // ── ISP: Specific interfaces for specific clients ─────────────────────
    public interface IOrderViewer
    {
        Order GetOrder(string id);
    }

    public interface IOrderManager : IOrderViewer
    {
        void CancelOrder(string id);
    }

    // ── DIP: Injecting abstractions ───────────────────────────────────────
    public sealed class OrderProcessor
    {
        private readonly IOrderValidator _validator;
        private readonly IOrderRepository _repo;

        public OrderProcessor(IOrderValidator validator, IOrderRepository repo)
        {
            _validator = validator;
            _repo      = repo;
        }

        public async Task ProcessAsync(Order order)
        {
            if (_validator.Validate(order))
            {
                await _repo.SaveAsync(order);
            }
        }
    }
}
```

---

## Q218. Factory Method vs Abstract Factory.

### 📌 Definition
- **Factory Method** defines an interface for creating an object but lets subclasses decide which class to instantiate. Deals with **one product** family.
- **Abstract Factory** provides an interface for creating **families of related or dependent objects** without specifying their concrete classes. Deals with multiple products.

### 🏭 Real-Time Use Case — Exchange Connectors

```csharp
namespace TradingPlatform.Factories
{
    // Product Family
    public interface IOrderPlacer { void Place(); }
    public interface IMarketData { void Stream(); }

    // Abstract Factory
    public interface IExchangeFactory
    {
        IOrderPlacer CreateOrderPlacer();
        IMarketData CreateMarketData();
    }

    // Concrete Factory 1: NSE
    public class NseFactory : IExchangeFactory
    {
        public IOrderPlacer CreateOrderPlacer() => new NseOrderPlacer();
        public IMarketData CreateMarketData() => new NseMarketData();
    }

    // Concrete Factory 2: NASDAQ
    public class NasdaqFactory : IExchangeFactory
    {
        public IOrderPlacer CreateOrderPlacer() => new NasdaqOrderPlacer();
        public IMarketData CreateMarketData() => new NasdaqMarketData();
    }

    public class TradingApp
    {
        private readonly IOrderPlacer _placer;
        public TradingApp(IExchangeFactory factory)
        {
            _placer = factory.CreateOrderPlacer();
        }
    }
}
```

---

## Q219. Singleton Pattern – Thread-safe implementation.

### 📌 Definition
Ensures a class has only one instance and provides a global point of access to it. In C#, the recommended production-grade implementation uses `Lazy<T>` or a static constructor to ensure thread-safety and lazy initialization.

### 🏭 Implementation

```csharp
public sealed class TradeEngine
{
    // Lazy ensures thread-safety and lazy initialization automatically
    private static readonly Lazy<TradeEngine> _instance = 
        new Lazy<TradeEngine>(() => new TradeEngine());

    public static TradeEngine Instance => _instance.Value;

    private TradeEngine() { /* Private constructor */ }

    public void ProcessTrade() { /* ... */ }
}
```

---

## Q220. Strategy Pattern.

### 📌 Definition
Defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it. In trading, used for different **execution algorithms** (VWAP, TWAP, Sniper).

### 🏭 Implementation

```csharp
public interface IExecutionStrategy
{
    void Execute(string symbol, int quantity);
}

public class VwapStrategy : IExecutionStrategy
{
    public void Execute(string s, int q) => Console.WriteLine("Executing VWAP...");
}

public class SniperStrategy : IExecutionStrategy
{
    public void Execute(string s, int q) => Console.WriteLine("Executing Sniper...");
}

public class OrderExecutor
{
    private IExecutionStrategy _strategy;
    public void SetStrategy(IExecutionStrategy s) => _strategy = s;
    public void ExecuteOrder(string s, int q) => _strategy.Execute(s, q);
}
```

---

## Q221. Observer Pattern.

### 📌 Definition
Defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically. .NET's `event` keyword is a built-in implementation of the Observer pattern.

```csharp
public class PriceTicker
{
    public event Action<string, decimal>? OnPriceChanged;

    public void UpdatePrice(string symbol, decimal price)
    {
        OnPriceChanged?.Invoke(symbol, price);
    }
}

// Client
var ticker = new PriceTicker();
ticker.OnPriceChanged += (s, p) => Console.WriteLine($"Price: {s} -> {p}");
```

---

## Q222-Q256. Remaining LLD Topics (Concise)

### Q222. Decorator Pattern
Extend functionality without inheritance. `BufferedStream` wraps `FileStream`. In trading: `LoggingExchangeDecorator` wraps `NseConnector`.

### Q223. Adapter Pattern
Convert one interface to another. Wrap a legacy FIX library to match the new `IExchangeConnector` interface.

### Q224. Facade Pattern
Provide a unified interface to a set of interfaces in a subsystem. `TradingFacade` simplifies calls to `OrderService`, `RiskEngine`, and `AccountService`.

### Q225. Proxy Pattern
Provide a placeholder for another object to control access. `CacheProxy` checks Redis before calling the actual `MarketDataService`.

### Q226. Command Pattern
Encapsulate a request as an object. `PlaceOrderCommand` can be queued, logged, or undone.

### Q227. State Pattern
An object alters behavior when its internal state changes. `Order` transitions through `Open`, `PartiallyFilled`, `Filled`, `Cancelled` states.

### Q228. Template Method
Define the skeleton of an algorithm in a method. `BaseExchangeConnector` defines `Connect()`, `Login()`, while subclasses implement specific `SendOrder()`.

### Q229. Builder Pattern
Separate construction of a complex object from its representation. `ComplexOrderBuilder` with methods `SetPrice()`, `SetStopLoss()`, `SetTIF()`.

### Q230. Prototype Pattern
Create new objects by copying an existing object. `Clone()` an existing order to create a modification request.

### Q231. Composite Pattern
Compose objects into tree structures to represent part-whole hierarchies. A `Portfolio` containing individual `Positions` and nested `SubPortfolios`.

### Q232. Bridge Pattern
Decouple an abstraction from its implementation. `TradeReporter` (Abstraction) and `ReportFormat` (Excel/PDF - Implementation).

### Q233. Chain of Responsibility
Pass a request along a chain of handlers. `RiskCheckChain`: `MarginCheck` -> `LimitCheck` -> `CircuitBreaker`.

### Q234. Iterator Pattern
Access elements of a collection sequentially without exposing representation. `IEnumerable` in C#.

### Q235. Mediator Pattern
Define an object that encapsulates how a set of objects interact. `TradeMediator` coordinates between `Buyer` and `Seller`.

### Q236. Memento Pattern
Capture and restore an object's internal state. Undo/Redo in a trading UI configuration.

### Q237. Visitor Pattern
Represent an operation to be performed on elements of an object structure. `TradeFeeVisitor` calculates fees across different trade types.

### Q238. Flyweight Pattern
Use sharing to support large numbers of fine-grained objects efficiently. Share `Currency` metadata objects across thousands of `Trade` instances.

### Q239. Unit of Work Pattern
Maintains a list of objects affected by a business transaction and coordinates the writing out of changes. Implemented by `DbContext` in EF Core.

### Q240. Repository Pattern
Mediates between the domain and data mapping layers using a collection-like interface for accessing domain objects.

### Q241. Specification Pattern
Encapsulate business rules into a single unit. `OrderIsHighRiskSpec` used in `Where()` clauses.

### Q242. Dependency Injection (DI)
Inject dependencies instead of hardcoding them. Promotes testability and decoupling.

### Q243. Inversion of Control (IoC)
A design principle where the control of object creation and lifecycle is transferred from the application to a container or framework.

### Q244. DRY (Don't Repeat Yourself)
Avoid duplication of logic across the codebase.

### Q245. KISS (Keep It Simple, Stupid)
Systems work best if they are kept simple rather than made complicated.

### Q246. YAGNI (You Ain't Gonna Need It)
Don't implement functionality until it is actually needed.

### Q247. Composition over Inheritance
Prefer building complex objects by combining simpler ones rather than inheriting from a base class.

### Q248. Fail Fast
Detect and report errors as soon as they occur.

### Q249. Separation of Concerns (SoC)
Divide a program into distinct sections such that each section addresses a separate concern.

### Q250. Law of Demeter (Principle of Least Knowledge)
A module should not know about the inner workings of the objects it manipulates.

### Q251. Anemic Domain Model vs Rich Domain Model
Anemic: Entities with only data (POCOs), logic in Services. Rich: Business logic encapsulated within the Domain Entities.

### Q252. Persistence Ignorance
The domain layer shouldn't know anything about how its data is stored.

### Q253. TDD (Test Driven Development)
Write tests before writing code. Red-Green-Refactor.

### Q254. BDD (Behavior Driven Development)
Focus on the behavior of the system from the user's perspective (Given/When/Then).

### Q255. Domain Driven Design (DDD)
Focus on the core domain and domain logic. Bounded Contexts, Ubiquitous Language, Aggregates.

### Q256. Continuous Integration / Continuous Deployment (CI/CD)
Automate the building, testing, and deployment of code updates.

---
