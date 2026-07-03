# Assignment vs. Notebooks — Gap Analysis

_Generated 2026-07-03. Maps the AAA team assignment tasks against the current
state of the notebooks in `notebooks/`._

## Status at a glance

| # | Assignment task | Status |
|---|---|---|
| 1 | Data collection & preparation | 🟢 Mostly done |
| 1 | — Detailed trip-dataset description | 🟡 Partial (explicit TODOs) |
| 1 | — Multiple temporal discretisations (4-hourly/daily) | 🔴 Missing |
| 2 | — Demand patterns (start time, length, location, price) | 🟢 Done |
| 2 | — Average idle time between trips | 🔴 Missing |
| 2 | — Vary patterns across spatial resolution | 🟢 Done |
| 2 | — Vary across temporal bin sizes | 🟡 Partial |
| 2 | — Hot spots via GMM / KDE | 🟢 Done |
| 3 | — Feature engineering + validation strategy | 🟢 Done |
| 3 | — SVM (no-kernel → kernels → grid search) | 🟢 Done |
| 3 | — NN (feedforward) | 🟢 Done |
| 3 | — Census-tract as spatial unit | 🔴 Missing |
| 3 | — SVM vs NN comparison on holdout | 🟡 Partial (needs execution + verdict) |
| 4 | — Environment (DES) + MDP + Q-learning | 🟢 Done (MVP) |
| 4 | — DQN + robustness + baselines | 🟡 Partial (TODOs) |
| 5 | Discussion & Outlook (incl. private vs public charging) | 🔴 Not started |
| — | Report prose + figure embeds | 🔴 Scaffold only |

Legend: 🟢 sufficiently done · 🟡 partial · 🔴 missing/not started

---

## Task 1 — Data Collection & Preparation 🟢

**Done:** Large CSV pre-filtered before upload (nulls, zero trips, dropped
columns → 6.04M rows, `1.1` MD 2); dtype fixes, null analysis, H3 index at
**res 7 & 8**. Weather via ERA5/CDS API with merge, cleaning, unit conversion →
`weather_data.parquet` (`1.2`). POI via OSM, categorised, H3-aggregated at res
7 & 8 (`1.3`). Hourly grid join (`1.4`).

**TODO:**
- **Detailed dataset description** is explicitly unfinished — `1.1` MD 0 lists:
  "Provide a detailed description… no pending questions", "Add column
  description from website", "Add Outlier Analysis", "Miles to km".
- **4-hourly / daily temporal discretisation** — only hourly exists. Assignment
  asks to "consider different temporal discretization (hourly, 4-hourly,
  daily)". No `resample`/coarser bins found.
- Notebook hygiene: `1.2` MD 1 has cleanup TODOs (`uv add` packages, naming,
  move non-weather feature engineering out).

## Task 2 — Descriptive (Spatial) Analytics 🟢/🟡

**Done:** Demand patterns across start time, trip length, price, start/end
locations (`2.1`, `2.2`). Spatial-resolution comparison across community area /
census tract / H3-7/8/9 with the key insight that privacy-masked centroids mean
H3 adds aggregation levels, not detail (`2.1` MD 7). **Hot spots via
`GaussianMixture` + `KernelDensity`** (`2.1` cells 31–36) — satisfies the
GMM/KDE requirement.

**TODO:**
- **Average idle time between trips** — not implemented anywhere (requires
  per-`Taxi ID` sequential gap analysis). Explicitly named in the assignment.
- **Temporal bin-size variation** — only hourly / weekday-weekend; no
  4-hourly/daily comparison.
- Interpretation ("give possible reasons") is thin — a few markdown notes only.
- `2.3 TempHexagonVis` is self-marked "Remove this notebook when functions are
  copied into spatial analytics" — redundant.

## Task 3 — Predictive Analytics 🟢/🟡

**Done:** Feature engineering (`3.1`) — spatial (distance-to-Loop), cyclical
time, calendar/holiday, weather, POI, and a leakage-safe `base_demand`.
**Validation strategy** is solid: temporal 50/20/30 split, explicitly justified
against autocorrelation/leakage (MD 12). Target is correctly
**non-autoregressive**. SVM (`3.2`) follows the exact recipe: linear → poly →
RBF then `GridSearchCV` for C/γ, across feature sets, for res 7 & 8. NN (`3.3`)
— dynamic feedforward `FNN`, LR/architecture grid, early stopping, res 7 & 8.

**TODO:**
- **Census-tract prediction unit** — no `census` reference in `3.*`. Assignment
  explicitly requires predicting for "hexagon **and** census tract" and asks
  "how does performance change when you only use census tract?" Currently
  H3-only.
- **SVM vs NN verdict** — `3.4` has the machinery (metrics table, R² bars, hex
  error maps, pred-vs-actual, and it does reference the NN) but the `fig-svm-*`
  cells **have no saved outputs** — the notebook must be run before anything
  embeds, and the "is deep learning worth it?" conclusion isn't written.
- Improvement levers / feature importance are TODOs (`3.2` cell 15: SHAP,
  coarser H3, Tweedie/quantile loss for the zero-inflated target).
- Much of `3.3`'s evaluation viz (cells 14–15) is commented out.

## Task 4 — Smart Charging RL 🟢/🟡

**Done:** `SmartChargingEnv` discrete-event sim, MDP fully specified (state
`(b_t,t)`, actions {0, 7.33, 14.67, 22} kW, stochastic `D~N(30,5)` at
departure), tabular `QLearningAgent`, 5000-episode training, `fig-policy-heatmap`
(the one figure with saved output), and evaluation vs a naive baseline.
Exponential cost uses `np.exp` as specified.

**TODO** (notebook's own list): describe Env/Agent/Training cells; make
`alpha_t` time-varying; add stronger baselines; multi-seed robustness; and a
**DQN** agent for comparison.

## Task 5 — Discussion & Outlook 🔴

Not started (belongs in the report, not notebooks). Needs client implications,
further analyses, external data sources, and a **private-vs-public charging**
recommendation. Section `sections/06-conclusion.qmd` is scaffolded for it.

---

## Cross-cutting issues (reproducibility)

1. **Broken data paths / naming drift** — EDA notebooks read
   `../data/taxi_data_processed.parquet` and `../data/chicago_pois_agg.csv`, but
   `1.1`/`1.3` now write to `../data/processed/…` with suffixed names
   (`chicago_pois_agg_{7,8}.csv`). `1.4` also reads the old
   `chicago_pois_agg.csv`. **Several notebooks won't run top-to-bottom as-is.**
2. **H3 column-name mismatch** — `1.1` emits `h3_index_pickup_7/8`; `2.1`
   references `h3_7_index_pickup`, `h3_index_pickup`, `h3_9_index_pickup`.
3. **Embeds** — only `fig-policy-heatmap` has output. All `3.4` figures need
   execution before the report can show prediction results.

## Suggested priority order

1. Fix the data-path/column mismatches so notebooks run end-to-end (blocks
   everything downstream).
2. Add the two genuinely-missing analyses: **idle time between trips** (Task 2)
   and **census-tract prediction** (Task 3).
3. Execute `3.4` and write the SVM-vs-NN verdict; enable the embeds.
4. Add 4-hourly/daily temporal bins where feasible.
5. Extend RL (DQN, baselines) and write Task 5.
6. Fill the report prose in `sections/`.
