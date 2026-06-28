---
baseline_commit:
  backend-kalba: '58f683e33e932a9dd854d4eee213937835427440'
  frontend-kalba: '50edeaa617256ddfc56744333b79b3925bc95225'
branch: feat/host-kick-participant
---

# Story BL-002: Host can remove a participant from a live video call

Status: review

<!-- Source feature: feature-backlog.md → BL-002. No formal epics.md/sprint-status.yaml in this repo;
     story foundation = BL-002 + docs/ architecture set + direct code analysis. -->

## Story

As a **trainer hosting a live workshop call**,
I want to **remove (kick) any participant from the call while it is in progress**,
so that **I can deal with disruptive, mistaken, or unwanted attendees and keep the session under control**.

## Scope

**In scope:** Eject a single participant from the *live* Daily call, host-only, with confirmation, and a clear "you were removed" experience for the ejected user.

**Out of scope (follow-up — BL-002 part 2):** Persistent removal "from the group" (unsubscribe + block future workshops). That is a separate groups/subscription change; this story only ends the participant's *current* call session. Capture it as a new backlog/story item — do not implement here.

## Acceptance Criteria

1. **Given** I am the host (trainer) in a joined call with at least one other participant, **when** I open a participant's tile controls, **then** I see a "Remove" control that non-hosts never see.
2. **Given** I tap "Remove" on a participant, **when** the confirmation dialog appears and I confirm, **then** that participant is ejected from the Daily room within ~2s and disappears from every remaining participant's grid.
3. **Given** I am a participant and the host removes me, **when** the ejection lands, **then** my call screen leaves the meeting and I land back on the previous screen with a visible message "The host removed you from the call." (not a generic/error toast).
4. **Given** I am a non-host participant, **when** I inspect available controls, **then** there is no way for me to remove anyone (UI absent **and** server rejects a forged `remove_participant` with 403).
5. **Given** the host taps "Remove", **when** the action runs, **then** the backend receives a `host-action` call with `action="remove_participant"` and `target_user_id` set, validated host-only, recorded in logs (audit), and returns `200 accepted`.
6. **Given** a remove is attempted with no/invalid `target_user_id`, **when** the backend validates, **then** it returns `422` (missing target) without side effects.
7. **Given** the host removes themselves is attempted (`target_user_id == trainer_id`), **when** validated, **then** it is rejected (`400`) — the host cannot kick themselves via this path (they use the normal leave button).
8. **Given** a participant was just removed, **when** they immediately try to rejoin the same workshop, **then** the join is refused (`403`, with a `Retry-After` header) for a configurable cooldown window (default 60s) — checked before any budget reservation is consumed.
9. **Given** the cooldown window has elapsed, **when** the removed participant rejoins, **then** the join succeeds and the kick marker is cleared.

## Tasks / Subtasks

- [x] **Task 1 — Backend: add `REMOVE_PARTICIPANT` host action (AC: 4,5,6,7)**
  - [x] Added `REMOVE_PARTICIPANT = "remove_participant"` to `HostActionType` in `backend-kalba/app/models/video.py`; reused the existing `HostAction.target_user_id`.
  - [x] Added a `REMOVE_PARTICIPANT` branch in `host_action()` (`backend-kalba/app/api/v1/video.py`) right after the host-only check: `422` if `target_user_id` missing, `400` if `target_user_id == trainer_id`, audit log, early `return HostActionResponse(..., broadcast_sent=False)`.
  - [x] Branch returns before the rules load/commit; mute/camera branches untouched.
- [x] **Task 2 — Backend test (AC: 4,5,6,7)**
  - [x] Added `backend-kalba/tests/integration/test_video_host_action.py`: host `200` valid target, non-host `403`, missing target `422`, self-target `400`. NOTE: runs only with a Postgres test DB (skips locally — no Docker/Postgres here); verified via import + ruff + collection, to be executed in CI.
- [x] **Task 3 — Frontend API + types (AC: 5)**
  - [x] Added `"remove_participant"` to `HostActionType` union (`frontend-kalba/src/types/api.ts`).
  - [x] Extended `sendHostAction(workshopId, action, targetUserId?)` to include `target_user_id` when provided (`frontend-kalba/src/api/endpoints.ts`).
  - [x] Extended `useHostAction` to accept `{ action, targetUserId? }` (`HostActionInput`); updated both callers (`call.tsx`, `call.web.tsx`).
