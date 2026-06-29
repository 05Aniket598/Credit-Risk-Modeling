# Credit Risk Modelling — Binary Default Prediction

![Python](https://img.shields.io/badge/Python-3.10-blue)
![LightGBM](https://img.shields.io/badge/Model-LightGBM-green)
![ROC-AUC](https://img.shields.io/badge/ROC--AUC-0.9318-brightgreen)
![Recall](https://img.shields.io/badge/High--Risk%20Recall-80%25-orange)
![Status](https://img.shields.io/badge/Status-Complete-success)

---

## Project Overview

This project builds a **binary credit risk classifier** on ~42,000 real-world borrower records to identify whether a loan applicant is **High Risk (likely to default)** or **Low Risk (safe to approve)**.

The original problem was a 4-class classification (P1–P4). After deep data analysis, P3 was found to be a **boundary class** — 55.6% of P3 samples overlapped with P2 and 99.5% overlapped with P4 on key features. This made multi-class separation structurally impossible. The problem was redesigned as binary classification, which is also more aligned with real banking decisions.

---

## Business Problem

Banks lose crores of rupees when high-risk borrowers default on loans. A model that can identify these borrowers **before loan approval** can prevent significant financial loss.

> "Should we approve this loan application — or is this borrower likely to default?"

---

## Dataset

| Property | Detail |
|---|---|
| Source | Two case study Excel files (case_study1.xlsx, case_study2.xlsx) |
| Raw records | 51,336 |
| After cleaning | 42,064 |
| Features (final) | 54 |
| Target column | `Approved_Flag` → binary (0 = Low Risk, 1 = High Risk) |
| Class distribution | Low Risk 74%, High Risk 26% |

---

## Project Pipeline

```
Raw Data (2 files)
    → Data Cleaning (-99999 removal, column drops)
    → Merging on PROSPECTID
    → EDA (Chi-square, ANOVA, VIF analysis)
    → Feature Selection (54 features retained)
    → Feature Engineering (enq_pressure, deliq_income_ratio, PL_seeking, borrow_recency)
    → Label Encoding (categorical variables)
    → Binary Target Creation (P1+P2 = 0, P3+P4 = 1)
    → Train/Test Split (80/20)
    → Model Training (LR, RF, XGBoost, LightGBM)
    → Threshold Tuning
    → SHAP Explainability
    → Business Impact Analysis
```

---

## Data Cleaning

- Identified `-99999` as null placeholder across both datasets
- Dropped 8 columns with more than 10,000 missing values
- Removed remaining rows with `-99999` values
- Final dataset: **42,064 records, 54 features**
- Zero actual null values after cleaning

---

## Why Binary (Not Multi-class)?

Multi-class (P1–P4) was attempted first. P3 recall was stuck at 40–47% despite:
- Class weights and SMOTE
- Threshold tuning
- Feature engineering (enquiry pressure, PL seeking score, etc.)

Root cause analysis revealed:

```
P3 samples overlapping with P2 range: 55.6%
P3 samples overlapping with P4 range: 99.5%
```

P3 is a "middle zone" class with no clear feature boundary. Binary conversion resolved this structurally.

---

## Feature Engineering

Four new features were created based on domain logic and SHAP analysis:

| Feature | Formula | Business Meaning |
|---|---|---|
| `enq_pressure` | `enq_L3m / (time_since_recent_enq + 1)` | Recent loan seeking intensity |
| `deliq_income_ratio` | `max_recent_level_of_deliq / (NETMONTHLYINCOME + 1)` | Delinquency relative to income |
| `PL_seeking` | `pct_PL_enq_L6m_of_ever * PL_enq_L12m` | Personal loan seeking behaviour |
| `borrow_recency` | `1 / (time_since_recent_enq + 1)` | How recently borrower applied |

All four engineered features appeared in the top 15 SHAP features, confirming they added value to the model.

---

## Models Trained

| Model | Test Accuracy | High-Risk Recall | High-Risk Precision | ROC-AUC | Train-Test Gap |
|---|---|---|---|---|---|
| Logistic Regression | 81.5% | 76% | 49% | 0.8429 | 0% |
| Random Forest | 85.8% | 82% | 63% | 0.9069 | 1% |
| XGBoost | 87.1% | 80% | 74% | 0.9288 | 3% |
| **LightGBM** | **87.6%** | **80%** | **75%** | **0.9318** | **3%** |

**Final model: LightGBM** — highest ROC-AUC (0.9318) and Average Precision (0.8706) among all models.

> CatBoost's main advantage is native categorical handling. Since all features were pre-encoded, that advantage was unused. LightGBM outperformed on ROC-AUC and Average Precision in this setting.

---

## Threshold Tuning (LightGBM)

| Threshold | Recall | Precision | Accuracy |
|---|---|---|---|
| 0.25 | 0.87 | 0.67 | 84% |
| 0.30 | 0.83 | 0.72 | 86% |
| **0.35** | **0.80** | **0.75** | **87%** |
| 0.40 | 0.78 | 0.77 | 87% |
| 0.45 | 0.74 | 0.80 | 87% |
| 0.50 | 0.72 | 0.82 | 88% |

**Chosen threshold: 0.35** — best balance for a lending context where missing a defaulter is costlier than a wrong rejection.

---

## Final Model Performance

| Metric | Value |
|---|---|
| Test Accuracy | 87.6% |
| High-Risk Recall | 80% |
| High-Risk Precision | 75% |
| F1 Score | 0.77 |
| ROC-AUC | **0.9318** |
| Average Precision (AP) | **0.8706** |
| Train-Test Gap | ~3% |

---

## ROC-AUC Comparison

| Model | ROC-AUC | Avg Precision |
|---|---|---|
| Logistic Regression | 0.8429 | 0.6839 |
| Random Forest | 0.9069 | 0.8280 |
| XGBoost | 0.9288 | 0.8648 |
| **LightGBM** | **0.9318** | **0.8706** |

---

## SHAP Explainability

Top features driving High Risk prediction:

| Rank | Feature | SHAP Value | Business Meaning |
|---|---|---|---|
| 1 | `enq_L3m` | 1.20 | Enquiries in last 3 months — strongest financial stress signal |
| 2 | `Age_Oldest_TL` | 0.65 | Older credit history = more trustworthy borrower |
| 3 | `pct_PL_enq_L6m_of_ever` | 0.50 | % of personal loan enquiries recently |
| 4 | `deliq_income_ratio` | 0.46 | Delinquency level relative to income |
| 5 | `max_recent_level_of_deliq` | 0.42 | Maximum recent delinquency severity |
| 6 | `num_std_12mts` | 0.41 | Standard accounts in last 12 months |
| 7 | `enq_pressure` | 0.33 | Engineered: recent loan seeking intensity |

**Key SHAP findings:**
- High `enq_L3m` strongly pushes a borrower toward High Risk — borrower enquiring 3+ times in 3 months signals urgent financial need
- Low `Age_Oldest_TL` signals new-to-credit borrower — bank has limited credit history to assess risk
- High `pct_PL_enq_L6m_of_ever` means borrower is heavily seeking personal loans recently — stress signal
- Engineered features `deliq_income_ratio` and `enq_pressure` both appear in top 7 — feature engineering added real value

---

## What the model is actually doing?

After seeing SHAP plots here is the conclusion.

`enq_L3m` is the major driver of the model with SHAP value of 1.20. Higher enquiry leads to risk according to Beeswarm plot — if a customer is enquiring for a loan a lot, it is risky to give him a loan.

`Age_Oldest_TL` is the second major driver. If the age of oldest account is greater, the customer is safer — because bank has some credit history information about them.

`pct_PL_enq_L6m_of_ever` is the third major driver with 0.50 SHAP value. Higher personal loan enquiry in recent months leads to higher default probability.

**Single borrower explanation (index 1):** Borrower has +1.97 SHAP for `enq_L3m` — enquired 3 times in 3 months. `Age_Oldest_TL` is 6 months — very new to credit. 50% of all their enquiries are for personal loans. Model predicted 3.369 vs dataset average of -1.705. Classified as **High Risk**.

---

## Business Impact

### Confusion Matrix (Test Set — 8,413 borrowers)

| | Predicted Low Risk | Predicted High Risk |
|---|---|---|
| **Actual Low Risk** | 5,404 (TN) | 655 (FP) |
| **Actual High Risk** | 468 (FN) | 1,886 (TP) |

### Financial Impact

| Metric | Value |
|---|---|
| Avg loan amount assumed | Rs. 5,00,000 |
| Loss Given Default | 40% |
| Cost per wrong rejection | Rs. 2,000 |
| **Loss avoided (test set)** | **Rs. 37.72 Cr** |
| Loss still at risk (missed) | Rs. 9.36 Cr |
| Cost of wrong rejections | Rs. 13.1 Lakh |
| **Net benefit (test set)** | **Rs. 37.59 Cr** |
| **Net benefit (42K portfolio)** | **Rs. 187.65 Cr** |

### Risk Bucket Analysis

| Risk Bucket | Borrowers | Actual Default Rate | Action |
|---|---|---|---|
| Very Low (0–25%) | 5,409 | 5.8% | Auto Approve |
| Low-Medium (25–50%) | 930 | 37.5% | Manual Review |
| Medium-High (50–75%) | 627 | 59.8% | Conditional Reject |
| Very High (75–100%) | 1,447 | 90.9% | Auto Reject |

Model is able to classify 80 risky customers out of 100 actual risky customers. Borrowers with very high risk score (75–100%) have 90.9% actual default rate — almost every borrower in this bucket is genuinely risky. Borrowers in very low bucket (0–25%) have only 5.8% default rate — safe to approve directly.

---

## Business Decision Rule

```
Probability Score    Action
─────────────────────────────────────
0% – 25%            Auto Approve
25% – 50%           Send for Manual Review
50% – 75%           Conditional Reject
75% – 100%          Auto Reject
```

---

## Tech Stack

```
Language    : Python 3.10
Final Model : LightGBM (GPU accelerated)
All Models  : LightGBM, XGBoost, Random Forest, Logistic Regression
Libraries   : scikit-learn, pandas, numpy, matplotlib, seaborn, shap, lightgbm
GPU         : NVIDIA Tesla T4 (Google Colab)
Platform    : Google Colab
```

---

## Repository Structure

```
credit-risk-classification/
│
├── Data_Cleaning_Model_Building.ipynb   ← Main notebook
├── case_study1.xlsx                     ← Raw data file 1
├── case_study2.xlsx                     ← Raw data file 2
├── lgbm_model.pkl                       ← Saved final model
├── shap_feature_importance.png          ← SHAP bar plot
├── shap_summary.png                     ← SHAP beeswarm plot
├── shap_waterfall.png                   ← Single borrower explanation
├── roc_curve.png                        ← ROC curve comparison
├── pr_curve.png                         ← Precision-Recall curve
└── README.md
```

---

## How to Run

```bash
# 1. Clone the repository
git clone https://github.com/05Aniket598/credit-risk-classification.git

# 2. Install dependencies
pip install lightgbm xgboost catboost scikit-learn shap pandas numpy matplotlib seaborn openpyxl

# 3. Open notebook
jupyter notebook Data_Cleaning_Model_Building.ipynb
```

---

## Author

**Aniket Yadav**
B.Sc. Data Science & AI | Mumbai
[LinkedIn](https://linkedin.com/in/aniketyadavofficial/) | [GitHub](https://github.com/05Aniket598)
