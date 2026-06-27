---
title: 'BL-004 — Device-following orientation on the video call screen'
type: 'bugfix'
created: '2026-06-27'
status: 'done'
baseline_commit: '4d9fb5858019492f913788f38d620335e730a3c3'
context:
  - '{project-root}/../frontend-kalba/CLAUDE.md'
  - '{project-root}/_bmad-output/implementation-artifacts/investigations/bl-004-investigation.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** During a video call, rotating the phone makes the camera image flip to landscape for ~1s and then snap back to portrait, because the whole app is hard-locked to portrait (`frontend-kalba/app.config.js:65`) and nothing handles rotation.

**Approach:** Let the **call screen only** follow the device (landscape when horizontal, portrait when vertical, like normal apps) while every other screen stays portrait. Make the native config permit all orientations, lock the app to portrait globally at the root layout, and unlock only while the call screen is mounted — re-locking on every exit.

## Boundaries & Constraints

**Always:** Keep all non-call screens portrait-only. Re-lock to portrait on every call-screen exit path (normal leave, error, hardware back, unmount). Guard all orientation calls with `Platform.OS !== "web"`. Use the Expo-pinned `expo-screen-orientation` version (`npx expo install`).

**Ask First:** Any change that makes screens *other than the call screen* rotatable. Adding a config plugin or touching EAS/native build config beyond the `orientation` field.

**Never:** Don't change `call.web.tsx` (web has no device rotation lock). Don't redesign the call UI or move the self-view (that's BL-003). Don't introduce a third-party orientation lib other than `expo-screen-orientation`.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Rotate to landscape on call screen | Device turned horizontal during a call | View settles into landscape and stays; video fills correctly | N/A |
| Rotate back to portrait on call screen | Device turned vertical during a call | View returns to portrait and stays | N/A |
| Leave call while in landscape | User taps hang-up / back / error while horizontal | Returns to previous screen forced back to portrait | re-lock in cleanup |
| Rotate a non-call screen | Device turned on list/profile/detail | Screen stays portrait (no rotation) | N/A |
| Web build | Running on web | No orientation calls executed; unchanged behavior | Platform guard |

</frozen-after-approval>

## Code Map

- `frontend-kalba/app.config.js` -- `orientation: "portrait"` (line 65) → must become `"default"` so iOS Info.plist / Android allow landscape at all.
- `frontend-kalba/app/_layout.tsx` -- root layout; add global portrait lock on mount (native only) so the permissive config doesn't make every screen rotatable.
- `frontend-kalba/app/(app)/workshop/call.tsx` -- native call screen; unlock on mount, re-lock portrait on unmount; ensure layout/safe-areas work in landscape.
- `frontend-kalba/package.json` -- add `expo-screen-orientation`.
- `frontend-kalba/docs/MANUAL_TEST_CHECKLIST_*.md` -- add regression/screen items for call rotation.

## Tasks & Acceptance

**Execution:**
- [x] `frontend-kalba/package.json` -- run `npx expo install expo-screen-orientation` to add the SDK-54-compatible dependency. (added `~9.0.9`)
- [x] `frontend-kalba/app.config.js` -- change `orientation: "portrait"` to `orientation: "default"` so the native build permits all orientations (required for iOS rotation).
- [x] `frontend-kalba/app/_layout.tsx` -- on mount, when `Platform.OS !== "web"`, `ScreenOrientation.lockAsync(ScreenOrientation.OrientationLock.PORTRAIT_UP)` so all screens stay portrait by default.
- [x] `frontend-kalba/app/(app)/workshop/call.tsx` -- add an effect: on mount `ScreenOrientation.unlockAsync()`; in cleanup `ScreenOrientation.lockAsync(PORTRAIT_UP)` (native-guarded). Added left/right safe-area insets to the in-call root container for landscape notch.
- [x] `frontend-kalba/docs/MANUAL_TEST_CHECKLIST_RELEASE_REGRESSION.md` (+ SCREENS) -- added P1-22/23/24 (regression) and Ekran 7 step 6 (screens). Polish, no diacritics, `- [ ]` style.

**Acceptance Criteria:**
- Given an active call in portrait, when the device is rotated horizontal, then the call view settles into landscape and remains (no snap-back).
- Given the call screen was in landscape, when the user leaves the call, then the destination screen is portrait.
- Given any non-call screen, when the device is rotated, then the screen stays portrait.
- Given `npx tsc --noEmit`, when run, then it passes with no new errors.

## Spec Change Log

- 2026-06-27 (review): Edge-Case Hunter found portrait was re-locked only on
  unmount, so a notification deep-link pushed over a live call left orientation
  unlocked. Patched call.tsx to use `useFocusEffect` (re-lock on blur), satisfying
  the "re-lock on every exit path" Always-constraint. KEEP: focus-based lock, not
  mount-based. Landscape grid clipping (3+ participants) deferred to BL-003 per the
  Never-redesign constraint (see deferred-work.md). iPad decision RESOLVED
  2026-06-27 (Bartek): accept iPad rotation — phone-first app, no
  `requireFullScreen` (preserves iPad multitasking). Known limitation: on iPad all
  screens may rotate; the runtime portrait lock only fully applies on iPhone.

## Design Notes

iOS only rotates to orientations declared in `UISupportedInterfaceOrientations`, which Expo derives from `app.config.js` `orientation`. With `"portrait"` there, `ScreenOrientation.unlockAsync()` is a no-op on iOS — hence the config must become `"default"` and portrait is re-imposed at runtime everywhere except the call screen. This corrects the investigation's "keep config portrait" note (which would not work on iOS).

The call cleanup re-lock is the key guarantee that rotation never leaks to other screens; the root-layout lock is the baseline. Both layers are intentional.

Example (call.tsx):
```ts
useEffect(() => {
  if (Platform.OS === "web") return;
  ScreenOrientation.unlockAsync();
  return () => {
    ScreenOrientation.lockAsync(ScreenOrientation.OrientationLock.PORTRAIT_UP);
  };
}, []);
```

## Verification

**Commands:**
- `cd frontend-kalba && npx tsc --noEmit` -- expected: passes, no new type errors.

**Manual checks (dev client build required — native module added, not Expo Go):**
- Join a call, rotate the device: view follows to landscape and back; leaving the call returns to portrait; list/profile/detail screens never rotate.
- Mid-call, trigger a notification deep-link (push over the call): destination screen is portrait, call returns to rotatable on back.

## Suggested Review Order

**Orientation strategy**

- Native enabler — config must permit all orientations or iOS rotation is a no-op.
  [`app.config.js:68`](../../../frontend-kalba/app.config.js#L68)

- Baseline lock — app stays portrait everywhere by default.
  [`_layout.tsx:96`](../../../frontend-kalba/app/_layout.tsx#L96)

- Core fix — focus-based unlock/re-lock so portrait restores on every exit incl. push-over.
  [`call.tsx:233`](../../../frontend-kalba/app/(app)/workshop/call.tsx#L233)

**Landscape layout**

- Side-notch safe area applied to the in-call container for landscape.
  [`call.tsx:306`](../../../frontend-kalba/app/(app)/workshop/call.tsx#L306)

**Peripherals**

- SDK-54-pinned dependency added via `expo install`.
  [`package.json:59`](../../../frontend-kalba/package.json#L59)

- Regression + screen checklist items (mandatory per repo rules).
  [`MANUAL_TEST_CHECKLIST_RELEASE_REGRESSION.md:116`](../../../frontend-kalba/docs/MANUAL_TEST_CHECKLIST_RELEASE_REGRESSION.md#L116)
