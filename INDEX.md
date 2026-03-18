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
| Export walk-forward promoted RiskConfig directly to trading-loop file | learnings/patterns.md | kalshi, walkforward, promotion, risk-config, trading-loop, integration-tests | 2026-03-15 |
