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

