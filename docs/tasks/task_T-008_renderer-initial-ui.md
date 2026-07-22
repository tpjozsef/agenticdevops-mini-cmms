# Task T-008 — Renderer initial UI: login, typed API client, core screens

> Authorities: `docs/functional-spec.md` § 7 (screen inventory — the scope
> contract), § 2 (role matrix — display-only in the renderer);
> `docs/api-contract.md` (every request/response shape — now including
> T-005/6/7); `docs/design-guide.md` (starter baseline — tokens over raw
> values); `docs/architecture-facts.md` (hardest rule: renderer is
> presentation-only; renderer→backend is direct HTTP, main stays
> lifecycle-only).
>
> **Depends on T-005/T-006/T-007 merged.** The API surface this consumes is
> what `docs/api-contract.md` documents after T-007 — read it fresh, not from
> this spec's memory.

## 1. Background

The backend now covers the full v1 reactive loop; nothing user-facing exists.
This task ships the first real UI: login against `/auth`, a typed API client
mirroring the contract doc, and the FS § 7 screens wired to live endpoints.
The renderer stays a pure presentation layer — every decision it renders
(role gates, legal transitions) is display-only UX over server enforcement.

## 2. What Already Exists (Do Not Rewrite)

- `src/main.ts` — Electron main, lifecycle-only, ctx isolation on. **Do not
  touch** (no IPC, no proxying — architecture-facts § Process boundaries).
- `src/preload.ts` — intentionally empty; stays that way.
- `src/renderer.tsx`, `src/App.tsx`, `src/index.css`, `src/App.test.tsx` —
  the scaffold shell; `App.tsx` is replaced, `renderer.tsx` stays as-is.
- Backend: the whole T-001–T-007 API. **The only backend change permitted is
  the CORS middleware below.**
- Tooling: vitest + @testing-library/react configured; eslint/tsc strict.

## 3. What to Build

### 3a. Backend enablement (the one server-side change): CORS

`backend/app/main.py`: add FastAPI `CORSMiddleware`; allowed origins from
`CMMESS_CORS_ORIGINS` (comma-separated, via `app/config.py` helper), default
`http://localhost:5173,http://127.0.0.1:5173` (the Vite dev origin).
`allow_credentials` stays **false** (bearer header, no cookies), allow all
methods/headers. Packaged-app (`file://`) origin handling is the packaging
task's problem — noted, not solved here. Document in `docs/api-contract.md`
(same commit, Rule 12).

### 3b. Typed API client — the TS leg of Rule 12 goes live

- `src/api/types.ts` — TypeScript mirrors of every Pydantic shape the UI
  consumes (`UserOut`, `LoginResponse`, `AssetOut`, `AssetDetailOut`,
  `DowntimeEventOut`, `WorkOrderSummaryOut`, `WorkOrderOut`,
  `WorkOrderDetailOut`, `TransitionOut`, request bodies). Field names/types
  match `docs/api-contract.md` exactly — string-literal unions for the
  StrEnums (`"up" | "down"`, roles, origins, priorities, statuses).
- `src/api/client.ts` — one fetch wrapper: base URL from
  `import.meta.env.VITE_CMMESS_API_URL ?? "http://127.0.0.1:8000"`; injects
  `Authorization: Bearer <token>`; JSON in/out; non-2xx → typed `ApiError`
  carrying status + parsed `detail` (the FS-Q1 409 pointer body and 409/403
  transition errors must surface their detail to the UI, not vanish into a
  generic toast). One exported function per endpoint, typed end to end.
- **From this commit on, `docs/api-contract.md`'s "TS leg: N/A" notes are
  replaced** with pointers to `src/api/types.ts` (Rule 12 — same commit).

### 3c. App shell + auth state

- Token + current `UserOut` in React state/context only — **in-memory; no
  localStorage/sessionStorage persistence; relaunch = re-login** (v1
  decision: simplest correct handling of an opaque token, recorded here).
- Any 401 from the client → clear auth state → login screen.
- Navigation: top-level view state in React (no routing library — **zero new
  npm dependencies** in this task; avoids TRAP-001 lockfile churn entirely).
- Shell layout + tokens per `docs/design-guide.md`: left sidebar (Assets ·
  Work Orders · New WO · user + logout footer), content right; tokens land in
  `src/index.css` `:root` exactly as the guide tables them.

### 3d. Screens (FS § 7 — all seven, at MVP depth)

1. **Login** — username/password → `POST /auth/login`; error text on the
   uniform 401; role comes from the response, never a picker.
2. **Asset browser** — `GET /assets` rendered as the path hierarchy (pure
   client-side tree built from `path` segments — this is presentation, not
   business logic); status pill per asset; "Register asset" form
   (path/display_name/description → `POST /assets`, surfacing 409/422
   details); retired hidden (no `include_retired` UI in v1 — decided).
3. **Asset detail** — status, downtime history with derived durations
   rendered `42m`/`1h 05m` style, WO history linking into WO detail; actions:
   **Report downtime** (→ FS-Q1 409 shows the pointer to the existing
   event/WO), **Mark back up** (manual ongoing events), **Create WO**
   (prefilled asset), **Edit / Retire** (manual assets only — hidden for
   `uns_discovered`, display-only mirror of the server rule).
