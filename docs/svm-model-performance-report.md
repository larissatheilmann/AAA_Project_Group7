# SVM Demand Prediction: Performance Report

**Source notebooks:** `3.2a PredictionSVM_Standard.ipynb`, `3.2b PredictionSVM_Residual.ipynb`, `3.4 ModelPerformanceVisualization.ipynb`
**Target:** `Total_Trip_Start` per spatial cell × time bucket, non-autoregressive.

> **Note on sources.** Numbers below are the newest CSVs (Jul 20), all on a recomputed, **stronger baseline** than earlier runs. `residual/level_comparison_mae.csv` (Jul 19, old baseline) is **superseded** and excluded. Sections 1–3 are **validation**. Section 5 (test) is **stale — pending re-run**.

## 1. Setup

- 6 levels: 3 spatial (h3_7, census, community) × 2 temporal (hourly, daily).
- 2 strategies, tuned independently per level:
  - **Direct (3.2a):** SVR on `log1p(demand)`.
  - **Residual (3.2b):** SVR on `log1p(demand) − log1p(base_demand)`; reconstruct via `expm1`.
- Per level: scan 4 feature sets × 3 kernels → pick kernel → grid-search `C`/`γ` → pick feature set by validation R² (count space).
- Metrics in count space (MAE/RMSE/R²), `log1p` inverted, clipped at 0.
- **Skill** = `1 − MAE_model / MAE_baseline`; baseline = historical-mean `base_demand`.
- Fit on class-balanced sample; **evaluated at natural prevalence**.
- ⚠️ **Baseline is stronger than in prior report** → both strategies now lose at most levels.

| Level | Units | Train rows | Mean demand | Zero share |
|---|---:|---:|---:|---:|
| h3_7 / hourly | 119 | 1,230,936 | 2.45 | 93.6% |
| h3_7 / daily | 119 | 51,289 | 58.91 | 79.0% |
| census / hourly | 600 | 6,206,400 | 0.49 | 97.1% |
| census / daily | 600 | 258,600 | 11.68 | 89.7% |
| community / hourly | 77 | 796,488 | 8.38 | 42.5% |
| community / daily | 77 | 33,187 | 201.14 | 0.6% |

---

## 2. Master comparison: all 12 models (validation)

| Level | Strategy | Feature set | Kernel | C | γ | Val MAE | Val RMSE | Val R² | Base MAE | Skill (MAE) | Skill (RMSE) |
|---|---|---|---|---:|---|---:|---:|---:|---:|---:|---:|
| h3_7/hourly | Direct | basic+poi | rbf | 10 | scale | 0.932 | 6.013 | 0.925 | 0.851 | −9.6% | −2.3% |
| h3_7/hourly | Residual | basic | linear | 1 | scale | 0.882 | 5.588 | 0.935 | 0.851 | **−3.7%** | +5.0% |
| h3_7/daily | Direct | basic | rbf | 10 | scale | 15.459 | 88.600 | 0.953 | 14.658 | −5.5% | −3.4% |
| h3_7/daily | Residual | basic+poi | linear | 1 | scale | 13.904 | 77.947 | 0.963 | 14.658 | **+5.1%** | +9.0% |
| census/hourly | Direct | basic | rbf | 1 | scale | 0.291 | 1.929 | 0.891 | 0.215 | −35.2% ▼ | −3.2% |
| census/hourly | Residual | basic+poi | rbf | 1 | scale | 0.266 | 1.888 | 0.895 | 0.215 | **−23.5%** | −1.0% |
| census/daily | Direct | basic+poi | rbf | 10 | scale | 3.584 | 27.395 | 0.933 | 3.295 | −8.8% | −9.6% |
| census/daily | Residual | basic+poi+weather | rbf | 1 | scale | 3.371 | 25.891 | 0.940 | 3.295 | **−2.3%** | −3.6% |
| community/hourly | Direct | basic | rbf | 10 | 0.05 | 3.184 | 11.894 | 0.899 | 2.602 | −22.4% | −32.4% |
| community/hourly | Residual | basic | linear | 1 | scale | 2.646 | 9.167 | 0.940 | 2.602 | **−1.7%** | −2.0% |
| community/daily | Direct | basic+poi | rbf | 10 | scale | 38.464 | 142.824 | 0.961 | 34.805 | −10.5% | −13.4% |
| community/daily | Residual | basic+poi | linear | 1 | scale | 31.534 | 110.977 | 0.977 | 34.805 | **+9.4%** ★ | +11.9% |

