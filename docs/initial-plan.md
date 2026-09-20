# KatanaSketch — Initial Plan

> **Status:** draft for review · **Date:** 2026-09-18
> **Goal:** a private, invite-only "plus.excalidraw.com of our own" —
> sign in with Google/GitHub, a workspace of persistent boards, per-user
> view/edit sharing, and a single-editor lock (no real-time co-editing).
> Deploy to one free-tier (F1) Azure App Service; data in the existing
> provisioned-throughput free-tier Cosmos DB for MongoDB. Recurring cost: $0.
>
> Detail companion: `docs/specs.md` — user stories F1–F14 with UI/backend
> per feature. Where this plan and specs differ on transport, **specs.md
> is authoritative** (Server Actions first; Route Handlers only for auth,
> scenes, health).

## 1. Goal

Small-group whiteboarding web app ("KatanaSketch"):

1. Sign-in page (Google + GitHub only), invite-only.
2. Sessions that survive restarts.
3. Dashboard listing boards (create, rename, star, duplicate, delete, search).
4. Boards persist to the cloud per user; auto-save while editing.
5. Share a board with named users as viewer or editor; revoke access.
6. **Lock:** only one person edits a board at a time; everyone else gets
   read-only + "edited by X" until the lock releases or expires.

## 2. Non-goals (explicitly out of scope)

- Real-time multiplayer (cursors, live co-editing) — replaced by the lock.
- Custom domain (F1 App Service does not support it; app lives at
  `*.azurewebsites.net` with built-in HTTPS).
- Open signup, teams/orgs, billing, AI features, community libraries,
  analytics/Sentry, PWA offline shell, patching the Excalidraw editor.

## 3. Background & established facts

- The reference repo at `/mnt/FilesSSD/src/excalidraw` is a **static SPA
  only** — no backend. Deployed as-is it is a local-only whiteboard; its
  share/collab features call backends we don't control (Firebase,
  `json.excalidraw.com`, `excalidraw-room`). It stays as **reference**;
  KatanaSketch reuses only the editor component (`@excalidraw/excalidraw`).
- Verified Azure constraints: F1 = no custom domains, 5 concurrent
  WebSockets (unused — we use none), 60 CPU-min/day, 1 GB RAM, sleeps on
  idle, free HTTPS on `*.azurewebsites.net`.
- Verified Cosmos DB free tier: lifetime 1000 RU/s + 25 GB on
  **provisioned-throughput** accounts (confirmed: ours is provisioned).
  Mongo API max document 16 MB.
- Verified Better Auth ↔ Cosmos DB works with three config requirements
  (details in §8).
- Verified Next.js standalone → Azure App Service ZipDeploy pattern
  (details in §14).
- License note: this repo is GPL; the editor package is MIT, which is
  GPL-compatible to depend on. No action needed.

## 4. Decisions

| # | Decision |
|---|---|
| 1 | Fresh **Next.js App Router** app in this repo (`katanasketch/`). Standalone repo, npm (not pnpm — avoids hoisting issues on Azure). |
| 2 | Auth: **Better Auth**, Google + GitHub social providers, MongoDB adapter → existing Cosmos DB. |
| 3 | Editor: published **`@excalidraw/excalidraw` npm package** (not workspace-linked to the reference repo). |
| 4 | No WebSockets anywhere. Locking over HTTPS + heartbeats. |
| 5 | Single F1 Linux App Service, Next `standalone` output, GitHub Actions ZipDeploy. |
| 6 | All app data access via the `mongodb` driver; no ODM. No multi-doc transactions anywhere. |

## 5. Architecture

