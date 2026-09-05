## Herdr and Droid integration setup
**Date:** 2026-09-05
**Context:** macOS, Herdr 0.8.2, Droid 0.213.0
**Tags:** herdr, droid, factory, hooks, skills, terminal, configuration

### Problem / Observation
Herdr runs Droid without integration, but native conversation restoration needs the official SessionStart hook. Pi lifecycle reporting does not apply to Droid. Web search can return old forks and incompatible CLI syntax.

### Resolution / Insight
Install the bundled version-matched skill globally under ~/.factory/skills/herdr/SKILL.md and the official Droid integration. This binary installs v3, which reports only session identity. Agent state remains screen-detected. Agent start needs an existing shell pane. Use the installed binary as the syntax authority. Structurally compare parsed settings rather than JSON.stringify strings: the installer reorders nested object keys without changing values.

### Commands / Code
```sh
herdr --version
herdr --default-config
mkdir -p ~/.factory/skills/herdr
herdr --skill > ~/.factory/skills/herdr/SKILL.md
herdr integration install droid
herdr integration status
herdr config check
herdr server reload-config
# Inside Herdr; use an actual pane ID from a split/create response:
herdr agent start reviewer --kind droid --pane PANE_ID
```

Config and workflow guide: ~/.config/herdr/config.toml and ~/.config/herdr/DROID-WORKFLOW.md. Official sources: https://herdr.dev/docs/integrations/#droid and https://docs.factory.ai/harness/skills .
