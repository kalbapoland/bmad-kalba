---
title: 'BL-001: App fully follows device language (remove mixed PL/EN)'
type: 'bugfix'
created: '2026-06-28'
status: 'done'
context: []
baseline_commit: '50edeaa617256ddfc56744333b79b3925bc95225' # frontend-kalba main HEAD
branch: fix/bl-001-i18n-device-language
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The app is meant to render entirely in the device language, but users on a Polish phone see a mix: translated screens show Polish while a number of **hardcoded English strings** (whole screens and stray literals) ignore i18n. Root cause is confirmed coverage gaps, not detection or missing translations — language detection works (translated parts do switch to Polish) and `pl.json` already mirrors all 204 `en.json` keys.

**Approach:** Audit every screen/component for user-facing string literals that bypass `react-i18next`, move them into `en.json` + `pl.json` under the existing namespacing conventions (with real Polish translations), and replace the literals with `t("…")`. No change to the detection logic or the supported-language set (pl/en, fallback en). Language stays **device-driven only** — no in-app switcher.

## Boundaries & Constraints

**Always:**
- Keep the existing locale architecture: `src/lib/i18n.ts` (detection + `pickSupported`), `useTranslation()` + `t("namespace.key")`, keys split across the established namespaces (`workshop.*`, `group.*`, `errors.*`, `profile_screen.*`, `home.*`, `calendar.*`, `my_kalba.*`, plus flat top-level keys).
- Every new key MUST exist in BOTH `en.json` and `pl.json` with an idiomatic Polish translation (not an English placeholder).
- Preserve interpolation/pluralization: use i18next interpolation (`t("k", { name })`) and the existing `_one/_few/_many/_other` plural pattern where counts are shown.
- Reuse existing keys when a literal duplicates one already defined (e.g. `cancel`, `edit`, `delete`, `errors.title`).

**Ask First:**
- If a screen needs user-visible copy that has no obvious Polish equivalent (domain/brand terms), flag it rather than inventing a translation.

**Never:**
- Do not add an in-app language switcher, change `SUPPORTED`, or alter the detection/fallback logic.
- Do not set up ESLint / `eslint-plugin-i18next` here — that guardrail is split out to `deferred-work.md` (Goal B).
- Do not translate non-user-facing strings (log lines, `automationId`, analytics ids, style tokens).
- Do not touch the backend.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Polish device | system locale `pl-PL` | Every screen + alert + placeholder renders in Polish | N/A |
| English device | locale `en-US` | Everything renders in English | N/A |
| Unsupported locale | locale `de-DE` | Falls back to English everywhere (unchanged) | N/A |
| Interpolated/plural text | "Remove {name}?", "{n} spots" | Localized with correct interpolation + Polish plural form | N/A |
| Reused concept | a literal equal to an existing key | Uses the existing key, no duplicate added | N/A |

</frozen-after-approval>

## Code Map

- `frontend-kalba/src/locales/en.json`, `src/locales/pl.json` -- translation stores; add new keys here (both, in lockstep).
- `frontend-kalba/src/lib/i18n.ts` -- detection (reference only; do not change).
- `frontend-kalba/app/(app)/create-workshop.tsx` -- **fully unlocalized** (0 `t()`), ~23 literals — largest offender.
- `frontend-kalba/app/(app)/workshop/call.tsx` + `workshop/call.web.tsx` -- unlocalized call UI (controls, alerts, "Connecting…", host buttons, kick copy).
- `frontend-kalba/app/(app)/workshop/edit.tsx`, `workshop/[id].tsx` -- mostly localized; ~7–10 leftover literals each.
- `frontend-kalba/app/(app)/(tabs)/profile.tsx`, `create-group.tsx`, `group/edit.tsx`, `group/[id].tsx`, `(tabs)/groups.tsx`, `(tabs)/calendar.tsx`, `app/(app)/_layout.tsx`, `src/components/AuthScreen.tsx`, `src/components/HashtagTextInput.tsx` -- scattered leftover literals.

## Tasks & Acceptance

**Execution:** (order: locale keys first per screen, then swap literals; group the leftover-only files together)
- [x] `frontend-kalba/src/locales/en.json` + `src/locales/pl.json` -- Added `create_workshop.*` (35 keys), `call.*` (incl. `people_*` plurals), `common.*` (retry/previous/next/try_again/unknown_error/server_unreachable), and auth top-level keys (tagline, password_placeholder_login, invalid_email, register_failed, invalid_credentials, signup_failed, login_failed, google_signin_failed). Idiomatic Polish; en/pl parity verified by test.
- [x] `frontend-kalba/app/(app)/create-workshop.tsx` -- Added `useTranslation()`; all literals → `t()` (numeric/`YYYY-MM-DD`/`HH:MM` format placeholders intentionally kept; `Alert "Error"` reuses `errors.title`).
- [x] `frontend-kalba/app/(app)/workshop/call.tsx` + `call.web.tsx` -- Added `useTranslation()`; localized Connecting, people-count (plural), host control labels, You/Guest/Host/Participant/Leave, Connection Error + Permissions alerts. (Kick-specific copy lives on the parallel `feat/host-kick-participant` branch — see Design Notes; localize there/on merge.)
- [x] `frontend-kalba/app/(app)/workshop/[id].tsx`, `(tabs)/profile.tsx`, `(tabs)/calendar.tsx`, `_layout.tsx`, `src/components/AuthScreen.tsx` -- Remaining literals → `t()`, reusing existing keys (`workshop.enroll/unenroll`, `profile_screen.*`, `signout`). `_layout.tsx` gained `useTranslation()`. `edit.tsx`, `create-group.tsx`, `group/edit.tsx`, `group/[id].tsx`, `groups.tsx`, `HashtagTextInput.tsx`, `forgot-password.tsx`, `reset-password.tsx` were already fully localized (verified — flagged hits were `t(key,fallback)` defaults / classNames). `MonthView`/`WeekView`/`DayView` use library `LocaleConfig` / `Intl` locale switching (correct).
- [x] `frontend-kalba/src/locales/__tests__/locales.test.ts` (new) -- en/pl key parity (allowing pl-only `_few/_many`), no empty values. Passing.

