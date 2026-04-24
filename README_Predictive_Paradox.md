# Predictive Paradox — Electricity Demand Forecasting

A reproducible machine learning pipeline for **next-hour electricity demand forecasting** on the national grid using only **classical tabular ML models**.

> **Task goal:** predict `demand_mw` at time **t+1** using information available up to time **t**.  
> **Primary metric:** Mean Absolute Percentage Error (MAPE).  
> **Constraint:** no deep learning, no ARIMA/Prophet, and no future leakage.

---

## Project Summary

This repository contains a cleaned and debugged forecasting pipeline that:

- standardizes and cleans the hourly demand series,
- merges weather and annual macroeconomic data,
- engineers temporal, lag, rolling, and interaction features,
- trains classical regression models,
- evaluates them on a strict chronological hold-out period,
- and explains the main drivers behind grid demand with feature importance plots.

The current notebook uses **2024** as the hold-out test year. If you need a different split, change the `test_year` parameter in the chronological split section.

---

## Repository Contents

- `predictive_paradox_pipeline_FIXED.ipynb` — end-to-end notebook with preprocessing, modeling, evaluation, and interpretation
- `README.md` — project overview and documentation
- `assets/feature_importance_xgb.png` — recommended export of the feature importance plot for GitHub rendering

---

## Data Sources

The pipeline expects three inputs:

1. **`PGCB_date_power_demand.xlsx`**  
   Hourly demand and generation series.  
   Used as the core time series source and target construction base.

2. **`weather_data.xlsx`**  
   Hourly weather observations such as temperature, humidity, precipitation, sunshine, cloud cover, and wind direction.

3. **`economic_full_1.csv`**  
   Annual World Bank macroeconomic indicators used as slow-moving contextual features.

---

## Modeling Objective

The supervised target is defined as:

\[
y_t = demand\_mw_{t+1}
\]

while features at time \(t\) may only use information available at or before \(t\).

Because the model must remain tabular and non-sequential, the notebook explicitly converts the time series into a supervised learning dataset using lagged and rolling features.

---

## Data Cleaning Strategy

### 1) Time axis repair and duplicate handling
The raw demand file may contain irregular timestamps, duplicates, or gaps. The notebook resolves this by:

- parsing and sorting timestamps,
- rounding timestamps to hourly resolution,
- deduplicating by keeping the row with the highest demand for the same hour,
- reindexing to a strict hourly timeline.

This keeps the series internally consistent and avoids accidental duplication of the same hour.

### 2) Missing data strategy
Missing values are handled differently depending on their origin:

- **Short numeric gaps** in demand, generation, and weather are interpolated when the gap is small enough to preserve local continuity.
- **Weather** columns are linearly interpolated for short gaps.
- **Annual macroeconomic indicators** are forward-filled after pivoting to one row per year.
- **Source-data NaNs** in engineered features are filled with forward-fill, then median fallback, rather than dropping rows.

### Why this approach?
A time series forecast should preserve as much historical structure as possible. Dropping all rows with missing values would remove large parts of the training history and create a biased dataset, especially when some columns only start appearing later in time. Forward fill and interpolation are more appropriate for slowly varying contextual variables, while median fallback is a safe backstop for remaining sparse gaps.

### 3) Outlier strategy
The raw demand series contains severe spikes and extreme values. These are treated with a **rolling IQR fence**:

- a centered **168-hour** rolling window is used to estimate local quartiles,
- values outside the fence are flagged as outliers,
- flagged values are replaced with a **centered 24-hour rolling median**,
- a binary outlier flag is retained for transparency.

### Why this approach?
Electricity demand has strong daily and weekly seasonality, so a static global threshold would be too crude. A rolling IQR rule adapts to local context and suppresses undocumented spikes without flattening the underlying signal. Replacing spikes with a local median preserves trend and seasonality better than hard deletion.

---

## Feature Engineering

The model uses four feature families.

### 1) Calendar and cyclical time features
These capture regular demand cycles:

- `hour`
- `day_of_week`
- `day_of_month`
- `month`
- `quarter`
- `day_of_year`
- `week_of_year`
- `is_weekend`
- `is_month_start`
- `is_month_end`

Cyclical encodings are also created:

- `hour_sin`, `hour_cos`
- `month_sin`, `month_cos`
- `day_of_week_sin`, `day_of_week_cos`

### Why these features?
Electricity demand is strongly periodic. Morning, evening, weekday/weekend, and seasonal patterns are critical signals. Cyclical encoding avoids artificial discontinuities, such as treating hour `23` and hour `0` as far apart.

---

### 2) Lag features
The notebook creates lagged demand features at:

- `1h`, `2h`, `3h`, `6h`, `12h`, `24h`, `48h`, `168h`, `336h`

### Why these features?
These allow the model to infer:

- immediate persistence,
- daily repetition,
- weekly repetition,
- medium-term trend structure.

The longer lags are especially useful for capturing recurring load profiles from the same hour of previous days or weeks.

