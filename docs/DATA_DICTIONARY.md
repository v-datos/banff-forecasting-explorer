# Data Dictionary — Inputs, Live Web Exogenous APIs & Exported Deliverables

**Project:** `banff-forecasting` (`v0.1.0`)  
**Temporal Frequency:** Monthly (`ME`, calendar month-end dates `YYYY-MM-DD`)  
**Full Analytical Span:** `2013-07-31` through `2027-12-31` (`174` months)

---

## 1. Raw Canonical Inputs (`data/raw/`)

| File | Coverage | Format | Description & Critical Schema Notes |
| :--- | :--- | :--- | :--- |
| `zero_waste_dashboard_data.csv` | `2018-01..2024-12` (`84` months) | UTF-8 CSV | Canonical static Town of Banff Zero Waste Dashboard monthly tonnage archive (the upstream web dashboard is no longer online). **Important:** Despite being labeled `Tonnes`, numeric values are stored in **kilograms** and must be divided by `1000.0` to obtain metric tonnes (`load_waste_targets()`). |
| `tw_traffic_data.csv` | `2013-07-01..2026-11-11` (`4,882` complete daily rows; `160` complete months through `2026-10`) | UTF-16 TSV / UTF-8 CSV supported | Daily two-way traffic counter counts (`Combined TW` / `Combined_TW`, `Tw (002)` / `West_Entrance`, `Sum of TW` / `East_Entrance`, `Tw (003)` / `Mountain_Ave`), synced live from Tableau Public `BanffTrafficData-GS.twb` via `pantab`. Enforces explicit `groupby(["Counter", "Date"])`, `(Counter, TimeStamp)` deduplication, zero silent outage zero-fills on Counter `3`, `>= 23h/day` on primary counters `1` & `2`, partial trailing snapshot day filtering (`incomplete_day_ratio=0.15`), `completeness == 1.0`, and the 4x Tableau crosstab inflation guard (`median <= 1.5`). |
| `weather_df.csv` | `1995-04-01..2025-01-05` (`10,872` daily rows) | UTF-8 CSV | Historical Environment and Climate Change Canada (ECCC) daily station archive for **BANFF CS** (`CLIMATE_IDENTIFIER=3050519`, `51.1934°N, -115.5522°E`). Contains un-imputed gaps resolved cleanly in `fetch_weather_monthly()`. |
| `totals_wide_df_legacy.csv` | `2018-01..2024-12` | UTF-8 CSV | Legacy spliced workbook artifact retained **solely** as an audit baseline for `audit_legacy_splice_vs_clean()` and Figure 1 verification. Never used in model training or forecasting. |

### 1.1 Complete 33-Material Taxonomy in `zero_waste_dashboard_data.csv`

All `Total Collection` analyses aggregate `CORE_13_MATERIALS` as a mutually exclusive, collectively exhaustive 3-way partition (`Food Waste + Garbage + Other Diversion === Total Collection`):
- **Food Waste Streams (`2` streams → `Total Food Waste`):**  
  `Commercial Food Waste`, `Residential Food Waste`
- **Garbage Streams (`3` streams → `Total Garbage`):**  
  `Commercial Garbage`, `Residential Garbage`, `Town Run`
- **Other Diversion Core Streams (`8` streams → `Other Diversion`):**  
  `Commercial Cardboard`, `Commercial Paper`, `Commercial Recycables`, `Residential Glass`, `Residential Paper`, `Residential Recycables`, `Sundry`, `Yard and Garden`  
  *(Together, the `2 + 3 + 8 = 13` streams equal `Total Collection` = `7,281.867 t` in CY 2024 with exact `0.0000 t` discrepancy.)*
- **Non-Core / Bulk / Special Streams (`20` streams, retained only in `All_33_Materials_Total`):**  
  `Ballasts`, `Batteries`, `Commercial Glass`, `Concrete`, `Construction`, `Dry Wall`, `Electronics`, `Flourescent Bulbs`, `Flying Pigs`, `Food Rescue`, `Gravel`, `HHW`, `Kiln Dried Wood`, `Mattresses/Box Springs`, `Metal`, `Propane Tanks`, `Re-Use it`, `Scrub & Brush`, `Tires`, `Unwanted Item Pick Up`

