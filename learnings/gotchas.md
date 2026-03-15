# Gotchas & Edge Cases

Entries are appended by agents as they encounter footguns and edge cases.

---

## Skills must live in global skill dirs
**Date:** 2026-02-07
**Context:** Factory skills configuration
**Tags:** skills, factory, configuration, droids, agents

### Problem / Observation
Skills were mistakenly created inside a project repo under `.factory/droids` or `.claude/agents`, which are for droids/agents rather than skills.

### Resolution / Insight
Create skills only in the global root skill directories (e.g., `~/.factory/skills` and `~/.claude/skills`) so they are discoverable and loaded correctly.

---

## macOS tempfile paths can flip between /var and /private/var
**Date:** 2026-03-15
**Context:** Python pathlib + macOS temp dirs
**Tags:** macos, pathlib, tempfile, symlink, tests

### Problem / Observation
On macOS, `tempfile.TemporaryDirectory()` often returns a `/var/...` path, but calling `Path.resolve()` converts it to `/private/var/...` because `/var` is a symlink. String-equality assertions then fail even when both paths reference the same location.

### Resolution / Insight
Avoid forcing `resolve()` on caller-provided temp paths when exact string preservation matters (for env vars/log outputs), or normalize both sides with `os.path.realpath()`/`Path.resolve()` in tests before asserting.

### Commands / Code
```python
def apply_live_defaults(secret_dir: Path = _project_root() / ".secrets") -> dict[str, Any]:
    # Keep caller-provided path form (don't call secret_dir.resolve())
    private_key_path = secret_dir / "kalshi_private_key.key"
```

```python
self.assertEqual(os.path.realpath(actual), os.path.realpath(expected))
```

---

## Multi-city walk-forward probes can hit Kalshi historical read timeouts
**Date:** 2026-03-15
**Context:** kalshi-agent backtest tuning probes
**Tags:** kalshi, backtest, walkforward, timeout, diagnostics

### Problem / Observation
`python -m kalshi_agent.backtest_walkforward_probe` with long date ranges and multiple cities can fail with `TimeoutError: The read operation timed out` while fetching historical event payloads from Kalshi.

### Resolution / Insight
For tuning and regression comparisons, run a stable single-city baseline command (e.g., `NYC`) first, then use explicit strict-vs-tuned risk overrides to prove throughput deltas. Use multi-city backtest diagnostics on the lighter `backtest_probe` command for blocker analysis.

### Commands / Code
```bash
# Stable tuning baseline (single city)
KALSHI_MOCK_MODE=true python3 -m kalshi_agent.backtest_walkforward_probe \
  --start-date 2025-01-01 --end-date 2025-06-01 --cities NYC --compact

# Strict-vs-tuned comparison without editing config files
KALSHI_MOCK_MODE=true python3 -m kalshi_agent.backtest_walkforward_probe \
  --start-date 2025-01-01 --end-date 2025-06-01 --cities NYC --compact \
  --risk-min-entry-price-probability 0.05 --risk-max-entry-price-probability 0.90 \
  --risk-min-model-probability 0.52 --risk-max-model-probability 0.98 \
  --risk-max-bid-ask-spread-probability 0.08 --risk-min-book-depth-contracts 20

# Multi-city blocker analysis with diagnostics (faster than walk-forward)
KALSHI_MOCK_MODE=true python3 -m kalshi_agent.backtest_probe \
  --start-date 2025-01-01 --end-date 2025-04-01 \
  --cities NYC,CHI,LAX,MIA,AUS --max-events-per-city 35 \
  --target-modeled-events-per-city 8 --max-markets-to-price 4 --diagnostics
```

---

## Short single-city backtest probes can be blocked by entry price bounds
**Date:** 2026-03-15
**Context:** kalshi-agent backtest probe diagnostics
**Tags:** kalshi, backtest, diagnostics, risk-gates, entry-price

### Problem / Observation
On `python3 -m kalshi_agent.backtest_probe --start-date 2025-03-01 --end-date 2025-04-01 --cities NYC --diagnostics`, changing `TemperatureForecaster` EWMA/trend weights and `RollingErrorCalibrator` sigma behavior did not move trade count off zero. Diagnostics stayed dominated by `entry_price_out_of_bounds` and `rank_cutoff`.

### Resolution / Insight
If diagnostics are dominated by entry bounds, forecast tuning alone is often insufficient. Check entry snapshot sourcing and entry-price risk bounds before iterating on forecaster parameters.

### Commands / Code
```bash
python3 -m kalshi_agent.backtest_probe \
  --start-date 2025-03-01 --end-date 2025-04-01 --cities NYC --diagnostics
```

