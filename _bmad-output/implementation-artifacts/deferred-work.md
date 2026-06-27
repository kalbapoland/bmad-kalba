# Deferred Work

Findings surfaced during reviews that are intentionally NOT addressed in their
originating change. Pick up later as focused work.

## From BL-004 (call orientation) review — 2026-06-27

- **Landscape layout polish for the call grid (3+ participants).** In landscape,
  the 2-column grid (`call.tsx` `gridCell` `minHeight: 200`, no scroll) can exceed
  the reduced landscape height and overlap the bottom control bar. The BL-004 spec
  scoped responsive layout to **BL-003** (self-view positioning / preferred layout),
  so a proper landscape-aware call layout should be done there. Severity: Medium
  (only 3+ participant calls in landscape). Source: Edge-Case Hunter EC3.
- **iOS re-rotation timing (manual-verify).** After leaving a call held physically
  in landscape, `lockAsync(PORTRAIT_UP)` may not snap the next screen back to
  portrait until the next sensor event on some iOS versions. Confirm on a real
  device during BL-004 manual testing. Severity: Low. Source: Blind Hunter.
