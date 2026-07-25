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

**Headline finding — lead with it**: both winning models sit on **Community Areas** (77 units, the coarsest/densest/least zero-inflated level). The best SVM is the residual SVR at **community/daily** (test skill +6.5% MAE, R²=0.941); the best NN is the FNN at **community/hourly** (+9.7% MAE, R²=0.925). The two are complementary, not competing: the SVM owns the daily grain, the NN owns the hourly grain, and the head-to-head comparison in `3.4` is run at exactly these two levels.

**Data setup** (facts shared by both models — model-specific choices, incl. sampling and feature sets, are in the subsections below, not here):
- Trained on **six datasets**: 3 spatial resolutions (h3_7, census tract, community area) × 2 temporal resolutions (daily, hourly)
- Outcome variable is always `Total_Trip_Start`, predicted per spatial unit × time bucket, not autoregressive — but what each model actually *fits* differs (see below); don't describe "the target" as one single thing across both SVM strategies
- Zero share varies enormously by level (up to 96.5% at census/hourly, down to just 0.8% at community/daily) — this is a property of the data itself and drives the sampling choices described per model below
- Baseline: historical mean by (hour/month/day-of-week) or (month/day-of-week), i.e. the `base_demand` feature from Data Preparation — computed **only on the training split** and merged into val/test, so it is never leaking future information, and reused unchanged here both as a model feature and as the point of comparison for the skill score
- Train/val/test split: 50/20/30, temporal
- Both models fit in log1p space and invert with `expm1` (clipped at 0) to report metrics in count space; skill score = `1 − MAE_model / MAE_baseline`, used because raw MAE isn't comparable across levels (mean demand ranges ~80x)

