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

---

## `data/` gitignore pattern can hide new files under tracked `kalshi_agent/data/`
**Date:** 2026-03-21
**Context:** kalshi-agent Python feature work
**Tags:** git, gitignore, tracked-dirs, add-force, diagnostics

### Problem / Observation

The repo `.gitignore` had a broad `data/` rule. Existing files under `kalshi_agent/data/` were already tracked, but a new file added under that tree (`kalshi_agent/data/weather/forecast_snapshots.py`) was silently ignored and did not appear in `git status --short`.

### Resolution / Insight

Use `git check-ignore -v <path>` whenever a newly created file is missing from status output. If the file is intentionally needed, stage it with `git add -f <path>`.

### Commands / Code

```bash
git -C /path/to/repo check-ignore -v kalshi_agent/data/weather/forecast_snapshots.py
git -C /path/to/repo add -f kalshi_agent/data/weather/forecast_snapshots.py
git -C /path/to/repo status --short
```

---

## Backtest cache SQLite tables use `event_date` (not `date`)
**Date:** 2026-03-21
**Context:** kalshi-agent cache verification
**Tags:** sqlite, backtest-cache, schema, diagnostics

### Problem / Observation

Ad-hoc SQL checks against `data/backtest_cache.db` failed with `sqlite3.OperationalError: no such column: date` when querying cache coverage using `MIN(date), MAX(date)`.

### Resolution / Insight

The cache tables (`events`, `candles`, `orderbooks`, `forecast_snapshots`) store coverage under the `event_date` column. Use `event_date` for min/max and span checks.

### Commands / Code

```bash
python3 - <<'PY'
import sqlite3
conn = sqlite3.connect('data/backtest_cache.db')
cur = conn.cursor()
for table in ('events','candles','orderbooks','forecast_snapshots'):
    cur.execute(f"SELECT COUNT(*), MIN(event_date), MAX(event_date) FROM {table}")
    print(table, cur.fetchone())
conn.close()
PY
```

---

## Ignored `data/` paths can block staging modified tracked artifacts
**Date:** 2026-03-21
**Context:** kalshi-agent walkforward artifact refresh
**Tags:** git, gitignore, tracked-files, data, add-force

### Problem / Observation

`git add data/walkforward_results*.json` failed with `The following paths are ignored by one of your .gitignore files: data` even though the files were already tracked and modified.

### Resolution / Insight

When `data/` is ignored broadly, force-add (`-f`) is the reliable way to stage these artifact files explicitly.

### Commands / Code

```bash
git -C /path/to/repo status --short
git -C /path/to/repo add -f data/walkforward_results.json data/walkforward_results_strict.json data/walkforward_results_moderate.json
git -C /path/to/repo status --short
```

---

## Top-level CLI packages need setuptools include patterns for `pip install -e .`
**Date:** 2026-03-22
**Context:** Python packaging, editable installs, module execution
**Tags:** python, setuptools, editable-install, cli, packaging

### Problem / Observation

A top-level CLI package (`bot/`) worked when running from repo root (`python -m bot`) but can be missing after `pip install -e .` if setuptools package discovery only includes project packages like `kalshi_agent*`.

### Resolution / Insight

Include the CLI package in `[tool.setuptools.packages.find].include` so editable installs expose it reliably in fresh environments.

### Commands / Code

```toml
# pyproject.toml
[tool.setuptools.packages.find]
include = ["kalshi_agent*", "bot*"]
```

```bash
pip install -e /path/to/repo
python3 -m bot paper-probe
```

---

## `docker exec` needs `-i` when piping heredoc scripts into container Python
**Date:** 2026-03-22
**Context:** Docker, EC2 runtime validation, remote scripting over SSH
**Tags:** docker, exec, stdin, heredoc, ssh, ec2

### Problem / Observation

Running `docker exec <container> python - <<'PY' ... PY` over SSH produced no script output and silently returned, even though short `python -c` checks worked.

### Resolution / Insight

