## Verify Playwright page functions through the repository test entry point
**Date:** 2026-11-19
**Context:** Harbor, tsx, Playwright
**Tags:** typescript, tsx, esbuild, chromium, evaluate, tests

### Problem / Observation

A receipt fixture passed with Node running its MJS file directly but timed out under the normal TS test entry. A nested function declaration in a page.evaluate callback was transformed by tsx/esbuild into a reference to its host-side __name helper. That helper is absent in Chromium, which throws a ReferenceError. A classifier with bounded retries can conceal this error behind an unverified result.

### Resolution / Insight

Keep page functions self-contained after transpilation. An object method avoids the injected named-function assignment. During fixture debugging, surface the error from the page function. Verify via the parent TS suite importing the adapter, in addition to a direct run.

### Commands / Code

```js
const result = await page.evaluate(() => {
  const rendered = { text(node) { return node.innerText; } };
  return rendered.text(document.body);
});
```

```sh
node --test scripts/public-form/elderly-savior.test.mjs
node_modules/.bin/tsx --test tests/public-form-worker.test.ts
```
