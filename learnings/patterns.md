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
