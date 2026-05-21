# Data

This folder contains the synthetic FX cash demand dataset used in this project.

**File:** fx_cash_demand_data.csv

**Coverage:**
- USD: 253 branches
- GBP: 153 branches
- EUR: 153 branches
- Date range: January 2022 - March 2026
- Working days only (weekends and Nigerian public holidays excluded)
- Total rows: 598,130

**Columns:**
- trn_dt         --> transaction date
- br_code        --> branch identifier
- ac_ccy         --> currency (USD, GBP, EUR)
- witd_vol       --> withdrawal transaction count
- witd_value     --> total withdrawal value in foreign currency
- dep_vol        --> deposit transaction count
- dep_value      --> total deposit value in foreign currency
- avg_exch_rate  --> average NGN exchange rate for the day
- REGION         --> branch geopolitical region

**Note on amounts:**
Values are rounded to the nearest standard denomination:
USD → multiples of $100 | GBP → multiples of £50 | EUR → multiples of €50
This reflects the physical note denominations used in real FX cash transactions.

**Note on synthetic data:**
This dataset was programmatically generated to mirror the statistical
properties of real Nigerian bank FX cash flow data. It is not derived
from any real bank records.
