## Herdr and OpenCode version-aware configuration
**Date:** 2026-09-05
**Context:** Herdr 0.8.2; OpenCode stable v1.18.29 guide
**Tags:** herdr, opencode, plugins, version, config, validation

### Problem / Observation
OpenCode stable v1 and v2 preview docs expose different config and plugin APIs. Issue #43742 reports a Herdr named-export plugin failing in an August 21 v2 preview with a missing default export. This is a report for that build, not proof about every future preview.
Herdr 0.8.2 `config check` printed an unknown-key diagnostic for `ui.sound.agents.opencode` while still exiting zero. Trust diagnostic text as well as status.

### Resolution / Insight
Verify current stable releases before writing config. Use v1 `permission`/`bash`/agent `prompt` fields for v1; do not mix v2 examples. Install `herdr integration install opencode` plus binary-matched `herdr --skill` under OpenCode's native global skills path. The plugin reports lifecycle and session identity, unlike Droid's session-only integration. Use general `[ui.sound] enabled = true` in the checked baseline instead of the rejected per-agent key.

### Commands / Code
```sh
mkdir -p ~/.config/opencode/skills/herdr
herdr integration install opencode
herdr --skill > ~/.config/opencode/skills/herdr/SKILL.md
herdr integration status
HERDR_CONFIG_PATH=/absolute/path/to/candidate.toml herdr config check
# Require output: config: ok; exit status alone was insufficient.
```
Guide: ~/Documents/Codex/herdr-opencode/README.md.
Sources: https://herdr.dev/docs/integrations/#opencode ; https://github.com/anomalyco/opencode/issues/43742 ; https://opencode.ai/docs/skills/ .
