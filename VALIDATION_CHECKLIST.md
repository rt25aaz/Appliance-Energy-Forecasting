# Validation checklist

Every row was verified against an executed run. Nothing is marked implemented
unless the code ran and produced the stated evidence.

Legend: **Yes** = implemented and executed. **Partial** = implemented with a
documented limitation. **Not executed** = code exists but could not run here.

---

## Part 1 — Data retrieval, preprocessing and EDA

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| 1. Download from UCI URL | Yes | `data.download_raw` | UCI tried first; falls back to author mirror when blocked. MD5 `69ef922b5fcafcd49097cfc09e07167e`, 19,735 rows verified |
| 2. Parse `date` as datetime | Yes | `data.parse_raw` | Index dtype datetime64; range 2016-01-11 17:00 → 2016-05-27 18:00 |
| 3. Timestamp as index | Yes | `data.parse_raw` | `set_index("date")` |
| 4. Sort chronologically | Yes | `data.parse_raw` | `.sort_index()` |
| 5. Missing / duplicates / dtypes / anomalies | Yes | `data.assess_quality` | missing=0, dup timestamps=0, dup rows=0, 26 float + 2 int; 2,138 values beyond 1.5×IQR retained as genuine spikes |
| 6. Confirm sampling frequency | Yes | `data.assess_quality` | Modal step 10.0 min, **0 irregular steps**, 0 missing timestamps vs the expected regular grid |
| 7. Resample to hourly | Yes | `data.resample_hourly` | 19,735 → 3,289 hourly rows |
| 8. Justify aggregation method | Yes | `data.resample_hourly` docstring | Mean; sum = 6 × mean is a fixed linear rescaling leaving MASE/sMAPE and all rankings unchanged |
| 9. Missing values after resampling | Yes | `resample_hourly` → `info` | `missing_cells_after_resampling = 0` |
| 10. Handle missing without leakage | Yes | `data.check_interpolation_safety` | 0 values interpolated; 1 partial hour **dropped**, not imputed |
| 11. Save processed dataset | Yes | `data.build_dataset` | `data/processed/appliance_hourly.csv` |
| 12. EDA plots (7 required) | Yes | `plotting` figs 01–07 | Full series, recent, daily profile, weekly profile, histogram, boxplot-by-hour, boxplot-by-dayofweek |
| 13. Investigate trend and seasonality | Yes | `stationarity.component_strengths` | Fs = 0.3183, Ft = 0.0497 |

## Part 2 — Time-series analysis and stationarity

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| 1. Decomposition | Yes | `stationarity.decompose` (STL, robust) | Figure 08 |
| 2. ACF plot | Yes | `plotting.plot_acf_pacf` | Figure 09, lags 0–168 |
| 3. PACF plot | Yes | `plotting.plot_acf_pacf` | Figure 09 lower panel |
| 4. ADF test | Yes | `stationarity.adf_test` | stat −8.8194, p = 1.895e-14 |
| 5. Report stat / p / nobs / interpretation | Yes | `adf_test` return dict | n = 2,924, used_lag reported, critical values, explicit conclusion string |
| 6. Consider differencing if required | Yes | `assess_stationarity` | ADF rejects **and** KPSS does not (p = 0.10) → verdict "stationary → no differencing required" |
| 7. Test after differencing if appropriate | Yes (correctly skipped) | `assess_stationarity` | Only runs when the verdict is non-stationary; it was not, so `differenced_tests = null` |
| 8. Daily seasonality | Yes | `stationarity.seasonality_evidence` | ACF(24) = 0.2976, ACF(48) = 0.2301, ACF(72) = 0.2167, all ≫ band 0.0370 |
| 9. Weekly structure | Yes | same | ACF(168) = 0.3139 — the largest peak in range |
| 10. Justify s = 24 | Yes | `seasonal_period_justification` | Compares Fs at 24 vs 168; documents that s=168 needs 168 seasonal lags with only ~17 complete weeks |
| 11. KPSS + extra diagnostics | Yes | `stationarity.kpss_test`, fig 10 | KPSS stat 0.0689, p = 0.10; rolling mean/std figure |
| Do not difference reflexively | Yes | `combined_verdict` | Evidence-driven; the series was **not** differenced for analysis |

## Part 3 — Forecasting problem definition

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| Target `Appliances` | Yes | `config.TARGET` | — |
| Horizon 24 hours | Yes | `config.HORIZON` | 14 rolling origins × 24 steps |
| Test = last 14 days | Yes | `config.TEST_STEPS` | 336 hours, 2016-05-13 18:00 → 2016-05-27 17:00 |
| Chronological split, no random | Yes | `data.train_test_split_chronological` | Positional slicing only; no `shuffle` anywhere in the codebase |
| Document all design elements | Yes | `run_pipeline` → `experiment` | Written to `model_analysis_summary.json` |
| MAE, RMSE, MASE, Bias | Yes | `evaluation` | All four in `model_comparison.csv` |
| sMAPE (optional) | Yes | `evaluation.smape` | Included |
| Avoid MAPE | Yes | `experiment.mape_excluded_reason` | Documented rationale |