- [x] **Task 4 — Frontend: host "Remove" control + eject (AC: 1,2)**
  - [x] Added a host-only per-tile Remove button to `VideoTile` (`onRemove` prop) wired into the 1-remote and grid layouts in `call.tsx` (gated on `isHost`, remote-only).
  - [x] On confirm: `callRef.current?.updateParticipant(p.session_id, { eject: true })` + fire-and-forget `hostAction.mutate({ action: "remove_participant", targetUserId: p.user_id })`.
- [x] **Task 5 — Frontend: ejected-user experience (AC: 3)**
  - [x] Ejection is a Daily fatal `error` with `error.type === "ejected"` (confirmed in SDK types). `handleError` detects it and shows "The host removed you from the call."; added `exitedRef` guard so error+left-meeting navigate back exactly once.
- [x] **Task 6 — Frontend test (AC: 5)**
  - [x] Added `sendHostAction` payload tests to `frontend-kalba/src/api/__tests__/endpoints.test.ts` (target_user_id included for remove; omitted for broadcast actions). `call.tsx` has no RN render harness in this repo, so the contract-level test was chosen per the task's fallback.
- [x] **Task 7 — Re-join cooldown (AC: 8,9)** _(added per user request after initial review)_
  - [x] Added `kicked_at` to `WorkshopParticipant` (`backend-kalba/app/models/video.py`) + Alembic migration `c1f4a9b2e8d7` (nullable column; chains off head `b8e4c2f10937`).
  - [x] Added `workshop_kick_cooldown_seconds: int = 60` to `Settings` (`backend-kalba/app/core/config.py`).
  - [x] `remove_participant` now upserts the target's participant row with `kicked_at = now`.
  - [x] `join_workshop` refuses non-host joins within the cooldown window (`403` + `Retry-After`), checked **before** `reserve_seat` so a blocked join consumes no budget; clears `kicked_at` on an allowed rejoin.
  - [x] Frontend: no change needed — the existing join `onError` surfaces the backend `detail` message (same path as "workshop full"/"ended").
  - [x] Tests: added cooldown integration tests (blocked immediate rejoin, allowed after window, cooldown=0 disables) to `test_video_host_action.py`. Run in CI (no local DB).

## Dev Notes

### Enforcement model — READ THIS FIRST (prevents the #1 wrong approach)
Daily.co has **no server-side REST "eject" endpoint**. Ejection is a **client SDK action performed by a meeting owner / participant-admin**: `callObject.updateParticipant(sessionId, { eject: true })`. Meeting owners are always participant-admins. In this codebase the host joins with an **owner token** (`backend-kalba/app/api/v1/video.py:172` → `create_meeting_token(..., is_owner=is_host)`), so **the host's own app is the thing that ejects** — the backend cannot and should not try to call Daily to eject. The backend's role here is **authorization + audit only**, and a seam for the future "remove from group". Do **not** add a Daily REST eject call to `services/daily.py`.

