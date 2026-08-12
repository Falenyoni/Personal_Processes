# .NET Design Patterns — Model Interview Answers

These are written in first person, the way a strong candidate would actually speak in an interview. Read them out loud a few times — the goal isn't to memorize word-for-word, it's to internalize the *structure*: direct answer → technical detail → concrete .NET example → trade-off or nuance.

---

## CREATIONAL PATTERNS

### Q: "What's the difference between Factory Method and Abstract Factory?"

The simplest way to put it: Factory Method creates **one** product, Abstract Factory creates a **family** of related products that need to work together.

With Factory Method, you have a single creation method — usually defined in a base class or interface — and subclasses decide which concrete type to instantiate. A good example is `ILoggerFactory.CreateLogger<T>()` — one method, one product.

Abstract Factory is what you reach for when you need to create multiple related objects that have to be consistent with each other. The classic example is a cross-platform UI toolkit where a Windows factory produces a `WindowsButton` and a `WindowsScrollbar`, and a Mac factory produces matching Mac versions. You'd never want to mix them.

In modern .NET, you don't write these patterns from scratch very often — DI containers handle most of it. But you see Factory Method everywhere: `HttpClientFactory`, `IDbContextFactory<T>`, `IServiceProvider.GetService<T>()`. Abstract Factory shows up in EF Core's database providers, where the SQL Server provider knows how to create matching connections, command builders, and query translators that all belong together.

---

### Q: "When would you choose Abstract Factory over Factory Method?"

When the products you're creating **must work together as a set**. If a `WindowsButton` only makes sense alongside a `WindowsScrollbar` and a `WindowsMenu`, you need Abstract Factory to enforce that families don't get mixed.

If you just need to create one type of thing — like a logger or an HTTP client — Factory Method is enough. Abstract Factory adds complexity, so I'd only pull it out when the "family consistency" requirement is real. Otherwise it's over-engineering.

---

### Q: "Why use Builder when C# has named and optional arguments?"

A few reasons. Named arguments are great for simple cases, but Builder shines when:

First, you have **immutable types**. If your class has 15 properties and is immutable, you can't just keep mutating it. Builder lets you stage the construction, validate, and then produce the final immutable object.

Second, you need **validation between steps** — for example, you can't set the shipping address before you've set the country. A builder can enforce ordering and dependencies that named arguments can't.

Third, **fluent APIs**. The whole modern .NET configuration story is Builder: `WebApplication.CreateBuilder(args).Services.AddControllers()...Build()`. That's not just style — it gives you discoverability through IntelliSense and lets each step return a more specific builder type to limit what you can do next.

`StringBuilder` is the textbook example, but `HostBuilder`, `WebApplicationBuilder`, and EF Core's `DbContextOptionsBuilder` are the ones I see daily.

---

### Q: "Deep clone vs shallow clone — how do you do it in C#?"

Shallow clone copies the top-level object but keeps references to the same nested objects. Deep clone copies everything recursively — a full independent copy.

In C#, I'd avoid `ICloneable` from the BCL because it's deliberately ambiguous about whether it's deep or shallow. For shallow copies on a class, `MemberwiseClone()` works. For records, `with` expressions give you a clean shallow copy with modifications: `var updated = original with { Name = "New" }`.

For deep cloning, there's no built-in clean way. Options are:
- Manually copy each field (verbose but explicit)
- Serialize and deserialize — `System.Text.Json` round-trip works but it's slow and loses behavior
- Use a library like Mapster or AutoMapper

The pattern Prototype is genuinely useful when object construction is expensive — say you've loaded a complex object from a database or done heavy computation, and you need many similar copies. Cloning is faster than rebuilding from scratch.

---

### Q: "How do you make a thread-safe Singleton, and why is it considered an anti-pattern?"

The cleanest thread-safe implementation is `Lazy<T>`:

```csharp
public sealed class Logger
{
    private static readonly Lazy<Logger> _instance = new(() => new Logger());
    public static Logger Instance => _instance.Value;
    private Logger() { }
}
```

`Lazy<T>` handles the locking for you and is lazily initialized — you don't pay the cost until you use it. Marking the class `sealed` and the constructor `private` prevents subclassing and external instantiation.

