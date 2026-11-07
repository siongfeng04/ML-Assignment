# Model Development Improvements, Outputs and Handover — Daniel Puah (Member 4)

Notebook updated: `all_experiments/03_model_development.ipynb`  
Note: `reports/member2_report_sections.md` has been left untouched as originally provided. This report documents the specific model development improvements added in `03_model_development.ipynb`, the exact numerical outputs and files generated, and the handover instructions for Feng (Member 5).

---

## 1. Summary of Improvements Made in `03_model_development.ipynb`

Building on the three baseline model families (**Linear Regression**, **Random Forest**, and **Gradient Boosting**) trained on the agreed 14 features (`tod_sin`, `tod_cos`, `hour`, `day_of_week`, `is_weekend`, `usage_lag_1`–`usage_lag_4`, `usage_diff_1`, and four lagged electrical readings at $t-1$), four diagnostic and methodological improvements were added without altering the downstream test interface:

1. **Collinearity & L2 Regularisation Check on the Linear Baseline (`RidgeCV` — Section 3):**
   - *Why added:* In the baseline validation results, plain OLS Linear Regression had a worse MAE (`6.571 kWh`) than the naive Persistence benchmark (`5.756 kWh`), and the feature set contains exact collinearity (`usage_diff_1 = usage_lag_1 - usage_lag_2`, plus cyclical `tod_sin`/`tod_cos` alongside one-hot `hour`).
   - *What was done:* Added a `RidgeCV` pipeline ($\alpha \in \{0.01, 0.1, 1.0, 10.0, 100.0\}$) on the identical training split and exported the standardised numeric coefficients (`outputs/models/linear_coefficients.csv`) to test whether L2 shrinkage improves linear forecast accuracy.

2. **Extended Multi-Metric, Overfitting Gap & Tail-Error Validation Comparison (Section 6):**
   - *Why added:* Gradient Boosting (`RMSE = 8.775 kWh`, `MAE = 4.131 kWh`) and Random Forest (`RMSE = 8.843 kWh`, `MAE = 4.028 kWh`) are very close on standard metrics, with Random Forest slightly ahead on MAE.
   - *What was done:* Added three critical decision metrics to `outputs/models/validation_detailed_metrics.csv`:
     - **Generalisation Gap (`val_RMSE - train_RMSE`)** to measure overfitting to training spikes.
     - **95th-Percentile Absolute Error (`P95_AE`)** to quantify tail misses during rapid demand changes.
     - **Peak-Demand RMSE (`MaxLoad_RMSE`)** evaluated specifically on `Maximum_Load` intervals.

3. **Temporal Stability Verification via 5-Fold Expanding-Window `TimeSeriesSplit` (Section 6b):**
   - *Why added:* Evaluating models on a single validation period (`Sept 16 – Nov 8, 2018`) leaves open the possibility that the model ranking is specific to autumn operating conditions.
   - *What was done:* Executed a 5-fold chronological expanding-window cross-validation (`TimeSeriesSplit(n_splits=5)`) across the training timeline (`Jan – Sept 2018`) comparing `Persistence`, `Linear Regression`, `Random Forest`, and `Gradient Boosting` (`outputs/models/timeseries_cv_folds.csv`, `outputs/models/timeseries_cv_summary.csv`, and figure `outputs/figures/md_timeseries_cv.png`).

4. **Expanded Operating-Regime (`Load_Type`) Error Diagnostics (Section 8):**
   - *Why added:* Previously, residual breakdown by `Load_Type` only reported `MAE` and `mean_error`.
   - *What was done:* Kept `outputs/models/validation_error_by_load_type.csv` in its original format for compatibility while adding `RMSE` and 95th-percentile tail error (`P95_AE`) across `Light_Load`, `Medium_Load`, and `Maximum_Load` intervals in the notebook output.

---

## 2. Outputs and Numerical Findings from the Improvements

### 2.1 Output of Improvement 1: OLS vs. `RidgeCV` & Linear Coefficients
File generated: `outputs/models/linear_coefficients.csv`

- **Validation Metrics:**
  - **OLS Linear Regression:** `MAE = 6.571 kWh`, `RMSE = 11.191 kWh`, $R^2 = 0.868$
  - **RidgeCV (best $\alpha = 100.0$):** `MAE = 6.571 kWh`, `RMSE = 11.191 kWh`, $R^2 = 0.868$
- **Top 6 Standardised Numeric Coefficients:**

