# Useful Code Patterns

Entries are appended by agents as they discover effective patterns.

---

## Export walk-forward promoted RiskConfig directly to trading-loop file
**Date:** 2026-03-15
**Context:** kalshi-agent/python/backtest-live bridge
**Tags:** kalshi, walkforward, promotion, risk-config, trading-loop, integration-tests

### Problem / Observation

Walk-forward output could indicate `gate.status="promote"`, but there was no built-in helper to materialize the winning `RiskConfig` into `config/risk_params.promoted.yaml` for the trading loop to auto-load.

### Resolution / Insight

Add a probe-level helper that selects the best promoted candidate (`best_candidate.promoted`) by validation trade count + score and writes its dataclass fields as simple YAML key/value lines. Then expose CLI flags to optionally persist this file during walk-forward runs.

### Commands / Code

```python
def write_promoted_risk_config(*, result: object, output_path: Path) -> dict[str, object] | None:
    gate = getattr(result, "gate", None)
    if str(getattr(gate, "status", "")).lower() != "promote":
        return None

    candidate = _pick_promoted_candidate(result)  # promoted + RiskConfig + max trades/score
    if candidate is None:
        return None

    risk_yaml = "\n".join(
        f"{key}: {value}" for key, value in asdict(candidate.risk_config).items()
    ) + "\n"
    output_path.parent.mkdir(parents=True, exist_ok=True)
    output_path.write_text(risk_yaml, encoding="utf-8")
    return {"promoted_risk_path": str(output_path)}
```

```bash
python3 -m pytest tests/test_integration.py -v --tb=short
python3 -m kalshi_agent.live_probe
python3 -m pytest tests/ -v --tb=short
```

---

## Verify backtest cache by forcing a bad base URL on replay
**Date:** 2026-03-21
**Context:** kalshi-agent/python/backtest cache validation
**Tags:** kalshi, backtest, sqlite, cache, validation, replay

### Problem / Observation

A second backtest run can appear faster, but that alone does not prove zero outbound HTTP calls. We needed a deterministic way to prove replay came entirely from SQLite cache.

### Resolution / Insight

Run the first backtest normally to populate cache, then run the same command again with `KALSHI_BASE_URL` pointed at an invalid localhost endpoint. If replay still succeeds and outputs are identical, the run did not depend on HTTP.

### Commands / Code

```bash
cd /Users/dansullivan/workspace/kalshi-agent
python3 -m kalshi_agent.backtest_probe \
  --start-date 2025-01-01 --end-date 2025-03-31 \
  --cities NYC,CHI,LAX,MIA,AUS \
  --disable-ncei-history \
  --cache-path data/historical_data_cache_probe.db

KALSHI_BASE_URL=http://127.0.0.1:1/trade-api/v2 \
python3 -m kalshi_agent.backtest_probe \
  --start-date 2025-01-01 --end-date 2025-03-31 \
  --cities NYC,CHI,LAX,MIA,AUS \
  --disable-ncei-history \
  --cache-path data/historical_data_cache_probe.db
```

---

## Format Slack text separately from structured alert logs
**Date:** 2026-03-22
**Context:** kalshi-agent/python alerting and paper-trade monitoring
**Tags:** slack, alerts, logging, json, webhook, monitoring

### Problem / Observation

Slack webhook messages were being sent as `{"text": json.dumps(payload)}`, which made Slack notifications hard to read. At the same time, the local/file alert logs were useful precisely because they stayed structured JSON.

### Resolution / Insight

Keep the webhook envelope the same (`{"text": ...}`), but format the inner Slack text as a concise multiline summary by alert type. Preserve the raw structured payload for file logging and return values. This improves Slack readability without breaking machine-readable local logs or webhook wiring.

### Commands / Code

```python
def _dispatch(self, payload: dict[str, object]) -> dict[str, object]:
    message = dict(payload)
    self._post_webhook({"text": self._format_slack_text(message)})
    self._log(message)
    return message
```

```bash
cd /Users/dansullivan/workspace/kalshi-agent
python3 -m pytest tests/test_monitoring.py -v --tb=short
python3 -m pytest tests/ -v --tb=short
```

---

## Keep profit projections machine-readable while rendering Slack-readable bullets
**Date:** 2026-03-22
**Context:** kalshi-agent/python gas paper-trade reporting
**Tags:** slack, alerts, profit-projection, observability, gas, paper-trading

### Problem / Observation

Adding gas profit breakdowns as nested JSON (`profit_projection`) preserves structure for logs, but default Slack formatting prints the entire dict inline, which is hard for operators to scan.

### Resolution / Insight

