---
name: dotnet-production-code
description: Use this skill whenever writing or reviewing production C#/.NET code for ASP.NET Core services — new endpoints, services, data access, resilience, logging, feature flags, or any backend feature code. Trigger for "add an API", "create a service", "write an endpoint", data access work, or any .NET code review, even if the user doesn't say "production code" explicitly.
---

# .NET Production Code Conventions

## Do
- **Minimal APIs with endpoint groups** — `app.MapGroup("/api/v1/orders")` per resource, one file per group, handlers extracted to named static methods.
- Target **.NET 10 / latest C#**. Project files set `<Nullable>enable</Nullable>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- **Expected failures → exceptions**, caught by a global exception handler that maps to `ProblemDetails`. Domain exceptions carry meaning (`OrderNotFoundException`, `InsufficientStockException`). `ProblemDetails` always has: message, httpStatusCode, errorCode (e.g. `api.error.notfound`, `api.error.badrequest`). Throw the domain exception from the endpoint; the handler owns status-code mapping.
- **Async all the way** — no `.Result`, `.Wait()`, `async void`. Pass `CancellationToken` through every layer and honor it.
- Inject `TimeProvider`, never `DateTime.Now`. UTC everywhere; convert at the edge only.
- **Records** for DTOs/value objects, **classes** for entities/services. Prefer immutability.
- Primary constructors for DI. Sealed by default. `internal` unless it must be public.
- Structured logging with `ILogger<T>` + message templates (never string interpolation in logs). Never log secrets, tokens, or PII. Use Serilog + OpenTelemetry for logging/tracing.
- Validation at the edge (FluentValidation or a filter); the domain assumes valid input and enforces its own invariants.
- Separate project for DataAccess (EF Core), separate project for DataMigrations.
- Scalar for OpenAPI documentation. OAuth 2.0 JWT for securing APIs.
- External HTTP calls go through the .NET Resilience package: exponential retry, max 3 attempts, 8s → 16s → 32s.
- Microsoft.FeatureManagement for feature flags.
- EF Core: `AsNoTracking()` for reads, explicit `Include`. Flag any query that risks N+1. Migrations are generated, never hand-edited — surface the generated SQL for review before it's applied.
- Postgres is the default store; Redis for caching — every key gets a documented prefix and TTL.
- Parameterize all SQL. Never string-concatenate a query.
- Session-management framework and NoSQL-vs-relational choice: propose an option **per project** with a reason, don't default silently.
- implement throtlling, rate-limiting, and circuit-breaking for external calls using the .NET Resilience package.

## Don't
- Don't scaffold `[ApiController]` MVC controllers for new code.
- Don't use `Result<T>` / result-pattern wrappers for expected failures.
- Don't introduce MediatR, AutoMapper, or a repository layer over EF Core unless explicitly asked — explicit code first.
- Don't use `DateTime.Now` or unmanaged `DateTime.UtcNow` scattered through business logic — inject `TimeProvider`.
- Don't write blocking async calls (`.Result`, `.Wait()`) or `async void` (except top-level event handlers).
- Don't string-interpolate log messages — use `ILogger<T>` message templates.
- Don't hand-edit EF Core migrations.
- Don't silently substitute your own architecture when the user has specified an approach — flag the concern explicitly and let them decide.

## Output expectations
- Idiomatic C# 13 / .NET 10, nullable-aware, warnings-as-errors clean.
- New endpoints via `MapGroup()` + extension methods, not inline `Program.cs` clutter.
- Call out any deviation from these conventions and why, inline in the response.
