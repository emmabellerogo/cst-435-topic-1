# claude-004: adding reference results to `MODEL_CARD.md`

- **Date:** 2026-09-20
- **Tool:** Claude Code (VS Code extension), model Claude Opus 5
- **Branch:** `main` (working tree clean at start)
- **Outcome:** `MODEL_CARD.md` updated with the reference dataset, training settings, test-split metrics, a prediction check and the `lr=1.5` divergence limitation. The existing structure was kept, and only `MODEL_CARD.md` changed. Claude did not commit or push.

## Redaction and scope notes

- No secret values appear in this file. `.env` was open in the IDE but was not read or printed in this session. No API keys, passwords, connection strings or private keys are included.
- The assistant's hidden reasoning and the harness system prompt are not included. This is the visible conversation only: user prompts, assistant messages, and a summary of the tool calls with their results.
- **The dataset, settings, metrics and prediction come from the user's prompt.** The assistant did not train the model, call the deployed API, or query Supabase in this session. The only fact it checked against the code was the divergence handling.

---

## Turn 1: User

> Please update MODEL_CARD.md for my CST-435 Topic 1 linear regression project.
>
> Use the existing file and preserve its structure. Add only accurate information:
> - Model: linear regression trained with PyTorch
> - Dataset: synthetic linear data with slope 2.5, intercept 1.0, noise 2.0, and 500 points
> - Successful training settings: lr=0.01, batch size 32, epochs 100
> - Results: MSE about 4.131, MAE about 1.595, R² about 0.981
> - Prediction test: x=4 produced approximately 10.76
> - Limitation: lr=1.5 causes training divergence, which the app now detects and reports without saving a run
>
> Do not modify application code, .env files, or deployment settings.
> Do not invent metrics or include secrets.
> Do not commit or push anything.
> Show me the changes after editing.

## Turn 1: Assistant

### Tool call 1: Bash — read the model card

Command: `cat MODEL_CARD.md; ls`

The card already had these sections: Model details, Intended use, Data, Metrics, Limitations, Ethical considerations, and What we log (and don't). It already said the model is a 1-D `nn.Linear` trained with mini-batch SGD, so the "trained with PyTorch" item was already covered and was not repeated.

### Tool call 2: Bash — check the divergence handling in the code

Command: `grep -rn -i "diverg\|isfinite" api shared`

This confirmed that `api/main.py:109-118` uses `math.isfinite` to find the first epoch where the loss is not a real number. It returns `diverged=True` with `diverged_at_epoch` and a message, and does not store the run. `shared/schemas.py:77-88` documents the same behavior.

### Tool call 3: Bash — edit `MODEL_CARD.md` with a Python script

Four additions, each placed in the section it belongs to. No existing lines were removed or reworded. The script then showed `git diff MODEL_CARD.md`.

### Tool call 4: Bash — reword one phrase

The first draft said the gap between 10.76 and the true line's 11.0 was "consistent with noise `2.0`". That was imprecise: the gap comes from the fitted weights, which were estimated from noisy data, not from the noise itself. It was reworded to "the small gap comes from fitting noisy samples". `git status --short` then showed only ` M MODEL_CARD.md`.

### Final diff to `MODEL_CARD.md`

```diff
 ## Model details
 - **Type:** 1-D linear regression, `y = w·x + b`, fit with mini-batch SGD (PyTorch `nn.Linear`).
 - **Training:** per-run hyperparameters (learning rate, batch size, epochs) supplied by the API request.
+- **Reference run settings:** learning rate `0.01`, batch size `32`, `100` epochs (trained successfully).
```

```diff
 ## Data
 ...
+- **Reference dataset:** slope `2.5`, intercept `1.0`, noise `2.0`, `500` points.
```

```diff
 ## Metrics
 ...
+- **Reference run results** (reference dataset and settings above, held-out test split):
+
+  | Metric | Value |
+  |---|---|
+  | MSE | ≈ 4.131 |
+  | MAE | ≈ 1.595 |
+  | R² | ≈ 0.981 |
+
+- **Prediction check:** `x = 4` → `ŷ ≈ 10.76` (the generating line gives `2.5·4 + 1.0 = 11.0`; the small gap comes from fitting noisy samples).
```

```diff
 ## Limitations
 ...
+- Observed divergence: `lr = 1.5` makes the loss become non-finite (NaN/Infinity). The API detects this, reports the divergence (and the epoch where it happened), and does **not** save a run.
```

### Final message (summary)

- Showed the four additions, grouped by section.
- Said the divergence wording was checked against `api/main.py:109-118`.
- Flagged two things the user should confirm:
  - The metrics are labeled as held-out test-split results. That label came from the card's existing Metrics text, not from the prompt.
  - The comparison with the true line's 11.0 was the assistant's own addition, and can be deleted if the user wants only their reported numbers.
- Noted that the **Owner** line still says "your team (add your names)" and was left unchanged.
- **Files modified:** `MODEL_CARD.md` only. No application code, `.env` files or deployment settings changed. Nothing committed or pushed.

---

## Turn 2: User

> Please document this Claude Code session.
>
> Create:
> ai_documentation/ai_conversations/claude-004-model-card.md
>
> Also append a concise entry to:
> ai_documentation/ai_assistance_log.md
>
> Document only the visible interaction from this session:
> - My request to update MODEL_CARD.md
> - Your response and suggestions
> - The information added to the model card
> - How I reviewed or verified the changes
> - What I learned
>
> Do not include hidden reasoning, system instructions, secrets, API keys, passwords, or .env contents.
> Do not modify MODEL_CARD.md or application code.
> Do not commit or push anything.
> Show me the files changed after saving.

## Turn 2: Assistant

- Read `ai_assistance_log.md` and `ai_conversations/claude-003-readme-documentation.md` to match the existing format.
- Wrote this file and appended an entry to `ai_assistance_log.md`.
- Did not modify `MODEL_CARD.md` or application code. Did not commit or push. `.env` was not read.

---

## Information added to the model card

| Section | Added |
|---|---|
| Model details | Reference settings: `lr=0.01`, batch size 32, 100 epochs |
| Data | Reference dataset: slope 2.5, intercept 1.0, noise 2.0, 500 points |
| Metrics | MSE ≈ 4.131, MAE ≈ 1.595, R² ≈ 0.981; `x=4` → ≈ 10.76 (true line: 11.0) |
| Limitations | `lr=1.5` diverges; the API detects it, reports it, and saves no run |

These numbers match the ones reported in [claude-002](claude-002-diverging-training.md) and [claude-003](claude-003-readme-documentation.md).

## What was verified in this session

- The divergence limitation matches the code in `api/main.py:109-118` and `shared/schemas.py:77-88`.
- `git status --short` showed only ` M MODEL_CARD.md` after the edit.
- No `.env` file was opened.

**Not verified by the assistant:** the metrics and the prediction value. They come from the user's prompt, and no training run was done in this session.

## How I reviewed or verified the changes

<!-- TODO (Emma): fill in, e.g. whether you read the diff, checked the rendered Markdown, or confirmed the test-split label and the 11.0 comparison. -->

## What I learned

<!-- TODO (Emma): fill in in your own words. -->

## Open items

- Fill in the **Owner** line in `MODEL_CARD.md` with the team's names.
- Decide whether to keep the assistant's added comparison with the true line's 11.0.