| Feature | OLS Standardised Coef | Ridge ($\alpha=100$) Standardised Coef |
| :--- | :---: | :---: |
| `usage_lag_1` | +12.717 | +12.710 |
| `usage_lag_2` | +11.006 | +10.998 |
| `usage_lag_3` | +4.393 | +4.392 |
| `usage_diff_1` | +4.017 | +4.019 |
| `lead_power_factor_lag_1` | +2.698 | +2.696 |
| `lag_power_factor_lag_1` | +2.585 | +2.587 |

- **Finding:** L2 regularisation does not reduce validation RMSE because the linear model's bottleneck is **structural additivity** (inability to model non-linear interactions between shift hours and recent load momentum), not coefficient variance.

---

### 2.2 Output of Improvement 2: Extended Validation Metrics & Overfitting Gap
File generated: `outputs/models/validation_detailed_metrics.csv` (alongside `outputs/models/validation_comparison.csv`)

| Model | Kind | Val MAE (kWh) | Val RMSE (kWh) | Val $R^2$ | Train RMSE | Gen. Gap (`Val - Train`) | P95 Error (`P95_AE`) | `Maximum_Load` RMSE | RMSE vs. Persistence |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Gradient Boosting** | **ML (Selected)** | 4.131 | **8.775** | **0.919** | 7.620 | **+1.155** | 19.707 | 13.350 | **−30.9%** |
| **Random Forest** | ML | **4.028** | 8.843 | 0.917 | 7.420 | +1.424 | **19.647** | **13.264** | −30.4% |
| **Linear Regression** | ML Baseline | 6.571 | 11.191 | 0.868 | 12.794 | −1.603 | 24.328 | 14.940 | −11.9% |
| **Persistence** | Benchmark | 5.756 | 12.701 | 0.830 | — | — | 29.987 | 17.660 | 0.0% |
| **Weekly seasonal** | Benchmark | 13.032 | 25.065 | 0.337 | — | — | 59.657 | 34.040 | +97.3% |
| **Daily seasonal** | Benchmark | 15.283 | 28.906 | 0.118 | — | — | 66.498 | 43.000 | +127.6% |

