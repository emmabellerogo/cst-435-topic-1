# claude-002: diverging training run (`lr=1.5`) crashes `POST /train` with `PGRST102`

- **Date:** 2026-09-20
- **Tool:** Claude Code (VS Code extension), model Claude Sonnet 5 for the debugging session; Claude Opus 5 for this documentation turn
- **Branch:** `main` (working tree clean at start)
- **Outcome:** `POST /train` now detects a diverged run, returns HTTP 200 with `diverged: true` instead of a 500, and stores nothing. Four files changed, two tests added. Nothing committed or pushed (awaiting user review).

## Redaction and scope notes

- No secret values appear in this file. `.env` was not read or printed in this session. No API keys, passwords, connection strings or private keys are included.
- The assistant's hidden reasoning and the harness system prompt are not included. This is the visible conversation only: user prompts, assistant messages, and the tool calls with their results.
- Local absolute paths are shortened to `<repo>` = `/Users/emmarogoveanu/Desktop/school/search engines & data mines/projects/cst-435-topic-1`.
- The Render log excerpt below is quoted from the user's prompt. The assistant did not have access to the Render dashboard or the deployed logs; it reproduced the failure locally instead.

---

## Turn 1: User

> Hey Claude, I need help debugging one issue in my deployed Regress-It app.
>
> Training works normally with a learning rate of 0.01, but when I use 1.5 to demonstrate divergence, Streamlit shows an HTTP error. The Render logs show that POST /train returns a 500 error, ending with:
>
> PGRST102: Empty or invalid json
>
> The error happens while saving the training run to Supabase. I think the unstable training is producing NaN or Infinity values that Supabase cannot store.
>
> Please inspect api/training.py, api/main.py, api/db.py, shared/schemas.py, and the database migration. Make the smallest appropriate change so the app handles a diverging run gracefully and honestly instead of crashing. Do not fake the metrics or hide the fact that training diverged. Keep the successful 0.01 behavior working.
>
> Please add or update a test for this case if appropriate. Do not touch .env, expose secrets, commit, or push anything. Show me the diff and explain how I should test both a successful run and a diverging run.

## Turn 1: Assistant (investigation)

### Tool call 1: Bash — list tracked files and size the source

Command: `git ls-files | grep -v node_modules | head -80 && wc -l api/*.py shared/*.py`

Result (abridged): the tracked tree, plus

```
       0 api/__init__.py
     151 api/db.py
     165 api/main.py
      98 api/training.py
       0 shared/__init__.py
      32 shared/data.py
     108 shared/schemas.py
     554 total
```

### Tool call 2: Bash — read the five files named in the prompt, plus the tests

Command: `cat api/training.py api/main.py api/db.py shared/schemas.py db/migrations/001_init.sql tests/test_training.py tests/conftest.py`

The relevant findings:

**`api/training.py`** returns `(metrics, weights, loss_history)` straight out of torch with no finiteness check:

```python
    weights = {
        "slope": float(model.weight.item()),
        "intercept": float(model.bias.item()),
    }
    return metrics, weights, loss_history
```

**`api/main.py`** discarded the loss history and passed the metrics directly into the database insert:

```python
    metrics, weights, _loss = train_linear_regression(...)
    run = db.insert_run(
        dataset_id=req.dataset_id,
        ...
        mse=metrics["mse"],
        mae=metrics["mae"],
        r2=metrics["r2"],
        weights_json=weights,
    )
    return TrainResponse(run_id=run["id"], metrics=Metrics(**metrics), weights=weights)
```

**`api/db.py`** `insert_run()` is a thin pass-through to `supabase.table("runs").insert({...})`. It does no validation, so whatever the trainer produces goes on the wire.

**`db/migrations/001_init.sql`** declares the metric columns `not null double precision`:

```sql
    mse           double precision not null,
    mae           double precision not null,
    r2            double precision not null,
    weights_json  jsonb       not null,
```

**`shared/schemas.py`** required all three response fields, so a diverged run could not be represented:

```python
class TrainResponse(BaseModel):
    run_id: int
    metrics: Metrics
    weights: dict = Field(..., description="Fitted {'slope': .., 'intercept': ..}.")
```

**`tests/conftest.py`** provides a `client` fixture that monkeypatches every `api.db` function with an in-memory store, so the suite runs with no cloud access. It exposes the store as `client._store`.

### Tool call 3: Bash — read the UI and the remaining shared code; grep the docs for the divergence demo

Command: `cat ui/app.py shared/data.py tests/test_schema.py; grep -n -i "diverg\|1\.5\|nan" README.md TUTORIAL.md MODEL_CARD.md | head -30`

`ui/app.py` unconditionally dereferenced the success fields:

```python
        st.session_state["last_run_id"] = resp["run_id"]
        m = resp["metrics"]
        st.success(f"Run {resp['run_id']} complete.")
```

