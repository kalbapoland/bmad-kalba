# Development Guide — Kalba

> How to run each part locally · deep scan · 2026-06-27 · see also `README.md` (umbrella) and each repo's README

## Prerequisites

| Tool | Version |
|------|---------|
| Node.js | 20+ (Expo SDK 54) — local is v20.20.0 |
| Python | 3.13 |
| uv | latest (Python deps) |
| Docker | for local PostgreSQL |
| Xcode 15+ / CocoaPods | iOS builds |
| Java/Android SDK | Android builds |

## Backend (`backend-kalba/`)

```bash
cd backend-kalba
docker compose -f docker-compose.local.yml up -d     # PostgreSQL on :5432
cp .env.local.example .env.local                     # then fill secrets
uv sync                                               # install deps
uv run alembic upgrade head                           # migrate
uv run uvicorn app.main:app --reload --port 8000      # serve
```

- API docs: `http://localhost:8000/docs`. Health: `/health`.
- Key `.env.local` vars: `DATABASE_URL`, `JWT_SECRET_KEY`,
  `GOOGLE_CLIENT_ID` / `GOOGLE_IOS_CLIENT_ID` / `GOOGLE_ANDROID_CLIENT_ID`,
  `DAILY_API_KEY` / `DAILY_DOMAIN` / `DAILY_WEBHOOK_SECRET`,
  `BREVO_API_KEY` / `EMAIL_FROM_ADDRESS`, `EXPO_PUSH_ACCESS_TOKEN`.
  Daily budget knobs: `DAILY_FREE_MINUTES_PER_MONTH`,
  `DAILY_USAGE_SAFETY_RATIO`, `DAILY_BUDGET_ENFORCEMENT_ENABLED`.
- Tests: `uv run pytest` (unit + integration; `asyncio_mode=auto`).

## Frontend (`frontend-kalba/`)

```bash
cd frontend-kalba
npm install
# Run against LOCAL backend:
npm run ios          # or: npm run android / npm run web
# Run against the DEPLOYED Fly backend (copies .env.dev):
npm run ios:dev      # or: npm run start:dev / android:dev
```

- Requires a **dev client build** (not Expo Go) because of the native WebRTC
  (Daily) module. In dev builds the Axios base URL auto-tracks the Metro host.
- Env: `EXPO_PUBLIC_API_URL_NATIVE`, `EXPO_PUBLIC_API_URL_WEB`, `APP_ENV`.
- Tests: `npm test` (jest + jest-expo). Type-check: `npx tsc --noEmit`.

## Conventions

- Backend stores timestamps **UTC-naive**; API emits `Z`-suffixed ISO. Don't
  introduce tz-aware columns without following the existing pattern.
- **Cross-repo contracts:** keep `frontend-kalba/src/types/api.ts` and
  `frontend-kalba/src/lib/hashtags.ts` in sync with the backend (schemas /
  `app/services/hashtags.py`).
- **Always update** `frontend-kalba/docs/MANUAL_TEST_CHECKLIST_RELEASE_REGRESSION.md`
  for any feature change (project rule).
- Never push usage past Daily's free tier — the budget guard must stay enabled.

## Useful in-repo docs

`frontend-kalba/docs/`: `DESIGN.md`, `FEATURES-LIST.md`, `BUILDING_WITH_EAS.md`,
`IOS_PUSH_RUNBOOK.md`, `ANDROID_OAUTH_DEBUG_5_MIN.md`, manual-test checklists.
`backend-kalba/docs/AUDIT_REPORT_BACKEND.md`.
