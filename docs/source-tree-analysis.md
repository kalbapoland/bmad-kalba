# Source Tree Analysis — Kalba

> Annotated tree of both parts · deep scan · 2026-06-27

```
kalba/                              # umbrella PLANNING repo (BMAD) — not code
├── _bmad/                          # BMAD engine (installer-managed)
├── _bmad-output/                   # planning + implementation artifacts
├── docs/                           # ← this generated documentation set
├── backend-kalba/                  # PART: backend (own git repo)
└── frontend-kalba/                 # PART: frontend (own git repo)

backend-kalba/                      # FastAPI service
├── app/
│   ├── main.py                     # ENTRY: create_app(), CORS, lifespan→scheduler, /health
│   ├── db.py                       # async engine + get_db_session dependency
│   ├── api/
│   │   ├── v1/
│   │   │   ├── router.py           # aggregates all v1 routers (/api/v1)
│   │   │   ├── auth.py             # register/login/google/refresh/forgot/reset
│   │   │   ├── users.py            # GET/DELETE /users/me
│   │   │   ├── workshops.py        # CRUD + enroll (largest router, 583 LOC)
│   │   │   ├── groups.py           # CRUD + subscribe/members/workshops
│   │   │   ├── video.py            # join, rules, host-action, budget, webhook
│   │   │   ├── my_kalba.py         # dashboard, goal, schedule, notifications
│   │   │   ├── notifications.py    # push-token register/unregister
│   │   │   └── tags.py             # tag suggest
│   │   └── web.py                  # server-rendered reset-password + privacy HTML
│   ├── core/
│   │   ├── config.py               # Settings (env profiles, all secrets/knobs)
│   │   ├── security.py             # JWT, bcrypt, Google verify, auth deps
│   │   └── rate_limit.py           # in-memory sliding-window limiters
│   ├── models/                     # SQLModel tables + Pydantic schemas
│   │   ├── user.py  auth.py  group.py  workshop.py
│   │   ├── video.py  my_kalba.py  notification.py  tag.py
│   └── services/                   # business logic + external integrations
│       ├── daily.py                # Daily.co REST wrapper
│       ├── video_budget.py         # participant-minute cost guard
│       ├── scheduler.py            # reminder polling loop
│       ├── notifications.py        # Expo push dispatch + token mgmt
│       ├── my_kalba_notifications.py
│       ├── hashtags.py             # tag parsing (mirrors frontend)
│       ├── account.py              # account-deletion cascade
│       └── email.py                # Brevo password-reset email
├── migrations/versions/            # 21 Alembic revisions
├── tests/ {unit,integration,automated}
├── docs/AUDIT_REPORT_BACKEND.md
├── Dockerfile                      # python:3.13-slim + uv; CMD runs alembic then uvicorn
├── fly.toml                        # app=backend-kalba, region ams, port 8080
└── pyproject.toml                  # deps, pytest config

frontend-kalba/                     # Expo / React Native app
├── app/                            # EXPO ROUTER (file-based routes)
│   ├── _layout.tsx                 # ENTRY: providers, fonts, token restore, push deep-links
│   ├── sign-in / forgot-password / reset-password / oauthredirect
│   └── (app)/                      # authed area (redirects if no token)
│       ├── _layout.tsx             # auth gate + push registration
│       ├── (tabs)/                 # index, groups, my-kalba, calendar, profile
│       ├── workshop/[id], edit, call(.web)
│       └── group/[id], edit · create-workshop · create-group
├── src/
│   ├── api/                        # client.ts (Axios+refresh), endpoints.ts, types
│   ├── store/auth.ts               # Zustand auth + SecureStore
│   ├── hooks/                      # TanStack Query domain hooks
│   ├── components/                 # reusable UI (+ calendar/)
│   ├── lib/                        # i18n, hashtags, date, query-client, …
│   ├── locales/  theme/  types/
├── assets/  patches/  scripts/  test/
├── app.config.js                   # Expo config (bundle ids, perms, URL schemes)
├── eas.json                        # EAS build/submit profiles
└── docs/ DESIGN.md, FEATURES-LIST.md, APP_STORE_LAUNCH.md, runbooks, checklists
```

## Entry points

- **Backend:** `app.main:app` (uvicorn); migrations `alembic upgrade head`;
  background reminder loop via `lifespan`.
- **Frontend:** `expo-router/entry` → `app/_layout.tsx`; auth gate in
  `app/(app)/_layout.tsx`.

## Largest files (complexity hot-spots)

Backend: `workshops.py` (583), `video.py` (415), `auth.py` (380),
`groups.py` (351), `my_kalba.py` (326), `notifications.py` (307).
Frontend: `create-workshop.tsx` (657), `workshop/call.tsx` (643),
`workshop/edit.tsx` (637), `AuthScreen.tsx` (627), `my-kalba.tsx` (539),
`group/[id].tsx` (530).
