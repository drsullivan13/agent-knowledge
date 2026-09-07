## Frozen clocks hide live-time bugs: never validate an event timestamp against a pre-event clock capture
**Date:** 2026-09-07
**Context:** Python data pipeline with injected clocks (fantasy-researcher); live HTTP fetches
**Tags:** testing, clock-injection, determinism, timestamps, live-vs-fixture, bugs

### Problem / Observation

An offline test suite using FrozenClock passed 240+ cases, yet every live
fetch failed with "retrievedAt is after the generation time". The refresh
state machine had been handed a `generated_at` timestamp captured BEFORE
the fetch; a live fetch takes real wall-clock time, so the retrieval
timestamp recorded after the fetch was always "in the future" relative to
that pre-fetched reference. With a FrozenClock both timestamps are
identical, so the bug was invisible offline.

### Resolution / Insight

Validate an event's timestamp against a clock reading taken at or after the
event, never before it. In a pipeline that emits an artifact at the end of a
run, per-item classification should use the classification moment (post-
event clock), and final artifact assembly should re-derive any time-derived
field against the artifact's own generatedAt, which is guaranteed to be at
or after every per-item timestamp. Design tests so the same property is
exercisable live: any test that injects a pre-event timestamp for an event
that consumes real time is testing a fiction.

### Commands / Code

```python
# Before (bug): generated_at captured pre-fetch; live retrieval is later.
entry = refresh_source(spec, adapter, store, clock, format(clock.now()))

# After: classify against the post-fetch classification moment.
retrieved_at = format_rfc3339(clock.now())          # after the fetch
classify_freshness(retrieved_at=retrieved_at, ...,
                   generated_at=format_rfc3339(clock.now()))  # classification moment
# Ledger assembly re-derives freshness against the final artifact generatedAt.
```
