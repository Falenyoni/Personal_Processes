# .NET Design Patterns — Interview Prep Guide

Each pattern below has four parts:
- **What** — one-line definition
- **When** — typical use case
- **Q** — common interview question
- **Impress** — the answer that shows depth

---

## CREATIONAL PATTERNS

### Factory Method
- **What:** Defines an interface for creating an object, but lets subclasses decide which class to instantiate.
- **When:** You don't know exact types until runtime; want to centralize creation logic.
- **Q:** *"Difference between Factory Method and Abstract Factory?"*
- **Impress:** Real-world .NET examples: `HttpClientFactory`, `ILoggerFactory`, `IServiceProvider.GetService<T>()`. Factory Method creates **one** product; Abstract Factory creates a **family** of related products.

### Abstract Factory
- **What:** Creates families of related objects without specifying concrete classes.
- **When:** Cross-platform UIs, multi-database providers, theme systems.
- **Q:** *"When would you choose Abstract Factory over Factory Method?"*
- **Impress:** When products **must work together** (e.g., a `WindowsButton` + `WindowsScrollbar` belong as a set). EF Core's `IDbContextFactory<T>` plus its providers (SQL Server, PostgreSQL) is conceptually similar.

### Builder
- **What:** Constructs complex objects step by step.
- **When:** Many optional parameters, immutable types, fluent setup, validation between steps.
- **Q:** *"Why use Builder when C# has named/optional arguments?"*
- **Impress:** Builder allows step-by-step validation, fluent APIs, and is essential for immutable types. Real examples: `WebApplicationBuilder`, `HostBuilder`, `StringBuilder`, `HttpClientBuilder`.

### Prototype
- **What:** Creates new objects by cloning an existing one.
- **When:** Object creation is expensive (DB calls, heavy computation) and you need many similar copies.
- **Q:** *"Deep clone vs shallow clone? How do you do it in C#?"*
- **Impress:** `ICloneable` is shallow and ambiguous (avoid it). Modern C#: use `record` with `with` expressions for immutable copies. For deep clones: serialization, manual copy, or libraries like AutoMapper.

### Singleton
- **What:** Ensures one instance of a class with global access.
- **When:** Logging, caching, configuration, connection pools.
- **Q:** *"How do you make a thread-safe Singleton? Why is it considered an anti-pattern?"*
- **Impress:** Use `Lazy<T>` for thread-safe lazy init. In modern .NET, prefer `services.AddSingleton<T>()` from DI — it's testable. Classic Singleton hides dependencies and makes mocking impossible (violates DIP).

```csharp
public sealed class Logger
{
    private static readonly Lazy<Logger> _instance = new(() => new Logger());
    public static Logger Instance => _instance.Value;
    private Logger() { }
}
```

---

## STRUCTURAL PATTERNS

### Adapter
- **What:** Makes incompatible interfaces work together by wrapping one.
- **When:** Integrating legacy code or third-party libraries with different contracts.
- **Q:** *"Adapter vs Facade?"*
- **Impress:** Adapter **converts** an interface to what a client expects; Facade **simplifies** a subsystem. `StreamReader` adapts a raw `Stream` into a text-reading API.

### Facade
- **What:** Provides a simplified interface to a complex subsystem.
- **When:** Hiding the complexity of multiple services behind one entry point.
- **Q:** *"Real-world Facade in .NET?"*
- **Impress:** `HttpClient` is a facade over sockets, TLS, DNS, headers. `WebApplication` is a facade over Kestrel + middleware + DI. EF Core's `DbContext` is a facade over change tracking, query translation, and connection management.

### Bridge
- **What:** Decouples abstraction from implementation so they can vary independently.
- **When:** Avoiding subclass explosion (e.g., shapes × colors × renderers).
- **Q:** *"Bridge vs Adapter?"*
- **Impress:** Bridge is **designed upfront** to separate two dimensions of variation. Adapter is **retrofitted** to make existing things work. `ILogger<T>` working over different sinks (Console, File, Seq) is a bridge.

### Composite
- **What:** Treats individual objects and compositions uniformly.
- **When:** Tree structures — UI components, file systems, org charts, expression trees.
- **Q:** *"Give a .NET example."*
- **Impress:** Blazor component trees, Razor view hierarchies, `XElement` in LINQ to XML, `IConfigurationSection` (each section is itself an `IConfiguration`).

