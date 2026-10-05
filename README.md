# Minor_Project_6
# 🛒 QuickCart Predictive Inventory: Stockout Risk Forecasting

A robust **machine learning data pipeline and feature engineering project** designed to predict daily inventory stockout risks for **QuickCart**.

The project processes daily **panel inventory data** across multiple stores and SKUs and classifies inventory conditions into three actionable risk tiers:

* 🟢 **Safe** — Inventory is healthy and stockout risk is low.
* 🟡 **At-Risk** — Inventory requires monitoring or potential replenishment.
* 🔴 **Imminent** — High probability of stockout requiring immediate action.

The complete pipeline transforms raw relational inventory data into a **37-column numeric feature matrix** ready for predictive modeling.

---

## 📊 Project Overview

Supply chain disruptions, supplier delays, promotional campaigns, and localized demand spikes can create expensive inventory stockouts.

This project focuses on identifying **high-risk inventory moments before a stockout occurs**, allowing QuickCart to make proactive replenishment decisions.

### Dataset Scale

| Component                  |   Size |
| -------------------------- | -----: |
| 🏪 Stores                  |     12 |
| 📦 SKUs                    |     60 |
| 🚚 Suppliers               |     15 |
| 📅 Events                  |     30 |
| 📈 Daily Inventory Records | 21,600 |
| 🧮 Final ML Features       |     37 |

The dataset covers a **30-day period from October 1–30, 2026**.

---

# 🗄️ Data Architecture

The project follows a classic **Kimball Star Schema** architecture designed for daily-grain inventory and time-series analysis.

```text
                    ┌─────────────────┐
                    │   dim_stores    │
                    │    12 rows      │
                    └────────┬────────┘
                             │
                             │
┌─────────────────┐          ▼               ┌──────────────────┐
│  dim_suppliers  │ ───► fact_inventory ◄─── │    dim_skus      │
│    15 rows      │       _daily             │     60 rows      │
└─────────────────┘       21,600 rows        └──────────────────┘
                             ▲
                             │
                    ┌────────┴────────┐
                    │   dim_events    │
                    │    30 rows      │
                    └─────────────────┘
```

## 📁 Dataset Files

### `fact_inventory_daily.csv`

Central fact table containing daily inventory movements.

**Rows:** 21,600

Contains information such as:

* Store
* SKU
* Date
* Opening stock
* Closing stock
* Reorder point
* Demand
* Replenishment
* Inventory status

---

### `dim_stores.csv`

Store-level dimension containing information about the QuickCart store network.

**Rows:** 12

Includes:

* Store identifiers
* Store location
* City
* Store size
* Geographic attributes

---

### `dim_skus.csv`

Product-level dimension containing information about the 60 SKUs.

**Rows:** 60

Includes:

* SKU identifiers
* Product category
* Product characteristics
* Perishability flags

---

### `dim_suppliers.csv`

Supplier dimension containing supplier performance information.

**Rows:** 15

Includes:

* Supplier identifiers
* Base lead time
* Historical reliability score
* Supplier performance metrics

---

### `dim_events.csv`

Promotional and seasonal event calendar.

**Rows:** 30

Includes:

* Event dates
* Event names
* Promotional periods
* Demand multipliers
* Festival information

---

# 🎯 Prediction Target

The inventory condition is represented using three risk classes:

| Risk Level  | Encoding | Meaning                      |
| ----------- | -------: | ---------------------------- |
| 🟢 Safe     |      `0` | Healthy inventory position   |
| 🟡 At-Risk  |      `1` | Inventory requires attention |
| 🔴 Imminent |      `2` | High-cost stockout risk      |

The target labels are converted into numeric values for machine learning compatibility.

```text
Safe       → 0
At-Risk    → 1
Imminent   → 2
```

---

# 📈 Key Business Insights

Exploratory Data Analysis revealed several important **money-moment signals** affecting stockout risk.

## ⚖️ Class Imbalance

The target distribution reflects a realistic inventory-risk scenario:

| Risk Class  | Percentage |
| ----------- | ---------: |
| 🟢 Safe     | **65.42%** |
| 🟡 At-Risk  | **24.01%** |
| 🔴 Imminent | **10.57%** |

The **Imminent** class is relatively rare but has the highest business cost, making accurate detection particularly important.

---

## 🎆 Festival Multiplier

During **Diwali Week (October 22–26, 2026)**, inventory pressure increases significantly.

The Imminent stockout rate rises:

```text
Normal Period       → 9.51%
Diwali Week         → 23.31%
Increase            → 2.45×
```

This indicates that promotional and festival demand should be incorporated into stockout-risk forecasting.

---

## 🚚 Supplier Reliability

Supplier reliability is another major risk driver.

