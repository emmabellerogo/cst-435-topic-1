# AI Assistance Log

## 2026-09-20 — claude-001: FastAPI not loading root `.env`

- **Tool:** Claude Code (Claude Sonnet 5)
- **Problem:** `/healthz` returned `model_loader=true` but `supabase=false`, although `python -m db.seed` worked with the same local `.env`.
- **Cause:** `db/seed.py` calls `load_dotenv()`; the API tier never did. `get_client()` raised `KeyError` on `SUPABASE_URL`, and `ping()` swallowed it and returned `False`.
- **Files changed:** `api/db.py` (+7 lines). Also added `ai_documentation/ai_conversations/claude-001-api-env.md` and this log. `.env`, `TUTORIAL.md` and the API design were not touched.
- **Solution:** In `api/db.py`, `load_dotenv(Path(__file__).resolve().parent.parent / ".env")` at import time. The path is file-relative, so it works from any working directory. It does not override real environment variables (Render) and is a no-op if `.env` is missing. `python-dotenv` was already in `api/requirements.txt`.
- **Tests:** `pytest -q` gave 6 passed, 1 skipped. `GET /healthz` via FastAPI `TestClient` from the repo root returned `200 {'status': 'ok', 'model_loader': True, 'supabase': True}`.
- **What was verified:** The health check flipped to `supabase: true` after the change, and the existing tests still pass. The diff is limited to `api/db.py`, and no secrets were printed (`.env` unread, and git-ignored). Not yet verified: running uvicorn from another directory, and behavior on Render. My own review of the diff is pending before commit.
- **Status:** Uncommitted, pending review.
- **Full transcript:** [claude-001-api-env.md](ai_conversations/claude-001-api-env.md)
