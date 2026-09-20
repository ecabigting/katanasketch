# KatanaSketch — Initial Plan

> **Status:** draft for review · **Date:** 2026-09-18 · **Revised:** 2026-09-20 (Netlify + Atlas; open signup)
> **Goal:** a private "plus.excalidraw.com of our own" with **open signup** —
> anyone can sign in with Google/GitHub and start using it: a workspace of
> persistent boards, per-user view/edit sharing, and a single-editor lock
> (no real-time co-editing).
> Deploy via Netlify Git-connected auto-deploys (no custom domain,
> `*.netlify.app`); data in the existing MongoDB Atlas cluster.
> Recurring cost: $0.
>
> Detail companion: `docs/specs.md` — user stories F1–F14 with UI/backend
> per feature. Where this plan and specs differ on transport, **specs.md
> is authoritative** (Server Actions first; Route Handlers only for auth,
> scenes, health).

## 1. Goal

Small-group whiteboarding web app ("KatanaSketch"):

1. Sign-in page (Google + GitHub only), open signup — anyone can register and start using the app.
2. Sessions that survive restarts.
3. Dashboard listing boards (create, rename, star, duplicate, delete, search).
4. Boards persist to the cloud per user; auto-save while editing.
5. Share a board with named users as viewer or editor; revoke access.
6. **Lock:** only one person edits a board at a time; everyone else gets
   read-only + "edited by X" until the lock releases or expires.

## 2. Non-goals (explicitly out of scope)

- Real-time multiplayer (cursors, live co-editing) — replaced by the lock.
- Custom domain (deferred; app lives at `*.netlify.app` with built-in
  HTTPS via Netlify Git auto-deploy).
- Teams/orgs, billing, AI features, community libraries,
  analytics/Sentry, PWA offline shell, patching the Excalidraw editor.

## 3. Background & established facts

- The reference repo at `/mnt/FilesSSD/src/excalidraw` is a **static SPA
  only** — no backend. Deployed as-is it is a local-only whiteboard; its
  share/collab features call backends we don't control (Firebase,
  `json.excalidraw.com`, `excalidraw-room`). It stays as **reference**;
  KatanaSketch reuses only the editor component (`@excalidraw/excalidraw`).
- Deploy target: **Netlify Git-connected auto-deploys** (build on push,
  no custom domain, free HTTPS on `*.netlify.app`). No server to manage,
  no startup command, no `PORT` handling.
- Database: existing **MongoDB Atlas cluster** (already provisioned; app
  uses one database `katanasketch` plus a separate dev database on the
  same cluster). Real MongoDB — multi-document transactions, `$lookup`,
  and normal index types all work; none of the Cosmos DB for MongoDB
  workarounds apply.
- Better Auth ↔ MongoDB Atlas is the adapter's standard path — no special
  flags required (no `transaction: false`, no `retrywrites=false`, no
  mandatory `createdAt` index workaround).
- Next.js on Netlify uses the official Netlify Next.js runtime (no
  `standalone` output, no zip packaging).
- License note: this repo is GPL; the editor package is MIT, which is
  GPL-compatible to depend on. No action needed.
- License note: this repo is GPL; the editor package is MIT, which is
  GPL-compatible to depend on. No action needed.

## 4. Decisions

| # | Decision |
|---|---|
| 1 | Fresh **Next.js App Router** app in this repo (`katanasketch/`). Standalone repo, npm (kept for simplicity). |
| 2 | Auth: **Better Auth**, Google + GitHub social providers with **open signup**, MongoDB adapter → existing Atlas cluster. No allow-list, no admin role. |
| 3 | Editor: published **`@excalidraw/excalidraw` npm package** (not workspace-linked to the reference repo). |
| 4 | No WebSockets anywhere. Locking over HTTPS + heartbeats. |
| 5 | Netlify Git-connected site (auto-build on push, `*.netlify.app`), official Next.js runtime — no `standalone` output, no zip, no Actions deploy step. |
| 6 | All app data access via the `mongodb` driver; no ODM. Single-doc writes by design (transactions available on Atlas but unused — nothing needs them). |

## 5. Architecture

