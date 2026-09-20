# KatanaSketch — Feature Specs

> Companion to `initial-plan.md`. Every feature is a user story with
> acceptance criteria, then the required code split into **UI** and
> **Backend**. Transport rule (§0) applies to all features.

## 0. Transport & code conventions (applies everywhere)

- **Server Components read Atlas directly** via `src/lib/*.ts` data
  functions. No `fetch`, no API round-trip for page loads.
- **Mutations use Server Actions** (`src/actions/*.ts`, `"use server"`),
  called directly from client components. No `fetch` except where noted.
- **Route Handlers exist only for:** `/api/auth/[...all]` (Better Auth —
  OAuth requires real HTTP callbacks, cannot be an action),
  `GET`/`PUT /api/boards/[id]/scene` (multi-MB payloads + real 409/423/413
  statuses), `/api/health` (deploy smoke check).
- **Session in actions/components:**
  `const session = await auth.api.getSession({ headers: await headers() })`
  via `requireUser()` in `src/lib/session.ts` (throws coded error when
  absent; actions map it to `{ ok: false, error: { code:
  "UNAUTHENTICATED" } }`).
- **Action result shape (all actions):**
  `{ ok: true, data } | { ok: false, error: { code, message } }`.
  Codes: `UNAUTHENTICATED, FORBIDDEN, NOT_FOUND, LOCKED,
  CONFLICT, TOO_LARGE, VALIDATION`.
- After board-list mutations: `revalidatePath("/")`. After lock changes on
  a board page: `router.refresh()` from the client (manual-refresh policy —
  no polling, no WebSockets).
- Shared data layer `src/lib/boards.ts` holds all Mongo queries so actions,
  pages, and tests share one implementation. Single-doc writes by design
  (Atlas supports transactions, but nothing here needs them).
- **Gatekeeping (cross-cutting):** board `_id`s are random strings
  (`nanoid`, 21 chars) — never ObjectIds — so URLs are unguessable;
  `authorizeBoard(boardId, userId, minRole)` in `src/lib/boards.ts` is the
  single enforcement point every page, action, and scene handler calls
  first; authenticated-but-unauthorized access renders `notFound()`
  (uniform 404 — missing and forbidden are indistinguishable);
  unauthenticated access still redirects to `/signin` (authentication ≠
  authorization). There is no admin role — `forceReleaseLock` (F7) is
  owner-only.

---

## F1 — Sign in with Google / GitHub (open signup)

**Story.** As any visitor, I want to sign in with my Google or GitHub
account so that an account is created on first sign-in and I land in my
workspace immediately — no invite needed.

**Acceptance.**
- `/signin` shows two buttons; each completes OAuth and lands on `/`
  (or the `?next=` target when present).
- First sign-in with any Google/GitHub account → user created, in.
- Signed-in user visiting `/signin` redirects to `/`.
- Sign up and sign in are the same flow (asserted end-to-end in M1 against
  the Atlas dev database).

**UI.**
- `src/app/signin/page.tsx` (server): checks session, redirects if present,
  renders `SignInButtons` with `next` search param.
- `src/components/SignInButtons.tsx` (client): two buttons calling
  `authClient.signIn.social({ provider: "google" | "github",
  callbackURL: "/" })`.

**Backend.**
- `src/lib/auth.ts`: Better Auth config — `mongodbAdapter(db)` (standard
  Atlas path, no special flags), `socialProviders.google/github` (env ids +
  secrets), 30-day sessions. No `databaseHooks` gates, no `role` field, no
  admin plugin, no `ALLOWED_EMAILS`.
- `src/app/api/auth/[...all]/route.ts`: `GET`/`POST` via Better Auth
  handler (mandatory Route Handler).
- `src/lib/db.ts`: `MongoClient` singleton (dev HMR-safe global).
- Data: `user`, `account`, `session`, `verification` (Better Auth-owned).
  Recommended `createdAt` indexes on the four auth collections (routine
  Atlas performance indexes).
- GitHub private emails: Better Auth's github provider fetches
  `/user/emails` automatically — no extra code.

**Tests.** First sign-in creates a user for any Google/GitHub account;
repeat sign-in reuses the row; `getSession` round-trip (mocked providers
for unit, real OAuth only in manual M1 check).

