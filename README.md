# AI Coding Agent Instructions

Canonical, stack-agnostic AI agent instructions with per-stack overlays. Each project loads **base + exactly one stack** so agent context stays clean — a Flutter session never sees .NET content, and vice versa.

## Repository layout

```
.ai/
  base-instructions.md          ← stack-agnostic conventions (SemVer, Conventional
                                  Commits, TDD, Clean Code, 12-Factor, branching,
                                  git-cliff, Keep a Changelog, UI phase gates)
  stacks/
    dotnet-blazor.md            ← generated: dotnet-core + Blazor layer
    dotnet-webapi.md            ← generated: dotnet-core + WebAPI layer
    flutter.md                  ← single-file overlay (no layer split)
    _partials/
      dotnet-core.md            ← shared .NET conventions (C#, EF, Docker,
                                  logging, Makefile, CI, security, basic
                                  Minimal API + ProblemDetails)
    _layers/
      dotnet-blazor.md          ← Blazor + MudBlazor + bUnit + Playwright
                                  + UI workflow + UI-specific localization
      dotnet-webapi.md          ← REST conventions (versioning, auth, status
                                  codes, idempotency, ETag, rate limiting,
                                  CORS, HTTP logging, LRO, OpenAPI/Scalar,
                                  k6, Kiota, Bruno, integration tests)
  skills/
    commit.md           · push.md
    ui-brainstorm.md    · ui-flow.md · ui-build.md · ui-review.md

scripts/
  build-stacks.sh               ← concatenates _partials/dotnet-core.md +
                                  _layers/dotnet-*.md into the flat
                                  stacks/dotnet-*.md files

.claude/commands/               ← Claude Code slash-command wrappers for the skills above
```

`sync-ai-instructions` and `release-notes` used to live here as `.ai/skills/*.md`; they are now standalone plugins in the `freaxnx01/agent-skills` / `freaxnx01/claude-code-plugins` marketplaces and are available globally once installed.

The flat files under `.ai/stacks/` are the **published** overlays — that is what `sync-ai-instructions` and any direct-fetch consumer pulls. The split source under `_partials/` and `_layers/` exists so the .NET stacks don't duplicate their shared content. A CI check (`build-stacks-drift`) fails any PR where the flat files have drifted from the sources.

New stacks that don't share content with an existing stack (e.g. `flutter.md`, `node.md`) live as a single self-contained file under `.ai/stacks/`. Stacks that share a baseline split into a `_partials/<base>.md` plus one `_layers/<base>-<flavour>.md` per published overlay. Nothing else in the repo changes when a stack is added.

### Breaking change in this revision

The single overlay `dotnet.md` has been renamed to `dotnet-blazor.md`, with its content refactored through a `_partials/dotnet-core.md` + `_layers/dotnet-blazor.md` split so additional .NET flavours can be added without duplicating the shared conventions. Projects that previously ran `/sync-ai-instructions dotnet` should:

1. Run `/sync-ai-instructions dotnet-blazor`
2. Delete the old `.ai/stacks/dotnet.md` from the project — the sync skill won't remove it automatically

## How to use this repo in a project

### Option A — `/sync-ai-instructions` skill (recommended)

Install once:

```
/plugin marketplace add freaxnx01/agent-skills
/plugin install sync-ai-instructions@freax-agent-skills
/reload-plugins
```

Then from inside your target project (not this repo):

```
/sync-ai-instructions dotnet
```

The skill fetches `base-instructions.md`, `stacks/dotnet.md`, and all shared skill files from `main`, assembles them into the target project's `CLAUDE.md`, `.github/copilot-instructions.md`, `SKILL.md`, and writes the stack overlay to `.ai/stacks/dotnet.md`. The target project ends up with **only** the stack it uses.

Idempotent: safe to run for first-time setup **or** to refresh an already-initialized project. If `$ARGUMENTS` is omitted, the skill lists available stacks and asks. It refuses to fall back silently on a missing stack.

### Option B — clone as template (.NET Blazor only)

This repo's root `CLAUDE.md`, `.github/copilot-instructions.md`, and `SKILL.md` are the pre-assembled rendering for a .NET Blazor project. If you want that stack, cloning the repo gives you a working starting point. Fill in the TODO markers in `CLAUDE.md` (project name, purpose) and you're done.

## Supported stacks

| Stack | File | Covers |
|---|---|---|
| `dotnet-blazor` | `.ai/stacks/dotnet-blazor.md` | .NET 10 · ASP.NET Core · Blazor + MudBlazor · EF Core · xUnit / bUnit / Playwright · Serilog + OpenTelemetry · Alpine Docker |
| `dotnet-webapi` | `.ai/stacks/dotnet-webapi.md` | .NET 10 · ASP.NET Core REST API · Asp.Versioning.Http · ProblemDetails · OpenAPI + Scalar · JWT / API key / pass-through auth · `WebApplicationFactory` + Testcontainers · Bruno · k6 · Kiota |
| `flutter` | `.ai/stacks/flutter.md` | Flutter / Dart |

To add a new stack: see *Adding a new stack* below.

## Adding a new stack

### Single-file overlay (no shared baseline)

For a stack that doesn't share content with an existing one, create `.ai/stacks/<name>.md` directly. Cover at minimum:

- Tech-stack table
- Architecture conventions specific to the ecosystem
- Language conventions (style, naming, what to never generate)
- Testing framework choices, project layout, example templates
- UI component-library preferences (if applicable)
- Build / run / test commands
- Container / deployment specifics
- Stack-specific agent guardrails

Keep anything stack-agnostic (SemVer, Conventional Commits, TDD principles, Clean Code, 12-Factor, branching) in `base-instructions.md`, not in the overlay.

### Split overlay (shared baseline + flavour layer)

When a stack has multiple flavours that share substantial content (currently: the .NET family), split the source:

```
.ai/stacks/
  _partials/<base>.md           ← shared content
  _layers/<base>-<flavour>.md   ← per-flavour delta
  <base>-<flavour>.md           ← GENERATED — flat overlay consumers fetch
```

After editing the partial or any layer, run:

```bash
./scripts/build-stacks.sh
```

This concatenates each `_partials/<base>.md` + `_layers/<base>-<flavour>.md` pair into the corresponding `.ai/stacks/<base>-<flavour>.md`. The CI workflow `build-stacks-drift` fails any PR that touches the source files but forgets to commit the regenerated flat files.

**Never edit the generated `.ai/stacks/<base>-<flavour>.md` files directly** — changes will be overwritten on the next regen.

The build script currently handles `dotnet-*` only. Extending it to a second split family (e.g. `node-*`) is a one-line change to its glob.

## Keeping a project in sync

When `base-instructions.md` or the stack overlay changes, consumers re-run `/sync-ai-instructions <stack>` to regenerate their `CLAUDE.md` / copilot / SKILL files. The skill reports the source commit SHA so you know which version of the instructions is in use.