## Part 4 — Benchmark models

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| Mean, Naive, Daily SN, Weekly SN, Drift | Yes | `benchmarks` | All five in the comparison table |
| Correct mathematics | Yes | `seasonal_naive_forecast` | Closed form `y_{T+h−m(k+1)}`, `k = ⌊(h−1)/m⌋`; explicit rather than recursive appending |
| Same horizon and metrics | Yes | `rolling_origin_benchmarks` | All n = 336, identical origins |
| Identify strongest benchmark from results | Yes | `benchmarks.strongest_benchmark` | Reads the computed table: **`seasonal_naive_weekly`**, MASE 0.804 |

## Part 5 — SARIMA / SARIMAX

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| AIC-based parameter search | Yes | `sarimax_model.run_full_grid_search` | 168 models in `sarimax_grid_search.csv` |
| Full p ∈ [0,6], d ∈ [0,2], q ∈ [0,6] | Yes | `grid_search_stage1` | All 147 combinations fitted |
| Seasonal P, D, Q at s = 24 | Yes | `grid_search_stage2` | 21 seasonal specifications |
| Not a fixed (1,0,1)(1,1,1,24) | Yes | — | Selected **(5,0,3)(0,1,1,24)**, AIC 32,169.81 |
| Avoid invalid combinations | Yes | `_fit_one` | Constant trend auto-dropped when d>0 or D>0 |
| Handle convergence errors safely | Yes | `fit_sarimax_safe` | 151 ok / 10 not converged / 7 timeout — all recorded, none fatal |
| One failure must not stop the search | Yes | `fit_sarimax_safe` | Broad exception capture + SIGALRM wall-clock cap |
| Save grid results | Yes | — | `outputs/metrics/sarimax_grid_search.csv` (incremental, resumable) |
| Sort by AIC, select best, record params | Yes | `run_full_grid_search` | Best order + AIC/BIC in `model_analysis_summary.json` |
| Fit on training data, forecast 24 h | Yes | `fit_final_sarimax`, `rolling_origin_forecast` | Coefficients from training only |
| Computational cost handled openly | Partial | `config.py` comment block | Staged search documented with measured timings; **no part of the required (p,d,q) space was reduced**; seasonal grid limited to P,D,Q ∈ {0,1} and a 150 s per-fit cap, both disclosed |
| Exogenous investigation, not blind inclusion | Yes | `sarimax_model.select_exog` | Kept `T_out`, `RH_out`, `Windspeed`; dropped `Visibility`, `Tdewpoint` (\|r\| < 0.05); VIF screen applied |
| Distinguish operational vs conditional | Yes | two model variants | `sarima` (operational) vs `sarimax_exog_conditional`; leakage audit rows 8–10 |
| Residual time plot / ACF / hist / Q-Q | Yes | `plot_residual_diagnostics` | Figure 16, four panels |
| Ljung-Box | Yes | `residual_diagnostics` | p(24) = 0.7368 → no remaining autocorrelation |
| Residual summary statistics | Yes | same | mean 0.204, std 63.0, skew 1.77, kurtosis 8.02 |
| Point forecast + confidence intervals saved | Yes | `rolling_origin_forecast` | `sarima_target_only_intervals.csv`, `sarimax_exog_intervals.csv` |

## Part 6 — Feature engineering

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| Indoor sensor variables | Yes | `config.INDOOR_COLS` | T1–T9, RH_1–RH_9 |
| Outdoor weather variables | Yes | `config.WEATHER_COLS` | 6 variables |
| hour, dayofweek, weekend, month | Yes | `features.add_time_features` | — |
| Cyclical sin/cos for hour and dow | Yes | same | `hour_sin/cos`, `dow_sin/cos` |
| Lags 1,2,3,6,12,24,48,168 | Yes | `config.TARGET_LAGS` | All eight |
| Rolling mean and std, windows 3,6,12,24,168 | Yes | `config.ROLLING_WINDOWS` | 10 rolling features |
| No leakage in lag/rolling | Yes | `features.leakage_self_test` | Perturbation test: `leaking_columns = []` in both regimes, run every execution |
| Whole pipeline checked, not just those lines | Yes | `leakage_audit.md` | 16 audited components |

## Part 7 — Feature-based ML model

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| One strong ML model | Yes | `ml_model.build_regressor` | `HistGradientBoostingRegressor`; XGBoost/LightGBM auto-detected if installed |
| Test = final 14 days | Yes | — | n = 336 |
| Forecast 24 hours | Yes | `recursive_forecast` | Reset at each of 14 origins |
| No use of actual future target | Yes | `recursive_forecast` | Predictions fed back; NaN guard raises if any feature is unavailable |
| Recursive forecasting implemented properly | Yes | same | Working series extended one step at a time via the shared feature builder |
| Future covariates identified and discussed | Yes | two regimes | `ml_operational` (lags ≥ 24 h) vs `ml_conditional`; gap quantified |
| Time-series-aware validation | Yes | `select_hyperparameters` | `TimeSeriesSplit`, 4 folds, **23-hour gap**, training period only |
| No random CV | Yes | same | No `KFold`/`shuffle` anywhere |
| Avoid excessive tuning | Yes | `config.ML_PARAM_GRID` | 4 candidates only, never scored on test |

