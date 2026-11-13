# Legacy Notebook Audit & Remediation Report

This document summarizes the technical code audit of the two legacy Google Colab notebooks in Drive (`Collection_Forecasting_2024.ipynb` and `Exploring_Waste_Collection_and_Traffic_data.ipynb`) and documents how every identified defect—plus secondary modeling and coherence gaps identified during hardening—was eliminated in the production pipeline (`banff-forecasting v0.1.0`).

---

## Executive Summary of Audit Findings

The legacy notebooks successfully framed the problem (linking Banff tourism/traffic, weather, and macro drivers to municipal waste streams) but suffered from **seven data-engineering, econometric, and statistical defects** that invalidated historical backtests, muted macroeconomic transmission, broke accounting conservation, and destabilized multi-year extrapolations. **None** of the original Drive notebooks were altered; all fixes live cleanly inside the modular Python package with automated pytest verification (`tests/test_pipeline.py`).

---

## Defect-by-Defect Audit & Verified Resolutions

### 1. Waste Series Splicing Discontinuity (`collection_df.csv` + `zero_waste_dashboard_data.csv`)
- **Legacy Behavior:** Spliced raw weigh-scale ticket exports (`collection_df.csv`, `2018–2023`) onto the official Zero Waste Dashboard export (`zero_waste_dashboard_data.csv`) beginning `2024-01-01`. This introduced artificial structural jumps at the boundary and distorted 2018–2023 trends.
- **Root Cause & Fix (`load_waste_targets`):** Loaded **all 84 months (`2018-01..2024-12`) uniformly** from `zero_waste_dashboard_data.csv` partitioned across `CORE_13_MATERIALS` (`Total Food Waste + Total Garbage + Other Diversion === Total Collection`, exact `1e-9 t` tolerance).
- **Quantified Verification (`data_audit_comparison.csv`):**

| Calendar Year | Clean `Total Collection` (t) | Legacy Spliced (t) | Splice Discrepancy (Clean − Legacy) | Clean `Total Garbage` Diff (t) | Clean `Total Food Waste` Diff (t) |
| :---: | :---: | :---: | :---: | :---: | :---: |
| `2018` | 7,252.327 | 6,259.659 | **+992.668 t** | +313.276 t | -9.320 t |
| `2019` | 7,538.277 | 7,389.267 | **+149.010 t** | +15.125 t | -19.200 t |
| `2020` | 5,211.773 | 5,458.179 | **-246.406 t** | +36.200 t | -12.820 t |
| `2021` | 5,229.125 | 5,178.631 | **+50.494 t** | +10.262 t | 0.000 t |
| `2022` | 6,450.504 | 6,314.258 | **+136.246 t** | +14.095 t | -7.440 t |
| `2023` | 6,843.514 | 6,796.767 | **+46.747 t** | +37.065 t | -6.220 t |
| **`2024`** | **7,281.867** | **7,281.867** | **0.000 t** | **0.000 t** | **0.000 t** |

---

### 2. Tableau 4x Crosstab Traffic Inflation & Partial Trailing Month
- **Legacy Behavior:** Certain Tableau exports (`tw_traffic`) inflated daily combined two-way counts by ~4x (`Combined / (West + East) ~ 4.0`) and rolled partial trailing months into monthly totals without completeness checks.
- **Remediation (`load_traffic_targets`):**
  - Enforces a median sanity check (`median(Combined / (West + East)) <= 1.5`, tested in `test_traffic_completeness_and_tableau_4x_guard`).
  - Filters strictly on `observed_days / days_in_month == 1.0`, excluding partial Oct 2025 (`6` days) and retaining `147` clean complete months (`2013-07..2025-09`).

---

### 3. Fake Summer `SNOW_ON_GROUND` Mean Imputation & Station Gaps
- **Legacy Behavior:** Missing warm-season `SNOW_ON_GROUND` rows in `weather_df.csv` were imputed with the global daily mean—producing **>11 cm average snow depth in July and August**.
- **Remediation (`fetch_weather_monthly`):**
  - Warm-season missing snow depth (`month in {5..9}` or `MEAN_TEMPERATURE > 2.5°C`) is fixed to `0.0 cm`.
  - Isolated missing station temperature/precipitation days at **BANFF CS** (`3050519`) are repaired using month-bias-adjusted **Open-Meteo ERA5** (`r = 0.9935`) + live ECCC OGC + Open-Meteo trailing forecast updates.

