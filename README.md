# Town of Banff — Interactive Forecast & What-If Scenario Explorer

Public web explorer, methodological documentation, forecast CSV deliverables, and diagnostic figures for the Town of Banff monthly visitation and municipal waste forecasting pipeline.

**Live Interactive Explorer (GitHub Pages):** <https://v-datos.github.io/banff-forecasting-explorer/>

> [!NOTE]
> This public repository hosts the static web explorer (`site/`), exported CSV & PNG deliverables (`outputs/`), and methodological documentation (`docs/`). The Python modeling codebase, raw/cached datasets, test suite, and executable notebook live in the companion private repository (`v-datos/banff-forecasting-pipeline`).

## Repository Contents

```text
banff-forecasting-explorer/
├── .github/workflows/
│   └── deploy-pages.yml        # Automated GitHub Pages deployment for site/
├── docs/
│   ├── METHODOLOGY.md          # Two-stage architecture, MinT reconciliation, CV design, conformal PIs, AR(1) scenarios
│   ├── DATA_DICTIONARY.md      # Raw/API schemas, 33-material taxonomy, StatCan vector IDs, deliverable CSV specs
│   └── LEGACY_AUDIT_REPORT.md  # Audit of legacy notebooks & quantified fixes
├── outputs/
│   ├── figures/                # 5 high-DPI audit, CV benchmark, baseline+PI, scenario & elasticity figures
│   └── forecasts/              # Exported CSV tables (baseline+PIs, CV metrics, CY 2027 scenarios, elasticities)
└── site/
    ├── index.html              # Standalone interactive explorer (0 external CDNs, 9 live what-if sliders, CSV export)
    └── DEPLOY.md               # Public hosting & split-repo synchronization notes
```

## Model Scope & Benchmark Summary (`2013-07..2027-12`)

- **Stage 1 (`Total Vehicles`):** Observed `2013-07..2026-10` (`n = 160` complete months, synced from Town of Banff Tableau Public), with standby `TrafficNowcaster` (Hwy 1 Park Gates & Park Attendance fallback) and `Ensemble_Top3` forward forecasts through `2027-12`.
- **Stage 2 (`Total Food Waste`, `Total Garbage`, `Other Diversion`, `Total Collection`):** Uniform 13-stream municipal partition (`Food + Garbage + Other === Total Collection` identically at every historical, CV, baseline, and scenario month) calibrated over `2018-01..2024-12` (`7,281.867 t` in CY 2024) and closed-form MinT-reconciled forward through `2027-12`.

| Target Series | Best Direct / MinT | 3-Yr OOS MAPE (`2022–2024`) | 3-Yr OOS RMSE | OOS MASE | CY 2024 Actual | CY 2027 Baseline |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| `Total Vehicles` | `Ensemble_Top3` *(2025: `5.10%`, 2026 YTD: `3.68%`)* | **5.67%** | **31,870 veh** | **0.44** | 6,688,761 | 7,010,407 |
| `Total Food Waste` | **`MinT_Coherent`** *(Direct Hybrid `10.57%` / `18.09 t`)* | **10.17%** | **17.80 t** | **0.45** | 1,973.3 t | 2,382.2 t |
| `Total Garbage` | **`MinT_Coherent`** *(Direct SeasonalNaive `7.28%` / `26.44 t`)* | **6.69%** | **24.10 t** | **0.36** | 3,612.9 t | 3,507.6 t |
| `Other Diversion` | **`MinT_Coherent`** *(Direct LogAutoETS `11.08%` / `20.63 t`)* | **10.99%** | **19.90 t** | **0.58** | 1,695.6 t | 1,593.8 t |
| `Total Collection` | `SARIMAX_Exo` / **`MinT_Coherent`** | **6.23%** / **6.38%** | **45.21 t** / **45.92 t** | **0.38** / **0.39** | 7,281.9 t | 7,483.6 t |
