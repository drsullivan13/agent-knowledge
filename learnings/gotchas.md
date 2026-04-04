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

---

## Backtest entry snapshots should use ask (not midpoint) for YES taker fills
**Date:** 2026-03-15
**Context:** kalshi-agent weather backtest pricing accuracy
**Tags:** kalshi, backtest, entry-price, bid-ask, taker, pnl

### Problem / Observation
`_extract_entry_snapshot_from_candles()` used `(bid + ask) / 2` when both quotes existed. But `PaperTrader.place_yes_order()` models a taker YES buy, so executable cost should be the ask side. Midpoint pricing can admit unrealistic candidates (especially very wide spreads) and distort short-window autotune outcomes.

### Resolution / Insight
Use ask as `entry_probability` when bid+ask are present, still recording spread/depth for risk filters. Also avoid truthy `or` fallback for numeric fields (`0.0` is falsy) by selecting the first non-`None` parsed value.

### Commands / Code
```python
def _first_non_none(*values: float | None) -> float | None:
    for value in values:
        if value is not None:
            return value
    return None

bid = _first_non_none(_quote_value(yes_bid, "open"), _quote_value(yes_bid, "close"))
ask = _first_non_none(_quote_value(yes_ask, "open"), _quote_value(yes_ask, "close"))

if bid is not None and ask is not None and bid > 0.0 and ask > 0.0:
    return MarketPricingSnapshot(
        entry_probability=_clamp_probability(ask),
        bid_ask_spread_probability=max(0.0, ask - bid),
        available_depth_contracts=depth,
        source=source,
    )
```

---

## NCEI daily summary fetches should be batched per year
**Date:** 2026-03-15
**Context:** kalshi-agent NCEI historical backtest integration
**Tags:** ncei, noaa, backtest, pagination, limits, historical-data

### Problem / Observation
NOAA NCEI `cdo-web/api/v2/data` can return only up to `limit=1000` rows per request. Pulling 5-10 years of `TMAX/TMIN` in one request can silently underfetch historical temperature rows.

### Resolution / Insight
Batch `get_daily_summaries()` calls by calendar year, merge rows, then dedupe by observed date. This keeps each request under the row cap and consistently retrieves month-matched history for empirical distributions.

### Commands / Code
```python
for year in range(history_start.year, history_end.year + 1):
    year_start = max(history_start, date(year, 1, 1))
    year_end = min(history_end, date(year, 12, 31))
    payload = ncei.get_daily_summaries(
        station_id=station_id,
        start_date=year_start.isoformat(),
        end_date=year_end.isoformat(),
        data_types=("TMAX", "TMIN"),
        units="standard",
        limit=1000,
    )
```

---

## Kalshi `get_markets` can reject `status=active` on some environments
**Date:** 2026-03-15
**Context:** kalshi-agent live trading loop market scan
**Tags:** kalshi, api, markets, live-execution, error-handling

### Problem / Observation
Calling `GET /markets` with `status=active` may return HTTP 400 (`invalid status filter`) depending on API environment/cluster behavior.

### Resolution / Insight
Treat this as a recoverable API error in the trading loop: log the failure for that cycle and continue to the next poll interval without crashing or committing partial trade state.

### Commands / Code
```bash
KALSHI_MOCK_MODE=true POLL_INTERVAL_SECONDS=1 KALSHI_MAX_CYCLES=2 python3 -m kalshi_agent
```

```python
try:
    active_markets = kalshi.get_markets(status="active", limit=1000)
except TransportError as exc:
    # Log and continue next cycle
    print({"api_error": str(exc)})
```

---

## Kalshi order placement/cancel endpoints are portfolio-scoped
**Date:** 2026-03-15
**Context:** kalshi-agent live execution + monitoring milestone
**Tags:** kalshi, api, orders, transport, delete, compatibility

### Problem / Observation
Older client code used `POST /markets/{ticker}/orders` for placement and `POST /portfolio/orders/{id}/cancel` for cancel. Current Kalshi docs use portfolio-scoped endpoints and DELETE for cancel.

### Resolution / Insight
Use `POST /trade-api/v2/portfolio/orders` with `ticker` in body for placement, and `DELETE /trade-api/v2/portfolio/orders/{order_id}` for cancel. Add a `delete()` method to the shared transport protocol/implementation and normalize `status="active"` to `"open"` for market listing with a 400 fallback retry that omits status.

### Commands / Code
```python
# placement
url = f"{base_url}/portfolio/orders"
headers = signer.headers_for("POST", url)
transport.post(url, headers=headers, body={"ticker": ticker, ...})

# cancel
url = f"{base_url}/portfolio/orders/{order_id}"
headers = signer.headers_for("DELETE", url)
transport.delete(url, headers=headers)
```

```python
def _normalize_market_status(status: str) -> str:
    return "open" if status.strip().lower() == "active" else status.strip().lower()

try:
    return transport.get(".../markets", params={"status": _normalize_market_status(status)})
except TransportError as exc:
    if status and "HTTP 400" in str(exc):
        return transport.get(".../markets", params={"limit": limit})
    raise
```

---

