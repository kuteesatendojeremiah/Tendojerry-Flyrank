# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

An intern's working copy of the FlyRank ML internship starter: a content-refresh ranking problem on an anonymized slice of FlyRank search data (`data/raw/content_refresh_anonymized.csv`, 30,000 rows × 44 columns). The intern's own work (lane experiments, capstone notebooks, report) lives in `work/`; the rest of the repo is the shared reference.

## Commands

```bash
pip install -r requirements.txt      # pandas, numpy, scikit-learn, matplotlib, reportlab, duckdb, huggingface_hub
python scripts/run_all.py            # full pipeline on the bundled sample (~1 min), writes to outputs/
python scripts/03_train_model.py     # any stage can be run individually (01–05)
```

There is no separate test suite or linter. CI (`.github/workflows/smoke-test.yml`) re-runs the whole pipeline and fails if any dataset CSV/parquet/archive is committed beyond the starter slice.

## Pipeline architecture

Five sequential scripts, each reading the previous stage's output (paths and shared feature lists live in `scripts/ml_utils.py`):

1. `01_prepare_features.py` — cleans the raw CSV, adds engineered columns, defines the label → `data/processed/refresh_feature_vector.csv`
2. `02_baseline_score.py` — transparent hand-written rule score + reason codes → `data/processed/baseline_refresh_queue.csv`
3. `03_train_model.py` — logistic regression, decision tree, random forest with a **client-holdout split** (~20% of clients, not rows) → `data/processed/model_predictions.csv`, `outputs/model_results.json`
4. `04_evaluate_and_export.py` — blends model + baseline into the final ranked queue, charts, Markdown report
5. `05_build_pdf_report.py` — PDF summary

**The label:** `is_declining_label = (trend_direction == "down")`. Because it is derived from `trend_direction`, neither `trend_direction` nor `trend_pct` may ever be a model feature — this is the repo's canonical leakage example. Hashed `content_id`/`client_id` are for grouping/joining only, never features.

**Evaluation:** primary metric is Precision@50 (`precision_at_k` in `ml_utils.py`). The baseline reproduces exactly at 0.240; the random-forest number is library-version sensitive (~0.68–0.74) — the stable claim is the ~3x lift, not the third decimal.

## Hard rules

- **`scripts/` is the pristine reference pipeline — do not edit it.** The one exception is `scripts/ml_utils.py`, where feature-list changes (`MODEL_NUMERIC_FEATURES` / `MODEL_CATEGORICAL_FEATURES`) are allowed. For anything bigger, copy the script into `work/scripts/` and modify the copy.
- **All new work goes in `work/`** (`work/notebooks/`, `work/scripts/`, `work/figures/`, `capstone_report.md`).
- **Never commit datasets.** `data/` is read-only; `data/processed/` and most of `outputs/` are gitignored by design and regenerate on each run — a quiet `git status` after a pipeline run is expected. CI blocks any committed CSV other than the starter slice and `outputs/refresh_queue_sample.csv`.
- **Data-use rules** (`DATA_USE.md`): no client-identifying data anywhere, no pasting the data into third-party tools, public-facing language must be decision-support framed (observed/measured/directional — no causal or "predicted Google's algorithm" claims). Never hardcode a Hugging Face token in a notebook — the repo is public.
- New notebooks (weeks 3+) start with `%pip install -q duckdb huggingface_hub pandas scikit-learn matplotlib` so they run in a fresh Colab kernel.

## Weeks 3+ data path

The full ~79M-row release is hosted at the gated Hugging Face dataset `FlyRank/internship-warehouse`, queried via DuckDB without downloading (workflow in `notebooks/03_working_with_the_full_release.ipynb`; access setup in `SETUP.md`). Aggregate in SQL, model in sklearn. That dataset never enters the repo.

## Task-specific skills

`skills/` is a router of instruction files, one per internship task (framing, data contracts, DuckDB querying, signal audits, baselines, model training, leakage hunting, claims, paper writing). See the table in `skills/README.md` and load exactly one skill (plus `skills/flyrank/flyrank-data/SKILL.md` for data work) for the task at hand.

## Reference docs

- `docs/data-dictionary.md` — all 44 columns, meanings, and gotchas (check before adding a feature; rate columns like `ctr` are 0–100 percentages, so `0.76` means 0.76%)
- `GUIDE.md` — full operating manual for the repo
- `docs/ml-intern-dataset-and-lane-guide.md` — lanes and capstone workflow
