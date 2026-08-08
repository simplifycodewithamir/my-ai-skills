# .NET / C# / SQL Interview Q&A

Answers written at a senior/architect level (15+ yrs, .NET) — focused on the "why" and trade-offs, not just definitions.

---

## LINQ

**Q: Difference between IEnumerable, IQueryable, and List<T>?**
A: `IEnumerable<T>` is the base abstraction for in-memory iteration — LINQ-to-Objects, executes with delegates. `IQueryable<T>` extends it and carries an `Expression Tree` + `IQueryProvider`, so LINQ-to-Entities (EF) can translate the query into SQL and push filtering/paging to the database instead of pulling everything into memory. `List<T>` is a concrete, already-materialized collection — indexable, mutable. Rule I follow: expose `IEnumerable`/`IQueryable` from repository layers, return `List<T>` only at the boundary where the caller needs to mutate or index it.

**Q: Deferred execution vs immediate execution in LINQ.**
A: Deferred (`Where`, `Select`, `OrderBy`) builds up the query but doesn't run it until enumerated (`foreach`, `ToList()`, `Count()` in some cases). Immediate operators (`ToList`, `ToArray`, `Count`, `First`) execute right away. The gotcha I've hit in production: a deferred `IQueryable` re-executes against the DB every time it's enumerated (e.g., inside a loop) — always materialize with `.ToList()` once if you'll reuse it.

**Q: How do joins work in LINQ (inner, left, group joins)?**
A:
```csharp
// inner join
var q = from e in employees join d in depts on e.DeptId equals d.Id select new { e.Name, d.DeptName };

// left join (group join + DefaultIfEmpty)
var q = from e in employees
        join d in depts on e.DeptId equals d.Id into ed
        from d in ed.DefaultIfEmpty()
        select new { e.Name, DeptName = d?.DeptName };

// group join
var q = from d in depts join e in employees on d.Id equals e.DeptId into grp select new { d.DeptName, Employees = grp };
```
Method syntax equivalent uses `Join`/`GroupJoin`. In EF Core, prefer navigation properties over explicit joins when possible — cleaner and the query translator usually does better.

**Q: How can you optimize LINQ queries for performance?**
A: Filter and project as early as possible (push `Where`/`Select` before materializing), avoid `.ToList()` mid-chain which forces client-side evaluation, use `AsNoTracking()` for read-only EF queries, avoid N+1 by using `Include`/`ThenInclude` or projection instead of lazy loading, use `Any()` instead of `Count() > 0`, and watch for client-vs-server evaluation warnings in EF Core logs — anything that can't translate to SQL silently pulls the whole table.

**Q: Example of SelectMany and when it's useful.**
A: Flattens a sequence of sequences into one sequence.
```csharp
var allOrderItems = customers.SelectMany(c => c.Orders.SelectMany(o => o.Items));
```
Useful whenever you have a one-to-many/nested collection and want a flat projection — e.g., all line items across all orders, without nested loops.

---

## .NET Core

**Q: Main differences between .NET Framework and .NET Core?**
A: .NET Framework is Windows-only, monolithic, tied to IIS/GAC. .NET Core (now unified as .NET 5+) is cross-platform, open-source, modular (NuGet-based), built for containers/microservices, has better performance (Kestrel, span-based APIs), and gets active feature investment — Framework is effectively in maintenance mode.

**Q: Explain dependency injection in .NET Core.**
A: Built-in IoC container (`IServiceCollection`/`IServiceProvider`) registered in `Program.cs`. Constructor injection is the convention. It solves loose coupling and testability — I depend on interfaces, the container resolves concrete types at runtime, so unit tests can substitute mocks without touching consumer code.

**Q: Role of IHostedService in .NET Core?**
A: Interface for background/long-running tasks that run alongside the web host — start/stop hooks tied to app lifetime (`StartAsync`/`StopAsync`). Used it for things like queue consumers, scheduled cache warmers. `BackgroundService` is the abstract base class that simplifies implementing it for continuous loops.

**Q: What is Kestrel and how does it work in ASP.NET Core?**
A: Kestrel is the cross-platform, built-in web server for ASP.NET Core — event-driven, based on libuv/managed sockets. It can serve directly or sit behind a reverse proxy (IIS, Nginx, Azure App Service front end) which handles TLS termination, request buffering, and additional hardening. In our deployments it's almost always behind a reverse proxy/load balancer, never exposed raw to the internet.