## BLS release schedule times must be converted from US/Eastern with DST awareness
**Date:** 2026-03-22
**Context:** CPI release scheduler activation windows
**Tags:** bls, scheduler, timezone, dst, utc, polling

### Problem / Observation
BLS schedule rows are published in Eastern Time, and treating parsed `8:30 AM` as fixed UTC-5 caused wrong activation windows after the DST switch (e.g., March releases should map to 12:30 UTC, not 13:30 UTC).

### Resolution / Insight
Interpret parsed BLS schedule datetimes as `America/New_York` and convert to UTC with `zoneinfo`. Compute active windows from that UTC instant (`release - offset` to `release + edge_decay`) so polling cadence stays correct year-round.

### Commands / Code
```python
from datetime import UTC
from zoneinfo import ZoneInfo

eastern = ZoneInfo("America/New_York")
release_et = parsed_release_naive.replace(tzinfo=eastern)
release_utc = release_et.astimezone(UTC)
```

---

## Python signal handlers + long `time.sleep()` can delay shutdown indefinitely
**Date:** 2026-03-15
**Context:** kalshi-agent trading loop graceful shutdown
**Tags:** python, signals, sleep, graceful-shutdown, sigint, sigterm

### Problem / Observation
Installing SIGINT/SIGTERM handlers that only set a shutdown flag does not guarantee an immediate exit when the loop is in a long `time.sleep()` call. The process can remain blocked for the full poll interval.

### Resolution / Insight
Use chunked sleep (`<=1s`) and re-check the shutdown flag between chunks. This lets signal handlers request shutdown while preserving "finish current cycle" semantics and exiting promptly after the cycle or current sleep chunk.

### Commands / Code
```python
def _sleep_until_next_cycle(self) -> None:
    remaining = max(0.0, float(self.poll_interval_seconds))
    while remaining > 0.0 and not self.shutdown_requested:
        chunk = min(remaining, 1.0)
        self.sleep_fn(chunk)
        remaining -= chunk
```

```python
def _handle_shutdown(signum: int, _frame: Any) -> None:
    runner.request_shutdown(signal.Signals(signum).name)
```

---

## Terraform `templatefile` needs `$${VAR}` for literal shell expansion
**Date:** 2026-03-15
**Context:** Terraform EC2 user_data templates (`.tftpl`)
**Tags:** terraform, templatefile, user-data, bash, escaping

### Problem / Observation
In a Terraform `templatefile`, plain `${...}` is parsed as Terraform interpolation. Shell expressions like `${SECRET_JSON}` or `${ECR_REPOSITORY_URL%%/*}` inside user data can break rendering or resolve unexpectedly.

### Resolution / Insight
Use Terraform placeholders only for values you want injected from HCL (for example `${aws_region}`), and escape runtime shell expansion as `$${...}` so the rendered script keeps `${...}` for bash.

### Commands / Code
```hcl
user_data = templatefile("${path.module}/user_data.sh.tftpl", {
  aws_region = "us-east-1"
})
```

---

## Walk-forward probes can appear hung when each backtest run re-fetches uncached data
**Date:** 2026-03-21
**Context:** kalshi-agent comprehensive walk-forward validation
**Tags:** kalshi, walkforward, backtest, cache, runtime, ncei

### Problem / Observation
`backtest_walkforward_probe` over a 6+ month range repeatedly timed out (>5 minutes) even for tiny windows because each window executes multiple backtests, and uncached event/candle lookups (plus NCEI history fetches) made each run very slow.

### Resolution / Insight
Use a dedicated walk-forward cache DB with a stable cache namespace, run walk-forward in cache-only mode (skip live fetches on cache miss), and disable NCEI history inside walk-forward train/validation configs. This made full 180-day walk-forward runs complete within the command budget and allowed threshold verification.

### Commands / Code
```python
# walkforward.py
backtest_cache = BacktestCache(
    db_path=_project_root() / "data" / "walkforward_backtest_cache.db",
    schema_version=BACKTEST_CACHE_SCHEMA_VERSION,
    config_hash="walkforward-replay-v1",
)

# pass cache_only=True into sweep/holdout weather backtests
```

```bash
python3 -m kalshi_agent.backtest_walkforward_probe \
  --start-date 2025-09-17 --end-date 2026-03-15 \
  --dimension-sweep --max-markets-options 1 --confidence-bounds 0.10-0.12 \
  --disable-rank-cutoff-deep-sweep --disable-rank-cutoff-targeted-band-sweep \
  --disable-rank-cutoff-auto-promote-deep --top-candidates 1 --leaderboard-size 1 \
  --holdout-min-train-trades 1 --holdout-min-validation-trades 1 \
  --allow-negative-validation-pnl --compact --with-ci
```

```bash
AWS_REGION="${aws_region}"              # Terraform variable replacement
SECRET_JSON="$(aws ... --region "$${AWS_REGION}")"  # Literal bash expansion
ECR_REGISTRY="$(echo "$${ECR_REPOSITORY_URL}" | cut -d/ -f1)"
```

---

## Default VPC subnet ordering can pick an AZ that rejects `t3.micro`
**Date:** 2026-03-17
**Context:** Terraform EC2 deployment in fresh AWS accounts
**Tags:** terraform, aws, ec2, subnet, availability-zone, instance-type

