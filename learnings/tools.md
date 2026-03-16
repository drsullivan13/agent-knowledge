# Tool & CLI Knowledge

Entries are appended by agents as they discover useful tool flags, configs, and CLI tricks.

---

## Use project venv python when pyenv blocks `python3`
**Date:** 2026-03-15
**Context:** Python, pyenv, homework repos with `.python-version`
**Tags:** python, pyenv, venv, path, cli

### Problem / Observation

Running `python3` from a repo failed with:
`pyenv: version '3.14' is not installed` because the repo had a `.python-version` pin.

### Resolution / Insight

Bypass pyenv shim resolution and invoke the repo virtualenv interpreter directly.
This avoids version-manager mismatch and reliably runs project scripts.

### Commands / Code

```bash
"/absolute/path/to/project/.venv/bin/python" - <<'PY'
import sys
print(sys.version)
PY
```

---

## `.factory/init.sh` may append Terraform ignores into `.gitignore`
**Date:** 2026-03-16
**Context:** Factory mission setup scripts in kalshi-agent
**Tags:** git, init-script, gitignore, terraform, factory

### Problem / Observation

Running the repo's `.factory/init.sh` can modify `.gitignore` by appending Terraform patterns (`*.tfstate`, `.terraform/`, `terraform.tfvars`) when they are missing. This creates unrelated working-tree diffs before feature work starts.

### Resolution / Insight

After running init, check `git status` and explicitly `git restore .gitignore` if that diff is unrelated to your assigned feature. Keep only scoped feature changes in your commit.

### Commands / Code

```bash
git -C /path/to/repo status --short
git -C /path/to/repo restore .gitignore
```
