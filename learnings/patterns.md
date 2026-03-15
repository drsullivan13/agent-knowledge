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
