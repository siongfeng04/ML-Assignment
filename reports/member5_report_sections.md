# Final Test Evaluation and Integration — Feng

Notebook: [`all_experiments/04_model_testing.ipynb`](../all_experiments/04_model_testing.ipynb).
The evaluation uses the frozen models and feature definition from Member 4. No
feature selection, hyperparameter tuning, or model refitting was performed
after inspecting the test period.

## 1. Test protocol

The final evaluation uses the chronological test split only:

- **Target:** Usage_kWh for the next 15-minute interval
- **Test period:** 2018-11-08 00:15:00 to 2019-01-01 00:00:00
- **Rows:** 5,184
- **Models:** frozen Linear Regression, Random Forest, and Gradient Boosting
- **Predictors:** the agreed 14 features in `outputs/models/frozen_config.json`
- **Benchmarks:** persistence, daily seasonal, and weekly seasonal forecasts

All predictions and metrics use the same test rows. `analysis_load_type` is
used only for post-hoc error grouping; it is not used as a predictor.

## 2. Unbiased test results

Results are saved in
[`outputs/models/test_evaluation_metrics.csv`](../outputs/models/test_evaluation_metrics.csv).

| Model | MAE (kWh) | RMSE (kWh) | R² | P95 absolute error (kWh) |
|---|---:|---:|---:|---:|
| **Gradient Boosting** | 3.723 | **7.765** | **0.938** | 18.000 |
| Random Forest | **3.678** | 7.782 | 0.938 | 17.982 |
| Linear Regression | 6.519 | 10.451 | 0.888 | 22.069 |
| Persistence | 4.970 | 11.607 | 0.862 | 27.500 |
| Weekly seasonal | 12.746 | 24.337 | 0.393 | 59.178 |
| Daily seasonal | 14.540 | 26.924 | 0.257 | 62.775 |

Gradient Boosting remains the selected model because it was frozen using the
pre-agreed validation-RMSE rule. On the untouched test period it has the
lowest RMSE and improves on persistence by **33.1%**. Random Forest has a
slightly lower MAE, so the two tree ensembles should be described as closely
comparable rather than claiming a large practical difference.

## 3. Validation-to-test generalisation

The validation-to-test comparison is saved in
[`outputs/models/validation_test_comparison.csv`](../outputs/models/validation_test_comparison.csv).
The frozen models did not degrade on the test period:

- Gradient Boosting RMSE changed from **8.775** to **7.765 kWh**.
- Random Forest RMSE changed from **8.843** to **7.782 kWh**.
- Linear Regression RMSE changed from **11.191** to **10.451 kWh**.

This indicates stable performance over the held-out November–December period,
but it does not prove performance in other plants, years, production regimes,
or forecast horizons.

## 4. Error diagnostics

The notebook reports error by operating regime, hour, month, and peak demand.
The load-type table is saved in
[`outputs/models/test_error_by_load_type.csv`](../outputs/models/test_error_by_load_type.csv).

| Analysis load type | Intervals | MAE (kWh) | RMSE (kWh) | P95 absolute error (kWh) |
|---|---:|---:|---:|---:|
| Light_Load | 2,832 | 1.247 | 3.839 | 6.135 |
| Medium_Load | 1,344 | 7.210 | 11.298 | 24.903 |
| Maximum_Load | 1,008 | 6.033 | 9.923 | 21.891 |

Errors are substantially larger in medium and maximum operating regimes than
in light-load intervals. For the upper 10% of actual test usage
(actual Usage_kWh at least 69.3 kWh), RMSE is **17.123 kWh** and the mean
error is **+7.847 kWh** for `actual - forecast`, indicating under-prediction
during the highest-demand intervals. This is an important limitation if the
forecast is used for peak-load warning or operational planning.

Monthly results and visual diagnostics are saved in
[`outputs/models/test_error_by_month.csv`](../outputs/models/test_error_by_month.csv)
and [`outputs/figures/mt_test_extended_diagnostics.png`](../outputs/figures/mt_test_extended_diagnostics.png).
November RMSE is **9.140 kWh**, while December RMSE is **6.563 kWh**. These
descriptive differences should not be interpreted as causal seasonal effects.

## 5. Interpretation and limitations

The model forecasts short-term electricity usage effectively relative to the
benchmarks, especially for ordinary and light-load intervals. However:

1. Forecast accuracy is not evidence of energy savings or improved
   sustainability. A forecast only estimates demand; it does not show that
   consumption was reduced.
2. High consumption is not automatically waste. It may reflect valid
   production requirements, so sustainability claims require production
   volume, operating context, and emissions information.
3. CO2(tCO2) was excluded from predictors because it is nearly a copy of the
   target and would leak future information.
4. Target-time `Load_Type` and same-interval electrical measurements were not
   treated as available at the forecast origin.
5. The adopted midnight timestamp interpretation is a documented assumption.
   A sensitivity analysis using an independently confirmed timestamp meaning
   would strengthen future work.
6. The test period is one held-out period from one year. Longer-term,
   cross-year, and operational deployment validation remains outstanding.

## 6. Handover to Kennard

The frozen test evaluation is complete. Kennard should now take responsibility
for **training-period exploration and sustainability interpretation**, without
changing the frozen test results.

Recommended next steps:

1. Use only the training period for any new exploratory analysis or additional
   sustainability-related hypotheses.
2. Describe usage distributions, peak periods, load regimes, weekday/weekend
   patterns, and temporal changes without labelling high consumption as waste
   by default.
3. If discussing sustainability, distinguish clearly between forecast
   performance and actual energy/emissions outcomes. Do not claim savings,
   efficiency improvements, or emissions reductions without an intervention,
   suitable baseline, and supporting operational or emissions data.
4. Treat `CO2(tCO2)` as a descriptive or audit variable rather than a
   forecasting predictor because of leakage risk.
5. Preserve the chronological split and do not use test results to tune
   features, models, or conclusions.

The final evaluation artifacts can be used in the integrated report, while
Kennard’s section should provide the contextual interpretation of historical
usage and the limits of sustainability conclusions.
