## Recognize a booking label beside an inline SVG
**Date:** 2026-11-19
**Context:** Harbor, Playwright, public calendar widget
**Tags:** playwright, regression, whitespace, calendar, selector

### Problem / Observation

The scheduler showed Google Meet as a selectable meeting location. Its location container includes whitespace around an SVG before the visible label. A regular expression anchored to the literal label timed out in Playwright. The original browser fixture omitted the icon and passed.

### Resolution / Insight

Allow surrounding whitespace while retaining exact matching for the full label. Add the icon structure to the fixture and confirm that the original implementation fails the test before applying the fix.

### Commands / Code

```js
root.locator(".meeting-location-container").filter({ hasText: /^\s*Google Meet\s*$/ });
```

```sh
node --test scripts/public-form/elderly-savior.test.mjs
```
