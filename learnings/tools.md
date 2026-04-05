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

## If a `user-testing-flow-validator` Task produces artifacts but no report, rerun it in report-only mode
**Date:** 2026-04-04
**Context:** Factory Task subagents, mission validation, CLI artifact testing
**Tags:** factory, subagent, task, validation, user-testing, reports, fallback

### Problem / Observation

A `user-testing-flow-validator` Task launched a full CLI validation flow, created the expected isolated output tree and evidence snapshots, but returned `No output received from task subagent.` and never wrote the assigned flow JSON report.

### Resolution / Insight

First inspect whether the validator already produced usable artifacts under its assigned output root and evidence dir. If it did, launch a second `user-testing-flow-validator` Task with a much narrower prompt: forbid rerunning the flow, point it at the existing artifacts only, and ask it to write just the missing flow report plus any derived assertion-check JSON. That recovered the report cleanly without duplicating long-running validation work.

### Commands / Code

```text
Goal:
- Inspect the existing rerun artifacts and write the missing flow report.

Use ONLY these existing paths:
- Output root: /.../data/crypto_research/user-testing/<milestone>/<group>
- Evidence dir: /.../missions/.../evidence/<milestone>/<group>
- Flow report path: /.../.factory/validation/<milestone>/user-testing/flows/<group>.json

Instructions:
- Read the generated backtest summary/cycle ledger, final_evaluation_bundle, and decision_report.
- Determine pass/fail for the assigned assertions.
- Write a complete JSON flow report to the assigned flow report path.
- Do not rerun discovery/collection/strategy/report.
- Do not modify source files.
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

## Crypto backtest hedge validation can use a `python3` inline wrapper when the module CLI lacks strategy-config args
**Date:** 2026-04-04
**Context:** kalshi-agent crypto backtest user testing
**Tags:** kalshi, crypto, backtest, cli, validation, hedging, user-testing

### Problem / Observation

`python3 -m kalshi_agent.crypto_research.backtest` only exposes `--collection-run-manifest`, `--fee-assumptions-path`, and `--output-root`. During user-testing validation we needed to run hedged and mixed-leg hedge scenarios, but the CLI had no flag for passing custom `strategy_configs`.

### Resolution / Insight

Stay on the CLI/artifact surface by using `python3 - <<'PY'` to call `run_backtest(...)` directly with custom `strategy_configs`. If you need a forced mixed-leg hedge case, copy the collection run directory inside the allowed output root, mutate only that copied archive, rewrite the copied `run_manifest.json` artifact paths, and point `run_backtest(...)` at the copied manifest.

### Commands / Code

```bash
python3 - <<'PY'
import json
from pathlib import Path
from kalshi_agent.crypto_research.backtest import run_backtest

outputs = run_backtest(
    collection_run_manifest_path=Path("data/crypto_research/.../collection_runs/<run>/run_manifest.json"),
    fee_assumptions_path=Path("data/crypto_research/.../discovery/fee_assumptions.json"),
    output_root=Path("data/crypto_research/..."),
    strategy_configs=[{
        "configuration_id": "hedged-seed-entry-lte-49",
        "description": "Seed variant with a smaller opposite-side hedge leg.",
        "rule_parameters": {
            "entry_side": "yes",
            "entry_price_lte_cents": 49,
            "hold_minutes": 0,
            "exit_cutoff_minutes_before_close": 1,
            "contracts": 10,
            "hedge_mode": "opposite_side",
            "hedge_ratio": 0.4,
            "allow_settlement_fallback": False,
        },
    }],
)
print(json.dumps(outputs, indent=2, sort_keys=True))
PY
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

---

## Scrutiny can fail when `EndFeatureRun.commitId` points at a later cleanup commit
**Date:** 2026-04-04
**Context:** Factory missions, scrutiny validation, git handoffs
**Tags:** factory, scrutiny, handoff, commit, git, validation

### Problem / Observation

