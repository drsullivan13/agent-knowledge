## Verify browser checkpoint durability under process termination
**Date:** 2026-11-18
**Context:** Mobile-emulated Chromium with an isolated disposable profile
**Tags:** chromium, storage, transcript, crash-recovery, playwright

### Problem / Observation

A fresh Chromium profile lost a localStorage session despite two completed writes when its dedicated browser process terminated roughly 2.5 seconds after launch. A settled profile recovered all 1,000 checkpointed synthetic turns after six seconds. One newly injected turn was lost. A pagehide test exercises a different path.

### Resolution / Insight

Launch a dedicated Chromium child with a temporary user-data directory and connect Playwright to its DevTools endpoint. Use the default persistent context. Record the last checkpoint and append a tail marker. Terminate the owned browser process without a graceful close, then reopen the same profile and compare both the recovered storage and rendered history. Use fresh and settled profiles. Never signal unrelated browser processes.

### Commands / Code

```js
const context = browser.contexts()[0];
const exited = new Promise(resolve => child.once('exit', resolve));
child.kill('SIGKILL');
await exited;
```

Measure callback time, local writes and animation frame gaps separately. Report acknowledgement at the storage API separately from recovery after process death.