### Proxy
- **What:** Controls access to another object — for caching, lazy loading, security, logging.
- **When:** Lazy loading in EF Core, remote calls, access control, logging wrappers.
- **Q:** *"Difference between Proxy and Decorator?"*
- **Impress:** Proxy **controls access** (same interface, gatekeeping responsibility); Decorator **adds behavior**. EF Core uses dynamic proxies for lazy navigation properties. Castle DynamicProxy is the standard library for this.

---

## BEHAVIORAL PATTERNS

### Command
- **What:** Encapsulates a request as an object.
- **When:** Undo/redo, queueing, scheduled execution, audit logging, CQRS write side.
- **Q:** *"How does Command relate to CQRS?"*
- **Impress:** Each MediatR `IRequest` is literally a Command. Commands enable undo stacks, replay, and decoupling sender from receiver.

### Observer
- **What:** One-to-many dependency — when subject changes, observers are notified.
- **When:** Events, pub/sub, reactive UI.
- **Q:** *"Observer in C# vs events vs IObservable?"*
- **Impress:** C# `event` is built-in Observer (sync). `IObservable<T>` / `IObserver<T>` (Rx.NET) is for async streams with operators. Always mention the **memory leak risk** from forgetting to unsubscribe.

### Strategy
- **What:** Defines a family of interchangeable algorithms.
- **When:** Multiple ways to do the same thing — sorting, pricing, payment methods, validators.
- **Q:** *"Strategy vs State pattern?"*
- **Impress:** Strategy is **chosen by the client**; State **changes itself** based on context. Inject strategies via DI as `IEnumerable<IPaymentStrategy>` and select by key.

### State
- **What:** Object alters behavior when its internal state changes.
- **When:** Workflows, order status, game states, document lifecycle.
- **Q:** *"Why not just use a switch on an enum?"*
- **Impress:** State pattern avoids fat switch statements and follows Open/Closed. Each state is its own class. For real workflows, use the **Stateless** library or **MassTransit state machines**.

### Template Method
- **What:** Defines the skeleton of an algorithm; subclasses override specific steps.
- **When:** Common workflow with varying steps.
- **Q:** *"Template Method vs Strategy?"*
- **Impress:** Template Method = inheritance (compile-time, fixed). Strategy = composition (runtime, swappable). `BackgroundService.ExecuteAsync` is template method.

### Visitor
- **What:** Adds operations to objects without modifying them.
- **When:** Compilers, AST traversal, complex object hierarchies that need many operations.
- **Q:** *"Why Visitor instead of adding methods to the classes?"*
- **Impress:** Visitor follows Open/Closed — you can add operations without changing the classes. **Roslyn's `CSharpSyntaxVisitor<T>`** and `ExpressionVisitor` (LINQ expression trees) are textbook examples.

### Chain of Responsibility
- **What:** Passes a request along a chain of handlers until one handles it.
- **When:** Middleware pipelines, validation chains, request processing.
- **Q:** *"Real example in ASP.NET Core?"*
- **Impress:** **ASP.NET Core middleware pipeline IS this pattern.** Each middleware decides to handle, pass on, or short-circuit. Also: MediatR pipeline behaviors, FluentValidation rule chains.

### Memento
- **What:** Captures and restores object state without exposing internals.
- **When:** Undo, snapshots, transactional rollback.
- **Q:** *"How does Memento differ from serialization?"*
- **Impress:** Memento preserves **encapsulation** — only the originator can read/restore. Serialization exposes everything. EF Core's change tracker uses a similar idea (original values).

---

## ARCHITECTURE & ADVANCED