As for why it's considered an anti-pattern — it's not the *concept* of "one instance" that's bad, it's the **static global access** part. When code does `Logger.Instance.Log(...)`, that dependency is invisible. You can't mock it, you can't swap it for tests, and the class is now coupled to a concrete implementation. It violates the Dependency Inversion Principle.

In modern .NET, the right answer is `services.AddSingleton<ILogger, Logger>()` and inject it via constructor. You still get exactly one instance per container, but now the dependency is explicit, mockable, and follows DI conventions.

---

## STRUCTURAL PATTERNS

### Q: "Adapter vs Facade — what's the difference?"

Adapter **converts** an interface to one the client expects. Facade **simplifies** a complex subsystem behind a friendlier interface.

The intent is different. Adapter is about compatibility — you have an existing class with the wrong shape, and you wrap it so it fits where you need it. Facade is about complexity reduction — there are five subsystems, and you don't want callers to deal with all of them, so you provide one easy entry point.

A good Adapter example is `StreamReader` — it wraps a raw byte `Stream` and adapts it into a text-reading API. Facade in .NET would be `HttpClient`, which hides sockets, TLS, DNS resolution, header management, and connection pooling behind a few clean methods.

There's overlap in practice — sometimes the line gets blurry. But the question to ask is: am I converting an interface (Adapter), or am I simplifying a subsystem (Facade)?

---

### Q: "Real-world Facade in .NET?"

`HttpClient` is the obvious one — it's a facade over the entire HTTP stack: socket handling, TLS, DNS, connection pooling, headers, content negotiation. You just call `GetAsync` and you're done.

`WebApplication` in ASP.NET Core is another — it's a facade over Kestrel, the middleware pipeline, dependency injection, configuration, and logging. Without it, you'd have to wire up all of those manually.

EF Core's `DbContext` is a facade too — it hides change tracking, query translation, connection management, and the unit of work pattern behind a fairly simple API.

The thing I'd point out is that good Facades don't *prevent* you from accessing the underlying complexity — they just make the common case easy. `HttpClient` lets you drop down to a `HttpRequestMessage` when you need full control.

---

### Q: "Bridge vs Adapter?"

Bridge is **designed upfront** to separate two dimensions of variation so they can change independently. Adapter is **retrofitted** to make incompatible things work together after the fact.

The classic Bridge problem: you have shapes (Circle, Square) and renderers (Vector, Raster). Without Bridge, you end up with `VectorCircle`, `RasterCircle`, `VectorSquare`, `RasterSquare` — a class explosion. Bridge separates the two hierarchies: you have a Shape that holds a reference to a Renderer, and the two evolve independently.

`ILogger<T>` working over different sinks — Console, File, Seq, Application Insights — is conceptually a bridge. The logger abstraction is one dimension; the sink implementation is another, and they vary independently.

So the rule of thumb: Bridge is a planning decision; Adapter is a fix.

---

### Q: "Composite — give a .NET example."

Blazor's component tree is a great one — every component can contain other components, and they're all treated uniformly. You don't write different code for "leaf" components versus "container" components.

Razor view hierarchies work the same way. `XElement` in LINQ to XML is another — every element is itself a tree node that can contain more elements.

The one I think is most underrated is `IConfigurationSection` — every section *is* itself an `IConfiguration`. You can navigate `config.GetSection("Logging").GetSection("LogLevel")` and treat each step the same way. That's Composite.

The pattern shines whenever you have a tree where individual nodes and groups of nodes need to support the same operations.

---

### Q: "Difference between Proxy and Decorator?"

Proxy **controls access** to an object — same interface, but the proxy is a gatekeeper. Decorator **adds behavior** to an object — same interface, but the decorator extends what it does.

The intent is the differentiator. A caching proxy decides whether to call the real object based on a cache hit. A logging decorator adds logging *around* the call but always delegates. Mechanically, they look almost identical — both implement the same interface and wrap a target.

EF Core uses dynamic Proxy for lazy loading — when you access a navigation property, the proxy intercepts and loads the data on demand. That's classic access control.

