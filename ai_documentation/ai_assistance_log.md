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

## 2026-09-20 — claude-002: diverging run (`lr=1.5`) crashes `POST /train`

- **Tool:** Claude Code (Claude Sonnet 5 for the fix, Claude Opus 5 for this entry)
- **Problem:** Training at `lr=0.01` worked, but the `lr=1.5` divergence demo made `POST /train` return 500 and Streamlit show an HTTP error. The Render log ended with `PGRST102: Empty or invalid json` while saving the run to Supabase.
- **Cause:** At `lr=1.5` the loss reaches `inf` in epoch 1 and `nan` after, so every metric and weight is `nan`. `json.dumps` emits the bare tokens `NaN`/`Infinity`, which are not valid JSON, so PostgREST rejected the insert body before Postgres saw it — hence a parse error rather than a column error. Nothing between the trainer and `db.insert_run()` checked for non-finite values. The response itself would also have failed to serialize.
- **Files changed:** `api/main.py` (+21/-1), `shared/schemas.py` (+20/-3), `tests/test_training.py` (+34), `ui/app.py` (+31/-14) — 88 insertions, 18 deletions. Plus `ai_documentation/ai_conversations/claude-002-diverging-training.md` and this log. `.env`, `api/db.py`, `api/training.py`, `db/migrations/001_init.sql` and the professor's tutorial files were not touched.
- **Solution:** `/train` now keeps the loss history it used to discard and runs `math.isfinite` over the loss curve, metrics and weights before touching Supabase. If anything is non-finite it returns HTTP 200 with `diverged: true`, `diverged_at_epoch`, a plain-language `message`, and `run_id`/`metrics`/`weights` as `null` — and skips `insert_run` entirely. `TrainResponse` made those three fields optional to allow that. The Streamlit Train tab branches on `diverged` and shows `st.error(message)` instead of dereferencing null metrics; `last_run_id` only updates on success. No metric is invented and no sentinel is substituted — the divergence is reported, not hidden.
- **Trade-off:** diverged runs are not stored at all, so they do not appear in Run History and cannot be used on the Predict tab. Storing them would need a migration making `mse`/`mae`/`r2` nullable plus a status column, applied by hand in the Supabase dashboard. Deferred as out of scope for "the smallest appropriate change"; noted as an available follow-up. Consequence: `TUTORIAL.md:231` ("Both runs land in" the runs table) is now inaccurate and was deliberately left unedited.
- **Tests:** two added to `tests/test_training.py` — `test_diverging_lr_is_reported_not_stored` (lr 1.5 → 200, `diverged` true, null metrics, `client._store["runs"] == {}`) and `test_converging_run_is_not_flagged_diverged` (lr 0.01 → not flagged, exactly one run stored, R² > 0.9). `pytest -q` gives 8 passed, 1 skipped, up from 6 passed, 1 skipped. The skip is the Supabase round-trip, which needs real credentials. Both learning rates were also reproduced directly before the fix: `lr=0.01` → MSE ≈ 4.13, R² ≈ 0.981; `lr=1.5` → loss `[inf, nan, nan, ...]`, all metrics `nan`.
- **What was verified:** the failure was reproduced locally and confirms the user's NaN/Infinity hypothesis; the new tests fail if a diverged run is ever persisted; the full suite passes; the diff is limited to the four files above; no secrets were read or printed (`.env` untouched this session, and git-ignored). **Not verified:** the fix against the deployed Render service or a real Supabase instance, and the Streamlit change in a browser — both need a redeploy. The `PGRST102` explanation is reasoned from the JSON spec and the observed `nan` values, not from a captured request body. My own review of the diff is pending before commit.
- **Status:** Uncommitted, pending review.
- **Full transcript:** [claude-002-diverging-training.md](ai_conversations/claude-002-diverging-training.md)

## 2026-09-20 — claude-003: README documents the three-cloud deployment

- **Tool:** Claude Code (Claude Opus 5)
- **Task:** Update `README.md` to describe the finished Streamlit Community Cloud UI, the FastAPI API on Render and the Supabase PostgreSQL database, and record the deployed endpoint tests.
- **Files changed:** `README.md` (+32/-11). Plus `ai_documentation/ai_conversations/claude-003-readme-documentation.md` and this log. No application code, `.env` or deployment settings were touched.
- **Changes:** New title and intro. The live deployment table now names each tier's platform. A new "Deployment and testing" section lists the checks: `/healthz` ok with `model_loader` and `supabase` true; `POST /datasets` HTTP 200; `lr=0.01` MSE ≈ 4.131, MAE ≈ 1.595, R² ≈ 0.981; `x=4` → ≈ 10.76; `lr=1.5` handled as divergence with no run saved.
- **Correction during the session:** the first draft said a diverged run "returns an error". It was checked against `api/main.py:107-118` and corrected: the API returns a normal response with `diverged: true` and a message.
- **What was verified:** the divergence wording matches the code, and `README.md` is the only changed file. The endpoint results came from my prompt; Claude did not call the deployed API. The public URLs are still `<STREAMLIT_URL>` / `<RENDER_API_URL>` placeholders, because my prompt still had the template text.
- **Status:** I committed the README change as `1f279a7` (33 insertions, 11 deletions; Claude's edit was +32/-11). The real URLs are still pending.
- **Full transcript:** [claude-003-readme-documentation.md](ai_conversations/claude-003-readme-documentation.md)

## 2026-09-20 — claude-004: model card documents the reference run

- **Tool:** Claude Code (Claude Opus 5)
- **Task:** Update `MODEL_CARD.md` with the reference dataset, training settings, metrics, a prediction check and the divergence limitation, keeping the existing structure.
- **Files changed:** `MODEL_CARD.md` (additions only, no lines removed). Plus `ai_documentation/ai_conversations/claude-004-model-card.md` and this log. No application code, `.env` or deployment settings were touched.
- **Changes:** Model details now lists `lr=0.01`, batch size 32, 100 epochs. Data lists slope 2.5, intercept 1.0, noise 2.0, 500 points. Metrics has a table (MSE ≈ 4.131, MAE ≈ 1.595, R² ≈ 0.981) and the `x=4` → ≈ 10.76 check. Limitations says `lr=1.5` diverges, and the API reports it without saving a run.
- **Assistant suggestions:** it labeled the metrics as held-out test-split results, based on the card's existing text. It also added a comparison with the true line's value (11.0 at `x=4`). It flagged both for me to confirm, and noted the Owner line is still a placeholder.
- **What was verified:** the divergence wording matches `api/main.py:109-118`; `MODEL_CARD.md` is the only changed file. The metrics and prediction came from my prompt; Claude did not train the model or call the API.
- **Status:** Uncommitted, pending review.
- **Full transcript:** [claude-004-model-card.md](ai_conversations/claude-004-model-card.md)
