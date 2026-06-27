# Investigation: BL-004 — Camera image flips momentarily when rotating the device

## Hand-off Brief

1. **What happened.** During a video call, rotating the phone makes the camera image briefly rotate to landscape, then snap back to portrait — because the app is hard-locked to portrait at config level (`frontend-kalba/app.config.js:65`) while nothing in the call screen handles orientation (Confirmed).
2. **Where the case stands.** Stronghold confirmed; root-cause mechanism is Deduced (the transient flip is the native camera/OS sensor briefly following the device before the portrait lock re-asserts). No orientation library or handling exists anywhere in the app.
3. **What's needed next.** A product decision — unlock orientation app-wide vs. only on the call screen — then implement responsive layout for both orientations via `bmad-quick-dev`.

## Case Info

| Field            | Value                                                                      |
| ---------------- | -------------------------------------------------------------------------- |
| Ticket           | BL-004                                                                     |
| Date opened      | 2026-06-27                                                                 |
| Status           | Concluded — root cause Deduced (Medium-High); scope decided: call screen only |
| System           | React Native + Expo SDK 54, `@daily-co/react-native-daily-js`; native (iOS/Android), dev client build (not Expo Go) |
| Evidence sources | Source code (`frontend-kalba/`), `app.config.js`, `package.json`, backlog item BL-004 |

## Problem Statement

User report (Kasia, via Discord): "During a call, when I rotate the phone from portrait to landscape, the camera image rotates for a second to match the landscape position, then returns to portrait view." Desired behavior (Bartek): the view should follow the device — landscape when horizontal, portrait when vertical, as in typical apps.

## Evidence Inventory

| Source                          | Status      | Notes                                                                 |
| ------------------------------- | ----------- | -------------------------------------------------------------------- |
| `frontend-kalba/app.config.js`  | Available   | `orientation: "portrait"` at expo root (line 65) — app-wide lock      |
| `frontend-kalba/app/(app)/workshop/call.tsx` | Available | Video rendered via `DailyMediaView`; no orientation logic           |
| `frontend-kalba/package.json`   | Available   | No `expo-screen-orientation` / orientation lib installed              |
| Device-level repro / logs       | Missing     | No recording of the transient flip; needs dev build + physical device |
| Daily SDK orientation behavior  | Partial     | Behavior of native video track on rotation not yet directly observed  |

## Investigation Backlog

| # | Path to Explore | Priority | Status | Notes |
| - | --------------- | -------- | ------ | ----- |
| 1 | Confirm the transient-flip mechanism on a physical device (orientation locked vs unlocked) | Medium | Open | Needs dev client build |
| 2 | Decide scope: unlock orientation app-wide vs. only call screen | High | Done | RESOLVED 2026-06-27 (Bartek): call screen only — keep app portrait-locked, unlock landscape at runtime during a call |
| 3 | Check iOS `infoPlist` supported orientations + `supportsTablet` interaction | Low | Open | Expo derives these from `orientation`, but tablet may differ |

## Confirmed Findings

### Finding 1: App is hard-locked to portrait at config level

**Evidence:** `frontend-kalba/app.config.js:65` → `orientation: "portrait"`

**Detail:** This Expo setting locks the entire app's interface orientation to portrait on both iOS and Android. No per-screen override exists.

### Finding 2: No orientation handling exists anywhere in the codebase

**Evidence:** Repo-wide grep for `orientation|ScreenOrientation|lockAsync|landscape|portrait|rotate` returns only (a) `app.config.js:65` (the lock), (b) `call.tsx:475` an icon `transform: rotate "135deg"` (cosmetic hang-up icon, unrelated), and (c) a test name in `client.test.ts`. `package.json` has no `expo-screen-orientation` or equivalent.

**Detail:** There is no listener for device rotation and no responsive landscape layout. The app has never been built to render in landscape.

### Finding 3: Self-view/remote video is rendered with no rotation awareness

**Evidence:** `frontend-kalba/app/(app)/workshop/call.tsx:419-423` → `<DailyMediaView videoTrack={vTrack} mirror={mirror} objectFit="cover" />`

**Detail:** The video tile uses `objectFit="cover"` and a fixed (portrait) layout. Nothing adjusts tile geometry on rotation.

## Deduced Conclusions

### Deduction 1: The "1-second flip then snap back" is the OS/camera sensor briefly leading the portrait lock