### Identity mapping (app user ↔ Daily participant)
The meeting token stamps `user_id=str(user_id)` and `user_name` (`backend-kalba/app/services/daily.py:124-147`, called at `video.py:168-172`). So each Daily participant object exposes `user_id` (the app's user UUID) and `session_id`. Eject by **`session_id`**; send **`user_id`** to the backend as `target_user_id`. No extra lookup needed — both are already on the participant object the frontend already reads (`call.tsx:283-284` build `local`/`remotes` from `callRef.current.participants()`).

### Existing host-control architecture to mirror (do NOT reinvent)
- Backend endpoint: `POST /video/workshops/{id}/host-action` (`video.py:337-415`). Host-only guard already at `video.py:356-359`. `HostAction` schema (`models/video.py:100-103`) **already has** `target_user_id: UUID | None` — it was built for exactly this.
- Existing actions mutate `WorkshopRules` and broadcast an app-message via `daily.send_app_message`. **`remove_participant` does neither** — it returns early after the audit log. The mute/camera actions self-apply on each client via the `app-message` handler (`call.tsx:152-181`); the kick does not need that channel because Daily's `eject:true` already propagates `participant-left` to everyone.
- Frontend host actions: `useHostAction` (`src/hooks/useHostAction.ts`) → `sendHostAction` (`src/api/endpoints.ts:211-220`). Host buttons live in `call.tsx` (mute/cameras around `call.tsx:350-385`) and call `doHostAction` (`call.tsx:266-277`).

### Files to touch (current state → change → must-preserve)
- `backend-kalba/app/models/video.py` — `HostActionType` enum (add member); `HostAction` unchanged. **Preserve** existing enum values/order.
- `backend-kalba/app/api/v1/video.py` — `host_action()` branch. **Preserve** the existing host-only check and the mute/camera + broadcast logic untouched.
- `frontend-kalba/src/types/api.ts` — `HostActionType` union (add member).
- `frontend-kalba/src/api/endpoints.ts` — `sendHostAction` signature (add optional `targetUserId`). **Preserve** the existing 2-arg call sites (mute/cameras) — keep `targetUserId` optional.
- `frontend-kalba/src/hooks/useHostAction.ts` — mutation input shape. Update existing callers in `call.tsx` accordingly (`doHostAction`).
- `frontend-kalba/app/(app)/workshop/call.tsx` — host-only Remove control + eject + ejected-user alert. **Preserve** the orientation-lock `useFocusEffect` (BL-004, `call.tsx:228-240`) and existing `left-meeting`/`error` cleanup so a kick still re-locks portrait on exit.

### Verify-at-runtime (don't guess the event shape)
The exact `left-meeting` / `error` payload Daily emits to an *ejected* RN client (reason string / error code) must be confirmed against `@daily-co/react-native-daily-js` at the installed version. Inspect the event object in `handleLeft`/`handleError` and branch on the ejected reason. If the SDK gives no distinguishable reason, fall back to a neutral message ("You have left the call.") rather than mislabeling a network drop as a kick.

### Testing standards
- Backend: pytest, async integration tests under `backend-kalba/tests/integration/` (see `test_video_budget.py`, `conftest.py` for app/client/auth fixtures and the Daily service mocking pattern). Run: `cd backend-kalba && uv run pytest tests/integration/test_video_host_action.py`.
- Frontend: Jest. Run: `cd frontend-kalba && npx jest`. Typecheck: `npx tsc --noEmit`.

### Security / regression guardrails
- Server-side host-only check is the real authorization gate (AC4) — never rely on UI hiding alone. A forged `remove_participant` from a non-trainer must `403`.
- Daily already blocks non-owners from ejecting, so the client eject is also enforced — defense in depth.
- Known limitation to call out (not fix here): a kicked user could tap "Join" again and get a fresh token within the call window, re-entering. True re-join prevention (track ejected `user_id` per workshop and refuse `/join`) is **out of scope** — flag it as a follow-up; see Clarifications.

### References
- [Source: _bmad-output/planning-artifacts/feature-backlog.md#BL-002]
- [Source: docs/integration-architecture.md] (video/Daily.co flow), [Source: docs/architecture-backend.md], [Source: docs/api-contracts-backend.md] (video endpoints)
- [Source: backend-kalba/app/api/v1/video.py:337-415] host-action endpoint
- [Source: backend-kalba/app/models/video.py:24-29,100-103] HostActionType + HostAction.target_user_id
- [Source: backend-kalba/app/services/daily.py:124-162] meeting token (user_id/is_owner)
- [Source: frontend-kalba/app/(app)/workshop/call.tsx:152-181,266-284,350-385] app-message handler, host actions, participant lists
- [Source: frontend-kalba/src/hooks/useHostAction.ts], [Source: frontend-kalba/src/api/endpoints.ts:211-220], [Source: frontend-kalba/src/types/api.ts:108-118]
- [Daily updateParticipant eject: https://docs.daily.co/reference/rn-daily-js/instance-methods/update-participant]

## Clarifications (for the product owner — answer before or during dev)

1. **Re-join after kick:** ✅ RESOLVED — implemented (Task 7). A removed participant is blocked from rejoining for `workshop_kick_cooldown_seconds` (default 60s) via a `/join` guard backed by `WorkshopParticipant.kicked_at`.
2. **"From the group" (BL-002 full intent):** This story ejects from the *call* only. Confirm that persistent group removal is a separate later story (recommended) and not required for this one.
3. **Ejected message wording (PL/EN):** Confirm copy for the removed-user alert — it must be localized via the existing i18n setup (BL-001 context).

## Dev Agent Record

### Agent Model Used

claude-opus-4-8 (BMad dev-story workflow)

### Debug Log References

- Backend: `uv run pytest -q` → 77 passed, 173 skipped (integration tests, incl. the 4 new host-action tests, **skip** with no Postgres/Docker locally). `uv run ruff check` → all checks passed. Import check confirms `HostActionType.REMOVE_PARTICIPANT == "remove_participant"` and `app.api.v1.video` imports cleanly.
- Frontend: `npx tsc --noEmit` → clean. `npx jest` → 8 suites / 76 tests pass (incl. new `sendHostAction` payload tests).
- Ejection signal sourced from `@daily-co/daily-js` types: fatal `error` event, `error.type: "ejected"` (`DailyFatalErrorType`).

### Completion Notes List

- **Enforcement** is the host's own Daily owner client (`updateParticipant(session_id, { eject: true })`); the backend `remove_participant` action authorizes (host-only) + audits only — no Daily REST eject, no rules mutation, no broadcast. Matches the existing host-action architecture and the already-present `HostAction.target_user_id`.
- **Regression fixed:** the `useHostAction` input shape changed from bare `action` to `{ action, targetUserId? }`; updated both `call.tsx` and `call.web.tsx` callers.
- **Web call screen (`call.web.tsx`)** uses an embedded Daily iframe (no call object), so the kick UI is not added there — host can use the embed's own controls; a native-parity follow-up if needed.
- **Copy is hardcoded English** to match the rest of `call.tsx`; localization deferred to BL-001 (Clarification #3 still open).
- **⚠️ Verify before merge:** (1) backend integration tests must run in CI (no local DB); (2) on-device confirm the `error.type === "ejected"` path shows the right message and the host's Remove button ejects the target — the SDK event was confirmed from types, not a live device.
- Clarifications #1 (re-join prevention) and #2 (remove-from-group) remain **out of scope** as agreed.

### File List

**backend-kalba** (branch `feat/host-kick-participant`)
- `app/models/video.py` — added `HostActionType.REMOVE_PARTICIPANT`; added `WorkshopParticipant.kicked_at`
- `app/api/v1/video.py` — `host_action()` REMOVE_PARTICIPANT branch (authorize + record kick + audit); `join_workshop()` re-join cooldown guard + clears `kicked_at` on allowed rejoin
- `app/core/config.py` — added `workshop_kick_cooldown_seconds` (default 60)
- `migrations/versions/c1f4a9b2e8d7_add_kicked_at_to_workshop_participant.py` — new migration
- `tests/integration/test_video_host_action.py` — new (4 host-action + 3 cooldown tests)

**frontend-kalba** (branch `feat/host-kick-participant`)
- `src/types/api.ts` — `remove_participant` in `HostActionType`
- `src/api/endpoints.ts` — `sendHostAction` optional `targetUserId`
- `src/hooks/useHostAction.ts` — `HostActionInput { action, targetUserId? }`
- `app/(app)/workshop/call.tsx` — host Remove control + eject + ejected-user handling + `removeBtn` style
- `app/(app)/workshop/call.web.tsx` — updated `mutate` call to new input shape
- `src/api/__tests__/endpoints.test.ts` — `sendHostAction` payload tests

## Change Log

- 2026-06-28: Implemented BL-002 in-call host kick. Backend `remove_participant` host action (authorize + audit); frontend host-only Remove control with Daily owner eject and ejected-user message. Backend integration tests authored (run in CI). Status → review.
- 2026-06-28: Added re-join cooldown (Task 7, per user request). `WorkshopParticipant.kicked_at` + migration `c1f4a9b2e8d7`, `workshop_kick_cooldown_seconds` (default 60); `/join` refuses a just-kicked participant with `403`+`Retry-After` before any budget reservation. +3 integration tests. Migration head verified single (`c1f4a9b2e8d7`).
