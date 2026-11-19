## Bind an approval retry to its provider and consent
**Date:** 2026-11-19
**Context:** PostgreSQL booking APIs
**Tags:** idempotency, consent, postgres, multi-provider

### Problem / Observation

A single-provider approval RPC compared contact and window when reusing a request UUID. Once a second office was admitted, an earlier office’s receipt could be returned for that UUID if the family chose exactly the same contact and window.

### Resolution / Insight

Bind the retry to the office and consent version as well as its original fields. Keep the account rate limit separate from the office-specific duplicate check. Test reuse of the same UUID across offices and a fresh UUID with the other office’s consent.

### Commands / Code

```sql
if job.provider_id <> p_provider_id or job.consent_version <> p_consent_version
   or job.window_start <> p_window_start or job.window_end <> p_window_end
   or job.invitee_name <> p_name or job.invitee_phone <> p_phone
   or job.invitee_email <> verified_email then
  raise exception using errcode='23505', message='Request ID already used';
end if;
```

Provider, consent and window arguments are validated as non-null before this comparison.
