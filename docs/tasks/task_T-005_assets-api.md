# Task T-005 — Assets API (registry list/detail, manual register/edit/retire)

> Authorities: `docs/functional-spec.md` § 3 (assets & registry), § 2 (role matrix);
> `docs/data-model.md` (`assets`, `downtime_events`, `work_orders` tables);
> `docs/api-contract.md` (auth pattern + error semantics — this task extends it);
> `docs/architecture-facts.md` (derived vs. authoritative; DEC-005; DEC-008).

## 1. Background

First domain API slice (backend-first ordering, Architect 2026-07-22). The asset
registry is the root of the whole model: downtime events (T-006) and work orders
(T-007) both hang off assets. This task exposes the registry over REST: browse
(list + detail with **derived** up/down status and attached history), and the
manual-asset lifecycle — register, edit, retire (FS-Q7, DEC-008). No UNS code in
this task; `uns_discovered` assets exist only as rows a later ingestion task will
create (tests create them directly).

## 2. What Already Exists (Do Not Rewrite)

- `backend/app/models.py` — `Asset`, `DowntimeEvent`, `WorkOrder` (+ enums
  `AssetProvenance`, `DowntimeProducer`, `WorkOrderOrigin/Priority/Status`).
  The schema is complete for this task; **no migration is needed or wanted.**
- `backend/app/auth.py` — `require_user` / `require_planner` dependencies,
  `get_db`/`DbSession`, `UserOut`. Use these; do not invent a parallel auth path.
- `backend/app/db.py` / `config.py` — lazy engine + session factory; SQLite FK
  pragma already handled.
- `backend/tests/test_auth.py` — the fixture pattern to follow: tmp-path DB
  migrated via `upgrade_to_head`, env vars monkeypatched, `db.get_engine.cache_clear()`
  around each test, `TestClient(app)` context entry to run the lifespan.
- `backend/tests/test_migrations.py` — `upgrade_to_head` helper.
- `docs/api-contract.md` — auth section + uniform 401/403 semantics; extend this
  doc, don't restate its auth rules.

## 3. What to Build

A new router `backend/app/assets.py` (`APIRouter(prefix="/assets", tags=["assets"])`),
included in `app/main.py`. All endpoints require auth; per FS § 2 the asset
capabilities (browse, detail, register) belong to **both roles**, so every
endpoint below uses `require_user`. Edit/retire are also `require_user`
(FS § 3 lists them as asset-detail actions without a Planner gate; PM default,
flagged to the Architect).

### Pydantic models (in `assets.py`)

- `AssetOut`: `id`, `path`, `display_name`, `description`, `provenance`
  (`AssetProvenance`), `retired: bool`, `status: Literal["up", "down"]`
  (derived — see below), `created_at`, `updated_at`.
- `DowntimeEventOut`: `id`, `producer` (`DowntimeProducer`), `down_at`,
  `up_at: datetime | None`, `duration_seconds: float | None` (derived
  `up_at − down_at`; `None` while ongoing), `reported_by: int | None`,
  `ended_by: int | None`.
- `WorkOrderSummaryOut`: `id`, `origin`, `title`, `priority`, `status`,
  `created_at`. (T-007 owns the full WO surface; this is deliberately a summary.)
- `AssetDetailOut`: everything in `AssetOut` **plus**
  `downtime_history: list[DowntimeEventOut]` (newest first) and
  `work_orders: list[WorkOrderSummaryOut]` (newest first).
- `AssetCreate`: `path`, `display_name`, `description: str | None = None`.
- `AssetUpdate`: `display_name: str | None = None`, `description: str | None = None`.
  **`path` is deliberately absent from `AssetUpdate`** — path immutability
  (FS § 3) is enforced by the model having no such field, plus Pydantic
  `extra="forbid"` so a client sending `path` gets a 422, not a silent drop.

### Endpoints

1. **`GET /assets`** — list. Query param `include_retired: bool = false`;
   retired assets are excluded by default (FS-Q7: hidden from the browser).
   Returns `list[AssetOut]`, ordered by `path`. Flat list — the renderer builds
   the UNS-path hierarchy client-side; the backend does not return a tree.
   Derived `status` must not N+1: compute ongoing-downtime existence for the
   whole page in a bounded number of queries (e.g. one aggregate/EXISTS join,
   or one `IN`-clause query over the listed ids).
2. **`GET /assets/{asset_id}`** — detail, `AssetDetailOut`. **Reachable even
   when retired** (FS-Q7: hidden, not gone). 404 for unknown id.
3. **`POST /assets`** — manual registration, either role. Body `AssetCreate` →
   201 + `AssetOut`. Sets `provenance=manual`, `retired=false`. Path validation
   (server-side): after stripping surrounding whitespace, the path must be
   1–255 chars, contain no leading/trailing `/`, and split on `/` into
   segments that are all non-empty and non-whitespace — otherwise 422.
   Duplicate path (any provenance, retired or not) → **409** with a generic
   conflict detail. (One namespace — DEC-008's merge rule makes path collisions
   meaningful, so the API must surface them, not mangle the path to dodge them.)
