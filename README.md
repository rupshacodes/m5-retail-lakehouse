# m5-retail-lakehouse
# Retail Demand Forecasting on the M5 Walmart Dataset

Forecasting 28 days of daily unit sales for 70 Walmart store-department series, built from ~59M raw daily sales records. The pipeline includes automated data-quality checks and leakage-safe features. A LightGBM model is evaluated against seasonal baselines with a rolling-origin backtest.

**Headline result:** LightGBM cut forecast error (WAPE) from **13.0%** to **10.2%** on the final 28-day holdout, about a 22% relative reduction. Averaged over a 3-fold backtest, it achieved **10.1% vs. 13.2%** for the baseline.

---

## Results

| Evaluation | Seasonal baseline WAPE | LightGBM WAPE | Relative reduction |
| --- | --- | --- | --- |
| Final 28-day holdout (25 Apr – 22 May 2016) | 13.0% | 10.2% | ~22% |
| 3-fold rolling-origin backtest (mean) | 13.2% | 10.1% | ~23% |

- **Baseline:** seasonal naive, predicting each day's sales as the same weekday 4 weeks earlier. This was the stronger of the two baselines tested; the other was the same week last year.
- **WAPE** (weighted absolute percentage error) = total absolute error ÷ total actual units. It is used instead of MAPE because MAPE becomes unstable when daily sales are near zero, and WAPE weights errors by volume, which matches business impact.
- **Backtest folds:** three consecutive 28-day test windows ending 27 Mar, 24 Apr and 22 May 2016. In each fold the model was retrained using only data before the window.

<!-- Optional: save the "actual vs forecast" chart from the notebook as images/forecast.png and uncomment:
![Actual vs forecast, final 28 days](images/forecast.png)
-->

---

## Dataset

[M5 Forecasting – Accuracy](https://www.kaggle.com/competitions/m5-forecasting-accuracy) (Kaggle): daily unit sales of 3,049 products across 10 Walmart stores in California, Texas and Wisconsin, from 29 Jan 2011 to 22 May 2016.

| File | Contents |
| --- | --- |
| `sales_train_evaluation.csv` | 30,490 item-store rows × 1,941 daily columns (~59M daily records) |
| `calendar.csv` | Date mapping, events and SNAP (food-assistance) days per state |
| `sell_prices.csv` | Weekly price per store and item |

The data is not included in this repo; attach it through Kaggle or download it from the competition page.

---

## Pipeline

```
Raw CSVs (59M daily records)
   │
   ├─ 1. Data-quality checks ── 6 checks, pipeline stops if any fail
   │
   ├─ 2. Aggregate + reshape ── 70 store × dept series, long format,
   │                            unit totals reconciled against raw data
   │
   ├─ 3. Feature engineering ── lags, rolling stats, price, calendar
   │
   ├─ 4. Model ─────────────── LightGBM (Tweedie), early stopping
   │
   └─ 5. Evaluate ──────────── vs. 2 baselines, 3-fold backtest
```

### 1. Data-quality checks

Each check counts failing values, and the notebook asserts that all are zero before any modelling:

1. Duplicate item-store ids
2. Null daily sales values
3. Negative daily sales values
4. Sales days missing from the calendar
5. Duplicate price keys (store, item, week)
6. Non-positive prices

After reshaping, a **reconciliation** check confirms that total units equal the raw data's total, so nothing is lost or double-counted.

### 2. Aggregation level

Forecasts are made at the **store × department × day** level (10 stores × 7 departments = 70 series). This is a level at which inventory is planned, and it reduces noise compared with item level, where most products sell 0–2 units a day. Item rows are summed before the wide-to-long reshape, which keeps memory small while using every raw record.

### 3. Leakage-safe features

The model forecasts the next 28 days in one shot, so **every sales-based feature looks back at least 28 days**. A feature such as "sales 7 days ago" would not exist yet for most of the forecast window, and using it would inflate the score.

| Feature group | Features |
| --- | --- |
| Lags | 28, 35, 42 and 364 days |
| Rolling statistics | 7- and 28-day mean, 28-day std, all ending 28 days back |
| Price | % change in average department price vs. 28 days earlier |
| Calendar (known in advance) | weekday, month, event flag, SNAP day for the store's state |
| Identifiers | store and department (as categorical features) |

### 4. Model

- **LightGBM** with a **Tweedie** objective, which suits non-negative, right-skewed sales data.
- **Time-ordered split:** train → 28-day validation window (for early stopping) → 28-day test window, never random.
- After early stopping finds the number of trees, the model is refit on train + validation and then predicts the unseen test window.

### 5. Evaluation

The model is compared with two free baselines (4-weeks-ago and same-week-last-year) and then re-evaluated at three different cut-off dates. Using a rolling-origin backtest shows the improvement is consistent rather than the result of one favourable window.

---

## Key design decisions

| Decision | Reason |
| --- | --- |
| Validate data before modelling | Bad inputs fail loudly instead of silently corrupting results |
| Aggregate to store × department | Matches inventory planning and makes the signal less noisy |
| All sales features ≥ 28 days back | Prevents leakage for a 28-day horizon |
| Tweedie loss | Fits count-like, skewed sales better than squared error |
| Seasonal baselines first | A model is only useful if it beats free, simple methods |
| Rolling-origin backtest | One test window can be lucky; three give a mean and spread |

---

## How to run

**On Kaggle (recommended)**

1. Accept the rules on the M5 competition page.
2. Create → New Notebook → File → Import Notebook, and upload `m5_quick_forecast.ipynb`.
3. Add Input → Competitions → M5 Forecasting – Accuracy.
4. Run All (about 5–10 minutes on the free CPU).

**Locally**

```bash
pip install pandas numpy lightgbm matplotlib jupyter
export M5_DIR=/path/to/m5/csvs
jupyter notebook m5_quick_forecast.ipynb
```

## Repository structure

```
├── m5_quick_forecast.ipynb   # full pipeline: checks → features → model → backtest
├── README.md
└── images/                   # optional: saved charts
```

---

## Limitations

- **No inventory data:** M5 has no stock levels, so the project forecasts demand but cannot measure actual stockouts.
- **Store × department level only:** item-level forecasts, which are needed for replenishment, are not modelled yet.
- **Average department price** mixes price changes with changes in which items are on sale, so it is only a rough price signal.

## Next steps

- Rebuild the pipeline as a Databricks lakehouse (bronze/silver/gold Delta tables with PySpark), orchestrated as a scheduled job.
- Move data-quality checks into reusable functions with pytest unit tests and GitHub Actions CI.
- Extend forecasts to item level and add a dashboard highlighting likely under- and over-forecast store-departments.

## Tools

Python, pandas, NumPy, LightGBM, Matplotlib, Kaggle Notebooks.


