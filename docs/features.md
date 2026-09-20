# KatanaSketch — Detailed Features & User Stories

> **Status:** draft · **Date:** 2026-09-20 · **Revised:** open signup (no invite-only)
> **Relation to other docs:** `docs/specs.md` (F1–F15, F12 removed) is
> authoritative on transport and code split; `docs/initial-plan.md` is
> authoritative on architecture and deploy (Netlify Git auto-deploy +
> existing MongoDB Atlas cluster). This file expands the same features into
> detailed, layered user stories for **both** audiences: plain-language
> narrative up top, builder notes (UI / Backend / Edge cases / Tests) below.
> No new scope is introduced here — anything not traceable to F1–F15 is
> marked as such. Open signup: anyone with Google/GitHub can register; no
> allow-list, no admin role.

## How to read this file

Each feature has:

1. **Overview** — plain language, no jargon. Safe for stakeholders.
2. **Priority & milestone** — MoSCoW + M1–M4 (see legend below).
3. **Stories** (`F<n>.<k>`) — each story follows the same layered shape:
   - *Story* — "As a … I want … So that …"
   - *Acceptance* — checkable bullets. If it isn't checkable, it isn't done.
   - *UI (builders)* — components, routes, states.
   - *Backend (builders)* — actions/handlers/lib functions, guards, codes.
   - *Edge cases* — what must not break, what must fail cleanly.
   - *Tests* — Vitest unit/action/handler cases (plus manual where noted).

## Priority legend (MoSCoW)

| Tag | Meaning |
|---|---|
| **Must** | MVP — the app is not shippable without it. |
| **Should** | Expected at launch; may slip one milestone under pressure. |
| **Could** | Optional at launch (F15 is the only Could here). |
| **Won't** | Explicitly out of scope (see plan §2: realtime co-editing, teams, billing, AI, PWA offline shell). |

## Milestone map

| Milestone | Features |
|---|---|
| **M1** — app + auth + deploy | F1, F2, F14 (+ thin F3 shell: empty dashboard) |
| **M2** — boards + dashboard | F3, F4, F5 |
| **M3** — editor + save + lock | F6, F7, F8, F9, F10, F13 (board paths) |
| **M4** — sharing + hardening | F11, F15, F13 (full pass) |

## Cross-cutting rules (summary — specs §0 applies)

- Server Components read Atlas directly via `src/lib/*`; mutations are
  Server Actions; Route Handlers exist only for `/api/auth/[...all]`,
  `GET`/`PUT /api/boards/[id]/scene`, `/api/health`.
- Action results: `{ ok: true, data } | { ok: false, error: { code, message } }`
  with codes `UNAUTHENTICATED, FORBIDDEN, NOT_FOUND, LOCKED,
  CONFLICT, TOO_LARGE, VALIDATION`.
- Gatekeeping: board ids are unguessable `nanoid` strings;
  `authorizeBoard(boardId, userId, minRole)` is the single enforcement point;
  missing and forbidden both render uniform 404 via `notFound()`;
  signed-out users redirect to `/signin` (auth ≠ authz).
- Manual-refresh policy: `revalidatePath("/")` after board-list mutations;
  `router.refresh()` on board pages. No polling, no WebSockets.

---

## F1 — Sign in with Google / GitHub (open signup)

**Overview.** Anyone can get in. A visitor picks Google or GitHub, approves
OAuth, and lands in their workspace — first sign-in creates the account
automatically. No passwords, no invites.

**Priority:** Must · **Milestone:** M1

### F1.1 — Any user signs in with Google

- *Story.* As a visitor with Gmail, I want to sign in with Google so
  that I reach my workspace without a password.
- *Acceptance.*
  - `/signin` shows a Google button; completing OAuth lands on `/`
    (or `?next=` target when present).
  - First sign-in creates the user row automatically; repeat sign-ins reuse it.
  - Signed-in user visiting `/signin` redirects to `/`.
  - Asserted end-to-end in M1 against the Atlas dev database.
- *UI (builders).* `src/app/signin/page.tsx` (server: session check +
  redirect, renders `SignInButtons` with `next` param);
  `SignInButtons.tsx` (client: `authClient.signIn.social({ provider:
  "google", callbackURL })`).
- *Backend (builders).* `lib/auth.ts`: `mongodbAdapter(db)` (standard Atlas
  path), `socialProviders.google` (env id/secret), 30-day sessions. No
  `databaseHooks` gates, no `role` field, no admin plugin.
  `GET`/`POST /api/auth/[...all]` (mandatory Route Handler).
