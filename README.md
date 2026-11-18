# ClimRisk-LATAM: climate-conditioned robust portfolios in Latin American equity markets

Replication notebook for the paper *Climate-Conditioned Robust Portfolios in Latin American Equity Markets*. Notebook file: `ClimRisk_LATAM.ipynb`.

The notebook builds the model, downloads the data, runs the out-of-sample evaluation and exports every table and figure used in the paper. It is self-contained: each section starts with the theory it implements, and all intermediate results are printed as they are computed.

## 1. What the notebook does

The model is a distributionally robust CVaR portfolio over a type-2 Wasserstein ball. Climate information enters at every rebalancing date in three places:

1. Nominal measure. A kernel on stationary climate features (MEI level, 3-month change of the MEI, detrended temperature anomaly, 3-month commodity return) gives more weight to historical months whose climate state resembles the current one.
2. Transport metric. The cost is $(r-r')^\top M_t (r-r')$ with $M_t^{-1}=\Sigma^{\text{OAS}}_t+\kappa B_t\Lambda_t B_t^\top$, where $B_t$ are the betas of returns on climate innovations filtered with a linear Gaussian state-space model (Kalman filter, EM estimation), and $\Lambda_t$ is their EWMA covariance.
3. Feasible set. A transition constraint aligned with the EU Paris-aligned benchmark (50% reduction, 7% per year) and a physical constraint on a measured exposure (response to ENSO news plus WRI Aqueduct water risk).

The worst-case CVaR has the closed form $\text{CVaR}_\pi + (\varepsilon/\sqrt{\alpha})\,\|G_t^\top w\|_2$ with $G_tG_t^\top=M_t^{-1}$, so each decision is a second-order cone program (CLARABEL, with ECOS and SCS as fall-backs). The radius is chosen by rolling-origin cross-validation.

Evaluation: Brazil, Mexico, Colombia and Chile (55 stocks), first decision at the end of December 2017, 96 out-of-sample months (January 2018 to December 2025), quarterly rebalancing, 25 bp transaction costs, USD returns, 13-week Treasury bill as risk-free rate. Benchmarks: constrained 1/N, minimum variance with OAS covariance (MVO-OAS), empirical CVaR (ε = 0), Euclidean Wasserstein DRO, OAS-metric Wasserstein DRO, and the country ETF. Inference: stationary bootstrap with automatic block length (Politis–White), Diebold–Mariano with HLN correction, Holm adjustment and a pooled four-market panel.

## 2. Structure

| Section | Content | Runs computations? |
|---|---|---|
| §0–§10 | Theory and model engine: protocol and parameters, climate state, state-space model, nominal measure, metric, feasible set, closed-form worst case, metrics, inference, backtest | Definitions only |
| §A | Analysis functions: ablations, radius frontier, panel inference, figures, replication of the submitted manuscript's radius | Definitions only |
| §D | Documentary data (universe, expert prior, location of operations, corporate events) and download functions | Definitions only |
| §E0 | Run information (date, Python, package and solver versions) and data snapshot | Yes |
| §E | Data download, exposures and assembly by market | Yes |
| §F0 | Diagnostics before the evaluation | Yes |
| §F1–§F3 | Main backtest, weights and feasible set, inference | Yes |
| §F4 | Ablations, CVaR–ε frontier, alternative physical-exposure measures (`RUN_F4`) | Yes, the longest |
| §F5 | State-space diagnostics | Yes |
| §F6 | Robustness design: estimation 2014–2021, evaluation 2022–2025 (`RUN_F6`) | Yes |
| §F7 | Equivalence tests (TOST) of ΔCVaR, pooled and by market | Yes, seconds |
| §F8 | Convergence of the EM algorithm, sensitivity to initial conditions, spectral bound and iteration limit, and DRO-CC under each alternative (`RUN_F8`) | Yes, long |
| §F9 | Inputs not available at each decision: PHYS without Aqueduct data, no physical constraint, universe known at the first decision (`RUN_F9`) | Yes |
| §F10 | Export of LaTeX tables, environment, data vintage, solver status and saved results | Yes |
| §G | Annex for the referees: specification of the submitted manuscript | Yes |

Cells must be run in order. All parameters are in the `PARAMS` dictionary of §1 and are exported to `tables/params.tex`.

## 3. Requirements