### CQRS (Command Query Responsibility Segregation)
- **What:** Separate models for reads (queries) and writes (commands).
- **When:** Read/write workloads differ heavily, complex domain logic, different scaling needs.
- **Q:** *"When should you NOT use CQRS?"*
- **Impress:** Skip it for simple CRUD — it adds complexity. CQRS does **not** require separate databases (that's CQRS + Event Sourcing). Often paired with MediatR. The killer feature: read models can be denormalized for queries.

### Saga Orchestration
- **What:** Coordinates long-running, distributed transactions across services with compensating actions.
- **When:** Microservices where 2PC won't work — order flow, booking, payment processing.
- **Q:** *"Orchestration vs Choreography?"*
- **Impress:** Orchestration = central coordinator (easier to debug, single point of logic). Choreography = event-driven (more decoupled, harder to trace). Use **MassTransit** or **NServiceBus**. Always discuss **idempotency** and **compensating transactions**.

### Outbox
- **What:** Atomically write business data and outgoing events to the same DB; a worker reads the outbox table and publishes them.
- **When:** You need reliable event publishing alongside DB writes.
- **Q:** *"Why not publish directly to the message bus?"*
- **Impress:** This is the **dual-write problem**. If you write to DB then publish, the publish can fail and you lose the event. Outbox makes both writes part of the **same DB transaction**, then a relay publishes. Pair with at-least-once delivery + idempotent consumers.

### Circuit Breaker
- **What:** Stops calling a failing service so it can recover.
- **When:** External APIs, microservice-to-microservice calls.
- **Q:** *"What are the states of a circuit breaker?"*
- **Impress:** **Closed** (calls flow), **Open** (calls fail fast), **Half-Open** (probing recovery). Use **Polly** or the newer `Microsoft.Extensions.Http.Resilience`. Always combine with **retry + timeout + bulkhead** for full resilience.

### Repository
- **What:** Abstracts data access behind a collection-like interface.
- **When:** Decouple domain from persistence; encapsulate queries; testability.
- **Q:** *"Is Repository over EF Core an anti-pattern?"*
- **Impress:** **Controversial.** `DbContext` already IS Unit of Work + Repository. Wrapping it again often just hides EF's power (no `Include`, no projection). Use Repository when: you need to swap data sources, enforce specific query patterns, or want a clean DDD aggregate boundary. For simple apps — query `DbContext` directly.

### Service Collection Extension
- **What:** Extension methods on `IServiceCollection` to group related DI registrations.
- **When:** Library or feature module setup — `services.AddMyFeature()`.
- **Q:** *"Why use them?"*
- **Impress:** Keeps `Program.cs` clean, encapsulates a feature's dependencies, follows the standard convention used by every modern .NET library (`AddControllers`, `AddDbContext`, `AddAuthentication`).

```csharp
public static IServiceCollection AddOrdering(this IServiceCollection services)
{
    services.AddScoped<IOrderService, OrderService>();
    services.AddScoped<IOrderRepository, OrderRepository>();
    services.AddValidatorsFromAssemblyContaining<CreateOrderValidator>();
    return services;
}
```

### REPR (Request-Endpoint-Response)
- **What:** Each endpoint is its own class with its own request and response — an alternative to fat controllers.
- **When:** APIs with many endpoints; vertical slice architecture.
- **Q:** *"Why REPR over MVC controllers?"*
- **Impress:** Each endpoint is self-contained — easier to test, locate, and modify. No more 1000-line controllers. Pairs naturally with **vertical slice architecture**. The **FastEndpoints** library implements this in .NET. Minimal APIs lean this direction too.

### Options (IOptions)
- **What:** Strongly-typed access to configuration values.
- **When:** Reading `appsettings.json`, env vars, etc. into POCOs.
- **Q:** *"Difference between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`?"*
- **Impress:**
  - `IOptions<T>` — **singleton**, value loaded once at startup. Use for config that won't change.
  - `IOptionsSnapshot<T>` — **scoped**, re-evaluated per request. Use in web apps when config can be reloaded.
  - `IOptionsMonitor<T>` — **singleton with `OnChange` callback**. Use in singletons / background services that need live updates.

---

## GENERAL INTERVIEW TIPS

1. **Tie every pattern to a real .NET example** — interviewers love this.
2. **Know the SOLID principle** each pattern serves (mostly Open/Closed and DIP).
3. **Discuss trade-offs** — every pattern adds indirection and complexity.
4. **Mention modern .NET alternatives.** Many GoF patterns are baked into DI now.
5. **Read source code** — Roslyn, EF Core, ASP.NET Core. Patterns are everywhere.

## Likely "Combine the Patterns" Design Questions

| Scenario | Patterns to mention |
|---|---|
| Design a notification system | Strategy + Observer + Factory + Chain of Responsibility |
| Design a payment processor | Strategy + Circuit Breaker + Retry + Chain of Responsibility |
| Design an order system | CQRS + Saga + Outbox + Repository + Domain Events |
| Design a logging library | Singleton (via DI) + Decorator + Chain of Responsibility |
| Design a caching layer | Proxy + Decorator + Strategy (eviction policies) |
| Design a workflow engine | State + Command + Memento (for rollback) |

---

## Quick Pre-Interview Checklist

- [ ] Can you write a thread-safe Singleton from memory?
- [ ] Can you explain Strategy vs State without confusing them?
- [ ] Do you know all three `IOptions` variants?
- [ ] Can you describe Outbox + dual-write problem?
- [ ] Can you list circuit breaker states?
- [ ] Can you defend or critique Repository over EF Core?
- [ ] Can you point to ASP.NET Core middleware as Chain of Responsibility?
