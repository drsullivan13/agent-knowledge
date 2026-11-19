## Reconcile browser click and network acknowledgement in either order
**Date:** 2026-11-19
**Context:** Harbor, Playwright, browser-worker diagnostics
**Tags:** playwright, chromium, event-order, acknowledgement, receipts

### Problem / Observation

A Linux CI fixture observed the trusted booking button click but missed the mock HTTP 201. The implementation subscribed to request/response events only inside a console click-marker callback. Chromium can deliver the network signal before the corresponding console signal reaches the driver, even when the click caused the request.

### Resolution / Insight

Arm the three observers after acquiring the durable submit claim and before the one click. Correlate a specific origin, path, POST, intended frame, and corresponding request object. Retain the HTTP status only once both signals are known, irrespective of arrival order. Await persistence before closing the page. The status code establishes transport acknowledgement only.

### Commands / Code

In a fixture, hold the console marker, perform the mocked booking POST and then release the marker before changing the visible result. Check that both a click and HTTP 201 were recorded and the appointment remains unconfirmed.

```sh
node --test --test-name-pattern='HTTP acknowledgement' scripts/public-form/elderly-savior.test.mjs
node_modules/.bin/tsx --test tests/public-form-worker.test.ts
```
