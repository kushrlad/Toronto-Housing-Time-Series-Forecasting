# Toronto Housing Time Series Forecasting

**Forecasting Toronto house prices and home sales with chronological validation, stationarity testing, and classical time series models.**

## Project Overview

This project documents a time series forecasting workflow for two Toronto housing market indicators: average home price and number of home sales. The notebook builds a clean monthly modeling table, evaluates stationarity, checks for outliers, compares several forecasting model families on a validation window, and produces final SARIMA forecasts on the test set.

The project emphasizes reproducible forecasting practice: chronological data splitting, consistent evaluation metrics, reusable helper functions, non-mutating diagnostics, and forecast horizons derived from the actual validation and test set lengths rather than hardcoded indexes.

## Dataset

The notebook reads data from:

| Source file used in notebook | Rows filtered | Measures selected | Date construction |
|---|---:|---|---|
| `Toronto's Dashboard - Key metrics.csv` | `period_number_in_year <= 12` | `Average Home Price (City of Toronto)` and `Number of Home Sales (City of Toronto)` | `year` + `period_number_in_year` + day `01` |

After merging the two measures on `Date`, the notebook creates differenced features and drops rows with missing differenced values.

| First rows shown in notebook | Date | House_Price | House_Sales | House_Price_Diff1 | House_Price_Diff2 | House_Sales_Diff1 | Price_Change | Sales_Change |
|---:|---|---:|---:|---:|---:|---:|---:|---:|
| 0 | 2009-03-01 | 387,793.00 | 2,398.00 | -5,126.00 | -33,629.00 | 745.00 | -33,629.00 | 745.00 |
| 1 | 2009-04-01 | 421,470.00 | 3,222.00 | 33,677.00 | 38,803.00 | 824.00 | 38,803.00 | 824.00 |
| 2 | 2009-05-01 | 432,478.00 | 3,777.00 | 11,008.00 | -22,669.00 | 555.00 | -22,669.00 | 555.00 |
| 3 | 2009-06-01 | 441,703.00 | 4,362.00 | 9,225.00 | -1,783.00 | 585.00 | -1,783.00 | 585.00 |
| 4 | 2009-07-01 | 421,110.00 | 3,880.00 | -20,593.00 | -29,818.00 | -482.00 | -29,818.00 | -482.00 |

## Methodology

### 1. Data loading and feature preparation

The notebook loads the Toronto dashboard CSV, keeps valid monthly periods, constructs a monthly `Date`, and separates the two target series into `House_Price` and `House_Sales`. It then merges the targets into one chronological dataset.

Differencing is applied because raw time series often contain trends or seasonal structure that can violate stationarity assumptions used by classical time series models. The notebook tests the raw and differenced series with the Augmented Dickey-Fuller test and uses differenced columns for stationarity diagnostics.

### 2. Chronological train / validation / test split

The notebook uses chronological splitting instead of random splitting because time series forecasting must simulate the real forecasting problem: models should train on past observations and be evaluated on future periods. Random splitting would leak future temporal information into training.

| Split | Rule used in notebook | Rows |
|---|---|---:|
| Train | Dates before `2017-01-01` within pre-test data | 94 |
| Validation | Dates from `2017-01-01` through pre-test data | 36 |
| Test | Dates from `2020-01-01` onward | 60 |

### 3. Stationarity testing

The Augmented Dickey-Fuller test is used to evaluate whether each series is stationary at the 5% threshold. The raw price and sales series were not stationary. The second difference of house price and the first difference of sales were stationary according to the notebook output.

| Series | ADF Statistic | p-value | Used Lag | Observations | Stationary at 5% |
|---|---:|---:|---:|---:|---|
| House_Price | 3.42 | 1.00 | 12 | 81 | False |
| House_Sales | -2.18 | 0.21 | 12 | 81 | False |
| House_Price_Diff1 | -1.81 | 0.38 | 12 | 81 | False |
| House_Price_Diff2 | -10.02 | 0.00 | 11 | 82 | True |
| House_Sales_Diff1 | -3.69 | 0.00 | 11 | 82 | True |

### 4. Outlier detection

Outliers are detected using moving-average residuals with a 12-period centered rolling window and a threshold of 3 residual standard deviations. The helper function copies the input data before adding diagnostic columns, so the original training DataFrame is not mutated.

| Diagnostic | Count |
|---|---:|
| Price outliers | 0 |
| Sales outliers | 0 |
| Rows removed | 0 |
| Clean training rows | 94 |

### 5. Validation model comparison

The notebook compares four classical forecasting model families on the same validation horizon for both level targets: `House_Price` and `House_Sales`.