---

## 2. Live Web Exogenous Sources & Cached Tables (`data/cache/`)

All external web API calls persist deterministic CSV snapshots under `data/cache/` (`refresh=False` loads cache; `refresh=True` hits live government APIs with automatic fallback to cache on transient network errors).

### 2.1 Statistics Canada WDS (`data/cache/statcan_macro.csv`, `182` rows × `19` cols)
Endpoint: `POST https://www150.statcan.gc.ca/t1/wds/rest/getDataFromVectorsAndLatestNPeriods`  
Every series is strictly keyed by integer `vectorId` (`STATCAN_VECTORS`):

| Column | `vectorId` | Coverage | Description & Units |
| :--- | :---: | :--- | :--- |
| `AB_CPI_All` | `41692327` | `2011-10..2026-09` | Alberta CPI, All-items (`2002=100`) |
| `AB_CPI_Food` | `41692328` | `2011-10..2026-09` | Alberta CPI, Food (`2002=100`) |
| `AB_CPI_Gasoline` | `41692401` | `2011-10..2026-09` | Alberta CPI, Gasoline (`2002=100`) |
| `AB_CPI_Energy` | `41692458` | `2011-10..2026-09` | Alberta CPI, Energy (`2002=100`) |
| `CA_CPI_All` | `41690973` | `2011-10..2026-09` | Canada CPI, All-items (`2002=100`) |
| `AB_FoodServices_Receipts_SA` | `42170714` | `2011-09..2026-08` | Alberta Food Services & Drinking Places Receipts, SA (`$000s CAD`) |
| `AB_LFS_Pop` | `2064510` | `2011-11..2026-10` | Alberta Population 15+ (`000s`) |
| `AB_LFS_Emp_SA` | `2064512` | `2011-11..2026-10` | Alberta Employment, SA (`000s`) |
| `AB_LFS_UnempRate_SA` | `2064516` | `2011-11..2026-10` | Alberta Unemployment Rate, SA (`%`) |
| `AB_AvgHourlyWage` | `3075059` | `2011-11..2026-10` | Alberta Average Hourly Wage Rate (`$ CAD`) |
| `AB_Retail_Total_SA` | `1446859973` | `2017-01..2026-08` | Alberta Total Retail Trade Sales, SA (`$000s CAD`) |
| `CA_GDP_AllInd_M` | `65201210` | `2011-09..2026-08` | Canada Monthly Real GDP All Industries, SAAR (`2017 Chained $M`) |
| `CA_GDP_AccomFood_M` | `65201468` | `2011-09..2026-08` | Canada Monthly Real GDP Accommodation & Food [NAICS 72], SAAR |
| `AB_Visitors_US` | `1277197675` | `2011-09..2026-08` | US Residents Entering Canada via Alberta (monthly travelers) |
| `AB_Visitors_Overseas` | `1277197676` | `2011-09..2026-08` | Overseas Residents Entering Canada via Alberta (monthly travelers) |
| `AB_FoodSvc_Real` *(Derived)* | — | `2011-10..2026-08` | `AB_FoodServices_Receipts_SA / AB_CPI_All * 100` (real hospitality volume) |
| `AB_Retail_Real` *(Derived)* | — | `2017-01..2026-08` | `AB_Retail_Total_SA / AB_CPI_All * 100` |
| `AB_Real_Wage` *(Derived)* | — | `2011-11..2026-09` | `AB_AvgHourlyWage / AB_CPI_All * 100` |
| `AB_Visitors_Intl` *(Derived)* | — | `2011-09..2026-08` | `AB_Visitors_US + AB_Visitors_Overseas` |