A Decorator would be wrapping `IRepository` with a `LoggingRepository` that logs every call before delegating, or a `RetryRepository` that retries on failure.

In .NET, both are typically implemented with **Castle DynamicProxy** when you need them at runtime — it's the library Moq and many AOP frameworks use under the hood.

---

## BEHAVIORAL PATTERNS

### Q: "How does Command relate to CQRS?"

Command is one of the foundational patterns CQRS is built on. In CQRS, every write is expressed as a Command — an object that encapsulates an intent like `CreateOrder` or `CancelSubscription`.

If you're using MediatR, every `IRequest` is literally a Command in the pattern sense. The `CreateOrderCommand` class captures the request data, and a separate `CreateOrderCommandHandler` executes it. That separation between the command and its handler is what enables the things people love about CQRS — easy logging, validation, retries, and cross-cutting concerns through pipeline behaviors.

The deeper benefit of Command-as-an-object is that it decouples the sender from the receiver, makes operations queueable, schedulable, and replayable, and gives you a natural audit log. If you're doing event sourcing, your commands often translate directly into domain events.

---

### Q: "Observer in C# — events vs IObservable?"

C# gives you two built-in implementations of Observer.

`event` is the simple one — synchronous, one-shot notifications. It's perfect for UI events, lifecycle hooks, and most everyday cases. The catch is that subscribers are held by strong references, so if you forget to unsubscribe, you leak memory. That's the bug I've seen most often with events.

`IObservable<T>` and `IObserver<T>` from Rx.NET are the more powerful version. They're for **streams** of values over time, and they come with a huge set of operators — `Where`, `Select`, `Throttle`, `Buffer`, `CombineLatest`. You'd reach for Rx when you need to compose async data flows, debounce user input, merge multiple sources, or handle backpressure.

The decision rule I use: if it's a single discrete notification, use `event`. If it's a stream where you need composition or time-based logic, use `IObservable<T>`.

---

### Q: "Strategy vs State?"

The mechanics look almost identical — both inject a behavior object into a context. The intent is different.

Strategy is **chosen by the client** and typically doesn't change. The client decides "use the express shipping strategy" and that's that. The strategies don't know about each other.

State **changes itself** based on internal logic. An order in `Pending` state knows how to transition to `Paid`. The states are aware of each other and form a state machine.

So if I'm picking a payment processor based on the customer's country — that's Strategy. If I'm modeling an order that flows through Pending → Paid → Shipped → Delivered with rules about which transitions are allowed — that's State.

For Strategy in .NET, I usually inject `IEnumerable<IPaymentStrategy>` and pick by a key. For State, I'd reach for the **Stateless** library or a MassTransit state machine — hand-rolling state classes for nontrivial workflows gets ugly fast.

---

### Q: "Why not just use a switch on an enum for state?"

You can — and for a state machine with three states and two transitions, that's exactly what I'd do. The pattern only earns its complexity when the logic gets bigger.

The problem is when you have ten states and each one has different behavior across five operations. Now your switch statement is 50 cases long, lives in one giant class, and every time you add a new state you have to find every switch and update it. That violates Open/Closed and the code becomes a nightmare.

The State pattern moves each state's behavior into its own class. Adding a new state means adding a new class — no existing code changes. You also get to use polymorphism instead of conditionals, which means each state class can have its own dependencies and its own state.

So my rule: enum + switch for trivial cases, State pattern when you have meaningful behavior per state, and a library like Stateless when the state machine is the core of the system.

---

### Q: "Template Method vs Strategy?"

Template Method uses **inheritance** — the algorithm is defined in a base class, and subclasses override specific hook methods. It's compile-time and the structure is fixed.

Strategy uses **composition** — you inject a different algorithm at runtime. It's swappable.

The trade-off is the classic one: inheritance vs composition. Template Method is simpler when the variation is small and the algorithm structure is genuinely shared. Strategy is more flexible — you can swap behavior at runtime, you can mock for tests, and you avoid the fragility of deep inheritance hierarchies.

`BackgroundService.ExecuteAsync` is Template Method — the base class handles starting, stopping, and exception handling, and you override the one method that does the work. ASP.NET Core's filter attributes are similar.