The grep confirmed the divergence demo is part of the assignment:

```
MODEL_CARD.md:9:- **Primary:** an interactive teaching demo showing how a linear model trains and how learning rate affects convergence vs. divergence.
MODEL_CARD.md:22:- Sensitive to learning rate: too large diverges, too small under-trains within the epoch budget.
TUTORIAL.md:226:  divergence, and oscillation becomes tangible.
TUTORIAL.md:231:(converges) and again with `lr: 1.5` (diverges → NaN/huge MSE). Both runs land in
TUTORIAL.md:284:Supabase → UI). Try one converging and one diverging learning rate.
```

### Tool call 4: Bash — reproduce both learning rates locally

First attempt used the system `python3` and failed with `ModuleNotFoundError: No module named 'numpy'`. Re-run with the project virtualenv:

```
.venv/bin/python -c "
from shared.data import generate_linear
from api.training import train_linear_regression
xs,ys=generate_linear(2.5,1.0,2.0,500)
for lr in (0.01,1.5):
    m,w,l=train_linear_regression(xs,ys,lr=lr,batch_size=32,epochs=100)
    print(lr,m,w,l[:3],l[-3:])
"
```

Result:

```
0.01 {'mse': 4.1313910484313965, 'mae': 1.5952467918395996, 'r2': 0.9805991649627686} {'slope': 2.478285074234009, 'intercept': 0.8425126075744629} [24.399311933517456, 4.2664350891113285, 4.271069393157959] [4.254810028076172, 4.222260112762451, 4.169166841506958]
1.5 {'mse': nan, 'mae': nan, 'r2': nan} {'slope': nan, 'intercept': nan} [inf, nan, nan] [nan, nan, nan]
```

This confirmed the user's hypothesis. At `lr=1.5` the very first epoch's loss is `inf`; from the second epoch on everything is `nan`, and both the metrics and the fitted weights are `nan`.

### The Render error, explained

The deployed failure the user reported:

```
POST /train  ->  500
PGRST102: Empty or invalid json
```

`PGRST102` is PostgREST's parse error for a malformed request body. Python's `json.dumps` serializes `float('nan')` as the bare token `NaN` and `float('inf')` as `Infinity`. Neither is valid JSON under RFC 8259, so PostgREST rejects the whole insert body before it ever reaches Postgres — which is why the message is a JSON parse error rather than a column or constraint error. The `not null` metric columns in `001_init.sql` were never the thing that complained.

The exception propagated out of `db.insert_run()`, through the `/train` handler, and FastAPI turned it into the 500 that Streamlit surfaced as an HTTP error. Even with the database out of the picture, the response itself could not have been serialized: `TrainResponse` would have carried the same `nan` values into FastAPI's JSON encoder.

## Turn 1: Assistant (the change)

### Tool call 5: Bash — apply the edits via a Python script, append the tests, run the suite, show the diff

The change has four parts.

**1. `shared/schemas.py` — let the response represent a diverged run.**

```diff
 class TrainResponse(BaseModel):
-    run_id: int
-    metrics: Metrics
-    weights: dict = Field(..., description="Fitted {'slope': .., 'intercept': ..}.")
+    """Result of POST /train.
+
+    When training diverges (loss/metrics/weights become NaN or Infinity) there
+    are no real numbers to report or store, so ``diverged`` is True, ``run_id``,
+    ``metrics`` and ``weights`` are None, and nothing is written to Supabase.
+    """
+
+    run_id: Optional[int] = None
+    metrics: Optional[Metrics] = None
+    weights: Optional[dict] = Field(
+        None, description="Fitted {'slope': .., 'intercept': ..}."
+    )
+    diverged: bool = False
+    diverged_at_epoch: Optional[int] = Field(
+        None, description="1-based epoch where the training loss first became non-finite."
+    )
+    message: Optional[str] = None
```

**2. `api/main.py` — check for non-finite values before touching Supabase.**

```diff
+import math
 import os
 import subprocess
@@
-    metrics, weights, _loss = train_linear_regression(
+    metrics, weights, loss = train_linear_regression(
         dataset["xs"],
         dataset["ys"],
         lr=req.lr,
         batch_size=req.batch_size,
         epochs=req.epochs,
         test_size=req.test_size,
     )
+
+    # A too-large learning rate makes the loss blow up to Infinity/NaN. Those
+    # values are not valid JSON (Supabase rejects them with PGRST102) and are not
+    # meaningful metrics, so report the divergence instead of storing anything.
+    bad_epoch = next((i + 1 for i, v in enumerate(loss) if not math.isfinite(v)), None)
+    values = [*metrics.values(), *weights.values()]
+    if bad_epoch is not None or not all(math.isfinite(v) for v in values):
+        where = f" at epoch {bad_epoch}" if bad_epoch is not None else ""
+        return TrainResponse(
+            diverged=True,
+            diverged_at_epoch=bad_epoch,
+            message=(
+                f"Training diverged{where}: the loss became non-finite (NaN/Infinity) "
+                f"with lr={req.lr}. No metrics were produced and no run was saved. "
+                "Try a smaller learning rate."
+            ),
+        )
+
     run = db.insert_run(
```