## Part 8 — Foundation model

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| Implement a real foundation model | Yes (code) | `foundation_model.chronos_forecast` | Real `predict_quantiles` call; rolling-origin q10/q50/q90 |
| Prefer local, no API | Yes | — | Chronos-Bolt chosen over TimeGPT, with reasons |
| **Executed in this environment** | **Not executed** | `run_foundation_model` | `chronos-forecasting` + `torch` installed successfully, but weights blocked: *"We couldn't connect to 'https://huggingface.co'"* |
| State package / key / input / output / offline | Yes | `alternative_foundation_models`, `how_to_run_locally` | Full comparison of Chronos / TimesFM / TimeGPT |
| Never label a benchmark as the foundation model | Yes | `run_foundation_model` | Returns `status='not_executed'`, **no forecast produced, no row in the comparison table** |
| Separate implementation + instructions | Yes | `how_to_run_locally` | 4-step reproduction guide |

## Part 9 — Model evaluation

| Requirement | Implemented? | File / function | Evidence |
|---|---|---|---|
| Comparable test period/horizon for all | Yes | — | Every model n = 336, identical origins |
| Same metrics throughout | Yes | `build_comparison_table` | MAE/RMSE/MASE/Bias/sMAPE |
| Final comparison table sorted | Yes | — | `outputs/metrics/model_comparison.csv`, sorted by MASE |
| `all_forecasts.csv` with actual + each model | Yes | — | 9 model columns + `actual` + origin/step |
| Preserve and improve output structure | Yes | — | Original two files retained; DM tests, per-horizon errors, importance, ablation added |

## Part 10 — Visualisations

| # | Required figure | File |
|---|---|---|
| 1 | Overall time series | `01_full_series.png` |
| 2 | EDA plots | `02`–`07` |
| 3 | ACF/PACF | `09_acf_pacf.png` |
| 4 | Stationarity diagnostics | `10_stationarity_diagnostics.png` |
| 5 | Benchmark forecasts | `11_benchmark_forecasts.png` |
| 6 | SARIMAX with intervals | `12_sarima_forecast.png` |
| 7 | ML forecast | `13_ml_forecast.png` |
| 8 | Foundation-model forecast | **Absent by design** — not executed; a figure would imply a result that does not exist |
| 9 | Combined comparison | `15_forecast_comparison.png` |
| 10 | Residual diagnostics | `16_residual_diagnostics.png` |
| 11 | Model error comparison | `17_error_comparison.png` |
| Extra | Error by lead time; importance; ablation; STL | `18`, `19`, `20`, `08` |

Figure 14 is deliberately missing so the numbering makes the omission visible.

## Parts 11–15

| Requirement | Implemented? | File | Evidence |
|---|---|---|---|
| Machine-readable analysis file | Yes | `analysis/model_analysis_summary.json` + `.md` | Parameters, AIC, rankings, test statistics, diagnostics, importance, dates, horizon, runtimes |
| No unsupported interpretation | Yes | `analysis.py` docstring | Facts + questions only |
| Analytical prompts for the six questions | Yes | `analysis.analytical_prompts` | 4–5 questions per assignment question |
| Q1–Q6 answerable from outputs | Yes | see mapping below | — |
| Code quality: functions, docstrings, config, error handling, seeds, modules | Yes | `src/` | 12 modules, ~3,400 lines, every public function documented |
| README with all 16 sections | Yes | `README.md` | No fabricated results included |
| `requirements.txt` | Yes | — | 6 core packages; heavy optional ones commented out |
| Leakage audit with table | Yes | `analysis/leakage_audit.md` | 16 components + automated checks + residual risks |
| Never fabricate | Yes | — | Foundation model marked `NOT EXECUTED`; no invented numbers anywhere |

### Where each assignment question is answered

| Question | Primary evidence files |
|---|---|
| Q1 Strongest benchmark | `model_comparison.csv`, `diebold_mariano_tests.csv`, figs 03/04/09 |
| Q2 Does SARIMAX improve? | `model_comparison.csv`, `sarimax_grid_search.csv`, `sarima_target_only_diagnostics.json`, fig 16 |
| Q3 Feature-group value | `ml_feature_ablation.csv`, `ml_feature_importance_by_group.csv`, figs 19/20 |
| Q4 Foundation model | `model_analysis_summary.json → foundation_model` (status `not_executed`) |
| Q5 Covariate availability | `leakage_audit.md` rows 8–10; conditional vs operational pairs |
| Q6 Practical choice | `model_comparison.csv`, `runtimes_seconds`, interval files, `error_by_horizon_step.csv` |
