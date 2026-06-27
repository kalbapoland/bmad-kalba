# Backend Architecture — `backend-kalba`

> Part: **backend** · FastAPI service-/API-centric · deep scan · 2026-06-27

## Executive summary

A single async FastAPI application exposing a versioned REST API under
`/api/v1`, backed by PostgreSQL through SQLModel. It is a layered service:
**routers** (HTTP + auth/validation) → **services** (business logic, external
integrations) → **models** (SQLModel tables + Pydantic schemas) → **db**
(async engine/session). A background asyncio task runs the reminder scheduler
inside the app process.

## Technology stack

| Category | Technology | Version | Notes |
|----------|-----------|---------|-------|
| Language | Python | >=3.13 | |
| Web framework | FastAPI | 0.115.x | async, OpenAPI at `/docs`, `/redoc` |
| ORM | SQLModel | 0.0.22 | SQLAlchemy 2.0 + Pydantic |
| DB driver | asyncpg | 0.30 | async Postgres |
| Migrations | Alembic | 1.14 | run on container boot |
| Auth | PyJWT[crypto] + passlib[bcrypt] | 2.10 / 1.7 | HS256 JWT, bcrypt hashing |
| HTTP client | httpx | 0.28 | Google, Daily.co, Expo, Brevo |
| Server | uvicorn[standard] | 0.34 | |
| Packaging | uv | latest | `uv.lock`, Docker layer cache |
| Tests | pytest + pytest-asyncio | 8 / 0.23 | `asyncio_mode=auto` |

## Architecture pattern

**Layered / service-API-centric.** Request lifecycle:

```
HTTP → APIRouter (app/api/v1/*) 
     → FastAPI dependencies (auth, rate-limit, db session)
     → service functions (app/services/*) for non-trivial logic
     → SQLModel session (app/db.py) → PostgreSQL
```

- `app/main.py` — `create_app()` builds the FastAPI app, CORS, includes the v1
  router + a server-rendered web router, a `/health` endpoint, and a `lifespan`
  that starts/stops the reminder scheduler task.
- `app/api/v1/router.py` — aggregates all v1 sub-routers under prefix `/api/v1`.
- `app/api/web.py` — server-rendered HTML pages (password-reset form, Polish
  privacy policy) so email links and ASC references work without a marketing
  site.

## Key cross-cutting concerns

### Configuration (`app/core/config.py`)
Pydantic-`BaseSettings`, environment profiles `local|dev|stage|prod`. Loads
`.env.<env>` (or pure env vars in prod). Holds DB URL, JWT, Google client IDs,
Brevo email, Daily.co keys + **budget guard knobs**, Expo push, reminder
scheduler cadence, CORS. `pg_url` normalizes Fly's `postgres://` → `postgresql://`.

### Security (`app/core/security.py`)
- JWT access tokens (`type=access`, 1 week) and refresh tokens (`type=refresh`,
  30 days, with `jti`); refresh tokens are stored **hashed** (sha256) and
  rotated on use.
- `hash_password` / `verify_password` via bcrypt.
- `verify_google_id_token` calls Google's `tokeninfo` and checks `aud` against
  the configured web/iOS/Android client IDs.
- Dependencies `get_current_user_id` (required) and `get_optional_user_id`
  (anonymous-tolerant for mixed public/auth endpoints).

### Rate limiting (`app/core/rate_limit.py`)
In-memory sliding-window limiter (per-process). Guards Google auth (5/60s/IP),
password reset (3/300s/IP), and push-token writes (10/60s/user).

### Time handling
Convention: timestamps are stored **UTC-naive**; read schemas re-attach UTC
tzinfo so the API emits `Z`-suffixed ISO strings. Workshop `start_time` is
normalized to UTC-naive on write (`normalize_to_utc_naive`, year 2024–2100).

## Services layer (`app/services/`)

| Service | Responsibility |
|---------|----------------|
| `daily.py` | Async wrapper over Daily.co REST: create/delete **private** rooms, mint meeting tokens (10-min exp), send app-messages (host controls), verify webhook HMAC signatures. |
| `video_budget.py` | **Participant-minute cost guard.** Reserves each join's worst-case minutes (now→room expiry) against the calendar-month cap *before* a token is issued; settles to real minutes on `participant.left`. Guarantees Daily free tier is never exceeded. |
| `scheduler.py` | Background asyncio loop; every `notification_poll_seconds` atomically claims due workshops (`reminder_sent_at IS NULL → now()`) and dispatches reminders. Safe across multiple machines. Phase-1 recipient = trainer. |
| `notifications.py` | Expo Push dispatch (batched ≤100) + device-keyed push-token register/unregister; prunes `DeviceNotRegistered` tokens. |
| `my_kalba_notifications.py` | Creates in-app `UserNotification` rows for reschedules and reminders. |
| `hashtags.py` | Parse `#tags` from workshop descriptions (NFC + casefold, ≤5 tags, len 2–30), upsert `Tag`, set `WorkshopTag` links, prefix suggestions by popularity. Must stay in sync with `frontend/src/lib/hashtags.ts`. |
| `account.py` | Irreversible account deletion cascade (no DB-level `ON DELETE CASCADE`; deletes every referencing row in one transaction). App Store 5.1.1(v). |
| `email.py` | Brevo (single-sender) password-reset email; logs link when no API key. |

## Notable design decisions

- **Backend-centric video.** Rooms are `privacy: "private"`, so a backend-issued
  token is mandatory to join — the backend is the single chokepoint, making the
  budget guard unbypassable. Room name = `kalba-{workshop.id}`; created lazily on
  first join.
- **Group-gated visibility.** Workshops are only visible/enrollable to members
  (or owner) of their group; non-members get **404** (not 403) so existence
  isn't leaked.
- **Concurrency safety.** Enrollment capacity uses `SELECT … FOR UPDATE` on the
  workshop row; unique constraints + `ON CONFLICT` make enroll/subscribe/
  push-token writes idempotent under races.
- **Account-existence privacy.** `forgot-password` always returns 202.

## Testing strategy

`tests/unit/` (config, security, db, hashtags, scheduler, notifications, models)
and `tests/integration/` (auth, workshops, groups, enrollments, tags, video
budget, video webhook, push tokens, notifications dispatch, account deletion),
plus `tests/automated/` smoke/seed fixtures. `pytest-asyncio` in auto mode.

## Entry points

- App: `app.main:app` (ASGI) — run via `uvicorn app.main:app`.
- Migrations: `alembic upgrade head` (auto-run in container CMD).
- Background: reminder loop started in `lifespan`.

See [api-contracts-backend.md](./api-contracts-backend.md) and
[data-models-backend.md](./data-models-backend.md) for the full surface.