### Problem / Observation
Using `subnet_id = sort(data.aws_subnets.default_vpc.ids)[0]` selected `us-east-1e` in one account. `terraform apply` then failed with EC2 `RunInstances` 400: `Unsupported: Your requested instance type (t3.micro) is not supported in your requested Availability Zone`.

### Resolution / Insight
Select the subnet from AZs that currently offer the desired instance type. In Terraform, query `aws_ec2_instance_type_offerings` for `t3.micro`, filter default VPC subnets by those AZs, and pick from that filtered list.

### Commands / Code
```hcl
data "aws_ec2_instance_type_offerings" "t3_micro" {
  location_type = "availability-zone"
  filter {
    name   = "instance-type"
    values = ["t3.micro"]
  }
}

data "aws_subnets" "default_vpc_t3_supported" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
  filter {
    name   = "availability-zone"
    values = sort(data.aws_ec2_instance_type_offerings.t3_micro.locations)
  }
}

resource "aws_instance" "trading_agent" {
  instance_type = "t3.micro"
  subnet_id     = sort(data.aws_subnets.default_vpc_t3_supported.ids)[0]
}
```

---

## Apple Silicon ECR image can crash on EC2 amd64 with `exec format error`
**Date:** 2026-03-17
**Context:** Docker/ECR deployment for kalshi-agent on EC2 Amazon Linux 2023
**Tags:** docker, ecr, ec2, arm64, amd64, buildx, exec-format

### Problem / Observation
After deploy, `docker ps` on EC2 showed container repeatedly restarting. `docker logs` showed `exec /usr/local/bin/python: exec format error`. The image had been pushed from a macOS Apple Silicon host as `linux/arm64`, but EC2 host architecture was `linux/amd64`.

### Resolution / Insight
Rebuild and push explicitly for `linux/amd64` using `docker buildx`, then rerun the EC2 bootstrap/user-data script (or recreate container) so EC2 pulls the corrected image.

### Commands / Code
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 319025930540.dkr.ecr.us-east-1.amazonaws.com
docker buildx build --platform linux/amd64 \
  -t 319025930540.dkr.ecr.us-east-1.amazonaws.com/kalshi-agent:latest \
  --push /Users/dansullivan/workspace/kalshi-agent

ssh -i /Users/dansullivan/workspace/kalshi-agent/.secrets/ec2_key ec2-user@<ec2-ip> \
  'sudo bash /var/lib/cloud/instance/scripts/part-001'

ssh -i /Users/dansullivan/workspace/kalshi-agent/.secrets/ec2_key ec2-user@<ec2-ip> \
  "docker ps --format 'table {{.Names}}\t{{.Status}}'"
```

---

## EC2 `describe-instances` may not expose EBS encryption field reliably
**Date:** 2026-03-18
**Context:** AWS CLI verification after Terraform EC2 replacement
**Tags:** aws, ec2, ebs, encryption, verification, cli

### Problem / Observation
After replacing an EC2 instance with an encrypted root volume, `aws ec2 describe-instances --query 'Reservations[].Instances[].BlockDeviceMappings[].Ebs.Encrypted'` returned `[]` even though the volume was attached and encrypted.

### Resolution / Insight
Use `describe-instances` to get the root `VolumeId`, then verify encryption via `aws ec2 describe-volumes --query 'Volumes[].Encrypted'`.

### Commands / Code
```bash
VOL_ID=$(aws ec2 describe-instances --region us-east-1 \
  --filters Name=tag:Project,Values=kalshi-agent Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].BlockDeviceMappings[].Ebs.VolumeId' --output text)

aws ec2 describe-volumes --region us-east-1 --volume-ids "$VOL_ID" \
  --query 'Volumes[].Encrypted' --output text
```

---

## Terraform EC2 `user_data` updates may not recreate instances
**Date:** 2026-03-18
**Context:** Terraform AWS EC2 deploys with bootstrap scripts
**Tags:** terraform, aws, ec2, user-data, replacement, deployment

### Problem / Observation
Changing `aws_instance.user_data` updated the instance in-place and did not recreate it. The running container kept using old bootstrap behavior (old image tag/env values), even though Terraform apply succeeded.

### Resolution / Insight
Set `user_data_replace_on_change = true` on `aws_instance` so future `user_data` diffs force replacement. If user_data already changed before this flag was enabled, make one more user_data edit and apply again to trigger replacement.

### Commands / Code
```hcl
resource "aws_instance" "trading_agent" {
  # ...
  user_data_replace_on_change = true
  user_data = templatefile("${path.module}/user_data.sh.tftpl", {
    # ...
  })
}
```

```bash
cd /Users/dansullivan/workspace/kalshi-agent/infra
terraform apply -auto-approve
```

---

## Slack webhooks can return plain-text `ok`, which breaks strict JSON transport parsing
**Date:** 2026-03-18
**Context:** kalshi-agent monitoring alerts on EC2
**Tags:** slack, webhook, json, urllib, transport, monitoring, ec2

### Problem / Observation

Triggering `AlertDispatcher.send_daily_summary()` from the running EC2 container returned `{"channel":"file","sent":false,...}` with `Invalid JSON response ...` even though the webhook endpoint accepted the request.

### Resolution / Insight

Slack incoming webhooks commonly return HTTP 200 with plain-text `ok`, not JSON. A transport layer that always JSON-decodes response bodies can raise false failures for successful Slack sends. For live validation, verify by issuing a direct POST from inside the EC2 container and checking `200` + `ok`.

### Commands / Code

```bash
ssh -i /Users/dansullivan/workspace/kalshi-agent/.secrets/ec2_key ec2-user@<ec2-ip> \
  "docker exec kalshi-agent python -c 'import json; from kalshi_agent.monitoring.alerts import AlertDispatcher; print(AlertDispatcher().send_daily_summary({\"source\":\"manual\"}))'"

