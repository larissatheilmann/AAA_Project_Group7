# AAA Report — Content Guide per Section File
Matches the syllabus's five lettered report sections (2.2, a–e) as the five top-level `.qmd` files. Everything else — descriptive/predictive/RL analyses, individual data-prep notebooks — is nested as subsections underneath, per (d) Data Analytics being one syllabus item covering all analytical methods.

---

## `01-introduction.qmd` — (a) Problem Description
- Framing established: team acts as the Data Science Team of "Bane & Wayne Partners" (BWP), a management consultancy advising a German car manufacturer building a US ride-hailing platform on a fully electrified fleet
- Business gap stated precisely: the client can design/build vehicles but has no operational experience running a ride-hailing fleet, and none running an *electric* one where charging constrains availability
- Chicago taxi data justified as proxy: metered, on-demand urban trips subject to the same weather/time-of-day patterns the client's fleet will face — explicitly *not* claimed to be ride-hailing data itself
- Two downstream decisions given as the report's organizing spine: (1) where/when to expect demand at a fleet-planning resolution, (2) how a vehicle should charge at a driver's home given uncertain next-shift energy needs
- **Five research questions**, each feeding into the next — maps 1:1 to the analysis, though no longer 1:1 to file count now that Data Analytics is one section:
  1. How does demand vary across space/time, and what explains patterns across spatial units (census tract vs. H3 res 7/8) and temporal bins?
  2. Where are persistent spatial hot spots, per GMM and KDE?
  3. Can demand be predicted per spatial unit × time bucket (not extrapolated from the prior hour), and how does SVM compare to a feedforward NN?
  4. What charging policy should a driver follow at home, given adjustable 15-min charging rate, finite battery capacity, and probabilistic next-shift energy demand?
  5. What do the findings imply for the client's fleet operations — specifically, private home charging vs. public charging infrastructure?
- Report structure paragraph uses Quarto cross-ref labels (`@sec-data`, `@sec-eda`, `@sec-prediction`, `@sec-rl`, `@sec-conclusion`) — **now that Data Description and Data Preparation are separate files again, and Data Analytics is one merged section, update this paragraph's labels/wording to match**: likely `@sec-data` (Data Description) → also needs a `@sec-data-prep` or similar for Data Preparation, and `@sec-eda`/`@sec-prediction`/`@sec-rl` all need to become subsection references within one Data Analytics section rather than separate top-level ones
- Still to add per the syllabus requirement: an explicit one-line statement of the **data mining goal** separate from the business goal (descriptive / predictive / prescriptive breakdown)

---

