# Football Player Market Value — ML Project

**DAMA Hackathon 2026**

Predicting and explaining football player market values using machine learning, with a secondary transfer-risk classification task and full SHAP-based model interpretability.

---

## Project Structure

```
football-player-valuation-ml/
├── data/
│   └── fifa_player_performance_market_value.csv   # Source dataset (2800 players)
├── notebooks/
│   ├── 01_eda.ipynb                 # Exploratory Data Analysis
│   ├── 02_features.ipynb            # Feature Engineering & Preprocessing
│   ├── 03_regression.ipynb          # Market Value Regression (5 models)
│   └── 04_classification_shap.ipynb # Transfer Risk Classification + SHAP
├── outputs/                         # Generated figures, leaderboards, saved models
├── requirements.txt
└── README.md
```

---

## Dataset

| Feature | Description |
|---|---|
| `age` | Player age |
| `overall_rating` | Current ability rating |
| `potential_rating` | Peak potential rating |
| `matches_played`, `goals`, `assists`, `minutes_played` | Season performance stats |
| `contract_years_left` | Remaining contract length |
| `injury_prone` | Injury susceptibility flag |
| `position` | Playing position (GK, CB, LB, RB, CDM, CM, LW, RW, ST) |
| `nationality`, `club` | Categorical attributes |
| `market_value_million_eur` | **Regression target** (EUR millions) |
| `transfer_risk_level` | **Classification target** (Low / Medium / High) |

---

## Tasks

### Task 1 — Market Value Regression
Predict `market_value_million_eur` using a log-transformed target.
Models evaluated: **Ridge, Random Forest, Gradient Boosting, XGBoost, LightGBM**
Metrics: RMSE, MAE, R²

### Task 2 — Transfer Risk Classification
Predict `transfer_risk_level` (3-class).
Models evaluated: **Logistic Regression, Random Forest, XGBoost, LightGBM**
Metrics: Accuracy, Macro F1, ROC-AUC (OvR)

### Task 3 — Model Interpretability
SHAP values used throughout:
- Global summary / bar plots
- Local waterfall explanations (individual players)
- Dependence plots for top features

---

## Engineered Features

| Feature | Rationale |
|---|---|
| `goals_per_90`, `assists_per_90` | Volume-normalised performance |
| `contributions_p90` | Combined attacking output |
| `rating_gap` | Potential − Overall (development upside) |
| `rating_x_potential` | Interaction term |
| `age_rating_ratio` | Value-per-year-of-age proxy |
| `expiring_soon` | Contract risk flag |
| `position_group` | Broad role: GK / Defender / Midfielder / Attacker |

---

## How to Run

```bash
# Install dependencies
pip install -r requirements.txt

# Run notebooks in order
jupyter lab notebooks/01_eda.ipynb
jupyter lab notebooks/02_features.ipynb   # produces outputs/processed_data.pkl
jupyter lab notebooks/03_regression.ipynb
jupyter lab notebooks/04_classification_shap.ipynb
```

All outputs (figures, leaderboard CSVs, saved models) are written to `outputs/`.

---

## Key Findings

- `overall_rating` and `potential_rating` are the dominant drivers of market value
- Age has a non-linear effect; the 24–28 prime window commands a premium
- `rating_gap` boosts speculative value for young, high-potential players
- `contract_years_left` is the strongest predictor of transfer risk
- Tree-based ensembles (XGBoost / LightGBM) consistently outperform linear baselines
