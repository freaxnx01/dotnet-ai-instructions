[//]: # (GENERATED FILE — do not edit directly. Source: .ai/stacks/_partials/dotnet-core.md + .ai/stacks/_layers/dotnet-webapi.md. Run scripts/build-stacks.sh to regenerate.)

[//]: # (Stack partial — shared .NET conventions. Composed with a layer file under .ai/stacks/_layers/ by `scripts/build-stacks.sh` to produce a flat .ai/stacks/dotnet-*.md. Do not edit the generated file directly.)

# .NET Core Conventions

Shared baseline for every .NET stack overlay. Composed with a layer file (`dotnet-blazor` or `dotnet-webapi`) into the published flat overlay.

---

## Tech Stack (.NET baseline)

| Layer | Technology |
|---|---|
| Runtime | .NET 10 / C# |
| Backend | ASP.NET Core, Minimal API |
| ORM | Entity Framework Core |
| DB (small) | SQLite |
| DB (non-small) | PostgreSQL |
| Validation | FluentValidation |
| Logging | Serilog with structured output |
| Observability | OpenTelemetry (traces + metrics) |
| API docs | OpenAPI + Scalar |
| Containerization | Docker + docker-compose (Alpine base images) |
| Unit / integration testing | xUnit + FluentAssertions + NSubstitute |

---

## Architecture — Modular Monolith

- Separate top-level folders per module: `src/Modules/<ModuleName>/`
- Each module owns its Domain / Application / Infrastructure layers
- Modules communicate via in-process interfaces — never direct project references across modules
- Shared kernel in `src/Shared/` for cross-cutting types only
- Modules register their own DI services via `IServiceCollection` extension methods

```
src/
  Modules/
    Orders/
      Domain/
      Application/
      Infrastructure/
    Catalog/
      Domain/
      Application/
      Infrastructure/
  Shared/
  Host/           ← ASP.NET Core entry point, wires modules
```

### Hexagonal (Ports & Adapters) within a module

Apply when a module has multiple infrastructure adapters (e.g. REST + messaging) or needs strong testability isolation.

```
<Module>/
  Domain/           ← pure domain logic, no dependencies
  Application/
    Ports/
      Driving/      ← IOrderService (inbound)
      Driven/       ← IOrderRepository (outbound)
    UseCases/
  Infrastructure/
    Adapters/
      Persistence/
      Http/
      Messaging/
```

---

## C# Conventions

```xml
<!-- Directory.Build.props — applies to all projects -->
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <AnalysisLevel>latest-recommended</AnalysisLevel>
  <DebugType>embedded</DebugType>
  <DebugSymbols>true</DebugSymbols>
</PropertyGroup>
```

- File-scoped namespaces always
- `global using` for framework namespaces in each project
- `record` types for DTOs and value objects
- `sealed` by default on non-base classes
- No `var` when the type is not obvious from the right-hand side
- Prefer primary constructors (.NET 8+)
- Central Package Management via `Directory.Packages.props` — no versions in `.csproj`
- Use `ILogger<T>` for logging — never `Console.WriteLine`
- Use specific exception types — not generic `catch (Exception)`
- Use `CancellationToken` in all async methods that call external resources
- Use `async`/`await` end-to-end — never `Task.Result` or `.GetAwaiter().GetResult()`
- No `#nullable disable` or warning suppressions to fix build errors
- Never suppress nullable warnings with `!` without a clear comment

---

## API Design — Minimal API baseline

Every ASP.NET Core project (whether it exposes a REST surface or just a few endpoints for a Blazor app) follows these baseline conventions. The `dotnet-webapi` layer adds the deeper REST conventions on top.

- All endpoints grouped by module via `IEndpointRouteBuilder` extension methods
- One handler per file when the body is non-trivial; inline lambdas only for true one-liners
- Input validation via FluentValidation, run at the boundary before any handler logic
- Error responses are always `ProblemDetails` (RFC 9457) — never raw strings, anonymous error objects, or HTML error pages
- OpenAPI via `Microsoft.AspNetCore.OpenApi`; Scalar UI mounted at `/scalar`

```csharp
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
                       .WithTags("Orders")
                       .WithOpenApi();

        group.MapPost("/", CreateOrderAsync).WithName("CreateOrder");
        group.MapGet("/{id:guid}", GetOrderByIdAsync).WithName("GetOrderById");

        return app;
    }
}
```

---

## Entity Framework Core

- One `DbContext` per module (not one global context)
- Migrations in `<Module>/Infrastructure/Persistence/Migrations/`
- `IEntityTypeConfiguration<T>` per entity — no data annotations on domain models
- Never use `EF.Functions` in domain/application layers — only in infrastructure queries
- Always use `AsNoTracking()` for read-only queries
- Seed data via `IEntityTypeConfiguration.HasData()` or a dedicated seeder run at startup

```bash
# Add migration (run from repo root)
dotnet ef migrations add <MigrationName> \
  --project src/Modules/<Module>/Infrastructure \
  --startup-project src/Host

# Apply
dotnet ef database update \
  --project src/Modules/<Module>/Infrastructure \
  --startup-project src/Host

# Generate SQL script (for production review)
dotnet ef migrations script \
  --project src/Modules/<Module>/Infrastructure \
  --startup-project src/Host \
  --output migrations.sql
```

---

## Localization & Regional Formatting (server-side baseline)

Base rules for `de` / `en` support and regional formatting live in `base-instructions.md`. For every ASP.NET Core project on this stack:

- Configure `RequestLocalizationMiddleware` in `Program.cs`:
  ```csharp
  var supportedCultures = new[] { "de-CH", "de-DE", "de-AT", "en-US", "en-GB" }
      .Select(c => new CultureInfo(c)).ToList();
  var supportedUICultures = new[] { "de", "en" }
      .Select(c => new CultureInfo(c)).ToList();

  app.UseRequestLocalization(new RequestLocalizationOptions
  {
      DefaultRequestCulture = new RequestCulture("de-CH", "de"),
      SupportedCultures = supportedCultures,
      SupportedUICultures = supportedUICultures,
      ApplyCurrentCultureToResponseHeaders = true,
  });
  ```
- Culture resolution order: cookie (`.AspNetCore.Culture`) → `Accept-Language` header → default (`de-CH` / `de`)
- For language `de` with no recognized region (or a `de-*` region not in `SupportedCultures`), fall back to `de-CH` — never `de-DE`
- Format dates / numbers / currency via `CurrentCulture` — never `string.Format` with a hardcoded culture or `CultureInfo.InvariantCulture` for user-visible text

UI-specific localization rules (resource files for component strings, picker behaviour, language-switcher widgets) live in the Blazor layer.

---

## Testing Strategy

The base testing rules (TDD, no test modification to make green, full suite after implementation) live in `base-instructions.md`.

### Test project layout (baseline)

```
tests/
  <Module>.UnitTests/         ← xUnit, no I/O
  <Module>.IntegrationTests/  ← xUnit, real I/O via Testcontainers
```

Layer-specific test projects (Blazor component tests, Playwright E2E, API integration tests with `WebApplicationFactory`) are added by the layer overlay.

### Unit tests (xUnit)

- One test class per production class
- Naming: `MethodName_StateUnderTest_ExpectedBehavior`
- Use `FluentAssertions` for assertions
- Use `NSubstitute` for mocks/stubs
- No `[Fact]` with logic — use `[Theory]` + `[InlineData]` / `[MemberData]`
- After implementation, run the full test suite (`dotnet test`) — not just the new test

```csharp
public sealed class CreateOrderHandlerTests
{
    private readonly IOrderRepository _repository = Substitute.For<IOrderRepository>();
    private readonly CreateOrderHandler _sut;

    public CreateOrderHandlerTests() => _sut = new CreateOrderHandler(_repository);

    [Fact]
    public async Task Handle_ValidCommand_CreatesAndPersistsOrder()
    {
        // Arrange
        var command = new CreateOrderCommand(CustomerId: Guid.NewGuid(), Items: []);

        // Act
        var result = await _sut.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        await _repository.Received(1).AddAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
    }
}
```

---

## Essential Commands

```bash
# Restore / build (warnings as errors) / run
dotnet restore
dotnet build -c Release
dotnet run --project src/Host

# Run full stack locally
docker-compose -f docker-compose.yml -f docker-compose.override.yml up --build

# Tests
dotnet test                                         # all
dotnet test tests/<Module>.UnitTests                # unit only
dotnet test tests/<Module>.IntegrationTests         # integration (needs Docker)
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage

# Security / package checks
dotnet list package --vulnerable --fail-on-severity high
dotnet list package --outdated
```

**PDB symbols:** Release builds include embedded PDB symbols (`<DebugType>embedded</DebugType>` in `Directory.Build.props`) so exception stack traces contain source file names and line numbers in production. Never strip PDB symbols from release or Docker builds.

---

## Essential Make Targets

Projects using this stack should ship a repo-root `Makefile` standardizing the common commands. The recipe bodies may use project-local variables (`$(SLN)`, `$(API_DIR)`, `$(PROPS_FILE)`, `$(COMPOSE)`) but the target names are canonical.

A reference implementation lives at [`.ai/examples/dotnet/Makefile`](../examples/dotnet/Makefile) — copy it to your repo root and customize the top-of-file variables. Host/tool/project-specific targets (`run-edge`, `release-notes`, `package`) ship as stubs with per-OS examples in comments.

Document each target with an inline `## <description>` comment and expose a `help` target that greps them.

### Build & run
- `build` — build the solution in Release mode
- `watch` — run the API with hot reload (`dotnet watch`)
- `run-edge` — start the frontend and open it in the developer's preferred browser
  *Recipe is host-specific (Windows/WSL: powershell + msedge; macOS: `open -a Safari`; Linux: `xdg-open`). Standardize the target name; leave the body to each project.*

### Testing
- `test` — run every test project in the solution
- `test-unit` — run unit test projects only (iterate a `TEST_UNIT_PROJECTS` list)
- `test-coverage` — run tests with `--collect:"XPlat Code Coverage" --results-directory ./coverage`

### Docker (Compose)
- `docker-run` — `compose up --build` in the foreground
- `up` — `compose up -d --build`
- `down` — `compose down`
- `logs` — `compose logs -f`
- `rebuild` — `down` + `up`

### Quality
- `lint` — `dotnet format --verify-no-changes`
- `outdated` — `dotnet list package --outdated`
- `vuln` — `dotnet list package --vulnerable --include-transitive`

### Versioning (single source of truth: `Directory.Build.props` → `<Version>`)
- `version` — print current version
- `version-set V=X.Y.Z` — set version explicitly
- `bump-major` / `bump-minor` / `bump-patch` — SemVer bumps via `sed` on `Directory.Build.props`
- `bump-auto` — derive next version from Conventional Commits via `git-cliff --bumped-version`; refuse major bumps (require explicit `bump-major`)

### Release
- `changelog` — `git-cliff --output CHANGELOG.md`
- `release-notes` — generate user-friendly release notes for the current version
  *Recipe is tool-specific (Claude Code, Copilot CLI, llm CLI, OpenAI, hand-rolled). Standardize the target name; leave the body to each project.*
- `release` — tag `v$(VERSION)`, regenerate `CHANGELOG.md`, invoke `release-notes`, commit, tag (no auto-push)
- `release-auto` — `bump-auto` + `release` in one step
- `push-release` — `git push origin main "v$(VERSION)"` (run only after `release` succeeds)
- `package` — build a distributable artifact (ZIP / tarball / image) and deliver to the project's drop location
  *Recipe is project-specific (artifact format, drop location, signing). Standardize the target name; leave the body to each project.*

### Cleanup
- `clean` — remove `bin/`, `obj/`, `publish/` trees and `./coverage/`

---

## Docker

- Runtime base: `mcr.microsoft.com/dotnet/aspnet:10.0-alpine`
- Build base: `mcr.microsoft.com/dotnet/sdk:10.0-alpine`
- Multi-stage Dockerfile always
- Run as non-root user in final stage
- `docker-compose.yml` — production-like config
- `docker-compose.override.yml` — local dev overrides (ports, volumes, hot-reload)
- Secrets via environment variables or Docker secrets — **never in image or appsettings**

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-alpine AS build
WORKDIR /src
COPY . .
RUN dotnet publish src/Host -c Release -o /app/publish --no-self-contained

FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build /app/publish .
USER appuser
ENTRYPOINT ["dotnet", "Host.dll"]
```

---

## Logging & Observability

- Serilog configured in `Program.cs` via `UseSerilog()`
- Structured properties on every log entry: `{ModuleName}`, `{CorrelationId}`
- Use `LoggerMessage.Define` source-generated logging for hot paths
- Log levels: `Debug` local, `Information` production minimum
- OpenTelemetry: export traces to OTLP collector; expose `/metrics` (Prometheus format)
- Health checks: `/health/live` (liveness) and `/health/ready` (readiness, checks DB)

**12-Factor enforcement points for this stack:**
- Never write to the local filesystem inside a container for application state
- Never use `appsettings.Development.json` for secrets — always env vars
- EF Core migrations must be applied as a separate init container or pre-deploy step — **never** auto-migrated on `app.Run()`
- Serilog sink in production: stdout or OTLP — never file sink in Docker

---

## Security (stack baseline)

Base security rules live in `base-instructions.md`. For every project on this stack:

- HTTPS enforced in all environments; HSTS enabled
- Security response headers: `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`
- No secrets in `appsettings.json` — use `IConfiguration` with environment variable binding
- Run `dotnet list package --vulnerable --fail-on-severity high` in CI — fail build on HIGH/CRITICAL
- Validate all inputs at the API boundary with FluentValidation before any domain logic
- Error responses use `ProblemDetails` (no raw messages)

---

## Versioning (stack binding)

Base rules (SemVer, Conventional Commits → bump mapping, git-cliff) live in `base-instructions.md`. For this stack:

- One global version for all assemblies — defined once in `Directory.Build.props` as `<Version>`, never in individual `.csproj` files
- Docker images tagged with the same version + `latest` on stable releases

---

## CI/CD (GitHub Actions baseline)

Pipeline stages: `build` → `test` → `security-scan` → `docker-build` → `push`

```yaml
jobs:
  build-and-test:
    - dotnet restore
    - dotnet build --no-restore -c Release
    - dotnet test --no-build --collect:"XPlat Code Coverage"
    - dotnet list package --vulnerable --fail-on-severity high

  docker:
    needs: build-and-test
    - docker build
    - docker push (on main only)
```

Layer-specific CI jobs (E2E with Playwright for Blazor, k6 perf smoke for WebAPI) are added by the layer overlay.

---

## Project Scaffold Checklist (.NET baseline)

Inherits the base checklist from `base-instructions.md`, plus:

- [ ] `Directory.Build.props` with global compiler settings + `<Version>1.0.0</Version>`
- [ ] `Directory.Packages.props` with central package versions
- [ ] `.editorconfig` committed
- [ ] `global.json` pinning SDK version
- [ ] `cliff.toml` for `git-cliff` changelog generation
- [ ] `docker-compose.yml` + `docker-compose.override.yml`
- [ ] `Dockerfile` multi-stage, non-root user, Alpine
- [ ] `/health/live` and `/health/ready` endpoints wired
- [ ] Serilog + OpenTelemetry bootstrapped
- [ ] `RequestLocalizationMiddleware` configured for `de` / `en`
- [ ] GitHub Actions workflow for build + test + vulnerability scan

Layer-specific additions live in the layer's own checklist.

---

## Agent Guardrails (.NET baseline)

In addition to the base guardrails:

- Do not install additional NuGet packages without asking first
- Do not change project target frameworks
- Do not modify `.csproj` files unless the task requires it
- Do not introduce new patterns (e.g. MediatR, CQRS) unless explicitly asked

### Never generate (this stack)

- `async void` (except UI event handlers — see the Blazor layer)
- `Task.Result` or `.GetAwaiter().GetResult()` — always `await`
- Magic strings — use `const` or `nameof()`
- Direct `HttpClient` instantiation — always via `IHttpClientFactory`
- Secrets, connection strings, or credentials in source files
- Cross-module project references (use shared interfaces)
- Tests that are modified to pass (fix the implementation instead)
- Hardcoded return values, mock results, or stub logic to satisfy a test
- Silently swallowed exceptions to make a test green
- `#nullable disable` or warning suppressions to fix build errors
- Commented-out code blocks — delete them, git has history
- `Console.WriteLine` — use `ILogger<T>`
- Generic `catch (Exception)` — use specific exception types
- Missing `CancellationToken` on async methods that call external resources
- `using` statements for namespaces already covered by `global using`

---

[//]: # (Stack layer — composed with .ai/stacks/_partials/dotnet-core.md by `make build-stacks` to produce .ai/stacks/dotnet-webapi.md. Do not edit the generated file directly.)

# .NET WebAPI Layer

Backend-only ASP.NET Core REST API projects (no Blazor, no UI). Composed on top of the shared `dotnet-core` partial.

---

## Tech Stack (WebAPI additions)

| Layer | Technology |
|---|---|
| API style | REST · ASP.NET Core Minimal API |
| API versioning | `Asp.Versioning.Http` (URL-segment) |
| Authentication | One of: pass-through · API key (`X-API-Key`) · JWT bearer — **single scheme per project** |
| Error responses | `ProblemDetails` (RFC 9457) |
| API docs | `Microsoft.AspNetCore.OpenApi` + Scalar UI at `/scalar` |
| Manual / exploratory | Bruno (collections in `bruno/`) |
| Integration testing | xUnit + `WebApplicationFactory` + Testcontainers |
| Performance / load testing | k6 (scripts in `perf/`) |
| Client SDK generation | Kiota |

---

## API Design — Minimal API

- All endpoints grouped by module via `IEndpointRouteBuilder` extension methods
- Route prefix: `/api/v{version}/{module}/...` — see *API versioning* below for the URL format
- One handler per file when the body is non-trivial; inline lambdas only for true one-liners
- FluentValidation runs at the boundary, before any handler logic

```csharp
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/v{version:apiVersion}/orders")
                       .WithTags("Orders")
                       .WithOpenApi()
                       .HasApiVersion(1.0);

        group.MapPost("/", CreateOrderAsync).WithName("CreateOrder");
        group.MapGet("/{id:guid}", GetOrderByIdAsync).WithName("GetOrderById");

        return app;
    }
}
```

### HTTP status code conventions

| Code | Use |
|---|---|
| `200 OK` | Successful read or non-creating action |
| `201 Created` | Resource created — **must include `Location` header pointing at the new resource** |
| `202 Accepted` | Async work accepted — include `Location` to status resource (see LRO) |
| `204 No Content` | Successful PUT / PATCH / DELETE with no body |
| `400 Bad Request` | Malformed request (parse failure, missing required field) |
| `401 Unauthorized` | Missing or invalid credentials |
| `403 Forbidden` | Authenticated but not allowed |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Request conflicts with current resource state |
| `412 Precondition Failed` | `If-Match` ETag mismatch |
| `422 Unprocessable Entity` | Semantic validation failure (parsed OK, content invalid) |
| `429 Too Many Requests` | Rate limit hit — include `Retry-After` |

### HTTP GET with request body — forbidden for new endpoints

Per RFC 7231 / 9110, GET request bodies have no defined semantics. Servers, proxies, and caches frequently strip, ignore, or reject them, which causes silent breakage and cache poisoning.

- **New endpoints:** use query parameters. If the parameter set is too large or sensitive for a URL, use `POST /search` (or another action sub-resource).
- **Legacy endpoints:** allowed only when required for backward compatibility. Mark the endpoint with `[Obsolete]` (or `.WithMetadata(new ObsoleteAttribute())`) and emit a `Sunset` header carrying the planned removal date.

### Errors — always ProblemDetails

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

- Every error response — including those produced by middleware and model binding — is RFC 9457 `ProblemDetails`
- Never return raw strings, anonymous `{ error: "..." }` objects, or HTML error pages
- Populate `type`, `title`, `status`, `detail`, `instance` on every response; add a `traceId` extension keyed on the current `Activity.TraceId`

---

## API Versioning

Use `Asp.Versioning.Http` with **URL-segment** versioning. Format: `v1.0`, `v2.0`, `v2.1` — `MAJOR.MINOR`. The minor segment is part of the URL even when only the major bumps, so the URL shape stays consistent across the lifetime of the API.

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;   // unversioned URLs → v1.0
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
    options.ReportApiVersions = true;                     // emit api-supported-versions / api-deprecated-versions headers
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";                   // must produce "v{MAJOR}.{MINOR}" — verify against the installed Asp.Versioning version
    options.SubstituteApiVersionInUrl = true;
});
```

- **Unversioned URLs (`/api/orders/...`) are allowed only for backward compatibility.** They resolve to v1.0 explicitly — never to "latest". Rolling out v2.0 must not change what an unversioned caller hits.
- Deprecate an endpoint with `.HasDeprecatedApiVersion(1.0)` plus a `Sunset: <RFC 7231 date>` header on responses.
- Removal is a separate step from deprecation — no version is removed without an announced sunset window.

---

## Authentication

**One scheme per API project.** Do not mix schemes in the same service. The choice is made at project bootstrap and applies to every endpoint.

The three approved schemes:

### 1. Pass-through (BFF / wrapper APIs)

For projects that proxy or fan out to an upstream API, the upstream remains the source of authentication truth.

- Forward the incoming `Authorization` header verbatim to the upstream
- Do **not** validate, re-issue, decode, log, or transform the bearer token in transit
- Do not call `AddAuthentication()` for token validation in this project — auth lives upstream
- If the project also exposes its own non-proxied endpoints, it isn't a pass-through project; pick a different scheme

### 2. API key (`X-API-Key`)

Header-based key validation for service-to-service traffic where JWT is overkill.

- Header name is exactly `X-API-Key` — no alternatives, no query-string fallback
- Validate via a custom `AuthenticationHandler<ApiKeySchemeOptions>`; never inline-check in the endpoint
- Keys live in a secret store (env vars / Key Vault / Docker secret) — never in source, never in `appsettings.json`
- Constant-time comparison only (`CryptographicOperations.FixedTimeEquals`)
- Rotate keys without downtime by accepting a small set of valid keys, not a single value

### 3. JWT bearer (token validation)

For APIs consuming tokens issued elsewhere (Identity provider, OAuth2 / OIDC server).

- `AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(...)`
- Validate issuer, audience, lifetime, and signing key — never disable validation in any environment
- Authorization decisions via `AddAuthorization` policies — endpoints reference policy names, not raw role strings
- This API **consumes** tokens — token issuance belongs in a dedicated identity service, not here

### Cross-cutting auth rules

- `[Authorize]` (or `.RequireAuthorization()`) is the default for projects on the API key or JWT scheme; opt out individually with `[AllowAnonymous]`. Pass-through projects do not register an authentication scheme — the upstream enforces auth.
- Anonymous endpoints are limited to `/health/*`, `/scalar`, and the OpenAPI document
- Never log the `Authorization`, `Cookie`, or `X-API-Key` header (see HTTP logging below)

---

## Pagination

**Default to cursor-based** for new endpoints — offset pagination is unstable under concurrent inserts.

```
GET /api/v1.0/orders?pageSize=50&pageToken=<opaque>
```

Response:
```json
{
  "items": [ ... ],
  "nextPageToken": "<opaque>"   // null when exhausted
}
```

- `pageToken` is opaque to the client — base64 of an internal cursor (`{lastId, lastCreatedAt}`), never a row offset
- Maximum `pageSize` is bounded server-side; reject requests exceeding it with `400`
- Offset pagination (`?page=&pageSize=`) is allowed only for **small bounded admin lists** where stability under inserts is guaranteed (e.g. a fixed config table)

---

## Idempotency for unsafe methods

Accept an `Idempotency-Key` header on `POST` and `PATCH` (and `DELETE` if it triggers side-effects beyond removing a row).

- Cache the response keyed by `(route, key, principal)` for 24 h
- A retry with the same key returns the cached response — no duplicate side-effect, no second `201`
- A retry with the same key but a *different* request body is rejected with `409 Conflict`
- Keys are client-supplied opaque strings; the API does not generate them

---

## Optimistic concurrency

For mutable resources, surface the row version as an `ETag` and require `If-Match` on writes.

- `GET /resources/{id}` returns `ETag: "<rowversion>"`
- `PUT|PATCH|DELETE /resources/{id}` accepts `If-Match: "<rowversion>"` — when present, mismatch returns `412 Precondition Failed`; when absent, the write proceeds without a concurrency check (lenient default — clients opt in to concurrency control by sending the header)
- Wire to EF Core: `[Timestamp] public byte[] RowVersion { get; set; }`
- The handler maps `DbUpdateConcurrencyException` to `412`

---

## Rate limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("per-user", httpContext =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: httpContext.User.Identity?.Name ?? httpContext.Connection.RemoteIpAddress!.ToString(),
            factory: _ => new FixedWindowRateLimiterOptions { PermitLimit = 100, Window = TimeSpan.FromMinutes(1) }));

    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        ctx.HttpContext.Response.Headers.RetryAfter = "60";
        await ctx.HttpContext.Response.WriteAsync("Rate limit exceeded.", ct);
    };
});
```

- Named policies per endpoint group — never a single global limit
- Always emit `Retry-After` on `429`
- Partition by authenticated principal first; fall back to remote IP only for anonymous endpoints

---

## CORS

- Explicit origin allowlist per environment via `WithOrigins(...)`
- **Never** combine `AllowAnyOrigin()` with `AllowCredentials()` — the combination is rejected by browsers and indicates a misconfiguration
- Methods and headers are scoped to what the API actually accepts — no blanket `AllowAnyMethod()`
- Preflight cache via `SetPreflightMaxAge(TimeSpan.FromHours(1))`

---

## HTTP logging

```csharp
builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields = HttpLoggingFields.RequestMethod
                          | HttpLoggingFields.RequestPath
                          | HttpLoggingFields.ResponseStatusCode
                          | HttpLoggingFields.Duration;
    options.RequestHeaders.Clear();
    options.ResponseHeaders.Clear();
    options.RequestHeaders.Add("User-Agent");
    options.RequestHeaders.Add("X-Correlation-Id");
});
```

**Never log** `Authorization`, `Cookie`, `Set-Cookie`, `X-API-Key`, or any header that may carry credentials. The `RequestHeaders.Clear()` call above is mandatory — the framework defaults include sensitive headers.

---

## Long-running operations

For work that takes longer than a request can reasonably hold open:

1. `POST /api/v1.0/<resource>` returns `202 Accepted` + `Location: /api/v1.0/operations/{opId}` + the operation as JSON
2. `GET /api/v1.0/operations/{opId}` returns `200 OK` with status `running | succeeded | failed` while in progress
3. On completion, the same endpoint returns `303 See Other` + `Location: /api/v1.0/<resource>/{id}` pointing at the created/updated resource
4. Operations are retained for at least 24 h after completion so polling clients can observe the terminal state

---

## Response compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
});
```

- Brotli first, gzip fallback
- Exclude already-compressed media types (`image/*`, `application/zip`, `application/x-protobuf`, etc.) — wasted CPU otherwise

---

## OpenAPI & Scalar

```csharp
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((document, _, _) =>
    {
        document.Info = new OpenApiInfo
        {
            Title = "Orders API",
            Version = "v1.0",
            Description = "...",
            Contact = new OpenApiContact { Name = "Platform Team", Email = "platform@example.com" },
            License = new OpenApiLicense { Name = "Proprietary" }
        };
        return Task.CompletedTask;
    });
});

app.MapOpenApi();
app.MapScalarApiReference(options =>
{
    options.WithDefaultHttpClient(ScalarTarget.Shell, ScalarClient.Curl);    // bash curl examples
    options.WithDefaultHttpClient(ScalarTarget.PowerShell, ScalarClient.Invoke); // PowerShell Invoke-RestMethod
    options.WithTheme(ScalarTheme.Default);
});
```

- API metadata (Title / Version / Description / Contact / License) is mandatory — published APIs without metadata are rejected in review
- Scalar UI at `/scalar`; OpenAPI document at `/openapi/v1.0.json`
- Code samples enabled for **bash curl** and **PowerShell** at minimum; other clients are opt-in
- Deprecated endpoints carry the OpenAPI `deprecated: true` flag *and* return a `Sunset` response header

---

## Client SDK generation

**Kiota** is the default for first-party `.NET` and `TypeScript` consumers:

```bash
kiota generate -l CSharp -d https://api.example.com/openapi/v1.0.json -o ./clients/dotnet -n Acme.Orders.Client
```

- Other languages consume the OpenAPI document directly
- Do **not** introduce NSwag, Refit, AutoREST, or hand-rolled `HttpClient` wrappers without an explicit ask — the OpenAPI document is the contract

---

## Testing (WebAPI additions)

The unit-test conventions and the baseline `<Module>.UnitTests` / `<Module>.IntegrationTests` layout live in the `dotnet-core` partial. For WebAPI projects, the integration test project uses `WebApplicationFactory` + Testcontainers, and one optional contract project may be added:

```
tests/
  Api.ContractTests/          ← optional — pinned OpenAPI snapshot
```

No bUnit, no Playwright — those are Blazor-stack concerns.

### Integration tests — WebApplicationFactory + Testcontainers

```csharp
public sealed class OrderApiTests : IClassFixture<WebApiFactory>
{
    private readonly HttpClient _client;

    public OrderApiTests(WebApiFactory factory) => _client = factory.CreateClient();

    [Fact]
    public async Task PostOrder_ValidPayload_Returns201WithLocation()
    {
        var response = await _client.PostAsJsonAsync("/api/v1.0/orders",
            new CreateOrderRequest(CustomerId: Guid.NewGuid(), Items: []));

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

- `WebApiFactory : WebApplicationFactory<Program>` swaps real infrastructure for Testcontainers (Postgres, Redis, etc.)
- Each test class owns its database via Testcontainers — no shared mutable state across classes
- Authentication in tests: register a test scheme that injects a known principal — never call the real identity provider

### Manual / exploratory testing — Bruno

Collections in `bruno/`, committed to Git. One folder per module, mirroring API routes.

```
bruno/
├── bruno.json
├── environments/
│   ├── local.bru
│   └── staging.bru
└── <module>/
    ├── create-<entity>.bru
    ├── get-<entity>-by-id.bru
    ├── update-<entity>.bru
    └── delete-<entity>.bru
```

- One folder per module
- Request files named for the action: `create-order.bru`, `get-order-by-id.bru`
- Base URLs and tokens via Bruno environments — never hardcoded in `.bru` files
- When an endpoint is added or changed, the corresponding Bruno request is added or updated in the same PR
- Include realistic example bodies and useful assertions (status code, response shape)

### Performance / load testing — k6

Scripts in `perf/`, committed to Git. One scenario per critical user journey or hot endpoint.

```
perf/
├── scenarios/
│   ├── create-order.smoke.js
│   ├── create-order.load.js
│   └── browse-catalog.soak.js
├── lib/
│   └── auth.js              ← shared helpers (token acquisition, fixtures)
└── thresholds.js            ← shared SLO thresholds
```

- Scenario naming: `<endpoint-or-journey>.<profile>.js` where `<profile>` is `smoke` (1 VU, ~30 s), `load` (steady state at target rps), `stress` (ramp past expected peak), or `soak` (sustained over hours)
- Every script declares `thresholds` for `http_req_duration` (e.g. `p(95)<300`) and `http_req_failed` (e.g. `rate<0.01`); a failed threshold fails the run and the CI job
- Target environment via `K6_BASE_URL` env var — never hardcode hosts
- Authentication via shared helpers in `perf/lib/` — never embed real tokens in scripts
- CI: smoke profile runs on every PR (fast, blocking); load / stress / soak run on demand or on schedule (long, non-blocking gate)
- Output: write JSON results to a CI artifact via `--out json=results.json`; optional push to InfluxDB / Grafana for trending
- When an endpoint's expected throughput or latency budget changes, update the corresponding scenario and its thresholds in the same PR

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    steady: { executor: 'constant-arrival-rate', rate: 50, timeUnit: '1s', duration: '5m', preAllocatedVUs: 50 }
  },
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get(`${__ENV.K6_BASE_URL}/api/v1.0/orders?pageSize=20`);
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

---

## Project Scaffold Checklist (WebAPI additions)

Inherits the base `dotnet-core` checklist, plus:

- [ ] `Asp.Versioning.Http` registered with URL-segment versioning, default v1.0, group name format produces `v{MAJOR}.{MINOR}` URLs
- [ ] Single authentication scheme chosen and documented in `README.md`
- [ ] `AddProblemDetails()` + global `IExceptionHandler` wired in `Program.cs`
- [ ] `AddRateLimiter()` with at least one named policy
- [ ] `AddHttpLogging()` with sensitive headers explicitly cleared from logging
- [ ] CORS configured with explicit origin allowlist per environment
- [ ] `AddResponseCompression()` enabled
- [ ] OpenAPI metadata (Title / Version / Description / Contact / License) populated
- [ ] Scalar UI at `/scalar` with curl + PowerShell code samples enabled
- [ ] Kiota generation script committed (or documented as N/A)
- [ ] `bruno/` collection seeded with at least one happy-path request per endpoint
- [ ] Integration test project using `WebApplicationFactory` + Testcontainers
- [ ] `perf/` directory with at least one k6 smoke scenario per critical endpoint, wired into CI

---

## Agent Guardrails (WebAPI additions)

In addition to the base and `dotnet-core` guardrails:

- Do not enable a second authentication scheme in an API project that already has one — the choice is project-wide
- Do not accept a GET request with a body on a new endpoint — use query params or `POST /search`
- Do not return raw error strings, anonymous error objects, or HTML error pages — always `ProblemDetails`
- Do not combine `AllowAnyOrigin()` with `AllowCredentials()` in CORS configuration
- Do not log the `Authorization`, `Cookie`, `Set-Cookie`, or `X-API-Key` headers
- Do not omit the minor segment in URL paths (use `v1.0`, not `v1`)
- Do not let an unversioned URL resolve to "latest" — pin it to v1.0 explicitly
- Do not introduce NSwag, Refit, or AutoREST — Kiota is the default client generator
- Do not create POST or PATCH endpoints without considering whether `Idempotency-Key` should be supported
- Do not skip `Location` headers on `201 Created` or `202 Accepted` responses
- Do not disable JWT validation (issuer, audience, lifetime, signing key) in any environment, including local
- Do not introduce token issuance into a consumer API — token issuance belongs in a dedicated identity service