4. **`PATCH /assets/{asset_id}`** — edit `display_name`/`description`.
   **Manual assets only:** `uns_discovered` → **409** (`detail` naming the
   provenance rule — UNS-discovered rows are a cache, DEC-008). Omitted fields
   unchanged; explicit `description: null` clears it. Returns `AssetOut`.
   404 unknown id. Editing a retired manual asset is allowed (it's not gone).
5. **`POST /assets/{asset_id}/retire`** — sets `retired=true`. **Manual assets
   only** (FS § 7 screen inventory: edit/retire are manual-only actions);
   `uns_discovered` → 409. Idempotent: retiring an already-retired asset is a
   200 no-op. Returns `AssetOut`. 404 unknown id. There is **no un-retire
   endpoint in v1** — decided absence, not an oversight.

### Derived status (architecture-facts § Derived vs. authoritative)

`status = "down"` iff the asset has an ongoing downtime event
(`up_at IS NULL`), else `"up"`. Always computed from `downtime_events` at read
time — never stored on the asset row, no caching column, no denormalized flag.

## 4. Acceptance Criteria

Properties, not shapes — each must be observably true via the API:

1. Every `/assets` endpoint 401s without a valid bearer token, and both a
   `user`-role and a `planner`-role session can perform every action in this
   task (register, edit, retire included).
2. A registered asset appears in `GET /assets` with `provenance="manual"` and
   `status="up"`; registering a second asset at the same path returns 409 and
   creates no row.
3. Malformed paths (`""`, `"/a/b"`, `"a/b/"`, `"a//b"`, `"a/ /b"`) are rejected
   422 with no row created.
4. An asset with an ongoing downtime event (row inserted with `up_at IS NULL`)
   reports `status="down"` in both list and detail; once `up_at` is set it
   reports `status="up"` — with no schema change and no stored status anywhere.
5. Asset detail returns downtime history with `duration_seconds` equal to
   `up_at − down_at` for closed events and `None` for ongoing ones, plus
   work-order summaries for WOs attached to that asset (rows inserted directly
   in the test).
6. A retired asset is absent from default `GET /assets`, present with
   `include_retired=true`, and its `GET /assets/{id}` detail (including
   history) still returns 200.
7. `PATCH` on a `uns_discovered` asset and `retire` on a `uns_discovered`
   asset both return 409 and change nothing; `PATCH` cannot change `path` by
   any request shape (unknown fields → 422).
8. `retire` is idempotent (second call 200, still retired).
9. `docs/api-contract.md` documents every new endpoint (auth level, request/
   response models, error codes) **in the same commit** (Rule 12). TS leg
   remains N/A (no renderer client yet — T-008 adds it).
10. Full suite green (`pytest`), ruff + mypy strict clean, existing tests
    untouched and passing. All tests use tmp-path DBs only.

## 5. Files to Modify

- `backend/app/assets.py` — **new**: router + Pydantic models + derivation.
- `backend/app/main.py` — include the assets router (one import + one line).
- `backend/tests/test_assets.py` — **new**: integration tests per the ACs,
  following the `test_auth.py` fixture pattern.
- `docs/api-contract.md` — new `## Assets` section, same commit.

Nothing else. No migration, no model changes, no renderer files, no changes to
`auth.py`/`db.py`/`config.py`.

## 6. Coding-Agent Instructions

Read this spec file (`docs/tasks/task_T-005_assets-api.md`) in full before
writing any code.

Build `backend/app/assets.py`: five endpoints (list with `include_retired`,
detail with derived status + downtime/WO history, register, edit, retire) using
the existing `require_user` dependency and the existing session/engine plumbing;
include the router in `main.py`. Status and durations are **derived from
`downtime_events` at read time — never stored**; the list endpoint must compute
status without an N+1 query pattern. Manual-only rules: `PATCH`/retire 409 on
`uns_discovered`; `AssetUpdate` has no `path` field and uses `extra="forbid"`.
Update `docs/api-contract.md` with the new endpoints in the same commit
(Rule 12). Tests go in `backend/tests/test_assets.py` on tmp-path DBs following
the `test_auth.py` fixture pattern, inserting `uns_discovered` assets, downtime
events, and work orders directly via the ORM where the API can't create them yet.

The coding agent changes no files outside the list in § 5.

Standing invariants: honor docs/architecture-facts.md and CLAUDE.md; the
renderer holds no business logic, DB, or MQTT/UNS access; authorization is
enforced server-side; keep contract docs (Rule 12) and user-docs (Rule 18) in
the same commit; migrations run on both SQLite and Postgres; never
read/write/delete data outside the app's own store; build with npm run build
when done (backend-only task: ensure `pip install -r requirements.txt`
succeeds and the FastAPI app imports and starts).