In modern .NET, I lean toward Strategy by default because it works better with DI and testing. But Template Method still earns its keep in framework code where inheritance is the natural extension point.

---

### Q: "Why Visitor instead of just adding methods to the classes?"

Visitor is for when you have a stable class hierarchy but you keep wanting to add new operations to it. Adding methods to the classes works once or twice — but every new operation means modifying every class in the hierarchy. That violates Open/Closed and the classes start to bloat.

Visitor inverts this: the hierarchy stays clean, and each new operation becomes a new Visitor class. You can add as many operations as you want without touching the original classes.

The textbook .NET examples are Roslyn's `CSharpSyntaxVisitor<T>` and `ExpressionVisitor`. Both walk a tree of nodes — syntax nodes or expression tree nodes — and let you implement different operations as different visitors. One visitor might pretty-print the tree, another might rewrite it, another might extract metrics. The node classes never change.

The trade-off is that Visitor only works well when the **class hierarchy is stable** and the **operations are what change**. If you keep adding new node types, you have to update every visitor — Visitor flips the problem rather than solving it.

---

### Q: "Real example of Chain of Responsibility in ASP.NET Core?"

The ASP.NET Core middleware pipeline **is** Chain of Responsibility. Every middleware has the same signature — it receives the request, decides whether to handle it, do work before, do work after, and either pass it down the chain or short-circuit.

Authentication middleware might validate a token and either continue or return 401. Logging middleware records the request and always passes it on. Compression middleware modifies the response on the way back out. Each link is independent and you can rearrange or insert new ones without changing the others.

MediatR pipeline behaviors do the same thing for application logic — validation, logging, transactions all wrap the handler in the same chain pattern. FluentValidation rule chains are another example.

What I like about the pattern is that each handler has one job, and the pipeline composition is configuration rather than code. That's why ASP.NET Core's `app.Use...()` calls feel like building rather than coding.

---

### Q: "How does Memento differ from serialization?"

Memento is designed to preserve **encapsulation**. The originator class produces a memento that captures its internal state, and only the originator can read it back to restore. From outside, the memento is opaque — a black box.

Serialization just dumps the object's data into a portable format. It doesn't care about encapsulation — anyone with the serialized form can read everything.

So Memento is the right pattern when you need undo or snapshots **and** you care about not exposing internals. Serialization is fine when you just need to persist or transmit data.

EF Core's change tracker uses an idea similar to Memento — it captures the original values of an entity when it's loaded so it can detect changes and roll back if needed. You as a developer don't see the snapshot directly; it's encapsulated inside the tracker.

---

## ARCHITECTURE & ADVANCED

### Q: "When should you NOT use CQRS?"

For simple CRUD applications, CQRS is over-engineering. If your reads and writes use the same model, the same database, and have similar complexity — you're just adding indirection without benefit. I'd start with a normal layered architecture and introduce CQRS only when there's a clear trigger.

The triggers I look for:
- Reads and writes have **very different shapes** — for example, the write side enforces complex domain rules but reads need denormalized projections for performance
- They have **different scaling needs** — read-heavy systems where you want to scale reads independently
- The domain is complex enough to justify a separate write model (DDD)

One myth worth correcting: CQRS does not require separate databases. It just means separate models. You can have a single database where the write side uses one set of classes for commands and the read side uses Dapper to project directly into DTOs — that's CQRS, no extra infrastructure needed.

CQRS is also commonly paired with MediatR, which is what most teams reach for first. It gives you the command/query separation cleanly without committing to the heavier event sourcing and database split.

---

### Q: "Saga: Orchestration vs Choreography?"

Both coordinate long-running distributed transactions across services, but they differ in who's in charge.

**Orchestration** has a central coordinator — usually called the orchestrator or saga manager. It tells each service what to do and what to do next. The advantage is clarity: the entire flow lives in one place, you can debug it, you can add steps, you can see the state machine. The downside is the orchestrator becomes a central point of complexity and a kind of coupling — every service interaction goes through it.

