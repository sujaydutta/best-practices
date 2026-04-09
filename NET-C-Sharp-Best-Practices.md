# .NET & C# Development Best Practices (2026 Edition)

The .NET ecosystem in 2026 is mature, high-performance, and highly productive. **.NET 10** (LTS, released November 2025) is the current production recommendation with long-term support until November 2028. **.NET 11** is in preview. **C# 14** ships with .NET 10 and brings powerful productivity features such as **extension members**, **field-backed properties**, **null-conditional assignment**, and better Span ergonomics.

**ASP.NET Core 10** continues to evolve with excellent support for **Minimal APIs** (preferred for new greenfield projects), refined controllers, OpenAPI integration, and Native AOT.

These best practices focus on writing clean, performant, maintainable, and secure code while leveraging the latest language and runtime capabilities.

## 1. Choose the Right .NET Version

- **Production recommendation**: **.NET 10 LTS** — excellent performance, improved Native AOT, better JIT (AVX-512 / ARM SVE support), and mature tooling.
- **Bleeding-edge development**: **.NET 11** preview for testing upcoming features.
- Never start new projects on .NET Framework 4.x unless maintaining legacy code.

Set your target framework to `net10.0` and use `<LangVersion>14</LangVersion>` (default when targeting net10.0).

## 2. Project Setup & Tooling

- Use **Visual Studio 2026** or **JetBrains Rider 2025.3+** for full C# 14 support.
- Prefer **minimal project files** and **file-scoped namespaces**.
- Use **global.json** to pin the SDK version.
- Adopt **.NET Aspire** for cloud-native orchestration in distributed systems.
- Enable **nullable reference types** (`<Nullable>enable</Nullable>`) by default.

## 3. Leverage Modern C# Language Features (C# 14 & Earlier)

C# has evolved into one of the most expressive and safe languages. Use these features aggressively:

| Feature                        | Since      | Benefit                                      | Example |
|--------------------------------|------------|----------------------------------------------|---------|
| **var**                        | C# 3       | Local type inference                         | `var user = GetUser(id);` |
| **Records**                    | C# 9       | Immutable data with value equality           | `public record User(int Id, string Name, string Email);` |
| **Pattern Matching**           | C# 7+ (enhanced) | Exhaustive, concise logic                   | `user switch { { Age: >= 18 } => "Adult", ... };` |
| **Switch Expressions**         | C# 8       | Expression-based, readable                   | `var status = order.Status switch { OrderStatus.Pending => "Waiting", ... };` |
| **Target-typed new**           | C# 9       | Less repetition                              | `User user = new("Alice", "alice@example.com");` |
| **Init-only properties**       | C# 9       | Immutable objects                            | `public string Name { get; init; }` |
| **Required members**           | C# 11      | Enforce initialization                       | `public required string Email { get; init; }` |
| **Primary constructors**       | C# 12      | Concise classes                              | `public class UserService(UserRepository repo) { ... }` |
| **Extension members** (new)    | C# 14      | Add properties, operators, statics to existing types | `public static int WordCount(this string s) => ...;` (now with full members) |
| **Field keyword** (in properties) | C# 14   | Backing field without manual declaration     | `public string Name { get => field; set => field = value.Trim(); }` |
| **Null-conditional assignment**| C# 14      | Safe concise assignment                      | `user?.Preferences?.Theme = "dark";` |

**Other strong features**:
- **Raw string literals** (`""" ... """`)
- **List patterns** and **relational patterns**
- **Span<T>** and **ReadOnlySpan<T>** for high-performance code
- **Source generators** for compile-time code generation

**Rule**: Default to records for DTOs, primary constructors, and pattern matching. Use extension members to keep APIs clean without polluting original types.

## 4. Architecture & Dependency Injection

- Use **constructor injection** everywhere.
- Prefer **scoped**, **transient**, and **singleton** lifetimes correctly (avoid capturing scoped services in singletons).
- Organize code with **Vertical Slice Architecture** or **Feature Folders** instead of traditional layered by technical concern.
- For large solutions, use **.NET Aspire** or modular monoliths.

## 5. Building APIs in ASP.NET Core

**Recommendation for new projects (2026)**: Start with **Minimal APIs** — they are faster, have lower overhead, better Native AOT support, and produce cleaner code.

**When to use Controllers**:
- Very large APIs with heavy shared filters, model binding complexity, or when your team strongly prefers MVC-style structure.

**Best practices for Minimal APIs**:
```csharp
app.MapGet("/users/{id}", async (int id, IUserService service) =>
    TypedResults.Ok(await service.GetByIdAsync(id)))
    .WithName("GetUser")
    .WithOpenApi();

app.MapPost("/users", async (CreateUserRequest req, IUserService service) =>
    TypedResults.Created($"/users/{req.Id}", await service.CreateAsync(req)));
```