ssh -i /Users/dansullivan/workspace/kalshi-agent/.secrets/ec2_key ec2-user@<ec2-ip> \
  "docker exec kalshi-agent python -c 'import json, os, urllib.request; u=os.environ[\"SLACK_WEBHOOK_URL\"]; d=json.dumps({\"text\":\"ec2 webhook validation\"}).encode(); r=urllib.request.urlopen(urllib.request.Request(u,data=d,headers={\"Content-Type\":\"application/json\"},method=\"POST\"),timeout=10); print(r.status); print(r.read().decode())'"
```

---

## Terraform destroy can fail on non-empty ECR repositories
**Date:** 2026-03-17
**Context:** Terraform AWS teardown for kalshi-agent live-validation cleanup
**Tags:** terraform, aws, ecr, destroy, teardown, repository-not-empty

### Problem / Observation

`terraform destroy -auto-approve` failed after tearing down most resources with `RepositoryNotEmptyException` on `aws_ecr_repository` because the repo still contained tagged images.

### Resolution / Insight

When `force_delete` is not enabled on the ECR repository resource, explicitly delete all image digests first (`aws ecr batch-delete-image`) and rerun `terraform destroy`.

### Commands / Code

```bash
# 1) Remove all image digests from the repo
for digest in $(aws ecr list-images --repository-name kalshi-agent --query 'imageIds[*].imageDigest' --output text | tr '\t' '\n' | sort -u); do
  aws ecr batch-delete-image --repository-name kalshi-agent --image-ids imageDigest="$digest"
done

# 2) Retry destroy
cd /Users/dansullivan/workspace/kalshi-agent/infra && terraform destroy -auto-approve
```

---

## Terraform apply can fail after restoring a Secrets Manager secret unless it is imported
**Date:** 2026-03-22
**Context:** kalshi-agent EC2 deployment (Terraform + Secrets Manager)
**Tags:** terraform, aws, secretsmanager, import, state, ec2, deployment

### Problem / Observation

`terraform apply` failed creating `aws_secretsmanager_secret.kalshi_agent_credentials` with `InvalidRequestException` when the secret name was scheduled for deletion. After `restore-secret`, a second `terraform apply` failed with `ResourceExistsException` because the secret existed in AWS but was not in Terraform state.

### Resolution / Insight

Cancel deletion first, then import the restored secret into Terraform state before re-applying. Without the import step, Terraform repeatedly tries to create a secret that already exists.

### Commands / Code

```bash
aws secretsmanager restore-secret --region us-east-1 --secret-id kalshi-agent/credentials
terraform -chdir="/Users/dansullivan/workspace/kalshi-agent/infra" import \
  aws_secretsmanager_secret.kalshi_agent_credentials \
  kalshi-agent/credentials
terraform -chdir="/Users/dansullivan/workspace/kalshi-agent/infra" apply -auto-approve
```

---

## Argparse help strings treat `%` as interpolation placeholders
**Date:** 2026-03-21
**Context:** Python argparse CLI flags
**Tags:** python, argparse, cli, formatting, help-text

### Problem / Observation

`python3 -m kalshi_agent.backtest_probe --help` crashed with:
`ValueError: unsupported format character 'b' (0x62)` after adding help text that included `95% bootstrap ...`.

### Resolution / Insight

`argparse` uses old-style `%` formatting for help text internally. Literal percent signs must be escaped as `%%` in `help=` strings.

### Commands / Code

```python
parser.add_argument(
    "--with-ci",
    action="store_true",
    help="Compute 95%% bootstrap confidence intervals for PnL, Sharpe, win rate, and Brier",
)
```

```bash
python3 -m kalshi_agent.backtest_probe --help
```

---

## BLS CPI release HTML keeps key metrics split across two `<pre>` blocks
**Date:** 2026-03-21
**Context:** kalshi-agent CPI parser implementation
**Tags:** bls, cpi, html, parser, regex, fixtures

### Problem / Observation

The BLS CPI release page (`/news.release/cpi.nr0.htm`) has multiple `<pre>` blocks. The first block has the headline CPI-U/Core MoM and YoY percentages, but the CPI-U index level is in a later "Not seasonally adjusted CPI measures" block. Parsing only the first block misses the index level. Also, archive pages are easiest to source from `/bls/news-release/cpi.htm`, not `cpi.toc.htm`.

### Resolution / Insight

Extract all `<pre>` blocks, parse MoM/YoY from the first summary block, and parse CPI-U `index level of ...` from the combined pre text (or explicitly second block). Use archived fixtures from `/news.release/archives/cpi_<MMDDYYYY>.htm` for multi-month coverage.

### Commands / Code

```python
pre_blocks = re.findall(r"<pre[^>]*>(.*?)</pre>", html_text, re.I | re.S)
summary_text = normalize(pre_blocks[0])
all_pre_text = " ".join(normalize(block) for block in pre_blocks)

