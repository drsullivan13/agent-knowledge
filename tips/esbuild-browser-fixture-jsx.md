## Resolve JSX imports in virtual browser fixtures
**Date:** 2026-11-19
**Context:** esbuild, Playwright and React component fixtures
**Tags:** esbuild, jsx, fixtures, playwright

### Problem / Observation

A dashboard test that bundled a small Next Link stub failed to resolve react/jsx-runtime. The fixture entry had a resolveDir, but its virtual-module plugin returned a second TSX file without one.

### Resolution / Insight

Set resolveDir on the virtual TSX module returned by the onLoad callback. JSX transformation injects a runtime import even when the stub contains no explicit import.

### Commands / Code

```js
if (args.path === "next/link") {
  return {
    contents: "export default function Link(props){return <a {...props}/>}",
    loader: "tsx",
    resolveDir: process.cwd(),
  };
}
```