Emit both fields in trading-loop observability for gas paper trades: (1) a structured `profit_projection` object for machines and (2) a human sentence (`profit_projection_text`) explicitly marked success-case/not-realized. In Slack formatting, special-case `profit_projection` into readable bullet lines (entry basis, bot-sized notional, normalized `$100` example) instead of dumping raw JSON.

### Commands / Code

```python
if action_taken == "PAPER_TRADE" and resolved_strategy_name == "gas":
    payload["profit_projection"] = {
        "assumption": "success_case_projection_not_realized",
        "side": "YES",
        "entry_price_probability": 0.35,
        "bot_sized_notional": {...},
        "normalized_100_notional": {...},
    }
    payload["profit_projection_text"] = "Projected success-case profit (paper-only, not realized P&L): ..."

# alerts.py
if isinstance(message.get("profit_projection"), Mapping):
    lines.extend(_format_profit_projection_lines(message["profit_projection"]))
```

```bash
python3 -m pytest tests/test_monitoring.py -v --tb=short
python3 -m pytest tests/test_trading_loop.py -v --tb=short -k gas
python3 -m pytest tests/ -v --tb=short
```

---

## Probe deployed gas profit reporting with `_log_observability` and a temporary DB path
**Date:** 2026-03-22
**Context:** kalshi-agent/python ec2 paper-trade rollout validation
**Tags:** kalshi, ec2, docker, paper-trading, observability, profit-projection

### Problem / Observation

After rolling out gas profit-projection reporting to EC2, we needed a deterministic way to verify the deployed container could emit the new `profit_projection` and `profit_projection_text` fields without waiting for a natural gas paper trade and without touching the live paper-trade SQLite DB.

### Resolution / Insight

Run an in-container Python heredoc that builds the gas paper runner, overrides `KALSHI_TRADES_DB_PATH` to a throwaway file, and directly calls `TradingLoopRunner._log_observability(action_taken="PAPER_TRADE", ...)` with a synthetic gas `Signal`. This prints the exact deployed observability payload, including both bot-sized and normalized `$100` profit projections, while staying paper-only and avoiding persistent trade mutations.

### Commands / Code

```bash
ssh -i .secrets/ec2_key ec2-user@3.82.212.36 'docker exec -i kalshi-agent env KALSHI_TRADES_DB_PATH=/tmp/profit-probe.db python - <<"PY"
import os

from kalshi_agent.strategies.base import Signal
from kalshi_agent.trading_loop import build_gas_paper_runner

runner = build_gas_paper_runner(env=dict(os.environ), print_fn=print)
runner._log_observability(
    action_taken="PAPER_TRADE",
    reason="manual_profit_probe",
    cycle_id="manual-profit-probe",
    signal=Signal(
        event_ticker="KXAAAGASW-TEST",
        market_ticker="KXAAAGASW-TEST-B350",
        model_probability=1.0,
        market_probability=0.35,
        edge=0.65,
        recommended_notional=5.0,
        direction="BUY",
        side="YES",
        confidence=1.0,
        reason="manual_profit_probe",
        strategy_name="gas",
    ),
    observation={
        "parsed_values": {"gas_price": 3.456},
        "consensus_values": {"KXAAAGASW-TEST-B350": 0.35},
    },
    release_key="gas:manual-profit-probe",
    quantity=14,
    price=0.35,
    notional=5.0,
    execution_status="paper_filled",
)
PY'
```

---

## Wrap deployed AlertDispatcher with a recording proxy for real Slack validation
**Date:** 2026-03-29
**Context:** kalshi-agent/python deployed runtime-outcome validation
**Tags:** kalshi, slack, alerts, ec2, validation, proxy, paper-trading

### Problem / Observation

For deployed gas runtime validation we needed to prove that the live EC2 container actually sent meaningful Slack paper outcomes, while also confirming heartbeat reasons stayed Slack-silent. Replacing only the dispatcher transport made it awkward to preserve the real webhook send path and still capture which `event_type` / `reason` / `release_key` went out.

### Resolution / Insight

Keep the real `AlertDispatcher` intact and wrap `runner.outcome_alert_dispatcher` with a small proxy object that only overrides `send_daily_summary()`. The proxy forwards to the real dispatcher, records sanitized summary metadata from the already-built summary payload, and lets the deployed container keep using its injected webhook. Pair this with `/tmp` overrides for `KALSHI_TRADES_DB_PATH` and `KALSHI_ALERT_LOG_PATH` so the probe stays paper-only and does not touch the live trades DB or alert log.

### Commands / Code

