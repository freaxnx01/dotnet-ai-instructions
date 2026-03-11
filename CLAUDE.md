[//]: # (Source of truth: .ai/base-instructions.md — update conventions there first, then reflect changes here)

# CLAUDE.md

Agent context for Claude Code. Read this before taking any action in this repository.

---

## Project Overview

<!-- TODO: Fill in per project -->
**Name:** `<project-name>`  
**Purpose:** `<one-line description>`  
**Architecture:** Modular Monolith (Hexagonal within modules where needed)  
**Status:** `<active development / maintenance>`

---

## Essential Commands

### Build & Run

```bash
# Restore dependencies
dotnet restore

# Build (warnings as errors)
dotnet build -c Release

# Run locally (with override for dev DB/ports)
docker-compose -f docker-compose.yml -f docker-compose.override.yml up --build

# Run host directly
dotnet run --project src/Host
```

### Testing

```bash
# All tests
dotnet test

# Unit tests only
dotnet test tests/<Module>.UnitTests

# Integration tests (requires Docker for Testcontainers)
dotnet test tests/<Module>.IntegrationTests

# Blazor component tests
dotnet test tests/<Module>.ComponentTests

# E2E (requires running stack)
docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d
dotnet test tests/E2E
docker-compose down

# With coverage
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
```

### Database Migrations

```bash
# Add migration (replace <Module> and <MigrationName>)
dotnet ef migrations add <MigrationName> \
  --project src/Modules/<Module>/Infrastructure \
  --startup-project src/Host

# Apply to local DB
dotnet ef database update \
  --project src/Modules/<Module>/Infrastructure \
  --startup-project src/Host

# Generate SQL script (for production review)
dotnet ef migrations script \
  --project src/Modules/<Module>/Infrastructure \
  --startup-project src/Host \
  --output migrations.sql
```

### Security & Package Checks

```bash
# Check for vulnerable packages (fail on high/critical)
dotnet list package --vulnerable --fail-on-severity high

# Outdated packages
dotnet list package --outdated
```

---

## Repository Structure

```
.
├── src/
│   ├── Modules/
│   │   └── <ModuleName>/
│   │       ├── Domain/
│   │       ├── Application/
│   │       │   ├── Ports/
│   │       │   │   ├── Driving/
│   │       │   │   └── Driven/
│   │       │   └── UseCases/
│   │       └── Infrastructure/
│   │           └── Persistence/
│   │               └── Migrations/
│   ├── Shared/
│   └── Host/                    ← ASP.NET Core entry point
├── tests/
│   ├── <Module>.UnitTests/
│   ├── <Module>.IntegrationTests/
│   ├── <Module>.ComponentTests/  ← bUnit
│   └── E2E/                      ← Playwright
├── .ai/
│   └── base-instructions.md      ← canonical conventions reference
├── .github/
│   ├── copilot-instructions.md
│   └── workflows/
├── docker-compose.yml
├── docker-compose.override.yml
├── Directory.Build.props
├── Directory.Packages.props
├── global.json
├── CLAUDE.md                     ← this file
└── SKILL.md                      ← OpenClaw
```

---

## Architecture Decisions

### Modular Monolith

- Each module is self-contained: Domain, Application, Infrastructure
- Cross-module communication: in-process interfaces defined in `src/Shared/`
- No direct project references between modules
- Modules register their own DI services via `IServiceCollection` extension methods

### Hexagonal Architecture (within modules)

Apply when a module has multiple infrastructure adapters or needs strong testability isolation.

- Driving (inbound) ports: what the outside world calls into the module
- Driven (outbound) ports: what the module calls out to (DB, messaging, HTTP)
- Adapters live in `Infrastructure/Adapters/`

### API

- Minimal API endpoints, registered per module
- FluentValidation at the boundary — domain stays clean
- ProblemDetails (RFC 9457) for all errors
- OpenAPI via `Microsoft.AspNetCore.OpenApi`, Scalar UI at `/scalar`

### Blazor + MudBlazor

- MudBlazor only — no other component libraries
- CSR for full SPA scenarios, SSR for auth-heavy or SEO-critical pages
- bUnit for component testing in isolation

---

## Testing Rules — Non-Negotiable

1. **Write the failing test first** — then implement
2. **Never modify a test to make it green** — fix the implementation
3. **No shortcuts**: no `// TODO: test later`, no empty test bodies
4. Test naming: `MethodName_StateUnderTest_ExpectedBehavior`
5. E2E tests must be idempotent — seed and clean up their own data

---

## Environment Variables

| Variable | Description | Required |
|---|---|---|
| `ConnectionStrings__Default` | DB connection string | Yes |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OpenTelemetry collector | No (local) |
| `Serilog__MinimumLevel` | Log level override | No |

Never add secrets to `appsettings.json`. Use environment variables or Docker secrets.

---

## Docker

```bash
# Build image
docker build -t <image-name>:local .

# Start full stack (local)
docker-compose -f docker-compose.yml -f docker-compose.override.yml up --build

# Stop and clean volumes
docker-compose down -v
```

- Runtime base: `mcr.microsoft.com/dotnet/aspnet:10.0-alpine`
- Build base: `mcr.microsoft.com/dotnet/sdk:10.0-alpine`
- Runs as non-root user (`appuser`)

---

## Versioning

This project follows [SemVer 2.0.0](https://semver.org/). Version defined once in `Directory.Build.props`:

```xml
<Version>1.0.0</Version>
```

- Git tag on every release: `v<MAJOR>.<MINOR>.<PATCH>`
- Docker images tagged with same version + `latest` on stable
- Conventional Commits drive the bump: `feat` → MINOR · `fix`/`perf` → PATCH · `BREAKING CHANGE:` footer → MAJOR

```bash
# Tag a release
git tag -a v1.2.0 -m "release: v1.2.0"
git push origin v1.2.0

# Generate changelog with git-cliff
git cliff --output CHANGELOG.md
```

---

## Changelog

`CHANGELOG.md` in repo root following [Keep a Changelog](https://keepachangelog.com) format.

- `[Unreleased]` section accumulates changes until a release is cut
- Auto-generated via `git-cliff` from Conventional Commits (`cliff.toml` in repo root)
- CI blocks release branches if `[Unreleased]` is empty

---

## 12-Factor Compliance

See [12factor.net](https://www.12factor.net/). Critical rules for this repo:

- **Config (III):** All env-specific config via environment variables — nothing per-environment in `appsettings.json`
- **Logs (XI):** Serilog writes to **stdout only** in Docker — no file sinks inside containers
- **Processes (VI):** Stateless app — no local file state, no sticky sessions
- **Migrations (XII):** EF Core migrations run as a separate init container or pre-deploy step — **never** auto-migrate inside `app.Run()`
- **Build/Release/Run (V):** Multi-stage Docker enforces separation — never build inside a running container
- **Backing services (IV):** DB, cache, messaging treated as attached resources via env var connection strings

---

## Branching & Git

- Branch from `main`, PR back to `main`
- Squash or rebase merge — no merge commits
- Delete branch after merge

### Commit Messages (Conventional Commits)

```
feat(orders): add cancellation endpoint
fix(auth): handle expired token edge case
test(catalog): add handler unit tests
refactor(shared): extract correlation ID middleware
```

Types: `feat` `fix` `test` `refactor` `chore` `docs` `ci` `perf`

---

## Common Pitfalls — Avoid These

- `Task.Result` / `.GetAwaiter().GetResult()` — always `await`
- `async void` outside Blazor event handlers
- Magic strings — use `const` or `nameof()`
- Direct `HttpClient` instantiation — use `IHttpClientFactory`
- Suppressions of nullable warnings with `!` without a clear comment
- Cross-module project references — use shared interfaces
- Secrets in source files or appsettings

---

## Key Dependencies (from Directory.Packages.props)

<!-- Update versions as packages are updated in the project -->

| Package | Purpose |
|---|---|
| `FluentValidation.AspNetCore` | Input validation |
| `FluentAssertions` | Test assertions |
| `NSubstitute` | Mocking |
| `xunit` | Test framework |
| `bunit` | Blazor component testing |
| `Microsoft.Playwright` | E2E testing |
| `MudBlazor` | UI component library |
| `Serilog.AspNetCore` | Structured logging |
| `OpenTelemetry.AspNetCore` | Traces + metrics |
| `Microsoft.EntityFrameworkCore` | ORM |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | PostgreSQL driver |

---

## Health Endpoints

| Endpoint | Purpose |
|---|---|
| `/health/live` | Liveness (always 200 if process up) |
| `/health/ready` | Readiness (checks DB, dependencies) |
| `/scalar` | API documentation |
| `/metrics` | Prometheus metrics |
