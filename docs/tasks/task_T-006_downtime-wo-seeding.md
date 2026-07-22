# Task T-006 — Downtime events API + work-order seeding

> Authorities: `docs/functional-spec.md` § 4 (downtime events), § 2 (role
> matrix), § 5 (WO fields/origins); `docs/data-model.md` (`downtime_events`,
> `work_orders`, FS-Q1 partial unique index, origin↔event pairing CHECK);
> `docs/api-contract.md`; `docs/architecture-facts.md` (derived vs.
> authoritative; DEC-005).
>
> **Depends on T-005 (Assets API) being merged.** Build on `main` as it stands
> after T-005 — reuse its response models; do not duplicate them.

## 1. Background

The event→WO pipeline is the product's core primitive: a downtime event seeds a
work order at the moment the down transition is recorded (FS § 4). This task
implements the **manual producer** end to end — report down / mark up over REST
— and the seeding logic itself, written so the future UNS ingestion task can
reuse it with `producer="uns"` and no HTTP request. No MQTT code in this task.

## 2. What Already Exists (Do Not Rewrite)

- `backend/app/models.py` — `DowntimeEvent`, `WorkOrder`, enums. The FS-Q1
  partial unique index (`uq_downtime_events_ongoing_per_asset`) and the
  origin↔event pairing CHECK are **already DB-enforced** — the service layer
  adds the friendly rejection, the DB is the race backstop. **No migration.**
- `backend/app/assets.py` (T-005) — `DowntimeEventOut` (with derived
  `duration_seconds`) and `WorkOrderSummaryOut`. **Import and reuse these** in
  the new router's response models; do not redefine parallel shapes.
- `backend/app/auth.py` — `require_user`, `DbSession`/`get_db`.
- `backend/tests/test_auth.py` / T-005's `test_assets.py` — fixture pattern
  (tmp-path DB via `upgrade_to_head`, env monkeypatch, cache_clear, lifespan
  via `TestClient(app)` context).
- `docs/api-contract.md` — extend, don't restate.

## 3. What to Build

A new router `backend/app/downtime.py`, included in `app/main.py`. Both
endpoints `require_user` (FS § 2: report and end are ✓ for both roles).

### Seeding service (the reusable core)

A plain function in `downtime.py` (module-level, not request-coupled), e.g.
`record_downtime(db, asset, producer, reported_by) -> tuple[DowntimeEvent, WorkOrder]`:

- Creates the downtime event (`down_at` = server now, UTC; `up_at` null) **and**
  its seeded work order **in the same transaction** — both or neither.
- The WO: origin `manual_downtime` when producer is `manual`, `uns_downtime`
  when `uns` (FS § 4); `downtime_event_id` linked; title
  `f"Downtime — {asset.path}"` (deterministic); description null; priority
  `medium`; status `open`; `created_by` = the reporting user for manual,
  null (system) for UNS (FS § 5).
- **FS-Q1 handling:** check for an ongoing event first and reject cleanly; on
  the race where two requests pass the check, the partial unique index raises
  `IntegrityError` — roll back, re-query the ongoing event, and reject the
  same way. Under no interleaving do two ongoing events or an orphaned
  half-pair (event without WO / WO without event) exist.
- Callable without any HTTP machinery — the UNS ingestion task will call it
  directly from the MQTT client.

### Endpoints

1. **`POST /assets/{asset_id}/downtime-events`** — report down (either role).
   No request body fields in v1 — `down_at` is always server time (no
   backdating; decided absence). Responses:
   - `201` → `{"event": DowntimeEventOut, "work_order": WorkOrderSummaryOut}`.
   - `404` unknown asset.
   - `409` retired asset (no new activity on retired assets; PM default).
   - **`409` FS-Q1 rejection** with a structured pointer body:
     `detail = {"message": "asset already has an ongoing downtime event",
     "ongoing_event_id": <id>, "work_order_id": <id | null>}` — the FS requires
     the rejection to point at the ongoing event **and its work order**
     (`work_order_id` null only in the theoretical case no WO row links to the
     event).
