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
*(Content: the "Data Sources" overview material — what the datasets are, where they're from, size/coverage. Detailed cleaning/processing steps belong in Data Preparation, not here. Verified directly against `notebooks/1.1 TaxiDataPrep.ipynb`, `1.2 WeatherDataPrep.ipynb`, `1.3 POIDataPrep.ipynb`, and `1.4 TemporalSpatialDiscretization.ipynb` — several figures below correct stale numbers previously drafted here.)*

- **Taxi trips**: Chicago Data Portal, the "Taxi Trips (2024– )" dataset. Raw download is **16,086,255 rows × 23 columns**; after dropping 7 redundant/unused columns (Fare, Tips, Tolls, Extras, Payment Type, and the two centroid-location point columns — kept as separate lat/lon fields instead) **16 fields remain**: trip and taxi ID, start/end timestamp, trip seconds/miles, pickup/dropoff census tract and community area, `Trip Total` (the retained cost figure — not `Fare` alone, it's fare+tips+tolls+extras combined), company, and pickup/dropoff lat/lon.
  - **Correct the row count previously drafted here (6,041,177) — it does not match any actual pipeline output.** After cleaning, the pipeline branches into two parallel datasets (see Data Preparation): a **community-area-complete set of 14,494,804 trips** and a stricter **census-tract-complete set of 6,835,451 trips**, both further trimmed in Data Preparation to align with the weather series' coverage window (14,108,693 and 6,636,374 trips respectively — see below).
  - Date range actually spans **January 1, 2024 through May 31, 2026** (not "through April 2026").
  - Raw-trip-level missingness is severe and asymmetric: **55.3% of trips are missing a pickup census tract, 56.6% a dropoff** one (privacy suppression + trips ending outside Chicago), versus only **2.8%/8.8%** missing a pickup/dropoff *community area* — this asymmetry is exactly why the pipeline keeps two parallel datasets rather than one.
- **Weather**: ERA5-Land reanalysis (dataset `reanalysis-era5-land-timeseries`) via the Copernicus Climate Data Store API, single coordinate (42, -87.8) rather than a station network, hourly, **Jan 1, 2024 – May 12, 2026** — **20,712 hourly rows**, 6 raw variables (2m temperature, total precipitation, snow cover, snow depth, and the u/v components of 10m wind) — explicitly justified as a modeled reconstruction trading station-level accuracy for complete gap-free coverage. Worth citing as a positive contrast to the taxi/POI data: **zero missing values** across the entire series. Temperatures over the observation window range from about **−25°C to +34°C**, a plausible Chicago range worth a one-line sanity-check mention. (The source notebook itself carries a self-flagged TODO list — "clean this notebook," "create a logical order" — worth a brief, low-key caveat that this pipeline stage is rougher than the others, even though its output is complete and used correctly downstream.)
- **POI**: OpenStreetMap via OSMnx, "Chicago, Illinois, USA" boundary, 3 tag families (amenity/shop/tourism) — **43,237 raw points**, downloaded with **823 raw OSM tag columns** before trimming to the handful actually used (illustrates how heterogeneous/sparse OSM tagging is — most of those 823 columns are near-empty). A **snapshot of today's map**, not historically tied to the trip period; worth flagging this as a limitation. Full cleaning/categorisation pipeline described in Data Preparation.
- **Community area boundary layer**: 77 official areas, used for spatial reference/choropleth mapping and to spatially join POIs to census tract/community area — distinct from the tract/CA codes already present in the trip records themselves. Note for accuracy/reproducibility: different notebooks source this boundary two different ways — some (POI, early EDA) expect a dedicated `chicago_community_areas.gpkg`, while `3.4 ModelPerformanceVisualization.ipynb` notes that file **does not exist** and falls back to dissolving the census-tract file by `commarea` instead. Worth a one-line data-provenance caveat rather than silently picking whichever happens to load.
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

### Points of Interest — verified against `notebooks/1.3 POIDataPrep.ipynb`; corrects/extends the previous draft
- Justification: POI density is an established proxy for ride demand (restaurants/nightlife → evening trips, transit/offices → commute trips) — good framing since the assignment brief treats POI as optional and expects a justification.
- Extraction: OSMnx query against a rotating list of 3 Overpass mirrors (a 10-minute timeout, retried mirror-by-mirror) — worth a one-line mention as a concrete resilience measure against the public Overpass endpoint being overloaded for a city-wide, 3-tag-family query, not just a throwaway detail. Raw download: 43,237 points with **823 raw OSM tag columns** (illustrates how sparse/heterogeneous OSM tagging is — almost all of those columns are used for nothing) before trimming to the handful of relevant fields.
- Pipeline: combined `amenity`/`shop`/`tourism` into one `poi_type` (amenity, falling back to shop, then tourism — missing counts: amenity 7,053, shop 37,552, tourism 41,738; 138 unique amenities alone) → **311 distinct raw types**, none left unmapped after combining → manually mapped to **14 categories** → 55 unmapped raw types dropped as "other" (23,129 of the 43,237 points), leaving **20,108 categorized POIs**.
- **Full category breakdown** (before dropping "other"), worth a table or bar chart rather than just the aggregate 20,108 figure: food_drink 4,984, civic_community 2,654, shopping 2,184, transport 1,650, grocery 1,509, education 1,405, entertainment 1,259, services 1,186, nightlife 979, health 762, finance 695, automotive 451, lodging 230, leisure_sports 160 (plus other 23,129, dropped).
- **Verified, concrete example of the "judgment call" caveat** — don't just gesture at it, cite it: the raw type `cosmetics` used to be listed under *both* the `shopping` and `services` category rules in the source code, silently resolving to `services` only via dict-overwrite order. Fixed (the dead `shopping` entry was removed and the intent documented inline), but worth citing as a concrete, verified instance of exactly the "this silently shapes what features are downstream" risk already flagged qualitatively.
- POIs are matched to **H3 (resolution 7), census tract, *and* community area**. h3_index_8 was computed and aggregated in earlier versions of the pipeline but has since been **removed entirely** (it fed only `notebooks/2.3 TempHexagonVis.ipynb`, which was itself broken and has been deleted — nothing else consumed it, not even Data Analytics/EDA). The centroid of each POI geometry (about half of all POIs are building polygons rather than points, so this matters; centroids are computed in a projected CRS, EPSG:3435, not directly in lat/lon) is spatially joined against the same tract/community-area boundary file the taxi grid uses in Data Preparation, so POI and taxi features share identical spatial units. 43,156 of 43,237 POIs (99.8%) match a tract; 81 fall outside the official city boundary (OSMnx's query returns a slightly larger bounding area than the official limits) and are excluded from tract/community aggregation only, not from the H3 level (which is defined continuously over any lat/lon).
- **Final aggregated coverage, all three spatial levels**: h3_7 — 152 units, 20,108 POIs; census — **797 of 801 tracts**, 20,084 POIs; community — **all 77 areas**, 20,084 POIs. (The 24-POI gap between the H3 and tract/community totals is the residue of the 81 POIs that fall just outside the official boundary.)
- **Downstream use**: POI is aggregated and used at **h3_7, census tract, and community area alike** (all three feed the SVM/NN `POI_COLS` feature block, per `3.2 Prediction_CommonGround.ipynb`) — there is no separate EDA-only POI resolution anymore now that h3_8 is gone.
- Two output files per level, not one: `chicago_pois_agg_<level>.csv` (overall counts + the list of POI types/names present) and `chicago_pois_category_wide_<level>.csv` (one count column per category — this is the actual feature matrix consumed downstream, not the `_agg_` file). A third export, `chicago_pois_cleaned.csv`, has been removed — it was written but never read by anything.
- A Spearman correlation heatmap of POI category co-occurrence across hexagons is already produced in the notebook (which categories cluster in the same areas), now labelled `fig-poi-category-corr` — a ready-made supporting figure if Data Analytics wants a POI-structure visual beyond raw counts. The other POI visualizations in `1.3` are now similarly labelled (`fig-poi-top-types-raw`, `fig-poi-top-types-categorized`, `fig-poi-hex-count`, `fig-poi-hex-category`) and embeddable directly via `{{< embed >}}`.

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
- **Verified demand scale** (worth stating once, so the wide y-axis on the prediction plots isn't mistaken for a bug): `Total_Trip_Start` is a plain pickup count per unit per bucket (`groupby([pickup_time, pickup_unit]).size()`), so at h3_7/daily the busiest hexagon legitimately reaches four figures. The top cell `872664c1effffff` — centroid 41.895, −87.626, i.e. River North / the Loop's taxi core — carries 2.36M pickups over 863 days, averaging **2,735/day with a peak of 5,250**. Checked against every historical version of the prediction files: these values are correct and long-standing, not an aggregation artifact and not a side effect of the zero-mile-trip drop (which only moved the test-set max 5,515 → 5,177 and removed ~2,850 rows, all below-average demand — the test set's mean demand *rose* slightly, 69.5 → 70.9, as a result)

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

**Data setup** (facts shared by both models — model-specific choices, incl. sampling and feature sets, are in the subsections below, not here):
- Trained on **six datasets**: 3 spatial resolutions (h3_7, census tract, community area) × 2 temporal resolutions (daily, hourly)
- Outcome variable is always `Total_Trip_Start`, predicted per spatial unit × time bucket, not autoregressive — but what each model actually *fits* differs (see below); don't describe "the target" as one single thing across both SVM strategies
- Zero share varies enormously by level (up to 96.5% at census/hourly, down to just 0.8% at community/daily) — this is a property of the data itself and drives the sampling choices described per model below
- Baseline: historical mean by (hour/month/day-of-week) or (month/day-of-week), i.e. the `base_demand` feature from Data Preparation — computed **only on the training split** and merged into val/test, so it is never leaking future information, and reused unchanged here both as a model feature and as the point of comparison for the skill score
- Train/val/test split: 50/20/30, temporal
- Both models fit in log1p space and invert with `expm1` (clipped at 0) to report metrics in count space; skill score = `1 − MAE_model / MAE_baseline`, used because raw MAE isn't comparable across levels (mean demand ranges ~80x)

**Support Vector Machine — approach & results** (`3.2a` direct / `3.2b` residual; **all figures below are from the latest pipeline run *after* the zero-mile-trip drop in Data Preparation and supersede the earlier draft, whose numbers came from the pre-drop data** — `docs/svm-model-performance-report.md`, if kept, must be re-derived from this run too). Level dimensions this run: h3_7 = 108 active hexes (46,548 daily / 1,117,152 hourly training rows, mean 58.4 trips/day, 78.9% zero), census = 496 units, community = 77 units; zero share ranges 0.8% (community/daily) to 96.5% (census/hourly). The zero-mile drop shrank h3_7 from 119→108 hexes and h3_7/daily training rows from 51,289→46,548:
- **Fit target, per strategy — this is the one place SVM's "target" is not simply `Total_Trip_Start`**: the *direct* strategy fits `log1p(Total_Trip_Start)`. The *residual* strategy instead fits the correction `log1p(Total_Trip_Start) − log1p(base_demand)` and reconstructs counts as `expm1(log1p(base_demand) + correction_hat)` — the model never sees the raw target directly, only how far actual demand deviates from the historical mean in log space.
- Trained on a **class-balanced sample** (all non-zero rows up to a cap, plus an equal number of zero rows), because several levels are extremely zero-inflated; evaluated at natural prevalence on validation/test. This balancing is an SVM-only choice, driven by SVR's O(n²) fit cost forcing a small sample in the first place — the NN below trains on the full, unbalanced split instead.
- Feature sets: 4 candidates compared per level (basic / +weather / +POI / +weather+POI); a kernel scan (linear/poly/rbf) followed by a grid search on `C`/`gamma`, with the feature set picked by validation R² in count space.
- **Best model overall: residual SVR on community/daily** — basic+weather features, linear kernel, C=1; **test** skill (MAE) **+6.5%**, R² = 0.941. (Down from the pre-drop +9.4% / R²=0.977, and the winning feature set shifted from basic+poi to basic+weather — the coarsest, densest level is still the clearest win.)
- **Also clears the baseline at h3_7, both temporal levels (residual):** h3_7/daily — basic+poi, linear, C=1 — **test** skill (MAE) **+3.5%**, R²=0.919; h3_7/hourly — basic+weather, linear, C=1 — **test** skill (MAE) **+4.8%**, R²=0.909. Two changes from the pre-drop run worth flagging: h3_7/daily's margin fell (+5.1%→+3.5%), and the **direct** model at h3_7/daily now essentially *ties* the baseline (+0.2% validation MAE) instead of losing to it (was −5.5%) — so the residual reframing is no longer what rescues this level, it only widens an already-even gap.
- **On the held-out test split the residual model now clears the baseline at 5 of 6 levels** (all except census/hourly) — but only community/daily (+6.5%), h3_7/hourly (+4.8%) and h3_7/daily (+3.5%) do so by a meaningful margin. census/daily (+0.8%) and community/hourly (+0.9%) are within noise and their *validation* skill was ≈0 or negative (+0.3% and −1.0%), so treat them as ties, not wins. **Worst: census/hourly** — sparsest and most zero-inflated (96.5% zero rows) — loses heavily under both strategies (direct −35.5% val MAE, residual −28.5% test MAE); MAE skill strongly negative while RMSE skill is near flat, the signature of a mismatch between the class-balanced training sample and the natural-prevalence evaluation.
- Residual-on-baseline reframing beats the direct model on MAE-skill at all 6 levels (validation); the direct model clears the baseline at none of them except a marginal h3_7/daily (+0.2%). Daily ≥ hourly at every spatial resolution on validation; on test that ordering flips only at h3_7, where the small hourly test sample scores higher (+4.8% vs +3.5%) — read that one flip with caution rather than as evidence hourly genuinely beats daily.
- Kernel choice: residual winners are **linear at 4 of 6 levels** (both h3_7 levels, census/hourly, community/daily) and RBF at census/daily and community/hourly; every direct-target winner is RBF — a linear kernel's unbounded log-space predictions explode through `expm1`, which is why the direct target always needs RBF and the small-scale residual mostly doesn't. Weather now enters several winning residual feature sets (basic+weather at h3_7/hourly and community/daily, basic+poi+weather at census/daily and community/hourly), where the pre-drop run leaned on basic+poi.
- **Feature importance** — permutation on validation, reported for the **direct** SVR (`3.2a`): every level relies most on `base_demand` when it is shuffled (+292% to +2,744% MAE), but what each level actually *needs* when a whole block is dropped splits in two — history/`base_demand` at h3_7/hourly (+148% MAE), census/hourly (+32%), community/hourly (+22%); POI at h3_7/daily (+5.5%), census/daily (+9.4%), community/daily (+13.2%). *(The residual notebook's own ablation was not regenerated this run, so the earlier residual-specific figures — `base_demand` +166% MAE, `location` +29% MAE at h3_7/daily — are stale and currently unverified against the new data; re-run `3.2b`'s ablation cell before citing them.)*
- Improvement levers: weather×hour/spatial interactions, finer/spatially-resolved weather, richer POI features (distances, not just counts); natural-prevalence training, motivated by the census/hourly train/test mismatch and by the residual model's zero-demand overshoot surfaced in Model Comparison below.

**Neural Network — approach & results** (`3.3 PredictionDeepLearning.ipynb`):
- Fit target: `log1p(Total_Trip_Start)` only — the NN was run in the direct framing alone; there is no residual-on-baseline NN variant to mirror the SVM's two strategies.
- Trained on the **full, natural-prevalence training split**, unlike the SVM — no zero/non-zero balancing was applied. Whether that was the right call is discussed under Model Comparison below.
- One fixed feature set for every level — `basic+poi+weather` (calendar/location/history + hour for hourly grids, plus all 14 POI categories and all 5 weather variables) — not ablated across the SVM's 4 feature-set variants; state plainly that this was a compute-budget call, not a finding that the other feature sets don't matter for the NN.
- Architecture: feedforward network, ReLU hidden layers with a linear output unit, MSE loss, AdamW (weight decay 1e-4, decoupled from the gradient so plain-L2 scaling issues don't apply), batch size 250, up to 15 epochs with early stopping (patience 3). Grid search over 4 layer configurations ([64,32,16], [128,64,32], [128,64,32,16], [256,128,64,32]) × 3 learning rates (0.01, 0.001, 0.0001) × 2 dropout rates (0.0, 0.2) — 24 configurations per level — with the winner picked by validation MAE in count space.
- **Final architecture selected per level** (from the saved checkpoints):
  | Level | Layers | LR | Dropout |
  |---|---|---:|---:|
  | h3_7 / hourly | [64,32,16] | 0.01 | 0.0 |
  | h3_7 / daily | [256,128,64,32] | 0.01 | 0.0 |
  | community / hourly | [64,32,16] | 0.01 | 0.0 |
  | community / daily | [64,32,16] | 0.01 | 0.0 |
  | census / hourly | [128,64,32,16] | 0.001 | 0.0 |
  | census / daily | [256,128,64,32] | 0.001 | 0.2 |

  A small, shallow network with the largest learning rate wins at h3_7/hourly and both community levels; the sparser census levels need a deeper net and a smaller learning rate, and census/daily is the only level where dropout regularization was selected at all — consistent with it being the hardest level to fit without overfitting.
- **Results per level** (skill vs. the same historical-mean baseline):

  | Level | MAE | RMSE | R² | Skill (MAE) | Skill (RMSE) |
  |---|---:|---:|---:|---:|---:|
  | h3_7 / hourly | 0.949 | 6.611 | 0.908 | **+6.3%** | +8.6% |
  | h3_7 / daily | 19.243 | 113.724 | 0.921 | +3.6% | +3.0% |
  | community / hourly | 2.910 | 11.594 | 0.903 | +2.4% | −4.4% |
  | community / daily | 48.463 | 174.717 | 0.941 | +2.1% | +2.8% |
  | census / hourly | 0.239 | 2.037 | 0.861 | ≈0% (−0.1%) | −2.7% |
  | census / daily | 5.095 | 40.696 | 0.826 | **−20.3%** | −44.7% |

  The NN clears the baseline at 4 of 6 levels but never by a wide margin; it loses clearly at census/daily and is a coin-flip at census/hourly — the same sparsest, most zero-inflated corner where the SVM also struggles most.
- **Methodological caveat to flag explicitly in the report**: the table above is `3.3`'s own evaluation, computed directly on the **test** split inside the same notebook that does model selection — unlike the SVM notebooks, which select on validation and touch test only once, in `3.4`. Only the h3_7 rows have since been re-checked in `3.4` against the SVM on a genuinely independent basis (see Model Comparison); the census and community NN numbers above have no equivalent cross-check.

**Model Comparison** (grounded in `3.4 ModelPerformanceVisualization.ipynb`; only h3_7 has a direct, apples-to-apples SVM-vs-NN comparison on the held-out test set — census and community are NN-only, per above):
- **⚠ Numbers in this subsection are pending regeneration — do not cite them yet.** Two separate problems surfaced after the zero-mile-trip drop:
  1. **Merge bug — now fixed in `3.4`.** `load_comparison` used to join the SVR and FNN predictions on `[h3_index_7, timestamp, y_true]`, i.e. partly on *the actual demand itself*. Because the SVR file had been re-scored and the FNN file had not, their per-cell `y_true` disagreed and the inner join silently dropped every mismatched row — the high-demand cells first. The join now uses `[h3_index_7, timestamp]` only, with a single `y_true` column carried from the SVR file. Effect: h3_7/daily recovers 24,068 → **27,972** rows, h3_7/hourly 643,069 → **671,328**, and the visible demand range returns from a truncated max of 220 to the true **5,177** trips/day.
  2. **The FNN predictions are still stale.** `models_predictions/FNN/` was last written 21 Jul, before the data change; only the SVR was re-run. `y_pred_fnn` is therefore matched to the right cell/time but comes from a model trained on pre-drop data. **Rerun `3.3`, then re-execute `3.4`** before any SVM-vs-NN figure below is used — the outputs currently saved in `3.4` are from the broken-merge run and are not valid either.
  Until that is done, the only trustworthy SVM figures for h3_7 are the ones scored directly in `3.2b`: **test skill +3.5% MAE (daily)** and **+4.8% MAE (hourly)**. The pre-drop figures in the table below (SVM daily +10.0%, hourly +0.8%) predate the data change.
- **Overall test metrics, h3_7** (pre-drop run — see warning above):
  | Level | Baseline MAE | SVM (residual) MAE | SVM skill | NN (FNN) MAE | NN skill |
  |---|---:|---:|---:|---:|---:|
  | daily | 19.962 (R²=0.916) | 17.971 (R²=0.937) | **+10.0%** | 19.243 (R²=0.921) | +3.6% |
  | hourly | 1.013 (R²=0.890) | 1.005 (R²=0.906) | +0.8% | 0.949 (R²=0.908) | **+6.3%** |

  There is no single winner: **SVM wins outright at daily, NN wins outright at hourly.** The segment breakdown below explains why, rather than leaving it as an unexplained split.
- **By demand band** (test set, h3_7) — *magnitudes below are from the pre-drop run and do not match the currently saved (also invalid) `3.4` outputs, so treat only the **direction** as established until `3.4` is re-run; the direction does hold consistently across every version seen so far*: the **residual SVM is worse than the baseline on true zero-demand cells** — skill −0.29 (daily) and −1.25 (hourly), i.e. its residual correction overshoots on rows that turn out to be exactly zero — while the **NN is clearly better there** (skill +0.32 daily, +0.47 hourly). Since zero-demand rows are 79%–94% of all rows, this one segment is most of why the NN wins overall at hourly. The pattern reverses on **high-demand cells (>100 trips/day)**: the SVM wins clearly at both resolutions (skill +0.115 vs. +0.021 daily; +0.093 vs. +0.086 hourly, a near-tie) — it is the more reliable model exactly where fleet-provisioning stakes are largest.
- **By weather**: the SVM is stable across dry and rainy days; the NN is markedly worse on rainy days at daily resolution (skill −0.153 rainy vs. +0.060 dry) and only slightly ahead of the SVM on rainy hours (+0.057 vs. +0.004) — a robustness gap for a model meant to answer exactly the "rainy Sunday" scenario the assignment poses.
- **Spatially** (`fig-svm-vs-nn-hex-delta`, MAE_NN − MAE_SVM per hexagon): the few hexagons where the SVM meaningfully beats the NN cluster tightly around the Loop and O'Hare — the small number of consistently highest-demand cells — while nearly every other cell across the rest of the city is a slight NN win or a near-tie. This is the same high-demand-vs-low-demand split as the segment table, shown spatially, at both temporal resolutions.
- **Bottom line for the write-up**: neither model dominates uniformly, so avoid a single "X wins" headline. The honest framing is that the SVM is the better choice for daily, high-demand, weather-robust planning, and the NN is the better choice for hourly, sparse-cell, near-zero-demand forecasting — a genuine trade-off, not a tie broken by chance. Translate the MAE figures into business terms (what a given MAE means for fleet over/under-provisioning at busy vs. quiet cells), and note H3 doesn't add real spatial precision beyond tract/CA in this dataset, so any "finer resolution helps" claim needs that caveat.

**Evaluation question** (the brief's "is complexity worth it?" and "is deep learning worth it?" are the same question at different depths — treat as one):
- *Is the added complexity of a neural network worth it here?* — **Conditionally, and only at some levels.** On the one level with a genuine held-out comparison (h3_7), the NN's only clear edge is at hourly resolution, and that edge comes almost entirely from how it handles the enormous zero-demand majority of rows, not from better modelling of actual non-zero demand — on high-demand cells and rainy days the simpler SVM is at least as good and often better, and at daily resolution the SVM wins outright. The NN also costs a materially larger search (24 architecture/LR/dropout combinations × up to 15 epochs × 6 levels, on GPU) against the SVM's cheaper kernel-and-grid search. Complexity earns its keep specifically for the "will there be any demand at all in this sparse cell this hour" question, and does not for the demand-magnitude question at busy cells — arguably the more operationally important one for fleet provisioning.
- Convergence/complexity check still to add: NN train/validation loss curves are saved per config (`learning_curve_*.png`) but not yet interpreted for over-/underfitting in the report; there is no equivalent check for the SVM (e.g., whether the winning kernel's `C`/`gamma` sit near a validation-performance plateau or at a search-grid edge) — both should be added if space allows.

**Shortfalls & improvement levers**:
- NN:
  - Hyperparameter grid narrowed by compute budget (no batch-size or activation-function sweep beyond the 3 learning rates tested); a wider search could plausibly close some of the census-level gap.
  - Single feature set only — no equivalent to the SVM's 4-way feature-set ablation, so it's unknown whether weather or POI individually help or hurt the NN at each level.
  - No zero/non-zero balancing was tested against a balanced-sample variant; the NN's edge on zero-demand cells suggests the unbalanced choice was right for this architecture, but that is an assumption, not a demonstrated result, until the comparison is actually run.
  - Own reported per-level scores (census, community) are computed on the test split inside the same notebook used for model selection, not a separate held-out check the way the SVM's are — fix by either re-scoring all 6 levels validation-only inside `3.3`, or extending `3.4` beyond h3_7 for a true held-out comparison at every level.
  - Specific, evidenced weakness: materially worse than baseline on rainy days and on medium-demand cells at both resolutions (segment table above) — worth checking via permutation importance (never run for the NN, only for the direct SVR) whether weather features are actually being used effectively by the network.
- SVM:
  - Fixed-size class-balanced sample (up to 8k/16k rows per class) imposed by SVR's O(n²) fit cost, unlike the NN, which trains on the full split — but frame this as a deliberate, low-cost trade-off rather than an open shortfall: more training rows would not obviously translate into better SVR predictions here, given the report's own finding that cell + calendar features are already close to the ceiling of what's predictable under the assignment's non-autoregressive framing (a historical mean over those same keys is close to optimal). The real lever worth testing is *how* the sample is drawn (natural-prevalence vs. class-balanced), not simply how much of it is used.
  - The residual model's overshoot on true-zero rows (skill −0.29 to −1.25 vs. baseline on that segment, per `3.4`) is a specific, fixable failure mode — the correction term isn't shrinking to 0 where it should. Worth testing a floor/clip on the residual, or a two-stage zero/non-zero gate before regression.
  - The two-stage tuning shortcut (kernel scan, then grid search only within the winning kernel) was checked against a full, independent per-kernel grid search (`models_predictions/SVR/residual_kernelgrid/models/level_comparison.csv`) — **not** an open shortfall: 5 of the 6 levels select the exact identical model (same kernel, feature set, `C`, skill). Only `community/hourly` differs, and only marginally — the full grid search picks a poly kernel over linear, nudging skill (MAE) from −1.7% to −1.0% but slightly worsening skill (RMSE) from −2.0% to −2.6%; the level still loses to baseline either way. The two-stage shortcut is validated, not a real limitation, and should be reported as such rather than flagged as future work.
  - Natural-prevalence training is untested and now motivated twice over: by the census/hourly train/test mismatch, and by the zero-demand overshoot `3.4` surfaces.

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
3. Prediction accuracy (post-zero-mile-drop run): residual SVR beats the baseline meaningfully at community/daily (+6.5% test MAE) and both h3_7 levels (hourly +4.8%, daily +3.5%), is a tie at census/daily and community/hourly (≈+0.8–0.9%), and loses badly at census/hourly (−28.5% test); the direct SVR clears the baseline at no level. The h3_7 SVM-vs-NN comparison in `3.4` is stale (FNN not re-run) and must be regenerated before it can be cited.
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
Good candidates already flagged elsewhere: full SVR hyperparameter grids, location-reliability analysis supporting plots if not already embedded in the main body.

---

## Cross-cutting reminders (not tied to one file)
- Every team member needs at least one authored Git commit, or the team fails the assignment (syllabus G.3 / §2.2)
- Any LLM-generated code or text must be explicitly marked, including the prompt used
- 10-page limit is exclusive of figures, references, and appendix — push detailed tables/hyperparameter grids to the appendix
- All Quarto cross-reference labels (`@sec-...`) need to be updated in the Introduction's "Report Structure" paragraph to match this 5-section structure