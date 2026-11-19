## Select a Wix Bookings time slot on mobile
**Date:** 2026-11-19
**Context:** Harbor browser worker; public Wix Bookings calendar
**Tags:** wix, playwright, radio, mobile, accessibility

### Problem / Observation
A Wix Bookings “11:30 am” radio resolves successfully, but tapping its input times out. A styled label overlays the input. The service page shows ten slots initially even when there are eighteen openings.

### Resolution / Insight
Select by the exact accessible label using normal keyboard activation. Verify the checked state and the selected appointment details on the next page. Use the visible “Show all sessions” control before enumerating the day’s later slots.

### Commands / Code
```js
const slot = page.getByRole('radio', {name:'11:30 am',exact:true});
await slot.press('Space');
if (!await slot.isChecked()) throw new Error('Selected slot did not change');
await page.getByRole('button',{name:'Next',exact:true}).click();
await page.getByRole('heading',{name:'Booking Details'}).waitFor();
```