- *Edge cases.* Google account with hidden primary email still resolves;
  double-click doesn't create two users; OAuth cancel returns to `/signin`
  without a crash.
- *Tests.* First sign-in creates a user for any Google account; repeat
  sign-in reuses the row; `getSession` round-trip with mocked provider.

### F1.2 — Any user signs in with GitHub

- *Story.* As a visitor with GitHub, I want to sign in with GitHub so that I
  don't need a Google account.
- *Acceptance.* Same as F1.1 via the GitHub button, including private-email
  accounts.
- *UI (builders).* Same components; second button with `provider: "github"`.
- *Backend (builders).* `socialProviders.github`; Better Auth fetches
  `/user/emails` automatically — no extra code.
- *Edge cases.* GitHub private email (no public email) still yields the
  verified primary address; emails compared case-insensitively (stored
  lowercased).
- *Tests.* GitHub provider path with private-email fixture resolves correctly;
  repeat sign-in reuses the row.

---

## F2 — Sessions & auth gating

**Overview.** Signing in sticks — restarts don't log you out. When a session
finally expires, you go back to sign-in and return to the board you were on.

**Priority:** Must · **Milestone:** M1

### F2.1 — Sessions survive restarts

- *Story.* As a signed-in user, I want to stay signed in across server
  restarts and redeploys so that shipping doesn't log out the group.
- *Acceptance.* Restart/redeploy (Netlify build) keeps sessions; dashboard
  and boards load without re-login while the 30-day session is valid.
- *UI (builders).* No UI; per-page gating via `requireUser()` in server
  components (no middleware).
- *Backend (builders).* Better Auth database sessions in the Atlas `session`
  collection; `lib/session.ts` exposes `requireUser()` (throws coded
  `UNAUTHENTICATED`) + `getUser()`.
- *Edge cases.* Cookie present but session row deleted (e.g. manual DB cleanup) →
  treated as signed-out, redirect to `/signin`.
- *Tests.* `requireUser` throws `UNAUTHENTICATED` without a cookie; valid
  cookie returns the user.

### F2.2 — Expired session returns to the right board

- *Story.* As a user with an expired session, I want to re-sign-in and land
  back on the board I was viewing.
- *Acceptance.* Protected page with expired session →
  `/signin?next=/board/<id>`; after sign-in, return to `next`.
- *UI (builders).* `signin/page.tsx` honors `next`; board pages pass their
  URL when redirecting.
- *Backend (builders).* Same `requireUser()` path; redirect (not 404) for
  unauthenticated access — authentication ≠ authorization.
- *Edge cases.* Malformed `next` (external URL) is ignored, falls back to
  `/`; signed-in user visiting `/signin?next=…` goes to `next` if internal.
- *Tests.* Expired-session redirect preserves `next`; open-redirect with
  absolute URL is rejected.

---

## F3 — Dashboard: view my boards

**Overview.** Your workspace home: boards you own and boards shared with you,
with titles, last-edited times, your role, lock dots, and stars — plus
search and a manual refresh.

**Priority:** Must · **Milestone:** M2 (thin empty-state shell in M1)

### F3.1 — See owned and shared boards

- *Story.* As a user, I want two sections (Mine / Shared with me) so that I
  can tell what I own versus what others shared.
- *Acceptance.*
  - `/` shows both sections with `{ title, updatedAt, role, lockHolderName,
    starred }` per row.
  - Owner sees own boards; editor/viewer sees shared boards with the correct
    role; strangers see nothing.
  - Lock dot + holder name on locked boards; empty states when a section is
    empty; header shows name + Sign out (→ `/signin`); signed-out → `/signin`.
- *UI (builders).* `app/page.tsx` (server: `requireUser()` →
  `listBoards(userId)` → `Dashboard`); `Dashboard.tsx` (sections, lock dots,
  Refresh `router.refresh()`, user menu `authClient.signOut()`).
- *Backend (builders).* `lib/boards.ts listBoards(userId)`: one query for
  `ownerId == me`, one for `access.userId == me` (single-field indexes).
- *Edge cases.* Board shared then revoked disappears on next refresh; lock
  holder name falls back gracefully if the user row is gone.
- *Tests.* List scoping per role; stranger sees nothing; lock holder resolves.

### F3.2 — Search and refresh the list