| Supplier Reliability | Imminent Stockout Rate |
| -------------------- | ---------------------: |
| Reliability < 0.75   |             **15.83%** |
| Highly Reliable      |              **3.82%** |

Low-reliability suppliers therefore represent a substantially higher stockout risk.

---

## 🍎 Perishability Penalty

Perishable products show slightly higher stockout risk compared with non-perishable products.

| Product Type      | Imminent Risk |
| ----------------- | ------------: |
| 🥬 Perishable     |     **12.8%** |
| 📦 Non-Perishable |      **9.3%** |

This suggests that perishability should be considered when prioritizing inventory monitoring.

---

# 🧹 Data Cleaning

Several data-quality operations were performed before feature engineering.

## 🌍 City Casing Standardization

Inconsistent geographic values can create duplicate groups during aggregation.

For example:

```text
ahmedabad
Ahmedabad
AHMEDABAD
```

were standardized using **title casing**:

```text
Ahmedabad
```

This prevents fragmented geographic groups during analysis.

---

## 🧠 Smart Supplier Imputation

Some supplier reliability values were missing (`N/A`).

Instead of using a single global value, missing reliability scores were imputed using the **median reliability score for the corresponding product category**.

The resulting feature is:

```text
supplier_reliability_clean
```

This preserves category-level supplier behavior more effectively than simple global imputation.

---

# ⚙️ Feature Engineering

The cleaned relational data was transformed into predictive features representing inventory pressure, replenishment behavior, and temporal demand conditions.

---

## 📉 1. Reorder Gap

### `reorder_gap`

Measures the numerical distance between current closing stock and the reorder point.

Conceptually:

```text
reorder_gap = closing_stock - reorder_point
```

Interpretation:

```text
Positive value → Stock above reorder point
Near zero       → Replenishment may be required
Negative value  → Stock below reorder point
```

This provides the model with a direct measurement of inventory pressure.

---

## 📊 2. Days of Cover Ratio

### `days_of_cover_ratio`

Measures available inventory coverage relative to expected supplier lead time.

Conceptually:

```text
days_of_cover_ratio =
    available_days_of_cover / expected_lead_time_days
```

Interpretation:

```text
> 1  → Inventory covers lead time comfortably
≈ 1  → Inventory is close to lead-time requirement
< 1  → Potential stockout exposure
```

---

## 🔄 3. Recent Reorder Memory

### `is_recent_reorder`

Inventory risk can depend on what happened during the previous few days.

A **3-day rolling-window maximum** was calculated separately for each:

```text
Store + SKU
```

This creates a sequential-memory feature that indicates whether a recent replenishment order occurred.

Conceptually:

```text
Store A + SKU 101
        │
        ├── Day -2
        ├── Day -1
        ├── Today
        │
        └── is_recent_reorder
```

This helps the model understand short-term replenishment behavior.

---

## 🎆 4. Festival Timing

### `days_since_festival_start`

Static event dates were transformed into a dynamic temporal feature.

Instead of simply storing:

```text
Festival = Diwali
```

the pipeline calculates the relative distance from the festival start date.

This provides the model with information such as:

```text
Days before festival
        ↓
Festival start
        ↓
Days after festival
```

This allows the model to learn changing demand pressure around major events.

---

# 🧪 Machine Learning Validation Strategy

Because this is **time-series panel data**, random train/test splitting can introduce data leakage.

For example, randomly selecting October 10 and October 25 into the training set could allow the model to indirectly learn information from the future.

To prevent this, the project uses a **strict chronological split**.

---

## 📅 Training Data

```text
October 1 – October 23, 2026
```

**Rows:** 16,560

This represents historical information available to the model.

---

## 🔮 Testing Data

```text
October 24 – October 30, 2026
```

**Rows:** 5,040

This represents future observations used to evaluate generalization.

---

### Timeline

```text
October 1                         October 23     October 24                    October 30
│-------------------------------------│---------------│----------------------------│
              TRAINING DATA                            TEST DATA
                16,560                                  5,040
```

This chronological approach provides a more realistic evaluation of how the model would perform when predicting future inventory risk.

---

# 🔐 Leakage Prevention

The validation strategy is specifically designed to avoid **temporal data leakage**.

### ❌ Avoided

```text
Random Train/Test Split
```

because future observations could influence the training dataset.

### ✅ Used

```text
Past → Training
Future → Testing
```

This better represents a real-world forecasting environment.

---

# 🧮 Final Feature Matrix

After cleaning, joining the star-schema tables, encoding the target, and engineering predictive variables, the dataset is transformed into a clean numerical matrix.

```text
X_train
16,560 rows × 37 features

X_test
5,040 rows × 37 features
```

The final modeling dataset removes:

* Redundant identifiers
* Unnecessary columns
* Unencoded categorical strings
* Non-predictive metadata

The resulting matrix is fully numeric and ready for classifier training.

