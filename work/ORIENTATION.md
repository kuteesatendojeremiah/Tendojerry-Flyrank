# Orientation — what this whole repo is, and what you're actually doing

You asked "what am I doing exactly" — fair question, this repo has a lot of files. This is
the plain-language map. Nothing here is graded; it's just for you.

## 1. The one-sentence version

You're using a small, safe slice of real FlyRank search data to answer one practical
question — **"which pages should the content team review first?"** — by building a ranked
list, checking it's better than a simple hand-written rule, and being honest about what the
data can and can't prove. You are not trying to reverse-engineer Google's algorithm. You are
building a decision-support tool: a sorted queue a human still reviews.

## 2. Two datasets, two different jobs

| | Starter CSV | Full warehouse |
|---|---|---|
| **File/location** | `data/raw/content_refresh_anonymized.csv`, ships in this repo | `FlyRank/internship-warehouse` on Hugging Face — gated, ~79M rows, never downloaded |
| **Size** | 30,000 rows, 44 columns, 32 clients | 5 tables, largest is 78.8M rows (daily panel), 104 clients |
| **When you use it** | Weeks 1–2, `notebooks/01` and `notebooks/02` | Weeks 3+ — your lane and capstone run on this |
| **How you touch it** | `pandas.read_csv` | DuckDB queries straight over `hf://...` Parquet files — SQL does the heavy lifting, only small results come back to pandas |
| **Time span** | one 90-day snapshot per row, no dates to slice | 2025-01-27 → 2026-06-30 (~17 months), so you pick a time window yourself |

Why two datasets at all: the CSV is small enough to learn the *workflow* on (Week 1–2), and
the warehouse is what makes the workflow real — a proper multi-month panel where you have to
define your own windows, labels, and leakage checks instead of having them handed to you.

## 3. What's "reference" vs. what's "yours"

```
scripts/            the pristine 5-step pipeline (prepare → baseline → train → evaluate → PDF)
                     — run it, copy from it, NEVER edit it (except ml_utils.py feature lists)
notebooks/01–03      the three guided Week 1–3 notebooks — run top to bottom, do the "your turn" cells
docs/                reference material: data dictionary, the lane guide, the ML framework doc
work/                YOUR folder — everything you build lives here. This is the actual deliverable.
```

`scripts/` exists so you always have a working, honest baseline to compare your own work
against. Reviewers expect it untouched. Everything you actually produce — notebooks,
figures, your capstone report — goes under `work/`.

## 4. Your lane: Refresh / Content Opportunity Scoring

There are 4 core lanes + 2 advanced ones (`docs/ml-intern-dataset-and-lane-guide.md` §8). You
picked **Lane 2**. The question it answers:

> Which pages should be reviewed first for refresh, expansion, protection, pruning, or
> monitoring?

Mechanically, this is a **ranking problem solved with a binary classifier**: a model
estimates the probability a page is declining, and that probability becomes the sort key for
a queue the editorial team works down from the top. You already wrote this framing in
`work/notebooks/w02_ml_task_framing.ipynb`.

Your chosen success metric is **Precision@50**: of the top 50 pages the queue puts first, what
fraction are truly declining? K=50 was chosen because that's roughly what a team can act on
in a cycle — ranking the other 29,950 pages accurately doesn't matter if nobody will ever
touch them.

Every lane (including yours) is expected to eventually produce the same seven artifacts:
**data contract → signal audit → baseline → model → validation → ranked action output →
public-safe write-up.**

## 5. Where you are right now

All eight official notebook skeletons for weeks 3+ have now been pulled from upstream (they
arrive via GitHub commits, not a single download — `git fetch`/`git pull` before trusting this
table, since Colab can push new ones without warning). Filenames were renamed to line up
sequentially with card order (they originally arrived with duplicate/out-of-order `w0N` prefixes
— e.g. two different files both named `w04_...`) — go by the card title either way, but the
prefix is now consistent too.