- *Story.* As a user with many boards, I want to filter by title and
  manually refresh so that the list stays usable without auto-reload magic.
- *Acceptance.* Search input filters locally (title substring,
  case-insensitive); Refresh button re-fetches; no polling/WebSockets.
- *UI (builders).* Local filter state in `Dashboard.tsx`; Refresh calls
  `router.refresh()`.
- *Backend (builders).* None — client-side filter over listed DTOs
  (small-group scale; Atlas text indexes exist but are unnecessary).
- *Edge cases.* Empty query shows all; no-match shows an empty state, not a
  blank page.
- *Tests.* Filter matches case-insensitively; refresh re-renders with fresh
  DTOs (component test).

---

## F4 — Dashboard: create board

**Overview.** One click, one new blank canvas, owned by you.

**Priority:** Must · **Milestone:** M2

### F4.1 — Create and open a board

- *Story.* As a user, I want a New board button so that I can start
  sketching immediately.
- *Acceptance.*
  - Click → board created (caller is owner, default title "Untitled board")
    → navigate to `/board/<id>` with an empty canvas; board appears at top
    of the dashboard.
  - Title validated (1–120 chars, trimmed); pending + error states on the
    button.
- *UI (builders).* `Dashboard.tsx` New board button → `createBoard` action →
  `router.push('/board/'+id)`.
- *Backend (builders).* `actions/boards.ts createBoard({ title? })`: fresh
  `nanoid()` `_id` (never ObjectId; `scenes._id` mirrors it); inserts
  `boards` (`version: 1`, `access: []`) then the empty `scenes` doc; on scene
  failure the board doc is deleted again (compensating rollback) and the
  error surfaces; `revalidatePath("/")`.
- *Edge cases.* Double-click creates one board (button disables while
  pending); unauthenticated call rejected `UNAUTHENTICATED`; over-long title
  rejected `VALIDATION`.
- *Tests.* Creates both docs with caller as owner; default title applied;
  unauthenticated rejected; rollback on scene-write failure.

---

## F5 — Dashboard: rename / star / duplicate / delete / search

**Overview.** Keep the workspace tidy: rename inline, star favorites,
duplicate to fork, delete with confirm. Permissions follow your role.

**Priority:** Must · **Milestone:** M2

### F5.1 — Rename a board

- *Story.* As an owner or editor, I want to rename a board inline so that
  titles stay meaningful.
- *Acceptance.* Inline rename; 1–120-char validation; viewers see no rename
  affordance; strangers rejected.
- *UI (builders).* Row menu + inline input in `Dashboard.tsx`; errors inline.
- *Backend (builders).* `renameBoard` action; `authorizeBoard` owner/editor
  gate; `revalidatePath("/")`.
- *Edge cases.* Blank/whitespace-only rejected; concurrent rename last-write
  wins (single-doc update, no merge).
- *Tests.* Owner/editor succeed; viewer/stranger `FORBIDDEN`; validation
  rejects empty/over-long.

### F5.2 — Star / unstar a board

- *Story.* As a user, I want to star boards so that favorites sort first.
- *Acceptance.* Star toggles per user; starred sort first in both sections.
- *UI (builders).* Star toggle on the row; optimistic toggle with rollback
  on failure.
- *Backend (builders).* `toggleStar` action (anyone with access, including
  viewers); stored in `boards.starredBy: [userId]`.
- *Edge cases.* Starring a board you then lose access to is harmless (row
  disappears).
- *Tests.* Toggle adds/removes caller id; viewer can star; stranger rejected.

### F5.3 — Duplicate a board

- *Story.* As an owner or editor, I want to duplicate a board so that I can
  fork it without touching the original.
- *Acceptance.* Copy gets fresh ids, `version: 1`, title + " (copy)",
  identical scene bytes; **no** lock and **no** access list (copies are
  always private to the duplicator).
- *UI (builders).* Row "Duplicate" action → navigate or stay with toast;
  pending state.
- *Backend (builders).* `duplicateBoard`: reads scene bytes, writes new
  `boards` + `scenes` docs under a new `nanoid()`.
- *Edge cases.* Duplicating a locked board succeeds (lock not copied);
  duplicating a huge scene still respects the 16 MB guard.
- *Tests.* Bytes preserved; lock/access absent on the copy; viewer/stranger
  rejected.

### F5.4 — Delete a board (owners only)

- *Story.* As an owner, I want to delete a board with confirmation so that
  accidents are rare but cleanup is possible.