**Choreography** has no central coordinator. Each service publishes events, and other services react. The order service publishes `OrderCreated`, the payment service reacts and publishes `PaymentProcessed`, the inventory service reacts to that, and so on. The advantage is decoupling — no service knows about the others. The disadvantage is the flow is hard to trace; you have to reconstruct it from logs, and it's easy to introduce subtle bugs through event ordering.

My default is **orchestration** for anything with non-trivial business rules, because the explicit flow is worth its weight in debugging time. Choreography fits when each step really is independent and you want maximum decoupling.

In .NET, **MassTransit** has excellent saga state machine support, and **NServiceBus** has long been the enterprise standard. Both let you write orchestration as a state machine with compensating actions for rollback.

The two things to always mention: **idempotency** (every step must be safe to retry) and **compensating transactions** (every step needs an "undo" because you can't use 2PC across services).

---

### Q: "Outbox — why not just publish directly to the message bus?"

Because you'd hit the **dual-write problem**. If your code does "save to database, then publish to bus" — those are two separate systems, and you can't make them atomic. The database commit can succeed and then the publish can fail. Now your data has changed but no one downstream knows. Or the reverse: publish succeeds, DB rollback happens, and you've sent a lie.

Outbox solves this by making both writes part of the **same database transaction**. You insert your business data and you insert a row into an outbox table — both in one transaction. Then a separate background process polls the outbox, publishes the message, and marks it sent.

That guarantees: if the business data is committed, the event will eventually be published. If the transaction rolls back, neither happens. You've collapsed two writes into one.

The trade-off is that publishing is now **eventually consistent** rather than synchronous, and you need **idempotent consumers** because the relay might publish the same message twice if it crashes between publish and mark-as-sent. At-least-once delivery is the standard guarantee, so consumers always need to handle duplicates.

In .NET, MassTransit has a built-in transactional outbox, and there are libraries like CAP that implement it specifically.

---

### Q: "What are the states of a circuit breaker?"

Three states.

**Closed** is the normal state — calls flow through to the dependency. The breaker counts failures. If failures cross a threshold within a window, it trips.

**Open** is the failure state — calls fail fast without ever reaching the dependency. This protects the failing service from being hammered while it recovers, and protects your service from waiting on calls that will time out anyway. After a configured cooldown period, the breaker moves to half-open.

**Half-Open** is the probing state — the breaker lets a small number of calls through to test whether the dependency has recovered. If they succeed, it goes back to closed. If they fail, it goes back to open and the cooldown resets.

In .NET, **Polly** is the long-standing library, but the newer `Microsoft.Extensions.Http.Resilience` package is the official Microsoft option and uses Polly under the hood. The important thing is to not use circuit breaker alone — it pairs with **retry**, **timeout**, and **bulkhead** policies. A retry policy without a timeout will hang. A circuit breaker without a retry will fail too eagerly. The combination is what gives you real resilience.

---

### Q: "Is the Repository pattern an anti-pattern over EF Core?"

This is a debated one, and I have a nuanced view.

The argument it's an anti-pattern: `DbContext` already implements Unit of Work, and `DbSet<T>` already implements Repository. Wrapping it again often hides EF Core's most powerful features — `Include`, projection with `Select`, `IQueryable` composition, change tracking. You end up with a generic `IRepository<T>` that takes lambdas, which is just a worse `DbSet`.

The argument for Repository: in DDD, the repository represents an **aggregate boundary**. It's not just data access — it's the contract that says "you can only load and save Orders as a whole, not OrderLines individually." That's a domain concept, not a persistence one. It also gives you a clean abstraction if you might swap data sources, or want to enforce specific query patterns.

My practical rule:
- For **simple apps and CRUD APIs**, query `DbContext` directly. Repository adds nothing but ceremony.
- For **DDD with aggregates**, use Repository — but make it specific (`IOrderRepository`), not generic (`IRepository<T>`), and have it return aggregates, not arbitrary queries.
- For **testability**, mock `DbContext` directly with InMemory provider or testcontainers. You don't need Repository just for tests.

The mistake I see most is the half-hearted generic repository that just re-exposes `IQueryable` — that's the worst of both worlds.

---

### Q: "Why use Service Collection Extension methods?"

They're the standard convention in modern .NET for grouping a feature's DI registrations into a single discoverable call.

Three reasons I use them:

First, **clean composition** in `Program.cs`. Without them, you'd have hundreds of lines of `services.AddScoped<...>()` calls. With extensions, you get `services.AddOrdering()`, `services.AddPayments()`, `services.AddNotifications()` — and each one encapsulates everything that feature needs.

Second, **library convention**. Every modern .NET library follows this pattern: `AddControllers`, `AddDbContext`, `AddAuthentication`, `AddSwaggerGen`. If you write a feature module, following the same convention makes it feel native.

Third, **encapsulation**. The feature's internal services can stay internal. The extension method exposes only the public registration surface, while the actual service classes can be `internal`. That's good module hygiene.

The full pattern usually includes a paired `IConfiguration` parameter for options binding, plus `IApplicationBuilder` extensions for any middleware the feature needs.

---

### Q: "Why REPR over MVC controllers?"

REPR — Request, Endpoint, Response — gives every endpoint its own class with its own request and response types. It's a reaction to the problem of fat controllers.

The issues with controllers: they accumulate over time. You start with `OrderController` having three actions. A year later it has thirty. They share fields, share dependencies most actions don't need, and finding the right action becomes a search exercise. The controller violates Single Responsibility — it's not one thing, it's twenty.

REPR puts each endpoint in its own class. `CreateOrderEndpoint` has its own request, response, dependencies, and validator. To find it, you search for the URL or the request name. To test it, you instantiate one class with its specific dependencies. Adding a new endpoint never modifies an existing file.

It pairs naturally with **vertical slice architecture**, where each feature is a folder containing the endpoint, handler, validator, and tests — everything related stays together instead of being scattered across Controllers/, Services/, Validators/, and DTOs/.

In .NET, the **FastEndpoints** library implements REPR directly. Minimal APIs lean toward this style — each `MapPost` is conceptually an endpoint, though without enforced structure. For larger projects I prefer FastEndpoints; for small APIs, Minimal APIs with a consistent folder layout work well.

---

### Q: "Difference between IOptions, IOptionsSnapshot, and IOptionsMonitor?"

This is one I've memorized because it comes up often.

**`IOptions<T>`** is registered as **singleton**. The value is built once at application startup and never changes. Use it for config that doesn't need to reflect changes — connection strings, feature flags set at deploy time. It works in any service lifetime.

**`IOptionsSnapshot<T>`** is **scoped** — you get a fresh value per request (per scope). It re-reads configuration each time a scope is created. Use it in web apps when you want config changes from `appsettings.json` to be picked up on the next request without restarting. Important: it only works in scoped or transient services. Inject it into a singleton and you'll get an exception.

**`IOptionsMonitor<T>`** is **singleton** with a **change notification API**. It exposes `CurrentValue` and an `OnChange` callback. Use it in singletons or background services that need to react to live config changes — for example, a long-running service that wants to update its behavior when a feature flag flips.

The decision tree:
- Config never changes after startup → `IOptions<T>`
- Web request scope, picks up changes → `IOptionsSnapshot<T>`
- Singleton or background service, needs live updates → `IOptionsMonitor<T>`

The mistake people make is injecting `IOptionsSnapshot<T>` into a singleton and getting confused when DI throws.

---

## CLOSING TIPS FOR DELIVERY

A few things that make these answers land in an interview:

**Start with a one-liner, then expand.** Interviewers can't process a long monologue. Lead with the direct answer ("Strategy is chosen by the client; State changes itself") and then unpack it.

**Always reach for a concrete .NET example.** "Like ASP.NET Core middleware" or "like `IOptionsMonitor`" is worth more than three minutes of theory. It signals you've actually used this stuff.

**Mention the trade-off unprompted.** Senior engineers don't just describe patterns — they describe when *not* to use them. If you can naturally say "I'd use this when X, but for simple cases I'd just do Y," you sound senior.

**It's okay to disagree.** "I'd push back on Repository over EF Core for simple apps" is a more interesting answer than parroting the pattern. Just have a reason.

**If you don't know, say so cleanly.** "I haven't used Visitor in production, but conceptually..." is much better than making something up. Interviewers can tell.

Good luck.