## `02-data-description.qmd` — (b) Data Description
*(Content: the "Data Sources" overview material — what the datasets are, where they're from, size/coverage. Detailed cleaning/processing steps belong in Data Preparation, not here.)*

- **Taxi trips**: Chicago Data Portal — **6,041,177 trips, January 2024 through April 2026** — 16 fields retained (identifiers, timestamps, duration, distance, tract/CA codes, fare, company, coordinates)
- **Weather**: ERA5-Land reanalysis via the Copernicus Climate Data Store API, single coordinate (42, -87.8) rather than a station network, hourly, **Jan 1 2024 – May 12 2026**, 6 variables (2m temp, total precip, snow cover, snow depth, u/v wind at 10m) — explicitly justified as a modeled reconstruction trading station-level accuracy for complete gap-free coverage
- **POI**: OpenStreetMap via OSMnx, "Chicago, Illinois, USA" boundary, 3 tag families (amenity/shop/tourism) — a **snapshot of today's map**, not historically tied to the trip period; worth flagging this as a limitation
- **Community area boundary layer**: 77 official areas, used only for spatial reference/choropleth mapping — distinct from the tract/CA codes already present in the trip records themselves
- Good candidate for a summary table: source × size × resolution × date range across all four datasets
- Note any known data quality issues up front here (missing periods, coverage gaps) — a preview of what's addressed in more depth in Data Preparation

---

## `03-data-preparation.qmd` — (c) Data Preparation
*(Content: everything that turns the raw sources above into analysis-ready datasets — the taxi cleaning, weather processing, POI pipeline, and spatial/temporal discretisation, plus the feature-engineering step that sits just before modeling.)*

### Taxi Trip Cleaning
- Explicit: dropped Fare/Tips/Tolls/Extras/Payment Type/centroid columns as redundant with lat-lon; locale-aware numeric parsing (comma-decimal source formatting); rows with nulls in key fields dropped (<0.02% share, no imputation needed); rows missing both Community Area and Tract dropped, justified via a co-missingness check
- Explicit: two parallel outputs — `taxi_data_processed_big` (CA-complete) for Community Area analysis, `taxi_data_processed_small` (tract-complete) for Census Tract analysis — this split propagates through the whole pipeline
- Explicit: outlier filtering limited to sanity checks only (no statistical outlier removal), reasoned that extreme values (e.g. airport transfers) may be legitimate
- Explicit: H3 indices (res 7/8/9) derived per-row from lat/lon
- Note but don't dwell on: raw lat/lon columns kept "in case needed" rather than a firm decision (unresolved TODO in the notebook); no dedup check performed on Trip ID despite it being the nominal unique key

### Weather Processing
- Explicit: single fixed coordinate point representing all of Chicago (no spatial variation in weather modeled) — worth stating as a simplifying assumption in the report
- Explicit: precipitation/snow-depth clipped to ≥0 (floating-point noise correction); unit conversions (K→°C, m→mm)
- Note: wind speed/direction derivation actually happens later in feature engineering, not here — if you describe the weather pipeline as one step, be accurate about where this split occurs
- Justify the chosen temporal resolution (hourly) and its effect on downstream predictive power, per the assignment brief's explicit prompt

### Points of Interest — fully drafted, use as-is
- Justification: POI density is an established proxy for ride demand (restaurants/nightlife → evening trips, transit/offices → commute trips) — good framing since the assignment brief treats POI as optional and expects a justification
- Pipeline: 43,237 raw points → combined into one `poi_type` (amenity, falling back to shop, then tourism) → 311 distinct raw types → manually mapped to **14 categories** → 55 unmapped raw types dropped as "other" (23,129 of the 43,237 points), leaving **20,108 categorized POIs**
- Aggregated to H3 at both resolutions: **152 hexagons** have ≥1 POI at res 7, **806 hexagons** at res 8
- Explicit downstream split: **res-7 POI feeds Data Analytics/prediction**, **res-8 POI feeds Data Analytics/descriptive EDA only** — consistent with h3_8 being excluded from the modeling feature set entirely
- Category mapping encodes a judgment call about what counts as "demand-relevant" (e.g., benches/parking/toilets excluded) — worth one sentence acknowledging this silently shapes what features are even available downstream

### Spatial and Temporal Discretisation (core grid build) — the most decision-dense step
- Explicit, memory-driven: narrow column subset read per file to avoid a ~22GB load; timestamps floored to hour and raw strings dropped immediately; taxi ID/company cast to categorical
- Explicit: taxi data truncated to end at the weather series' last covered hour, to prevent frozen/forward-filled weather values leaking into a temporally-split test set
- Explicit: Community Area level uses the "big" dataset, every other level uses "small" — because H3 computed on CA-only rows would snap to the coarser CA centroid rather than the tract centroid
- Explicit, quantified caveat: **267 of 801 census tracts never appear in the spatial universe** (too small for a res-8 hex), costing ~1.2M trips (~18% of the census-tract set) — flagged in the pipeline rather than fixed; **this belongs in the report's limitations, not just buried in code comments**
- Explicit: asymmetric empty-cell handling by column — trip counts filled with true zero; most-common-company filled with empty string; average fare/miles/seconds deliberately **left as NaN** ("there is no average of no trips")
- Explicit: empty-cell pickup lat/lon back-filled from the spatial unit's geometric centroid (a structural fill, not a demand estimate)
- Explicit: units with zero pickups across the entire ~2.4-year window dropped entirely (~half of res-8 hexes), verified via an explicit assertion that no actual trips are lost by this reduction
- Explicit: active-unit set derived once per spatial level (not per grid) so hourly and daily grids share the same unit universe
- **Explicit justification needed for choosing resolutions 7 and 8 specifically** — the assignment asks for this reasoning, not just the result
- **Important finding to include here**: raw lat/lon reduces to only ~634 unique coordinate pairs citywide — H3 indices carry **no additional spatial precision beyond tract/CA already known**. A genuinely important caveat for any spatial-resolution claim made later in Data Analytics
- Community Area data is actually *less* complete than Census Tract in this extract (counter to naive expectation) — trips crossing the city boundary carry a tract but no CA
- Demand concentration is extreme at every spatial level (Gini 0.94–0.97, top 10% of units carry ~99% of pickups) — a property of the data, not an artifact of grid choice; feed this forward into Data Analytics rather than only stating it here

### Feature Engineering (sits between prep and modeling)
- Explicit: temporal (not random) train/val/test split, **50/20/30** by unique calendar date, to avoid autocorrelation leakage
- Explicit: `base_demand` (historical mean keyed by spatial unit × hour/day-of-week/month) computed only on the training split, merged into all splits — this later turns out to be the most relied-upon feature in the SVR importance analysis
- Explicit: `distance_to_loop` uses a hardcoded downtown reference point (state this assumption plainly)
- Explicit: US/IL holiday calendar, "near holiday" defined as a ±1-day window
- Explicit: columns flagged as leakage risks (trip counts, avg fare/miles/seconds, raw lat/lon) dropped rather than imputed
- **Important scope note for Data Analytics**: `h3_8` is computed and explored extensively in prep/EDA but is **excluded from feature engineering and never modeled** — only h3_7, census tract, and community area are actually used in SVM/NN

### Open item to resolve before finalizing this section
- One EDA notebook states the final week of data should be excluded due to incomplete collection — but no such truncation exists anywhere in the actual pipeline. Confirm whether it's genuinely needed and, if so, implement it — otherwise this data-quality issue is silently present in every model's train/val/test split

---

## `04-data-analytics.qmd` — (d) Data Analytics
*(One syllabus section covering all analytical methods and their evaluation. Use `##` subsections for Descriptive, Predictive, and Reinforcement Learning rather than separate files/top-level sections.)*

### 4.1 Descriptive (Spatial) Analytics
- Demand pattern breakdown: start time, trip length, start/end location, price, idle time — shown across resolutions (tract vs. hex, coarse vs. fine time bins)
- Reasoned interpretation of each pattern, not just the chart
- Spatial hot-spot identification using Gaussian Mixture Models / Spatial Kernel Density Estimation, with method justification
- Key result: hot-spot map(s) at the chosen resolution(s)
- **Lead with the demand-concentration finding** carried over from Data Preparation (Gini 0.94–0.97, top 10% of units ≈ 99% of pickups) — it's a genuine, well-evidenced result, not just a modeling footnote

### 4.2 Predictive Analytics (SVM & Neural Networks)

**Data setup** (state once, applies to both models):
- Trained on **six datasets**: 3 spatial resolutions (h3_7, census tract, community area) × 2 temporal resolutions (daily, hourly)
- Target is always `Total_Trip_Start`; prediction is per spatial unit × time bucket, not autoregressive
- Zero sampling: balanced sample (all non-zero + equal zero) for both models; Community/daily is a near-exception at only ~0.6% zero rows
- Baseline: historical mean by (hour/month/day-of-week) or (month/day-of-week) — the `base_demand` feature from Data Preparation, reused here
- Train/val/test split: 50/20/30, temporal
- Feature sets: SVM compares Basic / +weather / +POI / +weather+POI; NN uses Basic+weather+POI only (note explicitly why NN wasn't run across the same ablation, e.g. compute budget)

**Support Vector Machine — shortfalls & results (updated Jul 20 against a recomputed, stronger baseline — supersedes the earlier `residual/level_comparison_mae.csv` numbers and the skill figures previously drafted here; see `docs/svm-model-performance-report.md`)**:
- **Headline finding, revised**: against the stronger baseline, SVR clears it at only **2 of 6 levels**. **Best model overall: residual SVR on community/daily** — basic+poi features, linear kernel, C=1; skill (MAE) **+9.4%**, R² = 0.977. Runner-up: **residual SVR on h3_7/daily** — basic+poi, linear, C=1; skill **+5.1%**. The other four levels (both hourly levels, both census levels) lose to the baseline even after the residual reframing.
- **Second-best model: residual SVR on h3_7/daily** — 119 hexagons, 51,289 training rows, mean demand 58.9 trips/day, 79.0% zero rows. Same recipe as the winner (basic+poi features, linear kernel, C=1), skill (MAE) **+5.1%**, skill (RMSE) **+9.0%** (a larger relative gain on RMSE than MAE, consistent with the residual model correcting the baseline's largest misses rather than shaving typical error), R² = 0.963. The direct model at this level *loses* to baseline (−5.5% MAE, −3.4% RMSE), so the residual reframing is what turns this level from a loss into the runner-up. `base_demand` is genuinely load-bearing here (ablating it costs +166% MAE, not a "reliance without necessity" case like the two daily levels below), and `location` is the single most important other block — dropping it costs +29.0% MAE, well above calendar (−0.3%, i.e. mildly unhelpful) — so the win is really being driven by which hexagon and how far into the baseline-corrected residual the model can push, not by weather or POI features (neither block was picked into the winning feature set).
- **Worst: census/hourly** — the sparsest, most zero-inflated level (97.1% zero rows) — loses badly under both strategies (direct −35.2%, residual −23.5%). MAE skill is strongly negative while RMSE skill is roughly flat, the signature of a mismatch between the class-balanced training sample and the natural-prevalence test/validation evaluation, not just a weak model.
- Residual-on-baseline reframing beats the direct `log1p(demand)` model at **all 6 levels** (skill MAE), but only two of those wins clear the baseline outright; direct models now lose the baseline at all 6 levels.
- Daily still beats hourly at every spatial resolution, under both strategies.
- **Kernel choice, revised**: the earlier "RBF is empirically forced" finding no longer holds under the residual target. 4 of 6 winning residual models pick a **linear** kernel (both h3_7 levels, both community levels); only census (hourly and daily) still selects RBF. All direct-target winners remain RBF, which is where the `expm1` back-transform blow-up argument still applies.
- `base_demand` reliance: **load-bearing** (ablating it clearly hurts) at 4 of 6 levels — h3_7/hourly (+27% MAE), h3_7/daily (+166%), census/hourly (+210%), community/hourly (+72%). At the two coarsest daily levels it's **reliance without necessity**: permutation importance looks enormous (+29,282% at census/daily, +2,700% at community/daily) but dropping the feature costs only +3.8% at census/daily and actually *improves* MAE by 5.0% at community/daily — the model can reconstruct it from location/calendar features instead.
- Beyond `base_demand`, `location` is the next most influential block — dominant at h3_7/daily (+29% MAE if dropped), census/hourly (+11%), and community/hourly (+32%); `poi` matters specifically at community/daily (+22%).
- Improvement levers for follow-up: weather×hour/spatial interactions, finer/spatially-resolved weather, richer POI features (distances, not just counts); natural-prevalence training is now a priority, motivated directly by the census/hourly zero-inflation finding above.
- **Open item**: the test-set evaluation (3.4 notebook) is stale relative to this validation re-run — model selection changed (best is now community/daily residual, not the earlier h3_7/daily direct result), so test numbers must not be cited in the report until 3.4 is re-run against the current best models. NN comparison is still blocked on a missing `models_predictions/ANN/standard/models/h3_7/daily/best.joblib`.

**Neural network — still largely TODO**:
- Architecture: grid search over hidden-layer counts/widths, "pyramid" pattern (e.g. 64→32→12) — **TODO: state winning config per dataset and why**
- Three learning rates tested (**TODO: state actual values**); AdamW instead of plain L2 (**TODO: reasoning**); ReLU + linear output (**TODO: reasoning**); 15 epochs, batch size 250 (**TODO: justification**); early stopping (patience 3); dropout for generalization

**Model Comparison**:
- Both SVM and NN evaluated independently across all six levels on validation; the best-performing model(s) are then compared on the test set exactly once
- Number of head-to-head test comparisons depends on validation results: one if the same level wins for both architectures, more than one otherwise (flag explicitly if comparing across different resolutions, since that conflates model and resolution effects)
- Planned figures: predicted-vs-actual scatter, spatial maps of MAE/mean relative error/R² per hexagon
- Translate error metrics into business terms (e.g., what a given MAE means for fleet over/under-provisioning) — and note H3 doesn't add real spatial precision beyond tract/CA in this dataset, so any "finer resolution helps" claim needs that caveat

**Evaluation questions**:
- *Is complexity worth it?* — TODO: analyze NN convergence/loss curves for over-/underfitting; consider the equivalent question for SVM kernel complexity
- *Is deep learning worth it?* — can only be answered once the final test-set comparison lands; may legitimately be resolution-dependent rather than a single yes/no

**Shortfalls & improvement levers**:
- NN: hyperparameter search narrowed by compute constraints; follow-up could explore more layer variations, AdamW parameter settings, alternative activations (team considers 3 learning rates sufficient)
- SVM: fixed-size sampling constraint from O(n²) fitting cost; two-stage tuning only explores the winning kernel's grid, not all three

### 4.3 Smart Charging (Reinforcement Learning)

**Problem Formulation (MDP)** — design is set, write-up still needed:
- State: `(b_t, t)` — battery level `b_t ∈ [0,50]` kWh, timestep `t ∈ {0..8}` (14:00–16:00 in 15-min steps)
- Actions: `{0, 7.33, 14.67, 22}` kW (zero/low/medium/high — even quarters of max power)
- Transition: deterministic charging for t<8; stochastic demand `D ~ N(μ=30, σ=5)` kWh revealed at t=8
- Reward: negative recharging cost plus a large penalty for failing to cover D
- Cost coefficient currently **constant** (α=0.0883 $/kWh, EIA Illinois retail rate) — the assignment's own cost formula supports time-varying α; flagged in the notebook as a TODO

**Environment & Agent** — still TODO:
- `SmartChargingEnv`: 50 kWh capacity, 22 kW max power, 8 fifteen-minute steps
- Tabular `QLearningAgent`: Q-table over (step, battery), epsilon-greedy; hyperparameters LR=0.1, γ=1 (justify: no discounting makes sense given the short fixed horizon), epsilon=0.1
- Training: 5,000 episodes — needs convergence discussion

**Learned Policy**:
- Heatmap already embedded (`fig-policy-heatmap`) — needs interpretation of when/how aggressively the agent charges as a function of remaining time and battery level, translated for a non-technical reader

**Evaluation** — still TODO, notebook itself flags these as outstanding:
- ~1,000 simulated days: failure rate + average cost vs. a naive baseline
- Time-varying α_t (currently the single biggest simplification — likely understates the value of an intelligent agent versus a flat-rate baseline; worth prioritizing if time is limited)
- Stronger baselines, multi-seed robustness, DQN comparison against the tabular MVP (assignment brief suggests DQN as the expected approach)

---

## `05-conclusion.qmd` — (e) Conclusions and Discussion
*(All four subsections below are still TODO in the draft, but the material mostly already exists across Sections 2–4 — this is a synthesis job.)*

### Summary of Findings
Recap the five research questions from the Introduction against what Data Analytics actually found:
1. Demand patterns: extreme concentration (Gini 0.94–0.97), Thursday peak / weekend trough
2. Spatial hot spots: GMM/KDE results
3. Prediction accuracy: SVR beats the (stronger, recomputed) baseline at only 2 of 6 levels — best is community/daily (+9.4%), h3_7/daily (+5.1%); worst is census/hourly (−23.5% to −35.2%); NN comparison pending (test re-run + NN model still outstanding)
4. Charging policy: tabular Q-learning MVP, under a constant electricity price
5. Client implications: covered in the next subsection

### Implications for the Client
- Demand concentration → concentrate initial fleet deployment rather than spreading evenly
- **Explicit private vs. public charging recommendation required** — should draw directly from RL results; be honest if the recommendation currently rests on a simplified constant-price cost model
- Translate every technical result into an explicit "so what" sentence per the assignment's own instruction

### Limitations — compiled from what's already surfaced across Sections 2–4
- Taxi-as-proxy assumption (from the Introduction) — revisit explicitly here
- Spatial masking: raw coordinates reduce to ~634 unique points citywide — H3 doesn't add real precision beyond tract/CA
- Coverage gap: 267 of 801 census tracts never appear in the spatial universe (~18% of census-level trips silently dropped)
- POI is a present-day snapshot, not historically tied to the 2024–2026 trip period
- Weather has no spatial variation (single citywide coordinate)
- Predictive ceiling appears largely inherent to the assignment's non-autoregressive framing
- RL's constant electricity price likely understates the value of an intelligent agent
- RL is a tabular Q-learning MVP, not yet the DQN the assignment suggests; single-seed results
- Open data-quality item if unresolved: the final-week-of-data exclusion claim never actually implemented in the pipeline

### Future Work
- Natural-prevalence training for SVR
- Time-varying electricity pricing in the RL cost model
- DQN comparison against the tabular baseline
- External data sources: real ride-hailing data (validating the taxi-proxy assumption), time-of-use electricity tariffs, real-time/forecasted weather for operational deployment
- Multi-seed robustness across both SVR and RL results

### Note on appendix
Good candidates already flagged elsewhere: full SVR hyperparameter grids, h3_8 POI aggregation (used in EDA but not modeling), location-reliability analysis supporting plots if not already embedded in the main body.

---

## Cross-cutting reminders (not tied to one file)
- Every team member needs at least one authored Git commit, or the team fails the assignment (syllabus G.3 / §2.2)
- Any LLM-generated code or text must be explicitly marked, including the prompt used
- 10-page limit is exclusive of figures, references, and appendix — push detailed tables/hyperparameter grids to the appendix
- All Quarto cross-reference labels (`@sec-...`) need to be updated in the Introduction's "Report Structure" paragraph to match this 5-section structure