- *Acceptance.* Confirm dialog; only owners see it; deletes both `boards`
  and `scenes` docs; succeeds even when someone else holds the lock (the
  lock dies with the board).
- *UI (builders).* Confirm dialog in `Dashboard.tsx`.
- *Backend (builders).* `deleteBoard` owner-only guard.
- *Edge cases.* Deleting an open board: other tabs get uniform 404 on next
  load/refresh; double-delete is idempotent-ish (second call `NOT_FOUND`).
- *Tests.* Owner deletes both docs despite foreign lock; editor/viewer
  rejected; missing board `NOT_FOUND`.

---

## F6 — Board page: open & load scene

**Overview.** Opening a board shows its saved content — shapes, images,
viewport. No access looks exactly like "doesn't exist"; signed-out users get
sign-in instead.

**Priority:** Must · **Milestone:** M3

### F6.1 — Owner/editor/viewer loads the saved scene

- *Story.* As a user with access, I want to open a board and see its saved
  elements, images, and viewport so that I can continue work.
- *Acceptance.*
  - Scene restores `{ elements, files, appState: { zoom, scroll } }`.
  - Viewers get permanent read-only: no Edit button, subtle "Viewer" chip
    (no lock involved).
- *UI (builders).* `app/board/[id]/page.tsx` (server: `requireUser()` →
  `getBoardMeta` → `loadScene` → `BoardEditor { boardId, initialScene,
  canEdit, lockState }`); `BoardEditor.tsx` mounts `<Excalidraw
  initialData={promise}>` with package CSS.
- *Backend (builders).* `lib/boards.ts getBoardMeta` (permission + stale-lock
  purge; throws uniform `NOT_FOUND` when missing *or* unauthorized) and
  `loadScene` (decrypt AES-256-GCM → decompress pako → DTO). Initial load is
  the server-component path; `GET /api/boards/[id]/scene` exists only for
  client re-fetch (F10).
- *Edge cases.* Corrupt/tampered `payload` fails closed (error, no partial
  render); huge images embedded base64 in the payload — no file endpoints.
- *Tests.* Meta permission branches; decrypt/decompress round-trip; tampered
  payload fails closed.

### F6.2 — No access and missing boards look identical

- *Story.* As the group, we want probing board URLs to reveal nothing, so
  that unauthorized users can't tell "private" from "nonexistent".
- *Acceptance.* Authenticated-but-unauthorized **or** missing → uniform 404
  via `notFound()` (no 403 page, no `forbidden.tsx`); signed-out → `/signin`.
- *UI (builders).* `app/board/[id]/not-found.tsx` (back-to-dashboard);
  `notFound()` called in the render path, never in a layout.
- *Backend (builders).* Same `getBoardMeta` throw path for both cases.
- *Edge cases.* Admin with no access gets 404 too (least privilege); timing
  side-channels out of scope.
- *Tests.* Missing vs forbidden both render `not-found`; signed-out
  redirects instead.

---

## F7 — Lock: acquire, heartbeat, release

**Overview.** One editor at a time. Claim the lock to edit; your tab keeps it
alive and releases it when you leave. Second tabs by the same person just
extend it.

**Priority:** Must · **Milestone:** M3

### F7.1 — Acquire the edit lock

- *Story.* As an editor opening an unlocked board, I want to claim the lock
  so that nobody edits under me.
- *Acceptance.* "Edit" acquires when free (or expired); re-acquire by the
  current holder (e.g. second tab) succeeds idempotently and extends the TTL;
  acquiring someone else's active lock → `LOCKED` (423 over HTTP) + holder
  name; my lock shows "Editing".
- *UI (builders).* `BoardEditor.tsx` Edit button → `acquireLock` action.
- *Backend (builders).* `actions/lock.ts acquireLock(boardId)`: CAS — set
  lock if none/expired, succeed silently if holder is already me. Lock shape
  `{ userId, userName, acquiredAt, expiresAt }`, TTL 8 min. Stale purge also
  runs in the `getBoardMeta` read path (crashed tabs need no manual action).
- *Edge cases.* Two tabs racing: both end "Editing" for the same user, one
  TTL; acquire with no access → `FORBIDDEN`/`NOT_FOUND`, never `LOCKED`.
- *Tests.* Acquire/contend/expire; same-user re-acquire succeeds and extends;
  non-holder acquire rejected with holder name.

