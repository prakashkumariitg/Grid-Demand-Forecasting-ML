# Grid-Demand-Forecasting-ML
Short-Term-Electricity-Demand-Prediction  Classical-ML-Power-Forecasting
This repository contains a robust machine learning pipeline for forecasting hourly electricity demand. Below is a detailed summary of the methodology, feature engineering, and model insights as required for the project documentation.

---

# 📊 Electricity Demand Forecasting Pipeline

This project aims to predict the next hour's electricity demand (`demand_mw`) using historical consumption, hourly weather data, and annual macroeconomic indicators. 

## 1. Data Cleaning Rationale

### Handling Missing Data
- **PGCB Grid Data:** The raw dataset contained non-hourly readings (e.g., 17:30). These were **rounded to the nearest hour**. In cases where multiple readings existed for the same hour, the **maximum demand** was kept to ensure the model accounts for peak load requirements.
- **Reindexing & Interpolation:** A strict hourly index was created. Missing gaps in `demand_mw` shorter than 6 hours were filled using **time-based linear interpolation**, while larger gaps remained as NaNs to be handled during model training.
- **Macroeconomic Data:** Indicators like GDP and Population were **forward-filled** across hourly timestamps. Crucially, these were **lagged by 1 year** to simulate a realistic scenario where World Bank data is only available after a reporting delay.

### Anomaly Detection & Smoothing
- **Rolling IQR Fence:** Instead of a global IQR (which would misidentify valid nightly lows as outliers), a **7-day rolling window (168h)** with a **3x multiplier** was used.
- **Replacement:** Spikes or drops outside this fence were replaced with a **24-hour rolling median** to maintain the integrity of the seasonal trend while removing sensor noise.

## 2. Feature Engineering Choices

### Temporal Features (Calendar & Cyclical)
- **Time Components:** Standard features like `hour`, `day_of_week`, and `month` were extracted.
- **Cyclical Encoding:** These were transformed into `sin` and `cos` components. This ensures the model recognizes that hour 23 and hour 0 are close together, preventing "jump" errors in tree-based splits.

### Lags and Rolling Statistics
- **Specific Lags:** Used 1h, 24h, and 168h (1 week) lags. These represent the three strongest signals in power demand: immediate trend, daily seasonality, and weekly routines.
- **Rolling Aggregates:** Calculated means and standard deviations over 3h, 24h, and 168h windows. 
- **Zero-Leakage Guarantee:** All rolling features were computed using a `shift(1)` on the target, ensuring the model never sees "future" information from the hour it is trying to predict.

### External & Interaction Features
- **Heat Index:** A mathematical interaction between `temperature` and `humidity` was engineered. This is a superior predictor of AC cooling load compared to temperature alone.
- **Economic Baseline:** GDP and Population density serve to shift the "base load" of the grid year-over-year, accounting for long-term industrial growth.

## 3. Model Performance & Evaluation

The pipeline compared two primary classical ML architectures on a 2024 hold-out test set:

| Model | MAPE (%) | MAE (MW) | RMSE (MW) |
| :--- | :--- | :--- | :--- |
| **XGBoost** | **1.91%** | 223.2 | 328.0 |
| **Random Forest** | **1.89%** | 213.6 | 326.3 |

Both models achieved an impressive error rate of under 2%, demonstrating high reliability for grid management operations.

## 4. Key Drivers of Grid Power Demand

Based on the model's **Feature Importance**, the following factors are the primary drivers of demand:

1.  **Lags (1h, 24h, 168h):** The most critical predictor is "what happened an hour ago," followed by the demand at the same time yesterday and last week.
2.  **Heat Index:** During summer months, the heat index correlates strongly with peak demand spikes due to increased cooling needs.
3.  **Rolling Mean (24h):** Captures the general "energy state" of the grid over the last day.
4.  **Hour Sin/Cos:** Captures the distinct "M-shaped" daily curve of residential and industrial activity.

---