2. **`POST /downtime-events/{event_id}/end`** — mark the asset back up (either
   role). Sets `up_at` = server now, `ended_by` = current user. Responses:
   - `200` → `DowntimeEventOut` (now with `duration_seconds` set).
   - `404` unknown event.
   - `409` producer is `uns` — FS § 2: people end **manual** events only; UNS
     events end via the UNS up-signal (later task).
   - `409` already ended (`up_at` set) — not idempotent, because `ended_by`
     attribution must not be silently rewritten.

### Explicit independence (FS § 4)

Ending a downtime event does **not** touch its work order's status, and no
work-order code path in this task ends events. The event log records what the
asset did; the WO records what people did.

## 4. Acceptance Criteria

1. Both endpoints 401 without a token; both a `user` and a `planner` session
   can report and end.
2. Reporting downtime on an up asset creates exactly one event and one WO
   atomically: origin `manual_downtime`, linked `downtime_event_id`, title
   `Downtime — {path}`, priority `medium`, status `open`, `created_by` = the
   reporter; the asset's derived status (T-005 endpoints) flips to `"down"`.
3. A second down-report while ongoing returns 409 whose body carries the
   ongoing event id and its WO id; no rows are created. After the event is
   ended, a new report succeeds and seeds a **new** WO even though the prior
   WO is still open (FS-Q2).
4. The seeding function called with `producer="uns"` (direct call in a test —
   no HTTP) produces origin `uns_downtime` and `created_by` null, proving the
   UNS reuse path.
5. Ending sets `up_at`/`ended_by`, flips derived asset status back to `"up"`,
   and leaves the linked WO's status untouched; ending a `uns`-producer event
   or an already-ended event returns 409 and changes nothing.
6. An `IntegrityError` race on the partial unique index surfaces as the same
   409 pointer response, not a 500 (exercise via two pre-checked inserts or a
   monkeypatched check).
7. `docs/api-contract.md` documents both endpoints, the FS-Q1 409 body shape
   included, in the same commit (Rule 12). TS leg N/A until T-008.
8. Full suite green, ruff + mypy strict clean, tmp-path DBs only, existing
   tests untouched.

## 5. Files to Modify

- `backend/app/downtime.py` — **new**: router + seeding service + response
  models (importing `DowntimeEventOut`/`WorkOrderSummaryOut` from
  `app.assets`).
- `backend/app/main.py` — include the downtime router.
- `backend/tests/test_downtime.py` — **new**, per the ACs.
- `docs/api-contract.md` — new `## Downtime events` section, same commit.

Nothing else — no migration, no model changes, no edits to `assets.py` beyond
imports of it.

## 6. Coding-Agent Instructions

Read this spec file (`docs/tasks/task_T-006_downtime-wo-seeding.md`) in full
before writing any code.

Build `backend/app/downtime.py`: a reusable `record_downtime(...)` service that
atomically creates event + seeded WO (FS-Q1 rejection with the structured 409
pointer body, IntegrityError race handled), plus two `require_user` endpoints —
report down on an asset, end a manual event — and include the router in
`main.py`. Reuse T-005's `DowntimeEventOut`/`WorkOrderSummaryOut` — no
duplicate shapes. Ending an event never changes WO status. Update
`docs/api-contract.md` in the same commit (Rule 12). Tests in
`backend/tests/test_downtime.py` on tmp-path DBs following the existing
fixture pattern.

The coding agent changes no files outside the list in § 5.

Standing invariants: honor docs/architecture-facts.md and CLAUDE.md; the
renderer holds no business logic, DB, or MQTT/UNS access; authorization is
enforced server-side; keep contract docs (Rule 12) and user-docs (Rule 18) in
the same commit; migrations run on both SQLite and Postgres; never
read/write/delete data outside the app's own store; backend-only task — ensure
`pip install -r requirements.txt` succeeds and the FastAPI app imports and
starts.
