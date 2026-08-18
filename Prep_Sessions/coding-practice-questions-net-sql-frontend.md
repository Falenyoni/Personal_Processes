# Coding/Technical Practice Questions — .NET Fx 4.8 & .NET 8+, SQL, Frontend

Practice prompts, not pre-filled answers — the point is to actually work through these
yourself before Thursday. Mix of quick-fire explain/short-answer and real "write code"
exercises, since Cartrack/Picup candidates report live coding or HackerRank-style rounds
(see `INTERVIEW_RECITAL_MASTER.md` §9c).

---

## 1. .NET Framework 4.8 (older/legacy-flavoured questions)

These probe whether you actually know the pre-.NET-Core world, not just modern .NET —
worth being ready for if the role touches legacy systems.

1. What's the difference between `web.config` and `appsettings.json` — not just "one's XML,
   one's JSON," but how configuration transforms work (`web.Release.config`) and why that
   approach doesn't exist in modern .NET.
2. Explain the ASP.NET (non-Core) request lifecycle — `Global.asax`, `HttpApplication`,
   `HttpModule` vs `HttpHandler`. How does this compare to the ASP.NET Core middleware
   pipeline?
3. What is `System.Web`, and why can't it be referenced from a .NET Standard/.NET 8 class
   library? What did this force during a Framework → Core migration?
4. Write code: implement a simple synchronous method using classic `Thread` or
   `BackgroundWorker` (Framework-era concurrency), then show the modern `Task`-based
   equivalent. What actually changed underneath?
5. What's the difference between `WCF` and a modern ASP.NET Core Web API? Why did Microsoft
   not bring WCF server-side forward into modern .NET?
6. Explain `ConfigurationManager.AppSettings` vs `IConfiguration`/`IOptions<T>`. Why is the
   modern approach considered better for testability?
7. What is the GAC (Global Assembly Cache), and why is it largely irrelevant in modern .NET?
8. Framework-era `async`/`await` gotcha: what's `ConfigureAwait(false)` actually protecting
   against in an ASP.NET (non-Core) app specifically, and why is it less critical (though
   still good practice) in ASP.NET Core?
9. Write code: a Framework 4.8 app has a memory leak from event handlers never being
   unsubscribed (`someObject.SomeEvent += Handler;` with no matching `-=`). Explain why this
   causes a leak and fix it.
10. What's the difference between `.NET Framework`, `.NET Standard`, and modern `.NET`
    (5/6/7/8/9/10)? If asked to justify migrating a Framework 4.8 app, what's your pitch and
    what's the hardest part?

---

## 2. Modern .NET 8+ (coding/OOP-style)

1. Write code: implement a generic repository interface (`IRepository<T>`) with `GetById`,
   `Add`, `Update`, `Delete`, then a concrete in-memory implementation. Talk through why
   you'd (or wouldn't) genericize this in a real system.
2. Write code: given a list of orders, use LINQ to group by customer, sum order totals, and
   return the top 3 customers by spend. Do it once with method syntax, once with query
   syntax.