**Q: How do you manage configuration in .NET Core?**
A: Layered `IConfiguration` — `appsettings.json` → `appsettings.{Environment}.json` → environment variables → command-line args → (optionally) Key Vault/secret managers, each layer overriding the previous. Strongly-typed access via `IOptions<T>`/`IOptionsSnapshot<T>`/`IOptionsMonitor<T>` rather than pulling raw strings by key everywhere.

**Q: What are Minimal APIs introduced in .NET 6?**
A: A lightweight way to define HTTP endpoints without controllers/`Startup.cs` boilerplate — `app.MapGet("/x", () => ...)`. Good fit for small services/microservices or simple endpoints; for larger enterprise APIs I still lean toward controllers for structure (filters, model binding conventions, versioning, Swagger grouping are more mature there).

**Q: How does the middleware pipeline work in .NET Core?**
A: Ordered chain of request delegates registered in `Program.cs` via `app.Use...`. Each middleware can act before/after calling `next()`, forming a nested pipeline (like an onion). Order matters a lot — e.g., `UseAuthentication` must precede `UseAuthorization`, exception handling middleware should be registered first so it wraps everything downstream.

**Q: Difference between Transient, Scoped, and Singleton lifetimes?**
A: Transient — new instance every resolution (stateless, lightweight services). Scoped — one instance per HTTP request (e.g., `DbContext`). Singleton — one instance for the app's lifetime (e.g., config, caches). Classic bug: injecting a Scoped service into a Singleton — causes captive dependency issues; the container will throw or silently misbehave depending on validation settings.

**Q: How do you handle logging in .NET Core?**
A: Built-in `ILogger<T>` abstraction with providers (Console, Debug, EventSource) and pluggable sinks — we typically wire in Serilog/App Insights for structured logging, correlation IDs, and log levels per environment via configuration. Structured logging (not string concatenation) is what makes querying logs in production actually useful.

**Q: Role of Startup.cs in an ASP.NET Core application?**
A: Historically split into `ConfigureServices` (DI registration) and `Configure` (middleware pipeline). From .NET 6 onward this is merged into a single `Program.cs` using top-level statements and `WebApplicationBuilder` — same responsibilities, less ceremony.

**Q: Difference between IConfiguration and IOptions?**
A: `IConfiguration` gives raw, untyped key-value access to the whole config tree. `IOptions<T>` binds a config section to a strongly-typed POCO, validated once at startup, giving compile-time safety and IntelliSense instead of magic strings. I default to `IOptions` pattern for anything beyond a couple of ad-hoc lookups.

**Q: Significance of the Program.cs file in .NET Core applications?**
A: It's the application entry point — builds the host (`WebApplicationBuilder`), registers services, configures the middleware pipeline, and calls `app.Run()`. In .NET 6+ it effectively absorbed `Startup.cs`'s responsibilities.

---

## Core C# / Threading

**Q: Difference between ref and out?**
A: Both pass by reference. `ref` requires the variable to be initialized before the call; `out` doesn't (but must be assigned inside the called method before it returns). I use `out` for TryParse-style patterns, `ref` when the callee needs to both read and mutate an existing value.

**Q: What is async/await?**
A: Compiler-driven pattern for asynchronous, non-blocking code. `await` suspends the method (returning control to the caller) until the awaited `Task` completes, without blocking the calling thread — critical for I/O-bound work (DB calls, HTTP calls) so threads aren't held hostage waiting on I/O. Key practice: use `ConfigureAwait(false)` in library code, avoid `.Result`/`.Wait()` which can deadlock in synchronization-context-bound environments.

**Q: Explain parallel programming.**
A: Running independent units of work concurrently across multiple CPU cores to reduce wall-clock time for CPU-bound work — `Parallel.For`/`Parallel.ForEach`, PLINQ (`.AsParallel()`), or `Task.WhenAll` for independent tasks. Different concern from async: async is about not blocking on I/O; parallelism is about using multiple cores for CPU-bound work. Mixing them up is a common junior mistake.

**Q: What is a thread?**
A: The OS-level unit of execution within a process — has its own stack and register context but shares the process's memory/heap. Threads are relatively expensive (context-switch cost, ~1MB stack); that's why async/await over the ThreadPool is preferred over spinning up raw threads for I/O-bound work.

