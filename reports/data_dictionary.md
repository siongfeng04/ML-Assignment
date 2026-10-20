# Data Dictionary

Dataset: UCI Steel Industry Energy Consumption (ID 851).
Prepared file: all_datasets/steel_checked.csv
Size: 35,040 rows and 11 columns; observations every 15 minutes.

| Column | Type | Description / unit |
|---|---|---|
| date | Datetime | Observation timestamp, formatted YYYY-MM-DD HH:MM:SS. |
| Usage_kWh | Numeric | Electricity consumption in kWh. |
| Lagging_Current_Reactive.Power_kVarh | Numeric | Lagging reactive energy in kVarh. |
| Leading_Current_Reactive_Power_kVarh | Numeric | Leading reactive energy in kVarh. |
| CO2(tCO2) | Numeric | Reported CO2 emissions; column label indicates tonnes of CO2. |
| Lagging_Current_Power_Factor | Numeric | Lagging power factor (%). |
| Leading_Current_Power_Factor | Numeric | Leading power factor (%). |
| NSM | Integer | Seconds since midnight, from 0 to 85,500 in 900-second steps. |
| WeekStatus | Category | Weekday or Weekend. |
| Day_of_week | Category | Monday to Sunday. |
| Load_Type | Category | Light_Load, Medium_Load or Maximum_Load. |

## Timestamp interpretation

Based on the original row order, the team treats midnight records
following the same date's 23:45 record as the following day's midnight.
This is the team's adopted interpretation, not author-confirmed timing.

Preparation shifts 365 midnight timestamps forward by one day and
recomputes calendar fields. The prepared timeline runs from
2018-01-01 00:15:00 to 2019-01-01 00:00:00.

The raw CSV is unchanged. No measurements are changed or rows removed.
Do not apply the midnight adjustment again to steel_checked.csv.

## Forecasting use

Predict the next 15-minute Usage_kWh using information available at
the forecasting time. Use historical electrical measurements only.
Future calendar information is known in advance.

Exclude CO2(tCO2) from predictors. Use Load_Type only if its availability
before the forecast is justified. Split chronologically and fit learned
preprocessing on training data only.

Forecast accuracy does not prove actual energy or emissions savings.