4. **Work-order list** — `GET /work-orders` table (title, asset, origin,
   priority, status, age); filter controls for status/assignee/origin;
   **My work** = `assigned_to: me`; **Planner queue** = a canned
   `status: open` filter preset (Planner sees it as the default tab).
5. **Work-order detail** — full record + transitions audit trail + linked
   downtime event; action buttons per current status and role (plan form:
   assignee/window/priority — Planner only; start; complete with required
   notes; abandon with required note; cancel — Planner only). Buttons are
   shown/hidden for UX but every rejection (403/409) surfaces the server's
   detail — the server is the gate.
6. **Work-order create** — asset picker (from `GET /assets`), title,
   description, priority; → `POST /work-orders`.
7. *(Planner queue is the § 7.4 preset, not a screen — matching FS.)*

### 3e. Tests (vitest — pure logic + component smoke)

Unit-test the path→tree builder (multi-segment, shared prefixes, sort) and
the duration formatter; component tests: login renders and submits, WO detail
shows/hides actions by role+status fixture, client maps non-2xx to `ApiError`
with detail preserved. The renderer↔backend boundary is exercised for real in
the runtime test against a live backend (architecture-facts § Testing) — do
not mock the contract into triviality in lieu of that.

## 4. Acceptance Criteria

1. `npm run build` (tsc + package), `npm run lint`, `npm run test` all green;
   backend suite untouched and green; `pip install` + app import still work.
2. With a live backend: login as each role succeeds and shows role-correct
   chrome (Planner sees plan/cancel affordances; User doesn't); bad
   credentials show the uniform error; logout returns to login and the old
   token stops working.
3. Full reactive loop drivable end to end from the UI: register asset → see
   it in the tree (up) → report downtime → asset shows down + a seeded WO
   appears → start → complete (notes required) → asset still down until
   marked back up (FS § 4 independence, visible in the UI).
4. FS-Q1 in the UI: reporting downtime on a down asset shows the ongoing
   event/WO pointer from the 409 body — not a generic failure.
5. Every renderer↔backend shape the UI uses exists in `src/api/types.ts` and
   matches `docs/api-contract.md`; the contract doc's TS-leg notes point at
   the TS types in the same commit (Rule 12). No `any`-typed API surface, no
   hand-rolled shapes bypassing `client.ts`.
6. The renderer contains no business logic: no status derivation beyond
   rendering server-derived fields, no transition-legality logic beyond
   show/hide (server 403/409 details surface when a stale UI acts), no DB or
   MQTT access, no changes to `main.ts`/`preload.ts`.
7. CORS: the Vite dev origin can call the API; origins are configurable via
   `CMMESS_CORS_ORIGINS`; `allow_credentials` false.
8. All styling consumes `src/index.css` tokens per `docs/design-guide.md` —
   no hex literals in components; status/priority always color + text.
9. Zero new entries in `package.json` dependencies/devDependencies;
   `package-lock.json` unchanged.

## 5. Files to Modify

- `src/api/types.ts`, `src/api/client.ts` — NEW.
- `src/auth/` (context/provider), `src/components/`, `src/screens/` — NEW
  (agent's layout discretion within these directories).
- `src/App.tsx` — replaced (shell + nav + auth gating).
- `src/index.css` — design-guide tokens + component styles.
- `src/App.test.tsx` — updated; new test files alongside their subjects.
- `backend/app/main.py`, `backend/app/config.py` — CORS middleware + origins
  helper only.
- `docs/api-contract.md` — CORS note + TS-leg pointers, same commit.

Not touched: `src/main.ts`, `src/preload.ts`, `src/renderer.tsx`, all other
backend files, `package.json`/`package-lock.json`, migrations.

## 6. Coding-Agent Instructions

Read this spec file (`docs/tasks/task_T-008_renderer-initial-ui.md`) in full
before writing any code, then read `docs/api-contract.md` fresh — it, not
this spec, is the authority for every shape you type in `src/api/types.ts`.

Build the typed client (`src/api/types.ts` + `client.ts`, base URL via
`VITE_CMMESS_API_URL`, in-memory bearer token, `ApiError` preserving server
`detail`), the app shell with state-based navigation (no new npm
dependencies), and the FS § 7 screens wired live: asset tree with register/
edit/retire, asset detail with report-down/mark-up and the FS-Q1 409 pointer
surfaced, WO list with filters + planner-queue preset, WO detail with
role/status-appropriate transition actions, WO create. Add CORS middleware to
`backend/app/main.py` (origins from `CMMESS_CORS_ORIGINS`, default Vite dev
origins, `allow_credentials` false) — the only backend change. Style
exclusively with the `docs/design-guide.md` tokens landed in `src/index.css`.
Update `docs/api-contract.md` (CORS + TS-leg pointers) in the same commit
(Rule 12).

The coding agent changes no files outside the list in § 5.

Standing invariants: honor docs/architecture-facts.md and CLAUDE.md; the
renderer holds no business logic, DB, or MQTT/UNS access — presentation over
typed REST only; authorization is enforced server-side (renderer show/hide is
UX, never the gate); keep contract docs (Rule 12) and user-docs (Rule 18) in
the same commit; never read/write/delete data outside the app's own store;
build with npm run build when done, and lint + test green.