**Q: Difference between Authentication and Authorization?**
A: Authentication answers "who are you" (verifying identity — login, token validation). Authorization answers "what are you allowed to do" (permissions/roles/policies), evaluated after authentication succeeds. In ASP.NET Core: `UseAuthentication()` must run before `UseAuthorization()` in the pipeline.

**Q: What is JWT and how does it work?**
A: JSON Web Token — a self-contained, signed token (Header.Payload.Signature, base64url-encoded) carrying claims about the user. Server issues it after login; client sends it in the `Authorization: Bearer` header on subsequent requests; the API validates the signature (and expiry/issuer/audience) without needing a server-side session store — stateless auth, good for scaling horizontally. Downside: revocation is hard since it's valid until expiry unless you add a blacklist/short expiry + refresh token pattern.

**Q: What is Middleware?**
A: See ".NET Core" section above — pipeline components that process HTTP requests/responses in sequence.

**Q: What is an Interface? (DI / loose coupling angle)**
A: A contract with no implementation — defines what a type must do, not how. It enables loose coupling: consumers depend on the abstraction (`IEmailService`) rather than a concrete class, so implementations can be swapped (real vs mock, SMTP vs SendGrid) without touching consumer code. This is what makes DI and unit testing practical — you register the interface-to-implementation mapping once in the container.

**Q: Scopes in Dependency Injection?**
A: See ".NET Core" section above (Transient/Scoped/Singleton).

**Q: What is Rate Limiting?**
A: Controlling how many requests a client can make in a time window, to protect the API from abuse/overload. .NET 7+ has a built-in `Microsoft.AspNetCore.RateLimiting` middleware supporting fixed window, sliding window, token bucket, and concurrency limiter strategies. In production I've also seen it enforced at the gateway/API Management layer rather than in-app, so it's consistent across services.

---

## SQL

**Q: Query to select the second highest salary from an Employee table.**
A:
```sql
SELECT MAX(Salary) AS SecondHighest
FROM Employee
WHERE Salary < (SELECT MAX(Salary) FROM Employee);

-- generalized to Nth highest using window function
SELECT Salary FROM (
  SELECT Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS rnk
  FROM Employee
) t WHERE rnk = 2;
```
I prefer `DENSE_RANK()` in real work — it generalizes to "Nth highest" and correctly handles duplicate salaries.

