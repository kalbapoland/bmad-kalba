# Kalba — Product Backlog (Inbound)

Raw feature requests and bug reports collected from stakeholders/users, to be
refined into PRDs, epics, and stories via the BMAD workflows
(`bmad-create-prd`, `bmad-create-epics-and-stories`, `bmad-create-story`).

- **Source:** Discord (#feedback), reporter: Kasia
- **Captured:** 2026-06-27 (items posted ~2026-06-20)
- **Original language:** Polish. Items below are translated to English; the
  verbatim Polish is preserved under each item for fidelity.
- **Status legend:** `new` → `refined` (PRD/epic written) → `in-progress` → `done`

---

## BL-001 — Consistent interface language (PL/EN, user-selectable)

- **Type:** Feature
- **Area:** Frontend (i18n)
- **Priority:** TBD
- **Status:** new

**Request:** The user interface should use one consistent language — preferably
Polish, or selectable by the user. Today the interface is mixed
English/Polish.

> **PL (original):** "Interfejs po angielsku i po polsku — Tak, jak w tytule:
> zeby interfejs uzytkownika mial ustalony jeden spojny jezyk, najchetniej polski
> albo do wyboru."

**Notes:** Frontend already uses `react-i18next` + `expo-localization`. Likely a
matter of (a) auditing untranslated strings, (b) ensuring a single source of
truth for locale, and (c) optionally adding a user-facing language switcher.

---

## BL-002 — Host can remove participants from a session and the group

- **Type:** Feature
- **Area:** Video / Backend (host controls)
- **Priority:** TBD
- **Status:** refined (in-call kick) — story `../implementation-artifacts/story-bl-002-host-kick-participant.md` (ready-for-dev). Only the **live-call** removal is scoped there; "remove from the group" (persistent) is split out as a separate follow-up story.

**Request:** The person leading a session (trainer/host) should be able to
remove participants — both from the live class and from the group.

> **PL (original):** "Wykikowanie — Jak w tytule. Dobrze, gdyby osoba prowadzaca
> miala mozliwosc usuniecia uczestnikow z zajec i z grupy."

**Notes:** Builds on the existing host-action endpoint
(`POST /video/workshops/{id}/host-action`). NOTE: the `HostActionType` enum has
no `remove` action yet (only mute/camera); the story adds `remove_participant`
and reuses the already-present `HostAction.target_user_id` field. Ejection is a
client-side Daily owner action (`updateParticipant(sid, {eject:true})`), not a
backend REST call. "From the group" implies a persistent removal beyond the
single call (out of scope for this story).

---

## BL-003 — Movable / positionable self-view thumbnail in video calls

- **Type:** Feature
- **Area:** Video (frontend UI)
- **Priority:** TBD
- **Status:** new

**Request:** During a video call, the user should be able to set their preferred
window layout. For example, in a 2-person call, to place their own self-view
thumbnail at the top of the screen. Currently it sits in the (bottom-)right.

> **PL (original):** "Przesuwanie i ustawianie miniatury — Dobrze, gdyby w trakcie
> rozmowy wideo mozna bylo ustawic sobie preferowany widok okienek. W rozmowie
> dwoch osob na przyklad, zeby moc ustawic swoja miniature na gorze ekranu. Teraz
> jest w prawy..." (truncated in source — likely "w prawym [rogu/dolnym rogu]").

**Notes:** Self-view positioning in the Daily.co call screen
(`app/(app)/workshop/call.tsx` / `call.web.tsx`). Consider draggable thumbnail
and/or preset positions.

---

## BL-004 — Camera image flips momentarily when rotating the device

- **Type:** Bug
- **Area:** Video (camera orientation)
- **Priority:** High
- **Status:** implemented, pending on-device verification (branch `bard/fix-call-orientation`; investigated + spec'd via quick-dev — see `../implementation-artifacts/spec-bl-004-call-orientation.md`)

**Report:** During a call, when rotating the phone from portrait to landscape,
the camera image rotates for a second to match the landscape orientation, then
snaps back to portrait view.

**Request** We have to fix it and when the phone is flipped the view should adjust accoordingly, so when phone 
has horizontal position the view should be landscape and when the phone is in the vertical positon the view should be a portrait 
view as it is usually in all apps. 

> **PL (original):** "Obracanie kamery — W trakcie rozmowy, gdy obracam telefon z
> pionu w poziom, obraz w kamery na sekunde obraca sie zgodnie z pozioma pozycja
> telefonu, a potem wraca do widoku pionowego."

**Notes:** Orientation handling in the native call screen / Daily.co video track.
Decide intended behavior: lock to portrait, or fully support landscape.