The loss history is now used rather than discarded, which is what makes `diverged_at_epoch` possible. The check covers both the per-epoch loss and the final metrics and weights, so a run that produced a finite loss curve but a non-finite metric is still caught.

**3. `ui/app.py` — show the divergence instead of dereferencing null metrics.**

```diff
-        st.session_state["last_run_id"] = resp["run_id"]
-        m = resp["metrics"]
-        st.success(f"Run {resp['run_id']} complete.")
-        mc1, mc2, mc3 = st.columns(3)
-        mc1.metric("MSE", f"{m['mse']:.3f}")
-        mc2.metric("MAE", f"{m['mae']:.3f}")
-        mc3.metric("R²", f"{m['r2']:.3f}")
-
-        # Fitted-line overlay against the raw dataset points.
-        ds = api_get(f"/runs/{resp['run_id']}")
-        w = resp["weights"]
-        st.markdown(
-            f"Fitted line: **y = {w['slope']:.3f}·x + {w['intercept']:.3f}**"
-        )
+        if resp.get("diverged"):
+            st.error(resp["message"])
+        else:
+            st.session_state["last_run_id"] = resp["run_id"]
+            m = resp["metrics"]
+            st.success(f"Run {resp['run_id']} complete.")
+            mc1, mc2, mc3 = st.columns(3)
+            mc1.metric("MSE", f"{m['mse']:.3f}")
+            mc2.metric("MAE", f"{m['mae']:.3f}")
+            mc3.metric("R²", f"{m['r2']:.3f}")
+
+            # Fitted-line overlay against the raw dataset points.
+            ds = api_get(f"/runs/{resp['run_id']}")
+            w = resp["weights"]
+            st.markdown(
+                f"Fitted line: **y = {w['slope']:.3f}·x + {w['intercept']:.3f}**"
+            )
```

`st.session_state["last_run_id"]` is only updated on success, so the Predict tab keeps pointing at the last run that actually exists.

**4. `tests/test_training.py` — two new tests appended.**

```python
def test_diverging_lr_is_reported_not_stored(client):
    """lr=1.5 blows up to NaN/Inf: report it honestly and save nothing."""
    ds = client.post(
        "/datasets",
        json={"name": "div", "slope": 2.5, "intercept": 1.0, "noise": 2.0, "n_points": 500},
    ).json()
    resp = client.post(
        "/train",
        json={"dataset_id": ds["id"], "lr": 1.5, "batch_size": 32, "epochs": 100},
    )
    assert resp.status_code == 200
    body = resp.json()
    assert body["diverged"] is True
    assert body["run_id"] is None
    assert body["metrics"] is None and body["weights"] is None
    assert body["diverged_at_epoch"] >= 1
    assert "diverged" in body["message"].lower()
    assert client._store["runs"] == {}  # nothing NaN was persisted


def test_converging_run_is_not_flagged_diverged(client):
    ds = client.post(
        "/datasets",
        json={"name": "ok", "slope": 2.5, "intercept": 1.0, "noise": 2.0, "n_points": 500},
    ).json()
    body = client.post(
        "/train",
        json={"dataset_id": ds["id"], "lr": 0.01, "batch_size": 32, "epochs": 100},
    ).json()
    assert body["diverged"] is False
    assert body["run_id"] == 1 and len(client._store["runs"]) == 1
    assert body["metrics"]["r2"] > 0.9
```

The first test asserts on `client._store["runs"]` directly, so it fails if a diverged run is ever persisted — the regression that caused the 500. The second is the paired guard that the successful path is unchanged.

### Test results

`.venv/bin/python -m pytest -q`:

```
8 passed, 1 skipped, 3 warnings in 3.88s
```

Up from 6 passed, 1 skipped before this session (the two new tests). The skip is `tests/test_supabase_roundtrip.py`, which is skipped unless real Supabase credentials are in the environment. The three warnings are pre-existing: the pydantic `Field "model_loader" in Health has conflict with protected namespace "model_"` warning, unrelated to this change.

`git diff --stat`:

```
 api/main.py            | 21 ++++++++++++++++++++-
 shared/schemas.py      | 20 +++++++++++++++++---
 tests/test_training.py | 34 ++++++++++++++++++++++++++++++++++
 ui/app.py              | 31 +++++++++++++++++--------------
 4 files changed, 88 insertions(+), 18 deletions(-)
```

## Turn 1: Assistant (final message)

