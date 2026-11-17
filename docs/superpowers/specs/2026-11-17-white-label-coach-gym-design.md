# White-label openGym for coach-led and gym-led studios

Status: design, awaiting review. Nothing here is built yet.

## Goal

Sell openGym to premium gyms and well-known trainers as their own app. A trainer has a list of
clients, decides what each one trains today, and watches the session from a dashboard without
asking "how many sets are left". The member gets a small, calm app that carries the gym's name and
shows only what they need to train.

## Decisions taken

| Question | Decision |
|---|---|
| Hosting | One instance per gym: own containers, domain, data folder and branding. |
| App delivery | A branded PWA per gym first. Then one native app under our own name, where a gym code switches it to that gym's server and branding. |
| Licence | Undecided. The work is laid out so it holds under AGPL and under a commercial licence. |
| Control | Coach-led. One consent screen at joining grants the trainer live view and direct changes. The member can still switch things off later. |

Out of scope for now: gym notices, closures and events (parked on purpose), billing and
memberships, multi-location chains, a shared multi-tenant server.

## What exists today

Most of the people model is already in the code, behind `TRAINERS=on` (docs/TRAINERS.md).

| Need | State | Where |
|---|---|---|
| Gyms, gym leads, trainers, members | Built | `api/trainers/org-store.js`, `api/permissions.js` |
| Member joins by code or QR, with a consent record | Built | `api/trainers/routes.js`, `api/consent.js` |
| Trainer dashboard with a client list and flags | Built, phone-sized | `frontend/src/views/Trainer.jsx` |
| Trainer assigns a routine to up to 50 clients | Built, lands in plans or on a weekday | `POST /api/trainer/assign` |
| Trainer edits plans and workouts, member can undo | Built | `api/trainers/ops.js` |
| Stripping the app down | Built as a personal choice | `frontend/src/lib/modules.js`, "Just training" preset |
| Live "training now" with exercise and set counts | Half built: the phone sends it, only an admin can read it | `POST /api/activity`, `presence` in `api/server.js` |
| Set-by-set detail of a running workout | Missing. The running workout never leaves the device (`delete state.active`) | `api/server.js` commitState |
| "Train this today" | Missing. Assign targets a weekday, not today | |
| Gym lead allocates clients to trainers | Missing. A member connects to a trainer by that trainer's code | |
| Gym decides what the member's app shows | Missing | |
| Branding: name, logo, colours, icon, manifest | Missing. All hard-coded | `BrandLogo.jsx`, `public/manifest.json` |

So this is not a new product. It is five additions to an existing one.

## Product shape

One codebase, one server per gym, three experiences chosen by role:

1. **Member app.** Home shows "Today, from Marco: Pull" and a Start button. Then history and
   progress. No AI Coach, integrations, community, themes, tools or plan editor unless the gym
   turns them on.
2. **Coach console.** The trainer dashboard grown to tablet and desktop width. A roster, a
   "Today" picker per client, and a live floor board.
3. **Gym owner.** The existing gym lead view, plus branding and the member-app policy.

The coach track and the gym track are the same build with different settings. A personal trainer
is a gym with one trainer. A big gym without personal training is a gym whose members have a
default programme and no assigned trainer. Keeping them one product avoids two forks.

## The five additions

### 1. Brand layer

- `DATA_DIR/brand.json`: name, short name, accent colour, logo (SVG or PNG), icon set, support
  contact, legal links. Edited by the gym owner in the admin screen, or dropped in at provisioning.
- `GET /api/brand`, public, since the sign-in screen needs it. The api also serves
  `/manifest.json` and the icons from it, so the installed PWA carries the gym's name and icon.
  nginx proxies those paths.
- `BrandLogo.jsx`, the default accent, the page title, push notification titles and the share
  image read the brand instead of the constants.
- An "About" row keeps "Built on openGym" with a link to the source. Under AGPL that link is
  required, so it is not optional decoration.

### 2. Gym policy, the lean member app

- `gyms[].policy = { modules: { id: boolean }, locked: boolean, nav: 'member' | 'full' }` in
  `org.json`, sent to members in `/api/config`.
- `modules.js` already has a hook for "this build cannot have these modules"
  (`setUnavailableModules`). The policy feeds the same gate, so every surface that checks `isOn`
  obeys it with no new checks in views.
- A new preset, **Member**: logging, history, progress, body weight. Everything else off.
- `nav: 'member'` swaps the tab bar for Today, History, Progress, Profile, hides the plan editor
  when plans come from a trainer, and cuts Settings to units, language, notifications, sharing and
  account.
- Trainers and leads on the same server keep the full app. The policy applies per membership.

### 3. Coach-led joining

- A new consent text, `gym-coached`, shown once when the member uses the gym's join code. It
  grants the gym's assigned trainers: workouts and plans at Change directly, body weight at See,
  and the new `live` category at See.
- A new scope category `live` in `api/trainers/scopes.js` (and its hand copy in
  `frontend/src/lib/sharing-scopes.js`), highest level See.
- `POST /api/gym/allocate`: the gym lead puts a member on a trainer's list. The grant is created
  under the consent the member gave at joining, which named "the trainers this gym assigns to
  you". The member gets a notice and a push each time, and can switch any category off.
