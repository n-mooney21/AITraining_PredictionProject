# Network Traffic Analysis & Brute Force Detection

A machine learning project that explores a dataset of ~6,000 network sessions, prepares the data for modelling, and trains two Random Forest models:

- a **classifier** that labels a session as `Benign` or `BruteForce`
- a **regressor** that predicts a session's `avg_packet_size`

> **Note:** This project was originally written and run in **Google Colab** as a Jupyter notebook (`AIDataProject.ipynb`). The Python file in this repo (`aidataproject.py`) is the script export of that notebook, so it still contains Colab-style cell separators and markdown headings as comments.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Dataset](#dataset)
3. [Repository Structure](#repository-structure)
4. [Getting Started](#getting-started)
5. [Methodology](#methodology)
6. [Results](#results)
7. [Limitations & Future Work](#limitations--future-work)
8. [Tech Stack](#tech-stack)

---

## Project Overview

The goal is to understand how a network is affected by different types of traffic and to see whether a model can help flag malicious activity. The work is split into three parts:

1. **Dataset understanding and exploration** – inspect the data and visualise it.
2. **Data preparation** – handle missing values, engineer new features, encode categorical data and split into train/test sets.
3. **Model training and evaluation** – train and evaluate a Random Forest classifier and regressor.

## Dataset

The script expects a CSV named `Dataset2_odd_students.csv`, supplied separately and not published here. It is **not included** in this repository, so place your copy in the same folder as the script (or upload it to your Colab session).

- **Rows:** 6,000 network sessions
- **Columns:** 19 attributes

| Type | Columns |
|------|---------|
| Categorical | `packet_session_id`, `protocol`, `device_type`, `vpn_status`, `region`, `tcp_flags`, `privilege_level`, `attack_category` |
| Numerical | `login_attempts`, `failed_logins`, `bytes_sent`, `bytes_received`, `packet_rate`, `connection_count`, `port_number`, `session_retries`, `signal_strength`, `anomaly_score`, `avg_packet_size` |

**Missing values:** `protocol`, `bytes_received` and `signal_strength` each have 5,580 non-null entries out of 6,000.

**Class balance (`attack_category`):**

| Class | Sessions |
|-------|----------|
| Benign | 4,161 |
| BruteForce | 1,839 |

## Repository Structure

```
.
├── aidataproject.py                      # Script export of the Colab notebook
├── AI_Data_Science_Project_Report.pdf      # Written project report
├── Dataset2_odd_students.csv             # (not included) dataset used by the script
└── README.md
```

## Getting Started

### Option 1: Google Colab (original environment)

1. Create a new Colab notebook and paste in the contents of `aidataproject.py` (or upload the original `.ipynb` if you have it).
2. Upload `Dataset2_odd_students.csv` to the Colab session files.
3. Run all cells (**Runtime → Run all**).

### Option 2: Run locally

**Requirements:** Python 3.8+ and the following packages:

```bash
pip install pandas numpy matplotlib seaborn scikit-learn
```

**Run:**

```bash
python aidataproject.py
```

Make sure `Dataset2_odd_students.csv` is in the working directory. Because the script was exported from a notebook, plots are shown with `plt.show()` and bare expressions such as `df.describe()` will not print anything outside of a notebook. Wrap them in `print()` if you want that output when running locally.

## Methodology

### 1. Exploration

The script produces the following visualisations:

- **Correlation heatmap** of all numeric features
- **Bar chart** of attack categories
- **Histogram** (with KDE) of `avg_packet_size`
- **Boxplot** of `avg_packet_size` to check for outliers
- **Scatter plot** of `bytes_sent` vs `avg_packet_size`, coloured by attack category

Key observations from the report:

- `avg_packet_size` is roughly bell-shaped, centred around ~290 bytes, with outliers on both sides.
- `avg_packet_size` shows a moderate positive correlation with `packet_rate` (around 0.5).
- Most sessions cluster tightly at lower `bytes_sent`, with a small number of high-volume sessions standing out.

### 2. Data Preparation

**Missing values**

| Column | Type | Strategy |
|--------|------|----------|
| `bytes_received` | Numerical | Filled with the median |
| `signal_strength` | Numerical | Filled with the median |
| `protocol` | Categorical | Filled with the mode |

The median was chosen over the mean because it is less affected by skew and outliers.

**Engineered features**

| Feature | Formula | Purpose |
|---------|---------|---------|
| `total_traffic` | `bytes_sent + bytes_received` | Total data moved in a session |
| `failure_ratio` | `failed_logins / (login_attempts + 1)` | Highlights sessions with a high proportion of failed logins, typical of brute force attempts (the `+1` avoids division by zero) |

**Encoding**

- `attack_category` is label-encoded (`Benign` → 0, `BruteForce` → 1).
- `packet_session_id` is dropped as an identifier.
- Remaining categorical columns are one-hot encoded with `pd.get_dummies`.

**Train/test split**

80% training / 20% testing (`test_size=0.2`, `random_state=42`).

### 3. Models

Both models are `scikit-learn` Random Forests with `n_estimators=100` and `random_state=42`. Each uses every column except the two targets (`attack_category` and `avg_packet_size`) as input features.

| Model | Class | Target | Metrics |
|-------|-------|--------|---------|
| Classifier | `RandomForestClassifier` | `attack_category` | Confusion matrix, classification report, 5-fold cross-validation |
| Regressor | `RandomForestRegressor` | `avg_packet_size` | Mean Squared Error (MSE), R² |

## Results

### Regression – predicting `avg_packet_size`

| Metric | Value |
|--------|-------|
| MSE | 246.52 |
| R² | 0.54 |

An MSE of 246.52 corresponds to a typical error of roughly 15–16 bytes (RMSE ≈ 15.7). An R² of 0.54 means the model explains a little over half of the variation in packet size.

### Classification – `Benign` vs `BruteForce`

Confusion matrix on the 1,200-row test set:

|  | Predicted Benign | Predicted BruteForce |
|--|------------------|----------------------|
| **Actual Benign** | 841 | 6 |
| **Actual BruteForce** | 349 | 4 |

- Overall accuracy: about 70.4% (845 / 1,200)
- The model identifies almost all benign sessions but **misses the vast majority of brute force sessions** (4 of 353 detected).

Accuracy alone is misleading here because the classes are imbalanced (roughly 69% benign). The classifier is not yet reliable for detecting attacks.

## Limitations & Future Work

- **Poor recall on `BruteForce`.** Ideas to improve this:
  - address class imbalance (`class_weight="balanced"`, SMOTE, or resampling)
  - tune hyperparameters (e.g. with `GridSearchCV`)
  - evaluate with precision/recall/F1 and ROC-AUC rather than accuracy
  - gather more labelled attack data
- **Unused scaler.** A `StandardScaler` is created in the script but never applied. Random Forests are not sensitive to feature scale, so this does not affect the results, but it can be removed or used if other model types are tried.
- **Outliers** in `avg_packet_size` were identified but not removed or investigated further.
- Only two classes (`Benign`, `BruteForce`) are present in the data; other attack types would require new data and a multi-class setup.

## Tech Stack

- Python
- pandas, NumPy
- matplotlib, seaborn
- scikit-learn
- Google Colab / Jupyter
