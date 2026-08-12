# Appliance Energy Forecasting

Hourly forecasting of household appliance energy consumption, comparing
benchmark methods, SARIMA/SARIMAX, a feature-based gradient-boosting model, and
a time-series foundation model, under a leakage-audited rolling-origin protocol.

---

## 1. Project objective

Produce and evaluate 24-hour-ahead forecasts of household appliance energy use,
and determine which modelling approach is most suitable for practical
smart-home energy forecasting once accuracy, uncertainty, interpretability and
computational cost are all taken into account.

The project is built so that every reported number is reproducible from the
code, and so that any assumption which would not hold in live operation
(principally, knowledge of future weather) is measured and labelled rather than
hidden.

## 2. Dataset source

UCI Machine Learning Repository — *Appliances Energy Prediction* (Candanedo,
Feldheim & Deramaix, 2017):

```
https://archive.ics.uci.edu/ml/machine-learning-databases/00374/energydata_complete.csv
```

`src/data.py` fetches this URL first. If it is unreachable (some networks block
`archive.ics.uci.edu`), it falls back automatically to the mirror published by
the dataset authors, then asserts the expected row count.

## 3. Dataset description

| Property | Value |
|---|---|
| Raw observations | 19,735 |
| Raw sampling interval | 10 minutes (perfectly regular, verified) |
| Coverage | 2016-01-11 17:00 to 2016-05-27 18:00 (~4.5 months) |
| Columns | 29 (target, 2 energy channels, 18 indoor sensors, 6 weather, 2 random) |
| Missing values | 0 |
| Duplicated timestamps / rows | 0 / 0 |
| Target | `Appliances` — energy use in Wh per 10-minute interval |
| After hourly aggregation | 3,289 hourly observations |

Variable groups:

- **Target** — `Appliances`.
- **Indoor sensors** — `T1`–`T9` (temperature, °C), `RH_1`–`RH_9` (relative humidity, %).
- **Outdoor weather** (Chièvres airport station) — `T_out`, `RH_out`, `Windspeed`,
  `Visibility`, `Tdewpoint`, `Press_mm_hg`.
- **Other energy channel** — `lights` (lighting circuits, Wh).
- **Random variates** — `rv1`, `rv2`. These are noise columns included by the
  original authors as controls; they are dropped, with justification, in
  `src/data.py`.

## 4. Installation

Requires **Python 3.10 or newer** (developed and tested on 3.12).

```bash
git clone <your-repo-url>
cd appliance-energy-forecasting

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

### Required packages

Core (all mandatory): `numpy`, `pandas`, `scipy`, `matplotlib`, `scikit-learn`,
`statsmodels`.

Optional:

- `chronos-forecasting` — needed for the foundation-model section (Part 8).
  Without it the pipeline runs to completion and records the foundation model as
  `NOT EXECUTED`.
- `xgboost` / `lightgbm` — alternative gradient-boosting backends. The default
  backend is scikit-learn's `HistGradientBoostingRegressor`, so neither is
  required.

## 5. Project structure

```
appliance-energy-forecasting/
├── README.md
├── requirements.txt
├── .gitignore
│
├── data/
│   ├── raw/                    # cached energydata_complete.csv
│   └── processed/              # appliance_hourly.csv
│
├── src/
│   ├── config.py               # all constants and experiment settings
│   ├── data.py                 # download, quality checks, hourly aggregation
│   ├── eda.py                  # exploratory statistics
│   ├── stationarity.py         # ADF, KPSS, STL, ACF/PACF, seasonal strength
│   ├── benchmarks.py           # mean, naive, seasonal naive x2, drift
│   ├── sarimax_model.py        # AIC grid search, exog selection, diagnostics
│   ├── features.py             # feature engineering + leakage self-test
│   ├── ml_model.py             # gradient boosting, recursive forecasting
│   ├── foundation_model.py     # Chronos (guarded, never faked)
│   ├── evaluation.py           # MAE/RMSE/MASE/Bias/sMAPE, Diebold-Mariano
│   ├── plotting.py             # all figures
│   └── analysis.py             # analysis summary + leakage audit writers
│
├── scripts/
│   ├── run_pipeline.py         # main entry point
│   └── run_grid_search.py      # SARIMAX grid search only (expensive)
│
└── outputs/
    ├── figures/                # 19 PNG figures at 200 dpi
    ├── forecasts/              # all_forecasts.csv + interval files
    ├── metrics/                # comparison table, grid search, importance, DM tests
    ├── models/                 # cached fitted models and derived artefacts
    └── analysis/               # analysis summary (JSON + MD), leakage audit