# summary metrics
re.search(r"CPI-U\)\s+(?P<change>.+?)\s+on a seasonally adjusted basis", summary_text, re.I)

# index level metric
re.search(r"CPI-U\).*?index\s+level\s+of\s+(?P<value>[0-9]+(?:\.[0-9]+)?)", all_pre_text, re.I)
```

```bash
# discover archive links and fetch fixtures
python3 - <<'PY'
import re, urllib.request
html = urllib.request.urlopen("https://www.bls.gov/bls/news-release/cpi.htm").read().decode("utf-8", "replace")
print(sorted(set(re.findall(r"/news.release/archives/cpi_\d+\.htm", html)))[:5])
PY
```

---

## AAA national average text may include header labels between `Regular` and the first price
**Date:** 2026-03-21
**Context:** kalshi-agent AAA gas parser
**Tags:** aaa, gas, parser, regex, html, fixtures

### Problem / Observation

A parser regex that expected `Regular` immediately followed by a dollar price failed on synthetic and real AAA table layouts where grade headers (e.g., `Mid-Grade`, `Premium`) appear between `Regular` and the first numeric cell.

### Resolution / Insight

Use two-step extraction on normalized text: first match the `National average gas prices` section marker, then search for the first 3-decimal price within a bounded window after the marker. Keep a separate higher-priority pattern for `Today’s AAA National Average $X.XXX`.

### Commands / Code

```python
text = normalize_html_to_text(html_text)

primary = re.search(r"today(?:’|')s\s+aaa\s+national\s+average\s+\$?\s*(\d+\.\d{3})", text, re.I)
if primary:
    return float(primary.group(1))

marker = re.search(r"national\s+average\s+gas\s+prices", text, re.I)
if marker:
    window = text[marker.end(): marker.end() + 1000]
    first_price = re.search(r"\$?\s*(\d+\.\d{3})", window)
    if first_price:
        return float(first_price.group(1))
```

---

## `AlertDispatcher` custom transports must implement `post`, not `post_json`
**Date:** 2026-03-22
**Context:** kalshi-agent monitoring user-testing probes
**Tags:** python, testing, alerts, transport, slack, kalshi

### Problem / Observation

An ad-hoc validation probe for Slack profit-projection text instantiated `AlertDispatcher` with a custom capture transport that only exposed `post_json(...)`. The probe then failed with `IndexError: list index out of range` because no transport call was recorded.

### Resolution / Insight

`AlertDispatcher._post_webhook()` uses `transport.post(url, headers=..., body=...)` whenever a custom transport is supplied and it is not an `UrllibTransport`. For local/user-testing probes, the fake transport must implement `post(...)`; implementing only `post_json(...)` will silently miss the webhook call.

### Commands / Code

```python
from kalshi_agent.monitoring.alerts import AlertDispatcher


class CaptureTransport:
    def __init__(self):
        self.calls = []

    def post(self, url, headers=None, body=None):
        self.calls.append({"url": url, "headers": headers, "body": body})
        return {"status": 200, "body": "ok"}


transport = CaptureTransport()
dispatcher = AlertDispatcher(webhook_url="https://example.invalid/webhook", transport=transport)
dispatcher.send_daily_summary({...})
print(transport.calls[0]["body"]["text"])
```

---

## Crypto backtest scrutiny must validate window slicing, no-lookahead lags, and hedged aggregate fields
**Date:** 2026-04-04
**Context:** kalshi-agent crypto execution-aware backtest scrutiny
**Tags:** crypto, backtest, scrutiny, family-window, lookahead, hedging, ledgers

### Problem / Observation

The execution-aware crypto backtest milestone passed its feature tests while still hiding four audit-grade bugs: same-family multi-window archives were evaluated by `family_id` only, `missed_exit` rows realized quoted exits that never filled, cycle ledgers could attach a future external-reference tick (negative `reference_alignment_lag_ms`), and hedged trade rows copied top-level fill/exit fields from the primary leg only.

### Resolution / Insight

Do not treat passing backtest tests as sufficient here. Validate/fix the engine against three invariants:
1. archive rows must be filtered by both `family_id` **and** the current window bounds before replay;
2. decision context must never use a future reference row (`reference_alignment_lag_ms` should stay non-negative or the slice must be downgraded/skipped);
3. hedged trade-level aggregates must either sum across legs or be explicitly defined as leg-specific fields, and `missed_exit` must remain unexecuted/open (or use a declared settlement fallback) rather than realizing quoted PnL.

### Commands / Code

```bash
python3 -m pytest tests/test_crypto_backtest_core.py tests/test_crypto_execution_model.py tests/test_crypto_hedge_accounting.py -v --tb=short
```

```python
# Family-window replay must use window-scoped rows, not the whole family archive.
window_rows = [
    row for row in family_rows
    if window_start_utc <= row_timestamp(row) < window_end_utc
]