`docker exec` does not attach stdin unless `-i` is set. Heredoc-fed scripts require `docker exec -i` so Python receives the script from stdin.

### Commands / Code

```bash
# No script stdin attached (fails silently for heredoc use-cases)
ssh ... "docker exec kalshi-agent python - <<'PY'\nprint('hello')\nPY"

# Correct: attach stdin for heredoc/script piping
ssh ... "docker exec -i kalshi-agent python - <<'PY'\nprint('hello')\nPY"
```

---

## Flow-validator subagents can permission-abort unless prompts are narrowed to explicit read-only commands
**Date:** 2026-03-22
**Context:** Factory Task subagents, mission validation, local CLI testing
**Tags:** factory, subagent, task, permissions, validation, prompts

### Problem / Observation

A `user-testing-flow-validator` subagent exited immediately with `Exec ended early: insufficient permission to proceed. Re-run with --skip-permissions-unsafe.` when given a broad local-validation prompt, even though the intended work was only bounded pytest runs and a one-cycle mock CLI smoke test.

### Resolution / Insight

Rewrite the Task prompt to tightly scope the validator to explicit read-only commands and constraints. Listing the allowed pytest commands, forbidding SSH/background processes/source edits, and limiting execution to bounded mock-mode CLI checks avoided the permission abort and let the validator complete normally.

### Commands / Code

```text
Use only these kinds of commands:
- python3 -m pytest tests/test_scheduler.py -v --tb=short
- python3 -m pytest tests/test_trading_loop.py -v --tb=short -k gas
- python3 -m pytest tests/test_monitoring.py -v --tb=short
- optional bounded smoke command like env KALSHI_MOCK_MODE=true KALSHI_MAX_CYCLES=1 python3 -m kalshi_agent.gas_paper_runner if safe

Constraints:
- Do not modify any source files.
- Do not SSH anywhere.
- Do not start background processes.
- Keep execution bounded and safe.
- KALSHI_MOCK_MODE=true for any runner execution.
```

---

## Docker CLI may fail until Docker Desktop is explicitly started in exec sessions
**Date:** 2026-03-29
**Context:** macOS Factory exec mode, Docker buildx, ECR rollout
**Tags:** docker, docker-desktop, macos, buildx, ecr, deployment

### Problem / Observation

`docker buildx build --platform linux/amd64 ... --push` failed with `Cannot connect to the Docker daemon at unix:///Users/.../.docker/run/docker.sock` even though the Docker CLI binary was installed and `docker --version` succeeded.

### Resolution / Insight

In this environment, Docker Desktop was installed but not running. Starting Docker Desktop from CLI and waiting for `docker info` readiness restored daemon connectivity so buildx/push commands could run.

### Commands / Code

```bash
open -a Docker
for i in $(seq 1 60); do
  docker info >/dev/null 2>&1 && echo "DOCKER_READY" && break
  sleep 2
done

docker buildx build --platform linux/amd64 \
  -t 319025930540.dkr.ecr.us-east-1.amazonaws.com/kalshi-agent:<tag> \
  --push /path/to/repo
```

---

## Use scoped stash for pre-existing `.factory/` mission changes before worker handoff
**Date:** 2026-04-04
**Context:** Factory worker missions in repos with preloaded `.factory/` edits
**Tags:** git, stash, factory, mission, working-tree

### Problem / Observation

Worker sessions can begin with pre-existing modified/untracked files under `.factory/` before feature implementation starts. If you commit only feature files, `git status` remains dirty and violates clean-tree handoff requirements.

### Resolution / Insight

Commit only your feature files first, then stash pre-existing `.factory/` changes with a scoped pathspec so the working tree is clean without mixing unrelated mission scaffolding into the feature commit.

### Commands / Code

```bash
git -C /path/to/repo add <feature-files...>
git -C /path/to/repo commit -m "feat(...): ..."
git -C /path/to/repo stash push -u -m "worker-preexisting-factory-state" -- .factory
git -C /path/to/repo status --short
```
