## Make Codex delegate substantial coding to a Herdr Droid
**Date:** 2026-09-07
**Context:** Codex global instructions, Herdr 0.8.2, Factory Droid 0.213.0
**Tags:** codex, agents-md, herdr, droid, factory, delegation, configuration

### Problem / Observation

Codex should use a user's Factory AI subscription for moderate or significant implementation work, but Herdr agent control is available only when the calling Codex session itself has `HERDR_ENV=1`.

### Resolution / Insight

Put the delegation policy in `~/.codex/AGENTS.md` so it applies globally. Define a concrete threshold, require a sibling Herdr pane and `--kind droid`, prevent overlapping file edits, and make the outside-Herdr behavior explicit. A desktop Codex session with `HERDR_ENV` unset cannot safely control the focused Herdr session; it should tell the user to run or resume Codex inside Herdr before substantial implementation.

### Commands / Code

```sh
test "${HERDR_ENV:-}" = 1
herdr pane split --current --direction right --cwd "$PWD" --no-focus
herdr agent start <unique-name> --kind droid --pane <returned-pane-id>
herdr agent prompt <unique-name> "<bounded implementation task and acceptance criteria>" --wait --timeout 120000
herdr agent read <unique-name> --source recent-unwrapped --lines 120
```

Suggested qualifying threshold: roughly 50 or more changed lines, multiple files, a new feature/module, a non-trivial refactor, or a migration. Review and verify the Droid's changes before reporting completion.
