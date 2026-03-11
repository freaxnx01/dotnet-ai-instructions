# AI Agent Base Instructions

Canonical reference for all AI coding agents. Tool-specific files (CLAUDE.md, copilot-instructions.md, SKILL.md) derive from this.

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
| Containerization | Docker + docker-compose (Alpine base images) |

---

## Architecture

### Default: Modular Monolith

- Separate top-level folders per module: `src/Modules/<ModuleName>/`
- Each module owns its domain, application, infrastructure layers
- Modules communicate via in-process interfaces, never direct project references across modules
- Shared kernel in `src/Shared/` for cross-cutting types only

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

### When Hexagonal (Ports & Adapters)

Apply hexagonal architecture within a module when:
- The module has multiple infrastructure adapters (e.g. both REST and messaging)
- The module needs testability isolation from external systems

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
<!-- Directory.Build.props - applies to all projects -->
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <AnalysisLevel>latest-recommended</AnalysisLevel>
</PropertyGroup>
```

- File-scoped namespaces always
- `global using` for framework namespaces in each project
- `record` types for DTOs and value objects
- `sealed` by default on non-base classes
- No `var` when the type is not obvious from the right-hand side
- Prefer primary constructors (.NET 8+)
- Central Package Management via `Directory.Packages.props` — no versions in `.csproj`

---

## API Design (Minimal API)

- All endpoints grouped by module using `IEndpointRouteBuilder` extension methods
- Route prefix: `/api/v{version}/{module}/...`
- Error responses: RFC 9457 `ProblemDetails` — never return raw strings on error
- Input validation: FluentValidation, validated before handler logic
- OpenAPI via `Microsoft.AspNetCore.OpenApi` + Scalar UI at `/scalar`

```csharp
// Endpoint registration pattern
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

## Testing Strategy

### Principle: TDD — Tests First, No Shortcuts

1. Write failing unit test
2. Write minimum implementation to pass
3. Refactor
4. **Never modify a test to make it green — fix the implementation**

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
- Use `[Parameter]` only for public API of component; internal state via fields
- `EventCallback<T>` for child-to-parent communication

---

## Entity Framework Core

- DbContext per module (not one global context)
- Migrations in `<Module>/Infrastructure/Persistence/Migrations/`
- `IEntityTypeConfiguration<T>` per entity — no data annotations on domain models
- Never use `EF.Functions` in domain/application layers — only in queries
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
```

---

## Docker

- Base images: `mcr.microsoft.com/dotnet/aspnet:10.0-alpine` for runtime
- Build images: `mcr.microsoft.com/dotnet/sdk:10.0-alpine` for build stage
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

## Security

- HTTPS enforced in all environments; HSTS enabled
- Security response headers: `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`
- No secrets in `appsettings.json` — use `IConfiguration` with environment variable binding
- Run `dotnet list package --vulnerable` in CI — fail build on HIGH/CRITICAL
- Validate all inputs at API boundary with FluentValidation before any domain logic

---

## Logging & Observability

- Serilog configured in `Program.cs` via `UseSerilog()`
- Structured properties on every log entry: `{ModuleName}`, `{CorrelationId}`
- Log levels: `Debug` local, `Information` production minimum
- OpenTelemetry: export traces to OTLP collector; expose `/metrics` (Prometheus format)
- Health checks: `/health/live` (liveness) and `/health/ready` (readiness, checks DB)

---

## Versioning (SemVer)

All projects follow [Semantic Versioning 2.0.0](https://semver.org/):

```
MAJOR.MINOR.PATCH  →  e.g. 2.4.1
```

| Increment | When |
|---|---|
| `MAJOR` | Breaking change — incompatible API or behaviour change |
| `MINOR` | New functionality, backwards-compatible |
| `PATCH` | Bug fix, backwards-compatible |

**Mapping from Conventional Commits:**

| Commit type | Version bump |
|---|---|
| `BREAKING CHANGE:` footer or `!` after type | MAJOR |
| `feat` | MINOR |
| `fix`, `perf` | PATCH |
| `chore`, `docs`, `ci`, `test`, `refactor` | No bump |

- Version is the single source of truth in the `.csproj` / `Directory.Build.props` as `<Version>`
- Git tags follow `v<MAJOR>.<MINOR>.<PATCH>` (e.g. `v1.3.0`) — tag on `main` after merge
- Pre-release: `v1.0.0-alpha.1`, `v1.0.0-beta.2`, `v1.0.0-rc.1`
- Docker images tagged with the same version + `latest` on stable releases
- Tools like `git-cliff` or `semantic-release` can automate bump + tag from commit history

---

## Changelog

All projects maintain a `CHANGELOG.md` in the repo root following [Keep a Changelog](https://keepachangelog.com) conventions.

```markdown
# Changelog

