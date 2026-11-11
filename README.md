# Card data quality and reconciliation controls

A small, working data governance project: profile the data, run SQL-based quality controls, reconcile
to independent control totals, quarantine bad records, and report findings. Covers the usual data-governance loop: discover, validate, reconcile, document, report.

Data is synthetic (20k card transactions, 2k customers, daily control totals) with **planted defects and an
answer key**, so the controls themselves can be graded, not just run. The same author wrote the data generator and the
controls, so the grading shows the controls behave as designed, not that they would generalise to real data.

## Run it
```
python -m engine.run      # runs everything, writes outputs/ and regenerates docs
python -m pytest -q       # 28 tests
```
Setup: `pip install -r requirements.txt`. SQL runs on SQLite (standard library).
Open `outputs/dq_report.html` for the result.

## Results on the supplied data
* 18 controls, 10 fail, overall RED. 365 of 20,015 transaction rows (1.8%) held back.
* **18 of 18 controls match the answer key exactly**; the 8 controls with nothing planted raise no false alarms.
* Reconciliation catches **4 of 5** tampered days with no false positives. It misses 2026-08-21 (8 amounts +50% sit
  inside normal daily noise) - documented as a blind spot with a recommended fix, and locked in by a test.

## Layout
| Path | What |
|---|---|
| `rules/rules.yaml` | Control register: id, dimension, severity, owner, rationale, tolerance, SQL. The single source of truth |
| `engine/` | `loader` (CSV to SQL, source line numbers), `rules` (run controls), `reconcile`, `profile`, `scorecard`, `report`, `docs_gen`, `run` |
| `tests/` | Unit tests on a hand-built dataset plus end-to-end checks against the answer key |
| `outputs/` | `audit_log.csv` (append-only, one row per control per run), `dq_report.html`, `control_results.csv`, `exceptions.csv`, `reconciliation.csv`, `profile.csv`, `scorecard.csv`, `quarantine_summary.csv`, `run_summary.json` |
| `docs/` | `findings_memo.md`, `data_dictionary.md`, `governance_standards.md`, `automation_and_gcp.md`, `control_register.md` (generated) |
| `gcp/bq_dq_checks.sql` | The same controls as a BigQuery scheduled query (generated, not run - no cloud access here) |

## What each part demonstrates
| Area | Where it shows |
|---|---|
| Data discovery and validation | `profile.py`, `loader.py`, 18 SQL controls |
| Quality controls with monitoring | `rules.yaml` (severity, tolerance, owner), `rules.py`, `audit.py` (run history) |
| Finding issues by analysis | `reconcile.py`, `findings_memo.md` (including what the controls cannot see) |
| Documentation and standards | `governance_standards.md`, `data_dictionary.md`, generated control register |
| Automation | one-command run, docs and SQL generated from the YAML, `automation_and_gcp.md` |
| Risk mindset | severity model, quarantine instead of silent fixes, an owner per control, controls graded against known defects |

## Limits
* Synthetic data; owners and thresholds are examples, not real policy.
* Negative amounts are treated as errors because there is no transaction-type field (open question in the data dictionary).
* BigQuery SQL is generated and syntax-adapted but has never been run on BigQuery.
* Owners such as "Payments data owner" are placeholders, not real teams.
* Daily-total reconciliation cannot detect small amount changes; row-level hashing would.