A worker handoff claimed a feature added `kalshi_agent/crypto_research/discovery.py` and `tests/test_crypto_discovery.py`, but the recorded `commitId` in the handoff JSON pointed to a later `.factory/` scaffolding commit instead of the feature diff. Reviewers could not reproduce the claimed verification from the handed-off SHA because `git show --stat <commitId>` contained none of the implementation files.

### Resolution / Insight

During scrutiny, always compare the handoff's `commitId` against the claimed files/tests before trusting the verification narrative. If the SHA only contains cleanup/scaffolding, inspect nearby commits and fail the feature as non-auditable. During feature implementation, keep `EndFeatureRun.commitId` anchored to the commit that actually contains the feature files and tests; do not point it at a later unrelated cleanup commit.

### Commands / Code

```bash
python3 - <<'PY'
import json
from pathlib import Path
handoff = Path("/path/to/handoff.json")
data = json.loads(handoff.read_text())
print(data["commitId"])
print(data["handoff"]["whatWasImplemented"])
PY

git -C /path/to/repo show --stat --summary <handoff-commit>
git -C /path/to/repo show --name-only <handoff-commit>
git -C /path/to/repo log --oneline -n 5
```

---

## Scrutiny reruns should snapshot prior synthesis as `synthesis.roundN.json`
**Date:** 2026-04-04
**Context:** Factory missions, scrutiny validation reruns
**Tags:** factory, scrutiny, validation, synthesis, rerun

### Problem / Observation

On a scrutiny rerun, the current milestone synthesis already lived at `.factory/validation/<milestone>/scrutiny/synthesis.json`. If you overwrite it directly, the new `previousRound` field points at a path that no longer contains the prior round, which breaks auditability for later reruns.

### Resolution / Insight

Before writing the new synthesis, copy the existing file to `.factory/validation/<milestone>/scrutiny/synthesis.round<N>.json` and then overwrite `synthesis.json` with the new round, setting `previousRound` to that archived path. This preserves an auditable chain across scrutiny reruns.

### Commands / Code

```bash
# First rerun after an existing round 1 synthesis
cp .factory/validation/data-collection/scrutiny/synthesis.json \
  .factory/validation/data-collection/scrutiny/synthesis.round1.json

# Then write the new round 2 synthesis with:
# "round": 2
# "previousRound": "/abs/path/to/.factory/validation/data-collection/scrutiny/synthesis.round1.json"
```

---

## Scrutiny reruns can overwrite prior review JSON in place
**Date:** 2026-04-04
**Context:** Factory missions, scrutiny validation reruns
**Tags:** factory, scrutiny, validation, review, rerun, auditability

### Problem / Observation

Even after preserving `synthesis.roundN.json`, a scrutiny rerun can still lose audit history because the reviewer is often told to read the existing `.factory/validation/<milestone>/scrutiny/reviews/<feature>.json` file and then write the new report back to that exact same path. The prior review content disappears as soon as the rerun overwrites it, so `addressesFailureFrom` can end up pointing at a path that no longer contains the previous review.

### Resolution / Insight

Before overwriting a rerun review, archive the previous review JSON to a round-specific filename (for example `reviews/<feature>.round2.json`) and have the new review point `addressesFailureFrom` at that archived file. If the skill is not yet automated, at least record the limitation in synthesis so the orchestrator can update the scrutiny procedure.

### Commands / Code

```bash
git -C /Users/dansullivan/workspace/kalshi-agent show \
  HEAD:.factory/validation/data-collection/scrutiny/reviews/crypto-research-storage-window-and-namespace-fix.json \
  > /Users/dansullivan/workspace/kalshi-agent/.factory/validation/data-collection/scrutiny/reviews/crypto-research-storage-window-and-namespace-fix.round2.json

# Then update the rerun review JSON to reference the archived path:
# "addressesFailureFrom": "/abs/path/to/.../reviews/crypto-research-storage-window-and-namespace-fix.round2.json"
```

---

