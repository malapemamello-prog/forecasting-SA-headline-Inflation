# Forecasting South African Headline Inflation for 2026

**A backtested comparison of ARIMA, damped-trend exponential smoothing (Holt) and XGBoost**



---

## Overview

This project forecasts South Africa's headline consumer price inflation (year-on-year, %) for each month of 2026. Three models were trained on 60 monthly observations (January 2021 to December 2025) and compared under one leakage-safe, expanding-window, one-step-ahead backtest over 2025.

**Headline findings**

- XGBoost had the lowest MAE (0.204 pp) and MAPE (6.35%) and was kept as the primary model. Holt had the lowest RMSE (0.266), and ARIMA had the highest errors on every measure.
- XGBoost projects inflation of roughly **3.5% throughout 2026**, with a 95% interval widening from about 3.0 to 4.1% in January to 1.6 to 5.4% in December.
- January to June 2026 actuals were later published (3.5% rising to 5.0%). All three models missed the second-quarter acceleration. This was used only as an out-of-sample check, never for model selection.

---

## Data

| Item | Detail |
|---|---|
| Source | Statistics South Africa, Consumer Price Index (P0141) |
| Target variable | Headline CPI year-on-year inflation rate (%) |
| Training window | January 2021 to December 2025 (60 monthly observations) |
| Held-out data | January to June 2026 (used only for the out-of-sample check) |
| Data quality | No missing values, no duplicate timestamps, no implausible readings; no imputation or outlier treatment |
| Transformations | None (no logs, scaling or stored differencing); differencing occurs only inside ARIMA (d = 1) |

**Why the year-on-year rate rather than the index level?** It is policy-relevant (the Reserve Bank targets an annual rate), easy to interpret, published directly by Stats SA, and far more stable than the trending index.

**Descriptive statistics (training series)**

| Statistic | Value | Statistic | Value |
|---|---|---|---|
| Observations | 60 | Mean | 4.99% |
| Std. deviation | 1.51 pp | Median | 5.1% |
| Minimum | 2.7% (Mar 2025) | Maximum | 7.8% (Jul 2022) |
| Skewness | 0.14 | Excess kurtosis | −1.01 |

---

## Methods

### Exploratory analysis

- The series contains one full up-and-down inflation cycle (peak 7.8% in July 2022, trough 2.7% in March 2025).
- Additive decomposition shows most variation sits in the trend, with very small seasonality, as expected for a year-on-year rate.
- Rolling volatility is not constant (roughly 0.3 to 1.05 pp).
- An ADF test on levels (statistic −1.210, p = 0.669) cannot reject a unit root; on the first difference (statistic −5.809, p < 0.001) the series is stationary, so **d = 1**.
- ACF/PACF of the differenced series show no clean cut-off and no lag-12 spikes, so seasonal terms were dropped and orders were chosen by AIC.

### Backtesting design

- **Evaluation window:** January to December 2025 (12 origins).
- **Procedure:** expanding window; refit on all data strictly before each month, forecast one step ahead, then move the origin forward.
- **Tuning:** ARIMA orders and XGBoost hyperparameters selected using only January 2021 to December 2024 (XGBoost used its own inner expanding-window validation on the last 12 months of that period).
- **Leakage control:** all XGBoost features are built from lagged values (shift ≥ 1).

### Models

| Model | Specification | Notes |
|---|---|---|
| **ARIMA** (statistical) | ARIMA(0,1,1), chosen by AIC from a p, q grid of 0 to 3; MA(1) = 0.816 | Equivalent to simple exponential smoothing (weight about 0.18); residuals behave like white noise; forecast is flat (3.69%) |
| **XGBoost** (machine learning, primary) | Max depth 4, 100 trees, learning rate 0.1 (best of 18 grid combinations), seed 42 | Features: lags 1, 2, 3, 6, 12; 3-month rolling mean and std. dev.; calendar month. 48 usable training rows; multi-step forecasts are recursive |
| **Holt damped trend** (supplementary benchmark) | Additive-trend exponential smoothing with damping (Gardner & McKenzie, 1985) | Trend-aware benchmark evaluated under the same design |

**Prediction intervals:** ARIMA intervals come from the fitted model. For XGBoost and Holt, the 95% interval is the point forecast ± 1.96 × the backtest residual standard deviation × √horizon (a heuristic).

### Evaluation metrics

MAE (primary selection criterion), RMSE, MAPE and MASE.

---

## Results

### 2025 backtest (one-step-ahead, 12 forecasts)

| Model | MAE (pp) | RMSE (pp) | MAPE (%) | MASE |
|---|---|---|---|---|
| ARIMA(0,1,1) | 0.252 | 0.278 | 8.09 | 0.740 |
| **XGBoost (depth 4, 100 trees, lr 0.1)** | **0.204** | 0.268 | **6.35** | **0.598** |
| Holt damped trend | 0.218 | **0.266** | 6.88 | 0.639 |

- ARIMA's errors flip sign almost every month, which is typical of an MA(1) coefficient near 1.
- XGBoost and Holt both missed the March dip (2.7%) and the July jump (3.5%).
- With only 12 test points, the gaps between models are small and could be reversed by a single month's error.

