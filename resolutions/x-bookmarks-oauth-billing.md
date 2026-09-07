## X bookmarks OAuth2 success can still be blocked by billing
**Date:** 2026-09-07
**Context:** X API v2 / OAuth2 authorization code with PKCE / bookmarks
**Tags:** x-api, oauth2, bookmarks, billing, security

### Problem / Observation

OAuth2 `GET /2/users/me` succeeded, but a single `GET /2/users/{id}/bookmarks?max_results=5` returned HTTP `402` with `exhausted_balance`.

### Diagnosis

This is a billing blocker, not an empty bookmark collection. Successful identity lookup does not prove bookmark scopes; the billing error neither proves nor disproves sufficient scopes. xAI credentials and X API credentials are separate. An OAuth1 token pair does not satisfy the documented OAuth2 PKCE flow.

### Resolution / Insight

- Stop on `402 exhausted_balance`; do not retry or substitute public search. Ask the user to resolve X API credits before further bookmark testing.
- Follow the official bookmarks quickstart with OAuth2 PKCE and scopes `bookmark.read tweet.read users.read offline.access`.
- For secure probes, suppress callback query logging; never `source .env` (parse it as data). Keep plaintext credential directories owner-only (`0700`) and files owner-only (`0600`). Write replacements to a protected temporary file in the same directory, then atomically replace the destination.
- Keep account identity, token paths, bookmarked content, and secret values out of knowledge entries.

### Verification

Verified in the supplied session: identity lookup succeeded; one bookmark request returned `402 exhausted_balance`. No successful bookmark sample or token refresh was verified; both remained blocked by billing. After the user resolves credits, bookmark retrieval and refresh still require verification.

### Sources

- Official bookmarks quickstart: https://docs.x.com/x-api/posts/bookmarks/quickstart/bookmarks-lookup
- Official pricing: https://docs.x.com/x-api/getting-started/pricing — recheck current rates; do not rely on stale price constants.