---

# 🤖 Machine Learning Pipeline

The overall pipeline can be summarized as:

```text
Raw CSV Files
      │
      ▼
Star Schema Join
      │
      ▼
Data Cleaning
      │
      ├── City Standardization
      └── Missing Value Imputation
      │
      ▼
Feature Engineering
      │
      ├── Reorder Gap
      ├── Days of Cover Ratio
      ├── Recent Reorder
      └── Festival Timing
      │
      ▼
Target Encoding
      │
      ▼
Chronological Train/Test Split
      │
      ├───────────────┐
      ▼               ▼
   X_train          X_test
  16,560 rows      5,040 rows
  37 features      37 features
      │               │
      └───────┬───────┘
              ▼
      Machine Learning
          Classifier
              │
              ▼
     Stockout Risk Prediction
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
     Safe  At-Risk Imminent
```

---

# 💰 Business Value

The goal of this project is not simply to maximize model accuracy.

The primary objective is to identify **high-cost inventory moments early enough for the business to act**.

Potential business actions include:

### 🟢 Safe

Continue normal inventory operations.

### 🟡 At-Risk

Consider:

* Monitoring inventory
* Reviewing upcoming demand
* Checking supplier status
* Preparing replenishment

### 🔴 Imminent

Prioritize immediate action:

* Expedite replenishment
* Contact suppliers
* Reallocate inventory between stores
* Adjust promotional exposure
* Prioritize critical SKUs

---

# 📌 Project Highlights

* ⭐ **21,600** daily inventory observations
* 🏪 **12** stores
* 📦 **60** SKUs
* 🚚 **15** suppliers
* 🎆 **30** event records
* 🧮 **37** final ML features
* 📅 Chronological time-series validation
* 🧹 Category-aware missing-value imputation
* 📉 Inventory-pressure feature engineering
* 🔄 Rolling-window sequential features
* 🎆 Festival-demand modeling
* 🎯 Three-class stockout-risk prediction
* 🔐 Leakage-aware ML pipeline

---

# 🛠️ Technologies & Concepts

The project demonstrates concepts including:

* Python
* Pandas
* NumPy
* Data Cleaning
* Exploratory Data Analysis
* Feature Engineering
* Time-Series Analysis
* Panel Data
* Star Schema
* Machine Learning
* Classification
* Categorical Encoding
* Missing-Value Imputation
* Rolling Window Features
* Chronological Train/Test Validation
* Data Leakage Prevention

---

# 📂 Recommended Project Structure

```text
QuickCart-Predictive-Inventory/
│
├── data/
│   ├── fact_inventory_daily.csv
│   ├── dim_stores.csv
│   ├── dim_skus.csv
│   ├── dim_suppliers.csv
│   └── dim_events.csv
│
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   ├── 02_data_cleaning.ipynb
│   ├── 03_feature_engineering.ipynb
│   └── 04_modeling.ipynb
│
├── src/
│   ├── data_processing.py
│   ├── feature_engineering.py
│   └── modeling.py
│
├── outputs/
│   ├── X_train.csv
│   └── X_test.csv
│
├── README.md
└── requirements.txt
```

---

# 🚀 Future Improvements

Potential extensions to the project include:

* Train multiple classification algorithms
* Handle class imbalance using appropriate techniques
* Compare macro F1-score across models
* Optimize the decision threshold for the **Imminent** class
* Add model explainability using feature importance
* Implement SHAP-based explanations
* Introduce weather and regional demand signals
* Add real-time inventory monitoring
* Build a stockout-risk dashboard
* Deploy the model as an API
* Automate daily risk predictions

---

# 📊 Evaluation Focus

Since the **Imminent** class represents high-cost stockouts, evaluation should not rely exclusively on overall accuracy.

Recommended metrics include:

```text
Accuracy
Precision
Recall
F1-Score
Macro F1
Confusion Matrix
Imminent-Class Recall
Imminent-Class Precision
```

Particular attention should be given to **Imminent recall**, because failing to identify a genuine high-risk inventory situation can have significant business consequences.

---

# 🎯 Final Objective

The QuickCart Predictive Inventory pipeline converts raw inventory, store, SKU, supplier, and promotional data into a structured machine-learning dataset capable of identifying daily stockout risk.

The ultimate objective is:

> **Predict inventory risk early, prioritize high-cost stockout situations, and enable proactive replenishment decisions.**

---

## 👤 Project

**QuickCart Predictive Inventory — Stockout Risk Forecasting**

**Domain:** Supply Chain Analytics / Machine Learning / Predictive Inventory Management

**Data Granularity:** Store × SKU × Day

**Prediction Type:** Multi-Class Classification

**Risk Classes:** Safe · At-Risk · Imminent

**Validation:** Chronological Time-Series Split