3. Explain records vs classes in modern C#. When would you choose a `record` for a domain
   entity vs a plain class? (Hint: think about your own `AggregateRoot` pattern in Utano —
   why isn't that a record?)
4. Write code: implement the Outbox pattern's core loop in pseudocode/C# — claiming unpublished
   events with optimistic concurrency, marking them published, handling a failed publish.
5. What's the difference between `IEnumerable<T>`, `IQueryable<T>`, and `List<T>`? Write a
   short example showing a LINQ query that behaves differently (deferred execution bug) if
   you're not careful about when it's materialized.
6. Explain nullable reference types (`string?` vs `string`). What problem do they actually
   catch at compile time vs runtime, and what's a real bug this would've caught?
7. Write code: a small `IPipelineBehavior<TRequest, TResponse>` (MediatR-style) that logs
   execution time around the inner handler — same pattern as `PermissionAuthorizationBehavior`
   in your own Utano code.
8. Explain the difference between `Task.Run`, `Task.WhenAll`, and `Task.WhenAny`. Write code
   using `Task.WhenAll` to run 3 independent async lookups concurrently and combine results.
9. What's a captive dependency (scoped service injected into a singleton)? You already have
   a great answer for this in your recital notes — be ready to also *write* the broken code
   and the fix, not just explain it verbally.
10. Write code: implement a basic circuit breaker from scratch (no Polly) — track failure
    count, open the circuit after N failures, half-open after a cooldown. Then explain why
    you'd use Polly instead in production.

---

## 3. SQL

1. Write a query: given `Orders(Id, CustomerId, OrderDate, Total)` and
   `Customers(Id, Name)`, return each customer's total spend and order count, customers with
   zero orders included (which JOIN type, and why).
2. Explain clustered vs non-clustered indexes. If a table is slow on a `WHERE Email = ?`
   lookup, what index would you add, and what's the trade-off of adding it?
3. Write a query using a window function (`ROW_NUMBER() OVER (PARTITION BY ...)`) to find
   the most recent order per customer — this is literally the same pattern used in the
   Utano email-dedup script from this session, be ready to explain it live.
4. What's the difference between `WHERE` and `HAVING`? Write a query that requires `HAVING`
   (can't be done with `WHERE` alone).
5. Explain database transaction isolation levels (Read Committed, Repeatable Read,
   Serializable). What's a real bug you'd expect from Read Committed that Serializable
   would prevent, and what does that protection cost you?
6. Write a query: find duplicate rows in a table (same `Email`, different `Id`) — same
   real-world problem as Utano's #40 login bug this session.
7. What's the N+1 query problem? Write a small EF Core example that causes it, then fix it
   with `.Include()` or a projection.
8. Explain optimistic vs pessimistic concurrency at the database level. Which does EF Core's
   default concurrency token approach use, and how would you implement it (`[Timestamp]`
   / `RowVersion`)?
9. Write a query: a self-join to find employees who earn more than their manager, given
   `Employees(Id, Name, Salary, ManagerId)`.
10. What's a covering index, and how does it avoid a "key lookup" in a query plan? When would
    you deliberately avoid adding one?

---

## 4. Frontend (React/TypeScript — matches your real Utano frontend experience)

1. Explain closures with a concrete example — write a counter function using a closure
   instead of a class or external state.
2. What's the difference between `useEffect` and `useLayoutEffect`? Give a real scenario
   where using the wrong one causes a visible bug (flicker, stale value).
3. Write code: a custom hook `useDebounce(value, delay)` — same kind of pattern you'd use
   for a search input, though you haven't needed it explicitly in Utano yet.
4. Explain React's reconciliation/key prop. Write a broken example (using array index as
   key with a reorderable list) and explain what breaks.
5. What's the difference between controlled and uncontrolled form inputs? Which does Utano's
   codebase use throughout (you'll know this one cold from this session's work)?
6. Write code: implement a simple debounced search box that calls an API only after the
   user stops typing for 300ms.
7. Explain `useMemo` vs `useCallback` — when does each actually matter for performance, and
   when is adding them premature optimization?
8. What's prop drilling, and what are the alternatives (Context, state libraries)? You've
   used `AuthContext`/`FeaturesContext` in Utano — be ready to explain why those are Context
   and not prop-drilled.
9. Explain the difference between `any`, `unknown`, and a properly typed generic in
   TypeScript. Why is `unknown` safer than `any`?
10. Write code: a TypeScript discriminated union for a network request state
    (`{status: 'idle'} | {status: 'loading'} | {status: 'success', data: T} | {status: 'error', error: string}`)
    and a component that renders correctly for each state.

---

## How to actually practice this

Don't just read these — pick 2-3 per category, set a timer (10-15 min each), and actually
write the code or say the explanation out loud. The live-coding risk isn't "do you know
syntax," it's "can you produce working code while talking through your thinking under mild
pressure" — that's a different skill from recognizing the right answer when you read it.