### 2026 forecasts (selected months, %)

| Month | XGBoost | Lower 95% | Upper 95% | ARIMA | Holt | Actual |
|---|---|---|---|---|---|---|
| Jan | 3.52 | 2.98 | 4.06 | 3.69 | 3.61 | 3.5 |
| Feb | 3.53 | 2.77 | 4.30 | 3.69 | 3.61 | 3.0 |
| Mar | 3.59 | 2.65 | 4.53 | 3.69 | 3.62 | 3.1 |
| Apr | 3.59 | 2.50 | 4.68 | 3.69 | 3.62 | 4.0 |
| May | 3.59 | 2.38 | 4.81 | 3.69 | 3.62 | 4.5 |
| Jun | 3.57 | 2.24 | 4.90 | 3.69 | 3.63 | 5.0 |
| Jul | 3.47 | 2.03 | 4.90 | 3.69 | 3.63 | 4.3 |
| Dec | 3.47 | 1.59 | 5.36 | 3.69 | 3.64 | n/a |

All three models place 2026 inflation at about 3.5 to 3.7%, inside the new 2 to 4% target band (3% target announced November 2025). The flatness is a property of the models, not a prediction that inflation will stay constant.

### Out-of-sample check, January to June 2026

| Month | Actual | XGBoost | ARIMA | Holt | XGBoost error (actual − forecast) |
|---|---|---|---|---|---|
| Jan | 3.5 | 3.52 | 3.69 | 3.61 | −0.02 |
| Feb | 3.0 | 3.53 | 3.69 | 3.61 | −0.53 |
| Mar | 3.1 | 3.59 | 3.69 | 3.62 | −0.49 |
| Apr | 4.0 | 3.59 | 3.69 | 3.62 | +0.41 |
| May | 4.5 | 3.59 | 3.69 | 3.62 | +0.91 |
| Jun | 5.0 | 3.57 | 3.69 | 3.63 | +1.43 |
| **MAE** | | **0.632** | **0.650** | **0.645** | |

- The models were close in January and too high in February to March, then too low from April onward.
- June's 5.0% exceeded the upper end of the XGBoost 95% interval (4.90%), so the interval was too narrow.
- Six-month MAEs were almost identical (0.63 to 0.65 pp): none of the models could anticipate a shock that was not yet visible in past inflation. Commentary attributes the rise to a fuel-price shock, but the models contain no fuel, exchange-rate or oil variables, so this could not be verified.

---

## Limitations

| Limitation | Implication |
|---|---|
| **Short sample** | 60 points contain only one inflation cycle, so rare or extreme episodes are barely represented and intervals likely understate tail risk |
| **Heuristic XGBoost interval** | Based on only 12 backtest residuals, assumes √horizon scaling and is symmetric (the lower bound of 1.59% in December is mechanically low); read it as an approximate range |
| **One-step evaluation, 12-step use** | Models were ranked on one-month-ahead accuracy but used for a 12-month path; multi-step accuracy could rank them differently and recursive forecasts compound errors |
| **Tree models do not extrapolate** | XGBoost cannot predict outside the range of targets it has seen, and with 48 training rows it collapses toward recent levels |
| **Univariate information set** | Only past inflation is used; fuel, exchange rate, food and administered prices, interest rates and expectations (usual drivers of turning points) are excluded |
| **Regime change** | The target framework changed in November 2025 (3% target, 2 to 4% band); the training data does not reflect this |
| **Small evaluation window** | 12 test points cannot separate models whose MAEs differ by hundredths of a percentage point |
| **Flat forecasts** | The point forecasts encode an assumption of stability, not a prediction of it |

**Suggested improvements:** add leading indicators (fuel, exchange rate, food prices), run a direct multi-horizon backtest, use longer histories or multiple evaluation windows, and refit as new releases arrive.

---

## Contributors

| Name | Role |
|---|---|
| Mamello Malape | Author |
| Boipelo Letwaba | Co-author |

Prepared for **EDAB 6808, Assignment 2, Question 2**.

---

## References

- Chen, T. & Guestrin, C. (2016). XGBoost: A scalable tree boosting system. *KDD*, 785-794.
- Dickey, D. A. & Fuller, W. A. (1979). Distribution of the estimators for autoregressive time series with a unit root. *JASA*, 74(366), 427-431.
- Gardner, E. S. & McKenzie, E. (1985). Forecasting trends in time series. *Management Science*, 31(10), 1237-1246.
- Hyndman, R. J. & Athanasopoulos, G. (2021). *Forecasting: Principles and Practice* (3rd ed.). OTexts.
- Hyndman, R. J. & Koehler, A. B. (2006). Another look at measures of forecast accuracy. *IJF*, 22(4), 679-688.
- Perktold, J. & Seabold, S. (2010). statsmodels: Econometric and statistical modeling with Python. *Proceedings of the 9th Python in Science Conference*.
- Statistics South Africa (2020-2026). Consumer Price Index (P0141). https://www.statssa.gov.za/?page_id=1859
- Ministry of Finance and South African Reserve Bank (2025). Announcement of the new inflation target for South Africa. Media advisory, 12 November.
- Ecofin Agency (2026). South Africa's central bank faces first real test of its 3% consumer price target.