**Support Vector Machine — approach & results** (`3.2a` direct / `3.2b` residual-on-baseline; both `%run` the shared `3.2 PredictionCommonGround.ipynb`. All numbers below are from the current post-zero-mile-drop run — the FNN and both SVR notebooks were re-executed on the same splits, so the SVM-vs-NN comparison in `3.4` is now valid and the earlier "pending regeneration" caveats are resolved). Per-level training dimensions (`overview_table` in `3.2b`): h3_7 = 108 hexes (1,117,152 hourly / 46,548 daily rows), census = 496 tracts (5,130,624 / 213,776), community = 77 areas (796,488 / 33,187); mean demand runs 0.53 trips/cell-hour (census/hourly) to 181.9 trips/area-day (community/daily); zero share 0.8% (community/daily) to 96.5% (census/hourly).
- **Fit target differs by strategy — the one place the SVM's "target" is not simply `Total_Trip_Start`**: the *direct* model (`3.2a`) fits `log1p(Total_Trip_Start)`. The *residual* model (`3.2b`) fits the log-space correction `log1p(Total_Trip_Start) − log1p(base_demand)` and reconstructs counts as `expm1(log1p(base_demand) + correction_hat)` — it never sees the raw target, only the deviation from the historical mean. The residual framing has a built-in floor: a correction of exactly 0 reproduces `base_demand` (skill = 0), so any positive skill is a genuine gain.
- **Class-balanced training sample**, forced by SVR's O(n²) fit cost: kernel scan on 8k rows (4k/class), grid search on up to 16k rows (8k/class) — all non-zero rows up to the cap plus an equal number of zeros — then evaluated at natural prevalence on the full validation split. At community/daily (0.8% zero) the zero class runs out, so the sample is effectively all the non-zero area-days. This balancing is SVM-only; the NN trains on the full unbalanced split.
- Per level: 4 feature sets compared (basic / +weather / +POI / +weather+POI), a kernel scan (linear/poly/rbf at C=1), then a grid search (C ∈ {1,10,50}, gamma ∈ {scale, 0.05, 0.2}), feature set picked by validation R² in count space. The test split is untouched here — used once, in `3.4`.
- **Best SVM overall: residual SVR at community/daily** — basic+weather, linear kernel, C=1. Selected on validation (skill **+5.3% MAE**, R²=0.970 — the only clearly positive level; `tbl-svr-residual2-level-comparison`), and on the held-out **test** split it reaches **+6.5% MAE skill**, R²=0.941 (MAE 46.34 vs baseline 49.58; confirmed identically in `3.4`). This is the SVM carried into the model comparison. **Both the SVM's and the NN's best models land on Community Areas** — the coarsest, densest, least zero-inflated level.
- **Residual validation skill, all six levels** (`tbl-svr-residual2-level-comparison`, sorted): community/daily +5.3%; h3_7/daily (basic+poi, linear) +1.2%; census/daily (basic+poi+weather, rbf) +0.3%; h3_7/hourly (basic+weather, linear) +0.3%; community/hourly (basic+poi+weather, rbf) −1.0%; census/hourly (basic, linear) −32.1%. Only community/daily is a meaningful win on validation; the rest are ties or, at census/hourly, a heavy loss.
- **On the held-out test split** (`3.2b`'s own quick score, matching `3.4`) the residual model clears baseline at 5 of 6 levels — community/daily +6.5%, h3_7/hourly +4.8%, h3_7/daily +3.5%, community/hourly +0.9%, census/daily +0.8% — losing only at census/hourly (−28.5%). But since selection is on validation, only community/daily is a defensible "win"; h3_7's larger test gains sit on ≈0 validation skill and small daily/hourly test samples, so read them with caution rather than as evidence hourly/h3_7 genuinely beats community/daily.
- **Direct vs. residual** (`tbl-svr-direct-vs-residual-final`, validation): the residual reframing beats the direct model at **all 6 levels** (delta +1.0 to +11.5 pts of skill); the direct model clears the baseline at none except a marginal h3_7/daily (+0.2%). **Worst level for both is census/hourly** — sparsest and 96.5% zero — where the mismatch between the class-balanced training sample and the natural-prevalence evaluation shows as strongly negative MAE skill (RMSE skill near flat, its signature).
- Kernel choice: residual winners are **linear at 4 of 6 levels** (both h3_7, census/hourly, community/daily) and RBF at census/daily and community/hourly; **every direct winner is RBF** — a linear kernel's unbounded log-space output would explode through `expm1`, which is why the direct target always needs RBF while the small-scale residual usually does not. Weather enters several winning residual feature sets (basic+weather at h3_7/hourly and community/daily; basic+poi+weather at census/daily and community/hourly).
- **Feature importance** (`fig-svr-feature-importance`, permutation + drop-column ablation on the **direct** SVR winners, `3.2a`): when a block is *shuffled*, every level's fitted model leans most on `base_demand` (+292% to +1,412% MAE) — except census/daily, which leans most on POI (+2,744%). But when a block is *dropped and the model refit*, what each level needs splits by temporal grain — `base_demand` at the hourly levels (h3_7 +148%, census +32%, community +22%), POI at the daily levels (h3_7 +5.5%, census +9.4%, community +13.2%). The gap is the story: the fitted model depends on `base_demand` almost entirely, yet a model refit without it barely degrades, because lat/lon + hour + day-of-week reconstruct it. (This ablation exists only for the direct SVR; `3.2b` runs no importance analysis, so there are no residual-specific importance figures to cite.)
- Improvement levers: natural-prevalence (vs class-balanced) training — motivated by census/hourly and by the SVM's hourly weakness on busy cells (see Model Comparison); weather×hour/space interactions; richer POI features (distances, not just counts).

**Neural Network — approach & results** (`3.3 PredictionDeepLearning.ipynb`):
- Fit target: `log1p(Total_Trip_Start)` directly, inverted with `expm1` clipped at 0 — no residual-on-baseline variant, so there is no NN mirror of the SVM's two strategies.
- Trained on the **full, natural-prevalence training split** — no zero/non-zero balancing (unlike the SVM). One fixed feature set for every level, `basic+poi+weather` (calendar + location + `base_demand`, plus hour_sin/cos on hourly grids, all 14 POI categories, all 5 weather variables) — not ablated across the SVM's 4 feature-set variants; state plainly that this was a compute-budget call, not a finding that the other sets don't matter.
- Architecture: feedforward, ReLU hidden layers + dropout, linear output, MSE loss on the log target, AdamW (weight decay 1e-4, decoupled so plain-L2 scaling issues don't apply), batch size 250, up to 15 epochs with early stopping (patience 3). Grid search over 4 layer configs ([64,32,16], [128,64,32], [128,64,32,16], [256,128,64,32]) × 3 learning rates (0.01, 0.001, 0.0001) × 2 dropout rates (0.0, 0.2) — **24 configurations per level** — winner picked by **validation MAE in count space**, then scored **once** on test. This is the same select-on-validation / touch-test-once discipline the SVM uses (a change from the earlier draft, which noted the NN selecting on test).
- **Selected architecture per level** (from the saved `best_fnn_*` checkpoints):
  | Level | Layers | LR | Dropout |
  |---|---|---:|---:|
  | community / hourly | [256,128,64,32] | 0.001 | 0.0 |
  | community / daily | [128,64,32,16] | 0.01 | 0.0 |
  | h3_7 / hourly | [128,64,32,16] | 0.0001 | 0.0 |
  | h3_7 / daily | [128,64,32,16] | 0.01 | 0.2 |
  | census / hourly | [64,32,16] | 0.001 | 0.0 |
  | census / daily | [64,32,16] | 0.001 | 0.2 |

  The best NN (community/hourly) is the largest, deepest net at a mid learning rate; dropout is selected only at the two daily levels that overfit most (h3_7/daily, census/daily).
- **Results per level** (`fig-fnn-skill-by-level`; test skill vs. the historical-mean baseline, sorted):

  | Level | MAE | RMSE | R² | Skill (MAE) | Skill (RMSE) |
  |---|---:|---:|---:|---:|---:|
  | community / hourly | 2.620 | 9.678 | 0.925 | **+9.7%** | +12.7% |
  | h3_7 / hourly | 1.041 | 7.358 | 0.883 | +2.5% | +1.0% |
  | census / hourly | 0.276 | 2.218 | 0.844 | −1.0% | −6.1% |
  | community / daily | 54.518 | 190.617 | 0.921 | −10.0% | −4.8% |
  | census / daily | 6.355 | 42.094 | 0.819 | −29.2% | −42.7% |
  | h3_7 / daily | 28.209 | 155.722 | 0.844 | **−31.9%** | −28.7% |

  The NN is essentially an **hourly-only** model: it clears the baseline meaningfully only at community/hourly (+9.7%), marginally at h3_7/hourly (+2.5%), and **loses to the baseline at every daily level** (−10% to −32%) as well as at census/hourly. Its single strong configuration — community/hourly — is exactly the level and grain where the SVM is weakest, which is what makes the two models complementary rather than redundant. Note the sharp temporal divide: every hourly level outscores its daily counterpart, the mirror image of the SVM, whose best levels are daily.
- **Methodological note**: because `3.4` compares the two models only at the community level (see below), the census and h3_7 NN rows have no independent cross-check — they are `3.3`'s own once-only test scoring. The community rows, which carry the headline result, are re-derived identically in `3.4`.

**Model Comparison** (`3.4 ModelPerformanceVisualization.ipynb` — now rebuilt at the **community** level, both temporal resolutions, because that is where each model peaks: best SVM = community/daily, best NN = community/hourly. The earlier h3_7-only comparison, the broken merge, and the stale-FNN warnings are all resolved — the merge joins on `[commarea, timestamp]` only, and the FNN and SVR predictions come from the same post-drop run. Test rows: community/daily 19,943; community/hourly 478,632):
- **Overall test metrics** (`tbl-svm-vs-nn-overall`):
  | Level | Baseline MAE (R²) | SVM residual MAE (R²) | SVM skill | NN MAE (R²) | NN skill |
  |---|---:|---:|---:|---:|---:|
  | community / daily | 49.58 (0.928) | 46.34 (0.941) | **+6.5%** | 54.52 (0.921) | −10.0% |
  | community / hourly | 2.903 (0.902) | 2.878 (0.899) | +0.9% | 2.620 (0.925) | **+9.7%** |

  A clean, symmetric split with no single winner: **the SVM is the daily model, the NN is the hourly model.** At daily the SVM beats the baseline and the NN falls 10% below it; at hourly the NN beats the baseline by ~10% while the SVM only ties it on MAE (and is fractionally worse on RMSE/R²).
- **By demand band** (`tbl-svm-vs-nn-segments`, skill vs. baseline) — this explains the split:
  - **community/daily**: the SVM beats the baseline in **every** segment — zero-demand +5.3%, low (1–10) +5.1%, medium (11–100) +3.4%, high (>100) +7.0%, dry +6.4%, rainy +7.3% — while the NN is **below** the baseline in every segment, worst on medium-demand area-days (−30.2%). The SVM is uniformly the better daily model.
  - **community/hourly**: the pattern inverts on non-zero demand. The **NN wins every non-zero band** — low +2.2%, medium +11.3%, high +11.1% — whereas the residual **SVM's only real strength is true-zero cells (+19.4%, its best segment of all)**, and it slips *below* baseline on the busiest cells (high-demand −2.9%, low −0.7%). The SVM ties baseline overall precisely because it trades a big zero-cell win against losses on the high-demand cells that matter most operationally.
  - This **reverses the earlier h3_7 finding**: at the community level the residual SVM no longer overshoots on zero-demand rows — it is now the *best* zero-demand model — and the NN's edge is clearly located in *non-zero within-day demand*, not in the zero majority.
- **By weather**: no robustness gap at the community level. At daily the SVM is stable and even slightly best on rainy days (+7.3% vs +6.4% dry); the NN is poor on both (rainy −12.2%, dry −9.7%). At hourly **both** models are marginally *better* on rainy than dry (SVM +2.3% vs +0.7%; NN +10.1% vs +9.7%) — so the rainy-day weakness flagged in the old h3_7 draft does not appear here.
- **Spatially** (`fig-svm-hex-mae`, `fig-svm-hex-mre`, `fig-svm-hex-r2`, and the head-to-head `fig-svm-vs-nn-hex-delta` = MAE_NN − MAE_SVM per area): at daily the SVM's lower error covers essentially the whole city; at hourly the NN wins across most areas, its advantage concentrated in the high-demand central areas (Loop / Near North) where absolute errors — and fleet-provisioning stakes — are largest, consistent with the NN's medium/high-band wins in the segment table. The temporal breakdowns (`fig-svm-vs-nn-dayofweek`, `fig-svm-vs-nn-hourofday`, and the city-wide `fig-svm-vs-nn-timeseries`) show the same per-day / per-hour ordering.
- **Bottom line for the write-up**: no uniform winner — report a genuine trade-off. The **SVM is the choice for daily, whole-city planning** (beats the baseline across every demand band and both weather regimes, strongest on the busiest area-days); the **NN is the choice for hourly operational forecasting** (captures within-day non-zero demand at busy hours the SVM misses). Both peak at community resolution, consistent with community being the densest, least zero-inflated level — and with the Data-Preparation finding that H3 adds no real spatial precision beyond tract/CA, so a "finer grid is better" claim is not supported here. Translate the MAE figures into business terms (what a given MAE means for fleet over/under-provisioning at busy vs. quiet areas).

**Evaluation question** (the brief's "is complexity worth it?" and "is deep learning worth it?" are the same question at different depths — treat as one):
- *Is the added complexity of a neural network worth it here?* — **Conditionally, and only at hourly resolution.** On the head-to-head community comparison the NN's advantage is real but narrow: it beats both the baseline and the SVM at community/hourly (+9.7% vs the SVM's +0.9%), and that edge comes from modelling within-day non-zero demand (the medium/high bands) — the operationally useful part — not merely from the zero majority, where the SVM is actually better. At **daily** resolution the NN is strictly worse: it loses to the far cheaper SVM at community/daily and falls *below the historical-mean baseline at all three daily levels* (down to −32% at h3_7/daily). The NN also costs a much larger search (24 architecture/LR/dropout configs × up to 15 epochs × 6 levels on GPU) against the SVM's cheap kernel-scan-plus-grid on a 16k sample. So the complexity earns its keep for **hourly within-day forecasting** and does **not** for daily planning, where the SVM both wins and costs less.
- Convergence/complexity check still to add: NN train/validation loss curves are saved per config (`learning_curve_*.png`) but not yet interpreted for over-/underfitting in the report; there is no equivalent check for the SVM (e.g., whether the winning kernel's `C`/`gamma` sit near a validation-performance plateau or at a search-grid edge) — both should be added if space allows.

**Shortfalls & improvement levers**:
- NN:
  - **Catastrophic at daily resolution** — below the baseline at all three daily levels (−10% to −32%), the clearest weakness in the whole predictive section. Worth diagnosing: MSE-on-`log1p` trained on the full split likely over-smooths the heavy right tail of daily counts, where a handful of very busy area-days dominate the error. Permutation importance was never run for the NN, so whether it even uses weather/POI effectively is unknown.
  - Single feature set only (`basic+poi+weather`) — no equivalent to the SVM's 4-way feature-set ablation, so it's unknown whether weather or POI individually help or hurt the NN at each level.
  - No zero/non-zero balancing was tested against a balanced-sample variant; the NN's strength on community/hourly non-zero demand suggests the full unbalanced split is right for this architecture, but that is an assumption, not a demonstrated result, until the comparison is actually run.
  - Hyperparameter grid narrowed by compute budget (no batch-size or activation-function sweep beyond the 3 learning rates tested).
- SVM:
  - Fixed-size class-balanced sample (up to 16k rows, 8k/class) imposed by SVR's O(n²) fit cost, unlike the NN, which trains on the full split — but frame this as a deliberate, low-cost trade-off rather than an open shortfall: more training rows would not obviously translate into better SVR predictions here, given the report's own feature-importance finding that cell + calendar features are already close to the ceiling of what's predictable under the assignment's non-autoregressive framing (a historical mean over those same keys is close to optimal). The real lever worth testing is *how* the sample is drawn (natural-prevalence vs. class-balanced), not simply how much of it is used.
  - **At community/hourly the SVM only ties the baseline**: it wins zeros (+19.4%) but slips below baseline on the busiest cells (high-demand −2.9%, per `3.4`). The residual correction under-serves the high-variance hourly cells — a specific, fixable failure mode worth a per-band or two-stage (zero/non-zero gate) treatment.
  - **census/hourly remains the worst level** (−28.5% test): sparsest and 96.5% zero, a mismatch between the class-balanced training sample and natural-prevalence evaluation. Natural-prevalence training is untested and now motivated twice — by census/hourly and by the community/hourly busy-cell weakness above.
  - The two-stage tuning shortcut (kernel scan, then grid search only within the winning kernel) was validated against a full, independent per-kernel grid search (`models_predictions/SVR/residual_kernelgrid/`) in an earlier run — but that comparison predates the zero-mile-drop and would need re-running before it can be cited on the current data.



---

## `05-conclusion.qmd` — (e) Conclusions and Discussion
*(All four subsections below are still TODO in the draft, but the material mostly already exists across Sections 2–4 — this is a synthesis job.)*

### Summary of Findings
Recap the five research questions from the Introduction against what Data Analytics actually found:
1. Demand patterns: extreme concentration (Gini 0.94–0.97), Thursday peak / weekend trough
2. Spatial hot spots: GMM/KDE results
3. Prediction accuracy (current post-zero-mile-drop run): **both winning models sit on Community Areas** — the residual SVR at community/daily (test skill +6.5% MAE, R²=0.941) and the FNN at community/hourly (+9.7% MAE, R²=0.925). The now-valid SVM-vs-NN comparison in `3.4` (merge fixed, FNN re-run) shows a genuine trade-off rather than a winner: the SVM owns the daily grain (beats baseline across every demand band; the NN falls 10% below baseline at daily) and the NN owns the hourly grain (wins every non-zero demand band; the SVM only ties baseline hourly). The direct SVR clears the baseline at no level, and the NN loses to the baseline at all three daily levels — deep learning helps only for hourly within-day forecasting.
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