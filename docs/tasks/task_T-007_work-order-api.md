# Task T-007 — Work-order API (state machine, planning, audit trail)

> Authorities: `docs/functional-spec.md` §§ 2, 5, 6 (role matrix, state
> machine, planning); `docs/data-model.md` (`work_orders`,
> `work_order_transitions`); `docs/api-contract.md`;
> `docs/architecture-facts.md` (DEC-005). FS § 5's transition table is the
> behavioral contract this task implements verbatim.
>
> **Depends on T-005 and T-006 being merged.** Build on `main` as it stands
> after both; reuse their response models.

## 1. Background

T-006 seeds work orders; this task makes them workable: list/filter, detail
with full audit trail, direct manual creation (origin `manual`), the FS § 5
state machine with server-side role gates, and Planner planning fields. Every
status change writes a `work_order_transitions` audit row. FS-Q8 UNS publishing
of transitions is **not** in this task (lands with the UNS/MQTT task) — but the
transition write path should be a single choke point so publishing can hook it
later without hunting.

## 2. What Already Exists (Do Not Rewrite)

- `backend/app/models.py` — `WorkOrder`, `WorkOrderTransition`, all enums.
  **No migration; no model changes.** There is deliberately no `executor`
  column — see § 3 executor rule.
- `backend/app/assets.py` (T-005) — `WorkOrderSummaryOut`, `DowntimeEventOut`.
- `backend/app/downtime.py` (T-006) — seeding creates WOs with status `open`;
  seeding writes **no** transition row (creation is not a transition), and this
  task must not retrofit one.
- `backend/app/auth.py` — `require_user`, `require_planner`, `DbSession`.
- Existing test fixture pattern (`test_auth.py` → `test_assets.py` →
  `test_downtime.py`).

## 3. What to Build

A new router `backend/app/work_orders.py` (`prefix="/work-orders"`), included
in `app/main.py`.

### Executor rule (PM decision, flagged to the Architect)

The FS gates several transitions on "the executor" but the schema has no
executor column — and needs none: **starting a WO sets `assigned_to` to the
starter** (a self-serve start on an Open WO claims it; a Planned WO's assignee
already matches). While a WO is `in_progress`, the executor **is**
`assigned_to`. Abandon back to Open clears `assigned_to` (the work returns to
the unclaimed queue); abandon back to Planned keeps it.

### Read + create endpoints

1. **`GET /work-orders`** — `require_user`. Optional exact-match query params:
   `status`, `asset_id`, `assigned_to`, `origin`, `priority` (combinable,
   ANDed). Ordered `created_at` desc. Returns `list[WorkOrderOut]`.
   The Planner queue (FS § 6) is `status=open` — a canned filter, not an
   endpoint. "My work" is `assigned_to=<own id>` — client-side concern.
2. **`GET /work-orders/{id}`** — `require_user`. `WorkOrderDetailOut`: all
   `WorkOrderOut` fields + `downtime_event: DowntimeEventOut | None` (reused
   from T-005) + `transitions: list[TransitionOut]` (`from_status`,
   `to_status`, `at`, `by_user`, `note`; chronological). 404 unknown.
3. **`POST /work-orders`** — `require_user` (either role). Body
   `WorkOrderCreate`: `asset_id`, `title`, `description?`,
   `priority?` (default `medium` — FS-Q6; settable at creation by either
   role). Sets origin `manual`, no event, status `open`, `created_by` =
   current user. 201 → `WorkOrderOut`. 404 unknown asset; 409 retired asset
   (PM default, same rule as T-006).
4. **`PATCH /work-orders/{id}`** — edit before work starts (FS § 2 last row).
   Body: `title?`, `description?`. Allowed while status is `open`/`planned`,
   by the **creator or any Planner**; otherwise 403 (wrong person) / 409
   (work already started or terminal). Priority is **not** in this PATCH —
   post-creation priority changes are Planner work and live in `plan` (FS § 5:
   "adjustable by Planners").

### Transition endpoints — one per intent (Rule 17), each writing an audit row

Every successful transition: update `status`, write one
`work_order_transitions` row (`from_status`, `to_status`, `at` = now UTC,
`by_user` = current user, `note` where given) **in the same transaction**, via
a single shared helper (the future FS-Q8 publish hook point). Illegal
current-state → **409** naming the current status; wrong person/role → **403**.
All return the updated `WorkOrderDetailOut`.

5. **`POST /work-orders/{id}/plan`** — `require_planner`. Body (all optional,
   at least one required): `assigned_to?`, `scheduled_start?`,
   `expected_duration_minutes?`, `priority?`. From `open`: sets fields, status
   → `planned` (transition row). On an already-`planned` WO: re-plan — update
   fields, status unchanged, **no** transition row (rows record status
   changes only). Any other status → 409. `assigned_to` must reference an
   existing active user → 422 otherwise.
6. **`POST /work-orders/{id}/start`** — `require_user`. From `open`: anyone,
   self-serve (the 3am case) — sets `assigned_to` = starter. From `planned`:
   **assignee only** (403 for anyone else, Planners included — FS § 2 says
   "assignee only" in both role columns). Status → `in_progress`.
