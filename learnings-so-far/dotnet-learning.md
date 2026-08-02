# .NET 10 / ASP.NET Core / EF Core — Learning Notes

*A working notebook, not a tutorial — 15 years in C#/.NET already, so this skips fundamentals and tracks what's actually new or different in .NET 10, ASP.NET Core, and EF Core. Updated as it's learned.*

**Stack:** .NET 10 / ASP.NET Core / EF Core

---

## 1. API documentation — OpenAPI + Scalar (2026-08-02)

Swashbuckle is no longer the default story. ASP.NET Core ships `Microsoft.AspNetCore.OpenApi` built-in for generating the OpenAPI document, and **Scalar** is the modern interactive UI on top of it — replacing Swagger UI.

### Old vs. new

| Old (Swashbuckle) | New (built-in OpenApi + Scalar) |
|---|---|
| `AddSwaggerGen()` + `UseSwaggerUI()` | `AddOpenApi()` + `MapOpenApi()` |
| Swagger UI at `/swagger` | Scalar UI at `/scalar/v1`, via `MapScalarApiReference()` |
| Third-party package for doc generation | Doc generation is in-box (`Microsoft.AspNetCore.OpenApi`); Scalar is the one NuGet package needed, just for the UI |

### Minimal wiring

```csharp
builder.Services.AddOpenApi();

var app = builder.Build();

app.MapOpenApi();              // serves the OpenAPI JSON, e.g. /openapi/v1.json
app.MapScalarApiReference();   // serves the Scalar UI, e.g. /scalar/v1

app.Run();
```

- `AddOpenApi()` / `MapOpenApi()` — built into ASP.NET Core itself, no third-party package.
- `MapScalarApiReference()` — from the `Scalar.AspNetCore` NuGet package. Reads the generated OpenAPI document and renders a browsable, testable UI.
- Neither is restricted to Development by default — wrap in `if (app.Environment.IsDevelopment())` deliberately if docs shouldn't ship to prod, same call the team used to make with Swagger.

### Still to figure out
- Customizing the OpenAPI document (titles, auth schemes, examples) — the old `SwaggerGenOptions` equivalent.
- Whether Scalar's UI supports the same "try it out" auth flows (bearer token, API key) the team relied on in Swagger UI.

---

## Roadmap

- [ ] EF Core: what's changed since EF Core 8/9 — complex types, JSON columns, interceptors.
- [ ] ASP.NET Core minimal APIs: anything new in request/response handling, validation.
- [ ] .NET 10 runtime/language: notable C# version features, performance changes.

---

*Living document — add a dated entry per topic as it's learned, don't rewrite past entries.*