```text
Browser ──HTTPS──> [ Netlify: CDN + Next.js runtime (Git auto-deploy) ]
                      ├─ pages (server components read Atlas directly):
                      │   `/`, `/board/[id]`, `/signin`
                      ├─ Server Actions (`src/actions/*`): boards CRUD,
                      │   lock, sharing — called directly, no fetch
                      ├─ Route Handlers (only): `/api/auth/*` (Better Auth),
                      │   `/api/boards/[id]/scene` (large payloads + 409/423),
                      │   `/api/health`
                      └─ mongodb driver ──> [ MongoDB Atlas (existing cluster) ]
                        database "katanasketch" (+ separate dev database):
                          user/session/account/verification (Better Auth)
                          boards, scenes, boardInvites
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
        health/route.ts               # liveness probe (deploy smoke check)
        boards/[id]/scene/route.ts    # GET load (re-fetch), PUT save (version CAS)
    actions/
      boards.ts                       # create/rename/star/duplicate/delete + getBoardMetaAction
      lock.ts                         # acquire/heartbeat/release/forceRelease (owner-only force)
      access.ts                       # invite-by-email, claim-on-signin, revoke (no user search)
    components/
      SignInButtons.tsx
      Dashboard.tsx
      BoardEditor.tsx                 # "use client": mounts <Excalidraw>, lock flow, autosave
      AccessDialog.tsx                # share dialog
      LockBanner.tsx
      lib/
        auth.ts                         # Better Auth config (open signup, no gates)
        db.ts                           # Atlas MongoClient singleton (dev/prod safe)
        session.ts                      # requireUser()/getUser()
        boards.ts                       # board/scene/lock/access data functions + validation
        scene.ts                        # serialize/compress/encrypt scene blobs, 16 MB guard
        env.ts                          # validated env access (fail fast on boot)
  tests/                              # vitest: unit + action/handler integration
  public/                             # static assets (fonts only if §13 fallback triggers)
  .env.example
  netlify.toml                        # Netlify build config (Next.js runtime)
  .nvmrc                              # pins Node 22 for Netlify + local dev
```

## 7. Pinned stack (verified 2026-09-18, re-check at scaffold)

**Version policy:** mature majors with a proven Netlify track record; patch
releases are stability releases and always taken; exact versions pinned in
the lockfile at scaffold (lockfile committed); CVEs re-checked at scaffold.

| Package | Version | Why |
|---|---|---|
| `next` | **^15 (latest 15.x at scaffold)** | Mature major, supported by the Netlify Next.js runtime, React 19 supported since 15.0. Next 16 deliberately skipped: newer major, behavioral unknowns not yet verified. |
| `react` / `react-dom` | **^19 (exact patch pinned at scaffold)** | Editor peers require `^17 \|\| ^18 \|\| ^19`; reference repo itself runs React 19. No 19.x novelty needed — take the latest 19.x *patch*, not a new major. |
| `node` (dev + Netlify) | **22** (`.nvmrc` + `NODE_VERSION`) | Active/maintenance LTS with runway past 2027; supported by the Netlify Next.js runtime. (Node 20 EOL'd 2026-04-30; Node 24 skipped per version policy.) |
| `@excalidraw/excalidraw` | **0.18.1** | Same minor as the reference repo (0.18.0); the `.1` is a bugfix-only patch, i.e. the stable choice, not a novelty. |
| `better-auth` | latest v1 at scaffold | v1 is the stable major line (not beta/canary). Verify handler export shape in M1 (see §20.4) |
| Mongo adapter | per current Better Auth docs (`better-auth/adapters/mongodb`) | Standard config — Atlas supports transactions, no special flags |
| `mongodb` driver | v6 | Atlas `mongodb+srv://` connection string |
| `pako` | latest 4.x/5.x at scaffold (mature, unchanged API for years) | Scene compression (same lib the editor uses) |
| `nanoid` | latest 5.x at scaffold (mature, tiny) | Unguessable board IDs + public-link tokens (gatekeeping) |
| `vitest` + `mongodb-memory-server` | latest v1/v3 at scaffold | Unit + action/handler tests without touching Atlas |

## 8. Auth design (Better Auth)

`lib/auth.ts`:

- `database: mongodbAdapter(db)` — standard Better Auth MongoDB adapter
  against Atlas. No `transaction: false` / `retrywrites=false` (those were
  Cosmos-only workarounds; Atlas is real MongoDB).
- `socialProviders: { google: {…}, github: {…} }` (built-in providers).
- `session: { expiresIn: 30 days, updateAge: 1 day }`, cookies default
  (`httpOnly; Secure; SameSite=Lax`).
- **Open signup:** no `databaseHooks` gates — any Google/GitHub account
  creates a user on first sign-in and can start using the app immediately.
  No `role` field, no admin plugin, no `ALLOWED_EMAILS`, no `allowedUsers`.
- **Invite claim:** a post-sign-in step (session creation + lazy
  `ensureInvitesClaimed` on dashboard load) resolves pending `boardInvites`
  for the sign-in email into `boards.access` entries, then deletes the
  claimed invites. There is deliberately **no user directory or user-search
  endpoint** — sharing never looks users up.
- **Recommended indexes** (create once via script or Atlas UI, documented
  in README): `createdAt` on `user`, `account`, `session`, `verification`
  for session cleanup/query performance. These are routine MongoDB
  indexes — not a Cosmos workaround.
- `advanced.database.joins` left at its default — Atlas supports `$lookup`,
  so no constraint either way; the app simply doesn't need joins.

## 9. Data model (Atlas database `katanasketch` + separate dev database)

Better Auth owns `user`, `session`, `account`, `verification`. App owns:

| Collection | Shape |
|---|---|
| `boards` | `{ _id: nanoid (unguessable URL id, never ObjectId), ownerId, title, createdAt, updatedAt, version, starredBy: [userId], lock?: { userId, userName, acquiredAt, expiresAt }, access: [{ userId, role: "view"\|"edit" }], publicToken?: string \| null }` — `publicToken` null = no public link (F15) |
| `scenes` | `{ _id: boardId, version, sceneVersion, updatedAt, payload }` — AES-256-GCM blob (key from `SCENE_KEY`) of pako-compressed `{ elements, files, appState: {zoom, scroll} }`. 16 MB guard (MongoDB document limit) → friendly error. |
| `boardInvites` | `{ _id, boardId, email (lowercased), role: "view"\|"edit", invitedBy, createdAt }` — pending invite-by-email; **not** a user lookup. Claimed into `boards.access` on the invitee's sign-in, then deleted. |

Indexes (normal MongoDB indexes on Atlas): `boards.ownerId`,
`boards.access.userId`, `boards.publicToken`, unique `(boardInvites.boardId,
boardInvites.email)`, plus the §8 `createdAt` set.
Compound/text indexes are available if ever needed; the current queries
don't require them.

## 10. Backend transport (decided: Server Actions first)

Reads happen in Server Components directly via `src/lib/*` (no fetch).
Mutations are Server Actions (`src/actions/*`); Route Handlers exist only
for `/api/auth/[...all]`, `GET`/`PUT /api/boards/[id]/scene`, and
`/api/health`. Full per-feature mapping (which action/handler, guards,
statuses) lives in `docs/specs.md` §0 + F1–F14 — that file is authoritative
here. Shared rules: `requireUser()` session helper, `{ ok, data | error }`
action results, `revalidatePath("/")` after board-list mutations,
`router.refresh()` on board pages (manual-refresh policy, no polling, no
WebSockets). Single-doc writes by design (Atlas supports transactions,
but nothing in this design needs them).

## 11. Lock semantics

- One editor per board. Lock `{ userId, userName, acquiredAt, expiresAt }`
  on the board doc; TTL 8 min; client heartbeat every 60 s;
  `beforeunload`/`visibilitychange` → best-effort release.
- Open flow: `GET meta` → no active lock + can edit → "Edit" acquires.
   Locked by another → editor mounts **read-only** (controlled
   `viewModeEnabled` prop) + `LockBanner` ("Being edited by X") with Retry
   and (owner-only) Force-release.
- Expired locks purge on read — crashed tabs need no manual action.
- Residual double-writer window (TTL expiry + unsaved idle tab) is closed
  by version CAS + `reconcileElements` merge on 409.

## 12. Pages & flows

- **Signed-out (any route)** → `/signin`: provider buttons; sign up and
  sign in are the same flow (first OAuth creates the account).
- **`/` dashboard (server component):** My boards / Shared with me,
  last-edited, lock dot, search; row actions create/rename/star/duplicate/
  delete; empty-state onboarding.
- **`/board/[id]`:** server component checks session + permission
  (`authorizeBoard` — missing *or* unauthorized both render uniform 404;
  signed-out goes to `/signin`) + lock, then renders `BoardEditor` with
  `{ boardId, initialScene, canEdit, lockState }`. Save states: saved /
  saving / offline-queued / conflict.
- **`/share/[token]`:** no auth; resolves `publicToken` (bad token →
  uniform 404), renders forced read-only editor with no account UI.
  Public viewers never touch locks, saves, or the board list.
- **Sharing (invite-by-email, no user search):** `AccessDialog` (dashboard
  row or editor menu): owner enters any email + viewer/editor role → stored
  as a `boardInvites` record. No user lookup happens: the response never
  reveals whether the email is registered. When that email registers and
  signs in, the claim step converts the invite into `boards.access` and the
  board appears under Shared with me. Dialog shows members + pending invites;
  revoke covers both; shows lock holder.
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

## 14. Netlify deployment ($0)

1. Netlify site connected to this repo (Git auto-deploy on push to
   `main`), site `katanasketch…` → `https://<site>.netlify.app`.
2. `netlify.toml`: build command `npm ci && npm run build`, publish
   `.next`, Next.js runtime (official Netlify plugin/runtime — no
   `standalone` output, no zip packaging, no startup command).
3. Node version pinned via `.nvmrc` (Node 22) / `NODE_VERSION` env so
   Netlify builds with the same major as local dev.
4. Environment variables set in the Netlify UI (all server-side; no
   `NEXT_PUBLIC_*`): `BETTER_AUTH_SECRET`,
   `BETTER_AUTH_URL=https://<site>.netlify.app`, `GOOGLE_*`,
   `GITHUB_*`, `MONGODB_URI` (Atlas `mongodb+srv://…`), `MONGODB_DB`
   (e.g. `katanasketch`), `SCENE_KEY`, plus
   `NEXT_TELEMETRY_DISABLED=1`. Preview deploys share the dev database
   or a scratch database — never prod.
5. HTTPS is built-in on `*.netlify.app`. No `PORT` handling, no
   slots/health-check wiring — `/api/health` is a deploy smoke check
   only (open it after deploy, expect `{ ok: true }`).
6. OAuth registrations: Google Cloud OAuth client + GitHub OAuth App with
   prod redirect `https://<site>.netlify.app/api/auth/callback/...`
   and localhost dev counterparts.
7. Accepted free-tier caveats: serverless/function cold starts after
   idle, Netlify free execution limits (fine for a small group).

## 15. Local development

`npm run dev` (Next, :3000) against a **separate dev database on the same
Atlas cluster** (`MONGODB_DB` e.g. `katanasketch-dev`; usage is tiny) or
`mongodb-memory-server` for unit tests. `.env.local` (gitignored) mirrors
`.env.example`. Dev OAuth apps use `http://localhost:3000` callbacks.
Pre-deployment check: `npm run build` passes locally before pushing
(Netlify builds on push).

## 16. Testing & verification

- **Vitest:** ACL matrix per action,
  lock acquire/contend/expire/heartbeat/force-release (+ same-user
  re-acquire), scene CAS 409 + merge path, 16 MB guard, session issuance
  with mocked providers. Full matrix in `docs/specs.md`.
- **DB in tests:** `mongodb-memory-server` (no Atlas burn); one smoke run
  against the Atlas dev/scratch database.
- **Gates (no "done" without green):** `tsc --noEmit`, `eslint`,
  `vitest run`, `next build`, Netlify preview deploy.

## 17. Milestones & exit criteria

- **M1 — Webapp + auth + deploy:** Next scaffold, Better Auth (Google+
  GitHub, open signup) vs Atlas dev database, sign-in page, empty dashboard,
  Netlify Git auto-deploy, health check. **Exit:** sign in on the
  live URL; dashboard loads; sessions survive restarts. *(Proves the two
  riskiest integrations first.)*
- **M2 — Boards actions + scene endpoints + dashboard:** CRUD/duplicate/
  star/search via Server Actions, scene GET/PUT handlers,
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
| Better Auth↔Atlas quirks | M1 proves it against the Atlas dev database; adapter `debugLogs`; fallback: hand-rolled session table (§9 already shaped for it) |
| Open-signup abuse (spam accounts) | Out of scope for this group-sized app; invites are email records claimed on registration (no user search); rate limits on scene/invite/token endpoints; revisit CAPTCHA/rate limits only if abused |
| Netlify function cold start / execution limits | Small group; accept occasional cold start; autosave debounce + retry covers transient slowness |
| Lock TTL race → conflicting saves | Version CAS + reconcile merge; 8-min TTL + 60-s heartbeat |
| Scene + images > 16 MB (MongoDB doc limit) | Client estimate + server guard + clear error; editor downscales pasted images |
| React 19 + Next 15 + editor CSS edge cases | Pinned mature versions; visual smoke test in M3 |
| Atlas free/cluster limits | Small group, tiny usage; dev + prod separated by database on the same (existing) cluster |
| pnpm friction | Using npm — sidesteps hoisting issues entirely |

## 19. Environment variables

`.env.example` documents: `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`,
`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GITHUB_CLIENT_ID`,
`GITHUB_CLIENT_SECRET`, `MONGODB_URI` (Atlas `mongodb+srv://…`),
`MONGODB_DB` (`katanasketch`, dev uses a separate name), `SCENE_KEY`
(32-byte, generated).
No `VITE_APP_*` anywhere.

## 20. Verification log (line-by-line check, 2026-09-18)

> Historical record from the Azure/Cosmos design. Items mentioning Azure,
> standalone/ZipDeploy, or Cosmos workarounds are **superseded by §21**
> (2026-09-20 Netlify + Atlas revision) and kept here for audit trail only.

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

## 21. Revision log — Netlify + Atlas (2026-09-20)

Supersedes all Azure/Cosmos items in §20. Decisions from deploy-strategy
review: Netlify Git auto-deploy with no custom domain, existing Atlas
cluster, Atlas dev database for local/dev, keep Next 15 + Netlify Next.js
runtime (no `standalone`, no zip, no Actions deploy).

1. Next.js on Netlify — official Netlify Next.js runtime supports Next 15;
   no `output: "standalone"`, no `public/` + `.next/static` copy step, no
   `server.js` startup command, no `PORT` handling. Build config lives in
   `netlify.toml` + `.nvmrc` (Node 22). **In plan (§4.5, §14).**
2. Node version — Node 22 pinned via `.nvmrc` / `NODE_VERSION` so Netlify
   builds match local dev. Node 20 EOL rationale from §20 carries over;
   Node 24 still skipped per stability policy. **In plan (§7, §14.3).**
3. Better Auth ↔ Atlas — standard `mongodbAdapter(db)` path, no
   `transaction: false`, no `retrywrites=false`. Transactions and
   `$lookup` work on Atlas; the app keeps single-doc writes by design
   (simplicity), not by constraint. **In plan (§3–§4, §8).**
4. Indexes — `createdAt` on auth collections is a routine performance
   index on Atlas, not a Cosmos unindexed-sort workaround. Compound/text
   indexes available if needed; current queries need single-field only
   (`boards.ownerId`, `boards.access.userId`, `boards.publicToken`).
   **In plan (§8–§9).**
5. 16 MB guard stays — it is the MongoDB document limit, not a
   Cosmos quirk. **In plan (§9) and specs (F9).**
6. Health check — `/api/health` is a post-deploy smoke check only
   (open it, expect `{ ok: true }`); Netlify has no App-Service-style
   health-check path wiring and preview deploys must never point at
   prod. **In plan (§14.5) and specs (F14).**
7. OAuth callbacks — prod redirect
   `https://<site>.netlify.app/api/auth/callback/...` plus localhost dev
   counterparts; `BETTER_AUTH_URL` set in the Netlify UI per environment.
   **In plan (§14.6–§14.7).**

## 22. Revision log — open signup (2026-09-20)

Invite-only removed. Anyone with Google/GitHub can register and start using
the app; there is no allow-list, no `ALLOWED_EMAILS`, no `admin` role, and
no `allowedUsers` collection. Board sharing is invite-by-email (see §23 —
no user search; access granted when the invited email registers/signs in).
`forceReleaseLock` is owner-only (the former admin-with-access path is
gone with the role). F12 (allow-list admin) is removed in specs/features;
F-numbers are otherwise unchanged.

## 23. Revision log — invite-by-email sharing (2026-09-20)

No user search or user directory anywhere. Owners invite by email; the
invite is stored as a `boardInvites` record (`boardId + email + role`) and
the response never reveals whether that email is registered (no
enumeration oracle). Claim: on the invitee's sign-in (session creation)
and lazily via `ensureInvitesClaimed` on dashboard load, matching invites
are converted into `boards.access` entries and deleted. Pending invites are
visible only to the board owner in `AccessDialog` (members + pending) and
confer no access until claimed. Revoke covers both granted access and
pending invites.