**Acceptance Criteria:**
- Given a device set to Polish, when navigating create-workshop, the call screen, edit, profile, groups, and calendar, then all visible text, placeholders, and alert dialogs are in Polish with no English leakage.
- Given a device set to English (or any unsupported locale), when using the app, then all the same surfaces render in English.
- Given a count-based string (spots/participants), when displayed in Polish, then the correct Polish plural form is used.
- Given `git grep` for hardcoded UI literals across the audited files, then none remain (see Verification).
- Given `npm test` and `tsc --noEmit`, then both pass including the new locale-parity test.

## Design Notes

Namespacing: follow the existing split — flat keys for global/auth terms, nested namespaces per area. Add `create_workshop` and `call` namespaces rather than dumping flat keys. Example:

```jsonc
// en.json / pl.json
"call": { "connecting": "Connecting…" / "Łączenie…",
          "remove_title": "Remove participant" / "Usuń uczestnika",
          "remove_confirm": "Remove {{name}} from the call?" / "Usunąć {{name}} z rozmowy?" }
```
```tsx
const { t } = useTranslation();
Alert.alert(t("call.remove_title"), t("call.remove_confirm", { name }));
```

Pluralization (Polish needs `_few`/`_many`): for counts use `t("home.spots_count", { count })` with `spots_count_one/_few/_many/_other` already-style keys.

## Verification

**Commands:**
- `cd frontend-kalba && git grep -nE '<Text[^>]*>[A-Za-z]{2,}|Alert\.alert\("|placeholder="[A-Za-z]' -- 'app/**/*.tsx' 'src/**/*.tsx' | grep -v __tests__` -- expected: no user-facing literals remain (only `t(...)` calls / non-UI matches).
- `cd frontend-kalba && npx tsc --noEmit` -- expected: clean.
- `cd frontend-kalba && npx jest` -- expected: all pass, incl. the new locale-parity test.

**Manual checks:**
- Switch the simulator/device language to Polish, then English; walk create-workshop, a live call (host + participant), edit, profile, groups, calendar; confirm no mixed-language text and no missing-key fallbacks.

## Suggested Review Order

Start with the riskiest change, end with the safety net:

1. **`src/locales/en.json` + `src/locales/pl.json`** — the source of truth. Confirm
   the new `create_workshop.*`, `call.*` (incl. `people_*` plurals), `common.*`,
   and auth keys exist in both files with idiomatic Polish. **Pay special attention
   to `create_workshop.label`** — it restores the flat top-level key that the new
   `create_workshop` namespace object would otherwise have clobbered (the review's
   one real regression, now fixed).
2. **`app/(app)/group/[id].tsx`** — the single caller repointed from the old flat
   `t("create_workshop")` to `t("create_workshop.label")`. Verifies the collision fix
   end-to-end.
3. **`app/(app)/create-workshop.tsx`** — largest swap (was 0 `t()` calls). Check the
   two interpolated keys (`schedule_label`/`schedule_hint` with `{{timezone}}`) and
   that `YYYY-MM-DD`/`HH:MM` format placeholders were intentionally left literal.
4. **`app/(app)/workshop/call.tsx` + `call.web.tsx`** — call UI + the `call.people`
   plural (`{{count}}`). Note `VideoTile` has its own `useTranslation()`. Kick-specific
   copy is intentionally absent (lives on `feat/host-kick-participant`).
5. **`src/components/AuthScreen.tsx`** — `resolveAuthError` now threads `t`; single
   call site updated.
6. **`_layout.tsx`, `workshop/[id].tsx`, `(tabs)/profile.tsx`, `(tabs)/calendar.tsx`**
   — leftover literals → existing keys; low risk.
7. **`src/locales/__tests__/locales.test.ts`** (new) — the parity safety net that
   makes future en/pl drift fail CI.

## Review Outcome (2026-06-28)

Three parallel review agents (Blind Hunter, Acceptance Auditor, Edge-Case Hunter), no shared context.

- **Patched (auto-fixed):**
  - 🔴 HIGH — `create_workshop` flat-key collision (flagged by 2 of 3 agents). The new
    `create_workshop` namespace object overwrote the existing flat label still used at
    `group/[id].tsx:278`, which would have rendered the raw key on the button. Fixed:
    added `create_workshop.label` to both locales + repointed the caller.
  - 🟢 LOW — `doHostAction` in `call.tsx` used `t` but omitted it from its dependency
    array. Fixed: deps → `[hostAction, t]`.
- **Deferred (Low, pre-existing, outside frozen boundary → `deferred-work.md`):**
  first-frame language flash, server-supplied error text bypassing i18n, `pickSupported`
  underscore-locale split. All three touch detection logic / backend, which this spec's
  **Never** list forbids changing.
- **Rejected (false positives):** Blind Hunter's "missing key" flags
  (`profile_screen.*`, `errors.title`, `workshop.enroll`, `password_placeholder_register`)
  and "`[id].tsx` lacks `useTranslation`" were diff-only blindness — all verified to
  pre-exist. Calendar `LocaleConfig` Polish strings are intentional library config.
- **Final validation:** `tsc --noEmit` clean, `jest src/locales` 3/3 pass, no residual
  flat `create_workshop` usages.
