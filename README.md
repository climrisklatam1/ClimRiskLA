# Clim-Risk LATAM
## ClimRisk-LA. Notebook for replicating the paper: Climate-Conditioned Robust Portfolios in Latin American Equity Markets."

Replication notebook for the paper Climate-Conditioned Robust Portfolios in Latin American Equity Markets. Notebook file: ClimRisk_LATAM.ipynb.

The notebook builds the model, downloads the data, runs the out-of-sample evaluation and exports every table and figure used in the paper. It is self-contained: each section starts with the theory it implements, and all intermediate results are printed as they are computed.

1. What the notebook does

The model is a distributionally robust CVaR portfolio over a type-2 Wasserstein ball. Climate information enters at every rebalancing date in three places:

Nominal measure. A kernel on stationary climate features (MEI level, 3-month change of the MEI, detrended temperature anomaly, 3-month commodity return) gives more weight to historical months whose climate state resembles the current one.
Transport metric. The cost is $(r-r')^\top M_t (r-r')$ with $M_t^{-1}=\Sigma^{\text{OAS}}_t+\kappa B_t\Lambda_t B_t^\top$, where $B_t$ are the betas of returns on climate innovations filtered with a linear Gaussian state-space model (Kalman filter, EM estimation), and $\Lambda_t$ is their EWMA covariance.
Feasible set. A transition constraint aligned with the EU Paris-aligned benchmark (50% reduction, 7% per year) and a physical constraint on a measured exposure (response to ENSO news plus WRI Aqueduct water risk).

The worst-case CVaR has the closed form $\text{CVaR}_\pi + (\varepsilon/\sqrt{\alpha})\,\|G_t^\top w\|_2$ with $G_tG_t^\top=M_t^{-1}$, so each decision is a second-order cone program (CLARABEL, with ECOS and SCS as fall-backs). The radius is chosen by rolling-origin cross-validation.

Evaluation: Brazil, Mexico, Colombia and Chile (55 stocks), first decision at the end of December 2017, 96 out-of-sample months (January 2018 to December 2025), quarterly rebalancing, 25 bp transaction costs, USD returns, 13-week Treasury bill as risk-free rate. Benchmarks: constrained 1/N, minimum variance with OAS covariance (MVO-OAS), empirical CVaR (ε = 0), Euclidean Wasserstein DRO, OAS-metric Wasserstein DRO, and the country ETF. Inference: stationary bootstrap with automatic block length (Politis–White), Diebold–Mariano with HLN correction, Holm adjustment and a pooled four-market panel.

3. Requirements
Google Colab with a standard CPU runtime. A GPU is not used; selecting one only consumes quota.
Python 3.10 or later if run locally. Packages: numpy, pandas, scipy, scikit-learn, matplotlib, cvxpy, clarabel, ecos, xarray, netCDF4, requests, yfinance. The first code cell installs the packages that Colab does not ship by default; outside Colab add yfinance and scikit-learn to that line.
Internet access to Yahoo Finance, NOAA PSL (psl.noaa.gov, downloads.psl.noaa.gov, and the ERDDAP mirror upwell.pfeg.noaa.gov) and datahub.io (World Bank carbon-price data).

4. Files to provide
File	Required	Where to put it	Source
WRI Aqueduct 4.0 zip (aqueduct-4-0-water-risk-data.zip, or a zip that contains it)	Yes	Working directory (/content in Colab)	WRI, Aqueduct 4.0 data: https://www.wri.org/data/aqueduct-global-maps-40-data
indices/<CODE>.csv (columns date, close; CODE in BR, MX, CO, CL)	No	indices/ folder in the working directory	Optional external reference for a market. If the file exists it replaces the Yahoo download of that market's ETF; dates are read day-first

The notebook finds the Aqueduct zip with the pattern *queduct*.zip, extracts it into aqueduct/ and reads only the annual baseline CSV (Aqueduct40_baseline_annual_y2023m07d05.csv). Placing that CSV anywhere under aqueduct/ also works. No geometry files are needed.

All other data are downloaded automatically. Three downloads are cached in the working directory and reused on later runs; if a source is unreachable, the file can be placed there manually under the same name:

Cache file	Content
meiv2.data	NOAA Multivariate ENSO Index v2 (https://psl.noaa.gov/enso/mei/data/meiv2.data)
gistemp_air.2x2.1200.mon.anom.comb.nc	NASA GISTEMP v4, 1200 km smoothing, via NOAA PSL (created only when OPeNDAP access fails)
wb_carbon_prices.csv	World Bank Carbon Pricing Dashboard data, columns initiative, type, jurisdiction, year, price_usd_per_tco2e

No cell generates random, synthetic or sample data. If a source fails, the notebook stops with an explicit message. The only randomness is the bootstrap resampling, controlled by SEED.

5. How to run

Open the notebook in Colab with a CPU runtime and upload the Aqueduct zip to /content.
Run the cells in order (Runtime → Run all). §E prints a coverage report of every series; check it before reading the results.
To obtain the main results first, set RUN_F4 = False and RUN_F6 = False, run everything, and later set them to True and run only those cells and §F7. The cells after §F4 and §F6 do not depend on them, except the export in §F7, which writes their tables only if they exist.
To change a parameter (for example the transition threshold or the radius grid), edit PARAMS in §1 and rerun from that cell.

6. Reproducibility notes
Yahoo Finance revises adjusted prices when dividends or splits are recorded, and NOAA revises recent MEI and GISTEMP values. A rerun at a later date can therefore differ slightly from the figures in the paper. The coverage reports and the provenance tables (tables/price_series.tex, tables/carbon_sources.tex, tables/carbon_audit.tex) document the vintage used.
The bootstrap uses SEED; the optimization is deterministic.
The transition intensity CI is a declared expert prior written in §D2. The physical exposure PHYS is computed from data in §D10 and §E2.
The universe of 55 stocks was selected with market capitalization in 2020; this is stated as a limitation in the paper.

Notes:
For Replication download Aqueduct 4.0 database from: https://www.wri.org/data/aqueduct-global-maps-40-data
Note: No geometry files are needed. Reads only the annual baseline CSV (`Aqueduct40_baseline_annual_y2023m07d05.csv`). Placing that CSV anywhere under `aqueduct/` also works. No geometry files are needed.





