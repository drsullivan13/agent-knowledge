## Verify private fact history after an account switch
**Date:** 2026-11-19
**Context:** Harbor account dashboard, React, Chromium
**Tags:** harbor, privacy, owner, inquiry, browser

### Problem / Observation

The form disabled saving on owner mismatch, but cached facts could still appear if the viewer returned to the original ZIP. Database policies do not govern data already loaded into a tab.

### Resolution / Insight

Verify the owner's identity before rendering saved history. On an account switch, exercise both the disabled save button and the original ZIP, and check that old rows disappear. A locally retained draft has separate behavior.

### Commands / Code

Run from the dashboard repository with its locked dependencies installed.

```sh
node --import tsx --test tests/private-inquiry-facts-ui.test.ts
```
