Assemble AI agent instruction files for this project by pulling base + the chosen stack overlay from `github.com/freaxnx01/ai-instructions`.

Run from the target project's working directory. Follow the full playbook in `.ai/skills/init-instructions.md`.

Context / stack (optional): $ARGUMENTS

## Steps (short)

1. Resolve the stack in this order:
   a. If `$ARGUMENTS` names a stack, use it (verify against source repo).
   b. Else auto-detect from `.ai/stacks/*.md` in the target project — exactly one file → use silently (this is an update). Zero → fall through. More than one → stop and ask the user to clean up.
   c. Else (first-time init, no argument): fetch available stacks via `gh api repos/freaxnx01/ai-instructions/contents/.ai/stacks --jq '.[].name'` and ask the user.
2. If the chosen stack does not exist in the source repo, stop — do not silently fall back.
3. Fetch `base-instructions.md`, `stacks/<stack>.md`, and all files under `skills/` from `main` (raw URLs).
4. Write them into the target project:
   - `.ai/base-instructions.md`
   - `.ai/stacks/<stack>.md` (only the chosen stack)
   - `.ai/skills/*.md`
   - `.claude/commands/*.md` (mirror of skills)
   - `CLAUDE.md`, `.github/copilot-instructions.md`, `SKILL.md` — assembled as base + stacks/<stack>.md with a tool-specific header each
5. Create `CHANGELOG.md` with an empty `[Unreleased]` section if it doesn't exist.
6. Report files written, stack chosen, and the source commit SHA. Do not auto-commit.

## Rules

- Target project gets exactly one stack under `.ai/stacks/` — never all of them
- Show a diff before overwriting existing `CLAUDE.md` / copilot / `SKILL.md`
- Never commit on the user's behalf — they review first
- If the user asks for a stack that doesn't exist yet, stop and suggest: "Add `stacks/<name>.md` to `freaxnx01/ai-instructions` first, then re-run."
