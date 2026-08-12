# Data leakage audit

Every component of the pipeline that could move future information into the past, audited honestly. Two rows record leakage that is **present by design**: these are conditional forecasts retained as an upper bound, always labelled, and never used to support a claim about operational accuracy.

| Component | Potential leakage? | Evidence | Corrective action |
|---|---|---|---|
| 1. Train/test split | No | Strictly chronological: train ends 2016-05-13 17:00:00, test begins 2016-05-13 18:00:00. No shuffling anywhere in the codebase; `train_test_split_chronological` slices by position only. | None required. |
| 2. Scaling / transformation | No | No scaler is fitted. Gradient-boosted trees are invariant to monotone feature scaling and SARIMAX is estimated on the raw series, so there is no scaler that could be fitted on the full sample. | None required. |
| 3. Hourly aggregation | No | Resampling averages the six 10-minute readings *within* each hour. No information crosses an hour boundary. Partial hours dropped: 1. | Partial hours dropped rather than imputed. |
| 4. Missing-value imputation | No | No interpolation was applied (zero missing cells after aggregation), so no imputation-induced leakage is possible. | Time interpolation is implemented but was not triggered on this dataset. |
| 5. Target lag features | No | `shift(lag)` with lag >= 1 for every target lag. Verified programmatically: perturbing y_t leaves row t unchanged (leaking columns: []). | None required; the self-test runs on every pipeline execution. |
| 6. Rolling-window features | No | Rolling statistics are computed on `y.shift(1)` *before* `.rolling(w)`, so the window at row t spans y_{t-w}..y_{t-1}. Covered by the same perturbation self-test. | None required. |
| 7. Multi-step forecast construction | No (operational variant) | Recursive forecasting: within a 24-hour block, short lags are filled with the model's own predictions, never with observed future values. Observed test data is used only to reset the working series at each new origin, which represents data genuinely available at that origin. | The original demo pipeline scored the ML model on observed future lags; that path was removed and replaced with `recursive_forecast`. |
| 8. Future weather variables (ML, conditional variant) | YES - by design, labelled | The `conditional` regime reads contemporaneous weather and sensor values at time t, which no operator possesses at a 24-hour origin. It is retained only as an upper bound on the value of perfect covariate foresight. | Reported under an explicit `_conditional` suffix and excluded from any claim about operational accuracy. The `operational` variant uses lags >= 24 h only. |
| 9. Future sensor variables | YES - by design, labelled | Indoor sensors (T1-T9, RH_1-RH_9) and `lights` are outputs of the same household process as the target, so their contemporaneous values are arguably more informative than weather and correspondingly less available. | Same treatment as row 8; in the operational regime they enter only at lags of [24, 48] hours. |
| 10. SARIMAX exogenous regressors | YES - by design, labelled | `get_forecast(exog=X_test)` supplies test-period weather. This is a CONDITIONAL forecast that assumes future weather is known exactly. Selected regressors: ['T_out', 'RH_out', 'Windspeed']. | A target-only SARIMAX is fitted and reported alongside it; the exogenous variant carries a `_conditional` label. The pair quantifies the assumption's value. |
| 11. SARIMAX parameter estimation | No | Coefficients are estimated on training data only. Rolling-origin forecasting advances the state with `append(..., refit=False)`, which filters through newly observed data without re-estimating parameters. | None required. |
| 12. Order selection (AIC grid search) | No | All 168 candidate models were fitted on the training series only; the test period was never touched during selection. | None required. |
| 13. Exogenous variable selection | No | Correlation and VIF screens are computed on the training period only (threshold |r| >= 0.05, VIF <= 10.0). | None required. |
| 14. Hyper-parameter tuning | No | `TimeSeriesSplit` with 4 expanding-window folds and a 23-hour gap, run on the training period only. No random K-fold anywhere. Grid limited to 4 candidates to avoid tuning against the test set. | None required. |
| 15. Feature importance | No | Permutation importance is measured on a held-out tail of the *training* period, not on the test set. | None required. |
| 16. Foundation model inputs | No | Status: not_executed. Chronos is zero-shot and receives only the context strictly preceding each origin, matching the other models' protocol. | If not executed, no forecast is produced and no row enters the comparison table -- a benchmark is never substituted for it. |

## Automated checks that run on every execution

- `src.features.leakage_self_test` perturbs a single target observation and asserts that the corresponding feature row is unchanged. A non-empty `leaking_columns` list fails the audit.
- `src.ml_model.recursive_forecast` raises if any feature row contains NaN, which would indicate that a value expected to be observed is in fact missing.
- Rolling-origin helpers slice history with `index < block[0]`, a strict inequality, so the block being forecast is never visible.

## Residual risks not eliminated

1. **Single split.** All models are compared on one 14-day window. Metric differences of a few percent should not be over-read; the Diebold-Mariano tests in the analysis summary address this directly.
2. **Design choices informed by full-sample EDA.** Lag and window lengths were chosen from domain reasoning (24 h, 168 h) rather than tuned on the test set, but the exploratory plots were produced on the whole series.
3. **Conditional variants.** Their accuracy is not achievable operationally without a weather forecast, whose own error would propagate.