---

### 3) Rolling features
Backward-looking rolling statistics are engineered over:

- `3h`
- `6h`
- `12h`
- `24h`
- `48h`
- `168h`

including:

- mean
- standard deviation
- minimum
- maximum

Additional derived features include:

- `demand_vs_24h_avg`
- `demand_hourly_diff`
- `demand_24h_vs_168h`

### Why these features?
Rolling statistics summarize local momentum, volatility, and short-term demand regime changes. They help the model distinguish stable demand periods from rapidly changing ones.

---

### 4) Weather and interaction features
The notebook adds:

- `heat_index`
- `is_hot`
- `is_cool`
- `temp_x_hour`

### Why these features?
Electricity demand is highly weather-sensitive, especially through cooling and heating behavior. Interaction terms help the model learn that the same temperature can have different effects depending on the hour of day.

---

### 5) Macroeconomic features
Selected annual indicators include:

- GDP
- GDP growth
- GDP per capita
- population
- urban population
- electricity access
- electricity use per capita
- manufacturing share of GDP
- population density
- fossil fuel share

These are merged with a **1-year lag** to prevent look-ahead bias.

### Why these features?
These variables provide slow-moving structural context for national demand growth and electrification trends. The one-year lag keeps the pipeline temporally valid.

---

## Train / Validation Strategy

The dataset is split **chronologically** rather than randomly.

- training data: all years before the hold-out year
- test data: the full hold-out year
- no shuffling
- no leakage from future timestamps into past features

This is the correct validation design for forecasting problems and gives a realistic estimate of operational performance.

---

## Models Used

The notebook benchmarks two classical regressors:

- **XGBoost Regressor** — primary model
- **Random Forest Regressor** — baseline comparison

XGBoost is used as the main model because it handles nonlinear tabular relationships well and tends to perform strongly on engineered time-series features.

---

## Results

On the current notebook setup, the strict hold-out evaluation reports:

- **XGBoost MAPE:** **1.91%**
- **Random Forest MAPE:** **2.67%**

Additional diagnostics:

- **XGBoost MAE:** 223.2 MW
- **XGBoost RMSE:** 328.0 MW
- Mean residual: 108.02 MW
- Residual standard deviation: 309.73 MW
- Share of predictions within ±500 MW: 90.6%

These results are taken from the notebook’s final evaluation and residual analysis. fileciteturn0file0

---

## Feature Importance Interpretation

The XGBoost feature importance plot shows that the strongest drivers are:

1. current demand level (`demand_mw`)
2. current generation (`generation_mw`)
3. very recent lagged demand (`demand_mw_lag_1h`, `demand_mw_lag_2h`)
4. cyclical time features (`hour`, `hour_sin`, `hour_cos`)
5. weather context (`sunshine_s`)
6. operational / structural context (`load_shedding`, `elec_access_pct`)

This indicates that next-hour demand is driven primarily by:
- persistence in recent demand,
- intraday seasonality,
- weather conditions,
- and broader grid/context variables.

The notebook prints the top features directly after fitting the model. fileciteturn0file0

---

## Recommended Figures for the README

Add these exports to make the repository presentation stronger:

- `assets/feature_importance_xgb.png`
- `assets/prediction_vs_actual.png`
- `assets/residual_analysis.png`

These visuals make the model easier to understand and help reviewers validate the pipeline quality quickly.

---

## How to Run

1. Install dependencies:
   ```bash
   pip install pandas numpy matplotlib scikit-learn xgboost openpyxl
   ```

2. Place the raw data files in the project root:
   - `PGCB_date_power_demand.xlsx`
   - `weather_data.xlsx`
   - `economic_full_1.csv`

3. Open and run:
   - `predictive_paradox_pipeline_FIXED.ipynb`

4. To change the hold-out year, update:
   ```python
   test_year = 2024
   ```

---

## Notes for Reproducibility

- Keep all feature engineering strictly behind the chronological split.
- Do not use random train/test splits for forecasting.
- Save plots explicitly if you want them visible in GitHub.
- Document any assumptions about missing weather or economic coverage.
- Preserve the outlier flags, even if they are not used directly by the final model, because they help with auditability.

---

## Suggested Improvements for the Repository

To make the GitHub repository stronger, consider adding:

- a short project architecture diagram,
- a table of model results,
- saved figures in an `assets/` folder,
- a `requirements.txt`,
- a `data/` folder note explaining that raw datasets are not committed,
- a short “methodology” section linking the cleaning and feature design to the task constraints,
- and a compact “limitations and next steps” section.

---

## Limitations

- Economic indicators are annual and therefore coarse relative to hourly demand.
- Feature importance from tree models reflects split utility, not causal effect.
- Outlier smoothing improves robustness, but extreme real-world events may still be difficult to separate from genuine demand shifts.

---

## Acknowledgment

This repository was built for the **Predictive Paradox** recruitment task focused on short-term electricity demand forecasting under classical machine learning constraints.
