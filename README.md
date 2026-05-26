# FX Cash Demand Forecasting
## Branch-Level Daily Prediction - USD, GBP, EUR

![Python](https://img.shields.io/badge/Python-3.9+-blue)
![scikit-learn](https://img.shields.io/badge/scikit--learn-1.0+-orange)
![License](https://img.shields.io/badge/License-MIT-green)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

---

## Overview

This project builds a **multi-currency foreign exchange (FX) cash demand
forecasting system** that predicts daily net FX cash flow at the branch level
for a Nigerian bank, covering USD, GBP, and EUR.

The system helps operations teams answer a precise daily question:
*How much foreign currency does each branch need delivered  or evacuated tomorrow?*

Holding too little causes customer service failures and reputational damage.
Holding too much ties up capital in non-interest-bearing inventory. Accurate
daily forecasts at the branch level allow cash managers to schedule deliveries
and evacuations precisely, reducing operational waste and improving service levels.

---

## Business Problem

Foreign currency cash management in Nigerian banking is operationally complex:

- Multiple currencies (USD, GBP, EUR) each have distinct demand patterns
- Branch demand varies significantly by size, region, and customer base
- Demand spikes sharply around salary week, month-end, quarter-end, and
  Nigerian public holidays
- The NGN exchange rate trajectory adds a volume growth dimension over time
- Manual forecasting based on intuition and spreadsheet averages cannot
  capture the full interaction of these factors

This project replaces manual estimation with a data-driven forecasting pipeline
that learns branch-specific patterns and calendar effects directly from
historical transaction data.

---

## Target Variable

**`net_value = dep_value − witd_value`**
(daily deposits minus withdrawals per branch in foreign currency units)

| Net Value Sign | Business Meaning |
|---|---|
| **Negative** | Net cash demand - branch needs foreign currency delivered |
| **Positive** | Net cash surplus - excess foreign currency to be evacuated |

---

## Dataset

Synthetic dataset programmatically generated to mirror the statistical
properties of Nigerian bank FX cash transaction records.

| Property | Detail |
|---|---|
| Currencies | USD, GBP, EUR |
| USD branches | 253 |
| GBP branches | 153 |
| EUR branches | 153 |
| Date range | January 2022 – March 2026 |
| Frequency | Daily (working days only) |
| Total rows | 598,130 |
| Nigerian holidays excluded | ✅ |
| Weekends excluded | ✅ |

**Amount denominations:**
Transaction values are rounded to standard physical note denominations:
USD → multiples of $100 | GBP → multiples of £50 | EUR → multiples of €50

The dataset is available in the `/data` folder.

---

## Methodology

### 1. Feature Engineering

**Lag and Rolling Features**
Historical net_value at specific lookback windows per branch per currency:

| Feature | Captures |
|---|---|
| lag_1, lag_2, lag_3 | Short-term momentum |
| lag_7, lag_14 | Weekly seasonality |
| lag_28 | Monthly cycle |
| roll_mean_7, roll_mean_14, roll_mean_28 | Smoothed baseline trend |

**Calendar Features**
- `day_of_week` - systematic weekday effects (Monday quieter, Wednesday peak)
- `is_month_end` / `is_month_start` - boundary effects
- `is_quarter_end` - corporate FX settlement spike
- `is_salary_week` - last 10 days before month-end (Nigerian salary cycle)
- `days_to_month_end` - continuous proximity to month boundary
- `exch_rate_return_1d` - daily NGN rate movement

**Holiday Features (Nigerian Calendar)**
- `is_holiday` / `is_pre_holiday` / `is_post_holiday`
- `days_to_next_holiday` / `days_since_last_holiday`
- Covers: fixed holidays (Independence Day, Christmas), Easter,
  Eid al-Fitr, Eid al-Adha

---

### 2. Leakage Detection and Removal

Same-day raw values (`witd_value`, `dep_value`, `witd_vol`, `dep_vol`)
are excluded from all models. These are unavailable at prediction time
and their inclusion would constitute data leakage, producing misleadingly
strong validation metrics.

The leakage-safe model uses only:
- net_value lag and rolling features (prior days only)
- Calendar and holiday features (known in advance)
- Prior-day exchange rate return

---

### 3. Model Comparison - USD

Five model architectures are evaluated and compared:

| Model | Rationale |
|---|---|
| Leakage-safe Random Forest | Interpretable baseline |
| Magnitude-weighted RF | Prioritises accuracy on high-demand spike days |
| Magnitude-weighted RF + Bias Correction | Corrects systematic under/over-prediction |
| **Huber Gradient Boosting** | Robust to outlier spike days |
| Two-stage spike-aware model | Specialist regressor for extreme demand days |
| Hybrid blend (RF + Huber) | Combines global and spike-focused predictions |

**Winner selected dynamically by minimum WAPE on the validation set.**

---

### 4. Per-Branch Modelling Strategy

Beyond the global model, branch-specific models are trained where
sufficient data exists:

| Training Records | Strategy |
|---|---|
| ≥ MIN_HYBRID | Per-branch hybrid (Magnitude-weighted RF + Huber GBR) |
| MIN_RF_ONLY to MIN_HYBRID−1 | Per-branch magnitude-weighted RF |
| < MIN_RF_ONLY | Fallback to global best model |

Thin-history branches fall back gracefully rather than producing
unreliable local predictions.

---

### 5. GBP and EUR Replication

The USD pipeline is replicated for GBP and EUR using the same
baseline-first architecture. Lower volumes in GBP and EUR naturally
produce higher WAPE — consistent with the signal-to-noise characteristics
of smaller transaction flows.

---

### 6. 7-Day Forward Forecast

The approved models generate a 7-day forward prediction
(April 7–13, 2026) for all branches and currencies, outputting
predicted net_value per branch per day - the operational output
that drives cash delivery scheduling.

---

## Key Results

| Currency | Model Selected | WAPE | MAE | R² |
|---|---|---|---|---|
| **USD** | Robust Huber GBR | **0.3238** | 901.51 | **0.8363** |
| **EUR** | Huber GBR | **0.4083** | 550.08 | **0.7729** |
| **GBP** | Huber GBR | **0.4625** | 545.94 | **0.7415** |

**Primary metric:** WAPE (Weighted Absolute Percentage Error) - chosen because
it accounts for the scale of actual values, making it meaningful across branches
with very different transaction volumes.

USD achieves the strongest performance due to its larger branch network
providing richer training signal. Huber GBR wins across all three currencies,
confirming its robustness to the spike days that drive outsized errors in
standard models.

**Worst-error days** consistently fall on month-end, quarter-end, and
pre-holiday dates - structurally the hardest days to forecast in any
FX cash demand system.

---

## Business Insights

### 1. Calendar Effects Drive the Hardest Forecast Days
Month-end (especially December 31 and March 31) produces the largest
prediction errors. These are days where customer behaviour is driven by
external forces like salary obligations, regulatory filings, holiday travel
that amplify demand beyond what historical averages can anticipate.


### 2. The Salary Week Effect Is Real and Learnable
The `is_salary_week` feature (last 10 days before month-end) is a
consistently strong predictor across all currencies. Nigerian salary payment
cycles create a systematic and repeatable demand pattern that the model
learns reliably. People might decide to save in foreign currency after getting
their salary.

### 3. Huber Loss Outperforms Standard MSE for FX Cash
Standard Random Forest with equal sample weights underperforms on
extreme demand days because it optimises for average error across all
observations, most of which are ordinary days. The Huber loss function
provides robustness to outliers, producing better overall WAPE without
sacrificing ordinary-day accuracy.

### 4. Branch Heterogeneity Matters
High-volume branches in commercial centres (South West, Head Office)
have consistently lower prediction errors than low-volume branches
in less active regions. A single global model serves well as a fallback.

### 5. Exchange Rate Movements Have Modest Short-Term Impact
The `exch_rate_return_1d` feature has measurable but limited importance
relative to lag and calendar features. Day-to-day rate fluctuations do
not dramatically change physical FX cash demand — customers react to
sustained rate trends over weeks, not daily movements.

---

## Repository Structure

```
fx-cash-demand-forecasting/
│
├── README.md
├── requirements.txt
├── .gitignore
│
├── data/
│   ├── README.md
│   └── fx_cash_demand_data.csv
│
└── notebooks/
├── README.md
└── fx_cash_demand_forecasting.ipynb

```
---

## Getting Started

```bash
git clone https://github.com/gpfalade/fx-cash-demand-forecasting.git
cd fx-cash-demand-forecasting
pip install -r requirements.txt
```

Open `notebooks/fx_cash_demand_forecasting.ipynb` and run all cells.

---

## Tech Stack

- **Language:** Python 3.9+
- **ML Libraries:** Scikit-learn (Random Forest, Gradient Boosting)
- **Data Processing:** Pandas, NumPy
- **Visualisation:** Matplotlib, Seaborn
- **Calendar:** holidays (Nigerian public holiday library)
- **Environment:** Jupyter Notebook

---

## Professional Context

This project is a public demonstration of forecasting methodology applied
in Nigerian financial services. The techniques, lag feature engineering,
leakage detection, magnitude-weighted training, Huber loss regression,
and per-branch fallback logic, reflect production-grade approaches to
operational cash demand forecasting in a multi-branch banking environment.

---

## Author

**Gbemileke Falade**

Senior Data Analyst | AI/ML Practitioner | Data Consultant

Lagos, Nigeria

https://www.linkedin.com/in/gbemileke-falade
