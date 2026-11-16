## Verify an embedded consultation calendar
**Date:** 2026-11-16
**Context:** Harbor, New Jersey home-care scheduling, HighLevel
**Tags:** booking, timezone, highlevel, calendar, receipt

### Problem / Observation
A care agency can display dated consultation openings, a separate inquiry form and a message promising staff follow-up on the same page. A calendar's configured timezone can also differ from the timezone selected by the visitor.

### Resolution / Insight
Verify the provider identity, rendered event name and duration, selected day, displayed timezone and concrete starts together in the live browser. Read the appointment-specific status after an authorized write. A successful HTTP response alone cannot distinguish a pending consultation from a confirmed one. Recover an ambiguous write by reading the matching event before creating another.

### Commands / Code
From the Harbor dashboard checkout, run the adapter and phone contract checks with the project-local TypeScript loader:
```sh
node --import tsx --test tests/highlevel-booking.test.ts tests/calendly-booking.test.ts tests/phone-consultation.test.ts
```
Compare the read-only browser evidence with the office record in `research/north-jersey/consultations/booking-routes.md`. Any listed openings are historical observations.