**Based on:** Findings 1 + 2.

**Reasoning:** When the device is physically rotated, the native camera capture / OS momentarily reflects the new physical orientation in the video frame. Because the app UI is hard-locked to portrait and nothing handles the rotation event, the rendered view re-settles to portrait. The user perceives this as the image rotating for ~1s and then returning.

**Conclusion:** The symptom is a side effect of the portrait lock, not a Daily SDK bug. The fix is to *support* orientation (the desired behavior), not to suppress the flip.

## Hypothesized Paths

### Hypothesis 1: Transient flip originates in native camera capture orientation racing the UI lock

**Status:** Open

**Theory:** The native camera capture orientation follows the accelerometer for a moment before the portrait-locked React Native view re-composes.

**Supporting indicators:** App is portrait-locked (F1); no JS-side rotation handling (F2); symptom is brief and self-reverting.

**Would confirm:** On a dev build, temporarily set `orientation: "default"` and observe whether the view rotates and stays (no snap-back).

**Would refute:** The flip persists even with orientation unlocked, or originates from a Daily-specific setting.

## Missing Evidence

| Gap | Impact | How to Obtain |
| --- | ------ | ------------- |
| Direct on-device observation of the flip | Confirms H1 vs. a Daily SDK quirk | Dev client build on a physical phone, rotate during a call |
| Intended product scope (whole app vs. call only) | Determines implementation surface | Decision from Bartek/UX |

## Source Code Trace

| Element       | Detail                                                                 |
| ------------- | --------------------------------------------------------------------- |
| Error origin  | `frontend-kalba/app.config.js:65` (`orientation: "portrait"`) — the constraint that produces the snap-back |
| Trigger       | Physical device rotation during an active call                          |
| Condition     | App UI locked to portrait + no rotation handling + fixed call layout    |
| Related files | `frontend-kalba/app/(app)/workshop/call.tsx` (video render/layout), `frontend-kalba/app.config.js` (orientation lock), `frontend-kalba/package.json` (missing orientation lib) |

## Conclusion

**Confidence:** Medium-High

**Confirmed:** The app is portrait-locked at config level and contains no orientation handling; the call screen renders video in a fixed portrait layout. This fully explains why landscape "doesn't stick." **Deduced:** the momentary flip is the OS/camera sensor briefly leading before the lock re-asserts. **Hypothesized (Open):** the precise origin of the transient frame, pending on-device confirmation. The root cause is well enough understood to act: this is a "feature was never built" situation, not a defect in the Daily SDK.

## Recommended Next Steps

### Fix direction — DECIDED: call screen only

Scope resolved 2026-06-27 (Bartek): keep the app portrait-locked globally; unlock landscape only on the call screen. Three changes combine:
1. **Keep** `app.config.js` `orientation: "portrait"` (global lock unchanged).
2. **Add `expo-screen-orientation`** (install + config plugin + dev-client rebuild). In `call.tsx`, `ScreenOrientation.unlockAsync()` on mount and `lockAsync(OrientationLock.PORTRAIT_UP)` on unmount so rotation is scoped to the call and never leaks to other screens (including on early exit / hardware back).
3. **Make the call layout responsive** in `call.tsx` — adjust tile geometry and self-view placement for landscape so it isn't a stretched portrait layout (ties into BL-003 self-view positioning).

Implementation watch-outs for the dev agent: ensure the unlock is reverted on *every* exit path (unmount, navigation away, error, call-end); verify `DailyMediaView`/`objectFit="cover"` behaves on rotation; rebuild the dev client since a native module is added; this is native-only (no `call.web.tsx` change needed).

### Diagnostic

Confirm H1 cheaply: on a dev build, set `orientation: "default"` and rotate during a call. If the view rotates and stays, the lock was the sole cause and no Daily-specific work is needed.

## Reproduction Plan

1. Build the dev client (not Expo Go — native WebRTC).
2. Join a workshop call on a physical phone.
3. Rotate the device portrait → landscape.
4. **Observed:** camera image rotates ~1s, then snaps back to portrait.
5. **Expected (target):** view settles into landscape and stays; rotating back returns to portrait.

## Side Findings

- `call.tsx:475` uses `transform: rotate("135deg")` purely to orient the hang-up icon — unrelated to the bug, noted to avoid a false lead.
- No `expo-screen-orientation` dependency means option (1b) requires a new native module + dev client rebuild, not just a JS change.