All notable changes to this project will be documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.1.0] - 2025-06-01
### Added
- Order cancellation endpoint

### Fixed
- Token refresh edge case on expiry boundary

## [1.0.0] - 2025-04-15
### Added
- Initial release
```

**Sections per release:** `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`

- `[Unreleased]` section accumulates changes until a release is cut
- Auto-generation: use `git-cliff` with `cliff.toml` configured for Conventional Commits
- CI can validate that `[Unreleased]` is not empty before allowing a release branch

---

## 12-Factor App Compliance

Projects follow the [12-Factor App](https://www.12factor.net/) methodology. Each factor and its implementation:

| Factor | Implementation |
|---|---|
| **I. Codebase** | One repo per service/app, tracked in Git |
| **II. Dependencies** | All declared in `.csproj` + `Directory.Packages.props`, nothing assumed from environment |
| **III. Config** | All config via environment variables — nothing environment-specific in `appsettings.json` |
| **IV. Backing services** | DB, cache, message broker treated as attached resources via connection string env vars |
| **V. Build, release, run** | Docker multi-stage: SDK image = build, final image = release+run. Never build in production |
| **VI. Processes** | Stateless processes — no sticky sessions, no local file state |
| **VII. Port binding** | App is self-contained; ASP.NET Core exports HTTP via Kestrel on configurable port |
| **VIII. Concurrency** | Scale via multiple container replicas, not threads |
| **IX. Disposability** | Fast startup, graceful shutdown via `IHostApplicationLifetime` + `CancellationToken` |
| **X. Dev/prod parity** | `docker-compose.override.yml` mirrors prod config as closely as possible |
| **XI. Logs** | Treat logs as event streams — Serilog writes to stdout, never to files in container |
| **XII. Admin processes** | Migrations and seed scripts run as one-off commands, not baked into app startup |

**Key enforcement points for AI agents:**
- Never write to the local filesystem inside a container for application state
- Never use `appsettings.Development.json` for secrets — always env vars
- EF Core migrations must be applied as a separate init container or pre-deploy step, not auto-migrated on `app.Run()`
- Serilog sink in production: stdout or OTLP — never file sink in Docker

---

## Branching Strategy (GitHub Flow + protection rules)

```
main              ← always deployable, protected
  └── feature/<issue-id>-short-description
  └── fix/<issue-id>-short-description
  └── chore/<short-description>
  └── release/<version>   ← only if needed for staged releases
```

- `main` requires: passing CI, at least 1 PR review, no direct push
- Branch from `main`, PR back to `main`
- Delete branch after merge
- Rebase or squash merge — no merge commits on `main`

---

## Commit Messages (Conventional Commits)

```
<type>(<scope>): <short summary>

[optional body]

[optional footer: Closes #<issue>]
```

**Types:** `feat`, `fix`, `test`, `refactor`, `chore`, `docs`, `ci`, `perf`  
**Scope:** module or layer name, e.g. `orders`, `auth`, `infra`, `blazor`

**SemVer mapping:** `feat` → MINOR bump · `fix`/`perf` → PATCH bump · `BREAKING CHANGE:` footer → MAJOR bump · all others → no bump

```
feat(orders): add order cancellation endpoint

Implements POST /api/v1/orders/{id}/cancel.
Validates order is in Pending state before cancelling.

Closes #42
```

- Subject line: imperative mood, ≤72 chars, no period
- Body: explain *why*, not *what*
- Breaking changes: add `BREAKING CHANGE:` footer

---

## Pull Request Conventions

### PR Title
Follow Conventional Commits format: `feat(orders): add cancellation endpoint`

### PR Description Template

```markdown
## Summary
<!-- What does this PR do and why? -->

## Changes
- 
- 

## Testing
- [ ] Unit tests added/updated
- [ ] Component tests (bUnit) added if Blazor changes
- [ ] E2E test added/updated if user-facing flow changed
- [ ] Tested locally with docker-compose

## Checklist
- [ ] Tests pass (`dotnet test`)
- [ ] No new `--vulnerable` packages
- [ ] No secrets committed
- [ ] Migrations included if schema changed
- [ ] OpenAPI spec still valid
```

### Review Guidelines
- PRs should be small and focused — one concern per PR
- Reviewers check: architecture adherence, test quality, security, no shortcuts that make tests green
- Auto-assign reviewers via `CODEOWNERS`

---

## CI/CD (GitHub Actions)

Pipeline stages: `build` → `test` → `security-scan` → `docker-build` → `push`

```yaml
# Minimal stage outline
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

## Project Scaffold Checklist

New project minimum viable setup:

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
