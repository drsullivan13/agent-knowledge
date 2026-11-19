## Verify a live form button click without creating another booking
**Date:** 2026-11-19
**Context:** Harbor, Playwright browser worker
**Tags:** booking, duplicate-prevention, playwright

### Problem / Observation
A durable pre-submit claim proves that a worker reached the final step. It cannot establish that the button received a click or the provider accepted a booking.

### Resolution / Insight
For a diagnostic with approved data, use a fresh browser context with service workers blocked and no open WebSockets. Prepare the real form through the normal adapter. In its before-click callback block all requests, take the context offline, then add a capture-phase listener for the actual button. Only then allow the original worker click to proceed. Record the trusted click and an attempted booking POST separately. Neither signal is a provider receipt. Do not automatically retry an earlier uncertain live submission.

### Commands / Code
```js
const context = await browser.newContext({ serviceWorkers: "block" });
await context.routeWebSocket("**/*", socket => socket.close());
// Run immediately before the worker's final click:
await context.route("**/*", route => route.abort("internetdisconnected"));
await context.setOffline(true);
await page.evaluate(() => document.addEventListener("click", event => {
  if (event.target.closest("button")?.innerText.trim() === "Schedule meeting")
    console.info("scheduling-button-click", event.isTrusted);
}, true));
```