# Decision context should never look ahead.
assert cycle_row["reference_alignment_lag_ms"] is None or cycle_row["reference_alignment_lag_ms"] >= 0

# Hedged trade aggregates should reconcile from leg rows.
assert trade_row["contracts_filled"] == sum(leg["contracts_filled"] for leg in trade_row["legs"])

# Missed exits should not realize quoted prices unless a declared fallback executed.
if trade_row["exit_mode"] == "missed_exit":
    assert trade_row["fill_basis"] == "none"
    assert trade_row["net_pnl_usd"] in (None, 0.0)
```

```bash
python3 - <<'PY'
from kalshi_agent.monitoring.alerts import AlertDispatcher

class CaptureTransport:
    def __init__(self):
        self.calls = []
    def post(self, url, headers=None, body=None):
        self.calls.append({"url": url, "headers": headers, "body": body})
        return {"status": 200, "body": "ok"}

transport = CaptureTransport()
dispatcher = AlertDispatcher(webhook_url="https://example.invalid/webhook", transport=transport)
dispatcher.send_daily_summary({"alert_type": "strategy_event", "event_type": "paper_trade"})
print(transport.calls[0]["body"]["text"])
PY
```

```bash
curl -sSL "https://gasprices.aaa.com/" -o /tmp/aaa_live.html
curl -sSL "https://web.archive.org/web/20250115000000/https://gasprices.aaa.com/" -o /tmp/aaa_wayback.html
rg -n "Today.?s AAA National Average|National average gas prices|\$[0-9]+\.[0-9]{3}" /tmp/aaa_*.html
```

---

## BLS CPI pages expose release time as an embargo line, not `For release ...`
**Date:** 2026-03-22
**Context:** kalshi-agent historical archive collection
**Tags:** bls, cpi, html, timestamp, parsing

### Problem / Observation

Historical BLS CPI HTML fixtures did not include the expected `For release ...` sentence. Release time instead appeared as an embargo header line like `Transmission of material in this release is embargoed until 8:30 a.m. (ET) Tuesday, January 13, 2026`, causing timestamp extraction to fail.

### Resolution / Insight

Extract timestamp from the generic time/date pattern (`8:30 a.m. (ET) ... Month DD, YYYY`) regardless of leading phrase, then convert from `America/New_York` to UTC.

### Commands / Code

```python
_BLS_RELEASE_PATTERN = re.compile(
    r"(?P<hour>\d{1,2}):(?P<minute>\d{2})\s*"
    r"(?P<ampm>a\.m\.|p\.m\.)\s*\(ET\),?\s*"
    r"(?:[A-Za-z]+,\s*)?"
    r"(?P<month>[A-Za-z]+)\s+(?P<day>\d{1,2}),\s*(?P<year>\d{4})",
    re.IGNORECASE,
)

local_dt = datetime(..., tzinfo=ZoneInfo("America/New_York"))
release_utc = local_dt.astimezone(UTC)
```

```bash
rg -n "Transmission of material in this release is embargoed|8:30 a\.m\. \(ET\)" tests/fixtures/bls_cpi_*.html
```

---

## Release backtest replays should use `price_previous_dollars` for consensus
**Date:** 2026-03-22
**Context:** kalshi-agent data-release backtester
**Tags:** kalshi, backtest, orderbook, consensus, replay, cpi, gas

### Problem / Observation

Historical candlestick snapshots often had `yes_bid_close_dollars=0.0` and `yes_ask_close_dollars=1.0` near settlement. Reconstructing an orderbook with both levels made implied probability collapse to ~0.50 (`yes/(yes+no)`), which created fake edges in replay.

### Resolution / Insight

For replay consensus, expose a single YES level from `price_previous_dollars` and leave NO empty. Keep bid/ask fields only for entry-fill simulation. This preserves historical market belief while still allowing realistic side-specific fill prices.

### Commands / Code

```python
yes_probability = _first_non_none(
    snapshot.get("price_previous_dollars"),
    snapshot.get("yes_bid_close_dollars"),
    snapshot.get("yes_ask_close_dollars"),
)