- **Finding:** Both tree ensembles cut 95th-percentile tail errors by **~34%** compared to Persistence (`29.99 kWh` $\rightarrow$ `~19.7 kWh`). **Gradient Boosting** (`learning_rate=0.05, max_depth=5, n_estimators=391, subsample=0.8`) is selected as the final model because it achieves the lowest validation RMSE (`8.775 kWh`), the highest $R^2$ (`0.919`), and a **19% smaller overfitting gap (`+1.155 kWh` vs. Random Forest's `+1.424 kWh`)**.

---

### 2.3 Output of Improvement 3: 5-Fold Expanding-Window `TimeSeriesSplit`
Files generated: `outputs/models/timeseries_cv_folds.csv`, `outputs/models/timeseries_cv_summary.csv`, and figure `outputs/figures/md_timeseries_cv.png`

| Chronological Fold (Unseen Period) | Persistence RMSE | Linear Regression RMSE | Random Forest RMSE | Gradient Boosting RMSE | Winning Model |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **Fold 1** (`Feb 18–Apr 01`) | 15.430 | 13.937 | 12.822 | **12.758** | **Gradient Boosting** |
| **Fold 2** (`Apr 01–May 13`) | 14.228 | 12.551 | 10.662 | **10.538** | **Gradient Boosting** |
| **Fold 3** (`May 13–Jun 24`) | 13.929 | 12.488 | 10.432 | **10.048** | **Gradient Boosting** |
| **Fold 4** (`Jun 24–Aug 05`) | 11.855 | 10.464 | 8.416 | **8.325** | **Gradient Boosting** |
| **Fold 5** (`Aug 05–Sep 16`) | 14.170 | 12.324 | 9.798 | **9.750** | **Gradient Boosting** |
| **5-Fold Mean ± SD** | **13.922 ± 1.294** | **12.353 ± 1.239** | **10.426 ± 1.599** | **10.284 ± 1.610** | **1st in 5/5 Folds** |

- **Finding:** **Gradient Boosting achieves the lowest RMSE in all 5 out of 5 chronological folds** (`mean R² = 0.899 ± 0.021`), proving that the selected model generalises consistently across winter, spring, and summer rather than only on the autumn validation slice.

---

### 2.4 Output of Improvement 4: Expanded `Load_Type` Regime Diagnostics
File generated: `outputs/models/validation_error_by_load_type.csv` (and figure `outputs/figures/md_residuals.png`)

| Target `Load_Type` | Validation Intervals | MAE (kWh) | RMSE (kWh) | P95 Abs. Error (kWh) | Mean Bias (`Actual - Forecast`) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Light_Load** | 2,736 (53.8%) | **1.256** | 3.749 | 3.249 | −0.299 kWh |
| **Medium_Load** | 1,344 (26.4%) | **6.199** | 11.452 | 24.496 | −0.504 kWh |
| **Maximum_Load** | 1,008 (19.8%) | **9.177** | 13.350 | 28.081 | +0.007 kWh |

---

## 3. List of Files Added / Updated

### New Files Added by Daniel (Member 4)
1. `reports/member4_report_sections.md` — Dedicated Member 4 report documenting improvements, outputs, and handover.
2. `outputs/models/linear_coefficients.csv` — Standardised OLS vs. RidgeCV (`alpha=100`) numeric coefficients.
3. `outputs/models/validation_detailed_metrics.csv` — Extended validation comparison table including `train_RMSE`, `gen_gap_RMSE`, `P95_AE`, and `MaxLoad_RMSE`.
4. `outputs/models/timeseries_cv_folds.csv` — Per-fold metrics (`MAE`, `RMSE`, `R2`) across the 5 expanding chronological folds.
5. `outputs/models/timeseries_cv_summary.csv` — Mean and standard deviation of `MAE`, `RMSE`, and `R2` across the 5 chronological folds.
6. `outputs/figures/md_timeseries_cv.png` — Line plot of out-of-sample RMSE across the 5 chronological folds.

### Existing Model Development Files Re-Verified & Preserved for Compatibility
- **Notebook:** `all_experiments/03_model_development.ipynb` (executed sequentially in-place with all outputs saved).
- **Frozen Models (`train + validation`, 29,184 rows):**
  - `outputs/models/gradient_boosting.joblib`
  - `outputs/models/random_forest.joblib`
  - `outputs/models/linear_regression.joblib`
  - `outputs/models/frozen_config.json`
- **Standard Validation Tables & Figures:**
  - `outputs/models/validation_comparison.csv`, `outputs/models/validation_predictions.csv`, `outputs/models/validation_error_by_load_type.csv`, `outputs/models/permutation_importance_validation.csv`, `outputs/models/tuning_random_forest.csv`, `outputs/models/tuning_gradient_boosting.csv`
  - `outputs/figures/md_gb_validation_curves.png`, `outputs/figures/md_validation_comparison.png`, `outputs/figures/md_validation_week.png`, `outputs/figures/md_residuals.png`, `outputs/figures/md_permutation_importance.png`

---

## 4. Handover Instructions to Feng (Member 5 — Final Test Evaluation & Integration)

Everything in `outputs/models/` is frozen and ready for your final evaluation on the held-out **Test set** (`2018-11-08 00:15:00` to `2019-01-01 00:00:00`, **5,184 rows**):

1. **Load Frozen Configuration & Models:**
   - Read `outputs/models/frozen_config.json` to get the exact 14 feature names (`features`), target column (`target_usage_kwh`), selected final model (`Gradient Boosting`), and `.joblib` file paths (`outputs/models/gradient_boosting.joblib`, `random_forest.joblib`, `linear_regression.joblib`).
   - All models are already refitted on `train + validation` (`29,184` rows) using `scikit-learn 1.9.0`.
2. **Filter Strictly to the Test Split:**
   - Load `all_datasets/steel_features.csv` and select `test = data[data.split == 'test']` (5,184 rows).
   - **Do not retune any hyperparameters or modify the feature list.**
3. **Evaluate & Compare Against Benchmarks:**
   - Generate predictions `model.predict(test[features])` for all three saved models.
   - Compute test **MAE, RMSE, and $R^2$** alongside the three benchmark columns (`bench_persistence`, `bench_daily_seasonal`, `bench_weekly_seasonal`) on the exact same 5,184 test rows.
4. **Recommended Diagnostics for Final Integration:**
   - Compare **Validation RMSE vs. Test RMSE** to check how well the frozen models generalised to November–December.
   - Group test residuals by `analysis_load_type` (`Light_Load`, `Medium_Load`, `Maximum_Load`) and hour of day to confirm whether the error patterns observed on validation hold on the test period.