The assistant reported the cause, the four-file diff, the trade-off, and this test plan:

1. **Offline:** `.venv/bin/python -m pytest -q`. The new tests use the stubbed DB, so they need no Supabase access.
2. **Deployed:**
   - Redeploy the API on Render, then the UI on Streamlit if it does not auto-deploy.
   - Create a dataset in the Train tab.
   - *Success:* lr 0.01, batch 32, epochs 100 → "Run N complete", MSE ≈ 4, R² ≈ 0.98, and a new row in Run History.
   - *Divergence:* lr 1.5 → a red error saying training diverged at epoch 1, no metrics, and no new Run History row.
3. **Optional API-only check** (`$API` is the Render URL):
   `curl -s -X POST $API/train -H 'content-type: application/json' -d '{"dataset_id":1,"lr":1.5}'` → `"diverged": true` with HTTP 200.

It also noted that the Render redeploy must finish before retesting, or the old 500 will still appear.

---

## Turn 2: User

> Please document this Claude Code session before any commit.
>
> 1. Create:
> ai_documentation/ai_conversations/claude-002-diverging-training.md
>
> Include the complete transcript from this session:
> - My prompts
> - Your responses
> - The debugging process
> - The Render error
> - The code changes
> - The tests and results
> - The trade-off that diverged runs are not stored
>
> 2. Update:
> ai_documentation/ai_assistance_log.md
>
> Add a short entry with:
> - Date
> - Problem
> - Cause
> - Files changed
> - Solution
> - Tests performed
> - What I reviewed and verified
>
> Do not include .env contents, API keys, passwords, or private keys. Do not commit or push anything. Show me the files when finished.

## Turn 2: Assistant

- Read `ai_documentation/ai_assistance_log.md` and `ai_conversations/claude-001-api-env.md` to match the existing format. `ai_conversations.md` is still empty.
- Re-ran `git diff --stat` and `git status --short` for the figures quoted above.
- Wrote this transcript file, then appended the entry to `ai_assistance_log.md`.
- Did not commit or push. `.env` was not read in this session.

---

## Limitation: diverged runs are not stored

This is a deliberate trade-off, and it is the part of the change most worth reviewing.

**What it means.** A diverged run leaves no trace in Supabase. It does not appear in the Run History tab, it gets no `run_id`, and it cannot be used on the Predict tab. Only the HTTP response reports it; refresh the page and the evidence is gone.

**Why it was done this way.** `mse`, `mae` and `r2` are `not null double precision` in `db/migrations/001_init.sql`. Storing a diverged run honestly would mean either writing `NaN`/`Infinity` — which is what broke in the first place, and Postgres `double precision` accepts `'NaN'` but PostgREST will not carry it through JSON — or making those columns nullable and adding a status column to distinguish "diverged" from "not yet computed". That is a migration the user would have to apply by hand in the Supabase dashboard, and it changes the shape of the `runs` table that the anon-key Run History query reads. The assistant chose the smaller change, as the prompt asked, and flagged the alternative.

**Why it is still honest.** The response says plainly that training diverged and at which epoch. No metric is invented, no sentinel value is substituted, and nothing is silently swallowed. The failure is reported rather than stored.

**If the demo needs diverged runs in Run History**, the follow-up is: a migration making `mse`/`mae`/`r2` nullable and adding a `status` column, `db.insert_run` writing nulls with `status='diverged'`, and the Run History tab rendering that status. The assistant offered this and did not implement it.

## Related note

`TUTORIAL.md:231` still describes the `lr: 1.5` run as producing "NaN/huge MSE" and says "Both runs land in" the runs table. That is no longer accurate — the diverged run does not land there. The assistant left the professor's tutorial file untouched because the prompt did not ask for documentation edits, and flagged the discrepancy in its final message.

## Verification results

| Check | Result |
|---|---|
| `lr=0.01` local repro, before the fix | finite: MSE ≈ 4.13, MAE ≈ 1.60, R² ≈ 0.981, slope ≈ 2.478 |
| `lr=1.5` local repro, before the fix | loss `[inf, nan, nan, ...]`; metrics and weights all `nan` |
| `pytest -q`, after the fix | 8 passed, 1 skipped (was 6 passed, 1 skipped) |
| `git status --short` | ` M api/main.py`, ` M shared/schemas.py`, ` M tests/test_training.py`, ` M ui/app.py`, plus the new files in `ai_documentation/` from Turn 2 |
| Secrets | none printed; `.env` not read this session, and git-ignored |

Not verified: the fix running against the deployed Render service or a real Supabase instance; the Streamlit UI change rendering in a browser (no Streamlit run in this session); and that PostgREST specifically accepts the new payload, since the diverged path no longer sends one. The `PGRST102` explanation is reasoned from the JSON spec and the observed `nan` values, not from a captured request body.