- Use `TypedResults` for explicit status codes.
- Leverage endpoint filters for cross-cutting concerns (validation, authorization).
- Integrate **OpenAPI** automatically (excellent support in .NET 10).
- Return consistent `ProblemDetails` for errors.

Combine with the REST API best practices document for idempotency, pagination, status codes, etc.

## 6. Data Access

- Use **EF Core 10** with compiled models for performance.
- Apply `AsNoTracking()` for read-only queries.
- Use **Dapper** for high-performance read paths when EF overhead is noticeable.
- Always use asynchronous APIs (`await`).

## 7. Security

- Use **Microsoft Identity** or **OpenIddict** / **Duende IdentityServer** for authentication.
- Prefer **JWT** + **OAuth 2.1** / OpenID Connect.
- Enable **HTTPS** enforcement, CORS, rate limiting (built-in in ASP.NET Core), and anti-XSS protections.
- Store secrets with **User Secrets**, **Azure Key Vault**, or **AWS Secrets Manager**.
- Use **minimal APIs endpoint filters** or **authorization attributes** consistently.

## 8. Testing Best Practices

- **xUnit** or **NUnit** + **FluentAssertions** + **Moq** / **NSubstitute**.
- Use **WebApplicationFactory** for integration tests of Minimal APIs.
- Add **Testcontainers** for realistic DB and infrastructure tests.
- Write **unit**, **integration**, and **contract tests**.
- Aim for high coverage with meaningful tests.

## 9. Observability & Logging

- Use **OpenTelemetry** + **Microsoft.Extensions.Logging**.
- Structured logging (JSON) with **Serilog** or built-in providers.
- Enable health checks, metrics, and distributed tracing.
- Integrate with **Prometheus**, **Grafana**, or cloud-native monitoring.

## 10. Performance, Resilience & Deployment

- Use **Native AOT** where cold start and memory usage matter (see dedicated section below).
- Prefer **async/await** for I/O-bound work; avoid it for pure CPU work.
- Implement resilience with **Polly** (or built-in retry policies).
- Cache aggressively with **HybridCache** (new in recent .NET versions).

## 11. Native AOT Compilation – Best Practices

**Native AOT** in .NET 10 produces small, fast-starting, self-contained native executables — ideal for serverless, containers, and edge scenarios.

**How to enable**:
```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <OptimizationPreference>Size</OptimizationPreference> <!-- or Speed -->
</PropertyGroup>
```

**Best practices**:
- Use **Minimal APIs** + **System.Text.Json source generators** (`[JsonSerializable]`).
- Avoid reflection-heavy libraries (or use `[DynamicDependency]` / trimming metadata).
- Run the **AOT compatibility analyzer** during build.
- Test thoroughly in Native AOT mode (some libraries are not fully compatible).
- Trim aggressively but monitor for runtime failures.
- Great for microservices, AWS Lambda, Azure Functions, and high-density Kubernetes deployments.
- **Trade-offs**: Longer build times, limited dynamic features (no runtime code generation in some cases).

**Pro tip**: Start with regular publish, then enable Native AOT for performance-critical services. In .NET 10, many apps achieve sub-10ms startup and <10 MB binaries.

## 12. Additional Critical Best Practices

- Enforce code quality with **Roslyn analyzers**, **EditorConfig**, **SonarQube**, and **ReSharper**.
- Maintain consistent **naming**, **date formats** (DateTimeOffset + UTC), and error envelopes.
- Use **source generators** for repetitive code (DTO mapping, validation, etc.).
- Version APIs (URL or header) and keep backward compatibility.
- Document with **OpenAPI** / Swagger UI.

## 13. Common Pitfalls to Avoid

- Blocking calls on async threads.
- Capturing scoped services in singletons.
- Overusing reflection in hot paths (especially with Native AOT).
- Ignoring trimming warnings.
- Mixing Minimal APIs and Controllers inconsistently in one project.
- Returning large collections without pagination or streaming.
- Not using `TypedResults` or proper status codes.

---

**Final Advice**:  
In 2026, **.NET 10 + C# 14** offers an outstanding developer experience. Leverage **records**, **primary constructors**, **extension members**, **pattern matching**, and **Minimal APIs** to write concise, safe, and high-performance code. Use **Native AOT** for services where startup time and memory footprint matter. Focus on vertical slices, strong typing, observability, and automated testing.

Treat your .NET applications like products: clean, observable, secure, and evolvable. Follow these practices and your services will be fast, reliable, and a joy to maintain for years to come.

Happy coding! 🚀