# Data Science Portfolio

Four projects, built to demonstrate modelling, deployment, retrieval, and the
ability to turn a model into a number someone can act on. Every figure quoted is
reproducible from the repo, and each project has a test suite that a reviewer can
run.

| Project | What it demonstrates | Headline result |
|---|---|---|
| [01 Churn & business impact](https://github.com/l1ght14/01-churn-business-impact) | Classification, class imbalance, cost-based thresholding | PR-AUC 0.657 vs 0.265 baseline; $97k projected retention value |
| [02 Fraud detection, served](https://github.com/l1ght14/02-fraud-detection-serving) | Severe imbalance, model serving, CI, containers | 0.82 average precision; **90.7% fraud recall at 0.49% FPR** |
| [03 RAG knowledge assistant](https://github.com/l1ght14/03-rag-knowledge-assistant) | Hybrid retrieval, rank fusion, abstention | **Incomplete — see its [STATUS.md](03-rag-knowledge-assistant/STATUS.md)** |
| [04 Executive dashboard](https://github.com/l1ght14/04-exec-dashboard) | Star-schema SQL, metric governance, BI | Revenue reconciles to R$16,008,872.12 across 99,441 orders |

219 passing tests across the four repos.

## The thread running through all four

**A model score is not a result.** Each project exists to produce a number somebody
can act on, and each one forced a decision that a notebook would have hidden:

- **01** — the operating threshold is set by break-even probability
  (`offer_cost / cost_of_churn`), not by maximising F1. The same model supports a
  campaign targeting 49% of the base or 77% of it depending purely on what an offer
  is assumed to cost.
- **02** — ROC-AUC ranks the models *backwards* on a 0.17%-positive problem. The
  model with the best ROC-AUC had the worst average precision.
- **03** — abstention is the feature. A retriever that always returns something is a
  hallucination factory, so the threshold ships non-zero by default.
- **04** — revenue reconciles to the published olist total, which is the only
  available proof that the grain decisions and the payment rollups are right.

## What I got wrong, and how it was found

This is the part I'd most want a reviewer to read. Eight of the ten defects found
while building these were caught by **running** something rather than reading it.

The sharpest case: a 434-line star schema, carefully commented, in which **every raw
column name was invented.** The real dataset has `order_purchase_timestamp`, not the
`order_paid_timestamp` the code assumed, and `order_delivered_customer_date`, not
`order_delivered_timestamp`. The schema could not bind against real data at all.

The same file also shipped a `repeat_rate` that was **structurally 0.00%** — this
copy of olist has one generated `customer_id` per order, so the metric could never
move — and a region map covering 24 of Brazil's 27 states, which hid 5,782 orders and
R$1.03M in an `Unknown` bucket while the headline total still reconciled, so nothing
looked broken.

A synthetic test suite passed the whole time. **A fixture that invents columns is
worse than no fixture**, and a green suite is evidence about the fixture, not the data.

Two more, both found by executing against a throwaway dataset: DuckDB's `strftime` has
no `%q` quarter specifier (that is a Postgres extension), and DuckDB's `/` is true
division rather than integer division, so `(month - 1) / 3 + 1` returned `4.667` for
December. The SQL's own comment asserted the opposite of what the engine does.

In 02, computing the false-positive rate as `1 - precision` — the false *discovery*
rate — cut fraud recall from 91% to **1.3%** while still printing a clean-looking FPR
column. Both threshold bugs in that project are now caught by tests that measure the
achieved FPR independently of the code that computes it.

## Stack

Python 3.12 · pandas · scikit-learn · XGBoost · imbalanced-learn · FastAPI · DuckDB ·
Streamlit · Plotly · pytest · Docker · GitHub Actions

## Reproducing any repo

```bash
pip install -r requirements.txt
python -m pytest tests/ -q
```

01 and 02 additionally need their Kaggle dataset; the exact download command is in
each README. 04's `data/` is gitignored and the download command is in its README too.