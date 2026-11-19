## Keep browser callbacks portable across Node and tsx
**Date:** 2026-11-19
**Context:** Playwright page.evaluate callbacks imported from a TypeScript test via tsx
**Tags:** playwright, tsx, browser, javascript

### Problem / Observation
A browser fixture passed under plain Node but failed under the application's TypeScript test runner. The actual page error was `ReferenceError: __name is not defined`. esbuild inserted a function-name helper into a nested arrow assigned to a variable. The serialized callback ran in the webpage without that helper.

### Resolution / Insight
Use an object method for a nested helper in an evaluate callback, or assign the event callback to the computed property that already holds its listener handle. Both forms run as ordinary JavaScript in the browser.

### Commands / Code
```js
await page.evaluate(() => {
  const view = { text(element) { return element.innerText; } };
  return view.text(document.body);
});
```

Compare a plain `node --test fixture.test.mjs` with a TypeScript entry point that imports the same fixture: `node --import tsx --test wrapper.test.ts`. Loading the MJS file as the direct entry point alone may skip the transform that triggered the bug.
