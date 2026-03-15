# Monetary Policy Shocks and Startup Funding Transitions

**How do monetary policy shocks influence the timing of startup transitions from Seed funding to Series A?**

This project uses discrete-time hazard modeling to estimate the effect of unanticipated changes in the federal funds rate on the pace at which FinTech and AI startups progress from Seed to Series A funding. The analysis covers 2016 Q1 through 2023 Q3, a period spanning two tightening cycles, a pandemic-era easing, and the fastest rate-hike campaign in four decades.

**Author:** Ricardo Sanchez

---

## Key Findings

The Kaplan-Meier analysis and log-rank tests (Notebook 05) established the following patterns, which directly motivated the hazard model specifications:

| Finding | Log-Rank p-value | Implication |
|---------|-----------------|-------------|
| Startups seeded during **contractionary** quarters transition to Series A significantly slower | **0.008** vs. Neutral, **0.046** vs. Expansionary | Core finding — contractionary monetary policy delays VC deployment |
| **SF Bay Area** transitions significantly faster than all other metros | 0.001 vs. GLA, 0.028 vs. GMA, 0.027 vs. GNYA | Motivates the SFBA × shock interaction model |
| **Sector does not matter** — FinTech and AI transition at indistinguishable rates | 0.962 | Sector FE retained as control but not expected to be significant |
| **Cohort timing** does not significantly predict transition speed | > 0.43 for all pairwise comparisons | Cohort FE retained for unobserved time trends, not driving the result |

## Methodology

The analysis uses a **complementary log-log (cloglog) discrete-time hazard model** estimated on a startup × quarter person-period panel. This approach is appropriate because:

1. The underlying transition process is continuous, but we observe it at discrete quarterly intervals.
2. The cloglog link maps to a proportional hazards interpretation, making coefficients comparable to Cox model hazard ratios.
3. Time-varying covariates (monetary shocks, macro controls) change each quarter and enter naturally in the person-period format.

Standard errors are **clustered by calendar quarter** to account for within-period correlation in macro shocks.

Three model specifications are estimated, each motivated by a specific finding from the descriptive analysis:

| Model | Description | Motivation |
|-------|-------------|------------|
| **Baseline** | Full specification: shocks + 5 lags, macro controls, firm covariates, duration/cohort/sector/metro FE | Establishes the core relationship with all standard controls |
| **Parsimonious** | Drops sector FE (p = 0.962) and cohort FE (p > 0.43); retains all other controls | Tests whether the shock effect survives without non-significant fixed effects |
| **SFBA Interaction** | Adds SFBA × shock and SFBA × lag1 interactions to the baseline | Tests whether the Bay Area's deep VC ecosystem buffers startups from contractionary shocks |

## Data