## If a `scrutiny-feature-reviewer` Task never writes its report, recover from the mission handoff
**Date:** 2026-04-05
**Context:** Factory missions, scrutiny validation, Task subagents
**Tags:** factory, scrutiny, task, subagent, review, handoff, fallback

### Problem / Observation

During `decision-gate` scrutiny, one `scrutiny-feature-reviewer` Task returned only internal reasoning text and never created the requested review JSON under `.factory/validation/<milestone>/scrutiny/reviews/`, even after a tighter retry prompt. That left the milestone missing one required review artifact even though the feature handoff and worker transcript were present in the mission directory.

### Resolution / Insight

If a reviewer Task silently fails to emit its report file, first confirm the file is still missing. Then recover the feature context directly from the mission handoff and worker transcript: extract the worker session's handoff JSON to get the authoritative `commitId`, claimed validation, and feature scope, inspect the relevant commit diff, and write the review JSON manually using the existing scrutiny review schema. This preserves synthesis progress while still recording that the review subagent was spawned.

### Commands / Code

```bash
# 1) Confirm the requested review file was not created
ls .factory/validation/decision-gate/scrutiny/reviews

# 2) Find the feature handoff / transcript by worker session or feature id
rg -n "3e02b010-125a-4e8b-af67-28e84b7f9b00|crypto-end-to-end-lineage-and-guardrails" \
  /Users/dansullivan/.factory/missions/3ad7418a-7c60-4763-8e01-da5b9053b15e

# 3) Inspect the handoff to recover the commit and claimed verification
python3 - <<'PY'
import json
from pathlib import Path
handoff = Path("/Users/dansullivan/.factory/missions/3ad7418a-7c60-4763-8e01-da5b9053b15e/handoffs/2026-04-05T00-30-47-728Z__crypto-end-to-end-lineage-and-guardrails__3e02b010-125a-4e8b-af67-28e84b7f9b00.json")
data = json.loads(handoff.read_text())
print(data["commitId"])
print(data["handoff"]["salientSummary"])
PY

# 4) Review the actual diff before writing the fallback JSON
git -C /Users/dansullivan/workspace/kalshi-agent show --stat bc597621c84a6f02ded6ae216bd5306281e0ae7b
git -C /Users/dansullivan/workspace/kalshi-agent show bc597621c84a6f02ded6ae216bd5306281e0ae7b -- \
  kalshi_agent/crypto_research/backtest.py \
  kalshi_agent/crypto_research/strategy_research.py \
  tests/test_crypto_pipeline_lineage.py

# 5) Then write .factory/validation/<milestone>/scrutiny/reviews/<feature>.json manually
# using the same schema as neighboring scrutiny review files.
```

---

## Crypto collection/manual checks must use canonical `data/crypto_research` output roots
**Date:** 2026-04-05
**Context:** kalshi-agent/python crypto research CLI validation
**Tags:** crypto, collection, namespace, output-root, validation, cli

### Problem / Observation

Running `run_collection(...)` with a temporary directory outside the repository (for example under `/var/.../tmp`) fails with:
`ValueError: output_root must be under canonical data/crypto_research namespace`.

### Resolution / Insight

For ad-hoc/manual checks, place temporary outputs under the repo’s canonical namespace (for example `data/crypto_research/tmp-...`) or monkeypatch `_canonical_crypto_research_root` in tests. The namespace guard is strict by design and rejects paths merely named `data/crypto_research` outside the repo root.

### Commands / Code

```bash
# Fails: outside canonical namespace
python3 - <<'PY'
from pathlib import Path
import tempfile
from kalshi_agent.crypto_research.collection import run_collection
run_collection(output_root=Path(tempfile.mkdtemp())/"data"/"crypto_research")
PY

# Works: under repo-root canonical namespace
python3 - <<'PY'
from pathlib import Path
import uuid
from kalshi_agent.crypto_research.collection import run_collection
root = Path("/Users/dansullivan/workspace/kalshi-agent/data/crypto_research")/f"tmp-inspect-{uuid.uuid4().hex[:8]}"
root.mkdir(parents=True, exist_ok=True)
run_collection(
    output_root=root,
    window_start_utc="2026-04-01T00:00:00+00:00",
    window_end_utc="2026-04-01T01:00:00+00:00",
)
PY
```