```text
Browser ──HTTPS──> [ F1 App Service: node server.js (Next standalone) ]
                      ├─ pages (server components read Cosmos directly):
                      │   `/`, `/board/[id]`, `/signin`
                      ├─ Server Actions (`src/actions/*`): boards CRUD,
                      │   lock, sharing, invites — called directly, no fetch
                      ├─ Route Handlers (only): `/api/auth/*` (Better Auth),
                      │   `/api/boards/[id]/scene` (large payloads + 409/423),
                      │   `/api/health`
                      └─ mongodb driver ──> [ Cosmos DB for MongoDB ]
                        shared-throughput DB "katanasketch":
                          user/session/account/verification (Better Auth)
                          boards, scenes, allowedUsers
OAuth ──> Google / GitHub (two app registrations,
          callbacks …/api/auth/callback/google|github)
```

One origin → no CORS. `NEXT_PUBLIC_*` avoided entirely (all config
server-side; public vars bake in at build time and complicate deploys).

## 6. Repository layout

```text
katanasketch/
  docs/initial-plan.md
  docs/specs.md                   # user stories + UI/backend per feature (transport authority)
  src/
    app/
      page.tsx                        # / dashboard (server component)
      signin/page.tsx                 # /signin
      board/[id]/page.tsx             # /board/:id (server: meta+permission; renders editor)
      board/[id]/not-found.tsx        # 404 boundary — uniform for missing AND no-access
      share/[token]/page.tsx          # /share/:token — public read-only, no auth (F15)
      api/
        auth/[...all]/route.ts        # Better Auth handler (mandatory Route Handler)
        health/route.ts               # liveness probe (smoke check; F1 health-check is metrics-only)
        boards/[id]/scene/route.ts    # GET load (re-fetch), PUT save (version CAS)
    actions/
      boards.ts                       # create/rename/star/duplicate/delete + getBoardMetaAction
      lock.ts                         # acquire/heartbeat/release/forceRelease
      access.ts                       # setAccess/revokeAccess
      invites.ts                      # allow-list management (admin)
    components/
      SignInButtons.tsx
      Dashboard.tsx
      BoardEditor.tsx                 # "use client": mounts <Excalidraw>, lock flow, autosave
      AccessDialog.tsx                # share dialog
      LockBanner.tsx
    lib/
      auth.ts                         # Better Auth config (+ role field, dual invite gates)
      db.ts                           # Cosmos MongoClient singleton (dev/prod safe)
      session.ts                      # requireUser()/getUser()
      boards.ts                       # board/scene/lock/access data functions + validation
      scene.ts                        # serialize/compress/encrypt scene blobs, 16 MB guard
      env.ts                          # validated env access (fail fast on boot)
  tests/                              # vitest: unit + action/handler integration
  public/                             # static assets (fonts only if §13 fallback triggers)
  .env.example
  next.config.ts                      # output: "standalone"
  .github/workflows/deploy.yml
```

## 7. Pinned stack (verified 2026-09-18, re-check at scaffold)

**Version policy:** mature majors with a proven Azure track record; patch
releases are stability releases and always taken; exact versions pinned in
the lockfile at scaffold (lockfile committed); CVEs re-checked at scaffold.

| Package | Version | Why |
|---|---|---|
| `next` | **^15 (latest 15.x at scaffold)** | Mature major, largest guide/answer base for Azure standalone deploys, React 19 supported since 15.0. Next 16 deliberately skipped: newer major, behavioral unknowns not yet verified. |
| `react` / `react-dom` | **^19 (exact patch pinned at scaffold)** | Editor peers require `^17 \|\| ^18 \|\| ^19`; reference repo itself runs React 19. No 19.x novelty needed — take the latest 19.x *patch*, not a new major. |
| `node` (dev + Azure) | **22 LTS** (`NODE\|22-lts`) | Maintenance LTS with runway past 2027; every Azure standalone guide targets it. (Node 20 EOL'd 2026-04-30; Node 24 skipped per version policy.) |
| `@excalidraw/excalidraw` | **0.18.1** | Same minor as the reference repo (0.18.0); the `.1` is a bugfix-only patch, i.e. the stable choice, not a novelty. |
| `better-auth` | latest v1 at scaffold | v1 is the stable major line (not beta/canary). Verify handler export shape in M1 (see §20.4) |
| Mongo adapter | per current Better Auth docs (`better-auth/adapters/mongodb`) | `transaction: false` (see §8) |
| `mongodb` driver | v6 | Cosmos connection string, `retrywrites=false` |
| `pako` | latest 4.x/5.x at scaffold (mature, unchanged API for years) | Scene compression (same lib the editor uses) |
| `nanoid` | latest 5.x at scaffold (mature, tiny) | Unguessable board IDs + public-link tokens (gatekeeping) |
| `vitest` + `mongodb-memory-server` | latest v1/v3 at scaffold | Unit + action/handler tests without burning RU |

## 8. Auth design (Better Auth)

`lib/auth.ts`:

- `database: mongodbAdapter(db, { transaction: false })` — Cosmos Mongo
  API has no multi-document transactions; the adapter enables them whenever
  a `client` is passed, so this flag is **mandatory**.
- `socialProviders: { google: {…}, github: {…} }` (built-in providers).
- `session: { expiresIn: 30 days, updateAge: 1 day }`, cookies default
  (`httpOnly; Secure; SameSite=Lax`).
- **Invite-only:** `databaseHooks.user.create.before` hook — allow only if
  `email ∈ allowedUsers` or `email ∈ ALLOWED_EMAILS` bootstrap list, else
  return `false` (aborts creation; verified supported). Rejected sign-ins
  redirect to `/signin?error=not-invited`.
- Bootstrap: `ALLOWED_EMAILS` env (owner). Admin UI on dashboard manages
  `allowedUsers` (`{ _id: lowercased email, addedBy, createdAt }`).
- **One-time indexes** (script or portal, documented in README):
  `createdAt` on `user`, `account`, `session`, `verification` — Cosmos
  rejects unindexed sorts (observed failure mode for this exact adapter).
- Leave `advanced.database.joins` off (default) — Cosmos lacks `$lookup`.

## 9. Data model (one shared-throughput Cosmos DB, `katanasketch`)

Better Auth owns `user`, `session`, `account`, `verification`. App owns:

| Collection | Shape |
|---|---|
| `boards` | `{ _id: nanoid (unguessable URL id, never ObjectId), ownerId, title, createdAt, updatedAt, version, starredBy: [userId], lock?: { userId, userName, acquiredAt, expiresAt }, access: [{ userId, role: "view"\|"edit" }], publicToken?: string \| null }` — `publicToken` null = no public link (F15) |
| `scenes` | `{ _id: boardId, version, sceneVersion, updatedAt, payload }` — AES-256-GCM blob (key from `SCENE_KEY`) of pako-compressed `{ elements, files, appState: {zoom, scroll} }`. 16 MB guard → friendly error. |
| `allowedUsers` | `{ _id: email, addedBy, createdAt }` |

Indexes (all single-field, Cosmos-safe): `boards.ownerId`,
`boards.access.userId`, plus §8 set. (Compound indexes exist on Cosmos but
are unnecessary here; multi-field sorts in `findOneAndUpdate` are
unsupported — nothing in this design needs them.)

## 10. Backend transport (decided: Server Actions first)

Reads happen in Server Components directly via `src/lib/*` (no fetch).
Mutations are Server Actions (`src/actions/*`); Route Handlers exist only
for `/api/auth/[...all]`, `GET`/`PUT /api/boards/[id]/scene`, and
`/api/health`. Full per-feature mapping (which action/handler, guards,
statuses) lives in `docs/specs.md` §0 + F1–F14 — that file is authoritative
here. Shared rules: `requireUser()` session helper, `{ ok, data | error }`
action results, `revalidatePath("/")` after board-list mutations,
`router.refresh()` on board pages (manual-refresh policy, no polling, no
WebSockets). Single-doc writes only, no transactions.

## 11. Lock semantics

- One editor per board. Lock `{ userId, userName, acquiredAt, expiresAt }`
  on the board doc; TTL 8 min; client heartbeat every 60 s;
  `beforeunload`/`visibilitychange` → best-effort release.
- Open flow: `GET meta` → no active lock + can edit → "Edit" acquires.
  Locked by another → editor mounts **read-only** (controlled
  `viewModeEnabled` prop) + `LockBanner` ("Being edited by X") with Retry
  and (owner, or admin with access) Force-release.
- Expired locks purge on read — crashed tabs need no admin action.
- Residual double-writer window (TTL expiry + unsaved idle tab) is closed
  by version CAS + `reconcileElements` merge on 409.

## 12. Pages & flows

- **Signed-out (any route)** → `/signin`: provider buttons;
  `?error=not-invited` explains invite-only.
- **`/` dashboard (server component):** My boards / Shared with me,
  last-edited, lock dot, search; row actions create/rename/star/duplicate/
  delete; admin allow-list section; empty-state onboarding.
- **`/board/[id]`:** server component checks session + permission
  (`authorizeBoard` — missing *or* unauthorized both render uniform 404;
  signed-out goes to `/signin`) + lock, then renders `BoardEditor` with
  `{ boardId, initialScene, canEdit, lockState }`. Save states: saved /
  saving / offline-queued / conflict.
- **`/share/[token]`:** no auth; resolves `publicToken` (bad token →
  uniform 404), renders forced read-only editor with no account UI.
  Public viewers never touch locks, saves, or the board list.
- **Sharing:** `AccessDialog` (dashboard row or editor menu): email must be
  an allow-listed user; pick viewer/editor; revoke; shows lock holder.
- **Errors:** uniform 404 (missing and no-access alike), scene-too-large,
  sync-failed banner with retry, expired session → re-sign-in preserving
  the board URL.

## 13. Editor integration (ported from reference repo)

- `BoardEditor.tsx` (`"use client"`):
  `import { Excalidraw } from "@excalidraw/excalidraw"` +
  `@excalidraw/excalidraw/index.css`. Keep package menus, welcome screen,
  export/import, locale/theme wiring; nothing forked in the editor.
- `initialData` = promise resolving to `{ elements, appState, files }`
  from `GET scene` (cloud-sourced `initializeScene` equivalent; restore via
  the package's restore semantics).
- `onChange` → 300 ms debounced `PUT scene` with `baseVersion`; file blobs
  (images) travel embedded base64 in the payload — no file endpoints.
- Read-only via the controlled `viewModeEnabled` prop (a hook the
  reference app never passes).
- **Fonts:** Next bundles woff2 referenced by the package CSS automatically;
  no copy step by default. Fallback only if font 404s appear: copy fonts to
  `public/` + set `window.EXCALIDRAW_ASSET_PATH` (documented in the package
  README; the reference `with-nextjs` example does the copy pre-emptively).
- Old browser localStorage drafts from excalidraw.com are **not** migrated.

## 14. Azure deployment (F1, $0)

1. App Service (Linux, F1 Free), runtime **`NODE|22-lts`**, site
   `katanasketch…` → `https://<app>.azurewebsites.net`.
2. `next.config.ts`: `output: "standalone"`.
3. GitHub Actions `deploy.yml` (build on `ubuntu-latest`, Node 22):
   `npm ci` → `next build` → `cp -r public .next/standalone/`
   → `cp -r .next/static .next/standalone/.next/` → zip `.next/standalone`
   → `azure/webapps-deploy` (OIDC federated credentials; **no `slot-name`**
   — F1 has no slots). Startup command: `node server.js` (portal setting).
4. App Settings: `SCM_DO_BUILD_DURING_DEPLOYMENT=false`,
   `NEXT_TELEMETRY_DISABLED=1`, `NODE_ENV=production`,
   `WEBSITE_NODE_DEFAULT_VERSION="~22"` + §8 secrets (`BETTER_AUTH_SECRET`,
   `BETTER_AUTH_URL=https://<app>.azurewebsites.net`, `GOOGLE_*`,
   `GITHUB_*`, `MONGODB_URI`, `SCENE_KEY`, `ALLOWED_EMAILS`).
5. HTTPS-only ON, minimum TLS 1.2. Do NOT override `PORT` (standalone
   server respects Azure's automatically).
6. OAuth registrations: Google Cloud OAuth client + GitHub OAuth App with
   prod redirect `https://<app>.azurewebsites.net/api/auth/callback/...`
   and localhost dev counterparts.
7. Accepted F1 caveats: cold starts after idle, 60 CPU-min/day (builds run
   on GitHub, not on the app).

## 15. Local development

`npm run dev` (Next, :3000) against a **separate dev database** in the same
free-tier Cosmos account (shared RU; usage is tiny) or
`mongodb-memory-server` for unit tests. `.env.local` (gitignored) mirrors
`.env.example`. Dev OAuth apps use `http://localhost:3000` callbacks.
Pre-deployment check: `node .next/standalone/server.js` locally after
copying static assets (§14.3).

## 16. Testing & verification

- **Vitest:** allow-list hooks, role assignment, ACL matrix per action,
  lock acquire/contend/expire/heartbeat/force-release (+ same-user
  re-acquire), scene CAS 409 + merge path, 16 MB guard, session issuance
  with mocked providers. Full matrix in `docs/specs.md`.
- **DB in tests:** `mongodb-memory-server` (zero RU burn); one CI smoke run
  against a scratch Cosmos database.
- **Gates (no "done" without green):** `tsc --noEmit`, `eslint`,
  `vitest run`, `next build`, standalone smoke, deploy.

## 17. Milestones & exit criteria

- **M1 — Webapp + auth + deploy:** Next scaffold, Better Auth (Google+
  GitHub) vs real Cosmos, allow-list hook + sign-in page, empty dashboard,
  standalone build, Actions → F1, health check. **Exit:** sign in on the
  live URL; dashboard loads; sessions survive restarts. *(Proves the two
  riskiest integrations first.)*
- **M2 — Boards actions + scene endpoints + dashboard:** CRUD/duplicate/
  star/search via Server Actions, scene GET/PUT handlers, invites admin UI,
  indexes, action tests green.
- **M3 — Editor + save/load + lock:** `BoardEditor`, scene endpoints, lock
  flow + read-only + banner, autosave + offline queue + conflict merge,
  fonts check.
- **M4 — Sharing + public links + hardening:** access dialog + endpoints,
  public view links (F15), force-release, uniform-404 error paths, rate
  limits, size guards, README/docs, full verification.

## 18. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Better Auth↔Cosmos quirks beyond the three known flags | M1 proves it against real Cosmos; adapter `debugLogs`; fallback: hand-rolled session table (§9 already shaped for it) |
| Rejected-OAuth UX differs by Better Auth version | M1 spike asserts the `?error=not-invited` path end-to-end |
| Next standalone cold start / 1 GB F1 | Small group; ~50 MB artifact; accept occasional cold start |
| Lock TTL race → conflicting saves | Version CAS + reconcile merge; 8-min TTL + 60-s heartbeat |
| Scene + images > 16 MB | Client estimate + server guard + clear error; editor downscales pasted images |
| React 19 + Next 15 + editor CSS edge cases | Pinned mature versions; visual smoke test in M3 |
| F1 60 CPU-min/day | Builds on GitHub Actions; app serves traffic only |
| pnpm/Azure friction | Using npm — sidesteps hoisting issues entirely |

## 19. Environment variables

`.env.example` documents: `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`,
`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GITHUB_CLIENT_ID`,
`GITHUB_CLIENT_SECRET`, `MONGODB_URI` (Cosmos; includes
`retrywrites=false`), `SCENE_KEY` (32-byte, generated), `ALLOWED_EMAILS`.
No `VITE_APP_*` anywhere.

## 20. Verification log (line-by-line check, 2026-09-18)

Each load-bearing claim was checked before writing. Issues found are
marked **[ISSUE]** with their resolution.

1. Next.js version — **checked** nextjs.org: latest stable 16.3.4 (Sept
   2026); React 19.3 released 2026-09-09. **[DECISION — stability policy]**
   plan targets **Next 15.x**, not 16: 16 is a newer major with behavioral
   unknowns (caching/sub-API changes) not verified for this stack, while 15
   has the largest proven Azure standalone-deploy track record and supports
   React 19 since 15.0. Mature majors + patches; no new majors.
2. Next 15 standalone on Azure — **checked** (multiple 2026 standalone
   guides, all covering v15): `output:"standalone"`, copy `public/` +
   `.next/static`, zip `.next/standalone`, startup `node server.js`,
   `SCM_DO_BUILD_DURING_DEPLOYMENT=false`. **Solid.**
3. Node runtime — **checked** MS Learn + App Service blog: Node 24 LTS live
   on Linux (`NODE|24-lts` / `~24`); Node 20 EOL'd 2026-04-30.
   **[DECISION — stability policy]** plan targets **Node 22 LTS**
   (`NODE|22-lts` / `~22`, maintenance LTS with runway past 2027): every
   Azure standalone guide targets it; newest LTS gains nothing here.
4. Editor package — **checked** npm: `0.18.1` (Apr 2026, a bugfix-only
   patch on the 0.18 minor the reference repo runs — the stable choice);
   **checked** reference `packages/excalidraw/package.json:76-79`: peers
   include React 19. **Solid.**
5. `reconcileElements` import — **checked** reference
   `packages/excalidraw/index.tsx:422`: exported from package root. **Solid.**
6. Better Auth `transaction:false` — **checked** adapter source
   (`mongodb-adapter.ts`): required when the server lacks transactions.
   **Solid** (Cosmos Mongo API has none).
7. Better Auth `databaseHooks.user.create.before → false` aborts creation —
   **checked** `with-hooks.ts` source (`if (result === false) return null`).
   **Solid**, but the exact end-user redirect for social sign-in rejection
   is version-dependent → M1 spike item (§18).
8. Cosmos `createdAt` index requirement — **checked** Better Auth issue
   #5454 (Cosmos users; fixed by indexing `createdAt` on auth
   collections). **In plan (§8).**
9. Cosmos index capabilities — **checked** MS Learn: compound ≤8 fields
   (not on arrays) supported; single-field ≤500; TTL ✓; unique ✓; text ✗;
   case-insensitive ✗; `findOneAndUpdate` multi-field sort ✗.
   **[NOTE]** adapter does case-insensitive matching via `$regex` (query-
   level, supported) not index collation — no conflict. Multi-field-sort
   paths, if any in the adapter, would 400 on Cosmos → covered by the M1
   real-Cosmos auth run.
10. Deployment slots on F1 — F1 has none. **[ISSUE]** generic guides pass
    `slot-name: Production` → plan omits it.
11. Package manager — 2026 pnpm guide documents hoisting workarounds needed
    for Azure. **[DECISION]** plan specifies npm to avoid the issue class.
12. Fonts — **checked** reference `index.tsx:36` + `fonts.css`: woff2 are
    CSS-relative imports, which Next bundles automatically; the reference
    example's `copy:assets` step exists for `EXCALIDRAW_ASSET_PATH`
    self-hosting. **[DECISION]** no copy step by default; fallback
    documented in §13.
13. `PORT` handling — **checked** (multiple guides): standalone respects
    Azure's `PORT`; do not override. **In plan (§14.5).**
14. Build arch — ubuntu-latest (x64) matches App Service Linux (x64), so
    any native modules (e.g. `sharp` via `next/image`) build correctly.
    **[NOTE]** we avoid `next/image` optimization; plain `<img>`/CSS only.
15. Cosmos connection string — standard Mongo API format
    (`…mongo.cosmos.azure.com:10255/?ssl=true&retrywrites=false…`).
    `retrywrites=false` is **mandatory** on Cosmos. **In plan (§19).**
16. Stability-policy revision (post-draft review) — first draft targeted
    newest majors (Next 16, Node 24). Revised per project policy to mature
    majors (Next 15, Node 22): newest-majors bring unverified behavioral
    surface; the conservative set is fully covered by the guides checked in
    items 2–3 above. Patch releases are always taken; exact versions pinned
    via lockfile at scaffold.
