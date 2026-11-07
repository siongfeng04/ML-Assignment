# Feature Engineering, Selection and Model Development — Kapiln

Notebooks: `all_experiments/02_feature_engineering.ipynb`, `all_experiments/03_model_development.ipynb`.
All figures below are in `outputs/figures/`; all tables are in `outputs/features/` and `outputs/models/`.

## Target alignment
Each row of `all_datasets/steel_features.csv` is one target interval: the Usage_kWh of the next 15 minutes. The forecast is issued 15 minutes earlier, at the origin. All electrical inputs are shifted back by at least one interval, so only completed readings are used. Rolling windows end at the previous interval and never include the target. Calendar inputs describe the target timestamp, which is known in advance. Assertions in the notebook check each shift against the raw data.

The weekly lag needs 672 earlier intervals, so 1–7 January serves only as history. This leaves 34,368 eligible target intervals.

## Chronological split (whole days, no shuffling)
| Split | Target intervals | Rows | Share |
|---|---|---|---|
| Train | 2018-01-08 00:15 – 2018-09-16 00:00 | 24,096 | 70.1% |
| Validation | 2018-09-16 00:15 – 2018-11-08 00:00 | 5,088 | 14.8% |
| Test | 2018-11-08 00:15 – 2019-01-01 00:00 | 5,184 | 15.1% |

Boundaries are recorded in `outputs/features/split_boundaries.json`. Kennard should use only training rows for exploratory decisions.

## Candidate features and exclusions
There were 22 candidate predictors in six groups:
- **Calendar:** time-of-day sine/cosine, hour, day of week and weekend flag.
- **Recent usage:** lags 1–4 and the latest change.
- **Seasonal lags:** same interval yesterday and last week.
- **Rolling statistics:** 1-hour, 4-hour and 24-hour windows.
- **Electrical lags:** lagged reactive power and power factors.
- **Lagged load type.**

Excluded, with reasons:
- **CO2(tCO2):** almost a scaled copy of the target (r ≈ 0.99), so it would leak the answer.
- **Same-interval electrical readings:** not yet measured at the forecast origin.
- **Load_Type at the target time:** not a fixed schedule in this data. For a given time of day and weekday/weekend, the most common label covers only about 79% of rows on average, so it cannot be known in advance. It is kept only for grouped error analysis.
- **Month:** the validation and test months lie outside the training range, so a month feature could only extrapolate.

## Feature selection
On the training period, recent usage was the most informative predictor by both mutual information and correlation (lag 1: r = 0.91). Feature groups were then added one at a time and scored by validation RMSE with a fixed probe model. Calendar and recent usage were always kept. Each later group was kept only if it reduced RMSE by at least 1%.

| Step | Probe validation RMSE (kWh) | Kept |
|---|---|---|
| Calendar only | 19.56 | yes (core) |
| + recent usage | 8.81 | yes (core) |
| + seasonal lags | 8.92 | no (worse) |
| + rolling statistics | 8.83 | no (worse) |
| + electrical lags | 8.66 | yes (−1.6%) |
| + lagged load type | 8.60 | no (−0.7%, below threshold) |

**Final feature set (14):** time-of-day sine/cosine, hour, day of week, weekend flag, usage lags 1–4, last usage change, and the four lagged electrical readings.

Same-time-yesterday and same-time-last-week lags added nothing once the latest readings were known. At a 15-minute horizon, the plant's current state matters far more than its past schedule.

## Models and tuning (validation only)
- **Linear Regression:** one-hot hour and day of week, standardised numeric inputs.
- **Random Forest:** 12-configuration grid. Selected 200 trees, max_depth 16, min_samples_leaf 5, max_features 0.5.
- **Gradient Boosting:** learning rate {0.05, 0.1} × depth {3, 5}, with the tree count chosen at the validation-RMSE minimum. Selected learning rate 0.05, depth 5, 391 trees, subsample 0.8.

## Validation results (5,088 intervals)
| Model | MAE (kWh) | RMSE (kWh) | R² | RMSE vs persistence |
|---|---|---|---|---|
| **Gradient Boosting** | 4.13 | **8.78** | 0.919 | −30.9% |
| Random Forest | **4.03** | 8.84 | 0.918 | −30.4% |
| Linear Regression | 6.57 | 11.19 | 0.868 | −11.9% |
| Persistence benchmark | 5.76 | 12.70 | 0.830 | — |
| Weekly seasonal benchmark | 13.03 | 25.07 | 0.337 | +97.3% |
| Daily seasonal benchmark | 15.28 | 28.91 | 0.118 | +127.6% |

All three models beat every benchmark. Gradient Boosting was selected on validation RMSE, the pre-agreed metric, because RMSE emphasises large misses at peak intervals. Random Forest is effectively tied with it and has a slightly lower MAE, so the two tree models should be presented as comparable.

Validation results are somewhat optimistic, because the same period guided feature selection and tuning. The untouched test period is the unbiased check.

## Error analysis and importance (validation)
**Error by load level.** The selected model's mean absolute error rises with load: 1.26 kWh in Light_Load, 6.20 kWh in Medium_Load and 9.18 kWh in Maximum_Load intervals. Mean bias is close to zero in every group.

**Error by time of day.** Errors are concentrated in working hours, roughly 08:00–21:00. They are largest at shift start-ups, shutdowns and short spikes that the latest reading cannot anticipate. Between 01:00 and 08:00, errors are below 0.5 kWh.

**Permutation importance.** The model relies overwhelmingly on the last usage reading: shuffling it raises RMSE by about 29 kWh. Next come time of day (sine/cosine), the latest change in usage and lagged leading reactive power. Importance shows association, not causation.

## Handover to Feng (evaluation)
Frozen configurations are in `outputs/models/frozen_config.json`. Each model was refitted on train + validation and saved as a `.joblib` file in `outputs/models/`. To evaluate:
1. Load `steel_features.csv`.
2. Predict the `split == 'test'` rows using the 14 listed features.
3. Score MAE, RMSE and R² alongside the `bench_persistence`, `bench_daily_seasonal` and `bench_weekly_seasonal` columns on the same rows.

Do not retune after seeing test results. The models were trained with scikit-learn 1.9.0; load them with the same version, or rerun notebook 03.
