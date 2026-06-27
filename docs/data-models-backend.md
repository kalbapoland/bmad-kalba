# Data Models — `backend-kalba`

> SQLModel tables on PostgreSQL · Alembic migrations in `migrations/versions/` · deep scan · 2026-06-27

All `id` columns are `UUID` PKs (`uuid4`) unless noted. Timestamps are stored
**UTC-naive**. There are **no DB-level `ON DELETE CASCADE`** on user FKs — the
account-deletion service deletes referencing rows explicitly (workshop_tag and
workshop↔tag links do use `ondelete="CASCADE"`).

## Identity & auth

### `user`
`id`, `email` (unique, indexed), `full_name`, `is_active`, `google_id`
(unique, nullable), `hashed_password` (nullable — Google-only users have none),
`role` (`user|trainer`). Relationship → `trainer_profile`.

### `trainer_profile`
`id`, `user_id` (FK user, unique), `bio`, `specialties: list[str]` (JSONB).

### `refresh_token`
`id`, `user_id` (FK), `token_hash` (sha256, unique indexed), `expires_at`,
`issued_at`, `revoked_at?`. Rotated on `/auth/refresh`; revoked en masse on
password reset.

### `password_reset_token`
`id`, `user_id` (FK), `token_hash` (unique indexed), `expires_at`, `created_at`,
`used_at?`. Single-use; superseded tokens marked used.

## Groups

### `group`
`id`, `trainer_id` (FK user), `title`, `description`, `created_at`, `updated_at`,
`deleted_at?` (soft delete). `"group"` is auto-quoted (reserved keyword).

### `group_membership`
`id`, `group_id` (FK), `user_id` (FK), `role` (`member|trainer_subscriber`),
`joined_at`. **Unique** `(user_id, group_id)`.

## Workshops

### `workshop`
`id`, `trainer_id` (FK user), `group_id` (FK group), `title`, `description`,
`start_time`, `duration_minutes` (≥1), `timezone` (IANA, default UTC),
`price` (Decimal 10,2), `max_participants` (≥1), `video_room_id?`
(`kalba-{id}` once created), `deleted_at?`, `reminder_minutes_before` (default
60), `reminder_sent_at?`. Relationships → trainer, rules, participants, tags.

### `workshop_enrollment`
`id`, `user_id` (FK), `workshop_id` (FK), `enrolled_at`. **Unique**
`(user_id, workshop_id)` — backs idempotent enroll.

### `tag` / `workshop_tag`
`tag`: `id`, `name` (String(30), unique, lowercase+NFC), `created_at`.
`workshop_tag`: composite PK `(workshop_id, tag_id)`, both FK `ON DELETE CASCADE`.

## Video

### `workshop_rules`
`id`, `workshop_id` (FK, unique), `force_camera_on` (T), `force_mic_muted_on_join`
(T), `allow_unmute_after` (sec, 0=immediate), `allow_camera_toggle` (T),
`late_join_behavior` (`allow|allow_muted|deny`, default allow_muted), `all_muted`
(F), `all_cameras_off` (F). Last two are live host-controlled state.

### `workshop_participant`
`id`, `user_id` (FK), `workshop_id` (FK), `role` (`host|participant`),
`joined_at?`.

### `video_usage_session` — Daily.co cost accounting
One row per issued join token. `id`, `workshop_id` (FK), `user_id` (FK),
`reserved_minutes` (worst-case now→room expiry), `actual_minutes?`, `settled`
(bool, indexed), `created_at` (indexed), `settled_at?`. Reservation counts
against the monthly cap immediately; settled to real minutes on `participant.left`.

## My Kalba & notifications

### `user_goal`
`id`, `user_id` (FK, unique), `monthly_target` (1–60, default 4), `created_at`,
`updated_at`. Created on demand via `INSERT … ON CONFLICT DO NOTHING`.

### `user_notification` (in-app)
`id`, `user_id` (FK), `type` (`workshop_rescheduled|workshop_reminder`, PG enum),
`title`, `body`, `payload: dict` (JSON), `is_read` (indexed), `read_at?`,
`created_at` (indexed), `deleted_at?` (soft delete).

### `push_token`
`id`, `user_id` (FK), `token` (`ExponentPushToken[...]`, unique indexed),
`platform` (`ios|android`, PG enum), `created_at`, `last_seen_at`. **Keyed by
device, not user** — re-registration reassigns `user_id`.

## Migration history (highlights)

`migrations/versions/` (21 revisions): initial → video models → workshop
enrollment → password reset token → groups & membership → require group on
workshop → workshop reminder fields → tags + popularity index → my-kalba goal &
in-app notifications → push token table → native auth fields on user → timezone
on workshop → video usage session → security & architecture hardening (+ merge
revisions). Run with `alembic upgrade head` (auto on container boot).
