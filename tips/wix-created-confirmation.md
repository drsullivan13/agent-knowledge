## Observe Wix Bookings after CREATED
**Date:** 2026-11-19
**Context:** Public booking form, browser-driven Wix checkout and receipt
**Tags:** wix, playwright, booking, confirmation, hydration

### Problem / Observation
A verified Wix Bookings bulk-create response can report a successful item with status CREATED, while the same site's thank-you page is still blank or says “request received.” A genuine confirmation email may already have arrived. The browser can also paint its confirmation before Playwright has finished reading the response body.

### Resolution / Insight
Keep the scoped service/slot-matched creation reference as provisional evidence and keep sampling the visible receipt after navigation. Promote only a matching affirmative receipt or a confirmed provider booking. On deadline return pending with its original reference. Await an in-flight matching response body with a short timeout before returning an affirmative DOM receipt. Never resubmit just to clarify the outcome.

### Commands / Code
Run the offline transition fixtures from Harbor: `node --import tsx --test --test-name-pattern='CREATED' tests/public-form-worker.test.ts`. They cover both the late affirmative receipt without an on-page reference and the permanently pending shell.
