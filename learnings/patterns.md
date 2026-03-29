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
