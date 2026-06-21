# Credit Risk Classification — Binary Default Prediction

![Python](https://img.shields.io/badge/Python-3.10-blue)
![CatBoost](https://img.shields.io/badge/Model-CatBoost-green)
![ROC-AUC](https://img.shields.io/badge/ROC--AUC-0.9313-brightgreen)
![Recall](https://img.shields.io/badge/High--Risk%20Recall-81%25-orange)
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
    → Label Encoding (categorical variables)
    → Binary Target Creation (P1+P2 = 0, P3+P4 = 1)
    → Train/Test Split (80/20)
    → Model Training (LR, RF, XGBoost, CatBoost)
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

## Models Trained

| Model | Test Accuracy | High-Risk Recall | High-Risk Precision | Train-Test Gap |
|---|---|---|---|---|
| Logistic Regression | 81.5% | 76% | 49% | 0% |
| Random Forest | 85.8% | 82% | 63% | 1% |
| XGBoost | 87.1% | 82% | 70% | 3% |
| **CatBoost** | **87.7%** | **82%** | **72%** | **2%** |

**Final model: CatBoost** — best balance of recall, precision, and stability.

---

## Threshold Tuning (CatBoost)

| Threshold | Recall | Precision | F1 |
|---|---|---|---|
| 0.30 | 0.85 | 0.71 | 0.77 |
| **0.35** | **0.81** | **0.74** | **0.77** |
| 0.40 | 0.78 | 0.77 | 0.77 |
| 0.45 | 0.75 | 0.79 | 0.77 |
| 0.50 | 0.72 | 0.82 | 0.77 |

**Chosen threshold: 0.35** — best balance for a lending context where missing a defaulter is costlier than a wrong rejection.

---

## Final Model Performance

| Metric | Value |
|---|---|
| Test Accuracy | 87.7% |
| High-Risk Recall | 81% |
| High-Risk Precision | 74% |
| F1 Score | 0.77 |
| ROC-AUC | **0.9313** |
| Average Precision (AP) | **0.8709** |
| Train-Test Gap | ~2% |

---

## ROC-AUC Comparison

| Model | ROC-AUC | Avg Precision |
|---|---|---|
| Logistic Regression | 0.8429 | 0.6839 |
| Random Forest | 0.9069 | 0.8280 |
| XGBoost | 0.9286 | 0.8646 |
| **CatBoost** | **0.9313** | **0.8709** |

---

## SHAP Explainability

Top features driving High Risk prediction:

| Rank | Feature | Business Meaning |
|---|---|---|
| 1 | `enq_L3m` | Enquiries in last 3 months — financial stress signal |
| 2 | `Age_Oldest_TL` | Older credit history = more trustworthy |
| 3 | `pct_PL_enq_L6m_of_ever` | % of personal loan enquiries recently |
| 4 | `recent_level_of_deliq` | Recent delinquency severity |
| 5 | `time_since_recent_enq` | How recently they applied for a loan |

**Key SHAP finding:** High `enq_L3m` (many recent loan applications) strongly pushes a borrower toward High Risk. Low `Age_Oldest_TL` (short credit history) also signals risk.

---

## Business Impact

### Confusion Matrix (Test Set — 8,413 borrowers)

| | Predicted Low Risk | Predicted High Risk |
|---|---|---|
| **Actual Low Risk** | 5,379 (TN) | 680 (FP) |
| **Actual High Risk** | 446 (FN) | 1,908 (TP) |

### Financial Impact

| Metric | Value |
|---|---|
| Avg loan amount assumed | Rs. 5,00,000 |
| Loss Given Default | 40% |
| Cost per wrong rejection | Rs. 2,000 |
| **Loss avoided (test set)** | **Rs. 38.16 Cr** |
| Cost of wrong rejections | Rs. 13.6 Lakh |
| **Net benefit (test set)** | **Rs. 38.02 Cr** |
| **Net benefit (42K portfolio)** | **Rs. 189.83 Cr** |

### Risk Bucket Analysis

| Risk Bucket | Borrowers | Actual Default Rate |
|---|---|---|
| Very Low (0–25%) | 5,336 | 5.8% → Auto Approve |
| Low-Medium (25–50%) | 1,014 | 35.1% → Manual Review |
| Medium-High (50–75%) | 733 | 62.1% → Conditional Reject |
| Very High (75–100%) | 1,330 | 92.9% → Auto Reject |

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
Models      : CatBoost, XGBoost, LightGBM, Random Forest, Logistic Regression
Libraries   : scikit-learn, pandas, numpy, matplotlib, seaborn, shap
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
├── catboost_model.pkl                   ← Saved final model
├── shap_feature_importance.png          ← SHAP bar plot
├── shap_summary.png                     ← SHAP beeswarm plot
├── roc_curve.png                        ← ROC curve comparison
├── pr_curve.png                         ← Precision-Recall curve
└── README.md
```

---

## How to Run

```bash
# 1. Clone the repository
git clone https://github.com/your-username/credit-risk-classification.git

# 2. Install dependencies
pip install catboost xgboost lightgbm scikit-learn shap pandas numpy matplotlib seaborn openpyxl

# 3. Open notebook
jupyter notebook Data_Cleaning_Model_Building.ipynb
```

---

## Author

**Aniket Yadav**
B.Sc. Data Science & AI | Mumbai [LinkedIn](https://linkedin.com/in/aniketyadavofficial/) | [GitHub](https://github.com/05Aniket598)
