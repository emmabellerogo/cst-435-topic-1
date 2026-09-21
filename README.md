# Regress-It — CST-435 Topic 1: Three-Cloud Architecture

- **Author:** Emma Rogoveanu
- **Course:** CST-435 Search Engines and Data Mining Lecture & Lab
- **GitHub repository:** https://github.com/emmabellerogo/cst-435-topic-1
- **Streamlit UI:** https://cst-435-topic-1-fbnuquwxgxz65zpninqfqu.streamlit.app/
- **Render API:** https://regress-it-api-0wsi.onrender.com
- **Supabase project reference:** `mwehaosstwlcpyhitdzl`

For this project I deployed the Regress-It linear-regression demo across three
cloud platforms: **Streamlit (UI) ⇄ FastAPI (Model API) ⇄ Supabase (Data)**.
Each tier runs on a separate platform and talks to the next over HTTPS.

## Live deployment

| Tier | Platform | URL |
|------|----------|-----|
| **UI** | Streamlit Community Cloud | https://cst-435-topic-1-fbnuquwxgxz65zpninqfqu.streamlit.app/ |
| **API** | FastAPI on Render | https://regress-it-api-0wsi.onrender.com |
| **Data** | Supabase PostgreSQL | Project `mwehaosstwlcpyhitdzl` (accessed only by the API and, read-only, by the UI) |

---

## What it does

Regress-It is an interactive teaching demo for 1-D linear regression. You pick a
learning rate, batch size, and epoch count; the API trains `y = w·x + b` with
PyTorch mini-batch SGD on synthetic data, reports held-out **MSE / MAE / R²**,
and persists successful training runs. The UI shows the metrics and the fitted
line for each run, lets you make predictions, and lets you browse run history.

## Architecture

```
┌──────────────────────┐   HTTPS/JSON    ┌──────────────────────┐   service-role   ┌──────────────────┐
│  Streamlit Cloud     │ ──────────────► │  FastAPI on Render   │ ───────────────► │  Supabase        │
│  (ui/app.py)         │                 │  (api/main.py)       │   full access    │  Postgres        │
│  thin client, no ML  │                 │  PyTorch training    │                  │  datasets/runs/  │
│                      │ ◄────anon key,  │                      │                  │  predictions     │
│                      │   read-only ────┼─────────────────────┼──────────────────►│  (RLS: anon can  │
└──────────────────────┘   SELECT runs   └──────────────────────┘                  │   only SELECT)   │
                                                                                    └──────────────────┘
```

- **UI never touches the model or writes SQL.** It calls the API over HTTPS and
  performs one read-only `SELECT` on `runs` with the anon public key.
- **API owns the model and all writes**, using the Supabase **service-role** key.
- **Supabase is the single source of truth** for datasets, runs, and predictions.

See [`TUTORIAL.md`](./TUTORIAL.md) for the full step-by-step build and deploy guide.

## Project structure

```
cst-435-topic-1/
├── TUTORIAL.md               # Full build + deploy guide (start here)
├── README.md                 # This file
├── MODEL_CARD.md             # Model details, intended use, limitations
├── shared/                   # Code shared by both tiers
│   ├── schemas.py            # Pydantic API contract
│   └── data.py               # Synthetic linear data generator
├── api/                      # FastAPI tier (deploys to Render)
│   ├── main.py               # Endpoints
│   ├── training.py           # PyTorch linear regression
│   ├── db.py                 # Supabase (service-role) data access
│   ├── configs/default.yaml  # Default hyperparameters
│   └── requirements.txt
├── ui/                       # Streamlit tier (deploys to Streamlit Cloud)
│   ├── app.py                # 5-tab thin client
│   ├── requirements.txt      # No torch
│   └── .streamlit/secrets.toml.example
├── db/                       # Database tier (Supabase)
│   ├── migrations/001_init.sql
│   └── seed.py
├── tests/                    # pytest suite
├── docs/screenshots/         # Training screenshots used in this README
├── ai_documentation/         # Record of AI assistance used on the project
├── render.yaml               # Render blueprint
├── requirements-dev.txt      # Both tiers + pytest (local dev)
└── .env.example
```

