# Deployment Guide — Kalba

> deep scan · 2026-06-27

## Backend → Fly.io

- **App:** `backend-kalba`, primary region `ams`, account `kalba.poland@gmail.com`.
- **Container** (`Dockerfile`): `python:3.13-slim`, deps via `uv sync --frozen
  --no-dev`. Startup `CMD` runs **`alembic upgrade head` then uvicorn** on port
  `8080` — **migrations run on every boot**.
- **`fly.toml`:** `APP_ENV=prod`, `force_https`, `auto_stop_machines=stop` /
  `auto_start_machines`, `min_machines_running=0` (scale-to-zero), shared-cpu
  256MB, request concurrency soft 200 / hard 250.
- **CI:** `.github/workflows/ci-backend.yml` (tests) and `fly-deploy.yml`
  (deploy). Secrets configured via `fly secrets set` (DB URL, JWT, Google, Daily,
  Brevo, Expo).
- Password-reset page + privacy policy are served by the backend itself
  (`/reset-password`, `/privacy`) at `https://backend-kalba.fly.dev`.

## Frontend → EAS / App Store

- **EAS profiles (`eas.json`, `appVersionSource: remote`):**
  - `development` — dev client, internal distribution.
  - `tester` — internal, production env, Android APK.
  - `production` — store distribution, `autoIncrement`.
- **Submit:** iOS `ascAppId` `6761315112`.
- **Build commands:** `npm run android:eas:*`, or `npx eas-cli build -p ios
  --profile production`.
- iOS bundle `com.kalba.app`; `ITSAppUsesNonExemptEncryption=false` set to avoid
  TestFlight "Missing Compliance".

## Launch status (see `frontend-kalba/docs/APP_STORE_LAUNCH.md`)

Code blockers are done. Remaining items per project memory: fresh iOS build,
demo account/content, ASC App Privacy questionnaire, and Submit for Review.

## External services to provision

| Service | Used for | Config |
|---------|----------|--------|
| Fly.io Postgres | primary DB | `DATABASE_URL` |
| Daily.co | video rooms/tokens | `DAILY_API_KEY`, `DAILY_DOMAIN`, `DAILY_WEBHOOK_SECRET`; keep budget guard on |
| Google OAuth | sign-in | web/iOS/Android client IDs; reverse-client-id URL scheme |
| Brevo | password-reset email | `BREVO_API_KEY`, verified single sender `kalba.poland@gmail.com` |
| Expo Push | reminders | `EXPO_PUSH_ACCESS_TOKEN` |