**Startup funding data** is sourced from [Crunchbase](https://www.crunchbase.com/) and covers Seed and Series A rounds for FinTech and AI startups. The raw data was downloaded in multiple export batches and merged during cleaning.

**Macroeconomic data** includes:

| Variable | Source | Frequency | Coverage |
|----------|--------|-----------|----------|
| Monetary policy shocks | High-frequency surprise measure | Quarterly | 2014 Q1 – 2023 Q3 |
| Core CPI (YoY % change) | FRED (`CPILFESL`) | Quarterly | 2015 Q1 – 2025 Q3 |
| Real GDP (YoY % change) | FRED (`GDPC1`) | Quarterly | 2015 Q1 – 2025 Q3 |

> **Note:** The raw Crunchbase data is proprietary and not included in this repository. The macro CSVs are included in `data/raw/macro/`.

## Project Structure

```
├── README.md
│
├── metro_regions.ipynb                       # Reference: city-to-metro lookup table
├── 01_exploratory_data_analysis.ipynb        # Raw data profiling (startup + macro)
├── 02_fintech_data_cleaning.ipynb            # FinTech: raw exports → survival dataset
├── 03_AI_data_cleaning.ipynb                 # AI: raw exports → survival dataset
├── 04_startup_panel_with_shocks_final.ipynb  # Panel construction + macro merges + feature engineering
├── 05_descriptive_statistics.ipynb           # Summary stats, KM curves, log-rank tests
├── 06_discrete_time_hazard_analysis.ipynb    # Three cloglog hazard models + comparison
│
├── data/
│   ├── raw/
│   │   ├── fintech/                          # Crunchbase FinTech exports (not tracked)
│   │   ├── AI/                               # Crunchbase AI exports (not tracked)
│   │   └── macro/                            # Monetary shocks, Core CPI, Real GDP
│   ├── cleaned/
│   │   ├── cleaned_fintech/                  # fintech_simple_survival.csv
│   │   ├── cleaned_ai/                       # ai_simple_survival.csv
│   │   ├── fintech_loc_map/                  # City-to-metro mapping (FinTech)
│   │   ├── ai_loc_map/                       # City-to-metro mapping (AI)
│   │   ├── discrete_time_hazard_data/        # Final person-period panel
│   │   └── city_to_metro.csv                 # Combined metro lookup (from metro_regions.ipynb)
│   └── reference/                            # Static lookup tables
│
└── outputs/
    ├── tables/                               # Model results (if exported)
    └── figures/                              # Plots (if exported)
```

## Notebook Pipeline

The notebooks are numbered in execution order. Each notebook documents its inputs and outputs in a header cell.

```
metro_regions ──────────────────────────┐
                                        │  (city-to-metro lookup)
01_EDA ─────────────────────────────────│── (exploration only, no outputs)
                                        ▼
02_fintech_cleaning ──┐                 │
                      ├──→ 04_panel_construction ──→ 05_descriptive_stats ──→ 06_hazard_models
03_AI_cleaning ───────┘
```

| Notebook | Purpose | Key Outputs |
|----------|---------|-------------|
| `metro_regions` | Build city-to-metro lookup table (LA, SFBA, GNY, Miami) from manually curated city lists | `city_to_metro.csv` |
| `01` | Profile raw data quality: schema comparison, package overlap, date ranges, missing values, funding amounts, location distributions, macro coverage and correlations | — |
| `02` | Merge FinTech seed packages, build survival dataset (one row per firm), assign metro regions, attach covariates | `fintech_simple_survival.csv` |
| `03` | Same pipeline for AI sector (handles mixed CSV/XLSX inputs) | `ai_simple_survival.csv` |
| `04` | Union sectors, expand to startup × quarter person-period panel, merge macro variables (shocks, CPI, GDP), create shock lags (L1–L5), feature engineering (winsorize + log seed amount, log partner investors, duration bins) | `startup_seriesA_person_period_quarterly_final_all.csv` |
| `05` | Summary statistics by sector/metro/cohort, Kaplan-Meier curves (overall, by sector, by metro, by cohort, by shock regime), pairwise log-rank tests | Inline plots and tables |
| `06` | Three cloglog discrete-time hazard models (baseline, parsimonious, SFBA interaction), coefficient comparison table, model fit statistics | Model summaries |

## Variables

### Dependent Variable

| Variable | Description |
|----------|-------------|
| `event_seriesA` | Binary indicator: 1 in the quarter a startup receives Series A, 0 otherwise |

### Key Independent Variables

| Variable | Description |
|----------|-------------|
| `shock_25bp_units` | Monetary policy shock in the current quarter (in 25-basis-point units) |
| `shock25_lag1` – `shock25_lag5` | 1- to 5-quarter lags of the monetary policy shock |
| `core_cpi_yoy` | Core CPI year-over-year % change (excludes food and energy) |
| `real_gdp_yoy` | Real GDP year-over-year % change |

### Firm-Level Covariates

| Variable | Description |
|----------|-------------|
| `log_seed_amt` | Log of winsorized (1st/99th percentile) seed funding amount in USD |
| `log_npi` | Log of number of partner investors (zero-filled for missing values) |

### Fixed Effects

| Variable | Description |
|----------|-------------|
| `dur_bin` | Duration bins (0–2, 3–5, 6–8, 9–12, 13+ quarters) — piecewise-constant baseline hazard |
| `cohortY_*` | Seed year dummies — cohort fixed effects (not significant in KM, retained as controls) |
| `sector_*` | Sector dummy (FinTech vs. AI; not significant in KM, retained as control) |
| `metro_*` | Metro region dummies (SFBA, GNYA, LA, Miami; SFBA is significantly faster) |

### Model 3 — Interaction Terms

| Variable | Description |
|----------|-------------|
| `sfba_x_shock` | SFBA indicator × contemporaneous monetary policy shock |
| `sfba_x_shock_lag1` | SFBA indicator × one-quarter lagged monetary policy shock |

## Metro Regions

The four metro regions used as fixed effects in the hazard model are defined in `metro_regions.ipynb`:

| Code | Region | Cities Mapped |
|------|--------|---------------|
| `LA` | Greater Los Angeles | LA County, Orange County, Inland Empire, Ventura County |
| `SFBA` | San Francisco Bay Area | SF, Peninsula, South Bay, East Bay, North Bay, Santa Cruz |
| `GNY` | Greater New York Area | NYC boroughs, Long Island, Northern NJ, Hudson Valley |
| `Miami` | Greater Miami Area | Miami-Dade, Broward, Palm Beach counties |

Startups outside these four metros are assigned to an implicit "Other" category during the cleaning merge.

## Requirements

```
python >= 3.10
pandas
numpy
matplotlib
statsmodels
lifelines
```

Install dependencies:

```bash
pip install pandas numpy matplotlib statsmodels lifelines
```

## How to Reproduce

1. **Data access:** Obtain Crunchbase export files for FinTech and AI Seed/Series A funding rounds (2016–2023 Q3). Place them in `data/raw/fintech/` and `data/raw/AI/` following the file naming conventions documented in notebook `01`.

2. **Run notebooks in order:**
   ```
   metro_regions → 01 → 02 → 03 → 04 → 05 → 06
   ```
   `metro_regions` must run first to produce the city-to-metro lookup CSV. Notebook `01` is exploratory and does not produce output files, but should be reviewed before proceeding. Notebooks `02` and `03` can run in either order. From `04` onward, execution must be sequential.

3. **Note on paths:** Notebooks currently use relative Windows-style paths (e.g., `..\\data\\raw\\...`). If running on macOS or Linux, replace backslashes with forward slashes or update to use `pathlib.Path` objects.

## License

This project was developed as an academic capstone. The code is available for educational and research purposes. Crunchbase data is subject to Crunchbase's terms of service and is not redistributed in this repository.