### 2.2 Bank of Canada FX (`data/cache/boc_fx.csv`, `167` rows)
Endpoints: `GET https://www.bankofcanada.ca/valet/observations/FXUSDCAD/json?start_date=2013-01-01` + legacy noon series `GET https://www.bankofcanada.ca/valet/observations/IEXE0101/json?start_date=2013-01-01&end_date=2017-04-30`:
- `FXUSDCAD`: Monthly arithmetic mean of daily USD/CAD spot rates (`>= 5` business days required; modern `FXUSDCAD` covers `2017-01..2026-11`, and overlap-scaled BoC legacy `IEXE0101` extends realized history seamlessly back across `2013-01..2016-12`, giving `167` realized monthly rows `2013-01..2026-11` and `161` analytical realized rows `2013-07..2026-11`). Any un-spliced pre-sample dates would automatically be flagged `is_realized_fx = False`. Forward values beyond the realized cutoff use trailing 3-month mean.

### 2.3 Alberta Economic Dashboard Tourism (`data/cache/alberta_tourism.csv`, `369` rows)
- `Banff_Park_Attendance`: Monthly attendance from `https://api.economicdata.alberta.ca/data?table=NationalParkAttendance&Type=Standard&Indicator=Attendance&TourismName=Banff%20National%20Park` (`2012-04..2026-09`, `n=174`).
- `Hwy1_Banff_Vehicles`: Monthly vehicle counts from `https://api.economicdata.alberta.ca/data?table=Highway_count` filtered to `TourismName == "Hwy 1 - Banff Park Gates"` & `Type == "Total Vehicles"` (`1996-01..2026-08`, `n=364`, explicitly tracked by `is_realized_hwy1`).
- Extended past cutoff via scale-normalized damped seasonal ETS fitted on the post-COVID regime (`>= 2021-07`).

### 2.4 Reconstructed Monthly Climate (`data/cache/weather_monthly.csv`, `160` rows)
Combines `weather_df.csv` (`1995–2025-01`), live **ECCC OGC** (`https://api.weather.gc.ca/collections/climate-daily/items`, `CLIMATE_IDENTIFIER=3050519`), **Open-Meteo ERA5** (`https://archive-api.open-meteo.com/v1/archive`), and Open-Meteo trailing forecast (`https://api.open-meteo.com/v1/forecast`) over `2013-07..2026-10` (`n=160` complete months):
- `MEAN_TEMPERATURE` (°C, monthly mean), `TOTAL_PRECIPITATION` (mm, monthly sum), `HEATING_DEGREE_DAYS` (sum below 18°C), `COOLING_DEGREE_DAYS` (sum above 18°C), `SNOW_ON_GROUND` (cm, monthly mean; `0.0 cm` enforced for warm summer missing days), `PRECIP_DAYS` (`days >= 1.0 mm`), `FROST_DAYS` (`days <= 0.0°C`), `TEMP_NORMAL_10Y` (`2016–2025` climatology), `TEMP_ANOM` (`MEAN_TEMPERATURE - TEMP_NORMAL_10Y`).

### 2.5 Master Panel (`data/cache/exogenous_panel.csv`, `174` rows × `36` data cols)
Aligns all `31` exogenous columns plus **5 provenance boolean flags** across `2013-07-31..2027-12-31`:
- `is_realized_weather`: `True` across `2013-07-31..2026-10-31` (`n=160`)
- `is_realized_macro`: `True` across `2013-07-31..2026-08-31` (`n=158`, `AB_FoodServices_Receipts_SA` cutoff)
- `is_realized_tourism`: `True` across `2013-07-31..2026-09-30` (`n=159`, `Banff_Park_Attendance` cutoff)
- `is_realized_fx`: `True` across `2013-07-31..2026-11-30` (`n=161`, BoC `IEXE0101` + `FXUSDCAD` domain window)
- `is_realized_hwy1`: `True` across `2013-07-31..2026-08-31` (`n=158`, observed `Hwy1_Banff_Vehicles` domain window)

---

## 3. Exported Outputs (`outputs/`)

### 3.1 Forecast & Diagnostic CSVs (`outputs/forecasts/`)