## F2 — Sessions & auth gating

**Story.** As a signed-in user, I want to stay signed in across restarts
and be sent back to sign-in (remembering my board URL) when the session
expires.

**Acceptance.** Restarting the server keeps sessions; expired session on a
protected page → `/signin?next=/board/<id>`; after sign-in, return to `next`.

**UI.** `src/app/signin/page.tsx` honors `next` param; middleware-free
gating done per-page (server components call `requireUser()`).

**Backend.** Better Auth database sessions (Atlas `session` collection);
`src/lib/session.ts`: `requireUser()` + `getUser()` helpers used by every
page and action.

**Tests.** Expired-session redirect preserves `next`; `requireUser` throws
`UNAUTHENTICATED` without cookie.

## F3 — Dashboard: view my boards

**Story.** As a user, I want a workspace listing boards I own and boards
shared with me (title, last-edited, my role, lock presence, stars) with
search, so I can find and open work.

**Acceptance.** `/` shows two sections (Mine / Shared with me), lock dot
on locked boards, working search filter, empty states, manual Refresh
button; header user menu shows name + Sign out (→ `/signin`);
signed-out users go to `/signin`.

**UI.**
- `src/app/page.tsx` (server): `requireUser()` → `listBoards(userId)` →
  renders `Dashboard`.
- `src/components/Dashboard.tsx` (client): sections, search input (local
  filter), lock dots, Refresh button (`router.refresh()`), user menu with
  Sign out (`authClient.signOut()` → `router.push("/signin")`), row
  actions wiring to F4/F5/F11 dialogs.

**Backend.** `src/lib/boards.ts`: `listBoards(userId)` — one query for
`ownerId == me`, one for `access.userId == me` (single-field indexes both);
returns DTOs `{ id, title, updatedAt, role, lockHolderName | null,
starred }`.

**Tests.** Owner sees own; editor/viewer sees shared with correct role;
strangers see nothing; lock holder name resolves.

## F4 — Dashboard: create board

**Story.** As a user, I want a New board button so I can start sketching
immediately.

**Acceptance.** Click → board created (I am owner, default title
"Untitled board") → navigate to `/board/<id>` with an empty canvas;
appears at top of dashboard.

**UI.** `Dashboard.tsx`: New board button → `createBoard` action →
`router.push('/board/'+id)`; pending + error states.

**Backend.** `src/actions/boards.ts`: `createBoard({ title? })` — title
validated (1–120 chars, trimmed; default "Untitled board"). `_id` is a
fresh `nanoid()` (21 chars, unguessable — never an ObjectId); `scenes._id`
is the same string. Inserts `boards` doc (`version: 1`, `access: []`),
then the empty `scenes` doc; if the scene write fails, the board doc is
deleted again (compensating rollback, single-doc writes only) and the
error surfaces; `revalidatePath("/")`.

**Tests.** Creates both docs; caller is owner; unauthenticated rejected.

## F5 — Dashboard: rename / star / duplicate / delete / search

**Story.** As a user, I want to organize my boards (rename, star, duplicate,
delete) so the workspace stays usable.

**Acceptance.** Rename inline; star toggles (starred sort first); duplicate
copies scene with fresh version chain and "(copy)" title; delete asks for
confirm and only owners see it; editors can rename/star/duplicate but not
delete; viewers none of these.

**UI.** `Dashboard.tsx` row menu + inline rename input + confirm dialog
for delete; all call actions then rely on `revalidatePath("/")`.

**Backend.** `src/actions/boards.ts`: `renameBoard` (same 1–120-char
validation), `toggleStar`, `duplicateBoard` (copies scene bytes only —
fresh `version: 1`, new ids, title + " (copy)", **no** lock and **no**
access list: copies are always private to the duplicator),
`deleteBoard` (owner-only guard; deletes scene doc too; succeeds even if
another user holds the lock — the lock dies with the board). Search is
client-side filter over the listed DTOs (small-group scale; Atlas text
indexes exist but are unnecessary here).

**Tests.** Full permission matrix per action (owner/editor/viewer/stranger);
duplicate preserves scene bytes but copies neither lock nor access list;
delete removes both docs even when locked by someone else.