- Google Colab with a standard CPU runtime. A GPU is not used; selecting one only consumes quota.
- Python 3.10 or later if run locally. Packages: `numpy`, `pandas`, `scipy`, `scikit-learn`, `matplotlib`, `cvxpy`, `clarabel`, `ecos`, `xarray`, `netCDF4`, `requests`, `yfinance`. The first code cell installs the packages that Colab does not ship by default; outside Colab add `yfinance` and `scikit-learn` to that line.
- Internet access to Yahoo Finance, NOAA PSL (`psl.noaa.gov`, `downloads.psl.noaa.gov`, and the ERDDAP mirror `upwell.pfeg.noaa.gov`) and datahub.io (World Bank carbon-price data).

## 4. Files to provide

| File | Required | Where to put it | Source |
|---|---|---|---|
| WRI Aqueduct 4.0 zip (`aqueduct-4-0-water-risk-data.zip`, or a zip that contains it) | Yes | Working directory (`/content` in Colab) | WRI, Aqueduct 4.0 data: https://www.wri.org/data/aqueduct-global-maps-40-data |
| `indices/<CODE>.csv` (columns `date`, `close`; `CODE` in BR, MX, CO, CL) | No | `indices/` folder in the working directory | Optional external reference for a market. If the file exists it replaces the Yahoo download of that market's ETF; dates are read day-first |

The notebook finds the Aqueduct zip with the pattern `*queduct*.zip`, extracts it into `aqueduct/` and reads only the annual baseline CSV (`Aqueduct40_baseline_annual_y2023m07d05.csv`). Placing that CSV anywhere under `aqueduct/` also works. No geometry files are needed.

All other data are downloaded automatically. Three downloads are cached in the working directory and reused on later runs; if a source is unreachable, the file can be placed there manually under the same name:

