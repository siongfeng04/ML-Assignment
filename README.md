# Steel energy forecasting — Fahim's updated preparation
Updated 19 October 2026 to match the team's next-day midnight interpretation.

## Run locally
1. Back up your existing project folder before replacing your own preparation files.
2. Copy all_datasets, all_experiments/01_data_preparation.ipynb, outputs/data_quality and reports/member1_report_sections.md into your project. Do not overwrite teammates' notebooks or reports.
3. Open Anaconda Prompt and activate your existing ml environment.
4. Start Jupyter in the project folder, open 01_data_preparation.ipynb, restart the kernel and run all cells, then save.
5. Confirm 35,040 rows, 365 midnight date shifts, no missing/duplicate timestamps and a range of 2018-01-01 00:15 to 2019-01-01 00:00.

Dependencies: Python, pandas, NumPy and Jupyter. Actual verification versions appear in quality_summary.json. Code cells were executed sequentially in a fresh namespace, with real outputs saved. Run in your local Jupyter environment to confirm your installation.

## Changes
Midnight dates now move one day forward based on the original source row order. Day_of_week changes in 365 rows; WeekStatus changes in 104 rows. NSM is recalculated but unchanged. All other measurements and labels are preserved. This is a documented assumption, not independent confirmation of author intent. Endpoint-based calendar labels are used consistently.

The notebook always reads the unchanged original CSV, preventing repeated date shifts on rerun. It regenerates steel_checked.csv, quality summaries, row lineage, midnight_adjustments.csv and the report contribution. No model fitting, statistical outlier removal, imputation, scaling, encoding or GitHub operations occur.

Source: Sathishkumar, V. E., Shin, C., and Cho, Y. (2021). Steel Industry Energy Consumption. UCI Machine Learning Repository. https://doi.org/10.24432/C52G8C. CC BY 4.0. Derived changes: midnight dates, calendar fields and timestamp text format.
