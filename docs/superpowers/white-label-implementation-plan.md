# White-label implementation plan (detailed)

Design: [../specs/2026-11-17-white-label-coach-gym-design.md](specs/2026-11-17-white-label-coach-gym-design.md).
Live state: [../HANDOVER.md](HANDOVER.md).

This plan is written so any person or any AI model can pick it up cold. It does not depend on
earlier conversations. Each work item has an id (P1.3), names its files, gives the contract to
build, the tests to write first, and the command that proves it is done.

Contents: [How to work](#how-to-work-from-this-plan) · [Rules](#rules) ·
[Codebase facts](#codebase-facts-you-will-need) · [Phase 0](#phase-0-groundwork) ·
[Phase 1](#phase-1-brand-layer) · [Phase 2](#phase-2-gym-policy-the-lean-member-app) ·
[Phase 3](#phase-3-coach-led-joining-and-train-this-today) · [Phase 4](#phase-4-live-floor) ·
[Phase 5](#phase-5-one-native-app-and-coach-events) · [Parked](#parked)

## How to work from this plan

**At the start of every session**, read in this order: `CLAUDE.md`, the design, this plan, then
`docs/superpowers/HANDOVER.md`. HANDOVER.md says which item is next. Do that item, nothing else.

**For every item**, in this order:

1. Read the files the item names. If they differ from what this plan says, the code wins: note the
   difference in HANDOVER.md under "Corrections to the plan".
2. Write the tests listed for the item. Run them and see them fail for the right reason.
3. Write the code until they pass.
4. Run the full checks below.
5. Tick the item in HANDOVER.md, record decisions, commit code and HANDOVER.md together with the
   commit message the item gives.

**If you stop half way**, write under "In progress" in HANDOVER.md: the item id, which numbered
step you reached, which tests pass, and what you were about to do.

**Starter prompt** to paste into a new assistant:

> You are continuing a planned project in this repository. Read CLAUDE.md, then
> docs/superpowers/specs/2026-11-17-white-label-coach-gym-design.md, then
> docs/superpowers/white-label-implementation-plan.md, then
> docs/superpowers/HANDOVER.md. Do the next unticked item in HANDOVER.md and only that item. Follow
> the plan's "Rules" and "For every item" sections. Then update HANDOVER.md, commit, and tell me
> what you did, what the checks printed, and what is next.

## Rules

1. **Flags off means upstream behaviour.** Every addition sits behind an env var or a file that is
   absent by default. With none set, the app and every existing test behave exactly as before. Do
   not edit an existing test to make it pass; if one fails, the change broke the default path.
2. **No new dependencies.** `CLAUDE.md` calls this a hard constraint.
3. **Pure logic goes in a helper with a test beside it.** `frontend/src/lib/x.js` with
   `x.test.js` (vitest); `api/x.js` with `api/test/x.test.js` (`node --test`).
4. **A new route needs its entry in `api/openapi.yaml`** (`api/test/openapi-routes.test.js`
   compares them), then `node scripts/build-api-docs.mjs`.
5. **A new `t()` string goes into every pack in `frontend/src/locales/`** (26 files: ar, bn, cs,
   da, de, es, fa, fr, hi, hu, id, it, ko, lt, pl, pt, pt-BR, ru, sv, th, tr, uk, vi, zh, zh-TW,
   and English is the key itself). pt-BR inherits from pt and has a fingerprint test.
6. **Code that exists twice changes on both sides**, with its parity test:
   `api/trainers/scopes.js` and `frontend/src/lib/sharing-scopes.js`;
   `frontend/src/lib/sync-merge.js` and `api/sync-stamps.js`;
   `frontend/src/lib/trainer-notices.js` generates `api/trainers/push-copy.js` through
   `api/scripts/build-trainer-copy.mjs`.
7. **User-facing text is plain, with no em dashes.**
8. **Never add, replace, enlarge or feed to an AI tool anything in `catalogue/media/`.**
9. **One phase, one branch, one pull request**: `wl/phase-1-brand`, `wl/phase-2-policy`,
   `wl/phase-3-coached`, `wl/phase-4-live`, `wl/phase-5-native`. One commit per item.
10. **Member data a trainer fetches stays in the screen's memory.** Never the store,
    `localStorage` or a file (docs/TRAINERS.md).

Checks before every commit (Node 22; run `npm ci` once in the repo root, `frontend/` and `api/`):

```bash
cd frontend && npm test
cd api && npm test
cd frontend && node scripts/check-locales.mjs && node scripts/check-source-strings.mjs --strict
node scripts/build-api-docs.mjs --check
```

One file while working:

```bash
cd api && node --test test/brand.test.js
cd frontend && npx vitest run src/lib/brand.test.js
```

## Codebase facts you will need

Checked on 2026-11-17 at commit `e4a68a94`. Line numbers drift; search for the quoted text.

**API (`api/server.js`, plain `node:http`)**
- Routes are entries of the `routes` object (`const routes = {`, about line 2542), keyed
  `'METHOD /path'`. The dispatcher looks the key up from the method and the path without the query.
- `json(res, code, obj, extraHeaders)` answers JSON with `Cache-Control: no-store`. For anything
  else, call `res.writeHead` and `res.end` yourself.
- `readBody(req)` parses a JSON body. `readSession(req)` returns the user or null.
- Env flags are read once at the top: `const TRAINERS = /^(1|true|yes|on)$/i.test(process.env.TRAINERS || '')`
  (about line 105). Copy that shape for new flags.
- `RP_NAME` (line 72) is the name passkey prompts show; default `'openGym'`.
- `GET /api/config` (about line 2555) builds one object with conditional spreads. The trainers
  block is the last line: `trainers: { on: true, leadAccess, roles }`.
- Trainer routes live in `api/trainers/routes.js` and `api/trainers/dashboard-routes.js`. Both
  wrap handlers in `guard(budgetName, async (req, res, user) => …)`, which answers 404 when
  trainers are off, 401 without a session, 429 over budget, and turns an `OrgError` into its
  status and code.
- `api/trainers/org-store.js` changes `org.json` only inside `mutate(o => …)`. Errors are
  `new OrgError(status, code, message)`.
- `api/permissions.js` has one `switch` on the action name (`'gym:staff'`, `'gym:invite'`, …).
- `api/budgets.js` has the `BUDGETS` table: `'trainer.list': { max: 120, windowMs: H }`.

**API tests.** A server test spawns the real server: see `api/test/server-config-session.test.js`.
It makes a temp `DATA_DIR`, writes `secret` and `db.json`, spawns `server.js` with `PORT=0`, and
waits for the port with `boundPort` from `api/test/helpers.mjs`. A session cookie is
`gymsid=<payload>.<hmac>` where payload is `${uid}:${expiryMs}:0` and the HMAC is SHA-256 with the
secret, base64url. Copy `startServer` and `mint` from that file.

**Frontend**
- `t()` is in `frontend/src/lib/i18n-core.js` (about line 85), with `tn` and `tnf` for plurals.
  `lib/i18n.js` re-exports them.
- `lib/api.js` is the only file that calls the server.
- `lib/modules.js` must stay free of imports. `isOn(S, id)` is the one gate; `setUnavailableModules`
  is the existing hook for "this build cannot have these".
- The route table is in `App.jsx` (about lines 399 to 437); the tab bar is `components/TabBar.jsx`.
- The accent is applied in `App.jsx` with `applyAccent(el, accentValue(S), theme)` from
  `lib/accent.js`. A profile's `accent` is a preset name, `'brand'`, or `'custom'`.
- The running workout is `S.active`. The server strips it on every write
  (`delete state.active` in `commitState`), so it never leaves the device.
- The heartbeat is in `views/Workout.jsx` (search `api('/api/activity'`), every 20 seconds.

**Sync.** `dayPlan` and `week` are per-key maps: a change writes the value and a stamp in
`S.edited['dayPlan.<date>']` or `S.edited['week.<day>']`. See `setWeek` and `noteEdited` in
`api/trainers/ops.js` for how a server-side change stamps itself.

## New settings, all optional

| Setting | Phase | Meaning |
|---|---|---|
| `DATA_DIR/brand/brand.json` present | 1 | This server wears a gym's brand. |
| `TRAINERS=on` | exists | Trainers, gyms and leads. Required from phase 2 on. |
| `TRAINERS_JOIN=member\|coached` | 3 | `coached`: the gym join code asks one consent that covers assigned trainers. Default `member`, as today. |
| `TRAINERS_LIVE=on` | 4 | The live floor board. |

---

## Phase 0: groundwork

Not code. Runs alongside. Answers go into HANDOVER.md under "Decisions".

| Id | Item | Owner |
|---|---|---|
| P0.1 | CLA workflows guarded to run only upstream (in the planning PR; effective once merged to `main`) | done |
| P0.2 | Licence for the exercise media from Gym visual for a rebranded product. Until then, demos use the stock openGym build | you |
| P0.3 | Talk to the upstream maintainer: commercial licence, and which generic pieces he would take upstream | you |
| P0.4 | Product name and domain pattern, for example `<gym>.yourbrand.app`. Passkeys bind to the hostname, so a gym's domain is fixed once members sign up | you |
| P0.5 | Pilot gym: square logo PNG at 512 px, wide logo PNG, brand colour as hex, trainer and member counts | you |
| P0.6 | Lawyer reads the `gym-coached` consent text (P3.2) before real members see it | you |

---

## Phase 1: brand layer

**Goal.** One server can carry a gym's name, logo, colour and home-screen icon with no code change
per gym. **Branch:** `wl/phase-1-brand`. **Depends on:** nothing. **Size:** 1 to 2 weeks.

The brand folder a gym gets:

```
DATA_DIR/brand/
  brand.json        required; without it the server is plain openGym
  logo.png          optional, square, shown on sign-in
  logo-wide.png     optional, shown in headers
  icon-180.png      optional, home-screen icon
  icon-512.png      optional, install and notification icon
```

```json
{
  "name": "Iron Temple",
  "shortName": "IronTemple",
  "accent": "#d4af37",
  "themeColor": "#0c0e12",
  "backgroundColor": "#0c0e12",
  "supportEmail": "hello@irontemple.example",
  "supportUrl": "https://irontemple.example/help",
  "privacyUrl": "https://irontemple.example/privacy",
  "termsUrl": "https://irontemple.example/terms",
  "sourceUrl": "https://github.com/arm-pz/openGyms"
}
```

### P1.1 `api/brand.js`

**Files:** create `api/brand.js`, `api/test/brand.test.js`.

**Contract**

```js
export const DEFAULT_BRAND = {
  name: 'openGym', shortName: 'openGym',
  accent: null,                       // null: the app's own default accent
  themeColor: '#0c0e12', backgroundColor: '#0c0e12',
  supportEmail: '', supportUrl: '', privacyUrl: '', termsUrl: '',
  sourceUrl: 'https://github.com/DuarteSantos8/openGym',
};
export const ASSET_NAMES = ['logo', 'logo-wide', 'icon-180', 'icon-512'];

/** Every field present. Anything invalid becomes that field's default. Never throws. */
export function cleanBrand(raw) {}

/** { brand, custom, assets: { logo: bool, 'logo-wide': bool, 'icon-180': bool, 'icon-512': bool } }
 *  custom is true only when brand.json exists and parses. A missing folder is not an error. */
export function loadBrand(dataDir) {}

/** Absolute path of an asset, or null: unknown name, missing file, or not a PNG. */
export function assetPath(dataDir, name) {}

/** The web app manifest for this brand. `assets` decides whether icons point at the brand route. */
export function manifestFor(brand, assets) {}
```

**Validation rules**
- `name`: string, control characters removed, trimmed, 1 to 40 chars. `shortName`: 1 to 12; when
  missing, the first 12 chars of `name`.
- Colours: `/^#[0-9a-f]{6}$/i` only, stored lowercase.
- URLs: must parse and start with `https://`, at most 200 chars. `supportEmail`: contains one `@`,
  at most 120 chars.
- `assetPath` reads the first 8 bytes and requires the PNG signature
  `89 50 4E 47 0D 0A 1A 0A`. PNG only: an SVG can carry script.
- `manifestFor` returns the same keys as `frontend/public/manifest.json`. With a custom icon the
  `src` is `api/brand/asset?name=icon-512`; without, `icon-512.png`. Same for 180.

**Tests to write first**
1. `cleanBrand(undefined)` deep-equals `DEFAULT_BRAND`.
2. A valid object survives unchanged apart from lowercase colours.
3. Each bad field falls back alone: `accent: 'red'`, `accent: 'url(x)'`, `name: ''`, a 60 char
   name is cut to 40, `supportUrl: 'javascript:alert(1)'`, `supportUrl: 'http://x'`, `name: 42`.
4. `loadBrand` on a temp dir with no `brand` folder: `custom: false`, all assets false.
5. `loadBrand` with a broken `brand.json`: `custom: false`, and it does not throw.
6. `assetPath` returns null for `../secret`, for an unknown name, and for a file named
   `logo.png` that holds text.
7. `manifestFor(DEFAULT_BRAND, {})` deep-equals the parsed `frontend/public/manifest.json`. This
   is the parity test: read the file from the repo inside the test.

**Verify:** `cd api && node --test test/brand.test.js`
**Commit:** `Brand: api/brand.js, the brand file and its manifest`

### P1.2 Brand routes

**Files:** modify `api/server.js`, `api/openapi.yaml`; create `api/test/server-brand.test.js`.

Load the brand once at boot next to the other top-level constants:
`const BRAND = loadBrand(DATA_DIR);`

| Route | Auth | Answer |
|---|---|---|
| `GET /api/brand` | none | `200 { ...brand, custom, assets }` through `json()` |
| `GET /api/brand/manifest.json` | none | `200`, `Content-Type: application/manifest+json`, `Cache-Control: no-cache`, body `manifestFor(...)` |
| `GET /api/brand/asset?name=` | none | `200 image/png` with `Cache-Control: public, max-age=3600` and an `ETag` from size and mtime; `304` on a matching `If-None-Match`; `404 { error, code: 'no-asset' }` otherwise |

All three are public on purpose: the sign-in screen draws them before anyone has a session. They
reveal nothing a visitor to the page does not see.

**Tests to write first** (copy `startServer` from `server-config-session.test.js`)
1. No brand folder: `GET /api/brand` gives `custom: false` and `name: 'openGym'`; the manifest
   equals the static file; any asset is 404.
2. With a brand folder (write `brand.json` and a 1 pixel PNG as `icon-512.png`): name and colours
   come back, `assets['icon-512']` is true, the asset answers `image/png` with the same bytes, and
   a second request with `If-None-Match` answers 304.
3. `GET /api/brand/asset?name=../secret` is 404, and so is `name=logo` when only the icon exists.
4. The existing `GET /api/config` answer is unchanged (the existing test already proves this; run it).

**openapi:** add the three paths under a new `Brand` tag, then `node scripts/build-api-docs.mjs`.

**Verify:** `cd api && node --test test/server-brand.test.js test/openapi-routes.test.js`
**Commit:** `Brand: public routes for the brand, its manifest and its images`

### P1.3 Serving the manifest

**Files:** modify `web/nginx.conf.template`, `frontend/vite.config.js`; check `helm/` and
`kubernetes/` for a second copy of the nginx config; check `frontend/public/sw.js`.

Only the manifest needs proxying. The icons are reached through the manifest and the brand route,
so no icon fallback is needed in nginx.

1. In `web/nginx.conf.template`, add an exact-match location that proxies
   `${BASE_PATH}/manifest.json` to the api's `/api/brand/manifest.json`. Copy the `proxy_pass`
   target and headers from the existing `location ^~ ${BASE_PATH}/api/` block. It must be
   `location = …`: an exact match beats the regex location `\.(js|css|json|html)$` further down,
   which would otherwise serve the static file. Repeat the security headers of that regex block,
   because an `add_header` inside a location replaces the inherited set (the template's own
   comments explain both traps).
2. In `frontend/vite.config.js` (the `proxy` object, about line 117), add
   `'/manifest.json': { target: backend, changeOrigin: true, rewrite: () => '/api/brand/manifest.json' }`.
3. `frontend/public/sw.js` names `icon-512.png` for notifications (about line 216). Change it to
   use an `icon` from the push payload when present, else the static file. P1.7 sends that field.
4. If a second nginx config exists under `helm/` or `kubernetes/`, mirror step 1 and run
   `api/test/k8s-web-port.test.js`, which reads those files.

**Verify by hand** (write the result in HANDOVER.md):

```bash
cp .env.example .env && docker compose up -d --build
curl -s localhost:8080/manifest.json | head -5        # default: name "openGym"
mkdir -p data/brand && echo '{"name":"Iron Temple"}' > data/brand/brand.json   # then restart the api
curl -s localhost:8080/manifest.json | head -5        # the gym's name
```

**Commit:** `Brand: nginx and the dev server serve the brand's manifest`

### P1.4 `frontend/src/lib/brand.js`

**Files:** create `frontend/src/lib/brand.js`, `brand.test.js`; modify `lib/api.js`, `App.jsx`.

**Contract**

```js
export const DEFAULT_BRAND = { /* same fields as api/brand.js, plus custom: false, assets: {} */ }
export function cleanBrand(raw) {}        // lenient copy of the server's rules; never throws
export function getBrand() {}             // the current brand, DEFAULT_BRAND until one is set
export function setBrand(raw) {}          // cleans, stores in memory and in localStorage 'gym_brand_v1'
export function loadCachedBrand() {}      // reads localStorage; wrapped in try/catch
export const isCustom = () => getBrand().custom === true
export function assetUrl(name) {}         // appBase + 'api/brand/asset?name=…', or null when absent
export function onBrandChange(fn) {}      // subscribe; returns an unsubscribe
```

- `lib/api.js` gets `export const fetchBrand = () => api('/api/brand')`, swallowing errors to null.
- `App.jsx`: before the first render call `loadCachedBrand()`, then `fetchBrand().then(setBrand)`.
  A cold offline start therefore still shows the gym's name.
- On the phones (`MOBILE`), the brand comes from the paired server; with no server, the default.

**Tests to write first**
1. `getBrand()` before anything is set equals `DEFAULT_BRAND`.
2. `setBrand` with junk colours keeps the defaults for those fields.
3. `setBrand` then a fresh `loadCachedBrand` (simulate with a stubbed `localStorage`) gives the
   same brand back.
4. `localStorage.getItem` throwing does not throw out of `loadCachedBrand`.
5. `assetUrl('logo')` is null when `assets.logo` is false.
6. `onBrandChange` fires once per `setBrand` and not after unsubscribe.

**Verify:** `cd frontend && npx vitest run src/lib/brand.test.js`
**Commit:** `Brand: the app loads and caches the server's brand`

### P1.5 Apply the brand in the UI

**Files:** modify `components/BrandLogo.jsx`, `App.jsx`, `lib/accent.js` (read only unless needed),
`views/Login.jsx`, `views/MobileOnboarding.jsx`, `components/LiveGlance.jsx`,
`components/ShareWorkout.jsx`, `frontend/index.html` is left alone (it is static).

1. `BrandLogo` and `BrandMark`: when `isCustom()`, draw `<img src={assetUrl('logo-wide') || assetUrl('logo')}>`
   at the same height, with `alt` set to the brand name. With no image, draw the name as text in
   the lockup's type. When not custom, draw exactly what is drawn today.
2. In `App.jsx`, in an effect on brand change: set `document.title`, the content of
   `meta[name="apple-mobile-web-app-title"]` and `meta[name="theme-color"]`, and point
   `link[rel="icon"]` and `link[rel="apple-touch-icon"]` at `assetUrl('icon-180')` when present.
3. Accent. Where `App.jsx` calls `applyAccent(de, accent, theme)`: when the brand is custom and
   has an `accent`, and the profile's own choice is `'brand'` or unset, pass the brand's hex. A
   profile that picked a colour keeps it. Do not write anything to the store. Put the decision in
   a small pure function in `lib/brand.js`, `accentFor(S, brand)`, and test it.
4. `ShareWorkout.jsx` and `LiveGlance.jsx`: replace the drawn openGym mark and name with the
   brand's.

**Tests**
- `BrandLogo.test.jsx` (new): default render matches today's markup; a custom brand with a logo
  renders an `img` with the right `src` and `alt`; a custom brand without a logo renders the name.
- `brand.test.js`: `accentFor` for the four cases (custom brand with and without accent, profile
  with its own pick, default brand).
- `accent.migration.test.js` and every existing `Settings.accent`, `Settings.themes` test pass
  untouched.

**Commit:** `Brand: logo, title, icons and default accent follow the brand`

### P1.6 The word "openGym" in text

**Files:** modify `frontend/src/lib/i18n-core.js`, `lib/i18n.js` (re-export), the brand effect in
`App.jsx`; create or extend `i18n-core` tests.

"openGym" appears in about 124 source files and in many locale strings. Do not edit them.

1. In `i18n-core.js` add a module variable and a setter:
   `let brandName = null; export function _setBrandName(n) { brandName = n || null }`.
2. Split `t`: move today's body into an internal `translate(s, args)`. In it, do the brand
   replacement on the translated template **before** the `{0}` arguments are filled in, so a
   routine the user named "openGym push" is never rewritten:
   `if (brandName && v.includes('openGym')) v = v.replaceAll('openGym', brandName)`.
   Apply the same in `tn` and `tnf`.
3. Export `tRaw(s, ...args)`: the same without the replacement, for text that means the
   open-source project.
4. `App.jsx` calls `_setBrandName(isCustom() ? getBrand().name : null)` on brand change.
5. Find the places that must keep saying openGym and switch them to `tRaw`: the About page, the
   licence and third-party notices, the "source code" link, the update checker's text about
   openGym releases. `grep -rn "openGym" frontend/src/views/Settings.jsx frontend/src/views/settings-pages.js`
   is the starting point. List what you changed in HANDOVER.md.

**Tests to write first**
1. With no brand name, `t('Welcome to openGym')` is unchanged.
2. With brand name `Iron Temple`, it reads `Welcome to Iron Temple`.
3. `t('Routine: {0}', 'openGym push')` keeps `openGym push`.
4. `tRaw('Built on openGym')` keeps `openGym` with a brand name set.
5. `tn` with a brand name set replaces in both plural forms.

**Verify:** `cd frontend && npx vitest run src/lib/i18n` then the two string checks.
**Commit:** `Brand: the app's name in text follows the brand`

### P1.7 Server-side names

**Files:** modify `api/push-messages.js` (the literal `'openGym'` near lines 72 and 100),
`api/server.js` (`RP_NAME`, and where push payloads are built); extend `api/test/push-messages.test.js`.

1. `push-messages.js` functions that return a title take the app name as a parameter with
   `'openGym'` as default, so their existing tests pass as they are.
2. `server.js` passes `BRAND.brand.name`.
3. `RP_NAME`: `process.env.RP_NAME || (BRAND.custom ? BRAND.brand.name : 'openGym')`.
4. When `BRAND.assets['icon-512']`, add `icon: 'api/brand/asset?name=icon-512'` to push payloads
   (the service worker reads it since P1.3).
5. `api/trainers/push-copy.js` was checked: it holds no "openGym" literal. Nothing to do there.

**Tests:** the default title is still `openGym`; with a name passed, the title is that name.
**Commit:** `Brand: push titles and the passkey prompt use the brand's name`

### P1.8 About row

**Files:** modify the About section in `views/Settings.jsx` (or its page in `settings-pages.js`);
add strings to all locales.

When the brand is custom, the About page shows a row "Built on openGym" (`tRaw`), the running
version, and "Source code" linking to `brand.sourceUrl`. Under AGPL section 13, people using the
app over a network must be offered its source, so this row is shown whatever the brand says and
`sourceUrl` must point at the code actually running (the fork, not upstream, once it differs).

**Test:** a Settings test in the style of `Settings.account-id.test.jsx`: with a custom brand the
row and the link are present; with the default brand the page is as before.
**Commit:** `Brand: About names openGym and links the source`

### P1.9 Provisioning

**Files:** create `scripts/provision-gym.mjs`, `scripts/provision-gym.test.mjs` (or a test beside
the other script tests; check `scripts/README.md` for how they are run), `docs/WHITE_LABEL.md`,
`docs/examples/brand.json`, `docs/examples/gym.json`; modify `docs/README.md`, `scripts/README.md`,
`.env.example` (commented lines only), the Helm chart under `helm/`.

**Contract**

```
node scripts/provision-gym.mjs gym.json --out ./deploy/iron-temple
```

`gym.json`: `{ slug, host, name, shortName, accent, themeColor, backgroundColor, support…, sourceUrl,
logo, logoWide, icon180, icon512 }` where the last four are paths to PNG files.

Writes into `--out`: `.env` (`RP_ID=<host>`, `ORIGIN=https://<host>`, `TRAINERS=on`,
`INVITE_ONLY=on`, `PASSWORD_LOGIN` left off), `data/brand/brand.json`, the PNGs copied under their
fixed names, and a `docker-compose.yml` that references the published images. It refuses to
overwrite an existing `--out` and refuses a non-PNG or a host that is not a plain hostname.

**Helm:** a `brand` value: an existing ConfigMap or volume name mounted at `DATA_DIR/brand`.

**`docs/WHITE_LABEL.md`** covers: what the brand folder holds, the provisioning command, that the
hostname is permanent once members hold passkeys, backups of `data/`, how to roll a new image
across gyms, the licence notes from the design, and a checklist of screens to look at for a new gym.

**Tests:** the script on a temp dir writes the expected files; bad host, missing logo, non-PNG and
an existing output folder each exit non-zero with a clear message.
**Commit:** `Brand: provisioning script, Helm value and the white-label guide`

### Phase 1 done when

- [ ] With `data/brand/` present: browser tab, sign-in screen, installed PWA name and icon, a test
      push and the share image all show the gym. No screen says openGym except About.
- [ ] With `data/brand/` absent: `GET /api/brand` answers the defaults and every pre-existing test
      passes without edits.
- [ ] `docker compose up -d --build` with the sample brand works end to end.
- [ ] All four checks pass.

**Handover:** the brand schema as built, the sample `gym.json`, the list of `tRaw` call sites, a
screenshot per screen in the checklist, the manual verification output from P1.3.

---

## Phase 2: gym policy, the lean member app

**Goal.** The gym decides what its members' app shows. Staff keep the full app.
**Branch:** `wl/phase-2-policy`. **Depends on:** `TRAINERS=on`. **Size:** 1 to 2 weeks.

The policy object, stored on a gym in `org.json`:

```json
{ "modules": { "coach": false, "community": false, "body": true }, "locked": true, "nav": "member" }
```

### P2.1 Policy helper and its copy

**Files:** create `api/trainers/policy.js`, `frontend/src/lib/gym-policy.js`,
`frontend/src/lib/gym-policy.test.js` (holds the parity test, modelled on
`sharing-scopes.test.js`), `api/test/policy.test.js`.

```js
export const MODULE_IDS = ['effort','plates','strength','videos','watch','checkin','body','health',
  'motivation','coach','integrations','community','seasonal'];   // copy of modules.js MODULE_IDS
export const NAVS = ['full', 'member'];
export const MEMBER_DEFAULT = { modules: { /* body, plates, watch: true; the other ten: false */ },
  locked: true, nav: 'member' };
/** null for no policy; otherwise a clean object. Unknown module ids and non-booleans are dropped. */
export function cleanPolicy(raw) {}
/** Module ids the policy forces off. */
export const policyOff = policy => (policy ? MODULE_IDS.filter(id => policy.modules[id] === false) : []);
```

**Tests:** `cleanPolicy(null)` is null; unknown ids dropped; `nav: 'x'` becomes `'full'`;
`MEMBER_DEFAULT` survives `cleanPolicy` unchanged; the two files export identical `MODULE_IDS`,
`NAVS` and `MEMBER_DEFAULT`; the frontend copy's `MODULE_IDS` equals `modules.js` `MODULE_IDS`.
**Commit:** `Policy: the gym policy shape, on both sides`

### P2.2 Storage, route, config

**Files:** modify `api/trainers/org-store.js` (`updateGym`), `api/permissions.js`,
`api/trainers/dashboard-routes.js`, `api/server.js` (the `trainers` block of `GET /api/config`),
`api/openapi.yaml`, `api/test/permissions.test.js`; create `api/test/server-policy.test.js`.

1. `updateGym(gid, patch)`: accept `patch.policy` (already cleaned; `null` removes it).
2. `permissions.js`: new case `'gym:policy'`, allowed for admins and for a lead of that gym.
   Model it on `'gym:invite'`.
3. `POST /api/gym/policy { gymId, policy }` in `dashboard-routes.js` under `guard('trainer.answer', …)`:
   checks `can(user, 'gym:policy', { gymId })`, cleans, stores, audits (`gym.policy`), answers
   `{ ok: true, policy }`.
4. `GET /api/gym` (the lead's view) includes the gym's `policy`.
5. `GET /api/config`: add `policy` inside the `trainers` block. Rule, as one function
   `policyFor(uid)` in `policy.js` taking the org: the policy of the user's active gym memberships,
   skipping any gym where they are staff; with several, a module is off if any policy turns it off,
   `locked` if any is locked, `nav: 'member'` if any says so; no membership gives `null`.

**Tests:** permissions: a lead of another gym, a trainer of this gym and a plain member are all
refused; this gym's lead and an admin are allowed. Server: a member gets the policy in
`/api/config`; a trainer of the same gym does not; a user in no gym does not; with `TRAINERS`
unset the config answer is byte-for-byte as before.
**Commit:** `Policy: a gym's lead sets it, members receive it`

### P2.3 The module gate

**Files:** modify `frontend/src/lib/modules.js`, `modules.test.js`, and the place that reads
`/api/config` for the trainers block (search `trainers?.roles` or `trainers.roles` in
`frontend/src`).

```js
let policyOffSet = new Set(); let policyLocked = false
export function setPolicyModules(ids, locked) { policyOffSet = new Set(Array.isArray(ids) ? ids : []); policyLocked = !!locked }
export const modulesLocked = () => policyLocked
// in isOn, right after the `unavailable` line:
if (policyOffSet.has(id)) return false
```

Keep the file free of imports: the caller passes `policyOff(policy)` from `gym-policy.js`.
When the config has no policy, call `setPolicyModules([], false)`.

**Tests:** policy off beats `S.modules[id] === true`; no policy changes nothing (run the whole
existing `modules.test.js` untouched); an unknown id is harmless; `moduleStates` reflects it.
**Commit:** `Policy: the module gate obeys the gym`

### P2.4 Server-side readers of `S.modules`

**Files:** modify `api/wrapped.js`, `api/coach/core/due.js`, `api/integrations/index.js`, and
their tests.

Each reads `S.modules` today. Give each an optional `policyOff` list (default empty) from
`policyFor(uid)` and treat those modules as off. `api/coach/core/` is also imported by the phones
(`lib/coach-local.js`), so the parameter must be optional and the module must stay free of
server-only imports.

**Tests:** for each of the three: with the module forced off by policy, nothing is sent; with no
policy, behaviour is as before.
**Commit:** `Policy: scheduled reviews, Wrapped and webhooks obey the gym`

### P2.5 Member navigation

**Files:** create `frontend/src/lib/member-nav.js`, `member-nav.test.js`; modify
`components/TabBar.jsx`, `App.jsx`.

```js
/** policy: the cleaned policy or null. roles: { trainer, lead } from the config. */
export function navFor(policy, roles) {}
// -> { mode: 'full' | 'member', tabs: [{ id, path, label }], allows: path => boolean }
```

Member mode applies when `policy?.nav === 'member'` and the user is neither trainer nor lead.
Tabs: Today (`/home`), History (`/history`), Progress (`/stats`), Profile (`/settings`).
Blocked paths in member mode: `/plan` and below except viewing a routine, `/library`, `/tools`,
`/journal`, `/structural-balance`, `/community`, `/assistants`, `/coach` and below. `/workout`,
`/muscles`, `/measurements` and `/progress-photos` (when `body` is on) stay.
In `App.jsx`, a blocked path redirects to `/home`, the same way the catch-all route does.

**Tests:** a table of paths against `allows`; a trainer with a member policy still gets full nav;
no policy gives full nav; labels go through `t()`.
**Commit:** `Policy: four tabs for a gym's members`

### P2.6 Settings

**Files:** modify `views/settings-pages.js` (the `PAGES` and `SEARCH` tables already carry a `mod`
key; add `member: true` to the pages a member keeps), `views/Settings.jsx`, `views/Modules.jsx`;
tests beside `settings-pages.test.jsx` and `Modules.test.jsx`.

Member keeps: units, language, notifications, look (theme only), Trainer and gym, account, About.
With `modulesLocked()`, the Modules page shows one line, "Your gym chose what this app shows",
and no switches, and the Settings search does not list module switches.
**Commit:** `Policy: a member's Settings`

### P2.7 The lead's editor

**Files:** modify `views/Trainer.jsx` (the lead's gym view), `lib/trainer-api.js`; add a test in
`Trainer.test.jsx`; strings in every locale.

A "Member app" card: a "Member preset" button (`MEMBER_DEFAULT`), one switch per module with the
titles from `modules-copy.js`, "Members can change this" (the inverse of `locked`), and a "Full
app or four tabs" choice. Saves through `POST /api/gym/policy`.
**Commit:** `Policy: the gym lead edits the member app`

### Phase 2 done when

- [ ] A member of a gym with the Member policy sees four tabs and none of the hidden surfaces, on
      web and in the installed PWA.
- [ ] A trainer account on the same server sees the full app.
- [ ] Removing the policy brings everything back with no data lost.
- [ ] With `TRAINERS` unset, nothing changed.

**Handover:** the policy schema as built, the blocked-path list, screenshots of member Home and
trainer Home side by side, the accounts used to test.

---

## Phase 3: coach-led joining and "train this today"

**Goal.** A member joins once, the gym puts them on a trainer's list, the trainer sets today's
workout in two taps. **Branch:** `wl/phase-3-coached`. **Depends on:** phase 2. **Size:** 2 to 3 weeks.

### P3.1 The `live` category

**Files:** modify `api/trainers/scopes.js`, `frontend/src/lib/sharing-scopes.js` and its parity
test, `views/MyTrainer.jsx` (the consent screen lists categories), `api/trainers/projections.js`
(must ignore `live`: it has no stored data), `mcp/src/labels.js` if it names categories, every
locale.

```js
{ id: 'live', max: 'view' }                       // appended to CATEGORIES on both sides
TRAINER_DEFAULTS.live = 'off'
export const COACHED_DEFAULTS = { metrics: 'view', workouts: 'apply', plans: 'apply',
  bodyweight: 'view', measurements: 'off', photos: 'off', notes: 'off', nutrition: 'off', live: 'view' };
```

An existing grant has no `live` key; `cleanScopes` already turns a missing key into `'off'`. Add a
test that says so. Label: "Your workout while you train".
**Commit:** `Coached: a live category in sharing`

### P3.2 Consent text

**Files:** modify `api/consent.js` (`CONSENT_TEXTS`), its test, the frontend's translated copy of
consent texts (search for the `lead` text in `frontend/src` to find it), every locale.

```js
'gym-coached': { v: 1, text:
  'You are joining this gym. The trainers this gym assigns to you can see the categories marked here, '
  + 'see your workout while you train, and change the ones marked apply. You are told each time a trainer is assigned. '
  + 'You can switch any category off or leave the gym any time, and it stops working the moment you do. '
  + 'Every look and every change is written to your sharing log, which only you can read.' }
```

Do not ship to real members before P0.6. A changed text is a new `v`, never an edit.
**Commit:** `Coached: the consent text for a coach-led gym`

### P3.3 `TRAINERS_JOIN=coached`

**Files:** modify `api/server.js` (read the env, pass `joinMode` into `trainerRoutes`),
`api/trainers/routes.js` (`POST /api/sharing/code` and `/connect`), `api/trainers/org-store.js`
(`joinGym`), `api/openapi.yaml`, `docs/SELF_HOSTING.md` (the env table); tests in
a new `api/test/server-coached.test.js` (see `server-trainers.test.js` for the setup).

With `coached`:
- `POST /api/sharing/code` for a gym code answers, besides today's fields,
  `coached: true, defaults: COACHED_DEFAULTS, consent: currentConsent('gym-coached')`.
- `POST /api/sharing/connect` for a gym code checks the body's consent against `gym-coached`
  (`consentBody(res, body.consent, 'gym-coached')`), cleans `body.scopes`, and `joinGym` stores on
  the membership `coached: { scopes, consentId }` with a consent record of kind `gym-coached`
  whose `subjectId` is the gym id. The lead row is made as today.
- Everything else, and everything with the env unset or `member`, answers as today.

**Tests:** with the env unset the two routes answer exactly as before (compare whole bodies); with
`coached` the code lookup carries the new consent, a connect with the `lead` consent hash is
refused 409 `consent-outdated`, and a correct one stores the membership's `coached` block.
**Commit:** `Coached: one consent when joining the gym`

### P3.4 Allocation

**Files:** modify `api/trainers/org-store.js`, `api/permissions.js`, `api/trainers/dashboard-routes.js`,
`api/openapi.yaml`, `frontend/src/lib/trainer-notices.js` (new kind `allocated`), then regenerate
`api/trainers/push-copy.js`; every locale; `docs/TRAINERS.md`.

- `permissions.js`: case `'gym:allocate'`, admins and this gym's leads.
- `org-store.js`: `allocate({ gymId, memberId, trainerId, lang })` inside `mutate`. Refusals, each
  an `OrgError`: 404 `no-member` (no active membership), 409 `no-consent` (membership has no
  `coached` block, or its consent record is revoked), 404 `no-trainer` (not active staff with role
  `trainer` in this gym), 409 `already-connected`. Otherwise it pushes a grant
  `{ kind: 'trainer', granteeId: trainerId, gymId, scopes: <the membership's coached scopes>,
  status: 'active', via: 'allocation', consentId }`.
  `unallocate({ gymId, memberId, trainerId })` ends that grant through the existing `endGrant`.
- When the member later changes scopes on such a grant (`updateScopes`), also write the result
  back to the membership's `coached.scopes`, so the next allocation never raises a category the
  member lowered.
- Routes `POST /api/gym/allocate` and `/unallocate` under `guard('trainer.answer', …)`: permission
  check, call the store, `log.append(memberId, { ev: 'grant.allocated', … })`,
  `notify.notice(memberId, 'allocated', …)` and a push, audit.
- `GET /api/gym` lists for each member whether they joined coached and who their trainers are.

**Commit:** `Coached: the gym lead puts a member on a trainer's list`

### P3.5 Abuse tests

**Files:** `api/test/permissions.test.js` and `api/test/trainers-core.test.js`. One test each:

1. A lead of another gym allocating here: refused, 404.
2. A member who joined under the old `lead` consent: 409 `no-consent`.
3. A member who left the gym: 404 `no-member`.
4. A trainer who is not staff of this gym: 404 `no-trainer`.
5. A member who switched `workouts` off on a first allocation, then is allocated to a second
   trainer: the new grant has `workouts: 'off'`.
6. `TRAINERS_JOIN` unset: the route answers 404 `not-coached`.
7. An allocated trainer removed from the gym: the grant reads paused, as with any gym trainer.
8. A hand-edited `coached.scopes` with `photos: 'apply'`: capped at the category's maximum.

**Commit:** `Coached: abuse cases for allocation`

### P3.6 Op `today.set`

**Files:** modify `api/trainers/ops.js`, `api/test/trainer-ops.test.js`.

```js
// OP_CATEGORY: 'today.set': 'plans'
// cleanOps:   { op: 'today.set', date: '2026-11-17', id?: <routine id> }   id absent: follows a routine.put
// applyOps:
case 'today.set': {
  const rid = o.id ?? lastRoutine;
  if (rid == null || !list(S, 'routines').some(r => r?.id === rid)) throw new OpError(404, 'no-routine', 'no such routine');
  const before = isMap(S.dayPlan) && S.dayPlan[o.date] != null ? clone(S.dayPlan[o.date]) : null;
  rec.inverse.push({ list: 'dayPlan', key: o.date, value: before });
  setDayPlan(S, o.date, rid, t);          // like setWeek: writes the value and noteEdited(S, `dayPlan.${o.date}`, t)
  rec.summary = { kind: o.op, d: o.date, name: … };
  rec.touched.push({ list: 'dayPlan', key: o.date });
}
```

- `cleanOps` requires `isISO(date)` and a date within one day of the server's UTC date.
- `applyRestores` and `fingerprintOf` must learn the `dayPlan` list the way they know `week`.
  Read both functions before writing.
- `S.dayPlan[iso]` is a scalar: a routine id, or the `'rest'` sentinel
  (`frontend/src/lib/history.js`, about line 579). Do not write an array.

**Tests:** sets the id and stamps `edited['dayPlan.<date>']` later than any stamp in the document;
the inverse restores a previous value and restores "nothing"; a date three days out is 400
`bad-op`; an unknown routine is 404; after `routine.put` with no `id` it uses the new routine;
merging the result with an older client document through `sync-merge.js` keeps the trainer's value.
**Commit:** `Today: a trainer op that sets today's routine`

### P3.7 Route `POST /api/trainer/today`

**Files:** modify `api/trainers/dashboard-routes.js`, `api/openapi.yaml`, `docs/TRAINERS.md`,
`frontend/src/lib/trainer-api.js` and its test.

Body: `{ memberIds: [1..50], routineId, date, message? }`. Answer: the same shape as assign,
`{ ok, results: [{ memberId, name, outcome: 'applied'|'suggested'|'skipped', reason? }] }`.

Refactor first, behaviour unchanged: move the body of `POST /api/trainer/assign` from "read the
trainer's own routine" to the end of the per-member loop into
`assignTo(user, { memberIds, routineId, tailOps, message, noticeKind })`. Run the existing assign
tests: they must pass untouched. Then `assign` calls it with `tailOps` of `week.add` or nothing,
and `today` calls it with `[{ op: 'today.set', date }]` and `noticeKind: 'today'`.
`today.set` is internal like `week.add` in that it follows the `routine.put`; unlike it, it is
also allowed in `POST /api/trainer/ops` with an explicit `id`.
**Commit:** `Today: one call sets a routine for today for up to 50 members`

### P3.8 Notice and push

**Files:** `frontend/src/lib/trainer-notices.js` (add kind `today` to `TRAINER_PUSH`: title
"{0} set today's workout", body "{0}"), its text module `trainer-notice-text.js`, every locale,
then `node api/scripts/build-trainer-copy.mjs` to regenerate `api/trainers/push-copy.js`, and
`trainer-notices.test.js`.
**Commit:** `Today: the member is told`

### P3.9 The member's Today card

**Files:** create `frontend/src/lib/today-card.js`, `today-card.test.js`; modify `views/Home.jsx`;
add `Home.today-card.test.jsx` beside `Home.startdoor.test.jsx`.

```js
/** What Home shows for today. */
export function todayCard(S, todayIso) {}
// -> null | { routineId, name, from: string|null, rest: boolean, done: boolean }
```

Reads `S.dayPlan[todayIso]`, falls back to the week plan and queue exactly as the existing start
door does (reuse its helper; do not copy the logic). `from` is `routine._by.name` when
`_by.kind === 'trainer'`. `done` when a workout of that routine is logged today.
Home shows: "Today, from {0}", the routine's name, exercise count, and Start. In member nav it is
the first thing on the screen.
**Commit:** `Today: the member's Home leads with today's workout`

### P3.10 Console

**Files:** modify `views/Trainer.jsx`, its CSS, `lib/trainer-api.js`; tests in `Trainer.test.jsx`;
strings in every locale.

- From 900 px wide: two panes, the roster on the left, the selected member on the right. Below
  that, today's one-column layout, unchanged.
- Each roster row: a "Today" button opening a picker over the trainer's own routines (from the
  store's `S.routines`), then send. A checkbox per row and "Today for selected" for a class.
- The result list shows applied, suggested and skipped per member with the reason in words.
- The lead's gym view: per member, "Assign to trainer" with the gym's trainers, and "Remove".

**Commit:** `Console: a wide layout, Today in two taps, allocation for the lead`

### Phase 3 done when

- [ ] With `TRAINERS_JOIN=coached`: a new member scans the gym code and accepts once, the lead
      allocates them, the trainer sets Pull for today, and the member's phone shows the push and
      the Today card. Nothing is asked of the member after the first consent.
- [ ] The member's Undo puts the previous plan for today back.
- [ ] The member switches `plans` off and the trainer's next "Today" for them is skipped with the
      reason shown.
- [ ] With the env unset, joining behaves exactly as before.

**Handover:** the consent text as shipped with its version and hash, the allocation rules as
built, a walkthrough script with the four accounts (admin, lead, trainer, member).

---

## Phase 4: live floor

**Goal.** The trainer sees each client's exercise, set and rest in near real time.
**Branch:** `wl/phase-4-live`. **Depends on:** P3.1. **Size:** 2 weeks.

The presence row, memory only:

```json
{ "name": "Pull", "exIdx": 3, "exTotal": 6, "setsDone": 9, "setsTotal": 18, "startedAt": 1700000000000,
  "updatedAt": 1700000600000,
  "ex": { "id": "barbell-row", "name": "Barbell Row" },
  "sets": [ { "w": 80, "r": 8, "unit": "kg", "done": true, "type": "" }, { "w": 80, "r": 8, "unit": "kg", "done": false, "type": "" } ],
  "restUntil": 1700000645000 }
```

### P4.1 The payload

**Files:** create `frontend/src/lib/live-presence.js`, `live-presence.test.js`; modify
`views/Workout.jsx` (the heartbeat effect).

```js
/** The heartbeat body for the running workout A (S.active). detail: whether to add ex, sets, restUntil. */
export function presenceBody(A, { detail, restUntil, exName }) {}
```

Move today's computation (`supersetUnits`, `setsDoneActive`, `setUnitsTotal`) into it unchanged,
so `Workout.jsx` only calls it. With `detail`, add the current exercise and at most 20 of its
sets. `detail` is true only when `/api/config` says `trainers.live === true` for this user (P4.4
sets it when the member has at least one active grant with `live` at view).

**Tests:** without `detail` the body equals what the page sends today (take a fixture workout and
assert the six fields); with it, `sets` is capped at 20, a per-side or timed set keeps its type,
weights are in the exercise's unit, a workout with no entries yields no `ex`.
**Commit:** `Live: the heartbeat can carry the current exercise and its sets`

### P4.2 `api/presence.js`

**Files:** create `api/presence.js`, `api/test/presence.test.js`; modify `api/server.js`.

Move the presence block (search `/* ---------- live presence (in-memory) ---------- */`) and the
body handling of `POST /api/activity` into the module, behaviour unchanged:

```js
export function createPresence({ now = Date.now } = {}) {}
// -> { set(uid, body), hide(uid), drop(uid), get(uid), sweep() }
export function cleanPresence(body) {}   // bounds every field
```

`cleanPresence` bounds: `name` 60 chars; counters as integers 0 to 999; `ex.id` 80 chars,
`ex.name` 80 chars; `sets` at most 20, each `w` 0 to 2000, `r` 0 to 1000, `unit` one of `kg`,
`lb`, `type` 12 chars, `done` boolean; `restUntil` a time within 30 minutes of now, else dropped.
Unknown keys are dropped. Nothing is ever written to disk.

**Tests:** today's behaviour (expiry at 70 seconds, the 45 second hidden grace, a heartbeat
clearing the hidden mark) with an injected clock; an oversized or malformed body is bounded and
never throws; the existing admin dashboard tests pass untouched.
**Commit:** `Live: presence in its own module, with bounds`

### P4.3 Heartbeat on set tick

**Files:** `views/Workout.jsx`; a test beside `Workout.test.jsx`.

After a set is ticked or its weight or reps change, send one heartbeat, at most one a second
(trailing debounce). The 20 second timer and the leave signals stay as they are.
**Test:** with fake timers, three ticks in 300 ms cause one extra request.
**Commit:** `Live: a ticked set is reported at once`

### P4.4 The live routes

**Files:** modify `api/server.js` (env `TRAINERS_LIVE`, the `trainers` config block),
`api/trainers/dashboard-routes.js`, `api/budgets.js`, `api/openapi.yaml`, `api/share-log.js` if a
new event name needs registering; create `api/test/server-live.test.js`.

- `BUDGETS['trainer.live'] = { max: 1500, windowMs: H }` (a 5 second poll is 720 an hour).
- `GET /api/trainer/live` under `guard('trainer.live', …)`: for each active grant of this trainer,
  `perms.can(user, 'view', { memberId }, 'live')`; where ok and a presence row exists, add
  `{ id, name, ...row }`. Answer `{ members: [...], at: now }`. With `TRAINERS_LIVE` unset: 404
  `live-off`. A look is logged once per day and member, folded like the other trainer looks.
- `GET /api/gym/live?id=` gives a lead the same through the lead row.
- Config: `trainers.live: true` for a member with at least one active grant whose `live` is
  `view`, so the phone knows to send detail; and `trainers.liveBoard: true` for a trainer or lead
  when the env is on.

**Tests:** a member without the grant never appears; with it, appears with detail; after the
member sets `live` to off, the next request leaves them out; a trainer whose gym role was removed
gets nothing; a stranger gets 404; over budget is 429; the env unset is 404.
**Commit:** `Live: a trainer reads who is training, and only who shares it`

### P4.5 The floor board

**Files:** create `frontend/src/lib/floor.js`, `floor.test.js`, `components/FloorBoard.jsx`, its
CSS, `FloorBoard.test.jsx`; modify `App.jsx` (route `/trainer/floor`), `views/Trainer.jsx` (a
link), `lib/trainer-api.js`; every locale.

```js
/** rows from GET /api/trainer/live, now in ms. */
export function floorGroups(rows, now) {}
// -> { working: [...], resting: [...], quiet: [...] }   each row plus { elapsed, restLeft, currentSet, lastDone }
```

`resting`: `restUntil > now`. `quiet`: `now - updatedAt > 3 minutes`. Order inside a group: the
member's name. `currentSet` is the first set not done; `lastDone` the last done one.

The card: name, routine, "Exercise 3 of 6", the exercise's name, "Set 2 of 4", the last set as
"80 kg × 8", a rest countdown, time elapsed. Poll every 5 seconds through a `setInterval` that
runs only while `document.visibilityState === 'visible'`; on 429 back off to the `Retry-After`.
Rows live in component state only.

**Tests:** `floorGroups` with a fixed clock for each group and the ordering; the component with a
mocked API shows a card, moves it to resting, and stops polling when hidden.
**Commit:** `Live: the floor board`

### P4.6 The member's switch

**Files:** `views/MyTrainer.jsx`; test in `MyTrainer.test.jsx`; locales.
A line per trainer: "{0} can see your workout while you train", with the switch that sets `live`
through the existing `POST /api/sharing/update`.
**Commit:** `Live: the member sees and controls it`

### P4.7 Docs

`docs/TRAINERS.md` (the category table, the routes, the budget), `docs/SELF_HOSTING.md` (the env),
`docs/ARCHITECTURE.md` if it draws presence.
**Commit:** `Live: docs`

### Phase 4 done when

- [ ] Two browsers: the member ticks a set and the trainer's card shows it within 6 seconds.
- [ ] The member switches `live` off and the card is gone at the next poll.
- [ ] A server restart empties the board and it refills at the next heartbeats.
- [ ] With `TRAINERS_LIVE` unset nothing changed, and `POST /api/activity` accepts today's body.

**Handover:** the payload as built, the measured delay, whether a 5 second poll felt fast enough.

---

## Phase 5: one native app, and coach events

**Branch:** `wl/phase-5-native`. **Depends on:** phases 1 to 4; P0.2 and P0.3 settled.
**Size:** 3 to 4 weeks.

### P5.1 Join link

**Files:** `views/MobileOnboarding.jsx`, a pure parser `frontend/src/lib/join-link.js` with a test,
`App.jsx` (route `/join`), the lead's gym view (show the link as a QR).

```js
/** 'https://gym.example/#/join?code=GY-ABCDE-FGHIJ' -> { server: 'https://gym.example', code } or null. */
export function parseJoinLink(text) {}
```

https only; the host must be a plain hostname; the code must match the gym code pattern. On the
phone, scanning or pasting it sets the server, starts pairing, loads `/api/brand` and caches it.
On the web, `/join?code=` opens Trainer and gym with the code filled in.
**Tests:** valid links; `http://`, an IP address with a path trick, a missing code, a code of the
wrong shape, and trailing junk are all null.

### P5.2 Our own app identity

Bundle id and application id, display name, icons and store assets in `frontend/ios`,
`frontend/android` and `fastlane/` under the product name from P0.4. Read `docs/MOBILE.md` and
`docs/APP_STORES.md` first. Builds are made and signed on the release machine, never in CI. Ship
only media the P0.2 licence covers; with no licence for larger sizes, ship the 180 px set only.

### P5.3 Coach events

**Files:** create `api/coach-events.js` and its test; modify `api/trainers/dashboard-routes.js`,
`api/server.js` (the answer of `POST /api/activity`), `api/budgets.js`, `api/openapi.yaml`.

- `POST /api/trainer/live/event { memberId, event }` under `guard('trainer.ops', …)`. Needs
  `can(apply, workouts)` and `can(view, live)` for that member, and the member to be present.
- A mailbox per member in memory, at most 20 events, each dropped after 2 minutes. Modelled on
  `api/watch-relay.js`.
- Whitelist, each bounded like `cleanPresence`:
  `{ type: 'set.adjust', exIdx, setIdx, w?, r? }`, `{ type: 'set.add', exIdx }`,
  `{ type: 'note', text }` (200 chars). Each carries `id`, `by: { id, name }`, `at`.
- The member's next `POST /api/activity` answers `{ ok: true, events: [...] }` and empties the
  mailbox, so the phone needs no second poll. Without events the answer is `{ ok: true }` as today.

**Tests:** a trainer without `apply` on workouts is refused; an absent member is 409 `not-training`;
events arrive once and in order; the mailbox caps at 20; an unknown type is 400.

### P5.4 Applying events on the phone

**Files:** create `frontend/src/lib/coach-live.js`, `coach-live.test.js`; modify
`views/Workout.jsx` (read `events` from the heartbeat answer).

```js
/** Applies one event to a draft of S.active. Returns { applied: boolean, line: string|null }. */
export function applyCoachEvent(active, event) {}
```

Read `lib/watch-sync.js` (how watch events are applied) and `lib/coach-adapt.js` (how the AI Coach
changes a workout in progress) and follow their patterns. An event for a set already done is not
applied. A weight is rounded to the active gym's plates as the app does elsewhere. Each applied
event shows a line: "{0} changed set 3 to 82.5 kg". Seen event ids are remembered for the session
so a repeat is ignored.

**Tests:** adjust an open set; refuse a done set; add a set; an out-of-range index; the same id
twice; a `note` changes nothing but returns its line.

### P5.5 Store listings and review

Under our own developer accounts. Record the review notes and anything rejected in HANDOVER.md.

### Phase 5 done when

- [ ] A member installs the one app, scans a gym's QR and sees that gym's brand.
- [ ] The trainer changes the next set's weight on the floor board and it shows on the member's
      screen with the line saying who changed it.

**Handover:** release steps as actually run, store review notes, the event whitelist as built.

---

## Parked

Gym notices, closures and events. Billing and memberships. Chains with several locations. A shared
multi-tenant server. Server-sent events for the floor board. An admin screen for editing the brand
(phase 1 provisions it from files).
