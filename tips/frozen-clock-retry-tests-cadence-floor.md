## Frozen-clock transport tests: retry elapsed time is max(backoff, cadence)
**Date:** 2026-09-07
**Context:** Deterministic retry/cadence test design (Python, injected clock + sleep)
**Tags:** testing, retries, rate-limit, cadence, determinism, backoff

### Problem / Observation

Asserting exact elapsed time after a retried fetch kept failing by a small
delta: the engine slept the computed backoff delay, but the retry attempt then
waited the remaining minimum-cadence gap for the same host, so the observed
clock advance was larger than the backoff delay alone whenever the delay was
below the cadence floor.

### Resolution / Insight

When a retry engine enforces both a per-host minimum cadence and backoff, the
elapsed time to attempt N+1 is `max(backoff_delay(N), min_cadence)` (plus prior
delays), not just the backoff. In deterministic tests with a frozen clock and a
sleep that advances it, compute the expected schedule with the same formula the
engine uses. Keeping cadence applied to retry attempts (not only logical
requests) is itself the right production behavior: a hammered host sees the
same spacing during error storms.

### Commands / Code

```python
backoff = backoff_delay(cfg, source_id, attempt=1, jitter_seed)
expected_elapsed = max(backoff, cfg.min_cadence_seconds)
assert frozen_clock.now() == start + timedelta(seconds=expected_elapsed)
```
