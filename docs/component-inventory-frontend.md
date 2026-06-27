# Component Inventory — `frontend-kalba`

> Reusable UI in `src/components/`; screens live in `app/` (see architecture-frontend.md) · deep scan · 2026-06-27

## Primitives / display

| Component | Purpose |
|-----------|---------|
| `AppText` | Typography wrapper (font tokens). |
| `Button` | Primary/secondary button. |
| `PressableScale` | Press feedback (scale + haptics). |
| `Badge` | Small status/label pill. |
| `Skeleton` | Loading placeholder. |
| `EmptyState` | Empty-list illustration + copy. |
| `SectionHeader` | Section title row. |
| `DateBlock` | Compact date display. |
| `BreathingCircle` | Animated meditation visual. |

## Composite / domain

| Component | Purpose |
|-----------|---------|
| `AuthScreen` | Shared auth UI (largest component, ~627 LOC) — Google + email/password. |
| `GroupCard` | Group summary card (member count, owner/member state). |
| `FloatingTabBar` | Custom bottom tab bar used by `(tabs)/_layout`. |
| `DescriptionWithHashtags` | Renders descriptions with highlighted `#tags`. |
| `HashtagTextInput` | Description editor with live tag suggestion. |

## Calendar (`src/components/calendar/`)

| Component | Purpose |
|-----------|---------|
| `MonthView`, `WeekView`, `DayView` | Calendar renderings for the calendar tab. |
| `dateUtils.ts` | Calendar date helpers. |

## Hooks (`src/hooks/`)

Server-state and side-effect hooks: `useUser`, `useWorkshops`,
`useWorkshopDetail`, `useMyWorkshops`, `useCreateWorkshop`, `useUpdateWorkshop`,
`useDeleteWorkshop`, `useEnrollment`, `useJoinWorkshop`, `useHostAction`,
`useGroups`, `useMyKalba`, `useTagSuggestions`, `usePushRegistration`.

## Library helpers (`src/lib/`)

`date.ts`, `workshopSchedule.ts`, `hashtags.ts` (mirrors backend), `i18n.ts`,
`oauth-url.ts`, `query-client.ts`, `haptics.ts`, `entrance.ts`, `user.ts`.

## Design system

Color/spacing tokens in `src/theme/tokens.ts`, consumed by NativeWind
(`tailwind.config`). Fonts: Fraunces (display) + Inter (body). The full visual
design rationale lives in `frontend-kalba/docs/DESIGN.md`.