★ best skill overall · ▼ worst skill overall

**Takeaways:**
- **Residual beats direct at all 6 levels** (skill MAE), but on the stronger baseline **only 2 levels beat the baseline**: community/daily (+9.4%) and h3_7/daily (+5.1%).
- **Direct loses at all 6 levels** now (all negative skill).
- **Kernel story reversed:** residual winners are **mostly linear** (4/6: both h3_7, both community); only census picks rbf. Direct winners all rbf. → "RBF empirically forced" no longer holds under the residual target.
- **census/hourly still the worst** (−23.5% / −35.2%); MAE skill strongly negative while RMSE skill ≈0 → zero-inflation signature.
- **Daily > hourly** at every spatial resolution, both strategies.

---

## 3. Feature importance & ablation (direct models, validation)

Permutation = shuffle block, see val-MAE degrade (what the fit *leans on*). Ablation = drop block, retrain (what it *needs*).

### 3.1 `base_demand`: reliance vs necessity

| Level | Permutation Δ MAE | Ablation Δ MAE | Read |
|---|---:|---:|---|
| h3_7/hourly | +416% | +27% | Load-bearing |
| h3_7/daily | +2,556% | +166% | Load-bearing |
| census/hourly | +309% | +210% | Load-bearing |
| census/daily | +29,282% | +3.8% | **Reliance w/o necessity** |
| community/hourly | +3,506% | +72% | Load-bearing |
| community/daily | +2,700% | **−5.0%** | **Reliance w/o necessity** (dropping *improves*) |

- At the 2 coarse daily levels, `base_demand` is leaned on almost entirely yet nearly free to drop — reconstructable from location/calendar.
- Load-bearing at the other 4 levels.

### 3.2 Other blocks (drop-column Δ MAE, direct)

| Level | calendar | location | hour | weather | poi |
|---|---:|---:|---:|---:|---:|
| h3_7/hourly | +1.3% | −0.1% | +3.9% | — | +3.4% |
| h3_7/daily | −0.3% | +29.0% | — | — | — |
| census/hourly | −2.1% | +11.4% | −0.9% | — | — |
| census/daily | +3.5% | +3.2% | — | — | +4.0% |
| community/hourly | −5.4% | +32.4% | +3.7% | — | — |
| community/daily | −5.2% | +4.2% | — | — | +22.3% |

- **`location` dominates** at h3_7/daily (+29%), census/hourly (+11%), community/hourly (+32%).
- **`poi`** matters at community/daily (+22%).
- Negative calendar/hour drops = mild overfit to noisy signal.

---

## 4. Test evaluation (Section 5, from 3.4) — STALE

- **Pending re-run.** No new test CSV; and model selection changed (best is now **community/daily residual**, and h3_7/daily residual winner is now **basic+poi linear**, not basic rbf).
- Prior h3_7/daily test numbers (skill +13.3%) no longer match the selected models → do not cite until 3.4 is re-run.
- NN comparison still blocked: `models_predictions/ANN/standard/models/h3_7/daily/best.joblib` does not exist.

---

## 5. Key points

- On the stronger baseline, **SVR beats climatology at only 2 of 6 levels** (community/daily, h3_7/daily), both residual + linear.
- **Residual reframing helps everywhere** (beats direct at all 6) but doesn't clear the baseline at the 4 sparser levels.
- **Best model: residual SVR, community/daily** (basic+poi, linear, C=1; skill +9.4%, R²=0.977).
- **Worst: census/hourly** — sparsest/most zero-inflated; no config beats baseline on MAE. Likely a train/natural-prevalence mismatch (MAE−RMSE split).
- **Kernel:** linear now wins most residual levels; RBF only forced under the direct target.

### To do
- Re-run 3.4 for test scores + NN comparison (blocked on 3.3 output).
- Address census/hourly zero-inflation (natural-prevalence training or zero/non-zero gate).
- Weather × hour / × spatial interactions; finer weather; richer POI features.