orderbook = {
    "orderbook": {
        "yes": [[yes_probability, 100]] if yes_probability is not None else [],
        "no": [],
    }
}
```

```bash
python3 -c "from kalshi_agent.backtest.release_backtest import run_default_release_backtests; import json; r=run_default_release_backtests(); print(json.dumps({'cpi': r['cpi']['metrics'], 'gas': r['gas']['metrics']}, indent=2))"
```

---

## TradingLoopRunner mock mode logs `TRADE` unless `paper_trader` is attached
**Date:** 2026-03-22
**Context:** kalshi-agent gas paper runner + tests
**Tags:** kalshi, trading-loop, mock-mode, paper-trading, tests, observability

### Problem / Observation

In `TradingLoopRunner._execute_signal`, mock mode alone does not guarantee `PAPER_TRADE` observability output. If `settings.kalshi_mock_mode=True` but `paper_trader` is `None`, the runner uses `kalshi.place_order(...)` and logs `action_taken="TRADE"` (still mock-safe, but different log semantics).

### Resolution / Insight

Attach `PaperTrader` whenever tests or runners expect paper-trade semantics/logging. For gas paper runner construction, always instantiate `PaperTrader` and force `KALSHI_MOCK_MODE=true`.

### Commands / Code

```python
runner = TradingLoopRunner(
    settings=Settings(kalshi_mock_mode=True),
    kalshi=kalshi,
    strategy=strategy,
    risk_manager=risk_manager,
    trade_store=TradeStore(db_path=db_path),
    paper_trader=PaperTrader(starting_balance=1000.0),
)
```

```bash
python3 -m pytest tests/test_trading_loop.py -v --tb=short -k gas
```

---

## AlertDispatcher custom transports must implement `post`, not `post_json`
**Date:** 2026-03-22
**Context:** kalshi-agent Slack formatting probes
**Tags:** slack, alerts, transport, testing, probes, kalshi

### Problem / Observation

An ad-hoc local Slack probe used a fake transport with `post_json(...)`, then called `AlertDispatcher.send_daily_summary(...)` and tried to inspect captured webhook calls. No call was recorded and the probe crashed with `IndexError: list index out of range`.

### Resolution / Insight

`AlertDispatcher._post_webhook()` sends through `transport.post(...)` for any custom transport that is not the built-in `UrllibTransport`. For local capture probes and tests, implement `post(url, headers, body)` on the fake transport (matching `_FakeTransport` in `tests/test_monitoring.py`), then inspect `body["text"]`.

### Commands / Code

```python
class CaptureTransport:
    def __init__(self):
        self.calls = []

    def post(self, url, headers=None, body=None, timeout=None):
        self.calls.append({
            "url": url,
            "headers": headers or {},
            "body": body,
            "timeout": timeout,
        })
        return {"status": 200, "body": "ok"}

transport = CaptureTransport()
dispatcher = AlertDispatcher(
    webhook_url="https://example.invalid/webhook",
    transport=transport,
)
dispatcher.send_daily_summary(summary)
print(transport.calls[0]["body"]["text"])
```

```bash
python3 -m pytest tests/test_monitoring.py -v --tb=short -k profit_projection
```

---

## Fixed idle poll sleeps can overshoot short release activation windows
**Date:** 2026-03-29
**Context:** kalshi-agent gas paper runner on EC2
**Tags:** scheduler, polling, gas, ec2, release-window, observability

### Problem / Observation

The gas paper runner was healthy on EC2 and logging once per minute while idle, but its last idle cycle before the Monday release happened at `2026-03-23T12:59:20Z`. The next cycle did not run until `2026-03-23T13:00:20Z`, even though the scheduler's pre-release activation offset is only 30 seconds for a `13:00:00Z` gas release.

### Resolution / Insight

`WeeklyGasScheduler.get_decision()` correctly reports `active=True` only inside `[release_time - 30s, release_time + 30s]`, but `TradingLoopRunner._sleep_until_next_cycle()` sleeps the full prior idle interval (60s). If the previous idle cycle lands before `active_start`, the loop can sleep straight past the intended pre-release activation boundary and only wake partway through the live window. When investigating missed release runs, compare the final idle timestamp to the scheduled release time; a one-minute idle cadence can consume most of a 60-second active window.

### Commands / Code

```bash
ssh -i /Users/dansullivan/workspace/kalshi-agent/.secrets/ec2_key ec2-user@<ec2-ip> \
  'docker logs --since 2026-03-23T12:59:00Z --until 2026-03-23T13:01:00Z kalshi-agent 2>&1 | egrep "2026-03-23T12:59:20|2026-03-23T13:00:20|2026-03-23T13:00:30"'
```

```text
2026-03-23T12:59:20Z -> scheduler_idle, poll_interval_seconds=60.0
2026-03-23T13:00:20Z -> scheduler_active=true, seconds_past_release=20.633
2026-03-23T13:00:30Z -> scheduler_idle for next week's gas release
```

---

## Manual EC2 container replacements can bypass Secrets Manager env injection
**Date:** 2026-03-29
**Context:** kalshi-agent EC2 gas runner rollout
**Tags:** aws, ec2, docker, secretsmanager, slack, env, deployment

### Problem / Observation

The Terraform/user-data path correctly fetches `kalshi-agent/credentials` from AWS Secrets Manager and passes `SLACK_WEBHOOK_URL` into `docker run`, but later gas-only EC2 rollouts replaced the container manually with `docker run ... -e KALSHI_MOCK_MODE=true ... python -m kalshi_agent.gas_paper_runner`. That ad-hoc replacement preserved volumes and command overrides but skipped the secret-export step, so runtime env vars like `SLACK_WEBHOOK_URL` disappeared even though the secret still existed in AWS.

### Resolution / Insight

If a deployment was originally bootstrapped by EC2 `user_data`, do not assume later manual `docker run` replacements inherit the same env wiring. For follow-up rollouts, either (a) recreate the container through the same Secrets Manager fetch path before `docker run`, or (b) parameterize the infra/bootstrap so the desired command override still flows through the managed secret injection path.

### Commands / Code

```text
Managed path:
- infra/user_data.sh.tftpl fetches AWS Secrets Manager secret JSON
- exports KALSHI_API_KEY_ID / KALSHI_PRIVATE_KEY / SLACK_WEBHOOK_URL / NCEI_TOKEN
- runs docker with --env SLACK_WEBHOOK_URL

