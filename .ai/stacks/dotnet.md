[//]: # (Stack overlay — loaded together with .ai/base-instructions.md for .NET projects)

# .NET Stack Overlay

Applies on top of `.ai/base-instructions.md` for .NET 10 / ASP.NET Core / Blazor / MudBlazor projects.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | .NET 10 / C# |
| Backend | ASP.NET Core, Minimal API |
| Frontend | Blazor CSR or SSR (per project) |
| UI Components | MudBlazor |
| ORM | Entity Framework Core |
| DB (small) | SQLite |
| DB (non-small) | PostgreSQL |
| Validation | FluentValidation |
| Logging | Serilog with structured output |
| Observability | OpenTelemetry (traces + metrics) |
| API Docs | OpenAPI + Scalar |
| API Testing | Bruno (collections in `bruno/`) |
| Containerization | Docker + docker-compose (Alpine base images) |
| Testing | xUnit + FluentAssertions + NSubstitute + bUnit + Playwright |

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
- No `#nullable disable` or warning suppressions to fix build errors
- Never suppress nullable warnings with `!` without a clear comment

---

## API Design (Minimal API)

- All endpoints grouped by module using `IEndpointRouteBuilder` extension methods
- Route prefix: `/api/v{version}/{module}/...`
- Error responses: RFC 9457 `ProblemDetails` — never return raw strings on error
- Input validation: FluentValidation, validated before handler logic
- OpenAPI via `Microsoft.AspNetCore.OpenApi` + Scalar UI at `/scalar`

```csharp
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/v1/orders")
                       .WithTags("Orders")
                       .WithOpenApi();

        group.MapPost("/", CreateOrderAsync).WithName("CreateOrder");
        group.MapGet("/{id:guid}", GetOrderByIdAsync).WithName("GetOrderById");

        return app;
    }
}
```

---

## API Testing (Bruno)

Use [Bruno](https://www.usebruno.com/) for manual and exploratory REST API testing. Collections are stored in `bruno/` at repo root and committed to Git.

```
bruno/
├── bruno.json                     ← collection config
├── environments/
│   ├── local.bru                  ← http://localhost:<port>
│   └── staging.bru
└── <module>/
    ├── create-<entity>.bru
    ├── get-<entity>-by-id.bru
    ├── update-<entity>.bru
    └── delete-<entity>.bru
```

- One folder per module, mirroring the API route structure
- Request files named with the action: `create-order.bru`, `get-order-by-id.bru`
- Use Bruno environments for base URL and auth tokens — never hardcode URLs or secrets in `.bru` files
- Keep requests in sync with endpoints — when adding/changing an API endpoint, update or add the corresponding Bruno request
- Include example request bodies with realistic test data
- Add assertions in Bruno where useful (status code, response shape)

---

## Testing Strategy

### Test Projects Structure

```
tests/
  <Module>.UnitTests/         ← xUnit, no I/O
  <Module>.IntegrationTests/  ← xUnit + WebApplicationFactory + Testcontainers
  <Module>.ComponentTests/    ← bUnit for Blazor components
  E2E/                        ← Playwright
```

### Unit Tests (xUnit)

- One test class per production class
- Naming: `MethodName_StateUnderTest_ExpectedBehavior`
- Use `FluentAssertions` for assertions
- Use `NSubstitute` for mocks/stubs
- No `[Fact]` with logic — use `[Theory]` + `[InlineData]` / `[MemberData]`
- **After implementation, run the full test suite** (`dotnet test`) — not just the new test

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

### Blazor Component Tests (bUnit)

- Test components in isolation using `bUnit` + `Bunit.Web.AngleSharp`
- Use `Ctx.RenderComponent<T>()` with parameter builders
- Assert on rendered markup and component state
- Mock services via `Ctx.Services.AddSingleton<IMyService>(mock)`
- Test event handlers: `cut.Find("button").Click()` then assert resulting state
- Test parameter changes: `cut.SetParametersAndRender(p => p.Add(x => x.Param, newValue))`
- Test async lifecycle: use `cut.WaitForState(() => condition)` for loading states

```csharp
public sealed class OrderListComponentTests : TestContext
{
    [Fact]
    public void OrderList_WithOrders_RendersOrderRows()
    {
        // Arrange
        Services.AddSingleton(Substitute.For<IOrderService>());

        // Act
        var cut = RenderComponent<OrderList>(p =>
            p.Add(c => c.Orders, [new OrderDto(Guid.NewGuid(), "Pending")]));

        // Assert
        cut.FindAll("tr.order-row").Should().HaveCount(1);
    }
}
```

### E2E Tests (Playwright)

- Tests in `tests/E2E/`
- Use `Microsoft.Playwright.NUnit` or xUnit wrapper
- Page Object Model (POM) pattern — no raw selectors in test methods
- Tests must be independent and idempotent (seed + teardown own data)
- Run against `docker-compose` stack in CI

```csharp
public sealed class OrderCreationTests : PageTest
{
    [Test]
    public async Task CreateOrder_ValidInput_ShowsConfirmation()
    {
        var page = new OrderPage(Page);
        await page.GotoAsync();
        await page.FillOrderFormAsync(customerId: "test-001");
        await page.SubmitAsync();
        await Expect(page.ConfirmationBanner).ToBeVisibleAsync();
    }
}
```

---

## Blazor Conventions

- CSR (WebAssembly) for full SPA, SSR for SEO-critical or auth-heavy pages
- MudBlazor as the only component library — no mixing with other UI libs
- Components in `src/Host/Components/` or per-module `Components/` folder
- `@code` block kept minimal — extract logic to services or `ViewModel` classes
- Use `[Parameter]` only for the public API of component; internal state via fields
- `EventCallback<T>` for child-to-parent communication

### MudBlazor Conventions

- Prefer MudBlazor components over raw HTML at all times
- Use `MudDataGrid` for tabular data (not `MudTable` unless legacy)
- Use `MudForm` + `MudTextField` / `MudSelect` for forms with validation
- Use `MudDialog` for confirmations and modals (not custom overlays)
- Use `MudSnackbar` for user feedback / toast messages
- Use `MudSkeleton` for loading states
- Layout: `MudLayout` → `MudAppBar` + `MudDrawer` + `MudMainContent`
- Icons: use `Icons.Material.Filled.*` consistently

### Component Conventions

- One component per file
- Component files: `PascalCase.razor`
- Code-behind files: `PascalCase.razor.cs` (partial class)
- Services injected via `@inject` or constructor in code-behind
- No business logic in `.razor` files — only binding and UI events
- Reuse components from `/src/Shared/` before creating new ones

### State & Data Flow

- Components do not call APIs directly — always go through a service
- Services are registered in `Program.cs` with appropriate lifetime
- Use `EventCallback` for child→parent communication
- Use `CascadingParameter` only for truly global state (e.g. auth, theme)

### UI workflow — stack-specific hints

The phase order and gates are defined in `base-instructions.md`. For .NET/Blazor projects:

- **Phase 1 (wireframe):** think in MudBlazor regions — `MudAppBar`, `MudDrawer`, `MudMainContent`, `MudDataGrid`, `MudForm`, `MudDialog`.
- **Phase 2 (flow):** use MudBlazor component names in the component & state map.
- **Phase 3 (build):** code-behind `.razor.cs` for all logic; use `MudSkeleton` / `MudProgressLinear` for loading, `MudSnackbar` for errors, `MudDialog` for destructive confirmations, `MudForm` + `DataAnnotations` for validation, `ma-*` / `pa-*` / `MudStack` / `MudGrid` for spacing.
- **Phase 4 (review):** verify no raw HTML where a MudBlazor component exists; `MudDataGrid` (not `MudTable`), `MudSnackbar` (not custom toast), `Icons.Material.Filled.*`, a bUnit test file exists for the component.

---

## Entity Framework Core

- One `DbContext` per module (not one global context)
- Migrations in `<Module>/Infrastructure/Persistence/Migrations/`
- `IEntityTypeConfiguration<T>` per entity — no data annotations on domain models
- Never use `EF.Functions` in domain/application layers — only in infrastructure queries
- Always use `AsNoTracking()` for read-only queries
- Seed data via `IEntityTypeConfiguration.HasData()` or dedicated seeder run at startup

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
dotnet test tests/<Module>.ComponentTests           # bUnit
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

## Security

- HTTPS enforced in all environments; HSTS enabled
- Security response headers: `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`
- No secrets in `appsettings.json` — use `IConfiguration` with environment variable binding
- Run `dotnet list package --vulnerable --fail-on-severity high` in CI — fail build on HIGH/CRITICAL
- Validate all inputs at API boundary with FluentValidation before any domain logic
- Error responses use ProblemDetails (no raw messages)

---

## Versioning (stack binding)

Base rules (SemVer, Conventional Commits → bump mapping, git-cliff) live in `base-instructions.md`. For this stack:

- One global version for all assemblies — defined once in `Directory.Build.props` as `<Version>`, never in individual `.csproj` files
- Docker images tagged with the same version + `latest` on stable releases

---

## CI/CD (GitHub Actions)

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

  e2e:
    needs: docker
    - docker-compose up -d
    - dotnet test tests/E2E
    - docker-compose down
```

---

## Project Scaffold Checklist (.NET)

- [ ] `Directory.Build.props` with global compiler settings + `<Version>1.0.0</Version>`
- [ ] `Directory.Packages.props` with central package versions
- [ ] `.editorconfig` committed
- [ ] `global.json` pinning SDK version
- [ ] `CHANGELOG.md` with `[Unreleased]` section
- [ ] `cliff.toml` for `git-cliff` changelog generation
- [ ] `docker-compose.yml` + `docker-compose.override.yml`
- [ ] `Dockerfile` multi-stage, non-root user, Alpine
- [ ] `.github/copilot-instructions.md`
- [ ] `CLAUDE.md`
- [ ] `SKILL.md`
- [ ] `README.md` with setup + migration commands
- [ ] `/health` endpoints wired
- [ ] Serilog + OpenTelemetry bootstrapped
- [ ] GitHub Actions workflow
- [ ] Branch protection on `main`

---

## Agent Guardrails (stack-specific additions)

In addition to the base guardrails:

- Do not install additional NuGet packages without asking first
- Do not change project target frameworks
- Do not modify `.csproj` files unless the task requires it
- Do not introduce new patterns (e.g. MediatR, CQRS) unless explicitly asked

### Never generate (this stack)

- `async void` (except Blazor event handlers)
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