```

## 6. How to run

```bash
# Full pipeline, including the SARIMAX grid search on first run
python scripts/run_pipeline.py

# Reuse a cached grid search (much faster)
python scripts/run_pipeline.py --skip-grid

# Run only the expensive grid search
python scripts/run_grid_search.py
```

**Caching.** The grid search appends every fitted model to
`outputs/metrics/sarimax_grid_search.csv` as it goes and skips anything already
present, so an interrupted search resumes exactly where it stopped. Final
SARIMAX fits and their derived artefacts (forecast intervals, residuals,
diagnostics) are also cached under `outputs/models/`.

**Memory.** A fitted state-space SARIMAX result occupies roughly 1.3 GB once its
smoother output is materialised. The pipeline processes the two SARIMAX variants
one at a time and keeps only their derived artefacts, which holds peak usage
within about 3 GB. Delete `outputs/models/*.pkl` to force a refit.

### Reproducing figures and model results

All figures are regenerated by any pipeline run and written to
`outputs/figures/`. To regenerate only the figures without refitting, run with
`--skip-grid` after the model caches exist; the plotting functions in
`src/plotting.py` can also be called directly against the cached CSVs in
`outputs/forecasts/` and `outputs/metrics/`.

Model results are reproduced by the same command. Determinism comes from
`config.RANDOM_STATE = 0`, the chronological (never random) split, and the fixed
`TimeSeriesSplit` folds.

## 7. Models

| Model | Description |
|---|---|
| `mean` | Flat forecast at the historical mean. |
| `naive` | Flat forecast at the last observed value. |
| `seasonal_naive_daily` | Repeats the value from the same hour 24 h earlier. |
| `seasonal_naive_weekly` | Repeats the value from the same hour 168 h earlier. |
| `drift` | Random walk with drift equal to the average historical change. |
| `sarima` | SARIMA selected by AIC grid search, target-only. **Operational.** |
| `sarimax_exog_conditional` | Same order plus selected weather regressors. **Conditional** — assumes future weather is known. |
| `ml_operational` | Gradient boosting, recursive 24-step forecasting, exogenous inputs at lags ≥ 24 h only. **Operational.** |
| `ml_conditional` | Same model with contemporaneous weather/sensor values. **Conditional.** |
| `foundation_chronos` | Amazon Chronos-Bolt, zero-shot. Appears only when actually executed. |

### Experiment design

- **Split**: strictly chronological. Training 2016-01-11 17:00 → 2016-05-13
  17:00 (2,953 hours); test = final 14 days (336 hours).
- **Protocol**: rolling origin — 14 origins spaced 24 hours apart, each
  producing a 24-step-ahead forecast. Metrics are pooled over all 336 predicted
  points, satisfying both the 24-hour-horizon and 14-day-test-period
  requirements with a single comparable sample.
- SARIMAX advances between origins with `append(..., refit=False)`: the state is
  filtered through newly observed data, but coefficients stay fixed at their
  training-data estimates.

## 8. Evaluation metrics

| Metric | Definition | Why included |
|---|---|---|
| **MAE** | Mean absolute error | Interpretable in Wh. |
| **RMSE** | Root mean squared error | Penalises the large evening spikes more heavily. |
| **MASE** | MAE ÷ in-sample MAE of the seasonal naive | Scale-free; **primary metric**. MASE < 1 beats the seasonal naive. |
| **Bias** | Mean signed error | Detects systematic over/under-forecasting that MAE and RMSE hide. |
| **sMAPE** | Symmetric percentage error | Secondary relative metric. |

MAPE is deliberately excluded as a headline metric: the target is strictly
positive so it is computable, but it is dominated by low night-time values and
penalises over-forecasting asymmetrically.

Accuracy differences are additionally tested with the **Diebold-Mariano** test
(Harvey-Leybourne-Newbold small-sample correction, `h−1` autocovariance lags for
overlapping multi-step errors), so improvements are assessed against sampling
noise rather than read off raw metric gaps.

## 9. Foundation-model requirements

The foundation model is Amazon **Chronos-Bolt** (`amazon/chronos-bolt-base`).

- **Package**: `pip install chronos-forecasting` (pulls in `torch` and `transformers`).
- **Weights**: downloaded from the Hugging Face Hub on first use (~200 MB).
  Network access to `huggingface.co` is required **for that first download
  only**; afterwards the model is cached and runs fully offline
  (`HF_HUB_OFFLINE=1`).
- **API key**: none. This is why Chronos is preferred over TimeGPT, which
  requires a hosted API, cannot run offline, and would mean sending household
  sensor data to a third party.
- **Hardware**: CPU is sufficient; 14 origins × 24 steps takes seconds.

`src/foundation_model.py` guarantees that a benchmark is **never** relabelled as
a foundation-model result. If Chronos cannot run, `run_foundation_model` returns
`status='not_executed'`, no forecast is produced, and no row enters the
comparison table.

## 10. Known limitations

1. **Single test window.** All models are compared on one 14-day period. Small
   metric differences should not be over-interpreted — the Diebold-Mariano tests
   in `outputs/metrics/diebold_mariano_tests.csv` address this directly.
2. **Short record.** ~4.5 months provides roughly 17 complete weeks, which is
   thin for estimating weekly structure and rules out any annual seasonality.
3. **Seasonal period restricted to s=24.** A weekly SARIMAX term (s=168) would
   add 168 seasonal lags to the state vector, which is computationally
   prohibitive here. Weekly structure is instead captured by the weekly seasonal
   naive benchmark and by `lag_168` / `roll_*_168` in the feature model.
4. **Staged grid search.** The complete required `(p, d, q)` space is searched
   exhaustively, but the seasonal grid is crossed only against the best
   non-seasonal candidates, and a per-fit wall-clock cap is applied. Both
   decisions are documented in `src/config.py`; capped models are recorded with
   status `timeout` rather than dropped.
5. **Conditional variants are not operationally achievable.** They assume exact
   future weather, whose own forecast error would propagate in practice.
6. **Single household.** Results describe one dwelling and should not be assumed
   to transfer.

## 11. Reproducibility notes

- `config.RANDOM_STATE = 0` seeds the gradient booster; the split and CV folds
  are deterministic by construction.
- The raw file is cached and its MD5 printed on every run.
- Package versions and platform are recorded in
  `outputs/analysis/model_analysis_summary.json` under `environment`.
- Exact numerical reproduction of SARIMAX AIC values can vary slightly across
  BLAS/LAPACK builds and `statsmodels` versions, since the optimiser is
  iterative; model *selection* has been stable in testing.
- Chronos reproducibility is tied to a specific weight revision on the Hugging
  Face Hub. Pin `CHRONOS_MODEL_ID` if exact numbers must be reproduced.

## 12. Results

Numerical results are **not** reproduced in this README. They live in
machine-readable form, generated only by executed runs:

- `outputs/metrics/model_comparison.csv` — ranked comparison of all models.
- `outputs/analysis/model_analysis_summary.json` / `.md` — parameters, test
  statistics, diagnostics, rankings, runtimes, and analytical questions.
- `outputs/analysis/leakage_audit.md` — component-by-component leakage audit.

## 13. References

- Candanedo, L.M., Feldheim, V. & Deramaix, D. (2017). *Data driven prediction
  models of energy use of appliances in a low-energy house.* Energy and
  Buildings, 140, 81–97.
- Hyndman, R.J. & Athanasopoulos, G. (2021). *Forecasting: Principles and
  Practice* (3rd ed.).
- Diebold, F.X. & Mariano, R.S. (1995). *Comparing predictive accuracy.*
- Harvey, D., Leybourne, S. & Newbold, P. (1997). *Testing the equality of
  prediction mean squared errors.*
- Ansari, A.F. et al. (2024). *Chronos: Learning the Language of Time Series.*
