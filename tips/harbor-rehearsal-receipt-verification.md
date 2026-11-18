## Verify Harbor rehearsal receipts locally
**Date:** 2026-11-18
**Context:** Harbor dashboard, Node, tsx, PostgreSQL
**Tags:** harbor, pglite, tsx, sandbox, jsonpath

### Problem / Observation

The tsx CLI starts a local IPC listener even for database-only tests. A managed sandbox can reject it before any test runs. PostgreSQL also rejects regex repetition counts above 255 in JSONPath predicates.

### Resolution / Insight

Use Node’s test runner with the tsx loader. Validate structured JSON using strict JSONPath to avoid implicit array unwrapping. For a 240-character field use the bounded repeat below. For limits above 255 use a separate length check.

### Commands / Code

```sh
node --import tsx --test tests/outreach-rehearsal-receipts-db.test.ts
node_modules/.bin/tsc --noEmit --incremental false
```

```sql
select '["example"]'::jsonb @? 'strict $[*] ? (@ like_regex "^.{1,240}$" flag "s")';
```
