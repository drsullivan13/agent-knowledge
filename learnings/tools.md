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

---

## `storizzi/notes-exporter` exports all Apple Notes folders; isolate the target folder from output
**Date:** 2026-03-18
**Context:** macOS Apple Notes export to Markdown
**Tags:** apple-notes, macos, markdown, export, applescript, cli

### Problem / Observation

When testing `storizzi/notes-exporter` against Apple Notes on macOS, the exporter processed every Notes folder in the account. It did not expose a simple CLI option to restrict export to one folder such as `SWOL NATION`.

### Resolution / Insight

Use the exporter to produce the full export once, then read or copy only the folder-specific Markdown subdirectory it creates (for example `output/md/iCloud-SWOL-NATION`). This is enough to work with a shared iCloud Notes folder without manually copying individual notes.

### Commands / Code

```bash
# Confirm the target Apple Notes folder exists and count notes in it
osascript <<'APPLESCRIPT'
tell application "Notes"
	set output to {}
	repeat with a in every account
		repeat with f in every folder of a
			if name of f is "SWOL NATION" then
				set end of output to ((name of a as text) & " | " & (name of f as text) & " | notes=" & (count of notes of f as text))
			end if
		end repeat
	end repeat
	return output
end tell
APPLESCRIPT

# Run the exporter to Markdown
PATH="/path/to/venv/bin:$PATH" /path/to/notes-exporter/exportnotes.zsh \
  --root-dir "/path/to/output" \
  --convert-markdown true \
  --update-all true

# Work only with the target folder's exported Markdown
cp -R "/path/to/output/md/iCloud-SWOL-NATION" "/path/to/swol-nation-md"
```

---

## CloudWatch `filter-log-events` with `--limit` returns earliest events unless you bound time
**Date:** 2026-03-18
**Context:** AWS CLI live-validation log checks
**Tags:** aws, cloudwatch, logs, cli, validation

### Problem / Observation

Running `aws logs filter-log-events --log-group-name ... --limit 20` returned old bootstrap errors and missed the newest trading cycles, which made validation look stale.

### Resolution / Insight

Always include a recent `--start-time` (epoch ms) when validating current behavior, then parse JSON cycle payloads from that bounded window.

### Commands / Code

```bash
NOW=$(date -u +%s)
START=$(((NOW-1800)*1000))
aws logs filter-log-events \
  --region us-east-1 \
  --log-group-name /kalshi-agent/trading-loop \
  --start-time "$START" \
  --limit 200 \
  --output json
```