| Cache file | Content |
|---|---|
| `meiv2.data` | NOAA Multivariate ENSO Index v2 (https://psl.noaa.gov/enso/mei/data/meiv2.data) |
| `gistemp_air.2x2.1200.mon.anom.comb.nc` | NASA GISTEMP v4, 1200 km smoothing, via NOAA PSL (created only when OPeNDAP access fails) |
| `wb_carbon_prices.csv` | World Bank Carbon Pricing Dashboard data, columns `initiative, type, jurisdiction, year, price_usd_per_tco2e` |

### Data snapshot

Section §E0 saves the output of every download (Yahoo Finance, MEI, GISTEMP, World Bank) in `data_snapshot/`, one pickle file per call, and records in `data_snapshot/manifest.json` the UTC time of the download and the SHA-256 hash of each file. With `SNAPSHOT_MODE = 'auto'` (default) a later run reads the saved files instead of downloading again and checks their hashes, so that the results are reproduced with exactly the same data vintage even if the providers revise their series. `SNAPSHOT_MODE = 'refresh'` downloads again and overwrites the snapshot; `'off'` disables it. To reproduce the paper exactly, place the archived `data_snapshot/` folder in the working directory before running §E. Yahoo Finance data may be subject to the provider's terms of use; the snapshot is intended for replication, and the manifest (dates and hashes) can be published in any case.

No cell generates random, synthetic or sample data. If a source fails, the notebook stops with an explicit message. The only randomness is the bootstrap resampling, controlled by `SEED`.

## 5. Directories and outputs

The notebook writes to the working directory (`/content` in Colab); no path needs to be edited.

| Location | Content |
|---|---|
| `aqueduct/` | Extracted Aqueduct files |
| `tables/` | LaTeX tables: parameters, performance, inference, pooled panel, bootstrap-block sensitivity, exposures, Aqueduct regions, carbon sources and audit, price coverage, ENSO diagnostics, PHYS dispersion and path, radius choice, constraint activity, closed-form check, average weights and attainable weights by market, ETF performance, robustness design, ablations, CVaR–ε frontier, alternative PHYS, and the annex tables |
| `wealth_<CODE>.png`, `wealth_rob_<CODE>.png` | Cumulative wealth, main and robustness designs |
| `feasible_<CODE>.png` | Maximum attainable weights at the first rebalancing |
| `eps_frontier_<CODE>.png` | Out-of-sample CVaR against a fixed radius |
| `riccati_trace.png` | Convergence of tr(P) in the state-space model |
| `eps_decomp_<CODE>.png` | Decomposition of the radius of the submitted manuscript (annex) |
| `em_loglik_paths.png` | Log-likelihood paths of the EM estimations that reach the iteration cap (§F8) |
| `tables/equivalence.tex`, `tables/em_*.tex`, `tables/lookahead.tex` | Equivalence tests (§F7), EM convergence and sensitivity (§F8), information available at each decision (§F9) |
| `tables/environment.tex`, `tables/data_vintage.tex`, `tables/solver_status.tex`, `tables/state_space_log.tex` | Software and solver versions, download dates and hashes, status of every conic problem, annual state-space estimations |
| `tables/feasible_path_<CODE>.tex`, `tables/weights_<CODE>_<strategy>.csv` | Maximum attainable weight of each asset and weights of each strategy at every rebalancing date |
| `data_snapshot/` | Saved downloads and `manifest.json` (§E0) |
| `run_info.json`, `requirements_lock.txt` | Date and time of the run, Python, package and solver versions (pinned) |
| `results_saved.pkl` | All result tables, monthly net returns, weights and attainable weights in one file (pandas objects) |

Colab deletes `/content` when the runtime is recycled; download `tables/` and the figures, or mount Google Drive and change the working directory with `os.chdir(...)` before running §E.

## 6. Running time

Measured on a 2-vCPU Intel Xeon (2.1–2.3 GHz) machine, comparable to a standard Colab CPU runtime, with the full parameters of the paper. For the measurement the downloads were replaced by local inputs with the same dimensions, so the download times are an estimate and the computing times may change somewhat with the real data.

| Section | What it computes | Time |
|---|---|---|
| Package installation (first cell) | `pip install` | about 1 min |
| §0–§D | Definitions | < 5 s |
| §E | Downloads (Yahoo Finance, NOAA, World Bank), Aqueduct reading, exposures, assembly | a few minutes, depending on the network (estimate) |
| §F0 | Diagnostics | about 10 s |
| §F1 | Main backtest, 6 strategies × 4 markets × 32 rebalancings, with cross-validated radius | 3.5–5 min |
| §F2–§F3 | Weights, feasible set, inference and panel | 20–30 s |
| §F4 | 13 ablations × 4 markets (about 7 min per market), CVaR–ε frontier (about 4 min per market), 5 alternative PHYS specifications (about 10 min) | 45–65 min |
| §F5 | State-space diagnostics | < 10 s |
| §F6 | Robustness design 2022–2025 | 2.5–3.5 min |
| §F7 | Equivalence tests | about 10 s |
| §F8 | 40 EM estimations × 8 specifications (the 2,000-iteration version takes about 20 s per estimation) and 7 reruns of DRO-CC | 30–45 min |
| §F9 | 2 specifications × 4 strategies × 4 markets | 6–10 min |
| §F10 and §G | Export of tables, saved results and annex | about 15 s |
| Total | | about 2 to 2.5 hours |

With `RUN_F4 = False`, `RUN_F8 = False` and `RUN_F9 = False` the notebook runs in about 15 minutes including downloads. The times of §F8 and §F9 are estimates from the per-estimation and per-strategy times measured on the same machine.

## 7. How to run

1. Open the notebook in Colab with a CPU runtime and upload the Aqueduct zip to `/content`.
2. Run the cells in order (Runtime → Run all). §E prints a coverage report of every series; check it before reading the results.
3. To obtain the main results first, set `RUN_F4`, `RUN_F6`, `RUN_F8` and `RUN_F9` to `False`, run everything, and later set them to `True` and run only those cells and §F10. §F9 uses the robustness design of §F6 for its last row; the export in §F10 writes the tables of each section only if they exist.
4. To change a parameter (for example the transition threshold or the radius grid), edit `PARAMS` in §1 and rerun from that cell.
5. To archive the run that supports the paper: save the executed notebook (File → Download .ipynb, with outputs), and download `tables/`, the figures, `results_saved.pkl`, `run_info.json`, `requirements_lock.txt` and `data_snapshot/` (for example `!zip -r run.zip tables data_snapshot *.png *.json *.txt *.pkl` and download `run.zip`). Commit them to the repository, create a tagged release and archive it (for instance with the GitHub–Zenodo integration) to obtain the commit hash and the DOI cited in the paper.

## 8. Reproducibility notes

- Yahoo Finance revises adjusted prices when dividends or splits are recorded, and NOAA revises recent MEI and GISTEMP values. A rerun with new downloads can therefore differ slightly from the figures in the paper; a rerun with the archived `data_snapshot/` reproduces them. The manifest, the coverage reports and the provenance tables (`tables/data_vintage.tex`, `tables/price_series.tex`, `tables/carbon_sources.tex`, `tables/carbon_audit.tex`) document the vintage used.
- `requirements_lock.txt` pins the versions of the run; outside Colab, `pip install -r requirements_lock.txt` recreates the environment.
- The bootstrap uses `SEED`; the optimization is deterministic.
- The transition intensity `CI` is a declared expert prior written in §D2. The physical exposure `PHYS` is computed from data in §D10 and §E2.
- Two inputs were not available at the first decisions: the universe of 55 stocks was selected with market capitalization in 2020, and the Aqueduct 4.0 baseline was published in 2023. Both are stated in the paper, and §F9 reports specifications that do not use them.
