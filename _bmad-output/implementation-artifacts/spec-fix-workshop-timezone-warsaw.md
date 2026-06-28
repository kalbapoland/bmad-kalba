---
title: 'Fix workshop time +1h bug — remove broken device-timezone fast path'
type: 'bugfix'
created: '2026-06-28'
status: 'done'
context: []
baseline_commit: '50edeaa617256ddfc56744333b79b3925bc95225' # frontend-kalba HEAD
---

## Intent

**Problem:** Creating a workshop on iPhone at 17:10 stores/shows it as 18:10 (+1h). `localTimeToUTC` in `workshopSchedule.ts` has a "device-timezone fast path" (added in commit `24002ea`) that converts wall-clock→UTC with the bare `new Date(y,m,d,h,min)` constructor. On the user's Hermes build that constructor applies Poland's *standard* offset (CET, +1h) instead of the active *summer* offset (CEST, +2h) during DST, so the saved UTC instant is 1 hour too early. The matching read fast paths in `toLocalDateInput`/`toLocalTimeInput` have the same defect.

**Approach:** Remove the three device-timezone fast-path branches so all wall-clock⇄UTC conversion flows through the existing `Intl.DateTimeFormat`-based path. Evidence that this is correct on the current build: `date.ts` already formats every displayed time via `Intl` with an explicit timezone and shows the right +2h — only the bare-`Date` write path is wrong. The app stays fully timezone-aware (uses `getDeviceTimezone()`); no library, no hardcoded zone.

## Boundaries & Constraints

**Always:**
- Keep the app timezone-aware: continue passing the device IANA timezone (`getDeviceTimezone()`) into conversion; the workshop's stored `timezone` field is unchanged.
- All wall-clock⇄UTC conversion must go through one path (the `Intl`-based algorithm already present in each function). Read and write must stay symmetric so they round-trip.
- Preserve the round-trip + DST-gap verification already in `localTimeToUTC`, and the format/range validation in all three functions.
- Keep the non-conversion improvements from `24002ea` (picker `minimumDate`, time clamping) untouched.

**Ask First:**
- If, while removing the fast paths, any conversion result fails an existing test for a non-device timezone (UTC / Los Angeles), stop — that signals the shared `Intl` path itself is wrong, not just the fast path.