Drifted manual path observed in rollout evidence:
- docker rm -f kalshi-agent
- docker run -d --name kalshi-agent --restart unless-stopped -e KALSHI_MOCK_MODE=true -v <data>:/app/data -v <secrets>:/app/.secrets <image> python -m kalshi_agent.gas_paper_runner

Result:
- gas-only runner command updated successfully
- Secrets Manager-backed env vars were no longer guaranteed inside the container
```

---

## Weekly gas idle loops use `gas_weekly_schedule` reason, not `scheduler_idle`
**Date:** 2026-03-29
**Context:** kalshi-agent gas runtime Slack outcome filtering
**Tags:** gas, scheduler, slack, alerts, observability, skip-filter

### Problem / Observation

Runtime Slack suppression initially filtered only `reason="scheduler_idle"` and `reason="duplicate_release"`. In gas mode, `WeeklyGasScheduler.get_decision()` sets `reason="gas_weekly_schedule"` even when inactive, so idle heartbeat cycles still emitted `paper_skip` Slack messages every minute.

### Resolution / Insight

Treat scheduler-heartbeat skip reasons as a dedicated non-notifiable set (including `gas_weekly_schedule`, `scheduler_idle`, `duplicate_release`, and `schedule_unavailable`) in the runtime outcome notifier. Keep `_log_observability` unchanged so local structured logs still capture heartbeat/duplicate context. Also label mock-mode BUY NO executions as `PAPER_TRADE` so they flow through the same paper-trade Slack path as BUY YES.

### Commands / Code

```python
_NON_NOTIFIABLE_RUNTIME_SKIP_REASONS = frozenset(
    {"scheduler_idle", "gas_weekly_schedule", "duplicate_release", "schedule_unavailable"}
)

if action_taken == "SKIP" and reason in _NON_NOTIFIABLE_RUNTIME_SKIP_REASONS:
    return

action_taken = "PAPER_TRADE" if self.settings.kalshi_mock_mode else "TRADE"
```

```bash
python3 -m pytest tests/test_trading_loop.py -v --tb=short -k slack
```

---

## Mixed-asset discovery cannot reuse a BTC-only reference manifest
**Date:** 2026-04-04
**Context:** kalshi-agent crypto market-discovery contracts
**Tags:** crypto, discovery, manifests, reference-data, lineage, schemas

### Problem / Observation

Discovery accepted `eth-usd-15m` while publishing only BTC reference contracts/surfaces and mapping ETH families to BTC source IDs. This made settlement/reference lineage internally inconsistent and broke scrutiny.

### Resolution / Insight

When any non-BTC family is accepted, publish asset-specific reference profiles (candidate sources + primary/fallback + methodology) and map each accepted family to the profile for its own asset. Also make downstream run-manifest contracts multi-family aware (`families_evaluated`, `family_windows_evaluated`) instead of singular `family_id`.

### Commands / Code

```python
profiles = source_manifest["reference_profiles"]
for family in catalog["accepted_families"]:
    profile = profiles[family["asset"]]
    mapping = family["reference_methodology_mapping"]
    assert mapping["asset"] == family["asset"]
    assert mapping["primary_source_id"] == profile["selected_primary_source_id"]
    assert mapping["fallback_source_id"] == profile["selected_fallback_source_id"]
```

```bash
python3 -m pytest tests/test_crypto_discovery.py -v --tb=short
python3 -m kalshi_agent.crypto_research.discovery --output-dir data/crypto_research/discovery-reference-consistency-check
python3 -m pytest tests/ -v --tb=short
```



## Keyword-only helper refactors can fail at runtime when old positional calls remain
**Date:** 2026-04-04
**Context:** Python backtest execution-model refactor
**Tags:** python, keyword-only, refactor, runtime-error, backtest

### Problem / Observation

During a crypto backtest execution-model refactor, a helper was changed to keyword-only parameters (`def _spread_width_cents(*, quote_context, entry_side)`), but an existing call site still passed the first argument positionally. Tests failed at runtime with `TypeError: _spread_width_cents() takes 0 positional arguments but 1 positional argument (and 1 keyword-only argument) were given`.

### Resolution / Insight

When converting helper APIs to keyword-only signatures, update every call site to explicit keywords immediately and run focused tests before full-suite validation.

### Commands / Code

```python
# Helper signature
def _spread_width_cents(*, quote_context: dict[str, Any], entry_side: str) -> int:
    ...

# Correct call style
entry_spread_cents = _spread_width_cents(
    quote_context=fill_row.get('quote_context', {}),
    entry_side=entry_side,
)
```

```bash
python3 -m pytest tests/test_crypto_execution_model.py tests/test_crypto_backtest_core.py -v --tb=short
```