7. **`POST /work-orders/{id}/complete`** — `require_user`. Executor only
   (`assigned_to` == current user; 403 otherwise, Planners included — FS § 2
   puts "executor" in both columns). Body: `completion_notes` — required,
   non-empty after strip (422). From `in_progress` only. Status → `completed`;
   notes stored on the WO.
8. **`POST /work-orders/{id}/abandon`** — `require_user`. Executor **or any
   Planner** (FS-Q4). Body: `note` — required, non-empty (422). From
   `in_progress` only. Target state: the `from_status` of the WO's most
   recent transition **into** `in_progress` (`planned` if it was planned,
   else `open`) — FS § 5 "moves back". Back to `open` additionally clears
   `assigned_to`; back to `planned` keeps it. The note lands on the
   transition row.
9. **`POST /work-orders/{id}/cancel`** — `require_planner` (FS-Q4). From any
   non-terminal status; terminal → 409. Body: `note?` (optional). Status →
   `cancelled`.

### Explicit independence (FS § 4)

Completing or cancelling a WO never ends its downtime event — no code path in
this task touches `downtime_events` rows.

## 4. Acceptance Criteria

1. All endpoints 401 without a token. Role gates hold server-side: a `user`
   session gets 403 from `plan` and `cancel`; a planner who is not the
   assignee gets 403 starting a `planned` WO and 403 completing another's
   `in_progress` WO; a non-creator `user` gets 403 on PATCH.
2. Direct creation yields origin `manual`, no linked event, status `open`,
   `created_by` set; priority defaults `medium` and is settable at creation
   by a plain `user`.
3. List filters (`status`, `asset_id`, `assigned_to`, `origin`, `priority`)
   each narrow correctly and combine; `status=open` yields the Planner queue.
4. The full lifecycle works and is audited: seed/create → plan → start →
   complete produces exactly the matching transition rows in order (with
   `by_user` and `at` populated), and detail returns them chronologically.
   Re-planning a `planned` WO updates fields without adding a row.
5. Self-serve path: any user starts an `open` WO directly (Open →
   In Progress), `assigned_to` becomes the starter, and only that user can
   then complete it; completion without non-empty notes is a 422 and changes
   nothing.
6. Abandon returns a planned-then-started WO to `planned` (assignment kept)
   and a self-serve-started WO to `open` (assignment cleared), each writing a
   transition row carrying the required note; the WO is then startable again.
7. Cancel works from `open`, `planned`, and `in_progress` for a planner,
   409s on `completed`/`cancelled`, and 403s for a `user`.
8. Illegal state moves (start a `completed` WO, complete an `open` WO,
   abandon a `planned` WO, plan an `in_progress` WO) all 409 naming the
   current status, with no status change and no audit row.
9. Completing a WO whose downtime event is still ongoing leaves the event
   ongoing (FS § 4 independence, proven in a test).
10. `docs/api-contract.md` documents every endpoint, gate, and error in the
    same commit (Rule 12). TS leg N/A until T-008.
11. Full suite green, ruff + mypy strict clean, tmp-path DBs only, existing
    tests untouched.

## 5. Files to Modify

- `backend/app/work_orders.py` — **new**: router, models, the single
  transition helper.
- `backend/app/main.py` — include the router.
- `backend/tests/test_work_orders.py` — **new**, per the ACs.
- `docs/api-contract.md` — new `## Work orders` section, same commit.

Nothing else — no migration, no model changes, no edits to
`assets.py`/`downtime.py` beyond importing their response models.

## 6. Coding-Agent Instructions

Read this spec file (`docs/tasks/task_T-007_work-order-api.md`) in full before
writing any code.

Build `backend/app/work_orders.py`: list/detail/create/PATCH plus five
intent-named transition endpoints (`plan`, `start`, `complete`, `abandon`,
`cancel`) implementing FS § 5's table exactly, with every status change writing
a `work_order_transitions` row through one shared helper in the same
transaction. Executor rule: starting sets `assigned_to` to the starter;
executor-gated actions check `assigned_to`; abandon targets the `from_status`
of the latest transition into `in_progress` and clears assignment only when
returning to `open`. Wrong person → 403, wrong state → 409 (naming the current
status). Reuse T-005/T-006 response models; update `docs/api-contract.md` in
the same commit (Rule 12). Tests in `backend/tests/test_work_orders.py` on
tmp-path DBs following the existing fixture pattern.

The coding agent changes no files outside the list in § 5.

Standing invariants: honor docs/architecture-facts.md and CLAUDE.md; the
renderer holds no business logic, DB, or MQTT/UNS access; authorization is
enforced server-side; keep contract docs (Rule 12) and user-docs (Rule 18) in
the same commit; migrations run on both SQLite and Postgres; never
read/write/delete data outside the app's own store; backend-only task — ensure
`pip install -r requirements.txt` succeeds and the FastAPI app imports and
starts.
