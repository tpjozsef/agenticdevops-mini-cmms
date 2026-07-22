# CMMess — Completed Development

> The full history of completed work. **Read the recent entries before assuming something isn't built.** Log an entry **only after reading the actual output files** — it records what was *built*, not what was planned.

## How to use this log

- **Entries are immutable once written**, except to add/update the "verified by human" line. A future reader who finds an older entry restating a mechanism should **not** "clean it up" — that's drift in the wrong direction.
- Newest entries at the top of `## Log`.
- When this file gets large, archive by release into `docs/archive/completed_development_<release>.md` and keep only recent entries live.

## Per-entry convention

The entry shape — header block, the required body sections in order, the length caps, and the anti-patterns — is owned by **`checklists/close-out.checklist.md`**. Follow it there.

The one line worth repeating because it's mandatory and cheap: every entry carries a **`User-facing impact:`** line, never omitted. `None.` is a valid, considered answer.

## Log

### T-001 — Backend skeleton: FastAPI `GET /health`, test, tooling; API contract doc seeded

**Date:** 2026-07-22
**Spec:** `docs/tasks/task_T-001_backend-skeleton.md`
**Verified by human:** ✅ 2026-07-22 — booted `uvicorn app.main:app`; live `GET /health` returned 200 `{"status":"ok"}`

**What was built.** The Python/FastAPI backend skeleton (commit `0a3e2a2`): `backend/app/main.py` exposes `GET /health` through a typed Pydantic `HealthResponse` (`status: Literal["ok"]`) — the typed-boundary invariant honored from the very first endpoint. One pytest exercises it via `TestClient`, asserting 200 and the exact body. `backend/requirements.txt` holds exactly six deps (fastapi, uvicorn, pytest, httpx, ruff, mypy — no SQLAlchemy/Alembic/MQTT); `backend/pyproject.toml` configures ruff (E/F/I/W) and mypy strict (py311) so bare `ruff check .`, `mypy .`, and `pytest` all exit clean from `backend/`. The commit also seeds `docs/api-contract.md` with the `/health` entry (Rule 12, same commit; TypeScript leg N/A until the renderer lands) and a root `.gitignore`. Cursor QA: PASS on all checks. Known non-blocking: pytest surfaces a Starlette-internal deprecation warning — revisit at the next dependency bump.

**Files touched.** All NEW:
- `backend/app/__init__.py`
- `backend/app/main.py`
- `backend/tests/__init__.py`
- `backend/tests/test_health.py`
- `backend/requirements.txt`
- `backend/pyproject.toml`
- `docs/api-contract.md`
- `.gitignore`

**Deviations from spec.** One, procedural: committed directly to `main` rather than branch→PR — acceptable this once because no CI exists yet to gate a PR (that lands with T-002); branch→PR resumes from T-002 onward. Code content: none.

**Architectural impact.** None — first instance of the typed renderer↔backend boundary pattern per `docs/architecture-facts.md`; `docs/api-contract.md` now live (announced in `docs/authority-docs-by-area.md`'s terms as the REST-boundary authority).

**User-facing impact.** None. No user-visible surface exists yet; no user-doc changes required.