- This is the one place the privacy model changes: today no grant exists without the member
  choosing that exact trainer. It needs its own abuse tests in `permissions.test.js`, and the
  member-led flow stays available with `TRAINERS_JOIN=member`.

### 4. "Train this today"

- A new whitelisted op `today.set` (category `plans`) in `api/trainers/ops.js`: copy one of the
  trainer's routines to the member if needed, and make it today's plan. It reuses the copy and
  origin logic of `POST /api/trainer/assign`, keeps its inverse for Undo, and syncs like any other
  trainer change.
- Console: tap a client, pick Push, Pull or Legs from the trainer's own library, send. Multi-select
  for a class.
- Member: a push ("Marco set today's workout: Pull") and the Today card on Home.

### 5. Live floor

- The phone already heartbeats `name, exIdx, exTotal, setsDone, setsTotal, startedAt` to
  `POST /api/activity`. Extend it with the current exercise, the sets of that exercise (weight,
  reps, done) and when the rest timer ends. Still in memory only, never written to disk.
- `GET /api/trainer/live`: presence for the members who share `live` with this trainer, checked
  per member on every request through `can()`. Budgeted like `trainer.list`. Each day a trainer
  watches is one line in the member's sharing log.
- Console floor board: one card per client in the gym. Routine, exercise 3 of 6, set 2 of 4, last
  set 80 kg × 8, resting 0:45, time elapsed. Idle and finished clients drop to a second row.
- Transport: polling every 5 seconds first. Server-sent events later if polling feels slow; plain
  `node:http` can do it, nginx needs buffering off for that path.
- Later, coach adjusts the running workout (change the next weight, add a set, swap an exercise).
  The running workout lives on the device, so the change has to be sent to it. The watch protocol
  (`lib/watch-sync.js`, `api/watch-relay.js`) already remote-controls a running workout through a
  server mailbox, and the same events fit.

## Delivery

**Per-gym instance.** A provisioning script plus the existing Helm chart: one release per gym with
`RP_ID`, `ORIGIN`, `TRAINERS=on`, `INVITE_ONLY=on`, the brand file and the Member policy. Backups
of each `data/` folder, and one image tag rolled across all gyms. Passkeys bind to the gym's
hostname, so the domain is fixed at the start and cannot be changed casually later.

**Branded PWA.** Falls out of the brand layer. The member scans the gym's QR, signs up with a
passkey, adds it to the home screen.

**One native app.** The phone apps already connect to a server by address
(`MobileOnboarding.jsx`). The gym's QR carries the address, the app pairs and loads `/api/brand`,
and reskins itself. A short typed code needs a small directory service that maps code to address.
Limits to tell gyms up front: the store name and home-screen icon stay ours, and Apple's rule
against templated apps is the reason we do not ship one store app per gym.

## Licence and media

Not legal advice. Have a lawyer confirm before the first paying gym.

- **Exercise media.** NOTICE.md says the animations are licensed for use within the openGym
  application only, and third parties have no right to use them in their own products. A
  rebranded product needs its own licence from Gym visual, on either licence route. This is the
  first thing to settle, because an app with no exercise images is not premium.
- **AGPL route.** Run a public fork, link the source from inside the app, keep the copyright
  notices. We sell hosting, setup, branding and support. Check whether the app store exception in
  NOTICE.md carries over to a fork before publishing the native app.
- **Commercial route.** The CLA lets the maintainer license the code commercially. Open that
  conversation early. It keeps our changes private and settles the store question.
- **For both.** Everything above is generic and sits behind flags, so most of it can be offered
  upstream. That shrinks the fork we have to keep merging, and it is the best opening for the
  commercial conversation. Only the brand assets and provisioning are ours alone.
- **Name.** Do not ship under "openGym" or its logo. The brand layer removes both anyway.

## Phases

Sizes are rough guesses for one developer, not commitments.

| Phase | What ships | Size |
|---|---|---|
| 0 | Gym visual licence, talk to the maintainer, one pilot gym signed up | runs alongside |
| 1 | Brand layer, per-gym provisioning. A branded PWA exists | 1 to 2 weeks |
| 2 | Gym policy, Member preset, member nav | 1 to 2 weeks |
| 3 | Coach-led consent, allocation by the lead, "train this today", wide console layout | 2 to 3 weeks |
| 4 | Live scope, richer presence, floor board | 2 weeks |
| 5 | One native app with gym QR, coach adjusts a running workout | 3 to 4 weeks |

A pilot gym can start after phase 3. Phase 4 is the feature that sells it.

## Risks

- **Fork drift.** Upstream moves fast. Keep every change behind a flag, touch shared files
  lightly, and upstream what we can.
- **Privacy.** Live view and allocation by the gym are the sensitive parts. One consent, a visible
  log, an off switch per category, and no live data on disk.
- **Rules that exist twice.** Scopes, sync stamps and push copy have hand-copied or generated
  twins with parity tests. Each addition above names its twin.
- **Presence is in memory.** A server restart empties the floor board until the next heartbeat.
  Acceptable.
- **Operations.** Many small instances means provisioning, backups and upgrades must be scripted
  from the first gym, not the tenth.

## Testing

Per CLAUDE.md: pure helpers in `frontend/src/lib` with a test beside each (policy gate, brand
defaults, Today card selection), `node --test` for the api (new op and its inverse, the live
route's permission cases, allocation abuse cases), route entries in `openapi.yaml`, and new strings
in every locale pack.
