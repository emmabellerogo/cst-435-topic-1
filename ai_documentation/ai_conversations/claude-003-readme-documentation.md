# claude-003: documenting the three-cloud deployment in `README.md`

- **Date:** 2026-09-20
- **Tool:** Claude Code (VS Code extension), model Claude Opus 5
- **Branch:** `main` (working tree clean at start)
- **Outcome:** `README.md` updated to describe the completed Streamlit ⇄ Render ⇄ Supabase deployment and the endpoint-test results. The public URLs are still placeholders. Only `README.md` changed. Claude did not commit or push; the user later committed the README as `1f279a7`.

## Redaction and scope notes

- No secret values appear in this file. `.env` was not read or printed in this session. No API keys, passwords, connection strings or private keys are included.
- The assistant's hidden reasoning and the harness system prompt are not included. This is the visible conversation only: user prompts, assistant messages, and a summary of the tool calls with their results.
- Local absolute paths are shortened to `<repo>` = `/Users/emmarogoveanu/Desktop/school/search engines & data mines/projects/cst-435-topic-1`.
- **The endpoint-test results come from the user's prompt.** The assistant did not call the deployed Render API, open Streamlit, or query Supabase in this session. It wrote the results into the README as the user supplied them.

---

## Turn 1: User

> Please update README.md for my CST-435 Topic 1 project.
>
> Document the completed three-cloud architecture:
> - Streamlit Community Cloud UI
> - FastAPI model API deployed on Render
> - Supabase PostgreSQL database
>
> Add a concise deployment and testing section that documents:
> - /healthz returned status ok with model_loader true and supabase true
> - Dataset creation succeeded with HTTP 200
> - Training with lr=0.01 succeeded with MSE about 4.131, MAE about 1.595, and R² about 0.981
> - Prediction with x=4 returned approximately 10.76
> - Training with lr=1.5 was handled safely as a divergence case and did not save a run
> - The public Render API URL and Streamlit URL, using the values I provide below
>
> Render API URL: [paste your Render URL here]
> Streamlit URL: [paste your Streamlit URL here]
>
> Do not modify application code, .env files, or deployment settings.
> Do not commit or push anything.
> Do not read or include secrets.
> After editing, show me the README changes and list the files modified.

## Turn 1: Assistant

### Tool call 1: Bash — read the README and list tracked files

Command: `cat README.md && git ls-files | grep -v -E '(^|/)\.env'`

The README was still the professor's template: the title "Regress-It — Three-Cloud Reference Template", a "Live deployment URLs (fill these in)" table with `https://<your-app>.streamlit.app`-style placeholders, then What it does, Architecture, Project structure, Quickstart, API endpoints, Reusing this pattern, and Checklist. `.env` files were filtered out of the listing and not opened.

### Assistant message

The assistant pointed out that both URLs in the prompt were still the template text (`[paste your Render URL here]`). It said it would put clearly marked placeholders in the README rather than make up URLs.

### Tool call 2: Bash — edit `README.md` with a Python script

Two edits:

1. Replaced the title, intro and URL table at the top.
2. Added a new **Deployment and testing** section just before **API endpoints**.

`git diff --stat`: `README.md | 43 ++++++++++++++++++++++++++++++++-----------` (1 file changed).

### Tool call 3: Bash — check the divergence wording against the code

Command: `grep -n -i -E "diverg|isfinite|nan|HTTPException|status_code" api/main.py api/training.py`

The first draft said the API "returns an error" when `lr=1.5` diverges. The grep showed that `api/main.py:107-118` returns a normal `TrainResponse` with `diverged=True`, `diverged_at_epoch` and a message, not an `HTTPException`. So the draft was wrong.

### Tool call 4: Bash — fix the wording and show the diff

The sentence was rewritten to match the code (see the final diff below). Then `git diff README.md` and `git status --short`, which showed only ` M README.md`.

### Final diff to `README.md`