## F6 — Board page: open & load scene

**Story.** As a user with access, I want to open a board and see its saved
content (elements, images, viewport) so I can continue work.

**Acceptance.** Owner/editor/viewer loads scene; **no access (or missing
board) → uniform 404 via `notFound()`** — the two cases are deliberately
indistinguishable (gatekeeping decision); signed-out users still go to
`/signin`, not 404; viewport restored; viewer (`role: "view"`, no lock
involved) gets permanent read-only: no Edit button, subtle "Viewer" chip
instead of `LockBanner`.

**UI.**
- `src/app/board/[id]/page.tsx` (server): `requireUser()` →
  `getBoardMeta` (permission + lock, purge stale; throws uniform
  NOT_FOUND when missing *or* unauthorized) → `loadScene` → renders
  `BoardEditor` with `{ boardId, initialScene, canEdit, lockState }`.
- `src/components/BoardEditor.tsx` (client): mounts `<Excalidraw
  initialData={promise} …>`; CSS import; restore semantics from package.

**Backend.** `src/lib/boards.ts`: `getBoardMeta(boardId, userId)`,
`loadScene(boardId)` (decrypt → decompress → `{ elements, files,
appState, version }`). Route Handler `GET /api/boards/[id]/scene` is NOT
used for initial load (server component path); it exists for client-side
re-fetch after conflicts (F10).

**Tests.** Meta permission branches; scene decrypt/decompress round-trip;
tampered `payload` fails closed.

## F7 — Lock: acquire, heartbeat, release

**Story.** As an editor opening an unlocked board, I want to claim the
edit lock (with visible confirmation) so nobody else edits under me; my
tab keeps it alive and releases it when I leave.

**Acceptance.** "Edit" acquires when free; re-acquire by the current
holder (e.g. second tab) succeeds idempotently and extends the TTL;
heartbeat every 60 s extends 8-min TTL; tab close/hide releases
best-effort; my lock shows "Editing" state; acquire on another user's
lock → 423 + holder name.

**UI.** `BoardEditor.tsx`: Edit button → `acquireLock` action; interval
calls `heartbeatLock`; `beforeunload`/`visibilitychange` call
`releaseLock` (fire-and-forget); lock state banner.

**Backend.** `src/actions/lock.ts` (all tiny payloads — actions, not API):
`acquireLock(boardId)` (CAS: set lock if none/expired; succeed silently
if holder is already me), `heartbeatLock` (holder-only extends
`expiresAt`), `releaseLock` (holder only), `forceReleaseLock`
(owner-only; separate action for audit clarity). Stale-lock purge inside
`getBoardMeta` read path.

**Tests.** Acquire/contend/expire/heartbeat/release/force-release;
same-user re-acquire succeeds; non-holder heartbeat rejected; owner
force-releases stranger lock; non-owner force-release rejected.

## F8 — Read-only when locked by someone else

**Story.** As a user opening a board someone else is editing, I want a
read-only view with "Being edited by X" (plus Retry) instead of an error,
so I can look without breaking their work.

**Acceptance.** Locked board mounts editor with controlled
`viewModeEnabled`, non-interactive toolbar; `LockBanner` shows holder +
Retry (re-checks lock action, then `router.refresh()`); owner sees
Force-release; lock freeing + Retry → Edit button appears.

**UI.** `BoardEditor.tsx` read-only branch; `src/components/LockBanner.tsx`
(holder, Retry, conditional Force-release).