```python
import os

from kalshi_agent.strategies.base import Signal
from kalshi_agent.trading_loop import build_gas_paper_runner

runner = build_gas_paper_runner(env=dict(os.environ), print_fn=lambda _: None)

class RecordingDispatcher:
    def __init__(self, delegate):
        self.delegate = delegate
        self.events = []

    def send_daily_summary(self, summary):
        result = self.delegate.send_daily_summary(summary)
        self.events.append(
            {
                "channel": result.get("channel"),
                "sent": bool(result.get("sent")),
                "event_type": summary.get("event_type"),
                "reason": summary.get("reason"),
                "release_key": summary.get("release_key"),
            }
        )
        return result

recording = RecordingDispatcher(runner.outcome_alert_dispatcher)
runner.outcome_alert_dispatcher = recording

runner._log_observability(
    action_taken="PAPER_TRADE",
    reason="manual_trade_probe",
    cycle_id="manual-trade-probe",
    signal=Signal(
        event_ticker="KXAAAGASW-TEST",
        market_ticker="KXAAAGASW-TEST-B350",
        model_probability=0.05,
        market_probability=0.65,
        edge=0.60,
        recommended_notional=5.0,
        direction="BUY",
        side="NO",
        confidence=0.95,
        reason="manual_trade_probe",
        strategy_name="gas",
    ),
    observation={"parsed_values": {"aaa_regular_national_average": 3.47}},
    release_key="gas:manual-trade-probe",
    quantity=14,
    price=0.65,
    notional=5.0,
    execution_status="paper_filled",
)
```

```bash
ssh -i /Users/dansullivan/workspace/kalshi-agent/.secrets/ec2_key ec2-user@3.82.212.36 \
  'docker exec -i kalshi-agent env KALSHI_TRADES_DB_PATH=/tmp/runtime-probe.db KALSHI_ALERT_LOG_PATH=/tmp/runtime-probe-alerts.log python - <<"PY"
import json
import os

from kalshi_agent.strategies.base import Signal
from kalshi_agent.trading_loop import build_gas_paper_runner

runner = build_gas_paper_runner(env=dict(os.environ), print_fn=lambda _: None)

class RecordingDispatcher:
    def __init__(self, delegate):
        self.delegate = delegate
        self.events = []

    def send_daily_summary(self, summary):
        result = self.delegate.send_daily_summary(summary)
        self.events.append(
            {
                "channel": result.get("channel"),
                "sent": bool(result.get("sent")),
                "event_type": summary.get("event_type"),
                "reason": summary.get("reason"),
                "release_key": summary.get("release_key"),
            }
        )
        return result

recording = RecordingDispatcher(runner.outcome_alert_dispatcher)
runner.outcome_alert_dispatcher = recording

# emit meaningful PAPER_TRADE and SKIP, then heartbeat reasons
...

print(json.dumps({"alerts_logged": len(recording.events), "events": recording.events}, sort_keys=True))
PY'
```

---

## Use run-scoped dirs plus manifest hashes for append-safe research collection scaffolds
**Date:** 2026-04-04
**Context:** kalshi-agent/python crypto research collection foundations
**Tags:** crypto, collection, manifests, lineage, reproducibility, data-isolation

### Problem / Observation

Crypto research collection foundations need to prove three things early: writes stay under `data/crypto_research/`, each run is append-safe (no silent overwrite), and downstream stages can verify exactly which raw artifacts a run used.

### Resolution / Insight

Use a strict namespace guard (`data/crypto_research` in the resolved output root), then write each run into a unique `collection_runs/<run_id>/` directory. Generate `run_manifest.json` that includes schema versions, source provenance contract, parameters, and per-artifact SHA256 hashes. Keep per-row lineage fields (`source_timestamp_utc`, `collector_timestamp_utc`, `source_observation_id`, `sequence_in_source`) in both Kalshi and external archives so no-lookahead alignment can be audited later.

### Commands / Code

```python
run_dir = output_root / "collection_runs" / run_id
run_dir.mkdir(parents=True, exist_ok=False)  # append-safe

artifacts = [
    {"artifact_id": "kalshi_archive", "path": str(kalshi_path), "sha256": sha256(kalshi_path.read_bytes())},
    {"artifact_id": "external_reference_archive", "path": str(ext_path), "sha256": sha256(ext_path.read_bytes())},
]

run_manifest = {
    "schema_version": "crypto-collection-run-manifest-v1",
    "run_id": run_id,
    "schema_versions": {...},
    "source_set": source_set,
    "artifacts": artifacts,
}
```

```bash
python3 -m pytest tests/test_crypto_storage.py -v --tb=short
python3 -m kalshi_agent.crypto_research.collection --output-root data/crypto_research
python3 -m kalshi_agent.crypto_research.collection --output-root data/crypto_research
python3 -m pytest tests/ -v --tb=short
```

---

## Record run-level timing quality plus raw epoch-ms fields in external reference archives
**Date:** 2026-04-04
**Context:** kalshi-agent/python crypto external reference collection
**Tags:** crypto, collection, external-reference, timestamp, lineage, alignment

