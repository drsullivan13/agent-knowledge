## Zod gate over Python canonical JSON: never compare dicts via JSON.stringify
**Date:** 2026-09-07
**Context:** TypeScript Zod schema validating a JSON artifact produced by Python canonical dumps (fantasy-researcher)
**Tags:** zod, typescript, python, json, key-order, schema-validation

### Problem / Observation

A Zod `.superRefine` reconciliation check compared a counts map with
`JSON.stringify(declared) !== JSON.stringify(computed)` and failed even though
the maps were equal: Python's canonical writer sorts object keys, while the
JS-side object was built in ledger-iteration order. JSON.stringify is
key-order-sensitive, so identical maps with different key order mismatched.

### Resolution / Insight

Compare dictionaries semantically (same key count and per-key equality), never
by serialized string. This matters whenever a JS gate re-validates artifacts
emitted by another language's canonical serializer. Arrays are fine to compare
as strings if both sides sort them first.

### Commands / Code

```ts
const countsReconcile =
  Object.keys(declaredCounts).length === Object.keys(counts).length &&
  Object.entries(counts).every(([k, v]) => declaredCounts[k] === v);
```
