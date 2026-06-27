# API Contracts — `backend-kalba`

> Base path: **`/api/v1`** · all timestamps ISO-8601 UTC (`Z`) · auth = `Authorization: Bearer <access_token>` · deep scan · 2026-06-27

Auth column legend: **None** = public · **JWT** = valid access token required ·
**Opt** = works anonymous but enriches response when authed · **Trainer** = JWT
whose user has `role=trainer` · **Owner** = must own the resource.

## Auth — `/auth`

| Method | Path | Auth | Body | Returns |
|--------|------|------|------|---------|
| POST | `/auth/register` | None | `{email, password, full_name}` | 201 `AuthResponse` |
| POST | `/auth/login` | None | `{email, password}` | `AuthResponse` (401 invalid) |
| POST | `/auth/google` | None (rate-limited) | `{id_token}` | `AuthResponse` (creates/links user) |
| POST | `/auth/refresh` | None (refresh token in body) | `{refresh_token}` | `AuthResponse` (rotates token) |
| POST | `/auth/forgot-password` | None (rate-limited) | `{email}` | 202 (always; no existence leak) |
| POST | `/auth/reset-password` | None | `{token, password}` | `AuthResponse` (revokes all refresh tokens) |

`AuthResponse = {access_token, refresh_token, token_type="bearer", user_id}`.
Password policy: ≥8 chars, ≥1 letter, ≥1 digit.

## Users — `/users`

| Method | Path | Auth | Returns |
|--------|------|------|---------|
| GET | `/users/me` | JWT | `UserRead {id, email, full_name, is_active, role}` |
| DELETE | `/users/me` | JWT | 204 — irreversible account + data deletion |

## Workshops — `/workshops`

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/workshops/` | Opt | Upcoming/ongoing workshops from caller's groups; empty if anonymous/no groups. `skip`, `limit≤100`. |
| GET | `/workshops/mine` | JWT | Caller's trainer + enrolled workshops (calendar). Params `role=trainer\|enrolled\|all`, `from`, `to`, `limit≤500`. |
| GET | `/workshops/{id}` | Opt | 404 if not a member of its group (existence hidden). |
| POST | `/workshops/` | Trainer | Body `WorkshopCreate`; creates default `WorkshopRules`; parses hashtags; requires owning `group_id`. |
| PATCH | `/workshops/{id}` | Owner | Partial update; reschedule → notifications + reminder re-arm; description → re-derives tags. |
| DELETE | `/workshops/{id}` | Owner | Soft-delete; deletes Daily room if created. |
| POST | `/workshops/{id}/enroll` | JWT | Member-only; capacity-safe (`FOR UPDATE`); idempotent; 409 if full. |
| DELETE | `/workshops/{id}/enroll` | JWT | 204; idempotent unenroll. |

`WorkshopRead` adds caller-context fields `is_owner`, `is_enrolled`,
`enrolled_count`, and `tags: string[]`.

## Groups — `/groups`

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/groups/` | Opt | All groups; `is_owner`/`is_member`/`member_count` per caller. |
| GET | `/groups/mine` | JWT | Owned or subscribed groups. |
| GET | `/groups/{id}` | Opt | Single group. |
| POST | `/groups/` | Trainer | `{title, description}`. |
| PATCH | `/groups/{id}` | Owner | Partial update. |
| DELETE | `/groups/{id}` | Owner | Soft-delete. |
| POST | `/groups/{id}/subscribe` | JWT | Idempotent; can't subscribe to own group. |
| POST | `/groups/{id}/unsubscribe` | JWT | 204; idempotent. |
| GET | `/groups/{id}/members` | JWT | `GroupMemberRead[]`. |
| DELETE | `/groups/{id}/members/{user_id}` | Owner | Remove a member. |
| GET | `/groups/{id}/workshops` | JWT (member/owner) | Group's workshops. |

## Video — `/video`

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| POST | `/video/workshops/{id}/join` | JWT | Time-window + capacity + **budget** gates; lazily creates Daily room; returns `JoinResponse {token, room_url, role, rules}`. Token exp 10 min. |
| GET | `/video/workshops/{id}/rules` | JWT | Current `RulesRead` incl. live `all_muted`/`all_cameras_off`. |
| POST | `/video/workshops/{id}/host-action` | Owner/host | Body `{action: mute_all\|unmute_all\|cameras_off_all\|cameras_on_all}`; persists live state + broadcasts Daily app-message. |
| GET | `/video/budget` | JWT | `VideoBudgetStatus` (used/cap/remaining minutes, period, enforced). |
| POST | `/video/webhooks/daily` | Signature (HMAC) | Daily events; `participant.left` settles usage reservation. Probe `{"test":"test"}` acknowledged. |

Join time window: host any time; participants from 5 min before start to
`duration + 10 min` after. Budget exceeded → **503**.

## My Kalba — `/users/me/my-kalba`

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/dashboard` | JWT | `{goal, stats, unread_notifications}`. |
| GET / PUT | `/goal` | JWT | `monthly_target` 1–60 (default 4). |
| GET | `/schedule` | JWT | Upcoming enrolled workshops (`limit≤100`). |
| GET | `/notifications` | JWT | In-app notifications; `unread_only`, paging. |
| PATCH | `/notifications/{id}/read` | JWT | `{is_read}`. |
| POST | `/notifications/mark-all-read` | JWT | 204. |
| DELETE | `/notifications/{id}` | JWT | Soft-delete. |

## Push tokens — `/users/me/push-tokens`

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| PUT | `` (collection root) | JWT (rate-limited) | `{token: ExponentPushToken[...], platform: ios\|android}`; device-keyed upsert; 204. |
| POST | `/unregister` | JWT (rate-limited) | `{token}` in body (kept out of logs); 204 always. |

## Tags — `/tags`

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/tags/suggest` | JWT | `q` prefix (1–30), `limit≤20`; popularity-sorted canonical names. |

## Misc

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/health` | None | `{status: "ok"}`. |
| GET | `/reset-password` | None | Server-rendered HTML reset form (reads `?token`). |
| GET | `/privacy` | None | Polish privacy policy (linked from app + ASC). |
