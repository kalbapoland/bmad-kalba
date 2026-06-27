# Integration Architecture — Kalba (frontend ↔ backend ↔ external services)

> How the two parts and external services talk · deep scan · 2026-06-27

## Topology

```
┌─────────────────────────┐         HTTPS / REST (/api/v1)         ┌──────────────────────────┐
│  frontend-kalba          │ ───────────────────────────────────▶ │  backend-kalba (FastAPI) │
│  Expo / React Native     │   Axios + Bearer JWT, refresh-on-401  │  on Fly.io (ams)         │
│                          │ ◀─────────────────────────────────── │                          │
└───────┬─────────┬────────┘            JSON responses             └───┬───────┬────────┬─────┘
        │         │                                                    │       │        │
        │         │ Daily.co media (WebRTC, token from backend)        │       │        │
        │         └──────────────────────────────▶ Daily.co ◀─────────┘       │        │
        │                                                                       │        │
        │ Google Sign-In (native) → id_token                                    │        │
        └──────────────────────────────────────────────────────────────────────┘        │
                                                                                          │
   PostgreSQL ◀── SQLModel/asyncpg ── backend ──▶ Expo Push, Brevo email, Google tokeninfo
```

## Integration points

| From | To | Protocol | Details |
|------|----|----------|---------|
| Frontend | Backend | REST `/api/v1` over HTTPS | Axios client; `Bearer` access token; 401 auto-refresh via `/auth/refresh`, then sign-out on failure. |
| Frontend | Google | OAuth (expo-auth-session) | Native Sign-In yields an `id_token` sent to `POST /auth/google`. Native redirect URI must use the reverse-client-id scheme. |
| Frontend | Daily.co | WebRTC (RN Daily SDK) | Frontend never holds Daily API keys — it joins `room_url` with the short-lived `token` returned by `POST /video/workshops/{id}/join`. |
| Backend | Google | HTTPS (httpx) | `tokeninfo` verifies the `id_token` and its `aud`. |
| Backend | Daily.co | REST (httpx) | Create/delete **private** rooms, mint meeting tokens, send app-messages, verify webhook HMAC. |
| Daily.co | Backend | Webhook | `POST /video/webhooks/daily`; `participant.left` settles the usage reservation. |
| Backend | Expo Push | REST (httpx) | Workshop reminders + dispatch; prunes dead tokens. |
| Backend | Brevo | REST (httpx) | Password-reset emails (single-sender). |
| Backend | PostgreSQL | asyncpg | Primary datastore. |

## Auth flow (end to end)

1. **Google:** app gets `id_token` → `POST /auth/google` → backend verifies →
   returns `{access_token, refresh_token, user_id}`.
   **Email:** `POST /auth/register` or `/auth/login` → same `AuthResponse`.
2. Tokens saved to SecureStore under API-URL-scoped keys (Zustand `signIn`).
3. Every request carries `Authorization: Bearer <access>`.
4. On **401**, the Axios interceptor calls `/auth/refresh` (rotating the refresh
   token), retries once, and signs out if refresh fails.
5. **Password reset:** `/auth/forgot-password` → Brevo email → link opens the
   backend-served `/reset-password` page → `/auth/reset-password` (revokes all
   refresh tokens).

## Live video flow

1. Member opens a workshop near start → `POST /video/workshops/{id}/join`.
2. Backend gates **time window → budget reservation → capacity**, lazily creates
   the private Daily room (`kalba-{id}`), mints a 10-min token, returns
   `{token, room_url, role, rules}`.
3. App joins via the Daily SDK using that token; host uses
   `/host-action` (mute/cameras all) which persists state and broadcasts a Daily
   app-message.
4. On leave, Daily webhooks `participant.left` → backend settles real minutes
   against the monthly budget.

## Contract-coupling to keep in sync

- **Types:** `frontend-kalba/src/types/api.ts` mirrors backend read/write schemas.
- **Hashtags:** `frontend-kalba/src/lib/hashtags.ts` ⇄ `backend-kalba/app/services/hashtags.py` (identical parsing rules).
- **Video rules:** `WorkshopRules` shape shared by both parts.
- A change to a backend response shape is a **cross-repo change** — update the
  matching TS type (and any mirrored logic) in the same planning cycle.
