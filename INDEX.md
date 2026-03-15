# Knowledge Base Index

A flat index of all entries. Agents MUST update this when adding new entries.
Scan this file first to avoid duplicating existing knowledge.

---

| Title | File | Tags | Date |
|-------|------|------|------|
| Skills must live in global skill dirs | learnings/gotchas.md | skills, factory, configuration, droids, agents | 2026-02-07 |
| macOS tempfile paths can flip between /var and /private/var | learnings/gotchas.md | macos, pathlib, tempfile, symlink, tests | 2026-03-15 |
| Multi-city walk-forward probes can hit Kalshi historical read timeouts | learnings/gotchas.md | kalshi, backtest, walkforward, timeout, diagnostics | 2026-03-15 |
| Short single-city backtest probes can be blocked by entry price bounds | learnings/gotchas.md | kalshi, backtest, diagnostics, risk-gates, entry-price | 2026-03-15 |
| Backtest entry snapshots should use ask (not midpoint) for YES taker fills | learnings/gotchas.md | kalshi, backtest, entry-price, bid-ask, taker, pnl | 2026-03-15 |
| NCEI daily summary fetches should be batched per year | learnings/gotchas.md | ncei, noaa, backtest, pagination, limits, historical-data | 2026-03-15 |
| Kalshi `get_markets` can reject `status=active` on some environments | learnings/gotchas.md | kalshi, api, markets, live-execution, error-handling | 2026-03-15 |
| Kalshi order placement/cancel endpoints are portfolio-scoped | learnings/gotchas.md | kalshi, api, orders, transport, delete, compatibility | 2026-03-15 |
| Python signal handlers + long `time.sleep()` can delay shutdown indefinitely | learnings/gotchas.md | python, signals, sleep, graceful-shutdown, sigint, sigterm | 2026-03-15 |
| Use project venv python when pyenv blocks `python3` | learnings/tools.md | python, pyenv, venv, path, cli | 2026-03-15 |
