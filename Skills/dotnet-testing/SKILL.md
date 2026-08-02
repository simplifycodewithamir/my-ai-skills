---
name: dotnet-testing
description: Use this skill whenever writing or reviewing unit tests or integration tests for .NET/ASP.NET Core code — xUnit tests, Testcontainers setup, WebApplicationFactory, mocking, or test-first work for a non-trivial change. Trigger for "write tests", "add unit tests", "integration test this endpoint", or any request to verify .NET behavior, even if "test" isn't said explicitly (e.g. "make sure this works").
---

# .NET Testing Conventions (Unit + Integration)

## Do
- **xUnit** for everything.
- Write the failing test **before** the implementation for any non-trivial change — "done" is defined by a passing test that was failing first.
- Test **behavior through the public surface**, never private methods. Internals are exposed to tests/mocking frameworks via `internal` + `InternalsVisibleTo`, not by making things public.
- Name tests `Method_Scenario_ExpectedResult`.
- **Integration tests**: use **Testcontainers** against real Postgres/Redis — this is the default, not a mock. An in-memory DB (e.g. SQLite) is an acceptable alternative when Testcontainers is impractical for the case at hand.
- Integration and performance tests use **`WebApplicationFactory`**, not `TestServer`.
- Every test asserts something meaningful — no test that runs code and checks nothing.
- Honor `CancellationToken` in test setup/teardown the same way production code does.

## Don't
- Don't mock the database in integration tests — use Testcontainers (or in-memory DB) against the real engine.
- Don't reach into private members via reflection to test internals — expose via `internal`/`InternalsVisibleTo` instead.
- Don't use `TestServer` directly for integration/performance tests — use `WebApplicationFactory`.
- Don't write a test after the implementation for non-trivial changes — write it first.
- Don't write assertion-free tests ("smoke tests" that just don't throw) as a substitute for real coverage.
- Don't rely on `DateTime.Now`/real wall-clock time in tests — the production code injects `TimeProvider`; use a fake/test provider.

## Output expectations
- Unit tests: fast, isolated, no real I/O.
- Integration tests: spin up Testcontainers (or in-memory DB) + `WebApplicationFactory`, exercise the real HTTP pipeline.
- Flag any behavior that can't be tested through the public surface as-is — that's a design smell to raise, not silently work around.