**Backend.** `src/actions/boards.ts`: `getBoardMetaAction(boardId)` —
thin action wrapper over the `getBoardMeta` lib function (lib functions
can't be imported by client components) used by Retry;
`forceReleaseLock` from F7. No new endpoints.

**Tests.** Read-only prop set when locked; Retry transitions on freed lock;
force-release restricted to owner.

## F9 — Autosave while editing

**Story.** As the lock holder editing, I want changes auto-saved to the
cloud so I never lose work.

**Acceptance.** Edits debounce 300 ms → `PUT` scene; save indicator cycles
saved/saving/failed; only holder's writes accepted (non-holder → 423);
oversize payload → clear "too large" error; offline/network fail → queued
banner with Retry (manual, per refresh policy).

**UI.** `BoardEditor.tsx`: `onChange` debounce → `fetch PUT
/api/boards/[id]/scene` with `{ elements, files, appState, baseVersion }`;
save-state indicator; failure banner + Retry; pre-save size estimate
(JSON length check vs 16 MB cap).

**Backend.** Route Handler `PUT /api/boards/[id]/scene`: session →
editor-role + holder check → size guard → decrypt-compare `baseVersion`
vs `version` → CAS write (`version+1`) **plus `boards.updatedAt = now`**
(dashboard "last-edited" ordering depends on it) or **409 + server
scene**. Body cap ~20 MB; rate-limited. (Route Handler, not action —
decided: large payload + real status codes.)

**Tests.** Save bumps version; non-holder rejected; oversize rejected;
CAS mismatch → 409 with server scene bytes.

## F10 — Conflict merge

**Story.** As an editor whose save conflicts (stale tab after TTL expiry), I
want my work merged with the newer saved version rather than overwritten.

**Acceptance.** On 409: fetch server scene, merge local edits via
`reconcileElements` (package root export), rebase `baseVersion`, auto-retry
once; unresolvable → banner offering reload-server / keep-editing (locks
re-checked).

**UI.** `BoardEditor.tsx` conflict branch (mostly automatic; banner only on
second failure).

**Backend.** None new (uses F9's 409 payload + F6 re-fetch).

**Tests.** reconcile unit: local + remote element sets merge
deterministically; retry succeeds after rebase.

## F11 — Sharing: grant / change / revoke view & edit

**Story.** As an owner, I want to share a board with named people as viewer
or editor, change roles, and revoke, so collaboration stays controlled.

**Acceptance.** Email must belong to a registered user — i.e. must have
signed in at least once (a `user` row must exist; else "X hasn't signed in
yet" — access is granted by userId, and there is nothing to grant to a
stranger); roles viewer/editor; owner cannot revoke
self; revoked editor loses edit immediately (next action fails
`FORBIDDEN`; open editor drops to read-only on next heartbeat/check);
access list shows current lock holder.

**UI.** `src/components/AccessDialog.tsx` (opened from dashboard row or
editor menu): user list with roles, add-by-email + role picker, revoke
buttons; calls actions; errors inline.

**Backend.** `src/actions/access.ts`: `setAccess(boardId, { email, role })`
(owner only; resolves email → userId; upserts `access` entry),
`revokeAccess(boardId, { userId })` (owner only; refuses self-revoke).
Reads reuse `getBoardMeta`.

**Tests.** Grant/change/revoke matrix; never-signed-in email rejected;
self-revoke refused; revoked user blocked from save + lock acquire.

## F12 — Removed (allow-list admin; open signup has no admin)

> Deleted 2026-09-20 with the invite-only model. Open signup needs no
> allow-list, no `ALLOWED_EMAILS`, no `admin` role, and no
> `src/actions/invites.ts`. F-number kept as a tombstone so F13–F15
> references stay stable.

## F13 — Error & edge states

**Story.** As a user, I want every failure (no access, missing board, too
large, sync failed, session expired) to explain itself with a next step,
never a blank screen or silent data loss.

**Acceptance.** 404 page (shown for missing **and** no-access boards —
uniform response, deliberate) with back-to-dashboard; scene-too-large
names the limit; sync-failed banner with Retry; expired session →
re-sign-in preserving `?next=` URL.

**UI.** `src/app/board/[id]/not-found.tsx` (`notFound()` covers both
cases — no `forbidden.tsx`, no 403 page, by gatekeeping decision);
`src/app/board/[id]/page.tsx` calls `notFound()` in the render path
(never in a layout); `BoardEditor` banners; signin `next` handling (F2).

**Backend.** Coded errors from §0 shape surfaced verbatim; no new logic.

**Tests.** Each state renders (component tests); expired-session flow (F2).

## F14 — Deploy & health (ops, not a user story)

**Code required.**
- `netlify.toml`: Netlify build config using the official Next.js runtime
  (no `standalone` output, no zip packaging, no startup command).
- `.nvmrc` / `NODE_VERSION`: pins Node 22 so Netlify builds match local dev.
- `src/app/api/health/route.ts`: returns `{ ok: true }` (no auth) —
  post-deploy smoke check (open it after deploy; Netlify has no
  App-Service-style health-check wiring and preview deploys must never
  point at prod).
- Deploys: Netlify Git integration auto-builds on push to `main`
  (`npm ci` → `next build` on Netlify); no GitHub Actions deploy step.
- `.env.example` (plan §19); environment variables set in the Netlify UI
  per environment (prod vs preview), including `BETTER_AUTH_URL` and
  `MONGODB_URI` / `MONGODB_DB` (Atlas; preview uses the dev/scratch
  database, never prod).
- OAuth registrations: Google Cloud OAuth client + GitHub OAuth App with
  prod redirect `https://<site>.netlify.app/api/auth/callback/...`
  and localhost dev counterparts. **Google trap:** an External app in
  Testing mode caps at 100 users and shows an "unverified app" screen —
  either add each group member as a test user or publish to Production
  (basic profile/email scopes need no verification).
- Standalone smoke: `npm run build` passes locally before pushing
  (Netlify builds on push).

## F15 — Public view link (optional, owner-controlled)

**Story.** As an owner, I want to flip a board to link-visible read-only
so an outsider (no account) can view it; as the group, we want that link
revocable at any time and useless for anything but viewing.

**Acceptance.** Owner enables per board → unguessable URL `/share/<token>`
renders the scene read-only with no sign-in; no Edit, no lock
interaction, no user identity, no save; anyone (including strangers) with
a bad/expired token gets uniform 404; owner can disable (token cleared)
or regenerate (old token dies instantly); public viewers never affect
locks and see the board even while locked; public page shows no board
list, no dashboard, no user data.

**UI.**
- `AccessDialog.tsx` "Public link" section (owner only): enable/disable
  toggle, copy-link button, regenerate button (with confirm — kills the
  old link).
- `src/app/share/[token]/page.tsx` (server, **no auth**): resolve token →
  `notFound()` on miss → `loadScene` → renders `BoardEditor` in forced
  read-only mode (`canEdit: false`, `publicMode: true` hides Edit,
  sharing, and account UI).

**Backend.**
- Data: `boards.publicToken?: string | null` (`nanoid()`, 21+ chars;
  null = disabled). Indexed single-field for token lookup.
- `src/actions/access.ts`: `setPublicLink(boardId, { enabled })`
  (owner only; generates/clears token), `regeneratePublicLink(boardId)`
  (owner only; replaces token).
- Token resolve path (`getBoardByToken`) shared by the share page; bad
  token → `notFound()` — same uniform-404 rule, no existence signal.
  Rate-limit token resolution (enumeration defense in depth, alongside
  unguessable tokens).
- Public page loads the scene server-side via lib (no session); the scene
  Route Handlers keep requiring session + access — public viewing never
  touches them.

**Tests.** Enable/disable/regenerate lifecycle; old token 404s after
regenerate/disable; bad token 404; public page renders no edit affordances;
public viewer cannot save, lock, or list (handler-level rejections).

---

## Test matrix summary (unit + integration only, Vitest)

| Area | Cases |
|---|---|
| Auth (F1) | first sign-in creates user for any Google/GitHub account; repeat sign-in reuses row; `getSession` round-trip |
| Sessions (F2) | expiry redirect + `next`, `requireUser` unauthenticated, sign-out lands on `/signin` |
| Boards (F3–F5) | list scoping, create docs + default title + rollback, title validation, full ACL matrix × rename/star/duplicate/delete, duplicate copies neither lock nor access, delete-despite-lock |
| Lock (F7–F8) | acquire/contend/expire/heartbeat/release/force, same-user re-acquire, read-only branch, viewer permanent read-only |
| Scene (F9–F10) | version CAS, `updatedAt` bump, 409 + merge, size guard, holder-only writes |
| Sharing (F11) | grant/change/revoke, never-signed-in email rejected, self-revoke refusal, post-revoke blocking |
| Public link (F15) | enable/disable/regenerate lifecycle, old/bad token 404, no edit/save/lock/list for public viewers |

`mongodb-memory-server` for all DB tests; one smoke job against the Atlas
dev/scratch database (M1 proves auth; M2+ proves app collections).
