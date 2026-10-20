# Handover — updated midnight interpretation
Use all_datasets/steel_checked.csv from this updated package. Parse date with %Y-%m-%d %H:%M:%S. Do not shift these dates again.

35,040 rows and 11 columns; no missing cells or duplicate timestamps. Continuous 15-minute grid from 2018-01-01 00:15 to 2019-01-01 00:00. Original measurement order preserved. Raw CSV is unchanged.

## Timestamp decision
365 original midnight dates were advanced one day because each follows 23:45 of the recorded date. This is the team's documented assumption. Recalculated calendar fields describe the adopted interval endpoint: 365 day-name changes, 104 weekday/weekend changes, no NSM value changes. See midnight_adjustments.csv and row_lineage.csv. Load_Type is retained as supplied; do not assume it is a known production schedule. Source clarification or a sensitivity analysis can address uncertainty about author intent.

## Responsibilities
- Fahim: preparation notebook, audit, checked CSV and preparation report.
- Kapiln: target-aligned splits, historical features and benchmarks; share training boundary with Kennard.
- Kennard: training-period exploration and sustainability interpretation.
- Daniel: model training and validation-based selection.
- Feng: final testing of frozen choices and integration.

Target: next 15-minute Usage_kWh. If the target is indexed t+1, inputs may include observations through t; do not shift the target twice. Define daily seasonal lags relative to the target timestamp. Past rolling features must exclude the target observation. Calendar features for the target time are known in advance. Agree an exact feature list and identical eligible timestamps before comparing models.

Use chronological 70/15/15 splits, no shuffling; fit learned preprocessing on training data. Exclude CO2(tCO2) from predictors. Load_Type needs explicit availability justification before use. Historical electrical readings may be used only if available by forecast origin. Keep valid high-consumption records; high consumption is not automatically waste. Never use test results to tune models.

This replaces the earlier literal-timestamp checked dataset. Regenerate any dependent features if anyone used that earlier version. Share through the group's one repository; avoid overwriting others' files. Every member contributes their report section and presentation.
