## Match the Next dev server origin in browser checks
**Date:** 2026-11-19
**Context:** Harbor dashboard, Next.js 16.4.6, local Playwright
**Tags:** nextjs, playwright, origin, localhost, hydration

### Problem / Observation

The HTML route returned 200, yet two Playwright jobs timed out waiting for interactive controls. Next printed a blocked cross-origin dev-resource message for 127.0.0.1 while advertising localhost.

### Resolution / Insight

Open the exact local origin printed by Next in Playwright. Confirm the console connects to HMR and a dashboard interaction succeeds. This keeps the repository's origin configuration intact.

### Commands / Code

In one shell from the dashboard checkout:

```sh
node_modules/.bin/next dev --webpack -p 4351
```

With Next advertising http://localhost:4351, run from the same checkout in a second shell:

```sh
PLAYWRIGHT_BASE_URL=http://localhost:4351 node_modules/.bin/playwright test tests/e2e/compact-search.spec.ts tests/e2e/regional-search.spec.ts --workers=2 --reporter=line
```
