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
| AlertDispatcher custom transports must implement `post`, not `post_json` | learnings/gotchas.md | slack, alerts, transport, testing, probes, kalshi | 2026-03-22 |
| Fixed idle poll sleeps can overshoot short release activation windows | learnings/gotchas.md | scheduler, polling, gas, ec2, release-window, observability | 2026-03-29 |
| Manual EC2 container replacements can bypass Secrets Manager env injection | learnings/gotchas.md | aws, ec2, docker, secretsmanager, slack, env, deployment | 2026-03-29 |
| Format Slack text separately from structured alert logs | learnings/patterns.md | slack, alerts, logging, json, webhook, monitoring | 2026-03-22 |
| Keep profit projections machine-readable while rendering Slack-readable bullets | learnings/patterns.md | slack, alerts, profit-projection, observability, gas, paper-trading | 2026-03-22 |
| Probe deployed gas profit reporting with `_log_observability` and a temporary DB path | learnings/patterns.md | kalshi, ec2, docker, paper-trading, observability, profit-projection | 2026-03-22 |
| Wrap deployed AlertDispatcher with a recording proxy for real Slack validation | learnings/patterns.md | kalshi, slack, alerts, ec2, validation, proxy, paper-trading | 2026-03-29 |
| Docker CLI may fail until Docker Desktop is explicitly started in exec sessions | learnings/tools.md | docker, docker-desktop, macos, buildx, ecr, deployment | 2026-03-29 |
| Weekly gas idle loops use `gas_weekly_schedule` reason, not `scheduler_idle` | learnings/gotchas.md | gas, scheduler, slack, alerts, observability, skip-filter | 2026-03-29 |
| Use scoped stash for pre-existing `.factory/` mission changes before worker handoff | learnings/tools.md | git, stash, factory, mission, working-tree | 2026-04-04 |
| Scrutiny can fail when `EndFeatureRun.commitId` points at a later cleanup commit | learnings/tools.md | factory, scrutiny, handoff, commit, git, validation | 2026-04-04 |
| Scrutiny reruns should snapshot prior synthesis as `synthesis.roundN.json` | learnings/tools.md | factory, scrutiny, validation, synthesis, rerun | 2026-04-04 |
| Scrutiny reruns can overwrite prior review JSON in place | learnings/tools.md | factory, scrutiny, validation, review, rerun, auditability | 2026-04-04 |
| Crypto backtest hedge validation can use a `python3` inline wrapper when the module CLI lacks strategy-config args | learnings/tools.md | kalshi, crypto, backtest, cli, validation, hedging, user-testing | 2026-04-04 |
| Mixed-asset discovery cannot reuse a BTC-only reference manifest | learnings/gotchas.md | crypto, discovery, manifests, reference-data, lineage, schemas | 2026-04-04 |
| Use run-scoped dirs plus manifest hashes for append-safe research collection scaffolds | learnings/patterns.md | crypto, collection, manifests, lineage, reproducibility, data-isolation | 2026-04-04 |
| Record run-level timing quality plus raw epoch-ms fields in external reference archives | learnings/patterns.md | crypto, collection, external-reference, timestamp, lineage, alignment | 2026-04-04 |
| Expose canonical namespace helpers so strict path guards stay testable | learnings/patterns.md | python, pytest, monkeypatch, pathlib, namespace, crypto, collection | 2026-04-04 |
| Default CLI output roots should be derived from canonical namespace helpers | learnings/patterns.md | python, argparse, pathlib, cwd, namespace, crypto, collection | 2026-04-04 |
| Keyword-only helper refactors can fail at runtime when old positional calls remain | learnings/gotchas.md | python, keyword-only, refactor, runtime-error, backtest | 2026-04-04 |
| Reconcile hedged backtest trades from explicit per-leg accounting | learnings/patterns.md | crypto, backtest, hedging, leg-accounting, reconciliation, pnl | 2026-04-04 |
| Crypto backtest scrutiny must validate window slicing, no-lookahead lags, and hedged aggregate fields | learnings/gotchas.md | crypto, backtest, scrutiny, family-window, lookahead, hedging, ledgers | 2026-04-04 |
| Lookahead-safe reference selection can silently zero out trades if fixture timing drifts ahead of decision snapshots | learnings/gotchas.md | crypto, backtest, lookahead, timestamps, fixtures, ledgers | 2026-04-04 |
| Hedged exit semantics should key on outcome mode, not per-leg price parity | learnings/gotchas.md | crypto, backtest, hedging, summary-counters, exit-semantics, ledgers | 2026-04-04 |
| Strategy research matched-window and holdout checks must preserve same family-window lineage | learnings/gotchas.md | crypto, strategy-research, scrutiny, matched-window, holdout, lineage, hedging | 2026-04-04 |
| Build strategy research by post-processing execution-aware backtest ledgers | learnings/patterns.md | crypto, strategy-research, backtest, ranking, regime-slices, hedging | 2026-04-04 |