---

## Force-add ignored `data/` discovery scan captures used as runtime inputs
**Date:** 2026-04-05
**Context:** kalshi-agent structural-mispricing discovery auditability
**Tags:** git, gitignore, data, structural-mispricing, discovery, scan-capture

### Problem / Observation

`run_discovery()` was refactored to require an archived scan capture under `data/structural_mispricing/...`, but the repo-level `.gitignore` has `data/` ignored. New scan capture files looked present locally, tests passed, and `git status` stayed clean unless you explicitly checked ignore rules.

### Resolution / Insight

When a mission runtime input must live under `data/`, confirm ignore behavior with `git check-ignore` and force-stage the file with `git add -f`. Otherwise the commit will compile locally but fail on fresh clones because the default runtime input file is missing.

### Commands / Code

```bash
git -C /Users/dansullivan/workspace/kalshi-agent check-ignore -v \
  data/structural_mispricing/discovery/scan_captures/public_kalshi_discovery_scan.json

git -C /Users/dansullivan/workspace/kalshi-agent add -f \
  data/structural_mispricing/discovery/scan_captures/public_kalshi_discovery_scan.json
```

---

## Scrutiny reviewer subagents may need explicit allowed files and commands to avoid permission aborts
**Date:** 2026-04-05
**Context:** Factory scrutiny validation, Task subagents
**Tags:** factory, scrutiny, task, subagent, permissions, prompts

### Problem / Observation

A `scrutiny-feature-reviewer` Task for a rerun fix exited immediately with `Exec ended early: insufficient permission to proceed. Re-run with --skip-permissions-unsafe.` when the prompt broadly asked it to review the feature, inspect prior failures, and write a JSON report.

### Resolution / Insight

Retry with a tightly scoped prompt that enumerates the exact files it may read, the exact read-only `git show` commands it may run, and the single review JSON path it may create. Narrowing the prompt to those concrete inputs/outputs let the reviewer complete and emit the report without needing unsafe permissions.

### Commands / Code

```text
Allowed scope only:
- Read these files:
  - /Users/dansullivan/.factory/missions/.../handoffs/...__structural-mispricing-discovery-auditability-fix__....json
  - /Users/dansullivan/workspace/kalshi-agent/.factory/validation/universe-discovery/scrutiny/synthesis.json
  - /Users/dansullivan/workspace/kalshi-agent/.factory/validation/universe-discovery/scrutiny/reviews/structural-mispricing-universe-scan-and-taxonomy.json
  - /Users/dansullivan/workspace/kalshi-agent/.factory/validation/universe-discovery/scrutiny/reviews/structural-mispricing-discovery-lineage-and-screen-eligibility.json
- Run only these git read-only commands:
  - git -C /Users/dansullivan/workspace/kalshi-agent show --stat 611df980cfe72533692953e36631091bdf7cf956
  - git -C /Users/dansullivan/workspace/kalshi-agent show 611df980cfe72533692953e36631091bdf7cf956 -- kalshi_agent/structural_mispricing/discovery.py tests/test_structural_mispricing_discovery.py data/structural_mispricing/discovery/scan_captures/public_kalshi_discovery_scan.json
  - git -C /Users/dansullivan/workspace/kalshi-agent show --stat 392e67c
  - git -C /Users/dansullivan/workspace/kalshi-agent show --stat 13828969463e073a03a734ad7d970a747f530141
- Create exactly one file:
  - /Users/dansullivan/workspace/kalshi-agent/.factory/validation/universe-discovery/scrutiny/reviews/structural-mispricing-discovery-auditability-fix.json
```

---

## Fallback comparator captures may map to source-selection inventory under a different source role
**Date:** 2026-04-05
**Context:** Factory user-testing flow validator, structural-mispricing collection provenance checks
**Tags:** factory, user-testing, structural-mispricing, collection, provenance, fallback, artifacts