| File | Shape | Key Columns | Purpose |
| :--- | :---: | :--- | :--- |
| `baseline_monthly_2025_2027.csv` | `36 × 27` | `ds`, `phase`, 5 series (`Total Vehicles`, `Food Waste`, `Garbage`, `Other Diversion`, `Total Collection`), `{Target}_pi80_low/high`, `{Target}_pi95_low/high` | Complete 36-month MinT-reconciled baseline (`2025-01..2027-12`) with phase provenance (`Observed Traffic \| Realized-Driver Waste` through `2026-10` plus `Forward Baseline Forecast` for `2026-11..2027-12`), fallback bridge PIs when applicable, and bias-centered conformal PIs. |
| `cv_fold_metrics.csv` | `148 × 11` | `target`, `fold`, `n_test`, `model`, `is_common_fold`, `MAE`, `RMSE`, `MAPE`, `sMAPE`, `MASE`, `Bias_pct` | Per-fold OOS metrics (`5 series × 3 folds × 8 models + 4 waste × 3 folds × MinT_Coherent + 8 Fold_2025 + 8 Fold_2026_YTD`). |
| `cv_summary_metrics.csv` | `44 × 8` | `target`, `model`, `MAE`, `RMSE`, `MAPE`, `sMAPE`, `MASE`, `Bias_pct` | Mean metrics across the 3 common annual folds (`2022–2024`) ranking all 8 base models + causal leave-one-fold-out `MinT_Coherent`. |
| `data_audit_comparison.csv` | `7 × 10` | `ds` (`2018..2024`), `{WasteTarget}_clean/legacy/diff` | Annual verification proving exact `0.000 t` CY 2024 match vs. legacy pre-2024 splice breaks. |
| `elasticity_diagnostics.csv` | `5 × 11` | `Target`, `n`, `Traffic Elasticity`, `Temp Coef (/+1C)`, `Precip Elasticity`, `Macro/FX Elasticity`, `AB Unemp Coef (/+1pp)`, `AR(1) Phi`, `Log R2`, `Durbin-Watson`, `Ljung-Box p(12)` | Two-stage AR(1) structural regression estimates across Stage 1 (`Total Vehicles`, `n=155`, incl. lagged FX & AB Unemployment) and Stage 2 (all 4 waste streams, `n=84`). |
| `scenarios_monthly_2025_2027.csv` | `216 × 8` | `ds`, `Total Vehicles`, `Total Food Waste`, `Total Garbage`, `Total Collection`, `Other Diversion`, `phase`, `scenario` | Long-format monthly counterfactual trajectories (`36 months × 6 scenarios`) with exact `Food + Garbage + Other === Collection` at every row. |
| `scenarios_annual_2027.csv` | `6 × 11` | `scenario`, 5 annual sums (`Total Vehicles`, `Total Food Waste`, `Total Garbage`, `Total Collection`, `Other Diversion`), and 5 `(% vs Base)` columns | Calendar Year 2027 totals and percentage deltas vs. Baseline across all 6 presets. |

### 3.2 Figures (`outputs/figures/`)

1. `01_data_audit_fixes.png` — Side-by-side verification of 2024 waste splicing fix & summer `SNOW_ON_GROUND` fix.
2. `02_cv_model_benchmark.png` — Horizontal bar comparison of `3-Yr OOS MAPE` and `MASE` across all models + `MinT_Coherent`.
3. `03_baseline_forecasts_with_intervals.png` — Observed actuals (`2019–2026-10` for Vehicles; `2019–2024` for Waste), `2025–2026-10` realized-driver waste estimates (plus Hwy 1 / Park bridge PIs whenever Tableau lags), and `2026-11..2027-12` MinT-reconciled forecasts with `80% / 95%` conformal PIs.
4. `04_scenario_comparisons.png` — `2026–2027` monthly scenario divergence trajectories with CY 2027 percentage callouts.
5. `05_structural_elasticities.png` — 3-panel AR(1) forest plots showing Stage-1 macro/climate drivers (`FXUSDCAD`, `AB Unemp`), Stage-2 traffic-to-waste elasticities across all 4 waste series, and hospitality/climate sensitivities with 95% CIs.