## Quickstart (local)

```bash
cd cst-435-topic-1

# 1. Install everything (both tiers + test tools)
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt

# 2. Run the tests (8 passed, 1 skipped; the live-Supabase test skips without creds)
pytest -q

# 3. Configure secrets
cp .env.example .env                                   # API: SUPABASE_URL + SERVICE key
cp ui/.streamlit/secrets.toml.example ui/.streamlit/secrets.toml

# 4. Run the API
uvicorn api.main:app --reload --port 8000

# 5. In another terminal, run the UI
streamlit run ui/app.py
```

To deploy to the three clouds, follow **Part E** of [`TUTORIAL.md`](./TUTORIAL.md):
apply `db/migrations/001_init.sql` in the Supabase SQL Editor → deploy the API
from `render.yaml` on Render → deploy the UI on Streamlit Community Cloud.

## Deployment and testing

The three tiers are deployed and connected:

- **Streamlit Community Cloud** hosts the UI (`ui/app.py`), a thin client that calls the API.
- **Render** hosts the FastAPI model API (`api/main.py`), which trains and serves the PyTorch model.
- **Supabase PostgreSQL** stores datasets, runs, and predictions (schema in `db/migrations/001_init.sql`).

Secrets (Supabase URL and keys) are set as environment variables on Render and
as Streamlit secrets; they are not committed to this repository.

### Verification results (deployed API)

| Check | Request | Result |
|-------|---------|--------|
| Health | `GET /healthz` | `status: ok`, `model_loader: true`, `supabase: true` |
| Dataset creation | `POST /datasets` | HTTP 200, dataset saved to Supabase |
| Training (stable) | `POST /train` with `lr=0.01` | Succeeded — MSE ≈ 4.131, MAE ≈ 1.595, R² ≈ 0.981 |
| Prediction | `POST /predict` with `x=4` | ŷ ≈ 10.76 |
| Training (divergent) | `POST /train` with `lr=1.5` | Handled safely as a divergence case; no run was saved |

The `lr=1.5` case confirms that when the loss becomes non-finite (NaN/Infinity),
the API returns a response flagged `diverged: true` with an explanatory message
and does not write a run to Supabase.

### Training screenshots

![Successful training run with lr=0.01: MSE 4.131, MAE 1.595, R² 0.981](docs/screenshots/convergent%20training.png)

*Successful run on dataset 5 (true slope 2.5, intercept 1.0, noise 2.0, 500 points) with
lr=0.01, batch size 32, and 100 epochs. The fitted line was y = 2.478·x + 0.843.*

![Divergent training attempt with lr=1.5 reported as diverged at epoch 1](docs/screenshots/divergent%20training.png)

*The same dataset and settings with lr=1.5. The API reported that the loss became
non-finite at epoch 1, and no run was saved.*

## Data persisted / Data not persisted

**Persisted in Supabase:**

- **Datasets** — every dataset created through `POST /datasets` (its parameters plus the generated x and y values).
- **Successful training runs** — hyperparameters, held-out MSE/MAE/R², and the fitted weights.
- **Predictions** — every `POST /predict` call is logged with its run ID, x, and ŷ.

**Not persisted:**

- **Divergent training attempts.** When the loss becomes NaN or Infinity there are no valid
  metrics or weights to store (Supabase also rejects those values as JSON), so the API
  returns `diverged: true` with a message and writes nothing to the `runs` table.

## Engineering Report

**Why I used lr=0.01.** For my successful run I trained on dataset 5, which I
generated with a true slope of 2.5, a true intercept of 1.0, noise with a standard
deviation of 2.0, and 500 points. I used lr=0.01, batch size 32, and 100 epochs.
I chose 0.01 because it is the default in `api/configs/default.yaml` and it is
small enough that SGD settles toward the minimum instead of overshooting. The
model learned y = 2.478·x + 0.843, close to the real line, with a held-out MSE of
4.131 and R² of 0.981. Since the noise I added has a variance of 2.0² = 4, an
MSE around 4 is about as low as this model can get. The remaining error is mostly
the noise I added on purpose.