### Problem / Observation

While validating `VAL-DATA-006`, a downgraded comparator capture (`selected_fallback`) looked like a provenance failure when the check joined raw captures to `source_selection_inventory` on `(family_id, source_id, source_role, source_url)`. The archived capture kept `source_role="reference_value"`, but the candidate inventory exposed the same fallback source as `source_role="fallback"`.

### Resolution / Insight

For provenance checks, match fallback captures to source-selection candidates on `family_id + source_id + source_url` (or otherwise allow either the business role or `fallback`) and then verify `sampled_at_utc`, raw-capture hash, and normalized-row lineage. A strict role-equality join can create false negatives even when the archived fallback provenance is complete.

### Commands / Code

```python
selection_candidates = {}
for family_row in source_selection_inventory["family_source_rows"]:
    for candidate in family_row["comparator_candidates"]:
        selection_candidates.setdefault(
            (family_row["family_id"], candidate["source_id"], candidate["source_url"]),
            [],
        ).append(candidate)

candidate_matches = selection_candidates.get(
    (capture["family_id"], capture["source_id"], capture["source_url"]),
    [],
)
selection_sampled_at_present = any(match["sampled_at_utc"] for match in candidate_matches)
```

---

## Structural screen flow reports can be built from run-scoped stale/thin ledgers plus screen summary
**Date:** 2026-04-05
**Context:** Factory user-testing flow validator, structural-mispricing screens
**Tags:** factory, user-testing, structural-mispricing, screens, flow-report, artifacts

### Problem / Observation

For structural-screen flow validation, the assigned assertions (`VAL-SCREEN-007/008/009/010/017`) had to be proven from the CLI/artifact surface only. The run produced one accepted stale-repricing row but zero accepted thin-book rows, so the validator needed a deterministic way to distinguish “fail-closed, still valid” from “screen never emitted the required audit fields.”

### Resolution / Insight

Use the run-scoped artifacts only: `screen_run_manifest.json`, `screen_summary.json`, `stale_repricing_ledger.json`, `thin_book_overshoot_ledger.json`, `screen_lineage_manifest.json`, and `screen_denominators.jsonl`. Validate accepted stale rows for anchor/timing fields directly from `stale_repricing_ledger.json`. For thin-book, treat zero accepted candidates as a pass only when every row stays non-accepted with explicit `reason_code` / `non_tradeable_reason` and preserved anchor lineage (for example `related_contract_anchor` + `related_contract_ids`) instead of disappearing from denominators.

### Commands / Code

```bash
python3 -m kalshi_agent.structural_mispricing.screens \
  --collection-run-manifest "/Users/dansullivan/workspace/kalshi-agent/data/structural_mispricing/collection_runs/<run_id>/run_manifest.json" \
  --output-root "/Users/dansullivan/workspace/kalshi-agent/data/structural_mispricing/user-testing/structural-screens/<group>"
```

```python
manifest = json.loads((run_dir / "screen_run_manifest.json").read_text())
summary = json.loads((run_dir / "screen_summary.json").read_text())
stale = json.loads((run_dir / "stale_repricing_ledger.json").read_text())
thin = json.loads((run_dir / "thin_book_overshoot_ledger.json").read_text())

accepted_stale = [row for row in stale["rows"] if row["status"] == "accepted_candidate"]
assert all(
    accepted_stale[0][field] is not None
    for field in [
        "anchor_type",
        "anchor_source_id",
        "anchor_event_id",
        "anchor_timestamp",
        "pre_anchor_market_timestamp",
        "post_anchor_market_timestamp",
        "repricing_lag_ms",
    ]
)

thin_rows = thin["rows"]
assert all(row["status"] == "skipped_window" for row in thin_rows)
assert all(row["reason_code"] == "insufficient_depth_snapshot" for row in thin_rows)
assert all(row["anchor_source_id"] == "related_contract_anchor" for row in thin_rows)
assert all(row["related_contract_ids"] for row in thin_rows)
```