### Problem / Observation

Per-row `timestamp_quality` alone was not enough to prove collection-time quality for a full run, and downstream validators needed raw timing fields to audit no-lookahead alignment deterministically.

### Resolution / Insight

Add top-level `source_provenance` and `timing_quality` sections to the external archive. In each row, persist both ISO and epoch-millisecond timestamps for source and collector times, plus explicit lag (`source_to_collector_lag_ms`). Use discovery-selected primary/fallback public/free source profiles to keep provenance family-appropriate and auditable.

### Commands / Code

```python
{
  "source_timestamp_utc": source_iso,
  "source_timestamp_raw": source_iso,
  "source_timestamp_unix_ms": int(source_time.timestamp() * 1000),
  "collector_timestamp_utc": collector_iso,
  "collector_timestamp_unix_ms": int(collector_time.timestamp() * 1000),
  "source_to_collector_lag_ms": collector_ms - source_ms,
}
```

```python
timing_quality = {
    "best_available_timestamp_precision": "millisecond",
    "achieved_timestamp_precision": "millisecond",
    "fractional_second_source_timestamp_row_count": source_fractional_count,
    "fractional_second_collector_timestamp_row_count": collector_fractional_count,
    "source_to_collector_lag_ms": {"min": ..., "p50": ..., "p95": ..., "max": ...},
}
```

```bash
python3 -m pytest tests/test_crypto_btc_reference_collection.py -v --tb=short
python3 -m pytest tests/ -v --tb=short
```

---

## Expose canonical namespace helpers so strict path guards stay testable
**Date:** 2026-04-04
**Context:** kalshi-agent/python crypto collection namespace enforcement
**Tags:** python, pytest, monkeypatch, pathlib, namespace, crypto, collection

### Problem / Observation

Tightening `output_root` validation to an exact repo canonical path (`<repo>/data/crypto_research`) broke collection tests that intentionally write to `tmp_path/data/crypto_research`. The old segment-matching guard allowed those tests implicitly, but canonical anchoring correctly rejected them.

### Resolution / Insight

Move canonical-root derivation into a dedicated helper (`_canonical_crypto_research_root()`) and keep namespace checking based on `path.relative_to(canonical_root)`. In tests, monkeypatch that helper to a temporary canonical root fixture so production logic stays strict while fixture-backed tests remain isolated and fast.

### Commands / Code

```python
def _canonical_crypto_research_root() -> Path:
    return (Path(__file__).resolve().parents[2] / "data" / "crypto_research").resolve()

def _assert_crypto_research_namespace(path: Path) -> None:
    canonical_root = _canonical_crypto_research_root()
    path.relative_to(canonical_root)  # raises ValueError if outside
```

```python
@pytest.fixture
def output_root(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> Path:
    canonical_output_root = (tmp_path / "data" / "crypto_research").resolve()
    monkeypatch.setattr(collection_module, "_canonical_crypto_research_root", lambda: canonical_output_root)
    return canonical_output_root
```

```bash
python3 -m pytest tests/test_crypto_storage.py tests/test_crypto_kalshi_collection.py tests/test_crypto_btc_reference_collection.py -v --tb=short
python3 -m pytest tests/ -v --tb=short
```

---

## Default CLI output roots should be derived from canonical namespace helpers
**Date:** 2026-04-04
**Context:** kalshi-agent/python crypto collection CLI
**Tags:** python, argparse, pathlib, cwd, namespace, crypto, collection

### Problem / Observation

With strict canonical namespace enforcement, using a relative CLI default like `--output-root data/crypto_research` can fail when the command is launched outside the repo root. `Path("data/crypto_research").resolve()` binds to the caller's current directory, not the repository.

### Resolution / Insight

Make `run_collection(output_root=...)` optional and default it to `_canonical_crypto_research_root()` when not provided. In argparse, set `--output-root` default to `None` and only wrap with `Path(...)` when explicitly supplied. This keeps explicit overrides supported while making default invocations cwd-independent and canonical.

### Commands / Code

```python
def run_collection(*, output_root: Path | None = None, ...):
    output_root = _canonical_crypto_research_root() if output_root is None else output_root.resolve()
    _assert_crypto_research_namespace(output_root)
```

```python
parser.add_argument("--output-root", default=None, help="... defaults to canonical repo-root path")
artifacts = run_collection(output_root=Path(args.output_root) if args.output_root else None, ...)
```

```bash
python3 -m pytest tests/test_crypto_storage.py -v --tb=short
tmpdir=$(mktemp -d) && cd "$tmpdir" && python3 -m kalshi_agent.crypto_research.collection
python3 -m pytest tests/ -v --tb=short
```