---

### 4. Statistics Canada WDS Vector-Order Mismatch
- **Legacy Behavior:** Batch StatCan `getDataFromVectorsAndLatestNPeriods` responses were parsed positionally rather than keyed by `vectorId`.
- **Remediation (`fetch_statcan_vectors`):** Strictly keys parsed payloads by `int(obj["vectorId"])` (verified under deliberately reversed mock payloads in `test_statcan_vector_id_mapping_out_of_order`).

---

### 5. Pre-2017 Flat FX Backfill Contamination & Unmodeled Residual Hierarchy
- **Root Defect:** Because Bank of Canada `FXUSDCAD` starts in Jan 2017, naive `.bfill()` created 42 constant pre-2017 rows (`~1.307`) that washed out contemporaneous `log_fx` into Statistical insignificance (`p = 0.70`) and left `AB_LFS_UnempRate_SA` disconnected from scenario simulations. Meanwhile, modeling only `(Food, Garbage, Collection)` omitted the remaining 8 core streams (`Other Diversion`), leaving baseline and CV totals unreconciled.
- **Remediation (`web_exogenous.py`, `models.py`, `scenarios.py`):**
  - Added explicit `is_realized_fx` and `is_realized_hwy1` domain provenance flags; restricted Stage-1 structural `SARIMAX(1,0,0)` to genuinely realized FX (`2017-06+`, `n=100`) using booking-lead `log_fx_ma3_lag3` (`+1.504, p=0.0026`) and `ab_unemp_rate` (`-0.0210/pp, p=0.0098`).
  - Promoted **`Other Diversion`** to a first-class CV/baseline/scenario target and added closed-form **`reconcile_waste_mint`**, guaranteeing `Food + Garbage + Other === Total Collection` everywhere while winning #1 OOS RMSE on `Total Garbage` (`24.85 t`) and `Other Diversion` (`19.49 t`).

---

### 6. Multiscale Pooling, Leakage, LightGBM Collapse & Unbounded Waste Trend
- **Legacy Behavior:** Pooled incompatible scales (vehicles vs. tonnes), leaked test-period traffic into Stage-2 backtests, included collinear `t_years + diversion_ramp` in Waste models causing `SARIMAX` collapse, and left regularized LightGBM (`min_child_samples=20` on `n=48`) collapsing to zero tree splits.
- **Remediation (`models.py`, `evaluation.py`):**
  - Strictly separate Stage-1 OOF traffic (`Ensemble_Top3` with expanding inner-holdout inverse-RMSE weights) from Stage-2 Waste folds.
  - Tuned `Hybrid_Ridge_LGBM` (`min_child_samples=3, reg_lambda=1.0, reg_alpha=0.2, n_estimators=45`), dropping Food Waste OOS RMSE from `20.42 t` (`Ridge 22.06 t`) down to **`18.03 t`** (`10.57%` MAPE), verified in `test_cv_no_leakage_lgbm_noncollapse_and_mint`.
  - Removed unbounded `t_years` from Waste feature selectors and replaced hardcoded ensemble blend weights with data-driven expanding inner-holdout inverse-RMSE weighting.

---

### 7. Uncontrolled Residual Autocorrelation & Uncalibrated Bridge Uncertainty
- **Legacy Behavior:** Static OLS scenario regressions exhibited severe serial correlation (`DW = 0.21–0.31`), and `TrafficNowcaster` bridge months had no uncertainty intervals.
- **Remediation (`scenarios.py`, `models.py`):** Replaced OLS with `SARIMAX(1,0,0)` + 11 monthly fixed effects (`DW = 1.921–2.432`, Ljung-Box `p = 0.106–0.605`), calibrated trailing 12-month OOS bridge errors (`2.70%` Hwy 1 MAPE) for `2025-10..2026-09` PIs, and bias-centered log-normal conformal intervals.