- **Exponential Smoothing (ES)** was included to model level, trend, and additive seasonality directly.
- **AutoRegression (AR)** was included to test whether recent lagged values alone could explain future values.
- **ARIMA** was included to combine autoregression, differencing, and moving-average error correction.
- **SARIMA** was included to extend ARIMA with explicit 12-period seasonality for monthly data.

All validation forecasts use `len(Validate)` as the forecast horizon, ensuring the same 36-period comparison window across models.

| Model | Target | MAE | RMSE | MAPE (%) |
|---|---|---:|---:|---:|
| ES | House_Price | 27,716.75 | 40,770.28 | 3.24 |
| AR | House_Price | 64,603.94 | 76,237.26 | 7.57 |
| ARIMA | House_Price | 127,920.15 | 141,390.49 | 14.74 |
| SARIMA | House_Price | 74,634.42 | 81,875.77 | 8.78 |
| ES | House_Sales | 891.86 | 975.79 | 35.30 |
| AR | House_Sales | 1,452.91 | 1,566.50 | 57.31 |
| ARIMA | House_Sales | 1,342.72 | 1,513.10 | 45.51 |
| SARIMA | House_Sales | 1,402.41 | 1,515.43 | 54.65 |

Sorted by target and RMSE, ES produced the lowest validation RMSE for both house price and house sales in the notebook output.

| Rank within target | Model | Target | MAE | RMSE | MAPE (%) |
|---:|---|---|---:|---:|---:|
| 1 | ES | House_Price | 27,716.75 | 40,770.28 | 3.24 |
| 2 | AR | House_Price | 64,603.94 | 76,237.26 | 7.57 |
| 3 | SARIMA | House_Price | 74,634.42 | 81,875.77 | 8.78 |
| 4 | ARIMA | House_Price | 127,920.15 | 141,390.49 | 14.74 |
| 1 | ES | House_Sales | 891.86 | 975.79 | 35.30 |
| 2 | ARIMA | House_Sales | 1,342.72 | 1,513.10 | 45.51 |
| 3 | SARIMA | House_Sales | 1,402.41 | 1,515.43 | 54.65 |
| 4 | AR | House_Sales | 1,452.91 | 1,566.50 | 57.31 |

### 6. Final SARIMA forecast on the test set

The final notebook stage fits SARIMA models on `Train_clean + Validate` and forecasts exactly `len(Test)` periods. The final model uses SARIMA order `(1, 1, 1)` and seasonal order `(1, 1, 1, 12)` for each target.

The first five test forecasts shown in the notebook are:

| Index | Date | House_Price | Forecast_House_Price | House_Sales | Forecast_House_Sales |
|---:|---|---:|---:|---:|---:|
| 130 | 2020-01-01 | 884,385.00 | 897,181.76 | 1,603.00 | 1,510.83 |
| 131 | 2020-02-01 | 989,218.00 | 952,191.85 | 2,477.00 | 2,244.13 |
| 132 | 2020-03-01 | 987,787.00 | 991,119.37 | 2,771.00 | 3,124.09 |
| 133 | 2020-04-01 | 881,424.00 | 1,006,220.62 | 1,036.00 | 3,551.91 |
| 134 | 2020-05-01 | 955,273.00 | 1,018,427.97 | 1,491.00 | 3,815.82 |

Final test performance is reported with MAE, RMSE, and MAPE as a percentage.

| Model | Target | MAE | RMSE | MAPE (%) |
|---|---|---:|---:|---:|
| Final SARIMA | House_Price | 74,553.21 | 90,245.98 | 7.00 |
| Final SARIMA | House_Sales | 839.02 | 957.54 | 39.20 |


## Dependencies

| Package | Used for |
|---|---|
| pandas | CSV loading, data filtering, merging, date construction, tabular outputs |
| numpy | Numerical arrays and metric calculations |
| matplotlib | Time series and forecast visualization |
| seaborn | Validation RMSE bar charts |
| scikit-learn | MAE and MSE metric functions |
| statsmodels | ADF tests, Exponential Smoothing, AutoRegression, ARIMA, SARIMAX |
| pmdarima | Imported in the notebook |

## Key Lessons / Takeaways

- Time series forecasting should use chronological splits because future observations must not influence model training.
- Stationarity diagnostics matter: the notebook shows raw house price and sales series were not stationary, while selected differenced versions passed the 5% ADF threshold.
- Validation metrics should be interpreted by target, not across targets, because house prices and house sales are measured on different scales.
- MAPE is useful for percentage interpretation, but the sales target still showed high percentage error even when final SARIMA achieved a lower test RMSE than its validation SARIMA result.
- Consistent evaluation tables reduce modeling ambiguity and make it easier to compare model families fairly.