**Q: What is the ACID principle?**
A: Atomicity (all-or-nothing), Consistency (DB moves between valid states), Isolation (concurrent transactions don't interfere — governed by isolation levels), Durability (once committed, survives crashes). It's the guarantee relational DBs give you for transactional correctness — important to know which isolation level your ORM/queries actually run under (EF Core defaults to Read Committed on SQL Server).

**Q: Query to select duplicate rows.**
A:
```sql
SELECT Name, Email, COUNT(*) AS cnt
FROM Employee
GROUP BY Name, Email
HAVING COUNT(*) > 1;
```

**Q: Query to get the highest salary from each department (Employee, Department, Salary tables).**
A:
```sql
SELECT d.DeptName, MAX(s.Amount) AS MaxSalary
FROM Employee e
JOIN Department d ON e.DeptId = d.Id
JOIN Salary s ON s.EmployeeId = e.Id
GROUP BY d.DeptName;

-- if you also need the employee name attached, use a window function instead:
SELECT DeptName, EmployeeName, Amount FROM (
  SELECT d.DeptName, e.Name AS EmployeeName, s.Amount,
         RANK() OVER (PARTITION BY d.Id ORDER BY s.Amount DESC) AS rnk
  FROM Employee e JOIN Department d ON e.DeptId = d.Id JOIN Salary s ON s.EmployeeId = e.Id
) t WHERE rnk = 1;
```

**Q: What is Indexing?**
A: A data structure (typically B-tree) that lets the DB engine find rows without scanning the whole table — trades write cost and storage for read speed. Clustered index physically orders the table data (one per table); non-clustered indexes are separate structures pointing back to the row. Trade-off I always flag: over-indexing slows down inserts/updates and bloats storage, so index based on actual query patterns (WHERE/JOIN/ORDER BY columns), not defensively.

---

## Web API

**Q: What is REST, and how is it implemented in ASP.NET Web API?**
A: An architectural style for stateless, resource-oriented HTTP APIs — nouns as URLs, HTTP verbs as actions (GET/POST/PUT/DELETE), status codes conveying outcome, HATEOAS optionally for discoverability. ASP.NET Web API/Core implements it via attribute routing (`[HttpGet("api/orders/{id}")]`), model binding, content negotiation, and `IActionResult`/`ActionResult<T>` return types.

**Q: How do you secure a Web API (JWT, OAuth, API keys)?**
A: JWT bearer auth for stateless user-identity scenarios (validated via `AddAuthentication().AddJwtBearer(...)`), OAuth2/OpenID Connect when delegating auth to an identity provider (Azure AD, Auth0) — especially for third-party/client-credential flows, API keys for simpler service-to-service or partner integrations (less secure, easier to rotate/scope). In practice I layer these with HTTPS enforcement, short token expiry + refresh tokens, and scope/role-based authorization policies on top.

**Q: Difference between IActionResult and ActionResult<T>?**
A: `IActionResult` is the general interface for any action result (Ok, NotFound, BadRequest, etc.) but doesn't express the success payload type — Swagger/OpenAPI can't infer the return schema well. `ActionResult<T>` lets you return either `T` directly or any `IActionResult`, giving accurate API documentation while keeping the flexibility of status-code results. I default to `ActionResult<T>` for anything with generated API clients/Swagger consumers.

**Q: How do you implement versioning in Web API?**
A: URL segment (`/api/v1/orders`), query string (`?api-version=1.0`), or header-based versioning, typically via the `Asp.Versioning` (formerly `Microsoft.AspNetCore.Mvc.Versioning`) package. URL-segment versioning is what I've found easiest for consumers and caching/proxying; header-based is cleaner URLs but harder to test/debug ad hoc.

**Q: Difference between middleware and filters in ASP.NET Core?**
A: Middleware operates at the pipeline level — runs for every request regardless of which endpoint (or even if no endpoint matches), no access to MVC-specific context. Filters (`ActionFilter`, `ExceptionFilter`, `AuthorizationFilter`) run inside the MVC pipeline, scoped to controller/action execution, with access to action arguments/model state. Rule of thumb: cross-cutting concerns for the whole app (auth, CORS, exception handling, logging) → middleware; MVC-specific concerns (model validation, action-level logging, response shaping) → filters.

---

## Entity Framework

**Q: Difference between code-first, database-first, and model-first?**
A: Code-first — define C# entity classes, EF generates/migrates the schema (`dotnet ef migrations`); best for greenfield projects, git-friendly. Database-first — scaffold entity classes from an existing DB (`Scaffold-DbContext`); common when integrating with a legacy/DBA-owned schema. Model-first (EDMX-based, mostly legacy EF6/Framework, largely dead in EF Core) — design visually, generate both DB and classes from the model. On new work I always default to code-first.

**Q: Eager loading vs lazy loading vs explicit loading?**
A: Eager (`.Include()`/`.ThenInclude()`) — related data loaded in the same query, avoids N+1, predictable. Lazy — related data loaded automatically on first access via proxies, convenient but a classic performance trap (N+1 queries) if not watched. Explicit (`context.Entry(x).Collection(...).Load()`) — load related data on demand, after the fact, when you don't want it eager but need control over when it's fetched. I keep lazy loading disabled by default in our services and use eager/projection explicitly — it makes query cost visible in code review.

**Q: Difference between DbContext and ObjectContext?**
A: `ObjectContext` is the older EF (pre-EF Core, "EF 4/5/6") low-level API — more verbose, more control. `DbContext` is the simplified, higher-level wrapper introduced later (and is the only option in EF Core) — cleaner API, `DbSet<T>` collections, less boilerplate. In modern .NET, `ObjectContext` is essentially legacy/not relevant unless you're touching an old EF6 codebase.

**Q: How do you handle transactions in EF Core?**
A: `SaveChanges()` already wraps a single call in an implicit transaction. For multiple `SaveChanges()` calls or mixing EF with raw ADO.NET that need atomicity together, use `context.Database.BeginTransaction()` (or `context.Database.CreateExecutionStrategy().Execute(...)` when retry-on-failure is enabled, since transactions and automatic retry don't compose safely without it).

**Q: What are migrations in EF Core and how do you use them?**
A: Version-controlled, incremental scripts representing schema changes derived from model diffs. `dotnet ef migrations add <Name>` generates a migration class (Up/Down), `dotnet ef database update` applies it. In CI/CD I've always preferred generating idempotent SQL scripts (`dotnet ef migrations script --idempotent`) and running them as a controlled deployment step rather than calling `Database.Migrate()` at app startup, to avoid race conditions with multiple instances starting simultaneously.
