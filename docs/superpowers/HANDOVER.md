# Handover: white-label openGym

The one file that says where the project stands. Update it in the same commit as the work.
Plan: [plans/2026-11-17-white-label-implementation-plan.md](plans/2026-11-17-white-label-implementation-plan.md).
Design: [specs/2026-11-17-white-label-coach-gym-design.md](specs/2026-11-17-white-label-coach-gym-design.md).

## Now

- **Next item:** P1.1 (`api/brand.js` and its test).
- **Branch for it:** `wl/phase-1-brand`, cut from `main` once the planning pull request is merged.
- **In progress:** nothing.
- **Blocked:** nothing in code. P0.2 (media licence) blocks any rebranded demo that shows exercise
  animations.

## Progress

### Phase 0: groundwork
- [x] P0.1 CLA workflows run only upstream (effective once merged to `main`)
- [ ] P0.2 Gym visual media licence
- [ ] P0.3 Talk to the upstream maintainer
- [ ] P0.4 Product name and domain pattern
- [ ] P0.5 Pilot gym's logo, colour, trainer and member counts
- [ ] P0.6 Lawyer reads the `gym-coached` consent text

### Phase 1: brand layer
- [ ] P1.1 `api/brand.js`
- [ ] P1.2 Brand routes and openapi
- [ ] P1.3 nginx, dev proxy, service worker
- [ ] P1.4 `frontend/src/lib/brand.js`
- [ ] P1.5 Apply the brand in the UI
- [ ] P1.6 Name substitution in `t()`
- [ ] P1.7 Server-side names
- [ ] P1.8 About row with source link
- [ ] P1.9 Provisioning script and docs

### Phase 2: gym policy
- [ ] P2.1 Policy helper and its copy
- [ ] P2.2 Storage, route, config
- [ ] P2.3 Module gate
- [ ] P2.4 Server-side module readers
- [ ] P2.5 Member navigation
- [ ] P2.6 Settings
- [ ] P2.7 Lead's editor

### Phase 3: coach-led joining and today
- [ ] P3.1 `live` category
- [ ] P3.2 Consent text
- [ ] P3.3 `TRAINERS_JOIN=coached`
- [ ] P3.4 Allocation
- [ ] P3.5 Abuse tests
- [ ] P3.6 Op `today.set`
- [ ] P3.7 Route `/api/trainer/today`
- [ ] P3.8 Notice and push
- [ ] P3.9 Member's Today card
- [ ] P3.10 Console layout and actions

### Phase 4: live floor
- [ ] P4.1 Payload
- [ ] P4.2 `api/presence.js`
- [ ] P4.3 Heartbeat on set tick
- [ ] P4.4 `/api/trainer/live`
- [ ] P4.5 Floor board
- [ ] P4.6 Member's switch
- [ ] P4.7 Docs

### Phase 5: native app and coach events
- [ ] P5.1 Join link
- [ ] P5.2 App identity
- [ ] P5.3 Coach events
- [ ] P5.4 Applying events on the phone
- [ ] P5.5 Store listings

## Decisions

| Date | Decision | Why |
|---|---|---|
| 2026-11-17 | One instance per gym | Fits the code as it is, isolates each gym's data, passkeys bind to one hostname |
| 2026-11-17 | Branded PWA first, then one native app that a gym QR reskins | No store review per gym; Apple rejects templated apps |
| 2026-11-17 | Licence route open; build so both AGPL and commercial work | Everything generic sits behind flags and can go upstream |
| 2026-11-17 | Coach-led: one consent at joining, member can switch things off | Premium feel without dropping the privacy model |
| 2026-11-17 | CLA workflows guarded by repository name instead of deleted | Smallest difference from upstream, so merges stay clean |
| 2026-11-17 | Brand assets are PNG only | An SVG can carry script |
| 2026-11-17 | Brand name replaces "openGym" in one place inside `t()` | Avoids editing about 124 files and 26 locale packs |

## Open questions

- Product name and domains (P0.4).
- Will the pilot gym's trainers use tablets on the floor, or phones? It decides how much of P3.10
  matters before the pilot.
- Does the gym already run membership software the members sign in to?

## Corrections to the plan

Nothing yet. Write here whenever the code turned out different from what the plan said.
