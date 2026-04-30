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
