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
| Terraform `templatefile` needs `$${VAR}` for literal shell expansion | learnings/gotchas.md | terraform, templatefile, user-data, bash, escaping | 2026-03-15 |
| Use project venv python when pyenv blocks `python3` | learnings/tools.md | python, pyenv, venv, path, cli | 2026-03-15 |
| `.factory/init.sh` may append Terraform ignores into `.gitignore` | learnings/tools.md | git, init-script, gitignore, terraform, factory | 2026-03-16 |
| `storizzi/notes-exporter` exports all Apple Notes folders; isolate the target folder from output | learnings/tools.md | apple-notes, macos, markdown, export, applescript, cli | 2026-03-18 |
| CloudWatch `filter-log-events` with `--limit` returns earliest events unless you bound time | learnings/tools.md | aws, cloudwatch, logs, cli, validation | 2026-03-18 |
| Factory `Read` may not natively open PDFs larger than 3MB | learnings/tools.md | factory, pdf, read-tool, python, pypdf, workaround | 2026-03-18 |
| AWS account blocked can fail EC2 RunInstances during Terraform apply | resolutions/aws-account-blocked-ec2-runinstances.md | aws, terraform, ec2, runinstances, account-blocked, deployment | 2026-03-16 |
| Default VPC subnet ordering can pick an AZ that rejects `t3.micro` | learnings/gotchas.md | terraform, aws, ec2, subnet, availability-zone, instance-type | 2026-03-17 |
| Apple Silicon ECR image can crash on EC2 amd64 with `exec format error` | learnings/gotchas.md | docker, ecr, ec2, arm64, amd64, buildx, exec-format | 2026-03-17 |
| EC2 `describe-instances` may not expose EBS encryption field reliably | learnings/gotchas.md | aws, ec2, ebs, encryption, verification, cli | 2026-03-18 |
| Terraform EC2 `user_data` updates may not recreate instances | learnings/gotchas.md | terraform, aws, ec2, user-data, replacement, deployment | 2026-03-18 |
| Slack webhooks can return plain-text `ok`, which breaks strict JSON transport parsing | learnings/gotchas.md | slack, webhook, json, urllib, transport, monitoring, ec2 | 2026-03-18 |
| Terraform destroy can fail on non-empty ECR repositories | learnings/gotchas.md | terraform, aws, ecr, destroy, teardown, repository-not-empty | 2026-03-17 |
| Argparse help strings treat `%` as interpolation placeholders | learnings/gotchas.md | python, argparse, cli, formatting, help-text | 2026-03-21 |
| Walk-forward probes can appear hung when each backtest run re-fetches uncached data | learnings/gotchas.md | kalshi, walkforward, backtest, cache, runtime, ncei | 2026-03-21 |
| Export walk-forward promoted RiskConfig directly to trading-loop file | learnings/patterns.md | kalshi, walkforward, promotion, risk-config, trading-loop, integration-tests | 2026-03-15 |
| Verify backtest cache by forcing a bad base URL on replay | learnings/patterns.md | kalshi, backtest, sqlite, cache, validation, replay | 2026-03-21 |
| `data/` gitignore pattern can hide new files under tracked `kalshi_agent/data/` | learnings/tools.md | git, gitignore, tracked-dirs, add-force, diagnostics | 2026-03-21 |
| Backtest cache SQLite tables use `event_date` (not `date`) | learnings/tools.md | sqlite, backtest-cache, schema, diagnostics | 2026-03-21 |
| Ignored `data/` paths can block staging modified tracked artifacts | learnings/tools.md | git, gitignore, tracked-files, data, add-force | 2026-03-21 |
| BLS CPI release HTML keeps key metrics split across two `<pre>` blocks | learnings/gotchas.md | bls, cpi, html, parser, regex, fixtures | 2026-03-21 |
| AAA national average text may include header labels between `Regular` and the first price | learnings/gotchas.md | aaa, gas, parser, regex, html, fixtures | 2026-03-21 |
| Top-level CLI packages need setuptools include patterns for `pip install -e .` | learnings/tools.md | python, setuptools, editable-install, cli, packaging | 2026-03-22 |
| BLS CPI pages expose release time as an embargo line, not `For release ...` | learnings/gotchas.md | bls, cpi, html, timestamp, parsing | 2026-03-22 |
| Release backtest replays should use `price_previous_dollars` for consensus | learnings/gotchas.md | kalshi, backtest, orderbook, consensus, replay, cpi, gas | 2026-03-22 |
| BLS release schedule times must be converted from US/Eastern with DST awareness | learnings/gotchas.md | bls, scheduler, timezone, dst, utc, polling | 2026-03-22 |
| Terraform apply can fail after restoring a Secrets Manager secret unless it is imported | learnings/gotchas.md | terraform, aws, secretsmanager, import, state, ec2, deployment | 2026-03-22 |
| `docker exec` needs `-i` when piping heredoc scripts into container Python | learnings/tools.md | docker, exec, stdin, heredoc, ssh, ec2 | 2026-03-22 |
| Flow-validator subagents can permission-abort unless prompts are narrowed to explicit read-only commands | learnings/tools.md | factory, subagent, task, permissions, validation, prompts | 2026-03-22 |
| TradingLoopRunner mock mode logs `TRADE` unless `paper_trader` is attached | learnings/gotchas.md | kalshi, trading-loop, mock-mode, paper-trading, tests, observability | 2026-03-22 |
| Format Slack text separately from structured alert logs | learnings/patterns.md | slack, alerts, logging, json, webhook, monitoring | 2026-03-22 |
| Keep profit projections machine-readable while rendering Slack-readable bullets | learnings/patterns.md | slack, alerts, profit-projection, observability, gas, paper-trading | 2026-03-22 |
