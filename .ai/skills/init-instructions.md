# Init Instructions — Assemble AI agent files for this project

Pull the canonical AI agent instructions from `github.com/freaxnx01/ai-instructions` and assemble them into this project as `CLAUDE.md`, `.github/copilot-instructions.md`, `SKILL.md`, plus the shared skill files under `.ai/skills/`.

**Run this skill from the target project's working directory**, not from the `ai-instructions` repo itself.

**Target:** $ARGUMENTS  (optional: stack name, e.g. `dotnet`)

---

## Source repository layout

The source repository is `github.com/freaxnx01/ai-instructions`. Its shape:

```
.ai/
  base-instructions.md                    ← stack-agnostic
  stacks/
    <stack>.md                             ← one file per supported stack
  skills/
    commit.md · push.md · release-notes.md
    ui-brainstorm.md · ui-flow.md · ui-build.md · ui-review.md
```

A project consumes **base + exactly one stack overlay**. That is what keeps an agent's context clean: a Flutter project never sees .NET content, and vice versa.

---

## Steps

### Step 1 — Choose the stack

Resolve the stack in this order:

1. **If `$ARGUMENTS` names a stack**, use that. Verify it exists in the source repo (see available-stacks command below); stop if it doesn't.
2. **Otherwise, auto-detect from disk**: list `.ai/stacks/*.md` in the target project.
   - Exactly one file → use its basename (e.g. `dotnet.md` → `dotnet`) **silently**, as an update of an existing init.
   - Zero files → this is a first-time init; fall through to step 3.
   - More than one file → stop and ask the user to remove the stale ones. A project should carry exactly one stack overlay.
3. **Still nothing resolved** (first-time init, no argument): fetch the list of available stacks and ask the user which one to use.

Available-stacks command (use either):
- `gh api repos/freaxnx01/ai-instructions/contents/.ai/stacks --jq '.[].name'`
- `curl -s https://api.github.com/repos/freaxnx01/ai-instructions/contents/.ai/stacks | jq -r '.[].name'`

If the user (or `$ARGUMENTS`) names a stack that does not exist in the source repo, stop and tell them — do not silently fall back. Offer: "Create `stacks/<name>.md` in the ai-instructions repo first, then re-run."

When resolving by auto-detect, confirm in the report which stack was picked and from where (e.g. "detected `dotnet` from `.ai/stacks/dotnet.md`"), so the user can correct if it's wrong.

### Step 2 — Fetch the raw files

For the chosen stack `<name>`, fetch these raw files from `main`:

- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/base-instructions.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/stacks/<name>.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/skills/commit.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/skills/push.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/skills/release-notes.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/skills/ui-brainstorm.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/skills/ui-flow.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/skills/ui-build.md`
- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/skills/ui-review.md`

Use `curl -sSfL -o <path> <url>` or WebFetch.

### Step 3 — Write files into the target project

Create these files in the current working directory (the target project):

```
.ai/
  base-instructions.md                    ← copy of fetched base
  stacks/
    <stack>.md                             ← only the chosen stack
  skills/
    commit.md push.md release-notes.md
    ui-brainstorm.md ui-flow.md ui-build.md ui-review.md

.claude/
  commands/
    commit.md push.md release-notes.md
    ui-brainstorm.md ui-flow.md ui-build.md ui-review.md
    init-instructions.md                  ← this skill, so it can be re-run in the target

CLAUDE.md                                 ← assembled: base + stacks/<stack>.md (see Step 4)
.github/copilot-instructions.md           ← same assembled content, tool-specific framing
SKILL.md                                  ← same assembled content, OpenClaw framing
```

The target project gets exactly **one stack** under `.ai/stacks/`. That is the whole point — no other stacks land on disk, so they never enter any agent's context.

### Step 4 — Assemble CLAUDE.md / copilot / SKILL

Each of these three files is the concatenation of:

1. A short tool-specific header (see below)
2. The full contents of `base-instructions.md`
3. The full contents of `stacks/<stack>.md`

Headers:

- **`CLAUDE.md`**
  ```markdown
  [//]: # (Source of truth: .ai/base-instructions.md + .ai/stacks/<stack>.md — update those, then regenerate this file by re-running /init-instructions)

  # CLAUDE.md

  Agent context for Claude Code. Read this before taking any action in this repository.
  ```

- **`.github/copilot-instructions.md`**
  ```markdown
  [//]: # (Source of truth: .ai/base-instructions.md + .ai/stacks/<stack>.md — update those, then regenerate by re-running /init-instructions)

  # GitHub Copilot Instructions

  Follow all conventions below when generating or completing code.
  ```

- **`SKILL.md`**
  ```markdown
  [//]: # (Source of truth: .ai/base-instructions.md + .ai/stacks/<stack>.md — update those, then regenerate by re-running /init-instructions)

  # SKILL.md — OpenClaw Agent Skill

  This skill configures OpenClaw for this project.
  ```

### Step 5 — Add the `[Unreleased]` scaffolding if missing

If the target project has no `CHANGELOG.md`, create one with the Keep a Changelog header and an empty `[Unreleased]` section. If it has no `cliff.toml`, suggest running `git cliff --init` separately.

### Step 6 — Report

Print a summary of files written, the stack chosen, and the commit SHA of `ai-instructions` that was fetched (from `gh api repos/freaxnx01/ai-instructions/commits/main --jq .sha`). The user will commit the changes themselves — do not commit automatically.

---

## Rules

- Never fetch and write a stack the user didn't confirm
- Never write files outside the current working directory
- Never overwrite `CLAUDE.md` / `copilot-instructions.md` / `SKILL.md` without showing a diff first if they already exist
- If the fetch fails (network, missing stack), stop and report — do not fall back to stale local copies
- Do not commit the changes — leave that to the user
