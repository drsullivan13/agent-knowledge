## Verify an embedded public appointment widget
**Date:** 2026-11-19
**Context:** Harbor, public HighLevel calendar, Playwright
**Tags:** harbor, booking, dates, mobile, forms

### Problem / Observation

The Elderly Savior booking page contains two distinct contact forms and calendar cells from adjacent months. A date text matcher can find a hidden duplicate, and an input selected from the entire page can belong to the general inquiry form. The appointment step and contact fields hydrate separately.

### Resolution / Insight

Exercise the mobile browser via the exact calendar widget. Choose visible, in-month date cells; verify the selected day, Eastern timezone, meeting type and duration before completing fields. Compare both checkbox accessibility labels and visible terms. The public widget double-encodes an ampersand in one label. Keep the separate optional marketing box unchecked and defer user-visible human verification to the attendee.

### Commands / Code

```js
const calendar = page.locator("#calendarAppointmentBookingMain");
await calendar.locator("td.vdpCell.selectable:visible").first().waitFor();
const cell = calendar.locator('td[data-id="2026-11-30"]:not(.outOfRange)');
await cell.click();
await calendar.getByRole("heading", {name:"Choose Time Slot"}).waitFor();
const labels = await calendar.locator("li.widgets-time-slot").allInnerTexts();
```

A preview can verify available times. Only a matching provider confirmation can verify an appointment.
