# Task T-001 — Backend skeleton: FastAPI `/health` + Python tooling

## 1. Background

First code task of the project, deliberately sliced small to exercise the full development loop end to end (spec → coding agent → Cursor QA → PM read-verify → human runtime test → close-out). It stands up the Python/FastAPI backend skeleton only: one health endpoint, one passing test, and working lint/typecheck tooling — so the QA step has real checks to run. The renderer, CI workflow, database, auth, and MQTT are all explicitly out of scope and come in later tasks (renderer + CI is queued as T-002).

Authority docs consulted: `docs/architecture-facts.md` (all specs), `docs/contract-sync.md` (new REST endpoint = boundary change). `docs/api-contract.md` does not exist yet — this task creates it (see Rule 12 note below).

## 2. What Already Exists (Do Not Rewrite)

**No product code exists.** The repo holds only workflow docs (`docs/`), checklists (`checklists/`), agent configs (`CLAUDE.md`, `.cursor/`), and `README.md`. Do not modify any of these except the single new contract doc listed in Files to Modify. There is no `package.json`, no renderer, no CI workflow — do not create them.

## 3. What to Build

A minimal, tooling-clean FastAPI backend under `backend/`:

1. **`backend/app/main.py`** — a FastAPI app exposing `GET /health` returning HTTP 200 with body `{"status": "ok"}`. The response goes through a typed Pydantic response model (the renderer↔backend boundary is typed end to end — no hand-rolled dicts at the boundary; a `Literal["ok"]`-style field is fine). Include `backend/app/__init__.py` so `app` is a proper package.
2. **`backend/tests/test_health.py`** — one pytest test using FastAPI/Starlette's `TestClient` that calls `GET /health` and asserts status 200 and exact body `{"status": "ok"}`. Include `backend/tests/__init__.py`.
3. **`backend/requirements.txt`** — minimal: `fastapi`, `uvicorn`, `pytest`, `httpx` (required by `TestClient`), `ruff`, `mypy`. Loose lower-bound pins (`>=`) are fine. **No SQLAlchemy, Alembic, or MQTT libraries** — those arrive with their own tasks.
4. **`backend/pyproject.toml`** — `[tool.ruff]` and `[tool.mypy]` configuration such that, run from `backend/`, bare `ruff check .` and `mypy .` both exit clean over `app/` and `tests/`. Sensible strictness (mypy catching untyped defs in `app/` is the target); no packaging/build-system machinery needed.
5. **`docs/api-contract.md`** — create the contract doc with a short header (what the doc is: the authority for the renderer↔backend REST surface, per `docs/contract-sync.md`) and one entry documenting `GET /health` (path, method, auth: none, response model + example body). Keep it minimal; the PM will grow the doc's conventions later.
6. **`.gitignore`** (repo root) — minimal Python + Node ignores (`__pycache__/`, `*.pyc`, `.venv/`/`venv/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `node_modules/`, `dist/`, `out/`) so tooling artifacts never land in commits.

Target Python: 3.11+.

## 4. Acceptance Criteria

Verifiable properties, all run from `backend/` in a fresh virtualenv:

- [ ] `pip install -r requirements.txt` succeeds.
- [ ] `pytest` passes with at least one test that exercises `GET /health` and asserts both the 200 status and the exact body `{"status": "ok"}`.
- [ ] `ruff check .` exits 0.
- [ ] `mypy .` exits 0.
- [ ] The app imports and starts: `uvicorn app.main:app` boots without error and serves `/health` (the human will verify this at runtime; the agent verifies import + TestClient).
- [ ] The `/health` response is produced through a typed Pydantic model — no untyped dict at the boundary.
- [ ] `docs/api-contract.md` exists, documents `GET /health`, and lands **in the same commit** as the endpoint (Rule 12).
- [ ] No files outside the Files to Modify list are created or changed. In particular: no DB, no auth, no renderer files, no CI workflow, no changes to existing docs.

## 5. Files to Modify

All new files; nothing existing is edited:

- `backend/app/__init__.py` (new)
- `backend/app/main.py` (new)
- `backend/tests/__init__.py` (new)
- `backend/tests/test_health.py` (new)
- `backend/requirements.txt` (new)
- `backend/pyproject.toml` (new)
- `docs/api-contract.md` (new)
- `.gitignore` (new, repo root)

## 6. Coding-Agent Instructions

Read this spec file (`docs/tasks/task_T-001_backend-skeleton.md`) in full before writing any code.

Implement the backend skeleton exactly as scoped: a FastAPI app at `backend/app/main.py` with a single `GET /health` endpoint returning `{"status": "ok"}` through a typed Pydantic response model; one passing pytest in `backend/tests/` covering it via `TestClient`; `backend/requirements.txt` and `backend/pyproject.toml` configured so `ruff check .`, `mypy .`, and `pytest` all exit clean from `backend/`. Create `docs/api-contract.md` with the `GET /health` entry in the same commit as the endpoint (Rule 12); the TypeScript-type leg of contract-sync is **N/A for this task** — no renderer exists yet. Do not add SQLAlchemy, Alembic, MQTT, auth, renderer files, or a CI workflow.

Hard constraints decided by this spec:

- **Silencer decision:** no kept-but-unused symbols are expected in this task. If you believe you need to retain an unused import/export/function, **stop and flag it to the PM** — do not choose a silencer or delete the symbol on your own.
- **Build gate for this task:** `npm run build` does not exist yet (no `package.json`). The build gate is: from `backend/`, `pip install -r requirements.txt`, `ruff check .`, `mypy .`, and `pytest` all succeed, and `app.main` imports.
- **User-facing impact:** None — nothing user-visible exists yet. No user-doc changes required.

Standing invariants: honor docs/architecture-facts.md and CLAUDE.md; the renderer holds no business logic, DB, or MQTT/UNS access; authorization is enforced server-side; keep contract docs (Rule 12) and user-docs (Rule 18) in the same commit; migrations run on both SQLite and Postgres; never read/write/delete data outside the app's own store; build with npm run build when done (superseded for this task by the backend build gate above).