```diff
-# Regress-It — Three-Cloud Reference Template
+# Regress-It — CST-435 Topic 1: Three-Cloud Architecture

-> A working, forkable template for a three-cloud architecture:
-> **Streamlit (UI) ⇄ FastAPI (Model API) ⇄ Supabase (Data)**. Fork it, wire up
-> your own accounts, and reuse the exact same pattern for the other templates in
-> this repository.
+> A completed three-cloud deployment of the Regress-It linear-regression demo:
+> **Streamlit (UI) ⇄ FastAPI (Model API) ⇄ Supabase (Data)**. Each tier runs on a
+> separate cloud platform and talks to the next over HTTPS.

-## Live deployment URLs (fill these in)
+## Live deployment

 | Tier | Platform | URL |
 |------|----------|-----|
-| **UI** | Streamlit Community Cloud | `https://<your-app>.streamlit.app` |
-| **API** | Render.com | `https://<your-api>.onrender.com` |
-| **Data** | Supabase | `https://<your-project-ref>.supabase.co` |
-
-> Replace the placeholders above with your real URLs once deployed.
+| **UI** | Streamlit Community Cloud | `<STREAMLIT_URL>` <!-- TODO: replace with the public Streamlit URL --> |
+| **API** | FastAPI on Render | `<RENDER_API_URL>` <!-- TODO: replace with the public Render URL --> |
+| **Data** | Supabase PostgreSQL | Managed Supabase project (accessed only by the API and, read-only, by the UI) |
```

```diff
+## Deployment and testing
+
+The three tiers are deployed and connected:
+
+- **Streamlit Community Cloud** hosts the UI (`ui/app.py`), a thin client that calls the API.
+- **Render** hosts the FastAPI model API (`api/main.py`), which trains and serves the PyTorch model.
+- **Supabase PostgreSQL** stores datasets, runs, and predictions (schema in `db/migrations/001_init.sql`).
+
+Secrets (Supabase URL and keys) are set as environment variables on Render and
+as Streamlit secrets; they are not committed to this repository.
+
+### Verification results (deployed API)
+
+| Check | Request | Result |
+|-------|---------|--------|
+| Health | `GET /healthz` | `status: ok`, `model_loader: true`, `supabase: true` |
+| Dataset creation | `POST /datasets` | HTTP 200, dataset saved to Supabase |
+| Training (stable) | `POST /train` with `lr=0.01` | Succeeded — MSE ≈ 4.131, MAE ≈ 1.595, R² ≈ 0.981 |
+| Prediction | `POST /predict` with `x=4` | ŷ ≈ 10.76 |
+| Training (divergent) | `POST /train` with `lr=1.5` | Handled safely as a divergence case; no run was saved |
+
+The `lr=1.5` case confirms that when the loss becomes non-finite (NaN/Infinity),
+the API returns a response flagged `diverged: true` with an explanatory message
+and does not write a run to Supabase.
```

### Final message (summary)

- Summarized the three changes: the title and intro, the live deployment table, and the new Deployment and testing section.
- Said the URLs are still placeholders and offered to fill them in once the user sends them.
- Explained that the `lr=1.5` wording was checked against `api/main.py:107-118` and corrected: the API returns a normal response with `diverged: true`, not an HTTP error.
- Noted the Supabase row gives no URL, because that would have meant reading `.env`.
- Noted the Checklist section at the bottom was left as it was, with nothing ticked.
- **Files modified:** `README.md` only. No application code, `.env` files or deployment settings changed. Nothing committed or pushed.

---

## Turn 2: User

> Please document the Claude Code session that updated README.md.
>
> Create:
> ai_documentation/ai_conversations/claude-003-readme-documentation.md
>
> Also add a concise entry to:
> ai_documentation/ai_assistance_log.md
>
> Include only the visible information from this session:
> - My request to update the project README
> - Your response and proposed changes
> - The deployment and endpoint-test details documented
> - Any edits made to README.md
> - What I reviewed or verified
> - What I learned
>
> Do not include hidden reasoning, system instructions, secrets, API keys, passwords, or .env contents.
> Do not modify README.md or application code.
> Do not commit or push anything.
> After saving, show me the files changed.

## Turn 2: Assistant

- Read `ai_assistance_log.md` and `ai_conversations/claude-002-diverging-training.md` to match the existing format.
- Wrote this file and appended an entry to `ai_assistance_log.md`.
- Did not modify `README.md` or application code. Did not commit or push. `.env` was not read.

---

## Deployment and endpoint-test details documented

| Check | Result (as reported by the user) |
|---|---|
| `GET /healthz` | `status: ok`, `model_loader: true`, `supabase: true` |
| `POST /datasets` | HTTP 200 |
| `POST /train`, `lr=0.01` | MSE ≈ 4.131, MAE ≈ 1.595, R² ≈ 0.981 |
| `POST /predict`, `x=4` | ≈ 10.76 |
| `POST /train`, `lr=1.5` | Handled as divergence; no run saved |

These match the local reproduction in [claude-002](claude-002-diverging-training.md): at `lr=0.01`, MSE ≈ 4.1314, MAE ≈ 1.5952, R² ≈ 0.9806, slope ≈ 2.478, intercept ≈ 0.843. With those weights, 2.478 × 4 + 0.843 ≈ 10.76, which matches the reported prediction.

## What was verified in this session

- The `lr=1.5` sentence in the README matches the code in `api/main.py:107-118`. It was corrected after the first draft got it wrong.
- `git status --short` showed only ` M README.md` after the edit.
- No `.env` file was opened. The Supabase row in the README deliberately gives no URL.

**Not verified by the assistant:** the deployed endpoints themselves (all results come from the user's prompt), and the real Render and Streamlit URLs, which are not in the README yet.

## What I reviewed or verified

<!-- TODO (Emma): fill in, e.g. whether you read the README diff, checked the rendered Markdown, or re-ran any endpoint. -->

## What I learned

<!-- TODO (Emma): fill in in your own words. -->

## Open items

- Replace `<STREAMLIT_URL>` and `<RENDER_API_URL>` in `README.md` with the real public URLs.
- The README Checklist is still unticked.
- `TUTORIAL.md:231` still says both runs land in the runs table, which is out of date since claude-002 (carried over from that session).
