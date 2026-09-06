## Diagnose Vercel runtime, asset, and Git authentication failures separately
**Date:** 2026-09-05
**Context:** Vercel Git deployments, Chromium, GitHub CLI
**Tags:** vercel, deployment, environment-variables, secrets, chromium, orb, cdn, github, gh, authentication

### Problem / Observation

A Vercel Git deployment can reach `READY` even though requests fail at runtime
after code externalizes previously bundled secrets. Separately, Chromium can
block HTTPS fantasy-logo requests with ORB when the third-party host returns an
unexpected response. Git pushes can also fail when invalid `GH_TOKEN` or
`GITHUB_TOKEN` environment variables override an otherwise valid GitHub CLI
keyring login.

### Resolution / Insight

Treat these as independent deployment layers:

1. For a `READY` deployment with failing requests, inspect runtime errors,
   add every newly required production secret through stdin with the sensitive
   flag, and redeploy the existing deployment to production.
2. Do not trust arbitrary third-party logo URLs merely because they use HTTPS.
   Allowlist trusted CDN hostnames, render team initials on rejection or image
   failure, and wait for delayed image requests before declaring browser
   verification complete.
3. If `gh auth status` succeeds from the keyring but Git operations fail, remove
   overriding token variables and configure Git from the keyring login. If the
   active OAuth token lacks `workflow` scope, pushing over SSH can use the
   account's SSH authorization instead.

### Commands / Code

```bash
# Inspect application/runtime output, not only deployment state.
vercel inspect <deployment-url-or-id> --logs

# Supply values via stdin; never place secret values in command history.
printf '%s' "$SECRET_VALUE" |
  vercel env add NAME production --force --sensitive

# Rebuild the existing deployment with the corrected production environment.
vercel redeploy <deploymentId> --target production
```

```ts
const TRUSTED_LOGO_HOSTS = new Set(["trusted-cdn.example"]);

export function safeLogoUrl(rawUrl: string | null): string | null {
  if (!rawUrl) return null;
  try {
    const url = new URL(rawUrl);
    return url.protocol === "https:" && TRUSTED_LOGO_HOSTS.has(url.hostname)
      ? url.href
      : null;
  } catch {
    return null;
  }
}

// Render initials when safeLogoUrl(...) is null or the image emits onError.
// During browser checks, wait for network idle (or the expected logo response)
// so delayed ORB failures are observed.
```

```bash
unset GH_TOKEN GITHUB_TOKEN
gh auth status
gh auth setup-git

# Alternative when an HTTPS OAuth token lacks workflow scope.
git remote set-url origin git@github.com:<owner>/<repo>.git
git push
```