### F7.2 — Heartbeat keeps the lock alive

- *Story.* As the lock holder, I want my tab to keep the lock so that long
  sessions aren't stolen mid-edit.
- *Acceptance.* Heartbeat every 60 s extends the 8-min TTL; non-holder
  heartbeat rejected; revoked user drops to read-only on next check.
- *UI (builders).* Interval in `BoardEditor.tsx` while holding.
- *Backend (builders).* `heartbeatLock` (holder-only extends `expiresAt`).
- *Edge cases.* Laptop sleep past TTL → lock expires; next heartbeat fails
  and the UI degrades to read-only + conflict path (F10) instead of
  overwriting.
- *Tests.* Holder heartbeat extends; non-holder rejected; expired lock
  acquirable by another editor.

### F7.3 — Release on leave; force-release for owners

- *Story.* As the holder, I want the lock released when I leave; as an
  owner, I want to free a stuck lock so that work isn't blocked.
- *Acceptance.* Tab close/hide releases best-effort (`beforeunload` /
  `visibilitychange`, fire-and-forget); `forceReleaseLock` is owner-only;
  others rejected.
- *UI (builders).* Release hooks in `BoardEditor.tsx`; Force-release button
  for eligible users (also surfaced in `LockBanner`, F8).
- *Backend (builders).* `releaseLock` (holder only); `forceReleaseLock`
  (separate action for audit clarity).
- *Edge cases.* Release for an already-expired lock succeeds silently;
  force-release while holder is mid-save → holder's next save gets 423 and
  must re-acquire.
- *Tests.* Release/force matrix: owner frees stranger lock; non-owner
  (including editors/viewers) rejected.

---

## F8 — Read-only when locked by someone else

**Overview.** If someone else is editing, you still get in — read-only, with
their name, a Retry button, and (for owners) a force-release.

**Priority:** Must · **Milestone:** M3

### F8.1 — View a locked board read-only

- *Story.* As a user opening a board someone else is editing, I want a
  read-only view with "Being edited by X" instead of an error.
- *Acceptance.* Editor mounts with controlled `viewModeEnabled`
  (non-interactive toolbar); `LockBanner` shows holder + Retry; Retry
  re-checks the lock action then `router.refresh()`; freed lock + Retry →
  Edit button appears.
- *UI (builders).* `BoardEditor.tsx` read-only branch; `LockBanner.tsx`
  (holder, Retry, conditional Force-release).
- *Backend (builders).* `getBoardMetaAction(boardId)` — thin action wrapper
  over `getBoardMeta` for Retry (lib can't be imported by clients);
  `forceReleaseLock` from F7. No new endpoints.
- *Edge cases.* Lock frees between render and Retry → Edit appears without a
  full reload; holder name missing → generic "someone else" label.
- *Tests.* Read-only prop set when locked; Retry transitions on freed lock;
  force-release visibility restricted to owner.

---

## F9 — Autosave while editing

**Overview.** Edits save themselves to the cloud a moment after you stop —
with a visible saved/saving/failed state and a manual retry when offline.

**Priority:** Must · **Milestone:** M3

### F9.1 — Debounced cloud save with visible state

- *Story.* As the lock holder, I want changes auto-saved so that I never
  lose work.
- *Acceptance.* Edits debounce 300 ms → `PUT` scene; indicator cycles
  saved / saving / failed; only the holder's writes accepted (non-holder →
  423); oversize → clear "too large" error naming the 16 MB MongoDB limit;
  offline/network fail → queued banner with manual Retry.
- *UI (builders).* `BoardEditor.tsx onChange` debounce → `fetch PUT
  /api/boards/[id]/scene { elements, files, appState, baseVersion }`;
  save-state indicator; failure banner + Retry; pre-save size estimate (JSON
  length vs 16 MB cap).
- *Backend (builders).* `PUT /api/boards/[id]/scene` (Route Handler, not an
  action — large payload + real statuses): session → editor-role + holder
  check → size guard → `baseVersion` vs `version` CAS → write `version+1`
  **plus `boards.updatedAt = now`** (dashboard ordering depends on it) or
  `409 + server scene`. Body cap ~20 MB; rate-limited.
- *Edge cases.* Save during lock expiry → 423, UI drops to read-only flow;
  rapid typing coalesces into one PUT; images travel embedded base64.
- *Tests.* Save bumps version + `updatedAt`; non-holder rejected; oversize
  rejected; CAS mismatch → 409 with server bytes.

