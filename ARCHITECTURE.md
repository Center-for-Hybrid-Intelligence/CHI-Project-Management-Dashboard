# Architecture

Deep-dive reference for continuing development on the CHI Project Management
Dashboard. Read this alongside [README.md](README.md) (which covers running
and hosting). This doc covers how the code is put together, the data model,
the API contract, the frontend's internal structure, and known rough edges.

There is no build step and no test suite. This is a small, trusted-team
internal tool — three files do all the work:

```
db/init.js       schema + seed data + migrations   (~540 lines)
server.js        Express API + static file server  (~620 lines)
public/index.html  the entire frontend: HTML + CSS + vanilla JS SPA (~3550 lines)
```

---

## 1. Tech stack

- **Backend:** Node.js + Express 4, [better-sqlite3](https://github.com/WiseLibs/better-sqlite3) (synchronous SQLite driver — no async/await needed for DB calls).
- **Frontend:** a single static HTML file with an inline `<style>` block and an inline `<script>` block. No framework, no bundler, no npm build step. It's served as-is by Express.
- **Auth:** `express-basic-auth` — one shared username/password for the whole team (see [§5](#5-auth-model-and-editor-identity)).
- **Persistence:** one SQLite file at `data/dashboard.db` (WAL mode). `data/` is gitignored and Docker-mounted so it survives rebuilds.

---

## 2. Request flow

```
Browser ── GET /            ──▶ Express catch-all ──▶ public/index.html (+ <base> tag if BASE_PATH set)
Browser ── GET /api/data    ──▶ server.js           ──▶ SQLite (db/init.js handle) ──▶ full JSON blob
Browser ── PATCH/POST/... ──▶ server.js route       ──▶ SQLite write ──▶ JSON response
```

`db/init.js` runs once at process start: it opens/creates `data/dashboard.db`,
applies the schema (`CREATE TABLE IF NOT EXISTS ...`), runs a couple of
idempotent migrations, and — **only if the `projects` table is empty** —
seeds it with the demo data baked into that file. `server.js` imports the
already-open `db` handle and never touches schema/seed logic itself.

There is exactly one "load everything" endpoint (`GET /api/data`) and a
handful of narrow write endpoints. The frontend fetches `/api/data` once on
load into in-memory JS globals, then patches those globals + the DB in
lockstep on every edit (see [§4](#4-frontend-architecture)). There is no
websocket/polling — two people editing at once will not see each other's
changes until they reload.

---

## 3. Data model

SQLite, `db/init.js` is the single source of truth for the schema. Foreign
keys with `ON DELETE CASCADE` are used throughout — deleting a project or
person cleans up its related rows automatically.

| Table | Purpose | Key columns |
|---|---|---|
| `projects` | One row per project | `status` (green/amber/red/gray), `goal`, `week_status`, `week_h`/`mo_h` (hours this week/month), `proposed`, `dormant`, `color`, `tl_start`/`tl_end` (0–1 fractional timeline position for the Gantt view) |
| `project_team` | Many-to-many: person ↔ project ↔ role | `role` is free text but the frontend treats `'PO'` and `'PM'` specially; anything else renders as a plain contributor |
| `project_milestones` | Milestones/deadlines per project | `type`: `'m'` milestone or `'d'` deadline |
| `project_status_history` | One row per status change (audit trail for the weekly status trail UI) | written automatically whenever `PATCH /api/projects/:id` includes `status` |
| `people` | One row per team member | `role` (`po`/`pm`/`c`), `avail` (green/amber/red), `total_h`/`avail_h`, `inactive` (soft-delete flag), `last_checkin_week` |
| `people_schedule` | Weekly availability grid | 5 days × 3 slots (`0`=morning, `1`=early afternoon, `2`=late afternoon) per person |
| `people_projects` | Per-person, per-project hour allocation | `hours_this`, `hours_next`, `note`, `updated_by`/`updated_at` (attribution shown in the allocation grid UI) |
| `checkins` | Raw submitted check-in forms | `data` is a JSON blob of the whole form — kept for history, not re-read by the UI |
| `monthly_reviews` | One row per month | `data` is a JSON blob (see note below); `saved`/`saved_on` track whether a manager finalized it |
| `managers` | Names with manager privileges | just a name primary key — no separate `people.id` foreign key, matched by name string |
| `change_log` | Audit trail of significant admin actions | written by `logChange()` in `server.js` |

**Note on `monthly_reviews`:** the JSON content (project retrospectives,
people hours-balance) is *not* generated live from `projects`/`people` — it's
static content written once at seed time. `POST /api/reviews/:month/save`
only flips the `saved`/`saved_on` flag. There is currently no in-app flow to
author a new month's review content; someone would need to insert a row
directly (see `seedReviews()` in `db/init.js` for the expected shape).

### Field-naming convention: snake_case in DB, camelCase in API/frontend

The DB uses `snake_case` columns; the frontend uses `camelCase`. `server.js`
translates both ways manually:

- **Reads:** `loadProject(id)` / `loadPerson(row)` in `server.js` rename columns and reshape rows (e.g. flattening `project_team` into `po`/`pm`/`contrib` arrays, `people_schedule` rows into a `{Mon: [bool,bool,bool], ...}` object).
- **Writes:** each `PATCH` route has an `allowed` array (DB columns it's willing to update) and a `keyMap` object (`{ goalApproved: 'goal_approved', weekStatus: 'week_status', ... }`) to translate incoming camelCase body keys.

**This mapping is manual and not validated anywhere** — see the cookbook in
[§7](#7-cookbook-adding-a-new-editable-field) for the full checklist when
adding a field.

---

## 4. Frontend architecture

`public/index.html` is one file: `<style>` (roughly lines 11–1823) then
`<script>` (roughly lines 2007–3531). No modules, no virtual DOM. Every
"component" is a function that builds an HTML string (template literals)
and assigns it to some container's `.innerHTML`, or builds DOM nodes with
`createElement`. Re-rendering means "rebuild this container from scratch,"
not diffing.

### Global state

Plain top-level `let`/`const` variables, not one unified store:

- `PROJECTS` — array of project objects (mirrors `GET /api/data`'s `projects`).
- `PEOPLE` — array of person objects.
- `STATUS_HISTORY` — `{ [projectId]: [{wk, s, note}, ...] }` (top 4 weeks).
- `REVIEW_DATA` — `{ [month]: {label, saved, savedOn, projects:[...], people:[...]} }`.
- `MANAGERS` — array of manager names.
- `currentEditor` — the name currently selected in the "Editing as" dropdown (see §5).
- `manageMode` — bool, toggles the manager-only sidebar controls.

**Important:** the source file also contains ~800 lines of hardcoded literal
seed data for `PROJECTS`/`PEOPLE`/`STATUS_HISTORY`/`REVIEW_DATA` (used only
as a fallback if `fetch('api/data')` fails, and useful as a shape reference).
`init()` overwrites these globals with the real API response on success. If
you're adding a field, it's easiest to treat these seed literals purely as
documentation of the expected shape — don't rely on them being complete or
internally consistent (some seed names in that literal don't appear
elsewhere and are known-stale).

### Load / edit / re-render flow

1. `init()` runs on page load, fetches `GET /api/data`, overwrites the globals above, then calls each view's initial `buildXxx()` render.
2. Edits (e.g. changing a status, typing an hours value) mutate the in-memory object **immediately** (optimistic UI), fire off the corresponding `fetch(...)` (mostly not awaited/checked for `response.ok`), and call the relevant `buildXxx()`/`updateStatusCounts()` to refresh just that part of the DOM.
3. The only full-page reload is after restoring a DB snapshot (`location.reload()`).

Because writes are optimistic and mostly unchecked, a failed request leaves
the UI showing a change the DB never received, with no visible error and no
rollback — a reload will look like the edit "disappeared."

### Views

Top nav switches between 4 `.view` divs via `switchView()` (no URL routing —
reload always resets to Projects/Table):

| View | Sub-tabs | Driven by |
|---|---|---|
| **Projects** | Table, Timeline | `buildTable()` (status/team/hours table, expandable detail row via `toggleDet()`); `buildTimeline()` (Gantt-style bars from `tl_start`/`tl_end` + milestone ticks, zoomable via `zoomTl()`) |
| **Allocation** | People this week, Allocation grid, Monthly review | `buildPeopleLayer()` (per-person availability card); `buildAllocGrid()` (per-project contributor hours/notes editor, debounced save via `recEdit()`); `buildReview(month)` (retro + forward-look) |
| **Check-in** | Contributor / PM / PO forms | `buildContributorForm()` / `buildPMForm()` / `buildPOForm()`, all submitting through `submitCheckin(type)` |
| **Admin** (manager-only) | Change log, Snapshots | `loadChangelog()`; `loadSnapshots()` / `createSnapshot()` / `restoreSnapshot()` / `deleteSnapshot()` |

The **left sidebar** is always visible (not a tab): live status counts
(`updateStatusCounts()`), a check-in freshness list (`buildSidebar()`), and —
in manage mode — add/remove-person and manager-list controls.

### Key functions (structural map)

| Function | Role |
|---|---|
| `init()` | Bootstraps: fetch `/api/data`, populate globals, first paint |
| `buildTable()` / `toggleDet()` | Projects table + expandable detail row |
| `renderProjectPanel()` / `applyStatus()` | Status display + PATCH save |
| `renderTeamReadOnly()` / `renderTeamEditMode()` / `saveTeam()` | Team roster view/edit + PUT save |
| `renderStatusHistory()` / `getStreak()` | Weekly status trail + "N weeks amber" streak badge |
| `buildTimeline()` / `zoomTl()` | Gantt view |
| `buildSidebar()` / `recomputeManager()` / `toggleManageMode()` | Sidebar + manager-mode gating |
| `buildPeopleLayer()` | "People this week" cards |
| `buildAllocGrid()` / `recEdit()` | Allocation grid + debounced (600ms) hours/notes save |
| `buildAgenda()` | Pre-meeting agenda aggregated from leadership notes + celebrate/decision flags |
| `buildContributorForm()` / `buildPMForm()` / `buildPOForm()` / `submitCheckin()` | Check-in forms and submission |
| `buildReview()` / `saveReview()` | Monthly review render + "mark saved" |
| `buildAdminView()` / `loadChangelog()` / `loadSnapshots()` / `createSnapshot()` / `restoreSnapshot()` / `deleteSnapshot()` | Admin panel |
| `updateStatusCounts()` | Recomputes sidebar counts/totals from `PROJECTS` |
| `flashMsg()` | Bottom-right toast helper |

### CSS conventions

One `<style>` block, CSS variables on `:root` for the whole palette
(`--gn`/`--gb` green pair, `--am`/`--ab` amber, `--rd`/`--rb` red,
`--bl`/`--bb` blue, `--dm`/`--db` dim/gray, `--or`/`--ob` orange, plus
`--bg`/`--sf`/`--tx`/`--mu` for background/surface/text/muted, `--r`/`--rl`
for border radii). No dark mode. Class names are short/cryptic: `.tb`
topbar, `.sb` sidebar, `.ct` content, `.pn` project name, `.dtr` detail row,
`.ap*` allocation-panel, `.rv-*` review, `.ci*` check-in, `.tl*` timeline,
`.sg*` schedule-grid, `.av` avatar. Fonts: "DM Sans" (body), "DM Mono"
(numbers/timestamps), both via Google Fonts `<link>`.

---

## 5. Auth model and editor identity

Two separate, unrelated concepts:

1. **HTTP Basic Auth** (`server.js`, `express-basic-auth`) — one shared
   `DASHBOARD_USER`/`DASHBOARD_PASS` gates the whole app. It answers "is this
   someone from CHI," not "which person." Disabled entirely if
   `DASHBOARD_PASS` is unset (local dev default).

2. **"Editing as" dropdown** (`#edsel` in the topbar) — purely a client-side
   convention. Selecting a name sets `currentEditor`, which is sent as the
   `editor` field on writes that need attribution (status changes,
   check-ins, manager actions, etc.). It is **not persisted** across
   reloads and **not verified server-side** — anyone can pick any name and
   act as them. `isManager(name)` in `server.js` does a DB lookup against
   the `managers` table, so manager-gated *server* routes can't be bypassed
   by picking a manager's name in the UI without also... actually, they
   can — there's no session tying a browser to a real identity, so this is
   spoofable by design. Acceptable for a small trusted internal team; worth
   revisiting before exposing this outside CHI.

   **Server-side manager checks exist on:** add/remove manager, add/remove
   person, delete project, all `/api/admin/*` routes.
   **Missing on (UI-only gated):** `POST /api/projects` (propose project)
   and `PATCH /api/people/:id` (used by "reactivate person"). Anyone who can
   reach the API directly can call these without being a manager.

---

## 6. Deployment notes (see also README.md)

- `BASE_PATH` env var lets the app live under a sub-path behind a reverse
  proxy (e.g. Caddy `handle_path`). The catch-all route in `server.js`
  injects `<base href="${BASE_PATH}/">` into the served HTML so relative
  `fetch('api/...')` calls resolve correctly; static assets are served
  normally by `express.static`.
- **Snapshots** (`data/snapshots/*.db`) are full SQLite file copies made via
  `better-sqlite3`'s `db.backup()`, triggered from the Admin tab. Restore
  works by `ATTACH DATABASE`-ing the snapshot and copying every table over
  inside a transaction — see `POST /api/admin/snapshots/:id/restore` in
  `server.js`. This is the closest thing this project has to a backup/rollback
  mechanism; there's no automated/scheduled snapshotting.
- `data/` (DB file + snapshots) is gitignored and Docker-volume-mounted —
  never committed, never rebuilt away.

---

## 7. Cookbook: adding a new editable field

Because the camelCase↔snake_case mapping and the seed/shape "documentation"
are all manual, adding one new field to (say) `projects` touches ~5 places.
Checklist, using a hypothetical `priority` field as an example:

1. **Schema** (`db/init.js`): add the column to the `CREATE TABLE` statement, and add a `try { db.exec('ALTER TABLE projects ADD COLUMN priority ...') } catch(e) {}` migration line so existing databases pick it up on next boot.
2. **Seed data** (`db/init.js`, optional): add the field to the `PROJECTS` seed array and `insertProject` prepared statement if you want demo data to exercise it.
3. **Read path** (`server.js`, `loadProject()`): if the DB column name differs from the desired frontend key, rename it here (snake_case column → camelCase key) and `delete` the raw key, same pattern as `goal_approved` → `goalApproved`.
4. **Write path** (`server.js`, the relevant `PATCH` route): add the snake_case column name to that route's `allowed` array, and add a `keyMap` entry if the frontend key differs from the column name.
5. **Frontend read** (`public/index.html`): reference `p.priority` (or whatever camelCase key) in the relevant `buildXxx()`/`renderXxx()` render function.
6. **Frontend write** (`public/index.html`): add the UI control, and include the field in the `fetch(..., {method:'PATCH', body: JSON.stringify({priority: ..., editor: currentEditor})})` call for that action.

Nothing enforces this end-to-end wiring — a typo in any one of these steps
fails silently (the column is either never updated or never rendered, with
no error).

---

## 8. Known limitations / things to be aware of

These aren't blocking bugs in normal trusted-team use, but matter if you're
extending the app or considering wider exposure:

- **No HTML-escaping on user-entered text** rendered via `innerHTML` (project names, notes, goals, etc.) — a name or note containing HTML/script would execute in every viewer's browser. Low risk today (small trusted team, Basic-Auth-gated), but a real concern if input ever comes from a less-trusted source.
- **`currentEditor` is spoofable** and two of the manager-only actions (`POST /api/projects`, `PATCH /api/people/:id`) aren't manager-checked server-side (see §5).
- **No conflict detection** — concurrent edits from two browser tabs are last-write-wins with no warning; the frontend never re-fetches after a write, so a stale tab can silently overwrite someone else's newer change.
- **Fragile allocation↔project join in `buildAllocGrid()`**: contributors are matched to projects by a 10-character prefix substring match on project *name* (not by ID). Renaming a project, or two project names sharing a 10-char prefix, can silently break or cross-contaminate the allocation grid.
- **ISO week-number logic is duplicated** (not truly ISO-8601 — an approximation) in `server.js` (`weekLabel()`) and ~5 places in `index.html`. If you need to fix a week-boundary bug, grep for `864e5` and fix every occurrence.
- **Monthly review content is static**, not derived live from current project/people data — see the note in §3.
- **No automated tests, linting, or CI.** Verify changes by running the app locally and clicking through the affected view (see README's "Running locally" section).

---

## 9. Where to look for what

| I want to... | Look at |
|---|---|
| Understand the DB shape | `db/init.js`, top of file |
| Add/change an API endpoint | `server.js` — routes are grouped with `// ─── COMMENT ───` banners, roughly in the order listed in §3 |
| Change how a view looks/behaves | `public/index.html`, find the relevant `buildXxx()`/`renderXxx()` function (see table in §4) |
| Change styling | `public/index.html` `<style>` block, CSS variables at `:root` |
| Understand who can do what | §5 above, plus `isManager()` call sites in `server.js` |
| Restore from a bad state | Admin tab → Snapshots (or manually copy `data/dashboard.db` from a backup) |