**Never:**
- Do not add a date/timezone library (no Luxon/moment/date-fns-tz). (Luxon would reintroduce the April Intl bug — it uses `Intl` internally.)
- Do not hardcode `Europe/Warsaw` or remove general timezone support.
- Do not touch the backend, DB, or API contract (backend already stores UTC and returns `Z`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Summer create, Warsaw (CEST) | `2026-06-28`, `17:10`, `Europe/Warsaw` | UTC ISO `2026-06-28T15:10:00.000Z` | N/A |
| Winter create, Warsaw (CET) | `2026-01-15`, `17:10`, `Europe/Warsaw` | UTC ISO `2026-01-15T16:10:00.000Z` | N/A |
| UTC→form prefill (summer) | `2026-06-28T15:10:00.000Z`, `Europe/Warsaw` | date `2026-06-28`, time `17:10` | N/A |
| Non-Warsaw zone still works | `2026-03-15`, `05:00`, `America/Los_Angeles` | `2026-03-15T12:00:00.000Z` | N/A |
| DST spring-forward gap | `2026-03-29 02:30`, `Europe/Warsaw` (non-existent) | Reject "Invalid calendar date/time" | validation error shown |

## Code Map

- `frontend-kalba/src/lib/workshopSchedule.ts` -- holds the three buggy device-timezone fast paths in `toLocalDateInput`, `toLocalTimeInput`, `localTimeToUTC`. Sole production fix site. `getDeviceTimezone` stays exported and in use.
- `frontend-kalba/src/lib/__tests__/workshopSchedule.test.ts` -- two tests assert the now-removed fast-path behavior; rework them into device-timezone correctness checks that no longer reference `new Date()` internals.
- `frontend-kalba/src/lib/date.ts` -- reference only; already uses `Intl` correctly, no change.
- `frontend-kalba/app/(app)/create-workshop.tsx`, `frontend-kalba/app/(app)/workshop/edit.tsx` -- callers; already pass `getDeviceTimezone()`, no change.

## Tasks & Acceptance

**Execution:**
- [x] `frontend-kalba/src/lib/workshopSchedule.ts` -- Deleted the `if (timezone === getDeviceTimezone()) { ... }` fast-path block in each of `toLocalDateInput`, `toLocalTimeInput`, and `localTimeToUTC`, so each function uses only its `Intl`-based path. Validation, round-trip verification, and `getDeviceTimezone` export left intact; updated the now-stale doc comment.
- [x] `frontend-kalba/src/lib/__tests__/workshopSchedule.test.ts` -- Replaced the two fast-path tests with a device-timezone Intl round-trip test, and added Warsaw summer (15:10Z) + winter (16:10Z) regression cases and the spring-forward-gap rejection. 18/18 pass.

**Acceptance Criteria:**
- Given any device, when a workshop is created for `17:10` on a summer date in `Europe/Warsaw`, then the request `start_time` is `…T15:10:00.000Z` (no +1h drift).
- Given the same on a winter date, then `start_time` is `…T16:10:00.000Z`.
- Given an existing workshop opened in the edit screen, then the prefilled date/time match the wall-clock originally entered (exact round-trip).
- Given a non-Warsaw timezone, when converting, then results are unchanged from today (general timezone support intact).
- Given `npm test`, then all `workshopSchedule` suites pass.

## Design Notes

Why removal is the fix, not a regression: each of the three functions already contains a correct `Intl.DateTimeFormat({ timeZone })` path; the fast path was an optimization/workaround that, on this build, is the *only* wrong branch. The `localTimeToUTC` Intl algorithm derives the offset by formatting a probe instant in the target zone and reflecting it (`new Date(2 * approxUTC - localAsUTCMs)`), then verifies the round-trip — which correctly yields +2h in CEST and +1h in CET and rejects spring-forward gaps. Reference check: `2026-06-28 17:10` Warsaw → probe `17:10Z` formats as `19:10` in Warsaw → offset +2h → result `15:10Z`. ✓

## Verification

**Commands:**
- `cd frontend-kalba && npm test -- workshopSchedule` -- expected: all green, including new DST cases.
- `cd frontend-kalba && npx tsc --noEmit` -- expected: no new type errors.

**Manual checks:**
- On a device set to Europe/Warsaw, create a workshop at the current local time; confirm detail screen and calendar show the same time entered (no +1h).

## Suggested Review Order

**The fix — write path (the actual bug)**

- Start here: doc now explains why the bare-`Date` fast path was wrong and removed.
  [`workshopSchedule.ts:120`](../../../frontend-kalba/src/lib/workshopSchedule.ts#L120)

- The single conversion path everything now flows through — `Intl` probe + round-trip verify (also rejects DST gaps / bad calendar dates).
  [`workshopSchedule.ts:146`](../../../frontend-kalba/src/lib/workshopSchedule.ts#L146)

**Read path (edit-form prefill) — kept symmetric**

- Fast path removed; UTC→date extraction now via the same `Intl` mechanism.
  [`workshopSchedule.ts:81`](../../../frontend-kalba/src/lib/workshopSchedule.ts#L81)

- Same for UTC→time extraction (keeps `"24"`→`"00"` midnight normalisation).
  [`workshopSchedule.ts:94`](../../../frontend-kalba/src/lib/workshopSchedule.ts#L94)

**Tests**

- Regression guard: Warsaw 17:10 summer must save 15:10Z (the original 18:10 bug).
  [`workshopSchedule.test.ts:105`](../../../frontend-kalba/src/lib/__tests__/workshopSchedule.test.ts#L105)

- Spring-forward gap rejected; device-tz round-trip replaces the old fast-path tests.
  [`workshopSchedule.test.ts:121`](../../../frontend-kalba/src/lib/__tests__/workshopSchedule.test.ts#L121)