---

## F10 — Conflict merge

**Overview.** If your tab went stale (TTL expired, someone else saved), your
work merges with theirs instead of overwriting — automatically on the first
try.

**Priority:** Should · **Milestone:** M3

### F10.1 — Automatic merge on version conflict

- *Story.* As an editor whose save conflicts, I want my work merged with the
  newer saved version rather than overwritten or lost.
- *Acceptance.* On 409: fetch server scene, merge local edits via
  `reconcileElements` (package root export), rebase `baseVersion`,
  auto-retry once; second failure → banner offering reload-server /
  keep-editing (locks re-checked).
- *UI (builders).* `BoardEditor.tsx` conflict branch (mostly automatic;
  banner only on second failure).
- *Backend (builders).* None new — uses F9's 409 payload + F6 re-fetch.
- *Edge cases.* Truly unresolvable (e.g. scene replaced wholesale) keeps the
  banner until the user chooses; merge never invents elements — deterministic
  `reconcileElements` semantics.
- *Tests.* Reconcile unit: local + remote sets merge deterministically;
  retry succeeds after rebase.

---

## F11 — Sharing: grant / change / revoke view & edit

**Overview.** Owners share boards with named people as viewers or editors,
change roles, and revoke. Sharing only works for people who have signed in
at least once.

**Priority:** Must · **Milestone:** M4

### F11.1 — Grant viewer or editor access

- *Story.* As an owner, I want to share a board with a named person as
  viewer or editor so that collaboration stays controlled.
- *Acceptance.* Email must belong to a registered user — must have signed
  in at least once (a `user` row must exist, else "X hasn't signed in yet"
  — access is granted by userId, and there is nothing to grant to a stranger).
- *UI (builders).* `AccessDialog.tsx` (dashboard row or editor menu): user
  list with roles, add-by-email + role picker, errors inline.
- *Backend (builders).* `actions/access.ts setAccess(boardId, { email, role })`
  (owner only; resolves email → userId; upserts `access` entry).
- *Edge cases.* Case-variant emails resolve to the same user; granting the
  owner's own email is a no-op; granting while the board is locked doesn't
  disturb the lock.
- *Tests.* Grant matrix per role; never-signed-in email rejected; stranger
  granter rejected.

### F11.2 — Change roles and revoke access

- *Story.* As an owner, I want to change roles and revoke so that ex-editors
  lose edit immediately and I can't lock myself out.
- *Acceptance.* Role change viewer↔editor takes effect on next action;
  revoked editor's next save/lock fails `FORBIDDEN` and open editors drop to
  read-only on next heartbeat/check; owner cannot revoke self; access list
  shows the current lock holder.
- *UI (builders).* Same dialog: role picker per row, revoke buttons;
  lock-holder row.
- *Backend (builders).* `setAccess` upsert for changes; `revokeAccess(boardId,
  { userId })` (refuses self-revoke); reads reuse `getBoardMeta`.
- *Edge cases.* Revoking the current lock holder doesn't clear the lock row
  (it purges on read/expiry; their writes already fail); revoking the last
  editor is allowed (owner remains).
- *Tests.* Change/revoke matrix; self-revoke refused; post-revoke save + lock
  blocked.

---

## F12 — Removed (allow-list admin; open signup has no admin)

> Deleted 2026-09-20 with the invite-only model. Open signup needs no
> allow-list, no `ALLOWED_EMAILS`, no `admin` role, and no
> `src/actions/invites.ts`. F-number kept as a tombstone so F13–F15
> references stay stable.

---

## F13 — Error & edge states

**Overview.** Every failure explains itself with a next step — never a blank
screen or silent data loss.

**Priority:** Must · **Milestone:** M3 (board paths) → M4 (full pass)

### F13.1 — Board failures: 404, too-large, sync-failed

- *Story.* As a user, I want missing/no-access boards, oversize scenes, and
  failed syncs to explain themselves.
- *Acceptance.* Uniform 404 page (missing **and** no-access) with
  back-to-dashboard; scene-too-large names the 16 MB limit; sync-failed
  banner with Retry.
- *UI (builders).* `app/board/[id]/not-found.tsx`; `BoardEditor` banners.
- *Backend (builders).* Coded errors from §0 surfaced verbatim; no new logic.
- *Edge cases.* 404 page itself never leaks existence; retry buttons never
  double-submit.
- *Tests.* Each state renders (component tests).

