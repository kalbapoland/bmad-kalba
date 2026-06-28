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

## From BL-001 (device-language consistency) scoping — 2026-06-28

- **ESLint i18n guardrail (deferred Goal B).** Stand up ESLint (none exists in
  `frontend-kalba` today — no config, dep, or lint script) and add
  `eslint-plugin-i18next` with `no-literal-string` to flag hardcoded JSX/Alert
  UI strings in CI, so the mixed-language regression can't silently return.
  Scope the config to just the i18next rule to avoid surfacing unrelated lint
  noise on a codebase with no prior linting. Split out from the BL-001
  translation fix (user chose Split). Follow-up spec after
  `spec-bl-001-i18n-device-language` lands.

## From BL-001 (device-language consistency) review — 2026-06-28

All Low severity; pre-existing and outside the BL-001 frozen boundary (which
forbids changing detection logic / backend). Surfaced by the Edge-Case Hunter.

- **First-frame language flash on non-English cold start (`src/lib/i18n.ts`).**
  `init` runs synchronously with `lng: 'en'` while device language is detected
  in a later async IIFE that calls `changeLanguage('pl')`. On a Polish device's
  cold start, any string painted before the switch flashes English (usually
  masked by the splash). Fix: detect synchronously before `init`
  (`Localization.getLocales()` and the web `navigator` path are both sync), or
  gate first render until language resolves. Touches detection logic →
  out of scope for BL-001.
- **Server-supplied error text bypasses i18n (`AuthScreen.tsx`).** A backend
  string `detail` / password-validation `item.msg` is shown verbatim, so a
  Polish user can see an English server message. Pre-existing. Fix (if desired):
  map known server codes → translation keys. Requires backend coordination →
  out of scope for the frontend-only BL-001.
- **`pickSupported` splits language tag on `-` only (`src/lib/i18n.ts`).** A
  non-BCP-47 tag like `pl_PL` would fall back to `en`. Won't trigger in practice
  (expo-localization returns hyphenated `languageTag` + bare 2-letter code), so
  negligible; noted for completeness. Detection logic → out of scope for BL-001.
