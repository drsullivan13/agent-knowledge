## Make Codex delegate substantial coding to a Herdr Droid
**Date:** 2026-09-07
**Context:** Codex global instructions, Herdr 0.8.2, Factory Droid 0.213.0
**Tags:** codex, agents-md, herdr, droid, factory, delegation, configuration

### Problem / Observation

Codex should use a user's Factory AI subscription for moderate or significant implementation work. Herdr agent control is available only when the calling Codex session itself has `HERDR_ENV=1`, but Factory's CLI also provides a non-interactive execution path for outside-Herdr sessions.

### Resolution / Insight

Put the delegation policy in `~/.codex/AGENTS.md` so it applies globally. Define a concrete threshold, use a sibling Herdr pane and `--kind droid` inside Herdr, and use `droid exec --auto medium --cwd <project> <prompt>` outside Herdr. Prevent overlapping file edits and require Codex to review and verify the delegated result. Prefer shell execution over opening Ghostty because it is non-interactive, captures output directly, and avoids UI overhead.

### Commands / Code

```sh
test "${HERDR_ENV:-}" = 1
herdr pane split --current --direction right --cwd "$PWD" --no-focus
herdr agent start <unique-name> --kind droid --pane <returned-pane-id>
herdr agent prompt <unique-name> "<bounded implementation task and acceptance criteria>" --wait --timeout 120000
herdr agent read <unique-name> --source recent-unwrapped --lines 120

# Outside Herdr:
droid exec --auto medium --cwd "$PWD" "<bounded implementation task and acceptance criteria>"
```

Suggested qualifying threshold: roughly 50 or more changed lines, multiple files, a new feature/module, a non-trivial refactor, or a migration. Review and verify the Droid's changes before reporting completion.