### F13.2 — Session failures preserve context

- *Story.* As a user whose session expired mid-board, I want re-sign-in to
  bring me back (see F2.2), not dump me on an empty dashboard.
- *Acceptance.* Expired session → re-sign-in preserving `?next=` board URL.
- *UI (builders).* Sign-in `next` handling (F2); board pages redirect (not
  404) when unauthenticated.
- *Backend (builders).* `requireUser()` coded error path.
- *Edge cases.* External `next` rejected (open-redirect defense).
- *Tests.* Expired-session flow (F2 tests).

---

## F14 — Deploy & health (ops, not a user story)

**Overview (builders).** What "shipped" means on Netlify + Atlas. No
standalone output, no zip, no Actions deploy — Git push builds, `/api/health`
proves it.

**Priority:** Must · **Milestone:** M1 (then every milestone)

- `netlify.toml` (Next.js runtime), `.nvmrc` / `NODE_VERSION` (Node 22).
- `GET /api/health` → `{ ok: true }`, no auth — post-deploy smoke check;
  preview deploys must never point at prod.
- Netlify Git integration auto-builds `main` (`npm ci` → `next build`);
  env vars (`BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `GOOGLE_*`,
  `GITHUB_*`, `MONGODB_URI`, `MONGODB_DB`, `SCENE_KEY`)
  set in the Netlify UI per environment.
- OAuth callbacks: `https://<site>.netlify.app/api/auth/callback/...` +
  localhost counterparts. Google Testing-mode trap (100 users / unverified
  screen) → add test users or publish (profile/email scopes need no
  verification).
- Pre-push gate: `npm run build` green locally.
- *Tests.* Health returns `{ ok: true }` unauthenticated; preview config
  never resolves the prod database (config test).

---

## F15 — Public view link (optional, owner-controlled)

**Overview.** Owners can flip a board to link-visible read-only for outsiders
with no account. The link is unguessable, revocable, regenerable — and good
for viewing only.

**Priority:** Could · **Milestone:** M4

### F15.1 — Enable, view, disable, regenerate

- *Story.* As an owner, I want to share a read-only link so that an outsider
  can view without an account; as the group, we want it revocable anytime.
- *Acceptance.*
  - Enable per board → `/share/<token>` renders read-only with no sign-in;
    no Edit, no lock, no identity, no save; no board list/dashboard/user data.
  - Bad/disabled token → uniform 404; disable clears the token; regenerate
    kills the old token instantly; public viewers see the board even while
    locked and never affect locks.
- *UI (builders).* `AccessDialog.tsx` "Public link" section (owner only):
  enable/disable toggle, copy-link, regenerate (confirm). `app/share/[token]/
  page.tsx` (server, **no auth**): resolve → `notFound()` on miss →
  `loadScene` → `BoardEditor` forced read-only (`canEdit: false`,
  `publicMode: true` hides Edit/sharing/account UI).
- *Backend (builders).* `boards.publicToken?: string | null` (`nanoid()`
  21+ chars; null = disabled; single-field index); `setPublicLink(boardId,
  { enabled })` + `regeneratePublicLink(boardId)` (owner only);
  `getBoardByToken` shared resolver (rate-limited; unguessable tokens are
  the primary defense). Scene handlers keep requiring session + access —
  public viewing never touches them.
- *Edge cases.* Regenerate-while-viewed: old page keeps rendering its loaded
  snapshot but refresh 404s; enumeration throttled by rate limit.
- *Tests.* Enable/disable/regenerate lifecycle; old token 404s after
  regenerate/disable; bad token 404; no edit affordances rendered; public
  save/lock/list rejected at handler level.

---

## Non-functional stories (separate section)

These constrain *how* the features behave. Each is acceptance-tested, not
just aspirational.

### NF1 — Performance (small-group scale)

- *Stories.* As a user, I want the dashboard to load fast and autosave to
  feel instant on normal broadband, so that the tool doesn't interrupt flow.
- *Acceptance.*
  - Dashboard first paint with 100 boards is interactive without pagination
    (client filter over DTOs suffices at this scale).
  - Autosave debounce 300 ms; a 1 MB scene PUT completes without blocking
    the editor thread.
  - Scene loads restore viewport without visible re-layout jumps.
- *Builders.* Server Components fetch directly (no client waterfall);
  `BoardEditor` debounces and estimates size pre-save; no polling loops.
- *Edge cases.* 15 MB scene near the cap still saves with a clear progress
  state; slow networks show saving → queued → retry, never silent loss.
- *Tests.* Debounce unit; size-estimate unit; handler body-cap (~20 MB)
  rejects oversize.

### NF2 — Security & privacy

- *Stories.* As the group, we want unguessable URLs, least-privilege access,
  encrypted scene blobs, and no existence leaks, so that private boards stay
  private.
- *Acceptance.*
  - Board ids and public tokens are `nanoid` 21+ chars; missing ≡ forbidden
    (uniform 404) everywhere including `/share/*`.
  - Every page/action/handler calls `authorizeBoard` first; there is no
    admin role and no privilege escalation path — `forceReleaseLock` is
    owner-only.
  - Scenes stored AES-256-GCM (`SCENE_KEY` 32-byte); tampered payload fails
    closed; secrets server-side only (no `NEXT_PUBLIC_*`).
  - Public pages expose no user data, no board list, no save/lock paths.
- *Builders.* `lib/boards.ts` enforcement; `lib/scene.ts` crypto + 16 MB
  guard; rate limits on scene PUT and token resolution.
- *Edge cases.* `SCENE_KEY` rotation is out of scope (documented — old blobs
  unreadable after rotation); deactivated users' sessions expire naturally.
- *Tests.* ACL matrix every action; uniform-404 branches; crypto round-trip
  + tamper rejection; public handler rejections.

### NF3 — Reliability & offline behavior

- *Stories.* As an editor, I want transient failures to queue with a retry
  instead of losing strokes, so that flaky wifi isn't catastrophic.
- *Acceptance.*
  - Offline/network fail during autosave → queued banner with manual Retry;
    no silent drop; no auto-retry storms (manual-refresh policy).
  - Conflict path (F10) auto-retries once, then asks — never overwrites
    blindly.
  - Lock heartbeats tolerate one missed beat; sleep-past-TTL degrades to
    read-only + merge instead of corrupting.
- *Builders.* `BoardEditor` save queue + conflict branch; 60 s heartbeat /
  8 min TTL; version CAS on every write.
- *Edge cases.* Netlify cold start slows one save → Retry covers it; double
  Retry doesn't double-write (version CAS).
- *Tests.* CAS mismatch → 409 + merge + rebase retry; heartbeat-after-expiry
  rejected.

### NF4 — Deployability & operations (Netlify + Atlas)

- *Stories.* As the maintainer, I want push-to-deploy with per-environment
  config so that prod is never touched by previews.
- *Acceptance.*
  - Push to `main` builds on Netlify; preview deploys use the dev/scratch
    database, never prod (`MONGODB_DB` per environment).
  - `/api/health` returns `{ ok: true }` unauthenticated post-deploy.
  - OAuth redirects registered for both prod (`*.netlify.app`) and localhost.
- *Builders.* `netlify.toml` + `.nvmrc`; env in Netlify UI; `.env.example`
  documents all vars.
- *Edge cases.* Missing env fails fast at boot with a clear message
  (`lib/env.ts`); preview URL leaked publicly still requires auth (or a
  valid share token) for every board.
- *Tests.* Env validation unit; health smoke; preview/prod DB separation
  config test.

---

## Traceability (feature → milestone → priority)

| Feature | Priority | Milestone | Stories |
|---|---|---|---|
| F1 sign-in (open signup) | Must | M1 | F1.1–F1.2 |
| F2 sessions & gating | Must | M1 | F2.1–F2.2 |
| F3 dashboard view | Must | M2 (shell M1) | F3.1–F3.2 |
| F4 create board | Must | M2 | F4.1 |
| F5 organize (rename/star/duplicate/delete) | Must | M2 | F5.1–F5.4 |
| F6 open & load | Must | M3 | F6.1–F6.2 |
| F7 lock | Must | M3 | F7.1–F7.3 |
| F8 read-only when locked | Must | M3 | F8.1 |
| F9 autosave | Must | M3 | F9.1 |
| F10 conflict merge | Should | M3 | F10.1 |
| F11 sharing | Must | M4 | F11.1–F11.2 |
| F12 (removed — allow-list admin) | — | — | tombstone, no stories |
| F13 errors & edges | Must | M3→M4 | F13.1–F13.2 |
| F14 deploy & health | Must | M1→all | ops checklist |
| F15 public link | Could | M4 | F15.1 |
| NF1–NF4 | Must/Should | all | cross-cutting |

*End of file.*
