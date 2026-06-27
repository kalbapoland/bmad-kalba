# Frontend Architecture — `frontend-kalba`

> Part: **mobile** (Expo / React Native) · component + file-based-routing · deep scan · 2026-06-27

## Executive summary

A React Native (Expo SDK 54, new architecture) app using **Expo Router** for
file-based navigation. Server state is owned by **TanStack Query v5**; auth/session
state by a **Zustand** store backed by `expo-secure-store`. All HTTP goes through
a single Axios client with an automatic refresh-token interceptor. Styling is
**NativeWind v4** (Tailwind v3). Live video uses the Daily.co React Native SDK
(requires a dev/EAS build — not Expo Go).

## Technology stack

| Category | Technology | Version |
|----------|-----------|---------|
| Runtime | React / React Native | 19.1.0 / 0.81.5 |
| Platform | Expo SDK | ~54 (newArchEnabled) |
| Routing | expo-router | ~6 (file-based) |
| Server state | @tanstack/react-query | ^5 |
| Client/auth state | zustand | (auth store) |
| HTTP | axios | ^1.13 |
| Styling | nativewind + tailwindcss | ^4 / v3 |
| Video | @daily-co/react-native-daily-js + webrtc | ^0.83 / 124 |
| Auth | expo-auth-session, expo-crypto, expo-web-browser | — |
| Storage | expo-secure-store (native), localStorage (web) | — |
| Push | expo-notifications | ~0.32 |
| i18n | i18next + react-i18next + expo-localization | — |
| Fonts | Fraunces + Inter (@expo-google-fonts) | — |
| Tests | jest + jest-expo | — |

`@/*` path alias → `./src/*`. Builds via **EAS** (profiles: development,
tester, production).

## Routing map (Expo Router)

```
app/
├── _layout.tsx                  # Root: fonts, i18n, QueryClientProvider,
│                                #   token restore, push deep-link routing
├── sign-in.tsx                  # Auth entry (Google + email/password)
├── forgot-password.tsx
├── reset-password.tsx
├── oauthredirect.tsx            # OAuth redirect handler
└── (app)/                       # Authed group — redirects to /sign-in if no token
    ├── _layout.tsx              # Auth gate + useUser() + push registration; Stack
    ├── (tabs)/                  # Bottom tab navigator (FloatingTabBar)
    │   ├── index.tsx            # Workshops feed
    │   ├── groups.tsx
    │   ├── my-kalba.tsx         # Goal + stats + notifications
    │   ├── calendar.tsx         # Month/week/day enrolled+trainer view
    │   └── profile.tsx          # Profile, sign out, delete account
    ├── create-workshop.tsx      # modal
    ├── create-group.tsx         # modal
    ├── workshop/[id].tsx        # detail / enroll / join
    ├── workshop/edit.tsx        # modal
    ├── workshop/call.tsx        # Daily video call (native)
    ├── workshop/call.web.tsx    # web variant
    ├── group/[id].tsx
    └── group/edit.tsx           # modal
```

## State management

- **Auth (`src/store/auth.ts`, Zustand).** Holds `token`, `user`,
  `isRestoringToken`, `pushToken`. Persists access + refresh tokens to
  SecureStore (localStorage on web) under **API-URL-scoped keys** (so switching
  backends doesn't mix sessions), migrates legacy keys, and unregisters the push
  token on sign-out. `restoreToken()` runs at app start.
- **Server state (TanStack Query).** `src/lib/query-client.ts`: `staleTime`
  5 min, `retry` 1. Domain hooks in `src/hooks/` (`useWorkshops`, `useGroups`,
  `useEnrollment`, `useJoinWorkshop`, `useMyKalba`, `useUser`,
  `usePushRegistration`, `useTagSuggestions`, create/update/delete mutations…).

## API layer (`src/api/`)

- `client.ts` — Axios instance. **Resolves base URL** per platform/env: dev
  builds rewrite the native host to the Metro host so local-IP changes don't
  break API calls; Android emulator `10.0.2.2` → `127.0.0.1`. 30s timeout.
  Request interceptor injects `Bearer` token; **response interceptor** does the
  401 → `/auth/refresh` → retry-once → else sign-out flow.
- `endpoints.ts` — typed functions for every backend route (auth, users,
  workshops, groups, video join/host-action, push tokens, tags, my-kalba).
- `auth-response.ts` — normalizes auth payloads.
- `src/types/api.ts` — TypeScript mirrors of backend read/write schemas (`User`,
  `Workshop`, `Group`, `JoinWorkshopResponse`, `WorkshopRules`, `MyKalba*`, …).

## Cross-cutting

- **i18n** (`src/lib/i18n.ts`, `src/locales/`) — loaded in root layout;
  `react-i18next` `t()` used in screens/tabs.
- **Notifications** — root layout configures the notification handler (only
  `reminder` types show a banner) and deep-links taps to `/workshop/{id}`;
  `usePushRegistration` registers the Expo token on every authed launch.
- **Hashtags** (`src/lib/hashtags.ts`) — must mirror backend
  `app/services/hashtags.py` parsing rules.
- **Theme** (`src/theme/tokens.ts`) — color tokens shared with NativeWind config.

## Notable native config (`app.config.js`)

`com.kalba.app` (iOS + Android), URL scheme `kalba`, reverse-client-id Google
URL scheme for native Sign-In, camera/mic usage strings, `UIBackgroundModes`
`voip|audio|remote-notification`, `ITSAppUsesNonExemptEncryption=false`, Android
cleartext only for local/private hosts. WebRTC enabled via
`@config-plugins/react-native-webrtc`.

## Components & guides

See [component-inventory-frontend.md](./component-inventory-frontend.md) and the
in-repo docs (`frontend-kalba/docs/DESIGN.md`, `FEATURES-LIST.md`).