| Task | Notebook | Status |
|---|---|---|
| Pick a lane, frame the question (ML-02/03) | `w01_research_question.ipynb`, `w02_ml_task_framing.ipynb` | ✅ done |
| Data contract (ML-04) | `w03_data_contract.ipynb` | ✅ done — run in Colab, real numbers on record: 321,546 content items in the March-2026 slice, honest AUC 0.611 (5 prev30 features), leaky AUC 1.000 with the trap column, restored to 0.611 |
| Feature vector + leakage/privacy check (ML-05) | `w04_feature_leakage_check.ipynb` | ✅ done — run in Colab, real numbers on record: 321,546 rows x 24 columns (5 numeric prev30 features + 4 categoricals found live on `dim_content`: `content_type`, `competition_level`, `main_intent`, `model_used`), base rate 0.285, honest AUC 0.664, leaky AUC 1.000, restored to 0.664 |
| Signal audit — do the flags hold? (ML-06) | `w05_signal_audit.ipynb` | ⬜ blank skeleton — not started |
| Baseline action score + top-20 review (ML-07) | `w06_baseline_score.ipynb` | ⬜ blank skeleton — not started |
| Capstone modeling lane (ML-08) | `w07_model.ipynb` | ⬜ blank skeleton — not started |
| Validation + research claim audit (ML-09) | `w08_validation_audit.ipynb` | ⬜ blank skeleton — not started |
| Content action playbook (ML-10) | `w09_action_playbook.ipynb` | ⬜ blank skeleton — not started |
| Research paper + deploy (ML-11) | `capstone_report_template.md` + a deployed page, `submission/paper_url.txt` | ⬜ not started |

So: three tasks fully done and run, six blank skeletons already sitting in the repo waiting
their turn in card order (ML-06 next).

## 6. Vocabulary worth holding onto

- **Label vs. feature vs. leakage** — the label is what you're predicting; a feature must be
  knowable *before* that moment. `trend_direction`/`trend_pct` (starter CSV) and raw
  same-period impressions (warehouse) are the two label traps you've now seen directly —
  each is illegal as a feature because it *is* (or determines) the label.
- **prev30 / mid-panel month** — because the warehouse has real dates, you develop against a
  month like `2026-03` in the middle of the panel, never the final month (June 2026,
  `fact_content_daily_performance_sample`) — that's the sealed test window.
- **`IS TRUE`** — `ga4_data_available` can be `TRUE`, `FALSE`, or `NULL`; a plain `WHERE
  ga4_data_available` silently drops the `NULL` rows without telling you. `IS TRUE` is
  explicit.
- **Client-holdout split** — pages from the same client never appear in both train and test,
  because a model that just memorizes "this client's pages behave like X" looks good but
  hasn't learned anything general.
- **Precision@50** — see §4. It's your one defensible number; ROC AUC and average precision
  are useful side numbers but not the one the task actually cares about.
- **Rate columns are already ×100 percentages** — `ctr = 0.76` means 0.76%, not 76%. Easy to
  misread once and quietly break every downstream number.

## 7. Rules that don't bend

- Never edit anything under `scripts/` except the feature lists in `ml_utils.py`.
- Never commit a dataset — CI fails the build if you do (that's `.github/workflows/smoke-test.yml`).
- Never hardcode your Hugging Face token in a notebook cell — the repo is public. `getpass`
  or Colab Secrets, always.
- Never claim you predicted Google's algorithm or proved a refresh *caused* recovery — the
  data is observational, so language stays observed/measured/directional/decision-support.
- Never publish client names, domains, URLs, or raw queries — only the pseudonymized hashes.

## 8. How we've been working together

You review and approve each notebook section before I write it (draft → your OK →
`NotebookEdit`); I never execute code myself since you run everything in Colab (no GPU
locally). After you run a notebook in Colab and save it back to GitHub, tell me and I'll pull
before making further edits — Colab and I are two different hands touching the same repo.
