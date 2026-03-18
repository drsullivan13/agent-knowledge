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

---

## Factory `Read` may not natively open PDFs larger than 3MB
**Date:** 2026-03-18
**Context:** Factory CLI, local PDF analysis
**Tags:** factory, pdf, read-tool, python, pypdf, workaround

### Problem / Observation

The Factory `Read` tool can open PDFs up to about 3MB natively. When a larger PDF was provided, the tool only reported that the file exceeded the native limit and did not expose the full contents directly.

### Resolution / Insight

If Python is available, use `pypdf` or `PyPDF2` from a one-off `python3` command to extract text from the needed pages. This is a reliable fallback for local PDF inspection when the built-in PDF reader limit is exceeded.

### Commands / Code

```bash
python3 - <<'PY'
from pathlib import Path

pdf = Path('/absolute/path/to/file.pdf')
for mod in ('pypdf', 'PyPDF2'):
    try:
        m = __import__(mod)
        reader = m.PdfReader(str(pdf))
        text = []
        for i, page in enumerate(reader.pages[:20]):
            text.append(f'--- PAGE {i+1} ---\n' + (page.extract_text() or ''))
        print('\n'.join(text)[:30000])
        break
    except Exception as e:
        print(f'{mod} unavailable: {e}')
else:
    print('No PDF reader module available')
PY
```