**Why lr=1.5 diverged.** When I trained on the same dataset with lr=1.5, the API
reported that training diverged at epoch 1. The x values range from -10 to 10, so
the gradient for the slope gets multiplied by fairly large x values. With a
learning rate of 1.5, each update overshoots the minimum by more than it corrects,
so the next error is larger, the next gradient is larger, and the weights keep
growing until the loss becomes Infinity or NaN. A rough check: the average of x²
over [-10, 10] is about 33, and gradient descent on a squared-error loss is only
stable when the learning rate is below about 2 divided by the curvature (here
about 2 × 33). That puts the limit near 0.03, so 0.01 is safely under it and 1.5
is far past it.

**Stopping criterion.** The training code in `api/training.py` does not use early
stopping. It runs exactly the number of epochs requested (100 in both of my runs)
and records the average training loss for each epoch. After the loop finishes,
`api/main.py` checks the loss history, metrics, and weights. If any of them are
NaN or Infinity, the API returns `diverged: true` with the first bad epoch and
does not save anything. So "diverged at epoch 1" tells you where the loss first
went bad, not where training stopped.

**Why a 20% held-out split.** Before training, the code shuffles the points with
a fixed seed and sets aside 20% of them (100 of my 500 points). The model never
trains on those points, and MSE, MAE, and R² are computed only on them. That way
the metrics show how the line does on data it has not seen, instead of how well
it memorized the training data. For a two-parameter model, 400 training points is
plenty. Because the seed is fixed, the same dataset always gets the same split, so runs
with different hyperparameters are scored on the same test points.

**How Run History helps.** Each successful run is saved to the `runs` table with
its dataset ID, learning rate, batch size, epochs, MSE, MAE, R², and timestamp.
The Run History tab reads the latest 50 runs straight from Supabase with the
read-only anon key and shows them newest first. This let me compare runs side by
side, for example the same dataset with different learning rates. One thing I
had to keep in mind is that divergent attempts never show up there, since they
are never saved.

**Reporting limitations honestly.** The biggest lesson for me was that a good
number is not the whole story. An R² of 0.981 sounds impressive, but a
non-technical client could easily read it as "this model is 98% accurate" on
their real data. That would be wrong. The model was trained on synthetic data
that I generated from a straight line, it only has one input, and it has never
been tested on real-world data. If I were presenting it to a client, I would say
that plainly and explain the error in terms they can picture (predictions are
typically off by about 1.6 units). From a Christian worldview, I see this
as a matter of honesty and stewardship. Proverbs 11:1 says that "a false balance
is an abomination to the LORD, but a just weight is his delight." A metric is a
kind of measurement, and presenting it in a way that makes the model look better
than it is would be like using a false weight. The client is trusting me with
something they cannot check themselves, so I owe them the truth about what the
model can do, even when that makes my work look less impressive.

## API endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/datasets` | Create a synthetic dataset |
| `POST` | `/train` | Train a model, persist the run if it succeeded, return metrics |
| `GET`  | `/runs/{run_id}` | Fetch one run |
| `GET`  | `/runs` | List recent runs |
| `POST` | `/predict` | Predict `ŷ` for an `x` using a fitted run |
| `GET`  | `/healthz` | Liveness / DB ping |
| `GET`  | `/version` | Build SHA + framework versions |

## Reusing this pattern

The three-cloud split and the file layout stay identical for every product. Swap
only the model in `api/training.py`, the Pydantic contract in `shared/schemas.py`,
the tables in `db/migrations/`, and the UI tabs — the UI stays a thin client and
Supabase stays the single source of truth. Two worked examples
(`income-insight`, `see-sense`) live alongside this one; see the
final section of [`TUTORIAL.md`](./TUTORIAL.md).

## Checklist

- [x] Three live URLs listed at the top of this README
- [x] `datasets`, `runs`, `predictions` tables in Supabase with RLS
- [x] 6+ API endpoints
- [x] 5 Streamlit tabs (Concepts, Train, Predict, Run History, Model Card)
- [x] PyTorch training with held-out MSE/MAE/R²
- [x] pytest suite passing
- [x] `MODEL_CARD.md` completed
