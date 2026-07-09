# Financial-Operations-Analytics
SaaS revenue forecasting, churn prediction &amp; customer segmentation (RFM/CLV) — Python, Power BI, 135K+ transactions.
# SaaS Financial Operations Analytics

End-to-end analytics on a synthetic SaaS company: revenue forecasting, churn prediction, customer segmentation, and profitability analysis across 5,000 customers and 135,218 transactions.

![Executive Dashboard](financial_viz/16_FINAL_EXECUTIVE_DASHBOARD.png)

## Contents

- [Overview](#overview)
- [Key Metrics](#key-metrics)
- [Dataset](#dataset)
- [Methodology](#methodology)
  - [1. Revenue Forecasting](#1-revenue-forecasting)
  - [2. Churn Prediction](#2-churn-prediction)
  - [3. Cohort, RFM & CLV Analysis](#3-cohort-rfm--clv-analysis)
  - [4. Profitability & Clustering](#4-profitability--clustering)
- [Known Limitation: Data Leakage](#known-limitation-data-leakage)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Setup](#setup)
- [Business Insights](#business-insights)
- [Possible Next Steps](#possible-next-steps)
- [License](#license)

## Overview

This project analyzes a synthetic SaaS company's financial and customer data across 60 months (Jan 2020–Dec 2024). It's built as a portfolio project, so the dataset is generated rather than pulled from a real company — but the analysis techniques, the model selection process, and the mistakes along the way are real.

The notebook works through four connected questions:

1. What will revenue look like next year?
2. Which customers are about to leave, and why?
3. What are customers actually worth, and how do they group together?
4. Which segments make the most money, and which look different once you cluster on behavior instead of on labels like "Enterprise" or "Startup"?

## Key Metrics

| Metric | Value |
|---|---|
| Total revenue (all-time) | $35,834,169.55 |
| Current MRR | $1,160,806 |
| Annual Recurring Revenue | $13,929,672 |
| Total customers | 5,000 (4,344 active / 656 churned) |
| Churn rate | 13.12% |
| Average CLV | $7,166.83 (median $2,907.19) |
| CLV : CAC ratio | 14.33x (assumed CAC $500) |
| 12-month revenue forecast | $15,717,864.32 |
| Forecast accuracy (MAPE) | 0.73% |
| Customers flagged at-risk (>50% churn probability) | 652 |

All figures above come straight from `kpi_summary.txt`, which the notebook generates from the underlying data — nothing here is hand-typed.

## Dataset

All data is synthetically generated (Section 1 of the notebook) rather than pulled from Kaggle or another public source. Two tables:

- **Customers** (5,000 rows, 22 columns) — signup date, segment, industry, plan, MRR, usage score, NPS, churn status, lifetime revenue, CLV
- **Transactions** (135,218 rows) — one row per billing event, tagged with type, payment method, and signup cohort

Generating the data myself meant I could bake in realistic patterns — churn correlating with low usage and high support-ticket volume, mild seasonality in revenue — while being upfront that it isn't real company data.

## Methodology

### 1. Revenue Forecasting

- Decomposed the monthly revenue series into trend, seasonal, and residual components. Seasonal strength came out at 0.27 against a trend strength of 1.00, so most of the movement in this series is trend, not seasonality
- Ran an Augmented Dickey-Fuller test for stationarity: the raw series was non-stationary (p = 0.994); first-differencing brought it to p = 0.0001
- Used ACF/PACF plots to narrow down candidate ARIMA parameters, then grid-searched 18 combinations of (p, d, q) and picked the best one by AIC — ARIMA(1,1,2), AIC 922.5
- Validated on a held-out 12-month test set: MAE $7,866.61, RMSE $9,225.94, MAPE 0.73%
- Cross-checked the ARIMA forecast against an independent Facebook Prophet model (additive seasonality, yearly seasonality on). The two forecasts landed within 0.72% of each other on the 12-month total, which is a reasonable sanity check for two unrelated methods
- Final 12-month forecast: $15,717,864.32, with 95% confidence intervals

![ARIMA Forecast](financial_viz/04_arima_forecast.png)

### 2. Churn Prediction

- Engineered 19 features from the raw customer table: 15 numeric (including customer age, tickets per month, and flags for low usage, low NPS, and short contracts) plus 4 encoded categoricals (segment, industry, plan, country)
- Split 80/20, stratified by churn label, and trained three classifiers
- Logistic Regression: 99.80% accuracy, 0.9947 ROC-AUC
- Random Forest and Gradient Boosting: both 100% accuracy, 1.0000 ROC-AUC

That last line should make you suspicious. It made me suspicious too — see the section below.

![Churn Model Evaluation](financial_viz/08_churn_model_evaluation.png)

### 3. Cohort, RFM & CLV Analysis

- Built a monthly cohort retention matrix across all 60 signup cohorts. Retention holds at 100% for the first three months, then decays gradually to 92.8% by month 12, averaged across cohorts
- Segmented active customers (4,344) using RFM scoring (recency, frequency, monetary — quintile-based) into 8 named groups: Champions, Loyal Customers, At Risk, Lost, and others. The "At Risk" group alone holds $10.3M in historical revenue, which is why RFM earns a place next to the churn model rather than being redundant with it
- Looked at CLV by segment (Mid-Market highest, $7,477.64 average) and by signup cohort year. Older cohorts show higher CLV, which mostly just reflects that they've had more time to accumulate revenue — CLV here is realized-to-date spend, not a modeled future value

![RFM Segmentation](financial_viz/13_rfm_analysis.png)

### 4. Profitability & Clustering

- Broke down profitability by segment, plan, and country under an assumed flat 70% gross margin. Small Business generates the most gross profit of any segment ($9.9M, from $14.1M in revenue across 1,993 customers); Mid-Market has the best per-customer economics (highest average CLV)
- Ran K-Means clustering on 6 behavioral features (MRR, lifetime, usage score, NPS, transaction count, support tickets), standardized, checked inertia from k=2 to k=10, and settled on 5 clusters. One of them — 459 customers, MRR near $1,000 (well above the other four clusters) but usage score slightly below the cross-cluster average — doesn't show up in either the segment labels or the RFM groups. High spend without matching engagement is exactly what should worry a retention team, so I named it "Premium At-Risk"

## Known Limitation: Data Leakage

The churn models above aren't deployable as-is, and it's worth being upfront about why rather than let the 100% ROC-AUC numbers stand unexplained.

Pulling feature importances from the Random Forest model shows that four features account for 93% of the model's decisions: `recency_days` (51%), `lifetime_months` (17%), `transaction_count` (13%), and `customer_age_days` (12%).

All four are mechanically tied to the churn event itself. In this synthetic dataset, once a customer churns, `recency_days` stops updating and grows large, while `lifetime_months` and `transaction_count` freeze at whatever they were on the churn date. The model isn't predicting churn — it's detecting churn that's already happened, using features that only look informative in hindsight. That's data leakage, and it's why three ordinary classifiers hit a near-perfect score on a problem that shouldn't score perfectly.

![Feature Importance](financial_viz/09_churn_feature_importance.png)

The fix, if this went to production, is straightforward: replace those four features with trailing-window behavioral signals computed as of 30/60/90 days *before* the observation date — usage trend over the last quarter, ticket volume trend, that kind of thing — so nothing in the feature set can see the churn event coming. I left the leaky version in the notebook instead of quietly fixing it, because catching this is a bigger part of the job than posting a high AUC number.

One more thing worth being honest about: the 652-customer at-risk list (`at_risk_customers.csv`) is generated by this same model, so it inherits the same problem. It's flagging customers who resemble already-churned customers on leaky features, not customers showing early warning signs. Treat the KPIs downstream of it (at-risk MRR, potential revenue loss) as a demonstration of what the pipeline *would* report, not a real prioritization list, until the features get fixed.

## Tech Stack

- **Language:** Python 3
- **Data handling:** pandas, NumPy
- **Statistics & time series:** statsmodels (seasonal decomposition, ADF test, ACF/PACF, ARIMA), Facebook Prophet
- **Machine learning:** scikit-learn (Logistic Regression, Random Forest, Gradient Boosting, K-Means, StandardScaler, LabelEncoder)
- **Visualization:** Matplotlib, Seaborn
- **BI / dashboarding:** Power BI
- **Environment:** Jupyter Notebook

## Project Structure

```
FINANCIAL_OPERATIONS_ANALYTICS_PROJECT/
├── MyProject_Financila_Analytics.ipynb   # Main notebook — 7 sections, 89 cells
├── financial_customers_new.csv           # 5,000 customers, 22 columns
├── financial_transactions_new.csv        # 135,218 transactions
├── monthly_revenue_new.csv               # 60-month revenue time series
├── rfm_segmentation.csv                  # RFM scores + segment labels
├── at_risk_customers.csv                 # Churn model output, 652 flagged customers
├── kpi_summary.txt                       # Headline KPIs
├── financial_viz/                        # 16 saved charts
└── README.md
```

One thing to fix before this goes on public GitHub: the notebook filename has a typo (`Financila`). Worth renaming to something clean, like `saas_financial_analytics.ipynb`, since it's the first thing a recruiter's eye will catch.

## Setup

```bash
git clone <your-repo-url>
cd FINANCIAL_OPERATIONS_ANALYTICS_PROJECT
pip install -r requirements.txt
jupyter notebook MyProject_Financila_Analytics.ipynb
```

Nothing here needs special hardware — the dataset is small enough that the whole notebook, ARIMA grid search included, runs on a normal laptop in a few minutes.

## Business Insights

Translating the above into plain terms:

- Revenue is on a steady upward trend (+13.26% over the last 6 months) with weak seasonality, so growth isn't concentrated in a handful of months
- 652 customers are currently flagged above a 50% churn probability, representing roughly $2.0M in annual revenue at stake
- Small Business generates the most total profit, on the back of having both the most customers and the highest revenue; Mid-Market customers are individually the most valuable
- $10.3M in revenue sits with customers RFM-labeled "At Risk" — worth a retention push before they join the 656 who've already left

## Possible Next Steps

- Rebuild the churn model on leakage-free, trailing-window features and re-measure the real AUC
- Score the K-Means clusters with silhouette analysis instead of just eyeballing the elbow plot — the notebook sets up a list for this and never fills it in
- Publish the Power BI dashboard as a shared link and add it here
- Move the CSVs into a small SQLite or Postgres database instead of flat files

## License

MIT — see `LICENSE`.
